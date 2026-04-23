---
name: sysdesign-dual-write-event-sourcing
description: Use when a design writes to two stores on one action (DB + Kafka, DB + cache, DB + search) or when deciding whether event sourcing is worth its complexity.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires whenever a diagram shows a single user action branching into two
independent persistences — classic cases include "insert into Postgres and
publish to Kafka", "update the row and invalidate the cache", "save the
order and mirror it into Elasticsearch". Also fires on explicit framings
like "event sourcing vale a pena aqui?" or "devemos guardar eventos ou só
o estado atual?". The skill surfaces the dual-write anti-pattern, walks
through the three consistent alternatives (transactional outbox, CDC,
event sourcing as source of truth), and forces an honest accounting of
what event sourcing actually costs before it is adopted.

## Preconditions

- There is at least one write path that currently touches (or is proposed
  to touch) two stores in the same request. If the design is single-store,
  this skill is not the right one.
- The team can articulate what "inconsistent state" would cost the
  business on this path (lost orders, double charges, stale search
  results). Without that number, the tradeoff call is hand-waved.
- There is a clear read pattern for the second store. Event sourcing is
  seductive in the abstract and painful when no one has thought about how
  the admin UI will page through a million events.
- If event sourcing is on the table, the team has a rough sense of schema
  evolution cadence — how often event shapes will change over one year.

## Execution Workflow

1. Draw the current (or proposed) write path and mark the two stores
   explicitly. Ask: what happens if the first write commits and the
   second fails? If the answer is "we retry" or "we log it", the design
   already has a silent data-drift bug — name it.
2. Introduce the dual-write anti-pattern as a vocabulary item. Two
   separate writes with no shared transaction cannot be atomic across
   networks; assume divergence will happen at the first sustained
   outage.
3. Walk through the three alternatives and their shapes:
   - **Transactional Outbox**: write the row and the event row in the
     same DB transaction; a background relay publishes from the outbox
     table. Simple, keeps the DB as source of truth.
   - **Change Data Capture (CDC)**: use Debezium/native log tailing to
     derive the event stream from the DB write-ahead log. Zero app code,
     but adds operational surface (Kafka Connect, schema registry).
   - **Event sourcing as source of truth**: the event log is primary,
     the relational store becomes a derived projection. Strongest story
     for auditability and replay, heaviest lift for everything else.
4. Score each alternative on the four axes that actually differ: read
   complexity, schema evolution cost, replay/audit value, ops surface.
   Present the scores before recommending, not after.
5. If event sourcing wins the scoring, pin down the painful parts
   explicitly: schema versioning policy, snapshot cadence, GDPR erasure
   strategy (immutable logs + right-to-be-forgotten is a real problem).
   A skill that ships event sourcing without those three answers has
   only transferred the pain to Q4.
6. If the pattern chosen is dual-write for lack of better options, make
   the divergence visible — emit a reconciliation job with alerts on
   drift rather than pretending the two stores stay in sync.
7. Persist the decision as an ADR with the tradeoff table attached. This
   is the main artifact the future team will reread when the pattern
   stops fitting.

## Rules: Do

- Name "dual-write anti-pattern" out loud whenever a design has two
  writes on one action. The vocabulary alone often flips the discussion
  toward outbox/CDC within minutes.
- Treat event sourcing as a source-of-truth decision, not a logging
  enhancement. If the event log is "nice to have" next to a relational
  primary, the team has accidentally signed up for dual-write under a
  fancier name.
- For event sourcing: require an explicit schema-evolution policy
  (upcasters, versioned event types, weak-schema payload fields) before
  the first event is written. Retrofitting this is the common failure
  mode.
- Keep one store as source of truth and the other as derived projection.
  When both sides claim authority, conflict resolution becomes policy,
  not engineering.
- Add a reconciliation/audit job even for "correct" outbox or CDC
  designs — the third line of defense against silent drift during
  deploy gaps.

## Rules: Don't

- Don't accept "we'll write to both and retry on failure" as a design.
  That is dual-write with optimism attached; durability engineering
  isn't optimism.
- Don't adopt event sourcing because the domain sounds auditable. Audit
  logs and event sourcing are not the same thing — you can get audit
  from a CDC stream without paying the source-of-truth cost.
- Don't design event sourcing without snapshots and a replay budget.
  Replaying two years of events on startup is a production outage
  waiting for its invitation.
- Don't let the outbox relay lag become invisible. Ship a lag metric and
  page on growth past a known threshold; a silent outbox is a silent
  dual-write.
- Don't conflate CQRS with event sourcing. CQRS is read/write split; it
  does not require the event store to be primary.

## Expected Behavior

After this skill, every dual-write on the whiteboard is either
converted to outbox/CDC/event-sourcing with a clear source of truth, or
it keeps the dual-write shape with eyes open and a reconciliation job
attached. The team can state, in one sentence per store, which is
primary and how derivations stay honest.

Event sourcing is either adopted with the three hard answers (schema
versioning, snapshots, erasure) in place, or rejected in favor of a
cheaper pattern with the same audit benefits.

## Quality Gates

- No design on the whiteboard has two uncoordinated writes.
- Source of truth is explicit and written down, one per data family.
- If event sourcing is adopted: schema versioning policy, snapshot
  cadence, and GDPR erasure strategy exist as prose, not intent.
- Outbox or CDC designs include a visible relay-lag metric and alert.
- An ADR captures the decision with the tradeoff scoring preserved.

## Companion Integration

Pairs with `sysdesign-tradeoff-framing` for the scoring-before-recommend
flow and with `sysdesign-nfr-clarification` when "strong vs eventual
consistency" is still open. When `matilha-harness-pack` is installed,
`harness-nfrs-as-prompts` can encode the chosen consistency contract as
an agent-side constraint. Methodology phase: 20-30 (spec + plan) and 40
(dispatch), depending on when the decision is made.

## Output Artifacts

- ADR (Architecture Decision Record) with the four-axis tradeoff table.
- Updated design diagram marking source of truth and derivation arrows.
- For event sourcing: schema-versioning policy note, snapshot cadence
  note, erasure strategy note (one paragraph each is fine; vagueness
  here is the problem).
- Reconciliation/lag-monitoring job spec when applicable.

## Example Constraint Language

- Use "must" for: naming a single source of truth per data family,
  ADR capturing the decision, monitoring outbox/CDC lag.
- Use "should" for: preferring outbox or CDC over event sourcing when
  audit is the only driver, including snapshots in any event-sourced
  design.
- Use "may" for: adopting event sourcing as source of truth when the
  domain genuinely demands replay and full audit, keeping a legacy
  dual-write with reconciliation while migrating to outbox.

## Troubleshooting

- **"The outbox relay is always behind"**: measure lag end-to-end and
  audit publisher batch size, poll interval, and consumer throughput.
  Sustained lag is usually a consumer problem masquerading as a
  publisher problem.
- **"Event store queries are impossible from the admin UI"**: that is
  the cost working as designed. Add a read-model projection (SQL or
  Elasticsearch) fed from the events; do not query the event log
  directly from the UI.
- **"We need to delete a user's data, but our events are immutable"**:
  separate PII from event metadata. Store PII behind a key and delete
  the key (crypto-shredding), keeping the event shape intact for
  downstream consumers.
- **"Schema evolution is a nightmare after six months"**: the policy
  was implicit. Adopt versioned event types with upcasters and keep a
  tolerant reader; greenfield ES without this policy will hit this
  wall without exception.

## Concrete Example

A checkout service writes the order to Postgres and publishes
`OrderPlaced` to Kafka. During a broker blip, three orders land in
Postgres with no Kafka event — downstream billing never charges them.
The team names the pattern (dual-write), switches to an outbox table
inside the same Postgres transaction, and runs a relay that publishes
committed outbox rows to Kafka. Four weeks later, someone asks for a
complete audit trail of every order state change; because the outbox
already captures domain events, they add a projection into a
queryable store instead of reaching for event sourcing. The team
avoided the event-sourcing cost while keeping consistency and audit.

## Sources

- `[[concepts/dual-write-event-sourcing]]`
- `[[concepts/scaling-databases]]`
- `[[concepts/nfr-system-design]]` — consistency section
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (chapter on scaling databases; event sourcing and saga coverage).
