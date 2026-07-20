# ADR-001 — Padrão Outbox no MySQL

- **Data:** 2026-07-20
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Fonte:** TRANSCRICAO.md — [09:06]–[09:08] Diego, Larissa, Bruno

## Status

Aceito

## Contexto

A mudança de status de pedido em `OrderService.changeStatus` já executa, na mesma transação Prisma, update em `orders`, insert em `order_status_history` e ajuste de estoque. Clientes B2B precisam ser notificados quando o status muda. Disparar HTTP de forma síncrona dentro dessa transação acoplaria a disponibilidade do cliente externo à consistência do pedido: cliente lento trava a API e cliente indisponível forçaria rollback indevido da mudança de status.

O time é pequeno e a stack atual já usa MySQL via Prisma. Não há Redis, filas ou barramento de eventos no repositório.

## Decisão

Adotar o padrão **Transactional Outbox** no MySQL existente:

1. Na mesma transação que altera o status do pedido, inserir uma linha em `webhook_outbox` com o evento (snapshot do payload).
2. Um worker separado lê a outbox e dispara as chamadas HTTP aos endpoints cadastrados.
3. Índice em `status` + `created_at` para o worker consumir apenas pendentes em batch pequeno.
4. Arquivamento de linhas entregues após ~30 dias fica **fora do escopo** desta feature.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| HTTP síncrono dentro de `changeStatus` | Acopla latência/disponibilidade do cliente à transação de pedido; rollback de status por falha de webhook é inaceitável ([09:04] Bruno). |
| Redis Streams / broker externo | Exigiria nova infra (ex.: Redis Cluster). Overengineering para time pequeno; MySQL já resolve ([09:07] Larissa, Diego). |

## Consequências

**Positivas**

- Atomicidade: se a transação commitou, o evento existe; se deu rollback, o evento some junto — sem inconsistência possível.
- Sem nova infraestrutura de mensageria.
- Reaproveita `DATABASE_URL` e Prisma já existentes.

**Negativas / trade-offs**

- A outbox cresce no mesmo banco operacional; precisa de índice e, no futuro, política de arquivamento.
- Latência de entrega depende do polling do worker (não é push imediato do banco).
- Throughput limitado pelo MySQL e pelo worker single-process nesta fase.
