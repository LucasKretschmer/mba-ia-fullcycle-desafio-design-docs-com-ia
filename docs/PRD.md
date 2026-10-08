# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de notificação de mudança de status de pedido |
| **Responsável de produto** | Marcos (PM) |
| **Tech Lead** | Larissa |
| **Autor do documento** | Lucas Kretschmer |
| **Status** | Aprovado em reunião, aguardando revisão de design |
| **Data** | 2026-10-07 |
| **Documentos técnicos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS vai passar a **avisar os clientes B2B, por webhook, sempre que o status de um pedido deles mudar**. Hoje esses clientes descobrem mudanças consultando a API de pedidos repetidamente. Com a feature, cada cliente cadastra uma URL, escolhe quais status quer receber e passa a receber uma chamada HTTP assinada a cada mudança relevante.

O pedido veio formalmente de três clientes: **Atlas Comercial, MaxDistribuição e Nova Cargo** [09:00 Marcos]. O fluxo é só de saída: nós notificamos, eles recebem [09:02 Marcos].

## 2. Problema e motivação

- **Integração lenta e cara para o cliente:** sem notificação, os clientes fazem polling em `GET /orders` "de tempos em tempos" para saber se algo mudou [09:00 Marcos].
- **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a feature não sair até o fim do trimestre [09:00 Marcos]. O prazo pedido por ela é fim de novembro [09:45 Marcos].
- **Lacuna no produto:** o OMS não tem nenhum mecanismo de notificação externa; o único jeito de saber de uma mudança é consultar.

## 3. Público-alvo e cenários de uso

| Público | Necessidade |
|---|---|
| **Desenvolvedores de integração dos clientes B2B** (Atlas, MaxDistribuição, Nova Cargo) | Receber mudanças de status sem polling, validar a autenticidade e evitar processar duplicados |
| **Usuários do OMS que representam o cliente** | Cadastrar e manter os webhooks pela API, autenticados com o JWT do OMS [09:32 Marcos] |
| **Administradores do OMS** (role `ADMIN`) | Reprocessar eventos que falharam definitivamente [09:36 Sofia] |

**Cenários**

1. **Acompanhar entregas:** um cliente cadastra um webhook só para `SHIPPED` e `DELIVERED` e passa a saber assim que o pedido sai e chega, sem polling [09:33 Marcos].
2. **Cliente em manutenção:** o endpoint de um cliente fica 2 horas fora do ar em manutenção planejada (já aconteceu com cliente nosso); as notificações são retentadas e chegam quando o sistema volta, sem ninguém intervir [09:16 Diego].
3. **Suporte investiga "não recebi":** o cliente consulta o histórico de entregas do webhook e vê tentativas, códigos de resposta e tempo de resposta [09:34 Marcos].
4. **Falha definitiva:** depois de ~15 h de falhas o evento vai para a fila de falhas; um admin corrige o problema com o cliente e reprocessa [09:18 Diego].
5. **Secret vazada:** o cliente pede uma nova secret e tem 24 h para trocar nos sistemas dele sem perder notificações [09:21 Sofia, 09:22 Diego].

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
|---|---|---|---|
| PRD-OBJ-01 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e a entrega bem-sucedida na 1ª tentativa (p95) | **< 10 segundos** [09:02 Marcos] |
| PRD-OBJ-02 | Cumprir o prazo pedido pela Atlas e evitar a perda do cliente | Data de entrada em produção | **Até o fim de novembro** [09:45 Marcos] |
| PRD-OBJ-03 | Nenhuma mudança de status sem notificação | Mudanças de status assinadas sem evento correspondente | **0** [09:40 Bruno, 09:41 Diego] |
| PRD-OBJ-04 | Resiliência a indisponibilidade do cliente | Janela coberta por retentativas automáticas antes de exigir ação manual | **~15 horas** [09:17 Diego, 09:17 Marcos] |

Indicador acompanhado, sem meta definida na reunião: redução das consultas a `GET /orders` feitas por esses três clientes, que é a dor original [09:00 Marcos].

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por customer, via API [09:31 Marcos, 09:33 Bruno].
- Escolha dos status que cada webhook recebe [09:33 Marcos].
- Notificação a cada mudança de status de pedido, com assinatura e identificador único do evento.
- Rotação de secret com período de convivência [09:21 Sofia].
- Retentativas automáticas e fila de falhas com reprocessamento manual por admin [09:17 Larissa, 09:18 Diego].
- Histórico de entregas por webhook [09:34 Marcos].
- Documentação para os clientes no portal do desenvolvedor [09:26 Marcos, 09:40 Marcos].

### 5.2 Fora de escopo

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-OUT-01 | **Aviso por e-mail** ao cliente quando o webhook falha repetidamente | Adiado para a próxima fase, depois de medir impacto | [09:37 Larissa] |
| PRD-OUT-02 | **Dashboard/painel visual** para o cliente ver seus webhooks | Descartado nesta feature; é projeto separado do time de frontend | [09:40 Larissa] |
| PRD-OUT-03 | **Rate limiting** de envio por cliente | Não entra; observar e decidir depois | [09:39 Diego, 09:39 Larissa] |
| PRD-OUT-04 | **Webhooks de entrada** (cliente enviando para nós) | Descartado; só outbound | [09:02 Marcos] |
| PRD-OUT-05 | **Arquivamento** de eventos entregues (~30 dias) | Fora desta feature | [09:08 Diego] |
| PRD-OUT-06 | **Garantia de ordem global** e escala para múltiplos workers | Futuro; hoje só ordem por pedido com um worker | [09:13 Diego, 09:14 Marcos] |
| PRD-OUT-07 | **Entrega exactly-once** | Descartado; garantia é at-least-once | [09:25 Diego] |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O usuário autenticado deve conseguir **cadastrar um webhook** informando o customer, a URL e a lista de status desejados. | [09:31 Marcos], [09:32 Larissa] |
| PRD-FR-02 | A **secret é gerada pela plataforma** e devolvida ao cliente na criação. | [09:31 Marcos] |
| PRD-FR-03 | Deve ser possível **listar os webhooks de um customer**. | [09:33 Bruno] |
| PRD-FR-04 | Deve ser possível **editar** e **remover** um webhook. | [09:33 Bruno] |
| PRD-FR-05 | Cada webhook escolhe **quais status quer receber**; status não assinados não geram notificação. | [09:33 Marcos], [09:34 Bruno] |
| PRD-FR-06 | A cada **mudança de status de um pedido**, todos os webhooks ativos do customer que assinam o novo status devem ser notificados. | [09:00 Marcos], [09:40 Bruno] |
| PRD-FR-07 | Toda notificação deve ser **assinada com HMAC-SHA256** usando uma secret exclusiva daquele webhook. | [09:20 Sofia], [09:21 Sofia] |
| PRD-FR-08 | O cliente deve conseguir **rotacionar a secret** pela API; a antiga continua válida por **24 horas**. | [09:21 Sofia] |
| PRD-FR-09 | Cada notificação leva um **identificador único do evento** (`X-Event-Id`) e o identificador do webhook (`X-Webhook-Id`), para deduplicação e para quem tem vários cadastros. | [09:25 Diego], [09:44 Sofia] |
| PRD-FR-10 | Notificações que falham devem ser **retentadas automaticamente** com intervalos crescentes. | [09:15 Diego], [09:17 Larissa] |
| PRD-FR-11 | Esgotadas as retentativas, o evento vai para uma **fila de falhas (DLQ)** que guarda payload e motivo, como evidência para debug e reprocessamento. | [09:18 Diego] |
| PRD-FR-12 | Um **administrador** (role `ADMIN`) pode **reprocessar** um evento da fila de falhas; quem fez o reprocessamento fica registrado. | [09:18 Diego], [09:36 Sofia] |
| PRD-FR-13 | O cliente consegue ver o **histórico de entregas** de um webhook: sucesso/falha, payload, resposta e tempo de resposta. | [09:34 Marcos] |
| PRD-FR-14 | A notificação traz um **payload enxuto** com os dados da mudança (pedido, status de/para, customer, total), sem os itens. | [09:43 Diego] |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | **Latência:** primeira tentativa em até ~2 s após a mudança; sempre abaixo de 10 s em condição normal. | [09:02 Marcos], [09:10 Larissa] |
| PRD-NFR-02 | **Consistência:** não pode existir mudança de status sem evento, nem evento de mudança que não aconteceu. | [09:06 Diego], [09:40 Bruno] |
| PRD-NFR-03 | **Isolamento:** a notificação não pode deixar a mudança de status mais lenta nem bloqueá-la por causa de um cliente lento ou fora do ar. | [09:04 Bruno] |
| PRD-NFR-04 | **Segurança de transporte:** URL do webhook obrigatoriamente `https`. | [09:23 Sofia] |
| PRD-NFR-05 | **Tamanho:** payload limitado a **64 KB**; acima disso é erro, sem truncar. | [09:23 Sofia], [09:24 Larissa] |
| PRD-NFR-06 | **Timeout:** o cliente tem **10 s** para responder. | [09:42 Diego] |
| PRD-NFR-07 | **Garantia de entrega:** at-least-once; o cliente deve deduplicar. | [09:24 Diego], [09:26 Larissa] |
| PRD-NFR-08 | **Ordem:** garantida apenas por pedido e enquanto houver um único worker. | [09:13 Larissa] |
| PRD-NFR-09 | **Operação:** sem infraestrutura nova; reaproveitar MySQL e padrões do projeto. | [09:07 Diego], [09:30 Larissa] |
| PRD-NFR-10 | **Disponibilidade:** o processamento das notificações roda separado da API; reinício da API não interrompe entregas. | [09:11 Diego] |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | Detalhe |
|---|---|---|
| Registrar eventos no próprio MySQL (outbox) | Latência mínima de ~2 s em troca de zero infra nova e consistência total | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15 h, depois fila de falhas | Evento pode chegar até ~15 h atrasado; aceito pelo produto [09:17 Marcos] | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| At-least-once | Cliente precisa deduplicar; mitigado com documentação [09:26 Marcos] | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Secret por webhook com rotação de 24 h | Mais gestão de secrets, menos estrago em vazamento | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Ordem só por pedido | Clientes não precisam de ordem global [09:14 Marcos] | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| Payload enxuto sem itens | Cliente precisa consultar `GET /orders/:id` se quiser detalhes | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Responsável | Origem |
|---|---|---|---|
| PRD-DEP-01 | Revisão de segurança (HMAC e geração de secret), mínimo 2 dias úteis antes do deploy | Sofia | [09:46 Sofia], [09:49 Sofia] |
| PRD-DEP-02 | Documentação no portal do desenvolvedor (integração, at-least-once, validação da assinatura) | Marcos | [09:26 Marcos], [09:40 Marcos] |
| PRD-DEP-03 | Confirmação do prazo com os clientes | Marcos | [09:47 Marcos], [09:49 Marcos] |
| PRD-DEP-04 | Sessão de revisão do design com Bruno e Diego antes de codar | Larissa | [09:50 Larissa] |
| PRD-DEP-05 | Clientes precisam expor endpoint `https`, validar a assinatura e deduplicar por `X-Event-Id` | Clientes B2B | [09:23 Sofia], [09:25 Diego] |
| PRD-DEP-06 | Um novo processo (worker) para deploy e operação | A definir na revisão de design | [09:11 Diego] |

**Prazo estimado:** 3 sprints, incluindo a revisão de segurança [09:46 Larissa, 09:47 Larissa]. Quebra estimada pela Larissa [09:46 Larissa]:

| ID | Bloco | Estimativa |
|---|---|---|
| PRD-PRAZO-01 | Modelagem de outbox e DLQ | 1 sprint |
| PRD-PRAZO-02 | Worker e retry | 1 sprint |
| PRD-PRAZO-03 | CRUD de configuração e histórico de entregas | ½ sprint |
| PRD-PRAZO-04 | Integração no `order.service` e testes ponta a ponta | ½ sprint |
| PRD-PRAZO-05 | HMAC, schemas, validações e revisão da Sofia (mín. 2 dias úteis) | "mais um pouco" (sem número) |

Observação: os quatro primeiros blocos já somam 3 sprints, e o total de 3 sprints foi um "chute" da Larissa [09:46 Larissa]. Isso deixa pouca folga para HMAC e para a revisão de segurança. O risco está coberto em PRD-RISK-01.

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISK-01 | Atraso na entrega e perda da Atlas para o concorrente | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limit fora); estimativa de 3 sprints; Marcos alinha prazo com os clientes [09:00 Marcos, 09:37 Larissa, 09:46 Larissa] |
| PRD-RISK-02 | Cliente não trata duplicados e processa a mesma mudança duas vezes | Média | Médio | `X-Event-Id` em toda notificação + destaque no portal [09:25 Sofia, 09:26 Marcos] |
| PRD-RISK-03 | Vazamento de secret de um cliente | Baixa | Alto | Secret por webhook, rotação com 24 h de convivência, revisão da Sofia [09:21 Sofia, 09:22 Diego] |
| PRD-RISK-04 | Cliente ser "bombardeado" com muitas notificações em pouco tempo | Baixa | Médio | Observar em produção; rate limiting como próxima decisão [09:38 Diego, 09:39 Larissa] |
| PRD-RISK-05 | Cliente fica sem saber que o webhook está falhando (sem e-mail nesta fase) | Média | Médio | Histórico de entregas consultável; DLQ monitorada pelo time; e-mail na próxima fase [09:34 Marcos, 09:37 Larissa] |
| PRD-RISK-06 | Qualquer usuário autenticado consegue alterar webhooks de qualquer customer | Média | Alto | Aceito nesta fase; endurecer permissões depois [09:37 Sofia] |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-CA-01 | Dado um webhook cadastrado para `SHIPPED`, quando um pedido do customer muda para `SHIPPED`, então o endpoint recebe uma notificação assinada em menos de 10 s. |
| PRD-CA-02 | Dado um webhook que não assina `PAID`, quando o pedido vai para `PAID`, então nenhuma notificação é enviada para ele. |
| PRD-CA-03 | Dado um cadastro com URL `http://`, então a API recusa com erro de validação. |
| PRD-CA-04 | Dado um endpoint fora do ar, quando ele volta dentro de ~15 h, então a notificação é entregue sem intervenção manual. |
| PRD-CA-05 | Dado um evento que esgotou as retentativas, então ele aparece na fila de falhas e um `ADMIN` consegue reprocessar; um usuário não-admin recebe 403. |
| PRD-CA-06 | Dado uma rotação de secret, então durante 24 h o cliente consegue validar notificações com a secret antiga ou com a nova. |
| PRD-CA-07 | Dado um webhook com entregas, então o histórico mostra sucesso/falha, payload, resposta e tempo de resposta. |
| PRD-CA-08 | Dado uma mudança de status que falha (rollback), então nenhuma notificação é enviada. |
| PRD-CA-09 | Toda notificação recebida pelo cliente tem `X-Event-Id`, e retentativas do mesmo evento usam o mesmo valor. |

Os critérios técnicos detalhados estão no [FDD, seção 12](FDD.md#12-critérios-de-aceite-técnicos).

## 12. Estratégia de testes e validação

- **Testes unitários:** assinatura HMAC, cálculo do próximo horário de retry, regras de validação do cadastro (https, lista de status), montagem do payload.
- **Testes de integração (API + MySQL):** mesmo padrão dos testes atuais em `tests/orders.test.ts` com `supertest`, cobrindo CRUD de webhooks, rotação, histórico, replay com `ADMIN` x `OPERATOR` e a garantia de que mudança de status com rollback não gera evento.
- **Testes do worker:** endpoint HTTP local simulando sucesso, erro, demora acima de 10 s e queda, para validar retry, DLQ e headers.
- **Regressão:** os testes existentes de pedidos e auth continuam passando.
- **Revisão de segurança:** Sofia revisa geração de secret e HMAC antes do deploy, com pelo menos 2 dias úteis reservados [09:46 Sofia].
- **Validação com os clientes:** documentação publicada no portal [09:26 Marcos] e prazo confirmado com os três clientes [09:47 Marcos]. O acompanhamento das metas da seção 4 começa no go-live.
