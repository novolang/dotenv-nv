# Changelog

All notable changes to dotenv-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-27

The first implementation of the interface published as 0.0.1.

### Changed

These change the interface's declarations, so a program written against
0.0.x needs the edits named here.

- `DotenvEntry` has a `written` field: the assignment's text as it
  appears in the file, or `""` for an entry a program built.  The
  writer renders an unchanged entry from it, which is what makes
  `render(parse_document(text))` equal `text`.
- `DotenvBlankLine` carries the line's text, so a line of spaces
  renders back as spaces.
- `DotenvLine` has a fourth variant, `DotenvSkippedLine(text, fault)`:
  a line a parse with `strict` off could not read, kept as written.
- `DotenvDocument` has `newline` and `final_newline` fields, so a file
  with `\r\n` endings or without a final newline renders back as it
  was.
- `DotenvOptions` is a `@value` struct.  A changed option is a new
  literal.
- `DotenvLoadFault` has a `DotenvFileUnwritable(path)` variant, which
  `save_document` answers when the file cannot be written.
- `dotenvparse.parse_with` is new: a parse whose references resolve
  against the file's earlier lines and then an environment passed in.
  `dotenvload.merged` uses it.
- The dependency is config-core-nv `^0.1.0`, and the toolchain floor is
  0.13.0.

### Behaviour the interface left open

- A double-quoted value accepts `\'` besides the escapes listed in
  0.0.1, as python-dotenv and dotenvy do.
- A parse resolves a reference against the lines above it and inserts
  the answer as written, so a parse cannot loop.  `expand` follows an
  answer that holds a reference, to `max_expansions` deep.
- `${NAME:-text}` uses its default when the name has no value at all,
  as python-dotenv does; an empty value stays empty.
- `unresolved_names` leaves out a reference that has a default.
- `set` keeps the replaced assignment's `export` prefix and its form
  when the form can hold the new value.  `unset`, `comment_out` and
  `annotate` leave a document unchanged when the key is not in it, and
  refuse a key that is not a legal name as `DotenvBadKey` with line 0.
- `save_document` keeps an existing file's permissions and creates a
  new file readable by its owner alone.
- `combine` and `merged` answer the file's entries with the policy
  applied, followed by the environment's variables the file does not
  assign.
- `dotenvconf.rule_for` adds the separator `__` to a prefix that does
  not end in `_`.  `layer_of` keeps every value a string whatever the
  rule's `infer_scalars` says, and `layer_of_inferred` infers.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `dotenvparse` — the grammar, `[]` throughout: three value forms with
  three different escape rules, `value_ends_at` as the public rule that
  makes `abc#123` seven characters and `abc #123` three, `${VAR}`
  interpolation through a resolver the caller supplies, defaults,
  the `export` prefix, multiline values, and a fault per line number.
- `dotenvwrite` — the comment-preserving writer, `[]` throughout:
  `render(parse_document(text)) == text`, `set` that replaces one line,
  `comment_out` and `annotate` because those are what a person does by
  hand, `quoting_for` choosing the narrowest form that round-trips, and
  `to_template` for the `.env.example` nobody keeps up to date.
- `dotenvload` — the file and the environment: `[fs]` for the file,
  `[io]` in the two functions that read the process environment, and
  `DotenvOverride` defaulting to the environment winning so that
  `FOO=bar ./myprogram` still works.  `combine` is `[]`, so both
  directions of the policy are asserted without a filesystem.
- `dotenvconf` — one call into config-core-nv's layer, `[]`
  throughout, with `key_for` inverting the mapping so a missing setting
  can be reported as a line to paste.
- API tests in `tests/dotenvparse_tests.nv` and
  `tests/dotenvload_tests.nv`, red until the bodies land.

### Design notes

The division of labour with config-nv, which the 0.0.1 README argued
and this one no longer does. config-nv 0.0.1 ships a `cfgdotenv`
module with `parse_dotenv`, `read_dotenv`, `dotenv_layer` and
`render_dotenv`, published before this package existed. The division
proposed here is that dotenv-nv owns the grammar and `cfgdotenv`
becomes an adapter over it: `parse_dotenv` becomes `dotenvparse.parse`
followed by `pairs_of` with the fault mapped, `read_dotenv` becomes
`dotenvload.load`, `dotenv_layer` becomes `dotenvconf.layer_of`, and
`render_dotenv` becomes `dotenvwrite.render_pairs`, which also gives
config-nv the comment-preserving `set` it does not have. `dotenvconf`
is the seam that makes that a small change: it hands config-core-nv's
`EnvKeyRule` the pairs and adds nothing, so a layer built either way
is the same value. config-nv is published and is not this package's to
edit, so this is a report to its maintainers.

The overlap with datafile-nv is one function. `dotenvload.find_upwards`
searches a project's directories towards the filesystem root;
datafile-nv answers where a platform puts a program's configuration.
Neither needs anything from the other.

### Named

`std.env` declares `[io]` and there is no `[env]` effect label; it has
no `set`, so this package reads values and never mutates the process.
config-nv's existing `cfgdotenv` module and the division of labour
between it and this package are argued in the README.  `datafile-nv` —
platform directories — is named as the next row and as out of scope.
