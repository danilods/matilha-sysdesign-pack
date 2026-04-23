---
name: sysdesign-top-k-count-min-sketch
description: Use when designing a real-time top-K dashboard (top trending, top errors, top products) — count-min sketch for bounded memory, Lambda vs Kappa architecture, mandatory checkpointing.
category: cog
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires on "dashboard top 10", "trending now", "most frequent errors in the
last hour", "top products by views in real time", and similar framings
where the system must return approximate top-K over a firehose of events
with bounded memory and bounded latency. Also fires when an existing
top-K dashboard is described as "slow", "wrong after failures", or
"blowing up Redis memory". The skill installs the count-min sketch as
the default data structure, walks through Lambda vs Kappa architecture
for the batch+stream pipeline, and makes checkpointing non-negotiable
because batch jobs fail inevitably.

## Preconditions

- The cardinality of the key space is high enough that exact counting
  doesn't fit in memory (millions of distinct keys). If it fits in a
  plain hash map, this skill is overkill.
- The product tolerates approximate answers within a known error bound.
  Top-10 that says "approximately the right ten, maybe swapped at the
  boundary" is acceptable; financial reconciliation is not.
- There is a window semantics on the table — last hour, last 24h,
  sliding or tumbling. Without windowing, "top" is ambiguous.
- The team has (or is willing to stand up) a stream processor with
  checkpoint support (Flink, Kafka Streams, Spark Structured Streaming).

## Execution Workflow

1. Confirm the approximation tolerance. Count-min sketch trades exactness
   for O(ε, δ) memory: width w ≈ ⌈e/ε⌉, depth d ≈ ⌈ln(1/δ)⌉. Pick ε and
   δ from product requirements ("95% confidence the count is within 1%
   of true") and size the sketch from there.
2. Choose the architecture shape:
   - **Kappa (stream-only)**: events stream through a processor that
     maintains per-window sketches; top-K is served from the current
     sketch. Simpler; all logic lives in the stream code path.
   - **Lambda (batch + stream)**: batch produces authoritative historical
     top-K daily; stream produces approximate recent top-K; serving
     layer merges them. More moving pieces; useful when historical
     accuracy must be exact.
   Default to Kappa unless a concrete requirement demands exact history.
3. Design the sketch lifecycle. One sketch per tumbling window; rotate
   on window close; retain the last N windows for lookback. Sliding
   windows are implementable via overlapping sketches — more memory,
   smoother output.
4. Top-K extraction. The count-min sketch gives point queries
   (frequency of a known key) cheaply, not top-K directly. Pair it with
   a heavy-hitters structure (heap of candidates, Space-Saving
   algorithm, or a heap gated by sketch counts) to maintain top-K as
   events stream in.
5. Checkpoint mandatorily. Stream-processor crashes are not
   hypothetical; they happen. Serialize sketch state and heap state to
   durable storage (S3, HDFS, RocksDB-backed state backend) at a
   cadence tied to the window size — typically every 10-60 seconds for
   minute-scale windows.
6. Plan for late events. In stream-time processing, events arrive out
   of order. Define allowed lateness; route late events to the window
   they belong to within the grace period, drop or side-output
   afterward. Document the choice.
7. Guard memory. Sketch width is bounded by design; heap size is your
   responsibility. Cap the top-K heap at K + a small margin (e.g., 2K)
   and evict by sketch-count eagerly.
8. Expose serving via a read-only endpoint backed by the latest sealed
   window's sketch+heap. Do not query live in-flight state — racy and
   slow.

## Rules: Do

- Pick ε and δ from product requirements, then size the sketch.
  Defaulting to "large enough" means nothing and usually means too
  large.
- Checkpoint stream state at a cadence that keeps recovery within the
  SLO. If the window is a minute and recovery takes ten, the dashboard
  is lying during outages.
- Combine count-min sketch with a heavy-hitters structure. Sketch alone
  does not give you top-K; it gives you counts on demand.
- Define and test the late-event policy. The correct policy is domain-
  dependent; the wrong thing is to have no policy.
- Keep sketches immutable once sealed. Late-window updates mutate
  history and make dashboards non-reproducible.

## Rules: Don't

- Don't maintain exact counts over millions of keys in Redis hoping it
  fits. Memory will find the ceiling at the worst moment.
- Don't skip checkpointing because "we haven't had a crash yet". Batch
  jobs and stream jobs fail; checkpointing is what converts a failure
  into a recovery instead of a wipe.
- Don't serve top-K from the actively-updating sketch. Consumers get
  jittery numbers and screenshots that don't reproduce.
- Don't run both Lambda batch and Kappa stream "just in case" without
  a merge strategy. Two authoritative sources with no merge rule is
  worse than one approximate source with a clear rule.
- Don't forget to size the heap. An unbounded candidate heap is a
  memory leak with a dashboard on top.

## Expected Behavior

After this skill, the top-K dashboard has a documented error bound,
windowed sketches on a defined cadence, a heavy-hitters structure
producing top-K, mandatory checkpointing, and a clear serving surface
that only reads sealed windows. Recovery after a crash restores state
from the last checkpoint without losing windows already published.

The team can answer "how accurate is this dashboard?" with a number
and "what happens if the stream processor dies?" with a concrete
recovery path.

## Quality Gates

- ε, δ, window size, and K are explicit in the design doc.
- Checkpoint cadence is set; recovery time is measured and within SLO.
- Top-K is served from sealed windows only.
- Late-event policy is defined and documented.
- Memory ceiling per window is computed from ε, δ, and heap size; it
  matches actual consumption under load.

## Companion Integration

Pairs with `sysdesign-event-streaming-kafka` (upstream source for
events), `sysdesign-fault-tolerance-patterns` (checkpoint cadence is
the recovery backbone), `sysdesign-monitoring-4-golden-signals`
(latency and saturation on the stream processor). When
`matilha-ux-pack` is installed, `ux-data-clarity` covers how to show
"approximate" without destroying user trust. Methodology phase: 20-30
(spec + plan) for new dashboards, 40 (dispatch) when splitting the
work across a stream team and a serving team.

## Output Artifacts

- Design doc with ε, δ, width, depth, window, K, and memory estimate.
- Architecture diagram showing Kappa (or Lambda) shape and checkpoint
  targets.
- Late-event policy note.
- Recovery runbook that walks through checkpoint-to-service restart.
- Monitoring plan with sketch-saturation and checkpoint-lag SLIs.

## Example Constraint Language

- Use "must" for: checkpointing stream state, serving from sealed
  windows, explicit ε and δ in the design.
- Use "should" for: Kappa as default, tumbling windows unless the
  product needs sliding, heavy-hitters structure (Space-Saving or a
  sketch-gated heap) for the K extraction.
- Use "may" for: Lambda architecture when historical exactness matters,
  sliding windows for smoother output, per-shard sketches merged at
  serving time when a single sketch would not fit.

## Troubleshooting

- **"Top-K flicks between items at the boundary"**: expected for
  approximate counts at the edge of K. Widen the sketch (reduce ε) or
  hysteresis the UI (only replace an item if its count exceeds the
  incumbent's by a margin).
- **"Counts drift higher than true values"**: count-min sketch is
  biased toward overestimation by design. Quantify it; use count-mean-
  min-sketch if underestimation is acceptable, or widen to tighten the
  bound.
- **"Recovery after a crash loses the last ten minutes"**: checkpoint
  cadence is too slow relative to window. Tighten the cadence or
  accept replay from the upstream log (Kafka retention covers the
  gap if long enough).
- **"Dashboard disagrees with batch on yesterday's numbers"**: you have
  Lambda without a merge rule. Decide which wins for yesterday
  (usually batch) and codify the hand-off at a fixed time.

## Concrete Example

A platform wants "top 10 most-viewed product pages in the last 5
minutes" across a billion-event-per-day firehose. The team sizes a
count-min sketch at ε=0.001, δ=0.01 (w≈2720, d≈5) — roughly 54KB per
window, times 10 windows kept for lookback = under 1MB per shard. A
Space-Saving heap of size 20 is maintained alongside the sketch. Flink
runs the pipeline with checkpoints to S3 every 30 seconds. Tumbling
windows of 5 minutes seal every 5 minutes; the serving API reads only
sealed windows. During an AZ failure, the job resumes from the last
checkpoint, replays ~30 seconds of Kafka events, and the dashboard is
back inside the minute. Accuracy is documented as "within 0.1% of true
count with 99% confidence" — approximation, not wrongness.

## Sources

- `[[concepts/design-cases]]` — Design Dashboard Top 10 case
- `[[concepts/nfr-system-design]]` — accuracy, fault-tolerance sections
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (chapter 17, Design Dashboard Top 10).
