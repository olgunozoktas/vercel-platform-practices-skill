# Observability — logs, drains, and what is actually queryable later

Sources: [`/docs/drains`](https://vercel.com/docs/drains) ·
[log drains reference](https://vercel.com/docs/drains/reference/logs) ·
[audit log drains](https://vercel.com/docs/drains/reference/audit-logs) ·
[Speed Insights drains](https://vercel.com/docs/drains/reference/speed-insights) ·
[drains security](https://vercel.com/docs/drains/security) ·
[function logs](https://vercel.com/docs/functions/logs) ·
[observability](https://vercel.com/docs/observability) ·
[manage and optimize](https://vercel.com/docs/manage-and-optimize-observability) ·
[Speed Insights](https://vercel.com/docs/speed-insights) · [analytics](https://vercel.com/docs/analytics)
Read `2026-08-06`.

## Function logs are not your log store

Runtime logs are retained for a limited window, and the window is shorter than the interval at
which you will be asked "when did this start?". **If you will need it next month, drain it.**

Set up a log drain **before** the incident. Every incident post-mortem that begins "we don't
have logs from then" was preventable by a configuration change nobody had a reason to make
yet.

## FIVE drain data types, and each answers a different question

**Corrected 2026-08-06** — this table said three. The docs list five: *"Drains support the
following data types: **Logs**, **Traces**, **Speed Insights**, **Web Analytics**, **Audit
Logs**."*

| Drain | Answers |
|---|---|
| **Logs** | What did the application do? |
| **Traces** | Where did the time go, across services? |
| **Speed Insights** | What did real users experience? |
| **Web Analytics** | What did they do? |
| **Audit Logs** | What did a *human on the team* do — settings, env vars, access? |

**Traces was the omission that mattered** — Vercel's own production checklist says "Consider
adding Tracing", and a skill about observability that never mentions distributed tracing is
missing the tool for "which downstream call is slow".

**Audit log drains are the one people forget**, and the one compliance asks for. Documented
destinations include Amazon S3, Splunk, **Datadog** and Panther. "Who changed that environment
variable, and when" is unanswerable retroactively without it.

## Practices

- **Log structured, and log the identifiers you will search by** — request id, deployment id,
  organization/tenant id. A message that reads well and cannot be filtered is not
  operationally useful.
- **Never log secrets, tokens, full request bodies, or personal data.** A drain copies logs
  into a second system with its own access rules, so a leak into logs is now a leak into two
  places. Read [drains security](https://vercel.com/docs/drains/security) before
  choosing a destination.
- **Log volume is a cost.** Debug logging left on in production is a line item. Sample the
  high-volume paths rather than dropping the level everywhere and losing the errors too.
- **Distinguish "measured zero" from "not measured".** A dashboard showing `0` for something
  you never instrumented is a lie the graph tells confidently. Where a value is not
  instrumented, render it as unknown.
- **Watch the metric that pays the bill**, not only the one that looks bad: invocations,
  duration, and bytes transferred. See [`production-readiness.md`](production-readiness.md).

## Where `vercel-optimize` fits

The official `vercel-optimize` skill collects these metrics via the Vercel API and ranks
findings. Prefer it for a metrics-driven investigation **where its Node scripts can run.**
This file is the standing configuration advice; that skill is the periodic audit.
