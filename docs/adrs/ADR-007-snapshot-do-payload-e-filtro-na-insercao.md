# ADR-007: Payload renderizado (snapshot) e filtro de eventos no momento da inserção na outbox

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (parte final, após a saída de Marcos e Sofia)
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

Duas perguntas sobre o conteúdo da outbox ficaram para o fim da reunião:

1. A linha da outbox guarda o **payload já pronto** ou só o `order_id`, renderizando na hora do envio? [09:51 Bruno]
2. O filtro "quais status cada webhook quer receber" é aplicado **na inserção** ou **no envio**? [09:34 Diego]

Cada webhook escolhe a lista de status que quer ouvir (ex.: só `SHIPPED` e `DELIVERED`) [09:33 Marcos].

## Decisão

1. **Snapshot na inserção**: o payload JSON é montado e gravado quando o evento entra na outbox, dentro da transação de `changeStatus`. Se o pedido mudar depois, o evento continua refletindo o estado do momento da transição [09:52 Larissa, 09:52 Diego, 09:52 Bruno].
2. **Filtro na inserção**: só se insere linha na outbox para webhooks ativos do customer que assinam aquele `to_status`. Se nenhum webhook quer aquele status, nada é inserido [09:34 Bruno, 09:34 Diego].
3. Payload enxuto: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`; **sem itens**. Quem quiser detalhes consulta `GET /orders/:id` [09:43 Diego, 09:44 Bruno].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Em retry ou replay o cliente receberia o estado atual do pedido, não o da transição, gerando eventos incoerentes ("caso esquisito") [09:52 Larissa]. |
| **Filtrar no envio** | Gravaria linhas que nunca seriam enviadas; filtrar na inserção economiza linha na tabela [09:34 Bruno]. |
| **Payload com itens do pedido** | Inflaria o payload sem necessidade [09:43 Diego]. |

## Consequências

**Positivas**
- Retry e replay entregam exatamente o mesmo conteúdo (mesma assinatura, mesmo `event_id`).
- Outbox só tem trabalho real; menos linhas, menos polling.

**Negativas / trade-offs**
- O trabalho de montar o payload e consultar os webhooks do customer roda **dentro** da transação de `changeStatus`, deixando-a um pouco mais longa.
- Alterar a lista de eventos de um webhook não afeta eventos já inseridos na outbox.
- Mudança no formato do payload não "corrige" eventos antigos já gravados.
