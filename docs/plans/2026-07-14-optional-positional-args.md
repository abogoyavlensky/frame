# Optional Positional Args Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `:optional? true` support for the last positional arg in tiny-cli, then migrate frame's `new` command from a variadic `name-args` to a real optional `name` positional.

**Tech Stack:** Clojure (`.cljc`, runs on let-go/Clojure/Babashka), let-go (`.lg`), lgx build tool.

---

## Design

### Context

**tiny-cli** (`../tiny-cli`, separate repo) models command positionals as:
- `:args` — a vector of fixed positional specs, **all required**, in order.
- `:variadic` — one optional spec that slurps every trailing token into a vector.

**frame** currently abuses `:variadic {:key :name-args}` to model an *optional single* project name. In `src/frame/new.lg` it then manually rejects more than one token (`"unexpected extra arguments"`) and takes `(first name-args)`. frame consumes tiny-cli by git tag: `:git/tag "v0.2.2"` in `lgx.edn`.

### Part A — tiny-cli: `:optional? true` on the last positional arg

**Semantics.** A fixed arg may declare `:optional? true`, valid **only on the last entry of `:args`**. When omitted at the CLI, its key is **absent** from the handler's `:args` map (so a lookup returns `nil`) — consistent with how unset options behave. No `:default` support for args (not requested; YAGNI).

**Parsing** — one change in `finalize-context`: derive a `required-count` (`arg-count` minus one when the last fixed arg is optional) and gate the *"Missing argument"* error on `required-count` instead of `arg-count`. The *"Too many arguments"* ceiling stays at `arg-count`. The existing zip that builds the `:args` map (`(map vector (map :key fixed-specs) positionals)`) already drops an omitted trailing arg, so no other parsing change is needed. Non-variadic interleaving of options/positionals is untouched.

**Spec validation** — two new checks in `command-spec-error` (runs at app-spec validation time):
1. `:optional?` on any non-last arg → spec error (message contains `Optional arg must be last`).
2. `:optional?` combined with a `:variadic` → spec error (message contains `Optional arg cannot be combined with a variadic`). A variadic already makes the tail optional; an optional fixed arg before it is ambiguous, so the combo is forbidden.

**Help** — `arg-placeholder` renders an optional arg as `[NAME]` instead of `<NAME>` in usage lines (root help + command help). Completion needs **no change** (`positional-spec` maps by index regardless of optionality).

**Release** — new git tag **`v0.2.3`** via `lgx release 0.2.3`. Pushing the tag is outward-facing; pause for the user's go-ahead before `git push --tags`.

### Part B — frame: variadic → optional positional

- `main.lg`: replace the `:variadic {:key :name-args}` block with a second fixed arg `{:key :name :optional? true ...}`; bump `:git/tag` to `v0.2.3`.
- `src/frame/new.lg` `run*`: read `(get-in ctx [:args :name])` (a string or `nil`) and pass it straight to `resolve-project-name!` (already nil-safe). **Delete** the manual `(> (count name-args) 1)` / `"unexpected extra arguments"` check — extra positionals are now rejected by tiny-cli as `"Too many arguments."` at parse time (exit 2). `resolve-project-name!` itself is unchanged.

### Key decisions

1. Omitted optional arg → **absent** from the `:args` map (nil on lookup).
2. `:optional?` + `:variadic` together is **forbidden** (spec error).
3. Usage renders an optional arg as `[NAME]`.
4. frame drops its custom extra-args error and relies on tiny-cli's `"Too many arguments."` (exit 2).
5. tiny-cli released as **`v0.2.3`**; frame bumps the tag to match.

### Sequencing (cross-repo)

This plan lives in the frame repo, but **Part A lands in the tiny-cli repo** and must be fully implemented, tested, and **released (tag pushed)** before Part B, because frame pins tiny-cli by tag. Order: tiny-cli tasks 1–5 → frame tasks 6–7.

---

## File Structure

**tiny-cli repo (`../tiny-cli`):**
- Modify `src/tiny_cli/core.cljc` — validation checks, `finalize-context` count logic, `arg-placeholder`.
- Modify `test/tiny_cli/core_test.cljc` — optional-arg parsing, validation, and help tests.
- Modify `README.md` — args section (drop "every declared arg is required" absolute; document `:optional?`).

**frame repo (this repo):**
- Modify `lgx.edn` — bump tiny-cli `:git/tag` to `v0.2.3`.
- Modify `main.lg` — swap variadic for an optional `:name` arg.
- Modify `src/frame/new.lg` — read `:name`, delete the manual extra-args guard.

---

## Task 1: tiny-cli — spec validation for `:optional?`

**Files:**
- Modify: `../tiny-cli/src/tiny_cli/core.cljc`
- Test: `../tiny-cli/test/tiny_cli/core_test.cljc`

- [ ] **Step 1: Write failing tests**
  In `core_test.cljc`, add a `deftest` covering two spec errors, each parsed via `cli/parse` on a minimal app:
  - Optional on a non-last arg, e.g. `:args [{:key :a :optional? true} {:key :b}]` → `:status :error`, message matches `#"Optional arg must be last"`.
  - Optional combined with variadic, e.g. `:args [{:key :a :optional? true}] :variadic {:key :rest}` → `:status :error`, message matches `#"Optional arg cannot be combined with a variadic"`.

- [ ] **Step 2: Run tests to verify they fail**
  Run: `cd ../tiny-cli && lgx test`
  Expected: FAIL (the two new assertions; currently no such validation).

- [ ] **Step 3: Implement the validation**
  In `core.cljc`, add two private helpers and wire them into `command-spec-error` (near the existing duplicate-arg-key check). Use these shapes so the tests match exactly:
  ```clojure
  (defn- misplaced-optional-arg
    "An arg with :optional? that is not the last in :args, else nil."
    [args]
    (first (filter :optional? (butlast args))))
  ```
  Add cond branches to `command-spec-error`:
  ```clojure
  (misplaced-optional-arg (:args command))
  (error-result (str "Optional arg must be last: "
                     (key-placeholder (:key (misplaced-optional-arg (:args command))))))

  (and (:variadic command) (some :optional? (:args command)))
  (error-result "Optional arg cannot be combined with a variadic arg.")
  ```

- [ ] **Step 4: Run tests to verify they pass**
  Run: `cd ../tiny-cli && lgx test`
  Expected: PASS.

- [ ] **Step 5: Commit**
  `cd ../tiny-cli && git commit -am "feat: validate :optional? arg placement"`

## Task 2: tiny-cli — optional arg parsing

**Files:**
- Modify: `../tiny-cli/src/tiny_cli/core.cljc`
- Test: `../tiny-cli/test/tiny_cli/core_test.cljc`

- [ ] **Step 1: Write failing tests**
  Add a `deftest` using an app whose command has `:args [{:key :src} {:key :name :optional? true}]` and a `:run` stub. Assert via `cli/parse`:
  - Both provided → `:args` is `{:src "s" :name "n"}`.
  - Optional omitted → `:args` is `{:src "s"}` (no `:name` key; `(contains? args :name)` is false).
  - Required omitted (zero positionals) → `:error`, message matches `#"Missing argument"`.
  - Too many (three positionals) → `:error`, message matches `#"Too many arguments"`.

- [ ] **Step 2: Run tests to verify they fail**
  Run: `cd ../tiny-cli && lgx test`
  Expected: FAIL (omitting the optional currently triggers `Missing argument`).

- [ ] **Step 3: Implement the count change**
  In `finalize-context`, compute `required-count` and gate the missing-arg check on it (leave the too-many check at `arg-count`):
  ```clojure
  required-count (if (and (seq fixed-specs) (:optional? (last fixed-specs)))
                   (dec arg-count)
                   arg-count)
  ```
  Change `(< provided-count arg-count)` to `(< provided-count required-count)`. The `Missing argument` message still uses `(nth fixed-specs provided-count)` (safe: `provided-count < required-count <= arg-count`). The map-building `fixed`/`args` code is unchanged.

- [ ] **Step 4: Run tests to verify they pass**
  Run: `cd ../tiny-cli && lgx test`
  Expected: PASS. Existing variadic and fixed-arg tests still pass.

- [ ] **Step 5: Commit**
  `cd ../tiny-cli && git commit -am "feat: parse optional trailing positional arg"`

## Task 3: tiny-cli — optional arg help placeholder

**Files:**
- Modify: `../tiny-cli/src/tiny_cli/core.cljc`
- Test: `../tiny-cli/test/tiny_cli/core_test.cljc`

- [ ] **Step 1: Write failing test**
  Add a test asserting `cli/command-help` for a command with an optional last arg renders `[NAME]` (regex `#"\[NAME\]"`) while a required arg still renders `<SRC>`.

- [ ] **Step 2: Run test to verify it fails**
  Run: `cd ../tiny-cli && lgx test`
  Expected: FAIL (optional currently renders as `<NAME>`).

- [ ] **Step 3: Implement the placeholder**
  Update `arg-placeholder`:
  ```clojure
  (defn- arg-placeholder
    [arg]
    (let [ph (key-placeholder (:key arg))]
      (if (:optional? arg)
        (str "[" ph "]")
        (str "<" ph ">"))))
  ```

- [ ] **Step 4: Run test to verify it passes**
  Run: `cd ../tiny-cli && lgx test`
  Expected: PASS.

- [ ] **Step 5: Commit**
  `cd ../tiny-cli && git commit -am "feat: render optional arg as [NAME] in help"`

## Task 4: tiny-cli — docs + cross-runtime verification

**Files:**
- Modify: `../tiny-cli/README.md`

- [ ] **Step 1: Update README args section**
  In the arg-spec block (around the `:args` comment "Every declared arg is required."), document `:optional? true`: allowed only on the last arg, absent from the handler `:args` map when omitted, incompatible with `:variadic`, and rendered `[KEY]` in usage. Soften the "every declared arg is required" line accordingly. Use /writing-clearly.

- [ ] **Step 2: Run full checks across all runtimes**
  Run: `cd ../tiny-cli && lgx fmt check && lgx lint && lgx test-all`
  Expected: PASS on let-go, Clojure, and Babashka.

- [ ] **Step 3: Commit**
  `cd ../tiny-cli && git commit -am "docs: document :optional? positional args"`

## Task 5: tiny-cli — release v0.2.3

**Files:** none (git tag + push).

- [ ] **Step 1: Confirm before releasing**
  This pushes a tag and triggers the GitHub release workflow (outward-facing). Get the user's explicit go-ahead first. Ensure the tiny-cli commits are on the branch that should be tagged (its normal release flow tags the current mainline).

- [ ] **Step 2: Tag and push**
  Run: `cd ../tiny-cli && lgx release 0.2.3`
  Expected: creates tag `v0.2.3` and pushes it; the `release` workflow publishes the GitHub release.

- [ ] **Step 3: Verify the tag is on the remote**
  Run: `cd ../tiny-cli && git ls-remote --tags origin | grep v0.2.3`
  Expected: the `v0.2.3` tag ref is listed (frame's lgx must be able to fetch it).

## Task 6: frame — migrate to the optional `name` arg

**Files:**
- Modify: `lgx.edn`
- Modify: `main.lg`
- Modify: `src/frame/new.lg`

- [ ] **Step 1: Bump the tiny-cli tag**
  In `lgx.edn`, change tiny-cli `:git/tag "v0.2.2"` → `"v0.2.3"`.

- [ ] **Step 2: Swap variadic for an optional arg in `main.lg`**
  Replace the `:variadic {:key :name-args ...}` entry with a second fixed arg so `:args` reads:
  ```clojure
  :args [{:key :source
          :doc "Template source: a local directory or https://host/owner/repo."}
         {:key :name
          :optional? true
          :doc "Project name (optional; prompted when omitted)."}]
  ```
  Remove the `:variadic` key entirely.

- [ ] **Step 3: Update `run*` in `src/frame/new.lg`**
  Read the name as `(get-in ctx [:args :name])` (string or nil) and pass it to `resolve-project-name!`. Delete the `name-args` binding and the `(when (> (count name-args) 1) ...)` "unexpected extra arguments" guard — tiny-cli now rejects extras as `"Too many arguments."`. Leave `resolve-project-name!` unchanged.

- [ ] **Step 4: Run frame checks**
  Run: `lgx fmt check && lgx lint && lgx test`
  Expected: PASS (fetches tiny-cli `v0.2.3`).

- [ ] **Step 5: Commit**
  `git commit -am "feat: make project name an optional positional arg"`

## Task 7: frame — verify end to end

**Files:** none (manual verification).

- [ ] **Step 1: Build the binary**
  Run: `lgx build` (or the project's build task) to produce `bin/frame`.

- [ ] **Step 2: Verify help renders the optional arg**
  Run: `bin/frame new --help`
  Expected: usage shows `frame new <SOURCE> [NAME]` (optional in brackets).

- [ ] **Step 3: Verify positional name and too-many-args**
  Run a generation with an explicit name from a local template fixture (e.g. `test/frame/fixtures/demo-template`) into a temp dir and confirm the project name is used without prompting. Then run `bin/frame new <template> a b` and confirm it errors with `Too many arguments.` (exit 2).

- [ ] **Step 4: Commit any fixups**
  If verification surfaced fixes, commit them: `git commit -am "fix: <description>"`. Otherwise no commit.
