# ADR-006: Webhooks como módulo padrão do projeto, reaproveitando AppError, Pino, middlewares e Zod

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Bruno, Diego, Sofia
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

O OMS já tem padrões bem definidos que o time conhece:

- Cada domínio é um módulo em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (ex.: `src/modules/orders/`, `src/modules/customers/`) [09:27 Bruno].
- Erros de negócio herdam de `AppError` (`src/shared/errors/app-error.ts`), com classes específicas em `src/shared/errors/http-errors.ts` como `InsufficientStockError` (`INSUFFICIENT_STOCK`) e `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) [09:28 Bruno].
- Logger Pino único em `src/shared/logger/index.ts` [09:29 Bruno].
- `src/middlewares/error.middleware.ts` trata `AppError`, `ZodError` e erros conhecidos do Prisma de forma centralizada [09:29 Bruno].
- Autorização por papel com `requireRole` em `src/middlewares/auth.middleware.ts` (já usado em `src/modules/users/user.routes.ts`) [09:36 Larissa].
- Validação de entrada com schemas Zod + `validate` (`src/middlewares/validate.middleware.ts`).

A feature de webhooks poderia trazer bibliotecas e convenções novas, ou se encaixar no que existe.

## Decisão

**Reuso máximo do que já existe** [09:30 Larissa]:

1. Novo módulo `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais `webhook.worker.ts` para o processamento [09:27 Bruno, 09:28 Bruno].
2. Erros do módulo herdam de `AppError`/classes de `http-errors.ts` e usam **prefixo `WEBHOOK_`** em todos os códigos (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, ...) [09:28 Bruno, 09:29 Larissa].
3. Logs com o mesmo `logger` Pino; nenhuma lib nova de log [09:29 Bruno].
4. Erros HTTP tratados pelo `errorMiddleware` existente, sem alteração nele [09:29 Bruno].
5. Endpoint de replay usa o `requireRole('ADMIN')` existente [09:36 Larissa].
6. Integração com pedidos via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient`, em vez de injetar um repository inteiro no `OrderService` [09:41 Bruno, 09:41 Diego].
7. IDs em UUID, como o resto do `prisma/schema.prisma` [09:51 Larissa].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Injetar o `WebhookRepository` no construtor do `OrderService`** | Acopla mais os módulos; uma função pura recebendo o `tx` resolve com menos superfície [09:41 Diego]. |
| **Estrutura própria para o módulo (ex.: lib de eventos/mensageria, logger dedicado)** | Plausível, mas aumentaria a curva de manutenção e contraria o "não vamos botar nada novo" [09:29 Bruno]. |
| **IDs auto incrementais na outbox** | Diego perguntou se a outbox usaria auto incremento ou UUID; Larissa decidiu UUID para seguir o padrão do projeto [09:51 Diego, 09:51 Larissa]. A ordenação do worker vem de `created_at`, não do id. |

## Consequências

**Positivas**
- Curva de aprendizado quase zero para o time; code review mais rápido.
- Respostas de erro no mesmo formato `{ error: { code, message, details } }` que os clientes já consomem.
- O worker reaproveita logger, config e Prisma sem duplicação.

**Negativas / trade-offs**
- O `validate` middleware converte qualquer `ZodError` em `VALIDATION_ERROR`; validações feitas no schema (como a de `https`) **não** saem com código `WEBHOOK_*` no campo `code`. O FDD resolve isso colocando o código `WEBHOOK_*` na mensagem do `details`, sem mexer no middleware.
- Ficamos presos às limitações do padrão atual (ex.: não há métricas nem tracing no projeto; a observabilidade da feature parte de logs estruturados).
