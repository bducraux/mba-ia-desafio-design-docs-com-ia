# ADR-004: Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h

## Status

Aceito ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h.").

- **Decisores:** Sofia (Segurança), Larissa, Bruno, Diego
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

A feature envia eventos com dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir **validar a origem** da requisição e **detectar adulteração** do payload ([09:19] Sofia).

- Já houve cliente que vazou secret no log da própria aplicação ([09:22] Diego).
- O cliente precisa de uma forma de trocar a secret sem interromper o recebimento ([09:21] Sofia).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**, com a assinatura enviada no header `X-Signature`; o cliente verifica do lado dele ([09:20] Sofia; [09:22] Sofia).
2. **Secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia). A configuração do webhook guarda url, secret, `customer_id` e estado ativo ([09:21] Bruno; [09:21] Sofia).
3. A **secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. **Rotação via API**: o cliente pede uma nova secret; a antiga continua válida em paralelo por **24 horas** e depois expira ([09:21] Sofia).
5. A implementação de HMAC e de geração de secret passa por **revisão de segurança da Sofia**, com pelo menos dois dias úteis antes do deploy ([09:46] Sofia).

Validações complementares, que a própria reunião classificou como requisitos e não como decisões arquiteturais ([09:23] Sofia; [09:24] Larissa), ficam detalhadas no FDD: URL obrigatoriamente `https` e limite de payload de 64KB.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Secret global da plataforma** | Um único vazamento comprometeria todos os clientes: "Senão se vaza uma, vaza tudo" ([09:21] Sofia). |
| **Rotação sem período de convivência** (troca imediata) | Plausível, mas o cliente precisa de tempo para migrar seus sistemas; por isso a antiga fica válida por 24h ([09:21] Sofia). |

## Consequências

**Positivas**

- Cliente valida autenticidade e integridade com bibliotecas padrão: "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- Vazamento de uma secret afeta só um endpoint, e a rotação permite contê-lo sem downtime.

**Negativas**

- A plataforma passa a armazenar secrets que precisam ser legíveis pelo worker (para assinar), o que aumenta a superfície sensível do banco e dos logs.
- Durante as 24h de rotação há duas secrets válidas por endpoint, o que adiciona estado e lógica (detalhes de assinatura nesse período no FDD).
- Toda a validação de assinatura fica a cargo do cliente.

**Trade-off explícito:** aceita-se a complexidade de guardar e rotacionar uma secret por endpoint em troca de isolamento de vazamentos e de uma troca de secret sem interrupção para o cliente.
