# A command started to measure cannot wait for a person

[The pages](INDEX.md) - [what x3 is](../README.md)

## A command started to measure cannot wait for a person

**What it catches:** a gate that never ends. A command started to *measure*
something has no audience. If it stops to ask a question, nobody types the
answer, and the run waits for as long as the machine stays up. This was
measured the expensive way: waiting children piled up on one desktop until the
database server could no longer start, twice.

⛔ **The question is not asked on stdin**, and that is what makes the obvious
fix useless. A password prompt is written to the **console** — `CONOUT$` on
Windows, the controlling terminal on Unix. Measured: four bindings of the
child's stdin (unset, an empty reader, the null device, the caller's own
stdin) and **all four waited forever**. Cut the console instead and the same
command exits in **6-7 ms**, with the reason in the output the report already
captures: `no console to write to`.

| | |
|---|---|
| the child is **measured** | it starts with no console at all: `DETACHED_PROCESS` on Windows, a session of its own on Unix |
| its stdin is **not written** | an unwritten stdin is the null device already, so a reader sees `EOF` at once; the engine does not touch it |
| the caller **feeds** it | the pipe the caller wrote stays exactly as written — handing a parser its input on stdin is a legitimate way to run one, and this rule does not close it |

**A streamed run is left alone.** Where a command's output is poured into the
terminal — `x3 guard -- <command>`, `x3 testdb run -- <command>`, the runner
behind `x3 test` — somebody is watching it. Taking that person's console away
would not rescue a run stuck on a prompt; it would only cut `Ctrl-C` off from
the child and leave the process behind. So the split is not "safe and unsafe",
it is **measured and attended**:

| Surface | How it starts |
|---|---|
| `command` criteria, `guard` checks and their trial steps, `syntax` external parsers, `testdb` setup steps, the mutation and example runners, `git` reads | measured — no console |
| `x3 guard -- <command>`, `x3 testdb run -- <command>`, the streamed test runner | attended — the console is inherited, and so is `Ctrl-C` |

⚠️ **What this does not buy.** Severing the console also removes the child from
the console's `Ctrl-C` group. A measured child is bounded by its timeout and is
killed when the run's context ends, but a parent killed outright still leaves
it behind — on Windows that was already true before, since no parent takes its
children down with it.

<!-- x3-dist version=v0.144.0 capabilities=c7a898f2ed45fa4107ced156c2151290063596e5205ab3e9fac74e3314f2808e template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
