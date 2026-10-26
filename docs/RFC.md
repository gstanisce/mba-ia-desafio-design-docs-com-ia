# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Guilherme Correa |
| **Status** | Em revisão |
| **Data** | 2026-10-26 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md), [FDD](FDD.md), [TRACKER](TRACKER.md), [ADRs](adrs/) |

## TL;DR

Propomos enviar notificações de mudança de status de pedido para clientes B2B por webhook, em vez de polling em `GET /orders`. A mudança de status grava um evento em uma tabela de outbox dentro da mesma transação. Um worker separado lê essa tabela a cada 2 segundos e entrega as chamadas HTTP com HMAC-SHA256, retry com backoff e garantia at-least-once. Eventos que falham definitivamente vão para uma dead letter queue com replay manual.

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) consultam `GET /orders` periodicamente para saber se o status de seus pedidos mudou. Isso deixa a integração lenta e custosa para eles. A Atlas condicionou a continuidade ao atendimento até o fim do trimestre ([09:00] Marcos). Para os clientes, "tempo real" significa atualização em até 10 segundos ([09:02] Marcos).

## Proposta

A solução tem quatro partes:

1. **Captura do evento.** Dentro da transação de `OrderService.changeStatus`, um evento é gravado na outbox. A mudança de status e o evento são atômicos ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
2. **Entrega.** Um worker em processo separado consulta a outbox em polling de 2 segundos e envia as requisições ([ADR-002](adrs/ADR-002-worker-polling-processo-separado.md)).
3. **Confiabilidade.** Falhas geram retry com backoff, e falhas definitivas vão para uma DLQ com replay manual por administrador ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)).
4. **Segurança e semântica.** Cada requisição é assinada com HMAC-SHA256 usando uma secret por endpoint, com rotação. A entrega é at-least-once, com `X-Event-Id` para deduplicação ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

O módulo segue a estrutura dos demais módulos da API e reutiliza o tratamento de erros e o logger existentes ([ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)).

Escopo: somente saída de eventos, da plataforma para o cliente. Os clientes não enviam webhooks para a plataforma ([09:02] Marcos).

## Alternativas consideradas

**Disparo síncrono dentro de `changeStatus`.** Descartado. Um cliente lento travaria a mudança de status de outros pedidos, e um cliente fora do ar não deve impedir a mudança de status ([09:04] Bruno). Trade-off: a simplicidade de implementação é menor do que o risco de acoplar a transação de pedido à disponibilidade de terceiros.

**Fila dedicada (Redis Streams).** Descartado. Exige infraestrutura nova para um time pequeno, e o padrão outbox atende ao requisito com o MySQL existente ([09:07] Larissa, [09:07] Diego). Trade-off: uma fila dedicada daria mais throughput, ao custo de operar mais um componente.

**Trigger do banco para acionar o worker.** Descartado. O MySQL não notifica processos externos, e os mecanismos improvisados foram considerados inadequados ([09:09] Diego). Trade-off: polling introduz até 2 segundos de latência, dentro do limite de 10 segundos.

**Exactly-once.** Descartado. Exigiria coordenação dos dois lados. At-least-once com identificador por evento é o padrão de mercado ([09:25] Diego). Trade-off: o cliente precisa deduplicar.

## Decisões tomadas na revisão

As questões levantadas na reunião foram decididas abaixo, com o motivo de cada uma. Os revisores devem confirmar ou contestar.

1. **Contagem de tentativas.** Um envio inicial mais cinco retentativas, nos intervalos de 1, 5, 30, 120 e 720 minutos. Motivo: são os cinco intervalos que a reunião detalhou, e o total de quase 15 horas citado em [09:17] Diego só fecha com eles. Detalhes em [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md).
2. **Nomes em `snake_case`** em todo o módulo de webhooks. Motivo: é o contrato externo e foi o que a reunião definiu. A divergência com o restante da API está no [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md).
3. **Posição de `customer_id`.** No corpo do `POST /webhooks` e como filtro de query na listagem. Motivo: o `POST` já tem corpo com URL e filtro; na listagem, o cliente é um filtro, não um recurso do caminho.
4. **Limite de 64 KB.** Verificado na criação do evento. Acima disso, o evento não é gravado e o erro é registrado. A mudança de status não é revertida. Motivo: reverter a mudança de status por causa do tamanho de um evento violaria a regra de [09:04] Bruno.
5. **Janela do `X-Timestamp`.** Unix em segundos, com tolerância de 300 segundos. Motivo: é o padrão usado por provedores de webhook, e como cada tentativa tem seu próprio `X-Timestamp`, retentativas tardias continuam válidas.
6. **Limite de taxa de envio.** Observar, sem implementar. Motivo: decisão da reunião ([09:39] Larissa). A métrica de acúmulo do FDD permite decidir depois.
7. **Isolamento entre clientes.** Nesta fase, a API de configuração exige apenas autenticação, como definido em [09:37] Sofia. O modelo atual não liga `User` a `Customer` (`prisma/schema.prisma`), então um usuário autenticado conseguiria, em tese, alterar endpoints de outro cliente. A reunião não tratou o tema. **Decisão:** a API só é liberada a clientes externos depois que existir o vínculo entre usuário e cliente. Essa condição entra como dependência no PRD.

## Questões em aberto

1. **Escala.** Com um único worker, a ordem é garantida por `order_id`. A forma de escalar (particionamento ou lock) foi adiada ([09:13] Diego). Condição para reabrir: volume que o worker único não consiga atender.
2. **Arquivamento da outbox.** As linhas entregues devem ser arquivadas após cerca de 30 dias, mas a política ficou fora desta fase ([09:08] Diego).

## Impacto e riscos

- **Impacto no código:** alteração em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), novo módulo `src/modules/webhooks`, novo entry point `src/worker.ts`, novos modelos no Prisma e uma nova migração. Nenhuma alteração em contratos existentes.
- **Impacto em banco:** novas tabelas e uma escrita adicional por mudança de status que tenha assinante.
- **Riscos principais:** vazamento de secret, clientes que não deduplicam, DLQ acumulando por indisponibilidade prolongada e prazo apertado. Detalhados no [PRD](PRD.md).

## Decisões relacionadas

- [ADR-001](adrs/ADR-001-outbox-no-mysql.md): Padrão outbox no MySQL
- [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md): Worker em processo separado com polling
- [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md): Retry com backoff e DLQ
- [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md): HMAC-SHA256 com secret por endpoint
- [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md): At-least-once com `X-Event-Id`
- [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md): Reuso dos padrões existentes
