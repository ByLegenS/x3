# The sentence that says why a debt is held

[The pages](INDEX.md) - [what x3 is](../README.md)

## The sentence that says why a debt is held

**What it catches:** the file that regenerates itself over the one thing in it a
machine cannot produce. A baseline is a list of debts, and some debts are not
oversights — they are decisions: *this one cannot be fixed, and here is why.*
Write that sentence into the file and the next `-update-baseline` used to erase
it, because the refresh rewrites the list from the run.

```json
{ "id": "c0ad7be70e96", "rule": "block_too_long", "path": "b.go",
  "why": "the generated client is replaced wholesale; editing it here is lost work" }
```

`why` is **written by hand** and belongs to no run. Nothing produces it, nothing
validates its content, and the engine never invents one.

### The refresh regenerates the list, and carries the sentence

A refresh rewrites every record from what the run measured. It could only keep a
hand-written field by knowing which record it belonged to — which is exactly what
the baseline already has: **a record is keyed by its identity**, so the sentence
is carried from the old record to the new one by `id`.

| In a refresh | What happens to `why` |
|---|---|
| the record is still measured | carried over, unchanged |
| the record moved to another segment | it moves with the record |
| the run refused to write (growth) | nothing is written, so nothing is lost |
| the record is no longer produced | it is dropped, and the run **says the sentence** |

### It is not part of the identity

The identity is the rule, the path and what the finding says — not the reason a
human gave for holding it. Were the sentence part of it, editing a typo would
give the debt a new identity: the same run would report the old record as
`dead_baseline` and the unchanged finding as growth the baseline does not hold.
Writing a reason would mean **growing** the debt.

### A dying record says its reason out loud

When the debt is paid the record is dropped, and so is the sentence attached to
it — the justification for carrying a debt has no subject once the debt is gone.
What it may not do is vanish quietly, so both places that name a dead record name
its reason too:

```text
dead_baseline: "c0ad7be70e96" is in the baseline but the run no longer finds it;
    refresh the baseline so the debt cannot come back
    (it was held because: the generated client is replaced wholesale)
DROPPED c0ad7be70e96 block_too_long b.go -- it was held because: ...
```

A `why` written with nothing in it is refused with exit `2`. An empty reason
reads like a reason and says nothing; the hand that wrote the field meant to
write a sentence.

<!-- x3-dist version=v0.150.0 capabilities=162fe2ced0d891cd8733aba17d14fcabc3618c79fd93d1547dabbbcdc64d0fcb template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
