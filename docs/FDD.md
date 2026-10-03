# FDD — Sistema de Webhooks de Notificação de Pedidos

> Documento de implementação. O "por quê" das decisões está nos [ADRs](adrs/) e a proposta geral no [RFC](RFC.md). Aqui está o "como construir".
> Convenção: itens marcados como **(proposta de implementação)** não foram decididos na reunião; são escolhas deste documento para tornar a especificação acionável e devem ser confirmadas na revisão com Bruno e Diego ([09:50] Larissa).
> Arquivos ainda inexistentes são citados sem o prefixo do diretório raiz (ex.: módulo `modules/webhooks/`, entry-point `worker.ts`) para não serem confundidos com arquivos que já existem.

## 1. Contexto e motivação técnica

A mudança de status de pedido é feita por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de `this.prisma.$transaction`, que já valida a transição (`src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e grava `order_status_history`. Não existe nenhum mecanismo de eventos, filas ou chamadas HTTP de saída no projeto.

Os clientes B2B precisam ser notificados em menos de 10 segundos ([09:02] Marcos) sem que a transação de pedidos dependa da disponibilidade deles ([09:04] Bruno). A solução técnica combina outbox transacional ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)), worker em polling ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)), retry e DLQ ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)), assinatura HMAC ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)), entrega at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)), reuso dos padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) e payload como snapshot ([ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md)).

## 2. Objetivos técnicos

| ID | Objetivo |
| --- | --- |
| FDD-OBJ-01 | Registrar o evento na outbox **na mesma transação** da mudança de status; falha na gravação reverte o status ([09:40] Bruno). |
| FDD-OBJ-02 | Entregar o evento ao endpoint do cliente em **menos de 10 segundos** em condições normais: até 2s de polling mais a chamada com timeout de 10s ([09:02] Marcos; [09:09] Diego). |
| FDD-OBJ-03 | Não perder eventos por indisponibilidade temporária do cliente: 5 retentativas com backoff 1m/5m/30m/2h/12h e DLQ ([09:17] Larissa). |
| FDD-OBJ-04 | Permitir que o cliente valide origem e integridade via HMAC-SHA256 com secret própria e rotacionável ([09:22] Sofia). |
| FDD-OBJ-05 | Não introduzir infraestrutura nem bibliotecas novas ([09:07] Diego; [09:29] Bruno). |

## 3. Escopo e exclusões

**Incluso**

- Evento `order.status_changed` emitido por `changeStatus` para webhooks ativos do customer que assinam o status de destino ([09:33] Marcos; [09:43] Diego).
- CRUD de configuração de webhooks, rotação de secret e histórico de entregas ([09:31] Marcos; [09:33] Bruno; [09:21] Sofia; [09:34] Marcos).
- Worker de entrega, retry, DLQ e replay manual por admin ([09:18] Diego; [09:36] Larissa).

**Exclusões**

| Item | Origem |
| --- | --- |
| Email de alerta ao cliente quando o webhook falha | Adiado para próxima fase ([09:37] Larissa) |
| Rate limiting de envio por cliente | "Observar e decidir depois" ([09:39] Larissa) |
| Dashboard visual para o cliente | Projeto separado do time de frontend ([09:40] Larissa) |
| Múltiplos workers em paralelo | Adiado ([09:13] Diego) |
| Arquivamento/limpeza de eventos entregues | Fora do escopo desta feature ([09:08] Diego) |
| Webhooks inbound (cliente enviando para nós) | Só outbound ([09:02] Marcos) |
| Evento na **criação** do pedido (`OrderService.create`) | A integração acordada é só em `changeStatus` ([09:40] Bruno); o estado inicial `PENDING` não é emitido |

## 4. Modelo de dados

Novos models em `prisma/schema.prisma`, seguindo o padrão do arquivo: `id String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), nomes de tabela em snake_case via `@@map`, `createdAt`/`updatedAt`.

| ID | Model / tabela | Campos principais | Origem |
| --- | --- | --- | --- |
| FDD-DADOS-01 | `Webhook` / `webhooks` | `id`, `customerId` (FK `customers`), `url`, `secret`, `previousSecret?`, `previousSecretExpiresAt?`, `events` (Json: lista de `OrderStatus`), `active` (Boolean), timestamps; `@@index([customerId])` | [09:21] Bruno; [09:21] Sofia; [09:33] Marcos |
| FDD-DADOS-02 | `WebhookOutbox` / `webhook_outbox` | `id`, `eventId` (UUID do evento → `X-Event-Id`), `webhookId`, `orderId`, `eventType`, `payload` (Json, snapshot), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attempts` (Int), `nextAttemptAt`, `lastError?`, timestamps; `@@index([status])`, `@@index([createdAt])` | [09:06] Diego; [09:08] Diego; [09:25] Diego; [09:52] Larissa |
| FDD-DADOS-03 | `WebhookDelivery` / `webhook_deliveries` | `id`, `webhookId`, `outboxId`, `eventId`, `attempt`, `success`, `responseStatus?`, `responseBody?`, `durationMs`, `error?`, `createdAt`; `@@index([webhookId, createdAt])` | [09:34] Marcos |
| FDD-DADOS-04 | `WebhookDeadLetter` / `webhook_dead_letter` | `id`, `outboxId`, `webhookId`, `eventId`, `payload` (Json), `reason`, `failedAt`, `replayedAt?`, `replayedById?` (FK `users`) | [09:18] Diego; [09:36] Sofia |

Notas:

- **Uma linha de outbox por par (evento, webhook)** (proposta de implementação). Um customer pode ter vários endpoints ([09:44] Sofia), cada um com secret, retries e histórico próprios. Todas as linhas geradas pela mesma mudança de status compartilham o mesmo `eventId`.
- `events` aceita apenas status de destino possíveis em `changeStatus` (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`, conforme `src/modules/orders/order.status.ts`).
- `FAILED` na outbox significa que o evento esgotou as retentativas e foi copiado para `webhook_dead_letter`.

## 5. Fluxos detalhados

### FDD-FLUXO-01 — Criação do evento na outbox (dentro da transação de status)

1. `changeStatus` executa como hoje: valida transição, ajusta estoque, `tx.order.update` e `tx.orderStatusHistory.create`.
2. Em seguida chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` com o **mesmo `tx`** ([09:41] Bruno).
3. A função busca, com o `tx`, os webhooks `active = true` do `order.customerId` cujo `events` contém `toStatus`. Se não houver nenhum, retorna sem inserir ([09:34] Bruno).
4. Gera um `eventId` (UUID) e renderiza o payload (snapshot, [09:52] Larissa) com os campos acordados ([09:43] Diego).
5. Insere uma linha `PENDING` em `webhook_outbox` por webhook elegível, com `attempts = 0` e `nextAttemptAt = now()`.
6. Qualquer exceção propaga e o Prisma faz rollback da transação inteira; o status **não** muda sem evento ([09:40] Bruno; [09:41] Diego).

### FDD-FLUXO-02 — Processamento pelo worker (polling)

1. O worker (`npm run worker`, processo próprio, [09:11] Larissa) roda um loop a cada **2 segundos** ([09:09] Diego).
2. Busca um lote pequeno ([09:08] Diego) de linhas `PENDING` com `nextAttemptAt <= now()`, ordenadas por `createdAt` (ordenação por pedido com worker único, [09:12] Diego).
3. Marca cada linha como `PROCESSING` e, para cada uma:
   - carrega o `Webhook`; se estiver inativo ou removido, encerra a linha sem envio (ver erros `WEBHOOK_INACTIVE` e `WEBHOOK_NOT_FOUND`);
   - serializa o payload; se passar de **64KB**, não envia (FDD-FLUXO-04);
   - calcula `X-Signature` = HMAC-SHA256 do corpo com a secret do webhook ([09:22] Sofia);
   - faz `POST` para a URL com timeout de **10 segundos** ([09:42] Diego) e os headers da seção 6.8.
4. **Resposta 2xx**: linha `DELIVERED`, grava `webhook_deliveries` com `success = true`, status, corpo da resposta e `durationMs` ([09:34] Marcos).
5. **Timeout, erro de rede ou resposta não-2xx**: grava `webhook_deliveries` com `success = false` e segue o FDD-FLUXO-03.

### FDD-FLUXO-03 — Retry com backoff

Interpretação de "5 tentativas, backoff 1m/5m/30m/2h/12h" e "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego): **1 envio inicial + 5 retentativas**.

| Falha nº | Próxima tentativa em | Tempo acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 (envio inicial) | +1 min | 1 min |
| 2 | +5 min | 6 min |
| 3 | +30 min | 36 min |
| 4 | +2 h | 2 h 36 min |
| 5 | +12 h | 14 h 36 min |
| 6 (5ª retentativa) | — vai para DLQ | — |

A cada falha: `attempts += 1`, `lastError` preenchido, `status = PENDING` e `nextAttemptAt = now() + intervalo[attempts]`.

### FDD-FLUXO-04 — DLQ

1. Na 6ª falha, ou em erro não recuperável (payload acima de 64KB, ou webhook sem secret válida), o worker, numa transação:
   - insere em `webhook_dead_letter` o payload, o `reason` (código `WEBHOOK_*` e mensagem) e `failedAt` ([09:18] Diego);
   - marca a linha da outbox como `FAILED`.
2. Erros não recuperáveis vão direto para a DLQ, sem retry (proposta de implementação: repetir não muda o resultado). Isso honra "se chegou nesse tamanho, tem algo errado" ([09:23] Sofia) **sem** bloquear a mudança de status, que já foi confirmada.

### FDD-FLUXO-05 — Replay manual da DLQ

1. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')` ([09:36] Larissa).
2. Numa transação: valida que a dead letter existe e ainda não foi reprocessada, e que o webhook está ativo. Cria uma **nova** linha `PENDING` na outbox com o **mesmo `eventId` e payload** ("Recoloca na outbox como pendente", [09:18] Diego; o `X-Event-Id` estável permite deduplicação, [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)), `attempts = 0`, e preenche `replayedAt` e `replayedById`.
3. Registra log de auditoria com o id do admin ([09:36] Sofia), como descrito na seção 9.

### FDD-FLUXO-06 — Rotação de secret

1. O cliente chama o endpoint de rotação (FDD-CONTRATO-05).
2. A secret atual vira `previousSecret`, com `previousSecretExpiresAt = now() + 24h`, e uma nova secret é gerada e devolvida uma única vez ([09:21] Sofia).
3. Durante as 24h o worker envia `X-Signature` com a nova secret e `X-Signature-Previous` com a antiga (**proposta de implementação**, a validar na revisão de segurança, [09:46] Sofia). Assim o cliente valida com qualquer uma das duas enquanto migra.
4. Após a expiração, `previousSecret` é ignorada e limpa na próxima rotação ou leitura.

## 6. Contratos públicos

Base: `/api/v1` (montagem em `src/app.ts`). Todos os endpoints de gestão usam `authenticate` ([09:32] Marcos: autenticados com o JWT da plataforma). O CRUD aceita qualquer role autenticada ([09:37] Sofia); o replay exige `ADMIN` ([09:36] Larissa). Erros seguem o formato do `src/middlewares/error.middleware.ts`: `{ "error": { "code", "message", "details?" } }`.

`customerId` é recebido no **body** (criação) e na **query** (listagem), nunca do JWT ([09:32] Larissa). A forma exata ficou em aberto na reunião (RFC-QA-02); esta é a proposta de implementação.

### FDD-CONTRATO-01 — Cadastrar webhook

`POST /api/v1/webhooks` → `201 Created` ([09:31] Marcos)

```json
{
  "customerId": "6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

```json
{
  "id": "0b8e7c3a-5d2f-4e61-8a9b-7c6d5e4f3a2b",
  "customerId": "6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<secret-gerada-pela-plataforma>",
  "createdAt": "2026-10-02T12:00:00.000Z"
}
```

A `secret` é gerada pela plataforma e só aparece nesta resposta e na de rotação ([09:31] Marcos). Erros: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENTS`, `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### FDD-CONTRATO-02 — Listar webhooks de um customer

`GET /api/v1/webhooks?customerId=6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f` → `200 OK` ([09:33] Bruno)

```json
{
  "data": [
    {
      "id": "0b8e7c3a-5d2f-4e61-8a9b-7c6d5e4f3a2b",
      "customerId": "6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-02T12:00:00.000Z",
      "updatedAt": "2026-10-02T12:00:00.000Z"
    }
  ]
}
```

A secret nunca é listada. Erros: `400 VALIDATION_ERROR` (`customerId` ausente ou inválido), `401 UNAUTHORIZED`.

### FDD-CONTRATO-03 — Editar webhook (url, eventos, ativo/inativo)

`PATCH /api/v1/webhooks/:id` → `200 OK` ([09:33] Bruno; estado ativo: [09:21] Bruno)

```json
{
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": false
}
```

```json
{
  "id": "0b8e7c3a-5d2f-4e61-8a9b-7c6d5e4f3a2b",
  "customerId": "6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": false,
  "updatedAt": "2026-10-02T13:10:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENTS`, `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-04 — Remover webhook

`DELETE /api/v1/webhooks/:id` → `204 No Content` ([09:33] Bruno)

```json
{}
```

Sem corpo de request (o bloco acima representa a ausência de corpo). Resposta sem corpo. As linhas de outbox, o histórico de entregas e a DLQ do webhook são removidos junto, via `onDelete: Cascade`, como `OrderItem` faz em `prisma/schema.prisma` (proposta de implementação). Assim, eventos ainda pendentes desse webhook deixam de ser enviados. Se a remoção acontecer enquanto o worker processa um lote que já continha a linha, o worker encerra o envio com `WEBHOOK_NOT_FOUND`. Para pausar sem perder histórico, use `PATCH` com `"active": false`. Erros: `404 WEBHOOK_NOT_FOUND`.

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

### FDD-CONTRATO-05 — Rotacionar secret

`POST /api/v1/webhooks/:id/secret/rotate` → `200 OK` ([09:21] Sofia; caminho: proposta de implementação)

```json
{}
```

```json
{
  "id": "0b8e7c3a-5d2f-4e61-8a9b-7c6d5e4f3a2b",
  "secret": "<nova-secret-gerada-pela-plataforma>",
  "previousSecretExpiresAt": "2026-10-03T13:30:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-06 — Histórico de entregas

`GET /api/v1/webhooks/:id/deliveries` → `200 OK`, últimas 100 entregas, mais recentes primeiro ([09:34] Marcos)

```json
{
  "data": [
    {
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
      "eventId": "3f2a1b0c-9d8e-4f7a-b6c5-d4e3f2a1b0c9",
      "attempt": 1,
      "success": false,
      "responseStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 412,
      "error": "WEBHOOK_DELIVERY_FAILED",
      "payload": { "event_id": "3f2a1b0c-9d8e-4f7a-b6c5-d4e3f2a1b0c9", "event_type": "order.status_changed" },
      "createdAt": "2026-10-02T14:00:02.000Z"
    }
  ]
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-07 — Replay de evento da DLQ (admin)

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` → `202 Accepted` ([09:18] Diego; [09:36] Larissa)

```json
{}
```

```json
{
  "deadLetterId": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
  "eventId": "3f2a1b0c-9d8e-4f7a-b6c5-d4e3f2a1b0c9",
  "outboxId": "e5f6a7b8-c9d0-4e1f-a2b3-c4d5e6f7a8b9",
  "status": "PENDING",
  "replayedBy": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "replayedAt": "2026-10-03T09:00:00.000Z"
}
```

Erros: `403 FORBIDDEN` (não-ADMIN, via `requireRole`), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `409 WEBHOOK_INACTIVE`.

### FDD-CONTRATO-08 — Entrega ao cliente (outbound)

`POST <url do webhook>`: requisição enviada pelo worker ([09:43] Diego; [09:44] Diego; [09:44] Sofia)

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, estável entre retentativas e replay |
| `X-Webhook-Id` | id do cadastro de webhook |
| `X-Timestamp` | instante do envio (formato ISO 8601: proposta de implementação); permite ao cliente detectar replay attack |
| `X-Signature` | HMAC-SHA256 (hex) do corpo, com a secret do webhook |
| `X-Signature-Previous` | só durante as 24h de rotação (proposta de implementação, FDD-FLUXO-06) |

```json
{
  "event_id": "3f2a1b0c-9d8e-4f7a-b6c5-d4e3f2a1b0c9",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-02T14:00:00.000Z",
  "order_id": "7d6c5b4a-3f2e-4d1c-9b8a-7f6e5d4c3b2a",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a9e-3b7d-4c51-9a0e-1f2b3c4d5e6f",
  "total_cents": 125000
}
```

O payload não inclui itens; detalhes via `GET /orders/:id` ([09:43] Diego). Qualquer resposta **2xx** é sucesso; não-2xx, timeout (10s) ou erro de rede é falha e segue para retry.

## 7. Matriz de erros

Classes novas estendem as de `src/shared/errors/http-errors.ts` (que estendem `AppError`) e passam pelo `src/middlewares/error.middleware.ts` sem alteração ([09:29] Bruno). Prefixo `WEBHOOK_` ([09:29] Larissa).

| ID | Código | HTTP | Onde | Quando | Tratamento |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | `WEBHOOK_NOT_FOUND` | 404 | API / worker | webhook inexistente; no worker, webhook removido durante o processamento do lote | API: erro ao cliente. Worker: encerra o envio sem enviar ([09:28] Bruno) |
| FDD-ERRO-02 | `WEBHOOK_INVALID_URL` | 400 | API | URL malformada ou não `https` | Rejeita o cadastro ou edição ([09:23] Sofia; [09:28] Bruno) |
| FDD-ERRO-03 | `WEBHOOK_SECRET_REQUIRED` | — | worker | webhook sem secret válida no momento da assinatura | Não envia; DLQ direto ([09:28] Bruno) |
| FDD-ERRO-04 | `WEBHOOK_INVALID_EVENTS` | 400 | API | `events` vazio ou com status que `changeStatus` nunca produz (ex.: `PENDING`) | Rejeita (filtro: [09:33] Marcos) |
| FDD-ERRO-05 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` inexistente no cadastro | Rejeita |
| FDD-ERRO-06 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — | worker | corpo serializado acima de 64KB | Não envia; DLQ direto ([09:24] Larissa) |
| FDD-ERRO-07 | `WEBHOOK_DELIVERY_TIMEOUT` | — | worker | sem resposta em 10s | Falha → retry ([09:42] Diego) |
| FDD-ERRO-08 | `WEBHOOK_DELIVERY_FAILED` | — | worker | resposta não-2xx ou erro de rede/TLS | Falha → retry ([09:15] Diego) |
| FDD-ERRO-09 | `WEBHOOK_MAX_RETRIES_EXCEEDED` | — | worker | 5 retentativas esgotadas | Move para DLQ ([09:17] Larissa) |
| FDD-ERRO-10 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API (admin) | id de DLQ inexistente no replay | Erro ao admin |
| FDD-ERRO-11 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API (admin) | dead letter já reprocessada | Evita reenfileiramento duplicado |
| FDD-ERRO-12 | `WEBHOOK_INACTIVE` | 409 | API (admin) / worker | webhook desativado: no replay, ou com evento pendente no worker | API: erro. Worker: encerra a linha sem envio |

Os erros sem HTTP são do worker: aparecem em `webhook_deliveries.error`, `webhook_dead_letter.reason` e nos logs. Sobre `WEBHOOK_INVALID_URL`: o `src/middlewares/validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`. Para expor o código específico, o schema Zod do módulo valida a URL (`z.string().url()` com refinamento `https`, [09:23] Sofia) dentro do service, via `safeParse`, e relança como `WebhookInvalidUrlError` (proposta de implementação).

## 8. Estratégias de resiliência

| ID | Estratégia | Detalhe |
| --- | --- | --- |
| FDD-RES-01 | **Timeout** | 10 segundos por chamada HTTP, via `AbortController` no `fetch` ([09:42] Diego) |
| FDD-RES-02 | **Retries** | 5 retentativas após a falha inicial (FDD-FLUXO-03) ([09:17] Larissa) |
| FDD-RES-03 | **Backoff** | Exponencial com intervalos fixos 1m/5m/30m/2h/12h, persistidos em `nextAttemptAt`, para sobreviver a restarts do worker ([09:17] Diego) |
| FDD-RES-04 | **Fallback** | DLQ em tabela separada com replay manual por admin ([09:18] Diego). O fallback por email está explicitamente adiado ([09:37] Larissa) |
| FDD-RES-05 | **Isolamento** | Worker em processo próprio, com `PrismaClient` próprio criado via `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno); falha do worker não afeta a API ([09:11] Diego) |
| FDD-RES-06 | **Recuperação de crash** | Na inicialização, o worker devolve para `PENDING` as linhas que ficaram em `PROCESSING`. É seguro com worker único ([09:12] Diego) e pode gerar reenvio, coberto pelo at-least-once (proposta de implementação; [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) |
| FDD-RES-07 | **Graceful shutdown** | SIGINT/SIGTERM: termina o lote corrente e chama `prisma.$disconnect()`, como `src/server.ts` faz |
| FDD-RES-08 | **Limite de payload** | Acima de 64KB não envia (FDD-ERRO-06) ([09:24] Diego) |

## 9. Observabilidade

O projeto não tem infraestrutura de métricas nem de tracing distribuído, e a decisão foi não adicionar nada novo ([09:29] Bruno). A observabilidade se apoia no logger Pino existente (`src/shared/logger/index.ts`) e nas próprias tabelas.

**Logs (Pino, JSON estruturado)**

| ID | Evento de log | Campos |
| --- | --- | --- |
| FDD-OBS-01 | `webhook_event_enqueued` (API) | `requestId`, `eventId`, `orderId`, `webhookIds`, `fromStatus`, `toStatus` |
| FDD-OBS-02 | `webhook_delivery_succeeded` / `webhook_delivery_failed` (worker) | `eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs`, `error` |
| FDD-OBS-03 | `webhook_dead_lettered` (worker, nível `warn`) | `eventId`, `webhookId`, `reason` |
| FDD-OBS-04 | `webhook_dead_letter_replayed` (API, auditoria) | `deadLetterId`, `eventId`, `userId` do admin ([09:36] Sofia) |

As secrets **nunca** são logadas: o caminho `*.secret` (e `*.previousSecret`) entra na lista `redactPaths` do logger. Já houve cliente que vazou secret em log ([09:22] Diego).

**Métricas (derivadas das tabelas e dos logs, sem nova stack)**

| ID | Métrica | Fonte | Uso |
| --- | --- | --- | --- |
| FDD-OBS-05 | Latência de entrega (`createdAt` da outbox até o sucesso) | `webhook_outbox` + `webhook_deliveries` | Verificar a meta de menos de 10s ([09:02] Marcos) |
| FDD-OBS-06 | Taxa de sucesso por webhook e por customer | `webhook_deliveries` | Saúde das integrações; base para decidir o alerta por email ([09:37] Larissa) |
| FDD-OBS-07 | Backlog: linhas `PENDING` vencidas | `webhook_outbox` | Detectar worker parado ou lento |
| FDD-OBS-08 | Volume na DLQ e replays | `webhook_dead_letter` | Falhas permanentes |
| FDD-OBS-09 | Eventos por customer por minuto | `webhook_outbox` | Dado para a decisão sobre rate limiting ([09:39] Diego: "a gente observa") |

**Tracing (correlação por identificadores)**

- FDD-OBS-10: o `requestId` do `PATCH /orders/:id/status`, gerado ou propagado pelo `src/middlewares/request-logger.middleware.ts` como `X-Request-Id`, é logado junto do `eventId` no `webhook_event_enqueued`. Isso liga a requisição que mudou o status ao evento.
- O `eventId` acompanha todos os logs do worker, as linhas de `webhook_deliveries` e `webhook_dead_letter`, e chega ao cliente como `X-Event-Id` ([09:25] Diego). Com isso dá para rastrear o caminho completo **requisição → outbox → tentativas → DLQ/replay → cliente** filtrando os logs por `eventId`. Tracing distribuído com instrumentação dedicada fica fora desta fase.

## 10. Dependências e compatibilidade

- **Sem dependências npm novas**: HTTP de saída com o `fetch` nativo do Node 20 e HMAC/secret com o módulo `crypto` do Node (o projeto usa `@types/node` 20 em `package.json`). UUID com o pacote `uuid` já presente.
- **Banco**: migration Prisma **aditiva** (4 tabelas novas), aplicada pelo fluxo existente (`npm run db:migrate`); nenhuma tabela existente é alterada.
- **Novo script** `worker` no `package.json`, análogo a `start`/`dev`, executando a nova entry-point `worker.ts` ([09:11] Larissa).
- **Compatibilidade da API de pedidos**: o contrato de `PATCH /orders/:id/status` não muda. A diferença de comportamento é que uma falha ao gravar na outbox passa a retornar erro e reverter o status ([09:40] Bruno).
- **Clientes**: precisam aceitar `https`, validar HMAC-SHA256 e deduplicar por `X-Event-Id`. Isso será documentado no portal de desenvolvedor ([09:26] Marcos).
- **Revisão de segurança**: pelo menos dois dias úteis da Sofia antes do deploy, focada em HMAC e geração de secret ([09:46] Sofia).

## 11. Critérios de aceite técnicos

| ID | Critério |
| --- | --- |
| FDD-CA-01 | Mudança de status para um status assinado cria uma linha `PENDING` por webhook ativo elegível, na mesma transação; mudança para status não assinado não cria linha ([09:34] Bruno). |
| FDD-CA-02 | Se a inserção na outbox falhar, o status do pedido, o histórico e o estoque permanecem inalterados (rollback) ([09:40] Bruno). |
| FDD-CA-03 | Com o cliente respondendo 2xx, o evento é entregue em menos de 10 segundos após o commit ([09:02] Marcos). |
| FDD-CA-04 | O corpo recebido pelo cliente valida contra `X-Signature` com HMAC-SHA256 e a secret do webhook ([09:22] Sofia). |
| FDD-CA-05 | Após rotação, ambas as secrets validam por 24h e só a nova depois disso ([09:21] Sofia). |
| FDD-CA-06 | Cliente fora do ar recebe as retentativas em +1m/+5m/+30m/+2h/+12h e, depois da última, o evento está na DLQ com motivo ([09:17] Larissa; [09:18] Diego). |
| FDD-CA-07 | Replay por `ADMIN` recoloca o evento como `PENDING` com o mesmo `X-Event-Id` e registra quem fez; `OPERATOR` recebe 403 ([09:36] Larissa; [09:36] Sofia). |
| FDD-CA-08 | Cadastro com URL `http://` retorna 400 `WEBHOOK_INVALID_URL` ([09:23] Sofia). |
| FDD-CA-09 | Payload acima de 64KB não é enviado e vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE` ([09:24] Larissa). |
| FDD-CA-10 | Com worker único, eventos do mesmo pedido chegam na ordem das mudanças de status ([09:12] Diego). |
| FDD-CA-11 | `GET /webhooks/:id/deliveries` retorna no máximo as 100 últimas entregas, com sucesso/falha, payload, resposta e tempo ([09:34] Marcos). |

## 12. Riscos e mitigação

| ID | Risco | Mitigação |
| --- | --- | --- |
| FDD-RISCO-01 | Aumento do tempo da transação de `changeStatus` (consulta de webhooks e inserts) | Consulta indexada por `customerId`; inserção só quando há webhook elegível ([09:34] Bruno) |
| FDD-RISCO-02 | Crescimento contínuo de `webhook_outbox` e `webhook_deliveries` | Índices em `status` e `createdAt` ([09:08] Diego); arquivamento planejado como trabalho futuro (RFC-QA-04) |
| FDD-RISCO-03 | Reenvio duplicado após crash do worker (FDD-RES-06) | Esperado pela semântica at-least-once; cliente deduplica por `X-Event-Id` ([09:24] Diego) |
| FDD-RISCO-04 | Secret exposta em logs ou respostas | Redaction no Pino, secret só nas respostas de criação e rotação, revisão de segurança ([09:22] Diego; [09:46] Sofia) |
| FDD-RISCO-05 | Worker parado sem ninguém perceber | Métrica de backlog (FDD-OBS-07); eventos não se perdem, ficam `PENDING` ([09:06] Diego) |
| FDD-RISCO-06 | Rajadas de eventos para um mesmo cliente | Observar com FDD-OBS-09 e decidir sobre rate limiting (RFC-QA-01) ([09:39] Larissa) |

## 13. Integração com o sistema existente

| ID | Arquivo existente | Como o módulo de webhooks se integra |
| --- | --- | --- |
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)` e antes de reler o pedido, chamar `publishWebhookEvent(tx, order, from, to)` com o mesmo `tx` (`Prisma.TransactionClient`, já tipado como `TxClient` no arquivo). Exceções propagam e o `$transaction` faz rollback ([09:41] Bruno; [09:40] Bruno). `create()` não é alterado. |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Fonte dos status de destino válidos (mapa `transitions`) para validar `events` no cadastro (FDD-ERRO-04). Nenhuma alteração no arquivo. |
| FDD-INT-03 | `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | As novas classes (`WebhookNotFoundError` estendendo `AppError` com 404 e código próprio, `WebhookInvalidUrlError` com 400, `WebhookDeadLetterAlreadyReplayedError` estendendo `ConflictError` etc.) seguem o padrão de `InvalidStatusTransitionError`/`InsufficientStockError`, com prefixo `WEBHOOK_` ([09:28] Bruno). Ficam no módulo e podem ser reexportadas pelo barrel. |
| FDD-INT-04 | `src/shared/errors/app-error.ts` | Base de todas as classes `WEBHOOK_*`: `AppError(message, statusCode, errorCode, details)`. |
| FDD-INT-05 | `src/middlewares/error.middleware.ts` | **Sem alteração**: já serializa qualquer `AppError` como `{ error: { code, message, details } }` ([09:29] Bruno). |
| FDD-INT-06 | `src/middlewares/auth.middleware.ts` | Rotas de gestão usam `authenticate`; o replay usa `requireRole('ADMIN')` ([09:36] Larissa). `req.user.id` alimenta `replayedById` e o log de auditoria. |
| FDD-INT-07 | `src/middlewares/validate.middleware.ts` | Schemas Zod do módulo (params `id` uuid, query `customerId`, body de cadastro e edição) aplicados com `validate({ body, params, query })`, como em `src/modules/orders/order.routes.ts`. |
| FDD-INT-08 | `src/routes/index.ts` e `src/app.ts` | `Controllers` ganha `webhooks`; `buildApiRouter` monta `router.use('/webhooks', …)` e `router.use('/admin/webhooks', …)`; `buildControllers` instancia repository → service → controller do novo módulo `modules/webhooks/`, como faz para orders. |
| FDD-INT-09 | `prisma/schema.prisma` | Novos models (seção 4); `Customer` ganha relação `webhooks Webhook[]` e `User` ganha a relação de `replayedBy`. Migration aditiva em `prisma/migrations/`. |
| FDD-INT-10 | `src/config/database.ts` | O worker cria seu próprio client com `createPrismaClient()` (outro processo, mesma `DATABASE_URL`) ([09:30] Bruno). |
| FDD-INT-11 | `src/server.ts` | Modelo para a nova entry-point `worker.ts` (bootstrap, logs `worker_started`, shutdown com `prisma.$disconnect()`) ([09:11] Larissa). |
| FDD-INT-12 | `src/shared/logger/index.ts` | Logger Pino reutilizado pelo módulo e pelo worker; inclusão de `*.secret` e `*.previousSecret` em `redactPaths` ([09:29] Bruno; [09:22] Diego). |
| FDD-INT-13 | `src/middlewares/request-logger.middleware.ts` | O `req.id` (`X-Request-Id`) é passado ao log de enfileiramento para correlação (FDD-OBS-10). |
| FDD-INT-14 | `package.json` | Novo script `worker` ao lado de `dev`/`start` ([09:11] Larissa). |
| FDD-INT-15 | `tests/orders.test.ts` e `tests/helpers/factories.ts` | Novos casos para outbox e rollback em transição de status, reaproveitando as factories. Testes do módulo num novo arquivo de teste de webhooks, no mesmo padrão vitest + supertest. |
