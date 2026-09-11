# What a mutation run breaks, and what it says

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 mutate`, what it breaks and what it says

The page before this one is the run: the two modes, the price, the overlay, and
which tests a mutation is measured against. This one is the vocabulary — what a
mutation *is* here, which codes stop a run, and what a finding says.

### What gets broken

| Operator | The change | What its survival means |
|---|---|---|
| `comparison` | `<`↔`<=`, `<`↔`>=`, `==`↔`!=` | the boundary is never tested at the boundary |
| `logic` | `&&`↔`\|\|` | one side of the condition is never exercised alone |
| `arithmetic` | `+`↔`-`, `*`↔`/`, `%`→`*`, and their assignments | the arithmetic is never checked against a known answer |
| `increment` | `++`↔`--` | the counter's value is never read back |
| `constant` | a number to `0`, and to `(n + 1)` | the number could be anything |
| `string` | a text to `""` (an empty one to a word) | the text is never compared |
| `bool` | `true`↔`false` | the flag is never observed |
| `condition` | `if c` to `if !(c)` | the branch is never taken both ways |
| `return` | a returned error to `nil` | the failure path is never asserted |

Three things are never touched, and none of it is a setting: **test files** (a
broken test proving a test broke measures nothing), **import paths and struct
tags** (text that carries no behaviour — breaking them yields a compile error,
not a finding), and **declared names**. `operators` narrows the list; an
operator named there that produces no mutant anywhere is `dead_operator`.

### Files no compiler reads — the database included

`text` mutations take a pattern and a replacement and apply to any file: a `SQL`
condition, a migration's `NOT NULL`, a routing table. The replacement uses the
regular expression's own expansion syntax (`$1`), so a condition can be carried
across the rewrite.

This reaches further than it looks. Where a query or a migration is **embedded**
into the binary at build time, the overlay reaches it too — the toolchain reads
the replacement, the embedded text changes, and the integration test that runs
against a real database sees the altered query. That is measured, not assumed:
in the control experiment, an embedded `SELECT … WHERE live AND ready LIMIT 10`
has its `AND` turned into `OR` and the test goes red; the `LIMIT 10` becomes
`LIMIT 0` and nothing notices, so it is reported as a survivor. **Both
directions, one tree.** A file read from disk at *run* time is a different case
and the overlay does not reach it; that bound is written in "Gaps we know about".

### Fail-closed: a scope that breaks nothing cannot be green

| Code | What happened |
|---|---|
| `empty_scope` | the **scope** produced not one mutant; the run proved nothing and is **red** |
| `planned` | the run was priced and not run; nothing was measured, so it cannot be green |
| `dead_exclusion` | an `exclude` pattern takes no file out of the scope |
| `dead_exemption` | an `allow` entry forgives no surviving mutant any more |
| `dead_operator` | a named operator produces no mutant anywhere in the tree |
| `baseline_red` | the untouched tree is not green; nothing is measured against it |
| `diff` | the change could not be read, so every file is mutated instead |

The first line is the whole law in one row. A misconfigured scope produces no
mutants, every mutant survives vacuously, and a gate that measured nothing would
print a perfect score — which is exactly the failure this command exists to
catch, committed by the command itself. `dead_exemption` and `dead_operator` are
measured in the **full** mode only: a narrow run not having reached a pattern
does not make it dead.

**A change that touches nothing is not that failure**, and the two have to be
told apart or the default mode can never sit in a gate: on a clean tree it would
be red every single run, and a gate that cries wolf is switched off by the
afternoon. The number that separates them is `inScope`, how many files the
configuration selects. None of them, and the scope is written wrong — **red**, in
either mode. Some of them, and the change simply reached none of them: the run
says *nothing in this change can be broken* and is green having claimed nothing.
The control experiment measures both, one after the other, in the same tree.

### The findings

| Code | What it says |
|---|---|
| `survivor` | the code was broken, the tests that could see it ran, and all stayed green |
| `no_test` | no test binary links the mutated unit; nothing could have seen it |
| `orphan` | the file belongs to no unit, so no test can even be named |
| `unmeasured` | the runner never started, so this mutant proves nothing either way |

A survivor may be forgiven — `allow` maps an identifier or a glob to a
**reason**, and a forgiveness without one is refused. The identifier names the
*spot* (`file:line:column:operator`), so forgiving it forgives every mutation
the engine writes there; that is deliberate, and it keeps the list short enough
to read. Forgiveness is spent: when the survivor is gone, the entry says so.

The report also counts what is **not** a finding, because those numbers are how
a score is read honestly: `caught`, `timedOut` (a run the mutation hung, counted
as caught and shown separately), `invalid` (the mutant does not compile — it is
discarded, never counted as caught, and kept apart from `unmeasured`, because a
tree whose runner cannot start would otherwise call every mutant invalid and go
green having measured nothing), `exempt`, and `runs`, the number of times
the runner was actually launched. `score` is caught over what was measured.

`invalid` is the one line in that list that must never bend. A mutation the
compiler refuses is not a behaviour anybody tested; counting it as caught would
raise the score without measuring one thing, which is the exact failure this
command exists to name. So a red run is never called caught on its own word —
the engine buys one compile pass and asks. The control experiment runs both
directions in a single tree: a package whose only mutation turns `one + two`
into `one - two` on two strings is reported `invalid`, produces no finding and
stays **out of the score's denominator**, while the package beside it produces
four real survivors, each named.

### The measurement

The control experiment, on a six-file tree, in **2.9 s**: 15 mutants, 21 runs.
A well-tested package gives 4 mutants and 4 catches; a package whose test calls
the function and asserts nothing gives 4 survivors; a package with no test
anywhere gives 4 `no_test` findings **and zero runs**; a package whose only
mutation cannot compile gives one `invalid` and no finding at all; the embedded
query gives one catch and one survivor.

On this engine's own source, one package of 143 lines produced **85 mutants** in
**68 s** (32 workers, 269 runs) and the answer was a real gap: that package has no
test file of its own, 21 mutants survived — a comparison boundary, a `+` turned
into a `-`, four swallowed errors — and the score was **0.738**. The finding is
not "coverage is low"; it is twenty-one namings, each a behaviour that can be
broken today with every gate still green.

Then a production Go repository, read-only, one package: **1,347 lines across
four files, carrying five test files**. Every file-name measure of coverage
calls that package tested.

| | |
|---|---|
| mutants | **380** |
| caught | 42 |
| **survived** | **256** |
| did not compile, discarded | 82 |
| runner launched | 679 times |
| wall clock, 32 workers | **135 s** |
| score | **0.141** |

The survivors are not a percentage, they are a list: 78 comparison boundaries,
64 constants that could be any number, **55 returned errors that could be `nil`
without one test noticing**, 26 texts never compared, 13 conditions where one
side is never exercised alone. Five test files stood over that package, and the
question "did you forget a test" had never had an answer before this run.

Two costs, said plainly. **21% of the mutants did not compile** — the generator
works from the syntax tree without types, so a `+` on two strings becomes a `-`
that no compiler accepts; those are discarded rather than counted, and the price
of not resolving types is paid in that fifth of the run — and paid twice over
since the compile pass moved to where it decides something: each of them now
costs two launches, the honest half of a change that takes a launch off every
mutant nothing caught. And the number that
matters for planning: **0.35 s of wall clock per mutant** on 32 processors,
which puts a repository of twenty thousand lines at a few hours — a night, not
a decision.

<!-- x3-dist version=v0.111.0 capabilities=f088e41a540b9aad743af8a7320405094bc2ed69445df36ccf6c69eaf20d959e template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
