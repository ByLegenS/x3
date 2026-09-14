# Changing one field for one run

[The pages](INDEX.md) - [what x3 is](../README.md)

## Changing one field for one run

**What it catches:** an experiment that measures its own copy. A gate proves it
is not blind by planting a violation and watching the rule go red — and the
usual way to do that is to write a cut-down settings file with the rule copied
into it. The copy stops being the rule the moment the real one changes: the
experiment stays green, and it is measuring nothing.

`-with` lays a **fragment** over the configuration for one run. The fragment
carries only what changes; everything else is the project's real settings.

```json
{ "syntax": { "checks": { "the-old-term-cannot-come-back": { "policy": "warn" } } } }
```

```
x3 syntax -with experiments/term-as-a-warning.json -only the-old-term-cannot-come-back
```

| Where it differs from `include` | Why |
|---|---|
| a scalar is **overwritten** | in a part that is a clash; a fragment's whole job is to change a value |
| a **map** over a list finds the entry by its `name` | "that rule's that field" has no other spelling |
| a list is **replaced**, or added to with `{"add": [...]}` | the parts share one configuration and add their lists together; a fragment is written to change it, and "that pattern is now this" has no other spelling |

A value that is **text** can be changed in place instead of retyped:

```json
{ "live": { "guards": { "no-test-company-left-behind":
    { "query": { "replace": ["LIKE 'qa%'", "LIKE '%'"] } } } } }
```

`replace` takes **pairs**, and applies them in order: one experiment often has
to change two places at once — the same condition sits on both sides of a join,
and neutralising one of them measures half of what the experiment says.

Copying the whole query into the fragment would work today and rot tomorrow —
the real query changes, the copy does not, and the experiment goes on measuring
text nobody runs.

**A fragment that lands on nothing is an error**, not a quiet pass: an
experiment that stopped matching its rule would take its green from a rule it
never touched. That covers a name no entry carries and text no value holds. The
run exits `2`, the code for "could not measure".

An entry is found by **the value, not the field**: a rule names itself with
`name`, a measured value with `label`, a query with `fact`, and a project may
use a word of its own. The entry whose text matches is the one; two matching
entries are an error, because an overlay that cannot say which one it changed
proves nothing.

A field can also be **removed**: `null` deletes it, and there is no other
spelling for "this is not there any more". The most common experiment is
exactly that — take an exemption away and see whether the rule still looks:

```json
{ "arch": { "rules": { "every-delete-route-hangs-on-a-guard":
    { "left": { "skip": null } } } } }
```

Removing something that is not there is an error, like every other overlay that
lands on nothing.

### Narrowing a run to one rule

`-only` runs the named rules and nothing else — on `arch`, `freeze`, `secrets`
and `docs`, next to the `guard -only` and `syntax -check` that were there
already — and an unknown name is an error for the same reason. It is what lets an
experiment run against a **fixture tree**: the real configuration, the real
rule, a planted violation next to it, and none of the other rules reporting
that they found no files there. The report prints `selected`, so a narrowed run
can never be mistaken for a full one.

**A narrowed run passes no dead-declaration judgement.** The rules that did not
run would have used some of the tree's exemptions and some of the baseline's
debt, and in a narrowed run those look unused. Calling them dead there would
delete, as a side effect of an experiment, a line that a full run still needs.
They come back as `unjudged_exemption`, a warning that says so.

An exemption that **names** the rule it covers is still judged when that rule
ran: the measurement is complete for it, and an experiment that plants a dead
exemption needs to see it go red.

<!-- x3-dist version=v0.207.0 capabilities=1035c1e3b02f7988670af256ea54d32fb4f09fe9686f0bacea84d06acb79df17 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
