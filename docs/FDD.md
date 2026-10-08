# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Autor** | Lucas Kretschmer |
| **Status** | Pronto para revisão técnica (Larissa, Bruno, Diego) e de segurança (Sofia) |
| **Data** | 2026-10-07 |
| **Base** | [RFC](RFC.md), [ADR-001 a ADR-007](adrs/README.md) |

> Convenção: trechos marcados com **(proposta do FDD)** são detalhes de implementação que não foram fechados na reunião e foram derivados das decisões + código. Eles precisam ser validados na sessão de revisão que a Larissa marcou com Bruno e Diego [09:50 Larissa]. Todo o resto tem origem rastreada no [TRACKER](TRACKER.md).

---

## 1. Contexto e motivação técnica

Hoje a única forma de um cliente B2B saber que o status de um pedido mudou é consultar `GET /api/v1/orders` repetidamente [09:00 Marcos]. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que:

1. lê o pedido com itens;
2. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
3. debita ou repõe estoque quando aplicável;
4. atualiza `orders.status`;
5. grava `order_status_history`.

O sistema não tem eventos, filas ou processos em background. Só existe o entry-point da API (`src/server.ts`). A feature precisa criar esse caminho de notificação sem tirar a consistência e a performance dessa transação [09:04 Bruno].

## 2. Objetivos técnicos

| ID | Objetivo |
|---|---|
| FDD-OBJ-01 | Toda mudança de status commitada gera evento para os webhooks interessados; nenhuma mudança com rollback gera evento [09:06 Diego, 09:40 Bruno]. |
| FDD-OBJ-02 | Primeira tentativa de entrega em até ~2 s após o commit (polling), dentro do limite de 10 s dos clientes [09:02 Marcos, 09:10 Larissa]. |
| FDD-OBJ-03 | Nenhuma chamada HTTP externa dentro da transação de pedidos [09:04 Bruno]. |
| FDD-OBJ-04 | Entregas autenticáveis pelo cliente (HMAC-SHA256) e idempotentes do lado dele (`X-Event-Id`) [09:22 Sofia, 09:26 Larissa]. |
| FDD-OBJ-05 | Falhas transitórias absorvidas por retry; falhas permanentes visíveis e reprocessáveis (DLQ) [09:17 Larissa, 09:18 Diego]. |
| FDD-OBJ-06 | Zero dependência ou infraestrutura nova; reuso dos padrões do projeto [09:30 Larissa]. |

## 3. Escopo e exclusões

**Dentro do escopo**
- CRUD de configuração de webhook por customer, com filtro de status [09:31 Marcos, 09:33 Bruno].
- Rotação de secret com grace period de 24 h [09:21 Sofia].
- Publicação na outbox dentro de `changeStatus` [09:40 Bruno].
- Worker separado (`src/worker.ts`) com polling, retry, DLQ [09:11 Larissa, 09:48 Larissa].
- Histórico de entregas por webhook [09:34 Marcos].
- Replay manual de DLQ por `ADMIN` [09:18 Diego, 09:36 Larissa].

**Exclusões técnicas (não implementar)** — as exclusões de produto (e-mail, dashboard, rate limiting, inbound) estão no [PRD §5.2](PRD.md#52-fora-de-escopo).

| Item | Origem |
|---|---|
| Job de arquivamento da outbox/deliveries | [09:08 Diego] |
| Mais de um worker, particionamento por `order_id` ou lock pessimista | [09:13 Diego] |
| Exactly-once / deduplicação do nosso lado | [09:25 Diego] |
| Evento de criação de pedido (`null → PENDING`): só `changeStatus` publica eventos | Código: `create` em `src/modules/orders/order.service.ts` não passa por `changeStatus`; a reunião só tratou mudança de status [09:40 Bruno] |

---

## 4. Modelo de dados

Novas tabelas no `prisma/schema.prisma`, todas com `id` UUID `Char(36)` como o resto do schema [09:51 Larissa].

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // lista de OrderStatus assinados
  active                  Boolean   @default(true)
  deletedAt               DateTime? // soft delete (proposta do FDD)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36) // = event_id / X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36) // sem FK: o snapshot é autossuficiente
  eventType     String              @db.VarChar(64)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook     WebhookEndpoint     @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([status, createdAt])
  @@index([createdAt])
  @@index([status, nextAttemptAt]) // proposta do FDD: query do worker filtra next_attempt_at
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  responseBody String?  @db.Text
  durationMs   Int
  errorCode    String?  @db.VarChar(64)
  createdAt    DateTime @default(now())

  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  webhookId    String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(500)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  outbox     WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook    WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  replayedBy User?           @relation("DeadLetterReplayedBy", fields: [replayedById], references: [id])

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:
- `webhook_endpoints` guarda `url`, `secret`, `customer_id` e estado ativo [09:21 Bruno], mais a lista de eventos [09:33 Marcos] e a secret anterior para o grace period [09:21 Sofia].
- `webhook_outbox.status` segue os quatro estados citados (pendente, processando, falhou, entregue) com índice em `status` e `created_at` [09:08 Diego]. `FAILED` significa "esgotou tentativas e está na DLQ".
- O `id` da outbox é o próprio `event_id`. O replay reaproveita a mesma linha, então o `X-Event-Id` não muda [09:25 Diego]. **(proposta do FDD)**
- `webhook_dead_letter` guarda payload, motivo e timestamp [09:18 Diego], mais quem fez o replay [09:36 Sofia]. `replayedById` segue o mesmo padrão de `OrderStatusHistory.changedById`.
- `events` é `Json` porque o MySQL não tem tipo array. Os valores válidos são do enum `OrderStatus`.
- `Customer` e `User` ganham as relações inversas (`webhooks`, `replayedDeadLetters`), exigência do Prisma.
- **Remoção é lógica** (`deletedAt` + `active = false`) para não apagar histórico de entregas nem a DLQ, que serve de evidência para debug [09:18 Diego]. O `onDelete: Cascade` só entra em ação se o próprio customer for apagado. **(proposta do FDD)**

---

## 5. Fluxos detalhados

### FDD-FLX-01 — Criação do evento na outbox

Executado dentro de `OrderService.changeStatus`, depois do `tx.orderStatusHistory.create` [09:40 Bruno].

```mermaid
sequenceDiagram
    participant C as OrderController
    participant S as OrderService.changeStatus
    participant P as publishWebhookEvent
    participant DB as MySQL (tx)
    C->>S: PATCH /orders/:id/status
    S->>DB: BEGIN; SELECT order + items
    S->>DB: estoque / UPDATE orders / INSERT order_status_history
    S->>P: publishWebhookEvent(tx, order, from, to)
    P->>DB: SELECT webhook_endpoints WHERE customer_id = ? AND active
    P->>P: filtra quem assina "to" (events contém to)
    P->>DB: INSERT webhook_outbox (1 linha por webhook, payload snapshot)
    S->>DB: COMMIT (ou ROLLBACK se qualquer passo falhar)
```

Passos:
1. Buscar `webhook_endpoints` ativos do `order.customerId`.
2. Manter só os que têm `to` em `events` [09:33 Marcos]. Se a lista ficar vazia, **não insere nada** [09:34 Bruno].
3. Para cada webhook, gerar um UUID (`event_id`) e montar o payload snapshot (FDD-CONTRATO-09) [09:52 Larissa].
4. `tx.webhookOutbox.createMany(...)` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
5. Qualquer exceção aqui é logada (`logger.error` com `code: 'WEBHOOK_OUTBOX_INSERT_FAILED'`, `orderId`) e **relançada sem embrulhar**. A transação faz rollback e o status do pedido não muda [09:40 Bruno]. Como o erro original é do Prisma (não é `AppError`), o `errorMiddleware` responde `500 INTERNAL_SERVER_ERROR` normalmente.

Esboço:

```ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const endpoints = await tx.webhookEndpoint.findMany({
    where: { customerId: order.customerId, active: true },
  });
  const targets = endpoints.filter((e) => (e.events as OrderStatus[]).includes(toStatus));
  if (targets.length === 0) return;

  const timestamp = new Date().toISOString();
  await tx.webhookOutbox.createMany({
    data: targets.map((endpoint) => {
      const eventId = uuidv4();
      return {
        id: eventId,
        webhookId: endpoint.id,
        orderId: order.id,
        eventType: 'order.status_changed',
        payload: buildPayload(eventId, timestamp, order, fromStatus, toStatus),
      };
    }),
  });
}
```

### FDD-FLX-02 — Processamento pelo worker

`src/worker.ts` faz o bootstrap e chama o loop de `src/modules/webhooks/webhook.worker.ts` [09:28 Bruno].

1. **Startup**: importa o `prisma` exportado por `src/config/database.ts`; como é outro processo Node, essa já é uma instância própria do worker [09:30 Bruno]. Faz a recuperação: `UPDATE webhook_outbox SET status = 'PENDING' WHERE status = 'PROCESSING'`. Isso só é seguro porque existe **um único worker** [09:12 Diego]; eventos que estavam em voo quando o processo caiu são reenviados (at-least-once) [09:24 Diego]. **(proposta do FDD)**
2. **Ciclo** (a cada `POLL_INTERVAL_MS = 2000`) [09:09 Diego]:
   - `SELECT ... WHERE status = 'PENDING' AND next_attempt_at <= NOW() ORDER BY created_at ASC LIMIT WEBHOOK_BATCH_SIZE` [09:08 Diego, 09:12 Diego].
   - Agrupa o lote por `order_id`. Grupos diferentes rodam **em paralelo**; dentro de um grupo, os eventos rodam **em sequência** por `created_at`. Assim a ordem por pedido se mantém [09:12 Diego] e um cliente lento (até 10 s por chamada) não segura os eventos dos outros clientes além do limite de 10 s [09:02 Marcos]. **(proposta do FDD)**
   - O próximo ciclo só é agendado depois que o lote atual termina (sem sobreposição).
3. **Para cada evento**:
   1. Marca `PROCESSING`.
   2. Carrega o `WebhookEndpoint`. Se estiver inativo ou removido → DLQ com `WEBHOOK_INACTIVE`, sem retry. **(proposta do FDD)**
   3. Serializa o payload. Se passar de 64 KB → DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry [09:23 Sofia, 09:24 Larissa].
   4. Assina e envia (FDD-CONTRATO-09) com timeout de 10 s [09:42 Diego].
   5. Grava uma linha em `webhook_deliveries` com status code, corpo da resposta, duração e erro [09:34 Marcos].
   6. 2xx → `DELIVERED`, `deliveredAt = now()`. Qualquer outro resultado → FDD-FLX-03.
4. **Shutdown**: em `SIGINT`/`SIGTERM`, para de buscar novos lotes, termina o evento em andamento e chama `prisma.$disconnect()`, igual ao `shutdown` de `src/server.ts`.

> Por que checar os 64 KB no worker e não no `changeStatus`? Se a checagem fosse na transação, um payload grande daria rollback na mudança de status, e a reunião deixou claro que webhook não pode impedir mudança de status [09:04 Bruno]. No worker, o evento vai direto para a DLQ e fica visível. **(proposta do FDD)**

### FDD-FLX-03 — Retry com backoff

Política do [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) [09:17 Larissa], na leitura de 1 envio inicial + 5 retentativas explicada lá (pendente de confirmação, RFC-QA-06).

| Envio | Quando acontece (após a falha anterior) | Tempo desde a 1ª falha |
|---|---|---|
| 1 (inicial) | ~2 s após o commit | — |
| 2 (retry 1) | +1 min | 1 min |
| 3 (retry 2) | +5 min | 6 min |
| 4 (retry 3) | +30 min | 36 min |
| 5 (retry 4) | +2 h | 2 h 36 min |
| 6 (retry 5) | +12 h | 14 h 36 min |
| — | falhou de novo → DLQ | — |

Em cada falha retentável (timeout, erro de rede, resposta não-2xx):
- `attempts = attempts + 1`;
- se `attempts <= 5` retentativas: `status = PENDING`, `nextAttemptAt = now() + BACKOFF[attempts - 1]`, `lastError = código`;
- senão → FDD-FLX-04.

Os valores decididos na reunião ficam como constantes em `src/modules/webhooks/webhook.constants.ts`, não como configuração: `BACKOFF_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]`, `POLL_INTERVAL_MS = 2_000`, `HTTP_TIMEOUT_MS = 10_000`, `MAX_PAYLOAD_BYTES = 65_536`, `SECRET_GRACE_PERIOD_MS = 86_400_000`.

Toda resposta não-2xx é tratada como retentável, inclusive 4xx: o cliente pode estar com deploy quebrado ou em manutenção [09:16 Diego]. **(proposta do FDD)**

### FDD-FLX-04 — DLQ e replay

**Entrada na DLQ** (em uma transação):
1. `webhook_outbox.status = FAILED`, `lastError = motivo`.
2. `INSERT webhook_dead_letter (outbox_id, webhook_id, payload, reason, failed_at)` [09:18 Diego].
3. Log `warn` `webhook_dead_lettered`.

**Replay** — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (só `ADMIN`) [09:18 Diego, 09:36 Sofia]:
1. Busca o dead letter; se não existir → `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
2. Se `replayedAt` já estiver preenchido → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
3. Se o webhook estiver inativo → `WEBHOOK_INACTIVE`.
4. Em transação: outbox volta para `PENDING`, `attempts = 0`, `nextAttemptAt = now()`; dead letter recebe `replayedAt = now()` e `replayedById = req.user.id`.
5. Log `info` `webhook_dead_letter_replayed` com `userId`, `deadLetterId`, `eventId` para auditoria [09:36 Sofia].

Se o evento falhar de novo depois do replay, uma **nova** linha de dead letter é criada; a antiga continua como histórico.

### FDD-FLX-05 — Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret` [09:21 Sofia]:
1. `previousSecret = secret atual`, `previousSecretExpiresAt = now() + 24h`.
2. `secret = nova secret` (`crypto.randomBytes(32).toString('hex')`) **(proposta do FDD; geração revisada pela Sofia [09:46 Sofia])**.
3. Enquanto `previousSecretExpiresAt > now()`, o worker envia as duas assinaturas no `X-Signature` (FDD-CONTRATO-09). Depois disso, só a nova.
4. Nova rotação durante o grace period substitui a `previousSecret` (só uma secret anterior é mantida).

---

## 6. Contratos públicos

Todos os endpoints ficam sob `/api/v1` (montagem em `src/app.ts`), exigem `Authorization: Bearer <jwt>` (`authenticate`) e respondem erros no formato do `errorMiddleware`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

Todos retornam `401 UNAUTHORIZED` sem token ou com token inválido (`authenticate`). O `customer_id` não vem do JWT, porque o JWT é do usuário operador [09:32 Bruno, 09:32 Larissa]. Vai no body do cadastro e na query da listagem, igual ao `customerId` de `src/modules/orders/order.schemas.ts` **(proposta do FDD, ver RFC-QA-08)**. O CRUD aceita qualquer role autenticada [09:37 Sofia].

### FDD-CONTRATO-01 — Criar webhook

`POST /api/v1/webhooks` [09:31 Marcos]

Request:
```json
{
  "customerId": "6f1c2a9e-1b7d-4c55-9a0e-3f2d8b7c1a10",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` — a secret só é exibida aqui e na rotação [09:31 Marcos]:
```json
{
  "id": "0b8e4a52-6f2d-4a8b-9c1e-7d5f3a2b9e44",
  "customerId": "6f1c2a9e-1b7d-4c55-9a0e-3f2d8b7c1a10",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c7a1e4b8d0c3f6a5e2d1b7c9f0a8e3d6b4c2a1f9e8d7c6b5a4f3e2d1c0b9a",
  "createdAt": "2026-10-07T14:03:11.204Z",
  "updatedAt": "2026-10-07T14:03:11.204Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado |
| 400 `VALIDATION_ERROR` | `url` não-https/inválida (`WEBHOOK_INVALID_URL` no details), `events` vazio ou com status inválido (`WEBHOOK_INVALID_EVENTS`) |
| 401 `UNAUTHORIZED` | Sem token / token inválido |
| 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

Validação (`webhook.schemas.ts`):
- `url`: `z.string().url({ message: 'WEBHOOK_INVALID_URL: invalid url' }).max(2048).refine((u) => u.startsWith('https://'), 'WEBHOOK_INVALID_URL: url must use https')` [09:23 Sofia].
- `events`: array não vazio, sem repetição, de `OrderStatus` **exceto `PENDING`**, que nunca é destino de transição em `src/modules/orders/order.status.ts` (`WEBHOOK_INVALID_EVENTS`). **(proposta do FDD)**

### FDD-CONTRATO-02 — Listar webhooks de um customer

`GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20` [09:33 Bruno]

Request: sem body; parâmetros na query. Webhooks removidos (`deletedAt`) não aparecem.

Response `200 OK` (formato `paginated` de `src/shared/http/response.ts`; a secret nunca é retornada):
```json
{
  "data": [
    {
      "id": "0b8e4a52-6f2d-4a8b-9c1e-7d5f3a2b9e44",
      "customerId": "6f1c2a9e-1b7d-4c55-9a0e-3f2d8b7c1a10",
      "url": "https://api.atlascomercial.com.br/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationInProgress": false,
      "createdAt": "2026-10-07T14:03:11.204Z",
      "updatedAt": "2026-10-07T14:03:11.204Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| 200 | OK (lista vazia se o customer não tem webhooks) |
| 400 `VALIDATION_ERROR` | `customerId` ausente ou não-UUID |
| 401 `UNAUTHORIZED` | Sem token |

### FDD-CONTRATO-03 — Editar webhook

`PATCH /api/v1/webhooks/:id` [09:33 Bruno]

Request (todos opcionais, pelo menos um):
```json
{ "events": ["PROCESSING", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK`:
```json
{
  "id": "0b8e4a52-6f2d-4a8b-9c1e-7d5f3a2b9e44",
  "customerId": "6f1c2a9e-1b7d-4c55-9a0e-3f2d8b7c1a10",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "events": ["PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotationInProgress": false,
  "createdAt": "2026-10-07T14:03:11.204Z",
  "updatedAt": "2026-10-07T15:20:47.932Z"
}
```

| Status | Quando |
|---|---|
| 200 | Atualizado |
| 400 `VALIDATION_ERROR` | Mesmas regras de `url`/`events` do POST, ou body vazio |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente ou removido |

Alterar `events` não afeta eventos já gravados na outbox (snapshot) [09:52 Larissa].

### FDD-CONTRATO-04 — Remover webhook

`DELETE /api/v1/webhooks/:id` [09:33 Bruno]

Request: sem body. Response `204 No Content`, sem body.

| Status | Quando |
|---|---|
| 204 | Removido logicamente (`deletedAt = now()`, `active = false`). Histórico de entregas e DLQ ficam preservados; eventos ainda pendentes vão para a DLQ com `WEBHOOK_INACTIVE` quando o worker os pegar **(proposta do FDD)** |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente ou já removido |

Para só pausar, usar `PATCH { "active": false }`.

### FDD-CONTRATO-05 — Rotacionar secret

`POST /api/v1/webhooks/:id/rotate-secret` [09:21 Sofia]

Request: sem body.

Response `200 OK`:
```json
{
  "id": "0b8e4a52-6f2d-4a8b-9c1e-7d5f3a2b9e44",
  "secret": "4e7d1a9c2b6f0e3d8a5c7b1f9e2d4a6c8b0f3e5d7a9c1b2e4f6a8c0d2e4f6a8b",
  "previousSecretExpiresAt": "2026-10-08T14:10:00.000Z"
}
```

| Status | Quando |
|---|---|
| 200 | Nova secret gerada; a anterior vale até `previousSecretExpiresAt` |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente ou removido |

### FDD-CONTRATO-06 — Histórico de entregas

`GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=100` [09:34 Marcos]

Request: sem body; `page` e `pageSize` na query. `pageSize` máximo 100 (mesmo limite de `listOrdersQuerySchema`), ordenado por `createdAt desc`. Funciona também para webhook removido, para não perder a investigação.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "c3a9f1e2-5b7d-4e8a-9c0f-1d2e3f4a5b6c",
      "eventId": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d",
      "eventType": "order.status_changed",
      "attempt": 2,
      "success": true,
      "statusCode": 200,
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d", "event_type": "order.status_changed", "...": "..." },
      "responseBody": "{\"received\":true}",
      "createdAt": "2026-10-07T14:05:02.110Z"
    },
    {
      "id": "b2f8e0d1-4a6c-4d7b-8e9f-0c1d2e3f4a5b",
      "eventId": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": false,
      "statusCode": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": { "event_id": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d", "...": "..." },
      "responseBody": null,
      "createdAt": "2026-10-07T14:04:01.950Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| 200 | OK |
| 401 `UNAUTHORIZED` | Sem token |
| 404 `WEBHOOK_NOT_FOUND` | `id` inexistente |

### FDD-CONTRATO-07 — Replay de dead letter (admin)

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` — `authenticate` + `requireRole('ADMIN')` [09:35 Diego, 09:36 Larissa]

Request: sem body.

Response `200 OK`:
```json
{
  "deadLetterId": "e1d2c3b4-a5f6-4e7d-8c9b-0a1f2e3d4c5b",
  "eventId": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d",
  "status": "PENDING",
  "replayedAt": "2026-10-08T09:12:44.501Z",
  "replayedById": "5d4c3b2a-1f0e-4d9c-8b7a-6f5e4d3c2b1a"
}
```

| Status | Quando |
|---|---|
| 200 | Evento recolocado como pendente na outbox |
| 401 `UNAUTHORIZED` | Sem token |
| 403 `FORBIDDEN` | Usuário não é `ADMIN` (erro padrão do `requireRole`) |
| 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` | `id` inexistente |
| 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Já reprocessado antes |
| 409 `WEBHOOK_INACTIVE` | Webhook está desativado |

### FDD-CONTRATO-08 — Listar dead letters (admin) **(proposta do FDD)**

`GET /api/v1/admin/webhooks/dead-letter?webhookId=<uuid>&replayed=false&page=1&pageSize=20` — `authenticate` + `requireRole('ADMIN')`

A reunião definiu o replay por `:id` [09:18 Diego], mas não disse como o admin descobre esse id. Sem uma listagem, ele precisaria de acesso direto ao banco. Este endpoint fecha essa lacuna e precisa ser validado na revisão de design.

Request: sem body; filtros opcionais na query.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "e1d2c3b4-a5f6-4e7d-8c9b-0a1f2e3d4c5b",
      "eventId": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d",
      "webhookId": "0b8e4a52-6f2d-4a8b-9c1e-7d5f3a2b9e44",
      "reason": "WEBHOOK_MAX_ATTEMPTS_EXCEEDED: last error WEBHOOK_DELIVERY_HTTP_ERROR (503)",
      "failedAt": "2026-10-08T04:41:02.000Z",
      "replayedAt": null,
      "payload": { "event_id": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d", "...": "..." }
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| 200 | OK |
| 401 `UNAUTHORIZED` | Sem token |
| 403 `FORBIDDEN` | Usuário não é `ADMIN` |

### FDD-CONTRATO-09 — Requisição enviada ao cliente (outbound)

`POST <url cadastrada>` feito pelo worker.

Headers [09:44 Diego, 09:44 Sofia]:

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (= `webhook_outbox.id`), igual em todas as tentativas |
| `X-Webhook-Id` | `id` do cadastro do webhook |
| `X-Timestamp` | Momento do envio desta tentativa, ISO 8601 |
| `X-Signature` | `sha256=<hex>` = HMAC-SHA256(secret, corpo bruto). Durante o grace period: `sha256=<hex_nova>,sha256=<hex_antiga>` **(proposta do FDD, validar com Sofia)** |

Body (snapshot gravado na outbox) [09:43 Diego]:
```json
{
  "event_id": "a7d3c2b1-9e8f-4a6b-8c5d-2e1f0a9b8c7d",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-07T14:04:00.312Z",
  "order_id": "9c8b7a6f-5e4d-4c3b-2a1f-0e9d8c7b6a5f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a9e-1b7d-4c55-9a0e-3f2d8b7c1a10",
  "total_cents": 18000
}
```

- Sem itens do pedido; detalhes via `GET /api/v1/orders/:id` [09:43 Diego].
- O `timestamp` do body é o momento da mudança de status (snapshot); o `X-Timestamp` é o momento do envio.
- Resposta esperada do cliente: qualquer **2xx** dentro de 10 s. Corpo da resposta é ignorado (só gravado no histórico).
- Redirects (3xx) não são seguidos e contam como falha, para não sair do `https` cadastrado. **(proposta do FDD)**

Verificação no lado do cliente (exemplo para o portal do desenvolvedor [09:26 Marcos]):

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValid(rawBody: string, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  return header.split(',').some((part) => {
    const received = part.trim().replace(/^sha256=/, '');
    return received.length === expected.length &&
      timingSafeEqual(Buffer.from(received), Buffer.from(expected));
  });
}
```

---

## 7. Matriz de erros

Todos os códigos usam o prefixo `WEBHOOK_` [09:29 Larissa]. Erros da API viram classes que herdam de `AppError` em `src/modules/webhooks/webhook.errors.ts`; erros do worker são gravados em `webhook_deliveries.error_code`, `webhook_outbox.last_error` e `webhook_dead_letter.reason`.

| ID | Código | Onde | HTTP | Causa | Retry? | Tratamento |
|---|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | API | 404 | `id` de webhook inexistente | — | `WebhookNotFoundError extends AppError` [09:28 Bruno] |
| FDD-ERR-02 | `WEBHOOK_CUSTOMER_NOT_FOUND` | API | 404 | `customerId` inexistente no cadastro | — | `AppError` 404 |
| FDD-ERR-03 | `WEBHOOK_INVALID_URL` | API (Zod) | 400 | URL inválida ou não-`https` [09:23 Sofia] | — | Sai como `code: VALIDATION_ERROR` com `details[].message` começando por `WEBHOOK_INVALID_URL` (ver 10.5) |
| FDD-ERR-04 | `WEBHOOK_INVALID_EVENTS` | API (Zod) | 400 | `events` vazio, repetido, inválido ou `PENDING` | — | Igual ao anterior |
| FDD-ERR-05 | `WEBHOOK_INACTIVE` | API / Worker | 409 / — | Replay para webhook inativo; evento pendente de webhook desativado | Não | API: `ConflictError(..., 'WEBHOOK_INACTIVE')`. Worker: DLQ direto |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | API | 404 | Dead letter inexistente | — | `AppError` 404 |
| FDD-ERR-07 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | API | 409 | `replayedAt` já preenchido | — | `ConflictError` |
| FDD-ERR-08 | `WEBHOOK_SECRET_REQUIRED` | — | — | Citado como exemplo na reunião [09:28 Bruno], mas sem cenário: a secret é sempre gerada pela plataforma [09:31 Marcos] e a coluna é NOT NULL | — | **Não implementar** nesta fase |
| FDD-ERR-09 | `WEBHOOK_PAYLOAD_TOO_LARGE` | Worker | — | Corpo serializado > 64 KB [09:24 Larissa] | Não | DLQ direto (não trunca) [09:23 Sofia] |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_TIMEOUT` | Worker | — | Sem resposta em 10 s [09:42 Diego] | Sim | Backoff (FDD-FLX-03) |
| FDD-ERR-11 | `WEBHOOK_DELIVERY_HTTP_ERROR` | Worker | — | Cliente respondeu não-2xx (inclui 3xx) | Sim | Backoff; `statusCode` gravado |
| FDD-ERR-12 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | Worker | — | DNS, conexão recusada, erro de TLS | Sim | Backoff |
| FDD-ERR-13 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Worker | — | Falhou após 5 retentativas [09:17 Larissa] | Não | DLQ; `reason` inclui o último erro |
| FDD-ERR-14 | `WEBHOOK_OUTBOX_INSERT_FAILED` | API (`changeStatus`) | 500 | Falha ao gravar na outbox | — | Código só de log: `publishWebhookEvent` loga com esse código e relança o erro original; rollback da mudança de status [09:40 Bruno]; o `errorMiddleware` devolve `INTERNAL_SERVER_ERROR` |

Observação: `NotFoundError` de `src/shared/errors/http-errors.ts` fixa o código `NOT_FOUND` no construtor, então os 404 do módulo precisam estender `AppError` diretamente para usar `WEBHOOK_*`. `ConflictError` e `UnprocessableEntityError` já aceitam o código como parâmetro.

---

## 8. Estratégias de resiliência

| ID | Estratégia | Detalhe | Origem |
|---|---|---|---|
| FDD-RES-01 | Isolamento da transação | HTTP nunca roda dentro de `changeStatus`; só um `INSERT` na outbox | [09:04 Bruno] |
| FDD-RES-02 | Atomicidade | Outbox na mesma transação; falha na outbox = rollback | [09:06 Diego], [09:40 Bruno] |
| FDD-RES-03 | Timeout | 10 s por chamada, via `AbortSignal.timeout(10_000)` no `fetch` nativo do Node 20 | [09:42 Diego] |
| FDD-RES-04 | Retry | 5 retentativas, 1m/5m/30m/2h/12h | [09:17 Larissa] |
| FDD-RES-05 | DLQ | Tabela separada + replay manual | [09:18 Diego] |
| FDD-RES-06 | Fallback | Não há canal alternativo nesta fase (e-mail adiado). O "fallback" é: o evento fica na DLQ e o cliente ainda pode consultar `GET /orders/:id` | [09:37 Larissa], [09:43 Diego] |
| FDD-RES-07 | Recuperação de crash | No startup, `PROCESSING → PENDING` (seguro só com 1 worker) | [09:12 Diego], [09:24 Diego] |
| FDD-RES-08 | Isolamento de processo | Worker em processo separado; restart da API não afeta entregas | [09:11 Diego] |
| FDD-RES-09 | Idempotência | `X-Event-Id` estável em retry e replay | [09:25 Diego] |
| FDD-RES-10 | Limite de payload | 64 KB, erro em vez de truncar | [09:23 Sofia], [09:24 Diego] |

---

## 9. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing; a única ferramenta é o Pino (`src/shared/logger/index.ts`), e a decisão foi não adicionar nada novo [09:29 Bruno]. Então a observabilidade desta fase é baseada em **logs estruturados + consultas nas tabelas**.

### 9.1 Logs (Pino)

Mesmo `logger`; no worker usar `logger.child({ component: 'webhook-worker' })` para separar a origem dos logs sem duplicar o campo `service` do `base`.

| Evento | Nível | Campos | Onde |
|---|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` | `publishWebhookEvent` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `recovered` | `src/worker.ts` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs` | worker |
| `webhook_delivery_failed` | warn | idem + `errorCode`, `nextAttemptAt` | worker |
| `webhook_dead_lettered` | warn | `eventId`, `webhookId`, `reason`, `attempts` | worker |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `userId` (auditoria) | API [09:36 Sofia] |
| `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` | API |

**Nunca logar secret.** Adicionar `*.secret` e `*.previousSecret` em `redactPaths` [09:22 Diego].

### 9.2 Métricas

Calculadas a partir das tabelas/logs (sem lib nova):

| Métrica | Como obter | Alerta sugerido |
|---|---|---|
| Latência de entrega (commit → 2xx) | `deliveredAt - createdAt` em `webhook_outbox` (p50/p95) | p95 da 1ª tentativa > 10 s [09:02 Marcos] |
| Backlog da outbox | `COUNT(*)` e idade do mais antigo com `status = PENDING AND next_attempt_at <= NOW()` | Pendente vencido mais antigo > 10 s = worker parado ou lento |
| Taxa de sucesso por webhook | `webhook_deliveries` agrupado por `webhook_id` | Queda brusca por cliente |
| Tentativas por evento | distribuição de `attempts` em eventos entregues | — |
| Entradas na DLQ | `COUNT(*)` em `webhook_dead_letter` por dia | Qualquer entrada nova |
| Replays | `replayed_at IS NOT NULL` | — |

### 9.3 Tracing

Sem tracing distribuído nesta fase. O rastreamento é por correlação de IDs:

`X-Request-Id` (gerado em `src/middlewares/request-logger.middleware.ts`) → `path` do log `http_request` do `PATCH /orders/:id/status` (o id do pedido está no path) → `webhook_event_enqueued` (`orderId` + `eventId`) → `webhook_deliveries` (`eventId`) → `X-Event-Id` no lado do cliente.

Com o `X-Event-Id` que o cliente informar no suporte, dá para reconstruir toda a trajetória do evento.

---

## 10. Integração com o sistema existente

| ID | Arquivo | Como integra |
|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, chamar `await publishWebhookEvent(tx, order, from, to)` [09:41 Bruno]. O `order` já carregado tem `id`, `orderNumber`, `customerId` e `totalCents`. Usa o `TxClient` (`Prisma.TransactionClient`) que já existe no arquivo. Nenhuma mudança no construtor do `OrderService` [09:41 Diego]. `create` não é alterado. |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Fonte da verdade dos status alcançáveis. O schema de `events` reaproveita `OrderStatus` e exclui `PENDING`, que não aparece como destino em `transitions`. |
| FDD-INT-03 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Classes do módulo (`webhook.errors.ts`) estendem `AppError` (404) e `ConflictError` (409) com códigos `WEBHOOK_*`, como `InsufficientStockError`/`InvalidStatusTransitionError` [09:28 Bruno]. |
| FDD-INT-04 | `src/middlewares/error.middleware.ts` | Sem alteração. Já serializa `AppError` como `{ error: { code, message, details } }` e trata P2002/P2025 do Prisma [09:29 Bruno]. |
| FDD-INT-05 | `src/middlewares/validate.middleware.ts` | Sem alteração. Ele transforma qualquer `ZodError` em `ValidationError` (`VALIDATION_ERROR`). Por isso os códigos `WEBHOOK_INVALID_URL`/`WEBHOOK_INVALID_EVENTS` vão como prefixo da mensagem no `details`, e não no `code`. |
| FDD-INT-06 | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` na rota de replay, igual a `src/modules/users/user.routes.ts` [09:36 Larissa]. `req.user.id` alimenta `replayedById` e o log de auditoria. |
| FDD-INT-07 | `src/shared/logger/index.ts` | Reuso do `logger` [09:29 Bruno]; incluir `*.secret` e `*.previousSecret` em `redactPaths`. |
| FDD-INT-08 | `src/config/env.ts` | Só uma variável nova no `envSchema`: `WEBHOOK_BATCH_SIZE` (valor a definir na revisão; a reunião só disse "batch pequeno" [09:08 Diego]). Os valores decididos (2 s, 10 s, 64 KB, backoff, 24 h) são constantes, não configuração (FDD-FLX-03). Atenção: o worker importa `env`, então também precisa de `JWT_SECRET` definido, mesmo sem usar. |
| FDD-INT-09 | `src/config/database.ts` | O worker importa o `prisma` exportado; por ser outro processo, já é uma instância própria com a mesma `DATABASE_URL` [09:30 Bruno]. |
| FDD-INT-10 | `src/server.ts` → novo `src/worker.ts` | `src/worker.ts` copia o padrão de bootstrap + shutdown (`SIGINT`/`SIGTERM`, `$disconnect`) do `server.ts` [09:11 Larissa]. |
| FDD-INT-11 | `src/app.ts` e `src/routes/index.ts` | `buildControllers` instancia `WebhookRepository`/`WebhookService`/`WebhookController`; `Controllers` ganha `webhooks`; `buildApiRouter` monta `/webhooks` e `/admin/webhooks`. A montagem em `/api/v1` já existe em `app.ts`. |
| FDD-INT-12 | `src/shared/http/response.ts` | `paginated(...)` nas listagens de webhooks e de deliveries. |
| FDD-INT-13 | `prisma/schema.prisma` + `prisma/migrations/` | Quatro models novos, enum `WebhookOutboxStatus`, relações inversas em `Customer` e `User`; migration gerada com `npm run db:migrate`. |
| FDD-INT-14 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"` [09:11 Larissa]. Sem dependência nova: `uuid` já existe, HMAC via `node:crypto`, HTTP via `fetch` nativo (`engines.node >= 20`). `tsconfig.build.json` já inclui `src/**/*.ts`, então `src/worker.ts` entra no build. |
| FDD-INT-15 | `tests/setup.ts` e `tests/helpers/factories.ts` | `beforeEach` passa a limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` antes das tabelas de pedidos/clientes; nova factory `createTestWebhook`. |

Estrutura do módulo [09:27 Bruno, 09:28 Bruno]:

```
src/
├── worker.ts                         # novo entry-point
└── modules/webhooks/
    ├── webhook.controller.ts
    ├── webhook.service.ts
    ├── webhook.repository.ts
    ├── webhook.routes.ts             # /webhooks e /admin/webhooks
    ├── webhook.schemas.ts
    ├── webhook.errors.ts
    ├── webhook.constants.ts          # 2 s, 10 s, 64 KB, backoff, 24 h
    ├── webhook.publisher.ts          # publishWebhookEvent + buildPayload
    ├── webhook.signer.ts             # HMAC-SHA256
    └── webhook.worker.ts             # loop, envio, retry, DLQ
```

---

## 11. Dependências e compatibilidade

- **Runtime:** Node.js >= 20 (já exigido no `package.json`), por causa de `fetch` e `AbortSignal.timeout`.
- **Banco:** MySQL 8 (`docker-compose.yml`), mesmo schema/usuário. Nenhuma infra nova [09:07 Diego].
- **Bibliotecas:** só as existentes (`@prisma/client`, `zod`, `pino`, `uuid`, `express`).
- **Compatibilidade de API:** nenhuma rota existente muda de contrato. `PATCH /orders/:id/status` continua com a mesma resposta; só fica um pouco mais lento por causa do insert na outbox.
- **Compatibilidade para o cliente:** o formato do payload é versionado pelo `event_type`. Mudança incompatível = novo `event_type`.
- **Deploy:** migration antes de subir API e worker. Exatamente **uma** instância do worker [09:12 Diego]. Revisão de segurança da Sofia antes do deploy [09:46 Sofia].
- **Prazo:** 3 sprints incluindo a revisão [09:46 Larissa].

## 12. Critérios de aceite técnicos

| ID | Critério |
|---|---|
| FDD-CA-01 | `PATCH /orders/:id/status` para um status assinado cria exatamente 1 linha em `webhook_outbox` por webhook ativo interessado, na mesma transação. |
| FDD-CA-02 | Se nenhum webhook assina o novo status, nenhuma linha é criada. |
| FDD-CA-03 | Se o insert na outbox falha, o status do pedido, o histórico e o estoque não mudam. |
| FDD-CA-04 | Com o worker rodando e o endpoint respondendo 200, o evento fica `DELIVERED` em até ~2 s + latência do cliente. |
| FDD-CA-05 | A requisição contém `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`, e a assinatura bate com HMAC-SHA256 do corpo bruto com a secret do webhook. |
| FDD-CA-06 | Timeout (> 10 s), erro de rede ou não-2xx incrementa `attempts` e agenda `nextAttemptAt` conforme 1m/5m/30m/2h/12h. |
| FDD-CA-07 | Depois da 5ª retentativa falha, outbox fica `FAILED` e existe 1 linha em `webhook_dead_letter` com payload e motivo. |
| FDD-CA-08 | O `X-Event-Id` é o mesmo em todas as tentativas e após replay. |
| FDD-CA-09 | Replay com `ADMIN` volta o evento para `PENDING` e grava `replayedById`; com `OPERATOR` retorna 403. |
| FDD-CA-10 | Cadastro com `http://` ou URL malformada retorna 400 com `WEBHOOK_INVALID_URL` no `details`. |
| FDD-CA-11 | Payload > 64 KB vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem nenhuma tentativa HTTP. |
| FDD-CA-12 | Após rotação, por 24 h o `X-Signature` traz as duas assinaturas; depois, só a nova. |
| FDD-CA-13 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens por página, mais recentes primeiro. |
| FDD-CA-14 | Matar o worker com evento em `PROCESSING` e reiniciar: o evento é reenviado. |
| FDD-CA-15 | A secret não aparece em nenhum log nem na listagem. |
| FDD-CA-16 | Os testes existentes (`tests/orders.test.ts`, `tests/auth.test.ts`) continuam passando. |
| FDD-CA-17 | `DELETE` de webhook preserva `webhook_deliveries` e `webhook_dead_letter` do webhook. |

## 13. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RISK-01 | Muitos clientes lentos ao mesmo tempo ainda podem atrasar o próximo ciclo (o ciclo espera o grupo mais lento, até 10 s) | Baixa | Médio | Paralelismo por `order_id` no lote (FDD-FLX-02); monitorar a idade do pendente mais antigo (9.2) |
| FDD-RISK-02 | Evento fora de ordem quando um evento anterior do mesmo pedido está em retry | Média | Baixo | Limitação conhecida [09:13 Larissa]; `timestamp` no payload; RFC-QA-07 |
| FDD-RISK-03 | Duas instâncias do worker rodando por engano (ordenação e recovery quebram) | Baixa | Médio | Documentar no deploy; log `webhook_worker_started` permite detectar |
| FDD-RISK-04 | Transação de `changeStatus` mais longa | Baixa | Baixo | Uma query indexada por `customer_id` + `createMany` |
| FDD-RISK-05 | Secret em texto no banco | Média | Alto | Redact nos logs, nunca devolvida após criação/rotação; proteção em repouso na revisão da Sofia [09:46 Sofia] |
| FDD-RISK-06 | Sem isolamento entre customers no CRUD | Média | Alto | Ver [RFC-RISK-04](RFC.md#6-impacto-e-riscos); nada a implementar nesta fase |
| FDD-RISK-07 | Outbox e deliveries crescem sem limite | Alta (longo prazo) | Médio | Índices [09:08 Diego]; arquivamento no backlog |
