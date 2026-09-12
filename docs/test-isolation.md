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

The engine's run layer does not know what a database is. It calls a preparer
before each unit and a closer after it, on **every** exit path - a database
created and left behind outlives the run that left it. What that preparer sets
up is the `testdb` section's business, and a run layer that knew about databases
would be dead weight in every project that has none.

<!-- x3-dist version=v0.131.0 capabilities=0ae611848b4d163cdcc7ff33b586ecfda3da08f1e6cf326dffcd8d785144d8ea template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
