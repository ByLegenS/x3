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

So the question goes to the repository's **history** instead of its tree: *was
this path ever here?* A path no commit on any branch has carried is `box_suspect`
— the same class as a criterion pointing at `go.mod`, from the other direction.
Measured on a real production Go application: of 16 such criteria, 11 named a
path the history carries, 2 were build outputs, and **3** named a path that was
never there.

Those build outputs are the edge, and they are not excused by hand. A path git is
told to ignore cannot be asked about at all — the history has nothing to say
either way — so git itself is asked whether it ignores the path, and the ones it
does are dropped from the rule and **counted** in the summary as `untracked`. An
exemption list would silence a verdict; a count keeps a growing blind spot
visible. A tree with no history is exit `2`, for the same reason as everywhere
else here: a rule that cannot read would call every departure a typo.

<!-- x3-dist version=v0.231.0 capabilities=f14f5a4bcc3e21610cfc2dc2e9f83279e51b5774778e90563a0a20156120f17f template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
