# A baseline two branches write

[The pages](INDEX.md) - [what x3 is](../README.md)

## A baseline two branches write

**What it catches:** two branches that shrink the **same** baseline file at the
same time. Git merges the file line by line and the removals merge correctly —
but `count` is **derived**, and when both sides dropped the same number of
records the two sides wrote the *same* line. There is nothing to conflict over,
the merge succeeds, and the file arrives disagreeing with its own list. The next
run exits `2`.

```text
branch A:  dropped 1 -> "count": 3, 3 findings
branch B:  dropped 1 -> "count": 3, 3 findings
merged:                 "count": 3, 2 findings      <- nobody was asked
```

Adjacent records are the other half: there the merge *does* conflict, and
resolving it means editing a data file by hand.

### A broken baseline can still be repaired

Such a file is refused with exit `2`, and the message names both ways it
happens — a hand edit **and a merge**. It used to name only the hand; on the day
this was measured nobody had touched the file.

`-update-baseline` **repairs it** instead of refusing it again. The list is the
data and `count` is written from the list, so the repair adds no finding and
drops none — it says what it rewrote (`REPAIRED <file>: count said 605 over a
list of 602`). Without it the only ways out were editing a data file by hand, or
deleting the baseline and regenerating it, which writes **"I owe nothing"** over
the whole debt.

<!-- x3-dist version=v0.147.0 capabilities=ec7474ac3ae437e0e7b4c481e6241021d5e7b6a9a998088c3a6178cf6d8f00af template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
