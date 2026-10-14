# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:10] Larissa`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), algum componente precisa ler os eventos pendentes e fazer as chamadas HTTP. O requisito de produto é entregar em menos de 10 segundos (`[09:02] Marcos`).

Hoje a aplicação tem um único entry-point, `src/server.ts`. Ele cria o app Express com `buildApp({ prisma })`, usa o `PrismaClient` exportado por `src/config/database.ts` e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`. O `package.json` não tem nenhum script além de `dev`, `build`, `start`, `db:*`, `test`, `lint` e `format`.

## Decisão

- **D1.** **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos, em batch pequeno, processa e marca o resultado (`[09:08] Diego`, `[09:09] Diego`, decidido em `[09:10] Larissa`).
- **D2.** O worker roda **em processo separado da API**, para que um restart da API não derrube o worker (`[09:11] Diego`).
- **D3.** Ganha um **entry-point próprio, `src/worker.ts` (arquivo novo, a criar)**, nos moldes de `src/server.ts`, com um script `npm run worker` (`[09:11] Larissa`, `[09:28] Bruno`). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` (`[09:28] Bruno`). O nome final do arquivo não foi fechado.
- **D4.** **Mesmo banco e mesma stack**, com **uma instância própria de `PrismaClient`**. O `PrismaClient` é por processo; o worker usa a mesma `DATABASE_URL`, mas cria uma instância nova (`[09:11] Bruno`, `[09:11] Diego`, `[09:30] Bruno`). A factory `createPrismaClient()` em `src/config/database.ts` já permite isso.
- **D5.** **Um único worker (single-worker).** A ordem de entrega é implícita por `order_id`, seguindo o `created_at` da outbox. **Não há garantia de ordem global**, e isso fica documentado como limitação conhecida (`[09:12] Diego`, `[09:13] Larissa`). Os clientes não pediram ordem global (`[09:14] Marcos`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **Trigger no banco para reagir à inserção** (proposta de `[09:09] Bruno`) | O MySQL não tem um listener nativo como o `NOTIFY/LISTEN` do Postgres. A trigger só executa SQL e não notifica processos externos; avisar o worker exigiria gambiarras, como escrever em arquivo ou chamar um endpoint (`[09:09] Diego`). |
| A2 | **Worker dentro do mesmo processo da API** | Se a API reinicia, o worker cai junto (`[09:11] Diego`). |
| A3 | **Vários workers em paralelo** | Adiado. Perde a garantia de ordem por pedido. No futuro dá para particionar por `order_id` ou usar lock pessimista, mas "é problema do futuro, não agora" (`[09:13] Diego`). |

## Consequências

**Positivas**

- **C+1.** **Atende ao requisito de latência.** O polling adiciona no máximo cerca de 2 s de espera, bem abaixo do limite de 10 s (`[09:09] Diego`, `[09:10] Marcos`).
- **C+2.** **Implementação simples e sem infraestrutura nova.** Reaproveita o padrão de bootstrap e shutdown de `src/server.ts`.
- **C+3.** **Isolamento de falhas.** Um deploy ou crash da API não interrompe as entregas, e vice-versa (`[09:11] Diego`).
- **C+4.** **Ordem por pedido preservada** enquanto houver um único worker (`[09:12] Diego`).

**Negativas**

- **C−1.** **Carga constante no banco.** Há uma consulta a cada 2 s mesmo sem eventos, o que é inerente ao polling.
- **C−2.** **Ponto único de falha e teto de throughput.** O single-worker limita a vazão; escalar exige rever a estratégia de ordem (A3).
- **C−3.** **Mais um processo para operar.** É mais um artefato de deploy e mais um pool de conexões no mesmo MySQL (D4).
- **C−4.** **Sem ordem global.** Só há ordem por `order_id`, e só com um worker (`[09:13] Larissa`).

**Trade-off explícito:** aceitamos até cerca de 2 s de latência extra e uma consulta periódica ao banco em troca de não depender de mecanismos de notificação que o MySQL não oferece nem de infraestrutura adicional.

## Divergências e pontos de atenção da transcrição

- **"Latência mínima" versus pior caso.** A Larissa registra que "a latência mínima vai ser 2 segundos no pior caso" (`[09:10] Larissa`). A frase é contraditória: os 2 s do polling são a espera **máxima** introduzida pelo ciclo no pior caso, não um mínimo. Este ADR registra o valor como o pior caso.
- **Ordem versus retry.** Isto não foi discutido na reunião. Com o backoff da [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), um evento antigo de um pedido pode estar aguardando retry enquanto um evento mais novo do mesmo pedido já está pendente. A garantia de ordem por `order_id` de D5 só se mantém se o worker não entregar o evento mais novo antes do mais antigo. Fica como questão em aberto, para ser resolvida no FDD.
- **Batch versus ciclo de 2 s.** Também não foi discutido. Com timeout de 10 s por chamada (`[09:42] Diego`) e processamento sequencial, um único cliente lento pode estender um ciclo além de 2 s. O tamanho do batch ("pequeno", `[09:08] Diego`) não foi quantificado.
