# vercel-platform-practices

A Claude Code / [Agent Skills](https://agentskills.io/) skill for the **platform layer** of a
Vercel application: `vercel.json`, function runtime settings, caching, secrets and OIDC,
deployment protection, the firewall, routing, observability, storage and queues, deploys, and
a pre-launch readiness pass.

Install by symlinking into your skills directory:

```sh
git clone https://github.com/olgunozoktas/vercel-platform-practices-skill.git \
  ~/Herd/vercel-platform-practices-skill
ln -s ~/Herd/vercel-platform-practices-skill ~/.claude/skills/vercel-platform-practices
```

Refresh with `git pull`. There is nothing to build and nothing to run.

## Provenance

Every factual claim traces to a `vercel.com/docs` URL, cited in the rule file that uses it.
The numbers were read out of a full documentation snapshot — `vercel.com/docs/llms-full.txt`,
**1,149 documents, every one stamped `2026-08-06`** — rather than recalled from memory.

Plan limits, pricing and beta scope drift. The rule files carry the source URL precisely so a
load-bearing number can be re-checked against the live doc; treat the snapshot date as an
"as of", not a guarantee.

> **Corrections after an adversarial review, 2026-08-06.** Every hard *number* survived — the
> nine-cell duration table (corroborated in three corpus locations), the Secure-Compute 800s cap,
> single-region ISR, the default `Cache-Control` string, the three token scopes, the five checklist
> pillars. But **wherever this skill organised rather than quoted, it drifted**, and **15 of 88
> cited URLs pointed at paths that do not exist** — which disabled this skill's own step-4
> instruction to verify against the live doc. All 15 are fixed. Also corrected: the cache-layer
> taxonomy (Data cache is a Next-14 construct, not a peer layer; the Image cache was missing;
> Runtime cache is *regional*, not in-instance), in-function concurrency (a toggle, not a dial),
> the reserved-variable list (`VERCEL_*` is not reserved — `TZ` is), three drain types → five,
> the production-checklist attribution, and `waitUntil` where the docs recommend `after()` on
> Next 15.1+. Each carries an inline note in its rule file.
>
> **The systemic lesson: this skill is reliable where it quotes and drifts where it organises.**
> If you are relying on a *table* here rather than a *quote*, check the doc.

## Why this exists alongside Vercel's official skills

[`vercel-labs/agent-skills`](https://github.com/vercel-labs/agent-skills) is excellent and you
should install it. This skill is **scoped at what it does not cover**, and it routes to the
official ones where they are better — the routing table is the first section of `SKILL.md`,
because a skill that duplicates a better one is worse than no skill.

### Honest comparison

Measured on the upstream repo at `7c180d9`:

| Skill | Size | Scope | Verdict |
|---|---|---|---|
| **`react-best-practices`** (official) | **70 rules, ~14.6k words** | React/Next component performance: waterfalls, bundle size, re-renders, server-side | **Better than anything here. Use it.** This skill deliberately does not cover component performance |
| **`vercel-optimize`** (official) | 15 scripts + ~75 lib modules + ~25k words of docs, **42** support topics | Metrics-driven cost/performance audit of a *deployed* project | **Better where it can run** — it sees real metrics; this skill does not. See the caveat below |
| **`react-native-skills`** (official) | 42 files, ~19k words | React Native / Expo | Out of scope here |
| **`react-view-transitions`** (official) | 8 files, ~11k words | View Transition API | Out of scope here |
| **`composition-patterns`** (official) | 14 files, ~5.7k words | Component API design | Out of scope here |
| **`deploy-to-vercel`** (official) | 4 files, ~1.8k words | Performing a deployment, CLI flow, team selection | Better for *doing* a deploy. This skill covers what to configure, not how to run the command |
| **`vercel-cli-with-tokens`** (official) | 1 file, ~1.5k words | Token-based CLI auth | Complementary — this skill covers token *scoping* policy |
| **`web-design-guidelines`** (official) | 1 file, **~176 words** | Thin wrapper that **fetches rules from a GitHub raw URL at runtime** | Always fresh; **fails offline** and silently breaks if the URL moves |
| **`writing-guidelines`** (official) | 1 file, **~177 words** | Same runtime-fetch pattern | Same trade |
| **`vercel-platform-practices`** (this) | 13 files, ~7.7k words | Platform config, functions, caching, secrets/OIDC, protection, firewall, routing, observability, storage, deploys, readiness | The gap: **authoring-time platform guidance**, offline, no scripts |

### The three differences that actually matter

**1. `vercel-optimize` needs host Node — which some policies forbid.** It drives ~15
`node scripts/*.mjs` steps plus the Vercel API. On a machine where all Node must run inside a
hardened container (this author's `dnpm` policy), those steps cannot run, and there is no
container-wrapped equivalent because the scripts need both network access and write access
outside the project mount. **This skill is reference-only** — no scripts, no network — so it
works under that policy and on a plane. Where `vercel-optimize` *can* run, prefer it for
investigation: real metrics beat good advice.

**2. Diagnosis-time versus authoring-time.** `vercel-optimize` answers "this deployed project
costs too much, what is wrong?" — retrospective, metric-driven, ranked. This skill answers
"I am configuring this now, what should it be?" — prospective. Both are useful; neither
replaces the other. The clearest split: `vercel-optimize` finds that a route is uncached;
[`rules/caching.md`](rules/caching.md) tells you what the four cache layers are and why
`Cache-Control` defaults to not caching, so the route is cached correctly the first time.

**3. FIVE of eight security/ops items are uncovered at authoring time — not eight.** Corrected
after review, which found three of the original eight ARE covered by `vercel-optimize`: **OIDC**
(its `ai-application.md` playbook calls an explicit API key "a regression… a long-lived secret"),
**WAF log-before-block** (`bot-protection-product-guardrails.md` prefers "a staged Log to Challenge
or Deny path"), and **queue-vs-post-response durability** (`post-response-work-waituntil.md`).

**Still genuinely absent:** deployment-protection automation bypass and the `OPTIONS` preflight
trap, audit log drains, skew protection, sensitive-variable rotation, and spend limits as a
pre-launch step.

**And the official skill endorses the gap** — `vercel-optimize`'s own `references/doctrine.md`
puts security out of scope: *"Security advisories and credential rotation… **refer to a security
skill, not this one.**"*

### What this skill is *worse* at, stated plainly

- **No metrics.** It cannot tell you which of your routes is expensive. `vercel-optimize` can.
- **No component-level performance advice.** `react-best-practices` is larger and written by
  the people who build the framework.
- **A snapshot, not a live feed.** The guidelines skills' runtime-fetch approach is genuinely
  better for freshness. The trade is offline capability and a silent failure mode; pick per
  environment.
- **Smaller.** ~7.7k words against ~14.6k on a like-for-like basis. Scoped, not comprehensive.

## Recommendation

Install both. Then:

- React/Next code → `react-best-practices`
- A cost or latency investigation on a deployed project → `vercel-optimize` (if its scripts
  can run in your environment)
- Running a deployment → `deploy-to-vercel`
- **Configuring, securing or operating the platform → this skill**

## Licence

MIT. The documentation it cites is Vercel's; the rules, structure and judgements are the
author's.
