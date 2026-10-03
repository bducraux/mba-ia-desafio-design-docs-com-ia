# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Escopo confirmado no resumo final da reunião ([09:48] Larissa; [09:49] Diego: "Tá fechado."); design técnico em revisão |
| **Data** | 2026-10-02 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS passa a avisar os clientes B2B automaticamente, por meio de **webhooks**, sempre que o status de um pedido deles muda (por exemplo, quando o pedido é pago, enviado ou entregue). O cliente cadastra um endereço `https` e escolhe quais status quer receber. A plataforma envia uma notificação assinada para esse endereço logo após a mudança e tenta de novo se ele estiver fora do ar.

Hoje não existe nenhum mecanismo de notificação externa no OMS. A demanda veio de um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos).

## 2. Problema e motivação

- Os clientes descobrem mudanças de status consultando `GET /orders` periodicamente, o que deixa a integração **lenta e cara** para eles ([09:00] Marcos).
- Para os clientes, qualquer atraso **abaixo de 10 segundos** já é "tempo real". O que eles não querem é ficar atualizando manualmente ([09:02] Marcos).
- **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos), e pediu a entrega até o fim de novembro ([09:45] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Cenário |
| --- | --- |
| **Time de integração do cliente B2B** (Atlas, MaxDistribuição, Nova Cargo) | Cadastra um endpoint via API, autenticado com o JWT da plataforma por usuários que representam o cliente ([09:32] Marcos), escolhe os status de interesse (ex.: só `SHIPPED` e `DELIVERED`, [09:33] Marcos) e passa a receber notificações em vez de fazer polling. |
| **Time de integração do cliente B2B** | Consulta o histórico das últimas entregas para investigar uma notificação que não chegou ([09:34] Marcos). |
| **Time de integração do cliente B2B** | Troca a secret de assinatura sem parar de receber notificações, por exemplo após um vazamento ([09:21] Sofia; [09:22] Diego). |
| **Administrador do OMS** | Reprocessa manualmente notificações que falharam definitivamente ([09:18] Diego; [09:36] Sofia). |
| **Usuário operador do OMS** | Muda o status do pedido como hoje; a notificação acontece sem afetar o fluxo dele ([09:04] Bruno). |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e o recebimento pelo cliente (endpoint saudável) | **< 10 segundos** ([09:02] Marcos) |
| PRD-OBJ-02 | Atender o prazo comercial do cliente que condicionou a renovação | Feature em produção, disponível para os **3 clientes** solicitantes (Atlas, MaxDistribuição, Nova Cargo, [09:00] Marcos) | Até **o fim de novembro**, prazo pedido pela Atlas ([09:45] Marcos) |
| PRD-OBJ-03 | Não perder notificações por indisponibilidade temporária do cliente | Janela coberta por retentativas automáticas antes de exigir ação manual | **~15 horas** (5 retentativas) ([09:17] Diego) |
| PRD-OBJ-04 | Não degradar o fluxo de pedidos | Mudanças de status que dependem da disponibilidade do cliente | **0** (nenhuma chamada ao cliente dentro da operação de status) ([09:04] Bruno) |

## 5. Escopo

### Incluso

- Notificação de **mudança de status de pedido** para endpoints dos clientes, só de saída (outbound) ([09:02] Marcos).
- Gestão de webhooks via API: cadastrar, listar, editar, ativar/desativar, remover e rotacionar a secret.
- Histórico de entregas por webhook.
- Retentativas automáticas e fila de falhas definitivas com reprocessamento manual por administrador.

### Fora de escopo

| ID | Item | Situação na reunião |
| --- | --- | --- |
| PRD-FORA-01 | **Aviso por email** ao cliente quando o webhook dele está falhando | Adiado: "Email tá fora de escopo dessa fase. Talvez próxima fase" ([09:37] Larissa) |
| PRD-FORA-02 | **Rate limiting** de envio por cliente | Não entra agora; "observar e decidir depois" ([09:39] Larissa) |
| PRD-FORA-03 | **Dashboard visual** para o cliente ver seus webhooks | Descartado nesta fase; projeto separado do time de frontend ([09:40] Larissa) |
| PRD-FORA-04 | **Webhooks de entrada** (cliente enviando para nós) | Descartado: "Só saindo da gente pra eles" ([09:02] Marcos) |
| PRD-FORA-05 | **Garantia de ordem global** entre pedidos diferentes | Não pedida pelos clientes ([09:14] Marcos); só há ordem por pedido ([09:13] Larissa) |
| PRD-FORA-06 | **Arquivamento** de notificações antigas | Fora do escopo desta feature ([09:08] Diego) |

## 6. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| PRD-FR-01 | O cliente cadastra um webhook informando a URL e a lista de status que deseja receber ([09:31] Marcos). |
| PRD-FR-02 | A plataforma gera a secret de assinatura e a devolve ao cliente na criação do webhook ([09:31] Marcos). |
| PRD-FR-03 | O cliente lista os webhooks de um customer, edita e remove um webhook ([09:33] Bruno). |
| PRD-FR-04 | Cada webhook pode ser ativado ou desativado (estado ativo) ([09:21] Bruno). |
| PRD-FR-05 | O cliente escolhe, por webhook, quais mudanças de status quer receber; só status escolhidos geram notificação ([09:33] Marcos). |
| PRD-FR-06 | Toda mudança de status de pedido gera notificação para os webhooks ativos do customer que assinam aquele status ([09:00] Marcos; [09:40] Bruno). |
| PRD-FR-07 | A notificação informa o pedido, o status anterior, o novo status, o cliente e o valor total, sem os itens ([09:43] Diego). |
| PRD-FR-08 | Cada notificação é assinada para que o cliente valide a origem e a integridade ([09:19] Sofia). |
| PRD-FR-09 | O cliente pode pedir uma nova secret; a anterior continua válida por 24 horas ([09:21] Sofia). |
| PRD-FR-10 | O cliente consulta o histórico das últimas 100 entregas de um webhook, com sucesso/falha, conteúdo enviado, resposta e tempo de resposta ([09:34] Marcos). |
| PRD-FR-11 | Notificações não entregues são reenviadas automaticamente e, esgotadas as tentativas, guardadas numa fila de falhas ([09:15] Diego). |
| PRD-FR-12 | Um administrador reprocessa manualmente uma notificação da fila de falhas, e a ação fica registrada com quem a executou ([09:18] Diego; [09:36] Sofia). |
| PRD-FR-13 | Cada notificação carrega um identificador único para o cliente descartar duplicatas ([09:25] Diego). |

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| PRD-NFR-01 | Latência de notificação abaixo de 10 segundos em condições normais ([09:02] Marcos). |
| PRD-NFR-02 | Garantia de entrega **at-least-once**: o cliente pode receber a mesma notificação mais de uma vez ([09:24] Diego). |
| PRD-NFR-03 | Só endereços `https` são aceitos; `http` é recusado com erro de validação ([09:23] Sofia). |
| PRD-NFR-04 | Notificações acima de 64KB não são enviadas e são tratadas como erro ([09:24] Larissa). |
| PRD-NFR-05 | O cliente tem até 10 segundos para responder; depois disso a tentativa conta como falha ([09:42] Diego). |
| PRD-NFR-06 | A secret é única por webhook, nunca compartilhada entre clientes ([09:21] Sofia). |
| PRD-NFR-07 | A gestão de webhooks exige usuário autenticado; o reprocessamento exige perfil de administrador ([09:36] Larissa; [09:37] Sofia). |
| PRD-NFR-08 | Uma mudança de status só é concluída se a notificação correspondente ficar registrada para envio ([09:40] Bruno). |
| PRD-NFR-09 | Notificações de um mesmo pedido chegam na ordem em que os status mudaram ([09:13] Larissa). |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Registro do evento no próprio banco, junto com a mudança de status | Pequeno custo extra na operação de status, em troca de nunca perder eventos e de não ter infraestrutura nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Envio por processo dedicado que verifica pendências a cada 2 segundos | Até 2s de espera e um único processo de envio, em troca de simplicidade | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15h e depois fila de falhas | Notificações podem atrasar horas quando o cliente está fora do ar; reenvio final é manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Assinatura HMAC-SHA256 com secret por webhook e rotação de 24h | Mais estado para guardar e rotacionar, em troca de isolar vazamentos | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com identificador por evento | O cliente precisa descartar duplicatas | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões existentes do OMS | Consistência acima de soluções especializadas | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Conteúdo da notificação "congelado" no momento da mudança | Pode diferir do estado atual do pedido consultado depois | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |

## 9. Dependências

| ID | Dependência |
| --- | --- |
| PRD-DEP-01 | **Revisão de segurança** da Sofia, com pelo menos dois dias úteis antes do deploy, focada em assinatura e geração de secret ([09:46] Sofia). |
| PRD-DEP-02 | **Documentação no portal de desenvolvedor**: como integrar, validar a assinatura e descartar duplicatas ([09:26] Marcos; [09:40] Marcos). |
| PRD-DEP-03 | **Confirmação do prazo com os clientes**, feita pelo Marcos ([09:47] Marcos). |
| PRD-DEP-04 | **Capacidade do time**: estimativa de três sprints com a revisão de segurança incluída ([09:47] Larissa). |

## 10. Riscos e mitigação

Os riscos vêm da reunião; probabilidade e impacto são avaliação da equipe de design com base no contexto discutido.

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RISCO-01 | Atrasar além do fim de novembro e perder a Atlas para um concorrente ([09:00] Marcos; [09:45] Marcos) | Média | Alto | Escopo fechado na reunião, itens não essenciais adiados (email, dashboard, rate limiting) e estimativa de três sprints ([09:47] Larissa) |
| PRD-RISCO-02 | Cliente não trata notificações duplicadas e processa o mesmo evento duas vezes ([09:25] Sofia) | Média | Médio | Identificador único por evento e destaque no portal de desenvolvedor ([09:26] Marcos) |
| PRD-RISCO-03 | Vazamento da secret de um cliente ([09:22] Diego) | Baixa | Alto | Secret por webhook, rotação com 24h de convivência e revisão de segurança antes do deploy ([09:21] Sofia; [09:46] Sofia) |
| PRD-RISCO-04 | Cliente fora do ar por mais de ~15h perde notificações automáticas ([09:17] Marcos) | Baixa | Médio | Fila de falhas com reprocessamento manual por administrador ([09:18] Diego) |
| PRD-RISCO-05 | Rajadas de notificações sobrecarregam o endpoint do cliente ([09:38] Diego) | Média | Médio | Monitorar o volume por cliente após o lançamento e decidir sobre rate limiting ([09:39] Larissa) |

## 11. Critérios de aceitação

| ID | Critério |
| --- | --- |
| PRD-CA-01 | Um cliente com webhook ativo para `SHIPPED` recebe a notificação em menos de 10 segundos quando o pedido dele é enviado, e não recebe nada para status que não assinou. |
| PRD-CA-02 | O cliente consegue cadastrar, listar, editar, desativar, remover e rotacionar a secret de um webhook via API autenticada. |
| PRD-CA-03 | Endereços `http` são recusados no cadastro. |
| PRD-CA-04 | O cliente valida a assinatura de cada notificação com a secret recebida; após uma rotação, as duas secrets funcionam por 24h. |
| PRD-CA-05 | Com o endpoint do cliente fora do ar, as retentativas acontecem automaticamente e, ao fim delas, a notificação aparece na fila de falhas. |
| PRD-CA-06 | Só administradores reprocessam a fila de falhas, e cada reprocessamento registra quem o fez. |
| PRD-CA-07 | O histórico mostra as últimas 100 entregas com resultado, conteúdo, resposta e tempo. |
| PRD-CA-08 | Mudar o status de um pedido continua funcionando normalmente mesmo com endpoints de clientes lentos ou fora do ar. |

## 12. Estratégia de testes e validação

- **Testes automatizados** no padrão do projeto (vitest + supertest, como `tests/orders.test.ts`): gestão de webhooks, filtro por status, registro da notificação junto com a mudança de status (inclusive o cenário de falha que reverte o status), assinatura, retentativas, fila de falhas e permissão de administrador. Os critérios técnicos estão no [FDD](FDD.md#11-critérios-de-aceite-técnicos).
- **Teste ponta a ponta** da integração com o fluxo de pedidos, previsto na estimativa ([09:46] Larissa).
- **Revisão de segurança** do código de assinatura e de geração de secret antes do deploy ([09:46] Sofia).
- **Validação com clientes**: piloto com a Atlas, MaxDistribuição e Nova Cargo usando o portal de desenvolvedor, medindo o tempo de entrega (PRD-OBJ-01) e o volume na fila de falhas.
- **Pós-lançamento**: acompanhar volume por cliente (insumo para rate limiting, PRD-FORA-02) e taxa de falhas (insumo para o aviso por email, PRD-FORA-01) ([09:37] Larissa; [09:39] Diego).
