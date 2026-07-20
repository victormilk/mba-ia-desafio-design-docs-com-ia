# ADR-004 — Garantia at-least-once com X-Event-Id

- **Data:** 2026-07-20
- **Decisores:** Diego (Plataforma), Sofia (Segurança), Larissa (Tech Lead), Marcos (PM)
- **Fonte:** TRANSCRICAO.md — [09:24]–[09:26] Diego, Bruno, Sofia, Marcos, Larissa

## Status

Aceito

## Contexto

Com outbox + worker + retries, falhas entre "HTTP enviado com sucesso" e "marcar entregue no banco" (ou reprocessamento de DLQ) podem fazer o mesmo evento ser entregue mais de uma vez. Exactly-once end-to-end exigiria coordenação complexa entre plataforma e cliente. O mercado (Stripe, GitHub) usa at-least-once com identificador estável para deduplicação no receptor.

## Decisão

1. Garantia de entrega: **at-least-once**.
2. Gerar `event_id` (UUID) no momento em que o evento entra na outbox.
3. Enviar `event_id` no header **`X-Event-Id`** em toda entrega HTTP.
4. O cliente é responsável por **deduplicar** pelo `event_id`.
5. Documentar a semântica de forma destacada no portal de desenvolvedores.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| Exactly-once | Exige coordenação dos dois lados; complexidade desproporcional. At-least-once + `event_id` cobre ~99% dos casos ([09:25] Diego). |
| Sem identificador de evento | Cliente não consegue deduplicar entregas repetidas de forma confiável ([09:25] Bruno/Diego). |

## Consequências

**Positivas**

- Implementação alinhada ao padrão de mercado.
- Deduplicação simples e estável no lado do cliente.
- Compatível com retries, DLQ replay e falhas de commit pós-envio.

**Negativas / trade-offs**

- Responsabilidade de idempotência fica no cliente ([09:25] Sofia) — mitigada por documentação clara ([09:26] Marcos).
- Clientes que não implementarem dedup podem processar o mesmo evento duas vezes.
