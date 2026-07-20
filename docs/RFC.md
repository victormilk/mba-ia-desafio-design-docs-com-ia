# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Título** | Outbound Webhooks para mudança de status de pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-07-20 |
| **Revisores** | Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Relacionados** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-retry-backoff-dlq.md), [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md), [ADR-004](adrs/ADR-004-at-least-once-x-event-id.md), [ADR-005](adrs/ADR-005-worker-polling-processo-separado.md), [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |

---

## 1. Resumo executivo (TL;DR)

Propusemos um **sistema de webhooks outbound** que notifica clientes B2B quando o status de um pedido muda, sem polling no `GET /orders`. A abordagem é **Transactional Outbox no MySQL** (mesma transação de `changeStatus`), com **worker em processo separado** em polling de 2s, **retry com backoff** (5 tentativas → DLQ), **HMAC-SHA256** por endpoint e garantia **at-least-once** via `X-Event-Id`. Escopo desta fase: CRUD de configuração + entregas + replay admin. Email, dashboard e rate limiting de saída ficam fora ou em aberto.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pedem notificação em tempo real de mudança de status. Hoje eles fazem polling em `GET /orders`, o que torna a integração lenta e cara. A Atlas condicionou a continuidade do contrato à entrega até o fim do trimestre (prazo alinhado a fim de novembro / ~3 sprints).

Para eles, latência **abaixo de 10 segundos** já é "tempo real". O fluxo é **somente outbound** (nós → cliente). O OMS atual não possui eventos, filas nem notificações externas — e a mudança de status já é uma transação pesada (pedido + histórico + estoque).

## 3. Proposta técnica

### 3.1 Visão geral

```
[API] PATCH /orders/:id/status
        │
        ▼
 OrderService.changeStatus  ──$transaction──► update order
                                              insert history
                                              adjust stock
                                              insert webhook_outbox (snapshot)
        │
        ▼
[Worker separado] poll 2s → HTTP POST ao endpoint do cliente
                              (HMAC, headers, timeout 10s)
                              retry / DLQ / deliveries
```

### 3.2 Componentes (nível arquitetural)

| Componente | Responsabilidade |
|---|---|
| Módulo `webhooks` | CRUD de endpoints, rotação de secret, histórico de deliveries, replay DLQ |
| Outbox MySQL | Persistência atômica do evento junto da mudança de status |
| Worker (`src/worker.ts`, **a criar**) | Polling, envio HTTP, retry/backoff, DLQ, registro de deliveries |
| Assinatura | HMAC-SHA256 do body; secret por endpoint; rotação com grace 24h |

### 3.3 Semântica de entrega

- **At-least-once**; cliente deduplica por `X-Event-Id`.
- Filtro de eventos (lista de status) aplicado **na inserção** da outbox.
- Payload enxuto (`order.status_changed` + campos básicos; **sem items**).
- TLS obrigatório (`https` only); payload máximo 64KB (erro se ultrapassar).
- Single-worker nesta fase; ordering por `order_id` enquanto houver um consumidor.

Detalhamento de fluxos, contratos HTTP, matriz de erros e integração pontual com o código → **[FDD](FDD.md)**. Decisões fechadas → **ADRs** listados nos metadados.

## 4. Alternativas consideradas

### 4.1 HTTP síncrono dentro de `OrderService.changeStatus`

Disparar o webhook na mesma request/transação da mudança de status.

- **Trade-off do descarte:** cliente lento bloqueia a API; cliente offline obrigaria rollback da mudança de status — inaceitável ([09:04] Bruno). Decisão: outbox assíncrona ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

### 4.2 Redis Streams / broker externo

Publicar eventos em Redis (ou fila equivalente) em vez de tabela outbox no MySQL.

- **Trade-off do descarte:** nova infra (ex. Redis Cluster) para time pequeno = overengineering; MySQL existente resolve ([09:07] Larissa, Diego).

### 4.3 (Complementar) Trigger MySQL / worker no processo da API

Descartados em favor de polling em processo separado — ver [ADR-005](adrs/ADR-005-worker-polling-processo-separado.md).

## 5. Questões em aberto

| # | Questão | Origem | Estado |
|---|---|---|---|
| Q1 | **Rate limiting de saída** — se um cliente tiver dezenas de mudanças/minuto, devemos limitar chamadas HTTP? | [09:38]–[09:39] Diego, Larissa | Observar em produção e decidir depois; **fora do escopo** desta fase |
| Q2 | **Escalabilidade multi-worker / ordering** — como particionar (por `order_id`) ou usar lock pessimista quando precisarmos de mais de um worker? | [09:12]–[09:13] Diego, Bruno, Larissa | Limitação conhecida; single-worker agora |
| Q3 | **Notificação por email** quando o webhook falha N vezes seguidas | [09:37] Marcos, Larissa | Adiado para próxima fase |
| Q4 | **Dashboard visual** de webhooks para o cliente | [09:39]–[09:40] Marcos, Larissa | Fora de escopo; projeto separado de frontend |

## 6. Impacto e riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| Crescimento da tabela outbox | Degradação do polling | Índices em status/created_at; arquivamento ~30d como follow-up (fora do escopo) |
| Cliente não implementa dedup | Processamento duplicado | Documentação destacada no portal; `X-Event-Id` obrigatório |
| Vazamento de secret no lado do cliente | Impersonação de entregas | Secret por endpoint + rotação com grace 24h; revisão de segurança da Sofia pré-deploy |
| Worker único como SPOF | Atraso nas entregas se o processo cair | Processo separado da API; monitoramento/alertas do worker (métricas no FDD) |
| Atraso no prazo Atlas (fim de novembro) | Risco comercial de churn | Estimativa de 3 sprints + 2 dias de revisão de segurança |

## 7. Decisões relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL |
| [ADR-002](adrs/ADR-002-retry-backoff-dlq.md) | Retry + backoff + DLQ |
| [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 e secret por endpoint |
| [ADR-004](adrs/ADR-004-at-least-once-x-event-id.md) | At-least-once + `X-Event-Id` |
| [ADR-005](adrs/ADR-005-worker-polling-processo-separado.md) | Worker separado + polling 2s |
| [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) | Reuso de padrões do OMS |

## 8. Pedido aos revisores

Pedimos feedback especialmente sobre: (a) aceitação da semântica at-least-once pelos clientes, (b) se o grace period de 24h na rotação de secret é suficiente, (c) se single-worker é aceitável até a primeira medição de volume pós-lançamento.
