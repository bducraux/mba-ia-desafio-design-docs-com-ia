# ADR-001: Padrão Outbox no MySQL para publicar eventos de mudança de status

## Status

Aceito — decidido na reunião técnica de webhooks ([09:08] Larissa: "Tá decidido então: outbox em MySQL.").

- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-na-insercao.md)

## Contexto

Três clientes B2B querem ser notificados quando o status dos seus pedidos muda, em vez de fazer polling em `GET /orders` ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação Prisma (`this.prisma.$transaction`) que já atualiza `orders`, insere em `order_status_history` e debita ou repõe `stock_quantity` dos produtos ([09:04] Bruno).

A primeira pergunta foi se a notificação seria disparada de forma síncrona dentro desse service ou desacoplada ([09:03] Larissa). Restrições levantadas:

- A transação já é pesada; uma chamada HTTP no meio dela faria um cliente lento travar mudanças de status de outros pedidos ([09:04] Bruno).
- Se o cliente estiver fora do ar, não é aceitável dar rollback na mudança de status ([09:04] Bruno).
- O evento precisa existir **se e somente se** a mudança de status for confirmada: "se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto" ([09:06] Diego).
- O time é pequeno e não quer subir infraestrutura nova ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox sobre o MySQL existente**:

1. Dentro da mesma transação de `changeStatus`, inserir uma linha na nova tabela `webhook_outbox` com o evento ([09:06] Diego; [09:40] Bruno).
2. Se a inserção na outbox falhar, a transação inteira sofre rollback: "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno; [09:41] Diego).
3. Um worker separado lê a tabela e faz as chamadas HTTP (detalhado no [ADR-002](ADR-002-worker-separado-em-polling.md)).
4. A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`; o worker lê os pendentes em batch pequeno ([09:08] Diego).
5. Identificadores em UUID, seguindo o padrão do `prisma/schema.prisma` ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo HTTP síncrono no `changeStatus`** | Acopla a latência e a disponibilidade do cliente à transação de pedidos; cliente lento trava status de outros pedidos; falha do cliente forçaria rollback do status ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams (ou fila externa equivalente)** | Exigiria subir e operar infraestrutura nova; para um time pequeno, "Subir Redis Cluster pra isso é overengineering" ([09:07] Larissa; [09:07] Diego). Além disso, publicar numa fila externa fora da transação do MySQL não teria a mesma atomicidade com a mudança de status. |

## Consequências

**Positivas**

- Atomicidade entre mudança de status e registro do evento, sem coordenação distribuída ([09:06] Diego).
- Nenhuma infraestrutura nova: usa o MySQL e o Prisma já existentes.
- A tabela serve de registro auditável de tudo o que foi (ou deveria ser) enviado.

**Negativas**

- A transação de `changeStatus` ganha mais uma escrita (e uma leitura das configurações de webhook do customer para aplicar o filtro de eventos — ver FDD).
- O MySQL passa a absorver a carga de leitura do worker; a tabela cresce e precisa de arquivamento, que ficou **fora do escopo** desta feature ([09:08] Diego: "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature").
- A latência de entrega passa a depender do intervalo de leitura do worker (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

**Trade-off explícito:** troca-se latência mínima e uma escrita extra na transação de pedidos por consistência garantida entre status e evento sem nova infraestrutura.
