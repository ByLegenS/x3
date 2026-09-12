# Prose is not code

[The pages](INDEX.md) - [what x3 is](../README.md)

## Prose is not code

A rule that reads text has to answer one question before it answers its own: is
this line running, or is it someone explaining why the rule exists? The two
kinds that read text — `deps:literal` and `vocabulary` — answer it the same way,
in every language.

**Catches, by not catching it:** the sentence that documents the rule. A gate
worth having is explained where it bites, and the explanation has to spell the
forbidden name to be an explanation at all.

```js
// The core must never name queue_orders; that is why this rule exists.
export const TOPIC = "queue_orders";
```

**Red:** line 2 — `component "core" must not spell "queue_orders"`.
**Green:** line 1. Both lines carry the same name; only one of them runs.

```json
{ "arch": { "syntax": { ".vue": { "line": ["//"],
                                  "block": [{ "open": "<!--", "close": "-->" }] } },
            "rules": [ { "kind": "deps", "match": "literal", "from": "core",
                         "pattern": "queue_[a-z0-9_]+", "comments": "exempt" } ] } }
```

See **how a rule reads a file** in [REFERENCE.md](../REFERENCE.md#how-a-rule-reads-a-file).

`syntax` is the field `adoption`, `extract` and `freeze` already use, with the
same shape: `line` openers, `block` pairs, and `quoted` runs where a comment
marker means nothing. The embedded list covers `.go` `.js` `.ts` `.java` `.c`
`.cpp` `.cs` `.py` `.ps1` `.sh` `.sql` `.yaml` `.yml` `.css` `.html`.

**An extension nobody declared has no comments, and the whole file is code.** A
rule of this kind falls on every file a project points it at — a dictionary, a
template, a plain text note — and stopping on an unknown extension would break a
working rule the day a new file type appeared. The direction of the mistake is
chosen, not accidental: not knowing the syntax makes the rule read **more**, and
a rule that reads too much shouts, while a rule that reads too little goes
quietly green.

### Inside a Go string is another language

In Go the file's own comment is never a string constant, so the reading above
needs no help. The **inside of a string** is a different matter: a raw literal
carrying SQL carries that language's comments with it, and a `--` line there
never runs either — yet the literal is read whole, so the name in it is counted
as code. No exemption can be written on that line: it sits inside a string, and
a Go comment cannot go there.

Declaring `.go` answers it. The declaration says how a comment is written **in
the text a Go string carries**, and it is applied to each literal, not to the
file:

```json
{ "arch": { "syntax": { ".go": { "line": ["--"] } } } }
```

```go
const q = `
SELECT id FROM ledger
-- queue_billing is drained by the module that owns it
`
const t = `SELECT 1 FROM queue_shipping`
```

Undeclared, both names are red and one of them is wrong. Declared,
`queue_billing` goes quiet and `queue_shipping` **stays red** — the declaration
narrows the reading, it is not an off switch. The embedded table is not
consulted here: only an extension the project declares changes anything, so a
project that writes nothing keeps exactly today's reading.

**What changes for a rule already written:** in Go, nothing — a comment was
never a string constant. Everywhere else the reading narrows to code, so a rule
that was red on its own explanation turns green. Measured on the web tree of a
real production Go application: a `literal` rule fell from 12 findings to **0**
and every one of the 12 was a comment; a `vocabulary` rule over the same tree
fell from 56 to **12**, and the 12 that stayed are code. Nothing became
unreachable —
`comments: "checked"` reads the prose again, and on the engine's own two-sided
test tree it takes the same run from 14 findings to 28.

<!-- x3-dist version=v0.125.0 capabilities=095fd8c2f3b2a4d369248a7cf091184c5d1e81337bdcd62da4b2f9f9fd3abfb4 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
