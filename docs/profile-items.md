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

That is the whole fence: the engine says which kind of token is legitimate
where, instead of widening the grammar everywhere and hoping.

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

<!-- x3-dist version=v0.177.0 capabilities=227cc35e8bdb07e3cf10686eee3fa0a683cc37b40b65fa933f681430fc453507 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
