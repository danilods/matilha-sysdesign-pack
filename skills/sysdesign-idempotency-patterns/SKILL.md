---
name: sysdesign-idempotency-patterns
description: Use when designing write endpoints that may be retried — install idempotency keys, dedup stores, and at-most-once semantics to prevent duplicate effects.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when a write endpoint can be retried by the client, the network, a
queue, or a human hitting submit twice, and the business cannot tolerate
duplicate side effects — a double charge, a second shipment, a
resurrected row. Fires when someone says "just make it idempotent" without
naming who generates the key, where it's stored, or how long it lives.
The skill installs a concrete idempotency-key contract, a dedup store
choice, and a policy for the replay response.

## Preconditions

- The endpoint performs a side effect (payment, shipment, email, inventory
  reservation). Idempotency on pure reads is trivially free; this skill is
  about writes.
- The team can name who generates the idempotency key — client library,
  browser, upstream service — and can reach them to enforce the contract.
- A dedup store exists or can be added (Redis, DynamoDB, a Postgres table).
  Idempotency without persistence is just wishful thinking.
- There is a defined TTL policy for keys. Keys stored forever cost money;
  keys dropped too fast let retries duplicate.

## Execution Workflow

1. Classify the operation. Is it naturally idempotent (PUT /resource/:id
   with full body), conditionally idempotent (PATCH with If-Match), or
   action-like (POST /payments, POST /emails)? Action-like is where
   idempotency keys are load-bearing; the other two have weaker protocols
   available.
2. Define the key. A client-provided UUID in an `Idempotency-Key` header is
   the standard for action-like endpoints. The key must be stable across
   retries of the same logical intent — generated once, reused on every
   retry. If the client regenerates on each retry, idempotency is lost.
3. Pick the dedup store. Redis with TTL is the default for high-traffic
   endpoints (sub-millisecond check). A DB table is the default when the
   side effect is already DB-transactional and you want atomic
   dedup+commit. Whatever the choice, the dedup check and the side effect
   must be atomic — separate checks and inserts race.
4. Define the replay response contract. When a key is seen again, return
   the stored prior response (same status, same body, same headers where
   meaningful) — not a fresh 200, not a 409. Clients rely on byte-equal
   replay to reconcile state. Stripe's contract is the reference model.
5. Set the TTL. Payment industry norm is 24 hours; internal APIs can be
   shorter (15 minutes to 1 hour). TTL must be longer than the longest
   plausible client retry window — queues with exponential backoff can
   retry for hours, so match that.
6. Handle the in-flight race. Two concurrent requests with the same key
   must not both succeed. Use a conditional write (SETNX, unique-index
   insert) to claim the key first; the loser polls or returns 409 with
   Retry-After.
7. Treat partial failure honestly. If the side effect succeeded but the
   response was lost, the next retry must see a stored success. If the
   side effect failed mid-transaction, the key must NOT be marked
   successful — otherwise a second, real retry returns a stale error.
8. Wire observability. Track dedup hit-rate per endpoint. A zero hit-rate
   means no retries happen (or the client isn't sending keys). A spiking
   hit-rate during an incident tells you the client is retrying correctly.

## Rules: Do

- Require `Idempotency-Key` on all action-like POST endpoints. Make the
  header mandatory for authenticated clients; 400 without it.
- Make the dedup-check and the side-effect commit atomic. Either a single
  transaction or a SETNX-then-commit with a compensating unset on failure.
- Store the full response (status code, headers the client needs, body) so
  replay is byte-equal. Half-stored replays confuse clients.
- Scope the key by tenant or API key when multi-tenant. A raw global key
  namespace collides on UUID collision (rare) and on attacker-supplied
  keys (common).
- Set TTL longer than the longest retry window the upstream might use.
  Async queues with backoff can retry for many hours.

## Rules: Don't

- Don't derive the key from the request body. Any tiny request change (a
  new client SDK version adding a field) silently becomes a new key and
  duplicates the side effect.
- Don't return 409 for a successful replay. The client intended the
  operation; returning an error surface tells them to undo something they
  completed.
- Don't rely on at-least-once with no dedup for financial or inventory
  actions. The correctness cost of a single duplicate swamps the infra
  cost of the dedup store.
- Don't skip the race-between-retries case. Network retries plus queue
  retries regularly produce concurrent in-flight duplicates; a check-then-
  insert without atomicity lets both through.
- Don't store keys forever. Unbounded growth hurts the dedup store and
  obscures legitimate replays against stale data.

## Expected Behavior

After applying the skill, every action-like POST documents an Idempotency-
Key header in its OpenAPI spec, enforces it at the edge, and replays the
stored response byte-equal on repeat. The dedup store is named, TTL is
documented, and observability shows dedup hit-rate per endpoint. Duplicate
side effects become a bug class the team can detect, not a mystery
category the on-call rotation fears.

## Quality Gates

- OpenAPI spec lists `Idempotency-Key` as a required header on every
  action-like POST.
- Dedup store choice (Redis, DynamoDB, DB table) named in the design doc
  with TTL.
- Atomic claim-then-commit flow documented; no separate check-and-insert.
- Stored response includes status, relevant headers, and body such that
  replay is byte-equal.
- Dedup hit-rate dashboard panel exists per endpoint.
- Partial-failure matrix (side effect succeeded / failed vs response
  lost / delivered) covered in the runbook.

## Companion Integration

Pairs tightly with `sysdesign-dead-letter-queue` (DLQ retries lean on
idempotency keys to avoid re-running side effects), `sysdesign-event-
streaming-kafka` (at-least-once delivery plus idempotent consumers is the
canonical Kafka contract), and `sysdesign-monitoring-4-golden-signals`
(dedup hit-rate is a traffic-signal variant). The
`matilha-harness-pack:harness-nfrs-as-prompts` companion mirrors this
pattern at the agent layer — idempotent tool-calls.

## Output Artifacts

- OpenAPI / API-spec entries with the header documented per endpoint.
- A design-doc section "Idempotency" naming key source, store, TTL, and
  replay contract.
- An example request/response pair showing first call and replay for one
  representative endpoint.
- Dashboard panel JSON for dedup hit-rate per endpoint.

## Example Constraint Language

- Use "must" for: atomic claim-then-commit, storing full response for
  replay, requiring the Idempotency-Key header on action-like POSTs,
  scoping the key by tenant.
- Use "should" for: 24-hour TTL for external-facing financial endpoints,
  client-provided UUIDv4 as the key format, 409 with Retry-After for
  in-flight concurrent retries.
- Use "may" for: shorter TTLs on internal APIs, deriving keys server-side
  for first-party clients under a documented contract, extending dedup to
  PATCH endpoints when If-Match isn't practical.

## Troubleshooting

- **"Duplicate charges during a retry storm"**: dedup store check and
  payment commit are in separate transactions, racing. Make them atomic
  (single DB transaction or SETNX before commit).
- **"Client sees 409 on what they believe was a successful call"**: replay
  is returning 409 instead of the stored response. Store the original
  success response and replay it byte-equal.
- **"A new SDK version is duplicating operations"**: the key is derived
  from the body and the new SDK added a field. Switch to a
  client-generated UUID the SDK persists across retries.
- **"TTL too short, real retries after one hour duplicate"**: extend TTL
  to cover the upstream retry window. Async queues with exponential
  backoff commonly exceed an hour.
- **"Hit-rate is zero in production"**: clients aren't sending the
  header or SDK isn't generating stable keys. Audit one client-library
  release and add a contract test.

## Concrete Example

A fintech's POST /payments gets double-charged during a flaky week — the
mobile app retries after a 30-second timeout, then a DLQ redelivers the
same message an hour later. The team adds Idempotency-Key (mandatory),
stores keys in DynamoDB with 24h TTL, and uses a conditional-put to claim
the key inside the same transaction that records the charge. Replay
returns the original response byte-equal. Post-launch dedup hit-rate is
2.3% (retries happen more than anyone thought), duplicate-charge tickets
drop to zero, and the DLQ reprocess flow becomes safe to automate.

## Sources

- `[[concepts/design-cases]]` — idempotency as recurring pattern
  (Craigslist, Messaging)
- `[[concepts/nfr-system-design]]` — fault-tolerance patterns including
  at-most-once semantics
- Zhiyong Tan, *Acing the System Design Interview*, Chapter 7 (Craigslist)
  and Chapter 14 (Messaging). Replay contract paraphrased from Stripe's
  published idempotency-key behaviour via Danilo's wiki paraphrase.
