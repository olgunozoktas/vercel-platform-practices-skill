# Deployment protection — and letting automation through without turning it off

Sources: [`/docs/deployment-protection`](https://vercel.com/docs/deployment-protection) ·
[methods](https://vercel.com/docs/deployment-protection/methods-to-protect-deployments) ·
[bypass](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection) ·
[automated & agent access](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection) ·
[exceptions](https://vercel.com/docs/deployment-protection/exceptions) ·
[OPTIONS allowlist](https://vercel.com/docs/deployment-protection/options-allowlist) ·
[retention](https://vercel.com/docs/deployment-retention)
Read `2026-08-06`.

## Why previews are protected by default

Every push creates a preview deployment with a guessable-ish URL, running your real code,
often against real-ish data. Unprotected, that is your unreleased product and its
configuration, publicly addressable. Protection is the default because the default should be
safe.

## The mistake this file exists to prevent

Automation hits a preview URL, gets `401`, and somebody turns protection off for the project.
That fixes the CI job and exposes every future preview.

**Use a documented bypass instead.** The docs provide several, and the right one depends on
who is being let in:

| Who needs in | Use |
|---|---|
| CI, E2E tests, a scripted check | **Protection bypass for automation** — a secret sent as a header/query param, scoped to the project |
| A specific human reviewer without a team seat | **Shareable link**, or an exception for that deployment |
| A specific domain that must stay public (a webhook receiver, a status page) | **Deployment protection exception** for that domain |
| A browser preflight that must not be challenged | **OPTIONS allowlist** — see below |

## The OPTIONS allowlist, which is the non-obvious one

A cross-origin browser request sends a preflight `OPTIONS` **without credentials.** Under
protection, the preflight is challenged, the browser never sends the real request, and the
symptom is a CORS error rather than an auth error — so the diagnosis goes down the wrong path
entirely.

**If a protected deployment must serve cross-origin requests, allowlist the `OPTIONS`
preflight for those paths.** Recognising the shape (CORS error on a protected deployment that
works when protection is off) saves an afternoon.

## Practices

- **Treat the automation bypass secret as a credential.** Sensitive environment variable,
  scoped to the project, rotated on staff change. It is a key to every preview.
- **Do not put the bypass into a client bundle.** It would make protection decorative.
- **Never grant a bypass to production** to solve a preview problem. They are separate
  settings for a reason.
- **Prefer an exception on one domain to lowering protection project-wide.** Exceptions are
  auditable and narrow; a disabled setting is neither.
- **Check retention alongside protection.** Old previews stay reachable until retention
  removes them, so a protection change is retroactive over a longer window than people
  expect. [`/docs/deployment-retention`](https://vercel.com/docs/deployment-retention).

## Password protection is not authentication

It keeps casual visitors out of a preview. It is a shared secret with no identity, no audit
trail, and no revocation-per-person. Fine for "don't index this"; not a substitute for SSO on
anything with real data.
