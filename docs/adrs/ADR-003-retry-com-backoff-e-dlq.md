# ADR-003: Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

- **Status:** Aceito (com o número exato de envios pendente de confirmação, ver nota abaixo e [RFC-QA-06](../RFC.md#5-questões-em-aberto))
- **Data da decisão:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos, Sofia
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou respondendo erro. Precisamos de uma política que (a) aguente indisponibilidades reais dos clientes e (b) não deixe eventos pendurados para sempre. Já houve cliente com indisponibilidade de duas horas em manutenção planejada [09:16 Diego].

## Decisão

1. **Backoff exponencial com teto de "5 tentativas"**, nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h** [09:17 Diego, 09:17 Larissa]. Isso dá quase 15 horas entre a primeira falha e a última tentativa, o que o PM considerou aceitável [09:17 Marcos].
2. Timeout de 10 segundos por chamada; sem resposta nesse tempo conta como falha e entra no retry [09:42 Diego].
3. Esgotadas as tentativas, o evento é considerado **falha permanente** e vai para a **DLQ**, que é uma **tabela separada `webhook_dead_letter`** com payload, motivo da falha e timestamp [09:18 Diego].
4. Reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente [09:18 Diego]. Exige role `ADMIN` e registro de quem fez o replay para auditoria [09:36 Sofia, 09:36 Larissa].

> **Nota de interpretação (análise nossa, não dita na reunião):** a reunião fala em "5 tentativas" [09:17 Larissa, 09:48 Larissa] e, ao mesmo tempo, em 5 intervalos que somam "quase 15 horas entre primeira falha e última tentativa" [09:17 Diego]. Antes, Diego tinha estimado uma janela "de até 12 ou 24 horas" [09:15 Diego]; os intervalos fechados somam 14h36, o que cabe nessa faixa. As duas coisas só fecham com **1 envio inicial + 5 retentativas** (6 envios): com 5 envios no total só cabem 4 intervalos (1+5+30+120 min ≈ 2h36). Adotamos essa leitura porque é a que preserva a janela de ~15 h aceita pelo produto [09:17 Marcos]. Precisa ser confirmada com Diego e Larissa antes da implementação. Esta é a explicação de referência; os outros documentos só apontam para cá.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | Evento pode ficar pendurado para sempre se o cliente sumiu [09:15 Diego]. |
| **3 tentativas** (mais agressivo) | Cobriria só ~30 minutos; uma indisponibilidade de manhã mataria o evento. Já houve cliente fora por 2 h [09:16 Bruno, 09:16 Diego]. |
| **DLQ como status `failed` dentro da própria outbox** | Misturaria falhas permanentes com o fluxo quente da outbox; tabela separada deixa a leitura da outbox mais limpa e serve de evidência para debug e reprocessamento [09:17 Larissa, 09:18 Diego]. |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15 h sem intervenção humana.
- DLQ isolada facilita investigação e replay controlado.
- Replay restrito a `ADMIN` e auditado.

**Negativas / trade-offs**
- Um evento pode chegar ao cliente até ~15 h depois do fato. Aceito pelo produto [09:17 Marcos].
- Com retry, um evento mais novo do mesmo pedido pode ser entregue antes de um mais antigo que está esperando a próxima tentativa. Isso enfraquece a ordenação por pedido do ADR-002 (ver questões em aberto do RFC).
- Replay manual depende de alguém olhar a DLQ; o alerta proativo ao cliente (e-mail) ficou para fase futura [09:37 Larissa].
