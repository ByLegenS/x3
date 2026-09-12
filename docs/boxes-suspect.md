# Criteria that stopped measuring

[The pages](INDEX.md) - [what x3 is](../README.md)

## Criteria that stopped measuring

The quietest way a work list dies is criteria that cannot fail. Three writings do
it, all three go green, and none measures anything:

```json
{ "boxes": { "suspect": { "repeat": 3, "always": ["go.mod", "README.md"],
                          "selfProof": true } } }
```

`repeat` finds one criterion carried by that many items or more; `always` a
criterion pointing at a path the project carries in **every** state; `selfProof` a
criterion whose scope is the very document the item is written in. Each is
`box_suspect`. `repeat` counts distinct items and skips `manual` criteria, and is
refused below `2`; `selfProof` asks the scope matcher, so a glob that reaches the
document is as visible as a path that names it. The section must ask for at least
one of the three.

### A selector whose name has left the tree

A `command` criterion usually hands its runner a **selector** — the name of the
one check that would prove the box. Nothing binds that selector to the tree: the
day the name is removed the selector matches nothing, the runner finds nothing to
run, and a runner with nothing to run exits zero. The box stays closed and its
proof is gone, without a single line changing in the list.

```json
{ "boxes": { "suspect": { "selector": true } } }
```

For every **closed** box the selector is compiled as a pattern and matched
against the names the tree declares. Two answers are kept apart, because the
repairs are different:

- no declaration anywhere carries a name it matches — the check is gone;
- the names it matches are declared **only outside** the place the criterion
  looks at — the check moved, and the criterion still points at its old home.

Both are `box_dead_selector`, the one finding in this family that **cannot be
frozen into a baseline**: the other three are how a list was written, this one is
a box closed without proof.

**A selector is asked in the parts its runner reads it in.** Compiled whole, a
selector that names two checks stays alive as long as *one* of them is left, and
the other one's measurement disappears without a word — the surviving half hides
the dead half. So the selector is first cut at its top-level subtest separator
(`/`, only the first element picks a top-level name) and then at its top-level
alternatives (`|`), and each part is asked on its own; the finding names the part
that died. A separator inside a group or a character class splits nothing —
`Test(A|B)` is one name — and an escaped separator is text.

### A gate step whose name has left the tree

A step in a gate script hands its runner the very same selector, and **it does not
need a box at all**. When the check it names is deleted, `-run` matches nothing,
the runner exits `0`, and the step burns green having measured nothing — with no
box anywhere to turn red on its behalf. Measured in a pilot: of five such steps,
three named a check that no longer existed.

When a project declares where its steps live and how to read a selector out of
them, the same question is asked of every line it finds:

```json
{ "boxes": { "suspect": { "selector": true },
             "holds": { "sources": ["gates/*.ps1"], "selects": ["-run\\s+'?([^\\s']+)"] } } }
```

Findings are `dead_step_selector` and carry the **file and line** they came from
plus the step's own text, because there is no box to name. The two answers are
kept apart exactly as above: a name that is nowhere, and a name declared only
outside the place the step runs. Without `holds.selects` no step is read; without
`suspect.selector` no step is judged — and a tree in which not one declaration can
be read is exit `2`, here for the same reason as above.

Open boxes are never asked. A criterion written before the check exists is how a
list is meant to be used, and asking there would fill a plan with the red of work
that has not started. The engine learns which argument carries the selector from
the kind (`batch.select`) — the same reading `-holds` uses from the other side —
so no runner's flag is written into the engine. A tree in which not one
declaration can be read is exit `2`: an empty reading would call every selector
dead.

The answer needs **no run at all**, and that is the point. In a shell that cannot
reach the runner, the database or the network, every criterion is unmeasured and
a box closed on a vanished name looks exactly like the rest of the noise.

<!-- x3-dist version=v0.153.0 capabilities=623ccd05726c1539c169bc853f4869d52533d5c7fa7810be91e81b097e7bfbb6 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
