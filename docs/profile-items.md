# A rule written once and measured per item

[The pages](INDEX.md) - [what x3 is](../README.md)

## One rule, one list, many checks

**What it catches:** the same rule written three times because three facts differ
by more than one word.

`each` builds a rule once per item, and for a long time an item was a single
value — it filled exactly one token. Counted by hand in one production settings
file (2026-09-13), that left **208 lines in four clusters** untouched: families
whose members differ by a name *and* a query, or a prefix *and* a namespace. A
rule that cannot take the second fact is copied, and a copied rule stops
following the rule it was copied from.

### An item that fills more than one token

When a rule names `per`, **each item is a small object** and the facts that vary
together sit next to each other:

```json
{ "each": "banned", "per": ["pattern", "why-not"], "text": ["pattern", "why-not"],
  "rule": { "name": "no-go-file-writes-a-{banned}", "sources": ["**/*.go"],
            "deny": "{pattern}", "reason": "{why-not}" } }
```

```json
"with": { "banned": [
  { "banned": "sudden-stop", "pattern": "panic\\(\"", "why-not": "a panic stops a run nobody asked to stop" },
  { "banned": "promise",     "pattern": "(?m)^\\s*// TODO", "why-not": "a TODO is a promise nobody measures" } ] }
```

Two checks, one rule. An item **short of a name the rule reads** is refused
(exit `2`): it would set up a check measuring something nobody named. An item
carrying a name **nothing reads** is refused for the mirror reason — the writer
would learn it measured nothing only on the day it stayed green.

### Free text, and where it may go

The token grammar is deliberately narrow — letters, digits and `_ . : @ / * -` —
because a token can end up in a check's **name**. A real query or expression
needs spaces, quotes and parentheses, so a rule may declare a token `text`:

- a free-text token may **not** appear in the rule's name, and may not be the
  `each` token. The name is what a project writes to override or to switch the
  check off, and a name carrying a sentence cannot be written twice the same.
- its value is **escaped into JSON** when it is filled in, so a quote it carries
  cannot break out of the rule; control characters are refused.
- it is one line of **at most 4 000 characters**. The ceiling is a reading
  limit, not a fence — the escaping does not care how long a value is — and it
  was measured rather than chosen: in a production settings file three of
  twelve live queries ran past the first ceiling of 400, the longest at 1 061,
  so that number was refusing real work.

That is the whole fence: the engine says which kind of token is legitimate
where, instead of widening the grammar everywhere and hoping.

### A token that carries a shape

Some of what varies between copies is not a word but a **list**: the files one
check reads, the exclusions another carries. A token declared `shape` takes a
list or an object instead of a string.

```json
{ "each": "spelling", "per": ["where", "pattern"], "shape": ["where"], "text": ["pattern"],
  "rule": { "name": "no-file-writes-a-{spelling}", "sources": "{where}", "deny": "{pattern}" } }
```

The rule writes `"{where}"` — quotes and all, because the library file is JSON
and has to parse before anything is filled in — and the filling takes the quotes
away with the value it puts there. It was measured before it was designed: in a
production settings file eight checks of one shape carried **seven different
source lists**, so a token holding one string could not have carried them and the
eight copies would have stayed.

The fence is the same as free text's: a shape may not sit in a rule's **name** and
may not be the `each` token, because a name is what a project writes to override
or switch a check off and a list cannot be written twice the same way. A value
that is neither a list nor an object is refused — the rule would be left looking
for quotes that are no longer there.


### A rule the project writes itself

The same shape is open to a project for a rule the library will never carry —
`own` takes exactly what the library carries, so nothing about measuring per
item is a privilege of the engine:

```json
{ "profile": { "own": [ { "into": "syntax.checks", "why": "...", "needs": { ... },
    "each": "banned", "per": ["pattern"], "text": ["pattern"], "rule": { ... } } ],
    "with": { "banned": [ { "banned": "...", "pattern": "..." } ] } } }
```

The rule is written once; the list is the project's own, and `x3 profile` prints
every check it grew. Drop an item and the check it stood for is simply gone —
which is what the [control experiment](experiments.md#control-experiments) measures in both
directions.

### The settings measuring their own repetition

A settings file grows two ways, and only one of them is a problem.

Writing a **new check** is growth nobody should slow down: a cap on it makes the
cheapest way past the gate *"write no rule"*, which is the exact trap a count of
files sets when it rewards merging them. Copying an **existing** check is the
other way — three, five, eight items whose only difference is a pattern or a
path. That one is measurable, and until it is measured it is free: the author of
eight copies and the author of one list get the same green.

```json
"profile": { "repeat": { "limit": 2, "vary": 4, "policy": "warn" } }
```

Every array of objects in the file is grouped by **shape** — the set of field
names, plus `kind` where there is one; a name, a `why` or a `reason` is not part
of a shape. A shape written more than `limit` times is reported with the lines it
costs and the names that share it.

`vary` is what keeps the measure honest. Before a cluster is reported, the engine
counts the **leaves that change** from item to item: a leaf that stays the same
becomes the carried rule's body, and a leaf that varies becomes one token of the
list. Past `vary` leaves there is no single rule to write — those items are
separate work that happens to live in one array, and reporting them would tell
the author to do something that cannot be done. Measured on this engine's own
gate: the sixteen steps share a shape and vary in dozens of leaves, so they are
not a cluster; a project's eight syntax checks vary in two, and they are.

A reported cluster carries its own remedy, which is the rest of this page: write
it once with `each`, and let the list carry what differs. `allow` takes a shape
and a **reason** for a repetition kept on purpose — and counts it, rather than
hiding it, because how much was kept deliberately is a measure too.

<!-- x3-dist version=v0.196.0 capabilities=42c9e4d25b581df1ee169b0e61079459d286eff37cd1289ab08a0064586cddf3 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
