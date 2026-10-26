# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Guilherme Correa |
| **Status** | Em revisão |
| **Data** | 2026-10-26 |
| **Relacionados** | [PRD](PRD.md), [RFC](RFC.md), [ADRs](adrs/), [TRACKER](TRACKER.md) |

> **Convenção de rastreabilidade.** Cada item é marcado como:
> - **[REUNIÃO hh:mm Nome]**: decisão ou requisito discutido na reunião.
> - **[CÓDIGO caminho]**: fato verificado no repositório.
> - **[PROPOSTA]**: detalhe de implementação que não foi discutido. Precisa de aprovação antes de ser tratado como decisão.

## Contexto e motivação técnica

O OMS não tem mecanismo de notificação externa, filas nem eventos. O único ponto de mudança de status é `OrderService.changeStatus`, que executa uma transação Prisma com `orders`, `order_status_history` e estoque ([CÓDIGO src/modules/orders/order.service.ts, linhas 126 a 179]). O webhook precisa capturar a mudança nesse ponto sem alterar o comportamento transacional existente.

## Objetivos técnicos

- Gravar um evento na outbox na mesma transação da mudança de status ([REUNIÃO 09:41 Bruno]).
- Entregar o evento em até 10 segundos após o commit, na maioria dos casos ([REUNIÃO 09:02 Marcos]).
- Garantir at-least-once com identificador único por evento ([REUNIÃO 09:26 Larissa]).
- Assinar cada entrega com HMAC-SHA256 e secret por endpoint ([REUNIÃO 09:22 Sofia]).
- Preservar os dados de cada tentativa para consulta pelo cliente ([REUNIÃO 09:34 Marcos]).

## Escopo e exclusões

**Incluso**

- Captura de eventos de mudança de status.
- Cadastro, edição, remoção, listagem e rotação de secret de endpoints.
- Worker de entrega com retry, DLQ e replay.
- Consulta de histórico de entregas.

**Fora de escopo**

- Webhooks de entrada, da plataforma para o cliente apenas ([REUNIÃO 09:02 Sofia e Marcos]).
- Notificação por e-mail ao cliente em caso de falhas ([REUNIÃO 09:37 Larissa]).
- Painel visual para o cliente ([REUNIÃO 09:40 Larissa]).
- Arquivamento de linhas entregues da outbox ([REUNIÃO 09:08 Diego]).
- Limitação de taxa de envio ([REUNIÃO 09:39 Larissa], "observar e decidir depois").

## Fluxos detalhados

### Fluxo 1: criação do evento na outbox

Executado dentro de `changeStatus`, após a inserção em `order_status_history` e antes do `findUnique` final.

1. `changeStatus` valida a transição e atualiza o estoque, como hoje ([CÓDIGO src/modules/orders/order.service.ts, linhas 147 a 167]).
2. Chama `publishWebhookEvent(tx, order, from, to)`, passando o cliente da transação ([REUNIÃO 09:41 Bruno]).
3. A função busca os endpoints ativos do `customer_id` do pedido cujo filtro de status contém `to`. Se nenhum, retorna sem inserir ([REUNIÃO 09:34 Bruno]).
4. Para cada endpoint elegível, insere uma linha em `webhook_outbox` com:
   - `id`: UUID gerado na inserção ([REUNIÃO 09:25 Diego], [REUNIÃO 09:51 Larissa]). Este é o `X-Event-Id`.
   - `endpoint_id`, `payload` (JSON renderizado no momento da inserção, [REUNIÃO 09:52 Larissa]), `status = pendente`, `attempts = 0`, `next_attempt_at = now()`.
5. Antes de gravar, calcula o tamanho do payload. Acima de 64 KB, o evento não é gravado, o erro `WEBHOOK_PAYLOAD_TOO_LARGE` vai para o log e a métrica `webhook_payload_rejected_total` é incrementada. A mudança de status não é revertida ([DECISÃO], base em [REUNIÃO 09:23 Sofia] e [REUNIÃO 09:04 Bruno]).
6. Se qualquer inserção falhar, a exceção propaga e a transação inteira sofre rollback ([REUNIÃO 09:40 Bruno]).

**Payload (versão 1)**

```json
{
  "event_id": "5b1f0c7e-6d2a-4f0e-9a8b-2c3d4e5f6a7b",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-26T14:32:10.123Z",
  "order_id": "9d8c7b6a-5f4e-4d3c-8b2a-1f0e9d8c7b6a",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "3a2b1c0d-9e8f-4a7b-6c5d-4e3f2a1b0c9d",
  "total_cents": 15900
}
```

Campos conforme [REUNIÃO 09:43 Diego]. O payload não inclui itens: "não manda items pra não inflar" ([REUNIÃO 09:43 Diego]).

> **Decisão: nomes em `snake_case`.** A reunião usa `snake_case` no payload ([REUNIÃO 09:43 Diego]). O código usa `camelCase` nas entidades e respostas (`orderNumber`, `totalCents` em `prisma/schema.prisma`). Decidimos manter `snake_case` em todo o módulo (payload, corpo das requisições e respostas), porque é o que a reunião definiu e porque o contrato é externo e precisa ser estável. A divergência com o restante da API está registrada no ADR-006.

### Fluxo 2: processamento pelo worker

Processo separado, iniciado por `src/worker.ts` via `npm run worker` ([REUNIÃO 09:11 Larissa]).

1. Cria seu próprio `PrismaClient` com a fábrica existente `createPrismaClient()` ([CÓDIGO src/config/database.ts] e [REUNIÃO 09:30 Bruno]).
2. A cada **2 segundos** ([REUNIÃO 09:09 Diego]), busca até `WEBHOOK_BATCH_SIZE` linhas com `status = pendente` e `next_attempt_at <= now()`, ordenadas por `created_at` ([REUNIÃO 09:12 Diego]). O tamanho do lote não foi discutido ([PROPOSTA] padrão 50).
3. Reivindica as linhas do lote, marcando-as como `processando` em uma única atualização, para que outro processo não as pegue. ([PROPOSTA] necessário pela regra de instância única, [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md).)
4. Para cada linha, envia `POST` ao `url` do endpoint com os headers da seção "Contratos públicos" e timeout de 10 segundos ([REUNIÃO 09:42 Diego]).
5. Resultado:
   - **2xx:** marca como `entregue`, registra a tentativa em `webhook_deliveries` com status, tempo de resposta e corpo truncado.
   - **Demais respostas ou timeout:** trata como falha e segue para o Fluxo 3 ([REUNIÃO 09:42 Diego]). A definição de "sucesso" como 2xx é uma [PROPOSTA].

### Fluxo 3: retry

Após uma falha, incrementa `attempts` e calcula `next_attempt_at` a partir da progressão **1 min, 5 min, 30 min, 2 h, 12 h** ([REUNIÃO 09:17 Diego]).

| Falha número | Próxima tentativa |
|---|---|
| 1 | +1 min |
| 2 | +5 min |
| 3 | +30 min |
| 4 | +2 h |
| 5 | +12 h |
| 6 | DLQ |

Esta tabela segue a leitura de 5 retentativas (ver [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)). A confirmação está nas questões em aberto.

### Fluxo 4: DLQ e replay

1. Ao esgotar as tentativas, a linha é movida para `webhook_dead_letter` com `payload`, `reason` (último erro), `failed_at` e `outbox_id` ([REUNIÃO 09:18 Diego]). A linha da outbox passa a `falhou`.
2. Replay manual: `POST /admin/webhooks/dead-letter/:id/replay` ([REUNIÃO 09:18 Diego]). Exige `ADMIN` e registra o usuário que executou ([REUNIÃO 09:36 Sofia]).
3. O replay insere uma nova linha em `webhook_outbox` como `pendente`, com o mesmo `id` do evento original, para que o `X-Event-Id` não mude ([PROPOSTA], coerente com [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). A linha de DLQ é marcada como reprocessada, sem ser apagada ([PROPOSTA]).

## Contratos públicos

Base: `/webhooks` e `/admin/webhooks`, autenticados por `Authorization: Bearer <JWT>`, como as demais rotas ([CÓDIGO src/middlewares/auth.middleware.ts]).

### Endpoints de configuração

| Método | Rota | Descrição | Origem |
|---|---|---|---|
| POST | `/webhooks` | Cria endpoint e devolve a secret uma vez | [REUNIÃO 09:31 Marcos] |
| GET | `/webhooks?customer_id=` | Lista endpoints de um cliente | [REUNIÃO 09:33 Bruno] |
| PATCH | `/webhooks/:id` | Edita URL, filtro de status e estado ativo | [REUNIÃO 09:33 Bruno] |
| DELETE | `/webhooks/:id` | Remove o endpoint | [REUNIÃO 09:33 Bruno] |
| POST | `/webhooks/:id/rotate-secret` | Gera nova secret; a anterior segue válida por 24 h | [REUNIÃO 09:21 Sofia]; rota [PROPOSTA] |
| GET | `/webhooks/:id/deliveries` | Histórico das últimas tentativas | [REUNIÃO 09:34 Marcos] |
| POST | `/admin/webhooks/dead-letter/:id/replay` | Reprocessa uma entrada da DLQ | [REUNIÃO 09:18 Diego] |

**Decisão:** a reunião disse que `customer_id` vem "no body ou no path" e não do JWT ([REUNIÃO 09:32 Larissa]). Ficou no corpo do `POST` e como filtro de query na listagem. Motivo: o `POST` já recebe URL e filtro no corpo; na listagem, o cliente é um filtro, não um recurso do caminho.

### POST /webhooks

**Request**

```http
POST /webhooks HTTP/1.1
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customer_id": "3a2b1c0d-9e8f-4a7b-6c5d-4e3f2a1b0c9d",
  "url": "https://integracao.atlascomercial.com.br/hooks/pedidos",
  "event_statuses": ["SHIPPED", "DELIVERED"]
}
```

**Response 201**

```json
{
  "id": "e1d2c3b4-a5f6-4789-8abc-def012345678",
  "customer_id": "3a2b1c0d-9e8f-4a7b-6c5d-4e3f2a1b0c9d",
  "url": "https://integracao.atlascomercial.com.br/hooks/pedidos",
  "event_statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_7Hq2mWk9Lp4RzX1vN8cT6bYs3DfG0aJe",
  "created_at": "2026-10-26T14:40:00.000Z"
}
```

A secret é gerada pela plataforma e devolvida na criação ([REUNIÃO 09:31 Marcos]). O prefixo `whsec_` e o tamanho são [PROPOSTA].

**Status codes:** `201` criado; `400 WEBHOOK_INVALID_URL` se a URL não for `https` ([REUNIÃO 09:23 Sofia]); `400 VALIDATION_ERROR` para campos inválidos; `401` sem token; `404 NOT_FOUND` se o cliente não existir.

### GET /webhooks/:id/deliveries

**Request:** `GET /webhooks/e1d2c3b4-a5f6-4789-8abc-def012345678/deliveries?page=1&pageSize=20`

**Response 200**

```json
{
  "data": [
    {
      "id": "7c6b5a4d-3e2f-4a1b-9c8d-7e6f5a4b3c2d",
      "event_id": "5b1f0c7e-6d2a-4f0e-9a8b-2c3d4e5f6a7b",
      "attempt": 1,
      "status": "SUCCESS",
      "http_status": 200,
      "response_time_ms": 184,
      "response_body": "{\"received\":true}",
      "created_at": "2026-10-26T14:32:12.406Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

O formato paginado segue `src/shared/http/response.ts`, usado por `OrderService.list` ([CÓDIGO src/modules/orders/order.service.ts, linha 41]). São retidas as últimas 100 tentativas por endpoint ([REUNIÃO 09:34 Marcos]), e a paginação percorre essas entradas.

**Status codes:** `200`; `404 WEBHOOK_NOT_FOUND`.

### POST /webhooks/:id/rotate-secret

**Request:** sem corpo. `POST /webhooks/e1d2c3b4-a5f6-4789-8abc-def012345678/rotate-secret`

**Response 200**

```json
{
  "id": "e1d2c3b4-a5f6-4789-8abc-def012345678",
  "secret": "whsec_NovaSecretGeradaAqui000000000000",
  "previous_secret_valid_until": "2026-10-27T14:45:00.000Z"
}
```

**Status codes:** `200`; `404 WEBHOOK_NOT_FOUND`. Não exige `ADMIN`: segue as demais rotas de configuração, que aceitam qualquer papel autenticado ([REUNIÃO 09:37 Sofia]).

### POST /admin/webhooks/dead-letter/:id/replay

**Request:** sem corpo. `POST /admin/webhooks/dead-letter/a9b8c7d6-e5f4-4a3b-8c2d-1e0f9a8b7c6d/replay`, com `Authorization: Bearer <JWT de ADMIN>`.

**Response 202**

```json
{
  "dead_letter_id": "a9b8c7d6-e5f4-4a3b-8c2d-1e0f9a8b7c6d",
  "outbox_id": "5b1f0c7e-6d2a-4f0e-9a8b-2c3d4e5f6a7b",
  "status": "pendente"
}
```

**Status codes:** `202` aceito; `401` sem token; `403 FORBIDDEN` se o papel não for `ADMIN` ([CÓDIGO src/middlewares/auth.middleware.ts, `requireRole`]); `404 WEBHOOK_DLQ_NOT_FOUND`; `409 WEBHOOK_DLQ_ALREADY_REPLAYED` [PROPOSTA].

## Cabeçalhos da entrega

Enviados pelo worker a cada tentativa ([REUNIÃO 09:44 Diego] e [REUNIÃO 09:44 Sofia]):

| Header | Valor | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [REUNIÃO 09:44 Diego] |
| `X-Event-Id` | UUID do evento, igual em todas as tentativas | [REUNIÃO 09:25 Diego] |
| `X-Signature` | HMAC-SHA256 do corpo com a secret do endpoint | [REUNIÃO 09:20 Sofia] |
| `X-Timestamp` | Instante do envio em Unix, segundos inteiros | [REUNIÃO 09:44 Diego]; formato [DECISÃO] |
| `X-Webhook-Id` | UUID do endpoint configurado | [REUNIÃO 09:44 Sofia] |

**Formato de `X-Signature`:** [PROPOSTA] `sha256=<hex>`, calculado sobre o corpo exato enviado. O cliente deve comparar com tempo constante.

**Janela de tolerância do `X-Timestamp`:** 300 segundos. O cliente rejeita entregas cujo `X-Timestamp` esteja mais de 300 segundos distante do seu relógio. Cada tentativa tem seu próprio `X-Timestamp`, então retentativas tardias são aceitas. Valor [DECISÃO], não discutido na reunião.

## Autenticação das entregas

- Uma secret por endpoint ([REUNIÃO 09:21 Sofia]).
- Rotação com janela de 24 horas em que ambas as secrets validam ([REUNIÃO 09:21 Sofia]). Durante a janela, a plataforma assina com a nova secret e o cliente deve aceitar ambas.
- Apenas `https` ([REUNIÃO 09:23 Sofia]).
- Limite de 64 KB verificado na criação do evento. Acima disso, o evento não é gravado e o erro é registrado ([REUNIÃO 09:23 Sofia], [REUNIÃO 09:24 Larissa]). A mudança de status não é revertida.

## Matriz de erros

| Código | HTTP | Quando | Origem |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint inexistente | [REUNIÃO 09:28 Bruno], padrão de [CÓDIGO src/shared/errors/http-errors.ts] |
| `WEBHOOK_INVALID_URL` | 400 | URL não `https` ou malformada | [REUNIÃO 09:28 Bruno], [REUNIÃO 09:23 Sofia] |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Tentativa de assinar sem secret ativa | [REUNIÃO 09:28 Bruno] |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | interno (log e métrica) | Payload acima de 64 KB na criação do evento; o evento não é gravado | [REUNIÃO 09:23 Sofia]; [DECISÃO] |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | ID da DLQ inexistente | [PROPOSTA] |
| `WEBHOOK_DLQ_ALREADY_REPLAYED` | 409 | DLQ já reprocessada | [PROPOSTA] |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | Cliente não respondeu em 10 s; vai para retry | [REUNIÃO 09:42 Diego] |

Todos os erros usam as classes de `src/shared/errors/http-errors.ts` (`NotFoundError`, `ValidationError`, `UnprocessableEntityError`, `ConflictError`) ou uma classe `WebhookError` que estende `AppError` ([CÓDIGO src/shared/errors/app-error.ts]). O `errorMiddleware` formata a resposta sem alterações ([CÓDIGO src/middlewares/error.middleware.ts]).

## Estratégias de resiliência

- **Timeout:** 10 segundos por requisição de entrega ([REUNIÃO 09:42 Diego]).
- **Retry:** backoff 1, 5, 30, 120 e 720 minutos; máximo de 5 retentativas ([REUNIÃO 09:17 Diego]).
- **Fallback:** DLQ em `webhook_dead_letter`, com replay manual ([REUNIÃO 09:18 Diego]).
- **Queda do worker:** a outbox é durável. Ao reiniciar, o worker retoma as linhas em `pendente`. As linhas em `processando` são reavaliadas após um tempo limite [PROPOSTA].
- **Queda da API:** não afeta a entrega, porque o worker é processo separado ([REUNIÃO 09:11 Diego]).
- **Falha na inserção da outbox:** rollback da mudança de status ([REUNIÃO 09:41 Diego]).

## Observabilidade

**Logs.** Pino, via `logger` de `src/shared/logger/index.ts` ([REUNIÃO 09:29 Bruno]). O logger já redige `authorization`, `cookie`, `password`, `passwordHash`, `token` e `accessToken` ([CÓDIGO src/shared/logger/index.ts]). Para os webhooks, a redação deve incluir `secret`, `*.secret` e o header `x-signature` [PROPOSTA], por causa do risco de vazamento relatado em [REUNIÃO 09:22 Diego].

**Métricas.** A reunião não definiu métricas. Propostas [PROPOSTA]:
- `webhook_events_created_total` por tipo de evento.
- `webhook_deliveries_total` por resultado (`success`, `failure`, `timeout`).
- `webhook_delivery_latency_ms` histograma, para acompanhar o objetivo de 10 segundos.
- `webhook_outbox_pending` gauge, para detectar acúmulo.
- `webhook_dlq_total` por motivo.

**Tracing.** Não discutido na reunião e sem dependência de tracing no projeto. [PROPOSTA] correlacionar logs por `event_id` e pelo identificador de requisição de `src/middlewares/request-logger.middleware.ts`.

## Dependências e compatibilidade

- **Prisma 5.22.0** e **MySQL**, já usados ([CÓDIGO package.json]).
- **Pino 9.5.0**, já usado ([CÓDIGO package.json]).
- **Zod 3.23.8**, para schemas ([CÓDIGO package.json], `src/modules/orders/order.schemas.ts`).
- **HTTP client:** o Node 20 fornece `fetch` nativo. A escolha não foi discutida; [PROPOSTA] usar `fetch` com `AbortSignal.timeout(10000)`.
- **Compatibilidade:** nenhum contrato existente muda. O `changeStatus` mantém sua assinatura e retorno.
- **Schema:** novas tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`, em nova migração ([PROPOSTA]).

## Integração com o sistema existente

Os caminhos abaixo foram verificados no repositório.

| Arquivo | Como o módulo se integra |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` (linhas 126 a 179) passa a chamar `publishWebhookEvent(tx, order, from, to)` dentro do `this.prisma.$transaction`, após o `orderStatusHistory.create` (linha 159). Assinatura pública e retorno permanecem os mesmos. A função recebe o `tx` e não o repositório ([REUNIÃO 09:41 Bruno]). |
| `src/modules/orders/order.status.ts` | Fonte de `canTransition`, `shouldDebitStock` e `shouldReplenishStock`. O evento usa `from` e `to` já validados. Nenhuma alteração. |
| `src/shared/errors/http-errors.ts` | Reuso de `NotFoundError`, `ValidationError`, `ConflictError` e `UnprocessableEntityError`. Os erros do módulo usam o mesmo formato, com código `WEBHOOK_*` ([REUNIÃO 09:28 Bruno]). |
| `src/shared/errors/app-error.ts` | Base de `WebhookError`, com `statusCode`, `errorCode` e `details`. |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` no replay ([REUNIÃO 09:36 Larissa]). |
| `src/middlewares/error.middleware.ts` | Trata `AppError` sem alterações. |
| `src/shared/logger/index.ts` | Logger Pino compartilhado. Proposta de ampliar `redactPaths`. |
| `src/config/database.ts` | `createPrismaClient()` reutilizado pelo `src/worker.ts`, para instância própria ([REUNIÃO 09:30 Bruno]). |
| `src/server.ts` | Padrão de bootstrap e encerramento (`SIGINT`, `SIGTERM`) a ser espelhado em `src/worker.ts`. |
| `src/routes/index.ts` | Registro de `router.use('/webhooks', ...)` e do router admin. |
| `src/modules/orders/order.routes.ts` | Modelo para `webhook.routes.ts`: `router.use(authenticate)` e `validate({ body })`. |
| `src/modules/orders/order.schemas.ts` | Modelo para `webhook.schemas.ts` com Zod e `z.nativeEnum(OrderStatus)` para `event_statuses`. |
| `prisma/schema.prisma` | Novos modelos com `@default(uuid()) @db.Char(36)`, seguindo o padrão de `Order`. O enum `OrderStatus` é reutilizado. |
| `package.json` | Novo script `"worker": "tsx --env-file=.env src/worker.ts"` [REUNIÃO 09:11 Larissa]. |

**Módulo novo** em `src/modules/webhooks/`: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.errors.ts`, `webhook.worker.ts` ([REUNIÃO 09:28 Bruno]).

## Critérios de aceite técnicos

- Uma mudança de status com endpoint assinante cria exatamente uma linha em `webhook_outbox` por endpoint elegível, na mesma transação.
- Uma falha na inserção da outbox faz rollback da mudança de status.
- Uma mudança de status sem endpoint elegível não cria linhas.
- O worker entrega em até 2 segundos após a inserção, em condição normal.
- Uma resposta não-2xx ou timeout de 10 segundos gera nova tentativa com o intervalo correto.
- Após 5 retentativas, a entrada vai para `webhook_dead_letter`.
- O replay exige `ADMIN` e registra o usuário.
- `X-Event-Id` é o mesmo em todas as tentativas do mesmo evento.
- `X-Signature` é verificável com a secret do endpoint.
- Durante a rotação, ambas as secrets validam até o fim da janela de 24 horas.
- URL `http` é recusada na criação e na edição.
- Payload acima de 64 KB não é enviado.
- Nenhuma secret aparece em logs.

## Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Vazamento de secret pelo cliente ([REUNIÃO 09:22 Diego]) | Secret por endpoint, rotação com janela, redação de logs. |
| Dois workers processando a mesma linha | Reivindicação por atualização de `status` antes do envio ([PROPOSTA]). |
| DLQ acumulando por indisponibilidade longa | Replay manual; monitorar `webhook_dlq_total`. |
| Clientes sem deduplicação | Documentação em destaque no portal ([REUNIÃO 09:26 Marcos]). |
| Crescimento da outbox | Índices; arquivamento em fase futura ([REUNIÃO 09:08 Diego]). |
