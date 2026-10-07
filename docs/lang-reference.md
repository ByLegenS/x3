# lang reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.338.0`**

## language settings

| Field | Meaning |
|---|---|
| `allowed` | the language outside comments; `en` is the only embedded dictionary, and any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone; `en` holds them to the dictionary |
| `strings` | `en` (default) holds every string constant to the dictionary; `any` leaves them alone |
| `sources` | which files are read; left out, the Go tree |
| `names` | which paths have their **name** read; left out, no name is read |
| `allow` | project terms no dictionary has. One lowercase word as the gate splits it; ASCII needs three letters or more, a word with a non-ASCII letter any length (`é`, `café`; case-folded, dotted and dotless i fold together) — an entry that could never match is rejected rather than ignored |
| `keywords` | a language's reserved words, per extension. Written, the list **replaces** the embedded one for that extension; written empty, that language has no reserved words |
| `layers` | readers for the names of the other layers - `script`, `style`, `markup`, `yaml`, `placeholder`, `sql`, `literal` - each with its own `sources`, `exclude` and `allow`; left out, none runs |
| `tags` | struct tag keys whose **name** is read (`json:"name"`); accepted only with `strings: "any"` |

<!-- x3-dist version=v0.338.0 capabilities=c97edf7bc2c7b7f3fae74ed6188fa7c41a96a3068da351af671960e287d7e0ed template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
