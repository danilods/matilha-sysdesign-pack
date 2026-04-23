---
name: sysdesign-fault-tolerance-patterns
description: Use when asked "como tolerar falhas?" / "o que fazer quando X cai?" — selects between circuit breaker, exponential backoff+jitter, bulkhead, DLQ, checkpointing, fallback, and replication patterns.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém nomeia uma dependência potencialmente instável — serviço
externo, banco, fila, API de terceiro — e pergunta "o que acontece se cair?".
Também fires quando um incidente pós-morte aponta falha em cascata, retry storm,
ou indisponibilidade de componente único que derrubou o sistema inteiro. A skill
mapeia o problema para um dos 7 patterns canônicos (Replication, Circuit Breaker,
Exponential Backoff + Jitter, Bulkhead, DLQ, Checkpointing, Fallback) e recusa a
resposta "adiciona retry" — retry ingênuo é quase sempre o amplificador da falha.

## Preconditions

- A dependência em risco está nomeada — "nossa chamada ao Stripe", "o worker de
  indexação", "o DB de escrita". "O sistema pode cair" é escopo inválido.
- O modo de falha foi identificado: timeout, erro 5xx, latência elevada,
  resposta corrompida, indisponibilidade total. Cada modo pede pattern diferente.
- Existe SLA alvo (NFR-Availability) para o serviço que chama a dependência —
  sem alvo, não dá pra dimensionar resiliência (mais patterns = mais custo e
  complexidade).
- Há medição atual de taxa de falha da dependência, mesmo que estimada.

## Execution Workflow

1. Classifique a dependência em 1 de 3 categorias:
   - **Crítica síncrona** (request bloqueia resposta ao usuário) → Circuit
     Breaker + Fallback + Timeout agressivo.
   - **Crítica assíncrona** (request pode ser diferido) → DLQ + Exponential
     Backoff com Jitter + Checkpointing.
   - **Não-crítica** (telemetria, analytics, enrichment) → Fire-and-forget com
     Bulkhead isolado; falha não deve propagar.
2. Para síncrona crítica, instale Circuit Breaker (Resilience4j, Hystrix-like
   ou equivalente). Estados: closed (normal) → open (curto-circuito após N
   falhas) → half-open (testa 1 request). Parâmetros: threshold de falha (ex:
   50% em 20 requests), janela (ex: 10s), tempo em open (ex: 30s).
3. Defina Fallback explícito. Cache stale, resposta degradada, valor default,
   ou erro amigável — NUNCA propagar 5xx opaco para o usuário. Fallback é
   decisão de produto; envolva quem decide experiência.
4. Para assíncrona crítica, desenhe DLQ. Mensagem que falhou N vezes
   (N típico = 3-5) vai para fila morta com razão da falha. Monitor alerta
   quando DLQ cresce — é sinal de bug, não de glitch.
5. Configure Exponential Backoff com Jitter entre retries. Sem jitter, todas as
   réplicas batem no serviço caído ao mesmo tempo = thundering herd. Jitter
   aleatório (ex: sleep = base * 2^n + random(0, base)) espalha retries.
6. Isole com Bulkhead. Reserve pool de threads/conexões separado por
   dependência. Se uma falha, não esgota o pool global. Aplica-se a thread
   pools, connection pools, e memória alocada.
7. Para estado longo-rodante, adicione Checkpointing. Pipeline que processa
   1M registros salva offset a cada K. Crash reinicia do último checkpoint, não
   do zero.
8. Para dependências que precisam de "sempre disponível", replique: 3 instâncias
   (1 leader + 2 followers) é o mínimo canônico. Leader eleito por coordination
   service (ZooKeeper, Raft, Redis Sentinel).
9. Teste chaos explicitamente. Derrube a dependência em ambiente de staging,
   verifique que o pattern ativa. Untested resilience é fé, não engenharia.

## Rules: Do

- Sempre combine Timeout + Circuit Breaker + Fallback para dependência síncrona
  crítica. Um sem os outros é buraco.
- Use jitter em todo retry. Backoff sem jitter sincroniza falhas e amplifica.
- Dimensione bulkhead por dependência, não global. Um pool compartilhado derrota
  o padrão.
- Monitore DLQ como first-class. Fila morta crescendo = bug silencioso que
  perdeu dados.
- Teste caos em staging antes de prod. Kill -9 no leader, simule latência de
  rede, corte 30% de pacotes. Patterns que não foram testados não valem.

## Rules: Don't

- Não faça retry sem limite + sem backoff. É a receita canônica de retry storm
  derrubando o próprio sistema.
- Não use fallback "silencioso" que mascara bug. Fallback deve logar e alertar
  com taxa — se cache stale é servido 50% do tempo, é bug, não resiliência.
- Não replique leader sem coordination service. Split-brain (dois leaders)
  corrompe estado de maneira difícil de reverter.
- Não configure circuit breaker com thresholds sem base em medição. Chute 50%
  é só chute; meça taxa de erro normal antes.
- Não confunda resiliência com disponibilidade. Patterns contêm falha, não
  evitam — o SLA precisa assumir falhas contidas.

## Expected Behavior

Depois de aplicar a skill, cada dependência externa tem um pattern nomeado com
parâmetros explícitos (threshold, janela, timeout, jitter) versionados no repo.
Quando uma dependência falha em prod, o sistema degrada graciosamente em vez de
cair — usuários veem fallback, log captura razão, DLQ preserva mensagens,
alertas tocam. Pós-mortems deixam de ter a linha "retry storm derrubou o
sistema inteiro".

## Quality Gates

- Cada dependência externa classificada (síncrona crítica / assíncrona crítica
  / não-crítica).
- Circuit breaker configurado com parâmetros medidos, não chutados.
- Todo retry usa exponential backoff + jitter.
- Bulkhead isolado por dependência em thread/connection pools.
- DLQ monitorada com alerta em crescimento.
- Chaos test rodado em staging antes de cada release que toca integração.
- Fallback documentado como decisão de produto, não default técnico.

## Companion Integration

Depende de `sysdesign-availability-sla-tiers` (tier dita quantos patterns valem
o custo). Pareia com `sysdesign-consistency-cap` (replicação decide leader
election). Complementa `matilha-harness-pack:harness-fault-injection` quando
existe — aquele injeta falhas em testes de agentes; este desenha patterns para
código em produção. Fase 30-40 da metodologia matilha (skills/agents + hunt).

## Output Artifacts

- `resilience.md` por serviço (ou seção na spec) listando cada dependência,
  pattern, parâmetros, e fallback.
- Configuração versionada de circuit breaker (thresholds, timeouts).
- Dashboards: taxa de open-circuit, tamanho de DLQ, latência de retry.
- Chaos test script (em staging) + resultado do último run.

## Example Constraint Language

- Use "must" para: circuit breaker em dependência síncrona crítica, jitter em
  retry, DLQ em pipeline assíncrono crítico, chaos test antes de release.
- Use "should" para: bulkhead por dependência, checkpointing em pipeline longo,
  replicação 1+2 para serviços stateful críticos.
- Use "may" para: Fallback com cache stale vs. valor default (decisão de
  produto), replicação 1+4 em cenários multi-region, circuit breaker adaptativo
  (ajusta threshold por feedback).

## Troubleshooting

- **"Circuit breaker fica open o tempo todo"**: threshold apertado ou
  dependência realmente morta. Meça taxa de erro base antes de ajustar.
  Se dep está morta, fix a dep; circuit breaker não é cura.
- **"DLQ inchou e ninguém notou"**: faltou alerta. Defina SLO em tamanho de
  DLQ (ex: > 100 msgs por 15 min = page). Trate crescimento como bug.
- **"Retry storm derrubou o sistema mesmo com backoff"**: faltou jitter.
  Todo mundo fez backoff sincronizado e bateu no mesmo segundo. Adicione
  random 0..base a cada sleep.
- **"Fallback mascarou um bug por semanas"**: faltou logar. Fallback deve
  emitir métrica — taxa > X% é alerta. Fallback silencioso é dívida.
- **"Split-brain corrompeu dados"**: leader eleito sem coordenação. Migre para
  Raft/ZooKeeper. Dois leaders não é cenário aceitável.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign resilience-scaffold
> <dep>` gera o bloco `resilience.md` com pattern sugerido por categoria, além
> de um chaos-test stub em shell.

## Sources

- `[[concepts/nfr-system-design]]` — Fault-tolerance patterns (7 canônicos)
- `[[sources/acing-system-design]]` — Tan, capítulo de fault-tolerance
- `[[concepts/distributed-transactions]]` — replicação e leader election
