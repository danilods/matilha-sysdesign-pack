# Changelog

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
