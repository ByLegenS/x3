# A snapshot the cache keeps

[The pages](INDEX.md) - [what x3 is](../README.md)

## A snapshot the cache keeps

A run already records, step by step, the files it read (with their content
digest) and the directories it walked. Union those and you have a picture of
the tree as the last runs saw it. `x3 snapshot` is the question nobody could
ask before: **what has moved since then?**

```
x3 snapshot                       # every remembered path, judged against the disk
x3 snapshot -only changed         # just the ones that moved
x3 snapshot -command case         # another command's cache
x3 snapshot -cache <file> <root>  # a cache file and a tree named outright
```

It prints one line per path and a count:

```
  a.txt: changed
  sub/d.txt: new
  gone.txt: gone
x3 snapshot: 1 changed, 3 unchanged, 1 new, 1 gone across 12 walked directory(ies)
```

Four states, and the third is the one a read-list alone cannot give you:

| State | Means |
|---|---|
| `changed` | no remembered digest of this path matches what is on disk today |
| `unchanged` | one of them does — the gate still has a valid memory of it |
| `new` | the path lives in a walked directory and no record has ever named it |
| `gone` | a record names it and the disk no longer holds it |

**No second record is written.** The union was measured before it was designed:
in the pilot the gate cache already held **1 439 of the 1 439** files git
tracks and **197 of the 197** directories they live in, so the snapshot was
already on disk — it simply had no reader. Writing a parallel record would have
meant the same bytes twice and two files free to drift apart.

**It never asks git.** A tree that is not a working copy answers the same way,
and a file nobody tracks is as visible as one everybody does. That is also why
`new` works: a directory is remembered by the digest of its *name list*, so a
file no step will ever open still shows up the moment it lands.

**Each path is weighed in its own terms.** A path the gate reads whole is
judged by its content; one it reads as *code* is not moved by a comment; one it
reads as a *surface* is not moved by a function body. The snapshot is what the
cache keeps, not a photograph of the disk — and what the cache keeps is exactly
what would make a step run again.

One limit, named: a slot holds several states of the same path (see
`cache.Pick`), so a file changed and changed back reads as `unchanged`. That is
the honest answer to "would the gate re-measure this?", not to "was this file
ever touched?".

<!-- x3-dist version=v0.278.0 capabilities=8e41d6bd5f1758414d116fe84cea9a0f5dda090feb78d675133f4927daf8bac3 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
