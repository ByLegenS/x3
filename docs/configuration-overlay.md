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
| a list is still **added** to a list | same law as the parts |

**A fragment that lands on nothing is an error**, not a quiet pass: an
experiment that stopped matching its rule would take its green from a rule it
never touched. The run exits `2`, the code for "could not measure".

### Narrowing a run to one rule

`-only` runs the named rules and nothing else (`syntax` spells it `-check`),
and an unknown name is an error for the same reason. It is what lets an
experiment run against a **fixture tree**: the real configuration, the real
rule, a planted violation next to it, and none of the other rules reporting
that they found no files there. The report prints `selected`, so a narrowed run
can never be mistaken for a full one.

<!-- x3-dist version=v0.187.0 capabilities=b707b17016a8225a6448125a5aec52642c7b8cb711b138fdf7ebf7781dfd5bd4 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
