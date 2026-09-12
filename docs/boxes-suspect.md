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

<!-- x3-dist version=v0.134.0 capabilities=b699977b12735de57dd29e063ac5b448eff274f05e35c42fbda29f7345d6d0b0 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
