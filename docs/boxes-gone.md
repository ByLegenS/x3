# A departure that never happened

[The pages](INDEX.md) - [what x3 is](../README.md)

## A path that was never there

A criterion phrased as a **departure** — *the file is gone* — can only prove
itself by losing once: the path was there, the work removed it, the criterion
turned green. Mistype the path and that day never happens. `gone src/push_tst.go`
is met the moment it is written and can never fail again, and no amount of
reading the tree will say so, because the tree agrees: that path is not there.

The rule that guards an **absence** criterion cannot be used here. That one asks
whether the path it reads is still on disk, and a departure asks the opposite; it
would refute every honest `gone` criterion ever written.

```json
{ "boxes": { "suspect": { "gone": true } } }
```

So the question goes to the engine's **own record** instead of the tree: *did
any run this tree remembers ever read that path?* The gate writes down every
file each step reads, so the cache is a history of the tree in the engine's own
hand. A path the record has never seen is `box_suspect` — the same class as a
criterion pointing at `go.mod`, from the other direction. Measured on a real
production Go application: of 16 such criteria, 11 named a path the record
carries, 2 were build outputs, and **3** named a path that was never there.

(The question used to go to the repository's commit history. It no longer does:
the engine answers it from what it measured itself, so a tree with no version
control answers just as well as one with it.)

Those build outputs are the edge, and they are not excused by hand. A path the
project tells its tools to ignore is never read by a step either, so the record
has nothing to say about it — x3 reads the ignore rules itself (`.gitignore`
files, parsed in `internal/source`, with no tool invoked) and the paths they
cover are dropped from the rule and **counted** in the summary as `untracked`.
An exemption list would silence a verdict; a count keeps a growing blind spot
visible. A tree whose record is missing — no `cache.dir` declared, or declared
and never filled — is exit `2`, in two separate sentences, for the same reason
as everywhere else here: a rule that cannot read would call every departure a
typo.

<!-- x3-dist version=v0.288.0 capabilities=b7fca59edbf65759483bfdca34f14aeafbe84562986ae2f4e8a4b427249da8fb template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
