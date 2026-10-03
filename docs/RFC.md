# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) — consolidação da reunião técnica de webhooks ([09:50] Larissa: "Eu vou abrir o doc de design da feature") |
| **Status** | Em revisão |
| **Data** | 2026-10-02 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenharia de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## Resumo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que o status de um pedido muda. O evento é gravado numa **tabela outbox no MySQL, na mesma transação da mudança de status**, e entregue por um **worker em processo separado que faz polling a cada 2 segundos**. As entregas são **assinadas com HMAC-SHA256** (uma secret por endpoint, rotacionável), têm garantia **at-least-once** com `X-Event-Id` para deduplicação, **5 tentativas com backoff exponencial** e, depois disso, vão para uma **DLQ** com replay manual por admin. Tudo é construído com os padrões que o OMS já usa, sem infraestrutura nova. A estimativa é de três sprints, incluindo revisão de segurança.

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos seus pedidos muda. Hoje eles consultam `GET /orders` periodicamente, o que torna a integração lenta e cara, e a Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos). Para eles, "tempo real" é qualquer coisa **abaixo de 10 segundos** ([09:02] Marcos). O fluxo é só de saída: os clientes recebem, não enviam ([09:02] Marcos).

O OMS não tem nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status acontece numa transação que já é pesada, porque atualiza o pedido, grava o histórico e mexe no estoque ([09:04] Bruno). Qualquer solução precisa:

- não colocar chamadas HTTP de clientes dentro dessa transação;
- nunca deixar um status mudar sem o evento correspondente ser registrado ([09:40] Bruno);
- não exigir infraestrutura nova, já que o time é pequeno ([09:07] Diego).

## Proposta técnica

**Visão geral do fluxo:**

```
PATCH /orders/:id/status
        │  (transação MySQL)
        ├─ atualiza orders + order_status_history + estoque   (como hoje)
        └─ grava evento em webhook_outbox (snapshot do payload, filtrado por status)
                         │
          worker separado (polling 2s, single-worker)
                         │
          POST assinado (HMAC-SHA256) → endpoint https do cliente
                         │
        sucesso → entregue │ falha → retry 1m/5m/30m/2h/12h → DLQ → replay manual (ADMIN)
```

**Componentes:**

1. **Outbox transacional.** A mudança de status chama uma função de publicação que recebe o client da transação corrente e grava o evento na outbox. Se a gravação falhar, a mudança de status é revertida. O evento é gravado só se algum webhook ativo do customer assina aquele status, e o payload é gravado já renderizado (snapshot). Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md).
2. **Worker de entrega.** É uma nova entry-point executada como processo próprio, que lê a outbox a cada 2 segundos em lotes pequenos. Usa o mesmo banco com um `PrismaClient` próprio. Rodando como instância única, preserva a ordem dos eventos de cada pedido. Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).
3. **Entrega confiável.** O timeout é de 10 segundos por chamada e o retry segue backoff exponencial (5 tentativas, cerca de 15h no total). Depois disso o evento vai para uma tabela de dead letter, de onde um admin pode recolocá-lo na fila. Ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).
4. **Segurança.** O corpo é assinado com HMAC-SHA256 usando a secret do endpoint. A secret é gerada pela plataforma e pode ser rotacionada, com a antiga valendo por mais 24h. Só URLs `https` são aceitas e o payload tem limite de 64KB. Ver [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md).
5. **Semântica de entrega.** A garantia é at-least-once, com `X-Event-Id` estável para deduplicação no cliente. Ver [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).
6. **API de gestão.** O CRUD de webhooks por customer, a rotação de secret e o histórico de entregas ficam disponíveis para usuários autenticados. O replay da DLQ é restrito a `ADMIN`. O módulo segue os padrões existentes de estrutura, erros (`WEBHOOK_*`), logs e validação. Ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).

Contratos, modelo de dados, payload, headers, matriz de erros e passo a passo de implementação estão no [FDD](FDD.md).

## Alternativas consideradas

| ID | Alternativa | Trade-off que motivou o descarte |
| --- | --- | --- |
| RFC-ALT-01 | **Disparo HTTP síncrono dentro do `changeStatus`** | Mais simples e com menor latência, mas um cliente lento travaria mudanças de status de outros pedidos, e um cliente fora do ar forçaria rollback do status ([09:04] Bruno; [09:06] Diego). |
| RFC-ALT-02 | **Redis Streams / fila dedicada** | Daria entrega mais reativa e escalável, mas exigiria subir e operar infraestrutura nova; "overengineering" para um time pequeno ([09:07] Larissa; [09:07] Diego). |
| RFC-ALT-03 | **Trigger no MySQL para acordar o worker** | Reduziria a latência do polling, mas o MySQL não notifica processos externos e a solução exigiria improvisos frágeis ([09:09] Diego). |
| RFC-ALT-04 | **Garantia exactly-once** | Evitaria duplicatas no cliente, mas exige coordenação dos dois lados e complexidade desproporcional ([09:25] Diego). |
| RFC-ALT-05 | **Retry indefinido, ou só 3 tentativas** | Indefinido deixa eventos pendurados para sempre ([09:15] Diego); 3 tentativas (~30 min) descartaria eventos de clientes em manutenção de horas ([09:16] Diego). |
| RFC-ALT-06 | **Secret global da plataforma** | Mais simples de gerir, mas um vazamento comprometeria todos os clientes ([09:21] Sofia). |

## Questões em aberto

| ID | Questão | Situação na reunião |
| --- | --- | --- |
| RFC-QA-01 | **Rate limiting de envio por cliente**: um cliente com 50 pedidos mudando de status num minuto recebe 50 chamadas? | Fora do escopo agora; "observar e decidir depois" ([09:38] Diego; [09:39] Larissa). |
| RFC-QA-02 | **`customer_id` no body ou no path** dos endpoints de gestão | Definido apenas que **não** vem do JWT ([09:32] Larissa); a forma ficou em aberto. O FDD propõe uma opção para revisão. |
| RFC-QA-03 | **Escala para múltiplos workers** | Exigirá particionar por `order_id` ou usar lock pessimista; "problema do futuro" ([09:13] Diego). |
| RFC-QA-04 | **Arquivamento de eventos entregues** | Mencionado "depois de 30 dias ou assim", explicitamente fora do escopo desta feature ([09:08] Diego). |
| RFC-QA-05 | **Endurecer permissões do CRUD** de webhooks (hoje: qualquer usuário autenticado) | "Mais pra frente a gente pode endurecer" ([09:37] Sofia). |
| RFC-QA-06 | **Alerta proativo por email** quando o webhook do cliente falha | Adiado para uma próxima fase, "depois que a gente medir o impacto" ([09:37] Larissa). |

## Impacto e riscos

**Impacto no sistema existente**

- **Pedidos:** a transação de `changeStatus` ganha a gravação do evento na outbox; uma falha nessa gravação passa a reverter a mudança de status ([09:40] Bruno). Os testes de pedidos precisam cobrir esse comportamento.
- **Banco:** novas tabelas (configuração de webhooks, outbox, histórico de entregas, dead letter) e carga de polling a cada 2s.
- **Operação:** um novo processo para implantar e monitorar (`npm run worker`) ([09:11] Larissa).
- **Clientes:** precisam validar a assinatura HMAC e deduplicar por `X-Event-Id`; isso será documentado no portal de desenvolvedor ([09:26] Marcos; [09:40] Marcos).

**Riscos**

| ID | Risco | Mitigação proposta |
| --- | --- | --- |
| RFC-RISCO-01 | Prazo: Atlas espera a entrega até o fim de novembro ([09:45] Marcos) | Escopo fechado na reunião; estimativa de três sprints com a revisão de segurança incluída ([09:47] Larissa). |
| RFC-RISCO-02 | Ordenação perdida se o worker for escalado | Manter single-worker; documentar como limitação conhecida ([09:13] Larissa). |
| RFC-RISCO-03 | Vazamento de secret de cliente | Secret por endpoint, rotação com 24h de convivência e revisão de segurança de dois dias úteis antes do deploy ([09:21] Sofia; [09:46] Sofia). |
| RFC-RISCO-04 | Rajadas de eventos sobrecarregando o endpoint do cliente | Observar após o lançamento e decidir sobre rate limiting (RFC-QA-01). |

## Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL, na mesma transação da mudança de status |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 segundos, single-worker |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com 24h de convivência |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) | Payload renderizado como snapshot na inserção da outbox |
