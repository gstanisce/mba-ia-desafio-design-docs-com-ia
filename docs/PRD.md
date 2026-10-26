# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Guilherme Correa |
| **Product Manager** | Marcos |
| **Status** | Em revisão |
| **Data** | 2026-10-26 |
| **Relacionados** | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/), [TRACKER](TRACKER.md) |

> **Convenção.** Os IDs (ex.: `PRD-FR-01`) são usados no [TRACKER](TRACKER.md). Itens marcados como **[ESTIMATIVA]** não foram discutidos na reunião e são julgamento do autor.

## Resumo e contexto

Três clientes B2B consultam `GET /orders` periodicamente para saber se o status de seus pedidos mudou. Isso é lento e caro para eles. Esta feature substitui essa consulta por notificações enviadas pela plataforma sempre que o status de um pedido muda, com entrega em até 10 segundos.

## Problema e motivação

- Os clientes Atlas Comercial, MaxDistribuição e Nova Cargo precisam saber em tempo quase real quando o status de seus pedidos muda ([09:00] Marcos).
- Hoje, eles batem em `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara ([09:00] Marcos).
- A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre ([09:00] Marcos).
- Para os clientes, qualquer atraso abaixo de 10 segundos é "tempo real". O essencial é não ficar pendurado e não precisar atualizar manualmente ([09:02] Marcos).

## Público-alvo e cenários de uso

| Público | Cenário |
|---|---|
| Clientes B2B (integradores) | Cadastram um endpoint, escolhem os status de interesse e recebem notificações assinadas. |
| Equipe de integração do cliente | Valida a assinatura, deduplica pelo `X-Event-Id` e consulta o histórico de entregas. |
| Operadores `ADMIN` da plataforma | Reprocessam entregas que foram para a DLQ, com registro de auditoria. |
| Time de Pedidos e Plataforma | Opera o worker, acompanha falhas e a fila de entregas. |

**Cenários principais**

1. Um pedido passa de `PAID` para `PROCESSING`. O cliente que escolheu `PROCESSING` recebe um evento assinado.
2. O endpoint do cliente está fora do ar. A plataforma tenta novamente em intervalos crescentes e, se não houver sucesso, o evento fica na DLQ aguardando replay.
3. O cliente rotaciona a secret pela API e tem 24 horas para migrar seus sistemas ([09:21] Sofia).

## Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Entregar atualizações de status em tempo quase real | Latência entre o commit da mudança de status e a primeira tentativa de envio | **p99 ≤ 10 s** | [09:02] Marcos |
| PRD-OBJ-02 | Não perder eventos de mudanças de status confirmadas | Eventos registrados por mudança de status confirmada que tenha assinante | **100%** | [09:06] Diego |
| PRD-OBJ-03 | Entregar a feature antes do fim do trimestre | Data de entrega em produção | **fim de novembro** | [09:45] Marcos |
| PRD-OBJ-04 | Reduzir consultas repetidas a `GET /orders` pelos clientes | Número de consultas por cliente | **a definir** com os clientes | [09:00] Marcos; meta não quantificada na reunião |

## Escopo

### Incluso

- Captura de eventos de mudança de status na mesma transação ([09:04] Bruno, [09:41] Diego).
- Cadastro, edição, remoção e listagem de endpoints ([09:31] Marcos, [09:33] Bruno).
- Filtro por status escolhido pelo cliente ([09:33] Marcos).
- Entrega assinada, com retry, DLQ e replay ([09:17] Diego, [09:18] Diego).
- Histórico de entregas ([09:34] Marcos).
- Rotação de secret ([09:21] Sofia).

### Fora de escopo

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-FORA-01 | Notificação por e-mail quando o webhook do cliente falha repetidamente | **Adiado** para próxima fase, após medir impacto | [09:37] Marcos, [09:37] Larissa |
| PRD-FORA-02 | Painel visual para o cliente acompanhar webhooks | **Descartado** nesta fase; projeto separado do time de frontend | [09:39] Marcos, [09:40] Larissa |
| PRD-FORA-03 | Webhooks enviados pelo cliente à plataforma | **Descartado**: é apenas saída | [09:02] Marcos, [09:03] Sofia |
| PRD-FORA-04 | Garantia de ordem global entre pedidos | **Descartado**: garantia apenas por `order_id`, com um worker | [09:13] Larissa, [09:14] Marcos |
| PRD-FORA-05 | Arquivamento de linhas entregues após 30 dias | **Adiado**, fora desta feature | [09:08] Diego |
| PRD-FORA-06 | Limitação de taxa de envio por cliente | **Adiado**: observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| PRD-FORA-07 | Execução de múltiplos workers em paralelo | **Adiado**: exige particionamento ou lock | [09:13] Diego |

## Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | Registrar um evento na outbox na mesma transação de cada mudança de status | [09:04] Bruno, [09:40] Bruno |
| PRD-FR-02 | Criar um endpoint com URL, e receber de volta uma secret gerada pela plataforma | [09:31] Marcos |
| PRD-FR-03 | Filtrar os eventos por lista de status escolhida pelo endpoint, na inserção | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-04 | Editar (PATCH), remover (DELETE) e listar (GET) endpoints de um cliente | [09:33] Bruno |
| PRD-FR-05 | Rotacionar a secret de um endpoint pela API, com a anterior válida por 24 horas | [09:21] Sofia |
| PRD-FR-06 | Entregar payload com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, sem itens | [09:43] Diego, [09:44] Bruno |
| PRD-FR-07 | Assinar cada entrega com HMAC-SHA256 sobre o corpo, usando a secret do endpoint | [09:20] Sofia, [09:22] Sofia |
| PRD-FR-08 | Enviar os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type` | [09:44] Diego, [09:44] Sofia |
| PRD-FR-09 | Tentar novamente até 5 vezes, com intervalos de 1, 5, 30, 120 e 720 minutos | [09:17] Diego, [09:17] Larissa |
| PRD-FR-10 | Mover eventos com falha definitiva para uma DLQ persistida, com payload, motivo e timestamp | [09:18] Diego |
| PRD-FR-11 | Reprocessar um item da DLQ por endpoint `ADMIN`, registrando quem executou | [09:18] Diego, [09:35] Larissa, [09:36] Sofia |
| PRD-FR-12 | Consultar as últimas tentativas de entrega de um endpoint, com status, payload, resposta e tempo de resposta | [09:34] Marcos |
| PRD-FR-13 | Considerar falha uma resposta sem sucesso ou um timeout de 10 segundos | [09:42] Diego |
| PRD-FR-14 | Aceitar apenas URLs `https`, recusando `http` com erro de validação | [09:23] Sofia |
| PRD-FR-15 | Recusar payloads acima de 64 KB, com erro | [09:23] Sofia, [09:24] Larissa |

## Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | Latência de entrega abaixo de 10 segundos (ver PRD-OBJ-01) | [09:02] Marcos |
| PRD-NFR-02 | Entrega at-least-once, com identificador único por evento para deduplicação | [09:24] Diego, [09:26] Larissa |
| PRD-NFR-03 | Mudança de status não pode ser bloqueada nem revertida por falha de cliente externo | [09:04] Bruno, [09:06] Diego |
| PRD-NFR-04 | Worker em processo separado da API | [09:11] Diego |
| PRD-NFR-05 | Uma secret por endpoint, nunca global | [09:21] Sofia |
| PRD-NFR-06 | Ordem garantida por `order_id` enquanto houver um único worker | [09:12] Diego |
| PRD-NFR-07 | Reutilizar padrões do projeto: erros `AppError`, Pino, middleware de erro, estrutura de módulo, códigos com prefixo `WEBHOOK_` | [09:30] Larissa |
| PRD-NFR-08 | Revisão de segurança de HMAC e geração de secret antes do deploy, com pelo menos dois dias úteis reservados | [09:46] Sofia |

## Decisões e trade-offs principais

| Decisão | Trade-off aceito | Referência |
|---|---|---|
| Outbox no MySQL, sem fila nova | Tabela cresce e precisa de arquivamento futuro | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Polling de 2 segundos, worker separado | Até 2 s de latência extra; um único worker | [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md) |
| Retry com DLQ e replay manual | Clientes fora do ar por mais de ~15 h dependem de ação manual | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| HMAC com secret por endpoint | Cliente precisa guardar a secret com segurança | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |

## Dependências

| Dependência | Responsável | Situação |
|---|---|---|
| Revisão de segurança de HMAC e geração de secret | Sofia | Dois dias úteis reservados antes do deploy ([09:46] Sofia) |
| Confirmação de prazo com os clientes | Marcos | Previsto para o mesmo dia da reunião ([09:47] Marcos) |
| Documentação no portal de desenvolvedores, com destaque para at-least-once | Marcos | Previsto ([09:26] Marcos) |
| Banco MySQL e Prisma existentes | Plataforma | Disponível ([CÓDIGO prisma/schema.prisma]) |
| Painel visual do cliente | Time de frontend | Projeto separado, fora desta feature ([09:40] Larissa) |
| Vínculo entre usuário e cliente no modelo de dados | Plataforma | **Condição de liberação** da API a clientes externos. Hoje `User` não referencia `Customer` (`prisma/schema.prisma`). Ver RFC, decisão 7 |

## Riscos e mitigação

> Probabilidade e impacto são **[ESTIMATIVA]**. A reunião não os quantificou.

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISCO-01 | Vazamento de secret por um cliente, como já ocorreu antes ([09:22] Diego) | Média [ESTIMATIVA] | Alto [ESTIMATIVA] | Secret por endpoint; rotação com janela de 24 h; redação de logs (FDD, seção Observabilidade). |
| PRD-RISCO-02 | Cliente não deduplica e processa o mesmo evento duas vezes | Média [ESTIMATIVA] | Alto [ESTIMATIVA] | `X-Event-Id` em todas as tentativas; documentação em destaque no portal. |
| PRD-RISCO-03 | Cliente fica fora do ar por mais de ~15 horas e o evento vai para a DLQ | Média [ESTIMATIVA] | Médio [ESTIMATIVA] | Replay manual por `ADMIN`; monitorar `webhook_dlq_total`. |
| PRD-RISCO-04 | Prazo de fim de novembro não cumprido, com a revisão de segurança incluída | Média [ESTIMATIVA] | Alto [ESTIMATIVA] | Revisão de segurança com dias reservados ([09:46] Sofia); escopo fechado na reunião. |
| PRD-RISCO-05 | Volume de chamadas sem limite de taxa sobrecarrega um cliente | Baixa [ESTIMATIVA] | Médio [ESTIMATIVA] | Observar métricas; implementar limite se virar problema ([09:39] Diego). |
| PRD-RISCO-06 | Crescimento da outbox sem arquivamento | Média [ESTIMATIVA] | Médio [ESTIMATIVA] | Índices em `status` e `created_at` ([09:08] Diego); arquivamento em fase futura. |

## Critérios de aceitação

Os critérios técnicos detalhados estão na seção "Critérios de aceite técnicos" do [FDD](FDD.md). Em nível de produto:

- PRD-CA-01: Cliente cadastra um endpoint, recebe a secret e recebe o primeiro evento assinado.
- PRD-CA-02: Cliente recebe apenas os status escolhidos no cadastro.
- PRD-CA-03: Cliente consegue rotacionar a secret e validar as duas durante 24 horas.
- PRD-CA-04: Um endpoint que falha é retentado conforme a progressão, e o histórico mostra cada tentativa.
- PRD-CA-05: Administrador reprocessa um item da DLQ, e a ação aparece com o usuário responsável.
- PRD-CA-06: Documentação no portal explica at-least-once e `X-Event-Id`.

## Estratégia de testes e validação

O projeto usa Vitest (`npm test`, [CÓDIGO package.json]) e já tem `tests/orders.test.ts` e `tests/helpers/factories.ts`. [PROPOSTA]:

- **Unitários:** cálculo de HMAC, progressão de backoff, filtro de status, validação de URL `https` e limite de 64 KB.
- **Integração:** `changeStatus` com endpoint assinante cria linha na outbox; falha na inserção faz rollback da mudança de status.
- **Worker:** entrega com servidor HTTP de teste, cobrindo sucesso, timeout de 10 s, falha, retry e DLQ.
- **Segurança:** revisão manual de HMAC e geração de secret pela Sofia, antes do deploy ([09:46] Sofia).
- **Validação com os clientes:** piloto com um cliente antes da liberação para os três. [PROPOSTA], não discutido na reunião.
