# ADR-006 — Reuso dos padrões existentes do projeto

- **Data:** 2026-07-20
- **Decisores:** Bruno (Pedidos), Larissa (Tech Lead), Diego (Plataforma)
- **Fonte:** TRANSCRICAO.md — [09:27]–[09:30] Bruno, Diego, Larissa; código em `src/`

## Status

Aceito

## Contexto

O OMS já possui convenções claras de organização e tratamento de erros. Introduzir um módulo de webhooks com stack ou estilo paralelo aumentaria custo de manutenção e inconsistência operacional. A feature deve se encaixar no que o time já conhece.

## Decisão

1. Novo domínio em `src/modules/webhooks/` (**a criar**) seguindo o padrão dos demais módulos: `controller`, `service`, `repository`, `routes`, `schemas` (Zod).
2. Lógica do consumidor em arquivo do módulo (ex.: `webhook.worker.ts` / `webhook.processor.ts`, **a criar**), acionado por `src/worker.ts` (**a criar**).
3. Erros de domínio via `AppError` / classes em `src/shared/errors/`, com códigos prefixados **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
4. Logger **Pino** existente (`src/shared/logger/index.ts`); sem novo framework de logs.
5. Middleware de erro centralizado (`src/middlewares/error.middleware.ts`) continua tratando `AppError`, Zod e Prisma — sem fork.
6. Autenticação JWT e `requireRole('ADMIN')` reutilizados de `src/middlewares/auth.middleware.ts` no replay de DLQ.
7. Integração com pedidos via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada de dentro da transação de `OrderService.changeStatus` em `src/modules/orders/order.service.ts`.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| Módulo/worker com padrões próprios (outro estilo de erro, logger diferente, pasta fora de `src/modules`) | Fragmenta a codebase; time já tem padrão claro e decidiu reuso máximo ([09:30] Larissa). |
| Injetar repository completo de webhooks no `OrderService` | Acoplamento desnecessário; função pura recebendo o `tx` da transação é suficiente ([09:41] Bruno, Diego). |

## Consequências

**Positivas**

- Curva de aprendizado baixa; code review alinhado ao restante do OMS.
- Erros e auth do replay caem no middleware/padrão já testados.
- Ponto de integração explícito e atômico com `changeStatus`.

**Negativas / trade-offs**

- `OrderService` passa a conhecer o gancho de publicação de webhook (acoplamento controlado via função pura).
- Prefixo `WEBHOOK_` e schemas Zod novos precisam ser mantidos consistentes com o restante do projeto.
- Worker tem PrismaClient próprio por processo — não compartilha pool com a API ([09:29]–[09:30] Bruno).

## Referências de código

| Arquivo / módulo | Papel no reuso |
|---|---|
| `src/modules/orders/order.service.ts` | Estender `changeStatus` para publicar na outbox na mesma `$transaction` |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Base para erros `WEBHOOK_*` |
| `src/middlewares/error.middleware.ts` | Tratamento centralizado sem alteração de contrato |
| `src/middlewares/auth.middleware.ts` | `authenticate` + `requireRole('ADMIN')` no replay |
| `src/shared/logger/index.ts` | Logger Pino compartilhado |
| `src/server.ts` | Modelo de entry-point; implementação criará `src/worker.ts` (ainda inexistente) |
| `src/routes/index.ts` | Registrar rotas do módulo webhooks |
