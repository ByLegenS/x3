# What is left of a migration

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 retire`

**What it catches:** a migration that stopped moving and nobody noticing - a
pile of files a project promised to delete, and a count that grew back while
everyone was reading the plan instead of the tree.

```
x3 retire [-config <file>] [-out <file>] [dir]
```

**It is a promise, not a mirror.** This is the one audit whose findings default
to `block`: a group on the ledger is a commitment, and a commitment that reads
green while the files are still standing is a plan, not a measure. A project
that wants the number without the teeth writes `"policy": "warn"`.

**The beginning is declared, not measured.** `start` is written by hand and
does not move. A beginning the engine measured would equal today's count on
every run, and the percentage would say nothing but "you are where you are".

```json
{ "retire": {
    "exclude": ["vendor/**"],
    "groups": [
      { "name": "go test files", "patterns": ["**/*_test.go"], "start": 658,
        "reason": "inline examples replace them" },
      { "name": "python gate scripts", "patterns": ["**/*.py"], "start": 80,
        "reason": "every check a script does by hand is a capability the engine owes" }
    ] } }
```

See **the retire report fields** in the [retire reference](retire-reference.md#the-retire-report-fields).

A file is counted by the FIRST group whose pattern matches it. Two groups
counting one file would report a single debt twice, and a total built that way
never reaches zero even on the day the tree does.

### Red

```
  TOTAL  [############........]  309 of 776 file(s) remain - 60.2% retired - 71463 line(s)

  go test files              [############........]    254 of   658 remain -  61.4% -  51509 line(s)
  python gate scripts        [#########...........]     44 of    80 remain -  45.0% -  14301 line(s)

BLOCK go test files files_remain
        254 of 658 file(s) still stand (61.4% retired, 51509 line(s)); this
        group is on the ledger to reach zero
```

The second red is the one a migration cannot see from the inside - the pile
growing back:

```
BLOCK tests count_grew
        2 file(s) stand where the ledger begins at 1; either the group grew or
        the beginning was written after it did
```

### Green

```
x3 retire: 0 of 4 file(s) remain - 100.0% retired - 0 line(s) - 1 of 1 group(s) done
```

**A finished group stays on the ledger.** Removing it from the configuration
would remove the guard with it, and the pile would be free to grow back under
the same name. A group at zero costs one line and one walk of the tree, and it
is the only thing standing between a migration that ended and a quiet second
one.

| Finding | When |
|---|---|
| `files_remain` | the group still holds files; the message carries the count, the share and the lines |
| `count_grew` | today's count stands above `start` |
| `dead_policy` | a policy exception that matched nothing in this run |

<!-- x3-dist version=v0.253.0 capabilities=90771c9baba0dd9cb9a7fb8f5bd2261bb484beb53539596f848bf8e09cae8e61 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
