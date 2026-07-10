# Variable Summary After Project Creation Implementation Plan

> **Status: COMPLETED** (2026-07-10) — see summary at the end.

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** After a successful `frame new`, print a `Variables:` block listing each declared template variable with the value that was actually used (from a prompt, `--var`, or a default).

**Tech Stack:** frame (let-go/lgx); changes confined to `src/frame/new.lg`, a new test file, and a one-line `README.md` update.

---

## Design

### Approach

This is purely an output change. At the success point of `run*` in `src/frame/new.lg`, both inputs already exist: `cfg` (the normalized config with `:vars` in `frame.edn` order) and `final` (the resolved string-keyed answers map, including built-ins and computed values). No changes to prompting, answer resolution, or generation.

The success output gains a `Variables:` block between the `Created …` line and `Next steps:`:

```
Created my-app (12 files) at /path/to/my-app

Variables:
  db: postgres
  auth: false
  author: Jane

Next steps:
  cd my-app
```

### Key decisions

- **Show declared `:vars` only, in `frame.edn` order.** Built-ins (`now-date`, `now-year`) and `:computed` values are not picked by the user, and `project-name` already appears in the `Created` line. The block answers "what did I choose?", not "what did the renderer see?".
- **Skip the block entirely when the template declares no vars** — zero-var templates keep today's output exactly.
- **Plain value printing.** Booleans render as `true`/`false`, enums and strings as-is via `str`; an empty string shows as an empty value after the colon. No quoting.
- **Testable core.** Extract a pure helper `var-summary-lines` in `frame.new` that maps config vars + final answers to a vector of output lines (`[]` when there are no vars). `print-summary` prints those lines; only the helper is unit-tested — printing stays untested like the rest of the CLI output.

### Shared shape

So the implementation and tests agree exactly:

- `(var-summary-lines vars answers)` where `vars` is `(:vars cfg)` and `answers` is the final answers map.
- Returns `[]` when `vars` is empty; otherwise `["Variables:" "  <key>: <value>" ...]`, one line per var in `vars` order, where for each var `skey` is `(name (:key var))`, the printed key is `skey`, and the printed value is `(str (get answers skey))`.

## File Structure

- Modify: `src/frame/new.lg` — add `var-summary-lines` (public, for testability); extend `print-summary` to take `cfg` and `final` and print the block between the `Created` line and `Next steps:`; update the call site in `run*`.
- Create: `test/frame/new_test.lg` — unit tests for `var-summary-lines`.
- Modify: `README.md` — mention the variable summary in the success-output sentence.

## Tasks

### Task 1: `var-summary-lines` with tests

**Files:**
- Modify: `src/frame/new.lg`
- Test: `test/frame/new_test.lg`

- [x] **Step 1: Write the failing tests**
  Create `test/frame/new_test.lg` (namespace `frame.new-test`, requiring `clojure.test` and `frame.new`, following the style of `test/frame/generate_test.lg`). Cases for `var-summary-lines`:
  - Mixed types and ordering: vars `[{:key :db :type :enum ...} {:key :auth :type :boolean ...} {:key :author :type :string ...}]` with answers `{"db" "postgres" "auth" false "author" "Jane"}` → `["Variables:" "  db: postgres" "  auth: false" "  author: Jane"]` (config order preserved, boolean rendered as `false`).
  - Empty vars: `(var-summary-lines [] {...})` → `[]`.
  - Extras ignored: answers containing `now-date`, `now-year`, `project-name`, and a computed key produce lines only for the declared vars.
  - Empty string value: answer `""` renders as `"  author: "`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — `var-summary-lines` unresolved in `frame.new`.

- [x] **Step 3: Implement**
  In `src/frame/new.lg`, add public `var-summary-lines` per the shared shape above. Extend `print-summary` to `(print-summary n project-name target cfg final)`: after the `Created` line, when `(seq (:vars cfg))`, print a blank line then each summary line; keep the existing blank line + `Next steps:` tail unchanged. Update the call in `run*` accordingly.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS (all suites).

- [x] **Step 5: Commit**
  `git commit -m "feat: print variable summary after project creation"` (c47fea4)

### Task 2: Verify end-to-end and finish

**Files:**
- Modify: `README.md`

- [x] **Step 1: Full checks**
  Run: `lgx check`
  Expected: fmt, lint, and tests all pass.
  > Deviation: the first `lgx check` after Task 1 failed on `cljfmt check` for the new test file (flagged by the Task 1 codex review); fixed with `lgx fmt` and committed as `style: format new_test fixture with cljfmt`.

- [x] **Step 2: Manual smoke**
  Run: `lgx build && bin/frame new --defaults --dir "$(mktemp -d)/demo-app" test/frame/fixtures/demo-template demo-app`
  Expected: output shows the `Created …` line, then a `Variables:` block with the demo template's vars at their defaults, then `Next steps:`.

- [x] **Step 3: Update README output description**
  In `README.md`, the "On success `frame` prints…" sentence: mention the variable summary.

- [x] **Step 4: Commit**
  `git commit -m "docs: mention variable summary in README"` (685da03)

---

## Completion Summary (2026-07-10)

**Implemented:** `frame new` now prints a `Variables:` block between the `Created …` line and `Next steps:`, listing each declared var (in `frame.edn` order) with the value actually used — from a prompt, `--var`, or a default. Zero-var templates keep the old output. The core is a public, unit-tested `var-summary-lines` in `frame.new`; built-ins, computed values, and `project-name` are excluded.

**Commits:** `c47fea4` feature + tests, `db38c4f` cljfmt fixup, `685da03` README mention, `0cf7b1d` README wording fix.

**Verification:** `lgx check` green (fmt, lint, 132 tests / 238 assertions); end-to-end smoke of `bin/frame new --defaults` (defaults shown) and with `--var db=postgres,auth=false` (overrides shown).

**Issues / deviations:**
- Task 1's codex review caught that the new test file failed `cljfmt check`; fixed with `lgx fmt` (`db38c4f`).
- Task 2's codex review caught that the README claimed "every variable" while only declared vars are printed; wording scoped to declared variables (`0cf7b1d`).

**What the plan could have specified better:** a "run `lgx fmt` before committing" reminder in the test-writing step — the repo's `check` gates on formatting, and new files won't be formatted by default.
