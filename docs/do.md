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

Two verbs in one step is a settings error, not a convenience: which of them ran
first cannot be read off the file, and which of them failed cannot be read off
the report. A step with no verb is worse — it shows up as "ran".

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

<!-- x3-dist version=v0.215.0 capabilities=cd8fa8e546bc67b7323831ec5fdf30a6b1f6686c528907907a45a93054264111 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
