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

**What changes for a rule already written:** in Go, nothing — a comment was
never a string constant. Everywhere else the reading narrows to code, so a rule
that was red on its own explanation turns green. Measured on the web tree of a
real production Go application: a `literal` rule fell from 12 findings to **0**
and every one of the 12 was a comment; a `vocabulary` rule over the same tree
fell from 56 to **12**, and the 12 that stayed are code. Nothing became
unreachable —
`comments: "checked"` reads the prose again, and on the engine's own two-sided
test tree it takes the same run from 14 findings to 28.

<!-- x3-dist version=v0.106.0 capabilities=e2484f923558bfc76a33acea1914087aa31ed63c1f55629274b8eda05f26b0cd template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
