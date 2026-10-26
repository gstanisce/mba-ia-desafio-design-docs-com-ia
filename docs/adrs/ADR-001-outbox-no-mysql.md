# ADR-001: Padrão outbox no MySQL para eventos de webhook

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)
- **Origem:** [09:04] Bruno, [09:06] Diego, [09:07] Diego, [09:08] Diego, [09:41] Bruno, [09:52] Larissa

## Contexto

Ao mudar o status de um pedido, o `OrderService.changeStatus` executa uma transação Prisma que atualiza `orders`, insere em `order_status_history` e ajusta `products.stockQuantity` (`src/modules/orders/order.service.ts`, linhas 126 a 179). Disparar o webhook com uma chamada HTTP dentro dessa transação faria um cliente lento travar a mudança de status de outros pedidos, e uma falha do cliente poderia causar rollback de uma mudança de status válida ([09:04] Bruno).

O time precisa de uma garantia de que todo status alterado gere um evento, sem que o evento exista quando a transação falhou.

## Decisão

Usar um **padrão outbox** no MySQL existente:

- Dentro da mesma transação que atualiza `orders` e `order_status_history`, o `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que insere uma linha em `webhook_outbox` ([09:41] Bruno, [09:41] Diego).
- A função recebe o cliente de transação (`tx`) e não o repositório inteiro ([09:41] Diego).
- O evento é criado apenas para os endpoints cujo filtro de status contém o `toStatus`. Se nenhum endpoint do cliente quer aquele status, nenhuma linha é inserida ([09:34] Bruno, [09:34] Diego).
- O payload é renderizado no momento da inserção (snapshot). Se o pedido mudar depois, o evento continua refletindo o estado no instante da mudança ([09:52] Larissa, [09:52] Diego).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
- Chave primária `CHAR(36)` com UUID, como no restante do projeto ([09:51] Larissa; `prisma/schema.prisma`, modelos `Order` e `OrderItem`).

## Alternativas consideradas

- **Disparo síncrono no `changeStatus`.** Descartado: a transação já é pesada e uma chamada HTTP lenta bloquearia outros pedidos; também não há como reverter a mudança de status se o cliente estiver fora do ar ([09:04] Bruno, [09:06] Diego: "Síncrono está fora de questão").
- **Redis Streams ou outra fila.** Descartado: exigiria subir mais infraestrutura, e o time é pequeno. Redis Cluster seria overengineering ([09:07] Larissa, [09:07] Diego).
- **Fila em memória com disparo após o commit.** Não discutida na reunião. Descartada por este ADR pois perderia eventos se o processo cair entre o commit e o envio.

## Consequências

**Positivas**

- Atomicidade: se a transação de status commitou, o evento está registrado; se fez rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova para subir.
- Clientes lentos não afetam a mudança de status.

**Negativas e trade-offs**

- A tabela cresce com o tempo. A política de arquivamento das linhas entregues (cerca de 30 dias) está fora do escopo desta fase ([09:08] Diego), então a tabela precisa de um plano futuro.
- A inserção na outbox entra no caminho crítico da transação de status. É uma escrita a mais, e o índice precisa ser mantido.
- Mudanças no `changeStatus` passam a exigir cuidado: a chamada a `publishWebhookEvent` deve estar dentro do `$transaction`. Mover a chamada para fora quebra a garantia ([09:41] Diego: "Se ficar fora da transação, perde a garantia toda").

## Referências

- [ADR-002](ADR-002-worker-polling-processo-separado.md): como os eventos são lidos.
- [ADR-005](ADR-005-at-least-once-com-x-event-id.md): identificador do evento gerado na inserção.
- [FDD](../FDD.md), seção "Fluxo 1: criação do evento na outbox".
