# Tracker de Rastreabilidade

Cada linha liga um item dos documentos à sua origem na transcrição (`TRANSCRICAO.md`) ou no código. Os IDs são os mesmos usados dentro dos documentos (ex.: `PRD-FR-03` aparece na tabela de requisitos do PRD).

- **Fonte `TRANSCRICAO`**: Localização = `[hh:mm] Falante` da fala principal que sustenta o item.
- **Fonte `CODIGO`**: Localização = caminho do arquivo no repositório.
- Itens marcados como *(proposta do FDD)* no FDD não foram fechados na reunião; aqui eles apontam para a decisão ou o trecho de código de onde foram derivados.

## PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em `GET /orders`, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Webhooks só de saída (outbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Contexto | OMS não tem mecanismo de notificação; único entry-point é a API | CODIGO | src/server.ts |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Usuários do OMS que representam o cliente cadastram via API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Administradores (role ADMIN) reprocessam a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário | Um cliente assina só `SHIPPED` e `DELIVERED` (exemplo genérico da reunião) | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Cliente fora do ar por 2 h em manutenção planejada | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente consulta histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-04 | docs/PRD.md | Cenário | Falha definitiva vai para DLQ e é reprocessada | TRANSCRICAO | [09:18] Diego |
| PRD-CEN-05 | docs/PRD.md | Cenário | Cliente vaza secret e rotaciona | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | p95 da entrega < 10 s ("tempo real" para os clientes) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Feature em produção até fim de novembro (prazo da Atlas) | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Zero mudança de status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de ~15 h coberta por retry automático | TRANSCRICAO | [09:17] Diego |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente quando webhook falha (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída (observar e decidir depois) | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Ordem global e múltiplos workers | TRANSCRICAO | [09:13] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com customer, URL e status desejados | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar (PATCH) e remover (DELETE) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificar a cada mudança de status assinada | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 com secret exclusiva do webhook | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | `X-Event-Id` e `X-Webhook-Id` em cada notificação | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry automático com intervalos crescentes | TRANSCRICAO | [09:15] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | DLQ consultável após esgotar retries | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay por ADMIN com registro de quem fez | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Histórico de entregas com payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Payload enxuto sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência ~2 s, sempre < 10 s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Consistência entre status e evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Notificação não bloqueia mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente `https` | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB, erro sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | At-least-once, cliente deduplica | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordem só por pedido e com um worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Sem infra nova, reuso de padrões | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Processamento separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-TO-01 | docs/PRD.md | Trade-off | Atraso de até ~15 h aceito pelo produto | TRANSCRICAO | [09:17] Marcos |
| PRD-TO-02 | docs/PRD.md | Trade-off | Clientes não precisam de ordem global | TRANSCRICAO | [09:14] Marcos |
| PRD-TO-03 | docs/PRD.md | Trade-off | Responsabilidade de dedup no cliente, mitigada com portal | TRANSCRICAO | [09:26] Marcos |
| PRD-TO-04 | docs/PRD.md | Trade-off | Payload sem itens; detalhes via `GET /orders/:id` | TRANSCRICAO | [09:43] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Sessão de revisão do design antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Cliente precisa deduplicar por `X-Event-Id` | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Novo processo worker para operar | TRANSCRICAO | [09:11] Diego |
| PRD-PRAZO-00 | docs/PRD.md | Restrição | Estimativa total de 3 sprints com revisão incluída | TRANSCRICAO | [09:47] Larissa |
| PRD-PRAZO-01 | docs/PRD.md | Restrição | Outbox e DLQ: 1 sprint | TRANSCRICAO | [09:46] Larissa |
| PRD-PRAZO-02 | docs/PRD.md | Restrição | Worker e retry: 1 sprint | TRANSCRICAO | [09:46] Larissa |
| PRD-PRAZO-03 | docs/PRD.md | Restrição | CRUD e deliveries: ½ sprint | TRANSCRICAO | [09:46] Larissa |
| PRD-PRAZO-04 | docs/PRD.md | Restrição | Integração no order.service e testes: ½ sprint | TRANSCRICAO | [09:46] Larissa |
| PRD-PRAZO-05 | docs/PRD.md | Restrição | HMAC, schemas, validações e revisão da Sofia | TRANSCRICAO | [09:46] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso e perda da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Muitas notificações em pouco tempo (sem rate limit) | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente sem aviso de falha (e-mail adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-RISK-06 | docs/PRD.md | Risco | CRUD liberado para qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Notificação assinada em < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | URL `http://` recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Entrega automática se o cliente volta em ~15 h | TRANSCRICAO | [09:17] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | Replay só por ADMIN | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Secret antiga e nova válidas por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Histórico mostra sucesso/falha, payload, resposta, tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Rollback da mudança não envia notificação | TRANSCRICAO | [09:06] Diego |
| PRD-CA-09 | docs/PRD.md | Critério de Aceite | Mesmo `X-Event-Id` nas retentativas | TRANSCRICAO | [09:25] Diego |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Integração com supertest + MySQL no padrão dos testes atuais | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Revisão de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-CTX-01 | docs/RFC.md | Requisito Não Funcional | "Tempo real" = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | `changeStatus` já roda transação com pedido, histórico e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-03 | docs/RFC.md | Restrição | Time pequeno, sem infra nova | TRANSCRICAO | [09:07] Diego |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox no MySQL na mesma transação | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão | `publishWebhookEvent(tx, ...)` chamado por `changeStatus` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Worker separado, polling 2 s, Prisma próprio | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Retry 1m/5m/30m/2h/12h + DLQ + replay ADMIN | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-06 | docs/RFC.md | Decisão | At-least-once com `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Reuso dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila dedicada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do MySQL para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Múltiplos workers sem perder ordem | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | E-mail quando webhook falha | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-06 | docs/RFC.md | Questão em aberto | "5 tentativas" = 5 envios ou 5 retentativas | TRANSCRICAO | [09:17] Diego |
| RFC-QA-07 | docs/RFC.md | Questão em aberto | Ordem por pedido durante retry | TRANSCRICAO | [09:12] Diego |
| RFC-QA-08 | docs/RFC.md | Questão em aberto | `customer_id` no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-IMP-01 | docs/RFC.md | Impacto | Novas tabelas e migration no schema Prisma | CODIGO | prisma/schema.prisma |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo de fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-RISK-02 | docs/RFC.md | Risco | Worker único como ponto de falha | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | `User` sem vínculo com `Customer`, sem isolamento no CRUD | CODIGO | prisma/schema.prisma |
| RFC-RISK-05 | docs/RFC.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-06 | docs/RFC.md | Risco | Crescimento da outbox | TRANSCRICAO | [09:08] Diego |

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Transação de `changeStatus` com update, histórico e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-R01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Falha na outbox = rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| ADR-001-R02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índice em status e created_at; leitura em lotes pequenos | TRANSCRICAO | [09:08] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Disparo síncrono | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams | TRANSCRICAO | [09:07] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-R01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-R02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | `src/worker.ts` + `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-R03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio, mesma `DATABASE_URL` | TRANSCRICAO | [09:30] Bruno |
| ADR-002-R04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker; ordem só por `order_id` (limitação conhecida) | TRANSCRICAO | [09:13] Larissa |
| ADR-002-CTX | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | `createPrismaClient` e entry-point único hoje | CODIGO | src/config/database.ts |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger do MySQL | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Múltiplos workers | TRANSCRICAO | [09:13] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 retentativas, 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-R01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| ADR-003-R02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-R03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restrição | Interpretação: 1 envio + 5 retentativas (~15 h) | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | 3 tentativas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | DLQ como status `failed` na outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, header `X-Signature` | TRANSCRICAO | [09:22] Sofia |
| ADR-004-R01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret por endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-R02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação com grace period de 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-R03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL obrigatoriamente https (Zod) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-R04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| ADR-004-R05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once | TRANSCRICAO | [09:26] Larissa |
| ADR-005-R01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | `X-Event-Id` = UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-R02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | Documentar no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup vai para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-R01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo `src/modules/webhooks` | TRANSCRICAO | [09:27] Bruno |
| ADR-006-R02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo `WEBHOOK_` nos códigos | TRANSCRICAO | [09:29] Larissa |
| ADR-006-R03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Pino, sem lib nova de log | TRANSCRICAO | [09:29] Bruno |
| ADR-006-R04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | `requireRole` existente no replay | TRANSCRICAO | [09:36] Larissa |
| ADR-006-R05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | IDs UUID | TRANSCRICAO | [09:51] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Injetar repository no `OrderService` | TRANSCRICAO | [09:41] Diego |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | `AppError`, `InsufficientStockError`, `InvalidStatusTransitionError` | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Middleware de erro centralizado | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | `requireRole('ADMIN')` já usado em users | CODIGO | src/modules/users/user.routes.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | `validate` converte ZodError em `VALIDATION_ERROR` | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| ADR-007-R01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Filtro de status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-R02 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Campos do payload, sem itens | TRANSCRICAO | [09:43] Diego |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Alternativa descartada | Renderizar no envio a partir do `order_id` | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Alternativa descartada | Filtrar no envio | TRANSCRICAO | [09:34] Diego |

## FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CTX-01 | docs/FDD.md | Contexto | Validação de transição com `canTransition` | CODIGO | src/modules/orders/order.status.ts |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento se e somente se o status commitou | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | 1ª tentativa em ~2 s | TRANSCRICAO | [09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Nenhum HTTP dentro da transação | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Entregas autenticáveis e idempotentes | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Retry para falhas transitórias, DLQ para permanentes | TRANSCRICAO | [09:18] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Sem dependência nova | TRANSCRICAO | [09:30] Larissa |
| FDD-ESC-01 | docs/FDD.md | Fora de escopo | Evento de criação de pedido não é publicado (`create` não passa por `changeStatus`) | CODIGO | src/modules/orders/order.service.ts |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | `webhook_endpoints` com url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | `webhook_outbox` com 4 status e índices | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | `webhook_deliveries` com resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | `webhook_dead_letter` com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados (proposta do FDD) | Remoção lógica (`deletedAt`) para preservar histórico e DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados (proposta do FDD) | Índice `(status, next_attempt_at)` para a query do worker | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | `replayedById` no padrão de `OrderStatusHistory.changedById`; ids `Char(36)` | CODIGO | prisma/schema.prisma |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro de `changeStatus` via `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Worker em polling, pendentes mais antigos primeiro | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-02-REC | docs/FDD.md | Fluxo (proposta do FDD) | Recovery `PROCESSING → PENDING` no startup, derivado de at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-FLX-02-64KB | docs/FDD.md | Fluxo (proposta do FDD) | Checagem de 64 KB no worker para não bloquear a mudança de status | TRANSCRICAO | [09:04] Bruno |
| FDD-FLX-02-PAR | docs/FDD.md | Fluxo (proposta do FDD) | Lote agrupado por `order_id`: grupos em paralelo, sequencial dentro do grupo | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Tabela de backoff (até 14 h 36 min) | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | DLQ e replay com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-00 | docs/FDD.md | Restrição | `customer_id` não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks?customerId=` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-02-COD | docs/FDD.md | Padrão existente | `customerId` em query e `pageSize` máx. 100 como em pedidos | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /webhooks/:id/rotate-secret` | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato (proposta do FDD) | `GET /admin/webhooks/dead-letter` para o admin achar o id do replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09-WID | docs/FDD.md | Contrato | Header `X-Webhook-Id` | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-09-BODY | docs/FDD.md | Contrato | Body `order.status_changed` com campos enxutos | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | `WEBHOOK_CUSTOMER_NOT_FOUND` (prefixo para tudo do módulo) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | `WEBHOOK_INVALID_EVENTS` | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-05 | docs/FDD.md | Erro | `WEBHOOK_INACTIVE` (estado ativo do cadastro) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-08 | docs/FDD.md | Erro (não utilizado) | `WEBHOOK_SECRET_REQUIRED` citado na reunião, sem cenário porque a secret é sempre gerada | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-09 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-10 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_HTTP_ERROR` | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-12 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_NETWORK_ERROR` | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-13 | docs/FDD.md | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | TRANSCRICAO | [09:17] Larissa |
| FDD-ERR-14 | docs/FDD.md | Erro | `WEBHOOK_OUTBOX_INSERT_FAILED` só em log, erro original relançado e rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-COD | docs/FDD.md | Restrição | `NotFoundError` fixa `NOT_FOUND`; 404 do módulo estende `AppError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | HTTP fora da transação | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-02 | docs/FDD.md | Resiliência | Atomicidade outbox + status | TRANSCRICAO | [09:41] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Retry com backoff | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | DLQ + replay | TRANSCRICAO | [09:18] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Sem fallback por e-mail nesta fase; cliente pode usar `GET /orders/:id` | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-07 | docs/FDD.md | Resiliência | Recovery de crash seguro só com 1 worker | TRANSCRICAO | [09:12] Diego |
| FDD-RES-08 | docs/FDD.md | Resiliência | Isolamento de processo | TRANSCRICAO | [09:11] Diego |
| FDD-RES-09 | docs/FDD.md | Resiliência | Idempotência por `X-Event-Id` | TRANSCRICAO | [09:25] Diego |
| FDD-RES-10 | docs/FDD.md | Resiliência | Limite de 64 KB sem truncar | TRANSCRICAO | [09:23] Sofia |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Só Pino, nada novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redact de secret nos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de auditoria do replay com `userId` | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Alerta quando pendente mais antigo > 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Tracing por correlação a partir do `X-Request-Id` | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | `changeStatus` chama `publishWebhookEvent(tx, ...)` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Status válidos para `events` (sem `PENDING`) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros do módulo estendem `AppError`/`ConflictError` | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | `errorMiddleware` sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Código `WEBHOOK_*` no `details` do `VALIDATION_ERROR` | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | `authenticate` + `requireRole('ADMIN')` | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | `redactPaths` com `*.secret` | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Só `WEBHOOK_BATCH_SIZE` no `envSchema`; valores decididos viram constantes | CODIGO | src/config/env.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Worker reusa o `prisma` exportado (instância própria por processo) | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | `src/worker.ts` copia bootstrap/shutdown do server | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Rotas `/webhooks` e `/admin/webhooks` no router e no `buildControllers` | CODIGO | src/routes/index.ts |
| FDD-INT-12 | docs/FDD.md | Integração | `paginated` nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Models novos e relações inversas em `Customer`/`User` | CODIGO | prisma/schema.prisma |
| FDD-INT-14 | docs/FDD.md | Integração | Scripts `worker`/`worker:dev`, sem dependência nova | CODIGO | package.json |
| FDD-INT-15 | docs/FDD.md | Integração | Limpeza das tabelas novas no `beforeEach` | CODIGO | tests/setup.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >= 20 (`fetch`, `AbortSignal.timeout`) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8 existente | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Restrição | Exatamente uma instância do worker | TRANSCRICAO | [09:12] Diego |
| FDD-DEP-04 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-05 | docs/FDD.md | Restrição | Prazo de 3 sprints | TRANSCRICAO | [09:46] Larissa |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | 1 linha de outbox por webhook interessado, na transação | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Sem webhook interessado, nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Falha na outbox não muda status | TRANSCRICAO | [09:41] Diego |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Entrega em ~2 s | TRANSCRICAO | [09:09] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | Headers e assinatura válidos | TRANSCRICAO | [09:44] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | Agenda de retry conforme backoff | TRANSCRICAO | [09:17] Diego |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | DLQ após esgotar retentativas | TRANSCRICAO | [09:18] Diego |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | `X-Event-Id` estável | TRANSCRICAO | [09:25] Diego |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | Replay ADMIN ok, OPERATOR 403 | TRANSCRICAO | [09:36] Larissa |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | `http://` recusado | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | > 64 KB vai para DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-12 | docs/FDD.md | Critério de Aceite | Duas assinaturas durante 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-13 | docs/FDD.md | Critério de Aceite | Deliveries com até 100 por página | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-14 | docs/FDD.md | Critério de Aceite | Reenvio após crash do worker | TRANSCRICAO | [09:24] Diego |
| FDD-CA-15 | docs/FDD.md | Critério de Aceite | Secret fora de logs e listagens | TRANSCRICAO | [09:22] Diego |
| FDD-CA-16 | docs/FDD.md | Critério de Aceite | Testes existentes continuam passando | CODIGO | tests/orders.test.ts |
| FDD-CA-17 | docs/FDD.md | Critério de Aceite | DELETE preserva deliveries e DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Clientes lentos atrasam o próximo ciclo do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Fora de ordem durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-03 | docs/FDD.md | Risco | Duas instâncias do worker por engano | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Transação de `changeStatus` mais longa | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-05 | docs/FDD.md | Risco | Secret em texto no banco | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-06 | docs/FDD.md | Risco | Sem isolamento entre customers no CRUD (detalhe no RFC-RISK-04) | TRANSCRICAO | [09:37] Sofia |
| FDD-RISK-07 | docs/FDD.md | Risco | Outbox e deliveries sem arquivamento | TRANSCRICAO | [09:08] Diego |
