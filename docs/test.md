# Only the tests a change can reach

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 test`

**What it catches:** a suite that grows with the repository until nobody runs
it. `x3 test` hands the runner only the units a change can reach, so the cost of
a run follows the **change**, not the size of the tree.

```
x3 test [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [-scope auto|working|head] [-reason <text>] [-full] [dir]
```

The engine knows no test runner: the command, the way a unit is written on the
command line, and the way a dependency is read all come from the configuration.

```json
{ "test": {
    "units": ["cmd/*/*.go", "internal/*/*.go"],
    "tests": ["**/*_test.go"],
    "ignore": { "docs/**": "the published documentation is compiled into nothing a test exercises" },
    "module": "x3",
    "imports": { "from": "regex", "select": "^\\s*(?:[\\w.]+\\s+)?\"(x3/[^\"]+)\"",
                 "comments": "exempt" },
    "run": { "command": ["go", "test"], "package": ["./{unit}"], "all": ["./..."] } } }
```

`units` declares which files **define** a unit; the unit is the directory
holding them. `package` is that unit's argument list — a list, because one runner
takes `./pkg` in one word and another takes `-p pkg` in two — and `{unit}` must
appear in it, or every unit would render the same argument and the selection
would be a lie. `imports` is the ordinary extractor with one capture group;
`module` is the prefix meaning *this repository*, and anything outside it never
enters the graph. `comments: "exempt"` earns its line: an import inside a
commented-out block never runs, and an edge drawn from it would drag a unit into
every run for nothing.

### A test's import does not travel

A test file's dependency belongs to **that unit alone**: a test binary links it,
the package does not, so an importer of that package never sees it.

This is not a detail. In the pilot, `database`'s test imports a shared helper,
that helper imports the engine package, and `ledger` imports `database`. With
test edges travelling, touching one leaf reached **54 of 85 packages**; with
`tests` declared, **17** — and the 37 others had no path to the change at all.

A `tests` pattern that matches nothing is *not* red, and the asymmetry is
deliberate: a dead `ignore` makes a run **narrower** than the tree justifies, a
dead `tests` pattern only makes it **wider**. The dead-escape-hatch law guards
the direction that can hide a failure.

### Which unit a file belongs to

| The file | Its unit |
|---|---|
| matches `units` | its own directory |
| does not match `units` | the nearest unit **above** it |
| has no unit above it | none: the run goes **full** |

The search upward never reaches the repository root. If it did, the root would
be every file's ancestor, and a change to a README would narrow the whole tree
down to one package — the quietest possible narrowing, exactly where we set out
to prevent it.

### Fail-closed: when the run goes full, it says so

| Reason | What happened |
|---|---|
| `orphan` | a changed file is in no unit and no `ignore` covers it |
| `graph` | a dependency resolves inside `module` but to no unit |
| `diff` | the change could not be read from git |
| `units` | the `units` patterns match no file |
| `forced` | `-full` was written |

A full run is not a fault and does not turn the command red — the engine says it
could not narrow, and does the work anyway. What *is* red is a dead `ignore`
pattern or a dead `skip` inside `imports`. `ignore` is measured against the whole
tracked tree, not this run's changed files: not having been touched today does
not make a pattern dead. Every entry carries a reason, because "this path cannot
change behavior" is a claim and a claim wants an owner.

### The cache

Each unit's result is stored under a digest of **everything its test binary
links** — its own files, everything it imports transitively, and what its test
files import. Selection and digest read the same set, so a unit that was not run
can never be sitting on a stale green.

**Only green is stored.** Which unit failed inside a batched run can only be read
out of the runner's output, and that output is specific to one language; a red
result simply runs again, which is what happens anyway while it is being fixed.
A **full** run does not consult the cache at all: the reason it went full is that
the effect of the change could not be computed, and what cannot be computed
cannot be looked up. The engine never adds `-count=1`, which would defeat the
runner's own cache.

### A monorepo with more than one module

```json
"run": { "dir": "backend", "command": ["go", "test"], "package": ["./{unit}"], "all": ["./..."] }
```

`run.dir` launches the runner somewhere other than the run root. Units still
carry their repository-relative names (`backend/internal/auth`), so the changed
set and the graph stay in one coordinate system and only the command line is
rewritten. Every module gets its own `test` section.

### Which unit is slow, and why

The engine cannot time a unit with its own clock: the units share one runner
process, and a process per unit would manufacture the cost it set out to
measure. The runner reports the number, the configuration reads it.

```json
"timing": { "pattern": "^(?:ok|FAIL)\\s+(\\S+)\\s+([0-9.]+)s", "name": 1, "took": 2 }
```

`name` and `took` are capture groups — the unit and its duration; `in` reads `ms`
instead of seconds, `slowest` (default 5) is how many units get named. The name
is read as a dependency key first, so `module` strips the prefix and no second
mapping table is born. **With no `timing` block nothing is timed and the report
stays byte-for-byte what it was.** A `timing` block that matches nothing is red
(`dead_timing`): a dead reader, judged like a dead exemption.

The report then carries `wall_ms` (the engine's own clock), `units_ms`, `cores`,
`overlap` (`units_ms / wall_ms`: how many units ran at once), the slowest units
by name, the units that reported no time, and one `cause`: **`one_unit`** — one
unit is over half the wall clock and *is* the run; **`outside`** — under half the
wall clock is inside any unit, so the cost is build, link and process start, not
the tests; **`overlapped`** — the runner ran units at once; **`spread`** — the
units account for the time and none dominates. Whether a unit *waits* or
*computes* is **not** measured: that needs the processor time of the whole tree,
and a child's accounting excludes its own children — measured, a direct child
reported 0.97 of its wall clock, one level deeper 0.01.

### The measurement — and what it honestly shows

Pilot: a production Go repository, one module of 85 packages, read-only. One
leaf package touched; 17 of 85 affected.

| Case | the runner alone | `x3 test` | ratio |
|---|---|---|---|
| cold build cache (a fresh CI machine) | 28.7 s | 24.0 s | 1.2× |
| warm cache, one package touched | 12.3 s | 12.1 s | 1.0× |
| nothing changed since the last run | 4.15 s | **0.34 s** | **12×** |
| everything, `-count=1` | 17.8 s | same under `-full` | 1.0× |

**Read the table honestly.** For Go, `go test ./...` is *already* incremental at
the same granularity this command selects at, so narrowing the package list buys
almost nothing on top of it. The one place it wins outright is the run where
nothing changed: the runner still walks all 85 packages to decide it has nothing
to do, and that walk grows with the repository. For a runner **without** a cache
of its own — most of them — the first three rows would look very different. The
engine does not assume either case; it measures.

<!-- x3-dist version=v0.91.0 capabilities=42fdb478204aa7bb7bc7b10d4593e3f217343dcde7c7797089bda96e8b263f48 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
