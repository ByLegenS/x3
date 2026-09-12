# The example that stands in for a counterpart

[The pages](INDEX.md) - [what x3 is](../README.md)

## An example inside the file is a counterpart

```json
{ "kind": "pairing", "sources": ["internal/**/*.go"],
  "counterpart": "in-directory:*_test.go", "requires": "references-a-declaration",
  "satisfiedBy": ["an-example-in-the-file"] }
```

A test moved into [an inline example](case.md) leaves no file behind. Without
this line the rule calls that subject untested on the day its test started
running — a gate that **turns red as the migration succeeds**, which is the
opposite of what it is for.

**Why not a third `requires` value.** `requires` says what is asked *of the
counterpart*. A third value would make "a neighbour names me" and "I carry my
own example" exclude each other, and a project mid-migration wants both at
once: some subjects have moved, most have not.

**Why not an exemption.** An exemption needs a reason because a person is
overriding a measurement. Nothing is overridden here — the subject *is* tested,
in the engine's own form. Demanding a written reason would produce one
sentence copied over every migrated file, and copied reasons are how a gate
goes blind.

**What counts.** The engine's own reading, the one `x3 case` runs: the
directive sits on a **function declaration in this file**, its body parses, it
compares something, and it fits that function's signature. So a directive
cannot be pasted into silence — `//x3:case: in=() out=_` compares nothing and
is not an example, and a directive floating in a comment binds to no
declaration.

**Where the line is.** This asks the same question of both forms and no more:
`requires: "exists"` accepts an empty `foo_test.go` without asking whether it
passes, and this accepts a well-formed example without running it. Whether the
example is *true* is `x3 case`'s answer, exactly as whether the sibling test
passes is the test runner's. A project that declares examples and never runs
them has that hole on both sides of the pairing rule.

**The declaration is not free.** `satisfiedBy` is an escape hatch and carries
the engine's law for one: a criterion that takes no finding away is
`dead_satisfier`, red. It is measured on findings *removed*, not on files that
happen to carry an example — a subject with a counterpart of its own was never
going to be red, and cannot keep the declaration alive. A rule with no subjects
at all reports `empty_scope` instead: there was nothing to rescue, and one fact
deserves one finding. Written empty (`"satisfiedBy": []`), a name written twice
or an unknown name is exit `2`.

Each rescue is counted: `rules[].satisfied` in the report, and a line of its
own on stderr. A rescue nobody can see is a hole nobody can find.

<!-- x3-dist version=v0.135.0 capabilities=20b1c981592aadc535186606e8f8a71ef40bca930051d3944300ed66b331d3e9 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
