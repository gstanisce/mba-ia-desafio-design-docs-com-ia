# ADR-004: Autenticação de payload com HMAC-SHA256 e secret por endpoint

- **Status:** Aceita
- **Data:** 2026-10-26
- **Decisores:** Sofia (Engenheira de Segurança), Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)
- **Origem:** [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Diego, [09:22] Sofia, [09:23] Sofia, [09:44] Diego

## Contexto

Os eventos contêm dados de pedidos e são enviados para endpoints fora da infraestrutura do time. O cliente precisa validar que a requisição veio da plataforma e que o payload não foi alterado no caminho ([09:19] Sofia). Já houve caso de um cliente vazar a secret em log de aplicação ([09:22] Diego), o que exige que um vazamento fique restrito a um único endpoint.

## Decisão

- **HMAC-SHA256** calculado sobre o corpo da requisição, enviado no header `X-Signature` ([09:20] Sofia, [09:22] Sofia).
- **Secret por endpoint**, e não uma secret global da plataforma. Armazenada junto com URL, `customer_id` e estado ativo do endpoint ([09:21] Bruno, [09:21] Sofia).
- **Rotação pela API.** O cliente pode pedir uma nova secret. A secret anterior continua válida por **24 horas** em paralelo, para que o cliente migre seus sistemas. Depois disso, ela deixa de valer ([09:21] Sofia).
- **TLS obrigatório:** a URL precisa ser `https`. Uma URL `http` é recusada com erro de validação no schema Zod ([09:23] Sofia). Sofia registrou que isso é validação e não decisão arquitetural; é mantido aqui por estar ligado à assinatura.
- **Revisão de segurança** do código de HMAC e da geração de secret antes do deploy, com pelo menos dois dias úteis reservados ([09:46] Sofia).

## Alternativas consideradas

- **Secret global da plataforma.** Descartado: um vazamento comprometeria todos os clientes ([09:21] Sofia).
- **Assinatura assimétrica (chave pública/privada).** Plausível, mas não discutida na reunião. Não adotada nesta fase, para manter a simplicidade pedida pelo time.

## Consequências

**Positivas**

- Vazamento de secret fica contido a um endpoint.
- Padrão de mercado, com bibliotecas disponíveis para os clientes ([09:20] Sofia).
- Rotação sem janela de indisponibilidade para o cliente.

**Negativas e trade-offs**

- A plataforma passa a armazenar secrets que precisam ser recuperáveis para assinar, o que exige cuidado com armazenamento e com logs.
- Período de 24 horas com duas secrets válidas aumenta a superfície durante a rotação.
- O formato exato de `X-Signature` (prefixo do algoritmo, codificação) e o tamanho da secret não foram discutidos e ficam definidos no FDD como proposta.

## Referências

- [ADR-005](ADR-005-at-least-once-com-x-event-id.md): o header `X-Timestamp` complementa a assinatura contra replay.
- [FDD](../FDD.md), seções "Autenticação das entregas" e "Integração com o sistema existente" (logger).
