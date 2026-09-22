# One call per unit, one database per unit

[The pages](INDEX.md) - [what x3 is](../README.md)

## Isolating a unit's run

**What it catches:** a suite that is green because its units share a
process and a database. A seed one unit plants covers another unit's
blindness, and the order that decides it is a race.

### One call for all units, or one call per unit

By default the runner is called **once** and every unit it must measure is an
argument of that one call. That is the fast shape: the runner starts once, and a
runner that parallelises packages does so freely.

`-per-unit` calls it **once per unit** instead. It buys two things.

| | one call | one call per unit |
|---|---|---|
| the unit that failed | unknown - only the runner's own output says, and that output is language-specific | **named**: `x3 test: <unit> is red`, and `broke` in the JSON |
| what a unit leaves behind | reaches every unit after it in the same process and the same external state | stays in that unit's run |
| a red run's cache | nothing is stored: the green units have no names | every unit that stayed green is stored, so the next run only repeats the red ones |
| cost | one process | one process per unit, and packages no longer overlap |

Measured on this engine's own tree, 32 units, warm runner cache: **9.5s** for one
call, **23.8s** per unit - 2.5x, and the whole difference is process startup and
lost overlap (0.77x overlap became 0.07x). The mode is not the default for that
reason; it is what you reach for when a suite is green and you do not trust it.

A full run is per-unit too in this mode, and `run.all` is not used - that
argument is the argument of the **one** call.

**The units run in parallel.** Isolation is not a reason to run in single file:
the units are independent, which is why the mode exists at all. Measured in a
production application: 61 units with a database each took **183 s in single
file** while the work itself was 32 s — the rest was every unit waiting for its
own database (0.18x overlap). Run in parallel the same suite took **56 s** (0.84x
overlap). Output is written **in unit order, not finish order**: interleaved
lines from parallel runs leave the reader unable to tell which line belongs to
which unit, and the timing parser reads that same stream. The worker count is
`GOMAXPROCS`, because this run may itself be a step inside a gate that is already
running its steps in parallel.

### A database per unit

`-fresh-db` gives every unit a database of its own, created before its run and
dropped after it, from the `testdb` section (§ `x3 testdb`). It implies
`-per-unit`, because a fresh database per unit is meaningless inside a single
call that shares one.

**Why it exists.** The rule is that a measurement is made in a *fresh* database
**per package**; when the whole tree shares one, a seed planted by one package
covers another package's blindness. The pilot measured exactly that: one
package's end-to-end test writes four rows into a catalogue table, another
package's test reads the whole catalogue and breaks - **7 red in 25 runs** when
aimed at it, and **not raised at all** in the full suite, because the order is a
race. A green suite in that state is not evidence.

The cost is the cost of `-per-unit` plus one database setup per unit: the
`testdb` section clones a template for exactly this reason, and a clone is the
difference between "a second per package" and "a migration per package".

⛔ **The unit is handed the same variables `testdb run` hands a command** — the
`dsnEnv` and the project's `runEnv` mapping, `x3:dsn` resolved. Measured (pilot,
2026-09-15) when it was not: `-fresh-db` built a fresh database per unit and
passed only `dsnEnv`, while the project's tests read the variable named in
`runEnv`. Every unit ran against the **development database** instead, and a
package that is red in five runs out of five showed up green. A mode whose whole
promise is isolation had none, and said nothing.

### Only the units that need a database

```json
{ "test": { "database": { "reaches": ["core/storage"] } } }
```

A database per unit is not free: measured in a production application, the same
suite took **22.7 s with no database and 65 s with one per unit** — 42 seconds of
setup. `database.reaches` names the units whose reach means "this one talks to a
database"; every unit that reaches one of them is prepared, and the rest are not.

⛔ **A unit that is not prepared is handed those variables EMPTY, not left
alone.** The caller's environment may already carry a DSN — a gate step declares
one — and an unprepared unit that sees it writes to the *development* database,
silently, because it stays green. An empty variable says so on the first line.

Declared narrowly it can cost more than it saves: in that same application a
two-root list pushed the suite to 86 s and left three units red for want of a
DSN, and widening the roots to the truth brought it back to 66 s — the width of
the unnarrowed run. Measure before writing it.

The engine's run layer does not know what a database is. It calls a preparer
before each unit and a closer after it, on **every** exit path - a database
created and left behind outlives the run that left it. What that preparer sets
up is the `testdb` section's business, and a run layer that knew about databases
would be dead weight in every project that has none.

<!-- x3-dist version=v0.288.0 capabilities=b7fca59edbf65759483bfdca34f14aeafbe84562986ae2f4e8a4b427249da8fb template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
