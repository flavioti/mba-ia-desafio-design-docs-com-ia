# Tracker de Rastreabilidade

Este tracker liga cada item registrado nos design docs à sua origem: a reunião (`TRANSCRICAO.md`) ou o código-fonte do repositório.

**Convenções**

- **ID:** `<DOC>-<SEÇÃO>-<NN>`. Nos ADRs, as seções são `CTX` (contexto), `D` (decisão), `ALT` (alternativa), `CON` (consequência) e `DIV` (divergência ou ponto em aberto). O número acompanha os rótulos `D1`, `A1`, `C+1`/`C−1` usados dentro de cada ADR.
- **Fonte:** `TRANSCRICAO`, com localização no formato `[hh:mm] Nome`, ou `CODIGO`, com o caminho do arquivo.
- **Uma linha por origem.** Quando um item tem mais de uma origem relevante, ele aparece em linhas separadas, com sufixo `a`/`b`.

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-CTX-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação; hoje fazem polling em GET /orders | TRANSCRICAO | `[09:00] Marcos` |
| ADR-001-CTX-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Requisito Não Funcional | "Tempo real" para o cliente = abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| ADR-001-CTX-03 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Webhooks apenas outbound (plataforma → cliente) | TRANSCRICAO | `[09:02] Marcos` |
| ADR-001-CTX-04 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | `changeStatus` roda em `prisma.$transaction`: valida a transição, mexe no estoque, atualiza a order e grava o history | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-CTX-05 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | Máquina de estados (`canTransition`, `shouldDebitStock`, `shouldReplenishStock`) | CODIGO | `src/modules/orders/order.status.ts` |
| ADR-001-CTX-06 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | Nenhuma infraestrutura de mensageria, fila ou evento nas dependências | CODIGO | `package.json` |
| ADR-001-D1 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Padrão outbox: insert em `webhook_outbox` na mesma transação de orders/history | TRANSCRICAO | `[09:06] Diego` |
| ADR-001-D1a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Fechamento da decisão "outbox em MySQL" | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001-D2 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Falha ao inserir na outbox → rollback da mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| ADR-001-D2a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Outbox fora da transação perderia a garantia | TRANSCRICAO | `[09:41] Diego` |
| ADR-001-D3 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` | TRANSCRICAO | `[09:41] Bruno` |
| ADR-001-D3a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Função pura recebendo o tx, sem injetar repository | TRANSCRICAO | `[09:41] Diego` |
| ADR-001-D3b | `docs/adrs/ADR-001-outbox-no-mysql.md` | Integração | Tipo `TxClient = Prisma.TransactionClient` já existente como contrato do parâmetro | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-D4 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Índices em `status` (pendente/processando/falhou/entregue) e `created_at` | TRANSCRICAO | `[09:08] Diego` |
| ADR-001-D5 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | PK UUID na outbox, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| ADR-001-D5a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Padrão | Models usam `@id @default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| ADR-001-D6 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Fora de escopo | Arquivamento das linhas entregues (~30 dias) | TRANSCRICAO | `[09:08] Diego` |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | HTTP síncrono no changeStatus: cliente lento trava as mudanças de status; rollback inviável | TRANSCRICAO | `[09:04] Bruno` |
| ADR-001-ALT-01a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | "Síncrono está fora de questão" | TRANSCRICAO | `[09:06] Diego` |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Streams: exige subir mais infraestrutura | TRANSCRICAO | `[09:07] Larissa` |
| ADR-001-ALT-02a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Redis Cluster é overengineering para time pequeno | TRANSCRICAO | `[09:07] Diego` |
| ADR-001-CON-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Consequência | Commit ⇒ evento registrado; rollback ⇒ evento some | TRANSCRICAO | `[09:06] Diego` |
| ADR-001-CON-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Risco | Crescimento da outbox / performance do worker | TRANSCRICAO | `[09:07] Bruno` |
| ADR-001-DIV-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Divergência | Bruno diz que a transação sempre decrementa estoque; o código só debita em PENDING→PAID | TRANSCRICAO | `[09:04] Bruno` |
| ADR-001-DIV-01a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Divergência | Débito condicional (`shouldDebitStock`) e reposição no cancelamento | CODIGO | `src/modules/orders/order.status.ts` |
| ADR-001-DIV-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Questão em aberto | `OrderService.create` grava histórico null→PENDING; não decidido se gera evento | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002-CTX-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Contexto | Entry-point atual com bootstrap e shutdown via `SIGINT`/`SIGTERM` | CODIGO | `src/server.ts` |
| ADR-002-CTX-02 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Contexto | Scripts npm existentes (sem script de worker) | CODIGO | `package.json` |
| ADR-002-D1 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Polling a cada 2s, eventos pendentes mais antigos | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-D1a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Leitura em batch pequeno dos pendentes, marcando como entregue | TRANSCRICAO | `[09:08] Diego` |
| ADR-002-D1b | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Registro da decisão "worker em polling, 2s" | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002-D2 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| ADR-002-D3 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Entry `src/worker.ts` + script `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| ADR-002-D3a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Lógica em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` | TRANSCRICAO | `[09:28] Bruno` |
| ADR-002-D4 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Mesmo banco, mesma stack, mesmo Prisma | TRANSCRICAO | `[09:11] Bruno` |
| ADR-002-D4a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | PrismaClient separado por processo, mesma `DATABASE_URL` | TRANSCRICAO | `[09:30] Bruno` |
| ADR-002-D4b | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Integração | Factory `createPrismaClient()` permite uma instância por processo | CODIGO | `src/config/database.ts` |
| ADR-002-D5 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | Single-worker; ordem implícita por order_id via created_at | TRANSCRICAO | `[09:12] Diego` |
| ADR-002-D5a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | Limitação conhecida: sem ordem global | TRANSCRICAO | `[09:13] Larissa` |
| ADR-002-D5b | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Requisito de Produto | Clientes não exigem ordem global | TRANSCRICAO | `[09:14] Marcos` |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Trigger no banco para reagir à inserção | TRANSCRICAO | `[09:09] Bruno` |
| ADR-002-ALT-01a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Trade-off | MySQL sem LISTEN/NOTIFY; trigger não notifica processo externo | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Worker no processo da API: restart da API derruba o worker | TRANSCRICAO | `[09:11] Diego` |
| ADR-002-ALT-03 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Adiado | Múltiplos workers (particionar por order_id ou lock pessimista) | TRANSCRICAO | `[09:13] Diego` |
| ADR-002-CON-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Consequência | Polling de 2s atende ao requisito de < 10s | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-CON-01a | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Consequência | PM aceita 2 segundos | TRANSCRICAO | `[09:10] Marcos` |
| ADR-002-DIV-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Divergência | "Latência mínima de 2s no pior caso": contraditório; registrado como espera máxima | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002-DIV-02 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Questão em aberto | Ordem por order_id versus evento antigo em backoff (não discutido) | TRANSCRICAO | `[09:12] Diego` |
| ADR-002-DIV-03 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Questão em aberto | Timeout de 10s por chamada pode estender o ciclo de 2s; tamanho do batch não definido | TRANSCRICAO | `[09:42] Diego` |
| ADR-003-CTX-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Contexto | Já houve cliente com 2h de indisponibilidade em manutenção | TRANSCRICAO | `[09:16] Diego` |
| ADR-003-D1 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Backoff exponencial com teto; ao estourar → DLQ | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-D2 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003-D2a | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Progressão proposta, "quase 15 horas" no total | TRANSCRICAO | `[09:17] Diego` |
| ADR-003-D2b | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Confirmação no resumo final | TRANSCRICAO | `[09:48] Larissa` |
| ADR-003-D3 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Requisito Não Funcional | Timeout HTTP de 10s conta como falha → retry | TRANSCRICAO | `[09:42] Diego` |
| ADR-003-D4 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Tabela `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-D5 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Requisito Funcional | `POST /admin/webhooks/dead-letter/:id/replay` devolve o evento à outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-D5a | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Requisito Funcional | Endpoint de replay reafirmado | TRANSCRICAO | `[09:35] Diego` |
| ADR-003-D6 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Restrição | Replay exige role ADMIN e loga quem executou | TRANSCRICAO | `[09:36] Sofia` |
| ADR-003-D6a | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Reaproveitar o `requireRole` existente | TRANSCRICAO | `[09:36] Larissa` |
| ADR-003-D6b | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Integração | `requireRole(...roles)` existente | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | Retry indefinido: evento pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | 3 tentativas (mais agressivo) | TRANSCRICAO | `[09:16] Bruno` |
| ADR-003-ALT-02a | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Trade-off | 3 tentativas se esgotam em ~30 min; insuficiente | TRANSCRICAO | `[09:16] Diego` |
| ADR-003-ALT-03 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | Marcar "failed" na própria outbox | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003-CON-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Consequência | Janela de ~15h aceitável para o PM | TRANSCRICAO | `[09:17] Marcos` |
| ADR-003-CON-02 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Fora de escopo | E-mail ao cliente sobre falhas fica para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| ADR-003-DIV-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Divergência / Questão em aberto | "5 tentativas" com 5 intervalos: 5 envios no total ou 1+5 retries? Em aberto | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003-DIV-02 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Divergência | Estimativa "12 ou 24 horas" versus ~14,6h calculadas | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-DIV-03 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Questão em aberto | Relação entre o status "falhou" da outbox e a DLQ separada | TRANSCRICAO | `[09:08] Diego` |
| ADR-003-DIV-04 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Adiado | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| ADR-004-CTX-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Requisito Não Funcional | Cliente deve validar a origem e a integridade do payload | TRANSCRICAO | `[09:19] Sofia` |
| ADR-004-CTX-02 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Contexto | Histórico de cliente que vazou secret em log | TRANSCRICAO | `[09:22] Diego` |
| ADR-004-CTX-03 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Contexto | `redactPaths` do Pino não cobrem `secret` | CODIGO | `src/shared/logger/index.ts` |
| ADR-004-D1 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC no header `X-Signature`, verificado pelo cliente | TRANSCRICAO | `[09:20] Sofia` |
| ADR-004-D1a | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004-D2 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Secret única por endpoint, não global | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-D3 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Configuração armazena url + secret + customer_id + ativo | TRANSCRICAO | `[09:21] Bruno` |
| ADR-004-D4 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| ADR-004-D5 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Requisito Funcional | Rotação via API; secret antiga válida por 24h em paralelo | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-D6 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Restrição | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| ADR-004-D7 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Restrição | URL https obrigatória, validada no schema Zod | TRANSCRICAO | `[09:23] Sofia` |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret global: se uma vaza, vazam todas | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-ALT-02 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret sem rotação / troca sem grace period | TRANSCRICAO | `[09:22] Diego` |
| ADR-004-CON-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Consequência | HMAC-SHA256 é padrão de mercado com bibliotecas disponíveis | TRANSCRICAO | `[09:20] Sofia` |
| ADR-004-CON-02 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Consequência | Secret precisa ser recuperável (diferente da senha, guardada como hash bcrypt) | CODIGO | `src/modules/users/user.service.ts` |
| ADR-004-DIV-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Divergência | `WEBHOOK_SECRET_REQUIRED` implica secret enviada pelo cliente | TRANSCRICAO | `[09:28] Bruno` |
| ADR-004-DIV-01a | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Divergência | Resolvida: secret gerada pelo servidor; código não adotado | TRANSCRICAO | `[09:31] Marcos` |
| ADR-004-DIV-02 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Questão em aberto | `X-Timestamp` não coberto pela assinatura (HMAC só sobre o corpo) | TRANSCRICAO | `[09:44] Diego` |
| ADR-004-DIV-03 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Questão em aberto | Qual secret assina durante as 24h de grace period | TRANSCRICAO | `[09:21] Sofia` |
| ADR-005-CTX-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Contexto | Duplicatas possíveis (timeout de 10s com cliente que já processou) | TRANSCRICAO | `[09:42] Diego` |
| ADR-005-D1 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Entrega at-least-once | TRANSCRICAO | `[09:24] Diego` |
| ADR-005-D1a | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Fechamento: at-least-once com X-Event-Id para dedup | TRANSCRICAO | `[09:26] Larissa` |
| ADR-005-D2 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | `X-Event-Id` = UUID gerado na entrada da outbox, único por evento | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-D4 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Dependência | PM documenta a deduplicação no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once: exige coordenação dos dois lados | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-ALT-02 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Alternativa descartada | At-most-once: conflita com a política de retry (cliente offline) | TRANSCRICAO | `[09:14] Larissa` |
| ADR-005-CON-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Trade-off | Joga responsabilidade de dedup para o cliente (objeção) | TRANSCRICAO | `[09:25] Sofia` |
| ADR-005-DIV-01 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Questão em aberto | Replay da DLQ mantém o event_id original? | TRANSCRICAO | `[09:18] Diego` |
| ADR-005-DIV-02 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Questão em aberto | event_id é o próprio id UUID da outbox ou um campo separado? | TRANSCRICAO | `[09:51] Larissa` |
| ADR-006-CTX-01 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Composição manual de controllers/services/repositories | CODIGO | `src/app.ts` |
| ADR-006-CTX-02 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Montagem de rotas sob `/api/v1` | CODIGO | `src/routes/index.ts` |
| ADR-006-CTX-03 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Classe base `AppError(message, statusCode, errorCode, details)` | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-CTX-04 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Subclasses `InvalidStatusTransitionError`, `InsufficientStockError` e códigos UPPER_SNAKE | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-CTX-05 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Middleware de erro trata AppError, ZodError e Prisma P2002/P2025 | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-CTX-06 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Validação via middleware `validate` com Zod | CODIGO | `src/middlewares/validate.middleware.ts` |
| ADR-006-CTX-07 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | `X-Request-Id` gerado no log de requisição HTTP | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| ADR-006-CTX-08 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Exemplo de `authenticate` + `requireRole('ADMIN')` em rota | CODIGO | `src/modules/users/user.routes.ts` |
| ADR-006-D1 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas | TRANSCRICAO | `[09:27] Bruno` |
| ADR-006-D1a | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006-D2 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Códigos de erro do padrão existente, ex.: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL | TRANSCRICAO | `[09:28] Bruno` |
| ADR-006-D2a | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Prefixo `WEBHOOK_` em todos os códigos do módulo | TRANSCRICAO | `[09:29] Larissa` |
| ADR-006-D3 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Logger Pino existente; nenhuma biblioteca nova; error middleware inalterado | TRANSCRICAO | `[09:29] Bruno` |
| ADR-006-D3a | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Padrão | Logger Pino com redact e base `service` | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-D6 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | CRUD de configuração: qualquer role autenticada "por enquanto" | TRANSCRICAO | `[09:37] Sofia` |
| ADR-006-D7 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | customer_id não vem do JWT; vai no body ou no path (fixado: path) | TRANSCRICAO | `[09:32] Larissa` |
| ADR-006-ALT-01 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Alternativa descartada | customer_id implícito do JWT | TRANSCRICAO | `[09:31] Marcos` |
| ADR-006-ALT-01a | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Trade-off | JWT atual é do usuário operador, não do cliente | TRANSCRICAO | `[09:32] Bruno` |
| ADR-006-ALT-01b | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Trade-off | Payload do JWT contém só `sub`, `email` e `role` | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-ALT-03 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Alternativa descartada | Compartilhar o PrismaClient entre API e worker | TRANSCRICAO | `[09:29] Diego` |
| ADR-006-CON-01 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Risco | Error middleware é do Express; não cobre erros do worker | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-DIV-01 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Divergência | "Temos usuários que representam o cliente": não existe vínculo User↔Customer | TRANSCRICAO | `[09:32] Marcos` |
| ADR-006-DIV-01a | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Divergência | Model `User` sem relação com `Customer` | CODIGO | `prisma/schema.prisma` |
| ADR-006-DIV-02 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Divergência | Status repetido usa `ConflictError` genérico com código `INVALID_STATUS_TRANSITION` | CODIGO | `src/modules/orders/order.service.ts` |

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-META-01 | `docs/RFC.md` | Metadado | Autora: Larissa abre o doc de design e marca a revisão com Bruno e Diego | TRANSCRICAO | `[09:50] Larissa` |
| RFC-META-02 | `docs/RFC.md` | Metadado | Revisores = participantes da reunião (Larissa, Marcos, Bruno, Diego, Sofia) | TRANSCRICAO | `[09:00] Larissa` |
| RFC-TLDR-01 | `docs/RFC.md` | Resumo | Resumo consolidado das decisões (outbox, polling, retry, HMAC, at-least-once, reuso) | TRANSCRICAO | `[09:48] Larissa` |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Três clientes B2B fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-02 | `docs/RFC.md` | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-03 | `docs/RFC.md` | Requisito Não Funcional | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| RFC-CTX-04 | `docs/RFC.md` | Restrição | Somente webhooks outbound | TRANSCRICAO | `[09:03] Sofia` |
| RFC-CTX-05 | `docs/RFC.md` | Contexto | `changeStatus` em transação Prisma (máquina de estados, estoque, history) | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-CTX-06 | `docs/RFC.md` | Contexto | Máquina de estados de pedidos | CODIGO | `src/modules/orders/order.status.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Publicação via `publishWebhookEvent(tx, ...)` na transação; rollback se falhar | TRANSCRICAO | `[09:41] Bruno` |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | `[09:34] Bruno` |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | `[09:52] Larissa` |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | Worker separado, PrismaClient próprio, polling 2s, single-worker | TRANSCRICAO | `[09:11] Diego` |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | Timeout de 10s por chamada | TRANSCRICAO | `[09:42] Diego` |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Backoff 1m/5m/30m/2h/12h → DLQ `webhook_dead_letter` → replay ADMIN | TRANSCRICAO | `[09:17] Larissa` |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | HMAC-SHA256 do corpo, secret por endpoint, rotação com 24h | TRANSCRICAO | `[09:22] Sofia` |
| RFC-PROP-08 | `docs/RFC.md` | Restrição | Apenas URLs https | TRANSCRICAO | `[09:23] Sofia` |
| RFC-PROP-09 | `docs/RFC.md` | Requisito Não Funcional | Payload limitado a 64KB; erro em vez de truncar | TRANSCRICAO | `[09:24] Larissa` |
| RFC-PROP-10 | `docs/RFC.md` | Decisão | At-least-once com X-Event-Id; documentação no portal | TRANSCRICAO | `[09:26] Larissa` |
| RFC-PROP-11 | `docs/RFC.md` | Decisão | Reuso dos padrões: módulo, AppError + WEBHOOK_, Pino, error middleware, requireRole | TRANSCRICAO | `[09:30] Larissa` |
| RFC-PROP-12 | `docs/RFC.md` | Requisito Funcional | CRUD de webhooks: POST (url, status assinados, secret devolvida) | TRANSCRICAO | `[09:31] Marcos` |
| RFC-PROP-13 | `docs/RFC.md` | Requisito Funcional | PATCH, DELETE e GET de listagem; filtro por status por endpoint | TRANSCRICAO | `[09:33] Bruno` |
| RFC-PROP-14 | `docs/RFC.md` | Requisito Funcional | Histórico de entregas (últimas 100: status, payload, resposta, tempo) | TRANSCRICAO | `[09:34] Marcos` |
| RFC-PROP-15 | `docs/RFC.md` | Requisito Funcional | Replay de DLQ com role ADMIN e log de auditoria | TRANSCRICAO | `[09:36] Sofia` |
| RFC-PROP-16 | `docs/RFC.md` | Contrato | Evento `order.status_changed`, JSON enxuto, sem items | TRANSCRICAO | `[09:43] Diego` |
| RFC-PROP-17 | `docs/RFC.md` | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | `[09:44] Diego` |
| RFC-PROP-18 | `docs/RFC.md` | Contrato | Header X-Webhook-Id | TRANSCRICAO | `[09:44] Sofia` |
| RFC-PROP-19 | `docs/RFC.md` | Integração | Graceful shutdown do worker nos moldes do server | CODIGO | `src/server.ts` |
| RFC-SCOPE-01 | `docs/RFC.md` | Fora de escopo | E-mail ao cliente sobre falhas (próxima fase) | TRANSCRICAO | `[09:37] Larissa` |
| RFC-SCOPE-02 | `docs/RFC.md` | Fora de escopo | Dashboard visual (projeto do time de frontend) | TRANSCRICAO | `[09:40] Larissa` |
| RFC-SCOPE-03 | `docs/RFC.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | `[09:02] Marcos` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | HTTP síncrono no changeStatus | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams / fila externa | TRANSCRICAO | `[09:07] Larissa` |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger no MySQL (sem LISTEN/NOTIFY) | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Worker no processo da API | TRANSCRICAO | `[09:11] Diego` |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Retry indefinido / apenas 3 tentativas | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | "failed" na própria outbox, sem DLQ separada | TRANSCRICAO | `[09:18] Diego` |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-08 | `docs/RFC.md` | Alternativa descartada | Exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-09 | `docs/RFC.md` | Alternativa descartada | customer_id extraído do JWT | TRANSCRICAO | `[09:32] Bruno` |
| RFC-ALT-10 | `docs/RFC.md` | Alternativa descartada | Guardar só order_id e renderizar o payload no envio | TRANSCRICAO | `[09:51] Bruno` |
| RFC-ALT-11 | `docs/RFC.md` | Alternativa descartada | Filtrar eventos no envio em vez da inserção | TRANSCRICAO | `[09:34] Diego` |
| RFC-QA-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | `[09:39] Diego` |
| RFC-QA-02 | `docs/RFC.md` | Adiado | E-mail de alerta de falhas: próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| RFC-QA-03 | `docs/RFC.md` | Adiado | Vários workers (partição por order_id / lock pessimista) | TRANSCRICAO | `[09:13] Diego` |
| RFC-QA-04 | `docs/RFC.md` | Questão em aberto | Endurecer permissões do CRUD ("por enquanto sim") | TRANSCRICAO | `[09:37] Sofia` |
| RFC-QA-04a | `docs/RFC.md` | Questão em aberto | Sem vínculo User↔Customer no modelo | CODIGO | `prisma/schema.prisma` |
| RFC-QA-05 | `docs/RFC.md` | Questão em aberto | Ambiguidade: "5 tentativas" vs 5 intervalos (~14,6h) | TRANSCRICAO | `[09:17] Diego` |
| RFC-QA-06 | `docs/RFC.md` | Adiado | Arquivamento das linhas entregues (~30 dias) | TRANSCRICAO | `[09:08] Diego` |
| RFC-QA-07 | `docs/RFC.md` | Questão em aberto | Nome do arquivo: webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | `[09:28] Bruno` |
| RFC-QA-08 | `docs/RFC.md` | Questão em aberto | Ordem por order_id com evento antigo em backoff | TRANSCRICAO | `[09:12] Diego` |
| RFC-QA-09 | `docs/RFC.md` | Questão em aberto | X-Timestamp fora do material assinado | TRANSCRICAO | `[09:44] Diego` |
| RFC-QA-10 | `docs/RFC.md` | Questão em aberto | Qual secret assina durante o grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| RFC-QA-11 | `docs/RFC.md` | Questão em aberto | Armazenamento da secret em repouso (precisa ser recuperável) | CODIGO | `src/modules/users/user.service.ts` |
| RFC-QA-12 | `docs/RFC.md` | Questão em aberto | Replay mantém o event_id original? | TRANSCRICAO | `[09:18] Diego` |
| RFC-QA-13 | `docs/RFC.md` | Questão em aberto | Criação do pedido (null→PENDING) gera evento? | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-QA-14 | `docs/RFC.md` | Questão em aberto | Respostas não-2xx contam como falha? (só o timeout foi definido) | TRANSCRICAO | `[09:42] Diego` |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Estimativa de 3 sprints, detalhada por bloco | TRANSCRICAO | `[09:46] Larissa` |
| RFC-IMP-02 | `docs/RFC.md` | Restrição | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| RFC-IMP-03 | `docs/RFC.md` | Restrição | Atlas quer a entrega até o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| RFC-RISK-01 | `docs/RFC.md` | Risco | Falha na outbox derruba a mudança de status (intencional) | TRANSCRICAO | `[09:40] Bruno` |
| RFC-RISK-02 | `docs/RFC.md` | Risco | Cliente sem deduplicação (responsabilidade transferida) | TRANSCRICAO | `[09:25] Sofia` |
| RFC-RISK-03 | `docs/RFC.md` | Risco | Vazamento de secret; redact do Pino não cobre `secret` | CODIGO | `src/shared/logger/index.ts` |
| RFC-RISK-04 | `docs/RFC.md` | Risco | Vazamento de secret já aconteceu com cliente | TRANSCRICAO | `[09:22] Diego` |

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CTX-01 | `docs/FDD.md` | Contexto | Etapas do `changeStatus` dentro de `prisma.$transaction` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-CTX-02 | `docs/FDD.md` | Contexto | Sem chamada HTTP na transação: cliente lento trava as mudanças de status | TRANSCRICAO | `[09:04] Bruno` |
| FDD-OBJ-01 | `docs/FDD.md` | Objetivo técnico | OT-1 atomicidade status↔evento | TRANSCRICAO | `[09:06] Diego` |
| FDD-OBJ-02 | `docs/FDD.md` | Objetivo técnico | OT-2 latência < 10 s (polling 2 s) | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBJ-03 | `docs/FDD.md` | Objetivo técnico | OT-3 resiliência com backoff e DLQ | TRANSCRICAO | `[09:17] Larissa` |
| FDD-OBJ-04 | `docs/FDD.md` | Objetivo técnico | OT-4 assinatura HMAC-SHA256 em toda entrega | TRANSCRICAO | `[09:22] Sofia` |
| FDD-OBJ-05 | `docs/FDD.md` | Objetivo técnico | OT-5 X-Event-Id estável | TRANSCRICAO | `[09:25] Diego` |
| FDD-OBJ-06 | `docs/FDD.md` | Restrição | OT-6 nenhum pacote novo ("não vamos botar nada novo") | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBJ-06a | `docs/FDD.md` | Dependência | Node ≥ 20 (`engines`) viabiliza o `fetch` nativo | CODIGO | `package.json` |
| FDD-ESC-01 | `docs/FDD.md` | Fora de escopo | E-mail de alerta | TRANSCRICAO | `[09:37] Larissa` |
| FDD-ESC-02 | `docs/FDD.md` | Fora de escopo | Dashboard visual | TRANSCRICAO | `[09:40] Larissa` |
| FDD-ESC-03 | `docs/FDD.md` | Fora de escopo | Rate limiting de saída (observar) | TRANSCRICAO | `[09:39] Larissa` |
| FDD-ESC-04 | `docs/FDD.md` | Fora de escopo | Múltiplos workers | TRANSCRICAO | `[09:13] Diego` |
| FDD-ESC-05 | `docs/FDD.md` | Fora de escopo | Arquivamento da outbox | TRANSCRICAO | `[09:08] Diego` |
| FDD-ESC-06 | `docs/FDD.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | `[09:02] Marcos` |
| FDD-DM-01 | `docs/FDD.md` | Modelo de dados | PKs UUID Char(36) nas novas tabelas | TRANSCRICAO | `[09:51] Larissa` |
| FDD-DM-01a | `docs/FDD.md` | Padrão | Convenção de models, `@@map` e UUID | CODIGO | `prisma/schema.prisma` |
| FDD-DM-02 | `docs/FDD.md` | Modelo de dados | Enum de status da outbox: PENDING/PROCESSING/FAILED/DELIVERED | TRANSCRICAO | `[09:08] Diego` |
| FDD-DM-03 | `docs/FDD.md` | Modelo de dados | `webhooks`: url, secret, customerId, active | TRANSCRICAO | `[09:21] Bruno` |
| FDD-DM-04 | `docs/FDD.md` | Modelo de dados | `events` (lista de status assinados) | TRANSCRICAO | `[09:33] Marcos` |
| FDD-DM-05 | `docs/FDD.md` | Modelo de dados | `previousSecret` + `previousSecretExpiresAt` (grace de 24h) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-DM-06 | `docs/FDD.md` | Proposta de design | Soft delete (`deletedAt`) para preservar evidência | TRANSCRICAO | `[09:18] Diego` |
| FDD-DM-07 | `docs/FDD.md` | Modelo de dados | Índices em status e createdAt na outbox | TRANSCRICAO | `[09:08] Diego` |
| FDD-DM-08 | `docs/FDD.md` | Proposta de design | `nextAttemptAt` + `attempts` agendam o backoff persistido | TRANSCRICAO | `[09:15] Diego` |
| FDD-DM-09 | `docs/FDD.md` | Proposta de design | Uma linha de outbox por endpoint, com `eventId` compartilhado (resolve ADR-005 DIV-02) | TRANSCRICAO | `[09:44] Sofia` |
| FDD-DM-10 | `docs/FDD.md` | Modelo de dados | `webhook_deliveries`: sucesso/falha, resposta, tempo | TRANSCRICAO | `[09:34] Marcos` |
| FDD-DM-11 | `docs/FDD.md` | Modelo de dados | `webhook_dead_letter`: payload, motivo, timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-DM-12 | `docs/FDD.md` | Proposta de design | Outbox marcada FAILED + cópia na DLQ (resolve ADR-003 DIV-03) | TRANSCRICAO | `[09:17] Larissa` |
| FDD-FLX-01 | `docs/FDD.md` | Fluxo | `publishWebhookEvent` após o history, antes da releitura; exceção → rollback | TRANSCRICAO | `[09:40] Bruno` |
| FDD-FLX-02 | `docs/FDD.md` | Fluxo | Assinatura `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:41] Bruno` |
| FDD-FLX-03 | `docs/FDD.md` | Fluxo | Filtro por status na inserção; nenhum interessado → não insere | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLX-04 | `docs/FDD.md` | Fluxo | Payload renderizado como snapshot na inserção | TRANSCRICAO | `[09:52] Larissa` |
| FDD-FLX-04a | `docs/FDD.md` | Decisão | Snapshot confirmado | TRANSCRICAO | `[09:52] Bruno` |
| FDD-FLX-05 | `docs/FDD.md` | Divergência | Limite de 64 KB no worker e não na inserção (lançar na tx violaria "status muda e evento sai") | TRANSCRICAO | `[09:23] Sofia` |
| FDD-FLX-06 | `docs/FDD.md` | Fluxo | Worker com PrismaClient próprio via `createPrismaClient()` | CODIGO | `src/config/database.ts` |
| FDD-FLX-07 | `docs/FDD.md` | Fluxo | Bootstrap/shutdown do worker nos moldes do server | CODIGO | `src/server.ts` |
| FDD-FLX-08 | `docs/FDD.md` | Fluxo | Ciclo: pendentes mais antigos, batch pequeno, a cada 2 s | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLX-09 | `docs/FDD.md` | Proposta de design | Reset de PROCESSING → PENDING no boot (crash recovery) | TRANSCRICAO | `[09:24] Diego` |
| FDD-FLX-10 | `docs/FDD.md` | Proposta (provisório QA-8) | Head-of-line por (webhookId, orderId) para manter a ordem por pedido | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLX-11 | `docs/FDD.md` | Fluxo | POST com timeout de 10 s via AbortSignal | TRANSCRICAO | `[09:42] Diego` |
| FDD-FLX-12 | `docs/FDD.md` | Proposta (provisório QA-14) | Não-2xx, rede e timeout contam como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-FLX-13 | `docs/FDD.md` | Fluxo | Backoff 1m/5m/30m/2h/12h; `MAX_DELIVERY_ATTEMPTS` parametrizado (QA-5) | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLX-14 | `docs/FDD.md` | Fluxo | Ida para a DLQ com payload, motivo e failedAt | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLX-15 | `docs/FDD.md` | Fluxo | Replay recoloca na outbox como PENDING | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLX-16 | `docs/FDD.md` | Proposta (provisório QA-12) | Replay mantém o mesmo eventId | TRANSCRICAO | `[09:25] Diego` |
| FDD-FLX-17 | `docs/FDD.md` | Fluxo | Log de auditoria do replay com quem executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-FLX-18 | `docs/FDD.md` | Fluxo | Rotação: antiga válida por 24 h em paralelo | TRANSCRICAO | `[09:21] Sofia` |
| FDD-FLX-19 | `docs/FDD.md` | Proposta (provisório QA-10) | Duas assinaturas no header durante o grace period | TRANSCRICAO | `[09:22] Sofia` |
| FDD-FLX-20 | `docs/FDD.md` | Restrição | Geração da secret revisada pela Sofia | TRANSCRICAO | `[09:46] Sofia` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /customers/:customerId/webhooks (url, events; secret devolvida) | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-01a | `docs/FDD.md` | Contrato | customerId no path, não no JWT | TRANSCRICAO | `[09:32] Larissa` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET de listagem dos webhooks do customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /webhooks/:id/rotate-secret (path proposto) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries, últimas 100 | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-07a | `docs/FDD.md` | Restrição | Replay exige ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| FDD-CONTRATO-08 | `docs/FDD.md` | Restrição | CRUD acessível a qualquer role autenticada | TRANSCRICAO | `[09:37] Sofia` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Payload: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | `[09:44] Diego` |
| FDD-CONTRATO-11 | `docs/FDD.md` | Contrato | Header X-Webhook-Id | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-12 | `docs/FDD.md` | Requisito Não Funcional | Corpo ≤ 64 KB | TRANSCRICAO | `[09:24] Diego` |
| FDD-CONTRATO-13 | `docs/FDD.md` | Padrão | Envelope de erro `{ error: { code, message, details? } }` | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CONTRATO-14 | `docs/FDD.md` | Padrão | Prefixo `/api/v1` | CODIGO | `src/app.ts` |
| FDD-CONTRATO-15 | `docs/FDD.md` | Dependência | Exemplo de verificação HMAC para o portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| FDD-ERR-01 | `docs/FDD.md` | Padrão | Prefixo WEBHOOK_ em todos os códigos | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERR-02 | `docs/FDD.md` | Erro | WEBHOOK_NOT_FOUND / WEBHOOK_INVALID_URL citados na reunião | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-03 | `docs/FDD.md` | Restrição | `NotFoundError` fixa NOT_FOUND → precisa de subclasse própria | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERR-04 | `docs/FDD.md` | Divergência | `validate` converte Zod em VALIDATION_ERROR → WEBHOOK_INVALID_URL vai em details | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-ERR-04a | `docs/FDD.md` | Divergência | Sofia: https é "só uma validação no schema Zod" | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERR-05 | `docs/FDD.md` | Erro | WEBHOOK_PAYLOAD_TOO_LARGE: erra em vez de truncar | TRANSCRICAO | `[09:24] Larissa` |
| FDD-ERR-06 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_TIMEOUT (10 s) | TRANSCRICAO | `[09:42] Diego` |
| FDD-ERR-07 | `docs/FDD.md` | Divergência | WEBHOOK_SECRET_REQUIRED não adotado (secret gerada pelo servidor) | TRANSCRICAO | `[09:31] Marcos` |
| FDD-ERR-08 | `docs/FDD.md` | Padrão | Classe base AppError reutilizada | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout de 10 s | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Backoff persistido em nextAttemptAt | TRANSCRICAO | `[09:17] Larissa` |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Sem fallback automático; e-mail fora da fase | TRANSCRICAO | `[09:37] Larissa` |
| FDD-RES-04 | `docs/FDD.md` | Resiliência | Polling de 2 s sem sobreposição (setTimeout encadeado) | TRANSCRICAO | `[09:09] Diego` |
| FDD-RES-05 | `docs/FDD.md` | Proposta de design | Batch padrão 10 ("batch pequeno") | TRANSCRICAO | `[09:08] Diego` |
| FDD-RES-06 | `docs/FDD.md` | Integração | Novas variáveis no envSchema com default | CODIGO | `src/config/env.ts` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs via Pino existente, eventos snake_case | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-01a | `docs/FDD.md` | Padrão | Logger Pino com redact | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Redact de secret por histórico de vazamento | TRANSCRICAO | `[09:22] Diego` |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Log de auditoria `webhook_dlq_replayed` com userId | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-04 | `docs/FDD.md` | Observabilidade | Métrica `webhook_delivery_lag_ms` mede o SLO < 10 s | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBS-05 | `docs/FDD.md` | Observabilidade | Métricas de volume por cliente como insumo para o rate limiting | TRANSCRICAO | `[09:38] Diego` |
| FDD-OBS-06 | `docs/FDD.md` | Observabilidade | Tracing via correlação eventId/orderId/requestId | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | MySQL 8 com suporte a JSON | CODIGO | `docker-compose.yml` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | `uuid` já presente gera o eventId | CODIGO | `package.json` |
| FDD-DEP-03 | `docs/FDD.md` | Dependência | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | `[09:46] Sofia` |
| FDD-DEP-04 | `docs/FDD.md` | Dependência | Script `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-01 | `docs/FDD.md` | Integração | changeStatus chama publishWebhookEvent; TxClient; construtor inalterado | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Máquina de estados inalterada; OrderStatus define os events | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | `z.nativeEnum(OrderStatus)` como padrão de schema | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Novas classes de erro exportadas junto às existentes | CODIGO | `src/shared/errors/index.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | authenticate + requireRole('ADMIN'); req.user.id para auditoria | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | buildControllers instancia o módulo webhooks | CODIGO | `src/app.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Controllers + montagem das rotas `/` e `/admin/webhooks` | CODIGO | `src/routes/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Rotas `/:id` de customers não colidem com `/:customerId/webhooks` | CODIGO | `src/modules/customers/customer.routes.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Testes seguem supertest + factories; setup limpa as novas tabelas | CODIGO | `tests/setup.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Helpers getTestApp/bootstrapAuthenticatedUser | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Estrutura de arquivos do módulo webhooks | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Função importada e não injetada no OrderService | TRANSCRICAO | `[09:41] Diego` |
| FDD-AC-01 | `docs/FDD.md` | Critério de aceite | Falha na outbox → rollback total do changeStatus | TRANSCRICAO | `[09:41] Diego` |
| FDD-AC-02 | `docs/FDD.md` | Critério de aceite | Filtro de status sem linha inútil | TRANSCRICAO | `[09:34] Bruno` |
| FDD-AC-03 | `docs/FDD.md` | Critério de aceite | Contrato de saída sem items, headers completos | TRANSCRICAO | `[09:43] Diego` |
| FDD-AC-04 | `docs/FDD.md` | Critério de aceite | http:// rejeitado com 400 | TRANSCRICAO | `[09:23] Sofia` |
| FDD-AC-05 | `docs/FDD.md` | Critério de aceite | Replay: 403 para OPERATOR, 202 para ADMIN, auditoria | TRANSCRICAO | `[09:36] Sofia` |
| FDD-AC-06 | `docs/FDD.md` | Critério de aceite | Isolamento entre API e worker | TRANSCRICAO | `[09:11] Diego` |
| FDD-AC-07 | `docs/FDD.md` | Critério de aceite | Ordem por pedido preservada com evento em backoff | TRANSCRICAO | `[09:13] Larissa` |
| FDD-RISK-01 | `docs/FDD.md` | Risco | Timeout padrão de 5 s da transação interativa do Prisma | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-RISK-02 | `docs/FDD.md` | Risco | Single-worker: ponto único de falha | TRANSCRICAO | `[09:12] Diego` |
| FDD-RISK-03 | `docs/FDD.md` | Risco | Duplicatas aceitas pelo at-least-once | TRANSCRICAO | `[09:24] Diego` |
| FDD-RISK-04 | `docs/FDD.md` | Risco | Secret em claro no banco (QA-11) | CODIGO | `src/modules/users/user.service.ts` |
| FDD-RISK-05 | `docs/FDD.md` | Risco | Crescimento da outbox e do histórico | TRANSCRICAO | `[09:07] Bruno` |
| FDD-RISK-06 | `docs/FDD.md` | Risco | CRUD sem vínculo usuário↔cliente | CODIGO | `prisma/schema.prisma` |

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-META-01 | `docs/PRD.md` | Metadado | Prazo-alvo: fim de novembro (Atlas) | TRANSCRICAO | `[09:45] Marcos` |
| PRD-META-02 | `docs/PRD.md` | Metadado | Estimativa de 3 sprints | TRANSCRICAO | `[09:47] Larissa` |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | `docs/PRD.md` | Contexto | Nenhum mecanismo de notificação, evento ou fila na plataforma | CODIGO | `package.json` |
| PRD-PROB-01 | `docs/PRD.md` | Problema | Polling em GET /orders deixa a integração lenta e cara | TRANSCRICAO | `[09:00] Marcos` |
| PRD-PROB-02 | `docs/PRD.md` | Risco de negócio | Atlas pode migrar para concorrente sem entrega no trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-PROB-03 | `docs/PRD.md` | Contexto | Rotas de pedidos existentes (GET /orders, PATCH /:id/status) | CODIGO | `src/modules/orders/order.routes.ts` |
| PRD-PUB-01 | `docs/PRD.md` | Público-alvo | Usuários da API que representam o cliente cadastram webhooks | TRANSCRICAO | `[09:32] Marcos` |
| PRD-PUB-02 | `docs/PRD.md` | Público-alvo | Perfis ADMIN e OPERATOR existentes | CODIGO | `prisma/schema.prisma` |
| PRD-PUB-03 | `docs/PRD.md` | Público-alvo | PM documenta o contrato no portal do desenvolvedor | TRANSCRICAO | `[09:40] Marcos` |
| PRD-CEN-01 | `docs/PRD.md` | Cenário | Assinatura seletiva (só SHIPPED e DELIVERED) | TRANSCRICAO | `[09:33] Marcos` |
| PRD-CEN-02 | `docs/PRD.md` | Cenário | Aviso enxuto; detalhes via GET /orders/:id | TRANSCRICAO | `[09:43] Diego` |
| PRD-CEN-03 | `docs/PRD.md` | Cenário | Cliente em manutenção de 2 h | TRANSCRICAO | `[09:16] Diego` |
| PRD-CEN-04 | `docs/PRD.md` | Cenário | Falha prolongada → guarda e reprocessamento por ADMIN | TRANSCRICAO | `[09:18] Diego` |
| PRD-CEN-05 | `docs/PRD.md` | Cenário | Vazamento de secret em log do cliente → rotação | TRANSCRICAO | `[09:22] Diego` |
| PRD-CEN-06 | `docs/PRD.md` | Cenário | Aviso duplicado deduplicado pelo event_id | TRANSCRICAO | `[09:25] Diego` |
| PRD-CEN-07 | `docs/PRD.md` | Cenário | Diagnóstico pelo histórico de entregas | TRANSCRICAO | `[09:34] Marcos` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo / Métrica | Aviso em < 10 s | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo / Métrica | 0 mudanças de status sem evento | TRANSCRICAO | `[09:40] Bruno` |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo / Métrica | Janela de retentativa de ≈ 15 h | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-03a | `docs/PRD.md` | Objetivo / Métrica | Janela de 15 h aceitável para o produto | TRANSCRICAO | `[09:17] Marcos` |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo / Métrica | 3 de 3 clientes até o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo / Métrica | ≤ 3 sprints incluindo a revisão de segurança | TRANSCRICAO | `[09:46] Larissa` |
| PRD-ESC-01 | `docs/PRD.md` | Fora de escopo (adiado) | E-mail ao cliente sobre falhas | TRANSCRICAO | `[09:37] Larissa` |
| PRD-ESC-02 | `docs/PRD.md` | Fora de escopo (descartado) | Dashboard visual: projeto do frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-ESC-03 | `docs/PRD.md` | Fora de escopo (adiado) | Rate limiting: observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| PRD-ESC-04 | `docs/PRD.md` | Fora de escopo (descartado) | Webhooks inbound | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESC-05 | `docs/PRD.md` | Fora de escopo (descartado) | Ordem global | TRANSCRICAO | `[09:14] Marcos` |
| PRD-ESC-06 | `docs/PRD.md` | Fora de escopo (descartado) | Exactly-once | TRANSCRICAO | `[09:25] Diego` |
| PRD-ESC-07 | `docs/PRD.md` | Fora de escopo (adiado) | Arquivamento dos eventos entregues | TRANSCRICAO | `[09:08] Diego` |
| PRD-ESC-08 | `docs/PRD.md` | Fora de escopo (adiado) | Vários processos de entrega em paralelo | TRANSCRICAO | `[09:13] Diego` |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com URL e status desejados | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Editar webhook | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Remover webhook | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Cliente-alvo explícito na requisição, não pelo token | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Aviso por mudança de status, filtrado pelos status assinados | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Conteúdo enxuto do aviso, sem itens | TRANSCRICAO | `[09:43] Diego` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Aviso reflete o pedido no momento da mudança (snapshot) | TRANSCRICAO | `[09:52] Larissa` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Aviso assinado e verificável pelo cliente | TRANSCRICAO | `[09:19] Sofia` |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Identificador único do evento estável entre tentativas | TRANSCRICAO | `[09:25] Diego` |
| PRD-FR-11a | `docs/PRD.md` | Requisito Funcional | Identificador do webhook de destino no aviso | TRANSCRICAO | `[09:44] Sofia` |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Retentativas em 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Larissa` |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Guarda dos avisos que esgotaram as tentativas, com motivo | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Reprocessamento por ADMIN com registro de quem fez | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-16 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | `[09:34] Marcos` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | < 10 s até o envio | TRANSCRICAO | `[09:02] Marcos` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Atomicidade entre status e aviso | TRANSCRICAO | `[09:41] Diego` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | `[09:26] Larissa` |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento não afeta outras mudanças de status | TRANSCRICAO | `[09:04] Bruno` |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 s por aviso | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | URL somente HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | HMAC-SHA256 com secret por webhook | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Reprocessamento só para ADMIN; cadastro para qualquer autenticado | TRANSCRICAO | `[09:37] Sofia` |
| PRD-NFR-08a | `docs/PRD.md` | Requisito Não Funcional | `requireRole` existente aplica a restrição ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | `[09:46] Sofia` |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64 KB; erro em vez de truncar | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Ordem garantida apenas por pedido | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-12 | `docs/PRD.md` | Requisito Não Funcional | Entrega isolada da API; sem infraestrutura nova | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-13 | `docs/PRD.md` | Requisito Não Funcional | Padrões do projeto (módulo, WEBHOOK_, logs) | TRANSCRICAO | `[09:30] Larissa` |
| PRD-NFR-13a | `docs/PRD.md` | Padrão | Classe base de erros reutilizada | CODIGO | `src/shared/errors/app-error.ts` |
| PRD-DEC-01 | `docs/PRD.md` | Decisão / Trade-off | Registro do aviso junto da mudança de status (outbox) | TRANSCRICAO | `[09:08] Larissa` |
| PRD-DEC-02 | `docs/PRD.md` | Decisão / Trade-off | Entrega separada verificando a cada 2 s | TRANSCRICAO | `[09:10] Larissa` |
| PRD-DEC-03 | `docs/PRD.md` | Decisão / Trade-off | ~15 h de retentativas e depois guarda | TRANSCRICAO | `[09:17] Larissa` |
| PRD-DEC-04 | `docs/PRD.md` | Decisão / Trade-off | Secret por webhook com rotação | TRANSCRICAO | `[09:22] Sofia` |
| PRD-DEC-05 | `docs/PRD.md` | Decisão / Trade-off | At-least-once: o cliente deduplica | TRANSCRICAO | `[09:25] Sofia` |
| PRD-DEC-06 | `docs/PRD.md` | Decisão / Trade-off | Reuso dos padrões; herda a falta de vínculo usuário↔cliente | CODIGO | `prisma/schema.prisma` |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança da Sofia antes do deploy | TRANSCRICAO | `[09:49] Sofia` |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Marcos confirma prazo e comunica os clientes | TRANSCRICAO | `[09:49] Marcos` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | MySQL existente, sem infraestrutura nova | TRANSCRICAO | `[09:07] Diego` |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | MySQL 8 provisionado | CODIGO | `docker-compose.yml` |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Atraso → perda da Atlas | TRANSCRICAO | `[09:00] Marcos` |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Cliente não deduplica | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Volume alto de avisos sobrecarrega o cliente | TRANSCRICAO | `[09:38] Diego` |
| PRD-RISK-05 | `docs/PRD.md` | Risco | Cliente fora do ar além da janela de ~15 h | TRANSCRICAO | `[09:17] Marcos` |
| PRD-RISK-06 | `docs/PRD.md` | Risco | CRUD aberto a qualquer usuário autenticado | TRANSCRICAO | `[09:37] Sofia` |
| PRD-RISK-07 | `docs/PRD.md` | Risco | Ambiguidade da quantidade de tentativas | TRANSCRICAO | `[09:17] Diego` |
| PRD-AC-01 | `docs/PRD.md` | Critério de aceitação | Aviso assinado em < 10 s para status assinado | TRANSCRICAO | `[09:10] Marcos` |
| PRD-AC-02 | `docs/PRD.md` | Critério de aceitação | Status não assinado não gera aviso | TRANSCRICAO | `[09:34] Bruno` |
| PRD-AC-03 | `docs/PRD.md` | Critério de aceitação | Falha no registro impede a mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| PRD-AC-04 | `docs/PRD.md` | Critério de aceitação | http:// recusado | TRANSCRICAO | `[09:23] Sofia` |
| PRD-AC-05 | `docs/PRD.md` | Critério de aceitação | Reprocessamento: acesso negado a não ADMIN; auditoria | TRANSCRICAO | `[09:36] Larissa` |
| PRD-AC-06 | `docs/PRD.md` | Critério de aceitação | Isolamento entre API e entrega | TRANSCRICAO | `[09:11] Diego` |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de testes | Integração com supertest + factories no padrão de `tests/` | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de testes | Revisão de segurança focada em HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-TEST-03 | `docs/PRD.md` | Estratégia de testes | Medir o impacto antes da fase de e-mail | TRANSCRICAO | `[09:37] Larissa` |
| PRD-TEST-04 | `docs/PRD.md` | Estratégia de testes | Métricas de volume para decidir o rate limiting | TRANSCRICAO | `[09:39] Diego` |
