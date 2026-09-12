# Gaps in what a run reaches

[The pages](INDEX.md) - [what x3 is](../README.md)

## Gaps in what a run reaches

"Gaps we know about" bounds what the engine can READ in a tree. These bound what
a run can REACH outside one: the release it pulls, the toolchain it mutates
through, the traffic it records, the live world it asks, and the database it
borrows.

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
observable behavior) are counted as survivors, because telling one apart from a
missing test is undecidable in general and a reason in `allow` is the honest
place to say which it was. And **which tests can see a mutation is answered per
package, not per function** — a mutant runs against every unit whose test binary
links its own (2.6 on average, measured) rather than the units that actually
reach the mutated function; the narrower answer wants a call graph, because
`fmt` calls a `String()` without writing the name and a watcher dropped by
mistake would call a tested behavior untested. The floor is elsewhere anyway:
every mutant pays a compile pass *and* a test run, and a runner able to report a
build failure in a form the configuration declares would remove half the
launches. Nothing declares one today.

**Recording and replay.** Parallel replay is opt-in and all-or-nothing. A masked
value cannot be sent back, so a suite behind authentication replays as an
unauthenticated one unless a value is carried. Outbound recording is plain HTTP
only — `CONNECT` is refused rather than tunnelled. And a recording is only as
good as the traffic it saw.

**Guards compare strings, and one set.** `http` guards are `GET` only, guards
run one after another, and the guard report is written once, after the command
finishes. `sameRowsAs` reads whole row sets, but only from `sql`, only one
column wide, and only into memory — two schemas that do not fit are not
streamed. The comparison is control-tested against a real server **when the
gate is given one**; the `http` kind is control-tested in Go rather than in the
gate script. Effective checks compare strings too, never remember, and `map` is
a lookup table, not a rule.

**`x3 testdb` speaks PostgreSQL only.** Nothing prevents two runs from sharing a
template, and it keeps no record of its own beyond what it encodes in a name.

<!-- x3-dist version=v0.137.0 capabilities=ffea64aad496c0c9166d36c68ab6bb408247d2a234a93d00c5f07efb99ce7f94 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
