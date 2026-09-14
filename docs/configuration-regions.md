# A region that runs on its own

[The pages](INDEX.md) - [what x3 is](../README.md)

## A region that runs on its own

A large repository wants two things at once: one gate over the whole tree, and a
gate a single region can run on its own while working in it. `include` gives the
first. The second needs the region's own file to be **runnable**, and that is
where a copied configuration fails three different ways.

```toml
# apps/web/x3.yaml — the combiner. The root never reads this file.
extends = '../../x3.yaml'
include = ['x3/*.toml']
```

```toml
# x3.yaml — the root reads the leaves, not the combiners.
include = ['apps/*/x3/*.toml', 'core/x3/*.toml']
```

The pattern `apps/*/x3/*.toml` does not reach `apps/web/x3.yaml`, one directory
up. So a combiner is never a part, and the rule that a part may not declare
parts is never in the way. A part may not name a parent either: the file a run is
pointed at is the one that says where its settings come from.

`extends` **names** the parent; it does not copy it. Copying breaks three things
at once, and two of them are silent:

| what a copy does | what it costs |
|---|---|
| duplicates the global block | it goes stale — a ceiling changed at the root stays old in the region, and the two runs answer the same tree differently |
| puts the block where the root can read it too | `profile.use` is written a second time and the run stops |
| moves the base of every relative path | `baseline.dir` lands on another file, and the measured root is written as `..` |

That last one is the wall. A baseline stores the root it was measured against,
and refuses a run that measures a different one — *"the baseline was measured
against `.`, this run measures `..`"*. The region and the root measure the same
tree, so they have to write the same root.

**The anchor** is what settles it: the directory every relative path in the
settings is resolved against, and the directory the measured root is written
against. With no `extends` it is the settings file's own directory. With
`extends` it is the directory of the topmost ancestor — so a region's
`baseline.dir`, `cache.dir` and measured root come out exactly as the root run
writes them, and the two runs share one baseline file.

The region still sees only its own rules. An ancestor contributes its own body
and **not its parts**: were the parent's parts inherited, a regional run would be
the root run wearing another name.

**Measured, all four arms, on a planted tree with one baselined finding:**

| run | exit |
|---|---|
| the root, over the whole tree | `0` — debt held |
| the region's combiner, `extends` | `0` — same baseline file, same debt held |
| the same region with the globals copied instead | `2` — the baseline refuses the second root |
| a leaf part that names a parent | `2` — settings error |

<!-- x3-dist version=v0.217.0 capabilities=d5a549f6a4e0603d8c785a64d94e45a05d5266e6a799b7912be83923b7cde7b3 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
