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

A framework's public prefix (`NEXT_PUBLIC_`, `VITE_`, …) **inlines the value into the client
bundle at build time.** Consequences people discover late:

- It is readable by anyone who loads the page. Not "hard to find" — readable.
- It is baked into every build artifact, so removing the variable later does not remove it
  from deployments that already shipped.
- **A secret that was ever public-prefixed must be rotated, not un-prefixed.** Treat the
  discovery as a leak, because it is one.

**Default to no prefix.** Add one only for a value you would print on the page.

## Sensitive variables exist — use them for secrets

Vercel supports marking a variable **sensitive**, which makes it write-only: it can be set
and used but not read back through the dashboard or API. That is the right setting for every
credential, and it costs nothing.

The trade is real and worth stating: **you cannot recover a sensitive value.** If it is the
only copy, losing it means rotating at the provider. Store the source of truth in your
password manager, not in Vercel's UI.

## Per-environment values, and the trap in preview

Variables scope to Production / Preview / Development. Two rules:

- **Never point Preview at production data.** Preview deployments are created by every push
  and are, by default, more widely reachable than production. A preview with production
  credentials is a production breach one shared link away.
- **Branch-scoped preview values** are the mechanism when one preview genuinely needs
  different config. Prefer that to loosening production.

## Reserved names

Vercel reserves a set of names (`VERCEL_*` and others — see the reserved-variables doc) and
will reject or override collisions. **Read that list before naming anything `VERCEL_`-ish.**
The failure is confusing: your value silently is not what you set.

The system-provided ones are useful rather than merely reserved — deployment URL, git commit
SHA, environment name. Use them instead of re-deriving the same facts from your own
variables, which is a second source of truth that drifts.

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
