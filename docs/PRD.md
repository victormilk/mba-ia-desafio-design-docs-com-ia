# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks outbound de mudança de status de pedidos |
| **PM** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica (transcrição) — documentação formal |
| **Prazo-alvo** | Fim de novembro / ~3 sprints (+ revisão de segurança) |
| **Documentos técnicos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) |

---

## 1. Resumo e contexto da feature

Clientes B2B integrados ao OMS precisam saber, em quase tempo real, quando o status de um pedido muda (ex.: `PAID` → `SHIPPED`). Hoje a única opção é polling em `GET /orders`. Esta feature introduz **webhooks outbound**: a plataforma chama um endpoint HTTPS cadastrado pelo cliente sempre que um status de interesse muda, com autenticação por assinatura e histórico de entregas.

A decisão de construir a feature foi tomada em reunião técnica (~55 min) entre produto, engenharia e segurança; este PRD consolida o *porquê* e o *quê*. O *como* está no RFC/FDD/ADRs.

## 2. Problema e motivação

- **Dor atual:** Atlas Comercial, MaxDistribuição e Nova Cargo fazem polling contínuo; integração lenta e cara.
- **Risco comercial:** Atlas sinalizou possível migração para concorrente se a notificação não for entregue até o fim do trimestre.
- **Definição de “tempo real” acordada:** latência **abaixo de 10 segundos** é suficiente; não é necessário push sub-segundo.
- **Direção do fluxo:** somente **nós → cliente** (outbound). Clientes não enviam webhooks para o OMS nesta fase.

## 3. Público-alvo e cenários de uso

| Persona | Necessidade |
|---|---|
| Integrador B2B (ex. Atlas) | Receber POST assinado quando pedido muda de status; deduplicar por event id |
| Operador / usuário autenticado do OMS | Cadastrar, listar, editar e remover webhooks de um `customerId` via API |
| Admin da plataforma | Reprocessar eventos que caíram em DLQ; ação auditada |
| Time de engenharia OMS | Implementar sem nova infra de filas; reusar padrões do monólito |

**Cenário principal**

1. Integrador cadastra `https://...` + lista de status (`SHIPPED`, `DELIVERED`, …).
2. Operador altera status do pedido no OMS.
3. Em poucos segundos o endpoint do integrador recebe o evento assinado.
4. Integrador valida HMAC e deduplica por `X-Event-Id`.
5. Se o endpoint estiver fora, a plataforma retenta e, se esgotar, move para DLQ; admin pode reprocessar.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Entregar percepção de “tempo real” acordada com os clientes | Tempo entre commit da mudança de status e a 1ª tentativa HTTP do worker | **&lt; 10 segundos** ([09:02] Marcos; polling 2s em [09:09] Diego) |
| Cobrir indisponibilidade temporária do receptor | Janela automática de retry antes de DLQ | **5 tentativas** com backoff **1m/5m/30m/2h/12h** (~15h) ([09:15]–[09:17] Diego) |
| Atender os demandantes da reunião | Clientes com ≥ 1 webhook ativo e ≥ 1 delivery bem-sucedida em homologação/produção | **3/3** (Atlas, MaxDistribuição, Nova Cargo) até o prazo alinhado ([09:00]/[09:45] Marcos) |
| Garantir canal autenticado | Cadastros HTTP rejeitados; entregas com assinatura HMAC | **100%** HTTPS + `X-Signature` ([09:20]/[09:23] Sofia) |

## 5. Escopo

### 5.1 Incluído

- CRUD de configuração de webhooks (URL, filtros de status, ativo/inativo).
- Secret gerada pelo sistema + rotação com grace period de 24h.
- Publicação atômica de eventos na mudança de status do pedido.
- Worker assíncrono de entrega com retry/backoff e DLQ.
- Histórico de deliveries (últimos ~100 por endpoint).
- Replay manual de DLQ por admin.
- Documentação de integração no portal (responsabilidade do PM).

### 5.2 Fora de escopo

Itens **explicitamente descartados ou adiados** na reunião:

1. **Email de alerta** quando o webhook falha N vezes seguidas — adiado para próxima fase ([09:37] Larissa).
2. **Dashboard visual / painel** para o cliente ver webhooks — projeto separado de frontend ([09:39]–[09:40] Larissa).
3. **Rate limiting de saída** — observar e decidir depois ([09:38]–[09:39] Diego, Larissa).
4. **Arquivamento** de linhas entregues da outbox (~30 dias) — fora desta feature ([09:08] Diego).
5. **Multi-worker** e ordering global — limitação conhecida / futuro ([09:12]–[09:13]).
6. **Webhooks inbound** (cliente → OMS) — fora; só outbound ([09:02] Marcos, Sofia).

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| FR-01 | Cliente/usuário autenticado pode **criar** webhook (`POST`) informando URL, lista de status e `customerId`; secret é gerada e devolvida na criação |
| FR-02 | Usuário autenticado pode **listar** webhooks de um customer (`GET`) |
| FR-03 | Usuário autenticado pode **editar** webhook (`PATCH`) — URL, filtros, ativo |
| FR-04 | Usuário autenticado pode **remover** webhook (`DELETE`) |
| FR-05 | Por endpoint, é possível escolher **quais status** receber; filtro aplicado na inserção da outbox |
| FR-06 | Na mudança de status do pedido, eventos são enfileirados de forma **atômica** com a transação de negócio |
| FR-07 | Sistema **entrega HTTP** ao URL cadastrado com payload de `order.status_changed` (sem items) |
| FR-08 | Cliente pode consultar **histórico de deliveries** (`GET /webhooks/:id/deliveries`, ~últimos 100) |
| FR-09 | Admin pode **reprocessar** item de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) com auditoria de quem executou |
| FR-10 | Cliente pode **rotacionar secret**; secret antiga válida por 24h em paralelo |
| FR-11 | Entregas incluem headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` |
| FR-12 | URL do webhook deve ser **HTTPS**; HTTP é rejeitado na validação |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| NFR-01 | Latência percebida &lt; 10s (polling de 2s aceito) |
| NFR-02 | Timeout de chamada HTTP do worker: 10s |
| NFR-03 | Retry: 5 tentativas; backoff 1m/5m/30m/2h/12h |
| NFR-04 | Assinatura HMAC-SHA256 do body; secret por endpoint |
| NFR-05 | Garantia **at-least-once**; dedup no cliente via `X-Event-Id` |
| NFR-06 | Limite de payload 64KB; erro se ultrapassar (não truncar) |
| NFR-07 | Worker em processo separado da API |
| NFR-08 | Reuso de padrões: módulo em `src/modules`, `AppError`, Pino, middleware de erro, códigos `WEBHOOK_*` |
| NFR-09 | IDs UUID alinhados ao restante do schema Prisma |
| NFR-10 | Payload como **snapshot** no momento da mudança de status |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off |
|---|---|
| Outbox no MySQL vs Redis/fila | Menos infra; outbox cresce no DB operacional |
| Assíncrono vs síncrono | Desacopla API do cliente; introduz latência de polling |
| At-least-once vs exactly-once | Simplicidade; cliente implementa dedup |
| 5 retries vs 3 ou infinito | Cobre ~15h de indisponibilidade; além disso, DLQ manual |
| Single-worker | Ordering por pedido; escala horizontal fica para depois |
| Sem email/dashboard nesta fase | Menor escopo; operação e DX dependem de API + portal |

Detalhes: [RFC](RFC.md) e ADRs.

## 9. Dependências

- OMS existente (pedidos, JWT, roles `ADMIN`/`OPERATOR`, Prisma/MySQL).
- Disponibilidade do endpoint HTTPS do cliente.
- Portal de desenvolvedores (documentação de HMAC, headers, dedup) — Marcos.
- Janela de revisão de segurança (Sofia) antes do deploy — ≥ 2 dias úteis.
- Operação do processo `npm run worker` em produção.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Churn da Atlas se atrasar o go-live | Média | Alto | Prazo de 3 sprints; buffer de revisão de segurança; comunicação proativa do PM |
| Cliente não implementa deduplicação | Alta | Médio | Documentação destacada no portal; exemplos com `X-Event-Id` |
| Vazamento de secret no ambiente do cliente | Média | Alto | Secret por endpoint + rotação 24h; nunca logar secret |
| Worker parado sem detecção | Média | Alto | Métricas de fila pendente + alerta |
| Crescimento da outbox | Baixa (início) | Médio | Índices; arquivamento como follow-up |

## 11. Critérios de aceitação

1. Os 3 clientes demandantes conseguem cadastrar webhook HTTPS e receber evento &lt; 10s após mudança de status (ambiente de homologação).
2. Evento não é perdido se o cliente estiver fora por até a janela de retry; após esgotar, aparece em DLQ e admin consegue replay.
3. Request sem `https` é rejeitado; request de entrega sempre traz assinatura HMAC verificável.
4. Replay de DLQ por não-admin retorna 403; por admin é auditado.
5. Histórico de deliveries disponível via API.
6. Nenhuma das itens da seção “Fora de escopo” é entregue como requisito desta fase.
7. Critérios técnicos do [FDD](FDD.md) §12 atendidos.

## 12. Estratégia de testes e validação

| Camada | O quê |
|---|---|
| Unitário | Filtro de status na publicação; cálculo de backoff; validação HTTPS; montagem/verificação HMAC |
| Integração | `changeStatus` + outbox na mesma transação (commit e rollback); CRUD webhooks; replay admin + role |
| Contrato | Payloads/headers de entrega; códigos `WEBHOOK_*` |
| E2E / manual | Worker real contra stub HTTPS do cliente; simular downtime e observar retry → DLQ → replay |
| Segurança | Revisão Sofia (HMAC, geração/rotação de secret, redact de logs) antes do deploy |
| Aceite de produto | Homologação com Atlas (ou stub acordado) medindo latência e dedup |

## 13. Cronograma (da reunião)

| Fatia | Estimativa |
|---|---|
| Modelagem outbox + DLQ | 1 sprint |
| Worker + retry | 1 sprint |
| CRUD + deliveries | ~0,5 sprint |
| Integração `order.service` + testes E2E | ~0,5 sprint |
| HMAC/schemas/validações + revisão Sofia | incluso no total de **3 sprints** |
