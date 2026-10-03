# ADR-007: Payload do evento renderizado como snapshot na inserção da outbox

## Status

Aceito ([09:52] Bruno: "Beleza, snapshot. Decidido.").

- **Decisores:** Larissa, Diego, Bruno (conversa pós-reunião, [09:51]–[09:52])
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

Ao modelar a outbox, era preciso decidir se a linha do evento guarda o **payload já renderizado** ou apenas o `order_id`, renderizando no momento do envio ([09:51] Bruno).

- Entre a mudança de status e o envio podem passar de segundos (polling) até cerca de 15 horas (retries do [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)). Nesse intervalo o pedido pode mudar de novo.
- O payload combinado é enxuto: `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`, sem os itens ([09:43] Diego). Quem quiser detalhes consulta `GET /orders/:id` ([09:43] Diego).

## Decisão

Renderizar o payload **na inserção**, dentro da transação de `changeStatus`, e gravá-lo na outbox. O evento passa a refletir o estado do pedido **no momento em que o status mudou** ([09:52] Larissa; [09:52] Diego: "snapshot na inserção").

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Guardar só o `order_id` e renderizar no envio** | Se o pedido mudar depois, o evento enviado não corresponderia à transição que o originou; "Senão tem caso esquisito" ([09:52] Larissa). |

## Consequências

**Positivas**

- Cada evento é imutável e fiel à transição que o gerou, inclusive em retries e replays da DLQ.
- O worker não precisa consultar `orders` para montar o envio: lê a linha e assina o corpo pronto.
- A assinatura HMAC ([ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)) é calculada sobre um corpo estável.

**Negativas**

- Ocupa mais espaço na outbox do que guardar só o id.
- Se o formato do payload mudar, eventos já enfileirados saem no formato antigo.
- Dados que o cliente veja depois em `GET /orders/:id` podem ser mais novos que o snapshot recebido; isso é esperado e precisa ser comunicado.

**Trade-off explícito:** fidelidade do evento ao momento da transição, em troca de armazenamento extra e de payloads "congelados" na versão do momento da inserção.
