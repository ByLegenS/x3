# A criterion that brings its own expectation

[The pages](INDEX.md) - [what x3 is](../README.md)

## A criterion that brings its own expectation

A runner's expectation belongs to the **kind**: what `go test` prints when it
passes is a property of the runner, not of the hundred items that ask it. But
two claims asked of the same runner are not the same expectation — "the audit is
clean" and "the gate is green" are different sentences, and declaring one kind
per sentence grows the vocabulary by the number of items.

`expect` lets the line pick one by name:

```json
"checks": { "when": "command", "expect": { "mark": " => ", "named": {
    "clean": { "mustNot": ["matched no packages", ".go:"] },
    "green": { "must": ["GREEN"] } } } }
```

```
criterion: checks go vet ./core/... => clean
criterion: checks ./run-gate --all => green
```

The line is cut at the **first** mark: the command before it, the name of an
expectation after it. A name the kind never declared is refused, and the refusal
lists the names it does declare — a misspelling and an unwritten expectation must
not look alike.

**The expectation is named, never written out.** A line that spelled its own
`must` would build its own expectation object, and a shared run groups criteria
by the *address* of theirs: batching would die without a word, one process per
item, and nothing in the report would change to say so. Two criteria that name
one expectation point at one object and keep sharing one call.

`expect` and `output` compose: `output` is the kind's own weight, used by every
line that names nothing, and `expect` is what a line may reach for instead. With
neither, the command carries no weight and does not run.

<!-- x3-dist version=v0.228.0 capabilities=b0cc2c8264c0a7273678fc24026ff67c881b6f49da751e9fe3698afacae3da12 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
