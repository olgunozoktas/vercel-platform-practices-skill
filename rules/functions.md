# Functions — duration, memory, regions, concurrency

Sources: [`/docs/functions/configuring-functions`](https://vercel.com/docs/functions/configuring-functions) ·
[`/duration`](https://vercel.com/docs/functions/configuring-functions/duration) ·
[`/memory`](https://vercel.com/docs/functions/configuring-functions/memory) ·
[`/region`](https://vercel.com/docs/functions/configuring-functions/region) ·
[`/concurrency`](https://vercel.com/docs/functions/configuring-functions/concurrency) ·
[`/limitations`](https://vercel.com/docs/functions/limitations) ·
[`/fluid-compute`](https://vercel.com/docs/fluid-compute) ·
[`/functions/usage-and-pricing`](https://vercel.com/docs/functions/usage-and-pricing)
Read `2026-08-06`.

## Maximum duration — the numbers, and why raising it is rarely the fix

Read from the duration doc's own table:

| Plan | Default | Maximum | Extended maximum |
|---|---|---|---|
| Hobby | 300s (5 min) | 300s (5 min) | — |
| Pro | 300s (5 min) | 800s | 1800s (30 min) |
| Enterprise | 300s (5 min) | 800s | 1800s (30 min) |

Three things about this table are load-bearing:

- **The default is already 300s on every plan.** A function timing out at 10s is not
  hitting a Vercel limit — it is hitting a framework or gateway limit, or it is being
  killed by something else. Find out which before touching `maxDuration`.
- **800s is generally available on Pro/Enterprise; 1800s is a beta** that must be
  configured per function in code or `vercel.json` — project-level defaults do not reach
  it. Verify the beta's current scope before designing around 30 minutes.
- **Secure Compute and Static IPs do not support durations above 800s** during that beta.
  If the project uses either, 800s is the real ceiling regardless of plan.

**Where to set it depends on the runtime**, and mixing the two is how a setting appears not
to apply:

```ts
// Next.js App Router (and Pages Router / Node /api routes via a config object)
export const maxDuration = 5;
```

```json
// vercel.json — for runtimes and frameworks without a code-level export.
// Note the /src prefix requirement when functions live under src/.
{ "functions": { "api/heavy.ts": { "maxDuration": 30 } } }
```

**Raising the duration is not a performance fix.** It gives a slow invocation more time to
stay expensive. Under fluid compute a longer ceiling also changes how instances are reused,
so the ceiling has cost consequences beyond the one slow request. Reach for it when you
have measured a legitimately long job — a large export, a batch — and otherwise fix the
work.

**A long job usually wants a different shape.** If the work does not have to finish before
the response, `waitUntil` from `@vercel/functions` continues it after the response is sent;
if it is genuinely long-running, a queue is the right tool. See
[`storage-and-queues.md`](storage-and-queues.md).

## Fluid compute is the default, and it changes the economics

Per the fluid-compute doc: it is enabled by default with per-plan preset defaults, and its
distinguishing feature is **optimized concurrency — multiple invocations handled inside one
function instance** (Node.js and Python runtimes). It scales by using existing resources
before adding instances.

Two consequences worth designing for:

- **Module scope is shared between concurrent invocations in the same instance.** A
  module-level mutable cache, a request-scoped variable hoisted to module scope, or a
  singleton holding per-user state will leak across requests. This is the failure mode that
  did not exist under strict one-request-per-instance serverless, and it does not announce
  itself — it looks like a rare, unreproducible data mix-up.
- **Cold starts are reduced, not eliminated.** Keep heavy imports out of the hot path;
  initialisation cost is paid per instance, and it is amortised across more requests than
  before but still real.

## Memory and CPU

Memory and CPU are configured together — raising memory raises CPU proportionally, so
"raise memory" is also "pay more per unit time". Verify the current per-plan options in
[`/memory`](https://vercel.com/docs/functions/configuring-functions/memory) rather than
assuming; the tiers change.

**Measure before raising.** A function that is slow because of an unindexed query, a
serial waterfall, or a cold external API gets no faster with more memory, and now costs
more per invocation while still being slow.

## Regions — the setting most likely to be silently wrong

A function runs in the region you configure; its **database usually does not**. A function
in `iad1` talking to a database in `fra1` pays the round trip on every query, and a request
that issues five sequential queries pays it five times.

- **Co-locate the function with its primary data store.** This is the single highest-value
  region decision and it is invisible in code review.
- **ISR is stored in ONE function region** selected from your configured list (per the
  caching doc). Multi-region configuration does not multiply the ISR cache.
- Changing regions changes latency for everyone, not just the slow route. Treat it as a
  deployment-wide decision.

## Concurrency

In-function concurrency is configurable
([`/concurrency`](https://vercel.com/docs/functions/configuring-functions/concurrency)) and
interacts with memory: more concurrent invocations share one instance's resources. Raising
concurrency on a memory-hungry handler trades cost for the risk of resource contention
inside the instance.

Leave the default unless you have measured a specific bottleneck. The default is chosen to
balance cost and performance, and both directions have a failure mode — too low wastes
instances, too high starves them.

## What to check when a function is slow

In this order, because the cheapest checks rule out the most:

1. **Is it the region?** Compare function region against database/API region.
2. **Is it a waterfall?** Sequential awaits that could be `Promise.all`.
3. **Is it cold start?** Look at the spread, not the mean — cold starts are a tail.
4. **Is it the external dependency?** A slow upstream is not fixed by any Vercel setting.
5. **Only then**: memory, then duration.

[`/docs/functions/debug-slow-functions`](https://vercel.com/docs/functions/debug-slow-functions)
is the platform's own guide, and `vercel-optimize` automates the metrics half of this when
its scripts can run.
