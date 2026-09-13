# A forbidden list the database writes

[The pages](INDEX.md) - [what x3 is](../README.md)

## A forbidden list that comes from a query

**Catches:** a value that lives in the database reaching text people outside it
read — another tenant's name in an example string, a user's name in a fixture, a
record that exists being mentioned to somebody who should not know it exists.

This happened in production to a project running this engine: one customer's
panel showed *"e.g. \<another customer\> Branch"* in a search box. Nothing leaked
from the database — it was a hard-coded example — but the second customer's
**existence** was being told to the first.

A hand-written deny list cannot hold that line, and this is the whole reason the
feature exists: such a list protects today's values, the record opened tomorrow is
not in it, and the gate keeps passing while measuring nothing. That is the worst
state a gate can be in, because its green claims something was checked.

```json
{ "name": "no-tenant-name-in-the-interface",
  "sources": ["ui/**/*.html", "ui/**/*.json"],
  "denyFrom": {
    "dsnEnv": "APP_DSN",
    "query": "SELECT code FROM tenants UNION SELECT name FROM tenants",
    "least": 5,
    "head": true,
    "skip": { "example": "the fixture tenant every environment carries" }
  },
  "comments": "exempt",
  "reason": "a tenant's existence is not told to another tenant" }
```

### What each part is for, and why it is the project's to write

- `least` drops values too short to search for. A three-letter name matches inside
  ordinary words and buries the gate in false reds. The threshold is the
  project's because whoever knows the data knows what counts as short.
- `head` also looks for the **first word**: an example usually writes it rather
  than the full registered name.
- `skip` takes a value and a **reason**. The comparison folds accents and case,
  because a language's case folding breaks matching silently — measured: in one
  project the company's own name failed to match its own exemption, and the gate
  went red against itself on the first day.
- values are searched as **literal text**, quoted before they become a pattern, so
  a dot or a parenthesis inside a record's name cannot start matching other
  things.

### It says when it did not measure

An empty environment variable is **exit 2**, not a pass: a check that could not
read its list has measured nothing, and a green there would be a lie of the
cheapest kind. A query that returns no rows is the same — an empty list passes
every file.

The report prints **how many** values were searched and never the values
themselves: a report listing them would leak exactly what the check exists to
protect.

<!-- x3-dist version=v0.185.0 capabilities=1ee338e5c8d1ec7040cc5325fc6cbca863abdaec69e021877be96075e8ad3a4a template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
