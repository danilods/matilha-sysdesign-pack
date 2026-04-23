---
name: sysdesign-interview-flow-50min
description: Use when approaching a new system design — a 50-minute flow (requirements → API → data model/arch → deep dive → monitoring) that doubles as a spec-authoring template.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires on "how do I structure a system design review?", "como estruturar
uma spec técnica?", "I have 50 minutes to design X", "estamos abrindo um
desenho novo — por onde começo?", and on any fresh system design prompt
where no framework is already in play. Although the flow comes from
interview prep, it is load-bearing for real work: a team that runs this
flow for a new service ends up with a better spec than one that dives
straight into boxes-and-arrows. The skill enforces the five phases in
order, with a time budget per phase that prevents the common failure
mode — spending 40 minutes on the "fun" data model and 5 minutes on
requirements.

## Preconditions

- The design is genuinely new (no prior spec, no existing service to
  evolve). For incremental changes, a lighter ADR-style flow is
  usually better.
- The human or team driving the design can commit 50 minutes (or a
  proportional block — 25, 75, 100 minutes scale the same shape).
- A whiteboard or shared doc exists where artifacts from each phase
  persist. Flow phases that produce nothing in writing are
  indistinguishable from free-form discussion.
- There is a rough problem statement — one or two sentences of what
  the system needs to do. Not a spec, but a seed.

## Execution Workflow

1. **Phase 1 — Requirements (10 min, 20% of budget).** Ask what the
   system must do (functional) and what constraints it must satisfy
   (non-functional). Delegate the non-functional half to
   `sysdesign-nfr-clarification` if present. By the end of this phase,
   the board has: 3-7 functional requirements, a first-pass answer on
   scalability / availability / latency / consistency / cost, and any
   assumption explicit in writing. If phase 1 takes less than 8
   minutes, you did not ask enough; if more than 15, force a timebox.
2. **Phase 2 — API draft (5 min, 10%).** Sketch the handful of
   endpoints or RPCs the system exposes. REST verbs and paths, or
   function signatures. Do not design auth headers or pagination
   yet — just the shapes. This phase grounds the data model in what
   the system actually does for its callers.
3. **Phase 3 — Data model + high-level architecture (10 min, 20%).**
   Name the entities (one line each), pick a storage shape per entity
   (relational, document, KV, time-series, object store), and draw
   the top-level boxes: clients, gateway, services, stores, queues.
   Mark which NFR each box is there for. The diagram is chunky, not
   detailed — details belong to phase 4.
4. **Phase 4 — Deep dive on critical components (15 min, 30%).** Pick
   the 1-3 components that carry the hardest NFR load (highest QPS,
   tightest consistency, biggest blast radius) and zoom in. Data
   partitioning, cache strategy, failure modes, and their interaction.
   If the interviewer or reviewer cares about a specific component,
   follow their lead here — phase 4 is where the real signal lives.
5. **Phase 5 — Logging, monitoring, edge cases (10 min, 20%).** Name
   the 4 golden signals per service, the failure cases (what if the
   DB is down? what if a queue backs up? what if a dependency
   returns 500s?), and the mitigation per case. Include a first-pass
   answer on security and privacy if the system touches user data.
   This phase is where amateurs run out of time; defend it.
6. At the end, do a 2-minute self-summary — "here's what I built, here
   are the three things I didn't solve, here's what I'd revisit".
   Knowing what you didn't solve is a signal of design maturity.
7. Persist all five artifacts (requirements list, API draft, data +
   arch diagram, deep-dive notes, monitoring + edge cases) as a
   single doc, dated and signed. This is the spec.

## Rules: Do

- Enforce phase time boxes with a timer. A spec-authoring session that
  drifts to "let's finalize the auth flow" in minute 40 has abandoned
  the structure that makes the flow work.
- Write down assumptions the moment they are made. "Assume single
  region for now" is a different design from the multi-region version
  and must be tagged, not remembered.
- Let phase 1 drive phase 4. The deep-dive component selection is a
  function of which NFRs bind hardest; if phase 1 was weak, phase 4
  will be arbitrary.
- Use the flow for real specs, not just interviews. A 50-minute
  structured session beats three 20-minute ad-hoc meetings for any
  greenfield service.
- Timebox aggressively on phase 2 (API). The API shape is a scaffold,
  not the deliverable; over-investing here starves phase 4.

## Rules: Don't

- Don't skip phase 1 "to save time". Every skipped requirement becomes
  a redesign in week three.
- Don't do deep-dive before the high-level architecture exists. You
  will optimize a component that isn't needed or miss one that is.
- Don't reach phase 5 with five minutes left. That is the signal to
  trim earlier phases next time, not to compress the observability
  answer.
- Don't treat the flow as rigid ritual. If the problem is "design a
  rate limiter", 50% of the budget on phase 4 is fine — the shape
  scales to the problem.
- Don't forget to write the "what I didn't solve" list. A spec that
  claims to solve everything is untrustworthy.

## Expected Behavior

After this skill, any new system design session produces a five-part
artifact in roughly 50 minutes: requirements, API, data+arch,
deep-dive, and monitoring/edge cases. Time pressure stops being an
excuse for skipping observability. The team has a common vocabulary
for what "done enough for first review" means.

In interview settings, the candidate avoids the two classic failures
(all-whiteboard no-requirements, or five minutes of everything with
no depth) and lands a deep-dive phase that earns the grade.

## Quality Gates

- All five phases have written output, even if short.
- Requirements include both functional and non-functional.
- Deep-dive targets the component(s) identified by NFR pressure, not
  by personal interest.
- Phase 5 names failure modes, not just happy-path monitoring.
- A "what I didn't solve" list exists and is specific.

## Companion Integration

Delegates phase 1's NFR half to `sysdesign-nfr-clarification`. Hands
phase-4 decisions to the pattern skills (`sysdesign-newsfeed-fanout`,
`sysdesign-autocomplete-trie-fuzzy`, `sysdesign-top-k-count-min-sketch`,
`sysdesign-dual-write-event-sourcing`, etc.) when applicable. Phase-5
observability delegates to `sysdesign-monitoring-4-golden-signals`.
When `matilha` core is present, this flow composes naturally with
`matilha-plan` in phase 20-30 (spec + plan authoring). Methodology
phase: 20-30 primarily; 10 for discovery when the design doubles as
a feasibility probe.

## Output Artifacts

- A single dated spec doc with five labeled sections
  (Requirements, API, Data + Architecture, Deep Dive, Monitoring +
  Edge Cases).
- "Not solved" appendix listing known gaps.
- Optional: timer log showing time spent per phase (useful for
  calibrating next session).

## Example Constraint Language

- Use "must" for: all five phases produce written output, phase 5
  receives at least 10 minutes, deep-dive targets NFR-bound components.
- Use "should" for: timeboxing with a literal timer, delegating NFR
  clarification to `sysdesign-nfr-clarification`, persisting the
  spec dated.
- Use "may" for: scaling the flow to 25/75/100-minute sessions
  proportionally, skipping phase 2 when the API is dictated (e.g.,
  gRPC proto already exists).

## Troubleshooting

- **"We ran out of time before monitoring"**: phases 2-4 bloated.
  Review the log; typically phase 2 (API) or a favorite subsystem in
  phase 4 consumed the overflow. Next session, set harder caps.
- **"The deep-dive felt arbitrary"**: phase 1's non-functional output
  was vague. No binding NFR means no signal for which component
  deserves depth. Reinvest in phase 1 next time.
- **"Interviewers say I lack depth"**: phase 4 is underpowered.
  Trade phases 2-3 for more phase 4; interviews reward depth on the
  hard component, not breadth on the easy ones.
- **"Real projects don't fit in 50 minutes"**: correct. Run the flow
  multiple times per subsystem. The 50-minute shape is the unit; a
  project of five subsystems is five units, scheduled separately.

## Concrete Example

A team opens a 50-minute design session for a usage-metering service
that bills customers on API calls. Phase 1 (11 min) surfaces four
functional requirements (count, aggregate, expose, bill-export) and
NFR pressure on accuracy, idempotency, and 24-hour data retention
with eventual consistency on roll-ups. Phase 2 (4 min) sketches two
endpoints: `POST /events` and `GET /aggregates/:customer/:window`.
Phase 3 (9 min) picks Kafka + a time-series store for raw events,
Postgres for daily roll-ups, and a stream processor between them.
Phase 4 (17 min) deep-dives on the stream processor — exactly-once
semantics, checkpointing, late event handling, and a count-min
sketch for approximate live view (delegated to
`sysdesign-top-k-count-min-sketch` patterns). Phase 5 (9 min) names
monitoring (ingestion lag, processor backlog, roll-up divergence)
and failure cases (Kafka outage, processor crash, DB write
rejection). A two-minute self-summary lists three things not solved:
cross-region replication, retention-based cost bounds, and
customer-tier-specific SLOs. The spec is one page, dated, signed,
and in the repo before the hour is up.

## Sources

- `[[concepts/nfr-system-design]]` — 50-min interview flow section
- `[[concepts/design-cases]]` — pattern across all 11 cases
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (introductory chapters on interview methodology).
