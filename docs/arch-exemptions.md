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

<!-- x3-dist version=v0.122.0 capabilities=15ca9e0e91f95ecffad1e8ad49dbf97d72b4af46193928ec5bd51865cf8290e0 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
