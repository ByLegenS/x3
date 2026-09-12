# A run leaves nothing behind

[The pages](INDEX.md) - [what x3 is](../README.md)

## A run leaves nothing behind

**What it catches:** the process a run started and walked away from. A command
that times out is killed, but nothing it started is; a runner that forks a
server and exits leaves the server. One desktop was measured under this:
**54 windows and 115 console processes** piled up, and the database server then
failed to start twice with `0xC0000142` — the machine had run out of room. The
rule that followed — *you kill what you start* — had no gate, and a rule with no
gate is a request.

Every run now answers one question before it exits: **are any of the processes
I started still alive?**

### The container, not the name

⛔ **Nothing is matched by name.** A gate that looked for a process called
`node` would find the developer's own dev server and stop the build for it.
What a run started is not a guess, it is **membership**: on Windows the engine
puts *itself* into a job object as its first act, and Windows makes membership
**inherited and inescapable** — every child, every grandchild, everything they
start in turn is in that container, and none of them can leave it.

| | |
|---|---|
| started **before** the run | never in the container — a panel, an API, a worker already up is not the run's business, and is measured to survive it untouched |
| started **by** the run | in the container, through any surface: a `command` criterion, a guard, a setup step, an external parser, the wrapped command, `git` |
| started by a **grandchild** | in the container too; this is the half a flag alone gets wrong, and it is where the console hosts of severed children show up |

Membership is not enough on its own: a process in the list is opened and asked
whether it is still running, and a list read once is read **again after 250 ms**.
Between a command's exit and the run's, a console host or a compiler is often
mid-shutdown, and counting that instant would be a red with nothing behind it.
Measured: the same clean run repeated **ten times, exit `0` ten times.**

### The verdict, and the one way out

A run that leaves a process behind **cannot be green**. Exit `0` becomes `1`;
any other code is left alone, because a run already red is explained by its
first reason and a wrapped command's own code is that command's word. The
survivors are named on **stderr** — number and program — so reports on stdout
are byte for byte what they were.

A project that deliberately starts something meant to outlive the run says so,
and says **why**:

```json
{ "x3": { "leaves": "the setup step starts the local server the guards read" } }
```

The run still names what it left; it just stays green. Written with no reason it
is exit `2` — a permission whose reason is nowhere is a gate switched off in
silence.

⛔ **Nothing is killed.** The audit knows one thing, that a process came from
this run; it does not know whether someone wanted it. Killing on that would be
irreversible the first time it was wrong. The number is in the report, and the
reader decides.

⚠️ **Measured on Windows only.** There is no job object on Unix, and the session
a severed child gets can be broken by a grandchild that opens its own. Where the
count cannot be trusted the engine says nothing rather than print a green it did
not measure — and a line repeated on every run is the first line anyone stops
reading. The gap is named on the gaps page.

<!-- x3-dist version=v0.156.0 capabilities=c8bb02269798cd209388b465a9941635adeb7d3cc673c6643bd116b9a446af22 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
