# Migrate clojure-lib-template to Frame Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the deps-new template at `/Users/andrew/Projects/clojure-lib-template` to frame, in place, keeping the deps-new files as a fallback — verified by frame and deps-new generating byte-identical projects.

**Tech Stack:** frame (let-go/lgx), deps-new (Clojure), bash for verification.

---

## Design

### Context

`clojure-lib-template` is a deps-new template. Its substitutable files live under
`resources/io/github/abogoyavlensky/clojure_lib_template/` — `root/` (copied as-is),
`src/core.clj` and `test/core_test.clj` (moved into per-project namespace dirs via
`:transform` in `template.edn`). It uses these deps-new substitutions:

| deps-new tag | Meaning (for `:name io.github.<user>/<proj>`) | frame equivalent |
|---|---|---|
| `{{main}}`, `{{main/ns}}` | `<proj>` | `{{ project-name }}` |
| `{{main/file}}` | `<proj>` with `-` → `_` | `{{ project-name \| snake_case }}` |
| `{{name}}` | `io.github.<user>/<proj>` | `{{ name }}` (computed) |
| `{{scm/user}}` | `<user>` | `{{ username }}` (var) |
| `{{scm/domain}}` | `github.com` | hardcoded `github.com` |
| `{{developer}}` | `<user>` capitalized | `{{ developer }}` (computed via `capitalize`) |
| `{{now/year}}`, `{{now/date}}` | current year / `yyyy-MM-dd` date | `{{ now-year }}`, `{{ now-date }}` (**new frame built-ins**) |

Two repos are touched:

1. **frame** (`/Users/andrew/Projects/frame`, work on `master`): add built-in
   `now-date` / `now-year` variables. Frame currently injects only `project-name`;
   the template's LICENSE and CHANGELOG need dates, and prompting for them would be
   bad UX with stale defaults.
2. **clojure-lib-template** (`/Users/andrew/Projects/clojure-lib-template`, branch
   `migrate-to-frame`, already created and checked out): add `frame.edn` and
   `template/` at the repo root. Every existing deps-new file stays untouched.

### Part 1 — frame built-ins `now-date` and `now-year`

- Injected into the answers map next to `project-name` (in `cmd-new` in
  `src/frame/new.lg`, before `prompt/compute-vars`), so they are available in
  content templates, path templates, and `:computed` values.
- Values via let-go Go-interop on `(now)`:
  `(.Format (now) "2006-01-02")` → `"2026-07-10"`, `(.Format (now) "2006")` → `"2026"`.
  The `yyyy-MM-dd` format matches deps-new's `now/date`.
- Reserved like `project-name`:
  - `config.lg` `validate-var`: declaring `:now-date` / `:now-year` in `:vars` is a
    config error (generalize the existing `:project-name` check to a reserved set).
  - `prompt.lg` `parse-cli-vars`: `--var now-date=...` / `now-year=...` rejected
    (generalize the existing `project-name` check).
- Documented in frame's README next to `project-name`.

### Part 2 — the frame template

New files in `clojure-lib-template` (nothing removed or modified):

```
frame.edn
template/                                          # root/* copied, tags rewritten
├── .clj-kondo/  .github/  dev/  (etc — all of root/*)
├── src/{{ project-name | snake_case }}/core.clj
└── test/{{ project-name | snake_case }}/core_test.clj
```

`frame.edn` (exact content):

```edn
{:description "A template for creating a Clojure library"
 :vars [{:key :username
         :prompt "GitHub username"
         :type :string
         :validate {:pattern "^.+$" :msg "Username cannot be empty"}}]
 :computed {:name "io.github.{{ username }}/{{ project-name }}"
            :developer "{{ username | capitalize }}"}
 :raw [".github/**"]}
```

Key decisions:

- **`username` has no `:default`** — it is a public template; the user must supply
  it (interactively or with `--var username=...`). `--defaults` alone will fail its
  non-empty validation, which is correct.
- **`.github/**` is `:raw`** — workflow files use GitHub Actions `${{ secrets.* }}`
  syntax, which frame would try to render and fail on (deps-new leaves unknown tags
  alone; frame errors on unresolved output tags). Raw files still get their paths
  rendered, contents copied verbatim. No other template file contains `{{` or `{%`
  except the ones listed in the mapping table (verified by grep).
- `scm/domain` is hardcoded as `github.com` in content — the template targets GitHub.

Files whose contents change during the copy (complete list, from grep):

| File in `template/` | Rewrite |
|---|---|
| `LICENSE` | `{{now/year}} {{developer}}` → `{{ now-year }} {{ developer }}` |
| `CHANGELOG.md` | `{{now/date}}` → `{{ now-date }}` |
| `deps.edn` | `{{name}}` → `{{ name }}`; `{{scm/domain}}` → `github.com`; `{{scm/user}}` → `{{ username }}`; `{{main}}` → `{{ project-name }}`; `{{developer}}` → `{{ developer }}` |
| `README.md` | same tags as deps.edn (`{{main}}`, `{{name}}`, `{{scm/*}}`, `{{now/year}}`, `{{developer}}`) |
| `dev/user.clj` | `{{main/ns}}` → `{{ project-name }}` |
| `src/.../core.clj` (from `resources/.../src/core.clj`) | `{{main/ns}}` → `{{ project-name }}` |
| `test/.../core_test.clj` (from `resources/.../test/core_test.clj`) | `{{main/ns}}` → `{{ project-name }}` |

All other `root/` files (`.gitignore`, `.mise.toml`, `bb.edn`, `.cljfmt.edn`,
`.clj-kondo/**` including its `.cache` files, `.github/**`) copy through unchanged.

### Part 3 — verification

Acceptance: deps-new and frame produce **byte-identical** trees for the same inputs.

```bash
OUT=$(mktemp -d)
# deps-new (run in clojure-lib-template repo; :local-new alias pins the local template)
clojure -X:local-new :name io.github.abogoyavlensky/verify-lib :target-dir "\"$OUT/deps-new-out\""
# frame (run in frame repo after lgx build)
bin/frame new --defaults --var username=abogoyavlensky \
  --dir "$OUT/frame-out" /Users/andrew/Projects/clojure-lib-template verify-lib
diff -r "$OUT/deps-new-out" "$OUT/frame-out"   # must print nothing
```

Run for a hyphenated name (`verify-lib`, exercises `snake_case` in paths) and a
plain name (`mylib`). Both generators run the same day, so the date-bearing files
match. `developer` parity relies on frame's `capitalize` matching Clojure's
`str/capitalize` on an all-lowercase username — the diff catches any mismatch.

### Error handling / testing

- Frame changes are TDD'd in frame's existing test suite (`lgx test`, `lgx check`).
- The template migration itself is tested by the diff-based verification; the
  existing deps-new tests (`bb test` in the template repo) must still pass,
  proving the fallback is intact.

## File Structure

**frame repo (branch `master`):**

- Modify: `src/frame/new.lg` — inject `now-date` / `now-year` into the vars map
- Modify: `src/frame/config.lg` — reserved-keys set in `validate-var`
- Modify: `src/frame/prompt.lg` — reserved-keys check in `parse-cli-vars`
- Modify: `README.md` — document the built-ins
- Modify: `test/frame/config_test.lg`, `test/frame/prompt_test.lg`,
  `test/frame/generate_test.lg` — coverage for the above

**clojure-lib-template repo (branch `migrate-to-frame`):**

- Create: `frame.edn`
- Create: `template/**` (copy of `resources/io/github/abogoyavlensky/clojure_lib_template/root/**`
  plus templated `src/` and `test/` dirs, tags rewritten per the table above)
- Modify: `README.md` — add a "Create a project with frame" usage note

## Tasks

### Task 1: frame — reserve `now-date` / `now-year` in config and --var

**Files:**
- Modify: `src/frame/config.lg`, `src/frame/prompt.lg`
- Test: `test/frame/config_test.lg`, `test/frame/prompt_test.lg`

- [ ] **Step 1: Write failing tests**
  In `config_test.lg`: a config declaring `{:key :now-date ...}` in `:vars` raises a
  config error naming the reserved key (mirror the existing `:project-name` reserved
  test). Same for `:now-year`. In `prompt_test.lg`: `parse-cli-vars` rejects
  `now-date=x` and `now-year=x` (mirror the existing `project-name` rejection test).

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: the new assertions FAIL (no error raised yet).

- [ ] **Step 3: Implement**
  In `config.lg` `validate-var` (src/frame/config.lg:23) replace the `:project-name`
  equality check with membership in a reserved set `#{:project-name :now-date :now-year}`;
  error message: `"<key> is reserved and cannot be declared in :vars"`. In
  `prompt.lg` `parse-cli-vars` (src/frame/prompt.lg:35) do the same for the string
  keys, with a message telling that built-in variables cannot be set with `--var`
  (keep the specific project-name wording for `project-name`, or unify — follow
  existing message style).

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [ ] **Step 5: Commit** (frame repo, `master`)
  `git commit -m "feat: reserve now-date/now-year as built-in variable names"`

### Task 2: frame — inject `now-date` / `now-year` values

**Files:**
- Modify: `src/frame/new.lg`
- Test: `test/frame/generate_test.lg` (or the test file that exercises `cmd-new` end-to-end — follow where `project-name` injection is covered)

- [ ] **Step 1: Write failing test**
  End-to-end: a fixture template whose content renders `{{ now-year }}` and
  `{{ now-date }}` produces the current year/date. Compute expected values in the
  test with the same interop calls: `(.Format (now) "2006")` and
  `(.Format (now) "2006-01-02")`. If existing tests exercise `compute-vars`/generate
  below `cmd-new`, inject at the same level the test can reach — keep the injection
  in one obvious place.

- [ ] **Step 2: Run test to verify it fails**
  Run: `lgx test`
  Expected: FAIL with unresolved variable `now-year`.

- [ ] **Step 3: Implement**
  In `src/frame/new.lg` (the `cmd-new` pipeline, where `project-name` is assoc'd
  before `prompt/compute-vars`, src/frame/new.lg:79), add a small private helper
  returning the built-ins map:
  `{"now-date" (.Format (now) "2006-01-02"), "now-year" (.Format (now) "2006")}`
  and merge it into the answers together with `project-name`. Built-ins are merged
  after user answers, so they always win.

- [ ] **Step 4: Run tests + full check**
  Run: `lgx check`
  Expected: PASS (fmt, lint, tests).

- [ ] **Step 5: Document in README**
  In frame's `README.md`, extend the `project-name` built-in paragraph: `now-date`
  (`yyyy-MM-dd`) and `now-year` are always available and reserved. Use /writing-clearly.

- [ ] **Step 6: Commit** (frame repo, `master`)
  `git commit -m "feat: built-in now-date and now-year variables"`

### Task 3: template repo — create `frame.edn` and `template/`

**Files (all in `/Users/andrew/Projects/clojure-lib-template`, branch `migrate-to-frame`):**
- Create: `frame.edn` (exact content in Design above)
- Create: `template/**`

- [ ] **Step 1: Confirm branch**
  Run: `git -C /Users/andrew/Projects/clojure-lib-template branch --show-current`
  Expected: `migrate-to-frame`.

- [ ] **Step 2: Write `frame.edn`**
  Exact content from the Design section.

- [ ] **Step 3: Copy `root/` into `template/`**
  Run: `cp -a resources/io/github/abogoyavlensky/clojure_lib_template/root/ template/`
  (from the template repo root). Verify dotfiles arrived:
  `ls -A template` shows `.github`, `.clj-kondo`, `.gitignore`, `.mise.toml`, etc.

- [ ] **Step 4: Add templated src/ and test/ dirs**
  Create `template/src/{{ project-name | snake_case }}/core.clj` from
  `resources/.../src/core.clj` and
  `template/test/{{ project-name | snake_case }}/core_test.clj` from
  `resources/.../test/core_test.clj` (directory names contain the literal tag text).

- [ ] **Step 5: Rewrite tags**
  Apply the per-file rewrite table from the Design section (LICENSE, CHANGELOG.md,
  deps.edn, README.md, dev/user.clj, core.clj, core_test.clj). Do not touch
  `template/.github/**`. Then verify no deps-new tags remain outside `.github`:
  Run: `grep -rn '{{' template | grep -v '.github' | grep -vE '\{\{ (project-name|name|username|developer|now-date|now-year)( \| [a-z_]+)* \}\}'`
  Expected: no output.

- [ ] **Step 6: Confirm fallback intact**
  Run: `git status --short` — only `frame.edn` and `template/` are new; nothing
  deleted or modified except (later) README. Run `bb test` — deps-new template
  tests still PASS.

- [ ] **Step 7: Commit** (template repo, `migrate-to-frame`)
  `git commit -m "feat: add frame template alongside deps-new"`

### Task 4: verification — byte-identical output

- [ ] **Step 1: Build frame**
  Run in frame repo: `lgx build`
  Expected: `bin/frame` produced.

- [ ] **Step 2: Generate both, hyphenated name**
  Run the verification commands from the Design section (deps-new via
  `clojure -X:local-new` in the template repo, frame via `bin/frame new --defaults
  --var username=abogoyavlensky` in the frame repo), then
  `diff -r "$OUT/deps-new-out" "$OUT/frame-out"`.
  Expected: no output. If files differ, fix the template (or frame, if the engine
  is at fault) and re-run until clean. Also compare tree shape:
  `(cd "$OUT/deps-new-out" && find . | sort) > a; (cd "$OUT/frame-out" && find . | sort) > b; diff a b`.

- [ ] **Step 3: Generate both, plain name**
  Repeat Step 2 with name `mylib` in a fresh temp dir.
  Expected: no diff.

- [ ] **Step 4: Commit any fixes**
  If Step 2/3 required template fixes, commit them in the template repo:
  `git commit -m "fix: match deps-new output exactly"`.

### Task 5: template repo — README usage note

**Files:**
- Modify: `/Users/andrew/Projects/clojure-lib-template/README.md`

- [ ] **Step 1: Add "Create a project with frame" subsection**
  Under Usage, alongside the deps-new instructions: build/install frame, then
  `frame new https://github.com/abogoyavlensky/clojure-lib-template my-lib`
  (interactive) or with `--defaults --var username=...`. Keep the deps-new
  instructions as-is (fallback). Use /writing-clearly.

- [ ] **Step 2: Commit** (template repo, `migrate-to-frame`)
  `git commit -m "docs: frame usage instructions"`
