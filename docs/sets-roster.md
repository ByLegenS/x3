# The engine read as a set

[The pages](INDEX.md) - [what x3 is](../README.md)

## `from: "x3"` — the engine's own roster

**Catches:** a rulebook sentence saying *"this one is guarded"* after the guard
was deleted, downgraded or never wired up. A `regex` over the settings file finds
the name in a rule turned down to `policy: "warn"` last month, which is exactly
the day the claim became false. `from: "x3"` reads the file **as a
configuration**, so the set carries the engine's verdict rather than the text.

```json
{ "left":  { "from": "regex", "file": "RULES.md", "select": "`x3: ([a-z-]+)`" },
  "right": { "from": "x3", "file": "x3.json", "select": "in-force" },
  "compare": "left-subset-of-right" }
```

`in-force` is every name the configuration puts in force — each rule, baseline,
pattern, guard and expectation by its `name`, each section by its key, and a
nameless mechanism by its **path in the configuration**. `commands` is the
subcommands it configures, and the map from a section to the command enforcing it
is the **engine's own roster** rather than a list kept beside it: a second copy
goes stale the day the engine grows a command, and a stale copy reports its own
blindness as *"all of it runs"*. Sections describing the engine's own workings
(`x3`, `update`, `baseline`, `cache`, `adoption`) enforce no check and are in
neither set. **`policy: "warn"` is not in force**, and neither is anything nested
under it.

**The file is read the way the engine reads it — parts included.** A large
project splits its configuration: the root declares its parts with `include`, and
every command merges them before measuring anything
([splitting the configuration](configuration.md#splitting-the-configuration)). The roster follows
the same reading. It did not always, and the bug is worth keeping written down: it
read the named file alone, so the day a project split its settings, every rule
that moved into a part left this set in silence. The rule went on running, and the
one question that could have noticed — *is this rule in force?* — answered **no**,
which makes the rulebook's true sentence about it red for a reason nobody can see.
A set that shrinks when a file is split is measuring the layout, not the rules.

### `invocations` — the commands a script actually runs

The other half of the same question: `commands` says what the settings put in
force, `invocations` says what the gate script really calls. Both answers come
from the engine, and that is the point — a hand-written pattern gets the shape of
a command name wrong in two measured ways. A name appearing **only inside a
string** is not a call, and a name carrying a colon (`guard:effective`) is one
word, not two. A pattern reading `x3 ([a-z]+)` counts the first and cuts the
second: the checker that never runs looks green, and the checker that does run
looks missing.

```json
{ "left":  { "from": "x3", "file": "x3.json", "select": "commands" },
  "right": { "from": "x3", "file": "gate.ps1", "select": "invocations" },
  "compare": "left-subset-of-right" }
```

Only names the engine actually carries enter the set. The engine is looked for by
its own name; a project calling it through a variable (`& $bin scan`) says so
with `invoke`, a list of patterns each carrying one capture group — the same
field, spelled the same way, that [`x3 adoption`](adoption.md#x3-adoption) reads.

<!-- x3-dist version=v0.131.0 capabilities=0ae611848b4d163cdcc7ff33b586ecfda3da08f1e6cf326dffcd8d785144d8ea template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
