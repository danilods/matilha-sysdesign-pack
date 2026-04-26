# Changelog

## [0.3.0] — 2026-04-26 — Wave 5h: deterministic trigger skill

### Added

- **`matilha-sysdesign-trigger` skill** — independent activation surface for system-design domain. Keyword-rich description (scalability, distributed, latency, throughput, availability, CAP theorem, cache, rate limiting, CDN, microservices, system design, bottleneck, capacity planning, SLA, NFR, message queue, etc.) ensures pack skills enter the conversation whenever the domain appears in user prompts.
- Complements `matilha-skills`'s routing table (`skills/matilha-compose/routing-table.md`); together they form Wave 5h's Maximum Deterministic Activation surface.

### Notes

- Fully additive: existing 19 skills untouched. Pack continues to work standalone or with `matilha-skills` (Matilha core).
- Trigger is a routing surface, not a craft skill — emits a compact domain acknowledgment and hands off to the most relevant pack skill via the Skill tool.
- No behavior change when pack is uninstalled: trigger emits a `/matilha-install` nudge and yields to default flow.

## [0.1.0] — 2026-04-23 — Wave 5e: initial release

### Added

- 19 skills synthesized from Zhiyong Tan's *Acing the System Design Interview* (Manning):
  - **NFR family** (6): `sysdesign-nfr-clarification`, `sysdesign-scalability-horizontal-vs-vertical`, `sysdesign-availability-sla-tiers`, `sysdesign-fault-tolerance-patterns`, `sysdesign-latency-targets-techniques`, `sysdesign-consistency-cap`.
  - **Common services** (3): `sysdesign-load-balancers`, `sysdesign-rate-limiting-strategies`, `sysdesign-monitoring-4-golden-signals`.
  - **Design patterns** (5): `sysdesign-idempotency-patterns`, `sysdesign-cdn-object-store`, `sysdesign-event-streaming-kafka`, `sysdesign-dead-letter-queue`, `sysdesign-dual-write-event-sourcing`.
  - **Case-specific patterns** (3): `sysdesign-newsfeed-fanout`, `sysdesign-autocomplete-trie-fuzzy`, `sysdesign-top-k-count-min-sketch`.
  - **Methodology** (2): `sysdesign-interview-flow-50min`, `sysdesign-tradeoff-framing`.

- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` — plugin manifest + marketplace registration (owner + metadata + plugins[] schema matching Wave 5d.1 discovery).
- `README.md` — pack overview with install instructions and overlap disclosure.
- `docs/wiki-ingestion-workflow.md` — 5-step paraphrase-discipline workflow used to synthesize skills from raw wiki content.
- `docs/skills-inventory.md` — full inventory with descriptions and triggering intents.

### Overlap disclosures

- `sysdesign-nfr-clarification` **Complements** `matilha-harness-pack:harness-nfrs-as-prompts` — sysdesign encodes NFRs as clarifying questions during system spec/design; harness encodes NFRs as system-prompt constraints for AI agents.

### Notes

- Pack shipped following the matilha core skill-authoring-guide (12 required sections + frontmatter schema + paraphrase discipline).
- Pack self-activates via its own skill descriptions; matilha-compose (when matilha core installed) detects and enriches brainstorming sessions via plugin-namespace `matilha-*-pack`.
