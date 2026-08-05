# Esquema da tabela de eventos

Uma linha por evento recebido. A tabela é a fonte de verdade do fluxo: se um evento não está aqui, ele não foi processado.

## Campos

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `chave` | string (64) | sim | Chave de idempotência (sha256). **Restrição de unicidade.** |
| `canal` | string | sim | `instagram` ou `whatsapp`. |
| `remetente` | string | sim | Identificador do remetente na origem. |
| `texto` | texto | sim | Conteúdo da mensagem. |
| `id_externo` | string | não | Id do evento na origem, quando fornecido. |
| `timestamp_origem` | string ISO 8601 | sim | Momento do evento na origem (ou da chegada, se a origem não informar). |
| `recebido_em` | string ISO 8601 | sim | Momento em que a ponte recebeu o evento. |
| `status` | string | sim | `entregue`, `duplicado`, `invalido` ou `falha`. |
| `motivo` | texto | não | Explicação legível do status. |
| `tentativas` | inteiro | sim | Número de tentativas de entrega (0 a 3). |
| `http_status` | inteiro | não | Último código HTTP devolvido pelo CRM. |
| `ultimo_erro` | texto | não | Corpo do erro, truncado em 500 caracteres. |
| `na_fila_reprocessamento` | booleano | sim | `true` quando o evento precisa de reprocessamento manual. |
| `finalizado_em` | string ISO 8601 | sim | Momento em que o evento recebeu status final. |
| `payload_original` | texto (JSON) | sim | Payload cru, para auditoria e reprocessamento. |

## Índices

- **Único** em `chave` — é o que garante a idempotência no nível do armazenamento, não só na lógica do fluxo.
- Índice em `status` — a fila de reprocessamento é uma consulta por `status = falha`.
- Índice em `recebido_em` — usado nos relatórios de volume e na investigação de picos de duplicata.

## Estados possíveis

| Status | Significado | Ação esperada |
| --- | --- | --- |
| `entregue` | CRM respondeu 2xx | Nenhuma. |
| `duplicado` | Chave já processada antes | Nenhuma. Volume alto indica loop na origem. |
| `invalido` | Campo obrigatório ausente ou CRM devolveu 4xx | Corrigir a origem ou o mapeamento. Retry não resolve. |
| `falha` | Três tentativas esgotadas contra 5xx/timeout | Reprocessar pelo runbook. |

Não existe estado intermediário persistido como final. Todo evento sai do fluxo em um dos quatro estados acima — é isso que impede a falha silenciosa.

## Onde plugar

No fluxo importado, o nó `Registra evento (Data Table)` é um marcador. Substitua por um nó de Data Table do n8n, Postgres ou Supabase com o esquema acima e a restrição de unicidade em `chave`. O nó recebe o item já no formato final — não há transformação pendente.

## Nota sobre o cache em memória

O nó de normalização mantém um cache das últimas 5.000 chaves em `workflowStaticData` para descartar duplicatas sem custo de consulta. Esse cache é uma otimização, não a garantia: a garantia é a restrição de unicidade na tabela. Em volume alto, troque a verificação do cache por uma consulta direta à tabela.
