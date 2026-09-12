# dotenv-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The `.env` format: the grammar, a document that keeps its comments so a
rewrite is a one-line edit, a loader whose override policy against the
process environment is an argument rather than a habit, and one call
that turns a file into a config-core-nv layer.

It is the format's package.  config-nv is the layered configuration
front that stacks a `.env` beside a TOML file and the process
environment; this is the thing under it that knows what
`PASSWORD=abc #123` actually means.

## Adding it, and checking it

```bash
novo pkg add dotenv-nv       # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/dotenvparse_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: dotenv-nv.<module>.<fn>`.

## The one example that will work

```novo
use dotenvload
use dotenvparse

// Read a .env beside the program, with the process environment winning
// — which is what makes `DATABASE_URL=… ./myprogram` work.
//
// `shadowed_keys` is the answer to "I changed .env and nothing
// happened": the file assigned those keys and the environment already
// had them.  The caller decides whether that is worth a line of output.
fn settings() -> Result<DotenvLoaded, DotenvLoadFault> [fs, io]
    let loaded = dotenvload.merged_optional(dotenvload.DOTENV_FILE_NAME,
                                            DotenvKeepEnvironment,
                                            dotenvparse.default_options())!
    let ignored = dotenvload.shadowed_keys(loaded)
    Ok(loaded)
```

## The layer, and why

`host`, and three of the four modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `dotenvparse` | `[]` throughout | the grammar; the interpolation resolver is an argument |
| `dotenvwrite` | `[]` throughout | rendering and editing a document |
| `dotenvconf` | `[]` throughout | the conversion into config-core-nv's tree |
| `dotenvload.load`, `.load_optional`, `.load_document`, `.save_document`, `.find_upwards`, `.is_world_readable` | `[fs]` | the file |
| `dotenvload.environment`, `.environment_value` | `[io]` | the process environment |
| `dotenvload.merged`, `.merged_optional` | `[fs, io]` | both |
| `dotenvload.combine` and the accessors | `[]` | the policy, so it can be asserted without either |

## What `std.env` declares, since the plan's row said `[env]`

**`[io]`, and there is no `[env]` effect label.**  SPEC § 5.1's
vocabulary puts pipes, processes and the environment together under
`[io]`, so `env.get`, `env.vars` and `env.names` all declare `[io]` and
nothing narrower exists to declare.  `dotenvload.environment` and
`.environment_value` are the two functions in this package that carry
it, and they are the only two.

**And `std.env` has no `set`.**  Setting a variable is
`process.env_set(name, value)`, in a different module, deliberately.
That settles a design question this package would otherwise have had:
"loading a `.env` file" here means **reading values**, never mutating
the process.  `merged` answers the pairs a program should use and what
the program does with them is the program's.

That is the better shape anyway.  A program that takes its
configuration as a value can be tested twice in one process with two
different configurations; one that reads a global cannot.  Every
`.env` library in every other language mutates the environment, and
every one of them has an open issue about tests interfering with each
other.

## The load-bearing interface

**`DotenvDocument` is a list of lines, not a map of keys.**

A `.env` file is a file a person edits.  It has comments explaining why
a value is what it is, a blank line between the database section and
the mail section, and a commented-out override somebody left for next
time.  It is usually in version control.

Every other implementation of this format parses it into a map, and
their `set_key` writes the map back — which rewrites the file, loses
the comments, and produces a forty-line diff to change one value.  The
practical consequence is that nobody uses the writer: people edit
`.env` by hand and the library's write path is dead code with a bug in
it.

So `parse_document` keeps every line as it was written, `render` puts
them back byte for byte, and `dotenvwrite.set` replaces **one line**.
The round trip is one assertion — `render(parse_document(text)) ==
text` — and it covers the whole of what the writer is for.

What that unlocks is the rest of `dotenvwrite`: `comment_out` rather
than `unset`, because that is what a person does when they want a value
back later; `annotate`, so a tool that changes a value can say why
beside it; and `to_template`, which turns a real `.env` into a
`.env.example` with the keys and the comments and none of the secrets —
so the example stops being a file somebody maintains by hand and
forgets to update.

The second decision is that **the interpolation resolver is a function
the caller supplies**.  `dotenvparse.expand` takes a `fn(Str) -> ?Str`,
which is what keeps the module `[]` — and it makes the lookup *order* a
value rather than an assumption.  `dotenvload.resolver_over(entries,
env)` is the order every implementation uses, written down: the file's
own earlier lines first, then the process environment.  The other order
would make a file unable to build a value out of its own earlier lines,
which is what `${BASE}/x` is for.

## The three value forms, which do not agree with each other

This is the whole difficulty of the format, and none of it is visible
when you look at a file.

| written | means |
| --- | --- |
| `A='a\nb'` | six characters, with a backslash.  Literal throughout; no escapes, no interpolation, and no way to write a single quote inside |
| `A="a\nb"` | five characters, with a newline.  Escapes processed, `${…}` interpolated |
| `A=a\nb` | bare.  A `#` after whitespace ends the value; a `#` without whitespace before it does not |

**The bare form's comment rule is the silent one.**
`PASSWORD=abc#123` is the seven characters `abc#123`.
`PASSWORD=abc #123` is the three characters `abc`.  One space, and the
password is wrong — and the program does not fail, it authenticates as
nobody, or connects to a database that accepts the connection and has
none of the data.

`dotenvparse.value_ends_at` is that rule as a public named function
precisely so a test can assert both halves of it directly, and so a
reader who does not believe it can run it.

**`${VAR}` with no value is empty, not an error**, because that is what
a shell does and what every existing file assumes.  A connection string
with an empty `${DB_PASSWORD}` in it connects somewhere rather than
failing — so `unresolved_names` and `dotenvload.unresolved_in` exist,
and a program that wants a missing variable to be fatal asks for that
list at start-up and refuses on its own terms.

**`export FOO=bar` is an assignment**: the prefix is there so the file
can be `source`d by a shell, it means nothing to a parser, and a parser
that kept it produces a variable called `export FOO`.

## Where this package sits next to config-nv

config-nv 0.0.1 already ships a `cfgdotenv` module with `parse_dotenv`,
`read_dotenv`, `dotenv_layer` and `render_dotenv`.  That is not a
duplicate to be resented; it is the front that was published first, and
the plan's row for this package says config-nv is "the layered front"
rather than the grammar's owner.

**The division this package proposes**: dotenv-nv owns the grammar —
the three value forms, interpolation with defaults, multiline values,
the comment-preserving document — and config-nv's `cfgdotenv` becomes
an adapter over it, four functions deep instead of a second parser.

Concretely, when both are implemented:

- `cfgdotenv.parse_dotenv(text, origin)` becomes
  `dotenvparse.parse` followed by `dotenvparse.pairs_of`, with the
  fault mapped — it already answers `[(Str, Str)]`, which is exactly
  what `pairs_of` answers.
- `cfgdotenv.read_dotenv(path)` becomes `dotenvload.load`.
- `cfgdotenv.dotenv_layer(rule, path, name, rank)` becomes
  `dotenvconf.layer_of` over that.
- `cfgdotenv.render_dotenv(pairs)` becomes
  `dotenvwrite.render_pairs` — and config-nv gains the
  comment-preserving `set` it does not have today.

`dotenvconf` is the seam that makes that a small change rather than a
rewrite: it hands config-core-nv's `EnvKeyRule` the pairs and adds
nothing of its own, so a layer built through this package and a layer
built through config-nv's own module are the same value.

**This is a report rather than an edit.**  config-nv is published and
is not this lane's package; the division above is what its maintainers
are being told about, and the row wants updating to say which package
owns the grammar.

## Out of scope, and the next row: `datafile-nv`

**Where a program's configuration, cache, data and state directories
are is not this package's question**, and it is the next row in this
section of the plan: `datafile-nv` (data/host, ports `directories` /
`platformdirs`).

It is a separate package because it is a separate kind of knowledge.
This one parses a format; that one knows that a program's
configuration lives in `$XDG_CONFIG_HOME/myapp` or
`~/.config/myapp` on Linux, `~/Library/Application Support/myapp` on
macOS, and `%APPDATA%\myapp` on Windows — four directory kinds times
three platforms, plus the XDG variables that override each of them,
plus the rule that a cache directory may be deleted at any moment and a
state directory may not.

The overlap is exactly one function: `dotenvload.find_upwards` walks
towards the filesystem root looking for a `.env`, which is a
*project*-relative search and not a platform directory at all.  A
program that wants `~/.config/myapp/config.toml` wants datafile-nv;
one that wants `./.env` wants this.  Neither should grow the other's
job — a dotenv library that knew about `%APPDATA%` would be one nobody
could review.

What `datafile-nv` would need from here: nothing.  What this package
would want from it: nothing either, and that is the check that the
split is in the right place.

## What this does not do, on purpose

- **It does not set environment variables.**  It cannot — `std.env`
  has no `set` — and it should not; see above.
- **It does not search for a file unless asked.**  `find_upwards` is a
  separate call, because a walk that surprised a caller by reading a
  file two directories above the one they named is a walk that reads
  somebody else's secrets.
- **It does not refuse a world-readable file.**  `is_world_readable` is
  a question a caller can ask; a container image where everything runs
  as one user is a perfectly good place for a mode-644 `.env`, and a
  library that refused would be wrong about it.
- **It does not guess types.**  `layer_of` builds string values,
  because a `.env` file has no types and guessing them is how
  `VERSION=1.10` becomes the number 1.1.  `layer_of_inferred` is the
  guess, named.
- **It does not do variable substitution in keys**, only in values.
- **It does not support `.env.local`, `.env.production` and the rest.**
  That convention is a framework's, and which files are loaded in which
  order for which environment is exactly the decision config-nv's
  `ConfigStack` exists to express.
- **No device claim.**  The package is `host`.

## The reference implementation

`dotenvy` (Rust) for the grammar — its handling of the three quoting
forms and of `${VAR}` with defaults is what this ports — and
`python-dotenv` for the operational surface, particularly its
`set_key`/`unset_key`, which is the writer this package is trying to
make good enough to use.

Three things change in the port.

Both references mutate the process environment and this one does not,
for the reason the `std.env` section gives.

`python-dotenv`'s `set_key` rewrites the file from a parsed map; here
the document keeps its lines and `set` replaces one of them, which is
what makes the writer something a person would let near a file they
maintain.

And both references default their override flag differently in
different entry points — `dotenv_values` versus `load_dotenv`, with and
without `override=True` — which is a thing people get wrong in both
directions.  Here there is one `DotenvOverride` argument, it appears on
every function that combines the two sources, and its default is the
one that keeps `FOO=bar ./myprogram` working.

## Status

| item | implemented |
| --- | --- |
| `dotenvparse` — `DotenvEntry`, `DotenvQuoting`, `DotenvLine`, `DotenvDocument`, `DotenvOptions`, `DotenvFault` | types only |
| `dotenvparse.default_options`, `.literal_options`, `.parse`, `.parse_document`, `.entries_of`, `.pairs_of` | no |
| `dotenvparse.lookup`, `.duplicate_keys`, `.key_is_valid`, `.value_ends_at`, `.unescape` | no |
| `dotenvparse.expand`, `.referenced_names`, `.unresolved_names`, `.default_of_reference`, `.name_of_reference` | no |
| `dotenvparse.fault_line`, `.entry`, `.entry_quoted`, the `message` impl | no |
| `dotenvwrite.render`, `.render_entry`, `.render_pairs` | no |
| `dotenvwrite.set`, `.set_quoted`, `.unset`, `.comment_out`, `.annotate` | no |
| `dotenvwrite.empty_document`, `.append`, `.append_comment`, `.append_blank` | no |
| `dotenvwrite.quoting_for`, `.needs_quoting`, `.escape`, `.can_single_quote` | no |
| `dotenvwrite.to_template`, `.missing_against` | no |
| `dotenvload` — `DotenvOverride`, `DotenvLoaded`, `DotenvLoadFault` | types only |
| `dotenvload.DOTENV_FILE_NAME`, `.DOTENV_TEMPLATE_NAME` | yes — they are constants |
| `dotenvload.load`, `.load_optional`, `.load_document`, `.save_document`, `.find_upwards`, `.is_world_readable` | no |
| `dotenvload.environment`, `.environment_value`, `.merged`, `.merged_optional`, `.combine` | no |
| `dotenvload.pairs_of_loaded`, `.value_of`, `.shadowed_keys`, `.resolver_over`, `.unresolved_in`, the `message` impl | no |
| `dotenvconf.layer_of`, `.layer_of_inferred`, `.value_of`, `.mapped_pairs`, `.skipped_entries`, `.bad_entries` | no |
| `dotenvconf.rule_for`, `.path_for`, `.key_for`, `.keys_for`, `.template_for` | no |
