# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

Pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos** do OMS, produzido com IA a partir da transcrição da reunião técnica ([`TRANSCRICAO.md`](TRANSCRICAO.md)) e do código existente. O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, MySQL via Prisma) decidiu, numa reunião de cerca de 55 minutos entre tech lead, PM, dois engenheiros e a engenheira de segurança, construir webhooks para avisar clientes B2B quando o status dos pedidos muda. Nada foi registrado além da transcrição. A tarefa foi transformar essa conversa, cruzada com o código real, em documentação acionável: PRD, RFC, FDD, ADRs e um tracker que liga cada item à sua origem.

A dificuldade não está em escrever, e sim em **filtrar**. A reunião mistura decisões fechadas, propostas rejeitadas, itens adiados, correções feitas um minuto depois ("customer_id implícito do JWT" → "não vem do JWT") e detalhes que os próprios participantes disseram que não eram decisões arquiteturais. Nada pode ser inventado, e o que foi descartado não pode reaparecer como requisito.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (modelo Claude Opus) | Ferramenta principal: leu o código do repositório e a transcrição, gerou o inventário, redigiu os documentos, montou o tracker e rodou as checagens. |
| **Scripts Python gerados com a IA** (fora do repositório) | Validação mecânica: (1) conferência de todo par `[hh:mm] Nome` citado nos documentos contra as falas reais da transcrição; (2) verificador de critérios de aceite (estrutura, seções, contagens, links de ADR, caminhos de código existentes, tracker e arquivos intocados). |

Meu papel foi de maestro: definir a ordem de produção e as regras de rastreabilidade, revisar criticamente cada entrega, apontar erros de interpretação e decidir onde a reunião deixou lacunas.

## Workflow adotado

1. **Critérios de saída antes de produzir.** Antes de gerar qualquer documento, li o enunciado e o verificador para saber exatamente o que o resultado final precisa ter. Isso revelou armadilhas que reprovariam a entrega mesmo com um bom conteúdo (ver "Iterações e ajustes").
2. **Contextualização (notas fora do repositório):**
   - **Mapa do código**, com caminhos reais (`git ls-files`): transação do `changeStatus`, máquina de estados, classes de erro, middlewares, logger, schema Prisma, wiring da API.
   - **Inventário da transcrição** em tabelas: decisões (25), requisitos funcionais, não funcionais, descartados (14, com motivo), adiados, em aberto e armadilhas de interpretação. Cada linha tem timestamp e falante.
3. **ADRs primeiro** (7): um por decisão principal, mais o snapshot do payload, que foi decidido explicitamente no fim da call.
4. **RFC** em cima dos ADRs: proposta em nível de arquitetura, alternativas descartadas com o trade-off e questões em aberto.
5. **FDD**: modelo de dados, fluxos, contratos com exemplos, matriz de erros, resiliência, observabilidade e integração arquivo a arquivo.
6. **PRD** por último entre os grandes documentos, como consolidação de produto, sem repetir contratos e fluxos.
7. **Tracker** montado a partir de todos os IDs dos documentos (157 IDs, 100% cobertos), mais as decisões complementares.
8. **Auditoria cruzada e verificação**: busca de itens descartados ou adiados escapando como requisito, de caminhos de código inexistentes e de timestamps inválidos. Por fim, a passagem pelo verificador de critérios.

Convenções que apliquei em todos os documentos:
- **IDs dentro dos documentos** (`PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`…), para o rastreio ser verificável.
- **Timestamp e falante ao lado de cada afirmação**.
- O rótulo **"(proposta de implementação)"** em tudo que o FDD precisou definir e a reunião não decidiu.

## Prompts customizados

**1. Inventário dirigido da transcrição** (adaptado do material do curso; o foco é separar o que entra do que não entra):

```text
Leia TRANSCRICAO.md inteira. Monte uma tabela com colunas: Tipo (Decisão | Requisito funcional |
Requisito não funcional | Restrição | Descartado | Adiado | Em aberto | Detalhe técnico),
Resumo em uma linha, Timestamp, Falante, Trecho literal curto.
Regras: só registre o que está explícito na fala; se algo foi proposto e depois rejeitado,
registre como Descartado com o timestamp da rejeição; não infira requisitos; não use
conhecimento externo sobre webhooks. Ao final, liste à parte as falas que se contradizem ou
são corrigidas depois na própria reunião.
```

**2. Geração dos ADRs com rastreabilidade obrigatória:**

```text
Usando o inventário (notas/02-inventario-transcricao.md) e o mapa do código
(notas/01-mapa-codigo.md), escreva um ADR por decisão principal em
docs/adrs/ADR-NNN-titulo-em-kebab-case.md, no formato MADR com as seções Status, Contexto,
Decisão, Alternativas Consideradas e Consequências (positivas, negativas e uma linha
"Trade-off explícito").
Regras:
- toda afirmação de contexto ou decisão leva [hh:mm] Nome da fala de origem;
- alternativas: use primeiro as rejeitadas na reunião; se usar uma alternativa plausível
  não discutida, diga que é plausível;
- HTTPS e limite de 64KB NÃO viram ADR (a reunião diz que não são decisões arquiteturais);
- cite só caminhos de arquivo que existem; arquivos futuros sem o prefixo src/
  (ex.: "novo módulo modules/webhooks/", "nova entry-point worker.ts").
```

**3. Auditoria cruzada** (usado na revisão final):

```text
Para cada requisito, decisão e restrição em docs/PRD.md, docs/RFC.md, docs/FDD.md e
docs/adrs/, procure a origem em TRANSCRICAO.md ou no código. Liste: (1) itens sem origem;
(2) itens que contradizem a transcrição; (3) itens descartados/adiados na reunião que aparecem
como requisito; (4) caminhos de arquivo citados que não existem. Não corrija nada, só liste
com evidência.
```

## Iterações e ajustes

Foram **5 ciclos principais** de geração, revisão e ajuste: inventário → ADRs → RFC e FDD → PRD → tracker com auditoria final. Os ajustes mais importantes:

1. **Requisito que a própria reunião corrigiu.** Em [09:31] Marcos diz "Customer_id implícito do JWT"; em [09:32] Larissa corrige: "Não vem do JWT". A primeira leitura do inventário registrou a fala do Marcos como requisito. Reclassifiquei como descartado e transformei a forma (body ou path) em questão em aberto (RFC-QA-02). O FDD assume uma proposta marcada como tal.
2. **O que não é decisão arquitetural.** A geração inicial tendia a criar ADRs para "URL https" e "limite de 64KB". A reunião diz explicitamente o contrário ([09:23] Sofia: "nem é decisão arquitetural"; [09:24] Larissa: "é só requisito não funcional"). Esses itens ficaram como requisitos no PRD e no FDD.
3. **Ambiguidade numérica no retry.** "5 tentativas, backoff 1m/5m/30m/2h/12h" pode ser lido como 5 envios ou como 5 retentativas. Fechei como **1 envio + 5 retentativas**, porque só essa leitura bate com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego): 1m+5m+30m+2h+12h = 14h36m. A tabela com o tempo acumulado está no FDD.
4. **Arquivos que ainda não existem.** A reunião cita `worker.ts` e o módulo de webhooks com caminhos completos, mas esses arquivos não existem no repositório. Citá-los como se existissem quebraria o critério "nenhum arquivo mencionado é inexistente". Passei a escrevê-los como "nova entry-point `worker.ts`" e "novo módulo `modules/webhooks/`". Também removi o `docs/adrs/README.md` do boilerplate, que ficava fora do padrão `ADR-NNN-*.md` exigido para a pasta.
5. **Contradição interna no FDD.** O contrato de `DELETE` dizia que o worker encerraria eventos pendentes com `WEBHOOK_NOT_FOUND`, mas o mesmo texto propunha `onDelete: Cascade`, que apagaria essas linhas antes. Corrigi a semântica e deixei a desativação (`PATCH active=false`) como forma de pausar sem perder histórico.
6. **Tracing sem base na reunião.** O FDD exige tracing, mas a reunião não fala disso e decide não adicionar nada novo ([09:29] Bruno). Em vez de inventar uma stack de tracing, documentei a correlação com o que já existe: o `X-Request-Id` do `request-logger.middleware`, propagado como `requestId` → `eventId` → `X-Event-Id`.

Validação mecânica ao final:
- 100% dos pares `[hh:mm] Nome` citados nos documentos existem na transcrição (545 referências);
- 0 caminhos de código inexistentes;
- tracker com 167 linhas, 89% com origem na transcrição e 100% dos IDs cobertos;
- `src/`, `prisma/`, `tests/` e `TRANSCRICAO.md` sem alteração.

## Como navegar a entrega

Ordem sugerida de leitura, do "por quê" ao "como":

| # | Documento | O que responde |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, público, escopo, metas e riscos de produto |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Como pretendemos resolver: proposta, alternativas descartadas e questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | Por que decidimos assim, decisão por decisão |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, observabilidade e integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item: fala da reunião ou arquivo do código |

ADRs:

- [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Payload como snapshot na inserção](docs/adrs/ADR-007-payload-snapshot-na-insercao.md)

O código da aplicação (`src/`, `prisma/`, `tests/`) e a transcrição não foram alterados; a entrega é puramente documental.
