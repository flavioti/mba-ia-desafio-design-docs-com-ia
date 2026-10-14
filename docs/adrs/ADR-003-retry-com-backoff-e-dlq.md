# ADR-003 — Retry com backoff exponencial e Dead Letter em tabela separada

- **Status:** Aceito, com uma ambiguidade em aberto sobre a contagem de tentativas (ver "Divergências")
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:17] Larissa`, replay decidido em `[09:36] Larissa`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança)
- **ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. A plataforma já teve cliente com duas horas de indisponibilidade em manutenção planejada (`[09:16] Diego`). É preciso definir:

- quantas vezes e com que intervalo reenviar;
- quando desistir;
- onde guardar o que falhou de vez;
- como reprocessar.

## Decisão

- **D1.** **Backoff exponencial com teto de tentativas.** Ao atingir o teto, a falha é considerada permanente e o evento vai para a DLQ (`[09:15] Diego`).
- **D2.** **"5 tentativas", com intervalos de 1 min, 5 min, 30 min, 2 h e 12 h**, somando cerca de 15 h entre a primeira falha e a última tentativa (`[09:17] Diego`, decidido em `[09:17] Larissa`, confirmado no resumo `[09:48] Larissa`). Veja a ambiguidade sobre a contagem em "Divergências".
- **D3.** Um cliente que **não responde em 10 segundos** conta como falha e é marcado para retry (`[09:42] Diego`).
- **D4.** **DLQ em tabela separada, `webhook_dead_letter`**, guardando o payload, o motivo da falha e o timestamp. Isso mantém limpa a leitura da outbox principal e preserva evidência para debug e reprocessamento (`[09:18] Diego`).
- **D5.** **Reprocessamento manual por endpoint administrativo:** `POST /admin/webhooks/dead-letter/:id/replay`. O endpoint devolve o evento à outbox como pendente (`[09:18] Diego`, `[09:35] Diego`).
- **D6.** O replay **exige a role `ADMIN`**, aplicada com o `requireRole` existente em `src/middlewares/auth.middleware.ts` (`[09:36] Sofia`, decidido em `[09:36] Larissa`). Todo replay **registra em log quem o executou**, para auditoria (`[09:36] Sofia`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumir (`[09:15] Diego`). |
| A2 | **Só 3 tentativas**, mais agressivo (proposta de `[09:16] Bruno`) | É pouco: uma indisponibilidade de manhã esgotaria as três tentativas em cerca de 30 min, e já houve cliente fora do ar por 2 h (`[09:16] Diego`). |
| A3 | **Marcar como `failed` na própria outbox**, sem tabela separada (levantada em `[09:17] Larissa`) | A tabela separada deixa a leitura da outbox mais limpa e funciona como evidência para debug e replay (`[09:18] Diego`). |

## Consequências

**Positivas**

- **C+1.** **Janela de cerca de 15 h.** Cobre indisponibilidades longas, como a manutenção de 2 h já vista. O PM considera aceitável: um cliente fora por 15 h "já tá com problema sério dele" (`[09:17] Marcos`).
- **C+2.** **Nenhum evento é descartado em silêncio.** O que esgota as tentativas fica auditável e reprocessável na DLQ.
- **C+3.** **Outbox enxuta.** A outbox só carrega o que ainda está em trânsito, o que favorece a consulta do worker ([ADR-002](ADR-002-worker-separado-em-polling.md)).

**Negativas**

- **C−1.** **Atraso alto em falhas.** Em um cenário de falha, a entrega pode atrasar até cerca de 15 h, o que está longe do requisito de menos de 10 s, válido só para o caminho feliz.
- **C−2.** **Reprocessamento manual.** Depende de uma ação operacional de um ADMIN. Não há reprocessamento automático, e o **alerta ao cliente por e-mail ficou fora desta fase** (`[09:37] Larissa`).
- **C−3.** **Duas tabelas para manter coerentes.** Mover da outbox para a DLQ e voltar no replay exige cuidado transacional.
- **C−4.** **Replay entrega duplicata.** Um evento reprocessado é entregue de novo, o que reforça a dependência da deduplicação no cliente ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).

**Trade-off explícito:** priorizamos não perder eventos e tolerar longas indisponibilidades do cliente, ao custo de entregas muito atrasadas nesses casos e de intervenção manual depois do teto de tentativas.

## Divergências e pontos de atenção da transcrição

- **Contagem de tentativas (ambiguidade, em aberto).** A decisão fala em "5 tentativas" (`[09:17] Larissa`, `[09:48] Larissa`), mas lista **cinco intervalos** (1m/5m/30m/2h/12h). O Diego diz que o total é de "quase 15 horas entre primeira falha e última tentativa" (`[09:17] Diego`). Isso só fecha se houver **1 envio inicial mais 5 retentativas, ou seja, 6 chamadas HTTP**, porque 1+5+30+120+720 min ≈ 14,6 h. Na leitura literal de "5 tentativas no total", o intervalo de 12 h nunca seria usado e a janela cairia para cerca de 2 h 36 min. **Questão em aberto, a ser fechada no RFC.**
- **Janela estimada.** O Diego estima primeiro que 5 tentativas cobrem "uma janela de até 12 ou 24 horas" (`[09:15] Diego`). Pela progressão que ele mesmo detalha depois, a janela real é de cerca de 14,6 h.
- **Status `falhou` na outbox versus DLQ separada.** A outbox tem um status `falhou` (`[09:08] Diego`), e há também a tabela `webhook_dead_letter` (D4). A relação entre os dois (marcar a linha da outbox como `falhou` ao copiá-la para a DLQ, ou removê-la) não foi definida e fica para o FDD.
- **Critério de falha.** Só o timeout de 10 s foi definido explicitamente como falha (`[09:42] Diego`). O tratamento de respostas HTTP fora da faixa 2xx não foi discutido.
- **`event_id` no replay.** Não foi definido se o replay reaproveita o `event_id` original ou gera um novo. Isso afeta a deduplicação no cliente ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)) e fica em aberto.
- **Rate limiting de saída.** Fica como ponto de observação: "observar e decidir depois" (`[09:39] Diego`, `[09:39] Larissa`). Não faz parte desta decisão.
