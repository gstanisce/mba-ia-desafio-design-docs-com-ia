# ADR-002: Worker em processo separado, com polling de 2 segundos

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)
- **Origem:** [09:08] Diego, [09:09] Diego, [09:10] Larissa, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:12] Diego, [09:13] Larissa, [09:14] Marcos

## Contexto

Com a outbox em MySQL ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e disparar as chamadas HTTP. Três questões foram discutidas:

1. Como o worker descobre que há eventos novos.
2. Onde o worker roda.
3. Como garantir a ordem de entrega dos eventos de um mesmo pedido.

O MySQL não tem `LISTEN/NOTIFY` como o Postgres. Trigger de banco executa SQL, mas não avisa um processo externo ([09:09] Diego).

## Decisão

- **Polling:** o worker consulta a outbox a cada **2 segundos**, busca os eventos pendentes mais antigos em lote pequeno, processa e marca o resultado ([09:09] Diego, [09:10] Larissa). Com isso, a latência no pior caso é de 2 segundos somada ao tempo de envio, dentro do requisito de "abaixo de 10 segundos" ([09:02] Marcos).
- **Processo separado:** o worker roda como processo próprio, com entry point `src/worker.ts` e script `npm run worker`, e não dentro da instância da API ([09:11] Diego, [09:11] Larissa). Assim, reiniciar a API não derruba o worker.
- **Conexão própria:** o worker cria seu próprio `PrismaClient`, apontando para o mesmo banco e a mesma `DATABASE_URL`, porque `PrismaClient` é por processo ([09:30] Bruno). A fábrica `createPrismaClient()` de `src/config/database.ts` é reutilizada.
- **Instância única:** há um único worker. A ordem de processamento segue `created_at` da outbox, o que dá ordem implícita por `order_id` enquanto houver um único worker ([09:12] Diego).

## Alternativas consideradas

- **Trigger do banco para acordar o worker.** Descartado: MySQL não notifica processo externo. Improvisar com escrita em arquivo ou chamada a endpoint foi considerado "esquisito" ([09:09] Diego).
- **Worker dentro do processo da API.** Descartado: uma reinicialização da API interromperia o worker ([09:11] Diego).
- **Múltiplos workers em paralelo.** Adiado. Quebra a ordem implícita; o caminho futuro é particionar por `order_id` ou usar lock pessimista ([09:13] Diego). Não há garantia de ordem global, e os clientes não a pediram ([09:14] Marcos).

## Consequências

**Positivas**

- Sem dependência de infraestrutura nova.
- Ordem por `order_id` garantida enquanto houver um único worker.
- Falha da API não afeta a entrega.

**Negativas e trade-offs**

- Latência de até 2 segundos antes da primeira tentativa, aceita pelo time ([09:10] Larissa: "Aceitamos").
- Consultas periódicas à tabela, mesmo sem carga. O índice em `status` e `created_at` mitiga o custo ([09:08] Diego).
- **Limitação conhecida:** ordem garantida apenas por `order_id` e apenas com um worker. Escalar horizontalmente exige nova decisão ([09:13] Larissa: "Documentamos como limitação conhecida").
- Dois processos Node acessando o mesmo banco exigem que a leitura da outbox não gere processamento duplicado. A reivindicação de lote precisa ser pensada no FDD.

## Referências

- [ADR-001](ADR-001-outbox-no-mysql.md)
- [ADR-003](ADR-003-retry-backoff-e-dlq.md)
- [FDD](../FDD.md), seção "Fluxo 2: processamento pelo worker".
