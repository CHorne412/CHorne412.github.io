---
title: "How to actually read a Postgres query plan"
date: 2026-06-18
tags: [postgres, performance]
description: >-
  EXPLAIN output looks like a wall of numbers until you know which three to
  look at first. A practical order of operations for finding the slow node.
---

`EXPLAIN (ANALYZE, BUFFERS)` gives you a tree of nodes, each with a handful of
numbers, and no indication of which one is ruining your day. After a few
hundred of these, I've settled on a fixed reading order.

## Read it inside-out, not top-down

The plan prints as a tree with the final result on top, but execution starts at
the leaves. Find the deepest indented nodes first — that's where the rows come
from, and where most problems live.

## Look at three numbers, in this order

**1. `rows` estimated vs. actual.** This line is the tell:

```
Seq Scan on orders  (cost=0.00..18.50 rows=1 width=64)
                    (actual time=0.02..14.81 rows=93204 loops=1)
```

The planner expected one row and got ninety-three thousand. Every decision
above this node — join strategy, memory allocation — was made on a bad estimate.
Fix the estimate (`ANALYZE`, extended statistics, a less exotic predicate) and
the plan above it often fixes itself.

**2. `loops`.** A node showing `actual time=0.4ms` and `loops=50000` did not
take 0.4ms. Postgres reports per-loop timing. Multiply before you panic — or
before you conclude something is fast.

**3. Buffers.** `Buffers: shared read=41290` means it went to disk for 300+ MB.
`shared hit` is cache. A query that's fast on your laptop and slow in production
is very often a `hit` that became a `read`.

## The node types worth recognising

- **Seq Scan** on a large table under a selective filter → missing index, or an
  index the planner declined to use because of a type mismatch.
- **Nested Loop** with a large outer row count → usually the downstream symptom
  of problem #1 above.
- **Sort** with `Sort Method: external merge Disk: 82MB` → raise `work_mem` for
  that query, or give it an index that provides the ordering for free.
- **Bitmap Heap Scan** with a high `Rows Removed by Filter` → the index got you
  to the right pages but not the right rows; consider a composite or partial index.

## A habit worth forming

Save the plan before you change anything, and diff it after. "It's faster now"
is a claim; a plan showing a Nested Loop replaced by a Hash Join, with buffer
reads down 40×, is evidence. I paste both into the PR description — future me
has thanked present me for this more than once.
