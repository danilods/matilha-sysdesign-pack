---
name: sysdesign-newsfeed-fanout
description: Use when designing a social feed (Twitter, Facebook, Instagram-style) and weighing fan-out on write vs fan-out on read, especially with celebrity-scale accounts.
category: cog
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Fires on any "design a news feed / timeline / following feed" framing —
Twitter clones, Facebook feeds, Instagram-style follow graphs, even
internal dashboards where posts from N producers must appear on M
consumers' screens. Also fires whenever the conversation names the
"celebrity problem" ("what happens when one user has ten million
followers?"). The skill walks the team through the two canonical
approaches, exposes why pure forms of each break at scale, and lands on
the hybrid design that most production feeds actually run.

## Preconditions

- There is a social graph with a clear producer/consumer (follower)
  relationship. Chronological feeds from a single source do not need
  this skill.
- The team knows (or can estimate) the rough follower distribution —
  typical median vs 99th percentile. If everyone has ~50 followers and
  no one has more, pure fan-out on write is fine and this skill is
  overkill.
- Read latency budget for the feed is on the table (e.g., "feed must
  render in <200ms P99"). Without a latency budget, the tradeoff
  collapses to taste.
- Write load expectations are articulated, even loosely. Fan-out on
  write at 10M daily posts looks very different from 10K.

## Execution Workflow

1. Quantify the graph. Ask for average followers, median followers, and
   the 99th and 99.9th percentile. The gap between median and p99.9 is
   the signal that decides the design — a median of 50 with a p99.9 of
   500 is a different system from a p99.9 of 50M.
2. Explain fan-out on write: when a user posts, the system pushes the
   post ID into each follower's materialized feed (Redis/sorted-set
   style). Read is O(page size) — cheap and fast. Write is
   O(followers) — unbounded for celebrities.
3. Explain fan-out on read: on feed load, fetch recent posts from each
   followed user and merge. Read is O(following × recent posts) —
   expensive. Write is O(1) — trivial.
4. Name the celebrity problem. A pure fan-out-on-write system dies when
   one account has tens of millions of followers: a single tweet
   generates tens of millions of Redis writes, which queue up and push
   p99 latency past the budget for every other user on the platform.
5. Land on the hybrid. Classify accounts by follower count; above a
   threshold (common: ~10K-100K followers), skip fan-out on write for
   that producer and materialize their posts on read instead. The
   follower's feed becomes "merge pre-computed feed with live fetches
   from the celebrities I follow." Most followers don't follow many
   celebrities, so the read cost stays bounded.
6. Add content moderation inline, before either path writes anything
   visible. Moderation after fan-out means pulling posts from millions
   of materialized feeds during takedown — expensive and racy.
7. Design for read/write asymmetry: feeds are read far more than
   written. Cache aggressively on the read path; accept staleness in
   seconds, not minutes.
8. Persist the design with a concrete threshold, explicit data layout
   (who owns the materialized feed store, what the merge query looks
   like), and a monitoring plan for the hybrid boundary.

## Rules: Do

- Design for the follower distribution you actually have, not the
  average. Averages hide the celebrity problem completely.
- Run content moderation before fan-out commits, not after. Takedowns
  are cheap when the post was never materialized to millions of feeds.
- Set the celebrity threshold as a tunable parameter, not a constant.
  As the platform grows, the right threshold moves.
- Cache materialized feeds in an in-memory store (Redis sorted sets are
  canonical) with a bounded length (e.g., last 500 items). The tail is
  fetched on demand.
- Measure p99 latency per feed type (normal vs celebrity-heavy follower)
  and alert when the celebrity-heavy path degrades — that is the
  signal the threshold needs to move.

## Rules: Don't

- Don't run pure fan-out on write "for simplicity" if the p99.9
  follower count is above a few thousand. "Simple" is another word for
  "unobserved failure mode" here.
- Don't run pure fan-out on read at scale. Merging hundreds of feeds
  on every page load burns read capacity and blows the latency budget.
- Don't materialize unbounded feeds. Feeds that keep every post a user
  ever saw grow without bound and destroy cache hit rates.
- Don't skip ranking/personalization considerations. If the feed is
  ranked (not just chronological), fan-out on write materializes a
  chronological list that still needs a ranker on read — plan for
  the ranker's cost.
- Don't invalidate feeds synchronously on delete. Mark-and-filter on
  read; cleanup is a background job.

## Expected Behavior

After this skill, the design shows a hybrid fan-out with an explicit
celebrity threshold, inline moderation, a bounded materialized feed
store, and a merge-on-read path for celebrity producers. The team can
explain, in one diagram, what happens when a median user posts vs when
a celebrity posts vs when a follower loads their feed.

Discussions about "what if Taylor Swift joins" stop being hypothetical
and get a concrete answer: her posts bypass fan-out and merge on read
for her followers.

## Quality Gates

- Follower distribution is quantified (at least median and p99.9).
- Celebrity threshold is explicit and configurable.
- Content moderation runs before fan-out, not after.
- Materialized feeds are bounded in length and have a TTL or eviction
  policy.
- Monitoring covers p99 feed-load latency segmented by follower-graph
  shape (how many celebrities the user follows).

## Companion Integration

Pairs with `sysdesign-event-streaming-kafka` (fan-out writes are often
Kafka-backed jobs), `sysdesign-dead-letter-queue` (failed per-follower
writes need a home), and `sysdesign-interview-flow-50min` when the
feed design is the interview or spec prompt itself. With
`matilha-ux-pack` installed, `ux-perceived-performance` covers how
stale-while-revalidate feels to the user. Methodology phase: 20-30
(spec + plan) for greenfield feeds; 10 (discovery) for "why is our
feed slow?" investigations.

## Output Artifacts

- Architecture diagram with both fan-out paths and the threshold
  router.
- Data model for the materialized feed (store, schema, length bound,
  eviction).
- Moderation hook placement, explicit in the diagram.
- Monitoring plan with the segmented-latency SLI.

## Example Constraint Language

- Use "must" for: moderation before fan-out commit, bounded
  materialized-feed length, p99-per-segment monitoring.
- Use "should" for: adopting the hybrid design when p99.9 follower
  count exceeds roughly 10K, caching materialized feeds in Redis
  sorted sets, tuning the celebrity threshold based on load.
- Use "may" for: running pure fan-out on write for small or private
  networks where the p99.9 follower count stays bounded, delaying
  ranking to a separate service.

## Troubleshooting

- **"Feed loads are fast for most users but slow for a few"**:
  investigate which users — usually they follow many celebrities and
  the merge-on-read path dominates. Consider caching the merged
  result briefly, or per-celebrity-post caches.
- **"Celebrity posts take minutes to appear for their followers"**:
  fan-out queue is backlogged. Either the celebrity is above threshold
  and should be on merge-on-read, or the queue needs more consumers.
  Do not "just add fan-out workers" indefinitely — that's the pattern
  the threshold exists to end.
- **"Content that violated policy was seen by millions before takedown"**:
  moderation ran after fan-out. Move it inline. For long-tail takedowns,
  implement filter-on-read by content ID without touching materialized
  feeds.
- **"Deletes leave orphaned entries in followers' feeds"**:
  expected. Materialized feed entries are post IDs; the read path
  dereferences them and filters out deleted posts. Do not chase
  deletes across millions of feeds.

## Concrete Example

A Twitter-style clone launches with pure fan-out on write on Redis
sorted sets. At 200K users it hums. When a 15M-follower account is
imported, a single post from that account queues 15M Redis writes,
pushes p99 feed-load latency from 80ms to 2.3s platform-wide, and
triggers a cascading timeout storm. The team introduces a hybrid:
accounts above 50K followers are flagged as high-fanout, their posts
are not pushed into followers' sorted sets, and the feed-load path
merges each follower's materialized feed with a live fetch from the
<=50 high-fanout accounts they follow. P99 returns to ~95ms, celebrity
posts propagate in under two seconds, and the platform survives the
next viral event without a war room.

## Sources

- `[[concepts/design-cases]]` — Design News Feed case
- `[[concepts/nfr-system-design]]` — latency and scalability sections
- `[[concepts/scaling-databases]]`
- Synthesized from Zhiyong Tan, *Acing the System Design Interview*
  (chapter 16, Design News Feed).
