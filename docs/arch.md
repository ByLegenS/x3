# The shape of the project

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 arch`

**What it catches:** the shape a project claims in prose — "the core does not
know the modules", "only the entry point wires them" — drifting from the shape
it has. The rules live in `x3.json`, the verifier in the engine: the project's
names never enter a verifier.

**All nine rule kinds are built:** `deps` with three matchers (`import` reads the
import graph, `literal` reads names the compiler never sees, `symbol` reads what
the code uses), `required`, `pairing`, `flow`, `exposure`, `duplication`,
`vocabulary`, `consistency`, `containment`. A kind with no verifier stops the run
with exit `2` — a planned kind that passed silently would be worse than no rule.

```
x3 arch [-config <file>] [-out <file>] [-baseline <file>] [-update-baseline] [dir]
```

**No `arch` section means exit `2`**, deliberately the opposite of the language
gate's default: a language has a universal default, an architecture does not, and
an invented default architecture is the most dangerous silent green there is.

### `arch` in `x3.json`

```json
{ "arch": {
    "components": { "contract": ["internal/engine/**"],
                    "checkers": ["internal/arch/**", "internal/lang/**"],
                    "entry":    ["cmd/**"] },
    "rules": [
      { "name": "the-contract-knows-no-implementation", "kind": "deps",
        "match": "import", "from": "contract", "deny": ["checkers", "entry"] },
      { "name": "only-the-entry-point-reaches-a-checker", "kind": "deps",
        "match": "import", "to": "checkers", "allowFrom": ["entry"] } ] } }
```

That is this repository's own section; the engine holds itself to it on every
`check.ps1` run.

### Components

A component is a name and a set of path patterns, declared here by path rather
than labelled in the source, so the shape is reviewed in one place.

See **component path patterns** in [REFERENCE.md](../REFERENCE.md#component-path-patterns).

That is the whole syntax, and the omission is loud on purpose: a pattern starting
with `!` is **refused, exit `2`**, in every section that takes patterns. In most
dialects it negates; here it would be read as an ordinary name, match nothing,
and whoever wrote it would read the green as proof the exclusion worked. A blank
pattern is refused for the same reason.

A single `*` catches the component's **instance** — `internal/modules/*/**` tells
`alpha` from `beta` — which is what `except: "self"` compares. Two boundaries the
engine enforces: **paths resolve against the module root**, so checking a subtree
does not invalidate them; and **a file has one component**, because a package
silently counted on the wrong side is worse than one with no component.

### Narrowing the source set

`sources` says what a rule reads; `exclude` takes files back out and is read
**first**, so it can only narrow. Both may sit at the top of `arch`, where rules
inherit them, and on a rule, where the rule's own list **replaces** the inherited
one — otherwise a rule could never escape a pattern declared above it.

```json
{ "arch": { "sources": ["**/*.go"], "exclude": ["**/*_test.go"] } }
```

An exclusion is an escape hatch, held to the same law as an exemption: a pattern
taking **no** file out is `dead_exclusion`, one taking **every** file out is
`empty_scope`, and `"exclude": []` is exit `2` — an empty list cannot be told
from an absent one, and whoever wrote it would have silently inherited instead.
This is not `surface.exclude`, which drops paths from an `exposure` rule's
surface; `exclude` drops files from what the rule reads at all.

### The two rule forms

| Form | Written | Asks |
|---|---|---|
| outward | `from` + `deny` | may **this** component touch those? |
| inward | `to` + `allowFrom` | who may touch **this** component? |

Mixing them is refused. `except: "self"` narrows a component's ban to *other
instances of itself*; it needs a single star in that component's patterns and the
component in its own `deny` list, or it would exempt everything. In the inward
form a component's own files are always inside, and a file in no declared
component is outside.

### The `literal` matcher — names the compiler never sees

**Catches:** a table, queue or bucket name spelled by a component that does not
own it. The name is a string, the code compiles, and the failure arrives at run
time. It reads **string constants** in Go and **whole lines** elsewhere.

```json
{ "kind": "deps", "match": "literal",
  "sources": ["**/*.go", "**/migrations/*.sql"],
  "pattern": "db_(?P<owner>[a-z0-9]+)_[a-z0-9_]+",
  "owner": "modules/${owner}", "alsoAllow": ["entry"] }
```

**Ownership** (`pattern` + `owner`) derives the owner from the name itself, so no
hand-kept list goes stale; **prohibition** (`from` + `pattern`) says this
component may not spell such a name at all. Writing both is refused. [Comments
are exempt](arch-prose.md#prose-is-not-code) in every language it reads; exemptions cover Go
only, since a `.sql` file has nowhere to write one.

**An owner no instance carries.** The pattern decides who the owner is, and it
can be wrong: `db_session_token` gives `modules/session`, and if no module is
called `session` then **no file can belong to it** — so every file spelling that
name is a violation, and the rule invents work that does not exist. Measured on
a real production Go application, one pattern produced **1601** such findings,
another 372, a third 44, and the only way out was writing character classes ugly
enough to exclude the words by hand.

```json
{ "owner": "modules/${owner}", "unknownOwner": "ignore" }
```

`ignore` says a captured word that names **no instance of the component** is not
an ownership claim at all — the pattern over-matched, and only the project can
know that. `report` is the default and keeps today's reading, so nothing changes
for a rule already written.

It cannot be used as an off switch: if **every** match was dropped, the rule
weighed no ownership at all and the run is `empty_scope` red, naming how many
were dropped. A setting that silences a rule completely has to say so.

### The `symbol` matcher — capabilities, not layers

**Catches:** a component using a capability — encrypting, opening an outbound
request, retrying — that is one call inside a package everybody imports.

```json
{ "kind": "deps", "match": "symbol", "from": "modules",
  "deny": ["crypto/**", "net/http.NewRequest", "**/*.Retry"] }
```

**`deny` holds path patterns here, not component names.** Both the import path
and each `qualifier.Name` selector are read: an alias is resolved to its full
path, while a qualifier that is not an import is kept as written, which is what
makes `**/*.Retry` find retry logic whose package cannot be known. Catching only
the import lets a file call the function through a package it already has;
catching only the call lets the package be aliased out of sight.

### `required` — the mark every file of a class must carry

**Catches:** the seventh file somebody adds without the protection every other
file of its kind carries.

```json
{ "kind": "required", "sources": ["scripts/**/*.ps1"],
  "marker": "regex:(?m)^\\[CmdletBinding\\(\\)\\]$" }
```

The only marker form is `regex:<pattern>`, with `^` and `$` bound to a **line** —
a mark need not open the file; one that must writes `\A`. A finding carries **no
line number**: what is missing is missing from the file, not from a place in it.

### `pairing` — does anybody touch this file?

```json
{ "kind": "pairing", "sources": ["internal/**/*.go"],
  "counterpart": "sibling:*_test.go", "requires": "references-a-declaration" }
```

**Not coverage** — the question is whether any file at all names this one. There
are two counterpart forms and they are **not the same question**.

| Form | What it looks for | `*` means |
|---|---|---|
| `sibling:<template>` | the subject's **namesake**: `beta.go` asks for `beta_test.go` | the subject's own base name |
| `in-directory:<pattern>` | **any** neighbour matching the pattern | an ordinary glob star |

Read as a glob either string also says which files are counterparts, and those
are never subjects. `requires` is `exists` (default) or
`references-a-declaration`; a subject with no declarations cannot be asked the
second question.

The directory form exists because one suite file can cover a whole directory.
Measured on a fixture where `suite_test.go` names both `service.go` and
`ledger.go`: `sibling:` reports **2 violations** — it wants `service_test.go`
and `ledger_test.go`, and both would be empty files — while `in-directory:`
reports **none**. Under `references-a-declaration` **one** neighbour naming one
declaration is enough; asking every match to name it would turn "does anybody
touch this file" into "does everybody".

It is not an off switch. In the same fixture a declaration no neighbour names is
red (`N file(s) here match "*_test.go" and none names anything declared here`),
and a directory with no matching file at all is red too (`no file in this
directory matches "*_test.go"`).

### `flow` — where a value may appear

**Catches:** a restricted handle escaping the one place allowed to hold it.

```json
{ "kind": "flow", "sources": ["internal/**/*.go"],
  "value": { "field": "Module.pool" }, "allow": ["receiver"] }
```

The places are `receiver`, `argument`, `result`, `assignment` and `other` — the
last so an unrecognised position is refused rather than skipped. **`allow` lists
what is permitted; everything else is red**, because a deny list would leave a
place added later silently free. **It reads names, not types**: the type in
`value.field` proves only that the field is declared somewhere the rule reads,
and if it is not the rule is `empty_scope` — a renamed field must not leave a
green rule behind.

### `exposure` — what reaches the outside

**Catches:** an internal cost or margin the day the struct holding it is written
out. Import and flow rules cannot see it: the field is already in that package,
and the violation is that it leaves.

```json
{ "kind": "exposure",
  "surface": { "components": ["core", "modules"], "exclude": ["**/admin/**"] },
  "fields": ["*cost*", "*margin*"], "carrier": ["json-tag", "map-key"] }
```

`surface` minus `exclude` is how the same field stays legal on an operator screen
and illegal on a tenant one. `fields` match case-insensitively; `carrier` is a
struct field's `json:"…"` name or a string key in a composite literal. **A field
with no tag is not seen** — deciding whether it is serialized needs type
resolution, and calling every field an exposure would drown the gate.

### `duplication` — the body written twice

```json
{ "kind": "duplication", "across": "modules", "minLines": 8 }
```

Bodies are reprinted from the syntax tree with blank lines and indentation
dropped, so formatting differences disappear. Below `minLines` nothing is
compared — two modules both writing `return nil` are not a finding. The
comparison is between **instances**, so `across` needs a single star, and with
fewer than two the rule is `empty_scope`. **Identifier normalization is off**:
with it on, two deliberately separate but similar bodies would be caught too.

### `vocabulary` — the words a layer must not know

**Catches:** the core *knowing* a module without calling it — the name living in
a constant, a field name, a configuration key, a log line.

```json
{ "kind": "vocabulary", "in": "core",
  "sources": ["**/*.go", "ui/**/*.js", "config/**/*.json"],
  "terms": { "componentNames": "modules" } }
```

`terms` is either `componentNames: "<component>"` (the **instance names** are the
terms, so a new module is covered the day it appears) or `words: [...]` written
out. The tokenizer is the language gate's, so `alphaTable` is `alpha` + `table`
and a name cannot hide inside camel case. **[Comments are
exempt](arch-prose.md#prose-is-not-code) by default**; `"comments": "checked"` covers prose too.

#### Word forms

A rename is not finished while an inflected form of the old word is in the tree,
and the list of forms cannot be kept by hand.

```json
{ "terms": { "words": ["invoice"], "match": "forms" } }
```

With `match: "word"` (default) only `invoice` is a finding; with `forms`, any
word that *starts with* the term is one — `invoices`, `invoicing`, `invoice_id` —
and the message names the term it came from. A term used this way must be at
least four characters: a short prefix falls inside innocent words.

### `containment` - a component's parts stay under its root

**Catches:** a component that is no longer movable — one part outside its root,
so deleting it leaves litter and copying it leaves the part behind.

The hard question is not where a file is but **which component it belongs to**,
and that cannot come from the directory: read that way every file is already
where it is. So ownership is declared as a **key**, a short prefix each component
puts at the start of the names of the things it owns:

```json
{ "kind": "containment", "sources": ["**"],
  "keys": { "billing": "blgx_", "orders": "ordx_" } }
```

```
apps/billing/blgx_handler.go          ok
core/blgx_helper.go                   part_outside_its_root
core/money.go                         carries no key, nobody's part
```

A key must be **5 to 10 characters** — shorter and it matches by coincidence,
longer and it is a name, which brings the coincidence back — and two keys may not
start alike. All are configuration errors. If no file carries any key the rule is
`empty_scope`: a rule that matched nothing has not passed, it did not run.

### Two sets that must agree

**Catches:** two places that must say the same thing and drift apart — a
generated list against the code that produced it, a rulebook against the settings
that enforce it. The `consistency` kind, the extractors that read each side, the
escape hatches they carry and the engine's own roster have a page of their own,
*The two sets a rule compares*.

### Fields a rule has

See **arch rule fields** in [REFERENCE.md](../REFERENCE.md#arch-rule-fields).

Configuration is validated **strictly and up front**: an unknown key, a key
belonging to another kind, a missing required key, a duplicate `name`, an
undeclared component, an unknown `policy` or an empty `rules` list stops the run
with exit `2`. An empty list is an error on purpose — a check with nothing in it
is a silent pass.

### Exemptions

`arch` adds no directive type; a violation is silenced with the dictionary's own:

```go
import (
	//x3:allow:arch: the ledger is wired to alpha here, and only here
	"example.com/app/modules/alpha"
)
```

**`skip` does not silence `arch`** — say what you are silencing by name. **An
exemption binds a line, not a tree**: above one import it covers that import,
above the `package` clause the file, and above a parenthesised `import (` block
it binds nothing and shows up dead, because a block-wide silence is a deleted
rule. A reason is required, exemptions are listed in the report separately from
violations, and a dead exemption is red.

### Scope integrity

A rule that matched nothing is `empty_scope` and red — engine behavior, not
something you choose. Every component the rule names is measured, the object side
included: a `deny` list pointing at a component with no files can never turn red.
The measurement follows the files a rule really reads: one that reads text
(`vocabulary`, `literal`) is weighed over **every** file and not only the Go
ones, or a layer written in no Go at all would be told it "cannot have run" in
the same run where it named a real violation — and its green would be
unreachable. A component a rule only takes **names** from
(`terms.componentNames`) is not asked this at all: it is never read, and its
emptiness is already the empty term set.
`empty_scope` and `dead_exemption` are always `block` whatever the `policy` says;
a policy grades how bad a violation is, and neither of these is a violation —
they are the measurement failing.

**`minimum` is the floor a scan must reach.** Zero is only the last step of a
fall: a rule that read a hundred paths still reports green after a rename leaves
it three.

```
BLOCK scope_below_minimum: every-root-a-gate-names-is-still-there
  the rule saw 3 subjects and 40 were declared; a scan that shrank is a gate that stopped looking
```

`minimum` belongs to **every kind**, counting the rule's own subject — text-reading
kinds included, weighed in the walk that counts them. Not [`expect`](scan.md#expectations),
which counts verified directives in a scan: same disease, two organs.

### What a run looks like

```
BLOCK …/modules/beta/beta.go:3 (modules-must-not-know-each-other): forbidden_dependency
	component "modules" must not import another instance of itself
	…/modules/beta -> …/modules/alpha
x3 arch: 5 file(s) - 3 rule(s) - 1 block, 0 warn, 0 exempted
```

The same tree with `policy: "warn"` prints `WARN` and exits `0`; with the
exemption in place it prints `ALLOW …` and exits `0`.

### The arch report

No timestamp, and violations sorted by rule, then file, then line.

```json
{ "version": 1, "files": 5,
  "rules": [ { "name": "modules-must-not-know-each-other", "kind": "deps",
               "policy": "block", "subjects": 4, "violations": 1 } ],
  "violations": [ { "rule": "…", "file": "…", "line": 3, "subject": "…",
                    "object": "…", "code": "forbidden_dependency",
                    "policy": "block", "message": "…" } ],
  "exemptions": [], "summary": { "rules": 3, "violations": 1, "warned": 0, "exempted": 0 } }
```

`subject` and `object` are the packages; the message names the components.
`summary.violations` counts only `block` findings, `warned` the rest.

### Error codes

See **arch error codes** in [REFERENCE.md](../REFERENCE.md#arch-error-codes).

<!-- x3-dist version=v0.119.0 capabilities=3461040f39065d6a34ecc51e2c04d0c0200bfb0a101424f5cc8efaec1cc68c21 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
