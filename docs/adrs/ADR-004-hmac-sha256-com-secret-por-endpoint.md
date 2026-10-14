# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito, sujeito à revisão de segurança antes do deploy (`[09:46] Sofia`)
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:22] Sofia`)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **ADRs relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Os webhooks enviam dados de pedidos para endpoints fora da infraestrutura da empresa. O cliente precisa conseguir verificar duas coisas: que a requisição veio de fato da plataforma, e que o payload não foi adulterado no caminho (`[09:19] Sofia`). A plataforma já teve um cliente que vazou a secret em log da própria aplicação (`[09:22] Diego`).

Hoje não existe nenhum mecanismo de assinatura de saída no código. A autenticação existente (`src/middlewares/auth.middleware.ts`) é de entrada, por JWT, e o redact do logger Pino (`src/shared/logger/index.ts`) cobre `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`. **Não há regra de redact para `secret`.**

## Decisão

- **D1.** **Assinatura HMAC-SHA256 sobre o corpo da requisição**, enviada no header `X-Signature`. O cliente verifica a assinatura do lado dele (`[09:20] Sofia`, decidido em `[09:22] Sofia`).
- **D2.** **Uma secret única por endpoint de webhook**, e não uma secret global da plataforma (`[09:21] Sofia`).
- **D3.** A configuração de webhook guarda **url, secret, customer_id e estado ativo** (`[09:21] Bruno`, confirmado em `[09:21] Sofia`).
- **D4.** A **secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook (`[09:31] Marcos`). O cliente não envia a secret.
- **D5.** **Rotação de secret via API.** O cliente pede uma nova secret por endpoint, e a antiga continua válida **por 24 horas em paralelo**, para que ele migre os próprios sistemas. Depois disso, a antiga é invalidada (`[09:21] Sofia`).
- **D6.** O código de HMAC e de geração de secret passa por **revisão de segurança da Sofia, com pelo menos 2 dias úteis reservados antes do deploy** (`[09:46] Sofia`, `[09:47] Larissa`).

Restrição complementar que não é decisão arquitetural: a URL do webhook **tem que ser `https`**, e um cadastro com `http` é recusado com erro de validação no schema Zod (`[09:23] Sofia`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **Secret global da plataforma** para todos os endpoints | Se uma vazar, vazam todas (`[09:21] Sofia`). |
| A2 | **Secret fixa, sem rotação**, ou troca imediata sem convivência | O cliente precisa conseguir trocar a secret depois de um vazamento (`[09:22] Diego`) e precisa de tempo para migrar os sistemas dele. Daí as 24 h de convivência (`[09:21] Sofia`). |

## Consequências

**Positivas**

- **C+1.** **Autenticidade e integridade verificáveis pelo cliente.** O cliente usa bibliotecas padrão de mercado, já que "todo cliente sério tem biblioteca pra isso" (`[09:20] Sofia`).
- **C+2.** **Raio de impacto limitado.** Um vazamento compromete um único endpoint (D2).
- **C+3.** **Rotação sem downtime** para o cliente, graças ao grace period (D5).

**Negativas**

- **C−1.** **A secret tem que ser recuperável.** Ao contrário da senha de usuário, guardada como hash bcrypt (`src/modules/users/user.service.ts`), o HMAC precisa da secret original na hora de assinar, então ela **não pode ser guardada apenas como hash**. A forma de armazenamento em repouso não foi discutida e fica como questão em aberto.
- **C−2.** **Mais estado para gerir.** Durante as 24 h de rotação, existem duas secrets válidas por endpoint.
- **C−3.** **Risco de vazamento em log.** Sem ajuste, a secret pode aparecer nos logs: os `redactPaths` de `src/shared/logger/index.ts` não cobrem campos `secret`. Isso é relevante dado o histórico de vazamento citado em `[09:22] Diego`.
- **C−4.** **A secret precisa ser bem guardada pelo cliente.** Ela é devolvida pela API (D4), e o cliente fica responsável por armazená-la com segurança.

**Trade-off explícito:** aceitamos a complexidade de gerir secrets individuais, recuperáveis e rotacionáveis em troca de isolar o impacto de vazamentos e permitir que o cliente valide cada entrega.

## Divergências e pontos de atenção da transcrição

- **Quem gera a secret.** O Bruno sugere o código de erro `WEBHOOK_SECRET_REQUIRED` (`[09:28] Bruno`), que só faria sentido se o cliente enviasse a secret. Logo depois o PM define que "secret é gerada pela gente e devolvida na criação" (`[09:31] Marcos`). **Prevalece a geração pelo servidor (D4)**, e `WEBHOOK_SECRET_REQUIRED` não é adotado.
- **Qual secret assina durante o grace period.** Não foi definido se o envio é assinado com a secret nova, com as duas (por exemplo, duas assinaturas no header) ou com qual delas. Fica em aberto.
- **`X-Timestamp` fora da assinatura.** A assinatura é sobre o **corpo** (`[09:22] Sofia`). O `X-Timestamp`, pensado para o cliente detectar replay attack (`[09:44] Diego`), vai num header que não é coberto pela assinatura e por isso pode ser alterado no caminho. Fica em aberto se o timestamp deve entrar no material assinado; isso pode ser tratado na revisão de segurança (D6).
- **Retorno da secret.** Só ficou definido o retorno na criação (`[09:31] Marcos`). Não foi decidido se ela pode ser consultada depois.
