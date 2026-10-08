# ADR-004: Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Sofia (Segurança), Larissa, Bruno, Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Os webhooks vão levar dados de pedido para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar que a requisição veio de nós e que o payload não foi adulterado no caminho [09:19 Sofia]. O fluxo é só de saída (outbound); clientes não enviam webhooks para nós [09:02 Marcos, 09:03 Sofia].

Já tivemos cliente que vazou secret em log da própria aplicação [09:22 Diego].

## Decisão

1. Assinar o **corpo da requisição com HMAC-SHA256** e enviar a assinatura no header `X-Signature` [09:20 Sofia, 09:22 Sofia].
2. **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma [09:21 Sofia]. A secret é **gerada por nós** e devolvida ao cliente na criação [09:31 Marcos].
3. A configuração do webhook guarda `url`, `secret`, `customer_id` e estado ativo [09:21 Bruno, 09:21 Sofia].
4. **Rotação via API**: o cliente pede nova secret; a antiga continua válida por **24 horas** em paralelo, depois é descartada [09:21 Sofia].
5. URL do webhook **obrigatoriamente `https`**; `http` é recusado por validação no schema Zod. Não é decisão arquitetural, mas fica registrado aqui por ser requisito de segurança [09:23 Sofia].
6. Código de geração de secret e de HMAC passa por revisão da Sofia (2 dias úteis) antes do deploy [09:46 Sofia].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Se vaza uma, vaza tudo [09:21 Sofia]. |
| **Rotação sem período de convivência** | O cliente não teria tempo de atualizar os sistemas dele; eventos assinados com a secret nova seriam rejeitados até ele migrar [09:21 Sofia]. |
| **Outros algoritmos de assinatura** | HMAC-SHA256 é o padrão de mercado e todo cliente sério tem biblioteca para isso [09:20 Sofia]. |

## Consequências

**Positivas**
- Cliente valida origem e integridade com bibliotecas comuns.
- Vazamento de uma secret afeta só um endpoint, e pode ser contido com rotação.

**Negativas / trade-offs**
- A secret precisa ser armazenada de forma recuperável (para assinar), não pode ser só um hash como senha. A forma de armazenamento/proteção em repouso não foi discutida e fica para a revisão de segurança.
- Durante as 24 h de grace period existem duas secrets válidas; o formato de como isso aparece para o cliente (uma ou duas assinaturas no header) é detalhado no FDD e precisa da validação da Sofia.
- A assinatura cobre só o corpo; o `X-Timestamp` não entra no HMAC (decisão literal da reunião), então a proteção contra replay depende do cliente usar o `X-Event-Id`/timestamp do payload.
