# TRACKER: rastreabilidade dos documentos

Este tracker liga cada decisão, requisito, restrição ou ponto de integração à sua origem.

- **Fonte `TRANSCRICAO`:** a coluna Localização traz o timestamp e o falante, no formato `[hh:mm] Nome`, conforme `TRANSCRICAO.md`.
- **Fonte `CODIGO`:** a coluna Localização traz o caminho real do arquivo no repositório. Os caminhos foram verificados com `find` e `Read`.
- Itens marcados como `[PROPOSTA]` nos documentos não têm linha de origem e não aparecem aqui como decisão. Os que aparecem são derivados de uma decisão ou de um padrão de código identificado.

## Resumo de cobertura

| Métrica | Valor |
|---|---|
| Total de linhas | 125 |
| Linhas com Fonte `TRANSCRICAO` | 102 (82%) |
| Linhas com Fonte `CODIGO` | 23 |

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL, gravado na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-R1 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | `publishWebhookEvent` deve receber o `tx` da transação atual | TRANSCRICAO | [09:41] Bruno |
| ADR-001-R2 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Chamada HTTP síncrona descartada por bloquear mudança de status | TRANSCRICAO | [09:04] Bruno |
| ADR-001-R3 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-R4 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em `status` e `created_at` | TRANSCRICAO | [09:08] Diego |
| ADR-001-R5 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de status aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-002 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-R1 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-R2 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Entry point `src/worker.ts` e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-R3 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Restrição | Ordem garantida só por `order_id`, com um único worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-R4 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Trade-off | Trigger do banco descartado por não notificar processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-R5 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Worker com `PrismaClient` próprio, mesma `DATABASE_URL` | TRANSCRICAO | [09:30] Bruno |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Limite de 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| ADR-003-R1 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Progressão de backoff 1, 5, 30, 120 e 720 minutos | TRANSCRICAO | [09:17] Diego |
| ADR-003-R2 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Timeout de 10 segundos por chamada | TRANSCRICAO | [09:42] Diego |
| ADR-003-R3 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela separada `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| ADR-003-R4 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual por `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| ADR-003-R5 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | Três tentativas descartadas por cobrir só 30 minutos | TRANSCRICAO | [09:16] Diego |
| ADR-003-R6 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| ADR-003-R7 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Restrição | Replay exige papel `ADMIN` e auditoria | TRANSCRICAO | [09:36] Sofia |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, em `X-Signature` | TRANSCRICAO | [09:22] Sofia |
| ADR-004-R1 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-R2 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| ADR-004-R3 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | URL do webhook obrigatoriamente `https` | TRANSCRICAO | [09:23] Sofia |
| ADR-004-R4 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004-R5 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| ADR-005-R1 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | UUID `X-Event-Id` gerado na entrada na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-R2 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado por exigir coordenação | TRANSCRICAO | [09:25] Diego |
| ADR-005-R3 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Restrição | Deduplicação fica a cargo do cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-R1 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo em `src/modules/webhooks` com a estrutura padrão | TRANSCRICAO | [09:27] Bruno |
| ADR-006-R2 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Códigos de erro com prefixo `WEBHOOK_` | TRANSCRICAO | [09:29] Larissa |
| ADR-006-C1 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão de código | Classe base de erro `AppError` | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-C2 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão de código | Erros HTTP reutilizáveis (`ValidationError`, `ConflictError`, `UnprocessableEntityError`) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C3 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão de código | Logger Pino com lista de redação | CODIGO | src/shared/logger/index.ts |
| ADR-006-C4 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão de código | Middleware de erro que trata `AppError`, `ZodError` e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-C5 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão de código | `requireRole` já existente | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência de entrega abaixo de 10 segundos para o cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Nenhum evento perdido quando a transação confirma | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Entrega até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes consultam `GET /orders` repetidamente | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Restrição de negócio | Atlas pode migrar para concorrente se não houver entrega no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Registrar evento na outbox na mesma transação de mudança de status | TRANSCRICAO | [09:41] Bruno |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Criar endpoint com URL e receber secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtrar eventos pela lista de status do endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar, remover e listar endpoints | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotacionar secret pela API | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Payload com campos definidos, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 de cada entrega | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Headers `X-Event-Id`, `X-Signature` e `X-Timestamp` | TRANSCRICAO | [09:44] Diego |
| PRD-FR-08b | docs/PRD.md | Requisito Funcional | Header `X-Webhook-Id` | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Até 5 tentativas com backoff | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | DLQ persistida para falhas definitivas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay de DLQ restrito a `ADMIN` | TRANSCRICAO | [09:35] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico de entregas consultável pelo cliente | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Timeout de 10 segundos trata cliente lento como falha | TRANSCRICAO | [09:42] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Recusar URL `http` com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Recusar payload acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não pode bloquear mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Ordem por `order_id` com single worker | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Reuso de padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-FORA-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente em falhas repetidas, adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | docs/PRD.md | Fora de escopo | Painel visual do cliente, projeto separado | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-03 | docs/PRD.md | Fora de escopo | Webhooks de entrada, somente saída | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-04 | docs/PRD.md | Fora de escopo | Garantia de ordem global, descartada | TRANSCRICAO | [09:14] Marcos |
| PRD-FORA-05 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues, adiado | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-06 | docs/PRD.md | Fora de escopo | Limite de taxa de envio, observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| PRD-FORA-07 | docs/PRD.md | Fora de escopo | Múltiplos workers, adiado | TRANSCRICAO | [09:13] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Confirmação de prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal, com destaque para at-least-once | TRANSCRICAO | [09:26] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono dentro de `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams, por exigir infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do banco para acionar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | docs/RFC.md | Decisão (observar) | Limite de taxa de envio, sem implementação nesta fase | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Decisão | Verificação do limite de 64 KB na criação do evento, sem rollback | TRANSCRICAO | [09:24] Larissa |
| RFC-QA-03 | docs/RFC.md | Decisão | Um envio inicial mais cinco retentativas | TRANSCRICAO | [09:17] Diego |
| RFC-QA-04 | docs/RFC.md | Decisão | Tolerância de 300 s para o `X-Timestamp` | TRANSCRICAO | [09:44] Diego |
| RFC-QA-05 | docs/RFC.md | Decisão | `customer_id` no corpo do POST e como filtro na listagem | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-06 | docs/RFC.md | Decisão | Isolamento entre clientes: API liberada só com vínculo usuário-cliente | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker lê pendentes a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff e limite de tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ e replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-CRIT-01 | docs/FDD.md | Critério técnico | Falha na inserção da outbox faz rollback da mudança de status | TRANSCRICAO | [09:41] Diego |
| FDD-DEC-01 | docs/FDD.md | Decisão | Limite de 64 KB verificado na criação do evento, sem rollback da mudança de status | TRANSCRICAO | [09:23] Sofia |
| FDD-DEC-02 | docs/FDD.md | Decisão | Retenção das últimas 100 entregas por endpoint | TRANSCRICAO | [09:34] Marcos |
| FDD-DEC-03 | docs/FDD.md | Decisão | `X-Timestamp` em Unix com tolerância de 300 s | TRANSCRICAO | [09:44] Diego |
| FDD-DEC-04 | docs/FDD.md | Decisão | Rotação de secret sem exigência de papel ADMIN | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTR-01 | docs/FDD.md | Contrato | `POST /webhooks` para cadastro | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTR-02 | docs/FDD.md | Contrato | `GET /webhooks` para listar endpoints de um cliente | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTR-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` para editar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTR-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` para remover | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTR-05 | docs/FDD.md | Contrato | `POST /webhooks/:id/rotate-secret` para rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTR-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` para histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTR-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Larissa |
| FDD-PAY-01 | docs/FDD.md | Contrato | Campos do payload do evento | TRANSCRICAO | [09:43] Diego |
| FDD-PAY-02 | docs/FDD.md | Contrato | Payload sem lista de itens | TRANSCRICAO | [09:44] Bruno |
| FDD-HDR-01 | docs/FDD.md | Contrato | Header `X-Event-Id` com UUID do evento | TRANSCRICAO | [09:25] Diego |
| FDD-HDR-02 | docs/FDD.md | Contrato | Header `X-Webhook-Id` com UUID do endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` para URL não `https` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | Prefixo `WEBHOOK_` em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs com Pino compartilhado | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redação de secret em logs, pelo risco de vazamento já ocorrido | TRANSCRICAO | [09:22] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | `changeStatus` passa a chamar `publishWebhookEvent` na transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Transições e estoque reutilizados como estão | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros HTTP reutilizados no módulo | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | `authenticate` e `requireRole('ADMIN')` no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Fábrica `createPrismaClient()` reutilizada pelo worker | CODIGO | src/config/database.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Padrão de bootstrap e encerramento espelhado em `src/worker.ts` | CODIGO | src/server.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Registro do router `/webhooks` | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Modelo de rotas para `webhook.routes.ts` | CODIGO | src/modules/orders/order.routes.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Modelo de schemas Zod para `webhook.schemas.ts` | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Novos modelos com UUID `CHAR(36)`, como `Order` | CODIGO | prisma/schema.prisma |
| FDD-INT-11 | docs/FDD.md | Integração | Novo script `worker` | CODIGO | package.json |
| FDD-INT-12 | docs/FDD.md | Integração | Formato paginado para o histórico de entregas | CODIGO | src/shared/http/response.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Ampliação de `redactPaths` para secret e `x-signature` | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Request logger para correlação de tracing | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-15 | docs/FDD.md | Decisão | `snake_case` no módulo de webhooks, divergindo do `camelCase` do código | CODIGO | prisma/schema.prisma |
| FDD-INT-16 | docs/FDD.md | Integração | `OrderService.list` como referência de resposta paginada | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-17 | docs/FDD.md | Integração | Testes de referência com Vitest | CODIGO | tests/orders.test.ts |

## Itens revisados e não rastreados

Itens que aparecem na transcrição mas não viraram requisito, e onde estão tratados:

| Item | Tratamento | Localização |
|---|---|---|
| Dashboard visual para o cliente | Fora de escopo (PRD-FORA-02) | [09:40] Larissa |
| Limite de taxa de envio | Questão em aberto (RFC-QA-01) e fora de escopo (PRD-FORA-06) | [09:39] Larissa |
| Validação de TLS | Tratada como requisito de validação (PRD-FR-14), mesmo com a observação de que não é decisão arquitetural | [09:23] Sofia |

Itens sem origem na reunião, marcados como estimativa do autor:

- Probabilidade e impacto dos riscos no PRD estão marcados `[ESTIMATIVA]`. A reunião não os quantificou, e por isso os riscos ficam fora da tabela como decisão.
- A meta de consultas evitadas em `GET /orders` (PRD-OBJ-04) está marcada como "a definir", porque a reunião não a quantificou.
