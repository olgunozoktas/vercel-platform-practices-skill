# OIDC federation — reach cloud providers without a stored key

Sources: [`/docs/oidc`](https://vercel.com/docs/oidc) ·
[AWS](https://vercel.com/docs/oidc/aws) · [GCP](https://vercel.com/docs/oidc/gcp) ·
[Azure](https://vercel.com/docs/oidc/azure) ·
[your own API](https://vercel.com/docs/oidc/api) ·
[reference](https://vercel.com/docs/oidc/reference)
Read `2026-08-06`.

## What it replaces, and why that is the point

Without OIDC, reaching AWS from a Vercel function means a long-lived access key sitting in an
environment variable. That key:

- is valid until somebody remembers to rotate it,
- is as strong as its weakest copy (CI logs, a laptop, a screenshot),
- and grants the same access from anywhere, so a leak is not detectable by origin.

With OIDC federation, Vercel mints a **short-lived, deployment-scoped identity token**, and
the cloud provider's trust policy exchanges it for temporary credentials. There is **no
long-lived secret to leak, rotate, or forget.**

## When to reach for it

**Whenever the counterparty can validate an OIDC token** — AWS, GCP, Azure all can, and so
can your own API (there is a dedicated doc for that case). If you are about to add a
`*_ACCESS_KEY_ID` / `*_SECRET_ACCESS_KEY` pair to environment variables, stop and check
whether federation covers it first.

## What to get right

- **Scope the provider-side trust policy narrowly.** The token carries claims identifying
  the team, project, and environment. A trust policy that accepts any Vercel token accepts
  *every* Vercel customer's token — this is the one configuration mistake that turns a
  security improvement into a hole. Constrain on issuer *and* subject.
- **Distinguish production from preview in the policy.** Preview deployments are created by
  every push. If preview tokens can assume the production role, a pushed branch reaches
  production data.
- **Check the token's audience and lifetime** against the reference doc rather than assuming
  defaults; both are part of the trust configuration.
- **For your own API**, verify the signature against Vercel's published keys and check the
  claims. Accepting an unverified token because it "looks like ours" is the same failure as
  the permissive trust policy, one layer down.

## What it does not solve

OIDC authenticates *your deployment* to a provider. It is not a session mechanism for your
users, and it does not remove the need for authorization on the provider side — a correctly
federated identity with an over-broad role is still over-broad.
