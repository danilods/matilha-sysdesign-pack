---
name: sysdesign-tradeoff-framing
description: Use when a team is about to make an architectural choice (X vs Y, SQL vs NoSQL, sync vs async) — forces explicit tradeoff articulation tied to NFR priorities.
category: cog
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires whenever a conversation reaches a pivot point — "devemos usar X
ou Y?", "prós e contras de microservices aqui", "SQL ou NoSQL?",
"síncrono ou assíncrono?", "monolito vs microservices", "Redis ou
Memcached?", "push ou pull?". Also fires on the softer signal of
unrecognized tradeoffs: when a team is about to pick something because
someone on the team "always uses it", or because it is the newest. The
skill installs Tan's core philosophy — system design is an art, not a
science, and every decision is a tradeoff — and forces an explicit
articulation that prevents hand-wave choices.

## Preconditions

- There is a concrete decision on the table with at least two viable
  options. If only one option is real, this skill does not apply; the
  decision is pre-made and the team is rationalizing.
- The team has some sense of NFR priorities for the system under
  design (scalability, availability, latency, consistency, cost, …).
  Without that, tradeoffs are unmoored and collapse to personal
  preference.
- The team is authorized to actually choose. Exercises that frame
  tradeoffs for a decision that will be made elsewhere are
  educational, not load-bearing.

## Execution Workflow

1. State the decision in one sentence, with both options named.
   "Should the session store be Redis or Postgres?" — not "how do we
   store sessions?". Bad framings produce hand-wave answers.
2. Enumerate options. Two is a debate; three forces real thinking.
   If the third option is "hybrid" or "it depends", keep it in the
   table as its own row — "it depends" with no configuration is not
   a choice.
3. For each option, write what it gives you and what it costs. The
   template is literal: "Option A gives X but costs Y." Keep the
   grammar because it prevents one-sided boosterism. If an option
   only gives things and costs nothing, the analysis is incomplete
   and you need to dig until the cost shows up.
4. Tie costs and benefits to the NFR priorities that matter for this
   system. Delegate to `sysdesign-nfr-clarification` if the priorities
   are unclear. A benefit in an NFR the team doesn't care about is
   not a benefit here, even if it looks impressive on a slide.
5. Name the binding constraint. Usually one or two NFRs dominate the
   decision — latency budget, consistency contract, cost ceiling. The
   option that respects the binding constraint wins even if it loses
   on the softer axes.
6. Write the decision with explicit reasoning. The template: "Option A
   gives X but costs Y. Option B gives Z but costs W. Given our NFR
   priorities (<cite the ones that mattered>), we choose A because
   <the X beats the W under our priorities>." Not "A seems better".
7. Persist the analysis as an ADR. The decision document is not
   paperwork; it is the artifact that prevents the next team from
   re-litigating the choice on vibes six months later.
8. Add a revisit trigger. The right decision today expires when a
   specific fact changes — traffic grows 10x, a new product surface
   changes consistency needs, a cost ceiling shifts. Name the trigger
   in the ADR.

## Rules: Do

- Write the "gives X but costs Y" form literally. The grammar is the
  point; it prevents the discussion from sliding into marketing.
- Reference the NFR priorities by name. Tradeoffs untethered from
  NFRs devolve into preference.
- Name the binding constraint before declaring the winner. This is the
  step that converts a list of bullets into a decision.
- Include "do nothing" or "defer" as an option when it is viable. Many
  architectural choices can be postponed until the system's actual
  behavior selects the answer — that counts as a real option.
- Preserve the rejected options in the ADR. Future readers benefit
  more from knowing why X was rejected than from knowing only that Y
  was chosen.

## Rules: Don't

- Don't accept "best practice" as a rationale. Best practice is a
  tradeoff someone else resolved in a context that isn't yours.
- Don't pretend the winning option has no cost. Every option has
  cost; hiding the cost is the pattern that produces regret in
  quarter three.
- Don't let personal familiarity decide. "I've shipped X three times"
  is a real signal (team comfort, time to delivery) but it is not
  the same signal as "X fits the NFRs". Keep them separate rows.
- Don't decide before the NFRs are clear. If the binding constraint is
  unknown, the tradeoff cannot be resolved — stop and clarify NFRs
  first.
- Don't skip the ADR. An undocumented decision is a re-negotiable
  decision.

## Expected Behavior

After this skill, decisions land with written reasoning, explicit
NFR citation, and a revisit trigger. Discussions that used to drift
("let's think about it more" or "we can always change it later") get
pinned to a specific document. The team stops hand-waving and stops
confusing personal preference with architectural fit.

Future teammates reading the ADR can reconstruct not just what was
chosen but why — and, critically, under what conditions the choice
would change.

## Quality Gates

- The decision is stated in one sentence with named options.
- Each option has both "gives" and "costs" articulated.
- NFR priorities are cited by name in the reasoning.
- The binding constraint is identified explicitly.
- An ADR exists with the full analysis; a revisit trigger is named.

## Companion Integration

Tightly paired with `sysdesign-nfr-clarification` — without NFRs, this
skill has nothing to anchor to. Used downstream by every pattern
skill in the pack (`sysdesign-dual-write-event-sourcing`,
`sysdesign-newsfeed-fanout`, `sysdesign-top-k-count-min-sketch`, etc.)
whenever their internal tradeoff tables need to be resolved against
real priorities. With `matilha-harness-pack` installed,
`harness-tradeoff-articulation` (if present) encodes a related
principle for agent-authored decisions. Methodology phase: 20-30
(spec + plan authoring) primarily; any phase where a binding choice
appears.

## Output Artifacts

- ADR with: decision sentence, options table, gives/costs per option,
  NFR priorities cited, binding constraint named, decision with
  reasoning, revisit trigger.
- Optional: rejected-options appendix for future readers.

## Example Constraint Language

- Use "must" for: the "gives X but costs Y" form for each option, NFR
  citation by name, an ADR capturing the decision.
- Use "should" for: three options on the table when possible, a named
  revisit trigger, preserving rejected options in writing.
- Use "may" for: including "do nothing" as an option, running the
  analysis as a short timeboxed meeting rather than an async doc.

## Troubleshooting

- **"We can't agree on the winner"**: the NFR priorities are
  unresolved or contested. Stop the tradeoff discussion and resolve
  priorities first; the decision falls out once priorities are firm.
- **"Every option looks equally good"**: the analysis is too coarse.
  Dig into the costs — usually one option has a cost the team has
  been dancing around (operational burden, hiring pipeline, vendor
  lock-in) and surfacing it resolves the tie.
- **"We chose X and now it's causing pain"**: check the revisit
  trigger. If the named condition was hit and the decision was not
  revisited, that is the process failure to fix, not the decision.
  If the pain reflects a condition no one predicted, add that to the
  trigger list and make a new decision from the current state.
- **"The team keeps going in circles"**: usually one stakeholder has
  an unstated constraint. Surface it. "What would make option B
  unacceptable to you?" often reveals the hidden priority.

## Concrete Example

A team is choosing between per-request database transactions and a
write-behind buffered batch for an analytics ingestion path. Framed
sloppily, the debate lasts days and loops on "performance". Framed
with this skill: "Option A (per-request txn) gives durability and
simple semantics but costs 3x the DB write capacity at peak. Option
B (write-behind batched) gives capacity headroom but costs a
recovery window where recent writes can be lost on crash. Option C
(outbox + batched publisher) gives both durability and capacity but
costs an extra moving piece and weeks of ops learning. Given our
NFR priorities (availability > cost > feature velocity this quarter),
we choose C because losing recent writes on crash violates our
availability SLA, and 3x write capacity breaches our cost ceiling.
Revisit when daily event volume drops 50% or when availability SLA
changes." The ADR takes twenty minutes, and the next three debates
about this path end by pointing at it.

## Sources

- `[[concepts/nfr-system-design]]` — tradeoff philosophy
- `[[concepts/design-cases]]` — tradeoff patterns across 11 cases
- `[[concepts/scaling-databases]]`
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (recurring framing across the book: "system design is an art, not
  a science — everything involves tradeoffs").
