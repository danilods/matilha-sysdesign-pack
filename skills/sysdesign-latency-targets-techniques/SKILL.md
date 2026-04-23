---
name: sysdesign-latency-targets-techniques
description: Use when asked "como reduzir latency?" / "P99 alto" — sets target per percentile, distinguishes latency vs throughput vs bandwidth, and applies GeoDNS, CDN, caching, RPC-over-REST.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Dispara quando alguém reclama de "lento", quando P99 estoura SLO, ou quando o
planejamento propõe latency target sem distinguir percentil. Também fires
quando latency e throughput são tratados como sinônimos — erro comum que leva
a otimizações erradas (throughput alto com P99 ruim é produto pior que o
inverso). A skill fixa alvos por percentil (P50/P95/P99), mapeia o caminho
usuário → serviço em hops com custo de latency, e seleciona técnicas
(GeoDNS, CDN, cache, RPC, database tuning) pela camada que domina o budget.

## Preconditions

- Existe medição atual de latency por percentil. "Parece lento" sem P50/P95/P99
  medidos é opinião, não bug.
- O caminho request → response está mapeado em hops (DNS, TLS, LB, API,
  DB, cache, external). Sem o mapa, a otimização é chute.
- Há alvo declarado em `nfrs.md` — consumer app, interno B2B, real-time, HFT.
  Cada categoria tem budget diferente.
- NFR-Cost permite o investimento. CDN global e multi-region baixam latency
  ao custo de 2-10x na fatura — a conversa precisa estar aberta.

## Execution Workflow

1. Separe os três conceitos:
   - **Latency** = tempo total request → response (ms).
   - **Throughput** = requests/segundo que o sistema sustenta.
   - **Bandwidth** = bytes/segundo no canal de rede.
   Otimizar um pode piorar outro. Batching sobe throughput e piora latency.
   Compression sobe latency de CPU mas economiza bandwidth.
2. Defina alvo por percentil, não média. P99 é o que o usuário frustrado sente.
   Targets típicos:
   - Consumer app web/mobile: P99 ≤ 1s, P50 ≤ 200ms.
   - API interna B2B: P99 ≤ 500ms, P50 ≤ 50ms.
   - Real-time (chat, trading): P99 ≤ 100ms, P50 ≤ 10ms.
   - HFT: P99 < 1ms, P50 em microsegundos.
3. Mapeie o budget por hop. Para consumer web típico:
   - DNS resolve: 10-50ms (cacheável)
   - TLS handshake: 50-150ms (reuse conn)
   - LB → API: 1-5ms (same DC)
   - API → DB: 1-20ms (idem; 100ms+ se cross-region)
   - Render + response: 50-200ms
   Some, compare com target. O hop que excede sozinho vira prioridade.
4. Aplique técnicas por hop dominante:
   - **DNS lento / usuário distante** → GeoDNS (roteia para região mais próxima)
   - **Estático pesado (imagens, JS bundle)** → CDN (CloudFront, Cloudflare)
   - **DB query repetitiva** → cache (Redis, Memcached) com TTL dimensionado
   - **Muitos round-trips entre serviços** → RPC binário (gRPC, Thrift) no lugar
     de REST/JSON; serialização mais rápida + multiplex
   - **Query lenta** → index review, denormalização seletiva, read replica
   - **Cold start** → warm pool, keep-alive connections, JIT warmup
5. Meça antes e depois. Otimização sem A/B de latency é dívida disfarçada de
   progresso.
6. Declare tail amplification. P99 multiplica em cadeia: se um request chama 5
   serviços em série, cada um com P99 100ms, o P99 agregado pode ser 500ms+.
   Paralelize ou reduza hops.
7. Reavalie custo. CDN global para 1k users/mês é overkill. Multi-region para
   latency sub-10ms global é receita feliz de estouro de budget.

## Rules: Do

- Declare target como P50 + P95 + P99. Single number sem percentil esconde o
  que importa.
- Mapeie budget por hop antes de otimizar. Atacar o hop errado não move P99.
- Prefira cache para leitura repetitiva antes de redesenhar banco. Redis na
  frente de Postgres derruba P99 de 50ms para 2ms em leitura quente.
- Use RPC binário (gRPC) entre serviços internos. JSON+HTTP/1.1 paga imposto
  de serialização e parse que some em mesh com 10+ hops.
- Meça P99 sob carga realista. P99 em baixa carga mente — stress test com
  tráfego parecido com pico.

## Rules: Don't

- Não otimize média (avg). Média esconde cauda. O usuário do P99 é o que
  escreve review ruim.
- Não adicione CDN para API dinâmica achando que vai acelerar. CDN é para
  estático + edge-cacheable; API pessoal não cacheia sem invalidação cara.
- Não trate cache como bala de prata. Cache stale em dado crítico vira bug
  funcional — ver accuracy/consistency nos outros NFRs.
- Não assuma que "mais hardware" baixa latency. Latency é sobre caminho, não
  capacidade. CPU ociosa não acelera rede.
- Não ignore cauda amplificada. 5 chamadas em série com P99 100ms cada não dá
  P99 100ms — dá ~500ms ou pior. Paralelize ou reduza.

## Expected Behavior

Depois de aplicar a skill, latency vira número por percentil + alvo versionado
+ budget por hop. A otimização é direcionada: "P99 está em 1.2s, budget de DB
é 200ms mas medimos 800ms — index faltando em user_lookup". O time para de
otimizar aleatoriamente e começa a atacar o hop dominante. Técnicas são
escolhidas por diagnóstico, não por moda.

## Quality Gates

- Target declarado como P50 + P95 + P99, não apenas média.
- Caminho mapeado em hops com budget estimado por hop.
- Medição atual sob carga realista (stress test recente).
- Técnica escolhida justifica hop dominante, não palpite.
- Cauda amplificada calculada quando há cadeia de hops síncronos.
- Re-medição pós-otimização confirmou ganho em P99, não só média.

## Companion Integration

Depende de `sysdesign-nfr-clarification` (target precisa existir em `nfrs.md`).
Pareia com `sysdesign-scalability-horizontal-vs-vertical` (mais réplicas
reduzem fila, que reduz latency sob carga) e com
`sysdesign-fault-tolerance-patterns` (timeout + circuit breaker protegem P99
de cauda patológica). Fase 30-40 matilha (spec + hunt).

## Output Artifacts

- Seção "Latency" em `nfrs.md` com P50/P95/P99 alvo.
- `latency-budget.md` com hop-by-hop (DNS, TLS, LB, API, DB, render).
- Dashboard de percentis em produção (não só média).
- Resultado de A/B antes/depois de cada otimização de latency.

## Example Constraint Language

- Use "must" para: target em percentil (não média), medição antes de
  otimizar, stress test realista.
- Use "should" para: budget por hop, cache para leitura quente, RPC binário em
  mesh interno, paralelização quando cadeia síncrona estoura cauda.
- Use "may" para: CDN em endpoints estáveis com invalidação longa,
  multi-region para usuários geograficamente distribuídos, JIT warmup para
  linguagens com cold start.

## Troubleshooting

- **"P99 alto mas média boa"**: cauda gorda. Investigue GC pauses, lock
  contention, queries raras pesadas, cold start. Média não vai te mostrar
  onde dói.
- **"Cache não ajudou"**: hit rate baixo (TTL curto, chave mal dimensionada)
  ou o gargalo não era leitura. Meça hit rate e reveja o mapa de hops.
- **"Adicionamos CDN e latency não mudou"**: endpoint não era cacheável ou
  usuário já estava próximo da origem. CDN é para estático + usuários
  distantes.
- **"P99 melhora em staging mas piora em prod"**: carga real é diferente.
  Stress test precisa imitar tráfego de pico — usuários concorrentes, não
  só requests/segundo.
- **"Reduzimos hops e latency não caiu"**: o hop removido não dominava o
  budget. Refaça o mapa com medições frescas antes de novo corte.

## CLI shortcut (optional)

> Se o CLI matilha estiver instalado, `matilha sysdesign latency-budget
> <path>` gera planilha de hops com defaults de domínio e formatação pronta
> para colar na spec.

## Sources

- `[[concepts/nfr-system-design]]` — Latency, throughput, técnicas
- `[[sources/acing-system-design]]` — Tan, capítulo de performance
- `[[concepts/scaling-databases]]` — caching, read replicas, tail amplification
