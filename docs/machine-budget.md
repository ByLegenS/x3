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
**An x3 started inside a step takes no slot:** `X3_CPU_HELD` tells it that its
parent holds the share, and it divides that share. If it asked for a slot, a
budget of one would make it wait for its own parent forever.

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

<!-- x3-dist version=v0.313.0 capabilities=17d7c952c83d131183aa1ec638aa096e3ac52de533b3d0cea3c70480d6a35004 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
