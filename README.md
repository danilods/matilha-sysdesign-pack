# matilha-sysdesign-pack

> **You lead. Agents hunt.**
> Matilha companion pack — system design skills.

> 🏠 **This is a companion pack.** The official matilha entry point is [**danilods/matilha-skills**](https://github.com/danilods/matilha-skills) — start there for the full install guide + ecosystem overview. Install this pack only after the core is set up (via `/matilha-install` in Claude Code, or the explicit commands below).

19 skills synthesized from Zhiyong Tan's *Acing the System Design Interview* plus practical NFR clarification patterns and 11 design case studies. Auto-activates when user intent touches distributed systems, scalability, non-functional requirements, or architecture decisions.

## What this pack covers

- **NFR framework** (6 skills): clarifying questions before design, scalability (horizontal vs vertical), availability/SLA tiers, fault-tolerance patterns, latency targets, consistency/CAP.
- **Common services** (3 skills): load balancers (L4 vs L7), rate limiting strategies, 4 golden signals monitoring.
- **Design patterns** (5 skills): idempotency, CDN + object store, Kafka event streaming, dead letter queue, dual-write + event sourcing.
- **Case-specific patterns** (3 skills): news feed fan-out, autocomplete trie, top-K with count-min sketch.
- **Methodology** (2 skills): 50-min interview flow, explicit tradeoff framing.

## Install

```bash
/plugin marketplace add danilods/matilha-sysdesign-pack
/plugin install matilha-sysdesign-pack@matilha-sysdesign-pack
```

**Recommendation: install at user scope** so the pack is available in any workspace where you plan distributed systems. See the matilha core companions-contract.md for install-scope guidance.

## Relationship to matilha core

This pack is a **companion** to the [matilha](https://github.com/danilods/matilha-skills) methodology plugin. When matilha is installed, its `matilha-compose` gateway detects this pack via plugin-namespace inspection (`matilha-*-pack`) and injects pack-aware preamble into brainstorming sessions whenever user intent touches system-design concerns.

The pack works standalone too — skills auto-activate via their own descriptions when the user prompt matches their trigger phrases, even without matilha core installed.

## Overlap with other matilha packs

- `sysdesign-nfr-clarification` **complements** `matilha-harness-pack:harness-nfrs-as-prompts`. sysdesign encodes NFRs as clarifying questions during system spec/design; harness encodes NFRs as system-prompt constraints for AI agents. Different angles of the same "requirements before decisions" principle.

## Source attribution

- **Zhiyong Tan** — *Acing the System Design Interview* (Manning Publications). Primary source for all 19 skills. All content is paraphrased — no verbatim copy from the book.

## Skills inventory

See [docs/skills-inventory.md](docs/skills-inventory.md) for the full list with descriptions.

## Contributing

Pack authors follow the matilha [skill-authoring-guide.md](https://github.com/danilods/matilha-skills/blob/main/docs/matilha/skill-authoring-guide.md) and [pack-authors.md](https://github.com/danilods/matilha-skills/blob/main/docs/matilha/pack-authors.md).

## License

MIT — see [LICENSE](LICENSE).
