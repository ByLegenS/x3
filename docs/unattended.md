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
stdin) and **all four waited forever**.

What the engine takes is the **screen**, not the console: on Windows the child
gets a console with **no window** (`GetConsoleWindow` returns nothing though
`CONOUT$` still opens), on Unix no terminal at all. ⛔ Removing it outright
(`DETACHED_PROCESS`) was **reverted** — grandchildren inherit none either, so
Windows gives each a NEW one and a compiler filled a desktop with windows.

| | |
|---|---|
| the child is **measured** | on Windows its console has no window; on Unix there is no terminal at all |
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

### And it cannot wait forever either

A run with no ceiling cannot even go red, because it never ends. Most surfaces
declare one, because they know the size of their work — a guard's read is
seconds, a whole suite is minutes. Three declared nothing, and a command
started from them ran for as long as it felt like: the external parsers behind
`syntax`, the `git` reads behind `docs`, `scope` and `published`, and anything
reaching the engine through a context that carries no deadline.

The engine's ceiling is **60 s**, and it is not a new number: two surfaces
picked it independently, for two different jobs, before this was written. It
**fills in, it never overrides** — a surface that declares its own size keeps
it; a `syntax` check writes `timeoutMs`; a project writes its own with
`{ "x3": { "commandTimeoutMs": 120000 } }`.

A parser that runs out of time is `parser_failed`, not `does_not_parse`: the
file is not at fault, and that code cannot be frozen into a baseline. The
**attended** surfaces take no ceiling at all — how long the command a person
is watching may run is that person's business.

⚠️ **What this does not buy.** Severing the console also removes the child from
the console's `Ctrl-C` group. A measured child is bounded by its timeout and is
killed when the run's context ends, but a parent killed outright still leaves
it behind — on Windows that was already true before, since no parent takes its
children down with it. What happens to those survivors is the next page.

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
