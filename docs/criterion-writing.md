# Three ways a criterion is lost in the writing

[The pages](INDEX.md) - [what x3 is](../README.md)

## Three ways a criterion is lost in the writing

A criterion can be written **almost** right, and each near miss used to be
silent. Measured on a real list: all three.

**The key beside it is not the criterion key.** A line reading `test: ...` where
the key is `proof` was skipped without a word, and the box was reported as one
**nobody wrote a criterion for** - a mis-spelled criterion could not be told from
a missing one. Worse, the test that line names then belongs to no criterion at
all, and a tidy-up round deletes it.

So a skipped line is checked once more: **would it be a valid criterion if the
key were right?** If it would, `unknown_criterion_key` says so - as a warning,
because the line is a writing mistake rather than a debt. If it would not, the
line is prose and the engine stays quiet. That narrowness is the point: a gate
that speaks on every `name: value` line is a gate nobody reads.

**The runner is handed a sentence.** A `command` criterion's argument goes to the
runner as written, and whitespace splits it into separate arguments. A box whose
criterion read `run-the-test this was finished last week by hand` failed on every
single pass, looked *unproven* for months, and the reason was written nowhere.
The kind can now declare the shape it needs:

```json
"kinds": { "test": { "when": "command", "prefix": ["go", "test", "-run"], "argument": "word" } }
```

`word` demands exactly one; `line` is the default and today's behaviour, so
nothing changes for a kind that does not declare it. A line that does not fit is
`criterion_cannot_run` - its own code, deliberately not `box_unproven`: one says
*"measured, and it does not hold"*, the other *"this could never have run"*, and
a reader who cannot tell them apart goes looking in the wrong place.

**The place carries a line number.** `contains READER.md:299 …` reads a path that
does not exist, so the criterion quietly measures nothing; and even where such a
path resolves, the number moves with the next edit and the criterion starts
reading a body nobody meant it to. This is the same law the finding baseline
lives by - an identity carries no line number - so a criterion whose place ends
in `:299` is `criterion_line_number`, a warning.

<!-- x3-dist version=v0.159.0 capabilities=7f149416d4d1ff326e5dfdfc03d02ef69f7f13faf78170206823bb1ba5536ceb template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
