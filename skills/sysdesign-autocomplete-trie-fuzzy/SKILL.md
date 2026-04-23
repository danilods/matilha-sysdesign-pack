---
name: sysdesign-autocomplete-trie-fuzzy
description: Use when designing autocomplete / search suggestions — weighted trie for top-k prefix matches, query sampling, fuzzy matching, and pre-serve moderation.
category: sysdesign
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires on prompts like "design autocomplete for search", "sugestão de
queries em tempo real", "type-ahead for product names", or "como o Google
sugere enquanto digito". Also fires when an existing autocomplete is
listed as too slow, too stale, or too permissive ("it suggested a banned
term"). The skill installs the four load-bearing pieces: a weighted trie
for top-K-by-prefix, query sampling so the index does not grow with
every search, fuzzy matching for typos and phonetics, and a content-
moderation filter that runs before any suggestion leaves the server.

## Preconditions

- The team can describe where suggestions come from — historical search
  queries, catalog entries, or both. Autocomplete with no corpus is a
  different problem (cold-start generation, not retrieval).
- There is a latency budget on the keystroke path (typical: <50ms from
  keystroke to rendered suggestions). Without it, the design collapses
  to "just query Elasticsearch".
- The team can articulate what counts as a "popular" query — a rough
  threshold (e.g., queries seen more than 10 times in the last 30 days).
  This threshold drives sampling.
- A moderation source of truth exists or can be bootstrapped (banned-
  terms list, slur filter, regulatory terms). Shipping autocomplete
  without this will surface the platform's worst content.

## Execution Workflow

1. Model the data structure. A weighted trie stores prefixes as paths
   and associates each node with the top-K completions ranked by weight
   (query frequency, recency, personalization signal). Reads are
   O(prefix length + K) — fast enough for the keystroke path.
2. Precompute per-node top-K at index time, not query time. Walking the
   subtree on every keystroke burns the latency budget; store the
   top-K list directly at each node and refresh it during index
   updates.
3. Introduce query sampling explicitly. Do not index every query the
   platform has ever seen — the long tail is noise and personal data.
   Keep queries above a popularity threshold; log everything but index
   only the sampled set. This keeps the trie memory-bounded and the
   suggestions quality-gated.
4. Layer fuzzy matching for typos. Options, in increasing cost order:
   (a) deletion-based indexing (Wang et al.) — precompute variants with
   one deletion per position; (b) Levenshtein automata against the
   trie — higher precision, more memory; (c) phonetic indexing
   (Soundex/Metaphone) for proper-name-heavy corpora. Pick one based on
   the corpus, not on all three.
5. Place moderation at the edge of the suggestion service — filter
   candidates against the banned list after trie retrieval and before
   the response leaves the server. Moderation at index time is not
   enough; terms get added to the list after indexing, and you must
   still block them on the read path.
6. Design the update path. The trie is read-heavy; build the new trie
   offline, swap atomically (pointer flip or rolling update), and keep
   the old trie around for rollback during the swap window. Avoid
   mutating the live trie in place.
7. Capacity plan. Store the trie in memory on the suggestion service
   (Redis with custom structures or an in-process C/Rust structure are
   both common). Shard by prefix when memory per instance exceeds
   comfort; shard routing is then a prefix-hash lookup.
8. Ingestion must absorb spikes. Accept and log every search query
   regardless of index state; the sampler runs offline on the log. A
   spike that overruns the indexer should not drop searches.

## Rules: Do

- Precompute top-K at each trie node; refresh during index build, not
  during read.
- Sample queries before indexing — the long tail is not content, it is
  noise plus PII.
- Run moderation after retrieval and before response, even if the
  corpus was pre-filtered. Lists evolve faster than indexes.
- Build the new trie offline and swap atomically. In-place mutation of
  a live read structure is how autocomplete services become unreliable.
- Log all search traffic for the sampler; index a subset. Log and index
  are different systems with different retention rules.

## Rules: Don't

- Don't index every query. The platform's long tail is mostly unique
  and mostly unhelpful as suggestions; you will also accidentally
  index PII typed by users (emails, phone numbers).
- Don't run Elasticsearch on the keystroke path without a trie cache in
  front. ES is fine as a fallback, not as a default for type-ahead.
- Don't skip phonetic/fuzzy for human-name corpora. Proper names are
  where typos are most common and exact-prefix suggestions feel
  broken.
- Don't mutate the trie live. Rebuild and swap, or pay for it later
  with stale partial writes.
- Don't rely on index-time moderation alone. The banned list is updated
  after incidents; the read-path filter is the enforcement point.

## Expected Behavior

After this skill, the autocomplete design has a clear separation:
logs capture everything, a sampler selects popular queries, a trie-
builder produces a versioned artifact, and the read service serves
from the current trie with a post-retrieval moderation pass. Latency
on the keystroke path stays inside the budget and is decoupled from
total corpus size.

Conversations about "why didn't it suggest X" get a specific answer
(X was below sampling threshold, or X was moderated out) rather than
"the search is weird".

## Quality Gates

- Keystroke-path p99 latency stays under the declared budget under
  realistic load.
- Moderation filter runs on every response; tested with a list-update
  scenario that post-dates the index.
- Index updates are atomic — no partial-tree reads possible.
- Sampling threshold is explicit and tunable; memory usage has a
  documented ceiling per shard.
- Ingestion logs all queries even during indexer downtime.

## Companion Integration

Pairs with `sysdesign-rate-limiting-strategies` (autocomplete endpoints
are abuse magnets), `sysdesign-cdn-object-store` (static parts of the
suggestion UI), and `sysdesign-monitoring-4-golden-signals` for the
keystroke-path SLIs. With `matilha-ux-pack` installed,
`ux-perceived-performance` covers debounce, cancellation, and render
cadence on the client — the frontend half of a fast autocomplete.
Methodology phase: 20-30 (spec + plan) when building new, 10
(discovery) when diagnosing an existing autocomplete.

## Output Artifacts

- Architecture diagram with ingestion → log → sampler → trie builder →
  read service → moderation filter → response.
- Data-structure note describing the trie node, top-K storage, and
  fuzzy strategy chosen.
- Sampling-threshold document, dated, with the rationale.
- Banned-terms integration point marked on the diagram.
- Capacity plan with memory ceiling per shard and rebuild cadence.

## Example Constraint Language

- Use "must" for: post-retrieval moderation, atomic index swap, a
  latency budget on the keystroke path.
- Use "should" for: a fuzzy strategy matched to the corpus (phonetic
  for names, deletion-based for short words, Levenshtein automata for
  precision-heavy cases), a sampling threshold above the noise floor.
- Use "may" for: personalization layered on top of the global top-K,
  per-locale tries when the corpus splits cleanly by language.

## Troubleshooting

- **"Suggestions feel stale"**: rebuild cadence is too slow. Check the
  sampler → builder → swap latency end-to-end; the bottleneck is
  usually the builder step for large corpora.
- **"P99 blows up under typing bursts"**: either the trie is evicted
  from cache (check memory pressure) or the service is saturating on
  the fuzzy step. Profile; Levenshtein automata on every keystroke is
  expensive.
- **"It suggested a banned term"**: moderation ran only at index time,
  and the term was added after the last build. Add the filter at
  response time; both layers should exist.
- **"Ingestion dies during traffic spikes"**: the ingestion path is
  coupled to the indexer. Decouple: log first, sample later. The
  indexer can lag; logging cannot.

## Concrete Example

A marketplace launches product-search autocomplete backed directly by
Elasticsearch queries on every keystroke. p99 lands at 240ms (budget
was 50ms) and during peak, the search cluster starts shedding writes.
The team introduces a weighted trie: last 30 days of queries above
ten occurrences become the sampled corpus, top-10 completions are
precomputed per node, and phonetic indexing is layered for product
names. A banned-terms filter runs on every response. P99 drops to
18ms, ES load halves, and when a new banned term is added after a
press incident, it is filtered within minutes without a reindex.

## Sources

- `[[concepts/design-cases]]` — Design Autocomplete case
- `[[concepts/nfr-system-design]]` — latency and accuracy sections
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (chapter 11, Design Autocomplete).
