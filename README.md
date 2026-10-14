# Da Reunião ao Documento: design docs do Sistema de Webhooks

> O enunciado original do desafio está preservado em [`README.old.md`](README.old.md).

## Sobre o desafio

Uma empresa que opera um Order Management System (OMS) em Node.js, TypeScript, Prisma e MySQL decidiu, em uma reunião técnica de cerca de 55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**: avisar clientes B2B, em menos de 10 segundos, sempre que o status de um pedido muda. A única fonte dessa decisão era a transcrição literal da call ([`TRANSCRICAO.md`](TRANSCRICAO.md)). O código da aplicação, em `src/` e `prisma/`, não tem nenhum mecanismo de eventos, filas ou notificação.

A tarefa foi transformar essas duas fontes em um pacote de design docs acionável: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade. **Nada podia ser inventado:** cada requisito, decisão ou restrição precisava apontar para uma fala da reunião (`[hh:mm] Nome`) ou para um arquivo real do código. Uma parte central do trabalho foi separar o que foi decidido do que foi descartado ou adiado e expor as contradições da própria transcrição, em vez de escondê-las.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (modelo Claude Opus 5.5), extensão do VS Code** | Ferramenta principal. Leu todo o código e a transcrição, levantou dúvidas e divergências, redigiu todos os documentos e manteve o Tracker. |
| **Ferramenta de perguntas estruturadas do Claude Code** (`AskUserQuestion`) | Pôs as decisões ambíguas na mão do autor antes de redigir. Exemplos: a contagem de tentativas, a origem da secret e o lugar do `customer_id`. |
| **Scripts Python gerados pela IA, rodados no terminal** | Validação automática do Tracker a cada documento: cada `[hh:mm] Nome` precisa existir na transcrição, cada caminho de código precisa existir no repositório, não pode haver ID duplicado e a proporção TRANSCRICAO/CODIGO é calculada. |
| **Memória persistente do Claude Code** | Guardou preferências do autor entre etapas, como "nenhum diagrama Mermaid". |

## Workflow adotado

A ordem seguiu a recomendada pelo enunciado: decisões primeiro, consolidação depois.

1. **Contextualização.** A IA leu `src/`, `prisma/schema.prisma`, a migration, `tests/` e a transcrição inteira, e devolveu um mapa do código: estrutura de módulos, `changeStatus` transacional, `AppError`, `requireRole`, error middleware e Pino. Devolveu também uma lista de decisões, descartes e adiamentos, além de 7 dúvidas abertas. O pedido era explícito: **pausar e perguntar em caso de dúvida.**
2. **Preparação.** O `README.md` original foi renomeado para `README.old.md` (`git mv`), para preservar o enunciado.
3. **ADRs primeiro.** Antes de escrever, a IA mapeou as divergências da transcrição e fez 4 perguntas de decisão (detalhe em "Iterações e ajustes"). Com as respostas, produziu 6 ADRs no formato MADR, com uma seção extra, "Divergências e pontos de atenção da transcrição", e abriu o `TRACKER.md`.
4. **RFC.** Consolidou as ADRs em uma proposta de 2 a 4 páginas. As 11 alternativas descartadas e as 14 questões em aberto ganharam lugar próprio, separando as que **foram levantadas na reunião** das que **foram identificadas na consolidação**.
5. **FDD.** É o desenho de implementação: modelo de dados, fluxos, 7 endpoints, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente", com 18 arquivos reais. Detalhes que a reunião não cobriu receberam marcações explícitas: **(proposta)** ou **(provisório — QA-n)**.
6. **Ajuste de forma.** Todos os diagramas Mermaid foram removidos e substituídos por listas numeradas.
7. **PRD.** É a consolidação no nível de produto, escrita por último entre os documentos grandes, em linguagem de negócio e com links para RFC, ADRs e FDD em vez de repetir detalhes.
8. **README.** É este documento.

**Organização da interação com a IA:**
- Um documento por rodada.
- Ao final de cada rodada, as linhas correspondentes eram acrescentadas ao Tracker e o script de validação rodava antes de seguir.
- A IA nunca fez commit sozinha.

## Prompts customizados

Os prompts abaixo seguem a mesma estrutura: papel, entradas, tarefa, restrições, formato de saída, regras do Tracker e condição de parada. As regras de rastreabilidade e de validação se repetem em todos eles. Assim, a mesma disciplina contra alucinação vale para todos os documentos.

**1. Contextualização (somente leitura)**

```text
Você é um arquiteto de software sênior ajudando a produzir design docs de uma feature.

## Entradas
- Código da aplicação: src/, prisma/schema.prisma, prisma/migrations/, tests/, package.json
- Transcrição da reunião: TRANSCRICAO.md (formato "[hh:mm] Nome: fala")
- Enunciado e critérios de aceite: README.old.md

## Tarefa (NÃO escreva nenhum documento ainda)
1. Mapeie o código e cite arquivo e linha para cada item:
   - estrutura de módulos e composição;
   - máquina de estados;
   - transação do changeStatus;
   - classes e códigos de erro;
   - middlewares (auth, requireRole, validate, error);
   - logger Pino;
   - convenções de ID e de rotas.
2. Classifique tudo que foi dito na reunião em uma tabela com as colunas
   | Item | Categoria | Quem/quando ([hh:mm] Nome) |, usando as categorias:
   DECIDIDO | DESCARTADO (com motivo) | ADIADO | EM ABERTO | DETALHE SECUNDÁRIO.
3. Liste as DIVERGÊNCIAS de três tipos:
   a) falas que se contradizem entre si;
   b) falas que contradizem o código;
   c) contas ou números que não fecham (ex.: tentativas x intervalos x janela total).
4. Liste as LACUNAS: o que a implementação vai exigir e a reunião não definiu.

## Restrições
- Não invente nada. Se algo não tem origem na transcrição ou no código, marque como lacuna.
- Não trate sugestão como decisão: só é DECIDIDO o que alguém fechou explicitamente.

## Saída
As tabelas acima e, no fim, uma lista numerada de perguntas para eu decidir.
Pare e aguarde minhas respostas.
```

**2. ADRs**

```text
Com base no mapeamento e nas minhas respostas às dúvidas, produza os ADRs.

## Escopo
- Um arquivo por decisão em docs/adrs/, nomeado ADR-NNN-titulo-em-kebab-case.md
  (siga o README.old.md, não o docs/adrs/README.md).
- Cubra as 6 decisões principais: Outbox no MySQL; worker separado em polling;
  retry com backoff + DLQ; HMAC-SHA256 com secret por endpoint; at-least-once com
  X-Event-Id; reuso dos padrões do projeto.
- Detalhes secundários (payload, headers, timeouts) NÃO viram ADR; ficam para o FDD.

## Estrutura de cada ADR (MADR)
1. Status, data e decisores (participantes da reunião).
2. Contexto, citando o problema e os arquivos de código envolvidos.
3. Decisão, em itens numerados D1, D2…, cada um com a citação `[hh:mm] Nome`.
4. Alternativas Consideradas: tabela com a alternativa e o motivo do descarte
   (preferir alternativas discutidas de fato na reunião).
5. Consequências: positivas (C+n) e negativas (C−n), mais uma linha de "Trade-off explícito".
6. Divergências e pontos de atenção da transcrição: contradições, contas que não fecham
   e pontos não decididos ligados a esta decisão.

## Restrições
- Toda afirmação deve ter citação `[hh:mm] Nome` ou caminho de arquivo real.
- Arquivo que ainda não existe deve aparecer marcado "(a criar)".
- Recomendação sua, não decidida na reunião, deve ser escrita explicitamente como recomendação.
- Pelo menos um ADR deve referenciar classes e arquivos reais do código.

## Tracker
Abra docs/TRACKER.md com a tabela
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
- IDs no formato ADR-NNN-{CTX|D|ALT|CON|DIV}-NN.
- Uma origem por linha; para um item com duas origens, use o sufixo a/b.
- Fonte = TRANSCRICAO (Localização `[hh:mm] Nome`) ou CODIGO (caminho real).

## Validação antes de me entregar
Rode um script que confira se:
- cada `[hh:mm] Nome` existe na transcrição;
- cada caminho de código existe;
- não há IDs duplicados;
- ao menos 70% das linhas têm fonte TRANSCRICAO.
Mostre o resultado.
```

**3. RFC**

```text
Produza docs/RFC.md: a proposta técnica submetida à equipe para revisão.

## Altura
Arquitetura. De 2 a 4 páginas. Responde "o que propomos e por quê".
Não inclua payloads, schemas, status codes nem códigos de erro: isso é do FDD.

## Seções obrigatórias
1. Metadados: autor (quem se comprometeu a abrir o doc na reunião), status, data e
   revisores (os participantes).
2. TL;DR em até 5 linhas.
3. Contexto e problema: o motivo de negócio e o ponto do código afetado.
4. Proposta técnica: componentes e responsabilidades, cada um com link para seu ADR.
5. Alternativas consideradas: tabela | Alternativa | Trade-off do descarte | ADR |,
   com TODAS as alternativas descartadas na reunião (mínimo 2).
6. Questões em aberto, em duas tabelas separadas:
   a) levantadas na reunião e adiadas ou não decididas;
   b) lacunas identificadas na consolidação, cada uma com o revisor sugerido.
7. Impacto (código, banco, operação, clientes, prazo) e riscos com mitigação.
8. Decisões relacionadas: links para todos os ADRs.

## Restrições
- Não repita o conteúdo dos ADRs: resuma em uma linha e linke.
- Toda afirmação precisa ter origem rastreável.

## Tracker
- Acrescente a seção "## RFC" com IDs RFC-{META|TLDR|CTX|PROP|ALT|QA|IMP|RISK}-NN.
- Uma linha só aceita fonte CODIGO se o caminho for de código; nunca use um ADR como fonte.
- Rode o mesmo script de validação e mostre o resultado.
```

**4. FDD**

```text
Produza docs/FDD.md: a especificação que um desenvolvedor usa para começar a codar.

## Base
ADRs e RFC já aprovados. Não reabra decisões.
Para cada questão em aberto do RFC, implemente um default marcado "(provisório — QA-n)"
e isolado em constante ou configuração.
Detalhes que a reunião não cobriu devem ser marcados "(proposta)".

## Seções obrigatórias
1. Contexto e motivação técnica.
2. Objetivos técnicos, cada um com critério verificável.
3. Escopo e exclusões.
4. Modelo de dados: models Prisma no padrão de prisma/schema.prisma.
5. Fluxos detalhados:
   criação do evento na outbox dentro do changeStatus, processamento pelo worker,
   retry com tabela de tempos e DLQ/replay.
6. Contratos públicos: no mínimo 4 endpoints, cada um com método, path, request e
   response de exemplo em JSON, headers e todos os status codes.
   Inclua também o contrato da requisição de saída enviada ao cliente.
7. Matriz de erros com códigos WEBHOOK_*, HTTP, classe base, onde ocorre e se há retry.
   Verifique no código se as classes existentes aceitam código customizado.
8. Resiliência: timeouts, retries, backoff e fallback.
9. Observabilidade: métricas, logs (nomes de evento e campos) e tracing.
   Use o que existe no projeto; se algo não existir, diga e proponha uma alternativa
   sem dependências novas.
10. Dependências e compatibilidade.
11. Integração com o sistema existente: tabela | arquivo real | como integra |,
    com no mínimo 4 arquivos que existam de fato.
12. Critérios de aceite técnicos, testáveis.
13. Riscos e mitigação (probabilidade e impacto).

## Restrições
- Ao detalhar, procure conflitos entre decisões e regras do código
  (ex.: uma validação que, feita dentro da transação, derrubaria a mudança de status).
  Registre cada conflito e a resolução adotada.
- Não altere nenhum arquivo em src/, prisma/ ou tests/.

## Tracker
- Acrescente a seção "## FDD" com IDs
  FDD-{CTX|OBJ|ESC|DM|FLX|CONTRATO|ERR|RES|OBS|DEP|INT|AC|RISK}-NN.
- Propostas apontam para a fala mais próxima que as motivou.
- Valide com o script.
```

**5. PRD**

```text
Produza docs/PRD.md consolidando RFC, ADRs e FDD na altura de produto e negócio.

## Seções obrigatórias
Resumo e contexto; Problema e motivação; Público-alvo e cenários de uso;
Objetivos e métricas de sucesso; Escopo (incluso e fora de escopo);
Requisitos funcionais; Requisitos não funcionais; Decisões e trade-offs principais;
Dependências; Riscos e mitigação; Critérios de aceitação; Estratégia de testes e validação.

## Regras
- Linguagem de produto: sem nomes de tabelas, headers ou códigos de erro.
  Para detalhes, linke o RFC, os ADRs ou o FDD.
- No mínimo 8 requisitos funcionais, numerados RF-NN, todos ditos na reunião.
- Métricas apenas com metas numéricas que tenham origem na reunião.
  Onde não houver meta, escreva isso em vez de inventar um número.
- "Fora de escopo" em tabela | Item | Descartado ou Adiado | Motivo |, no mínimo 2 itens.
- Riscos com probabilidade, impacto e mitigação (mínimo 2).
- Critérios de aceitação no formato Dado/Quando/Então.

## Antes de entregar
Releia cada frase do Problema e da Motivação e remova qualquer afirmação sem origem.
Acrescente a seção "## PRD" ao Tracker (IDs PRD-{CTX|PROB|CEN|OBJ|ESC|FR|NFR|DEC|DEP|RISK|AC|TEST}-NN)
e valide.
```

**Script de validação do Tracker**, rodado a cada documento:

```python
for r in rows:  # cada linha do TRACKER.md
    c = [x.strip() for x in r.strip().strip('|').split('|')]
    if c[4] == 'TRANSCRICAO':
        m = re.fullmatch(r'`\[(\d\d:\d\d)\] (\w+)`', c[5])
        if not m or f'[{m.group(1)}] {m.group(2)}:' not in transcricao:
            bad.append(c[0])          # timestamp/falante que não existe
    elif not os.path.exists(c[5].strip('`')):
        bad.append(c[0])              # caminho de código inexistente
```

## Iterações e ajustes

Foram **7 ciclos principais** (contexto, ADRs, RFC, FDD, remoção dos diagramas, PRD e README). Neles, estes foram os momentos em que a saída da IA precisou ser corrigida ou decidida pelo autor:

1. **Ambiguidades da transcrição decididas pelo autor, e não pela IA.** Antes das ADRs, a IA levantou 4 pontos:
   - **Retry:** "5 tentativas" com 5 intervalos (1m/5m/30m/2h/12h) dão cerca de 14,6 h só se forem 1 envio mais 5 retries. → **Decidido deixar em aberto** (RFC QA-5), com `MAX_DELIVERY_ATTEMPTS` isolado no FDD.
   - **Secret:** o código `WEBHOOK_SECRET_REQUIRED` (`[09:28] Bruno`) contradiz "a secret é gerada pela gente" (`[09:31] Marcos`). → **Secret gerada pelo servidor**, código descartado.
   - **`customer_id`:** pode ir no body ou no path, e o JWT é do operador. → **Path.**
   - **Quantidade de ADRs:** a IA propôs 7, incluindo uma sobre snapshot e filtro. → **O autor reduziu para as 6 principais.** Snapshot e filtro foram para o FDD.
2. **Decisão inventada rebaixada a recomendação.** Na ADR-006, a IA escreveu que os novos erros "devem" seguir subclasse dedicada. A reunião não decidiu isso, então o texto foi reescrito como recomendação explícita.
3. **O validador do Tracker pegou referências problemáticas.**
   - `src/worker.ts` e `src/modules/webhooks/` apareciam como caminhos "inexistentes". Ainda não existem porque serão criados, e passaram a ser marcados "(a criar)".
   - Uma linha `RFC-REL-01` apontava um ADR como fonte `CODIGO`. Foi removida.
4. **Conflitos descobertos ao detalhar o FDD**, registrados em vez de escondidos:
   - O limite de 64 KB, se validado na inserção, derrubaria a mudança de status, violando "status muda e evento sai". A checagem foi movida para o worker.
   - O middleware `validate` sempre devolve `VALIDATION_ERROR`, então `WEBHOOK_INVALID_URL` vai dentro de `details`.
   - `NotFoundError` fixa o código `NOT_FOUND`, então `WEBHOOK_NOT_FOUND` exige uma subclasse própria.
5. **Remoção dos diagramas Mermaid**, a pedido do autor. Ao substituí-los por texto, ficou claro que alguns detalhes só existiam no diagrama, como a query de leitura do worker e a marcação `PROCESSING`. Eles foram reescritos nas listas de passos. A preferência foi salva na memória da IA.
6. **Frase sem origem cortada do PRD.** A IA escreveu que o polling "gera carga desnecessária para nós", o que a reunião não diz (o Marcos fala em integração lenta e cara **para os clientes**). A frase foi removida. Pelo mesmo motivo, a redução do polling ficou sem meta numérica.
7. **Lacunas viraram questões em aberto, e não requisitos.** Alguns pontos que a reunião não discutiu foram tratados assim:
   - ordem de entrega com retry em andamento;
   - `X-Timestamp` fora da assinatura;
   - qual secret assina durante as 24 h de rotação;
   - armazenamento da secret em repouso;
   - `event_id` no replay;
   - evento na criação do pedido.

   Todos foram para o RFC (QA-8 a QA-14), cada um com um revisor sugerido. No FDD, quando foi preciso um default para implementar, ele aparece como **(provisório — QA-n)**.

## Como navegar a entrega

```
.
├── README.md                 ← este arquivo (processo)
├── README.old.md             ← enunciado original do desafio
├── TRANSCRICAO.md            ← fonte: reunião técnica (inalterada)
└── docs/
    ├── PRD.md                ← por quê e o quê (produto/negócio)
    ├── RFC.md                ← proposta, alternativas e questões em aberto (arquitetura)
    ├── adrs/
    │   ├── ADR-001-outbox-no-mysql.md
    │   ├── ADR-002-worker-separado-em-polling.md
    │   ├── ADR-003-retry-com-backoff-e-dlq.md
    │   ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
    │   ├── ADR-005-at-least-once-com-x-event-id.md
    │   └── ADR-006-reuso-dos-padroes-do-projeto.md
    ├── FDD.md                ← como construir (implementação)
    └── TRACKER.md            ← origem de cada item (transcrição ou código)
```

**Ordem sugerida de leitura**

1. [`docs/PRD.md`](docs/PRD.md): o problema, quem é afetado, o escopo e as metas.
2. [`docs/RFC.md`](docs/RFC.md): a proposta em 2 a 4 páginas, as alternativas descartadas e o que ainda está em aberto.
3. [`docs/adrs/`](docs/adrs/): o porquê de cada decisão, com as divergências da transcrição registradas em cada uma.
4. [`docs/FDD.md`](docs/FDD.md): o desenho para implementar. A §11 cobre a integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md): para auditar qualquer item. É agrupado por documento, e cada linha aponta para `[hh:mm] Nome` ou para um arquivo real.

**Convenções usadas nos documentos**

- `[hh:mm] Nome`: citação da transcrição.
- **(proposta)**: detalhe de implementação que a reunião não discutiu, derivado das decisões e do código.
- **(provisório — QA-n)**: default para a questão em aberto `QA-n` do RFC.
- **(a criar)**: arquivo novo, que ainda não existe no repositório.

O código da aplicação (`src/`, `prisma/`, `tests/` e as configurações) **não foi alterado**. A entrega é puramente documental.
