# frame

A declarative project templater. `frame` scaffolds a new project from a git or local template using a small Liquid-like template engine with exact whitespace control, templated file and directory names, and an interactive question flow.

A template is a plain directory (or git repo) with a `frame.edn` config and a `template/` directory that mirrors the generated project one-to-one. All template logic lives in three declarative places: variables in `frame.edn`, tags in file contents, and tags in file and directory names. There is no code in templates.

## Usage

```
brew install abogoyavlensky/tap/frame
frame new <source> [name] [options]
```

`<source>` is a local directory path or an `https://host/owner/repo` git URL. `[name]` is the project name; omit it to be prompted. The name must match `^[a-z][a-z0-9-]*$`.

**Options**

| Option | Effect |
| --- | --- |
| `--defaults` | Answer every remaining question with its default. Requires `name` as the positional argument. |
| `--dir <path>` | Write to this exact directory instead of `./<name>`. Does not change the `project-name` value used inside the template. |
| `--var key=value[,key=value...]` | Pre-answer variables and skip their prompts. Booleans accept `true`/`false`. `project-name` is not allowed. |

Examples:

```bash
# Interactive: prompts for the name and every variable
frame new ./my-template

# Non-interactive: name from the positional arg, everything else default
frame new ./my-template my-app --defaults

# Preset some answers, prompt for the rest
frame new ./my-template my-app --var db=postgres,auth=false

# Write somewhere other than ./my-app
frame new ./my-template my-app --defaults --dir build/out

# From a git URL (cloned and cached under ~/.cache/frame)
frame new https://github.com/owner/repo my-app
```

On success `frame` prints the created-file count, the target directory, a summary of the declared template variables with the values that were used, and the next step. A resolution, configuration, or generation error prints `frame: <message>` and exits 1. Cancelling a prompt exits 1 and writes nothing. Invalid command-line syntax (an unknown option or a missing `<source>`) is reported by the argument parser and exits 2.

## Installation

### With [Homebrew](https://brew.sh)

Works on macOS and Linux:

```sh
brew install abogoyavlensky/tap/frame
```

### With [mise](https://mise.jdx.dev)

```sh
mise use -g github:abogoyavlensky/frame@latest
```

Or pin a version in `.mise.toml`:

```toml
[tools]
"github:abogoyavlensky/frame" = "latest"
```

### Manual

Download the archive for your platform from the
[releases page](https://github.com/abogoyavlensky/frame/releases), extract it, and put
`frame` on your `PATH`:

## Writing a template

A template repo has two parts at its root:

```
frame.edn        the config: variables, computed values, raw globs
template/         mirrors the generated project one-to-one
```

Everything under `template/` is copied to the output. File and directory names and file contents are rendered by the same engine.

### frame.edn

```edn
{:description "Lightweight Clojure stack"       ; optional
 :min-frame-version "0.1.0"                     ; optional
 :root "template"                               ; optional, default "template"
 :vars [{:key :db :prompt "Database" :type :enum
         :options ["sqlite" "postgres"] :default "sqlite"}
        {:key :auth :prompt "Include authentication?" :type :boolean :default true}
        {:key :author :prompt "Author name" :type :string :default ""
         :validate {:pattern "^.+$" :msg "Author name cannot be empty"}}]  ; :validate optional
 :computed {:top-ns "{{ project-name | snake_case }}"}
 :raw ["**/*.png"]}
```

- **`:vars`** is ordered; the order is the question order. Each var needs a `:key` (keyword), a `:prompt` (string), and a `:type`:
  - `:string` renders a text input. An optional `:validate {:pattern "..." :msg "..."}` rejects input that does not match the pattern.
  - `:enum` renders a picker. It needs a non-empty `:options` vector; `:default` must be one of them.
  - `:boolean` renders a yes/no choice.
- **`project-name`** is a built-in first variable. You never declare it. It comes from the positional `[name]` argument or its own prompt, and it is available in every template and computed value.
- **`now-date`** and **`now-year`** are built-in variables holding the generation date (`2026-07-10`) and year (`2026`). Like `project-name`, they are always available, and all three names are reserved: declaring them in `:vars` or setting them with `--var` is an error.
- **`:min-frame-version`** is an optional exact semver string (`"0.3.0"`) naming the oldest frame release the template works with. An older frame refuses to generate — before any prompting — and asks the user to upgrade. Anything other than an exact `X.Y.Z` string is a config error.
- **`:computed`** maps variable names to templates rendered against the answers after all questions. Computed values run in a single pass, so one computed value cannot reference another.
- **`:raw`** is a list of glob patterns (relative to `:root`) copied verbatim, without content rendering. `*` matches within a path segment; `**` matches across segments. Any file with a null byte in its first 8000 bytes is also copied verbatim. Raw and binary files still get their *paths* rendered.

A missing or unparsable `frame.edn`, a too-old frame for `:min-frame-version`, an unknown `:type`, a missing `:options` on an enum, or a `:default` outside `:options` is reported before any prompting. An unrecognized top-level key in `frame.edn` prints a warning on stderr but does not block generation.

### Template syntax

**Output** substitutes a variable, with optional chained filters:

```liquid
{{ project-name }}
{{ project-name | snake_case }}
{{ project-name | snake_case | upper }}
```

Whitespace inside the delimiters is insignificant. An unresolved variable in an output tag is an error.

Filters: `snake_case`, `kebab_case`, `camel_case`, `pascal_case`, `lower`, `upper`, `capitalize`.

**If blocks** select the first branch whose condition is truthy:

```liquid
{% if auth %}enabled{% elsif guest %}guest{% else %}off{% endif %}
```

**Case blocks** match a variable against literal values:

```liquid
{% case db %}
{% when sqlite %}SQLite
{% when postgres %}PostgreSQL
{% else %}unknown
{% endcase %}
```

`when` values are bare or quoted (`sqlite` or `"sqlite"`). Text between `{% case %}` and the first `{% when %}` is discarded.

**Conditions** support `and`, `or`, `not`, `==`, and `!=`. `and` binds tighter than `or`; there are no parentheses.

```liquid
{% if db == postgres and auth %}...{% endif %}
{% if db != sqlite or verbose %}...{% endif %}
{% if not auth %}...{% endif %}
```

An identifier resolves against the variables. An unresolved identifier in a comparison is treated as a bare string literal, so `db == postgres` works without quotes (useful in file names). A bare identifier on its own tests truthiness. `false`, `nil`, and `""` are falsy; everything else is truthy.

**Raw blocks** emit their content verbatim. Use them when a file mixes frame tags with another tool's template syntax — a GitHub Actions workflow, for example:

```liquid
name: {{ project-name }}
{% raw %}
token: ${{ secrets.TOKEN }}
{% endraw %}
```

Nothing between `{% raw %}` and `{% endraw %}` is parsed, so it may contain anything — including text that would otherwise be a frame syntax error. The delimiters themselves follow the whitespace rule below: alone on a line they leave no trace; inline they are removed in place. The first `{% endraw %}` always ends the block, so raw blocks cannot nest or contain that literal text. An unclosed `{% raw %}` is a syntax error.

Raw blocks protect spans inside a rendered file; the `:raw` globs in `frame.edn` copy whole files verbatim. Use globs for fully foreign files and raw blocks for mixed ones.

### The whitespace rule

A tag that is the only non-whitespace content on its line owns the whole line. The tag consumes its leading whitespace, the tag text, and the trailing newline. A false branch therefore leaves no trace, and a taken branch lands exactly as written.

Template:

```liquid
# {{ project-name }}

{% if auth %}
Authentication is included.
{% endif %}
Done.
```

With `auth` true:

```
# my-app

Authentication is included.
Done.
```

With `auth` false:

```
# my-app

Done.
```

No phantom blank line appears where the `if` block was. A tag mixed with other content on a line is removed in place and never owns the line. Output tags (`{{ }}`) never own a line; they render where they sit.

### Templated and conditional paths

Every path segment under `template/` is rendered. Use substitution and filters in names:

```
src/{{ project-name | snake_case }}/core.clj
```

If any rendered segment is empty, `frame` skips that file (and the whole subtree under an empty directory segment). This drives conditional files and directories. Two sibling directories can target the same output path, and exactly one survives:

```
resources/{% if db == sqlite %}migrations{% endif %}/0001.sql
resources/{% if db == postgres %}migrations{% endif %}/0001.sql
```

With `db` = `sqlite`, the first renders to `resources/migrations/0001.sql` and the second is skipped. Bare literals in conditions keep quote characters out of file names.

If two files render to the same destination and both survive, generation fails and names both sources. A rendered path that would escape the target directory is rejected.

## Sources and caching

- A `<source>` starting with `https://` must have the form `https://host/owner/repo` (an optional trailing `.git` is stripped). `frame` resolves the default-branch HEAD, clones it, and caches it under `~/.cache/frame/templates/<host>/<owner>/<repo>/<sha>/` (honoring `XDG_CACHE_HOME`), reusing the cache on later runs. The cache is safe to delete — frame re-clones on the next use. Set `FRAME_HOME` to override the base directory; templates then live under `$FRAME_HOME/templates/...`.
- Any other `://` scheme (`ssh://`, `git://`, `git@host:` shorthand) is rejected.
- Anything else is a local directory path, used in place. This is the fast path for developing a template.

## Shell completions

`frame completion <shell>` prints a completion script for bash, zsh, or fish.
Completions cover the subcommands (`new`, `completion`, `help`) and the options
of `new` (`--defaults`, `--dir`, `--var`). The `<source>` argument falls back to
the shell's own filename completion, which is what you want for a local template
path.

Bash — add to `~/.bashrc`:

```sh
source <(frame completion bash)
```

Zsh — either source it the same way in `~/.zshrc` (after `compinit`), or
drop it on your `fpath`:

```sh
mkdir -p ~/.zfunc
frame completion zsh > ~/.zfunc/_frame
```

and make sure `~/.zshrc` contains `fpath+=~/.zfunc` before `compinit` runs.

Fish:

```sh
frame completion fish > ~/.config/fish/completions/frame.fish
```

## Projects using frame

- [clojure-stack-lite](https://github.com/abogoyavlensky/clojure-stack-lite)
- [clojure-lib-template](https://github.com/abogoyavlensky/clojure-lib-template)

## Development

Install dependencies with [mise](https://mise.jdx.dev/getting-started.html) (or read `.mise.toml` and install them manually):

```bash
mise trust && mise install
```

Common commands:

```bash
lgx run -- new --defaults ./path/to/template my-app   # run from source
lgx test                                               # run tests
lgx check                                              # fmt check + lint + test
lgx fmt                                                # format
lgx lint                                               # lint
lgx build                                              # build bin/frame
```

## License

MIT License. Copyright (c) 2026 Andrey Bogoyavlenskiy.
