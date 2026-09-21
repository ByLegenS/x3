# The tree read as a set

[The pages](INDEX.md) - [what x3 is](../README.md)

## The tree itself is a set

```json
{ "kind": "consistency",
  "left":  { "from": "tree", "select": "dirs:apps/*" },
  "right": { "from": "tree", "select": "files:apps/*/gate.ps1" } }
```

Every other extractor reads the **inside** of a file, and the inside of a file
can only be read once the file is there. So the question *"does every directory
of this shape carry these paths"* cannot be asked with them: the rule opens with
the very file it requires, and a directory that carries nothing never becomes a
subject at all. The gate is then green about the only case it existed for.

| `select` | A member is |
|---|---|
| `dirs:<pattern>` | a directory whose path matches |
| `files:<pattern>` | a file whose path matches |

**What enters the set is what the stars caught**, the same rule the other
extractors follow. `dirs:apps/*` yields `alpha`, `beta`; `files:apps/*/gate.ps1`
yields only the apps that have one — so the two sides are comparable and the
finding names the app, not a path. A pattern with no star puts the path itself
in the set, and the question becomes a plain "is it there".

A directory counts when it holds at least one file. Nothing is lost by that: git
cannot record an empty directory, so in a tree under version control "a
directory that exists" and "a directory holding a file" are the same set.

The rule's `sources` does not narrow this side — that narrowing is exactly the
blindness — but the rule's `exclude` still does, and a dead `exclude` pattern is
red as everywhere else. A pattern that matches nothing leaves the side empty,
and an empty side is `empty_scope`, not a tree full of debts: a mistyped pattern
must not be able to write its own mistake onto the code.

### Directories a walk steps over

**What it catches:** the clutter a gate cannot see. A source walk deliberately
does not enter `vendor`, `testdata`, `node_modules`, or any name beginning with
`.` or `_` — their insides are not this project's code, and reading `.git` on
every run is the most expensive way to learn nothing. For *"which directories
are here"* that same filter is blindness: a directory nobody can see is a
directory no rule can judge, which makes it the first place a rule is escaped.

```json
{ "kind": "consistency", "compare": "left-subset-of-right",
  "left":  { "from": "tree", "select": "dirs:*", "hidden": true },
  "right": { "from": "json", "file": "layout.json", "select": "keys:*" } }
```

`hidden` belongs to `from: "tree"` and to nothing else — every other extractor
reads the **inside** of a file, and a file the walk never reached has no inside
to read. A reader that cannot honour it refuses it rather than accepting it in
silence, because a declaration that opens nothing is worse than no declaration:
it tells the person who wrote it that the question is being asked.

**It cannot arrive through a star.** Nothing about `dirs:*` opens `.git`; the
word has to be written. This is the same law a criterion's `sources` already
follows — a skipped directory is entered only when it is *named* — and the
reason is the same: a filter that a wildcard can lift is a filter that is gone
the first time somebody writes `**`.

`syntax`, `arch`, `boxes` and `secrets` all read their sources this way, which
matters most for a **fixture tree**: an experiment keeps its planted files under
a skipped name so no other rule trips over them, and the one rule that measures
them writes that name.

**The declaration binds to the side that wrote it, not to the run.** In a
settings file with ten rules, one declaring `hidden` must not put every file of
`.git` and `vendor` in front of the other nine; they would drown in reds the
person never asked for. Measured in one run of the control experiment: the
declaring rule names three directories, and a second rule asking the *same
question without the declaration* stays green beside it.

`per` refuses the declaration: a comparison per instance binds every path to a
component, a path under a skipped directory belongs to none, and the rule would
open nothing while claiming to.

<!-- x3-dist version=v0.272.0 capabilities=46500cb8ad080cc4b9df6cfb91418347928fab317327288705dd32ff203740a5 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
