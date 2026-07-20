# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs para a feature **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição da reunião técnica e do código do OMS existente — sem alterar a aplicação.

> Enunciado original do desafio (referência): o README do repositório base em [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). A transcrição permanece em [`TRANSCRICAO.md`](TRANSCRICAO.md).

---

## Sobre o desafio

O OMS em produção precisa notificar clientes B2B quando o status de um pedido muda, mas a decisão técnica existia apenas como call gravada. O objetivo deste trabalho foi transformar `TRANSCRICAO.md` + o código Node/TypeScript/Prisma em um pacote acionável (PRD, RFC, FDD, ADRs, Tracker), com rastreabilidade explícita de cada requisito e decisão — sem inventar escopo e sem tocar em `src/`, `prisma/` ou `tests/`.

O papel humano foi de maestro: extrair o que entrou / o que ficou fora, cruzar com os ganchos reais do código (`changeStatus`, `AppError`, auth, logger) e iterar os documentos até ficarem consistentes entre si e fiéis à reunião.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Cursor (Agent)** | Ferramenta principal: leitura do repositório e da transcrição, estruturação dos documentos, geração e refinamento do pacote Markdown |
| **Exploração dirigida no código** | Localização de padrões reais (`OrderService.changeStatus`, middlewares, Prisma schema) para a seção de integração do FDD e para o ADR de reuso |

## Workflow adotado

Ordem seguida (alinhada à sugestão do desafio):

1. **Contextualização** — leitura integral de `TRANSCRICAO.md` e dos pontos de integração no código (orders, errors, auth, logger, server, routes, schema).
2. **Filtro do que NÃO entra** — email, dashboard, rate limiting, multi-worker, sync HTTP, Redis — marcados como fora de escopo / aberto antes de escrever requisitos.
3. **ADRs primeiro** — seis decisões principais formalizadas em `docs/adrs/` (esqueleto do “como”).
4. **RFC** — proposta concisa para revisão, alternativas descartadas e questões em aberto, com links aos ADRs.
5. **FDD** — especificação implementável: fluxos, contratos HTTP, erros `WEBHOOK_*`, resiliência, observabilidade, integração com arquivos reais.
6. **PRD** — consolidação de produto/negócio por último.
7. **Tracker** — varredura dos documentos prontos mapeando cada item à transcrição ou ao código.
8. **README do processo** — este arquivo, ao final.

## Prompts customizados

### Prompt 1 — Extração e filtragem da transcrição

```text
Você é um analista técnico. Leia TRANSCRICAO.md por completo.

Produza TRÊS listas distintas, sem inventar nada:
1) DECIDIDO — decisões fechadas com timestamp + falante
2) REQUISITO — funcionais e não funcionais explícitos, com timestamp
3) FORA / ABERTO — itens descartados, adiados ou "observar depois"

Regras:
- Se não houver timestamp + falante, não inclua.
- Não transforme brainstorm em requisito.
- Destaque conflitos (ex.: 3 vs 5 retries) e qual lado venceu.
```

### Prompt 2 — FDD ancorado no código existente

```text
Com base nas decisões dos ADRs e na TRANSCRICAO, escreva docs/FDD.md.

Obrigações:
- Seção "Integração com o sistema existente" com ≥ 4 caminhos REAIS do repo
  (comece por src/modules/orders/order.service.ts#changeStatus).
- ≥ 4 endpoints HTTP com request/response de exemplo e status codes.
- Matriz de erros só com prefixo WEBHOOK_ (exceto reuso de UNAUTHORIZED/FORBIDDEN/VALIDATION_ERROR).
- NÃO copie o nível de detalhe do RFC; foque em "como construir".
- Cada regra de negócio deve ser rastreável a [hh:mm] Falante.

Se algo não tiver origem, omita — não invente.
```

## Iterações e ajustes

Foram **4 iterações principais** até o pacote final:

1. **Primeira geração genérica** — um rascunho inicial misturava RFC e FDD (payloads longos no RFC) e quase elevava rate limiting a requisito. Correção: separar alturas (RFC decide/abre; FDD implementa) e mover rate limiting para “questões em aberto / fora”.
2. **Alucinação de escopo** — apareceu “webhook inbound” e “fila Redis como fase 1” em trechos. Correção: prompt de filtragem FORA/ABERTO + checklist explícita contra o resumo da Larissa em `[09:48]`.
3. **Integração fraca com o código** — FDD citava módulos hipotéticos. Correção: ancorar em arquivos reais (`order.service.ts`, `auth.middleware.ts`, `app-error.ts`, `error.middleware.ts`, `server.ts`, `routes/index.ts`, `database.ts`, `schema.prisma`) e descrever o gancho `publishWebhookEvent(tx, ...)`.
4. **Tracker como rede de segurança** — ao preencher `TRACKER.md`, itens sem timestamp/caminho foram removidos ou reescritos. Ex.: meta quantitativa do PRD rederivada do acordo de &lt; 10s (`[09:02] Marcos`), não de SLOs inventados.

## Como navegar a entrega

Ordem sugerida de leitura:

| Ordem | Arquivo | Por quê |
|---|---|---|
| 1 | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária da reunião |
| 2 | [`docs/adrs/`](docs/adrs/) | Decisões fechadas (esqueleto) |
| 3 | [`docs/RFC.md`](docs/RFC.md) | Proposta para revisão |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como implementar |
| 5 | [`docs/PRD.md`](docs/PRD.md) | Por quê / o quê de produto |
| 6 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item |

### Estrutura dos arquivos entregues

```text
.
├── README.md                          ← este processo
├── TRANSCRICAO.md                     ← não alterado
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-retry-backoff-dlq.md
│       ├── ADR-003-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-004-at-least-once-x-event-id.md
│       ├── ADR-005-worker-polling-processo-separado.md
│       └── ADR-006-reuso-padroes-existentes.md
├── src/                               ← não alterado (contexto)
├── prisma/                            ← não alterado
└── tests/                             ← não alterado
```

## Resumo das decisões (atalho)

| ADR | Decisão |
|---|---|
| 001 | Outbox transacional no MySQL |
| 002 | Retry 5x + backoff + DLQ |
| 003 | HMAC-SHA256 + secret por endpoint + grace 24h |
| 004 | At-least-once + `X-Event-Id` |
| 005 | Worker separado + polling 2s |
| 006 | Reuso dos padrões do OMS |
