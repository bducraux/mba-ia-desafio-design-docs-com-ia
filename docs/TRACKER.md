# Tracker de Rastreabilidade

Cada item identificável dos documentos (PRD, RFC, FDD e ADRs) com a sua origem: a fala da reunião (`TRANSCRICAO`, timestamp + falante de [`TRANSCRICAO.md`](../TRANSCRICAO.md)) ou o código existente (`CODIGO`, caminho do arquivo). Quando um item tem mais de uma fala de origem, a Localização lista a fala principal primeiro.

Itens marcados nos documentos como **(proposta de implementação)** não foram decididos na reunião: a linha aponta para a fala que originou a necessidade e o resumo deixa isso explícito.

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, inserção na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa; [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2s, single-worker | TRANSCRICAO | [09:10] Larissa; [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com deduplicação por X-Event-Id | TRANSCRICAO | [09:26] Larissa; [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso de módulo, AppError, Pino, error middleware, Zod e códigos de erro | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Bruno; [09:52] Larissa |

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificação em menos de 10 segundos ("tempo real" para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Feature em produção para os 3 clientes solicitantes até o fim de novembro (prazo da Atlas) | TRANSCRICAO | [09:45] Marcos; [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Janela de ~15h coberta por retentativas automáticas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Nenhuma mudança de status dependente da disponibilidade do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-FORA-01 | docs/PRD.md | Fora de Escopo | Aviso por email quando o webhook falha (adiado para próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | docs/PRD.md | Fora de Escopo | Rate limiting de envio ("observar e decidir depois") | TRANSCRICAO | [09:39] Larissa |
| PRD-FORA-03 | docs/PRD.md | Fora de Escopo | Dashboard visual (projeto separado do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-04 | docs/PRD.md | Fora de Escopo | Webhooks inbound (só outbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-05 | docs/PRD.md | Fora de Escopo | Garantia de ordem global entre pedidos | TRANSCRICAO | [09:14] Marcos; [09:13] Larissa |
| PRD-FORA-06 | docs/PRD.md | Fora de Escopo | Arquivamento de eventos antigos | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar por customer, editar e remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Ativar/desativar webhook (estado ativo) | TRANSCRICAO | [09:21] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status, por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Mudança de status gera notificação para webhooks ativos que assinam o status | TRANSCRICAO | [09:40] Bruno; [09:00] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação: pedido, status de/para, cliente, total, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Notificação assinada para validar origem e integridade | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reenvio automático e fila de falhas após esgotar tentativas | TRANSCRICAO | [09:15] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Reprocessamento manual por admin, registrando quem executou | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Identificador único por notificação para descartar duplicatas | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Restrição | Só URLs https; http recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Restrição | Payload acima de 64KB não é enviado (erro) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Restrição | Secret única por webhook, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | docs/PRD.md | Restrição | Gestão para autenticados; replay só para ADMIN | TRANSCRICAO | [09:36] Larissa; [09:37] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Status só muda se a notificação ficar registrada | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordem das notificações preservada por pedido | TRANSCRICAO | [09:13] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação do prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Capacidade estimada de três sprints | TRANSCRICAO | [09:47] Larissa |
| PRD-RISCO-01 | docs/PRD.md | Risco | Atraso além de novembro e churn da Atlas | TRANSCRICAO | [09:00] Marcos; [09:45] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não deduplica eventos repetidos | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Cliente fora do ar por mais de ~15h | TRANSCRICAO | [09:17] Marcos |
| PRD-RISCO-05 | docs/PRD.md | Risco | Rajadas de eventos sobrecarregando o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Notificação de status assinado em < 10s; nada para status não assinado | TRANSCRICAO | [09:02] Marcos; [09:33] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | CRUD completo e rotação de secret via API autenticada | TRANSCRICAO | [09:33] Bruno; [09:21] Sofia |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Assinatura válida; duas secrets válidas por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | Retentativas automáticas e fila de falhas ao final | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Só admin reprocessa; registro de quem fez | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Histórico com as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Mudança de status não afetada por clientes lentos ou fora do ar | TRANSCRICAO | [09:04] Bruno |

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / fila dedicada (overengineering) | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger do MySQL para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:15] Diego; [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente | TRANSCRICAO | [09:39] Larissa; [09:38] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | customer_id no body ou no path (não do JWT) | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em Aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-06 | docs/RFC.md | Questão em Aberto | Alerta por email após medir impacto | TRANSCRICAO | [09:37] Larissa |
| RFC-RISCO-01 | docs/RFC.md | Risco | Prazo de fim de novembro | TRANSCRICAO | [09:45] Marcos; [09:47] Larissa |
| RFC-RISCO-02 | docs/RFC.md | Risco | Perda de ordenação ao escalar o worker | TRANSCRICAO | [09:13] Larissa |
| RFC-RISCO-03 | docs/RFC.md | Risco | Vazamento de secret | TRANSCRICAO | [09:21] Sofia; [09:46] Sofia |
| RFC-RISCO-04 | docs/RFC.md | Risco | Rajadas de eventos para o cliente | TRANSCRICAO | [09:38] Diego |

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Evento na outbox na mesma transação; falha reverte o status | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Entrega em < 10s (polling 2s + timeout 10s) | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Não perder eventos: 5 retentativas + DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | Validação de origem/integridade via HMAC-SHA256 | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Restrição | Sem infraestrutura nem bibliotecas novas | TRANSCRICAO | [09:29] Bruno; [09:07] Diego |
| FDD-DADOS-01 | docs/FDD.md | Modelo de Dados | Tabela webhooks: url, secret, customer_id, events, active (+ secret anterior para rotação) | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| FDD-DADOS-02 | docs/FDD.md | Modelo de Dados | Tabela webhook_outbox com status pendente/processando/falhou/entregue, índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de Dados | Tabela webhook_deliveries para o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Modelo de Dados | Tabela webhook_dead_letter com payload, motivo, timestamp e quem fez replay | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, …) no changeStatus, filtro na inserção, rollback em falha | TRANSCRICAO | [09:41] Bruno; [09:34] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker em polling de 2s, lote pequeno, ordem por created_at, POST assinado com timeout | TRANSCRICAO | [09:09] Diego; [09:12] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry 1m/5m/30m/2h/12h (1 envio + 5 retentativas, ~14h36m) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Movimento para DLQ após esgotar tentativas ou em erro não recuperável | TRANSCRICAO | [09:18] Diego; [09:23] Sofia |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Replay manual: recoloca na outbox como pendente, mesmo event_id | TRANSCRICAO | [09:18] Diego; [09:36] Larissa |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Rotação de secret com 24h de convivência (header da secret anterior: proposta de implementação) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks (cadastro, secret devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId= (listagem por customer) | TRANSCRICAO | [09:33] Bruno; [09:32] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id (url, eventos, ativo) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Rotação de secret (endpoint pedido na reunião; caminho é proposta de implementação) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:35] Diego; [09:36] Larissa |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Entrega outbound: payload JSON e headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (malformada ou não https) | TRANSCRICAO | [09:28] Bruno; [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (worker sem secret válida) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENTS (lista de status inválida) | TRANSCRICAO | [09:33] Marcos |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (customer_id informado não existe) | TRANSCRICAO | [09:32] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (> 64KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-07 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED (não-2xx ou rede) | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-09 | docs/FDD.md | Erro | WEBHOOK_MAX_RETRIES_EXCEEDED (move para DLQ) | TRANSCRICAO | [09:17] Larissa |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND no replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-11 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (evita reenfileirar duas vezes) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-12 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (webhook desativado) | TRANSCRICAO | [09:21] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s por chamada | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | 5 retentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Backoff com intervalos persistidos em nextAttemptAt | TRANSCRICAO | [09:17] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Fallback DLQ + replay manual (email adiado) | TRANSCRICAO | [09:18] Diego; [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Worker isolado com PrismaClient próprio | TRANSCRICAO | [09:30] Bruno; [09:11] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Recuperação de linhas PROCESSING após crash (reenvio coberto por at-least-once; proposta de implementação) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Graceful shutdown no padrão do server existente | CODIGO | src/server.ts |
| FDD-RES-08 | docs/FDD.md | Resiliência | Limite de payload de 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Log webhook_event_enqueued com requestId e eventId | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs de sucesso/falha de entrega com tentativa e duração | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log webhook_dead_lettered | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log de auditoria do replay com o admin | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Métrica de latência de entrega vs meta de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Taxa de sucesso por webhook/customer (insumo para email futuro) | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Backlog de pendentes vencidos | TRANSCRICAO | [09:08] Diego |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | Volume na DLQ e replays | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | Eventos por customer por minuto (insumo para rate limiting) | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-10 | docs/FDD.md | Observabilidade | Tracing por correlação requestId → eventId → X-Event-Id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | Linha PENDING por webhook elegível; nada para status não assinado | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Falha na outbox mantém status, histórico e estoque inalterados | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Entrega em < 10s após o commit | TRANSCRICAO | [09:02] Marcos |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Corpo valida contra X-Signature (HMAC-SHA256) | TRANSCRICAO | [09:22] Sofia |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | Duas secrets válidas por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | Retentativas +1m/+5m/+30m/+2h/+12h e DLQ ao final | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | Replay por ADMIN com mesmo X-Event-Id; OPERATOR recebe 403 | TRANSCRICAO | [09:36] Larissa |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | URL http retorna 400 WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | Payload > 64KB vai para DLQ com WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | Ordem por pedido com worker único | TRANSCRICAO | [09:12] Diego |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | Histórico limitado às 100 últimas entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-RISCO-01 | docs/FDD.md | Risco | Transação de changeStatus mais longa | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-02 | docs/FDD.md | Risco | Crescimento das tabelas (arquivamento futuro) | TRANSCRICAO | [09:08] Diego |
| FDD-RISCO-03 | docs/FDD.md | Risco | Reenvio duplicado após crash | TRANSCRICAO | [09:24] Diego |
| FDD-RISCO-04 | docs/FDD.md | Risco | Secret exposta em logs/respostas | TRANSCRICAO | [09:22] Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | Worker parado sem detecção | TRANSCRICAO | [09:11] Diego |
| FDD-RISCO-06 | docs/FDD.md | Risco | Rajadas de eventos para um cliente | TRANSCRICAO | [09:39] Larissa |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent(tx, …) após gravar o histórico | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Mapa de transições como fonte dos status válidos para o filtro | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes WEBHOOK_* no padrão das classes de erro existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | AppError como base das classes do módulo | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Error middleware reaproveitado sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | authenticate nas rotas; requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Validação Zod via validate() | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Registro do router e do controller no wiring da API | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novos models e relações no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INT-10 | docs/FDD.md | Integração | Worker cria o próprio client com createPrismaClient() | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Nova entry-point worker.ts espelhando o bootstrap/shutdown | CODIGO | src/server.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Logger Pino reutilizado com redaction de secrets | CODIGO | src/shared/logger/index.ts |
| FDD-INT-13 | docs/FDD.md | Integração | req.id (X-Request-Id) para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Novo script worker | CODIGO | package.json |
| FDD-INT-15 | docs/FDD.md | Integração | Testes de outbox/rollback reaproveitando as factories | CODIGO | tests/orders.test.ts |

## Decisões complementares registradas na reunião (sem ID próprio nos documentos)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| TRK-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry-point separada e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| TRK-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo de webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| TRK-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| TRK-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Identificadores UUID na outbox | TRANSCRICAO | [09:51] Larissa |
| TRK-05 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Atraso de até ~15h aceito no pior caso | TRANSCRICAO | [09:17] Marcos |
| TRK-06 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| TRK-07 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Ordenação só por order_id enquanto houver um único worker | TRANSCRICAO | [09:13] Larissa |
| TRK-08 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Até 2s de espera do polling aceitos | TRANSCRICAO | [09:10] Larissa |
| TRK-09 | docs/FDD.md | Restrição | customer_id nunca vem do JWT | TRANSCRICAO | [09:32] Larissa |
| TRK-10 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Classes de erro existentes como modelo (InvalidStatusTransitionError, InsufficientStockError) | CODIGO | src/shared/errors/http-errors.ts |
