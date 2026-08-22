---
title: "Idempotency keys are a data model, not a header"
date: 2026-08-04
tags: [distributed-systems, api-design]
description: >-
  Most idempotency bugs come from treating the key as a cache lookup instead of
  a row you own. Here's the version that survives retries, races, and partial failures.
---

Every payments API grows an `Idempotency-Key` header eventually. The header is
the easy part. The part that bites is what happens when two requests carrying
the same key arrive four milliseconds apart, on two different app servers,
while the first one is still mid-transaction.

## The cache-shaped version, and why it breaks

The intuitive implementation is a lookup:

```go
if resp, ok := cache.Get(key); ok {
    return resp          // already did this
}
resp := doTheWork(req)
cache.Set(key, resp)
return resp
```

There is a window between `Get` and `Set` where a second request sees nothing
and does the work again. Under normal load you may never notice. Under a retry
storm — which is exactly when idempotency matters — you will charge someone
twice.

## Make the key a row

Insert the key *before* doing the work, and let the database's uniqueness
constraint arbitrate the race:

```sql
CREATE TABLE idempotency_keys (
  key            text PRIMARY KEY,
  request_hash   bytea       NOT NULL,
  state          text        NOT NULL,   -- 'in_progress' | 'done'
  response_body  jsonb,
  created_at     timestamptz NOT NULL DEFAULT now()
);
```

The request handler becomes a small state machine:

1. `INSERT ... ON CONFLICT DO NOTHING` with `state = 'in_progress'`.
2. If the insert took, you own the key — do the work, then flip to `'done'`
   and store the response **in the same transaction as the side effect**.
3. If it didn't take, read the existing row:
   - `state = 'done'` → replay the stored response.
   - `state = 'in_progress'` → return `409 Conflict`. The client retries.

Step 2 is the whole trick. If the response is written in a different transaction
than the charge, a crash between them leaves you with money moved and no record
that says so.

## Three details that matter

**Hash the request body.** Same key, different payload is a client bug, and the
only safe answer is `422`. Storing `request_hash` lets you detect it instead of
silently replaying the wrong response.

> A key that can be reused for a different request isn't an idempotency key,
> it's a footgun with good branding.

**Expire the rows.** 24 hours is a common window. Keys are not an audit log —
that's a separate table, and it should outlive them.

**Decide what `in_progress` means on a dead server.** If the process holding a
key dies, that key is stuck until something reaps it. A `created_at` older than
your request timeout is a safe reaping condition.

| Scenario | Cache version | Row version |
|---|---|---|
| Concurrent duplicates | Both execute | One executes, one gets 409 |
| Crash mid-work | Key lost, work repeats | Key reaped, safe retry |
| Same key, new body | Wrong response replayed | 422, loudly |

## What this costs

One extra round trip to Postgres per write request, and a table that needs a
janitor. In exchange, the retry path stops being the scariest code you own.
That has been a good trade every time I've made it.
