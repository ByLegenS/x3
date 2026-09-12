# case reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.145.0`**

## case finding codes

| Code | Means |
|---|---|
| `example_failed` | the declaration was called and the result is not what the example says |
| `never_ran` | no verdict was reported for it, it was skipped, or a run that died elsewhere never reached it |
| `crashed` | the run started this example and never came back — the process died inside it |
| `does_not_build` | the example does not compile — charged to its own line when the compiler names one, and to every example in the package when the fault is in the package's own source |
| `malformed` | the payload has no body, does not parse, or holds a proposition that cannot fail |
| `not_a_function` | the example sits above something that cannot be called |
| `in_a_test_file` | the example is in a `_test.go` file, where nothing would run it |
| `wrong_result_count` | the declaration returns a different number of values than the example expects, or a method was given no receiver |
| `dead_type` | a declared type that no example names, reported on the line that declares it |
| `dead_import` | a declared import that no example names — from `case.imports`, asked only of a run whose tree contains the configuration; or from a `//x3:import:` line, asked of the file that carries it |
| `does_not_parse` | a source file the gate could not read at all — the parser stopped, so nothing in that package was measured |

<!-- x3-dist version=v0.145.0 capabilities=053cb5bda96cabedacc8ad6dc3d302d827ec2d9dcde5c77e648e817df0e093a8 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
