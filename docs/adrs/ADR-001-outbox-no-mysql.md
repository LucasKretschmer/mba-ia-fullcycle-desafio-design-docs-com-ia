# ADR-001: Padrão Outbox no MySQL existente para eventos de webhook

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B querem ser avisados quando o status dos pedidos muda, em vez de ficar consultando `GET /orders` [09:00 Marcos]. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e mexe em `stock_quantity` [09:04 Bruno].

Precisamos registrar "aconteceu uma mudança de status e alguém precisa ser notificado" de forma que:

1. A notificação nunca seja perdida se o status mudou.
2. A notificação nunca exista se a mudança de status deu rollback.
3. A chamada HTTP para o cliente não fique dentro da transação de pedidos.

## Decisão

Usar o **padrão Transactional Outbox no MySQL que já existe**. Dentro da mesma transação de `changeStatus`, inserir uma linha na tabela `webhook_outbox` com o evento. Um worker separado (ver ADR-002) lê essa tabela e faz as chamadas HTTP.

- Se a transação commitar, o evento está gravado. Se der rollback, o evento some junto [09:06 Diego].
- Se a inserção na outbox falhar, a transação inteira dá rollback. Não pode existir status alterado sem evento [09:40 Bruno].
- A tabela tem índice em `status` (pendente, processando, falhou, entregue) e em `created_at`; o worker lê só os pendentes em lotes pequenos [09:08 Diego].
- Chave primária UUID, seguindo o resto do schema [09:51 Larissa].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo síncrono dentro de `changeStatus`** | A transação já é pesada; um cliente lento travaria a mudança de status de outros pedidos. E se o cliente estiver fora do ar não dá para fazer rollback da mudança de status por causa disso [09:04 Bruno, 09:06 Diego]. |
| **Redis Streams (ou fila parecida)** | Exige subir infraestrutura nova (Redis Cluster). Para um time pequeno isso foi considerado overengineering, já que o MySQL resolve [09:07 Larissa, 09:07 Diego]. *Análise complementar (não dita na reunião):* publicar no Redis depois do commit do MySQL também não seria atômico (dual write). |

## Consequências

**Positivas**
- Consistência garantida entre mudança de status e evento, sem coordenação distribuída.
- Zero infraestrutura nova: mesmo banco, mesmo Prisma, mesmo backup.
- A outbox vira também um registro consultável do que foi emitido.

**Negativas / trade-offs**
- A transação de `changeStatus` ganha mais um `INSERT` (por webhook interessado). Aceitamos um pouco mais de custo na transação em troca da garantia de consistência.
- A tabela cresce continuamente. O arquivamento de linhas entregues (~30 dias) foi citado, mas está **fora do escopo** desta feature [09:08 Diego]; precisa entrar no backlog antes que o volume vire problema.
- Latência mínima passa a depender do intervalo de leitura do worker (ver ADR-002), em vez de ser imediata.
