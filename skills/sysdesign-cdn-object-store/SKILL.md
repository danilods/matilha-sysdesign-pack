---
name: sysdesign-cdn-object-store
description: Use when serving images, video, or downloads globally — places a CDN in front of an object store, names invalidation strategy, and handles private-content auth.
category: matilha
version: "1.0.0"
requires: []
optional_companions: []
---

## When this fires

Use when the product needs to deliver static or semi-static assets
(photos, videos, documents, downloadable artifacts) to users around the
world with low latency and without hammering the origin. Fires when
someone says "just put a CDN in front of it" without naming the object
store, the invalidation strategy, or how private content is protected.
The skill defines the CDN + origin pair, the cache-control policy,
invalidation approach, and authentication flow for gated content.

## Preconditions

- There is identified static or cacheable content (media, JS/CSS bundles,
  PDFs, firmware). Dynamic per-user HTML needs a different pattern.
- The team has an object store available or can adopt one (S3, GCS, R2,
  Azure Blob). Serving assets from the application filesystem does not
  scale.
- A CDN vendor is at least shortlisted (CloudFront, Cloudflare, Fastly,
  Akamai). The skill stays vendor-neutral but requires the choice to be
  named.
- Someone can answer "can any of this content leak to the wrong user?"
  honestly. Private content dramatically changes the auth flow.

## Execution Workflow

1. Classify the content. Public and immutable (hashed bundle filenames),
   public and mutable (user profile photo URL that doesn't change), or
   private (paid-course video, signed medical record). Each category
   drives different Cache-Control, invalidation, and auth choices.
2. Put the canonical copy in an object store, never on the app server's
   disk. Application servers become stateless; scaling the origin means
   scaling the store, not the app tier.
3. Put the CDN in front of the bucket. For public content the bucket can
   be read-only-via-CDN with an origin-access identity blocking direct
   bucket reads. For private content the origin must check auth (signed
   URLs, CDN-level token validation, or an auth service at the edge).
4. Name the invalidation strategy. TTL-only is the simplest — set
   Cache-Control with a max-age that matches content freshness tolerance
   and accept staleness up to that window. Purge APIs give instant
   invalidation but cost money and rate-limit. Cache-busting URLs
   (content-hash in the filename) sidestep invalidation entirely for
   immutable content.
5. Wire a cache-busting convention for mutable assets. A profile photo
   at `/users/123/avatar.jpg` cannot be updated without a purge; the
   same photo at `/users/123/avatar.<hash>.jpg` updates by pointing the
   HTML to a new URL. Cache-busting is cheaper and more reliable than
   purges at scale.
6. Handle private content with signed URLs. Generate time-limited URLs
   server-side, carrying the user identity and an expiry. The CDN
   validates the signature at the edge without contacting the
   application for each asset. Expiries should be short (minutes to a
   few hours) and tied to session lifecycle.
7. Define the origin-fallback behaviour. What does the CDN do on origin
   5xx? Most CDNs can serve stale on error, which is usually the right
   choice for public content and the wrong choice for private. Configure
   explicitly.
8. Measure cost and hit-rate together. A 95% hit-rate at 10TB of egress
   is cheap; an 80% hit-rate at the same volume doubles origin egress
   and the bill. Optimise cache-key design (query string normalisation,
   vary headers) before negotiating vendor price.

## Rules: Do

- Treat the object store as the system of record for assets. Application
  servers are stateless; uploads go straight to the store (or through a
  presigned-URL flow), never to local disk.
- Use content-hash cache-busting for immutable assets. It makes
  deploys safe and eliminates most purge traffic.
- Sign URLs server-side for private content with short expiries. Never
  stream private bytes through the application tier unless the CDN
  cannot enforce auth at the edge.
- Set explicit Cache-Control on every asset. Default vendor behaviours
  vary and silent caching bugs are hard to reproduce.
- Monitor cache hit-rate and origin egress as first-class metrics. They
  are the two numbers that predict both latency and cost.

## Rules: Don't

- Don't serve assets from application disks in production. First
  autoscale event exposes the divergence.
- Don't use purge APIs as a primary invalidation strategy for mutable
  content with many URLs. Purge-per-URL rate limits and costs add up;
  switch to cache-busting.
- Don't rely on the CDN's default Cache-Control. Different vendors
  default differently, and the defaults change.
- Don't cache private content on a shared CDN cache key. A user
  identifier must be in the cache key (or the URL must be signed and
  unique) or cross-user leaks are one misconfiguration away.
- Don't serve all origins through a single root path. Multiple content
  types with different TTLs share cache behaviour and one misconfig
  breaks all.

## Expected Behavior

After applying the skill, the architecture diagram shows the object store
as the origin of record, the CDN in front with explicit Cache-Control
policy per path, and a signing service for private content. Invalidation
strategy is named (TTL, purge, cache-bust) per content category. Cache
hit-rate and origin egress are on a dashboard. Asset-related outages move
from "we don't know where it's cached" to "cache hit-rate dipped at 14:02
coinciding with a deploy — here's the cache-key change that caused it."

## Quality Gates

- Architecture diagram shows object-store → CDN → user flow with
  origin-access restricted.
- Cache-Control policy defined per content category (immutable, mutable,
  private) and applied via response headers or bucket metadata.
- Invalidation strategy named per category and documented in the design.
- Private content served via signed URLs with documented expiry; no
  through-application streaming except by explicit exception.
- Cache hit-rate and origin-egress dashboard panels exist.
- Cost estimate run against expected traffic before go-live, not after.

## Companion Integration

Pairs with `sysdesign-rate-limiting-strategies` (rate limits on upload /
signing endpoints), `sysdesign-monitoring-4-golden-signals` (CDN-level
latency and error panels per surface), and
`sysdesign-idempotency-patterns` (upload endpoints are action-like and
need idempotency). No direct UX companion, though cache-bust URL shape
decisions have downstream effects on `matilha-ux-pack` skills that
render media performance.

## Output Artifacts

- Architecture diagram section showing origin → CDN → client with auth
  boundaries.
- A `cache-policy.md` (or design-doc section) listing, per content
  category: Cache-Control header, TTL, invalidation strategy, auth
  mechanism.
- OpenAPI or endpoint docs for the signed-URL generator, including
  expiry and scope.
- Dashboard panel links for hit-rate and egress.
- Cost model spreadsheet or calculation in the design doc.

## Example Constraint Language

- Use "must" for: canonical assets in object store (not app disk),
  explicit Cache-Control on every asset path, signed URLs for private
  content with short expiries.
- Use "should" for: content-hash cache-busting for immutable assets,
  origin-access restrictions so the bucket isn't publicly readable
  directly, stale-on-error for public content.
- Use "may" for: purge APIs as an emergency invalidation lever,
  cross-region replication on the origin bucket for disaster recovery,
  streaming private bytes through the application when CDN signing
  isn't feasible.

## Troubleshooting

- **"Updated a profile photo, users still see the old one"**: relying on
  URL-stable updates with a TTL longer than user expectation. Switch to
  content-hash filenames and swap the HTML reference on update.
- **"Private video leaked to a user who wasn't authorised"**: the CDN
  cache key did not include user identity and signed URLs weren't used.
  Either sign every private URL or add the user ID to the vary key.
- **"Origin egress cost doubled after deploy"**: cache hit-rate dropped.
  Usually a cache-key change (new query param, new Vary header). Diff
  CDN config against pre-deploy.
- **"Purge API is rate limiting us during a content migration"**: batch
  purges are the wrong tool for large migrations. Use new URLs
  (cache-busting) and let old ones expire naturally.
- **"CDN serves stale private content to the wrong user during rotation"**:
  signed URL expiry was longer than session; tighten expiry to session
  duration and require re-signing on every request.

## Concrete Example

A course platform serves paid video through a signed-URL + CloudFront +
S3 stack. Content-hash naming on immutable lesson videos means deploys
never need purges. Signed URLs expire in 4 hours and are scoped to the
student's enrollment ID, so sharing a URL across users fails at the
edge. Cache hit-rate holds at 96% on a dashboard panel; origin egress
spikes line up with the content team's new-lesson publish cadence.
During one incident, a misconfigured Vary header drops hit-rate to 71%
and egress cost triples in three days — the dashboard flags it, rollback
is one config change.

## Sources

- `[[concepts/design-cases]]` — Design CDN (Chapter 13) and Design
  Flickr (Chapter 12) case studies
- `[[concepts/nfr-system-design]]` — latency techniques (GeoDNS, CDN,
  caching)
- Zhiyong Tan, *Acing the System Design Interview*, Chapters 12 and 13.
  Cache-busting via content hash is standard industry practice
  summarised through Danilo's wiki paraphrase rather than a direct
  quote.
