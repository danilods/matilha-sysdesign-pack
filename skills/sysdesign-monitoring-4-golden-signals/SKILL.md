---
name: sysdesign-monitoring-4-golden-signals
description: Use when deciding what to monitor and alert on — applies the 4 golden signals (Latency, Traffic, Errors, Saturation) and splits page-worthy from dashboard-only.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when a service is about to go live, is drowning in noisy alerts, or is
missing real outages because its monitoring was organised by metric source
instead of user impact. Fires when someone says "we should monitor more
things" without separating what wakes up an on-call engineer from what
belongs on a dashboard. The skill installs Latency, Traffic, Errors, and
Saturation as the four primary axes and assigns each signal a role — page,
ticket, or dashboard-only.

## Preconditions

- The service has a metrics pipeline (Prometheus, Datadog, CloudWatch,
  OpenTelemetry) wired up. This skill doesn't install agents.
- There is a defined SLO or, at minimum, a stated user-perceived latency
  target. Without it, "latency is high" is an opinion, not an incident.
- Someone owns the on-call rotation. A page without an owner is a missed
  page.
- The team can distinguish user-facing endpoints from internal-only ones.
  Golden signals apply to both but with different thresholds.

## Execution Workflow

1. List the user-facing surfaces of the service (top 3 to 7 endpoints by
   traffic or by business value). Golden signals are scoped per surface,
   not per host. Use Read or the observability tool of record to pull a
   baseline week of data for each.
2. For each surface, define Latency. Prefer P99 for user-facing, P50 for
   internal batch. Document the target next to the number — "checkout P99
   under 800ms" is useful, "checkout latency is monitored" is not.
3. Define Traffic. Requests per second (or per minute for low-volume) is
   the default; for event-driven systems use events-per-partition. Traffic
   alone rarely pages — it is the denominator that makes the other three
   signals interpretable.
4. Define Errors. Split 4xx and 5xx — they have different root causes.
   5xx rate is almost always page-worthy above a threshold (often 1% of
   traffic for user-facing). 4xx rate usually dashboard-only because
   clients cause most 4xx, though a spike can indicate a deploy
   regression.
5. Define Saturation. CPU, memory, disk, and queue depth per service. The
   saturation number that actually matters is utilisation of the tightest
   resource, not a CPU average. Disk filling is a classic missed-page
   cause — always include free-space alerts with lead time.
6. Assign each threshold to one of three roles. Page-worthy means 24/7 wake
   up an engineer. Ticket-worthy means create an incident within hours.
   Dashboard-only means show but do not alert. A system where every signal
   pages is untenable; a system where nothing pages is unobserved.
7. Wire burn-rate or multi-window alerting for SLOs. A flat threshold on
   latency causes alert storms during minor blips. A burn-rate rule
   (fast-burn + slow-burn) fires once on a real regression.
8. Add downstream symptoms. Golden signals on the service under design are
   necessary but not sufficient — include the golden signals of its
   primary downstream (DB, cache, message broker). A DB saturation
   incident shows first as latency on the service.

## Rules: Do

- Split Errors into 4xx and 5xx in the dashboard and alert rules. Treating
  them as one hides deploy regressions behind client noise.
- Use P99 for user-facing latency alerts. Averages hide the tail that
  users actually feel.
- Page on symptoms (slow checkouts, elevated 5xx) not causes (CPU > 80%).
  Cause-based paging wakes engineers for non-incidents.
- Include downstream saturation in your panels. Your latency regression
  often starts in someone else's queue depth.
- Name a runbook link inside each page-worthy alert. An alert without a
  runbook is a 3am Google search.

## Rules: Don't

- Don't page on CPU or memory alone for user-facing services. Autoscaling
  may resolve it before anyone reads the page; users don't care about CPU
  if latency is fine.
- Don't use the same error threshold for read and write paths. Write
  errors are almost always worse and deserve a tighter bound.
- Don't leave Saturation as a single number. "CPU average" is a lie across
  a cluster; percentile utilisation per host tells the truth.
- Don't create alert rules without a silence/acknowledge flow. Unanswered
  pages train engineers to ignore all pages.
- Don't monitor only the green path. Deploy-time regressions surface in
  4xx shape and queue depth long before they show up as 5xx.

## Expected Behavior

After applying the skill, each user-facing surface has four numbers with
thresholds: P99 latency, RPS, 4xx/5xx error rate, and the tightest
saturation metric. Each threshold is labelled page, ticket, or
dashboard-only. Every page-worthy rule links to a runbook. On-call noise
drops; real regressions surface faster because the signal-to-noise ratio
improved.

When an incident happens, the first question is "which golden signal
moved first?" — not "which graph should we look at?"

## Quality Gates

- Per-surface dashboard exists with four panels: Latency (P99), Traffic,
  Errors (split 4xx/5xx), Saturation (tightest resource).
- Each alert rule has a role label (page/ticket/dashboard) and a runbook
  link.
- 5xx page-worthy threshold expressed as a rate against traffic, not a
  raw count.
- Burn-rate or multi-window rules used for SLO-backed alerts, not flat
  thresholds.
- Downstream saturation panel included in the service's own dashboard.
- On-call rotation owner documented for every page-worthy rule.

## Companion Integration

Pairs with `sysdesign-rate-limiting-strategies` (block-rate per scope is a
golden-signal variant), `sysdesign-fault-tolerance-patterns` (circuit
breaker state is a saturation proxy), and
`sysdesign-dead-letter-queue` (DLQ depth is a queue saturation signal that
belongs on this dashboard). No direct UX or growth companion — this skill
is the operational backbone every other pack-pattern leans on.

## Output Artifacts

- A dashboard (JSON or screenshot) per user-facing surface with the four
  signals.
- An alert-rules file (Prometheus rules, Datadog monitor export, etc.)
  version-controlled beside the service code.
- A runbook stub per page-worthy alert, even if the initial content is
  "check deploys, check downstream".
- Design-doc section "Observability" naming signals, thresholds, and
  alert roles.

## Example Constraint Language

- Use "must" for: splitting 4xx from 5xx in alert rules, using P99 for
  user-facing latency, linking a runbook from every page-worthy alert.
- Use "should" for: burn-rate alerts on SLO-backed services, downstream
  saturation panels on the service dashboard, per-host saturation
  percentiles over cluster averages.
- Use "may" for: adding business-KPI panels (signup-rate, conversion) next
  to the four signals, paging on saturation directly for systems without
  autoscaling, using P95 instead of P99 for very low-volume surfaces.

## Troubleshooting

- **"Alert storms every deploy"**: flat threshold on latency. Replace with
  a burn-rate rule or a multi-window rule that requires sustained
  deviation.
- **"We missed an outage that lasted 30 minutes"**: the signal was there
  but not page-worthy, or the threshold was above the actual regression.
  Re-baseline with the post-incident data and tighten.
- **"CPU pages but users aren't affected"**: paging on cause instead of
  symptom. Move CPU to ticket-worthy and add a latency-based page.
- **"Disk filled up, we only noticed when writes failed"**: saturation
  alert missing or set too late. Add a free-space alert with at least 24
  hours of lead time based on growth rate.
- **"4xx spike during a deploy looked identical to client abuse"**: 4xx
  wasn't split by endpoint or by error code. Break out 400, 401, 404, 429
  as separate series.

## Concrete Example

A payments service rolls out new monitoring. Surfaces: POST /charge, GET
/invoice, POST /refund. For POST /charge the team sets P99 < 500ms
(page), 5xx rate > 0.5% over 5 minutes (page), 4xx rate > 5% over 15
minutes (ticket), DB connection pool saturation > 85% (page), traffic as
dashboard-only. Runbooks live in the repo and are linked from each
page-worthy rule. Two weeks later, a deploy regression doubles P99 for
/charge. The burn-rate alert fires once — not a storm — and on-call finds
the bad migration in under ten minutes using the runbook's "check deploys
in the last hour" step.

## Sources

- `[[concepts/nfr-system-design]]` — 4 golden signals section
- Google SRE Book, *Monitoring Distributed Systems* chapter (origin of
  the four signals), referenced via `[[concepts/nfr-system-design]]`
- Zhiyong Tan, *Acing the System Design Interview*, logging and
  monitoring sections across Chapters 3 and 17. Burn-rate framing
  adapted from SRE Workbook via Danilo's wiki paraphrase.
