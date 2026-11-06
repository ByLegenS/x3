# Names in every layer

[The pages](INDEX.md) - [what x3 is](../README.md)

## Names in every layer

**What it catches:** a second language in the names a project declares outside
its Go code - a JavaScript variable, a CSS class, a template prop, a settings
key, a translation placeholder, a table column, a struct tag - the layers a
project renames by hand and nothing keeps from coming back.

Reading those files whole (`sources`) is the wrong tool: it reads every token
outside strings and comments, so every **use** of a name declared elsewhere -
`document`, `addEventListener`, `SELECT` - arrives as a finding. A reader knows
where its language **declares** a name and reads only there, the same law the
Go reading follows.

```yaml
language:
  strings: any
  tags: [json, yaml]
  layers:
    script:      { sources: ['ui/**/*.js'] }
    style:       { sources: ['ui/**/*.css'] }
    markup:      { sources: ['ui/**/*.html'] }
    yaml:        { sources: ['x3.yaml', 'x3/**/*.yaml'], values: [name, id] }
    placeholder: { sources: ['**/i18n/*.json'] }
    sql:         { sources: ['**/migrations/*.sql'] }
    literal:     { sources: ['**/*_fixture.go'] }
```

| Reader | Reads | Does not read |
|---|---|---|
| `script` | declared variables, constants, functions, classes, interfaces, parameters; class members and methods; object keys, quoted or not; `this.x =` fields | comments, strings, template text, the use of any name |
| `style` | class and id in a selector, custom property (`--x`), `@keyframes` name | property values (`#fff`, `0.5em`, `url(a.png)`), at-rule preludes |
| `markup` | the classes in `class`, the `id`, the name of `data-*`, and on a **component** (a tag with a dash or a capital) the names of `:prop`, `@event`, `v-bind:` / `v-on:`; `v-directive` and `#slot` names | element text, other attribute values, interpolations, `<script>` / `<style>` content, bindings on built-in elements |
| `yaml` | every mapping key; the value (or list of values) of the keys named in `values` | other values, block text, comments |
| `placeholder` | the name inside `{name}`, `{{name}}`, `%{name}`, `{count, plural, ...}` | the translated text |
| `sql` | what `CREATE` names (table, index, type, function, view, ...), table columns and constraints, function parameters, `ALTER TABLE ... ADD`, the new name of a `RENAME` | the names a query uses (`SELECT`, `JOIN`, `ON`) |
| `literal` | code-shaped strings - lowercase, no space, cut by `-` `_` `.` (`order-status`, `invoice.paid`) - in code and on `x3:case` lines | sentences, capitalised text, paths, numbers |
| `data` | the words of the values of the fields named in `values`, at any depth, in a JSON or YAML file | every key, every other value |
| `tags` (Go) | the name part of each declared struct tag key (`json:"total_cents,omitempty"` → `total_cents`) | the options, undeclared keys |

**A binding on a built-in element is not the project's name.** `<input
:maxlength>` and `<div @keydown>` name HTML's attribute and the DOM's event -
the use of a name declared elsewhere. On a component, the same binding names
the component's own prop or event, and it is read.

**A digest is not a name.** A token of eight or more lowercase hex digits with
at least one digit (`a0834f1db4cb`) is an id; split on its digits it would
leave letter runs that look like words. Every reader skips it.

**Whether a string is a name or data, the engine cannot know.** That is why
`literal` reads only the files a project points it at; a value the project
keeps on purpose goes to `allow` with its reason, or to the baseline.

**A word can be a contract in one layer only.** A public URL anchor in markup,
a tenant's template variable in a placeholder, a data word used as a settings
key: written into the project's `allow`, the same word would pass as a Go
identifier everywhere. Each reader takes its own `allow` (same entries, same
matching, applied to that reader's findings alone) and its own `exclude` (the
paths it drops from its `sources`, same pattern language):

```yaml
language:
  layers:
    markup:
      sources: ['ui/**/*.html']
      exclude: ['ui/vendor/**']
      allow: [rechnung]       # a public anchor; still red as a Go name
    yaml:
      sources: ['x3/*.yaml']
      values: [name]
      allow: [é]              # a rulebook section code inside step names
```

`!` in front of a pattern is not negation in this glob; the error names the
reader's `exclude` to write it under. An `exclude` pattern that drops none of
the reader's paths is `empty_scope`, and so is a `sources` pattern whose every
path is excluded - a reader narrowed to nothing reads nothing, silently
otherwise. Written but empty, either list stops the run.

Each reader is a separate claim. A finding says which reader saw it (`where`
is the reader's name, `tag` for struct tags), so its baseline identity never
collides with a Go finding and the identities of existing findings do not
change. A reader whose pattern matches no path is `empty_scope`, a declared tag
key no struct carries is `empty_scope`, a reader name the engine does not have
stops the run, and a file a reader cannot parse (`yaml`) stops it too. The
per-file cache holds every reader's findings with the file's, salted by the
`language` section, so changing a reader re-reads every file once. With
readers declared, the summary adds a line:

```
x3 lang: by reader - literal 1, markup 1, placeholder 1, script 1, sql 1, style 1, yaml 1, tag 1
```

`x3 lang -with <fragment>` overlays a settings fragment for one run, so a
project can measure a reader before it writes it into its settings.

Control experiment, three arms over one planted tree with one foreign name per
reader and foreign words in its comments, strings and text: every reader
declared → exactly one finding per reader, eight in all; no reader declared →
the same tree is green; a reader pattern no file answers → one `empty_scope`.
A reader broken on purpose (the declaration pattern of `script` removed) turns
the planted arm red. Measured on a production tree (1,691 files, every reader
declared): cold 640 ms, warm 545 ms against 318 ms for the Go reading alone;
the default run's report is byte-for-byte the one before readers existed.

Control experiment for a reader's own `allow` and `exclude`, on a second
planted tree (one Go file, two markup pages): the markup reader allows a word
and a non-ASCII word and excludes one page → two findings, both the Go
identifiers carrying those same words; neither written → the markup reader
adds three; the same words in the project's `allow` → the Go names pass too;
an `exclude` matching none of the reader's paths, and a `sources` pattern whose
only path is excluded → `empty_scope` each. With the reader's `allow` and
`exclude` cut out of the engine on purpose, three arms turn red. Measured on a
production tree: the default run is byte-identical and as fast (warm ~300 ms
before and after); with every reader on, a markup `exclude` dropped 93 files
and 340 findings, and a `yaml` reader `allow` of two non-ASCII letters cleared the
four step-name findings while one of them stayed red in the script layer.

### Addresses and codes inside a data file

**What it catches:** a second language in the values a data file declares as
addresses or codes - a route's `path`, a form's `submit` - when the same file
rightly holds translated `title` and `label` text. The `yaml` reader cannot do
this: its `values` are shared by all its sources and it reads every key.

```yaml
language:
  layers:
    data:
      sources: ['themes/**/theme.json']
      values: [path, submit]      # required; the only thing this reader reads
      allow: [impressum]          # optional, as on every reader
```

```json
{ "title": "Kontakt", "routes": [{ "path": "kontakt/{slug}/", "title": "Schreiben Sie uns" }] }
```
```
themes/a/theme.json:1:45: not_in_dictionary: kontakt (data)
```

A field matches at any depth. A string is read, a list's items are read, and in
an object only the named fields are read (`links: {book: [...]}` with `book`
named). A value is cut at `/` `-` `_` `.` and space, and `{slug}` is not read.
Non-strings are not read. The reader names no language, so `no_reader_covers`
never counts on it. A source that does not parse is `unparsable`, one finding
per file that no baseline can hold. A `sources` pattern with no file and a
field no parsed file carries are `empty_scope`.

Control experiment (`lang data reader`, ten arms): English paths beside foreign
titles, a placeholder and an unread file → green; a foreign path, a non-ASCII
letter, a list item, a YAML source → one finding each; the reader's `allow` →
green; a broken file, a dead pattern, a dead field → one finding each; no
`values` → exit 2.

### The files no reader covers

**What it catches:** a gate blind to a new directory. A reader reads what its
`sources` name; a new top-level directory nobody added to the list produces no
finding, because a file nobody reads cannot be wrong, and the run says `0 finding(s)`.

So a declared reader claims its language. Once `script`, `style`, `markup` or `sql`
is declared, every file with that reader's extensions (`.js .mjs .cjs .jsx .ts .tsx`,
`.css .scss .less`, `.html .htm .vue`, `.sql`) must be in the `sources` of some
reader, or in `language.excluded` with a reason:

```
clients/shop:0:0: no_reader_covers: 4 file(s) no reader covers, none excluded: a.css, b.css, c.css and 1 more (scope)
```

```yaml
language:
  excluded:
    'apps/*/migrations/*.sql': sealed migrations; a file that ran in production cannot change
```

One line per directory (the first two path segments), three file names on each,
forty directories at most, then a count. `no_reader_covers` takes no baseline: it
is the gate's own blindness, not debt in the code.

`yaml`, `placeholder` and `literal` claim nothing: their `sources` name a role (a
settings file, a translation dictionary), and every other `.yaml` or `.json` in the
tree is not their blindness. A reader's own `exclude` is a stated decision and counts
as covered. The engine's skipped directories (`vendor`, `node_modules`, `testdata`,
names starting with `.` or `_`) and git-ignored paths are not in the tree to begin
with. An `excluded` pattern sparing no unread file is `empty_scope`, so an exclusion
outlives neither its directory nor the day a reader starts to cover it; an entry with
no reason stops the run.

Control experiment, `lang reader coverage control experiment`: a fully read tree is
green; a new directory gives one finding with its files; a reader's `exclude` and a
reasoned `excluded` silence it; an entry with no reason stops the run; a dead pattern
is red; another language, a skipped directory stay silent; `-update-baseline` does not
freeze it. Measured on a production tree: eleven directories, among them three
stylesheets a `style` reader had never seen, in 0.7 s for the whole run.

<!-- x3-dist version=v0.348.0 capabilities=3bfeaae6d5bff6885cacb90e0b4f5b6f4dba87120045133beee6c3ab9349265a template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
