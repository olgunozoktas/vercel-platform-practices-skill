# Deployments — promote, roll back, and survive a version skew

Sources: [managing deployments](https://vercel.com/docs/deployments/managing-deployments) ·
[instant rollback](https://vercel.com/docs/instant-rollback) ·
[skew protection](https://vercel.com/docs/skew-protection) ·
[deployment retention](https://vercel.com/docs/deployment-retention) ·
[deployment checks](https://vercel.com/docs/deployment-checks) ·
[deploy hooks](https://vercel.com/docs/deploy-hooks) ·
[incremental migration](https://vercel.com/docs/incremental-migration) ·
[custom error pages](https://vercel.com/docs/custom-error-pages) ·
[draft mode](https://vercel.com/docs/draft-mode)
Read `2026-08-06`.

## Know the rollback path before you need it

The production checklist puts this first under operational excellence, and it is the right
place: **instant rollback**, and familiarity with stage/promote/rollback, before launch.

- **Practise it once on purpose.** A rollback you have never performed is a procedure, not a
  capability. Do it in a quiet hour and time it.
- **Rollback does not undo a migration.** This is the gap that turns a 30-second rollback into
  a two-hour incident: the previous build expects the previous schema. Forward-only,
  backwards-compatible migrations are what make rollback actually available — deploy the
  schema change first, in a form the old code tolerates, then the code.
- **Rollback does not undo a cache.** A wrong page cached with a long TTL survives the
  rollback. Purge the affected paths. See [`caching.md`](caching.md).

## Skew protection — the failure that looks like a random bug

A user loads your app, you deploy, and their still-open tab requests an asset or calls a server
action that no longer exists on the new deployment. Symptoms: a chunk-load error, a failed
action, or a subtly wrong response — reported by one user, unreproducible, and gone by the
time you look.

**Skew protection pins a client's requests to the deployment it was served from.** Two corrections
to what this file first said:

- **It is probably already on.** *"Projects created after **November 19th 2024** using one of the
  supported frameworks **already have Skew Protection enabled by default**."* "Enable it if…" was
  the wrong instruction for most current projects — check, don't enable.
- **It does not remove the whole class.** It pins *framework-managed* requests only: *"The
  framework **doesn't** automatically pin custom `fetch()` calls you make from client components"*
  and *"**doesn't pin full-page navigations by default**."* So the residue is exactly the custom
  `fetch` and hard-refresh paths a dashboard or editor leans on — which is the app type this
  section is aimed at. Pin those deliberately.

## Deployment checks and hooks

- **Checks** gate a deployment on your own criteria before promotion. Use them for the things
  a human always forgets — a smoke test against the preview URL, a bundle-size ceiling.
- **Hooks** trigger deploys from outside git (a CMS publish). Treat the hook URL as a
  credential: anyone with it can spend your build minutes.

## Retention

Old deployments stay reachable until retention removes them. Two consequences:

- **A protection change IS retroactive — corrected 2026-08-06.** This file previously said the
  opposite, and contradicted `deployment-protection.md` while cross-referencing it. The docs are
  explicit: *"Disabling Vercel Authentication renders **all existing deployments** unprotected"*,
  and the same for Password Protection and Trusted IPs. Protection is evaluated per request from
  current project settings, so it spans the whole retention window.
  The *real* non-retroactivity is cookie-shaped: *"When you re-enable it, previously authenticated
  users can maintain access without a new login, provided they already authenticated to the
  specific deployment and have a cookie set in their browser."*
- **A leaked secret in an old build is still in that old build.** Rotate; do not rely on the
  deployment aging out.

## Custom error pages

A default platform error page during an incident tells your users nothing and looks like your
whole company is down. A custom one can say what is happening and where the status page is.
**Write it before the incident** — it is fifteen minutes of work that only pays off on the
worst day.

## Incremental migration

If you are moving an existing site onto Vercel, there is a documented incremental path — route
by route, rather than a cutover. **Prefer it.** A big-bang DNS switch has one rollback and it
is the whole migration.
