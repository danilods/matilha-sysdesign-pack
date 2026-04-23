---
name: sysdesign-rate-limiting-strategies
description: Use when choosing a rate-limiting algorithm — picks token bucket, leaky bucket, fixed window, or sliding window and places it stateful vs stateless.
category: matilha
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when the team is protecting an API from abuse, runaway clients, or cost
blowouts and has to pick a specific rate-limiting algorithm plus a place to
store counters. Fires when someone says "just rate limit it" without naming
burst tolerance, stateful storage cost, or per-user vs per-endpoint scope.
The skill narrows the choice to a single algorithm, names the counter store,
and flags the "don't punish good users" bias that Zhiyong Tan emphasises
throughout the rate-limiting chapter.

## Preconditions

- The service already has observable traffic — raw QPS or a representative
  spike profile. Rate-limit tuning without traffic data is cargo culting.
- Someone can answer "what does legit burst traffic look like for our users?"
  even roughly. Without that, every algorithm will over- or under-limit.
- The team has agreed on the scope of the limit (per user, per API key, per
  endpoint, per IP, or a combination). These scopes compose badly if chosen
  after the fact.
- There is a fallback behaviour defined — 429 with Retry-After, a soft queue,
  or shadow-mode logging. Rate limiting that silently drops requests is a
  debugging nightmare.

## Execution Workflow

1. Pull the traffic shape. Use Read or the observability tool of record to
   fetch P50 and P99 QPS per user (or per key) over a representative window.
   Burst ratio (P99 over P50) drives algorithm choice more than absolute
   volume.
2. Pick the algorithm against the burst profile. Token bucket absorbs bursts
   up to the bucket size while enforcing a long-run average — the default
   when real users are spiky (human clickers, mobile retries). Leaky bucket
   enforces a strict steady rate — the default when a downstream dependency
   cannot handle spikes (legacy DB, third-party API with its own quota).
   Fixed window is the simplest counter but has a 2x burst failure mode at
   window boundaries — fine for coarse limits, dangerous for tight ones.
   Sliding window log or sliding window counter smooths that boundary
   problem at the cost of more storage per user.
3. Place the counter. Stateful means a shared store (Redis, DynamoDB) holds
   counters keyed by user or endpoint — single source of truth, accurate
   across replicas, but adds a hop and a hot-key failure mode. Stateless
   means each sidecar or LB holds its own local counter — fast, no extra
   hop, but sums across replicas drift. Use stateful when limits are tight
   and fairness matters (paid tiers, anti-abuse). Use stateless when the
   limit is loose and per-instance is "close enough" (DDoS shield).
4. Name the scope grid. Typical grids are per-user-per-endpoint (billing
   limits), per-IP-global (anti-DDoS), and per-API-key-per-endpoint (B2B
   plans). Write the grid down before coding — scopes added later require
   schema changes in the counter store.
5. Define the 429 contract. Include Retry-After in seconds, a rate limit
   policy header if the API is external, and a correlation ID for support
   debugging. Undocumented 429 responses break client retry logic.
6. Shadow-mode before enforcement. Log "would have blocked" for a full week
   against production traffic. If the count of would-have-blocks against
   paying customers is non-trivial, the limits are wrong, not the users.
7. Wire dashboards for block-rate per scope plus a 4 golden signals panel —
   see `sysdesign-monitoring-4-golden-signals`. A rate limiter without a
   dashboard is a production outage waiting for a customer email.

## Rules: Do

- Favour token bucket as the default for user-facing APIs. Real human
  traffic is bursty and leaky bucket will frustrate normal users on
  legitimate retries.
- Store counters in a shared store (Redis with eviction) when the limit
  enforces billing or fairness. Replica-local counters drift and cost the
  business real money.
- Pair every limit with an explicit 429 contract including Retry-After.
  Clients need a deterministic recovery path.
- Ship in shadow mode first, then enforce. The false-positive cost on real
  users is higher than the abuse cost during one week of observation.
- Prefer per-user scopes over per-IP scopes for authenticated APIs. IP
  limiting breaks mobile carriers and corporate NAT.

## Rules: Don't

- Don't choose fixed window for tight limits (under 100 req/min). The
  boundary-burst failure lets a caller double the limit by straddling
  windows.
- Don't build counters in the application process for fairness-critical
  limits. Autoscaling changes the replica count and the effective limit
  silently shifts.
- Don't alert on high 429 rate without splitting by scope. A spike from
  one abusive key looks identical to a broad regression.
- Don't omit Retry-After. Clients without it will retry in tight loops and
  amplify the problem the limiter was meant to contain.
- Don't limit logged-out and logged-in traffic with the same policy. Signal
  quality is very different; the scopes deserve distinct limits.

## Expected Behavior

After applying the skill, the team has named exactly one algorithm, one
counter store, one scope grid, and one 429 contract. Shadow-mode logs run
for at least a week before enforcement. Dashboards show block-rate per
scope, and there is a documented rollback if paying customers start seeing
429s unexpectedly.

Debates about "which algorithm is best" stop; the answer is now a function
of the burst profile and the fairness requirement, both written down.

## Quality Gates

- Algorithm choice cites the burst profile (P99/P50 ratio) or downstream
  constraint that motivated it.
- Counter store is named (Redis cluster, DynamoDB, in-process) with the
  drift cost explicitly accepted.
- Scope grid listed as a table — user, IP, key, endpoint combinations —
  not implied.
- 429 response contract includes Retry-After and a rate-limit policy
  header; sample response pasted in the design doc.
- Shadow-mode window agreed (one week minimum for weekly seasonality).
- Block-rate dashboard exists per scope, not just globally.

## Companion Integration

Pairs with `sysdesign-monitoring-4-golden-signals` (observability on the
limiter itself), `sysdesign-fault-tolerance-patterns` (circuit breakers
downstream of a limiter), and `sysdesign-load-balancers` (where L7 LBs can
host the limiter natively). The `matilha-harness-pack:harness-nfrs-as-prompts`
companion is the agent-side mirror — encoding rate-limit NFRs into an agent
system prompt instead of an API enforcement layer.

## Output Artifacts

- A design-doc section titled "Rate limiting" naming algorithm, store,
  scope grid, and 429 contract.
- Optional: a `rate-limits.yaml` file listing (scope, limit, window,
  algorithm) rows checked into the repo beside OpenAPI specs.
- Dashboard link or panel JSON for block-rate per scope.
- Shadow-mode log sample pasted into the design doc before enforcement.

## Example Constraint Language

- Use "must" for: defining the 429 contract before enforcement, shadow-mode
  observation window of at least one week, storing fairness-critical
  counters in a shared store.
- Use "should" for: token bucket as default for human-facing APIs, per-user
  over per-IP for authenticated traffic, dashboard per scope.
- Use "may" for: stateless sidecar counters on best-effort DDoS shields,
  fixed window on coarse limits above a few hundred RPM, application-level
  fallback queueing instead of hard 429.

## Troubleshooting

- **"Legit users keep hitting 429 after a UI retry"**: the algorithm is
  probably leaky bucket or fixed window. Switch to token bucket sized to
  absorb a normal retry burst (3-5 requests in two seconds).
- **"Limits work on one replica, fail on another"**: counters are
  in-process and replicas drifted after autoscale. Move to a shared store.
- **"One abusive key is tanking the global error rate"**: dashboards are
  not split by scope. Add per-scope block-rate panels and a separate alert
  path for single-key anomalies.
- **"Shadow mode shows we would block 8% of paid customers"**: the limits
  are mis-tuned, not the users. Pull P99 per paid user, set the limit at
  the 99.5 percentile, re-run shadow.
- **"Mobile carrier NAT is getting blocked by per-IP limit"**: switch the
  scope to per-user for authenticated traffic and keep per-IP only for
  unauthenticated endpoints.

## Concrete Example

A SaaS team launches a public API with a 1000 req/min per-key limit using
fixed window in Redis. Boundary bursts let a client spike 1900 requests in
two seconds across the window boundary, tripping the downstream DB.
Switching to token bucket with bucket size 200 and refill 1000/min keeps
the long-run average identical while smoothing the boundary. Shadow mode
shows three paying keys hit the new limit, so the team adds a scoped
override for those tiers before enforcing. Post-launch, 429 rate stays at
0.3% globally and the DB stops seeing the boundary spikes.

## Sources

- `[[concepts/nfr-system-design]]` — rate limiting as an availability and
  fault-tolerance lever
- `[[concepts/design-cases]]` — Design Rate Limiting (Chapter 8) case study
- Zhiyong Tan, *Acing the System Design Interview*, Chapter 8. The
  "em caso de dúvida, não limite o usuário" bias is paraphrased from
  Tan's discussion of false-positive cost.
