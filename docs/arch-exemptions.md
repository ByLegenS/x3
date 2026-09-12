# The one line a rule may not reach

[The pages](INDEX.md) - [what x3 is](../README.md)

## Exemptions — one line, with a reason

`arch` adds no directive type; a violation is silenced with the dictionary's own:

```go
import (
	//x3:allow:arch: the ledger is wired to alpha here, and only here
	"example.com/app/modules/alpha"
)
```

**`skip` does not silence `arch`** — say what you are silencing by name. **An
exemption binds a line, not a tree**: above one import it covers that import,
above a declaration that declaration, above a **statement inside a body** that
statement, above the `package` clause the file, and above a parenthesised
`import (` block it binds nothing and shows up dead, because a block-wide
silence is a deleted rule. A reason is required, exemptions are listed in the
report separately from violations, and a dead exemption is red.

The statement binding is what makes `flow` and `exposure` excusable at all:
their findings are born inside a body, and without it the only way to write one
legitimate exception would be to silence the whole function — including the
violations that walk into that function next year.

```go
func (m *Module) Provision() error {
	//x3:allow:arch: the provisioner is handed the pool once, at startup
	if err := hand(m.pool); err != nil {
		return err
	}
	return hand(m.pool) // still red: the exemption bound the statement above
}
```

### Name the rule you are excusing

```go
//x3:allow:arch:an-app-pool-never-leaves-its-receiver: handed over once, at startup
```

Naming the rule changes two things. It silences **only** that rule, so a second
rule finding something at the same line still speaks. And it ties the
dead-exemption law to the run that could have judged it: a configuration that
does not load that rule answers `unjudged_exemption` — a **warning**, counted
in `summary.unjudged` and printed — instead of calling the line dead. Without
that tie, running a *subset* of the rules pays for exemptions no rule in it
ever looked at.

The law is unchanged. When the named rule does run and excuses nothing the line
is `dead_exemption` and red, and an exemption naming no rule silences every
rule, so every run judges it. A name no rule carries is not an error — a subset
run is legitimate — so it stays a warning on every run instead of hiding.

<!-- x3-dist version=v0.151.0 capabilities=fcf618b09dada98e38d202fcc0d01a4e8208be5be29d181aa1a392505c3c3703 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
