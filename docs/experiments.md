# Control experiments, and the documentation gate

[The pages](INDEX.md) - [what x3 is](../README.md)

## Control experiments

**No gate here is trusted because it is green.** Every capability has a pair in
`check.ps1` — one tree it must pass and one it must fail — and the pair is the
evidence, because a green nobody has seen fail proves only that nothing ran. The
step names below are the ones the gate prints.

| Step | The pair, and what only the red half proves |
|---|---|
| `control experiment` | a well-formed sample `0`, a broken one `1` |
| `case control experiment` | an example that holds, one whose value is wrong, one with no payload, and one **nothing ran** — the last is why `never_ran` exists; plus an example using its file's imports `0`, and a tree with one broken example `1` **whose three sound neighbours still passed** |
| `language gate` | the repository `0`; a planted word `1`; a green tree with its allow list `0` **and without it `1`** — an allow list never seen to change an answer is decoration |
| `docs gate` | this repository `0`; a rule whose counterpart directory cannot exist `1` |
| `arch control experiment` | a green/red pair for every rule kind and every escape hatch: `absent` present, missing and dead; `skip` off, on and dead; `comments` read, exempt and embedded; `relativeTo` both ways; `minimum` met and short; `exclude` applying, dead and emptying the rule |
| `freeze control experiment` | the surface green, one name added red, and **`-update` on the grown tree red with the file unchanged** |
| `freeze count control experiment` | held, grown, shrunk and capped trees, each also under `-update`; the capped key stays out of the baseline **and the update itself exits `1`** |
| `surface control experiment` | six directions on one tree: **no baseline** (red, because growth is green here), recorded, untouched, a changed signature red **and naming its caller**, the same break allowed, a symbol *added* green, and the allow gone dead |
| `freeze scope control experiment` | one tree five ways; the exclusion written properly is `0` **on a tree built to be red without it**, and the same intent written as `"!…"` inside `sources` is `2` |
| `baseline control experiment` | no baseline, written, re-run, **the same debt moved down the file** (`0`), grown, `-update-baseline` refused, and one debt paid leaving `dead_baseline` |
| `secrets control experiment` | clean, leaky, exempted, a dead exemption, and this repository — the clean and exempted rows are what separate a gate from a noise generator |
| `secrets ignore control experiment` | a noisy tree with and without its exclusions, **then a real address added back**, then a dead exclusion, then a lookaround (`2`) |
| `comments control experiment` | inside the limit, one line over, exempted with a reason, an exemption that silences nothing |
| `boxes control experiment` | finished work left open, unfinished work closed, a condition unmet then met, two boxes closed with and without proof, and a `sql` criterion whose DSN is empty — counted, never green |
| `boxes document list control experiment` | the same for a document list, and **the same tree with one state declared `open` then `silent`** — one line of configuration decides, nothing else changes |
| `boxes criterion fidelity control experiment` | criteria that stopped measuring, each red with a control that removes the rule and returns the tree to green. Its fourth row is deliberately **green**: a test that was never written, measured by exit code alone, with no `output` |
| `boxes batch control experiment` | one tree measured twice, batched and not, three times over: a package whose criteria all hold, **a test that was never written beside two that pass**, and one criterion holding while another in the same package does not. The evidence is not the exit code but the **reports being identical** with and without batching, and the unmet criterion being the one the report names |
| `syntax control experiment` | parsing, broken, a parser that is not installed (red), and the same check under `missing: "warn"` |
| `scope control experiment` | inside the lane, crossing it, crossing with a reason; then the branch form both ways, including a violation in the first commit under a clean one, and a closed lane |
| `test control experiment` | six directions on one tree, including **a full run when a file belongs to no unit** and a cache that answers, then measures again once the file changes |
| `record` / `replay control experiment` | a ledger whose credential header and planted key are hidden **while an ordinary field is still there**; then a replay without a `normalize` rule (red), with it (green), and against a drifted application (red) |
| `mutate control experiment` | one tree, nine directions: a well-tested package where every mutant is **caught**, a package whose test asserts nothing where every mutant **survives**, a package no test binary links (`no_test`, and **not one run launched**), an **embedded query** whose condition is caught while its `LIMIT` survives, the quick scope narrowed to the one changed file, a scope that produces **no mutant at all** (`empty_scope`, red — a gate that measured nothing would otherwise print a perfect score), a dead exclusion, survivors forgiven with a reason (green), and a forgiveness with nothing left to forgive (red); then the same scope **priced** — no launch, no green, and the real run's launch count lands between the two bounds — and the **worker ceiling** measured at half the processors |
| `outbound control experiment` | a call recorded through the proxy, the same answer served **with the far side shut down**, and an unrecorded call refused `502` |
| `guard control experiment` | green, blocked and warned — the launched command proves it ran by writing a file, and the blocked row proves it did not |
| `guard selection control experiment` | one file, only the flags changing; a mistyped tag exits `2` rather than skipping nothing |
| `multi-step trial control experiment` | the same trial green, red once an import is *written* into the copy, red on an empty removal — and **zero working areas left behind, the reds included** |
| `effective control experiment` | agreement, divergence under `block`, the same divergence under `warn` |
| `update control experiment` | installed, a planted checksum refused, the version gate both ways, and **a pin disagreeing with a release whose own checksum list is perfect** — which is exactly how a compromised release looks |
| `testdb control experiment` | a foreign name refused at the gate (`1`) **and our own name reaching an unreachable server (`2`)** — a gate that refused every name would also exit `1` |
| `expect control experiment` | the count met, one guard short, the directives deleted, and the same tree with no expectation |
| `public leak gate` | the published documents clean, and a planted tree in which **every** forbidden pattern speaks |
| `public size gate` | every published document under its own cap, then **each cap in turn** asked with one line too many — one document over the line would leave the other caps unmeasured |
| `public language gate` | the published documents in one language, and the template's **real** maintainer note planted; the plant is not invented text, so the experiment measures the assumption too — a note rewritten in English would leave the gate unable to prove itself, and it says so |
| `dist gate` | the publication current, the same question asked with a deliberately wrong document hash, and a copy of the publication with one page missing |

Whatever cannot be arranged from a shell — a database, a network, a fake driver,
a mapping, a retry — is control-tested in Go instead, to the same rule: each
green is shown next to the red that proves it was measured.

## The documentation gate

This repository holds itself to the rule it ships: a change under `internal/` or
`cmd/` must carry a change under `docs/` in the same diff. The gate is
[`x3 docs`](docs.md#x3-docs) reading this repository's own `x3.json` — the same command
any project would run.

```
docs: none - <why the reader loses nothing>
```

A reasoned skip is written in the commit body; for the run before the commit
exists, pass the same line with `-reason`. The marker with nothing after it is
red, on purpose.

<!-- x3-dist version=v0.65.0 capabilities=30f2211593ea62df95d9a529b650866118e447096978014873bc8ee488525447 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
