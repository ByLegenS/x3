# A guard that builds what it measures

[The pages](INDEX.md) - [what x3 is](../README.md)

## A guard that builds what it measures

**What it catches:** the gate that cannot be written as a guard at all — a
question whose answer does not exist until the gate creates it. *Do the ledger
and the balance still agree when a charge is written?* No reading answers that:
the row has to be put there, the sums weighed, and the row taken away again.
Projects write this as a script, and the script grows to a thousand lines
because setup, measurement and cleanup all live in it.

A trial's `steps` can already build. What it could not do is **take away**.
The measuring phase stops at the first step that does not hold — that is its
whole point — so a cleanup written as the last step is precisely the step that
does not run on the day the gate goes red. The `workspace` is no answer either:
it removes a copied **tree**, and the rows a trial wrote into a database are not
in it.

```json
{ "name": "the ledger still agrees once a charge is written",
  "kind": "steps", "policy": "block",
  "steps": [
    { "name": "a charge is written into the ledger",
      "command": "psql", "args": ["-v", "ON_ERROR_STOP=1", "-f", "trial/charge.sql"] },
    { "name": "the ledger and the balance agree",
      "command": "psql", "args": ["-tAc", "select count(*) from ledger_disagreements"],
      "output": { "must": ["^0$"] } } ],
  "after": [
    { "name": "the charge is taken away again",
      "command": "psql", "args": ["-v", "ON_ERROR_STOP=1", "-f", "trial/undo.sql"] } ] }
```

`after` is the trial's **teardown**, and it takes the same fields a step does.
Two laws, and both are the measuring phase inverted:

| | `steps` | `after` |
|---|---|---|
| a step that does not hold | ends the trial; the ones behind it **never run** | reported; the ones behind it **still run** |
| the run is already red | — | it runs anyway |
| what it says when it holds | nothing | nothing |

The first law is the reason the field exists: a teardown that only runs on the
green path is missing on exactly the runs that created something and stopped
half-way. The second is the same argument one level down — a cleanup that stops
at its own first failure leaves behind the very thing it was written to remove.

A teardown that does not hold is **red on its own**, even when the measurement
held: the data left behind does not make today's answer wrong, it makes
tomorrow's run measure a tree nobody set up. The measurement's red stays in
front of it, because that is the sentence the reader came for:

```
BLOCK the ledger still agrees once a charge is written (steps):
      step 2 of 2 "the ledger and the balance agree": want "^0$", got 3;
      the teardown did not hold: after 1 of 1 "the charge is taken away again":
      the command exits 1
```

**What the teardown is not.** Its output is never the observed value — a guard's
`equals` weighs the last **measuring** step, because what the cleanup printed
says nothing about the question. It does not run when the workspace could not be
opened: no step ran, so the trial built nothing to take away. And `"after": []`
is refused like every emptied declaration — a teardown that runs nothing takes
nothing away, and it should be deleted rather than left standing as a promise.

<!-- x3-dist version=v0.233.0 capabilities=78be6a6944cb9d9d97cea8113ab43bbf0740902426cf469414793bfb9d383db1 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
