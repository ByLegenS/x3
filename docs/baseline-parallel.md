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

### A segment owns its paths, and writes its own file

```json
{ "baseline": { "dir": "baselines",
                "segments": { "billing": ["apps/billing/**"],
                              "web":     ["services/web/**"] } } }
```

| Finding under | Written to |
|---|---|
| `apps/billing/**` | `baselines/comments.billing.json` |
| `services/web/**` | `baselines/comments.web.json` |
| anything else | `baselines/comments.json` |

Two workers, each in its own region, now write two different files. Git has
nothing to merge.

`segments` is an **object**, so the parts a root configuration `include`s merge
into it key by key: each region declares its own baseline where it declares
everything else about itself. The directory stays single and is still resolved
[beside the root configuration](configuration.md#a-relative-path-is-relative-to-the-configuration)
— the split changes the file **name**, never the base.

### The split divides the files, not the set

Reading is unchanged: a finding is held if **any** of the files holds it, so
declaring a segment on a tree that already has a baseline keeps every gate
exactly as green as it was. The next `-update-baseline` re-routes each record to
the file that owns it and says so:

```text
x3 comments: 612 finding(s) moved into the segment that owns them;
             the debt did not change, only the file that holds it
```

A move is a third direction next to growth and shrink: the debt neither appears
nor dies, so the growth refusal does not block it and the drop report does not
claim it. It happens once.

| Written this way | Result |
|---|---|
| a name that is not a file name | exit `2` — the name becomes part of a path |
| a segment owning no path | exit `2` — nothing would ever be written to it |
| `segments` without `dir` | exit `2` — the split lives under the directory |
| one path in two segments | exit `2` — a finding has one home, or two runs write it |
| the same id in two files | exit `2` — each run would believe the other froze it |

The last two are the same law seen twice: **one debt, one record, one writer.**
Give it two and the split has bought nothing.

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

<!-- x3-dist version=v0.118.0 capabilities=70f1f255387ba0e0b3e37f307a3ec5c030d0ad4c3514eac7aef65a5cead99c77 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
