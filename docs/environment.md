# The environment a measurement needs

[The pages](INDEX.md) - [what x3 is](../README.md)

## The environment a measurement needs

The engine has always been able to **say** that a variable was missing. It could
not **stop**. Measured on this repository: one tree, one binary, one variable
written or not, and two different numbers — with it unset a criterion skips
itself, the run still produces a green or a red, and that number is read as a
regression. Every project that met this wrote an environment door by hand at the
head of its gate.

A project can now declare the contract, and the engine enforces it **before
every command it binds**, next to the minimum version gate:

```json
{ "x3": { "environment": {
    "require": ["APP_DSN"],
    "for": ["boxes", "guard"],
    "remedy": "start the local database and export APP_DSN" } } }
```

```
x3: RED - this run cannot measure: APP_DSN is not written in this shell
	run: start the local database and export APP_DSN
```

Exit `2`, because this is not a measurement — it is the knowledge that none can
be taken, and `2` is what the rest of the engine says for that.

| Field | Required | Meaning |
|---|---|---|
| `require` | yes | the variable names. **An empty value counts as unwritten**, the way a live guard already reads one — there is no connection opened with an empty DSN. Written empty (`[]`) is exit `2`: a contract that asks for nothing cannot be told from one nobody wrote |
| `for` | no | the commands the contract binds; **every** command when it is left out. A name that is no command of this engine is exit `2`, because a misspelled one would bind nothing and read as if it did |
| `remedy` | no | the one line printed under the refusal. The engine cannot know it — only the project knows which script writes the variable |

`for` exists because a variable is required for a **measurement**, and a command
that only reads source has nothing to do with that measurement. A contract
imposed on every command makes a project choose between a gate it cannot use and
no gate at all.

**What is not here: reachability.** A `require` is a `LookupEnv` — no timeout,
no failure mode of its own, no secret in an error message, and it runs before
*every* command. Asking instead whether a host answers puts a network round trip
in front of every invocation and gives the entry point a second way to fail. The
engine already has the place where reachability is measured, with a policy, a
timeout, redaction and a report: a `sql` or `http` guard in `x3 guard`. A weaker
second answer to the same question, running everywhere, would be the one that
gets trusted.

<!-- x3-dist version=v0.205.0 capabilities=d6ad45075401095d1e70395be01cd18df995c1892b169a7b9c527f3de04694ef template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
