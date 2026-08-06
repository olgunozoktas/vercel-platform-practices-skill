# Production readiness — the pre-launch pass

Source: [production checklist](https://vercel.com/docs/production-checklist) — prepared by
Vercel's engineering team, organised into five pillars. Read `2026-08-06`.

This file is a way to *work* the checklist, not a replacement for it. **Read the live doc**;
it is maintained and this is not.

> **Attribution corrected 2026-08-06.** The first version presented the bullets below as though
> they were Vercel's checklist items. Several are **not** — the Reliability list especially is
> mine, not theirs. Vercel's actual Reliability items are: Observability Plus, automatic Function
> failover, Secure Compute passive failover, caching headers, understanding caching-vs-ISR,
> Tracing, and load testing. And their Cost pillar leads with **"Enable Fluid compute"**, then ISR
> revalidation tuning, image-optimization pricing, and moving large media to Blob — none of which
> appeared here.
>
> Below, **(Vercel)** marks a real checklist item and **(added)** marks mine. That distinction is
> the point of the fix: a reader working this file was told they had walked Vercel's checklist when
> they had walked mine.

## The five pillars, and the question each one really asks

### Operational excellence — "who does what at 3am?"

- An incident response plan: escalation paths, communication channels, rollback strategy.
- Everyone who can deploy knows how to **stage, promote and roll back**.
- Monorepo build caching configured, so pushes do not trigger unnecessary builds.
- Zero-downtime DNS migration if you are moving domains.

**The test of a plan is that somebody other than its author can execute it.** Write it where
they will find it.

### Security — "what is exposed that should not be?"

**(Vercel)** — and there are **thirteen**, not the three an earlier revision listed: CSP and
security headers · Deployment Protection · WAF · **Log Drains** · **rate limiting** · **access
roles (RBAC)** · **SAML SSO** · **SCIM** · **Audit Logs** · **commit your lockfiles** · **Preview
Deployment Suffix** · **SSL certificate checks** · **a bad-bot firewall rule**. Listing three told
readers that ten genuine Vercel items were mine — the exact failure the attribution fix existed to
correct. **(added)**: the OIDC and sensitive-variable items below.

- A **Content Security Policy** and the standard security headers. Add it early in
  report-only mode; a CSP added late is a CSP that breaks a live page.
- **Deployment protection** enabled, with documented bypasses for automation rather than an
  off switch. See [`deployment-protection.md`](deployment-protection.md).
- **WAF** configured to monitor, block and challenge. Log-only first. See
  [`security-and-firewall.md`](security-and-firewall.md).
- No long-lived cloud credentials where OIDC federation reaches the provider. See
  [`oidc-federation.md`](oidc-federation.md).
- Every secret marked sensitive; nothing secret behind a public build-time prefix. See
  [`environment-and-secrets.md`](environment-and-secrets.md).

### Reliability — "what happens when a dependency is down?"

**(Vercel)** Observability Plus · automatic Function failover · Secure Compute passive failover ·
caching headers · understand caching vs ISR · **Tracing** · load testing.

**(added)** — mine, not on their list, and worth doing anyway:

- Every external call has a timeout and a defined behaviour on failure. A page that renders
  a degraded state beats a page that hangs.
- Retries are bounded and idempotent-safe.
- Rollback is actually available — which means migrations are backwards-compatible. See
  [`deployments.md`](deployments.md).
- Skew protection considered for long-lived tabs.

### Performance — "what does a real user on a real network experience?"

- Cacheable routes are actually caching. Check the cache reason, not the intent. See
  [`caching.md`](caching.md).
- Function region co-located with the primary data store.
- Core Web Vitals measured on real traffic, not a lab run.
- For component-level work, use the official **`react-best-practices`** skill.

### Cost — "which of invocations, duration, or bytes is this?"

Every cost question resolves to one of three quantities, and **the answer changes what you
do**:

**The billing dimensions under fluid compute are Active CPU, Provisioned Memory and Invocations**
— corrected 2026-08-06; an earlier revision used the pre-fluid invocations/duration/bytes model.
Active CPU *pauses while the function waits on I/O*, so "it takes 8 seconds" is not itself a cost.

| If this grew | Look at |
|---|---|
| **Invocations** | Cache hit ratio, middleware matchers, bot traffic, polling clients |
| **Active CPU** | Real compute — unindexed queries done in-process, serialisation, image work. Not I/O waiting |
| **Provisioned Memory** | A raised memory tier, and long-in-progress requests holding it |
| **Data transfer** | Payload sizes, image optimisation, cross-region egress |

**(Vercel)**, and their list leads with the one this file originally omitted: **enable Fluid
compute** · tune ISR revalidation · review image-optimization pricing · move large media to Blob.

- Set **spend management** limits and alerts *before* launch
  ([`/docs/spend-management`](https://vercel.com/docs/spend-management)). An alert you
  configure after the surprise is not an alert.
- Build minutes are a cost too — monorepo fan-out is the usual cause.

## How to run this pass

1. Read the live checklist. It changes.
2. Walk the five pillars and write down, per item, **the evidence** — not "yes" but "here is
   the rule / the setting / the graph".
3. For every item you skip, record why. A skipped item with a reason is a decision; a skipped
   item without one is a gap that will be rediscovered during an incident.
4. Re-run it after any architectural change, not just before the first launch.

**"We reviewed it" is not an outcome. "Here are the items, N with evidence and M deliberately
deferred, with reasons" is.** (An earlier revision said "the fourteen items" — an invented number,
in the peroration of a file whose closing rule is *state the limit you relied on*. Vercel's
checklist runs to roughly forty.)
