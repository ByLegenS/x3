# One processor budget per machine

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 machine`, one processor budget for every x3 on the machine

Each x3 process used to assume it had the machine to itself: the gate ran one
step per processor, each step got `processors / workers` cores, and the example
runner took half the processors. Three or four runs from different trees at once
asked for three or four times the processor, and a 16-core desktop sat at 96 °C
for half an hour. A limit inside one process cannot fix that: each process stays
within its own limit, and the total is still too high. So the budget belongs to the
**machine**, not to a project, and every x3 running at once shares it.

```text
x3 machine -cpu 50%          # or a core count: -cpu 16; -cpu off removes it
x3 machine -low              # below normal priority for x3 and what it starts
x3 machine                   # cpu budget 16 of 32 processor(s) (~/.x3cache/machine.yaml) ...
X3_CPU=4 x3 gate -only lint  # the environment overrides the file for one run
```

The file is `~/.x3cache/machine.yaml` (`cpu: 50%`, `low: true`); `X3_CPU` and
`X3_LOW` override it, and `X3_MACHINE_DIR` moves it. **With no budget written,
nothing changes:** no slot is taken, nothing is added to a child's environment,
and a run takes every processor as before.

**Slots, not a counter.** Every unit of parallel work — a gate step, a package
the example runner compiles and runs — takes one slot per core it will use
before it starts, and gives them back when it ends. A slot is a file locked with
`LockFileEx` on Windows and `flock` elsewhere, so **the operating system frees
it when the process dies**, however it dies. A named semaphore would keep the
count of a crashed process, and the budget would shrink for good. A unit takes
all its slots or none of them, so two waiting runs cannot each hold half of what
the other needs. A unit that has waited a second settles for the slots it can
get, and its core count becomes what it holds. Without that, a big unit could
wait forever behind small ones.

**What a child is told.** A unit's command gets `GOMAXPROCS`, `GOFLAGS=-p=<n>`
for the Go toolchain, and `X3_CPU_HELD=<n>`. **A variable the project declares
is never overwritten**, and an inherited `GOFLAGS` that already carries `-p`
keeps it. `{cores}` is cut from the budget, not from the processor count.
`X3_CPU_GOFLAGS` carries `GOFLAGS` as it was before x3 added its own `-p`, so a
nested x3 replaces only that `-p` and a project's own `-p` still stands.
**An x3 started inside a step does not wait for its parent:** `X3_CPU_HELD`
tells it that its parent holds the share, and that share is its **floor, not its
ceiling**. Its units run on the held share first; a unit beyond it borrows a
**free** slot of the machine budget, and if none is free it waits holding
nothing, so the held share always moves one unit forward and nothing deadlocks.
A borrowed slot is held for one unit only, so a step waiting in the gate waits at
most one unit. Before this, a gate of twelve workers on a budget of twelve gave
every step one core, and `x3 case` inside a step compiled its packages one after
another while the other steps had long finished and their slots stood empty:

```yaml
# x3/gate.yaml (no workers line: one core per step)
- name: inline examples
  run: x3 case -config x3.yaml
```

```text
# the same function-body change, measured on the same tree
x3 case                    18.2 s   (8 packages side by side)
X3_CPU_HELD=1 x3 case      39.5 s   (before: the 8 packages in a row)
X3_CPU_HELD=1 x3 case      21.2 s   (after: units borrow the idle slots)
```

**A borrowing unit asks for the share a direct run would get, and the run's
own threads borrow too.** The cores per unit are cut from the machine budget,
not from the held floor, and a unit gets the slots it finds free (at least the
floor). While `x3 case` weighs its packages, `machine.Grow` starts on the held
floor and adds one thread for every machine slot that frees up, without holding
anything while it waits, and gives the slots back when the weighing ends. The
package run first runs the package with the widest reach alone on the whole
share, so the importers of a changed body are compiled once into Go's cache
instead of once per package started side by side. Before this, the same change
ran its packages at `-p=1` and weighed them on one thread inside the gate, at
`-p=2` and on twelve threads outside it. The example keys of every changed
package go into ONE queue across packages, so a package with 1 775 examples is no
longer the single long path of the weighing (W585, VT gate, body edit in a core
package: weighing 6.3 -> 2.9 s on 4 threads, case step 24.0 -> 21.6 s; only the
schedule changed, keys are the same function of the same inputs). `X3_CACHE_WHY=1` shows both phases:

```text
x3 case: why .: weighed 119 package(s) in 3.8 s on 12 thread(s), ran 20 in 13.9 s
```

```text
# one function-body change in a core package, a new form each run, same tree
                                  before   after
x3 case (direct, under testdb)    20.3 s   19.6 s   (toolchain sum 55.5 -> 36.2 s)
gate -only "inline examples"      34.0 s   22.2 s
gate -region <core>               37.8 s   31.0 s
```

**What a run says.** A run that took slots ends with one line on stderr:

```text
x3: cpu budget 2 of 32 (X3_CPU) - shared by 2 x3 process(es) at most - 2 of 4 unit(s) waited 13.0 s in total for a slot
```

`X3_CPU_TRACE=<file>` appends every unit's slots and interval to a file;
`x3 machine -peak <file>` reads it back: `at most 2 slot(s) were held at once, by
2 unit(s); the trace names 2 process(es)`. That is how the control arms below
were measured.

**Priority** (`-low`, Windows `BELOW_NORMAL`, elsewhere `nice 10`; children
inherit it) keeps the desktop responsive. It does **not** cool the processor:
lower priority still uses idle cores at full speed. Only the budget lowers the
load.

| Arm | Measured |
|---|---|
| no budget, two gates of four 3 s steps | 3 s, no slot taken, no budget line |
| budget 2, the same two gates | 13 s; the trace peaks at 2 slots from 2 processes |
| budget 1, the holder killed mid-step | the waiting run starts within the poll interval and ends 6 s after it began (the killed step alone was 30 s) |
| budget 1, an x3 gate inside a step | finishes in 2 s; no deadlock |
| `X3_CPU=120%` | the run stops with exit 2; it does not fall back to every processor |
| x3 quick under a 50% budget (a 21-step Go band, 32 logical processors, results cache emptied, Go build cache warm) | 34 s on 16 workers, processor load 11.5% on average and 57% at the top; the trace peaks at 16 slots from 1 process; 0 red, exit 0 |
| the same band, no budget, same conditions | 32 s on 32 workers, load 16.1% on average and 70% at the top; 0 red, exit 0 |

Load is the whole machine's, sampled every 2 s. The top can pass the budget: a slot
bounds the units and `GOMAXPROCS`, not the threads a compiler or a database the
step talks to starts on its own.

**What it does not bound.** In-process walks (source reading, syntax, box
measurement) take no slot. They are capped by `GOMAXPROCS`, which is set to the
budget, and they are short. Slots are not taken in turn: a unit that is ready
when a slot frees takes it, whichever unit has waited longest.

<!-- x3-dist version=v0.331.0 capabilities=4dd95ef758f7d81516514d63de82e8b4bd7d4d60b318b986c8348e71d92492c2 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
