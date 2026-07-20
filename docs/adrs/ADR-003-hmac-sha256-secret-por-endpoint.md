# ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint

- **Data:** 2026-07-20
- **Decisores:** Sofia (Segurança), Diego (Plataforma), Bruno (Pedidos), Larissa (Tech Lead)
- **Fonte:** TRANSCRICAO.md — [09:19]–[09:23] Sofia, Bruno, Diego, Larissa

## Status

Aceito

## Contexto

Webhooks outbound enviam dados de pedidos para endpoints fora da nossa infra. O receptor precisa verificar (1) que a requisição veio da plataforma e (2) que o corpo não foi adulterado em trânsito. Já houve caso de cliente que vazou secret em log da aplicação dele; secret global da plataforma amplificaria o blast radius de um vazamento.

## Decisão

1. Assinar o **corpo do request** com **HMAC-SHA256** e enviar a assinatura no header `X-Signature`.
2. Cada endpoint de webhook tem **secret única** (não há secret global da plataforma).
3. Secret é **gerada pelo sistema** na criação do webhook e devolvida na resposta.
4. Suportar **rotação** com grace period de **24h**: após rotacionar, a secret antiga permanece válida em paralelo por 24h; depois, invalida.
5. Validação complementar (não ADR separado): URL deve ser `https` (rejeitar `http` no schema Zod).

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
|---|---|
| Secret global da plataforma | Vazamento de uma secret compromete todos os clientes ([09:21] Sofia). |
| Sem assinatura / apenas HTTPS | HTTPS protege o canal, mas não autentica a origem nem detecta adulteração do payload se o endpoint for exposto ([09:19] Sofia). |
| mTLS mútuo | Não discutido como opção viável na reunião; HMAC-SHA256 é padrão de mercado com bibliotecas amplamente disponíveis ([09:20] Sofia). |

## Consequências

**Positivas**

- Autenticidade e integridade verificáveis pelo cliente.
- Blast radius limitado a um endpoint em caso de vazamento.
- Grace period de 24h permite migração sem downtime do receptor.

**Negativas / trade-offs**

- Cliente precisa implementar verificação HMAC (documentar no portal — [09:26] Marcos).
- Gestão de duas secrets durante a janela de rotação aumenta complexidade no worker (tentar secret atual e, se necessário, a anterior).
- Sofia exige revisão de segurança do código de HMAC/geração de secret antes do deploy ([09:46] Sofia).
