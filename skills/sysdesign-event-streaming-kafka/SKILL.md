---
name: sysdesign-event-streaming-kafka
description: Use when deciding between Kafka and a simpler queue — picks Kafka for decoupling, ordered delivery, and replay, or rejects it for low-volume point-to-point work.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when a team is about to add Kafka (or debating it against SQS,
RabbitMQ, Pub/Sub, or a DB outbox). Fires when someone says "let's use
Kafka" without naming partition key, consumer groups, replay use case, or
ordering requirement. Also fires the other direction — when someone
reaches for Kafka for a low-volume point-to-point job that a queue would
handle at a tenth of the operational cost. The skill produces a written
yes/no with the three load-bearing reasons and, if yes, the partitioning
and consumer-group topology.

## Preconditions

- There is a concrete event flow in mind (producer → topic → consumers).
  This skill is not for vague "we need events" conversations; capture the
  flow first.
- The team can name at least three properties per event that would appear
  in the message schema. Kafka without schema discipline degenerates fast.
- Someone can answer "do we need replay?" and "do events within a group
  need strict ordering?" — the two questions that most distinguish Kafka
  from simpler queues.
- Operational appetite exists. Kafka pays rent: brokers, ZooKeeper or
  KRaft, monitoring, schema registry, consumer-lag tracking. Teams that
  cannot absorb that should not pick Kafka.

## Execution Workflow

1. Collect the three questions that decide the choice. Do we need replay
   from a point in history (debugging, backfill, new consumer bootstrap)?
   Do events within a logical group need strict ordering? Will more than
   one independent consumer need the same stream? Two or three yeses make
   Kafka the right tool. One yes or none usually means a queue (SQS,
   Pub/Sub) or a DB outbox will serve better.
2. If Kafka is rejected, pick the alternative explicitly. Low-volume
   point-to-point with retry and DLQ → SQS. Fan-out pub/sub without
   replay → Pub/Sub or SNS+SQS. Transactional change stream from a DB →
   outbox pattern + CDC. Name the chosen tool in the design doc so the
   discussion doesn't reopen.
3. If Kafka is accepted, design the topic plan. One topic per bounded
   context of event, not one topic per consumer. A `payments.events`
   topic carries many event types; a `payments-notifications-consumer`
   topic is an anti-pattern that destroys decoupling.
4. Pick the partition key. Keys must cluster events that need to be
   processed in order together. For a messaging app, user_id or
   conversation_id. For orders, order_id. Round-robin (null key) is
   acceptable only when ordering truly doesn't matter — in practice,
   rarely. Wrong key choice is expensive to fix later.
5. Size the partition count for future scale. Consumer parallelism is
   capped at partition count; start with more partitions than current
   consumers need (3x to 10x). Partition count can go up but down is
   painful.
6. Define consumer groups. Each independent downstream system is its own
   group so each gets the full stream. Scaling a group is adding
   consumers up to partition count; beyond that, more partitions.
7. Commit policy and at-least-once handling. Commit offsets AFTER
   side-effect completes, not before. Combine with
   `sysdesign-idempotency-patterns` for consumer-side dedup — at-least-
   once + idempotent consumer equals effectively-exactly-once at the
   business level.
8. Wire observability: consumer lag per group, under-replicated partitions,
   producer send-error rate. Consumer lag is the single most useful signal
   — growing lag means a consumer is falling behind and the incident is
   in flight.
9. Define retention. Compacted topics for "latest state per key" semantics
   (user profile snapshots). Time-based retention for event logs (7 days,
   30 days). Forever-retention is usually wrong; revisit against storage
   cost.

## Rules: Do

- Answer the three gating questions (replay, ordering, multi-consumer)
  before picking Kafka. Two yeses is usually the threshold.
- Key partitions by the natural entity that demands ordering. user_id,
  order_id, conversation_id — whichever groups events that must stay in
  sequence.
- Over-provision partitions up front. Growing consumers is cheap; growing
  partitions mid-life is messy.
- Commit offsets after the side effect succeeds. Consumer-side idempotency
  (via `sysdesign-idempotency-patterns`) absorbs the duplicates this
  creates.
- Track consumer lag per group as a first-class alert. Lag is the
  clearest signal that the system is degrading.

## Rules: Don't

- Don't use Kafka for low-volume point-to-point jobs. The operational
  cost swamps the benefit; SQS or a DB outbox is simpler and cheaper.
- Don't create a topic per consumer. The point of Kafka is that one
  topic serves many consumers independently; per-consumer topics
  reintroduce coupling.
- Don't round-robin partition a stream that has any ordering requirement.
  Debugging out-of-order processing in production is brutal.
- Don't commit offsets before the side effect. A consumer crash between
  commit and side-effect drops messages silently.
- Don't leave retention forever-by-default. Storage cost grows linearly
  and old events are almost never read.

## Expected Behavior

After applying the skill, there is a written yes/no decision citing the
three gating questions, a topic plan, a partition key per topic, a
consumer-group map, a commit policy, and observability wired for lag.
Teams stop adopting Kafka reflexively and stop rejecting it reflexively;
the call is motivated by replay, ordering, and multi-consumer needs, not
by resume-building or infrastructure anxiety.

## Quality Gates

- Decision record cites the three questions and the answers that drove
  the choice.
- If yes: topic plan lists topics with partition key and partition count.
- Consumer groups enumerated with purpose and expected lag SLO.
- Commit policy (after side-effect, with idempotent consumer) named in
  the design.
- Retention policy per topic documented.
- Consumer-lag alert threshold set per group with a page-worthy rule.
- Schema registry or equivalent schema-evolution plan named.

## Companion Integration

Pairs tightly with `sysdesign-idempotency-patterns` (consumer-side dedup
makes at-least-once safe), `sysdesign-dead-letter-queue` (poison-pill
handling per consumer group), and
`sysdesign-monitoring-4-golden-signals` (consumer lag and producer error
rate belong on the service dashboard). Case-study skills like
`sysdesign-newsfeed-fanout` and notifications patterns often depend on
this choice being made correctly.

## Output Artifacts

- Design-doc section "Messaging / Event streaming" with the decision and
  topic plan.
- A `topics.yaml` (or equivalent) listing topic name, partition key,
  partition count, retention, schema.
- Consumer-group map (producer and consumers per topic).
- Dashboard panels for lag, under-replicated partitions, producer
  send-error rate.
- Schema registry link or a versioned schema folder in the repo.

## Example Constraint Language

- Use "must" for: naming partition key per topic, committing offsets
  after side effect, tracking consumer lag per group, documenting
  retention.
- Use "should" for: schema registry adoption, over-provisioning
  partitions 3x to 10x current need, compacted topics for
  latest-state-per-key use cases.
- Use "may" for: exactly-once transactional producers when business
  value justifies the complexity, round-robin partitioning for truly
  order-insensitive fan-out, multi-region Kafka replication for
  disaster recovery.

## Troubleshooting

- **"Chose Kafka but we only have one producer and one consumer"**: the
  three-question test likely returned one yes or none. Consider
  migrating to SQS / Pub/Sub / outbox before the Kafka cluster becomes
  a sunk cost.
- **"Out-of-order processing caused a data corruption"**: partition key
  is wrong or null. Re-key by the entity that demands ordering, even
  if that requires a topic migration.
- **"Consumer lag spikes every deploy"**: partition count is capping
  parallelism. Add partitions ahead of scaling consumers.
- **"A poison-pill message stalled a consumer for hours"**: no DLQ
  wired. Add one per consumer group — see `sysdesign-dead-letter-queue`.
- **"Storage cost is growing uncontrollably"**: retention is unbounded
  or a compacted topic was never designed. Set explicit retention per
  topic and revisit quarterly.

## Concrete Example

A notifications service starts on SQS and outgrows it when marketing
adds three new downstream consumers that each need the full event
stream and two of them need replay for backfills. The team runs the
three-question test: replay (yes), ordering per user (yes), multi-
consumer (yes). Moving to Kafka with `notifications.events` keyed by
user_id, 60 partitions, 7-day retention. Each downstream is its own
consumer group with lag SLOs. A poison-pill during a template regression
doesn't stall everyone — per-group DLQs isolate it. Three months later,
backfilling a new analytics consumer takes a replay from offset zero
instead of a warehouse migration.

## Sources

- `[[concepts/design-cases]]` — Kafka as recurring pattern (Notifications,
  Auditing, Feed, Dashboard)
- `[[concepts/nfr-system-design]]` — async communication and scalability
- Zhiyong Tan, *Acing the System Design Interview*, Chapters 9, 10, 16,
  17. Three-question gating framing is Danilo's synthesis of the
  "decouple + replay + multi-consumer" criteria Tan applies case by
  case — not a direct quote.
