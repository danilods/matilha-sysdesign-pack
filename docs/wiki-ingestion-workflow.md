# Wiki ingestion workflow — matilha-sysdesign-pack

5-step workflow used to synthesize skills from raw wiki content while respecting paraphrase discipline (3 layers of remove: skill → wiki → source book).

## Step 1 — Select triggering intent

For each candidate skill, start from the user intent that should trigger it. Examples:
- `sysdesign-rate-limiting-strategies` — "Como implementar rate limiting?" / "Qual algoritmo pra limitar requests?"
- `sysdesign-latency-targets-techniques` — "Como reduzir latency?" / "Como atingir P99 < 100ms?"

The description must start with "Use when" or equivalent and match natural user phrasing for that intent.

## Step 2 — Load wiki sources

Read the relevant wiki pages (concepts + source summaries). For sysdesign-pack, the primary sources are:
- `wiki/concepts/nfr-system-design.md` — 10 NFRs + load balancers + 4 golden signals + interview flow
- `wiki/concepts/design-cases.md` — 11 case summaries + recurring patterns table
- `wiki/concepts/dual-write-event-sourcing.md` — dual-write + event sourcing tradeoffs
- `wiki/sources/acing-system-design.md` — source summary
- `wiki/sources/raw/acing-system-design-interview.md` (if present) — raw chapter notes

For each skill, pull 2-5 relevant sections. Do not copy — read, understand, then close.

## Step 3 — Paraphrase + synthesize

Apply 3 layers of remove:

1. **Source book** → wiki concept page (already paraphrased in wiki; we don't touch the source directly)
2. **Wiki concept page** → skill body (our paraphrase layer — rewrite in our own words)
3. **Skill body** → actionable guidance (apply to user context; not just textbook summary)

Rules:
- **One concrete example per principle** — do not enumerate every possibility from the book. Pick the most teachable example and go deep.
- **No verbatim sentences** — if a line reads too close to the source, rewrite. Flip clause order, change verbs, use Portuguese where the source was in English (or vice versa).
- **Add matilha framing** — show how the principle applies during software construction (spec authoring, clarifying questions, architecture decisions).

## Step 4 — Validate against style guide

Each skill must pass:
- Frontmatter: `name`, `description` (starts "Use when" or "When"), `category: sysdesign`, `version: 1.0.0`, `requires: []`, `optional_companions: []`.
- 12 required body sections: When this fires / Preconditions / Execution Workflow / Rules: Do / Rules: Don't / Expected Behavior / Quality Gates / Companion Integration / Output Artifacts / Example Constraint Language / Troubleshooting / CLI shortcut.
- Plus `## Sources` section (pack-specific — core skills don't have this).
- Body length: 150–300 lines (longer content goes to `references/<topic>.md`).
- Directory: `skills/<slug>/SKILL.md`.

## Step 5 — Cross-check activation uniqueness

Grep descriptions across the pack (and cross-pack where possible):

```bash
grep -rh "^description:" skills/*/SKILL.md | sort
```

If two skills have ≥80% word overlap in descriptions, refactor one — they'll compete for activation and confuse matilha-compose's routing.

Cross-pack: compare against `matilha-harness-pack`, `matilha-ux-pack`, `matilha-growth-pack` descriptions. Overlap is allowed if it's complementary (different angle on same concern) — in that case, disclose in the skill's `## Companion Integration` section using the formula:

> Complements `matilha-<otherpack>:<slug>` at <angle>

Or when the overlap is foundational without direct sibling:

> No direct matilha-<otherpack> sibling; this skill is foundational.
