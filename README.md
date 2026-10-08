# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

Entrega do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"** (MBA IA Full Cycle). O enunciado original está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

A tarefa é transformar a transcrição de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`) em um pacote de documentação para uma feature nova de um Order Management System em Node.js/TypeScript + Prisma/MySQL: um **sistema de webhooks que notifica clientes B2B quando o status de um pedido muda**. Na reunião, tech lead, PM, dois engenheiros e a engenheira de segurança fecharam as decisões (outbox no MySQL, worker em polling, retry com DLQ, HMAC, at-least-once). Também descartaram alternativas e deixaram pontos em aberto.

O pacote tem PRD, RFC, FDD, 7 ADRs e um tracker de rastreabilidade. A regra mais importante é que **tudo precisa ter origem identificável**, na transcrição (`[hh:mm] Falante`) ou no código. O código da aplicação não foi alterado. A entrega é só documental.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (modelo Claude Opus, no terminal, com acesso ao repositório) | Ferramenta principal. Leu a transcrição inteira e os arquivos de código relevantes (`order.service.ts`, `order.status.ts`, erros, middlewares, `schema.prisma`, testes) e redigiu todos os documentos. |
| **Subagentes do Claude Code** | Usados como revisores independentes, sem acesso ao "raciocínio" de quem escreveu: um fez revisão adversarial dos documentos contra a transcrição e o código; outro classificou a transcrição do zero, sem ler os docs, para comparação cruzada. |
| **Scripts Python de validação** (gerados com o Claude Code) | Checagem mecânica anti-alucinação: todo `[hh:mm] Nome` citado existe na transcrição com aquele falante; todo caminho de arquivo citado existe no repo; todo ID usado nos docs tem linha no tracker; proporção de fontes do tracker. |

## Workflow adotado

1. **Setup.** O meu repositório não tinha sido criado como fork e estava vazio. Fiz merge do repositório base (`--allow-unrelated-histories`) para trazer código, transcrição e templates sem perder o histórico original.
2. **Contextualização.** Leitura completa da `TRANSCRICAO.md` e do código, com foco nos pontos de integração: a transação de `changeStatus`, a máquina de estados, `AppError`/`http-errors`, `validate`/`error`/`auth` middlewares, logger, config e schema Prisma.
3. **ADRs primeiro**, um por decisão principal (as 6 do enunciado), mais um ADR para "snapshot do payload + filtro na inserção", que foi fechado no fim da call.
4. **RFC** em cima dos ADRs, curto, com alternativas descartadas e questões em aberto.
5. **FDD** com modelo de dados, fluxos, contratos, matriz de erros e a seção de integração com arquivos reais. Convenção adotada: tudo que é detalhe de implementação **não fechado na reunião** fica marcado como **(proposta do FDD)** e aponta para a decisão de onde foi derivado.
6. **PRD** por último entre os grandes, como consolidação de produto.
7. **Tracker**, montado a partir dos IDs que já estavam dentro dos documentos (`PRD-FR-03`, `RFC-ALT-02`, `FDD-CONTRATO-05`...), para que a referência cruzada funcionasse nos dois sentidos.
8. **Validação automática** por script (citações, caminhos, cobertura de IDs).
9. **Revisão adversarial** por subagente → correções.
10. **Classificação independente** da transcrição por outro subagente → comparação com os docs → correções.
11. **README** (este arquivo) e conferência final contra a checklist do enunciado.

## Prompts customizados

**1. Revisão adversarial dos documentos** (rodado depois da primeira versão completa):

```text
Você é um revisor técnico rígido (papel: professor de MBA que corrige um desafio de design docs).
NÃO edite nenhum arquivo, só leia e reporte.

Fontes da verdade: TRANSCRICAO.md (reunião) e o código em src/, prisma/, tests/, package.json.
Enunciado e critérios de aceite: desafio.txt.
Documentos a revisar: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-*.md.

Procure, com evidência (citação do doc + citação da transcrição/código):
1. Afirmações que CONTRADIZEM a transcrição ou o código (valores, nomes, quem disse o quê,
   timestamp citado que não sustenta a afirmação).
2. Itens que foram DESCARTADOS ou ADIADOS na reunião mas aparecem como requisito/escopo
   (email, dashboard, rate limiting, inbound, arquivamento, multi-worker, exactly-once etc.).
3. Requisitos/decisões/restrições SEM origem identificável que não estejam marcados como
   "(proposta do FDD)" — ou seja, invenção da IA disfarçada de decisão.
4. Decisões da reunião que ficaram de FORA dos documentos.
5. Duplicação indevida entre documentos (ex.: RFC descendo ao nível de detalhe do FDD).
6. Erros técnicos no FDD: inconsistências no modelo Prisma, no fluxo do worker, na matriz de
   erros, nos contratos HTTP (status codes incoerentes com o código existente, ex.: requireRole
   retorna FORBIDDEN, validate middleware retorna VALIDATION_ERROR), na conta do backoff.
7. Checklist de critérios de aceite do desafio.txt: marque cada um como OK/FALHA.

Formato: lista numerada de achados com Severidade, Arquivo:seção, Problema, Evidência,
Correção sugerida. No fim, o checklist. Não elogie.
```

**2. Classificação independente da transcrição** (o subagente só podia ler a transcrição, para não ser "contaminado" pelos documentos):

```text
Leia APENAS o arquivo TRANSCRICAO.md. Não leia a pasta docs/ nem nenhum outro arquivo.

Você é um analista de requisitos. Classifique TUDO o que foi dito na reunião em exatamente
uma destas categorias:
- DECIDIDO (decisão arquitetural fechada)
- REQUISITO FUNCIONAL
- REQUISITO NÃO FUNCIONAL / RESTRIÇÃO
- DESCARTADO (alternativa rejeitada; diga o motivo dado)
- ADIADO / FORA DE ESCOPO (diga para quando, se foi dito)
- EM ABERTO (levantado e não resolvido)
- DETALHE SECUNDÁRIO (sem peso de decisão)

Regras:
- Cada item em uma linha com: categoria | resumo curto | [hh:mm] Falante.
- Não infira nada que não esteja literalmente na fala. Se houver ambiguidade ou contradição
  entre falas (ex.: números que não fecham), liste numa seção final "AMBIGUIDADES".
- Não inclua opinião sobre a solução.
```

## Iterações e ajustes

Foram **3 iterações principais** (geração + validação mecânica → revisão adversarial → classificação independente), além de ajustes pontuais durante a escrita.

**Durante a primeira geração (leitura crítica do código e da transcrição)**
- **"5 tentativas" não fecha com a conta.** A reunião diz "5 tentativas" e lista 5 intervalos (1m/5m/30m/2h/12h) que somam "quase 15 horas". Com 5 envios no total só cabem 4 intervalos (~2h36). Adotei 1 envio + 5 retentativas, deixei a explicação no ADR-003 e abri a questão no RFC (RFC-QA-06), em vez de esconder a ambiguidade.
- **Códigos `WEBHOOK_*` x middleware existente.** O `validate.middleware.ts` transforma qualquer `ZodError` em `VALIDATION_ERROR`, então a validação de `https` "no schema Zod" (como a Sofia pediu) nunca sairia com `WEBHOOK_INVALID_URL` no `code`. Resolvi colocando o código no `details[].message`, sem mexer no middleware, coerente com o "não precisa mudar nada" do Bruno.
- **`NotFoundError` fixa `NOT_FOUND`.** Os 404 do módulo precisam estender `AppError` diretamente para usar o prefixo `WEBHOOK_`.
- **Ordenação durante retry.** O documento só garantia ordem "por pedido com um worker", mas um evento em retry deixa o seguinte passar na frente. Registrei como risco e questão em aberto (RFC-QA-07).

**Iteração 2: revisão adversarial (18 achados, todos tratados)**
- Cenários do PRD atribuíam fatos a clientes reais sem base ("a Nova Cargo assina SHIPPED/DELIVERED", "a MaxDistribuição ficou 2 h fora"). Na transcrição eram exemplos genéricos. Corrigido.
- Meta "3 de 3 clientes ativos até fim de novembro" foi inventada; virou "feature em produção até fim de novembro", que é o prazo da Atlas.
- ADR-003 marcado como "Aceito" com um ponto que o RFC deixava em aberto. O status passou a explicitar a pendência.
- `WEBHOOK_SECRET_REQUIRED` (citado pelo Bruno) não tinha cenário possível, porque a secret é sempre gerada pela plataforma. Ficou documentado como "não implementar" em vez de inventar um uso.
- `DELETE` com cascade apagava a DLQ, que o Diego quer como "evidence pra debug". Virou remoção lógica.
- Worker processando o lote em sequência quebrava a meta de < 10 s com clientes lentos. Virou paralelo por `order_id`, mantendo a ordem por pedido.
- Replay por `:id` sem nenhuma forma de descobrir o id. Proposto `GET /admin/webhooks/dead-letter`, marcado como proposta.
- Ajustes técnicos: usar o singleton `prisma` no worker, valores decididos como constantes (não env), índice `(status, next_attempt_at)`, mensagem da `.url()` do Zod, 401 em todos os contratos, menos duplicação entre PRD, RFC e FDD.

**Iteração 3: classificação independente**
- Bateu com os documentos em todas as decisões, descartes e adiamentos.
- Apontou que os blocos da estimativa já somam 3 sprints antes do "HMAC... mais um pouco". Eu tinha escrito "restante da 3ª sprint", o que era invenção. Corrigi e registrei a falta de folga.
- Apontou a janela de retry "12 ou 24 horas" x "quase 15 horas", que acrescentei à nota do ADR-003.

**Validação mecânica (rodada após cada iteração):** 0 citações `[hh:mm] Nome` inválidas, 0 caminhos inexistentes e todos os IDs dos documentos presentes no tracker. O tracker tem 270 linhas, 87% com fonte TRANSCRICAO e 35 com CODIGO.

## Como navegar a entrega

Ordem sugerida de leitura (do "por quê" para o "como"):

1. [`docs/PRD.md`](docs/PRD.md): problema, público, escopo, requisitos e métricas.
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/README.md): uma decisão por arquivo.
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
   - [ADR-004 HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-005 At-least-once com X-Event-Id](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 Snapshot do payload e filtro na inserção](docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)
4. [`docs/FDD.md`](docs/FDD.md): modelo de dados, fluxos, contratos HTTP, erros, resiliência, observabilidade e integração com o código.
5. [`docs/TRACKER.md`](docs/TRACKER.md): origem de cada item. Use para conferir qualquer afirmação dos documentos.

Fonte primária: [`TRANSCRICAO.md`](TRANSCRICAO.md). Código de referência em `src/` e `prisma/` (inalterados).
