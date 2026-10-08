# ADR-005: Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Diego, Larissa, Sofia, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com outbox + worker + retry, existem janelas em que o evento pode ser enviado mais de uma vez. Exemplo: o cliente processou e respondeu 200, mas a resposta chegou depois do timeout de 10 s; ou o worker caiu entre receber o 2xx e marcar o evento como entregue. Precisamos definir qual garantia damos para o cliente.

## Decisão

1. A plataforma garante **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado [09:24 Diego].
2. Cada evento leva um **`X-Event-Id`** no header: um **UUID gerado quando o evento entra na outbox**, único por evento. O cliente deduplica por esse id [09:25 Diego].
3. O mesmo `event_id` é reaproveitado em todas as retentativas e no replay da DLQ, senão a deduplicação não funciona.
4. O comportamento é documentado em destaque no portal do desenvolvedor [09:26 Marcos].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve a grande maioria dos casos e é o que Stripe e GitHub fazem [09:25 Diego]. |
| **At-most-once (enviar uma vez e não retentar)** | Plausível, mas incompatível com a política de retry do ADR-003; perderia eventos em qualquer instabilidade do cliente. |

## Consequências

**Positivas**
- Modelo simples, alinhado ao mercado; clientes já conhecem.
- Permite retry e replay sem medo de "duplicar" do nosso lado.

**Negativas / trade-offs**
- Transfere para o cliente a responsabilidade de deduplicar [09:25 Sofia]. Mitigação: documentação clara no portal.
- Gera de fato entregas duplicadas em cenários de timeout/queda do worker, o que aparece no histórico de entregas.
