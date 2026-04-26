---
name: matilha-sysdesign-trigger
description: "Use when the user mentions system design, scalability, distributed systems, latency, throughput, availability, CAP theorem, database design, caching, rate limiting, CDN, microservices, SLA, NFR, capacity planning, bottleneck, message queue, or any topic related to large-scale system architecture and infrastructure design. Fires independently of compose to ensure matilha-sysdesign-pack skills activate whenever system design domain appears."
category: sysdesign-pack
version: "1.0.0"
---

## When this fires

User prompt mentions any keyword from matilha-sysdesign-pack's domain: system design, scalability, distributed systems, latency, throughput, availability, CAP theorem, database design, caching, rate limiting, CDN, microservices, SLA, NFR, capacity planning, bottleneck, message queue.

This skill fires independently of `matilha:matilha-compose` and `CLAUDE.md` activation rules — its keyword-rich description is the activation surface for the Skill tool's matcher. It guarantees that whenever the user prompt touches the system-design / infrastructure domain, at least one matilha-sysdesign-pack skill enters the conversation.

## Execution Workflow

1. **Pack presence check.** Inspect the ambient skill list for skills in the `matilha-sysdesign-pack:*` namespace. Examples: `matilha-sysdesign-pack:sysdesign-nfr-clarification`, `matilha-sysdesign-pack:sysdesign-consistency-cap`, `matilha-sysdesign-pack:sysdesign-rate-limiting-strategies`.

2. **Pack installed path.** If ≥1 skill from `matilha-sysdesign-pack` is visible:
   - Identify the user's sub-intent within the sysdesign domain (clarifying NFRs, picking strong-vs-eventual consistency, choosing a rate-limit algorithm, sizing a CDN, modeling fan-out, picking a queue, sketching a 50-min interview, etc.).
   - Emit a compact domain acknowledgment (≤2 lines — no full sigil, that belongs to compose). Example: `System design domain detected. Pulling matilha-sysdesign-pack guidance for <sub-intent>.`
   - Invoke the most relevant pack skill via the Skill tool. Prefer one specific skill over listing many.

3. **Pack not installed path.** If no `matilha-sysdesign-pack` skills are visible:
   - Emit: "System Design skills not installed. Run `/matilha-install` and select `sysdesign` to add 19 system design skills."
   - Proceed with default flow (`matilha:matilha-compose` if visible, otherwise `superpowers:brainstorming`).

## Rules

- Do NOT emit the full compose sigil. The sigil belongs to compose; this skill emits a compact pack-specific acknowledgment.
- Do NOT block on pack absence — emit the nudge and continue with the default flow.
- Prefer invoking a specific pack skill over listing all skills.
- This trigger is a routing surface, not a craft skill — hand off to the chosen pack skill quickly.

## Why this exists

Wave 5h adds deterministic activation surfaces to compensate for probabilistic Skill-tool matching. Compose's routing table covers the central path; this trigger covers prompts that bypass compose (e.g., when CLAUDE.md is absent, or when a domain keyword appears mid-conversation without a compose-classifiable phase). Together they form the Maximum Activation guarantee for matilha-sysdesign-pack.
