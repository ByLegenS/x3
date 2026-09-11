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
| `case control experiment` | an example that holds, one whose value is wrong, one with no payload, and one **nothing ran** — the last is why `never_ran` exists; plus an example using its file's imports `0`, and a tree with one broken example `1` **whose three sound neighbours still passed**; a declared import no example names `1` in the tree that declares it and `0` from a narrower scope — **same tree, same setting, only the scope moves**; and a red example whose code writes its own log, so the finding must still speak the gate's sentence; a file that does not parse `1` called `does_not_parse` and the same tree readable `0`; and a proposition its predecessor rules out `1` **whose two neighbours still ran** |
| `language gate` | the repository `0`; a planted word `1`; a green tree with its allow list `0` **and without it `1`** — an allow list never seen to change an answer is decoration |
| `docs gate` | this repository `0`; a rule whose counterpart directory cannot exist `1`; and on a planted repository whose single commit excuses one rule by name, that rule `0` while a second rule the reason does not name stays `1` |
| `roster control experiment` | one tree, seven readings, only the *question* changing: two configured checkers really called `0`; neither called `1`, the red naming **both** — a stale roster would have named neither. Then a script naming one of them **only in a help string**: the old reading `0` (the blindness), `strings: "exempt"` and `invocations` both `1`. Then a colon command genuinely called: the old reading `1` — a false red on a checker that runs — and `invocations` `0` |
| `configuration section registry` | a settings section the engine reads and the roster claims `0`; the roster gone stale `1`, naming the section. The green half on this repository's own tree is the `arch gate` step |
| `expectation scope control experiment` | an expectation naming a fixture tree `0` — the walk enters a skipped directory only because a rule declared it; the same tree with the expectation unnamed `1`, which is the blindness itself; the run started **inside** the named tree `0`; and the directive deleted `1` |
| `arch control experiment` | a green/red pair for every rule kind and every escape hatch: `absent` present, missing and dead; `skip` off, on and dead; `comments` read, exempt and embedded; `relativeTo` both ways; `minimum` met and short; `exclude` applying, dead and emptying the rule |
| `freeze control experiment` | the surface green, one name added red, and **`-update` on the grown tree red with the file unchanged** |
| `freeze count control experiment` | held, grown, shrunk and capped trees, each also under `-update`; the capped key stays out of the baseline **and the update itself exits `1`** |
| `surface control experiment` | six directions on one tree: **no baseline** (red, because growth is green here), recorded, untouched, a changed signature red **and naming its caller**, the same break allowed, a symbol *added* green, and the allow gone dead |
| `freeze scope control experiment` | one tree five ways; the exclusion written properly is `0` **on a tree built to be red without it**, and the same intent written as `"!…"` inside `sources` is `2` |
| `baseline control experiment` | no baseline, written, re-run, **the same debt moved down the file** (`0`), grown, `-update-baseline` refused, and one debt paid leaving `dead_baseline` |
| `baseline shrink control experiment` | one tree carrying **both** a paid debt and a new one: locked (`1`, `dead_baseline` named), then `-update-baseline` — the drop lands and the growth does not, both named, the file holding one entry; the re-run keeps the real finding (`1`) and has no `dead_baseline` left. Then a `count` that disagrees with its own list (`2`) |
| `secrets control experiment` | clean, leaky, exempted, a dead exemption, and this repository — the clean and exempted rows are what separate a gate from a noise generator |
| `secrets ignore control experiment` | a noisy tree with and without its exclusions, **then a real address added back**, then a dead exclusion, then a lookaround (`2`) |
| `comments control experiment` | inside the limit, one line over, exempted with a reason, an exemption that silences nothing |
| `boxes control experiment` | finished work left open, unfinished work closed, a condition unmet then met, two boxes closed with and without proof, and a `sql` criterion whose DSN is empty — counted, never green |
| `boxes document list control experiment` | the same for a document list, and **the same tree with one state declared `open` then `silent`** — one line of configuration decides, nothing else changes |
| `boxes criterion fidelity control experiment` | criteria that stopped measuring, each red with a control that removes the rule and returns the tree to green. Its fourth row is deliberately **green**: a test that was never written, measured by exit code alone, with no `output` |
| `boxes batch control experiment` | one tree measured twice, batched and not, three times over: a package whose criteria all hold, **a test that was never written beside two that pass**, and one criterion holding while another in the same package does not. The evidence is not the exit code but the **reports being identical** with and without batching, and the unmet criterion being the one the report names |
| `boxes unmeasured control experiment` | one tree of four closed boxes, run twice with one line of configuration between the runs: the box whose test skipped reads `box_unproven` without it and `box_unmeasured` with it — **still red** — while the box whose test failed and the box whose test was never written stay `box_unproven` either way, the box whose test passed says nothing either way, and both runs name the **same boxes**. A text written as the proof and as its absence is exit `2` |
| `arch produced path control experiment` | one tree run in **two machine states** — a clean checkout and one that has been built — against three settings. With no hatch the rule is red before the build and green after, which is the row that proves it measures at all; `absent` reverses that and is red on exactly one of the two machines; `skip` is green on both, while a pattern that sifts nothing is still red (`skip` dead, in the row above)
| `boxes measurement shell control experiment` | the same tree, the same version, **two shells**: without the variable a closed box is red (`1` finding), with it the run is green (`0`), and the neighbouring box - measured in both - is what proves the difference came from the shell and not from the tree. The report names the variable and whether it was set in each run, and `summary.blind` says `1 output`. A fourth row asks a variable **nobody declared**: a `sql` criterion names its own connection variable, so the shell line carries it and the cause reads `1 environment`. A declared variable with no name is `2` |
| `boxes dead selector control experiment` | one tree, four questions, and **not one criterion able to run**: the runner does not exist, so every closed box is `box_unmeasured` and the static answer is the only one there is. A selector no declaration matches is `box_dead_selector`; a selector whose name is declared **outside the place the criterion looks at** is the same code with the place it went to named; a selector still reached and an **open** box whose check is not written yet both say nothing. The same tree with the rule undeclared carries **zero** findings of this code - the red comes from the declaration, not the tree - and a tree in which no declaration can be read is `2` |
| `boxes holds control experiment` | one tree asked six ways, only the *question* moving: a file no criterion names `0` over a list that really carries criteria, a test only a **fragment selector** holds `1` while a search of the documents for its full name finds nothing, the same tree with no selector declared `0`; then a second tree where the criterion names a place — the same declaration outside that place `0`, inside it `1` — and the outside one `1` again once a gate script that calls it by name is declared |
| `boxes holds selector control experiment` | one tree, four questions, only the *declaration* and the *place* moving: a gate line handing its runner a **fragment** of the name holds the file `1` with `how: pattern`, the same name outside the place that line names `0`, the same tree with no reading declared `0` — the direction that proves the hold comes from the declaration and not from the tree — and a reading that reaches no line `2`. One row measures the fixture itself: the whole name is spelled on **zero** lines of that gate, so a word match could not have found it
| `boxes holds bindability control experiment` | one tree, one gate script, four questions: the gate names an exported package-level test `1`, carries an unexported name `0`, carries a method name `0`, and opens a file by its path `1`. Two rows measure the fixture itself — **both discarded words really are on a line of that gate** — so the green is a decision and not an empty search; two more read the report, where `names` and `bindable` show the narrowing |
| `configuration relative path control experiment` | one configuration, two working directories, one answer. A run records a baseline from one directory `1`, and the file lands **beside the configuration** and beside neither working directory; the same configuration read from a second directory finds it, `0`; a new debt is `1` from both. Bound to the shell instead, the first run leaves the file in the wrong tree and the second sees no baseline at all |
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
| `adoption section weight control experiment` | one tree, two configurations. A settings section and a directive-backed section under `block`, `0`; the same tree with one section declaring one rule, `1`. Then the report's own numbers, so the silence is measured and not accidental: the settings section **is named, has names inside it, and holds zero rules**; the directive section is weighed by the tree, not the configuration; a real rule list is still counted; and the red names the section and where its weight was read |
| `adoption policy control experiment` | one tree, eight settings. The first two are the measurement: the same tree, the same `block` default, `0` with the known code excepted and `1` without — the exception is the only thing that moved. Then the escapes: no reason, a bare word, an unknown code, a missing `"*"`, and `dead_policy` excepted from itself, each `2`; and an exception matching nothing, `1`. The last two rows read the report itself — the silenced finding is **still there, named, with its reason**, and the exception is counted against what it touched |
| `update control experiment` | installed, a planted checksum refused, the version gate both ways, and **a pin disagreeing with a release whose own checksum list is perfect** — which is exactly how a compromised release looks |
| `testdb control experiment` | a foreign name refused at the gate (`1`) **and our own name reaching an unreachable server (`2`)** — a gate that refused every name would also exit `1` |
| `expect control experiment` | the count met, one guard short, the directives deleted, and the same tree with no expectation |
| `public leak gate` | the published documents clean, and a planted tree in which **every** forbidden pattern speaks |
| `public size gate` | every published document under its own cap, then **each cap in turn** asked with one line too many — one document over the line would leave the other caps unmeasured |
| `public language gate` | the published documents in one language, and the template's **real** maintainer note planted; the plant is not invented text, so the experiment measures the assumption too — a note rewritten in English would leave the gate unable to prove itself, and it says so |
| `comments baseline identity control experiment` | one file, four steps: the debt recorded (`1`, one entry written), the same tree frozen (`0`), **a new over-long block in that same file `1`** — named, with the old debt still held — and the frozen block re-indented `0`, because identity is the text and not its shape |
| `split configuration roster control experiment` | one tree asked twice, only the rulebook moving: a rule and a section declared in an `include` **part**, the rulebook naming neither `1` — both named in the red — and the rulebook naming both `0`. The same run reports that the part's rule really ran, which is the whole point: a name the engine is enforcing cannot be missing from the set of names in force |
| `published control experiment` | one planted world — a bare remote and a repository that knows it — asked three times as the one missing step is taken: the tag never created `1`, the tag created and not pushed `1`, the tag pushed `0`; then this repository's **own** publication `0`. The middle row is the release that was really made and could not be downloaded |
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
docs: none (code-changes-carry-documentation) - <why the reader loses nothing>
```

A reasoned skip is written in the commit body; for the run before the commit
exists, pass the same line with `-reason`. The marker with nothing after it is
red, on purpose.

<!-- x3-dist version=v0.96.0 capabilities=890b3ee4cf9878440f2cd2d8516be453e9d75846ffce0092312b5bb1236761b4 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
