# matilha-sysdesign-pack — skills inventory

19 skills organized in 5 families.

## Family 1 — NFR framework (6 skills)

| Skill | Triggering intent | Key principle |
|---|---|---|
| `sysdesign-nfr-clarification` | "Antes de desenhar X" / "Como levantar requisitos não-funcionais?" | Clarify 10 NFRs (scalability, availability, latency, consistency, cost, etc.) BEFORE drafting architecture |
| `sysdesign-scalability-horizontal-vs-vertical` | "Como escalo Y?" / "Vertical ou horizontal?" | Horizontal is "true" scalability; stateless services scale easily; writes on shared storage are hard |
| `sysdesign-availability-sla-tiers` | "Qual SLA realista?" / "99.9 vs 99.99?" | SLA tiers → downtime math; replication + CAP tradeoff + MTTR/MTBF |
| `sysdesign-fault-tolerance-patterns` | "Como tolerar falhas?" / "O que fazer quando X cai?" | Circuit Breaker + Backoff+Jitter + Bulkhead + DLQ + Checkpointing |
| `sysdesign-latency-targets-techniques` | "Como reduzir latency?" / "P99 alto" | Latency vs throughput; GeoDNS, CDN, caching, RPC > REST; consumer target 10ms-seconds, HFT < ms |
| `sysdesign-consistency-cap` | "Strong ou eventual consistency?" / "CAP trade?" | CAP theorem practical; linearizability (HBase, MongoDB, Redis) vs availability (Cassandra, CouchDB, Dynamo); ZooKeeper/Raft |

## Family 2 — Common services (3 skills)

| Skill | Triggering intent | Key principle |
|---|---|---|
| `sysdesign-load-balancers` | "Qual LB usar?" / "HAProxy vs ALB?" | L4 (fast, address routing) vs L7 (features, auth, TLS termination); sticky sessions; hardware vs LBaaS vs software |
| `sysdesign-rate-limiting-strategies` | "Como limitar requests?" / "Token bucket ou sliding window?" | Token/leaky bucket, fixed/sliding window; stateful (Redis) vs stateless (sidecar); "em caso de dúvida, não limite o usuário" |
| `sysdesign-monitoring-4-golden-signals` | "O que monitorar?" / "Quais métricas?" | Latency + Traffic + Errors + Saturation; alerting thresholds |

## Family 3 — Design patterns (5 skills)

| Skill | Triggering intent | Key principle |
|---|---|---|
| `sysdesign-idempotency-patterns` | "Como evitar duplicatas?" / "Write idempotente" | Idempotent write endpoints, deduplication keys, at-most-once semantics |
| `sysdesign-cdn-object-store` | "Como servir assets globalmente?" / "Fotos, vídeos" | CDN edge servers + S3/object store + cache invalidation + origin fallback |
| `sysdesign-event-streaming-kafka` | "Kafka ou SQS?" / "Event streaming" | Kafka for decoupling + ordered delivery + replay; partitioning strategies |
| `sysdesign-dead-letter-queue` | "O que fazer com mensagens falhas?" / "Retry" | DLQ + poison-pill handling + retry policies + alerting on DLQ growth |
| `sysdesign-dual-write-event-sourcing` | "Dois writes em fontes diferentes" / "Event sourcing vale a pena?" | Dual-write pitfalls; event sourcing tradeoffs (replay + auditability vs complexity) |

## Family 4 — Case-specific patterns (3 skills)

| Skill | Triggering intent | Key principle |
|---|---|---|
| `sysdesign-newsfeed-fanout` | "Feed type Twitter/Facebook" / "Celebridade com milhões de followers" | Fan-out on write vs read; celebrity problem (hybrid approach for high-degree nodes) |
| `sysdesign-autocomplete-trie-fuzzy` | "Autocomplete de search" / "Sugestão de queries" | Weighted trie + sampling + fuzzy matching + content moderation |
| `sysdesign-top-k-count-min-sketch` | "Dashboard top 10" / "Frequência de eventos" | Count-min sketch for approximate top-K; batch vs streaming (Lambda/Kappa) |

## Family 5 — Methodology (2 skills)

| Skill | Triggering intent | Key principle |
|---|---|---|
| `sysdesign-interview-flow-50min` | "Como estruturar design review?" / "Spec em 50 min" | 50-min flow: requirements (10) → API (5) → data model + arch (10) → deep dive (15) → monitoring (10) |
| `sysdesign-tradeoff-framing` | "Decisão arquitetural difícil" / "Prós e contras" | "System design é uma arte, não ciência. Tudo envolve tradeoffs." — explicit tradeoff articulation prevents hand-wave decisions |

## Overlap disclosures

- `sysdesign-nfr-clarification` **Complements** `matilha-harness-pack:harness-nfrs-as-prompts` — sysdesign encodes NFRs as clarifying questions during system spec/design; harness encodes NFRs as system-prompt constraints for AI agents.
