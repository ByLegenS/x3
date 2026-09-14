# The variables the wrapped command is handed

[The pages](INDEX.md) - [what x3 is](../README.md)

## The variables the wrapped command is handed

A setup step could always be handed the DSN under the application's own variable
name; the **wrapped command** could not. It got exactly one name, `dsnEnv`, and
if the code underneath it reads its connection string from another one — and a
real application does — then a fresh database is created, the code inside the
wrapper never sees it, and the suite runs against the **shared** database it
would have used anyway. Nothing is red. Every project that hit this wrote the
bridge by hand in its own script, and a project that did not know to look never
found out.

`runEnv` closes it, with the vocabulary a step's `env` already uses:

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "template": "app_test_template",
    "runEnv": { "APP_DATABASE_URL": "x3:dsn", "APP_ENV": "test" } } }
```

```
x3 testdb run -- go test ./...
```

The command — and everything it starts — now finds the fresh database under the
name its own code reads. The rules are the step's rules, checked at the same
door and with the same exit `2`: `"x3:dsn"` is the only reference the engine
answers, any other `x3:` value is a refused typo rather than text passed on, and
**`dsnEnv` itself may not be written** — that would point the command at another
database while the rest of the run talks to this one. Written empty
(`"runEnv": {}`) is exit `2` as well: a run that sets nothing does not write the
field.

It reaches further than it looks. `x3 testdb run -- x3 case ./internal/...`
gives the example runner, which has no environment seam of its own, the same
fresh database — the variables are inherited by every process under the wrapper.

<!-- x3-dist version=v0.202.0 capabilities=b7444b0c0fe5258026b96f39e4e34386e7733f4a960eaeceb1deb4973a1dd59a template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
