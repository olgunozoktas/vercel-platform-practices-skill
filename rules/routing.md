# Routing — redirects, rewrites, and what middleware costs

Sources: [`/docs/routing`](https://vercel.com/docs/routing) ·
[redirects](https://vercel.com/docs/routing/redirects) ·
[rewrites](https://vercel.com/docs/routing/rewrites) ·
[project-level rules](https://vercel.com/docs/routing/project-routing-rules) ·
[bulk redirects](https://vercel.com/docs/routing/redirects/bulk-redirects) ·
[middleware](https://vercel.com/docs/routing-middleware) ·
[edge middleware](https://vercel.com/docs/edge-middleware)
Read `2026-08-06`.

## Redirect vs rewrite — the distinction users can see

- **Redirect** changes the URL in the browser and returns 3xx. Use when the resource *moved*
  and you want the new URL to be the canonical one people bookmark and search engines index.
- **Rewrite** keeps the URL and serves different content. Use for proxying and for URL shapes
  you want to keep stable while the implementation moves.

**Getting this backwards is an SEO decision, not a cosmetic one.** A rewrite where a redirect
belonged leaves two URLs serving the same content; a redirect where a rewrite belonged exposes
an internal path structure you now have to keep forever.

**Permanent vs temporary is also semi-permanent in practice.** A 308/301 is cached by
browsers aggressively — shipping one by mistake is very hard to take back, because you cannot
purge a browser's redirect cache. **Start temporary, promote to permanent once you are sure.**

## Middleware runs on every matched request — so match narrowly

Middleware is the most expensive routing tool because it executes per request, before caching
decisions, on paths you may not have intended.

- **Write the narrowest matcher that works.** A middleware matching `/:path*` runs on static
  assets, favicons, and health checks. `vercel-optimize` has a dedicated support topic on
  middleware/proxy cost for exactly this reason.
- **Keep it cheap and non-blocking.** A database call in middleware puts that latency in front
  of every page.
- **Do not do in middleware what a WAF rule or a routing rule can do.** Geo-blocking, IP
  allowlisting and simple redirects are cheaper as platform config. See
  [`security-and-firewall.md`](security-and-firewall.md).
- **Middleware can make responses uncacheable.** If it varies on a cookie or header, it can
  suppress the CDN cache for routes that were cacheable. Check the cache reason after adding
  middleware, not before.

## Redirects at scale

Static redirect lists in config do not scale to thousands; the docs have both bulk redirects
and a managing-at-scale guide. When you cross a few hundred, move to the mechanism designed
for it rather than growing the config file — and keep the list in one place, because a
redirect defined in two layers is a loop waiting to happen.

## Ordering

Routing has a defined evaluation order across project rules, redirects, rewrites, middleware
and filesystem. **Read the order rather than inferring it from one experiment** — most
"my rewrite doesn't work" reports are ordering, and one working example does not establish
the rule.
