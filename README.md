# Ponte de Eventos -> CRM (n8n)

Camada de integração que recebe eventos de Instagram e WhatsApp e entrega cada um deles no CRM **uma única vez**, sem perda silenciosa.

O problema que este projeto resolve não é "conectar duas APIs" — isso um webhook simples resolve. O problema é o que acontece quando a rede oscila, quando a origem reenvia o mesmo evento três vezes, quando o CRM devolve 500 ou quando o payload chega com um campo obrigatório faltando. Sem tratamento, o resultado é sempre um dos dois: lead duplicado no funil ou lead perdido sem ninguém perceber.

---

## Garantias do contrato

| Risco real | Como a ponte trata |
| --- | --- |
| Evento reenviado pela origem | Chave de idempotência determinística; o segundo envio é descartado e registrado como duplicata |
| CRM fora do ar (5xx / timeout) | Retry com backoff progressivo: 1 min, 5 min, 15 min |
| Retry esgotado | Status `falha`, evento vai para a fila de reprocessamento e dispara alerta |
| Campo obrigatório ausente | Status `invalido`, payload gravado integralmente para auditoria, alerta enviado, CRM não é chamado |
| Falha silenciosa | Todo evento termina em um status final explícito; nada fica sem desfecho |
| Necessidade de reprocessar | Runbook documentado em `docs/runbook-reprocessamento.md` |

## Arquitetura

```mermaid
flowchart TD
  A[Instagram / WhatsApp] -->|webhook| B[Webhook: Evento In]
  B --> C[Normalização do payload]
  C --> D{Chave de idempotência já existe?}
  D -->|sim| E[Descarta e registra duplicata]
  D -->|não| F[Grava evento com status recebido]
  F --> G[POST no CRM]
  G -->|2xx| H[status entregue]
  G -->|4xx| I[status invalido + alerta]
  G -->|5xx ou timeout| J[Retry com backoff 1/5/15 min]
  J -->|tentativas esgotadas| K[status falha + fila de reprocessamento + alerta]
```

## Chave de idempotência

A chave é calculada na normalização, antes de qualquer escrita:

```
chave = sha256( canal + ":" + id_externo_do_evento )
```

Quando a origem não fornece um identificador de evento — acontece em parte dos webhooks do Instagram — a ponte usa uma chave derivada estável:

```
chave = sha256( canal + ":" + remetente + ":" + timestamp_truncado_ao_minuto + ":" + sha256(texto) )
```

A chave tem restrição de unicidade na tabela de eventos. A verificação acontece **antes** da chamada ao CRM, e a gravação do evento acontece **antes** da entrega. Nessa ordem, uma queda no meio do caminho deixa o evento recuperável em vez de invisível.

> **Nota de implementação:** o sandbox do nó Code do n8n bloqueia `require('crypto')` e não expõe o Web Crypto global, então o SHA-256 é uma implementação em JavaScript puro dentro do próprio nó, validada contra o SHA-256 de referência (incluindo textos com acento e emoji). Sem dependência externa.

## Retry e fila de reprocessamento

- `5xx` e timeout são tratados como falha transitória: reagenda com backoff de 1 min, 5 min e 15 min.
- `4xx` é falha permanente: não faz retry, porque repetir não muda o resultado. Marca `invalido` e alerta.
- Depois da terceira tentativa: status `falha`, o registro mantém o payload original e o último erro, e o evento entra na fila de reprocessamento manual.
- O contador de tentativas vem do `$runIndex` do próprio n8n, e não de um campo carregado entre os nós — foi o que corrigiu um bug em que o contador reiniciava a cada volta do laço e o retry nunca esgotava.

> **Backoff no arquivo publicado:** o JSON deste repositório está com o nó de espera em **segundos** (1, 5, 15), porque foi assim que os testes locais rodaram — cerca de 8 segundos em vez de 21 minutos. Em produção troque a unidade do nó `Backoff progressivo` para **minutos**. A lógica é a mesma; só a unidade muda.

## Alertas

Um único nó de notificação centraliza os três casos que exigem olho humano: payload inválido, retry esgotado e volume anormal de duplicatas (sinal de loop na origem). A mensagem traz canal, chave de idempotência, status e último erro — o suficiente para decidir sem abrir o n8n.

## Estrutura do repositório

```
fluxo/
  ponte-eventos-crm.json         fluxo n8n pronto para importar
docs/
  esquema-tabela-eventos.md      campos, tipos e indices da tabela de eventos
  runbook-reprocessamento.md     como reprocessar a fila de falhas
  resultado-dos-testes.md        o que foi executado e o que cada cenario devolveu
testes/
  crm-stub.json                  CRM falso em n8n: responde 201, 422 ou 503 sob comando
  injetor-testes.json            dispara os 6 cenarios contra o webhook da ponte
  01-evento-valido.json
  02-evento-duplicado.json
  03-campo-obrigatorio-ausente.json
  04-crm-indisponivel.json
  05-payload-malformado.json
```

## Como rodar

1. Suba o n8n com Docker:

```
docker run -d --name n8n --restart unless-stopped -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n:2.32.7
```

2. Importe `fluxo/ponte-eventos-crm.json` em **Workflows -> Import from File**.
3. Crie a Data Table de eventos conforme `docs/esquema-tabela-eventos.md`.
4. Configure a credencial HTTP do CRM de destino. **As credenciais e chaves de API são sempre do cliente** — nenhum segredo vive neste repositório.
5. Ative o fluxo e envie os payloads de `testes/` para a URL do webhook.

## Cenários de teste (executados)

O fluxo foi rodado de ponta a ponta em um n8n 2.32.7 local, com um stub de CRM em
`testes/crm-stub.json` e um injetor de cenários em `testes/injetor-testes.json`.
O detalhamento e as evidências estão em `docs/resultado-dos-testes.md`.

| Cenário | Resultado esperado | Resultado obtido |
| --- | --- | --- |
| Evento válido | `entregue` | `entregue`, http 201, 1 tentativa |
| Mesmo evento reenviado | `duplicado`, sem segunda escrita no CRM | `duplicado`, CRM não foi chamado |
| Campo obrigatório ausente | `invalido` + alerta, CRM não é chamado | `invalido` — campos obrigatórios ausentes: remetente, texto |
| CRM responde 422 | `invalido`, sem retry | `invalido`, http 422, 1 tentativa |
| CRM instável (503, 503, 201) | `entregue` depois do retry | `entregue` na 3ª tentativa, http 201 |
| CRM fora do ar (503 sempre) | `falha` + fila + alerta | `falha`, 3/3 tentativas, `na_fila_reprocessamento = true` |

Nenhum evento terminou sem status final, e cada desfecho gerou exatamente uma linha na Data Table.

## Limitações e escopo

- A entrega é feita em um endpoint HTTP genérico de CRM. Adaptar para um CRM específico é trocar um nó, não reescrever a ponte.
- O backoff de 1/5/15 min é o padrão e deve ser ajustado ao SLA do CRM de destino.
- A ponte não interpreta nem responde mensagens: ela garante a entrega do evento. A camada de atendimento com IA é outro projeto.
- O reprocessamento é manual por decisão de projeto: reprocessar automaticamente uma falha permanente só multiplica o problema.

---

## English summary

Reliable event bridge between Instagram/WhatsApp webhooks and a CRM, built on n8n. It guarantees at-most-once delivery through a deterministic idempotency key, retries transient CRM failures with progressive backoff (1/5/15 min), does not retry permanent failures, and routes exhausted events to a manual reprocessing queue with alerting. Every event ends in an explicit final state, so nothing fails silently. The repository ships the importable workflow, the event table schema, a reprocessing runbook and five end-to-end test payloads.

## Licença

MIT — ver `LICENSE`.
