# Environment variables and secrets

Sources: [`/docs/environment-variables`](https://vercel.com/docs/environment-variables) ·
[`/managing-environment-variables`](https://vercel.com/docs/environment-variables/managing-environment-variables) ·
[`/sensitive-environment-variables`](https://vercel.com/docs/environment-variables/sensitive-environment-variables) ·
[`/reserved-environment-variables`](https://vercel.com/docs/environment-variables/reserved-environment-variables) ·
[`/framework-environment-variables`](https://vercel.com/docs/environment-variables/framework-environment-variables) ·
[rotating](https://vercel.com/docs/environment-variables/managing-environment-variables) ·
[`/docs/accounts/access-tokens`](https://vercel.com/docs/accounts/access-tokens)
Read `2026-08-06`.

## The prefix decides who can read it — and it is irreversible once deployed

A framework's public prefix (`NEXT_PUBLIC_`, `VITE_`, …) exposes the value to the browser at
build time. (Vercel's docs say only that frameworks "use a prefix in order to expose environment
variables to the browser" and that values are available during the build step; the inlining
mechanics and the conclusions below are reasoning from that, not Vercel's text.) Consequences
people discover late:

- It is readable by anyone who loads the page. Not "hard to find" — readable.
- It is baked into every build artifact, so removing the variable later does not remove it
  from deployments that already shipped.
- **A secret that was ever public-prefixed must be rotated, not un-prefixed.** Treat the
  discovery as a leak, because it is one.

**Default to no prefix.** Add one only for a value you would print on the page.

## Sensitive variables — use them for secrets, with two operational caveats

Marking a variable **sensitive** makes it write-only: settable and usable, *"non-readable once
created"*, stored *"in an unreadable format"*. That is the right setting for every credential.

**Two constraints the first version of this file omitted, and both bite immediately:**

- **Production and Preview only.** Development targets stay merely `encrypted` — the API does not
  allow sensitive variables in development. So "mark every secret sensitive" is not achievable
  across all three environments.
- **To make an EXISTING variable sensitive you must remove and re-add it.** There is no toggle.
  Plan a brief window, or do it as part of a rotation.

The trade is real: **you cannot recover a sensitive value.** If it is the only copy, losing it
means rotating at the provider. Keep the source of truth in your password manager, not Vercel's UI.

## Per-environment values, and the trap in preview

Variables scope to Production / Preview / Development. Two rules:

- **Never point Preview at production data.** Preview deployments are created by every push
  and are, by default, more widely reachable than production. A preview with production
  credentials is a production breach one shared link away.
- **Branch-scoped preview values** are the mechanism when one preview genuinely needs
  different config. Prefer that to loosening production.

## Reserved names — and `VERCEL_*` is NOT the risk

**Corrected 2026-08-06.** This section previously said Vercel reserves `VERCEL_*` names. It does
not. The actual reserved list is eleven names — seven `AWS_*` plus four more:

`AWS_SECRET_KEY` · `AWS_EXECUTION_ENV` · `AWS_LAMBDA_LOG_GROUP_NAME` ·
`AWS_LAMBDA_LOG_STREAM_NAME` · `AWS_LAMBDA_FUNCTION_NAME` ·
`AWS_LAMBDA_FUNCTION_MEMORY_SIZE` · `AWS_LAMBDA_FUNCTION_VERSION` · `NOW_REGION` ·
**`TZ`** · `LAMBDA_TASK_ROOT` · `LAMBDA_RUNTIME_DIR`

**Rotation has its own doc**, and this file previously pointed at the wrong one:
[`/docs/environment-variables/rotating-secrets`](https://vercel.com/docs/environment-variables/rotating-secrets)
carries the operative rule — update Vercel *before* invalidating the old credential.

**`TZ` is the trap worth knowing** — it is a name an application might plausibly want to set,
and it is reserved. The earlier warning pointed at the wrong prefix and missed it.

`VERCEL_*` is a different thing: **system-PROVIDED** variables (deployment URL, git commit SHA,
environment name), not reserved-and-rejected ones. Use them instead of re-deriving the same facts
into your own variables, which is a second source of truth that drifts.

## Rotation

Rotation is a documented operation, not an emergency-only one. Two practices:

- **Support two valid credentials during a rotation window.** Read the new one, accept the
  old one, then remove the old one. A rotation that requires a synchronised flip requires
  downtime, which is why rotations get postponed, which is why credentials get old.
- **A rotation is not complete until the old value is invalid at the provider.** Removing it
  from Vercel only stops your app from using it.

## The credential you should not be storing at all

If the secret authenticates you to AWS, GCP, Azure, or your own API, **OIDC federation
replaces it with a short-lived token minted per deployment** — no long-lived key in any
environment. See [`oidc-federation.md`](oidc-federation.md). This is the single highest-value
change available in this area, and the reason to read that file before adding another
`*_SECRET_ACCESS_KEY`.

## Vercel access tokens (for CLI/API, not app runtime)

Per the access-tokens doc: a token's value is shown **once**, at creation, and cannot be
retrieved again. Scopes are **Full Account**, **Team**, or **Project** — and a project-scoped
token denies requests to any other project, to user-level resources, and to team-level
resources.

**Scope to the narrowest level that works, which is almost always Project.** CI that deploys
one project does not need account-wide authority, and the blast radius of a leaked token is
exactly its scope.
