# ADR-005 — Garantia de entrega at-least-once com deduplicação via `X-Event-Id`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:26] Larissa`)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Bruno (Pedidos), Marcos (PM)
- **ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e o retry ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), o mesmo evento pode ser entregue mais de uma vez. Por exemplo:

- o cliente processa o evento, mas responde depois do timeout de 10 s (`[09:42] Diego`);
- o worker cai entre receber a resposta e marcar o evento como entregue;
- um evento é reprocessado manualmente a partir da DLQ.

É preciso definir qual garantia de entrega a plataforma oferece e como o cliente distingue uma duplicata de um evento novo.

## Decisão

- **D1.** A plataforma garante **entrega at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso (`[09:24] Diego`, decidido em `[09:26] Larissa`).
- **D2.** Cada evento leva um **`event_id` (UUID)**, gerado **no momento em que o evento entra na outbox** e único por evento. Ele é enviado no header **`X-Event-Id`** (`[09:25] Diego`).
- **D3.** **A deduplicação é responsabilidade do cliente**, usando o `event_id` (`[09:25] Diego`).
- **D4.** O PM documenta esse comportamento **em destaque no portal do desenvolvedor** (`[09:26] Marcos`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **Exactly-once** | Exigiria coordenação dos dois lados e fica muito mais complexo (`[09:25] Diego`). |
| A2 | **At-most-once** (enviar uma vez, sem retry) | É incompatível com a política de retry decidida ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) e com o objetivo de não perder notificações quando o cliente fica fora do ar (`[09:14] Larissa`, `[09:16] Diego`). |

## Consequências

**Positivas**

- **C+1.** **Implementação simples.** O padrão é o mesmo de mercado; segundo o Diego, "Stripe faz assim, GitHub faz assim" (`[09:25] Diego`).
- **C+2.** **Encaixa na outbox e no retry.** Combinado com a outbox transacional e o retry, nenhum evento comitado deixa de ser tentado.
- **C+3.** **Identificador estável.** O `event_id` gerado na inserção não muda entre as tentativas, o que permite deduplicar entre retries.

**Negativas**

- **C−1.** **Transfere responsabilidade ao cliente.** Ele precisa implementar a deduplicação (objeção registrada por `[09:25] Sofia`). Um cliente que não deduplica pode processar um evento duas vezes.
- **C−2.** **Depende de comunicação.** A garantia depende de o cliente ler a documentação do portal (D4).

**Trade-off explícito:** aceitamos duplicatas eventuais, tratadas no cliente, em troca de não perder eventos e de não implementar coordenação exactly-once entre dois sistemas.

## Divergências e pontos de atenção da transcrição

- **Objeção da Sofia.** A Sofia aponta que a decisão "joga responsabilidade pro cliente" (`[09:25] Sofia`). O Diego reconhece a objeção, mas mantém a decisão por ser o padrão de mercado (`[09:25] Diego`). A objeção não foi revertida; a mitigação acordada é a documentação no portal (`[09:26] Marcos`).
- **Os "99% dos casos".** A afirmação do Diego de que at-least-once "resolve 99% dos casos" (`[09:25] Diego`) é uma estimativa opinativa, não uma métrica medida. Não deve ser usada como meta.
- **`event_id` no replay da DLQ.** Não foi definido se o replay ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) mantém o `event_id` original. Se um novo id for gerado, o cliente não consegue deduplicar um evento que talvez já tenha processado. Fica em aberto.
- **`event_id` versus id da outbox.** Não foi definido se o `event_id` é o próprio id (UUID) da linha da outbox (`[09:51] Larissa`) ou um campo separado. O detalhe fica para o FDD.
