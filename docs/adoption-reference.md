# adoption reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.144.0`**

## the adoption report fields

| Field | Meaning |
|---|---|
| `runners` | the gate scripts that call the engine; declared, not discovered |
| `invoke` | how this project spells a call — one capture group, the command name |
| `sources` / `exclude` | where `//x3:` directives are counted (default `**/*.go`) |
| `tests` | the files whose number is supposed to be falling |
| `token` | the ceiling under which a section is an example rather than an audit |
| `split` | the line count past which a configuration wants `include` |
| `exempt` | command → **reason**; a reason is required and a dead one is a finding |
| `policy` | `warn` (default), `block`, or an object keyed by finding code |

## adoption finding codes

| Code | Meaning |
|---|---|
| `command_unused` | no runner calls it and no exemption says why not |
| `section_token` | a section puts `token` or fewer rules (or directives) in force |
| `dead_exemption` | exempted, and run anyway |
| `tests_remain` | test files still stand where inline examples were meant to be |
| `dead_pin` | `update.pin` vouches for a release below `x3.min_version` |
| `config_one_file` | the configuration passed `split` lines and declares no `include` |
| `dead_policy` | `policy` excepts a code this run does not produce — always a block |

<!-- x3-dist version=v0.144.0 capabilities=c7a898f2ed45fa4107ced156c2151290063596e5205ab3e9fac74e3314f2808e template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
