# One name, one source in a component

[The pages](INDEX.md) - [what x3 is](../README.md)

### `members` and `left-declared-once` — two mixins, one method name

**Catches:** a component whose mixins declare the same name twice. A component
framework merges the component and its `mixins` into one instance; when two of
them declare the same method, computed value, prop or data field, one silently
replaces the other. Each file is correct on its own, every build and lint passes,
and the screen crashes the first time it calls the method that was replaced.

```yaml
- name: every-component-member-has-one-source
  kind: consistency
  compare: left-declared-once
  per: file:stem
  sources: [ui/screens/*.js]
  left: { from: template, select: members, resolve: [ui] }
```

`members` reads the object a file exports by default and takes the names it
writes onto the instance: the keys of `methods`, `computed` and `props` (object
or array form) and the fields `data` returns (`data() { return {…} }` or
`data: () => ({…})`). Then it follows `mixins: [a, b]` — a local `const a = {…}`
or a named import, alias included — to the object of that name and reads it the
same way, two levels deep. Every **declaration** is recorded with the file that
writes it, so a name written twice in one file is two records too. Lifecycle hooks (`mounted`, `created`) are left out: the framework calls
each of them, so the same hook in two mixins is legitimate. Comments and string
contents are blanked before the object is read, so a `{` in a string does not
move its end. A file with no default export (a mixin alone) gives nothing: it
is measured inside the component that takes it.

`left-declared-once` is a one-sided comparison and takes no `right`: every left
value must have a single record. A value recorded twice is `set_mismatch`, with
every place named. An extractor that reads per file records a value once per
file, so with any other set the question is "one file only". Reaching the same
mixin by two paths (a diamond) reads it once and is no clash. With `per: file:stem` each component is weighed with its own
chain only, so two unrelated screens that both have a `save` method agree.

<!-- x3-dist version=v0.353.0 capabilities=aa8be531db69cba5993e6f142d79236e32d23e7dcc790e359aa835050c5b1dcf template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
