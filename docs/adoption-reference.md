# adoption reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.217.0`**

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
| `config_one_file` | the configuration passed `split` lines and no `include` gave it a part |
| `rules_without_why` | a named rule carries no `why` — no sentence saying what it is for |
| `mixed_settings_format` | a settings file is still JSON with nothing holding it there |

<!-- x3-dist version=v0.217.0 capabilities=d5a549f6a4e0603d8c785a64d94e45a05d5266e6a799b7912be83923b7cde7b3 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
