# Two schemas, one question

[The pages](INDEX.md) - [what x3 is](../README.md)

## Two schemas, one question

**What it catches:** the change that lives only on the developer's machine. A
column added by hand while something was being made to work, and never written
into the migration file — everything passes locally, and in production the
column is never created. No checksum sees it, because the file that should have
changed did not.

The question has a shape: *apply the migrations to an empty database, then ask
both databases the same question and compare the answers.* The answer is not a
value; it is a **set of rows** — every column, every constraint, every index.

```json
{ "name": "the migration files produce the development schema",
  "kind": "sql", "dsnEnv": "APP_DB_DSN",
  "query": "select table_name || '.' || column_name || ' ' || data_type from information_schema.columns where table_schema = 'public'",
  "sameRowsAs": { "dsnEnv": "X3_TESTDB_DSN" } }
```

| Field | Required | Meaning |
|---|---|---|
| `dsnEnv` | yes | **name** of the variable holding the other side's DSN — the value never appears in the file, and a field holding one is refused with exit `2` |
| `query` | no | left out, the guard's own query: *the same question, two databases*. Written, the other side may ask its own — two schemas that keep the same truth in different shapes |
| `driver` | no | defaults to `pgx`, like the guard's own |

`sameRowsAs` is an expectation, so it takes the place of `equals` and
`contains`; writing it next to one of them is refused. A guard compares a set
or a value, never both — otherwise which reading turned it red is unreadable.

### One column, and a set

The query must answer with **one column**. Joining several would mean the
engine picks a separator, and the answer to *"are these the same"* would depend
on a character nobody wrote. How a row is spelled belongs to the query:
`table_name || '.' || column_name` is the writer's sentence, not the engine's.

The answer is a **set**, not a list. Order does not matter, so no `order by` is
needed, and a row that comes back twice counts once — the same constraint
definition can sit on two tables, and neither side should turn red for that.

### The difference is named

A comparison that only says *"not equal"* leaves the search to the reader. Both
directions are reported, row by row:

```
BLOCK the migration files produce the development schema (sql): 1 row(s) only in APP_DB_DSN, 1 row(s) only in X3_TESTDB_DSN
	want: the same rows as X3_TESTDB_DSN
	got:  412 row(s) here, 412 there
	only here:  app_orders.shipped_at timestamp with time zone
	only there: app_orders.note text
```

*Only here* is a change the migration files do not carry. *Only there* is
something the files build that the developer's database has not got. The report
carries the same two lists as `onlyHere` and `onlyThere`, capped at twenty
names each with the rest counted, so one drifted schema cannot swallow the
report. The counts in `detail` are always the whole difference.

### Where the second database comes from

Nothing here creates it. [`x3 testdb`](testdb.md#a-fresh-database-for-this-run)
does: it clones or creates one, runs the migrations as its setup steps, and
hands the DSN over through the environment under the name this guard reads.

<!-- x3-dist version=v0.138.0 capabilities=f63b177d435cdd61e34235067c446339025b12c519a1cb36f46e1e0052e6dd2c template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
