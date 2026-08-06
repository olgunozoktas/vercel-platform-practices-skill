---
name: vercel-platform-practices
description: "Apply this skill when configuring, deploying, securing, or operating an application ON the Vercel platform — as opposed to writing React components. Triggers on vercel.json and project configuration, function runtime settings (maxDuration, memory, regions, concurrency, fluid compute), Cache-Control and CDN/ISR/Data-cache layers, environment variables and secret handling, OIDC federation to AWS/GCP/Azure, deployment protection and automation bypass, the Web Application Firewall, rate limiting and bot management, log drains and observability, Queues and Blob storage, redirects/rewrites/middleware ordering, instant rollback, skew protection, spend management, and pre-launch production readiness. Also use when diagnosing a MISS that should be a HIT, a function that times out, a preview URL that 401s automation, or a bill that grew without a traffic change."
license: MIT
metadata:
  author: olgunozoktas
  version: "1.0.0"
  docs_snapshot: "2026-08-06"
---

# Vercel Platform Practices

Practices for the **platform layer** of a Vercel application: configuration, functions,
caching, security, storage, observability and deploys. Organised as an index of rule files.
Each rule teaches what to do and why.

**Grounded, not remembered.** Every factual claim traces to `vercel.com/docs`, cited by URL
in the rule file. The numbers were read out of a full documentation snapshot
(1,149 documents, all stamped `2026-08-06`) rather than recalled. **Version-sensitive
numbers — plan limits, pricing, beta scope — drift.** Re-verify anything load-bearing
against the live doc URL before relying on it, and treat the snapshot date above as the
"as of".

## Read this first: do not use this skill for React performance

This skill deliberately does **not** cover component-level performance, and there is a
better, larger, official skill for that. Route by question:

| Question | Use |
|---|---|
| Is this component/data-fetch pattern fast? Waterfalls, bundle size, re-renders | **`react-best-practices`** (official, Vercel Engineering, ~28k words) |
| My deployed project costs too much / a route is slow — find it from metrics | **`vercel-optimize`** (official; see the caveat below) |
| Deploy this app / create a preview | **`deploy-to-vercel`** (official) |
| Authenticate the CLI with a token instead of interactive login | **`vercel-cli-with-tokens`** (official) |
| Component API design, compound components | **`composition-patterns`** (official) |
| **How should this be configured, cached, secured, or operated on the platform?** | **this skill** |

**The `vercel-optimize` caveat that decides when you can use it.** It is a genuinely good
metrics-driven audit pipeline, and it drives ~15 Node scripts (`node scripts/*.mjs`) plus
the Vercel API. On a machine where host Node is forbidden by policy — see
`~/.claude/CLAUDE.md`'s dnpm rule — **those steps cannot be run**, and its container-wrapped
equivalent does not exist because the scripts need network access and write access outside
the project mount. This skill is reference-only and has no scripts, so it works under that
policy and offline. Prefer `vercel-optimize` where its scripts CAN run; it sees real
metrics and this skill does not.

## Consistency First

Before applying any rule, check what the project already does. Vercel supports several
valid ways to express most settings — `vercel.json`, dashboard project settings, framework
exports (`export const maxDuration`), and framework config — and **the best choice is the
one already in use.** Inconsistency costs more than a suboptimal-but-uniform choice,
because the next person has to discover both.

The precedence that actually matters when they disagree: per-function config beats
project-level defaults, and code/`vercel.json` beats the dashboard for anything a
deployment carries. If a setting exists in two places, say so rather than adding a third.

## How to Apply

1. **Identify the layer.** Platform config, function runtime, cache, security, storage,
   observability, or deploy. Cross-cutting changes touch more than one rule file.
2. **Read the mapped rule files before editing.** Skip the unrelated ones.
3. **Check what the project already does** — `vercel.json`, dashboard settings, framework
   config, existing headers. Follow the established pattern unless it is a correctness or
   security defect, and name the deviation when you make one.
4. **Verify version-sensitive numbers against the live doc URL** in the rule file. Plan
   limits and beta scope change; the snapshot date in the frontmatter is when these were
   read.
5. **Prefer the platform's own mechanism** over hand-rolled equivalents — a WAF rule over
   application rate limiting, OIDC over a long-lived secret, ISR over a cron that writes
   files.
6. **Re-read the diff against every mapped rule** before finishing, and state which limit
   or default you relied on so a reviewer can check it.

## Rule Index

| Concern | Read |
| --- | --- |
| `vercel.json`, build/install commands, project settings, precedence | [`rules/project-configuration.md`](rules/project-configuration.md) |
| `maxDuration`, memory/CPU, regions, concurrency, fluid compute, cold starts | [`rules/functions.md`](rules/functions.md) |
| `Cache-Control`, CDN vs ISR vs Data cache, purging, MISS diagnosis | [`rules/caching.md`](rules/caching.md) |
| Environment variables, sensitive values, rotation, reserved names | [`rules/environment-and-secrets.md`](rules/environment-and-secrets.md) |
| OIDC federation to AWS/GCP/Azure/your own API — no long-lived keys | [`rules/oidc-federation.md`](rules/oidc-federation.md) |
| Preview protection, password/SSO, automation bypass, OPTIONS allowlist | [`rules/deployment-protection.md`](rules/deployment-protection.md) |
| WAF rules, rate limiting, attack mode, bot management, security headers | [`rules/security-and-firewall.md`](rules/security-and-firewall.md) |
| Redirects, rewrites, middleware ordering and cost | [`rules/routing.md`](rules/routing.md) |
| Log/audit/Speed-Insights drains, function logs, what is queryable | [`rules/observability.md`](rules/observability.md) |
| Queues, Blob, Redis/Postgres marketplace storage, egress | [`rules/storage-and-queues.md`](rules/storage-and-queues.md) |
| Promote, rollback, skew protection, retention, deploy hooks, checks | [`rules/deployments.md`](rules/deployments.md) |
| Pre-launch review across operations, security, reliability, performance, cost | [`rules/production-readiness.md`](rules/production-readiness.md) |

## Decision Rules

- **A cached value must be a function of its cache key.** Not the clock, not the request,
  not the reader. This is the single most expensive class of Vercel bug in practice,
  because a stale-but-plausible response is served for the full expiry window and nothing
  errors. See [`rules/caching.md`](rules/caching.md).
- **Prefer the platform mechanism to an application one** for anything at the edge: a WAF
  rule stops traffic before it costs a function invocation, and application-level rate
  limiting bills you for the request you are rejecting.
- **Never a long-lived cloud credential in an environment variable** when OIDC federation
  reaches the same provider. See [`rules/oidc-federation.md`](rules/oidc-federation.md).
- **Raise a limit only with the failure in hand.** `maxDuration` and memory are the two
  settings most often raised speculatively; both change what you pay per invocation, and
  neither fixes a query that should have been indexed.
- **Deployment protection is on by default for previews for a reason.** When automation
  needs in, use a documented bypass — not "turn protection off".
- **Cost follows invocations, duration, and bytes.** Before optimising code, check which
  of the three moved. A cheap change to the wrong one is still the wrong change.
- **State the limit you relied on.** "Pro tops out at 800s" is checkable; "it should be
  long enough" is not.
