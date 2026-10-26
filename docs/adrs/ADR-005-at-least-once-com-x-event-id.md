# ADR-005: Entrega at-least-once com X-Event-Id para deduplicação no cliente

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança), Marcos (Product Manager)
- **Origem:** [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa, [09:44] Diego, [09:51] Larissa

## Contexto

Com retry e timeout de 10 segundos ([ADR-003](ADR-003-retry-backoff-e-dlq.md)), é possível que o cliente processe um evento e a plataforma não receba a confirmação a tempo, o que resulta em novo envio. Garantir exactly-once exigiria coordenação dos dois lados ([09:25] Diego).

## Decisão

- **Garantia at-least-once:** o mesmo evento pode chegar mais de uma vez ([09:24] Diego).
- **Identificador único por evento:** um UUID gerado quando o evento entra na outbox, enviado no header `X-Event-Id`. O mesmo valor é reenviado em todas as tentativas do mesmo evento ([09:25] Diego).
- **Deduplicação do lado do cliente**, usando o `X-Event-Id` ([09:26] Larissa).
- **Documentação destacada** no portal de desenvolvedores, a cargo de Marcos ([09:26] Marcos).
- **Identificador com UUID**, seguindo o padrão de chaves do projeto ([09:51] Larissa).

## Alternativas consideradas

- **Exactly-once.** Descartado: exige coordenação entre plataforma e cliente, o que aumenta muito a complexidade. Stripe e GitHub operam com at-least-once e identificador de evento ([09:25] Diego).

## Consequências

**Positivas**

- Simplicidade no lado da plataforma e padrão conhecido pelos clientes.
- Com `X-Event-Id`, o cliente consegue deduplicar sem depender de ordem de chegada.

**Negativas e trade-offs**

- A responsabilidade de deduplicar passa para o cliente. Sofia registrou que isso "joga responsabilidade pro cliente" ([09:25] Sofia), e a equipe aceitou.
- Clientes que não deduplicarem poderão processar o mesmo pedido duas vezes. A mitigação é a documentação ([09:26] Marcos).
- A proteção contra replay usa o header `X-Timestamp` (proposto por Diego em [09:44]). A reunião não definiu a janela de tolerância; a decisão é de 300 segundos, registrada no FDD e sujeita à revisão do time.

## Referências

- [ADR-001](ADR-001-outbox-no-mysql.md): UUID criado na inserção da outbox.
- [FDD](../FDD.md), seção "Contratos públicos".
