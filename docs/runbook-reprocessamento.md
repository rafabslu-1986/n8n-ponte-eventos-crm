# Runbook: reprocessamento da fila de falhas

Este documento é para quem está de plantão, não para quem escreveu o fluxo. Ele assume que você recebeu um alerta e precisa decidir o que fazer.

## 1. Identificar o que falhou

Consulte a tabela de eventos filtrando por `na_fila_reprocessamento = true`. O campo `status` diz de qual problema se trata:

- `falha` — o CRM não respondeu ou devolveu 5xx nas três tentativas. Problema do destino, o evento está íntegro.
- `invalido` — o payload não passou na validação ou o CRM devolveu 4xx. Problema da origem ou do mapeamento; reprocessar sem corrigir vai falhar de novo.

## 2. Confirmar que o CRM voltou

Antes de reprocessar qualquer coisa, faça uma chamada de teste ao endpoint do CRM. Reprocessar em cima de um destino ainda fora do ar só consome tentativas e polui os alertas.

## 3. Reprocessar

O reprocessamento reenvia o `payload_original` para o mesmo webhook da ponte. Isso é seguro por construção: a chave de idempotência é determinística, então o evento reentra pelo mesmo caminho e não gera duplicata no CRM se de fato já tiver sido entregue.

Ordem recomendada:

1. Reprocesse **um** evento e confirme que ele terminou em `entregue`.
2. Só então reprocesse o lote, do mais antigo para o mais recente.
3. Confira se a fila zerou consultando novamente `na_fila_reprocessamento = true`.

### Reenvio manual de um evento

```bash
curl -X POST "$URL_WEBHOOK_PONTE" \
  -H "Content-Type: application/json" \
  -d @payload_original.json
```

## 4. Quando NÃO reprocessar

- Status `invalido` por campo obrigatório ausente: o evento não tem a informação necessária. Corrija a origem; reprocessar não cria o dado que nunca chegou.
- Status `invalido` por 4xx do CRM: verifique o contrato do endpoint antes. Repetir uma requisição rejeitada devolve a mesma rejeição.
- Eventos com mais de 72 horas: confirme com o time de vendas se o lead ainda faz sentido antes de injetá-lo no funil.

## 5. Volume anormal de duplicatas

Muitos eventos em `duplicado` não é erro da ponte — é ela funcionando. Mas indica que a origem está reenviando. Verifique se o webhook da origem está recebendo o 200 de confirmação; um webhook que não recebe confirmação reenvia por conta própria.

## 6. Registro

Anote em cada rodada de reprocessamento: quantos eventos, qual a causa raiz e o que foi corrigido. A fila de falhas é um sintoma; o runbook só termina quando a causa foi tratada.

## Marcar como resolvido

Depois do reprocessamento bem-sucedido, atualize o registro antigo com `na_fila_reprocessamento = false` e mantenha o histórico. Não apague eventos falhos: eles são a evidência de que o problema existiu e de como foi resolvido.
