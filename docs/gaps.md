# Gaps we know about

[The pages](INDEX.md) - [what x3 is](../README.md)

## Gaps we know about

Stated plainly, because a capabilities document that lists only strengths is a
sales page.

**Baselines and exclusions.** A baseline is coarser than the finding it holds —
`comments` identifies a finding by rule and file, so a file already owing one
over-long block can grow a second unseen. An exclusion is only as narrow as
somebody wrote it: nothing checks that an `ignore` is not swallowing a real
credential, only that it swallows *something*. Exclusions apply to the scan, not
to redaction. `of: files` counts one directory, not a subtree. Nothing checks
that a baseline was reviewed — `-update-baseline` refuses growth, but the first
write accepts whatever the tree owes that day.

**Directive placement judges placement, and nothing else.** A file it calls
canonical can still be reformatted for a reason that has nothing to do with
directives — indentation, alignment, the wrapping of the prose above them. The
rule is not a formatting gate and does not stand in for one; it only says
whether a formatting run would move a `//x3:` line. It also stays silent
wherever the formatter itself does: indented comments, block comments, and doc
comments on `import` declarations.

**Expectations count directives, and only from `scan`.** They say a minimum,
never a maximum, and cannot say "these two exact directives".

**A proposition is checked for names, not for meaning.** `then=` refuses one
that names nothing the call, the receiver or the setup produced, so `then=(true)`
cannot pass — but a tautology written over a real name (`out0 == out0`) names
something and is accepted. Telling those apart needs the types, and the payload
is deliberately handed to the compiler rather than resolved here.

**The cache is per file, not per project.** A checker whose answer depends on
more than one file at a time — `arch`, `freeze`, `docs`, `boxes` — does not use
it.

**Every matcher reads shapes, not meaning.** `symbol` and `pairing` read names,
not types; `flow` does not follow a copy; `exposure` sees tags, not
serialization; `duplication` and `vocabulary` compare text and words, not
meaning; `containment` reads paths, not contents; `consistency` reads four
extractor shapes and no more; `secrets` reads formats; `suspect` finds shapes,
not lies; a lane is paths, not intent; an output expectation is a substring, not
an understanding; and `syntax` carries one parser of its own. A text file cannot
carry an exemption, and the inward `deps` form does not see a component's inside.

**`surface` reads syntax, not types.** The callers of a *member* cannot be
resolved without a type checker, so a changed field or method reports the files
that name its owning type and says so (`usersBasis: "type"`); a dot-import is
invisible either way. A constant's **value** is not frozen, only its declared
type — freeze a value set with `freeze` if the number is the contract. The
surface is the union across build constraints, so a platform-only symbol is in
it. And it measures the API a caller *writes*, never what a call *does*.

**`boxes` measures evidence, not completion.** A command criterion runs where the
gate runs, and a move is trusted once its target exists.

**A name a declared place mentions cannot be weighed.** A criterion carries a
place to compare against; a runner *is* the place. Such a record is `SUSPECT`
and its entry `unsure` - kept on the not-free side, never counted as a bond.

**Examples are one call, not a scenario.** An example cannot expect a panic and
cannot read a value it mutated.

**The language gate speaks one language** — `en` is the only embedded dictionary.

<!-- x3-dist version=v0.76.0 capabilities=1795187cd09d03d2dc0ea34f2fe0acd59b1733058375cf9b61793cec02fa64ca template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
