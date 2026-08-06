# Security and the firewall — stop it before it costs an invocation

Sources: [`/docs/vercel-firewall`](https://vercel.com/docs/vercel-firewall) ·
[concepts](https://vercel.com/docs/vercel-firewall/firewall-concepts) ·
[rate limiting SDK](https://vercel.com/docs/vercel-firewall/vercel-waf/rate-limiting-sdk) ·
[attack mode](https://vercel.com/docs/vercel-firewall/attack-mode) ·
[DDoS](https://vercel.com/docs/vercel-firewall/ddos-mitigation) ·
[IP allowlist](https://vercel.com/docs/vercel-firewall/vercel-waf/trusted-ip-access) ·
[observability](https://vercel.com/docs/vercel-firewall/firewall-observability) ·
[bot management](https://vercel.com/docs/bot-management) ·
[security headers](https://vercel.com/docs/cdn-security/security-headers) ·
[CDN security](https://vercel.com/docs/cdn-security)
Read `2026-08-06`.

## The economic argument, which is also the security argument

A WAF rule rejects a request **at the edge, before it becomes a function invocation.**
Application-level rate limiting rejects it *after* you have paid to run the function that
decided to reject it. Under a scraping or credential-stuffing load, that difference is the
whole bill.

So: **push the decision as far out as it can correctly be made.** Identity-dependent limits
have to be in the app; everything expressible as path, method, IP, geography, user-agent, or
rate belongs in the firewall.

## Layers, and what each is for

| Layer | Use for |
|---|---|
| **DDoS mitigation** | Always on, nothing to configure |
| **WAF rules** | Path/method/geo/IP/header conditions — block, challenge, or log |
| **Rate limiting** | Per-identifier request ceilings at the edge |
| **Bot management** | Automated traffic that is not a straightforward attack |
| **Attack mode** | An incident switch, not a configuration |
| **IP allowlist** | Admin surfaces and internal tools |

## Practices

- **Log before you block.** Every rule should spend time in log-only mode against real
  traffic first. A rule written from a threat model blocks legitimate users you did not model
  — and you find out from a support ticket, not a graph.
- **Rate-limit the expensive routes specifically**, not the whole site uniformly. Sign-in,
  search, export, anything that writes or fans out. A uniform limit is either too loose for
  the expensive routes or too tight for the cheap ones.
- **Attack mode is for incidents.** It challenges broadly, including real users, which is the
  reason not to leave it on — **not** cost. Two facts that cut against a cost-based reading:
  it is *"available for free on all plans"* and *"requests blocked by Attack Mode do not count
  towards your usage limits."* So during an actual attack it is cheaper on, not more expensive.
  Know where the switch is before you need it.
- **Read firewall observability after every rule change.** A rule with no matches is either
  unnecessary or wrong, and both are worth knowing.
- **Allowlist admin surfaces by IP** where the team's network allows it. Cheapest strong
  control available for an internal console.

## Security headers are not the firewall's job

A Content Security Policy and the standard security headers are application configuration and
belong in your headers config —
[`/docs/headers/security-headers`](https://vercel.com/docs/cdn-security/security-headers). The
production checklist calls out CSP explicitly, and Vercel's own conformance ruleset has a
rule for missing Next.js security headers
([`NEXTJS_MISSING_SECURITY_HEADERS`](https://vercel.com/docs/conformance/rules/NEXTJS_MISSING_SECURITY_HEADERS)).

**A CSP added late is a CSP that breaks things**, because by then something depends on inline
script. Add it early in report-only mode, read the reports, then enforce.

## Conformance, if you have access to it

Vercel ships a large conformance ruleset (79 documents in the snapshot) covering security,
performance and correctness rules it can check mechanically. If your plan includes it, it is
strictly better than a hand-maintained checklist because it runs. Browse
[`/docs/conformance`](https://vercel.com/docs/conformance) and adopt the rules that match
your stack rather than re-deriving them here.
