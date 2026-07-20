# ADR-005 — Worker em processo separado com polling

- **Data:** 2026-07-20
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Bruno (Pedidos), Marcos (PM)
- **Fonte:** TRANSCRICAO.md — [09:08]–[09:13] Diego, Bruno, Larissa, Marcos

## Status

Aceito

## Contexto

Eventos na outbox precisam ser consumidos e entregues via HTTP. MySQL não oferece `LISTEN/NOTIFY` nativo como Postgres; triggers SQL não notificam processo externo de forma limpa. O requisito de negócio aceita latência abaixo de 10 segundos como "tempo real" ([09:02] Marcos). Se o worker rodar no mesmo processo da API, restart da API derruba o consumidor.

## Decisão

1. Worker como **processo Node separado**: criar entry-point `src/worker.ts` (**ainda inexistente**) + script `npm run worker` (análogo ao existente `src/server.ts`).
2. **Polling** a cada **2 segundos**: buscar eventos pendentes mais antigos em batch, processar, marcar.
3. Mesmo banco e mesma stack Prisma; **PrismaClient novo por processo** (mesmo `DATABASE_URL`).
4. Nesta fase: **single-worker**. Ordering implícita por `created_at` / `order_id` enquanto houver um único consumidor.
5. Escalabilidade multi-worker (partição por `order_id` ou lock pessimista) fica como **limitação conhecida / futuro**.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| Worker dentro do processo da API | Restart da API perde o worker ([09:11] Diego). |
| Trigger MySQL para "notificar" o worker | Trigger só executa SQL; improvisar notificação (arquivo/endpoint) fica esquisito. MySQL não tem listener nativo tipo NOTIFY ([09:09] Diego). |
| Multi-worker paralelo agora | Perde garantia de ordering por `order_id`; problema do futuro ([09:12]–[09:13] Diego, Larissa). |

## Consequências

**Positivas**

- Latência típica ≤ 2s de polling + tempo de HTTP; atende &lt; 10s ([09:10] Marcos).
- Ciclo de vida do consumidor independente da API.
- Implementação simples, observável e alinhada à stack atual.

**Negativas / trade-offs**

- Latência mínima no pior caso ≈ 2s (aceita explicitamente).
- Operação precisa de dois processos (API + worker).
- Ordering global não é garantida; ordering por `order_id` só enquanto single-worker ([09:13] Larissa).
