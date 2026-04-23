---
name: sysdesign-availability-sla-tiers
description: Use when discussing SLA targets, uptime, or "quanto de downtime aceitável?" — translates availability tiers into minutes/hours, forces MTTR/MTBF awareness, and exposes CAP tradeoff cost.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara sempre que a palavra "disponibilidade", "SLA", "uptime", ou "três noves"
aparece na conversa. Também fires quando alguém promete "nunca cair" — promessa
que custa caro e raramente é o que o negócio precisa. A skill converte tiers
abstratos em minutos de downtime por ano, força escolha entre MTTR curto (cura
rápida) vs MTBF longo (prevenção), e conecta a escolha à negociação CAP
(availability vs consistency). Quem assina 99.999 sem entender o custo está
assinando um cheque em branco.

## Preconditions

- Existe um serviço com dono claro. Availability de "toda a empresa" é conversa
  de marketing; availability de um componente é contrato de engenharia.
- Há consumidor nomeado — B2B API, app mobile, pipeline interno. O consumidor
  define quanto downtime dói de fato.
- O custo de uma hora parada é estimável (mesmo que ordem de grandeza). Sem
  custo, o tier é arbitrário.
- Existe (ou pode existir) medição de uptime real. Assinar SLA sem instrumentação
  é teatro.

## Execution Workflow

1. Traduza cada tier candidate em minutos de downtime por ano. Use a tabela:
   - 99% → 3.65 dias/ano (87.6 horas)
   - 99.9% → 8.77 horas/ano
   - 99.99% → 52.6 minutos/ano
   - 99.999% → 5.26 minutos/ano
   - 99.9999% → 31.5 segundos/ano
2. Pergunte ao dono do consumidor: "quanto vale 1 hora fora do ar em receita
   perdida, churn, ou SLA penalty?" Multiplique por downtime do tier candidate.
   Se o custo anual do tier é maior que o custo da infra que o sustenta, o
   tier está errado.
3. Decida em qual lado do CAP você cai. Se o sistema precisa aceitar writes
   durante partição de rede → AP (Cassandra-like, Dynamo-like). Se precisa
   recusar writes inconsistentes mesmo que isso derrube leitura → CP
   (HBase-like, Redis-com-sentinela). A escolha muda o tier viável.
4. Separe MTTR de MTBF. MTBF (Mean Time Between Failures) é prevenção — caro,
   bom em médio prazo. MTTR (Mean Time To Recover) é cura — runbooks, rollback
   automático, feature flags. 99.99 geralmente vem de MTTR curto, não MTBF
   longo.
5. Desenhe redundância proporcional ao tier. 99.9 aceita 1 réplica. 99.99 exige
   multi-AZ. 99.999 exige multi-region ativo-ativo — e custo salta não-linear.
6. Declare o que NÃO está coberto pelo SLA: janelas de manutenção, dependências
   externas, bugs de cliente. SLA sem exclusões declaradas vira armadilha
   contratual.
7. Instrumente antes de assinar. Uptime medido em black-box (probe externo) +
   white-box (métrica interna) — os dois discordam em incidentes reais.

## Rules: Do

- Converta tier em minutos antes de qualquer discussão. "Três noves" sem
  "8.77 horas/ano" é idioma de marketing, não decisão.
- Escolha MTTR como alavanca primária para tiers 99.9-99.99. Runbooks +
  rollback + feature flags batem "infra perfeita".
- Decida CAP explicitamente. Quase todo sistema real é AP em prática — escolher
  CP sem reconhecer o custo em availability é auto-engano.
- Instrumente black-box + white-box. Se só tem um, você não sabe uptime real.
- Documente janelas de manutenção como exclusões. Fora delas, o contador roda.

## Rules: Don't

- Não assine 99.999 por reflexo. 5.26 min/ano implica multi-region ativo-ativo,
  automação de failover perfeita, e time on-call 24/7 — custo 5-10x de 99.99.
- Não trate availability como monolito. Cada rota/endpoint tem tier próprio;
  login pode ser 99.99 enquanto dashboard é 99.9.
- Não combine "strong consistency" com "alta availability" sem reconhecer que
  CAP força escolha. Você pode ter os dois com latência alta — ninguém quer.
- Não conte downtime planejado como "não conta". Conta para o usuário; só não
  conta se o contrato SLA excluir explicitamente.
- Não confie em SLA de dependência upstream. Se você depende de um serviço 99.9,
  seu teto é 99.9 — aritmética de encadeamento é multiplicativa.

## Expected Behavior

Depois de aplicar a skill, o SLA do sistema deixa de ser promessa vaga e vira
contrato numérico com três colunas: tier, minutos de downtime/ano permitidos,
custo de engenharia para sustentar. A escolha CAP está declarada no `nfrs.md`.
O time sabe se resolve incidentes por MTTR (runbooks) ou MTBF (redundância), e
investe na alavanca correta. Quando um incidente passa do teto, a conversa é
sobre contrato (revisamos tier?), não drama (por que caiu?).

## Quality Gates

- Tier declarado em minutos de downtime/ano, não só em "noves".
- Decisão CAP (AP ou CP) registrada com justificativa.
- MTTR alvo e MTBF alvo nomeados, mesmo que aproximados.
- Redundância dimensionada ao tier (single-AZ / multi-AZ / multi-region).
- Exclusões de SLA declaradas (manutenção, dependências, user error).
- Instrumentação black-box + white-box em produção antes de assinar externamente.

## Companion Integration

Depende de `sysdesign-nfr-clarification` ter rodado (Availability é NFR #2).
Pareia com `sysdesign-fault-tolerance-patterns` (tier alto exige patterns de
resiliência) e `sysdesign-consistency-cap` (CAP decide teto de availability).
Na metodologia matilha: fase 20 (spec) para contrato de SLA, fase 40-50
(hunt/review) para validar em produção, fase 60 (den) para pós-incidente.

## Output Artifacts

- Seção "Availability" em `nfrs.md` com tier + minutos/ano.
- Decisão CAP registrada (AP ou CP + justificativa).
- Runbook com MTTR alvo por tipo de incidente.
- Dashboard de uptime (black-box + white-box) em produção.
- Exclusões de SLA declaradas no contrato (se externo).

## Example Constraint Language

- Use "must" para: converter tier em minutos/ano, declarar CAP, instrumentar
  antes de publicar SLA externamente.
- Use "should" para: investir em MTTR antes de MTBF para tiers até 99.99,
  usar multi-AZ a partir de 99.99, declarar exclusões no contrato.
- Use "may" para: tratar endpoints com tiers diferentes, usar degradação
  graciosa (partial availability) como estratégia de MTTR, exigir 99.999 só
  em componentes de pagamento/vida-crítica.

## Troubleshooting

- **"Prometemos 99.99 e medimos 99.5"**: instrumentação estava ausente ou
  otimista. Instale probe externo, meça por 30 dias, então renegocie tier ou
  invista em MTTR. Prometer sem medir queima confiança.
- **"Downtime planejado quebra o SLA"**: contrato não excluiu manutenção.
  Renegocie exclusões ou migre para rolling updates (multi-AZ ativo-ativo).
- **"Uma dependência cai e derruba tudo"**: encadeamento multiplicativo bateu.
  Adicione circuit breaker + fallback (ver `sysdesign-fault-tolerance-patterns`)
  ou baixe seu tier para refletir a realidade.
- **"Tier está alto, mas custo virou proibitivo"**: overshoot. Pergunte ao
  negócio quanto vale 1 hora parada vs. o custo do tier. Muitas vezes 99.9 é
  o suficiente e 99.99 foi aspiração sem base.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign sla-calc <tier>` mostra
> minutos/ano, custo típico de redundância, e MTTR alvo implícito. Útil para
> onboarding de time novo ou revisão de contrato.

## Sources

- `[[concepts/nfr-system-design]]` — Availability, tiers, MTTR/MTBF, CAP
- `[[sources/acing-system-design]]` — Tan, capítulo de availability
- `[[concepts/distributed-transactions]]` — CAP em profundidade
