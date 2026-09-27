# dotenv-nv

A `.env` file is a list of `NAME=value` lines that a program reads at
startup to find its settings. The format has no specification. What it
means is what
[dotenvy](https://github.com/allan2/dotenvy) in Rust and
[python-dotenv](https://github.com/theskumar/python-dotenv) do, and this
package ports those two. It reads the format, edits a file without
losing its comments, and hands the result to
[config-core-nv](https://novo-lang.org/packages/config-core-nv) for a
program that stacks several sources of configuration.

## What it is

A file is a sequence of lines. A line is blank, a comment beginning
with `#`, or an **assignment**: a name, an `=`, and a value. A value is
written in one of three forms, and the three do not mean the same
thing.

| Written | What it means |
| --- | --- |
| `A=a\nb` | Bare. A `#` after whitespace ends the value; a `#` with no whitespace in front of it does not. |
| `A='a\nb'` | Single-quoted. Literal throughout: no escape sequences, no interpolation, and no way to write a single quote inside. |
| `A="a\nb"` | Double-quoted. The escape sequences `\n`, `\r`, `\t`, `\\`, `\"`, `\'` and `\$` are processed and `${…}` is interpolated. |

`A='a\nb'` is therefore four characters including a backslash, and
`A="a\nb"` is three characters including a newline.

**Interpolation** is `${NAME}` inside a double-quoted value, replaced
by that name's value. `${NAME:-text}` supplies a default for when the
name has no value at all. The lookup order is the file's own earlier
assignments first, then the process environment, which is what lets a
file build `${BASE}/x` out of a line above it. A reference to a later
line finds nothing, so a file cannot loop.

An assignment may carry the prefix `export`, so that the same file can
be read by a shell with `source`. It means nothing to a parser.

A **document** is the file with every line kept as it was written,
including the comments, the blank lines and the order. A program that
changes one value rewrites one line, and the rest of the file comes
back byte for byte.

An **override policy** decides what wins when a name is in both the
file and the process environment. `DotenvKeepEnvironment` gives it to
the environment, which is what makes `DATABASE_URL=… ./myprogram` work.
`DotenvFileWins` gives it to the file, for a deployment tool that wrote
the file and means it.

## Install

```
novo pkg add dotenv-nv
```

## Example

```novo
use dotenvload
use dotenvparse

fn main() [fs, io]
    // Read ./.env if it is there, and combine it with the process
    // environment. The environment wins, so `FOO=bar ./program` still
    // overrides one setting for one run.
    match dotenvload.merged_optional(dotenvload.DOTENV_FILE_NAME,
                                     DotenvKeepEnvironment,
                                     dotenvparse.default_options())
        Err(f)     => println(f.message())
        Ok(loaded) =>
            // One setting, by name.
            match dotenvload.value_of(loaded, "DATABASE_URL")
                None    => println("DATABASE_URL is not set")
                Some(u) => println(u)

            // The keys the file assigned that the environment already
            // had. This is why an edit to the file did nothing.
            for key in dotenvload.shadowed_keys(loaded)
                println("${key} came from the environment")
```

Build and test with `novo pkg build` and `novo test tests/`.

## What the package contains

| Module | Contents |
| --- | --- |
| `dotenvparse` | The grammar: the three value forms, the escape rules, interpolation, the options a parse takes, and every fault with the line it is on. |
| `dotenvwrite` | The writer: render a document, replace one assignment, comment one out, add a note beside one, and turn a file into an example with the keys and none of the values. |
| `dotenvload` | The file and the process environment: read one, write one back, search upwards for one, and combine the two under a policy. |
| `dotenvconf` | The conversion into config-core-nv's value tree, so a `.env` can be stacked with the other sources of a program's configuration. |

## How to choose an entry point

**`dotenvload.merged` and `merged_optional` are what a program calls at
startup.** They read the file, read the environment, apply the policy,
and answer the pairs to use. `merged_optional` treats a missing file as
the ordinary case, because a developer's machine has a `.env` and a
production host does not.

**`dotenvload.load` reads the file alone.** It declares `[fs]` and
never touches the environment, for a caller that only wants what the
file says.

**`dotenvparse.parse` takes text the caller already holds.** It
declares nothing at all. Use it when the bytes came from somewhere
else. `dotenvparse.parse_with` does the same with an environment the
caller passes in as a list of pairs.

**`dotenvload.load_document` and `dotenvwrite` are for changing a
file.** The document keeps every line, `dotenvwrite.set` replaces one
of them, and `dotenvload.save_document` writes it back.

**`dotenvconf.layer_of` turns entries into a configuration layer.** Use
it when a `.env` is one of several sources and something else decides
which wins.

## The rules a user needs

1. **A `#` in a bare value ends it only after whitespace.**
   `PASSWORD=abc#123` is the seven characters `abc#123`.
   `PASSWORD=abc #123` is the three characters `abc`. One space changes
   the password, and the program does not fail: it authenticates as
   nobody. `dotenvparse.value_ends_at` is that rule, exposed, so a
   caller can run it.
2. **The three value forms have three different escape rules.** See the
   table above. A single-quoted value cannot contain a single quote.
3. **`${NAME}` with no value expands to nothing, and is not an error.**
   `${NAME:-text}` uses its default only when the name has no value at
   all; an empty value stays empty, as in python-dotenv.
   That is what a shell does and what existing files assume. A
   connection string with an empty `${DB_PASSWORD}` connects somewhere
   rather than failing, so `dotenvparse.unresolved_names` and
   `dotenvload.unresolved_in` answer which names were empty. A program
   that wants a missing name to be fatal asks for that list at startup.
4. **Interpolation happens in double-quoted values only.** It can be
   turned off in `DotenvOptions` for a file the caller does not
   control, because interpolation pulls the process environment into a
   value.
5. **`export FOO=bar` is an assignment to `FOO`.** A parser that kept
   the prefix would produce a variable called `export FOO`.
6. **The same key assigned twice is not a fault, and the last one
   wins.** `dotenvparse.duplicate_keys` answers them, so a caller that
   wants to refuse can.
7. **This package never sets an environment variable.** The standard
   library has no `env.set`; setting one is `process.env_set`, in
   another module. Loading a file here means reading values. A program
   that takes its configuration as a value can be tested twice in one
   process with two different configurations.
8. **The environment wins by default.** `DotenvKeepEnvironment` is the
   policy that keeps `FOO=bar ./myprogram` working. It is an argument
   on every function that combines the two sources, so there is one
   spelling of the decision rather than one per entry point.
9. **`shadowed_keys` is the answer to "I changed the file and nothing
   happened".** It lists the keys the file assigned that the
   environment already had.
10. **The interpolation resolver is a function the caller supplies.**
    `dotenvparse.expand` takes a `fn(Str) -> ?Str`, which is what keeps
    the module free of effects. `dotenvload.resolver_over` builds the
    ordinary one: the file's earlier entries first, then the
    environment.
11. **A document round-trips.** `dotenvwrite.render` of
    `dotenvparse.parse_document` of a file's text is that text. The
    one exception is a file that mixes `\n` and `\r\n` endings, which
    comes back with its first line's ending throughout.
    `dotenvwrite.set` replaces one line and leaves the rest as they
    were, so changing one value produces a one-line difference. A
    changed line is written in the narrowest form that reads back as
    the new value.
12. **Nothing searches for a file unless asked.**
    `dotenvload.find_upwards` walks towards the filesystem root, and it
    is a separate call, because a walk that surprised a caller by
    reading a file two directories up is a walk that reads somebody
    else's secrets.
13. **A world-readable file is reported and not refused.**
    `dotenvload.is_world_readable` is a question a caller may ask. A
    container image where everything runs as one user is a good place
    for a `.env` anyone on the machine can read.
14. **`save_document` writes through a temporary file and a rename.** A
    process that dies mid-write leaves the previous file rather than
    half of the new one. An existing file keeps its permissions, and a
    new one is readable by its owner alone.
15. **Values are strings, and nothing is guessed.**
    `dotenvconf.layer_of` builds string values, because a `.env` file
    has no types and guessing them turns `VERSION=1.10` into the number
    1.1. `dotenvconf.layer_of_inferred` is the guess, under its own
    name.
16. **A parse is bounded.** `DotenvOptions` carries the largest file
    accepted. `dotenvparse.expand` follows an answer that itself holds a
    reference, up to `max_expansions` deep, and past that answers
    `DotenvCircularInterpolation`: `A=${B}` with `B=${A}` is refused
    rather than followed forever.
17. **Every fault carries the line it is on.** A `.env` file is a file
    a person edits, and `dotenvparse.fault_line` is what sends them to
    it. With `strict` off in `DotenvOptions`, a line that cannot be read
    is kept as a skipped line with its fault, and the rest of the file
    is read.

## What is not included

- **Setting the process environment.** See rule 7.
- **Searching for a file by default.** See rule 12.
- **Refusing a world-readable file.** See rule 13.
- **Type inference.** See rule 15.
- **Substitution in keys.** `${…}` is expanded in values only.
- **`.env.local`, `.env.production` and the rest of the convention.**
  Which files are read in which order for which deployment is a
  decision [config-nv](https://novo-lang.org/packages/config-nv)'s
  stack exists to express.
- **A microcontroller build.** The package's subject is a file on a
  host.

## Related packages

- [config-core-nv](https://novo-lang.org/packages/config-core-nv) is
  the value tree and the precedence rules that a layered configuration
  is made of. This package depends on it for one conversion:
  `dotenvconf.layer_of` hands it entries and adds nothing of its own.
- [config-nv](https://novo-lang.org/packages/config-nv) is the layered
  front that stacks a `.env` beside a TOML file and the process
  environment. Its `cfgdotenv` module carries a `.env` parser of its
  own, published before this package existed. Both can be in one
  program; a program that wants the grammar, the comment-preserving
  document or the writer wants this one.
- [datafile-nv](https://novo-lang.org/packages/datafile-nv) answers
  where a program's configuration directory is on each platform. This
  package reads `./.env`, which is a path relative to a project and not
  a platform directory at all.
- [toml-nv](https://novo-lang.org/packages/toml-nv),
  [yaml-nv](https://novo-lang.org/packages/yaml-nv) and
  [ini-nv](https://novo-lang.org/packages/ini-nv) are the other
  configuration formats on the registry. They carry nesting and types;
  a `.env` file is a flat list of strings.
- `std.env` in the standard library reads the process environment. It
  is what `dotenvload.environment` calls, and it declares `[io]`,
  because the environment sits with pipes and processes in SPEC section
  5.1.

## Tests

```bash
novo test tests/spec_tests.nv          #  7 tests: the grammar against its references
novo test tests/dotenvparse_tests.nv   # 24 tests: values, interpolation and faults
novo test tests/dotenvwrite_tests.nv   #  9 tests: rendering and the one-line edits
novo test tests/dotenvload_tests.nv    # 12 tests: files, the environment and the policy
novo test tests/dotenvconf_tests.nv    #  3 tests: the config-core-nv layer
bash tests/coverage.sh                 # line coverage over src/
```

The grammar's reference cases are python-dotenv's parser tests and
dotenvy's quoting rules. `tests/spec_tests.nv` holds each case with the
value it parses to. Where python-dotenv and this package agree, the
expected value was checked against python-dotenv 1.0.1's
`dotenv_values`. Where they differ, the test says so and follows
dotenvy: a single-quoted value is not interpolated, `\$` in double
quotes is a `$`, a bare value is not interpolated, and a quoted key or a
line with no `=` is refused.

The suites also assert that every accepted file renders back byte for
byte, that a value changed by `set` reads back as itself in every
quoting form, that `save_document` keeps a file's permissions, and that
the environment shadows the file under the default policy.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
