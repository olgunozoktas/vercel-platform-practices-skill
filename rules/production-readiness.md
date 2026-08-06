# Production readiness — the pre-launch pass

Source: [production checklist](https://vercel.com/docs/production-checklist) — prepared by
Vercel's engineering team, organised into five pillars. Read `2026-08-06`.

This file is a way to *work* the checklist, not a replacement for it. **Read the live doc**;
it is maintained and this is not.

## The five pillars, and the question each one really asks

### Operational excellence — "who does what at 3am?"

- An incident response plan: escalation paths, communication channels, rollback strategy.
- Everyone who can deploy knows how to **stage, promote and roll back**.
- Monorepo build caching configured, so pushes do not trigger unnecessary builds.
- Zero-downtime DNS migration if you are moving domains.

**The test of a plan is that somebody other than its author can execute it.** Write it where
they will find it.

### Security — "what is exposed that should not be?"

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

| If this grew | Look at |
|---|---|
| **Invocations** | Cache hit ratio, middleware matchers, bot traffic, polling clients |
| **Duration** | Region mismatch, waterfalls, an unindexed query, a raised `maxDuration` |
| **Bytes** | Payload sizes, image optimisation, cross-region egress |

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

**"We reviewed it" is not an outcome. "Here are the fourteen items, twelve with evidence and
two deliberately deferred, with reasons" is.**
