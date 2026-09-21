# Running the work itself

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 do`

```
x3 do <task> [-config <file>] [-root <dir>] [-with <flags>] [-set <name=value>] [-dry] [-out <file>]
```

**Catches:** the other half of the shell scripts. `x3 gate` took the ones that
**measure**; these are the ones that **act** — build, start the local services,
deploy to a server, prune the logs. Measured in a production repository: seven
scripts, 2 199 lines, and every one of them reimplementing the same four things —
run a command and check its exit, start a process without hanging on its output,
knock on an address until it answers, stop when something goes red.

A task is a list of steps, run **in order**, stopping at the first red. Order is
real here, unlike in the gate: step 3 runs the file step 2 sent. And a stop is
real too — a step that runs on top of a half-finished deployment moves the fault
away from its cause.

```yaml
do:
  stamp:
    format: "v{major}.{kloc}.{commits:0000}{dirty}"
    major: file:VERSION
    cache: _build/.kloc
  hosts:
    production:
      ssh: root@198.51.100.7
      port: 44
      key: ${USERPROFILE}/.ssh/deploy
      log: _build/remote.log
  tasks:
    - name: build
      say: a versioned build
      flags:
        - name: all
          say: the admin tools too
      steps:
        - say: the version this build carries
          set:
            version: stamp
        - say: panel
          run: go build -ldflags "-X main.version={version}" -o dist/panel ./cmd/panel
          env: { GOWORK: "off" }
        - say: admin
          when: all
          run: go build -o dist/admin ./cmd/admin
```

### One step does one thing

A step carries exactly one verb, and the verbs are:

| Verb | What it does |
|---|---|
| `run` | a command; `on: <host>` sends it to a machine instead |
| `send` | a local file to a host (`to:` names where) |
| `start` | a detached process; `log:` is where its output goes |
| `halt` | stops processes by name, and **counts** the ones that stopped |
| `alive` | measures that those processes are **up** — a start is not a measurement |
| `wait` | knocks on a URL or `host:port` until it answers |
| `prune` | trims a directory: `drop` goes entirely, `groups` keep their last `keeps` |
| `scan` | looks for files in directories: name matching `match`, body matching `find`, and **not** named in `except`. Exit 0 none, 1 some — so `want` asks either "is this here" or "is this gone" |
| `set` | a value — `stamp`, `file:<path>`, `line:<file>:<pattern>`, `env:<NAME>`, `now:<layout>` |
| `done` | ends the task **green**, here. A mode has its own end ("just stop the services"), and writing that end as a condition on every later step is a condition somebody forgets when they add the next one |
| `fail` | stops, and says why |
| `show` | prints a **section of a document**: `from`/`until` are patterns, `max` a ceiling (40 by default). A `from` that matches nothing is **red** — a section that was renamed would otherwise print as an empty one, and an empty urgent-work list reads as "nothing is urgent" |
| `block` | a group of steps in **one** ssh session on `on:` — each inner step still named, weighed and classified on its own |

`show` exists because the thing a session or a deployment has to read at its
start is usually **already written** in a document — the urgent list, the
deployment window, today's count. Every script that prints those is a second
copy of the document's structure, and it goes quiet the day a heading changes.
Measured in the pilot: a 212-line session-brief script, a third of it repeating
warnings that also lived in the project's own status file.

Two verbs in one step is a settings error, not a convenience: which of them ran
first cannot be read off the file, and which of them failed cannot be read off
the report. A step with no verb is worse — it shows up as "ran".

### `block` — many remote steps, one session

`block` comes from a measurement, not a preference. Every `on:` step is a
**separate `ssh` process**, and one handshake costs **591 ms** (measured
2026-09-14 against the pilot's production host: three sequential calls, 1 773 ms
total). A deployment chain written as forty `on:` steps spends 24 seconds
waiting — and, worse, opens forty separate connections, each its own chance to
drop mid-chain and leave the server half-updated.

Connection multiplexing is not the answer: Windows' OpenSSH client does **not**
support `ControlMaster` — measured, `getsockname failed: Not a socket`, and the
connection dies, while the control run without multiplexing is green. The fix
had to be portable, so it lives in the engine rather than in an ssh option.

```yaml
- say: update chain
  on: production
  write: true
  block:
    - say: services stop
      run: systemctl stop app-api app-worker
    - say: binaries move into place
      run: mv /opt/app/bin/server.new /opt/app/bin/server
    - say: health
      run: curl -fsS http://127.0.0.1:8080/
```

**A block is not a script**, and the difference is exactly where the engine keeps
looking:

- every inner step is **named in the settings** and weighed on its own — `want`,
  `says`, `not` and `keep` all hold inside a block
- every inner command is **classified separately**, so a destructive line cannot
  hide inside one, and a block that changes anything still has to say
  `write: true`
- the report carries the inner results under `steps:`, so "where did it stop" is
  **read off the report**, not guessed from a log

The chain stops at the first red, the same rule the outer loop follows. `set -e`
is deliberately **not** used: it would kill the session without telling anyone
the exit code, and the report needs that code. Each step is followed by a marker
line carrying its index and status. The marker is **random per run** — a fixed
one would be taken for a step boundary the day some command prints it.

A step whose marker never arrives **did not run**, and is reported that way.
That is how a dropped connection is told apart from a step that failed: the
first ends the session with steps still unreported, the second names its own
exit code.

Inside a block the verb is `run`, and an inner step names no `on:`, `dir:` or
`env:` — the session belongs to the block, and in a remote shell `cd` and an
assignment are simply commands.

### A flag nobody declared is named, not swallowed

`-with all,now` raises flags, and a task only accepts flags it **declares**.
Measured: a runner that accepts anything turns a misspelled flag into a silent
skip — the run is not red, it is the **wrong work**, finished green. The same
measurement is why `when:` and `unless:` write their condition into the report
when a step is skipped.

`-set name=value` hands the task a value it reads as `{name}`, so a task can take
the command or the target it acts on without a flag name being invented for each
one. A step that writes the same name overrides it.

A condition is a flag name, a flag name behind `!`, or a comparison — `{waiting}
> 0`, `{env} == production`. Both sides are numbers when both parse as numbers,
so an hour gate does not fall back to alphabetical order at nine and ten.

### A command that changes something on a live machine

Remote commands are **classified** before they leave: reads go, writes need
`write: true` on the step, and destructive ones do not go at all.

```yaml
- say: is it up
  on: production
  run: systemctl status app-panel      # reads, goes
- say: restart it
  on: production
  write: true
  run: systemctl restart app-panel     # writes, declared
```

The engine carries the lists, so a project that declares no `read:`/`refuse:` of
its own still gets the classification; `read:` and `refuse:` on a host replace
them. The default for an unknown command is **writes**, never reads — a gate with
a hole in it is worse than no gate, because a list that does not name a command
would otherwise wave every new one through. Each link of a chain is weighed on
its own, a redirection makes a reading command a writing one, and quotes are not
separators: `psql -tAc 'SELECT … WHERE at > now()'` is one link, and it reads.

A pattern in these lists is wrong in **two directions**, and both were measured
on 2026-09-14 while the pilot's deployment chain moved into settings:

| The pattern | What was wrong | Which way it fails |
|---|---|---|
| `^curl\s+(-[sSILk]+…` | no `f` flag, so `curl -fsS …/health` — the most common health probe there is — classified as **writes** | a **blindness**: the gate stops work that was only ever reading |
| `^ip\s+(addr\|route)\b` | any `ip route` line matched, `ip route replace default via …` included | a **hole**: the command that changes a machine's egress address never reached the gate |
| no `strings` in the list | reading a version out of a deployed binary classified as **writes** | a blindness |
| `>` anywhere means writes | `strings … 2>/dev/null \| grep …` counted as a redirection | a blindness — `/dev/null` is not a file, it is a bin, and writing to it writes nothing |

The second is the dangerous one, and the shape of the repair is the lesson: the
verb is now **required** (`show`, `list`, `get`, or nothing at all), because a
family name says what a command is *about*, never what it *does to* the machine.
A blindness costs a turn; a hole costs the thing the gate exists for.

### What was measured into these steps

Four of the rules above are not design, they are repairs:

- **`start` writes to a file, never to the caller's pipe.** A process whose
  output is piped back keeps the pipe open as long as it lives, and the script
  that started it never returns. Same start, own file handles: 0,03 s.
- **`halt` counts.** A turn that restarted only the front end left the process
  doing the actual work on a 22-hour-old build; the symptom read "I restarted it
  and nothing changed".
- **`wait` knocks, it does not sleep.** A fixed wait either wastes time or
  returns early, and a process that dies in its third second looks alive to a
  listing taken at the second.
- **`env:` belongs to the step, not the run.** A build step leaving `GOWORK=off`
  behind was inherited by the check that ran next; the check could not see half
  the tree and five services never started.
- **A shield that cannot see where it looks is not a shield.** `scan` refuses a
  directory that is not there (exit 2) instead of walking past it. Measured: two
  paths went stale after a move, the scan skipped them silently, and the check
  said "passed" while checking nothing — the exact thing it existed to prevent.

### Reading a value out of a machine, and deciding on it

`into:` puts a step's output into a name, `code:` puts its exit code there, and
`keep: true` lets the task go on past a red so the next step can act on what
happened:

```yaml
- say: update
  on: production
  write: true
  run: bash /root/update.sh
  code: updated
  keep: true
- say: roll back if it failed
  when: '{updated} > 0'
  on: production
  write: true
  run: bash /root/rollback.sh
```

The same three fields are how a deployment gate reads a live number — the hours
a company takes calls in, the calls running right now — and stops on it, instead
of holding a copy of those numbers in the settings where they go stale.

<!-- x3-dist version=v0.262.0 capabilities=a3bffcba54192a1718e3d64d08edf50dc8d1998d223b0a35a41478599ed20fa6 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
