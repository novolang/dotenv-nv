# Changelog

All notable changes to dotenv-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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

### Named

`std.env` declares `[io]` and there is no `[env]` effect label; it has
no `set`, so this package reads values and never mutates the process.
config-nv's existing `cfgdotenv` module and the division of labour
between it and this package are argued in the README.  `datafile-nv` —
platform directories — is named as the next row and as out of scope.
