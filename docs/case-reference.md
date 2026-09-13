# case reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.193.1`**

## case finding codes

| Code | Means |
|---|---|
| `example_failed` | the declaration was called and the result is not what the example says |
| `never_ran` | no verdict was reported for it, it was skipped, or a run that died elsewhere never reached it |
| `crashed` | the run started this example and never came back — the process died inside it |
| `over_ceiling` | the package reached `case.timeout` before this example was measured; **no example is accused** |
| `does_not_build` | the example does not compile — charged to its own line when the compiler names one, and to every example in the package when the fault is in the package's own source |
| `malformed` | the payload has no body, does not parse, or holds a proposition that cannot fail |
| `not_a_function` | the example sits above something that cannot be called |
| `in_a_test_file` | the example is in a `_test.go` file, where nothing would run it |
| `wrong_result_count` | the declaration returns a different number of values than the example expects, or a method was given no receiver |
| `dead_type` | a declared type that no example names, reported on the line that declares it |
| `dead_import` | a declared import that no example names — from `case.imports`, asked only of a run whose tree contains the configuration; or from a `//x3:import:` line, asked of the file that carries it |
| `does_not_parse` | a source file the gate could not read at all — the parser stopped, so nothing in that package was measured |

<!-- x3-dist version=v0.193.1 capabilities=dff9950ff823c07e765b340bb2b23450f91ffd4a7ed053d2afcc29ff7e044ae2 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
