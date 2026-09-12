# A change that stays in its lane

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 scope`

The other half of the coupled-change problem. `x3 docs` asks what a change must
bring **with** it; this one asks what it must **stay away from**. A part that
ships on its own stops shipping on its own the day it rides in the same commit
as something else.

```
x3 scope [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

A lane is a set of paths, plus the paths allowed to travel with them:

```json
{ "scope": { "lanes": [
    { "name": "site", "paths": ["web/site/**"], "also": ["docs/site/**"],
      "exempt": "lane: crossed" } ] } }
```

A change that touches nothing in `paths` is none of this lane's business; one
that does must stay inside `paths` and `also`, and anything else is named file
by file:

```
BLOCK site: outside_the_lane
        this change is in the "site" lane (1 file(s)) and also touches 1 file(s)
        outside it; split the change, or say "lane: crossed" followed by a reason
        outside: internal/core/money.go
```

Crossing is allowed when it is said out loud — in the commit message, or
`-reason` for a working-tree run — and a marker with nothing after it is red.
`-scope auto` reads the working tree when it is dirty and `HEAD` when it is
clean. Both this gate and `x3 docs` read the diff through the same code, because
two gates disagreeing about what a commit touched would each be right about a
different commit.

### A lane a branch declares

The lane above is opened by the **files**. That is the wrong reading for a
working lane — a branch that exists so two people can work without landing on
each other. There the promise is **"this branch only ever works here"**, and a
commit that touches nothing inside the lane and everything outside it is the
violation itself.

```json
{ "name": "site", "branch": "work/site*", "base": "main",
  "paths": ["web/site/**"], "exempt": false }
```

Three things change: **the branch opens the lane, not the files**, so on any
other branch the lane is silent; **the whole branch is measured**,
`merge-base(base, HEAD)..HEAD`, so a violation in the first commit does not go
out of sight once a clean commit is put on top of it; and **reading the range is
part of the answer** — if git cannot be read, or the base is not there, the run
is red.

`exempt: false` closes the lane: the marker is not even looked for. That is a
deliberate hole in the rule everywhere else in the engine, that an exemption
must be sayable. Write it only when the boundary is somebody's stated condition
rather than a convention. `base` without `branch` is refused.

<!-- x3-dist version=v0.116.0 capabilities=cb9ce75a339d999d0e2fb4646b107500b7661d9711f76b6f18cc8b94e308a691 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
