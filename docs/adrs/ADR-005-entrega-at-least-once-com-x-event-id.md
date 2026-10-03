# ADR-005: Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

## Status

Aceito ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.").

- **Decisores:** Diego, Larissa; Sofia registrou a ressalva; Marcos assumiu a comunicação aos clientes
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), um mesmo evento pode ser enviado mais de uma vez. Por exemplo, o cliente processa o evento mas a resposta não chega dentro do timeout, e o worker tenta de novo. Era preciso definir qual garantia a plataforma oferece e como o cliente trata duplicatas ([09:24] Diego; [09:25] Bruno).

## Decisão

1. A plataforma garante entrega **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado ([09:24] Diego).
2. Cada evento recebe um **UUID gerado no momento em que entra na outbox**, enviado no header **`X-Event-Id`**. Ele é único por evento e se mantém em todas as tentativas e no replay, e o cliente deduplica por ele ([09:25] Diego).
3. O comportamento é documentado em destaque no portal de desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Exactly-once** | Exigiria coordenação dos dois lados e "fica muito mais complexo" ([09:25] Diego). |
| **At-most-once** (enviar uma vez, sem retry) | Plausível, mas incompatível com a decisão de retry do [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) e com o objetivo de não perder notificações de clientes temporariamente fora do ar. |

## Consequências

**Positivas**

- Segue o padrão de mercado ("Stripe faz assim, GitHub faz assim", [09:25] Diego) e "resolve 99% dos casos".
- Combina naturalmente com outbox e retry: nenhum evento confirmado é perdido silenciosamente.

**Negativas**

- Transfere ao cliente a responsabilidade de deduplicar ([09:25] Sofia: "Isso joga responsabilidade pro cliente.").
- Clientes que não implementarem a deduplicação podem processar o mesmo evento duas vezes.

**Trade-off explícito:** simplicidade e confiabilidade do lado da plataforma em troca de exigir idempotência do consumidor.
