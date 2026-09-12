# x3 reference

The lookup tables behind the guides: field names, error codes and exit codes.
One page per family, each table under the capability it belongs to, all of them
generated from one document in one run - so no two can describe different
versions. Nothing here is new; every guide links to the table it uses.

[What x3 is](README.md) - [the pages](docs/INDEX.md)

**Current version: `v0.150.0`**

| Reference | Tables |
|---|---|
| [scan](docs/scan-reference.md) | scan exit codes, the directive dictionary, scan error codes, the scan report fields, expectation fields |
| [case](docs/case-reference.md) | case finding codes |
| [lang](docs/lang-reference.md) | language settings |
| [arch](docs/arch-reference.md) | component path patterns, arch rule fields, arch error codes, how a rule reads a file |
| [boxes](docs/boxes-reference.md) | boxes criteria fields, boxes criteria written in prose |
| [guard](docs/guard-reference.md) | guard exit codes, guard fields every kind has, guard fields for kind sql, guard fields for kind http, guard fields for kind exec, guard fields for kind steps, the guard report fields |
| [effective](docs/effective-reference.md) | effective check fields, effective reading fields |
| [adoption](docs/adoption-reference.md) | the adoption report fields, adoption finding codes |
| [testdb](docs/testdb-reference.md) | testdb settings |

<!-- x3-dist version=v0.150.0 capabilities=162fe2ced0d891cd8733aba17d14fcabc3618c79fd93d1547dabbbcdc64d0fcb template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
