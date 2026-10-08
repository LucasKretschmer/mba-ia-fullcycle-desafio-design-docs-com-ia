# ADR-002: Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com a outbox definida (ADR-001), falta decidir **como** e **onde** os eventos são consumidos. Os clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos [09:02 Marcos].

Hoje o projeto tem um único entry-point, `src/server.ts`, que sobe a API Express e cria o `PrismaClient` via `src/config/database.ts`. Não existe nenhum processo de background.

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos, processa e marca o resultado [09:09 Diego, 09:10 Larissa].
2. **Processo separado da API**: novo entry-point `src/worker.ts` e script `npm run worker` [09:11 Diego, 09:11 Larissa]. Se a API reiniciar, o worker não cai junto.
3. **Mesmo banco e mesma stack, mas `PrismaClient` próprio**: PrismaClient é por processo; usa a mesma `DATABASE_URL` [09:11 Bruno, 09:30 Bruno].
4. **Uma única instância do worker** nesta fase. Com um worker só, os eventos são processados por ordem de `created_at`, o que dá ordenação implícita por `order_id`. Isso é registrado como **limitação conhecida**: não há garantia de ordem global, e a ordem por pedido só vale enquanto houver um único worker [09:12 Diego, 09:13 Larissa].
5. A lógica de processamento fica dentro do módulo (`src/modules/webhooks/webhook.worker.ts`); `src/worker.ts` só faz o bootstrap [09:28 Bruno].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no MySQL para ser reativo** | MySQL não tem `LISTEN/NOTIFY` como o Postgres. Trigger só executa SQL, não avisa processo externo; teria que improvisar (escrever arquivo, chamar endpoint) [09:09 Diego]. |
| **Worker dentro do processo da API** | Se a API reinicia, o worker morre junto [09:11 Diego]. Também disputaria recurso com as requisições HTTP. |
| **Múltiplos workers em paralelo** | Perde a garantia de ordem por pedido. Particionar por `order_id` ou usar lock pessimista fica para o futuro [09:13 Diego]. |

## Consequências

**Positivas**
- Latência de pior caso ~2 s até a primeira tentativa, bem dentro do limite de 10 s pedido pelos clientes [09:10 Larissa].
- Implementação simples: um `setTimeout`/loop, sem dependência nova.
- Deploy independente: o worker pode ser reiniciado sem afetar a API.

**Negativas / trade-offs**
- Polling gera consultas constantes no banco mesmo sem eventos. Aceito porque a consulta usa índice e o volume é baixo.
- Ponto único de falha: se o único worker parar, os eventos se acumulam na outbox (não se perdem, mas atrasam). Precisa de monitoramento do backlog.
- A operação precisa garantir que **só uma instância** do worker rode; duas instâncias quebrariam a premissa de ordenação.
- Mais um processo para operar (deploy, logs, restart).
