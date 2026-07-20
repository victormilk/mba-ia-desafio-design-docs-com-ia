# FDD — Feature Design Document: Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks outbound para mudança de status de pedidos |
| **Status** | Pronto para implementação |
| **Autores** | Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma) |
| **Revisão de segurança** | Sofia — obrigatória antes do deploy |
| **ADRs relacionados** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) … [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |
| **RFC** | [RFC.md](RFC.md) |

---

## 1. Contexto e motivação técnica

O OMS já controla o ciclo de vida do pedido em `OrderService.changeStatus` com máquina de estados, estoque transacional e auditoria em `order_status_history`. Não existe mecanismo de notificação externa. Clientes B2B fazem polling em `GET /orders`. Esta feature preenche o vácuo com webhooks outbound desacoplados da transação de negócio via outbox.

## 2. Objetivos técnicos

1. Publicar evento na outbox **atomicamente** com a mudança de status.
2. Entregar HTTP ao endpoint do cliente com latência típica **&lt; 10s** (polling 2s + rede).
3. Resilir falhas temporárias (retry/backoff) e isolar falhas permanentes (DLQ + replay).
4. Autenticar entregas com HMAC-SHA256 e secret rotacionável por endpoint.
5. Expor CRUD de configuração, histórico de deliveries e replay admin reutilizando padrões do projeto.

## 3. Escopo e exclusões

**Incluído**

- Modelagem Prisma: `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter` (nomes finais a critério da implementação, semântica abaixo).
- Módulo `src/modules/webhooks/` (**a criar**) + entry-point `src/worker.ts` (**a criar**, espelhando `src/server.ts`).
- Integração em `changeStatus` via `publishWebhookEvent(tx, ...)`.
- Endpoints HTTP de configuração, deliveries, rotação de secret e replay DLQ.

**Excluído (confirmado na reunião)**

- Email de alerta quando webhook falha repetidamente.
- Dashboard visual / painel frontend.
- Rate limiting de saída.
- Arquivamento automático da outbox após 30 dias.
- Multi-worker / particionamento por `order_id`.
- Redis Streams ou broker externo.
- Envio síncrono de webhook.

## 4. Modelagem de dados (alvo)

### 4.1 `webhook_endpoints`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK — padrão do projeto |
| `customerId` | UUID | FK → `customers` |
| `url` | string | HTTPS obrigatório |
| `secret` | string | Secret atual |
| `previousSecret` | string? | Secret antiga durante grace |
| `previousSecretExpiresAt` | datetime? | Fim do grace (criação + 24h) |
| `eventStatuses` | JSON / enum[] | Status que o endpoint escuta |
| `active` | boolean | |
| `createdAt` / `updatedAt` | datetime | |

### 4.2 `webhook_outbox`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | = `event_id` enviado em `X-Event-Id` |
| `webhookEndpointId` | UUID | Destino |
| `orderId` | UUID | |
| `customerId` | UUID | |
| `eventType` | string | `order.status_changed` |
| `payload` | JSON | **Snapshot** renderizado na inserção |
| `status` | enum | `PENDING`, `PROCESSING`, `DELIVERED`, `FAILED` |
| `attempts` | int | |
| `nextAttemptAt` | datetime | |
| `lastError` | string? | |
| `createdAt` | datetime | Índice composto com `status` |

### 4.3 `webhook_deliveries`

Registro das últimas tentativas de entrega (sucesso/falha) para `GET /webhooks/:id/deliveries` — payload, response status/body truncado, latência, timestamp.

### 4.4 `webhook_dead_letter`

Cópia do evento + motivo + `failedAt` após 5 tentativas; removido/recriado como pendente no replay.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

```
1. OrderService.changeStatus inicia prisma.$transaction
2. Valida transição (canTransition) e ajusta estoque se necessário
3. update order + insert order_status_history
4. publishWebhookEvent(tx, order, from, to):
   a. Carrega endpoints ativos do customerId com eventStatuses contendo `to`
   b. Se nenhum endpoint → no-op (não insere)
   c. Para cada endpoint: monta snapshot do payload; se size > 64KB → erro WEBHOOK_PAYLOAD_TOO_LARGE (falha a transação)
   d. insert webhook_outbox (status=PENDING, attempts=0, nextAttemptAt=now, id=uuid)
5. Commit da transação
```

### 5.2 Processamento pelo worker

```
loop a cada 2s:
  1. SELECT ... FROM webhook_outbox
       WHERE status IN ('PENDING') AND nextAttemptAt <= now()
       ORDER BY created_at ASC
       LIMIT N
     (opcional: marcar PROCESSING com claim otimista)
  2. Para cada evento:
     a. POST url com body=payload, headers (ver §6.7)
     b. Timeout 10s
     c. Sucesso (2xx): status=DELIVERED; gravar delivery
     d. Falha: attempts++; gravar delivery; se attempts >= 5 → mover para DLQ e marcar FAILED;
        senão calcular nextAttemptAt pelo backoff e voltar a PENDING
```

### 5.3 Retry / backoff

| Tentativa após falha # | Delay até próxima |
|---|---|
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |
| Após 5ª falha | DLQ |

### 5.4 DLQ e replay

```
POST /admin/webhooks/dead-letter/:id/replay  (role ADMIN)
  1. Carrega DLQ por id
  2. Reinsere na outbox como PENDING (novo ciclo de attempts=0) — preservar event_id original
  3. Remove ou marca DLQ como replayed
  4. Loga auditoria: userId do JWT, deadLetterId, timestamp
```

## 6. Contratos públicos

Todos os endpoints de configuração exigem `Authorization: Bearer <JWT>` (`authenticate`). Replay exige `requireRole('ADMIN')`.

### 6.1 `POST /webhooks` — criar endpoint

**Request**

```json
{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://hooks.atlas.example/oms",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `201`**

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://hooks.atlas.example/oms",
  "secret": "whsec_8f3a2c1b0e9d7a6b5c4d3e2f1a0b9c8d",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-07-20T12:00:00.000Z"
}
```

> `secret` é gerada pelo servidor e retornada **somente** na criação (e na rotação).

**Erros:** `400 WEBHOOK_INVALID_URL` (http / URL inválida), `400 VALIDATION_ERROR`, `404` customer, `401`.

---

### 6.2 `GET /webhooks` — listar por customer

**Query:** `customerId` (obrigatório), paginação alinhada ao padrão do projeto (`page`, `pageSize`).

**Response `200`**

```json
{
  "items": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "url": "https://hooks.atlas.example/oms",
      "eventStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-20T12:00:00.000Z",
      "updatedAt": "2026-07-20T12:00:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

> Secret **não** é retornada na listagem.

---

### 6.3 `PATCH /webhooks/:id` — editar

**Request**

```json
{
  "url": "https://hooks.atlas.example/oms/v2",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200`**

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://hooks.atlas.example/oms/v2",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-20T13:00:00.000Z"
}
```

> Secret **não** é retornada no PATCH.

**Erros:** `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`.

---

### 6.4 `DELETE /webhooks/:id` — remover

**Request:** sem body; `id` no path.

**Response `204`** (sem body).

**Erros:** `404 WEBHOOK_NOT_FOUND`.

---

### 6.5 `POST /webhooks/:id/rotate-secret` — rotacionar secret

**Request:** body vazio.

**Response `200`**

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "secret": "whsec_novo_valor_gerado",
  "previousSecretExpiresAt": "2026-07-21T12:00:00.000Z"
}
```

Durante 24h o worker aceita assinar com a secret atual; se o cliente ainda validar só a antiga, a verificação do lado dele usa a antiga até migrar. Do nosso lado: assinamos sempre com a **secret atual**; o grace é para o **cliente** validar ambas.

---

### 6.6 `GET /webhooks/:id/deliveries` — histórico

**Query:** `limit` (default 100, max 100).

**Response `200`**

```json
{
  "items": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "eventId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "success": true,
      "httpStatus": 200,
      "latencyMs": 142,
      "requestPayload": { "event_id": "9b1deb4d-...", "event_type": "order.status_changed" },
      "responseBody": "{\"ok\":true}",
      "attemptedAt": "2026-07-20T12:00:02.100Z"
    }
  ]
}
```

---

### 6.7 Entrega outbound (não é endpoint nosso — contrato do POST ao cliente)

**Request do worker → URL do cliente**

Headers:

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do outbox |
| `X-Signature` | HMAC-SHA256 hex/base64 do body (definir encoding na implementação; documentar no portal) |
| `X-Timestamp` | ISO 8601 do momento do envio |
| `X-Webhook-Id` | id do endpoint |

**Body (exemplo)**

```json
{
  "event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-20T12:00:00.000Z",
  "order_id": "1a2b3c4d-5e6f-7890-abcd-ef1234567890",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "total_cents": 15990
}
```

> Sem `items`. Cliente detalha via `GET /orders/:id` se precisar.

**Response esperada do cliente:** qualquer `2xx` = sucesso. Demais status / timeout / erro de rede = falha para retry.

---

### 6.8 `POST /admin/webhooks/dead-letter/:id/replay`

**Auth:** JWT + role `ADMIN`.

**Request:** sem body; `id` da DLQ no path.

**Response `200`**

```json
{
  "deadLetterId": "dlq-uuid",
  "outboxId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "PENDING",
  "replayedBy": "user-admin-uuid",
  "replayedAt": "2026-07-20T15:00:00.000Z"
}
```

**Erros:** `403 FORBIDDEN`, `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.

## 7. Matriz de erros (`WEBHOOK_*`)

| Código | HTTP | Quando |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL não-HTTPS ou inválida |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige secret ausente/inválida |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot &gt; 64KB na publicação |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | Tentativa de operar endpoint inativo quando a regra exigir ativo |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de DLQ inexistente |
| `WEBHOOK_DELIVERY_FAILED` | — (interno/log) | Falha de entrega no worker (não necessariamente resposta HTTP da API) |
| `VALIDATION_ERROR` | 400 | Zod (reuso do middleware) |
| `FORBIDDEN` | 403 | Replay sem role ADMIN |
| `UNAUTHORIZED` | 401 | JWT ausente/inválido |

## 8. Estratégias de resiliência

| Mecanismo | Valor |
|---|---|
| Timeout HTTP do worker | 10 segundos |
| Polling | 2 segundos |
| Retry | 5 tentativas |
| Backoff | 1m / 5m / 30m / 2h / 12h |
| Fallback permanente | DLQ + replay manual admin |
| Deduplicação | Cliente via `X-Event-Id` (at-least-once) |
| Limite de payload | 64KB → erro (não truncar) |
| TLS | HTTPS obrigatório no cadastro |

## 9. Observabilidade

### Métricas (sugeridas)

- `webhook_outbox_pending_count`
- `webhook_delivery_success_total` / `webhook_delivery_failure_total`
- `webhook_delivery_latency_ms` (histogram)
- `webhook_dlq_count`
- `webhook_worker_poll_duration_ms`

### Logs (Pino)

- Eventos estruturados: `webhook_enqueued`, `webhook_delivery_attempt`, `webhook_delivery_success`, `webhook_delivery_failure`, `webhook_moved_to_dlq`, `webhook_dlq_replayed`.
- Campos: `eventId`, `webhookId`, `orderId`, `customerId`, `attempt`, `httpStatus`, `latencyMs`, `error`.
- **Redact** de secrets (estender `redactPaths` do logger para `*.secret`, `*.previousSecret`).
- Replay admin: logar `replayedBy` (user id).

### Tracing

- Propagar/gerar `requestId` / correlation id do worker por tentativa de entrega.
- Span lógico: `outbox.publish` (API) e `outbox.deliver` (worker), correlacionados pelo `event_id`.

## 10. Integração com o sistema existente

| Arquivo | Como integrar |
|---|---|
| `src/modules/orders/order.service.ts` | Dentro do `$transaction` de `changeStatus`, após update/history/estoque, chamar `publishWebhookEvent(tx, order, from, to)`. Se a inserção na outbox falhar, a transação inteira dá rollback — status não muda sem evento. |
| `src/modules/orders/order.status.ts` | Reusar `OrderStatus` / `canTransition` como fonte da verdade dos status filtráveis em `eventStatuses` do webhook (mesmos enums do Prisma). |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Criar erros de domínio webhook estendendo `AppError` / classes HTTP existentes, códigos `WEBHOOK_*`. |
| `src/middlewares/error.middleware.ts` | Sem mudança de contrato: `AppError` e Zod já são tratados; erros novos caem automaticamente no JSON `{ error: { code, message, details } }`. |
| `src/middlewares/auth.middleware.ts` | Rotas de CRUD usam `authenticate`; replay usa `requireRole('ADMIN')`. |
| `src/shared/logger/index.ts` | Worker e módulo usam o mesmo Pino; incluir secrets em `redactPaths`. |
| `src/server.ts` | Modelo de bootstrap; a implementação deve criar `src/worker.ts` (**ainda inexistente**) com Prisma próprio e graceful shutdown SIGINT/SIGTERM. |
| `src/routes/index.ts` | Registrar `buildWebhookRouter` em `/webhooks` e rota admin de DLQ. |
| `src/config/database.ts` | Worker chama `createPrismaClient()` (nova instância por processo, mesma `DATABASE_URL`). |
| `prisma/schema.prisma` | Adicionar models/enums da feature; IDs UUID `@default(uuid())` como o restante. |

### Função de publicação (contrato interno)

```ts
// proposta a criar — publishWebhookEvent no módulo webhooks (nome sugerido na implementação)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: { id: string; orderNumber: string; customerId: string; totalCents: number },
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>;
```

## 11. Dependências e compatibilidade

- Node.js + TypeScript + Express + Prisma + MySQL (já no projeto).
- Sem novas dependências de infra (sem Redis/fila).
- Biblioteca crypto nativa do Node para HMAC.
- Compatível com roles existentes `ADMIN` | `OPERATOR` (`UserRole` no Prisma).
- Não altera contratos atuais de `/orders`, exceto efeito colateral assíncrono (outbox) em `changeStatus`.

## 12. Critérios de aceite técnicos

1. Mudança de status e insert na outbox são atômicos (teste de rollback).
2. Sem endpoints ativos para o `toStatus` → nenhuma linha na outbox.
3. Worker entrega &lt; 10s em caminho feliz (poll 2s + HTTP).
4. Falha simulada → respeita backoff; após 5 falhas → DLQ.
5. Replay admin recoloca como `PENDING` e registra auditoria.
6. Assinatura HMAC verificável com a secret do endpoint.
7. URL `http://` rejeitada na criação/edição.
8. Payload &gt; 64KB falha a publicação com `WEBHOOK_PAYLOAD_TOO_LARGE`.
9. Mesmo `X-Event-Id` em retries do mesmo evento.
10. Códigos de erro usam prefixo `WEBHOOK_` onde aplicável.

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Regressão em `changeStatus` (transação mais longa) | Média | Alto | Manter publish enxuto; testes de integração do fluxo de status existente |
| Vazamento de secret em logs | Média | Alto | Redact Pino; revisão Sofia; secret só em create/rotate |
| Worker parado sem alerta | Média | Alto | Métrica de pending + alerta operacional |
| Duplicatas no cliente | Alta (by design) | Médio | Documentar dedup por `X-Event-Id` |
| Contenção no MySQL na outbox | Baixa (início) | Médio | Índice status+created_at; batch pequeno; observar Q1 rate limit |

## 14. Decisões de implementação secundárias (fechadas na call)

| Tópico | Decisão | Fonte |
|---|---|---|
| ID da outbox | UUID (padrão do projeto) | [09:51] Larissa |
| Payload | Snapshot na inserção (não renderizar no envio) | [09:51]–[09:52] Larissa, Diego, Bruno |
| Timeout | 10s | [09:42] Diego |
| Limite body | 64KB, erro se passar | [09:23]–[09:24] Sofia, Diego |
| Ordering | Por order_id enquanto single-worker | [09:12]–[09:13] |
