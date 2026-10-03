# ADR-002: Worker em processo separado, lendo a outbox por polling de 2 segundos

## Status

Aceito ([09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s.").

- **Decisores:** Larissa, Diego, Bruno; Marcos validou o impacto de produto ([09:10] Marcos: "2 segundos serve, perfeito.")
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com a outbox decidida no [ADR-001](ADR-001-outbox-no-mysql.md), faltava definir **como** os eventos são lidos e **onde** esse processamento roda ([09:08] Larissa).

- Os clientes consideram "tempo real" qualquer entrega abaixo de 10 segundos ([09:02] Marcos).
- O MySQL não tem um mecanismo nativo de notificação para processos externos como o `NOTIFY/LISTEN` do Postgres; triggers só executam SQL ([09:09] Diego).
- Se o worker rodar dentro do processo da API, um restart da API derruba o worker ([09:11] Diego).
- Hoje o projeto tem uma única entry-point, `src/server.ts`, que sobe o Express e fecha o `PrismaClient` no shutdown.

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos, processa e marca o resultado ([09:09] Diego). Isso atende o requisito de menos de 10 segundos com folga ([09:09] Diego; [09:10] Larissa).
2. **Processo separado da API**, com nova entry-point `worker.ts` ao lado de `src/server.ts` e um script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo de webhooks, em `webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
3. **Mesmo banco e mesma stack, instância própria de `PrismaClient`**: mesma `DATABASE_URL`, mas um cliente novo porque é outro processo Node ([09:11] Diego; [09:30] Bruno). A criação reutiliza `createPrismaClient()` de `src/config/database.ts`.
4. **Single-worker**: um único worker processa em ordem de `created_at`, o que dá ordenação implícita por `order_id`. Não há garantia de ordenação global, e isso é registrado como **limitação conhecida** ([09:12] Diego; [09:13] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Trigger do banco notificando o worker (reativo)** | Proposta por Bruno ([09:09] Bruno). MySQL não tem listener nativo; avisar um processo externo exigiria improvisos como escrever em arquivo ou chamar um endpoint ([09:09] Diego). |
| **Worker dentro do processo da API** | Restart ou deploy da API interromperia o processamento ([09:11] Diego: "Só não pode ser o mesmo processo"). |
| **Múltiplos workers em paralelo** | Perderia a garantia de ordem por pedido; escalar exigiria particionar por `order_id` ou usar lock pessimista. Adiado: "isso é problema do futuro, não agora" ([09:13] Diego). |

## Consequências

**Positivas**

- Implementação simples, sem dependências novas; atende a meta de menos de 10 segundos.
- Isolamento de falhas: a API não é afetada por lentidão ou travamento do worker, e vice-versa.
- Ordenação por pedido garantida enquanto houver um único worker.

**Negativas**

- Até 2 segundos de espera do polling, mais o tempo de entrega, antes de o cliente ser notificado ([09:10] Larissa: "Aceitamos").
- Consultas periódicas ao MySQL mesmo sem eventos pendentes.
- Um único worker é ponto único de processamento: se ele parar, as entregas atrasam (os eventos não se perdem, ficam pendentes na outbox).
- Escalar horizontalmente exigirá nova decisão (particionamento ou lock).

**Trade-off explícito:** aceita-se latência de até 2 segundos de polling e throughput limitado a um worker em troca de simplicidade operacional e ordenação por pedido.
