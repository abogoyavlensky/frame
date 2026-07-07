# Frame Initial Implementation Plan

> **Status: COMPLETE** (all 12 tasks). See the completion summary at the end.

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `frame`, a declarative project-templater CLI in let-go: a Liquid-like template engine with exact whitespace control, templated file/dir names, an EDN template config, and an interactive tiny-tui question flow behind `frame new <source> [name]`.

**Tech Stack:** let-go (lgx project), tiny-cli (already a dep), tiny-tui (to add), git via `os/sh`.

---

## Design

### Overview

A single `frame` binary with one core command: `frame new <source> [name]`. A template is a plain git repo (or local dir) containing a `frame.edn` config at the root and a `template/` directory that mirrors the generated project 1:1. All template logic lives in three declarative places: variables in `frame.edn`, tags in file contents, and tags in file/dir names. No code in templates, no substitution files, no transform tables.

The design target is migrating the clojure-stack-lite template: its substitution files become inline `{% if %}` blocks, its `EXTENSIONS`/`RENAMINGS` transform tables become conditional dir names, its `data-fn` validation becomes `:enum` vars, and its fifteen empty-string default vars become unnecessary because untaken branches emit nothing.

### Template syntax

Liquid-shaped and deliberately tiny. No for-loops, no template inheritance, no includes.

- **Output:** `{{ var }}`, with chainable filters: `{{ project-name | snake_case }}`. Whitespace inside delimiters is insignificant.
- **If blocks:** `{% if cond %}` / `{% elsif cond %}` / `{% else %}` / `{% endif %}`.
- **Case blocks:** `{% case var %}` / `{% when val %}` / `{% else %}` / `{% endcase %}`. `when` values are bare or quoted literals. Text between `{% case %}` and the first `{% when %}` is discarded.
- **Conditions grammar** (recursive descent, `and` binds tighter than `or`):

  ```
  cond       := or-expr
  or-expr    := and-expr ("or" and-expr)*
  and-expr   := unary ("and" unary)*
  unary      := ["not"] comparison
  comparison := term (("==" | "!=") term)?
  term       := identifier | quoted-literal | bare-literal
  ```

  An identifier resolves against the vars map; an unresolved identifier in a comparison position is treated as a bare string literal (so `db == postgres` works without quotes — important for file names). A bare comparison of a variable tests truthiness. No parentheses.
- **Truthiness:** `false`, `nil`, and `""` are falsy; everything else truthy.
- **Filters v1:** `snake_case`, `kebab_case`, `camel_case`, `pascal_case`, `lower`, `upper`, `capitalize`. Implemented as a plain name→fn registry map so adding a filter is one entry. Unknown filter is a render error.
- **Values:** answers are strings (`:string`, `:enum`) or booleans (`:boolean`). Filters coerce input with `str`.

### Whitespace rule (the key feature)

A tag that is the only non-whitespace content on its line **owns the whole line**: the tag consumes the leading whitespace, the tag text, and the trailing newline. Consequences:

- A false `if` branch contributes nothing — no phantom blank lines, ever.
- A taken branch's text lands exactly as written in the template.
- Tags mixed with content on a line are removed in place (zero-width).
- Output `{{ }}` expressions are never line-owning; they always render in place.

This is implemented in the lexer: after tokenizing, a pass marks tag tokens as line-owning when the line around them holds only whitespace, and widens their spans to cover the leading whitespace and trailing newline.

### Conditional and templated file/dir names

Every path segment of every file under `template/` is rendered by the same engine (single-line mode: the whitespace rule does not apply; tags are simply zero-width).

- **Substitution + filters:** `src/{{project-name | snake_case}}/core.clj`.
- **Empty-segment-skip:** if any rendered segment is the empty string, the file (or entire dir subtree) is skipped. So `{% if auth %}auth{% endif %}/handlers.clj` exists only when `auth` is true, and sibling source dirs `{% if db == sqlite %}migrations{% endif %}` and `{% if db == postgres %}migrations{% endif %}` both target `resources/migrations` with exactly one surviving.
- Bare literals in conditions keep quote characters out of file names.
- If two files render to the same destination path (and both survive), generation fails with an error naming both sources.

### `frame.edn` template config

Lives at the template repo root, next to `template/`:

```edn
{:description "Lightweight Clojure stack"
 :root "template"                          ; optional, this is the default
 :vars [{:key :db :prompt "Database" :type :enum
         :options ["sqlite" "postgres"] :default "sqlite"}
        {:key :auth :prompt "Include authentication?" :type :boolean :default true}
        {:key :author :prompt "Author name" :type :string :default ""
         :validate {:pattern "^.*$" :msg "..."}}]   ; :validate optional
 :computed {:top-ns "{{project-name | snake_case}}"}
 :raw ["**/*.png"]}
```

- `:vars` is ordered; order = question order. Types: `:string`, `:enum`, `:boolean`.
- `project-name` is a **built-in** first var (never declared in `:vars`): taken from the positional `[name]` arg or prompted first. Validated against `^[a-z][a-z0-9-]*$`.
- `:computed` maps var keys to template strings rendered against the answers (after all questions). Computed vars must not reference each other (single pass); referencing one yields a render error.
- `:raw` is a list of glob patterns (relative to `:root`, `*` = within segment, `**` = across segments) copied verbatim. Anything failing a null-byte binary sniff (first 8000 bytes) is also copied verbatim. Raw/binary files still get their *paths* rendered.
- A missing or unparsable `frame.edn`, an unknown `:type`, a missing `:options` on enum, or a `:default` not in `:options` is a config error reported before any prompting.

### Interactive flow

One tiny-tui `with-inline-session` wraps the whole question flow (no flicker, gum-style, drawn at the cursor):

- `:string` → `tui/input` with `:value` default and `:validate` from the pattern.
- `:enum` → `tui/select` with `:cursor-item` on the default.
- `:boolean` → `tui/confirm`.
- Cancel at any prompt aborts the run (exit 1, nothing written).
- `--var key=value` (repeatable) pre-answers a var and skips its question. Boolean vars accept `true`/`false`. `--var project-name=...` is rejected — the project name comes only from the positional arg or its prompt.
- `--defaults` answers every remaining question with its default (project name still required via positional arg). This is also how e2e tests run — no TTY needed.

### Sources and git handling

- A `<source>` starting with `https://` is a git URL of the form `https://host/owner/repo` (optional trailing `.git`): resolve default-branch HEAD via `git ls-remote`, shallow-clone (`git clone --depth 1`, checkout sha) into `~/.frame/templates/<host>/<owner>/<repo>/<sha>/`, reuse when cached. `FRAME_HOME` env overrides `~/.frame`.
- Any other `://` scheme (`ssh://`, `git://`, `git@host:` shorthand) is rejected with a clear error naming the supported form — same policy as lgx.
- Anything else is a local directory path, used in place (essential for template development).
- Follow lgx's `cache.lg`/`new.lg` idioms (`os/sh` for git, prefix-free error messages carrying `:stderr`).

### Generation pipeline

1. Resolve source → template root dir; read + validate `frame.edn`.
2. Determine target dir: `--dir <path>` when given is the **exact output directory**; otherwise `./<project-name>`. Fail if it exists non-empty (before prompting for remaining vars where possible — name is known first). `--dir` changes only where files are written; `project-name` still supplies the var used inside paths and contents, and the summary prints the actual target.
3. Collect answers: positional name → `--var` overrides → prompts (or defaults).
4. Render `:computed` vars into the answers map.
5. Walk `:root`; for each file: render its relative path (skip on empty segment), then render content (or copy raw/binary). **All in memory first.**
6. Detect destination collisions; on any render/config error, abort with `frame: <file>:<line>: <message>` and write nothing.
7. Write all files, `mkdir -p` parents; print created-file count and next steps.

### Error handling

- Template syntax errors (unclosed tag, unknown tag, mismatched `endif`, unknown filter, unresolved var in output position) report template-relative `file:line` and abort before writing.
- Unresolved var in an output `{{ }}` is an error; in a condition it's falsy (and a bare-literal candidate in comparisons).
- CLI errors follow lgx style: `frame: <message>` on stderr, exit 1.

### Testing strategy

- Engine (lexer/parser/render/filters/conditions) is pure — no IO — and unit-tested exhaustively, especially the whitespace rule.
- Path rendering and glob matching unit-tested.
- `generate` gets a golden end-to-end test: a fixture template under `test/fixtures/demo-template/` rendered with defaults into a temp dir, compared file-by-file against `test/fixtures/demo-expected/`. The whitespace guarantees live in these goldens (byte-exact comparison).
- Run everything with `lgx test`; keep `lgx check` (fmt + lint + test) green at every commit.

---

## File Structure

```
main.lg                          — tiny-cli app spec: `new` command, flags
src/frame/
  template/
    lexer.lg                     — tokens {:type :text|:output|:tag, :value, :line}, line-ownership pass
    filters.lg                   — filter registry map + case-conversion fns
    parser.lg                    — tokens → AST; condition mini-parser
    render.lg                    — AST + vars → string; render-string entry; render-segment (path mode)
  config.lg                      — read/validate frame.edn → {:vars [...] :computed {} :raw [] :root ""}
  prompt.lg                      — answers collection: --var parsing, defaults, tiny-tui session
  source.lg                      — resolve local path / git URL, clone + cache under ~/.frame
  glob.lg                        — minimal glob matcher (* and **) for :raw
  generate.lg                    — walk, render paths+contents in memory, collision check, write
  new.lg                         — `new` handler: orchestration + error reporting
  core.lg                        — (existing scaffold; delete greet once new.lg lands)
test/frame/
  template/lexer_test.lg
  template/filters_test.lg
  template/parser_test.lg
  template/render_test.lg
  config_test.lg
  prompt_test.lg
  glob_test.lg
  generate_test.lg
  fixtures/demo-template/        — frame.edn + template/ exercising every feature
  fixtures/demo-expected/        — golden output tree
```

Dependency order: filters → lexer → parser → render (engine, pure) ; glob, config, source (independent) ; prompt → generate → new → main.

---

### Task 1: Project setup — add tiny-tui dep, clean scaffold

**Files:**
- Modify: `lgx.edn`
- Modify: `README.md`

- [x] **Step 1: Add tiny-tui to deps**
  In `lgx.edn` `:deps`, add `tiny-tui {:git/url "https://github.com/abogoyavlensky/tiny-tui" :git/tag <latest tag>}` (check latest with `git ls-remote --tags https://github.com/abogoyavlensky/tiny-tui`). Run `lgx install`.
  > Deviation: latest tag is `v0.1.3`.

- [x] **Step 2: Update README intro**
  Replace the template description line with one sentence: frame is a declarative project templater; keep the development commands section.

- [x] **Step 3: Verify project runs**
  Run: `lgx test` — Expected: existing greet test passes.

- [x] **Step 4: Commit**
  `git commit -m "chore: add tiny-tui dependency"`

### Task 2: Filters

**Files:**
- Create: `src/frame/template/filters.lg`
- Test: `test/frame/template/filters_test.lg`

- [x] **Step 1: Write failing tests**
  Cover each filter through the public `(apply-filter name value)` fn: `snake_case` / `kebab_case` / `camel_case` / `pascal_case` from mixed inputs (`"my-app"`, `"my_app"`, `"MyApp"`, `"my app"`), `lower`, `upper`, `capitalize`; non-string input coerced via `str`; unknown filter throws `ex-info` with `{:reason :unknown-filter :filter name}`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/template/filters_test.lg` — Expected: FAIL (namespace not found).

- [x] **Step 3: Implement**
  A `filters` map of name→fn plus `apply-filter`. Case conversions share one tokenizer: split input into words on `-`, `_`, whitespace, and lower/upper camel boundaries; each case filter joins words its own way. Extension point is adding a map entry.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/template/filters_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: template filters with case conversions"`
  > Deviation: added `.clj-kondo/config.edn` (adapted from lgx) so `lgx check` lint passes — clj-kondo can't parse let-go's typeless `(catch e ...)` without it. Needed project-wide.

### Task 3: Lexer with line-ownership

**Files:**
- Create: `src/frame/template/lexer.lg`
- Test: `test/frame/template/lexer_test.lg`

- [x] **Step 1: Write failing tests**
  `(tokenize s)` returns a vector of tokens `{:type :text|:output|:tag :value <inner trimmed string> :line <1-based>}` (text tokens carry raw text in `:value`). Cases: plain text; `{{ var }}` mid-line; `{% if x %}` mid-line (zero-width: surrounding text preserved exactly); **a tag alone on a line consumes leading whitespace and the trailing newline** (the emitted text tokens around it prove it); two tags on one line are both non-owning; an output tag alone on a line is NOT line-owning; unterminated `{{` or `{%` throws `ex-info` with `{:reason :syntax :line n}`; `\n` inside text preserved; correct `:line` numbers across multi-line input.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/template/lexer_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  Single scan with `str/index-of` finding the next `{{` or `{%` (no regex needed): emit text token, find matching `}}`/`%}` (unterminated → throw), emit output/tag token, track line count. Then the ownership pass: for each `:tag` token, if the text token before it ends with `\n` (or is start-of-input) followed only by spaces/tabs, and the text after it starts with optional spaces/tabs then `\n` (or end-of-input), strip that whitespace from the neighbors including the trailing newline. Keep the two phases as separate fns; only `tokenize` is public.
  > Deviation: emptied text tokens are dropped after stripping (cleaner stream); text-token `:line` is the pre-strip line (only tag/output lines feed errors, so always accurate). Disjoint-strip reasoning documented in the source.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/template/lexer_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: template lexer with line-owning tag whitespace rule"`

### Task 4: Parser — AST and condition grammar

**Files:**
- Create: `src/frame/template/parser.lg`
- Test: `test/frame/template/parser_test.lg`

- [x] **Step 1: Write failing tests**
  Two public fns. `(parse-cond s)` → condition AST; cover: bare var, `not x`, `x == y` bare and `x == "y"` quoted, `!=`, `a and b`, `a or b and c` (and binds tighter), garbage → `ex-info {:reason :syntax}`. `(parse tokens)` → node tree; nodes are `{:node :text :value s}`, `{:node :output :var k :filters [names] :line n}`, `{:node :if :branches [{:cond ast :body [nodes]}...] :else [nodes]}`, `{:node :case :var k :whens [{:value s :body [nodes]}...] :else [nodes]}`. Cover: nesting if-in-if; `elsif` chains; case with else; errors with `:line`: `endif` without `if`, unclosed `if` at EOF, `elsif` after `else`, `when` outside `case`, unknown tag word.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/template/parser_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  Tag inner text split on whitespace to dispatch the keyword (`if`/`elsif`/`else`/`endif`/`case`/`when`/`endcase`). Recursive-descent block parser over the token vector (index-passing, since let-go is functional); condition parser tokenizes on whitespace with `==`/`!=` as their own tokens, then the grammar from the design. Output tokens split on `|`, first part the var, rest filter names.
  > Deviation: condition tokenizer is a char scanner (not a plain whitespace split) so quoted literals may contain spaces and `db==postgres` works unspaced. Term nodes tagged `:kind :ident|:str`; the renderer resolves idents and falls back to the ident string in comparisons.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/template/parser_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: template parser with if/case blocks and condition grammar"`

### Task 5: Renderer

**Files:**
- Create: `src/frame/template/render.lg`
- Test: `test/frame/template/render_test.lg`

- [x] **Step 1: Write failing tests**
  Public entry `(render-string template vars)` (string in, string out, composing lexer+parser internally) plus `(render-segment segment vars)` for paths. Cases:
  - substitution and chained filters;
  - unresolved var in output → `ex-info {:reason :unresolved-var :line n}`;
  - if/elsif/else selection; truthiness of `false`/`""`/missing var; `not`; `==` with bare and quoted literal; unresolved identifier on either side of `==` compared as bare literal; `and`/`or` precedence;
  - case/when/else, no matching when and no else → empty;
  - **whitespace goldens** (byte-exact `=` on multi-line strings): a 5-line template whose false if-branch leaves zero trace; taken branch preserved verbatim including indentation; inline if within a line; nested if inside case;
  - `render-segment`: tags zero-width, `""` result on false condition, filters work.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/template/render_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  Tree-walk: text → verbatim; output → lookup (throw if absent), thread through `apply-filter`, `str` the result; if → evaluate branch conds in order; case → `str` the var value, compare against when values. Condition evaluator resolves identifiers against vars (missing → `nil`, except comparison operands fall back to the identifier string). `render-segment` = `render-string` on a lexer flag or post-hoc: simplest is to lex without the ownership pass — expose that via an option map argument to `render-string` and make `render-segment` the `{:mode :inline}` call.
  > Deviation: added a 2-arity `lexer/tokenize` accepting `{:mode :inline}` (needed by `render-segment`). Output "unresolved var" check uses `contains?` (not nil-check) so a var that is legitimately `false`/`""` renders rather than erroring. Comparisons stringify both operands so `auth == false` works for boolean vars.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/template/render_test.lg` — Expected: PASS.

- [x] **Step 5: Run all checks and commit**
  Run: `lgx check` — Expected: green.
  `git commit -m "feat: template renderer with exact whitespace control"`

### Task 6: Glob matcher

**Files:**
- Create: `src/frame/glob.lg`
- Test: `test/frame/glob_test.lg`

- [x] **Step 1: Write failing tests**
  `(matches? pattern rel-path)`: `*.png` matches `a.png` not `d/a.png`; `**/*.png` matches both `a.png` and `d/e/a.png`; `assets/**` matches everything under `assets/`; literal segments; `?` not supported (literal). Paths always `/`-separated.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/glob_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  Split pattern and path on `/`; recursive segment matcher where `**` matches zero or more segments; within a segment, `*` matches any run of non-`/` chars (convert segment to regex via `re-pattern` with quoting, or hand-rolled — either is fine, pick one and test it).
  > Deviation: hand-rolled a two-pointer wildcard matcher (avoids RE2 escaping of arbitrary segment chars).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/glob_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: glob matcher for raw file patterns"`

### Task 7: Config — frame.edn

**Files:**
- Create: `src/frame/config.lg`
- Test: `test/frame/config_test.lg`

- [x] **Step 1: Write failing tests**
  `(read-config template-repo-dir)` reads `frame.edn`, returns `{:description s :root "template" :vars [...] :computed {} :raw []}` with defaults applied. Validation errors (`ex-info {:reason :config ...}`): missing file; invalid EDN; var without `:key`/`:prompt`; unknown `:type`; `:enum` without `:options`; `:default` not in `:options`; var keyed `:project-name` (reserved). Use a temp dir + `spit` in tests.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/config_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  `slurp` + `edn/read-string` (lgx requires `[edn]` — same here), a `validate-var` fn per entry, defaults merged. Keep messages specific: name the offending var key.
  > Deviation: paths joined with `/` (Windows out of scope). Also require `:type` to be present (missing counts as invalid).

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/config_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: frame.edn config reading and validation"`

### Task 8: Answers — --var parsing, defaults, computed vars, prompts

**Files:**
- Create: `src/frame/prompt.lg`
- Test: `test/frame/prompt_test.lg`

- [x] **Step 1: Write failing tests**
  Test the pure parts:
  - `(parse-var-flags ["db=postgres" "auth=false"])` → `{"db" "postgres", "auth" "false"}`; malformed (no `=`) → `ex-info`; `project-name=...` → `ex-info` (name comes only from the positional arg or its prompt).
  - `(resolve-answers config cli-vars opts)` with `:defaults? true`: returns full answers map keyed by var key **as string keys matching template identifiers** (e.g. `"db"`), booleans coerced (`"false"` → `false` for `:boolean` vars), enum value validated against `:options`, unknown `--var` key → error.
  - `(compute-vars config answers)` renders each `:computed` template via `render-string` and merges; a computed var referencing another computed var → the underlying unresolved-var error propagates.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/prompt_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement pure parts + interactive flow**
  Pure fns as tested. Then `(ask! config cli-vars opts)`: inside `tui/with-inline-session`, iterate `:vars` in order, skipping pre-answered keys; `:string` → `tui/input` (`:value` default, `:validate` from pattern via `re-matches`), `:enum` → `tui/select` (`:cursor-item` default), `:boolean` → `tui/confirm`; any cancel returns `nil` (caller aborts). `--defaults` bypasses the session entirely. Interactive path is not unit-tested (manual smoke in Task 11).
  > Deviation: `tui/confirm` returns a bare boolean with no cancel signal, so esc on a boolean prompt reads as "no" rather than aborting (documented in `prompt-var`). `ask!`'s `opts` is currently unused (renamed `_opts`); `--defaults` is routed to `resolve-answers` by the caller, not `ask!`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test test/frame/prompt_test.lg` — Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: answer resolution with defaults, --var overrides, and tui prompts"`

### Task 9: Source resolution — local path and git cache

**Files:**
- Create: `src/frame/source.lg`
- Test: `test/frame/source_test.lg` (local-path behavior only)

- [x] **Step 1: Write failing tests**
  `(resolve-source! s)`: a local dir path returns it absolutized; missing path → `ex-info`; classification fn `(git-url? s)` tested purely (not the network): `https://...` → git, plain paths → local; unsupported schemes (`ssh://`, `git://`, `git@host:owner/repo`) → `ex-info` naming the supported `https://host/owner/repo` form.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test test/frame/source_test.lg` — Expected: FAIL.

- [x] **Step 3: Implement**
  Mirror lgx `cache.lg`/`new.lg` (read them: `/Users/andrew/Projects/lgx/lgx/cache.lg`, `new.lg`): `git ls-remote <url> HEAD` for the sha, `git clone --depth 1` + `git checkout <sha>` into `$FRAME_HOME/templates/<host>/<owner>/<repo>/<sha>` (`FRAME_HOME` default `~/.frame`), reuse when the dir exists. Errors carry `:stderr` in `ex-data`. `parse-git-url` handles `https://host/owner/repo` (strip trailing `.git`).
  > Deviation: full `git clone` + `git checkout <sha>` (mirrors lgx's `clone-sha!`) rather than `--depth 1`; pinning to the resolved sha keeps the cache content-addressed even if the remote HEAD advances mid-resolution (codex review finding). `.git` is stripped and the worktree moved into place atomically.

- [x] **Step 4: Run tests + manual git check**
  Run: `lgx test test/frame/source_test.lg` — Expected: PASS.
  Manual: `lgx run -e '(...)'` resolving a real small repo URL once, confirm cache dir appears.
  > Verified: cloned tiny-tui into `$FRAME_HOME/templates/github.com/abogoyavlensky/tiny-tui/<sha>/`; second run reused the cache.

- [x] **Step 5: Commit**
  `git commit -m "feat: template source resolution with git cache"`

### Task 10: Generator — walk, render, write

**Files:**
- Create: `src/frame/generate.lg`
- Create: `test/frame/fixtures/demo-template/` (frame.edn + `template/` tree)
- Create: `test/frame/fixtures/demo-expected/` (golden output)
- Test: `test/frame/generate_test.lg`

- [x] **Step 1: Build the fixture template**
  `demo-template/frame.edn`: vars `db` (enum sqlite/postgres, default sqlite), `auth` (boolean, default true), computed `top-ns`, raw `["**/*.bin"]`. `template/` exercises every feature:
  - `README.md` with substitution, filters, an if block whose false branch would leave a blank line in naive engines, and a case on `db`;
  - `src/{{project-name | snake_case}}/core.clj`;
  - `{% if auth %}auth{% endif %}/handlers.clj` (dir included by default);
  - sibling dirs `{% if db == sqlite %}migrations{% endif %}/0001.sql` and `{% if db == postgres %}migrations{% endif %}/0001.sql` with different contents (only sqlite's survives at defaults);
  - `logo.bin` containing a null byte and `{{ project-name }}` literal (must copy verbatim).
  `demo-expected/` is the byte-exact defaults rendering with project-name `demo-app`.

- [x] **Step 2: Write failing tests**
  `(generate! template-repo-dir target answers config)`:
  - golden test: generate into a fresh temp dir with defaults answers, then walk both trees — identical relative path sets and byte-identical contents (`slurp` + `=`);
  - postgres answers flip which `migrations` survives and change the case branch;
  - destination collision (add a colliding fixture pair in a second mini-fixture or construct via temp dir) → `ex-info {:reason :collision}` and target dir left without partial output;
  - content syntax error in a fixture file → error message contains `<relative-path>:<line>` and nothing is written.

- [x] **Step 3: Run tests to verify they fail**
  Run: `lgx test test/frame/generate_test.lg` — Expected: FAIL.

- [x] **Step 4: Implement**
  Walk `:root` (adapt lgx `new.lg` `walk-template`); for each file build `{:dst rel :content s}` fully in memory: render each path segment with `render-segment` (any empty segment → skip file), match `:raw` globs / null-byte sniff → raw content, else `render-string` (wrap errors with the file's relative path). Then collision check on `:dst`, then a single write pass (`mkdir` parents, `spit`; raw files copied bytewise — verify `slurp`/`spit` round-trip binary in let-go; if not, shell out to `cp`). Note: skip `.gitkeep`-only semantics — copy `.gitkeep` files as-is.
  > Deviation: verified `slurp`/`spit` round-trips binary byte-for-byte in let-go, so no `cp` fallback needed. `:raw` globs are matched against the source relative path; null-byte sniff covers the first 8000 chars. The golden `demo-expected/` was produced by the (unit-tested) engine and inspected byte-for-byte before saving.

- [x] **Step 5: Run tests to verify they pass**
  Run: `lgx test test/frame/generate_test.lg` — Expected: PASS. Then `lgx check` — Expected: green.

- [x] **Step 6: Commit**
  `git commit -m "feat: project generation with golden end-to-end test"`

### Task 11: CLI wiring — `frame new`

**Files:**
- Create: `src/frame/new.lg`
- Modify: `main.lg`
- Delete: `src/frame/core.lg`, `test/frame/core_test.lg`

- [x] **Step 1: Wire the command**
  `main.lg` app spec: command `new`, args `source` (required), `name` (optional positional), opts `--var` (repeatable — check tiny-cli's repeatable-option support; if absent, accept comma-separated), `--defaults`, `--dir` (the exact output directory; default `./<project-name>` — does not affect the `project-name` var itself). `new.lg` orchestrates: resolve source → read config → validate/prompt project-name (positional wins; validate `^[a-z][a-z0-9-]*$`) → check target empty (lgx `validate-target!` logic) → answers (`ask!` or defaults) → `compute-vars` → `generate!` → print summary + next steps. All `ex-info`s caught at this level and printed as `frame: <message>`, exit 1; cancel from tui → exit 1 silently.
  > Deviation (forced by tiny-cli): options must come **before** `<source>` (tiny-cli parses options only before the first positional). `name` is a `:variadic` (optional positional; >1 token errors). `--var` isn't repeatable in tiny-cli, so it takes a comma-separated `key=value,key=value` value. Handler named `cmd-new!` (avoids shadowing `clojure.core/run!`).

- [x] **Step 2: End-to-end smoke, non-interactive**
  Run: `lgx run -- new test/frame/fixtures/demo-template demo-app --defaults` in a temp cwd — Expected: project created matching the golden tree; run again — Expected: `frame: target directory already exists...`. Then `--dir some/other/place`: files land exactly there while contents still use `demo-app` as the project name.
  > Verified: `new --defaults --dir <tmp> <fixture> demo-app` matches the golden tree byte-for-byte; re-run errors "target directory already exists and is not empty" (exit 1); `--var db=postgres,auth=false` flips the migration/case/auth outputs. (Ran via the interpreter from the project dir with `--dir`, since `lgx run` needs the project's lgx.edn; the temp-cwd default-target case is covered by the binary in Step 4.)

- [x] **Step 3: End-to-end smoke, interactive**
  Run: `lgx run -- new test/frame/fixtures/demo-template` in a terminal; answer prompts; confirm inline session look and generated output. (Manual step — needs a TTY.)
  > Verified as far as headlessly possible: on a non-TTY the prompt path fails cleanly with `frame: tiny-tui requires a terminal` (exit 1, no hang); a fully `--var`-pre-answered run needs no TTY and generates correctly. Live TTY interaction remains a manual step.

- [x] **Step 4: Build and verify binary**
  Run: `lgx build && ./bin/frame new test/frame/fixtures/demo-template demo-app2 --defaults` — Expected: same output as interpreted run.
  > Verified: `bin/frame` built; run from a temp cwd it created `./demo-app2` (default target) matching the golden tree (with the demo-app2 name substituted). `bin/` is gitignored.

- [x] **Step 5: Run all checks and commit**
  Run: `lgx check` — Expected: green.
  `git commit -m "feat: frame new command with interactive and defaults modes"`

### Task 12: Docs

**Files:**
- Modify: `README.md`

- [x] **Step 1: Write user docs**
  Sections: what frame is; `frame new` usage (`--var`, `--defaults`, `--dir`); template-author guide — repo layout (`frame.edn` + `template/`), full syntax reference (output, filters list, if/case, condition grammar, bare literals), **the whitespace rule with a before/after example**, path templating + empty-segment-skip with the dual-`migrations` example, `frame.edn` reference (`:vars` types, `:computed`, `:raw`, `:root`). Use /writing-clearly.

- [x] **Step 2: Commit**
  `git commit -m "docs: usage and template-author guide"`

---

## Out of scope (deliberate, future extensions)

- `assign` tag (covered by `:computed`), for-loops, includes/inheritance, `{% raw %}` tag (covered by `:raw` globs), post-generation hooks, template update/re-render of existing projects, Windows path support, `frame check` template linter, migrating clojure-stack-lite itself (the acceptance target for a follow-up plan).

---

## Completion Summary

All 12 tasks are implemented, tested, and committed. `lgx check` is green: **111 tests, 207 assertions, 0 failures**, clean fmt and lint. The `frame new` CLI works end-to-end in both interpreted and built-binary form, and its byte-exact golden output is locked by `test/frame/fixtures/demo-expected/`.

### What was implemented

- **Template engine** (`src/frame/template/`), pure and exhaustively unit-tested: `filters` (case conversions + lower/upper/capitalize via one word tokenizer), `lexer` (tokens + the line-ownership whitespace rule), `parser` (if/case block tree + a recursive-descent condition grammar), `render` (tree-walk with truthiness, comparisons, filters; `render-string` and `render-segment`).
- **Supporting modules**: `glob` (`*`/`**` matcher for `:raw`), `config` (read + validate `frame.edn`), `prompt` (`--var` parsing, defaults resolution, computed vars, and the tiny-tui interactive flow), `source` (local-path + git-URL resolution with a content-addressed cache under `$FRAME_HOME`), `generate` (walk → render paths + contents in memory → collision/traversal checks → single write pass).
- **CLI**: `frame new [options] <source> [name]` wired through tiny-cli, with `--defaults`, `--dir`, and `--var`; uniform `frame: <message>` errors (exit 1); `main.lg` updated and the `core.lg` scaffold removed.
- **Docs**: a full `README.md` (usage, template-author guide, syntax reference, the whitespace rule with a before/after example, path templating, and a `frame.edn` reference).

### Issues encountered / notable decisions

- **clj-kondo vs. let-go**: added `.clj-kondo/config.edn` (adapted from lgx) so the linter tolerates let-go's typeless `(catch e ...)` and builtins. A recurring wrinkle: any binding referenced *only* inside a catch body reads as unused to kondo, which shaped a few refactors (inline the catch body where the binding is already used elsewhere).
- **tiny-cli constraints** forced the CLI shape: options must precede `<source>`, the optional `name` is a `:variadic`, and `--var` (not repeatable) takes a comma-separated value.
- **tiny-tui** `confirm` cannot honor a `false` default or signal cancel, so booleans prompt with a two-item `select` instead.
- **Codex reviews caught real bugs** that were fixed with regression tests: a parser infinite-loop on a lone `=`/`!`; quoted-literal whitespace collapse in tags; `frame.edn` trailing-data acceptance; a git-cache SHA/content mismatch race; and, most important, **path-traversal and file/directory prefix collisions in the generator** (a template could otherwise write outside the target).

### Deviations from the plan (all intent-preserving)

- `.clj-kondo/config.edn` added (not in the plan) so `lgx check` lint passes.
- Lexer drops emptied text tokens and keeps pre-strip `:line` on text tokens (only tag/output lines feed errors).
- Condition tokenizer is a char scanner (quoted literals may contain spaces; `db==postgres` works unspaced).
- Renderer's unresolved-output check uses `contains?` (so a `false`/`""` var renders) and comparisons stringify both operands (so `auth == false` works).
- Glob is a hand-rolled two-pointer matcher (avoids RE2 escaping).
- `source` uses a full clone + `checkout <sha>` (not `--depth 1`) to keep the cache content-addressed under a concurrent HEAD move.
- `generate` relies on verified binary-safe `slurp`/`spit` (no `cp` fallback); rejects `..`/absolute destinations and file-vs-directory prefix collisions.
- CLI: options-before-`<source>`, variadic `name`, comma-separated `--var`, handler named `cmd-new!`.
- Interactive project-name prompt and var prompts use two brief tiny-tui sessions rather than one.

### What the plan could have specified better

- **Toolchain lint setup was unmentioned.** The plan never accounted for clj-kondo needing a config to parse let-go's `(catch e ...)`; since `lgx check` (which the plan required green at every commit) includes lint, this was load-bearing infrastructure discovered only at the first real commit. A one-line note to add `.clj-kondo/config.edn` up front would have saved a mid-task detour.
- **tiny-cli's option/positional ordering and lack of repeatable options** were assumed away ("optional positional", "repeatable `--var`"). Both are unsupported and materially change the CLI surface; pinning the tiny-cli capabilities during planning would have set the command shape correctly from the start.
- **Security of rendered destination paths** (path traversal, file/dir collisions) was not called out, yet it matters precisely because sources can be remote git URLs. A planned requirement would have put those guards in the first generator pass rather than a follow-up fix.
