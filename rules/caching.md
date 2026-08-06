# Caching — four layers, and the rule that prevents the expensive bug

Sources: [`/docs/caching`](https://vercel.com/docs/caching) ·
[`/cache-control-headers`](https://vercel.com/docs/caching/cache-control-headers) ·
[`/cdn-cache`](https://vercel.com/docs/caching/cdn-cache) ·
[`/cdn-cache/purge`](https://vercel.com/docs/caching/cdn-cache/purge) ·
[`/cache-status`](https://vercel.com/docs/caching/cache-status) ·
[`/incremental-static-regeneration`](https://vercel.com/docs/incremental-static-regeneration) ·
[`/docs/cdn`](https://vercel.com/docs/cdn)
Read `2026-08-06`.

## The rule that matters more than any setting

> **A cached value must be a function of its cache key. Not the clock, not the request, not
> the reader.**

This is the most expensive class of caching bug because **nothing errors.** A stale but
plausible response is served for the whole expiry window, to everybody, and the only symptom
is that somebody eventually notices the number is wrong.

Two real shapes, both of which pass code review:

- **Reader-dependent data inside a shared entry.** A price formatted with the first
  visitor's locale, a greeting with their name, a list filtered by their permissions. The
  offending line is a normal formatter call; what makes it wrong is the cache directive
  several lines above.
- **Clock-dependent data inside a shared entry.** `new Date().getFullYear()` in a footer, a
  "posted 3 hours ago", a token expiry computed at cache-write time. Correct when written,
  wrong for the rest of the window.

**The fix is always the same shape: resolve the varying input OUTSIDE the cached scope and
apply it after.** Cache the expensive, invariant part; compute the cheap, varying part per
request. Do not "fix" it by adding the varying value to the cache key unless you have
bounded its cardinality — a key derived from `Accept-Language` or a user id is
attacker-controllable or unbounded, and turns your cache into a memory amplifier.

**Guard it mechanically.** A source-level test that forbids clock, randomness, request, and
session reads inside cached scopes costs an hour and catches the whole class. Review does
not catch it — twice in two days, in the same codebase, is the observed rate.

## The four layers, and which one you are actually in

From the caching overview, the CDN checks layers in order and returns on the first hit:

| Layer | Scope | You control it with |
|---|---|---|
| **CDN cache** | Across Vercel regions worldwide | `Cache-Control` headers, or your framework's caching |
| **ISR cache** | Pre-rendered pages, in **ONE** function region from your configured list | Framework revalidation |
| **Runtime cache** | **Regional** and ephemeral — shared *across* functions, routing middleware and build execution **within a region** | TTLs and tags |
| **Image cache** | Optimised images | Image config / cache headers |

> **Corrected 2026-08-06.** The first version of this table listed "Data cache" as the fourth
> layer with the Runtime cache scoped "in-instance". Both were wrong, and both mattered.
> **Data cache is not a peer layer** — its doc sits *under* runtime cache and states it "is for
> Next.js 14 and below", so naming it a platform layer routes a Next 15/16 developer at a
> superseded construct. And **Runtime cache is regional, not in-instance** — it is shared across
> invocations in a region, which is the opposite safety property from the one "in-instance"
> implies, and precisely the sharing this file's top rule warns about. The **Image cache** is the
> real fourth layer and was missing.

**"Which layer am I in" is the first question in any cache bug**, because the fix differs:
purging the CDN does nothing to a stale ISR page, and `Cache-Control` does not govern the
runtime cache.

**ISR lives in one region.** Configuring five function regions does not give you five ISR
caches — it gives you one, and reads from the other four cross regions. Worth knowing before
attributing latency to the function.

## `Cache-Control` — the default is "do not cache"

Per the headers doc, the default is:

```
cache-control: public, max-age=0, must-revalidate
```

which tells **both** the CDN and the browser not to cache. So an uncached function response
is the *default*, not a misconfiguration — if you want CDN caching you must ask for it.

Practices:

- **Separate browser TTL from CDN TTL deliberately.** `max-age` governs the browser (which
  you cannot purge) and the CDN directives govern Vercel (which you can). A long `max-age`
  on a page you might need to correct is a decision you cannot take back for the length of
  that TTL. Prefer a short/zero `max-age` with a longer CDN TTL, so purging actually works.
- **`stale-while-revalidate` is how you get a fast response and a fresh one**, at the cost
  of serving one stale response per revalidation. That is usually the right trade for
  content and the wrong one for anything a user just edited.
- **`private` / `no-store` for anything reader-specific**, and prefer *not putting it in a
  shared cache at all* over trying to scope it with headers.
- Check the recommended-settings table in the doc for the current content-type → header
  mapping rather than inventing values.

## Diagnosing a MISS that should be a HIT

Two different docs answer two different halves, and conflating them wastes time (the first
version of this section did):

- **Why THIS response was a MISS/BYPASS** →
  [`/caching/cache-status`](https://vercel.com/docs/caching/cache-status). Its MISS reasons are
  **Cold**, **Request collapsed**, **Error**; its BYPASS reasons are **Draft Mode**, **Prerender
  Bypass**, **Crawler**.
- **Whether the response was cacheable at all** → the "Cacheable response criteria" in
  [`/caching/cdn-cache`](https://vercel.com/docs/caching/cdn-cache). This is the list you
  usually want:

  - The response set a `Set-Cookie`.
  - **The request carried an `Authorization` header** — which is why an authenticated route is
    not shared, and the thing to check first on any route behind auth.
  - The method was not `GET`/`HEAD`.
  - `Cache-Control` said not to (including by default — see above).
  - A `Range` header was present, or `Vary: *`.
  - The response exceeded the documented size ceiling.

**Read the reason before theorising** — the causes are mundane and each has a different fix.

`vercel-optimize` has a dedicated support topic for CDN-cache-vs-auth safety, and it is
worth reading if the route is behind auth: the safe pattern is that authenticated responses
are never shared, and the unsafe one looks identical until two users see each other's data.

## Purging

Purging exists ([`/cdn-cache/purge`](https://vercel.com/docs/caching/cdn-cache/purge)) and
is the right tool for "we shipped a wrong page". It does **not** reach the browser cache
you granted with `max-age` — that is inference from how browser caching works rather than a
documented sentence, and it is the argument for keeping `max-age` short.

**There are two methods and one of them can cause an outage.** The doc is built around
**Invalidate** versus **Delete**, and warns that deleting *"can cause many concurrent requests to
the origin leading to cache stampede"* — it is explicitly **"not recommended; prefer using
Invalidate instead."* Reach for Invalidate; treat Delete as a break-glass operation.

**Do not build a deploy step that purges everything on every deploy.** It converts your
cache into a cold cache on a schedule, and the first visitor after each deploy pays for it.
Purge the paths you changed.

## Cost

Cache hits are the cheapest possible response — no function invocation, no origin round
trip. So caching is a cost lever before it is a performance lever, and a route that
*should* be cacheable but is not is usually the single biggest line on an unexpectedly large
bill. Check invocation counts against cache hit ratio before optimising code.
