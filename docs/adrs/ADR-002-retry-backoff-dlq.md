# ADR-002 — Retry com backoff exponencial e DLQ

- **Data:** 2026-07-20
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Bruno (Pedidos), Marcos (PM)
- **Fonte:** TRANSCRICAO.md — [09:14]–[09:18] Diego, Bruno, Larissa, Marcos

## Status

Aceito

## Contexto

Endpoints de webhook dos clientes podem ficar offline, lentos ou retornar erro. Sem política de retry, notificações se perdem. Retry indefinido, por outro lado, deixa eventos pendurados para sempre quando o cliente desaparece. O time precisa de uma janela razoável de recuperação e de um destino claro para falhas permanentes, com capacidade de reprocessamento manual.

## Decisão

1. **5 tentativas** máximas por evento de entrega.
2. **Backoff exponencial** entre tentativas: `1m → 5m → 30m → 2h → 12h` (janela total ~15h desde a primeira falha até a última tentativa).
3. Após esgotar as tentativas, mover o evento para tabela separada `webhook_dead_letter` (payload, motivo da falha, timestamp), em vez de apenas marcar `failed` na outbox.
4. Reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| 3 tentativas (mais agressivo) | Janela curta demais (~30 min); não cobre manutenção planejada de 2h já vista em cliente ([09:16] Diego). |
| Retry indefinido com backoff | Evento fica pendurado para sempre se o cliente sumir ([09:15] Diego). |
| Marcar `failed` na própria outbox (sem tabela DLQ) | Polui a leitura da outbox operacional; tabela separada facilita evidência, debug e replay ([09:18] Diego). |

## Consequências

**Positivas**

- Cobre indisponibilidades de curto/médio prazo sem intervenção humana.
- DLQ limpa a outbox quente e preserva evidência para debug.
- Replay admin permite recuperação operacional sem redeploy.

**Negativas / trade-offs**

- Cliente offline por mais de ~15h perde a entrega automática (aceitável segundo PM — [09:17] Marcos).
- Operação precisa de processo/runbook para monitorar e reprocessar DLQ.
- Replay exige role `ADMIN` e auditoria de quem executou ([09:36] Sofia).
