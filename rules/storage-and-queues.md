# Storage and queues — and the egress that surprises people

Sources: [`/docs/queues`](https://vercel.com/docs/queues) ·
[queues concepts](https://vercel.com/docs/queues/concepts) ·
[poll mode](https://vercel.com/docs/queues/poll-mode) ·
[queues limits](https://vercel.com/docs/queues/pricing) ·
[`/docs/vercel-blob`](https://vercel.com/docs/vercel-blob) ·
[`@vercel/blob`](https://vercel.com/docs/vercel-blob/using-blob-sdk) ·
[marketplace storage](https://vercel.com/docs/marketplace-storage) ·
[redis](https://vercel.com/docs/redis) · [postgres](https://vercel.com/docs/postgres) ·
[`waitUntil`](https://vercel.com/docs/functions/functions-api-reference/vercel-functions-package)
Read `2026-08-06`.

## Pick by durability requirement, not by convenience

| Need | Use |
|---|---|
| Must survive the response, must not be lost | **Queues** |
| Should happen after the response, losable | **`after()`** (Next 15.1+) or **`waitUntil`** (other frameworks) |
| Large files, public or signed URLs | **Blob** |
| Relational, transactional | **Postgres** (marketplace) |
| Ephemeral, low-latency | **Redis** (marketplace) |

### On Next.js 15.1+, use `after()` — not `waitUntil`

**Corrected 2026-08-06 after review.** This file previously recommended `waitUntil` as *the*
post-response primitive. The docs reverse that on modern Next.js:

> "If you're using **Next.js 15.1 or above**, we recommend using the built-in
> [`after()`](https://nextjs.org/docs/app/api-reference/functions/after) function from
> `next/server` **instead** of `waitUntil()`."
> — [`/docs/functions/functions-api-reference/vercel-functions-package`](https://vercel.com/docs/functions/functions-api-reference/vercel-functions-package)

`waitUntil` from `@vercel/functions` remains correct for **other frameworks, or Next.js below
15.1.** On a Next 16 App Router codebase, reach for `after()`.

**Neither is a queue.** Both continue work after the response within the invocation's lifetime.
If the instance goes away the work is gone — no retry, no dead-letter, no visibility. Correct for
analytics and logging; wrong for "send the receipt".

That distinction is the most common design error in this area, because `waitUntil` is one
import and a queue is infrastructure. The test: **if losing this silently would be a bug
report, it needs a queue.**

## Queues

- **Make consumers idempotent.** At-least-once delivery is the norm; a consumer that charges a
  card must tolerate seeing the same message twice. Key the effect on the message id.
- **Read the limits** ([pricing and limits](https://vercel.com/docs/queues/pricing))
  before designing throughput — message size and rate ceilings shape the design, and
  discovering them at load is expensive.
- **Poll mode exists** for consumers that cannot be pushed to; know which mode you are in,
  because the failure modes differ.
- **Plan the failure path.** Where does a message go after N failures, and who looks at it?
  A queue without a dead-letter story loses data quietly, which is the thing you chose a
  queue to avoid.

## Blob

- **Never proxy large file bytes through a function** when the SDK can issue a client upload.
  The documented thresholds make this concrete: **server uploads are capped at 4.5 MB** (the
  function request/response body limit), while **client uploads reach up to 5 TB**. Proxying pays
  function duration and bandwidth for something the storage layer does natively — and above
  4.5 MB it simply fails.
- **Signed/uploaded URLs are credentials.** Short expiry, and never in a URL you log.
- **Public means public.** A "private" blob served from a guessable public URL is public.

## Egress is the line item nobody predicted

Cross-region traffic and bytes out of the platform cost money, and the pattern that generates
it is invisible in code: **a function in one region querying a database in another, in a loop.**
`vercel-optimize` has support topics for both database egress/pooling/region and Fast Data
Transfer payload size, and they are worth reading before an unexplained bill.

Three practices that remove most of it:

- **Co-locate function and database region.** See [`functions.md`](functions.md).
- **Use a pooled connection.** Serverless plus per-invocation connections exhausts a database's
  connection limit under load, and the symptom is errors under traffic rather than slowness.
- **Return what the client needs.** A list endpoint serialising whole rows to render three
  fields pays for the rest on every request.
