# Conditional Vars, `frame check`, and Next Steps Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three template-author features to frame: a `:when` condition on vars (questions that only apply to some answers), a `frame check` command that renders every valid combination of answers in memory and reports errors (with `--json` for CI matrices), and template-defined `:next-steps` printed after generation.

**Tech Stack:** let-go (`.lg`), lgx build tool, tiny-cli (commands/options), tiny-tui (prompts), let-go built-in `json` namespace.

---

## Design

### Context

frame templates are declarative: `frame.edn` declares ordered `:vars`, and `template/` files use `{% if %}`/`{% case %}` tags in contents and in path segments. A condition language already exists — `parser/parse-cond` (`src/frame/template/parser.lg:150`) parses `and`/`or`/`not`/`==`/`!=` into an AST, and `render/eval-cond` (`src/frame/template/render.lg:43`, private) evaluates it.

Driving use case: clojure-stack-lite is adding a `frontend` enum (`htmx | re-frame | react | replicant | none`). `auth` only exists for `htmx`; `daisyui` only applies when there is a frontend. Today every var is always asked, so `--var frontend=react,auth=true` silently generates a broken project, and nothing verifies that the ~50 valid combinations all render.

### Feature 1 — `:when` on vars

A var may declare `:when "<condition>"`, using the same condition syntax as template tags:

```clojure
{:key :frontend :prompt "Frontend" :type :enum
 :options ["htmx" "re-frame" "react" "replicant" "none"] :default "htmx"}
{:key :auth :prompt "Include auth?" :type :boolean :default false
 :when "frontend == htmx"}
{:key :daisyui :prompt "Include DaisyUI?" :type :boolean :default false
 :when "frontend != none"}
```

**Semantics.**
- Vars are resolved in declared order. When a var's `:when` is false against the answers so far, the var is **skipped**: not prompted, not defaulted, and given its type's **skipped value** — `false` for `:boolean`, `""` for `:enum` and `:string`. Both are falsy, so `{% if auth %}` and `{% if not auth %}` work, and `{% case cljs-flavor %}` falls through to `else`. A skipped var is still present in the vars map, so `{{ auth }}` never raises "unresolved variable".
- A skipped string var is **not** run through `:validate` (its `""` would fail `^.+$`).
- **Skips are transitive:** a var is skipped whenever any var in its `:when-refs` was skipped, without evaluating its condition. Evaluating against a skipped value would be wrong for `!=`: `eval-cmp` (`render.lg:37`) stringifies operands, so with a skipped enum `flavor` = `""`, `flavor != reagent` is true and a question that doesn't apply would be asked. The rule is "a question about a skipped answer doesn't apply".

**Validation at config load** (`{:reason :config}`, before any prompting). For each var with `:when`:
1. `:when` must be a string that `parse-cond` accepts. Error: `var :auth has invalid :when "…": <parser message>`.
2. **References must be earlier vars.** Collect every identifier term (`{:op :term :kind :ident}`) in the AST. Known names = names of vars declared *before* this one. Built-ins (`project-name`, `now-*`) are **not** allowed — `project-name` is merged in after answers, and no use case needs it.
   - A bare identifier term (truthiness test, possibly under `not`/`and`/`or`) must be a known name. Error: `var :daisyui :when references unknown or later var "frontnd"`.
   - In a comparison, at least one side must be a known name (otherwise both sides are literals — almost always a typo). Error: `var :auth :when compares two literals: "fronted == htmx"`.
   - If one side is a known **enum** var and the other side is a literal (a quoted string, or an identifier that is not a known name), the literal must be one of that var's `:options`. Error: `var :auth :when: "htmxx" is not an option of :frontend`. For a known **boolean** var the literal must be `true` or `false`.
3. The normalized var gains `:when-cond` (the AST) and `:when-refs` (the set of earlier var names it references, as strings), and keeps `:when` (the source string, for messages).

The literal check is what turns `frontend == htmxx` from "silently never asked" into an immediate config error.

**`--var` for a skipped var is an error** (`{:reason :var}`): `var auth does not apply: :when "frontend == htmx" is false`. This is the early warning for invalid combinations. To make it early in interactive mode too, `ask!` first runs a **pre-check over the `--var` answers alone**: walk vars in order building a partial map from coerced `--var` values and skipped values; whenever every var in its `:when-refs` is already in the partial map, evaluate it, and if it is false for a var given via `--var`, throw. So `frame new tpl app --var frontend=react,auth=true` fails before the first prompt. Vars whose references aren't known yet are checked again during the prompt loop.

**Summary output.** `var-summary-lines` prints a skipped var as `  auth: false (skipped)`, so users see why they weren't asked.

**Rejected:** an `:otherwise` key to choose a skipped var's value (YAGNI — every current case wants "off"; easy to add later without breaking anything); HoneySQL-style vector conditions (a second condition language next to the one already used in paths and tags).

### Feature 2 — `frame check <source> [--json]`

Renders the template in memory for **every valid combination** of answers and reports problems. Never writes files.

**Enumeration** (`check/combinations`): depth-first over vars in declared order, starting from an empty answers map.
- Var skipped by `:when` → one branch with the skipped value.
- `:enum` → one branch per option, in declared order.
- `:boolean` → two branches, `true` then `false`.
- `:string` → one branch: its `:default` when non-blank, else the literal `"example"`. Not validated.

Skipped vars yield one branch, so the result has no duplicates and its length is the number of valid combinations. `project-name` is the fixed value `"example-app"`.

**Per combination:** `prompt/compute-vars` (adds built-ins + `:computed`), then `generate/render-entries` (renders every path and content and checks for collisions and escaping paths, without writing), then renders `:next-steps`. Any thrown ex-info is caught and recorded with its message.

**Findings:**
- **Errors**, grouped by message. Each group records how many combinations failed and one example combination. The same broken tag fails in every combination that reaches it; grouping keeps the output readable.
- **Never-generated files:** template source files (relative to the root) that no combination generates, usually a typo in a path condition like `{%if frontend==htmxx%}`. Reported as failures. Computed only when there are no errors, because a failed combination's surviving files are unknown.

**Human output** (stdout):
```
Checked 52 combinations of ../clojure-stack-lite: OK
```
or, on failure, one block per error group and one line per never-generated file, then exit 1:
```
src/{{project-name|snake_case}}/views.clj:3: unresolved variable: fronted
  in 16 of 52 combinations, e.g. db=sqlite frontend=htmx auth=true daisyui=true deploy=kamal developer=example
never generated in any combination: resources/{%if frontend==htmxx%}x.css{%endif%}
```

**`--json` output** (stdout, a single object, same exit codes):
```json
{"ok": true,
 "count": 52,
 "combinations": [{"answers": {"db": "sqlite", "frontend": "htmx", "auth": true},
                   "vars": "db=sqlite,frontend=htmx,auth=true,developer=example"}],
 "errors": [{"message": "…", "count": 16, "example": "db=sqlite,frontend=htmx,…"}],
 "never_generated": []}
```
`answers` includes every declared var (skipped ones included, with their skipped value). `vars` lists only **applied** vars in declared order as `key=value` joined by `,`. It is meant to be passed straight to `frame new --defaults --var <vars>`, which would reject a skipped var. A CI workflow can feed `combinations` to a GitHub Actions matrix.

**let-go gotcha:** `json/write-json` in let-go 1.11.1 double-quotes **string** map keys (`{"db" 1}` → `{"\"db\"":1}`); keyword keys serialize correctly. Build the JSON value with keyword keys (`(keyword "db")`).

Config and source errors keep the usual `frame: <message>` on stderr, exit 1, with no JSON. `<source>` resolves the same way as `frame new` (local dir or git URL).

### Feature 3 — `:next-steps`

`frame.edn` gains an optional `:next-steps` vector of strings. Each entry is rendered **in inline mode** (`render/render-segment`: tags are zero-width) against the final vars (answers + `project-name` + built-ins + computed). Entries that render blank are dropped, the same rule as an empty path segment:

```clojure
:next-steps ["mise trust && mise install"
             "{% if db == postgres %}docker compose up -d{% endif %}"
             "bb clj-repl"]
```

frame keeps printing its own `cd <target>` first, then each rendered step on its own line, indented two spaces, under the existing `Next steps:` header. Steps are rendered **before** `generate!` writes anything, so a broken step aborts with nothing written. Validation: `:next-steps` must be a vector of strings (config error otherwise); default `[]`; add `:next-steps` to the known top-level keys.

### Compatibility note for template authors

An older frame silently ignores `:when` (it's a key inside a var) and would ask every question. Templates using `:when` or `:next-steps` should set `:min-frame-version` to the first release that ships them. The README says so; the version bump and release are separate.

### Key decisions

1. Skipped value is falsy by type (`false` / `""`), with no `:otherwise` key.
1a. Skips are transitive: a var whose `:when` references a skipped var is itself skipped.
2. `:when` may reference only **earlier** declared vars, with typo checks on literals.
3. `--var` for a skipped var is an error, pre-checked before any prompt when possible.
4. `frame check` treats never-generated files as failures, not warnings.
5. Check output groups errors by message; JSON gives a ready `--var` string per combination.
6. `:next-steps` is a vector of one-line templates; blank renders are dropped.

---

## File Structure

- Modify `src/frame/template/render.lg` — public `eval-condition [ast vars]` wrapping the private `eval-cond`.
- Modify `src/frame/config.lg` — `:when` validation (`validate-when`, identifier/literal checks), public `normalize-vars [vars]` (validates vars in order, attaching `:when-cond`), `:next-steps` validation, `:next-steps` in `known-keys`.
- Modify `src/frame/prompt.lg` — `applies?`, `skipped-value`, `:when` handling in `resolve-answers` and `ask!`, public `precheck-cli` (partial-map pre-check).
- Modify `src/frame/generate.lg` — extract public `render-entries` (render + collision check, no write) and `source-files` (relative source paths); `generate!` = `render-entries` + write.
- Modify `src/frame/new.lg` — skipped vars in `var-summary-lines`, public `next-steps-lines`, render steps before writing.
- Create `src/frame/check.lg` — `combinations`, `check-template`, `format-human`, `format-json`, `cmd-check!`.
- Modify `main.lg` — register the `check` command with `--json`.
- Modify `README.md` — `:when`, `:next-steps`, `frame check`, completions text, compatibility note.
- Tests: modify `test/frame/template/render_test.lg`, `test/frame/config_test.lg`, `test/frame/prompt_test.lg`, `test/frame/generate_test.lg`, `test/frame/new_test.lg`; create `test/frame/check_test.lg`.

Shared shapes the tasks must agree on:

```clojure
;; normalized var (config/normalize-vars output)
{:key :auth :prompt "…" :type :boolean :default false
 :when "frontend == htmx" :when-cond <parse-cond AST> :when-refs #{"frontend"}}
;; :when/:when-cond/:when-refs present only when declared

(render/eval-condition ast vars)            ;=> boolean; vars string-keyed
(prompt/applies? var answers skipped)       ;=> false when any :when-refs is in `skipped` (a set of var-name strings);
                                            ;   else true when no :when-cond, else eval
(prompt/skipped-keys vars answers)          ;=> set of skipped var names, recomputed in declared order from final answers
(prompt/skipped-value var)                  ;=> false | ""
(generate/render-entries repo-dir answers config) ;=> [{:dst :content :src}], throws like generate!
(generate/source-files repo-dir config)     ;=> sorted vector of root-relative source paths
(new/next-steps-lines config vars target)   ;=> ["cd <target>" "<step>" ...]
(check/combinations config)                 ;=> vector of string-keyed answers maps
(check/check-template repo-dir config)
;=> {:combinations [answers ...]
;    :errors [{:message s :count n :example answers} ...]
;    :never-generated [src ...]}
```

---

## Tasks

Run tests with `lgx test` from the frame repo root (baseline: 159 tests, 0 failures). Run `lgx check` (fmt + lint + test) before each commit.

### Task 1: Public condition evaluation

**Files:**
- Modify: `src/frame/template/render.lg`
- Test: `test/frame/template/render_test.lg`

- [x] **Step 1: Write failing tests** for `render/eval-condition`: takes a `parser/parse-cond` AST and a string-keyed vars map. Cases: `frontend == htmx` true/false; `not auth` with `auth` = `false` and with `auth` = `""`; `frontend != none and daisyui`.
- [x] **Step 2: Run** `lgx test` — expect the new tests to fail (unresolved var).
- [x] **Step 3: Implement** `(defn eval-condition [ast vars] (boolean (eval-cond vars ast)))` with a docstring. Leave `eval-cond` private.
- [x] **Step 4: Run** `lgx test` — PASS.
- [x] **Step 5: Commit** `git commit -m "feat: expose condition evaluation for frame.edn conditions"`

### Task 2: Config — `:when` and `:next-steps` validation

**Files:**
- Modify: `src/frame/config.lg`
- Test: `test/frame/config_test.lg`

- [x] **Step 1: Write failing tests** (via the existing `read-cfg` helper):
  - valid `:when` → normalized var has `:when-cond` (a map), `:when-refs` (`#{"frontend"}`), and the original `:when`; a var without `:when` has neither.
  - `:when` not a string → `:config`; unparsable (`"frontend =="`) → `:config` with `invalid :when` in the message.
  - references a later var, itself, an undeclared name in a bare term, or `project-name` → `:config`.
  - `"fronted == htmx"` (two literals) → `:config`, message contains `compares two literals`.
  - `"frontend == htmxx"` (not an option) → `:config`, message contains `not an option`; `"htmx == frontend"` (reversed) also checked; quoted `"frontend == \"htmx\""` accepted.
  - boolean compared with `yes` → `:config`; with `true` accepted.
  - `:next-steps` defaults to `[]`; `["a" "b"]` accepted; `"a"` or `[1]` → `:config`; `:next-steps` produces no unrecognized-key warning.
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement.**
  - `normalize-vars [vars]`: reduce in order, carrying the vars validated so far; for each var run `validate-var` then `validate-when` against the earlier vars; return the vector. `read-config` calls it instead of `(map validate-var …)`.
  - `validate-when [v earlier]` (private): no-op without `:when`; parse with `parser/parse-cond` (catch its ex-info and rethrow as a config error naming the var); walk the AST (`:or`/`:and` → left+right, `:not` → `:expr`, `:cmp`, `:term`) applying the rules in the Design; return `(assoc v :when-cond ast :when-refs refs)`.
  - `:next-steps`: validate as a vector of strings; normalized default `[]`; add to `known-keys`.
  - Require `frame.template.parser` in `config.lg`.
- [x] **Step 4: Run** `lgx test` — PASS.
- [x] **Step 5: Commit** `git commit -m "feat: validate :when conditions and :next-steps in frame.edn"`

### Task 3: Prompt — skip, reject, and pre-check

**Files:**
- Modify: `src/frame/prompt.lg`
- Test: `test/frame/prompt_test.lg`

- [x] **Step 1: Write failing tests.** Build configs with `config/normalize-vars` so `:when-cond` is present. Use the frontend/auth/daisyui shape from the Design plus a chained var (`:theme`, enum, `:when "daisyui"`).
  - `skipped-value`: boolean → `false`, enum/string → `""`.
  - **Transitive skip with `!=`:** add enum `flavor` (`["reagent" "uix"]`, `:when "frontend == re-frame"`) and boolean `ssr` (`:when "flavor != uix"`). With `{"frontend" "react"}` and `--defaults`, `ssr` is `false` (skipped because `flavor` was skipped), not its default; `{"frontend" "react" "ssr" "true"}` → `:var` error.
  - `skipped-keys` on final answers for `frontend=react` returns `#{"auth" "flavor" "ssr"}`.
  - `resolve-answers` with `--defaults` and `{"frontend" "react"}` → `auth` is `false`, `daisyui` takes its default, chained var follows `daisyui`.
  - `{"frontend" "none"}` → `daisyui` `false`, chained `theme` skipped → `""`.
  - `{"frontend" "react" "auth" "true"}` → `:var` error whose message contains `does not apply`.
  - a skipped string var with `:validate {:pattern "^.+$"}` and no default does not throw.
  - `precheck-cli`: `{"frontend" "react" "auth" "true"}` throws `:var`; `{"auth" "true"}` alone (frontend unknown) does not throw and returns a partial map without `frontend`; `{"frontend" "react"}` returns `{"frontend" "react" "auth" false}` (determinable skip included).
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement.**
  - `applies? [var answers skipped]`: false when `(some skipped (:when-refs var))`; else true when no `:when-cond`; else `render/eval-condition`.
  - `skipped-value [var]`.
  - `skipped-keys [vars answers]`: walk vars in order, accumulating the set of names where `applies?` is false (used by the summary).
  - Every loop below carries the `skipped` set alongside the answers map and adds a var's name to it when skipping.
  - `resolve-answers`: per var, if not `applies?` → error when the key is in `cli-vars`, else assoc the skipped value; otherwise the current logic.
  - `precheck-cli [config cli-vars]` (public): reduce over vars with a partial map and a skipped set; a var's `:when` is evaluable when every name in its `:when-refs` is a key in the partial map (skipped vars are in the map with their skipped value). Evaluable and false → throw if in `cli-vars`, else assoc skipped value. Otherwise assoc the coerced `--var` value when present.
  - `ask!`: call `precheck-cli` first. A var needs a prompt when its key is not in the pre-checked map. If none do, return the pre-checked map (it's complete). Otherwise run the loop, where each var is: not `applies?` → error if in `cli-vars`, else skipped value; in `cli-vars` → coerced value; else prompt. Delete `resolve-from-cli` (replaced by the pre-check).
- [x] **Step 4: Run** `lgx test` — PASS.
- [x] **Step 5: Manual smoke** of the interactive path (not unit-tested) with a scratch template under the scratchpad that has the frontend/auth vars:
  `lgx run -- new /path/to/scratch-tpl app --dir /tmp/x --var frontend=react,auth=true` → `frame: var auth does not apply…`, exit 1, before any prompt.
  `lgx run -- new /path/to/scratch-tpl app --dir /tmp/x` → choosing `react` skips the auth question.
- [x] **Step 6: Commit** `git commit -m "feat: skip vars whose :when is false and reject --var for them"`

### Task 4: Generate — pure `render-entries` and `source-files`

**Files:**
- Modify: `src/frame/generate.lg`
- Test: `test/frame/generate_test.lg`

- [x] **Step 1: Write failing tests:** `render-entries` on the demo fixture returns entries whose `:dst` set matches the `demo-expected` tree and writes nothing; a template file named `{{ missing }}/x.txt` throws with a message that starts with `{{ missing }}/x.txt: ` and contains `unresolved variable`; it throws `:collision` for the collision template used in `destination-collision-aborts-before-writing`. `source-files` on the demo fixture returns the sorted root-relative paths (e.g. includes `"{% if auth %}auth{% endif %}/handlers.clj"`).
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement:** `render-entries` = `build-entries` + `check-collisions!`, returning the entries; `generate!` calls it and then writes. `source-files` maps `relative-to root` over `walk-files`. Also wrap the `render-path` call in `build-entries` with the same catch/rethrow that content errors get (`generate.lg:91-100`), prefixing `<rel-src>: `, so path-segment errors (e.g. `{{ projectname }}` in a directory name) name their source file — `frame check` groups errors by message and needs the location in it.
- [x] **Step 4: Run** `lgx test` — PASS (all existing generate tests unchanged).
- [x] **Step 5: Commit** `git commit -m "refactor: split in-memory rendering from writing in generate"`

### Task 5: `frame new` — skipped summary and next steps

**Files:**
- Modify: `src/frame/new.lg`
- Test: `test/frame/new_test.lg`

- [x] **Step 1: Write failing tests:**
  - `var-summary-lines` with a normalized auth var gated on `frontend == htmx` and answers `{"frontend" "react" "auth" false}` → contains `"  auth: false (skipped)"`; with `htmx` → `"  auth: false"`.
  - `next-steps-lines` with `:next-steps ["mise install" "{% if db == postgres %}docker compose up -d{% endif %}" "cd {{ project-name }}-x"]` → `["cd /t" "mise install" "cd app-x"]` for sqlite and includes the docker line for postgres; an entry that renders to whitespace only is dropped; a step with an unresolved var throws.
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement:** `var-summary-lines` appends ` (skipped)` when the var's name is in `(prompt/skipped-keys vars answers)`. Add `next-steps-lines [config vars target]` (render each with `render/render-segment`, trim, drop blank, prefix `cd <target>`). In `run*`, compute the lines right after `compute-vars` and **before** `generate!`; pass them to `print-summary`, which prints each under `Next steps:` indented two spaces.
- [x] **Step 4: Run** `lgx test` — PASS. Also `lgx run -- new --defaults test/frame/fixtures/demo-template demo --dir /tmp/frame-demo-$$` and confirm the output is unchanged apart from nothing new (the fixture has no `:next-steps`). Remove the dir afterwards.
- [x] **Step 5: Commit** `git commit -m "feat: template-defined next steps and skipped vars in summary"`

### Task 6: Check — enumeration and template check

**Files:**
- Create: `src/frame/check.lg`
- Test: `test/frame/check_test.lg`

- [x] **Step 1: Write failing tests** (tmp templates written the same way as in `generate_test.lg`):
  - `combinations` for `db` (2 options) × `auth` boolean → 4 maps in order `sqlite/true, sqlite/false, postgres/true, postgres/false`.
  - With `frontend` (`htmx`, `react`, `none`), `auth` `:when "frontend == htmx"`, `daisyui` `:when "frontend != none"` → htmx: 2 (auth) × 2 (daisyui) = 4; react: 1 × 2 = 2; none: 1 × 1 = 1; total 7, with no duplicates. Adding the transitive `flavor`/`ssr` pair from Task 3 under a `re-frame` option does not add combinations for other frontends.
  - string var without default → `"example"`; with default `"ann"` → `"ann"`.
  - `check-template` on `test/frame/fixtures/demo-template` → no errors, no never-generated files, 4 combinations.
  - a template where one file under `{% if auth %}…` contains `{{ missing }}` → one error group whose `:count` equals the number of auth-true combinations and whose message contains the file path and `unresolved variable`.
  - a template with a file `{% if db == mysql %}x.sql{% endif %}` (unreachable) → `:never-generated` contains that source path.
  - a broken `:next-steps` entry is reported as an error.
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement** `combinations` (DFS per the Design, carrying the skipped set and using `prompt/applies?` and `prompt/skipped-value`) and `check-template [repo-dir config]`: for each combination, `compute-vars` with `project-name` `"example-app"`, then `generate/render-entries` and `new/next-steps-lines` with a dummy target (`check` requires `frame.new`; `new` never requires `check`, so there is no cycle). Catch ex-info per combination; group by `ex-message`, keeping count and the first combination. When there are no errors, `:never-generated` = `generate/source-files` minus the union of every entry's `:src`.
- [x] **Step 4: Run** `lgx test` — PASS.
- [x] **Step 5: Commit** `git commit -m "feat: enumerate and render every valid answer combination"`

### Task 7: Check — output, JSON, and CLI wiring

**Files:**
- Modify: `src/frame/check.lg`, `main.lg`
- Test: `test/frame/check_test.lg`

- [x] **Step 1: Write failing tests:**
  - `format-human` for an OK result → `"Checked 4 combinations of <src>: OK"`; for a failing result → contains the error message, `in N of M combinations, e.g. key=value …`, and `never generated in any combination: <path>`.
  - `format-json` → parse it back with `json/read-json` (`{:keywords? true}`) and assert `:ok`, `:count`, a combination's `:answers` (skipped var present with its skipped value) and `:vars` (skipped var absent, declared order, e.g. `"frontend=react,daisyui=true"`), and `:never_generated`. Assert the raw string contains no `\"\"` key quoting.
- [x] **Step 2: Run** `lgx test` — new tests fail.
- [x] **Step 3: Implement** `format-human`, `format-json` (build the value with **keyword** keys — see the let-go gotcha in the Design), and `cmd-check! [ctx]`: resolve source, read config, print warnings to stderr like `new`, run `check-template`, print human or JSON to stdout, `(os/exit 1)` when not OK. Wrap in the same `try`/`frame: <message>` handler as `cmd-new!`. Register in `main.lg`:
  ```clojure
  {:name "check"
   :doc "Render every valid combination of answers in memory and report errors."
   :args [{:key :source :doc "Template source: a local directory or https://host/owner/repo."}]
   :opts [{:key :json? :long "json" :doc "Print the result as JSON (for CI matrices)."}]
   :run check/cmd-check!}
  ```
- [x] **Step 4: Run** `lgx test` — PASS. Then:
  - `lgx run -- check test/frame/fixtures/demo-template` → `Checked 4 combinations of …: OK`, exit 0.
  - `lgx run -- check ../clojure-stack-lite` (branch `migrate-to-frame`) → OK with 16 combinations (db 2 × auth 2 × daisyui 2 × deploy 2). Note the wall time; if it is over ~15s, record it in the plan's notes (parse caching is a possible follow-up, out of scope here).
  - `lgx run -- check ../clojure-stack-lite --json | head -c 400` → valid JSON with `"ok":true`.
- [x] **Step 5: Commit** `git commit -m "feat: add frame check command with --json output"`

### Task 8: Documentation

**Files:**
- Modify: `README.md`

- [x] **Step 1: Update `README.md`:**
  - **Usage:** add `frame check <source> [--json]` with a one-paragraph description and an example GitHub Actions snippet that feeds `combinations` into a matrix and runs `frame new --defaults --var "${{ matrix.vars }}" <source> app`.
  - **frame.edn:** document `:when` (syntax, earlier-vars-only rule, skipped values, `--var` rejection, typo checks) and `:next-steps` (inline rendering, blank entries dropped, printed after `cd <target>`). Add both to the example config.
  - Note that templates using `:when`/`:next-steps` should set `:min-frame-version` to the first frame release that supports them, because older frame ignores `:when`.
  - **Shell completions:** mention the `check` subcommand and `--json`.
  - Keep the existing voice; use /writing-clearly.
- [x] **Step 2: Run** `lgx check` — PASS.
- [x] **Step 3: Commit** `git commit -m "docs: document :when, :next-steps, and frame check"`

### Task 9: Final verification

- [ ] **Step 1:** `lgx check` — fmt, lint, and all tests pass.
- [ ] **Step 2:** `lgx build` and run `bin/frame check ../clojure-stack-lite` and `bin/frame new --defaults --var developer=x ../clojure-stack-lite app --dir <scratch>/app` — both succeed; the summary and next steps print as before (clojure-stack-lite has no `:next-steps` yet).
- [ ] **Step 3:** Mark this plan's checkboxes complete and commit `git commit -m "docs: mark conditional vars/check/next-steps plan complete"`.
