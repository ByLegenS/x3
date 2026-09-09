# The test you forgot to write

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 mutate`

**What it catches:** the test nobody wrote. Every other measure of test coverage
answers a question next to the one that matters — *is there a test file?*, *was
this line executed?* — and a line can be executed by a test that asserts nothing
about it. `x3 mutate` breaks the code on purpose and asks the only question that
cannot be faked: **which test went red?** If none did, that behaviour is not
tested, and the command says so by name.

```
x3 mutate [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [-full] [-jobs <n>] [dir]
```

### Two modes, and only one of them is in a hurry

| Mode | Scope | For |
|---|---|---|
| default | the files the change touched | every commit; as short as the change is small |
| `-full` | every file in scope | called on purpose; **no ceiling, no sampling** |

The full mode has no limit of any kind — not a mutant budget, not a sample, not
a "that is enough looking". It runs until every mutation in scope has been tried
and prints the whole list of what nothing caught. It burns processors, not
attention: a run that takes all night costs the person who started it nothing.
The only ceiling anywhere is on a **single** run (`run.timeout`), and it has to
exist — a broken loop never returns, and without it one mutant would be the end
of the night.

### The audited tree is never written to

A mutation needs a changed source file, and the changed file lives in a
temporary directory. The runner is handed an **overlay** — a map from the real
path to the temporary one — so the compiler reads the broken file while the
working tree keeps the sound one. A run killed halfway leaves nothing behind to
clean up, and a gate that corrupted the tree it audits would lose the work in
the very moment the interruption happened.

```json
{ "mutate": {
    "sources": ["cmd/*/*.go", "internal/*/*.go"],
    "exclude": { "internal/*/testdata/**": "fixtures are inputs the gates read, not behaviour" },
    "text": [
      { "name": "sql-condition", "sources": ["**/*.sql"], "find": " AND ", "replace": " OR " },
      { "name": "sql-limit", "sources": ["**/*.sql"], "find": "LIMIT ([0-9]+)", "replace": "LIMIT 0" }
    ],
    "allow": { "internal/dump/*": "the dump format is compared against a golden file elsewhere" },
    "run": {
      "command": ["go", "test", "-count=1", "-overlay={overlay}"],
      "package": ["./{unit}"],
      "compile": ["-run", "x3_no_such_test"],
      "timeout": "120s" } } }
```

The engine knows no test runner here either. `{overlay}` is where the runner is
told about the replacement map, `{unit}` is one package on the command line, and
both must appear or the configuration is refused: a command that never reads the
overlay would run every mutant against the untouched tree and report a perfect
score. The unit graph is **not** written twice — `mutate` reads the same `test`
section (`units`, `tests`, `imports`, `module`), because the question *which
test can see this line* is the selective runner's graph read backwards.

### Which tests can see a mutation — and why that is the speed

Mutation testing is thousands of test runs, so the run that is never launched is
the only cheap one. For each mutant the engine already knows the answer: the
unit holding the mutated file, and every unit whose **test binary links** it.
Anything else on the tree cannot observe the change, and running it is pure
waste.

Three consequences, in the order they save time:

1. A mutant no test binary links at all is reported as `no_test` **without a
   single run**. This is the strongest form of the finding — not "no test
   caught it" but "no test could have".
2. The mutant's own unit runs **first**. It is the cheapest and by far the most
   likely to catch, so most mutants are answered by one run.
3. The first red **stops** the mutant. A second unit confirming the same verdict
   is the same answer bought twice.

Before any of it, the untouched tree runs once per unit. That pass does three
jobs: it proves the suite is green (a survivor measured against a red suite says
nothing, so `baseline_red` stops the run), it fills the toolchain's build cache
for everything that follows, and it **measures** each unit, so a slow unit gets
a proportionally longer ceiling instead of being called caught for being slow.

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
| `empty_scope` | not one mutant was produced; the run proved nothing and is **red** |
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

### The measurement

The control experiment, on a five-file tree, in **3.1 s**: 14 mutants, 23 runs.
A well-tested package gives 4 mutants and 4 catches; a package whose test calls
the function and asserts nothing gives 4 survivors; a package with no test
anywhere gives 4 `no_test` findings **and zero runs**; the embedded query gives
one catch and one survivor.

On this engine's own source, one package of 143 lines produced **85 mutants** in
**68 s** (32 jobs, 269 runs) and the answer was a real gap: that package has no
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
| wall clock, 32 jobs | **135 s** |
| score | **0.141** |

The survivors are not a percentage, they are a list: 78 comparison boundaries,
64 constants that could be any number, **55 returned errors that could be `nil`
without one test noticing**, 26 texts never compared, 13 conditions where one
side is never exercised alone. Five test files stood over that package, and the
question "did you forget a test" had never had an answer before this run.

Two costs, said plainly. **21% of the mutants did not compile** — the generator
works from the syntax tree without types, so a `+` on two strings becomes a `-`
that no compiler accepts; those are discarded rather than counted, and the price
of not resolving types is paid in that fifth of the run. And the number that
matters for planning: **0.35 s of wall clock per mutant** on 32 processors,
which puts a repository of twenty thousand lines at a few hours — a night, not
a decision.

<!-- x3-dist version=v0.62.0 capabilities=5d547e5d1468d10dcf3cde8e81331de887e52e005d41cf273ac2e67d2c9b1f06 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
