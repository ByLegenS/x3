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
x3 mutate [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [-full] [-workers <n>] [-plan] [dir]
```

### Two modes, and only one of them is in a hurry

| Mode | Scope | For |
|---|---|---|
| default | the files the change touched | every commit; as short as the change is small |
| `-full` | every file in scope | called on purpose; **no ceiling, no sampling** |

The full mode has no ceiling **on scope** — not a mutant budget, not a sample,
not a "that is enough looking". It runs until every mutation in scope has been
tried and prints the whole list of what nothing caught.

### Time and heat are not the same permission, and the price is said first

A run that takes all night spends hours the machine was idle anyway. A run that
fills every processor spends the machine. Measured: the same scope, the same 379
launches, at **8 / 16 / 32** workers took **48.4 s / 41.0 s / 41.1 s** — above
half the processors the extra width buys **nothing**, because the toolchain
already parallelises inside each run, while at full width the machine cannot be
worked on (55 toolchain processes, against a peak of 26 at 16 workers). So
`workers` defaults to **half the processors**, never below one, in the code
rather than in a note; `-workers <n>` or `{ "mutate": { "workers": 8 } }` moves
it deliberately, and the control experiment asserts the default is half and that
asking for one gets one. A ceiling that lives in a plea is not a ceiling.

`-plan` then answers *what would this cost* without launching anything: how many
mutants, how many are answered with no run at all, and two **bounds** on the
launches — `least`, units + mutants, every mutant answered by a single run that
stayed green; `most`, units + one **two-or-watchers** per mutant, whichever is
larger, because a mutant that goes red costs its run plus the one compile pass
that tells *caught* from *never built*. A priced
run prints `PRICED`, launches nothing and **cannot be green**: it reports
`planned` and exits 1, because a command that measured nothing must not read
like one that measured everything — this gate's own subject, applied to itself.
The control experiment prices the control tree, runs it, and asserts the real
count landed between the bounds (15 <= 21 <= 26).

On this engine's own source: **10,601 mutants** across 57 files, 9,061 running,
1,540 answered with no run at all, 26 units in the baseline, **9,087 to 30,307
launches** at 16 workers — at the measured 0.108 s per launch, between a quarter
of an hour and an hour. That is the number a person needs *before* deciding, and
until now it could only be learnt by spending it.

The only other ceiling is on a **single** run (`run.timeout`), and it has to
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

Measured on this engine's own source, that selection is already narrow: of the
9,049 mutants that run, **7,025 have exactly one watcher** and the widest has
24, for 2.6 on average. Going finer would mean asking which *functions* a test
reaches rather than which packages; the sound half of that was built, measured
and removed — a declaration no exported symbol of its package can reach is
unreachable from outside, and this tree has **not one**, because an unexported
helper exists in order to be called by an exported one. The unsound half is
refused: `fmt` calls a `String()` without ever writing the name, and a watcher
dropped by mistake would have the gate call a tested behaviour untested.

Three consequences, in the order they save time:

1. A mutant no test binary links at all is reported as `no_test` **without a
   single run**. This is the strongest form of the finding — not "no test
   caught it" but "no test could have".
2. The mutant's own unit runs **first**. It is the cheapest and by far the most
   likely to catch, so most mutants are answered by one run.
3. The first red **stops** the mutant. A second unit confirming the same verdict
   is the same answer bought twice.
4. The compile pass is bought **only where it decides something**. A run that
   passed has already proved the mutant builds, and asking the compiler again is
   the same answer bought twice; only a mutant whose *first* run went red is
   ambiguous — caught, or never built — and that difference cannot be read from
   an exit code, so it is bought exactly there, at most once per mutant.

Measured on this engine's own `comments` package, eight workers, the same tree:
**305 launches in 36.8 s** where the compile-first order took **379 in 46.0 s** —
a fifth of the run gone, and the verdict unchanged to the last count (196
mutants, 94 caught, 88 survived, 14 discarded, score 0.516). The launch itself
did not get cheaper; there is simply one less of it for every mutant that
survives. The cost is in the other direction: a mutant that never builds now
pays two launches instead of one, and that price is named in the measurement on
the next page.

Before any of it, the untouched tree runs once per unit. That pass does three
jobs: it proves the suite is green (a survivor measured against a red suite says
nothing, so `baseline_red` stops the run), it fills the toolchain's build cache
for everything that follows, and it **measures** each unit, so a slow unit gets
a proportionally longer ceiling instead of being called caught for being slow.

<!-- x3-dist version=v0.107.1 capabilities=a33d0a24b9931851dd74be101512bd0389f8cc3b6ca762f0a329588aad1acd05 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
