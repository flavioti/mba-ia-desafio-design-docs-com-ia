# ADR-001 — Padrão Outbox no MySQL, na mesma transação do `changeStatus`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:08] Larissa`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança)
- **ADRs relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders`, o que deixa a integração lenta e cara (`[09:00] Marcos`). Para eles, "tempo real" é qualquer coisa abaixo de 10 segundos (`[09:02] Marcos`). O fluxo é só de saída, ou seja, webhook outbound (`[09:02] Marcos`, `[09:03] Sofia`).

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um único `prisma.$transaction`. A transação:

1. lê o pedido com os itens;
2. valida a transição pela máquina de estados (`canTransition` em `src/modules/orders/order.status.ts`);
3. debita estoque (`shouldDebitStock`) ou repõe estoque (`shouldReplenishStock`), conforme o caso;
4. atualiza `orders.status`;
5. grava em `order_status_history`.

A aplicação não tem hoje nenhuma infraestrutura de mensageria, fila ou eventos. As dependências em `package.json` se limitam a Express, Prisma, Pino, Zod, JWT, bcrypt e uuid.

Quando o status muda, é preciso registrar o evento de forma confiável, sem acoplar a transação de pedidos à disponibilidade ou à latência do endpoint do cliente.

## Decisão

- **D1.** Adotar o **padrão Outbox no MySQL existente**. Na mesma transação SQL que atualiza `orders` e grava `order_status_history`, inserir uma linha na tabela `webhook_outbox` com o evento (`[09:06] Diego`, decidido em `[09:08] Larissa`).
- **D2.** A inserção na outbox é **parte obrigatória da transação**. Se ela falhar, a transação inteira sofre rollback: "não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`, `[09:41] Diego`).
- **D3.** O `OrderService` não recebe um repository de webhooks injetado. Ele chama uma **função `publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o `tx` da transação em curso (`[09:41] Bruno`, `[09:41] Diego`). O tipo `TxClient = Prisma.TransactionClient`, já declarado em `order.service.ts`, é o contrato natural para esse parâmetro.
- **D4.** A tabela tem índice em `status` (pendente, processando, falhou, entregue) e em `created_at` (`[09:08] Diego`).
- **D5.** A chave primária é **UUID**, como em todo o projeto (`[09:51] Larissa`). Todos os models em `prisma/schema.prisma` usam `@id @default(uuid()) @db.Char(36)`.
- **D6.** O arquivamento de linhas entregues (cerca de 30 dias) fica **fora do escopo** desta feature (`[09:08] Diego`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **Disparo HTTP síncrono dentro do `changeStatus`** | A transação já é pesada, e um cliente lento travaria a mudança de status de outros pedidos (`[09:04] Bruno`). Se o cliente estiver fora do ar, não faz sentido dar rollback na mudança de status (`[09:04] Bruno`). "Síncrono está fora de questão" (`[09:06] Diego`). |
| A2 | **Redis Streams (ou fila externa semelhante)** | Exigiria subir mais infraestrutura (`[09:07] Larissa`). Para um time pequeno, subir um Redis Cluster para isso é overengineering, e a outbox no MySQL existente resolve (`[09:07] Diego`). |

## Consequências

**Positivas**

- **C+1.** **Consistência garantida entre status e evento.** Se a transação comitou, o evento foi registrado; se deu rollback, o evento some junto (`[09:06] Diego`).
- **C+2.** **Nenhuma infraestrutura nova.** Usa o mesmo MySQL e o mesmo Prisma já em produção (`[09:07] Diego`).
- **C+3.** **Mudança de status desacoplada do cliente.** A disponibilidade e a latência do endpoint do cliente deixam de afetar a mudança de status (`[09:04] Bruno`).

**Negativas**

- **C−1.** **A transação do `changeStatus` fica maior.** Ela ganha pelo menos uma escrita a mais, e uma falha nessa escrita agora derruba a mudança de status (efeito intencional de D2).
- **C−2.** **A outbox cresce continuamente.** Sem arquivamento, que está fora do escopo (D6), as linhas entregues se acumulam e dependem dos índices de D4 para manter a leitura do worker eficiente (`[09:07] Bruno`, `[09:08] Diego`).
- **C−3.** **A entrega é assíncrona.** A latência passa a depender do ciclo de leitura do worker; veja a [ADR-002](ADR-002-worker-separado-em-polling.md).

**Trade-off explícito:** trocamos uma infraestrutura de mensageria dedicada (com mais throughput e reatividade) pela simplicidade operacional e pela atomicidade transacional do MySQL que já existe.

## Divergências e pontos de atenção da transcrição

- **Peso da transação.** O Bruno diz que a transação de mudança de status "decrementa `stock_quantity` dos produtos do pedido" (`[09:04] Bruno`). No código, o débito só ocorre na transição `PENDING → PAID` (`shouldDebitStock`), e a reposição só no cancelamento a partir de `PAID` ou `PROCESSING` (`shouldReplenishStock`), em `src/modules/orders/order.status.ts`. O argumento contra o modelo síncrono continua válido, mas o peso da transação varia conforme a transição.
- **Status `falhou` na outbox.** O Diego lista `falhou` entre os status da outbox (`[09:08] Diego`), mas mais adiante a reunião decide por uma tabela de DLQ separada (`[09:18] Diego`). A forma como os dois convivem está tratada na [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- **Eventos na criação do pedido.** A reunião só trata a mudança de status no `changeStatus`. O `OrderService.create` também grava histórico (`fromStatus: null → PENDING`), mas não ficou decidido se a criação do pedido gera evento. Fica como questão em aberto.
