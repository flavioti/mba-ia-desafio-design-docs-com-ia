# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Rascunho para implementação |
| **Data** | 2026-10-13 |
| **Base** | [RFC](RFC.md) · [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| **Público** | Engenharia (times de Pedidos e de Plataforma) e Segurança |

> **Convenção.** Itens marcados **(proposta)** são detalhes de implementação que a reunião não discutiu. Eles foram derivados das decisões registradas e do código existente e precisam ser validados no review. Itens marcados **(provisório — QA-n)** implementam um default para uma questão em aberto do [RFC §5](RFC.md#5-questões-em-aberto). Para trocar o default, basta mudar uma constante ou configuração.

---

## 1. Contexto e motivação técnica

`OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa, dentro de um `prisma.$transaction(async (tx) => …)`:

1. lê o pedido com os itens;
2. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
3. debita ou repõe estoque conforme `shouldDebitStock` e `shouldReplenishStock`;
4. faz `tx.order.update` e `tx.orderStatusHistory.create`;
5. relê o pedido com as relações.

A feature precisa publicar um evento a cada mudança de status sem fazer chamada HTTP dentro dessa transação ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) e entregá-lo ao cliente em menos de 10 s no caminho feliz. Hoje o projeto não tem eventos, filas, scheduler nem cliente HTTP de saída.

## 2. Objetivos técnicos

| # | Objetivo | Critério verificável |
| --- | --- | --- |
| OT-1 | **Atomicidade status↔evento** | Toda mudança de status que gera evento comitado tem a linha correspondente na outbox. Um rollback em qualquer ponto não deixa a linha. |
| OT-2 | **Latência** | Do commit ao primeiro envio HTTP, ≤ intervalo de polling (2 s) + tempo de processamento do batch. Meta: < 10 s no p95, medida por `webhook_delivery_lag_ms`. |
| OT-3 | **Resiliência** | Falhas seguem o backoff 1m/5m/30m/2h/12h. Esgotadas as tentativas, o evento fica na DLQ e pode ser reprocessado por um ADMIN. |
| OT-4 | **Autenticidade** | Toda entrega leva `X-Signature` HMAC-SHA256 do corpo com a secret do endpoint. |
| OT-5 | **Idempotência no cliente** | `X-Event-Id` fica estável entre tentativas e replays. |
| OT-6 | **Zero dependência nova** | Nenhum pacote novo em `package.json`. Usa `node:crypto` e o `fetch` nativo (Node ≥ 20, conforme `engines`). |

## 3. Escopo e exclusões

**Incluso**

- Tabelas de configuração, outbox, histórico de entregas e DLQ.
- A função `publishWebhookEvent` chamada pelo `changeStatus`.
- O worker (`src/worker.ts`) e o processor.
- Endpoints de CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ.
- Assinatura HMAC, retry e DLQ.
- Logs e métricas derivadas de log.

**Excluído**

- E-mail de alerta ao cliente (próxima fase).
- Dashboard visual (time de frontend).
- Webhooks inbound.
- Rate limiting de saída (vai ser observado; QA-1).
- Múltiplos workers (QA-3).
- Arquivamento ou limpeza da outbox (QA-6).
- Vínculo entre usuário e cliente / endurecimento das permissões (QA-4).
- Evento na criação do pedido (QA-13). Nesta fase, só `changeStatus` publica.
- Listagem de itens da DLQ via API (não discutida; o replay recebe o id direto).

---

## 4. Modelo de dados

Novos models em `prisma/schema.prisma`. Todos usam o padrão `@id @default(uuid()) @db.Char(36)` (`[09:51] Larissa`). A migration só **adiciona** tabelas e não altera nenhuma das existentes.

```prisma
enum WebhookOutboxStatus {   // [09:08] Diego: pendente, processando, falhou, entregue
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(255)        // QA-11: armazenamento em repouso pendente (Sofia)
  previousSecret          String?   @db.VarChar(255)        // vale durante o grace period de 24h
  previousSecretExpiresAt DateTime?
  events                  Json                              // lista de OrderStatus assinados
  active                  Boolean   @default(true)
  deletedAt               DateTime?                         // (proposta) soft delete preserva a evidência
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer                @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)   // X-Event-Id; igual em todas as linhas do mesmo evento
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64) // "order.status_changed"
  payload       Json                               // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())  // (proposta) agenda do backoff
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([status])                  // [09:08] Diego
  @@index([createdAt])               // [09:08] Diego
  @@index([status, nextAttemptAt])   // (proposta) consulta do worker
  @@index([webhookId, orderId, createdAt]) // (proposta) checagem de ordem por pedido
  @@map("webhook_outbox")
}

model WebhookDelivery {              // histórico de entregas, [09:34] Marcos
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  outboxId       String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text   // (proposta) truncado em 4 KB
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {            // [09:18] Diego: payload, motivo da falha, timestamp
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @db.Char(36)
  eventId       String    @db.Char(36)
  webhookId     String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Decisões de modelagem que o código precisa respeitar:

- **Uma linha de outbox por endpoint de destino** (proposta). Se o cliente tem dois webhooks assinando o mesmo status, a inserção gera duas linhas com o **mesmo `eventId`**. Cada linha tem seu próprio `X-Webhook-Id`, sua assinatura e seu estado de retry. Isso responde à ADR-005 (DIV-02): o `eventId` é um campo próprio e **não** o `id` da linha.
- **`FAILED` na outbox e DLQ convivem** (proposta, ADR-003 DIV-03). Quando as tentativas se esgotam, a linha da outbox vai para `FAILED`, e a cópia com o motivo vai para `webhook_dead_letter`. O worker só lê linhas `PENDING`.
- `Customer` ganha a relação inversa `webhooks Webhook[]`.

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro do `changeStatus`)

Sequência executada dentro da transação do `PATCH /api/v1/orders/:id/status`:

1. `BEGIN`; `findUnique` do pedido com os itens.
2. `canTransition` e débito ou reposição de estoque (comportamento atual).
3. `update` em `orders` e `insert` em `order_status_history` (comportamento atual).
4. `publishWebhookEvent(tx, order, from, to)`:
   1. Busca os webhooks do `order.customerId` com `active = true` e `deletedAt IS NULL`.
   2. Filtra, em memória, os que têm `to` em `events`.
   3. Se nenhum sobrar, retorna sem inserir nada.
   4. Senão, gera `eventId = uuidv4()`, renderiza o payload (snapshot) e faz `createMany` em `webhook_outbox`, uma linha por webhook, com `status = PENDING` e `nextAttemptAt = now`.
5. Relê o pedido com as relações; `COMMIT`.

Regras:

1. **Posição da chamada.** `publishWebhookEvent` é chamada **depois** de `tx.orderStatusHistory.create` e **antes** da releitura final. Qualquer exceção propaga e faz rollback da transação inteira (`[09:40] Bruno`).
2. **Filtro na inserção.** Se nenhum webhook ativo do `order.customerId` contém `to` em `events`, nada é inserido (`[09:34] Bruno`).
3. **Snapshot.** O payload é renderizado nesse momento a partir do `order` já carregado na transação (`[09:52] Larissa`). O campo `timestamp` do payload é o instante da mudança.
4. A função **não** valida o tamanho de 64 KB. Lançar erro aqui derrubaria a mudança de status. A checagem fica no worker (§5.2, passo 4).
5. A assinatura da função segue a reunião: `publishWebhookEvent(tx: Prisma.TransactionClient, order: Order, fromStatus: OrderStatus, toStatus: OrderStatus): Promise<void>` (`[09:41] Bruno`).

### 5.2 Processamento pelo worker

O processo `src/worker.ts` faz o bootstrap: Pino e uma instância própria de `PrismaClient` via `createPrismaClient()`. Ele trata `SIGINT` e `SIGTERM` como `src/server.ts`. Ao receber um desses sinais, termina o item em andamento, não começa outro ciclo e chama `prisma.$disconnect()`.

Passos:

1. **Recuperação de crash.** Ao iniciar, o worker devolve para `PENDING` as linhas presas em `PROCESSING` (proposta). Isso pode reenviar um evento que já tinha sido entregue, o que é coberto pela semântica at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
2. **Ciclo de leitura.** O worker lê os pendentes vencidos mais antigos, em batch pequeno (`[09:08] Diego`, `[09:09] Diego`). O tamanho do batch vem de `WEBHOOK_BATCH_SIZE`, com default 10 (proposta). A consulta é `WHERE status = PENDING AND nextAttemptAt <= now ORDER BY createdAt LIMIT WEBHOOK_BATCH_SIZE`. Antes de processar cada linha, o worker a marca como `PROCESSING` com um `updateMany` guardado por `where status = PENDING`.
3. **Ordem por pedido (provisório — QA-8).** O worker pula a linha se houver outra linha do mesmo `(webhookId, orderId)` com `createdAt` menor e status `PENDING` ou `PROCESSING`. Assim, um evento antigo em backoff segura os mais novos do mesmo pedido para o mesmo endpoint, e a ordem por `order_id` se mantém (`[09:12] Diego`).
4. **Pré-checagens.**
   - Endpoint inativo ou removido: vai direto para a DLQ (proposta), preservando a evidência.
   - Corpo acima de 64 KB: falha permanente e vai para a DLQ sem retry (`[09:24] Larissa`).
5. **Envio.** `fetch(url, { method: 'POST', body, headers, signal: AbortSignal.timeout(WEBHOOK_HTTP_TIMEOUT_MS) })`, com timeout de 10 s (`[09:42] Diego`). O corpo é a string JSON exata usada para calcular a assinatura.
6. **Registro.** Cada tentativa grava uma linha em `webhook_deliveries`, com sucesso, status HTTP, corpo da resposta truncado, `durationMs` e `errorCode`.
7. **Classificação (provisório — QA-14).** Resposta 2xx é sucesso. Qualquer outro status HTTP, erro de rede ou timeout é falha e entra no fluxo de retry.
8. Os itens do batch são processados **sequencialmente**, já que só há um worker ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

### 5.3 Retry com backoff

```ts
// src/modules/webhooks/webhook.retry.ts (proposta)
export const BACKOFF_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]; // 1m 5m 30m 2h 12h
export const MAX_DELIVERY_ATTEMPTS = 6; // provisório — QA-5: 6 = 1 envio + 5 retries; 5 = leitura literal
```

Quando uma tentativa falha, `attempts` é incrementado e passa a valer `n`:

- Se `n < MAX_DELIVERY_ATTEMPTS`: `status = PENDING`, `nextAttemptAt = now + BACKOFF_SCHEDULE_MS[n-1]` e `lastError = <código>`.
- Se `n >= MAX_DELIVERY_ATTEMPTS`: o evento vai para a DLQ (§5.4).

| Envio nº | Espera antes deste envio | Acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 | — (assim que o worker lê) | — |
| 2 | 1 min | 1 min |
| 3 | 5 min | 6 min |
| 4 | 30 min | 36 min |
| 5 | 2 h | 2 h 36 min |
| 6 | 12 h | ~14 h 36 min |

Com `MAX_DELIVERY_ATTEMPTS = 6`, a tabela inteira é usada e a janela bate com o "quase 15 horas" de `[09:17] Diego`. Com `5`, o envio nº 6 não acontece e o intervalo de 12 h fica sem uso. **A escolha está pendente em QA-5.** A retentativa respeita o mesmo timeout de 10 s.

### 5.4 Dead Letter Queue e replay

**Ida para a DLQ.** Em uma única transação:

1. `webhook_outbox.status = FAILED`;
2. `INSERT webhook_dead_letter` com `payload`, `failureReason` (último código e mensagem) e `failedAt` (`[09:18] Diego`).

O worker loga `webhook_moved_to_dlq` em nível `warn`.

**Replay** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, role ADMIN). Em uma única transação:

1. Busca o item da DLQ. Se não existir, responde `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se `replayedAt` já estiver preenchido, responde `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (proposta, para evitar replay duplo acidental).
2. Insere uma nova linha na outbox com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`, o **mesmo `payload`** e o **mesmo `eventId`** (provisório — QA-12: mantém a deduplicação no cliente).
3. Grava `replayedAt = now` e `replayedById = req.user.id`.

Depois do commit, a API loga `webhook_dlq_replayed` em nível `info` com `{ userId, deadLetterId, eventId, webhookId }`. Esse é o log de auditoria exigido (`[09:36] Sofia`).

### 5.5 Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret`:

1. Gera uma nova secret.
2. Move a atual para `previousSecret`, com `previousSecretExpiresAt = now + 24h` (`[09:21] Sofia`).
3. Grava a nova em `secret` e a devolve **uma única vez** na resposta.

Se já houver uma rotação em grace period (`previousSecretExpiresAt > now`), responde `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS` (proposta: no máximo duas secrets válidas ao mesmo tempo).

**Assinatura durante o grace period (provisório — QA-10).** O worker envia as duas assinaturas, separadas por vírgula: `X-Signature: sha256=<hmac_nova>,sha256=<hmac_antiga>`. Assim, o cliente que ainda usa a secret antiga continua validando. Passado o `previousSecretExpiresAt`, o worker ignora a secret antiga, e uma limpeza preguiçosa a zera na próxima escrita do registro.

**Geração da secret** (proposta, revisão obrigatória da Sofia, `[09:46] Sofia`): `crypto.randomBytes(32).toString('hex')`, de `node:crypto`.

---

## 6. Contratos públicos

Todos os endpoints ficam sob `/api/v1` (`src/app.ts`) e exigem `Authorization: Bearer <JWT>` (`authenticate`). Os erros seguem o envelope do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`. O `customerId` vai no path, e não no JWT ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)). As rotas `/customers/:customerId/webhooks` e `/webhooks/:id/deliveries` seguem os nomes da reunião (`[09:34] Marcos`, `[09:18] Diego`).

### 6.1 `POST /api/v1/customers/:customerId/webhooks`: cadastrar webhook

Qualquer role autenticada (`[09:37] Sofia`).

```http
POST /api/v1/customers/7c9e6679-7425-40de-944b-e07fc1f90ae7/webhooks
Content-Type: application/json

{
  "url": "https://erp.atlascomercial.com.br/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

```http
HTTP/1.1 201 Created

{
  "id": "3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://erp.atlascomercial.com.br/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2b6c0e4d…(64 hex)",
  "createdAt": "2026-11-03T14:02:11.000Z",
  "updatedAt": "2026-11-03T14:02:11.000Z"
}
```

- **Validação Zod:** `url` precisa ser uma URL válida com protocolo `https:` e até 2048 caracteres. `events` é um array não vazio e sem repetição de `z.nativeEnum(OrderStatus)`.
- **Secret:** é gerada pelo servidor e devolvida só aqui (`[09:31] Marcos`). As outras respostas nunca a incluem.
- **Status codes:**
  - `201` criado;
  - `400` `VALIDATION_ERROR` (em `details`, a URL não-https aparece com a mensagem `WEBHOOK_INVALID_URL`; ver §7);
  - `401` `UNAUTHORIZED`;
  - `404` `WEBHOOK_CUSTOMER_NOT_FOUND`.

### 6.2 `GET /api/v1/customers/:customerId/webhooks`: listar webhooks do cliente

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77",
      "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "url": "https://erp.atlascomercial.com.br/hooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationExpiresAt": null,
      "createdAt": "2026-11-03T14:02:11.000Z",
      "updatedAt": "2026-11-03T14:02:11.000Z"
    }
  ]
}
```

**Status codes:** `200`, `401`, `404` `WEBHOOK_CUSTOMER_NOT_FOUND`. Webhooks com `deletedAt` não aparecem.

### 6.3 `PATCH /api/v1/webhooks/:id`: editar webhook

```http
PATCH /api/v1/webhooks/3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

```http
HTTP/1.1 200 OK

{
  "id": "3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://erp.atlascomercial.com.br/hooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "secretRotationExpiresAt": null,
  "createdAt": "2026-11-03T14:02:11.000Z",
  "updatedAt": "2026-11-04T09:10:00.000Z"
}
```

- **Body:** `url`, `events` e `active` são opcionais, com as mesmas regras do POST. A secret não é editável por aqui; use §6.5.
- **Status codes:** `200`, `400` `VALIDATION_ERROR`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /api/v1/webhooks/:id`: remover webhook

```http
DELETE /api/v1/webhooks/3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77
```

```http
HTTP/1.1 204 No Content
```

- **Semântica:** soft delete (proposta). Preenche `deletedAt` e `active = false`. A partir daí, o webhook deixa de receber eventos novos e passa a responder 404 nos demais endpoints. Linhas já enfileiradas vão para a DLQ com `WEBHOOK_ENDPOINT_INACTIVE` quando o worker as lê. O histórico e a DLQ são preservados como evidência (`[09:18] Diego`).
- **Status codes:** `204`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret

Rota proposta. A reunião exige o endpoint (`[09:21] Sofia`), mas não definiu o path.

```http
POST /api/v1/webhooks/3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77/rotate-secret
```

```http
HTTP/1.1 200 OK

{
  "id": "3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77",
  "secret": "c41d7a…(64 hex)",
  "previousSecretExpiresAt": "2026-11-05T09:30:00.000Z"
}
```

**Status codes:** `200`, `401`, `404` `WEBHOOK_NOT_FOUND`, `409` `WEBHOOK_SECRET_ROTATION_IN_PROGRESS`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries`: histórico de entregas

Retorna as últimas 100 entregas, em ordem decrescente (`[09:34] Marcos`). A query `limit` vai de 1 a 100, com default 100.

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "b7e0…",
      "eventId": "e2a1c9d4-8f3b-4a6e-b1d2-5c7f9e0a3b14",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "e2a1c9d4-…", "event_type": "order.status_changed", "…": "…" },
      "createdAt": "2026-11-03T14:05:12.000Z"
    },
    {
      "id": "a91f…",
      "eventId": "e2a1c9d4-8f3b-4a6e-b1d2-5c7f9e0a3b14",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": { "…": "…" },
      "createdAt": "2026-11-03T14:04:02.000Z"
    }
  ]
}
```

- O campo `payload` vem de um join com a linha da outbox.
- **Status codes:** `200`, `400` `VALIDATION_ERROR` (limit inválido), `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: reprocessar item da DLQ

Exige `requireRole('ADMIN')` (`[09:36] Larissa`).

```http
POST /api/v1/admin/webhooks/dead-letter/5d2e8c1a-0f4b-4e7a-9c3d-1b6a8e2f4c90/replay
Authorization: Bearer <JWT de usuário ADMIN>
```

```http
HTTP/1.1 202 Accepted

{
  "deadLetterId": "5d2e8c1a-0f4b-4e7a-9c3d-1b6a8e2f4c90",
  "outboxId": "0a7b3c9d-2e4f-4a6b-8c1d-3e5f7a9b1c2d",
  "eventId": "e2a1c9d4-8f3b-4a6e-b1d2-5c7f9e0a3b14",
  "status": "PENDING",
  "replayedAt": "2026-11-04T10:00:00.000Z",
  "replayedById": "1b2c3d4e-…"
}
```

- **Por que `202`:** a entrega acontece de forma assíncrona, no próximo ciclo do worker.
- **Status codes:**
  - `202`;
  - `401`;
  - `403` `FORBIDDEN` (vindo do `requireRole` existente);
  - `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`;
  - `409` `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.

### 6.8 Contrato de saída: requisição enviada ao cliente

```http
POST https://erp.atlascomercial.com.br/hooks/oms
Content-Type: application/json
X-Event-Id: e2a1c9d4-8f3b-4a6e-b1d2-5c7f9e0a3b14
X-Webhook-Id: 3f1c2a9e-5b7d-4c11-9a0e-2d6f8b4e1a77
X-Timestamp: 2026-11-03T14:04:02.000Z
X-Signature: sha256=5f8a0c…

{
  "event_id": "e2a1c9d4-8f3b-4a6e-b1d2-5c7f9e0a3b14",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T14:04:00.512Z",
  "order_id": "9b8a7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "total_cents": 15900
}
```

| Elemento | Regra | Origem |
| --- | --- | --- |
| Corpo | JSON com os campos acima, **sem `items`**. O cliente que quiser detalhes consulta `GET /orders/:id`. | `[09:43] Diego` |
| `X-Event-Id` | UUID do evento, estável entre retries e replay | `[09:25] Diego` |
| `X-Signature` | `sha256=` + HMAC-SHA256 hex do corpo exato enviado. Durante o grace period, duas assinaturas separadas por vírgula (QA-10). | `[09:22] Sofia` |
| `X-Timestamp` | ISO 8601 do **envio**, diferente do `timestamp` do payload, que marca o evento. **Não é assinado** (QA-9). | `[09:44] Diego` |
| `X-Webhook-Id` | Id do cadastro de webhook | `[09:44] Sofia` |
| Tamanho | Corpo ≤ 64 KB. Acima disso, falha e vai para a DLQ sem envio. | `[09:24] Larissa` |
| Resposta esperada | Qualquer 2xx dentro de 10 s | `[09:42] Diego` |

**Exemplo de verificação no cliente** (para o portal do desenvolvedor, `[09:26] Marcos`):

```ts
const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = header.split(',').some((sig) => timingSafeEqual(Buffer.from(sig), Buffer.from(expected)));
```

---

## 7. Matriz de erros

Todos os códigos do módulo usam o prefixo `WEBHOOK_` (`[09:29] Larissa`). Na API, as classes estendem `AppError` e são tratadas pelo `errorMiddleware` sem mudança nele. Os códigos internos do worker não são respostas HTTP: eles vão para `webhook_deliveries.errorCode`, `webhook_outbox.lastError`, `webhook_dead_letter.failureReason` e para os logs.

| Código | HTTP | Classe proposta (base) | Onde ocorre | Quando | Retry? |
| --- | --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` (`AppError`) | API: PATCH, DELETE, rotate, deliveries | id inexistente ou com `deletedAt` | — |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` (`AppError`) | API: POST e GET de webhooks | `customerId` do path não existe | — |
| `WEBHOOK_INVALID_URL` | 400 | Regra Zod no schema; sai como `VALIDATION_ERROR` com `details[].message = "WEBHOOK_INVALID_URL: …"` | API: POST e PATCH | URL inválida ou não-https | — |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `ConflictError` com código customizado | API: rotate | já existe secret anterior dentro das 24 h | — |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` | API: replay | id de DLQ inexistente | — |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `ConflictError` com código customizado | API: replay | `replayedAt` já preenchido | — |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | interno | worker | sem resposta em 10 s | Sim |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — | interno | worker | resposta fora da faixa 2xx (QA-14) | Sim |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | interno | worker | DNS, conexão recusada, TLS | Sim |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — | interno | worker | corpo > 64 KB | Não; vai direto para a DLQ |
| `WEBHOOK_ENDPOINT_INACTIVE` | — | interno | worker | webhook desativado ou removido depois do enfileiramento | Não; vai direto para a DLQ |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | interno (`failureReason` da DLQ) | worker | tentativas esgotadas (QA-5) | Não; é a entrada na DLQ |

Observações:

- **Erros 404 precisam de subclasse própria.** `NotFoundError` (`src/shared/errors/http-errors.ts`) fixa o código `NOT_FOUND` e não aceita outro. Por isso `WEBHOOK_NOT_FOUND` e os demais 404 precisam de classe própria estendendo `AppError`, no mesmo molde de `InvalidStatusTransitionError` e `InsufficientStockError`. Já `ConflictError` e `BadRequestError` aceitam código customizado.
- **`WEBHOOK_INVALID_URL` sai como `VALIDATION_ERROR`.** O middleware `validate` (`src/middlewares/validate.middleware.ts`) converte todo `ZodError` em `ValidationError` com o código fixo `VALIDATION_ERROR`. Como a regra https fica no schema Zod (`[09:23] Sofia`), o código `WEBHOOK_INVALID_URL` (`[09:28] Bruno`) chega ao cliente como mensagem do issue em `details`. Validar de novo no service só para trocar o código duplicaria a regra.
- **`WEBHOOK_SECRET_REQUIRED` não é usado.** Ele foi citado em `[09:28] Bruno`, mas a secret é gerada pelo servidor ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

---

## 8. Estratégias de resiliência

| Mecanismo | Configuração | Detalhe |
| --- | --- | --- |
| **Timeout HTTP** | `WEBHOOK_HTTP_TIMEOUT_MS = 10000` | `AbortSignal.timeout`. Estourou, a tentativa conta como falha (`[09:42] Diego`). |
| **Retry / backoff** | `BACKOFF_SCHEDULE_MS = 1m, 5m, 30m, 2h, 12h` e `MAX_DELIVERY_ATTEMPTS` (QA-5) | O agendamento fica em `nextAttemptAt`, persistido no banco, e sobrevive a restart. |
| **Fallback / DLQ** | `webhook_dead_letter` | Evidência e replay manual por ADMIN. **Não há fallback automático**: o e-mail ficou fora desta fase (`[09:37] Larissa`). |
| **Polling** | `WEBHOOK_POLL_INTERVAL_MS = 2000` | Usa `setTimeout` encadeado, e não `setInterval`: o próximo ciclo só começa quando o anterior termina, o que evita sobreposição. |
| **Batch** | `WEBHOOK_BATCH_SIZE = 10` (proposta) | Batch pequeno (`[09:08] Diego`). No pior caso, um ciclo leva cerca de batch × 10 s, quando todos os clientes do lote dão timeout (ADR-002 DIV-03). |
| **Crash do worker** | Reset de `PROCESSING` para `PENDING` no boot | Pode reenviar um evento já entregue; coberto pelo at-least-once. |
| **Banco indisponível no worker** | Captura erro do ciclo, loga `error` e tenta no próximo tick | O processo não morre por falha transitória de consulta. |
| **Banco indisponível na API** | Comportamento atual | A transação do `changeStatus` falha inteira, como hoje. |
| **Ordem por pedido** | Head-of-line por `(webhookId, orderId)` (QA-8) | Um evento em backoff segura os seguintes do mesmo pedido para o mesmo endpoint. |

Novas variáveis em `src/config/env.ts`, todas com default (proposta): `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_HTTP_TIMEOUT_MS` e `WEBHOOK_BATCH_SIZE`. O `.env.example` recebe as três.

---

## 9. Observabilidade

O projeto já usa Pino estruturado (`src/shared/logger/index.ts`) e não tem nenhuma stack de métricas ou tracing. Pela [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md), não entra biblioteca nova nesta fase. Assim, **métricas e tracing são derivados de logs estruturados**, com nomes estáveis, para que uma ferramenta de agregação de logs possa contá-los.

### 9.1 Logs (Pino)

Os nomes de evento seguem o estilo snake_case existente (`server_started`, `http_request`). O worker usa `logger.child({ component: 'webhook_worker' })`.

| Evento | Nível | Campos | Emissor |
| --- | --- | --- | --- |
| `webhook_event_enqueued` | info | `eventId, orderId, fromStatus, toStatus, webhookIds[]` | `publishWebhookEvent` (API) |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs, batchSize` / `signal` | worker |
| `webhook_delivery_succeeded` | info | `eventId, outboxId, webhookId, orderId, attempt, responseStatus, durationMs, lagMs` | worker |
| `webhook_delivery_failed` | warn | os mesmos + `errorCode, nextAttemptAt` | worker |
| `webhook_moved_to_dlq` | warn | `eventId, outboxId, webhookId, deadLetterId, failureReason, attempts` | worker |
| `webhook_dlq_replayed` | info | `userId, deadLetterId, eventId, webhookId` (auditoria, `[09:36] Sofia`) | API |
| `webhook_secret_rotated` | info | `webhookId, userId, previousSecretExpiresAt` | API |
| `webhook_worker_cycle_error` | error | `err` | worker |
| `webhook_outbox_snapshot` | info (a cada N ciclos) | `pending, processing, oldestPendingAgeMs` | worker |

**Redact obrigatório.** Acrescentar `'*.secret'` e `'*.previousSecret'` aos `redactPaths` de `src/shared/logger/index.ts`, que hoje não cobrem secrets. O cliente já teve uma secret vazada em log (`[09:22] Diego`). As respostas do CRUD nunca são logadas com o corpo.

### 9.2 Métricas (derivadas de log)

| Métrica | Tipo | Fonte | Uso |
| --- | --- | --- | --- |
| `webhook_delivery_total{result=success\|failure, errorCode}` | contador | `webhook_delivery_succeeded` / `_failed` | taxa de sucesso por cliente e endpoint |
| `webhook_delivery_duration_ms` | histograma | `durationMs` | latência do endpoint do cliente |
| `webhook_delivery_lag_ms` | histograma | `lagMs` = envio − `createdAt` da outbox | **SLO de < 10 s** do caminho feliz (`[09:02] Marcos`) |
| `webhook_outbox_pending` / `webhook_outbox_oldest_pending_age_ms` | gauge | `webhook_outbox_snapshot` | represamento e saúde do worker |
| `webhook_dead_letter_total` | contador | `webhook_moved_to_dlq` | volume de falhas permanentes |
| `webhook_dlq_replay_total` | contador | `webhook_dlq_replayed` | uso operacional do replay |

Estas métricas também servem de insumo para a decisão adiada sobre rate limiting (QA-1): o volume por cliente e minuto é derivável de `webhook_delivery_total`.

### 9.3 Tracing (correlação)

Não há tracing distribuído no projeto. A correlação ponta a ponta usa IDs:

- **`eventId`** está no log de enfileiramento, na outbox, nos logs do worker, em `webhook_deliveries`, na DLQ e no header `X-Event-Id` recebido pelo cliente. Uma busca por esse campo reconstrói toda a vida do evento, inclusive do lado do cliente.
- **`orderId`** liga o evento ao log `http_request` da mudança de status. Esse log já carrega o `requestId` (`X-Request-Id`, `src/middlewares/request-logger.middleware.ts`) e o path `/orders/:id/status`.
- **`webhookId`** (`X-Webhook-Id`) recorta a busca por endpoint do cliente.

Adotar OpenTelemetry fica como evolução futura, fora desta fase.

---

## 10. Dependências e compatibilidade

- **Runtime:**
  - Node ≥ 20 (`package.json` → `engines`), necessário para o `fetch` global e `AbortSignal.timeout`;
  - `node:crypto` para o HMAC e a geração de secret;
  - **nenhum pacote novo**;
  - o `uuid` (já dependência) gera o `eventId`.
- **Banco:**
  - MySQL 8 (`docker-compose.yml`), com suporte a coluna `JSON` para `events` e `payload`;
  - a migration Prisma **só adiciona** tabelas e o enum, sem alterar colunas existentes, então pode ir para produção antes do código;
  - ver também a seção 4.
- **API existente:**
  - `PATCH /api/v1/orders/:id/status` mantém contrato e resposta;
  - a única mudança observável é a latência um pouco maior, por causa de um SELECT e um INSERT a mais na transação.
- **Transação interativa do Prisma:** o `$transaction(async tx => …)` tem timeout padrão de 5 s. As duas consultas extras são leves, mas a margem deve ser monitorada.
- **Deploy:**
  - novo processo `npm run worker` (`node --env-file=.env dist/worker.js`), com uma única réplica;
  - `npm run worker:dev` (`tsx watch …`) para desenvolvimento;
  - o build atual (`tsc -p tsconfig.build.json`) já inclui `src/worker.ts`.
- **Externas ao código:**
  - documentação de HMAC e deduplicação no portal do desenvolvedor (`[09:26] Marcos`);
  - revisão de segurança da Sofia, com ≥ 2 dias úteis, antes do deploy (`[09:46] Sofia`).

---

## 11. Integração com o sistema existente

| Arquivo existente | Como o módulo de webhooks se integra |
| --- | --- |
| `src/modules/orders/order.service.ts` | Em `changeStatus`, logo depois de `tx.orderStatusHistory.create(...)`, entra `await publishWebhookEvent(tx, order, from, to)`. O `tx` é do tipo `TxClient = Prisma.TransactionClient`, já declarado no arquivo. O construtor do `OrderService` **não muda**, porque a função é importada de `src/modules/webhooks/webhook.publisher.ts` e não injetada (`[09:41] Diego`). O `create` não publica evento (QA-13). |
| `src/modules/orders/order.status.ts` | Não muda. O worker não reaplica regras de transição: o evento só existe se `canTransition` já aprovou dentro da transação. O enum `OrderStatus` define os valores aceitos em `events`. |
| `src/modules/orders/order.schemas.ts` | Serve de referência de padrão: `events` usa `z.array(z.nativeEnum(OrderStatus))`, como o `updateOrderStatusSchema` faz com `toStatus`. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | As novas classes `WebhookNotFoundError`, `WebhookCustomerNotFoundError` e `WebhookDeadLetterNotFoundError` estendem `AppError`, porque `NotFoundError` fixa o código `NOT_FOUND`. Os conflitos usam `ConflictError(message, 'WEBHOOK_…')`. Elas são exportadas por `src/shared/errors/index.ts`, junto com `InvalidStatusTransitionError` e `InsufficientStockError`. |
| `src/middlewares/error.middleware.ts` | **Sem alteração.** Já serializa `AppError` (`code`, `message`, `details`), Zod e Prisma P2002/P2025 (`[09:29] Bruno`). Não cobre o worker, que trata as próprias exceções e loga com o Pino. |
| `src/middlewares/auth.middleware.ts` | `authenticate` protege todas as rotas do módulo. `requireRole('ADMIN')` protege o replay. `req.user.id` alimenta `replayedById` e o log de auditoria. |
| `src/middlewares/validate.middleware.ts` | `validate({ params, body, query })` em todas as rotas, com os schemas de `webhook.schemas.ts`. A regra https é feita aqui, via Zod (ver §7 sobre o código do erro). |
| `src/shared/logger/index.ts` | Logger reutilizado na API e no worker, via `logger.child`. **Muda:** inclusão de `'*.secret'` e `'*.previousSecret'` em `redactPaths`. |
| `src/config/env.ts` | Novas variáveis `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_HTTP_TIMEOUT_MS` e `WEBHOOK_BATCH_SIZE` no `envSchema`, com `z.coerce.number()` e default. |
| `src/config/database.ts` | O worker chama `createPrismaClient()` para ter a própria instância, com a mesma `DATABASE_URL` (`[09:30] Bruno`). A API continua usando o `prisma` exportado. |
| `src/server.ts` | Modelo do novo `src/worker.ts`: função `bootstrap`, `shutdown` em `SIGINT`/`SIGTERM` com `prisma.$disconnect()`, e `logger.fatal` + `process.exit(1)` se o bootstrap falhar. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` (e o controller admin), como faz com os demais módulos. |
| `src/routes/index.ts` | O tipo `Controllers` ganha `webhooks` e `webhookAdmin`. As rotas montadas são `router.use('/', buildWebhookRouter(...))`, cobrindo `/customers/:customerId/webhooks` e `/webhooks/:id…`, e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. As rotas de `customer.routes.ts` (`/:id`) não colidem com `/:customerId/webhooks`. |
| `src/middlewares/request-logger.middleware.ts` | Não muda. O `requestId` do `PATCH /orders/:id/status` correlaciona com `webhook_event_enqueued` via `orderId` (§9.3). |
| `prisma/schema.prisma` | Novos models (§4), enum `WebhookOutboxStatus` e relação inversa `Customer.webhooks`. A migration nova vai em `prisma/migrations/`. |
| `package.json` | Novos scripts `worker` e `worker:dev` (`[09:11] Larissa`). As dependências não mudam. |
| `tests/setup.ts` e `tests/helpers/factories.ts` | Os testes do módulo seguem o padrão `supertest` + `getTestApp()` + `bootstrapAuthenticatedUser()`. O `beforeEach` precisa limpar as novas tabelas antes de `order`/`customer`, por causa das FKs. |

**Arquivos novos** em `src/modules/webhooks/` (`[09:27] Bruno`):

- `webhook.controller.ts`
- `webhook.service.ts`
- `webhook.repository.ts`
- `webhook.routes.ts`
- `webhook.schemas.ts`
- `webhook.publisher.ts` (`publishWebhookEvent`)
- `webhook.signer.ts` (HMAC)
- `webhook.retry.ts`
- `webhook.processor.ts`, cujo nome ainda está em aberto (QA-7)

Fora do módulo, entra o novo entry-point `src/worker.ts`.

---

## 12. Critérios de aceite técnicos

1. **Atomicidade.** Um teste força erro em `publishWebhookEvent` e verifica que o `PATCH /orders/:id/status` falha, que `orders.status` e `order_status_history` ficam inalterados e que o estoque não muda.
2. **Filtro.** Um customer sem webhook ativo, ou sem webhook que assine o `toStatus`, gera zero linhas na outbox ao mudar o status.
3. **Fan-out.** Dois webhooks assinando o mesmo status geram duas linhas com o mesmo `eventId` e `webhookId`s diferentes.
4. **Snapshot.** Alterar o pedido depois do enfileiramento não altera o `payload` entregue.
5. **Contrato de saída.** O corpo tem exatamente os campos da §6.8, sem `items`. Os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` estão presentes.
6. **Assinatura.** `X-Signature` confere com HMAC-SHA256(secret, corpo) em um teste com receptor fake. Durante o grace period, as duas assinaturas validam.
7. **https.** `url` com `http://` resulta em `400` e `details` contendo `WEBHOOK_INVALID_URL`.
8. **Timeout.** Um receptor que demora mais de 10 s gera `WEBHOOK_DELIVERY_TIMEOUT`, `attempts = 1` e `nextAttemptAt ≈ now + 1 min`.
9. **Backoff.** Falhas sucessivas seguem 1m/5m/30m/2h/12h (testado com relógio fake). Ao atingir `MAX_DELIVERY_ATTEMPTS`, a outbox fica `FAILED` e uma linha aparece na DLQ com `failureReason`.
10. **64 KB.** Um payload acima do limite vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem chamada HTTP.
11. **Replay.**
    - Usuário `OPERATOR` recebe `403`.
    - Usuário `ADMIN` recebe `202`, e uma nova linha `PENDING` é criada com o mesmo `eventId`.
    - Um segundo replay recebe `409`.
    - O log `webhook_dlq_replayed` contém o `userId`.
12. **Rotação.** A rotação devolve uma secret nova e grava `previousSecretExpiresAt = now + 24h`. Uma segunda rotação dentro do prazo recebe `409`.
13. **Histórico.** `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, em ordem decrescente, com sucesso, payload, resposta e duração.
14. **Ordem.** Com um evento A em backoff para (webhook W, pedido X), um evento B mais novo para (W, X) não é entregue antes de A.
15. **Isolamento.** Derrubar a API não interrompe o worker, e derrubar o worker não afeta a API. No boot, o worker devolve linhas `PROCESSING` para `PENDING`.
16. **Logs.** Nenhuma secret aparece nos logs (teste do redact).
17. **Latência.** Em ambiente de teste, `webhook_delivery_lag_ms` fica abaixo de 10 s no caminho feliz.

---

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Erro em `publishWebhookEvent` bloqueia mudanças de status | Baixa | Alto | Função pequena, sem I/O externo e coberta pelo teste 1. Validação de tamanho movida para o worker. |
| Transação do `changeStatus` se aproxima do timeout de 5 s do Prisma | Baixa | Médio | Só um SELECT indexado (`customerId, active`) e um `createMany`; monitorar a duração do `http_request` da rota. |
| Single-worker como ponto único de falha | Média | Alto | Reinício automático pelo orquestrador; alerta em `webhook_outbox_oldest_pending_age_ms`; escalar fica para QA-3. |
| Endpoints lentos represam o ciclo (10 s × batch) | Média | Médio | Batch pequeno, métrica de lag, revisão do batch com dados reais. |
| Duplicatas (crash, timeout com processamento no cliente, replay) | Média | Médio | `X-Event-Id` estável, documentação no portal. É risco aceito pelo at-least-once. |
| Secret vazada em log nosso | Baixa | Alto | Redact de `secret` e `previousSecret`, teste 16, revisão da Sofia. |
| Secret em claro no banco | Média | Alto | Fica em aberto (QA-11). Decisão pendente na revisão de segurança antes do deploy. |
| Crescimento indefinido de `webhook_outbox` e `webhook_deliveries` | Alta | Médio | Índices em `status`, `nextAttemptAt` e `createdAt`; arquivamento planejado fora do escopo (QA-6). |
| CRUD sem vínculo usuário↔cliente | Média | Médio | Aceito "por enquanto" (`[09:37] Sofia`); rastreado em QA-4. |
| Ambiguidade das 5 tentativas | Média | Baixo | `MAX_DELIVERY_ATTEMPTS` isolado em uma constante; decisão em QA-5 antes do merge. |
