# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autora** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-10-13 |
| **Revisores** | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança), Marcos (Product Manager) |
| **Origem** | Reunião técnica de webhooks (`TRANSCRICAO.md`) |
| **Documentos relacionados** | ADRs em [`docs/adrs/`](adrs/), FDD em [`docs/FDD.md`](FDD.md), PRD em [`docs/PRD.md`](PRD.md) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que o status de um pedido mudar. O desenho tem quatro pontos:

1. O evento é gravado em uma **tabela outbox no MySQL existente, na mesma transação** do `changeStatus`. Assim, status e evento nunca divergem.
2. Um **worker em processo separado** lê a outbox **por polling a cada 2s** e entrega por HTTPS.
3. Cada entrega vai **assinada com HMAC-SHA256**, usando uma **secret por endpoint** que pode ser rotacionada.
4. A garantia é **at-least-once**, com `X-Event-Id` para o cliente deduplicar. Falhas têm **retry com backoff de cerca de 15h** e depois vão para uma **DLQ** com replay manual por ADMIN.

Não entra infraestrutura nova: tudo reusa os padrões atuais do projeto. A estimativa é de **3 sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser avisados quando o status dos seus pedidos muda. Hoje eles fazem **polling em `GET /orders`**, o que deixa a integração lenta e cara. A Atlas sinalizou que pode **migrar para um concorrente** se a funcionalidade não sair até o fim do trimestre. Para os clientes, "tempo real" significa **menos de 10 segundos**. O fluxo é só de saída.

No código, a mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação Prisma. A transação valida a máquina de estados (`src/modules/orders/order.status.ts`), movimenta estoque quando a transição pede, atualiza o pedido e grava `order_status_history`. **A aplicação não tem nenhum mecanismo de eventos, fila ou notificação externa.** O desafio é publicar eventos de forma confiável sem pendurar uma chamada HTTP para terceiros nessa transação.

## 3. Proposta técnica

### 3.1 Visão geral

A solução tem dois processos sobre o mesmo MySQL:

- **Processo API (`src/server.ts`).** O `OrderService.changeStatus` grava o evento na `webhook_outbox` dentro da própria transação. O novo módulo `webhooks` expõe o CRUD de configuração, o histórico de entregas e o replay da DLQ.
- **Processo Worker (`src/worker.ts`, novo).** A cada 2s lê a outbox e entrega por HTTPS, com assinatura HMAC e timeout de 10s. Quando as tentativas se esgotam, move o evento para a `webhook_dead_letter`. O replay feito por um ADMIN devolve o evento à outbox.

### 3.2 Componentes e responsabilidades

1. **Publicação transacional (outbox).** O `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` com o `tx` da transação em curso. A função faz duas coisas:
   - aplica o **filtro de eventos já na inserção**: se nenhum webhook ativo do cliente assina aquele status, nada é gravado;
   - grava o **payload já renderizado (snapshot)**, para que o evento reflita o pedido no momento da mudança.

   Se a gravação falhar, a mudança de status sofre rollback. → [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
2. **Worker de entrega.** É um processo Node separado, com entry-point próprio e uma instância própria de `PrismaClient` apontando para o mesmo banco. A cada 2s ele lê, em batch pequeno, os pendentes mais antigos e entrega um a um. Com **um único worker**, a ordem é preservada por pedido, mas não há ordem global. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
3. **Resiliência.** O timeout por chamada é de 10s. A falha entra em backoff exponencial (**1m, 5m, 30m, 2h e 12h**). Esgotadas as tentativas, o evento vai para a tabela `webhook_dead_letter`, de onde um ADMIN pode reenviá-lo para a outbox. → [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)
4. **Segurança.** As regras são:
   - assinatura **HMAC-SHA256 do corpo** no header `X-Signature`;
   - **secret única por endpoint, gerada pela plataforma** e devolvida na criação;
   - **rotação** com convivência de 24h entre a secret antiga e a nova;
   - **somente URLs `https`**;
   - payload de no máximo **64KB**; se passar disso, o evento falha em vez de ser truncado.

   → [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
5. **Semântica de entrega.** A garantia é **at-least-once**. Cada evento leva um `X-Event-Id` (UUID gerado na outbox) para o cliente deduplicar, e o comportamento será documentado no portal do desenvolvedor. → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
6. **Encaixe no código.** O código novo segue a mesma estrutura dos outros módulos:
   - novo módulo `src/modules/webhooks/`, a criar (controller, service, repository, routes e schemas);
   - erros como `AppError` com prefixo `WEBHOOK_`;
   - logger Pino e error middleware sem mudanças;
   - `requireRole('ADMIN')` no replay;
   - IDs UUID;
   - `customer_id` no path, porque o JWT identifica o operador e não o cliente.

   → [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

### 3.3 Superfície pública (resumo)

Contratos, payloads, headers e códigos de erro estão detalhados no [FDD](FDD.md).

- **Configuração de webhooks:** criar, listar, editar e remover, com qualquer usuário autenticado. Cada configuração tem URL, lista de status assinados e estado ativo, e a secret é devolvida na criação. Inclui o endpoint de **rotação de secret**.
- **Histórico de entregas por webhook:** as últimas 100 entregas, com sucesso ou falha, payload, resposta e tempo de resposta.
- **Replay administrativo de DLQ:** exige a role ADMIN e registra em log quem executou.
- **Evento `order.status_changed`:** um JSON enxuto com dados básicos do pedido, sem os itens. Os headers são `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

### 3.4 Fora de escopo nesta fase

- E-mail avisando o cliente sobre falhas (fica para a próxima fase).
- Dashboard visual (projeto separado, do time de frontend).
- Webhooks de entrada.
- Arquivamento das linhas entregues da outbox.
- Rate limiting de saída (vai ser observado; ver §5).
- Escalar para vários workers.

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que motivou o descarte | ADR |
| --- | --- | --- | --- |
| ALT-1 | **Disparo HTTP síncrono dentro do `changeStatus`** | Um cliente lento trava a mudança de status dos outros pedidos, e um cliente fora do ar forçaria um rollback indevido da mudança. | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| ALT-2 | **Redis Streams / fila externa** | Mais reatividade e throughput, ao custo de subir e operar infraestrutura nova. Para um time pequeno, é overengineering. | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| ALT-3 | **Trigger no MySQL para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`. A trigger só executa SQL, e notificar um processo externo exigiria improvisos frágeis. O polling de 2s já atende aos 10s. | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| ALT-4 | **Worker dentro do processo da API** | Mais simples de implantar, mas um restart da API derruba as entregas. | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| ALT-5 | **Retry indefinido** / **apenas 3 tentativas** | O retry indefinido deixa eventos pendurados para sempre. Três tentativas se esgotam em cerca de 30 min, e já houve cliente fora do ar por 2h. | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| ALT-6 | **Marcar como `failed` na própria outbox**, sem DLQ separada | Evita uma tabela a mais, mas polui a leitura da outbox e perde o papel de evidência para debug e replay. | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| ALT-7 | **Secret global da plataforma** | É mais simples de gerir, mas o vazamento de uma única secret compromete todos os clientes. | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| ALT-8 | **Exactly-once** | Elimina duplicatas, mas exige coordenação entre os dois lados e uma complexidade desproporcional. | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| ALT-9 | **`customer_id` extraído do JWT** | Seria mais ergonômico, mas o JWT atual é do operador e não do cliente. | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| ALT-10 | **Guardar só o `order_id` e renderizar o payload no envio** | O payload ficaria menor na outbox, mas o evento passaria a refletir o pedido no momento do envio e não no momento da mudança. A decisão foi pelo snapshot na inserção. | — (decisão no FDD) |
| ALT-11 | **Filtrar eventos no envio, e não na inserção** | Seria mais flexível, mas gera linhas inúteis na outbox. A decisão foi filtrar na inserção. | — (decisão no FDD) |

## 5. Questões em aberto

**Levantadas na reunião e não decididas ou adiadas**

| # | Questão | Situação |
| --- | --- | --- |
| QA-1 | **Rate limiting de saída por cliente**: 50 mudanças de status em um minuto viram 50 chamadas? | Observar em produção e decidir depois. |
| QA-2 | **Notificar o cliente (e-mail) quando o webhook falha repetidamente** | Adiado para a próxima fase, após medir o impacto. |
| QA-3 | **Escalar para vários workers**: particionar por `order_id` ou usar lock pessimista | Adiado. Hoje é single-worker e a limitação está documentada. |
| QA-4 | **Endurecer as permissões do CRUD**, hoje aberto a qualquer usuário autenticado | "Por enquanto sim; mais pra frente pode endurecer". Agrava o fato de não existir vínculo User↔Customer em `prisma/schema.prisma`. |
| QA-5 | **Contagem de tentativas**: a decisão diz "5 tentativas", mas lista 5 intervalos (cerca de 14,6h). São 5 envios no total ou 1 envio + 5 retries? | Ambíguo na transcrição. Precisa de confirmação de Diego e Larissa antes da implementação. |
| QA-6 | **Arquivamento** das linhas entregues da outbox (cerca de 30 dias) | Fora do escopo desta feature, sem dono definido. |
| QA-7 | **Nome do arquivo do processor**: `webhook.worker.ts` ou `webhook.processor.ts` | Não fechado. |

**Identificadas na consolidação (não discutidas na reunião, a validar com os revisores)**

| # | Questão | Revisor sugerido |
| --- | --- | --- |
| QA-8 | Com um evento antigo em backoff, o worker pode entregar um evento mais novo do mesmo pedido? Isso quebraria a ordem por `order_id`. | Diego |
| QA-9 | `X-Timestamp` não é coberto pela assinatura, que é feita só sobre o corpo. Ele deve entrar no material assinado? | Sofia |
| QA-10 | Qual secret assina as entregas durante as 24h de rotação: a nova, ou as duas? | Sofia |
| QA-11 | Como a secret é armazenada em repouso? O HMAC exige que ela seja recuperável, então não pode ser só um hash. | Sofia |
| QA-12 | O replay da DLQ mantém o `event_id` original? Se não mantiver, a deduplicação no cliente quebra. | Diego |
| QA-13 | A criação do pedido (null → PENDING), que também grava histórico, gera evento? | Marcos / Bruno |
| QA-14 | Respostas HTTP fora da faixa 2xx contam como falha? Só o timeout foi definido explicitamente. | Diego |

## 6. Impacto e riscos

**Impacto**

- **Código:** muda o `changeStatus`, que passa a fazer mais uma escrita na transação; surge um novo módulo `webhooks`; surgem um novo entry-point e um novo script de worker.
- **Banco:** três novas responsabilidades no mesmo MySQL (configuração, outbox e DLQ), além do histórico de entregas, e um novo pool de conexões vindo do worker.
- **Operação:** mais um processo para fazer deploy e monitorar, e o replay manual da DLQ fica com os ADMINs.
- **Clientes:** precisam validar o HMAC e deduplicar pelo `X-Event-Id`. Isso será documentado no portal do desenvolvedor.
- **Prazo:** 3 sprints, assim distribuídas:
  - outbox e DLQ: 1 sprint;
  - worker e retry: 1 sprint;
  - CRUD e deliveries: ½ sprint;
  - integração e testes ponta a ponta: ½ sprint;
  - revisão de segurança: pelo menos 2 dias úteis no fim.

  A Atlas espera a entrega até o fim de novembro.

**Riscos**

| Risco | Efeito | Mitigação proposta |
| --- | --- | --- |
| Falha na gravação da outbox derruba a mudança de status | A operação de pedidos fica indisponível por causa da feature | Comportamento intencional (consistência acima da disponibilidade). A escrita é simples e testada ponta a ponta. |
| Single-worker como ponto único de falha e teto de vazão | Atraso generalizado das entregas | Processo isolado da API, graceful shutdown nos moldes de `src/server.ts`; escalar está previsto em QA-3. |
| Cliente sem deduplicação | Pedido processado em dobro do lado do cliente | Documentação em destaque no portal; `X-Event-Id` estável. |
| Vazamento de secret, inclusive pelos nossos próprios logs | Requisições forjadas para o cliente | Secret por endpoint, rotação, revisão da Sofia e redact da secret no Pino (`src/shared/logger/index.ts` não cobre `secret` hoje). |
| Prazo apertado diante do risco de churn da Atlas | Perda de um cliente | Escopo enxuto (sem e-mail, dashboard ou rate limiting) e estimativa de 3 sprints validada na reunião. |
| CRUD aberto a qualquer usuário autenticado | Um operador altera webhooks de outro cliente | Aceito por enquanto, e rastreado em QA-4. |

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL, na mesma transação do `changeStatus`](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado, polling de 2 segundos](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e Dead Letter em tabela separada](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com deduplicação via `X-Event-Id`](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
