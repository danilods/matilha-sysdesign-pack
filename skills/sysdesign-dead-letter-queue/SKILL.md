---
name: sysdesign-dead-letter-queue
description: Use when handling message failures in a queue or stream — installs a DLQ with retry policy, backoff, alerting on growth, and a reprocess-after-fix flow.
category: matilha
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when a service consumes messages (SQS, Kafka, RabbitMQ, Pub/Sub) and
some messages will inevitably fail — malformed payloads, missing
references, transient downstream errors, code bugs that surface only on
specific inputs. Fires when someone says "we'll just retry" without
naming the retry cap, backoff shape, where poison-pill messages go, or
who is paged when the DLQ grows. The skill installs a concrete DLQ with
retry policy, growth alerting, and a reprocessing flow so that failed
messages isolate rather than stall the consumer.

## Preconditions

- A consumer exists (or is being designed) that processes messages from a
  queue or stream. Direct request/response work doesn't need this skill.
- The messaging substrate supports DLQ primitives or the team is willing
  to implement one (most managed services do; Kafka requires a separate
  topic).
- Someone owns the on-call rotation that will receive DLQ-growth pages.
  An unowned DLQ is a landfill.
- The business has an opinion on "can we afford to drop a message after
  N retries?" — different answers drive very different retry policies.

## Execution Workflow

1. Classify failure modes. Transient (downstream timeout, deploy blip) —
   retry will succeed. Permanent (malformed payload, missing foreign-key
   reference, deleted target) — retry will never succeed. Unknown (new
   exception in code) — usually permanent until a fix ships. The retry
   policy and DLQ behaviour differ per class.
2. Set the retry policy for transient failures. Exponential backoff with
   jitter (1s, 2s, 4s, 8s, 16s, 32s — with random jitter so consumers
   don't thunder together). Cap attempts at 5 to 10 for most workloads.
   More attempts mostly burn money; fewer miss legitimate recoveries.
3. After the retry cap, move the message to the DLQ. Do not keep
   retrying forever — a single poison message will pin one consumer
   thread indefinitely and starve the healthy work behind it. The DLQ is
   explicitly an isolation mechanism, not a trashcan.
4. Preserve context when moving to DLQ. Attach the original message, the
   exception trace, the attempt count, a timestamp, and the consumer
   version. A DLQ message without this context is unusable in an
   incident — no one can tell what went wrong or when.
5. Alert on DLQ growth, not absolute size. A DLQ with 3 messages that's
   been 3 for a month is healthy noise; a DLQ adding 50 messages per
   hour is an incident. Page on rate-of-change over a window. Any
   non-zero growth deserves human investigation within hours, not days.
6. Define the reprocess flow. After a fix ships, messages in the DLQ must
   be moveable back to the main queue. Ship this as a tested runbook
   (script, Lambda, CLI command), not an ad-hoc manual process. Expect
   to use it — not often, but reliably.
7. Handle idempotency at the consumer. Reprocessed DLQ messages may
   duplicate side effects if the consumer isn't idempotent. Pair this
   skill with `sysdesign-idempotency-patterns` and use dedup keys so
   safe replay is the default.
8. Bound DLQ retention. Unbounded DLQs accumulate forever and hide real
   trends. A retention window (14-30 days for most workloads) forces
   periodic review: if a message has been in the DLQ for a month, the
   organisation has decided to drop it; that decision should be explicit.

## Rules: Do

- Always pair a consumer with a DLQ. Consumers without a DLQ either
  retry forever (wasting resources and pinning partitions) or drop
  silently (losing data).
- Use exponential backoff with jitter. Synchronous retry loops thunder
  the downstream and extend outages.
- Preserve full context (message body, exception, attempt count,
  timestamp, consumer version) when writing to the DLQ. Post-incident
  debugging depends on it.
- Page on DLQ growth rate, not absolute depth. Growth is the signal an
  incident is active.
- Build and test the reprocess flow before it's needed in production. A
  runbook you've never executed will fail at 3am.

## Rules: Don't

- Don't let a poison message pin a consumer. If the consumer keeps
  retrying without a DLQ, healthy messages behind it starve.
- Don't retry permanent errors. Malformed payloads and missing
  references never fix themselves through repetition; route them to the
  DLQ on the first detectable signal.
- Don't ignore a growing DLQ. A non-zero DLQ growth rate is, by
  construction, messages the consumer was supposed to process and
  didn't — the business impact is already happening.
- Don't reprocess from the DLQ into a non-idempotent consumer. Side
  effects will duplicate. Fix idempotency first; reprocess second.
- Don't let the DLQ become a dumping ground. Set retention; if a
  message still sits there after the window, that is a decision to drop
  and must be made consciously.

## Expected Behavior

After applying the skill, every consumer has an attached DLQ with a
defined retry cap and backoff, full context preserved on every failed
message, and a growth-rate alert that pages on incident-level behaviour.
A tested reprocess-after-fix runbook exists. Incident response becomes:
check DLQ growth (did we start rejecting messages?), inspect a sample DLQ
message (what went wrong?), ship the fix, reprocess. Silent data loss
stops; poison-pill messages stop stalling the consumer line.

## Quality Gates

- DLQ wired per consumer group (not shared across unrelated consumers;
  one group's DLQ is noise to another).
- Retry policy documented: max attempts, initial backoff, multiplier,
  jitter, max backoff.
- DLQ message schema includes original body, exception trace, attempt
  count, timestamp, consumer version.
- Growth-rate alert exists with a page-worthy threshold; raw depth
  alert is dashboard-only.
- Reprocess runbook exists, has been tested at least once in staging.
- Retention policy set on the DLQ itself.

## Companion Integration

Pairs with `sysdesign-event-streaming-kafka` (DLQ as a separate Kafka
topic per consumer group), `sysdesign-idempotency-patterns` (safe
reprocessing requires idempotent consumers), and
`sysdesign-monitoring-4-golden-signals` (DLQ growth is a saturation
signal variant on the consumer's dashboard). The
`matilha-harness-pack:harness-evaluator-optimizer-loop` companion shares
the retry-with-cap-then-isolate shape at the agent loop level.

## Output Artifacts

- Design-doc section "Message handling" naming retry policy, DLQ,
  alerting, and reprocess runbook.
- Terraform / config entries for the DLQ resource(s) version-controlled.
- A reprocess runbook (script or Lambda) checked into the repo with
  usage documentation.
- Dashboard panel for DLQ depth and growth rate per consumer group.
- Alert rule file entries for DLQ growth.

## Example Constraint Language

- Use "must" for: DLQ per consumer, exponential backoff with jitter,
  preserving context on failed messages, growth-rate alerting, tested
  reprocess runbook.
- Use "should" for: 5-10 retry cap as a default, 14-30 day DLQ
  retention, pairing with idempotent consumer pattern, routing
  permanent errors to DLQ on first detection.
- Use "may" for: retry forever for truly transient low-cost failures
  within bounds (tight timeout, cheap downstream), separate DLQ
  consumers that auto-replay after cooldown, additional tiered retry
  queues between main and DLQ.

## Troubleshooting

- **"A malformed message stalled the consumer for two hours"**: retry
  cap is too high or absent, and there's no DLQ. Add DLQ, cap retries
  at 5-10, and route deserialisation failures to DLQ on attempt 1.
- **"DLQ has 50,000 messages and no one noticed"**: growth-rate alert
  missing; depth-based alerts rarely fire because depth climbs slowly.
  Add a rate-based page rule (e.g., > 10 new DLQ messages in 15 min).
- **"Reprocessed DLQ caused duplicate charges"**: consumer isn't
  idempotent. Ship dedup via `sysdesign-idempotency-patterns` before
  the next reprocess run.
- **"One consumer's failures are filling a shared DLQ"**: DLQs are
  shared. Split — one DLQ per consumer group — so failures isolate by
  owner.
- **"DLQ grew indefinitely, storage cost is hurting"**: retention is
  unbounded. Set 14-30 day retention and make the monthly review a
  standing meeting until the flow is tuned.

## Concrete Example

A notifications service consumes Kafka events and sends email/SMS/push.
A template regression ships that throws on a new locale code. Without a
DLQ, one poison message would pin a consumer and stall thousands of
notifications. The team has wired retry with exponential backoff (1s,
2s, 4s, jitter, cap 5), then routes failures to `notifications.dlq`
with the full exception trace. The growth-rate alert pages after 12
messages in 10 minutes. On-call reads a DLQ sample, identifies the bad
template, rolls back in 6 minutes. The reprocess runbook moves the
DLQ back into the main queue; the idempotent consumer (dedup by
notification_id) prevents any duplicates. Total user impact: 12 delayed
notifications.

## Sources

- `[[concepts/design-cases]]` — DLQ as recurring pattern
  (Notifications, Messaging)
- `[[concepts/nfr-system-design]]` — fault-tolerance patterns including
  DLQ and Circuit Breaker
- Zhiyong Tan, *Acing the System Design Interview*, Chapters 9
  (Notifications) and 14 (Messaging). Growth-rate alerting framing is
  Danilo's synthesis — Tan mentions DLQ alerting without specifying
  the rate-vs-depth distinction.
