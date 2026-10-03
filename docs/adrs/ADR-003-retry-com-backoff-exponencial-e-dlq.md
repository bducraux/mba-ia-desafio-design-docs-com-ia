# ADR-003: Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

Aceito ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."; DLQ separada e replay manual anotados em [09:19] Larissa e confirmados no resumo [09:48] Larissa).

- **Decisores:** Larissa, Diego, Bruno; Marcos validou a janela ([09:17] Marcos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar ou lento ([09:14] Larissa). Era preciso decidir quantas vezes tentar de novo, com que intervalo, e o que fazer quando as tentativas se esgotam.

- Clientes já tiveram indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).
- Uma chamada que não responde em 10 segundos é tratada como falha e vai para retry ([09:42] Diego).
- Eventos não podem ficar "pendurados pra sempre" se o cliente sumiu ([09:15] Diego).

## Decisão

1. **Backoff exponencial com 5 tentativas** e intervalos de **1 min, 5 min, 30 min, 2 h e 12 h**, cerca de 15 horas entre a primeira falha e a última tentativa ([09:17] Diego; [09:17] Larissa).
2. Esgotadas as tentativas, o evento é considerado **falha permanente** e vai para a **DLQ** ([09:15] Diego).
3. A DLQ é uma **tabela separada `webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp, e não um status "failed" na própria outbox. Isso deixa a leitura da outbox mais limpa e guarda evidência para debug e reprocessamento ([09:18] Diego).
4. O **reprocessamento é manual**, via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role `ADMIN` ([09:36] Larissa) e registra quem fez o replay, para auditoria ([09:36] Sofia).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Retry indefinido com backoff** | Evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Proposta por Bruno ([09:16] Bruno). Cobriria só cerca de 30 minutos e descartaria eventos de clientes em manutenção de horas ([09:16] Diego: "3 é pouco"). |
| **Marcar como "failed" na própria outbox** | Levantada por Larissa ([09:17] Larissa). Preterida pela tabela separada, que mantém a outbox enxuta para o worker e concentra a evidência de falha ([09:18] Diego). |

## Consequências

**Positivas**

- Cobre indisponibilidades de até cerca de 15 horas sem intervenção humana.
- Falhas permanentes ficam visíveis, auditáveis e reprocessáveis.
- Replay restrito a `ADMIN` e com log de quem executou.

**Negativas**

- Um evento pode chegar com até cerca de 15 horas de atraso no pior caso; Marcos considerou aceitável ([09:17] Marcos).
- O reprocessamento depende de ação manual de um admin; não há reenvio automático a partir da DLQ.
- O cliente não é avisado proativamente quando cai na DLQ: o email de alerta ficou para uma próxima fase ([09:37] Larissa).
- Mais uma tabela para manter.

**Trade-off explícito:** prioriza-se a não perda de eventos em indisponibilidades longas, com prazo finito e evidência para reprocessar, em troca de latência alta em cenários de falha e de operação manual da DLQ.
