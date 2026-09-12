# Getting the database ready

[The pages](INDEX.md) - [what x3 is](../README.md)

## A ready database is more than one command

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "dsnEnv": "X3_TESTDB_DSN",
    "setup": [
      { "command": "./app", "args": ["-migrate"], "env": { "APP_DSN": "x3:dsn" },
        "timeoutMs": 180000 },
      { "command": "./app", "args": ["-serve", "-announce-only"],
        "env": { "APP_DSN": "x3:dsn", "APP_ENV": "test" } }] } }
```

Migrations build the **schema**. An application usually declares more than that
at start-up — the rows it needs to exist, written by its own boot path and by
nothing else. A hook that takes one command leaves such a project two ways out,
and both are bad: write a shell line, or run the suite against a database that
is missing exactly the rows the application would have written. The second is
silent, which is what makes it the expensive one.

So `setup` is a **list**, and the order is the list's. Not two named hooks
(`migrate` then `seed`): the engine would then be deciding how many parts a real
application's preparation has, and the third part would have nowhere to go.

**Steps run in order and stop at the first red.** Step two writes onto the
schema step one built, so its failure after a failed step one is not its own.
The failure names **which** step it was, and the program it actually resolved
to: `setup step 2 of 3 "go" (/usr/local/go/bin/go) failed: …`. The timeout
(`timeoutMs`, 60 s by default) is **per step** — a migration may take minutes
and a boot announcement seconds, and one cap over the list would make the slow
step decide for the fast one.

### The variables a step is handed

Every step gets the created database's DSN in the variable `dsnEnv` names, and
`env` adds its own on top. Values are literal, with **one** reference the engine
answers:

| Value | What the step gets |
|---|---|
| `"x3:dsn"` | the DSN of the database this run created |
| anything else | itself, unchanged |
| anything else starting `x3:` | **exit `2`** — the namespace is closed, so a typo is refused rather than passed on as text |

This is what removes the shell. An application reads its DSN under its own
variable name, and without `env` the only way to put it there is a shell line
like `bash -c 'APP_DSN="$X3_TESTDB_DSN" exec ./app -migrate'`. Measured on
Windows: `bash` resolves to the WSL launcher when the gate is started from
PowerShell, that Linux has no `go` on its `PATH`, and **every** step exits
`127` — from a settings file that works by hand (§ *The program a command name
means*).

**A step may not write `dsnEnv` itself.** Overwriting it would point the step at
another database while everything else in the run still talks to this one, and
that class of failure — a suite green against the wrong database — is the reason
this section exists. It is a configuration error, exit `2`. Writing
`adminDsnEnv` is allowed and often the point: handing the application's own
variable the *test* DSN is how it stops seeing the maintenance one. Written
empty (`"setup": []`, `"env": {}`), a step with no `command`, or a variable with
no name is exit `2` as well.

<!-- x3-dist version=v0.142.0 capabilities=26bb19f71c95ebb12c25cee2d5a14f374ec748b090b8afa420df48fd5ddb1db7 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
