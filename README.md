# Da reunião ao documento: design docs do Sistema de Webhooks de Notificação de Pedidos

## Sobre o desafio

A tarefa foi transformar a transcrição de uma reunião técnica de 55 minutos em um pacote de design docs, pronto para o time de engenharia começar a implementar a feature de webhooks de notificação de pedidos. A reunião reuniu tech lead, PM, dois engenheiros e a segurança, e tomou decisões sobre outbox, worker, retry, DLQ, assinatura HMAC e entrega at-least-once. Nada foi registrado além da transcrição.

O código base é um Order Management System em Node.js, TypeScript, Prisma e MySQL, sem nenhum mecanismo de eventos. Toda informação dos documentos precisa ser rastreável à transcrição (`TRANSCRICAO.md`, com timestamp e falante) ou a um arquivo real do código. A entrega é apenas documental: `src/`, `prisma/` e `tests/` permanecem inalterados.

## Ferramentas de IA utilizadas

- **Claude Code** (CLI da Anthropic), usado como ferramenta principal. Leu o repositório, analisou a transcrição, propôs a estrutura do pacote e gerou os documentos em pt-BR.
- Nenhuma outra ferramenta foi usada até o momento.

## Workflow adotado

1. **Entendimento e plano.** Antes de escrever, foi feita uma leitura do enunciado e um plano de execução, revisado e aprovado antes de qualquer alteração.
2. **Preparação.** Clone do repositório base e leitura dos arquivos de estrutura: `prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/`, `src/middlewares/`, `src/shared/logger/index.ts`, `src/routes/index.ts` e `src/server.ts`.
3. **Filtragem da transcrição.** Em vez de pedir um resumo, a transcrição foi percorrida com perguntas dirigidas: o que foi decidido, o que foi descartado, o que foi adiado, o que ficou em aberto. Cada item recebeu timestamp e falante.
4. **Ordem de escrita.** ADRs primeiro, depois RFC, FDD, PRD, tracker e este README, como sugerido no enunciado. As decisões formam a base, e os demais documentos apenas referenciam o que já foi decidido.
5. **Convenção de rastreabilidade.** No FDD, cada item é marcado como `[REUNIÃO hh:mm Nome]`, `[CÓDIGO caminho]` ou `[PROPOSTA]`. Itens `[PROPOSTA]` são detalhes de implementação que a reunião não discutiu e precisam de aprovação.
6. **Revisão contra o tracker.** O tracker foi conferido com `grep`: contagem de linhas por fonte e verificação de que os timestamps existem na transcrição.

## Prompts customizados

Os prompts abaixo são os modelos de instrução usados para dirigir a IA neste trabalho. Foram escritos com escopo, formato e restrições, no lugar de pedidos genéricos ("gere um PRD"). Os textos são versões condensadas; o texto literal de cada ciclo deve ser substituído pelo efetivamente enviado.

**1. Filtragem da transcrição**

```text
Leia TRANSCRICAO.md. Não resuma. Liste cada item que se enquadre em uma destas categorias,
com timestamp e falante:
- DECISÃO FECHADA (a equipe confirmou explicitamente)
- REQUISITO FUNCIONAL (algo que o sistema deve fazer)
- RESTRIÇÃO (limite técnico, de segurança ou de prazo)
- DESCARTADO (explicitamente rejeitado)
- ADIADO (explicitamente deixado para depois)
- EM ABERTO (levantado, mas sem decisão)
Para cada item, cite a frase original entre aspas. Se um item não couber em nenhuma categoria, liste-o à parte.
```

**2. Geração de ADR com rastreabilidade**

```text
Escreva docs/adrs/ADR-00N-<titulo>.md em português do Brasil, no formato Status / Contexto /
Decisão / Alternativas Consideradas / Consequências.
Regras:
- Cada afirmação deve citar o timestamp e o falante da transcrição entre colchetes, ex.: [09:17] Diego.
- Se a reunião não discutiu algo, escreva [PROPOSTA] e não apresente como decisão.
- Se citar um arquivo do código, confirme que ele existe com Read antes de escrever.
- Alternativas devem ser as discutidas na reunião; "plausível, não discutida" só quando não houver nenhuma.
```

**3. Verificação de integração com o código**

```text
Para cada arquivo que o FDD citar em "Integração com o sistema existente", leia o arquivo e
confirme: (a) o caminho existe; (b) o trecho citado (ex.: linhas de changeStatus) corresponde
ao que o texto diz. Liste as divergências em vez de corrigi-las em silêncio.
```

## Iterações e ajustes

Os principais pontos em que o resultado precisou de correção ou verificação:

1. **Caminho do clone.** O clone foi criado em uma subpasta (`mba-ia-desafio-design-docs-com-ia`), e não na pasta esperada. Os caminhos de trabalho foram ajustados antes de começar a escrever.
2. **Ambiguidade no número de tentativas.** A reunião diz "5 tentativas", mas lista cinco intervalos. Cinco intervalos após a falha inicial resultam em seis envios. Decidi por um envio inicial mais cinco retentativas, com a justificativa registrada no ADR-003.
3. **Divergência de nomenclatura.** A transcrição define o payload com `snake_case` (`order_number`, `total_cents`), enquanto as entidades do código usam `camelCase` (`orderNumber`, `totalCents` em `prisma/schema.prisma`). Decidi manter `snake_case` em todo o módulo, e a divergência com o restante da API está registrada no ADR-006.
4. **Posição de `customer_id`.** A reunião primeiro perguntou se o `customer_id` viria do JWT, depois concluiu que viria "no body ou no path". Decidi `customer_id` no corpo do `POST` e como filtro na listagem, com a justificativa no FDD.
5. **Itens sem origem.** Probabilidade e impacto dos riscos e a meta de redução de consultas a `GET /orders` não foram quantificados na reunião. Foram marcados como `[ESTIMATIVA]` ou "a definir", e não como fatos.
6. **Momento do limite de 64 KB.** A reunião definiu o limite e o erro, mas não em qual etapa a verificação ocorre. Decidi verificar na criação do evento, sem reverter a mudança de status, porque reverter por tamanho violaria a regra de não bloquear a mudança de status.

> Esta seção deve ser complementada com as iterações reais de cada ciclo de geração e revisão.

## Como navegar a entrega

| Ordem | Arquivo | Para quê |
|---|---|---|
| 1 | [docs/RFC.md](docs/RFC.md) | Proposta técnica, alternativas descartadas e questões em aberto |
| 2 | [docs/adrs/](docs/adrs/) | Decisões fechadas, uma por arquivo |
| 3 | [docs/FDD.md](docs/FDD.md) | Como implementar: fluxos, contratos, erros e integração com o código |
| 4 | [docs/PRD.md](docs/PRD.md) | Problema, público, escopo, requisitos e métricas |
| 5 | [docs/TRACKER.md](docs/TRACKER.md) | Origem de cada item, na transcrição ou no código |
| - | [TRANSCRICAO.md](TRANSCRICAO.md) | Fonte primária, não alterada |

Os ADRs são:

- [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md): padrão outbox no MySQL
- [ADR-002](docs/adrs/ADR-002-worker-polling-processo-separado.md): worker em processo separado, com polling
- [ADR-003](docs/adrs/ADR-003-retry-backoff-e-dlq.md): retry com backoff e DLQ
- [ADR-004](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md): HMAC-SHA256 com secret por endpoint
- [ADR-005](docs/adrs/ADR-005-at-least-once-com-x-event-id.md): entrega at-least-once com `X-Event-Id`
- [ADR-006](docs/adrs/ADR-006-reuso-padroes-existentes.md): reuso dos padrões existentes
