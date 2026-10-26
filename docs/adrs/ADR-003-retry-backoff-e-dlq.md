# ADR-003: Retry com backoff exponencial e Dead Letter Queue em tabela separada

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno), Marcos (Product Manager)
- **Origem:** [09:14] Larissa, [09:15] Diego, [09:15] Bruno, [09:16] Bruno, [09:16] Diego, [09:16] Larissa, [09:17] Diego, [09:17] Marcos, [09:17] Larissa, [09:18] Diego, [09:18] Bruno, [09:18] Diego, [09:19] Larissa, [09:42] Sofia, [09:42] Diego

## Contexto

Um cliente pode estar fora do ar. O worker precisa tentar de novo sem bombardear o endpoint e sem deixar o evento pendente para sempre. Um cliente teve indisponibilidade de duas horas em manutenção planejada ([09:16] Diego), então uma política de três tentativas em 30 minutos seria agressiva demais.

## Decisão

- **Timeout** de 10 segundos por chamada HTTP. Um cliente que não responde nesse prazo é tratado como falha e entra em retry ([09:42] Diego).
- **Backoff exponencial** com a progressão **1 min, 5 min, 30 min, 2 h, 12 h**. Entre a primeira falha e a última tentativa, o intervalo total é de quase 15 horas ([09:17] Diego, [09:17] Marcos).
- **Limite de 5 tentativas**, conforme decidido por Larissa ([09:17] Larissa).
- **DLQ** em tabela separada `webhook_dead_letter`, com o payload, o motivo da falha e o timestamp. Não é marcada na própria outbox, para manter a leitura da outbox principal limpa e para servir de evidência de debug e reprocessamento ([09:18] Diego).
- **Replay manual** pelo endpoint `POST /admin/webhooks/dead-letter/:id/replay`. Ele recoloca o evento na outbox como pendente. Exige role `ADMIN`, e a ação precisa ser registrada para auditoria ([09:18] Diego, [09:35] Larissa, [09:36] Sofia).

## Alternativas consideradas

- **Três tentativas.** Descartado: cobre apenas 30 minutos de indisponibilidade ([09:16] Diego).
- **Retry indefinido com backoff.** Descartado: o evento ficaria pendurado para sempre se o cliente sumisse ([09:15] Diego).
- **Marcar como `failed` na própria outbox.** Descartado em favor da tabela separada, pelos motivos de leitura e evidência ([09:18] Diego).

## Consequências

**Positivas**

- Cobertura de uma janela de até cerca de 15 horas, o que atende clientes com manutenção planejada.
- Falhas definitivas ficam preservadas e podem ser reprocessadas sem recriar o pedido.
- Auditoria do replay via identificação do usuário `ADMIN`.

**Negativas e trade-offs**

- Um cliente fora do ar por mais de 15 horas gera eventos em DLQ que exigem ação manual. Marcos aceitou o risco: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- Notificação ao cliente sobre falha repetida (por e-mail) está fora desta fase ([09:37] Larissa). Não há alerta automático para o cliente.

**Contagem de tentativas (decidida)**

- A reunião fala em "5 tentativas" e lista **cinco intervalos** (1, 5, 30, 120 e 720 minutos). Decisão: um envio inicial mais cinco retentativas, uma para cada intervalo. Motivo: a reunião detalhou os cinco intervalos, e o total de quase 15 horas citado em [09:17] Diego só fecha com eles. Tratar como cinco envios no total deixaria o intervalo de 12 horas sem uso. Se o time preferir o contrário, a mudança é de uma linha na tabela de backoff.

## Referências

- [ADR-002](ADR-002-worker-polling-processo-separado.md)
- [FDD](../FDD.md), seções "Fluxo 3: retry" e "Fluxo 4: DLQ e replay".
