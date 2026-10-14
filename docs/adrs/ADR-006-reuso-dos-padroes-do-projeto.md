# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data da decisão:** reunião técnica de webhooks (decidido em `[09:30] Larissa`)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (PM)
- **ADRs relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

O OMS tem uma estrutura consolidada:

- **Módulos por domínio:** `src/modules/{auth,users,customers,products,orders}`, cada um com `*.controller.ts`, `*.service.ts`, `*.repository.ts` (exceto `auth`), `*.routes.ts` e `*.schemas.ts`. A composição é manual em `src/app.ts` (`buildControllers`), e as rotas são montadas sob `/api/v1` em `src/routes/index.ts`.
- **Erros:** a classe base é `AppError(message, statusCode, errorCode, details)`, em `src/shared/errors/app-error.ts`. As subclasses ficam em `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, `UnprocessableEntityError`...), e há erros de domínio como `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (`INSUFFICIENT_STOCK`).
- **Middleware de erro centralizado:** `src/middlewares/error.middleware.ts` trata `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002` e `P2025`), sempre no envelope `{ error: { code, message, details? } }`.
- **Validação:** schemas Zod aplicados pelo middleware `validate` (`src/middlewares/validate.middleware.ts`).
- **Autenticação e autorização:** `authenticate` e `requireRole(...)`, em `src/middlewares/auth.middleware.ts`, com as roles `ADMIN | OPERATOR`. Um exemplo de uso fica em `src/modules/users/user.routes.ts`.
- **Logs:** logger Pino em `src/shared/logger/index.ts` e log de requisição com `X-Request-Id` em `src/middlewares/request-logger.middleware.ts`.
- **Identificadores:** UUID `Char(36)` em todos os models (`prisma/schema.prisma`).

O time é pequeno (`[09:07] Diego`), e o objetivo é que o webhook não introduza um estilo novo.

## Decisão

**Reuso máximo do que já existe.** O webhook vira um módulo igual aos outros (`[09:30] Larissa`):

- **D1.** **Módulo `src/modules/webhooks/` (novo, a criar)** com controller, service, repository, routes e schemas, seguindo o padrão dos demais domínios (`[09:27] Bruno`). O worker segue a [ADR-002](ADR-002-worker-separado-em-polling.md): entry em `src/worker.ts` e a lógica dentro do módulo (`[09:28] Bruno`).
- **D2.** **Erros como subclasses de `AppError`**, com códigos em UPPER_SNAKE_CASE e **prefixo `WEBHOOK_`** para tudo do módulo, por exemplo `WEBHOOK_NOT_FOUND` e `WEBHOOK_INVALID_URL` (`[09:28] Bruno`, `[09:29] Larissa`).
- **D3.** **Logger Pino existente**, sem nenhuma biblioteca nova de log (`[09:29] Bruno`).
- **D4.** **Middleware de erro centralizado sem alterações.** Ele já trata `AppError`, Zod e Prisma (`[09:29] Bruno`).
- **D5.** **Schemas Zod** para validação de entrada (`[09:30] Larissa`), incluindo a regra de URL `https` (`[09:23] Sofia`).
- **D6.** **Autorização com os middlewares existentes.** O CRUD de configuração exige qualquer usuário autenticado (`authenticate`) (`[09:37] Sofia`), e o replay da DLQ exige `requireRole('ADMIN')` (`[09:36] Larissa`).
- **D7.** **O `customer_id` não vem do JWT.** O JWT é do usuário operador, não do cliente (`[09:32] Bruno`), e a Larissa decide que o `customer_id` é passado no body ou no path (`[09:32] Larissa`). **Escolha registrada: no path**, aninhado ao recurso de clientes (`/customers/:customerId/webhooks`), seguindo o padrão REST de `:id` em path já usado nos módulos.
- **D8.** **Chaves primárias UUID** nas novas tabelas (`[09:51] Larissa`).

## Alternativas Consideradas

| # | Alternativa | Por que foi descartada |
| --- | --- | --- |
| A1 | **`customer_id` extraído do JWT** (proposta inicial de `[09:31] Marcos`) | O JWT atual identifica o usuário operador, não o cliente (`[09:32] Bruno`). O payload do token em `src/middlewares/auth.middleware.ts` só traz `sub`, `email` e `role`. |
| A2 | **Introduzir novas bibliotecas ou estruturas próprias** para o módulo (outro logger, outra camada de erros) | "Não vamos botar nada novo" (`[09:29] Bruno`). Divergir do padrão aumenta a curva de manutenção para um time pequeno (`[09:07] Diego`). |
| A3 | **Compartilhar a mesma instância de `PrismaClient` entre API e worker** (dúvida levantada em `[09:29] Diego`) | Inviável. O `PrismaClient` é por processo, então o worker cria a própria instância com a mesma `DATABASE_URL` (`[09:30] Bruno`). |

## Consequências

**Positivas**

- **C+1.** **Curva de aprendizado mínima.** O módulo é previsível para qualquer pessoa do time.
- **C+2.** **Erros sem mudar o middleware.** Os erros do módulo chegam ao cliente no mesmo envelope `{ error: { code, message, details? } }`, sem mudanças em `error.middleware.ts` (`[09:29] Bruno`).
- **C+3.** **Autorização reaproveitada.** A autorização usa `requireRole` sem código novo (`[09:36] Larissa`).
- **C+4.** **Nenhuma dependência nova** em `package.json`.

**Negativas**

- **C−1.** **O middleware de erro não cobre o worker.** O `errorMiddleware` é um `ErrorRequestHandler` do Express, então **os erros no worker não passam por ele**. O worker precisa tratar e registrar as próprias falhas com o Pino.
- **C−2.** **O worker não tem `X-Request-Id`.** O `X-Request-Id` do `request-logger.middleware.ts` só existe no ciclo HTTP; para correlacionar logs, o worker depende de outro identificador, como o `event_id` da [ADR-005](ADR-005-at-least-once-com-x-event-id.md).
- **C−3.** **Autorização frouxa no CRUD.** Qualquer usuário autenticado pode gerenciar webhooks de qualquer cliente, porque o model `User` (`prisma/schema.prisma`) não tem vínculo com `Customer`. Foi aceito "por enquanto", com possível endurecimento no futuro (`[09:37] Sofia`).
- **C−4.** **O redact de logs precisa ser estendido.** O padrão de redact do Pino precisa cobrir a secret do webhook ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

**Trade-off explícito:** priorizamos consistência e velocidade de entrega, aproveitando a infraestrutura de código existente, em vez de soluções especializadas, ao custo de herdar limitações atuais como a autorização sem vínculo entre usuário e cliente e um tratamento de erro centrado em HTTP.

## Divergências e pontos de atenção da transcrição

- **Origem do `customer_id`.** O PM propõe tirá-lo do JWT (`[09:31] Marcos`). O Bruno aponta que o JWT é do operador (`[09:32] Bruno`), e o Marcos responde que "a gente tem usuários que representam o cliente" (`[09:32] Marcos`). **No código não existe vínculo entre `User` e `Customer`** (`prisma/schema.prisma`), então a afirmação não tem respaldo no modelo atual. A decisão final da Larissa, body ou path (`[09:32] Larissa`), é a que vale. Este ADR fixa o path (D7).
- **"Vai pegar nossos erros sem precisar mudar nada".** A frase (`[09:29] Bruno`) vale para a API. Para o worker, que não passa pelo Express, ela não se aplica (C−1).
- **`WEBHOOK_SECRET_REQUIRED`.** O código citado em `[09:28] Bruno` não é adotado, porque a secret é gerada pelo servidor ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- **Erros de status repetido.** Uma inconsistência do código já existente, não da transcrição: o `changeStatus` usa um `ConflictError` genérico com o código `INVALID_STATUS_TRANSITION` para um status repetido (`src/modules/orders/order.service.ts`), em vez da subclasse `InvalidStatusTransitionError`. Como recomendação, sem ter sido decidido na reunião, os novos erros `WEBHOOK_*` podem seguir o padrão de **subclasse dedicada**, como `InsufficientStockError`, em vez de repetir essa forma.
