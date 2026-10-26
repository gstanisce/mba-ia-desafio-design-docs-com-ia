# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)
- **Origem:** [09:27] Bruno, [09:28] Bruno, [09:28] Diego, [09:28] Bruno, [09:29] Larissa, [09:29] Bruno, [09:29] Diego, [09:30] Bruno, [09:30] Larissa, [09:35] Larissa, [09:36] Larissa

## Contexto

A codebase tem um padrão claro: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas. Há uma hierarquia de erros com `AppError`, middleware de erro centralizado e logger Pino compartilhado. O time quer que o módulo de webhooks não introduza convenções novas.

## Decisão

- **Módulo próprio** em `src/modules/webhooks`, com a mesma estrutura dos demais módulos ([09:27] Bruno). O processamento do worker fica em um arquivo dentro do módulo, como `webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
- **Erros:** classes que estendem `AppError` (`src/shared/errors/app-error.ts`), com códigos no prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa). Os erros de validação seguem o padrão de `ValidationError` e os de negócio o de `ConflictError` e `UnprocessableEntityError` (`src/shared/errors/http-errors.ts`).
- **Middleware de erro:** o `errorMiddleware` existente já trata `AppError`, `ZodError` e erros do Prisma. Nenhuma alteração é necessária ([09:29] Bruno).
- **Logger:** Pino, o mesmo de `src/shared/logger/index.ts`. Nenhuma dependência nova ([09:29] Bruno).
- **Schemas de entrada:** Zod, no padrão de `src/modules/orders/order.schemas.ts`.
- **Autorização:** `authenticate` para todas as rotas. O replay de DLQ usa `requireRole('ADMIN')`, já existente em `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
- **Identificadores:** UUID em todas as chaves, como no restante do projeto ([09:51] Larissa).

Resumo da decisão, registrado por Larissa: "reuso máximo do que já existe" ([09:30] Larissa).

## Alternativas consideradas

- **Módulo com convenções próprias** (estrutura, erros ou logger diferentes). Não discutida em detalhe na reunião. Descartada pelo princípio de reuso máximo ([09:30] Larissa).
- **Biblioteca externa de webhooks.** Não discutida na reunião. Não foi considerada, porque o time decidiu manter o controle do padrão de entrega.

## Consequências

**Positivas**

- Curva de aprendizado baixa para quem já conhece os outros módulos.
- Tratamento de erro e logs uniformes na API.
- Menos código novo para revisar.

**Negativas e trade-offs**

- O módulo fica acoplado às convenções atuais. Mudanças no `errorMiddleware` ou no logger afetam webhooks também.
- O logger atual redige apenas `authorization`, `cookie`, `password`, `passwordHash`, `token` e `accessToken` (`src/shared/logger/index.ts`). A secret e o header `X-Signature` não estão nessa lista, então o FDD propõe ampliá-la.
- Como o worker roda em outro processo, ele precisa de sua própria instância de Prisma e de logger ([ADR-002](ADR-002-worker-polling-processo-separado.md)).
- **Divergência de nomenclatura:** o módulo de webhooks usa `snake_case` no payload, no corpo e nas respostas, enquanto o restante da API usa `camelCase`. A decisão foi manter `snake_case` no contrato externo, porque é o que a reunião definiu e porque o contrato precisa ser estável para o cliente. A inconsistência é aceita.

## Referências

- `src/modules/orders/order.routes.ts`, `order.schemas.ts` e `order.service.ts`: modelos de módulo.
- [FDD](../FDD.md), seção "Integração com o sistema existente".
