# TRACKER — Rastreabilidade Design Docs ↔ Transcrição / Código

Tabela de referência cruzada. Cada item documentado deve apontar origem em `TRANSCRICAO.md` ou no código-fonte. Se não houver localização válida, o item não deveria existir na documentação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes Atlas, MaxDistribuição e Nova Cargo pedem notificação | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Polling em GET /orders torna integração lenta/cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco comercial | Atlas pode migrar se não entregar até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência &lt; 10s aceita como tempo real | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-01 | docs/PRD.md | Escopo | Fluxo somente outbound (nós → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | POST criar webhook; secret gerada pelo sistema | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | GET listar webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH editar webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | DELETE remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status por endpoint na inserção da outbox | TRANSCRICAO | [09:33] Marcos / [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Publicação atômica com mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega HTTP com payload order.status_changed sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries (últimos ~100) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay DLQ admin com auditoria | TRANSCRICAO | [09:18] Diego / [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego / [09:44] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | HTTPS obrigatório; HTTP rejeitado | TRANSCRICAO | [09:23] Sofia |
| PRD-OUT-01 | docs/PRD.md | Fora de Escopo | Email de alerta por falhas consecutivas adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de Escopo | Dashboard visual fora de escopo | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída observar depois | TRANSCRICAO | [09:39] Larissa |
| PRD-MET-01 | docs/PRD.md | Métrica | Meta de latência &lt; 10s derivada do acordo com clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-02 | docs/PRD.md | Métrica | Janela de retry 5x / backoff ~15h como meta de resiliência | TRANSCRICAO | [09:15]–[09:17] Diego |
| PRD-MET-03 | docs/PRD.md | Métrica | Adoção 3/3 clientes demandantes até o prazo | TRANSCRICAO | [09:00] Marcos / [09:45] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Churn Atlas por atraso no go-live | TRANSCRICAO | [09:00] Marcos / [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente sem dedup processa duplicatas | TRANSCRICAO | [09:25] Sofia / [09:26] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP 10s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | 5 retries; backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15]–[09:17] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256; secret por endpoint | TRANSCRICAO | [09:20]–[09:21] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | At-least-once + X-Event-Id | TRANSCRICAO | [09:24]–[09:26] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Payload máx. 64KB; erro se passar | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança Sofia ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-PLAN-01 | docs/PRD.md | Cronograma | Estimativa total 3 sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-TLDR-01 | docs/RFC.md | Proposta | Outbox MySQL + worker + retry + HMAC + at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / broker externo | TRANSCRICAO | [09:07] Larissa / Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger MySQL para notificar worker | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Multi-worker / particionamento | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Email de alerta (adiado) | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Dashboard visual (fora) | TRANSCRICAO | [09:40] Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Crescimento da outbox | TRANSCRICAO | [09:07] Bruno / [09:08] Diego |
| RFC-DEC-01 | docs/RFC.md | Decisão | Links para ADRs 001–006 | TRANSCRICAO | [09:48] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Insert outbox na mesma $transaction de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker poll 2s batch de pendentes | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Retry/backoff e move para DLQ | TRANSCRICAO | [09:15]–[09:18] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Replay POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks cria endpoint + devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks lista por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id edita | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id remove | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST rotate-secret com grace 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Payload outbound sem items + headers | TRANSCRICAO | [09:43]–[09:45] Diego / Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Replay DLQ exige ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | Prefixo WEBHOOK_ nos códigos | TRANSCRICAO | [09:28]–[09:29] Bruno / Larissa |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10s | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino estruturados (reuso) | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Auditoria de quem fez replay | TRANSCRICAO | [09:36] Sofia |
| FDD-DATA-01 | docs/FDD.md | Modelagem | UUID como PK da outbox | TRANSCRICAO | [09:51] Larissa |
| FDD-DATA-02 | docs/FDD.md | Modelagem | Snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-DATA-03 | docs/FDD.md | Modelagem | DLQ em tabela separada | TRANSCRICAO | [09:18] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | Extender changeStatus em order.service.ts | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reusar canTransition / OrderStatus | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros via AppError / http-errors | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Middleware de erro centralizado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate + requireRole ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Logger Pino compartilhado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Entry-point espelhando server.ts | CODIGO | src/server.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Registrar rotas em routes/index.ts | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | PrismaClient por processo | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Novos models no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INT-11 | docs/FDD.md | Integração | Classes de erro de domínio existentes como referência | CODIGO | src/shared/errors/http-errors.ts |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox transacional no MySQL | TRANSCRICAO | [09:06]–[09:08] Diego |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Descarte de sync HTTP e Redis | TRANSCRICAO | [09:04] Bruno / [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | 5 retries + backoff + DLQ separada | TRANSCRICAO | [09:15]–[09:18] Diego |
| ADR-002-ALT | docs/adrs/ADR-002-retry-backoff-dlq.md | Trade-off | Descarte de 3 tentativas e retry infinito | TRANSCRICAO | [09:15]–[09:16] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256; secret por endpoint; grace 24h | TRANSCRICAO | [09:20]–[09:22] Sofia |
| ADR-003-ALT | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Trade-off | Descarte de secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24]–[09:26] Diego |
| ADR-004-ALT | docs/adrs/ADR-004-at-least-once-x-event-id.md | Trade-off | Descarte de exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-polling-processo-separado.md | Decisão | Worker separado; polling 2s; single-worker | TRANSCRICAO | [09:09]–[09:13] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-worker-polling-processo-separado.md | Trade-off | Descarte de worker na API e trigger MySQL | TRANSCRICAO | [09:09] Diego / [09:11] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso AppError, Pino, módulo, WEBHOOK_ | TRANSCRICAO | [09:27]–[09:30] Bruno / Larissa |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Referência a order.service, errors, auth, logger, server | CODIGO | src/modules/orders/order.service.ts |
| ADR-006-CODE2 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | requireRole existente para ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| CODE-TX-01 | docs/FDD.md | Restrição | changeStatus já atualiza order + history + stock na mesma tx | CODIGO | src/modules/orders/order.service.ts |
| CODE-ROLE-01 | docs/FDD.md | Restrição | Roles ADMIN e OPERATOR no schema | CODIGO | prisma/schema.prisma |
| CODE-STATUS-01 | docs/PRD.md | Restrição | Enum OrderStatus (PENDING…CANCELLED) | CODIGO | prisma/schema.prisma |
| CODE-ERR-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Padrão | InvalidStatusTransitionError / InsufficientStockError como modelo de código | CODIGO | src/shared/errors/http-errors.ts |

---

## Notas de cobertura

- Itens sem origem identificável foram **excluídos** dos documentos (nada inventado sem fonte).
- Timestamps no formato `[hh:mm] Nome` conforme `TRANSCRICAO.md`.
- Caminhos de código verificados no repositório base (não alterado nesta entrega).
