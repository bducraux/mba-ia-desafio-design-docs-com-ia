# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

Aceito ([09:30] Larissa: "Decisão: reuso máximo do que já existe.").

- **Decisores:** Larissa, Bruno, Diego, Sofia (no ponto do `requireRole`)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

O OMS já tem padrões consolidados de estrutura, erros, logs e autorização ([09:27] Bruno: "A gente tem um padrão claro na codebase."). O time é pequeno e não quer introduzir bibliotecas ou convenções novas ([09:29] Bruno: "Não vamos botar nada novo.").

Padrões existentes no código:

| Padrão | Onde está |
| --- | --- |
| Módulo por domínio com controller, service, repository, routes e schemas | `src/modules/orders/` (ex.: `src/modules/orders/order.service.ts`, `src/modules/orders/order.routes.ts`) |
| Classe base de erro `AppError(message, statusCode, errorCode, details)` | `src/shared/errors/app-error.ts` |
| Erros específicos com código em CAIXA_ALTA (`InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) | `src/shared/errors/http-errors.ts` |
| Middleware de erro centralizado (trata `AppError`, `ZodError` e erros do Prisma) | `src/middlewares/error.middleware.ts` |
| Autorização por role (`requireRole('ADMIN')`) | `src/middlewares/auth.middleware.ts` |
| Validação com schemas Zod via `validate({ body, params, query })` | `src/middlewares/validate.middleware.ts` |
| Logger Pino com redaction | `src/shared/logger/index.ts` |
| Criação do Prisma client | `src/config/database.ts` |

## Decisão

O módulo de webhooks segue os mesmos padrões dos demais módulos ([09:30] Larissa: "Webhook fica como módulo igual aos outros"):

1. **Estrutura**: novo módulo `modules/webhooks/` com controller, service, repository, routes e schemas, espelhando `src/modules/orders/` ([09:27] Bruno). A lógica do worker fica no próprio módulo, em `webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
2. **Erros**: novas classes estendem `AppError` (ou `NotFoundError`, `ConflictError` e `UnprocessableEntityError`, de `src/shared/errors/http-errors.ts`), com códigos prefixados **`WEBHOOK_`**, por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno; [09:29] Larissa).
3. **Tratamento de erro HTTP**: sem alteração em `src/middlewares/error.middleware.ts`; ele já serializa qualquer `AppError` ([09:29] Bruno).
4. **Logs**: Pino via `src/shared/logger/index.ts`, sem nova biblioteca ([09:29] Bruno).
5. **Validação**: schemas Zod no padrão de `src/modules/orders/order.schemas.ts` (incluindo a regra de URL `https`, que a reunião tratou como validação de schema: [09:23] Sofia).
6. **Autorização**: o replay da DLQ reutiliza `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
7. **Integração com pedidos**: em vez de injetar um repository de webhooks no `OrderService`, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o client da transação corrente e é chamada por `changeStatus` em `src/modules/orders/order.service.ts` ([09:41] Bruno; [09:41] Diego: "função pura recebendo o tx").

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Introduzir biblioteca ou estrutura própria para o módulo** (logger dedicado, framework de jobs, hierarquia de erros nova) | Contraria a diretriz de não adicionar nada novo ([09:29] Bruno) e aumentaria a carga de manutenção de um time pequeno ([09:07] Diego). |
| **Injetar o repository de webhooks inteiro no `OrderService`** | Acoplaria pedidos ao módulo de webhooks; a função que recebe o `tx` é suficiente ([09:41] Bruno; [09:41] Diego). |

## Consequências

**Positivas**

- Curva de aprendizado mínima, porque o código novo se parece com o existente.
- Erros do módulo saem no mesmo formato JSON (`{ error: { code, message, details } }`) sem mudar o middleware.
- O acoplamento entre pedidos e webhooks fica restrito a uma única chamada dentro da transação.

**Negativas**

- O módulo herda as limitações dos padrões atuais (por exemplo, não existe infraestrutura de métricas ou tracing, então a observabilidade se apoia em logs Pino e em dados das tabelas).
- O `requireRole` só conhece `ADMIN` e `OPERATOR`; regras de permissão mais finas para o CRUD ficaram para depois ([09:37] Sofia).

**Trade-off explícito:** consistência e velocidade de entrega acima de soluções especializadas que talvez fossem mais adequadas isoladamente.
