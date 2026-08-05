# Resultado dos testes

Execucao de ponta a ponta em n8n **2.32.7** (Docker, volume `n8n_data`), com os tres
workflows publicados na mesma instancia:

| Workflow | Papel |
| --- | --- |
| `Ponte de Eventos -> CRM` | o fluxo em teste, webhook de producao `POST /webhook/ponte-eventos` |
| `CRM Stub (teste local)` | CRM falso, webhook `POST /webhook/crm-stub`, responde 201, 422 ou 503 conforme o texto da mensagem |
| `Injetor de Testes (ponte)` | gatilho manual que dispara os 6 cenarios, um a cada 1,5 s |

## Como o stub controla a falha

| Trecho no texto da mensagem | Resposta do stub |
| --- | --- |
| `FALHA_PERMANENTE` | 422 sempre |
| `FALHA_SEMPRE` | 503 sempre |
| `FALHA_TRANSITORIA` | 503 nas duas primeiras tentativas, 201 na terceira |
| qualquer outro | 201 |

O stub conta as tentativas por chave de idempotencia em `$getWorkflowStaticData`, o que
permite simular um CRM que se recupera sozinho.

## O que cada cenario devolveu

| # | Cenario | status | tentativas | http_status | na_fila_reprocessamento |
| --- | --- | --- | --- | --- | --- |
| 01 | evento valido | `entregue` | 1 | 201 | false |
| 02 | mesmo evento reenviado | `duplicado` | 0 | 0 | false |
| 03 | remetente e texto vazios | `invalido` | 0 | 0 | false |
| 04 | CRM responde 422 | `invalido` | 1 | 422 | false |
| 05 | CRM instavel (503, 503, 201) | `entregue` | 3 | 201 | false |
| 06 | CRM fora do ar (503 sempre) | `falha` | 3 | 503 | **true** |

Motivos gravados, na ordem:

- 02: `chave de idempotencia ja processada`
- 03: `campos obrigatorios ausentes: remetente, texto`
- 04: `CRM rejeitou o evento (422): falha permanente, sem retry`
- 05: `entregue na tentativa 3`
- 06: `tentativas de entrega esgotadas (3/3): falha transitoria no CRM (503) na tentativa 3`

Nos cenarios 02 e 03 o CRM nao foi chamado: o `http_status` gravado e 0 porque o evento
nem chegou ao nó de entrega.

## Duas correcoes que os testes revelaram

**1. `require('crypto')` e bloqueado no no Code.** A primeira execucao falhou com
`Module 'crypto' is disallowed`. A segunda tentativa, usando o Web Crypto global,
falhou com `crypto is not defined`. A solucao foi escrever o SHA-256 em JavaScript puro
dentro do proprio no, e conferir o resultado contra o SHA-256 de referencia com textos
ASCII, acentuados e com emoji.

**2. O contador de tentativas reiniciava a cada volta do laco.** O classificador lia o
estado anterior da saida da normalizacao, que nao muda entre as voltas — ou seja,
`tentativas` voltava a 1 toda vez e o retry nunca esgotava. Passou a usar `$runIndex`,
que e o contador de execucoes do proprio no. O cenario 06 existe justamente para provar
que agora o retry termina em `falha` depois de 3 tentativas.

## Diferenca em relacao a producao

O no `Backoff progressivo` esta em **segundos** (1, 5, 15) no arquivo publicado, para que
a bateria completa rode em cerca de 8 segundos. Em producao a unidade deve ser
**minutos**. Nada mais muda.

O `dataTableId` gravado no fluxo aponta para a Data Table desta instancia. Ao importar em
outro n8n, reselecione a sua tabela no no `Registra evento (Data Table)`.

## Modelo do registro

A Data Table funciona como um livro-razao: **uma linha por (chave de idempotencia,
desfecho)**, usando a coluna `chave_registro`. E por isso que o evento duplicado do
cenario 02 nao sobrescreve o registro da entrega do cenario 01 — as duas linhas existem,
com a mesma `chave` e desfechos diferentes.
