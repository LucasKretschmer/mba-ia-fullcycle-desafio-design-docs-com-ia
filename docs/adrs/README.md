# Architectural Decision Records

Decisões arquiteturais da feature **Sistema de Webhooks de Notificação de Pedidos**, no formato MADR simplificado (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação de 24 h | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) | Snapshot do payload e filtro de eventos na inserção | Aceito |
