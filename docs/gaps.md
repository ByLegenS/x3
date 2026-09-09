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

**Expectations count directives, and only from `scan`.** They say a minimum,
never a maximum, and cannot say "these two exact directives".

**A proposition is checked for names, not for meaning.** `then=` refuses one
that names nothing the call, the receiver or the setup produced, so `then=(true)`
cannot pass — but a tautology written over a real name (`out0 == out0`) names
something and is accepted. Telling those apart needs the types, and the payload
is deliberately handed to the compiler rather than resolved here.

**Updates verify a checksum, not a signature.** `update.pin` binds *bytes*, and
is worth exactly as much as the review of the commit that introduced the line;
nothing here checks a key. A pin has to be maintained by hand, and that friction
is the feature — it is still friction. The minimum version gate is enforced from
v0.30.0 on, so it protects you from binaries newer than the gate itself, not
from every old one.

**Mutation reaches the compiler, not the disk.** A mutated file is shown to the
runner through the toolchain's overlay, so it covers source and anything read at
**build** time, embedded queries and migrations included. A file the program
opens at **run** time is untouched by it, and a text mutation aimed at one is
reported the same as any other survivor while never having been applied — that
case wants a copied tree, and no copy is made today. Two further bounds: an
identifier names a spot rather than one mutation, so forgiving it forgives every
mutation written there; and equivalent mutants (a break that cannot change any
observable behaviour) are counted as survivors, because telling one apart from a
missing test is undecidable in general and a reason in `allow` is the honest
place to say which it was.

**The cache is per file, not per project.** A checker whose answer depends on
more than one file at a time — `arch`, `freeze`, `docs`, `boxes` — does not use
it.

**Recording and replay.** Parallel replay is opt-in and all-or-nothing. A masked
value cannot be sent back, so a suite behind authentication replays as an
unauthenticated one unless a value is carried. Outbound recording is plain HTTP
only — `CONNECT` is refused rather than tunnelled. And a recording is only as
good as the traffic it saw.

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

**Examples are one call, not a scenario.** An example cannot expect a panic and
cannot read a value it mutated.

**Guards compare strings.** `http` guards are `GET` only, guards run one after
another, and the guard report is written once, after the command finishes. The
`sql` and `http` kinds are control-tested in Go rather than in the gate script,
which covers `exec` only. Effective checks compare strings too, never remember,
and `map` is a lookup table, not a rule.

**`x3 testdb` speaks PostgreSQL only.** Nothing prevents two runs from sharing a
template, and it keeps no record of its own beyond what it encodes in a name.

**The language gate speaks one language** — `en` is the only embedded dictionary.

<!-- x3-dist version=v0.62.0 capabilities=5d547e5d1468d10dcf3cde8e81331de887e52e005d41cf273ac2e67d2c9b1f06 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
