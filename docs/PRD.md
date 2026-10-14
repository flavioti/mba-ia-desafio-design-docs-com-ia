# PRD — Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado para implementação |
| **Data** | 2026-10-13 |
| **Prazo-alvo** | Fim de novembro (pedido da Atlas Comercial), estimativa de 3 sprints |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS passa a **avisar os clientes B2B automaticamente, em até 10 segundos, sempre que o status de um pedido deles mudar**. O aviso é uma requisição HTTPS (webhook) enviada para um endereço que o próprio cliente cadastra. Hoje esses clientes só descobrem mudanças consultando a API repetidamente. Com a feature, a plataforma passa a avisá-los.

A demanda veio de um pedido formal de três clientes B2B: **Atlas Comercial, MaxDistribuição e Nova Cargo**. A solução reaproveita a infraestrutura e os padrões atuais da plataforma, sem componentes novos para operar além de um processo de entrega.

## 2. Problema e motivação

- **Integração lenta e cara.** Os clientes fazem polling periódico em `GET /orders` para descobrir mudanças de status. Isso deixa a integração lenta e cara para eles.
- **Risco comercial concreto.** A Atlas sinalizou que pode **migrar para um concorrente** se a funcionalidade não for entregue até o fim do trimestre.
- **Lacuna de produto.** A plataforma não tem nenhum mecanismo de notificação externa, evento ou fila. Hoje toda integração é puxada pelo cliente.

## 3. Público-alvo e cenários de uso

**Personas**

| Persona | Quem é | O que precisa |
| --- | --- | --- |
| **Integrador do cliente B2B** | Time técnico da Atlas, MaxDistribuição ou Nova Cargo, que recebe os webhooks no sistema próprio | Receber mudanças de status rápido, validar que o aviso veio da plataforma e não processar duplicatas |
| **Usuário da plataforma que representa o cliente** | Usuário autenticado da nossa API que cadastra e gerencia os webhooks do cliente | Cadastrar, editar, remover e consultar webhooks e o histórico de entregas |
| **Administrador da plataforma (ADMIN)** | Operação interna | Reprocessar entregas que falharam de vez, com rastro de auditoria |
| **Product Manager / portal do desenvolvedor** | Marcos | Documentar o contrato e as responsabilidades do cliente (verificar assinatura, deduplicar) |

**Cenários de uso**

1. **Assinatura seletiva.** A Atlas cadastra um endpoint `https` e informa que só quer saber quando o pedido vira `SHIPPED` ou `DELIVERED`. Recebe uma secret para validar os avisos. Mudanças para outros status não geram aviso.
2. **Aviso em tempo real.** Um pedido da MaxDistribuição passa de `PROCESSING` para `SHIPPED`. Em poucos segundos, o sistema dela recebe um aviso enxuto com o número do pedido, o status anterior, o status novo e o total. Se precisar dos itens, ela consulta o pedido pela API.
3. **Cliente em manutenção.** O endpoint da Nova Cargo fica fora do ar por 2 horas em manutenção planejada. A plataforma retenta automaticamente em intervalos crescentes, e o aviso chega quando o endpoint volta.
4. **Falha prolongada.** Um endpoint fica fora por mais de cerca de 15 horas. As tentativas se esgotam e o aviso fica guardado. Depois que o cliente se recupera, um ADMIN reprocessa o aviso.
5. **Vazamento de secret.** Um cliente vaza a secret em um log da aplicação dele. Ele pede uma nova pela API e tem 24 horas para atualizar os sistemas, período em que a secret antiga continua aceita.
6. **Aviso duplicado.** Por causa de um retry, o cliente recebe o mesmo aviso duas vezes. Ele identifica a duplicata pelo identificador único do evento e descarta.
7. **Diagnóstico.** O cliente quer entender por que não recebeu um aviso. Ele consulta as últimas 100 entregas do webhook, com sucesso ou falha, conteúdo enviado, resposta e tempo de resposta.

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| OBJ-1 | Entregar o aviso em "tempo real" na percepção do cliente | Tempo entre a mudança de status e o envio do aviso (`webhook_delivery_lag_ms`, FDD §9) | **< 10 segundos** no caminho feliz, com o endpoint do cliente disponível |
| OBJ-2 | Nenhuma mudança de status sem aviso correspondente | Mudanças de status assinadas e comitadas sem evento registrado | **0** (garantido por construção; ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md)) |
| OBJ-3 | Tolerar indisponibilidades longas do cliente | Janela coberta por retentativas automáticas antes da intervenção manual | **≈ 15 horas** (backoff 1m/5m/30m/2h/12h) |
| OBJ-4 | Atender aos clientes solicitantes no prazo | Clientes solicitantes com webhook disponível para integração | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) **até o fim de novembro** |
| OBJ-5 | Entregar dentro do esforço estimado | Sprints consumidas, incluindo a revisão de segurança | **≤ 3 sprints** |

A redução do polling em `GET /orders` é o efeito esperado, mas a reunião não fixou uma linha de base nem uma meta numérica. Fica para medir depois do lançamento, sem meta neste PRD.

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por cliente, com escolha dos status que interessam.
- Envio automático de aviso a cada mudança de status assinada.
- Assinatura dos avisos e secret exclusiva por webhook, com rotação.
- Identificador único por evento, para deduplicação no cliente.
- Retentativas automáticas, guarda dos avisos que falharam de vez e reprocessamento manual por ADMIN.
- Histórico das últimas 100 entregas por webhook.
- Documentação do contrato no portal do desenvolvedor.

### 5.2 Fora de escopo

| Item | Situação | Motivo |
| --- | --- | --- |
| **Notificação por e-mail** ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase | Medir o impacto da feature antes |
| **Dashboard visual** para o cliente ver os webhooks | **Descartado** desta feature | É projeto separado do time de frontend; agora, só endpoints |
| **Rate limiting** de envio por cliente | **Adiado** (observar e decidir depois) | Ainda não é um problema comprovado |
| **Webhooks de entrada** (cliente enviando para a plataforma) | **Descartado** | Os clientes só querem receber |
| **Garantia de ordem global** entre todos os pedidos | **Descartado** | Os clientes só precisam saber se cada pedido mudou; a ordem é garantida por pedido |
| **Entrega exatamente uma vez** (exactly-once) | **Descartado** | Complexidade desproporcional; o cliente deduplica |
| **Arquivamento** do histórico de eventos entregues | **Adiado** | Fora do escopo desta feature |
| **Escalar a entrega para vários processos em paralelo** | **Adiado** | Não é necessário agora |

## 6. Requisitos funcionais

| ID | Requisito | Prioridade |
| --- | --- | --- |
| RF-01 | O usuário cadastra um webhook para um cliente, informando a URL de destino e a lista de status que deseja receber. | Must |
| RF-02 | No cadastro, a plataforma **gera a secret** do webhook e a devolve na resposta. O cliente não escolhe a secret. | Must |
| RF-03 | O usuário lista os webhooks de um cliente. | Must |
| RF-04 | O usuário edita um webhook (URL, status assinados, ativo/inativo). | Must |
| RF-05 | O usuário remove um webhook. | Must |
| RF-06 | O cliente informa o cliente-alvo explicitamente na requisição, e não pelo token de login, porque o token identifica o usuário operador. | Must |
| RF-07 | A cada mudança de status de um pedido, a plataforma envia um aviso a cada webhook ativo do cliente que assina aquele status. Webhooks que não assinam o status não recebem nada. | Must |
| RF-08 | O aviso tem conteúdo enxuto: identificador do evento, tipo `order.status_changed`, data e hora, id e número do pedido, status anterior e novo, id do cliente e valor total. **Não inclui os itens.** | Must |
| RF-09 | O aviso reflete o pedido **no momento da mudança**, mesmo que o pedido seja alterado depois. | Must |
| RF-10 | Cada aviso é assinado de forma verificável pelo cliente, que confirma a origem e a integridade. | Must |
| RF-11 | Cada aviso traz um identificador único do evento, o mesmo em todas as tentativas, e o identificador do webhook de destino. | Must |
| RF-12 | O cliente pode rotacionar a secret pela API. A secret anterior continua válida por **24 horas** e depois deixa de valer. | Must |
| RF-13 | Avisos não confirmados pelo cliente são retentados automaticamente em intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h). | Must |
| RF-14 | Avisos que esgotam as tentativas ficam guardados, com o conteúdo e o motivo da falha. | Must |
| RF-15 | Um **ADMIN** pode reprocessar um aviso guardado, e a plataforma registra quem fez o reprocessamento. | Must |
| RF-16 | O usuário consulta as **últimas 100 entregas** de um webhook, com sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta. | Must |

## 7. Requisitos não funcionais

| ID | Categoria | Requisito |
| --- | --- | --- |
| RNF-01 | Desempenho | Aviso enviado **em menos de 10 s** após a mudança de status, com o cliente disponível. |
| RNF-02 | Confiabilidade | Não pode haver mudança de status assinada sem aviso correspondente. Se o registro do aviso falhar, a mudança de status também não acontece. |
| RNF-03 | Confiabilidade | Entrega **ao menos uma vez** (at-least-once). Duplicatas são possíveis e o cliente precisa estar preparado para elas. |
| RNF-04 | Confiabilidade | Um cliente lento ou fora do ar **não pode atrasar** mudanças de status de outros pedidos. |
| RNF-05 | Desempenho | O cliente tem **10 s** para responder a cada aviso; sem resposta, conta como falha e entra em retentativa. |
| RNF-06 | Segurança | A URL do webhook tem que ser **HTTPS**. Cadastros com `http` são recusados. |
| RNF-07 | Segurança | **HMAC-SHA256**, com secret **exclusiva por webhook**, sem secret global. |
| RNF-08 | Segurança | Reprocessamento restrito ao perfil **ADMIN**, com registro de auditoria. O cadastro é permitido a qualquer usuário autenticado nesta fase. |
| RNF-09 | Segurança | Revisão de segurança (HMAC e geração de secret) com **pelo menos 2 dias úteis**, antes do deploy. |
| RNF-10 | Capacidade | Avisos limitados a **64 KB**. Acima disso, o aviso não é enviado e é tratado como erro, nunca truncado. |
| RNF-11 | Ordem | Ordem garantida **por pedido**, e não globalmente. |
| RNF-12 | Operação | A entrega roda **isolada da API**: reiniciar uma não interrompe a outra. Sem infraestrutura nova além do banco existente. |
| RNF-13 | Manutenibilidade | Segue os padrões atuais do projeto (estrutura de módulo, erros com prefixo `WEBHOOK_`, logs estruturados). |

## 8. Decisões e trade-offs principais

| Decisão | O que ganhamos | O que aceitamos | Registro |
| --- | --- | --- | --- |
| Registrar o aviso junto com a mudança de status, no banco atual | Nenhum aviso perdido, nenhuma infraestrutura nova | A mudança de status depende do registro do aviso | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo de entrega separado, verificando novos avisos a cada 2 s | Isolamento de falhas e simplicidade | Até cerca de 2 s de espera e ordem garantida só por pedido | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| Retentativas por cerca de 15 h e depois guarda para reprocessamento manual | Tolera manutenções longas sem perder avisos | Em falhas, o aviso pode chegar com horas de atraso, e o reprocessamento é manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Assinatura com secret por webhook e rotação com 24 h | O cliente valida cada aviso, e um vazamento afeta um só webhook | Mais secrets para gerir | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega "ao menos uma vez", com identificador de evento | Simplicidade e padrão de mercado | O cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Entrega mais rápida e previsível | Herda limitações atuais, como a falta de vínculo entre usuário e cliente | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |

As alternativas descartadas e as questões em aberto estão consolidadas no [RFC §4 e §5](RFC.md#4-alternativas-consideradas).

## 9. Dependências

| Dependência | Responsável | Tipo |
| --- | --- | --- |
| Revisão de segurança antes do deploy (≥ 2 dias úteis) | Sofia | Interna, bloqueante para o go-live |
| Documentação do contrato, da verificação de assinatura e da deduplicação no portal do desenvolvedor | Marcos | Interna, bloqueante para a adoção |
| Confirmação de prazo e comunicação com os clientes | Marcos | Interna |
| Clientes implementarem a verificação HMAC e a deduplicação por identificador de evento | Clientes B2B | Externa |
| Endpoints dos clientes com HTTPS válido | Clientes B2B | Externa |
| Banco MySQL e stack atuais (sem infraestrutura nova) | Engenharia | Técnica |
| Painel visual (se vier a existir) | Time de frontend | Externa, fora desta feature |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R-1 | **Atraso na entrega leva à perda da Atlas** | Média | Alto | Escopo enxuto (sem e-mail, dashboard ou rate limiting), estimativa de 3 sprints validada na reunião, prazo confirmado com o cliente. |
| R-2 | **Cliente não deduplica e processa um aviso duas vezes** | Média | Médio | Identificador único estável entre tentativas; documentação em destaque no portal. |
| R-3 | **Vazamento de secret** (no cliente ou em logs nossos) | Baixa | Alto | Secret por webhook, rotação com 24 h, ocultação da secret nos logs, revisão de segurança. |
| R-4 | **Volume alto de avisos sobrecarrega o cliente** (por exemplo, 50 mudanças em 1 minuto) | Média | Médio | Monitorar o volume por cliente e decidir sobre rate limiting com dados (adiado). |
| R-5 | **Cliente fora do ar por mais de cerca de 15 h perde a janela automática** | Baixa | Médio | Avisos guardados com o motivo; reprocessamento por ADMIN. |
| R-6 | **Qualquer usuário autenticado gerencia webhooks de qualquer cliente** | Média | Médio | Aceito nesta fase; endurecimento de permissões previsto. |
| R-7 | **Ambiguidade na quantidade de tentativas** ("5 tentativas" com 5 intervalos) | Média | Baixo | Confirmar com Tech Lead e Plataforma antes da implementação (RFC QA-5). |

## 11. Critérios de aceitação

1. **Dado** um cliente com webhook ativo assinando `SHIPPED`, **quando** um pedido dele muda para `SHIPPED`, **então** o endpoint recebe um aviso assinado, com identificador do evento e do webhook, em menos de 10 s.
2. **Dado** um webhook que assina apenas `DELIVERED`, **quando** o pedido muda para `PAID`, **então** nenhum aviso é enviado.
3. **Dado** que o registro do aviso falha, **quando** a mudança de status é solicitada, **então** a mudança de status também não é efetivada.
4. **Dado** um cadastro com URL `http://`, **quando** é enviado, **então** é recusado com erro de validação.
5. **Dado** um cadastro bem-sucedido, **então** a resposta traz a secret gerada pela plataforma, e as consultas posteriores não a exibem.
6. **Dado** um endpoint que não responde em 10 s, **então** o aviso é retentado nos intervalos 1m/5m/30m/2h/12h e, esgotadas as tentativas, fica guardado com o motivo da falha.
7. **Dado** um aviso guardado, **quando** um usuário não ADMIN tenta reprocessá-lo, **então** recebe acesso negado. **Quando** um ADMIN reprocessa, **então** o aviso volta a ser enviado com o mesmo identificador de evento e o nome de quem reprocessou fica registrado.
8. **Dado** uma rotação de secret, **então** avisos assinados com a secret antiga continuam validáveis por 24 h, e só a nova vale depois disso.
9. **Dado** um webhook com entregas, **quando** o histórico é consultado, **então** retorna no máximo as 100 mais recentes, com sucesso ou falha, conteúdo, resposta e tempo de resposta.
10. **Dado** um aviso que passaria de 64 KB, **então** ele não é enviado e é tratado como erro.
11. **Dado** que a API é reiniciada, **então** as entregas em andamento continuam, e vice-versa.
12. A revisão de segurança da Sofia foi concluída antes do deploy.

## 12. Estratégia de testes e validação

- **Testes automatizados de integração** no padrão já usado pelo projeto (`tests/`, com supertest sobre a API e factories), cobrindo:
  - o CRUD;
  - a validação https;
  - as permissões ADMIN;
  - a atomicidade entre mudança de status e registro do aviso;
  - o filtro por status;
  - o histórico.
- **Testes da entrega contra um receptor simulado**, cobrindo assinatura válida, timeout, retentativas com relógio simulado, ida para a guarda de falhas, reprocessamento e deduplicação por identificador. A lista técnica completa está nos critérios de aceite do [FDD §12](FDD.md#12-critérios-de-aceite-técnicos).
- **Revisão de segurança dedicada** (Sofia, ≥ 2 dias úteis), com foco em HMAC e geração de secret, antes do deploy.
- **Validação com os clientes solicitantes.** O contrato fica publicado no portal antes do go-live, e Atlas, MaxDistribuição e Nova Cargo integram usando o portal como referência.
- **Validação pós-lançamento por métricas:**
  - latência até o envio (meta < 10 s);
  - taxa de sucesso das entregas;
  - volume de avisos que esgotam as tentativas;
  - volume de avisos por cliente e minuto, como insumo para decidir o rate limiting;
  - impacto geral, como insumo para decidir a fase de e-mail.
