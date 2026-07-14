# `--force` Generate Into Existing Directory Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `--force` flag to `frame new` that generates into an existing non-empty directory, writing template files on top without removing the directory or its untouched files.

**Tech Stack:** let-go (`.lg`), lgx build tool, clojure.test.

---

## Design

### Context

`frame new` resolves a target dir (`--dir`, else `<cwd>/<project-name>`) and calls `validate-target!` (`src/frame/new.lg:26`) before generating. Today `validate-target!` **fails** when the target is a non-empty directory or exists as a non-directory file; it proceeds only when the target is absent or an empty directory.

Generation itself (`src/frame/generate.lg`, `generate!`) already does the right thing for "write on top": it renders everything in memory, checks collisions **only within the template**, then `spit`s each file (overwriting any same-named file) and **never deletes** anything. So the only thing blocking a non-empty target is the `validate-target!` gate.

### Approach

`--force` is a small, contained change that relaxes that one gate. No merge/backup/skip machinery.

**Behavior with `--force`:** template files are written on top; a produced file that already exists is **overwritten**; a pre-existing file the template doesn't touch is **left untouched**; the directory is never removed.

**Caveat (accepted, not handled here):** if a pre-existing entry's *type* conflicts with what the template needs — e.g. the target already has a *file* named `src` but the template writes `src/core.clj` (needs `src` to be a directory) — the underlying `mkdir`/`spit` raises a normal write error. Because `generate!` writes file-by-file with no pre-check against pre-existing target files, that can leave a **partial** result. This is a rare edge (only reachable now that `--force` allows a populated target) and is left as ordinary error behavior; a pre-write type-conflict check is out of scope (YAGNI).

### Key decisions

1. **`--force` is a long-only boolean flag** (mirrors `--defaults`; no `-f`).
2. **It relaxes only the "non-empty directory" check.** A target that exists as a *file* (not a directory) still hard-errors — you can't write a tree into a file.
3. **Colliding files are overwritten silently** — no per-file prompt, no `.bak`. Existing `spit` behavior; matches "write on top". Non-templated files survive; nothing is deleted.
4. **Better error hint** when blocked: `target directory already exists and is not empty: <path> (use --force to write into it)`.
5. **`print-summary` wording unchanged** ("Created N files…").
6. **Refactor the gate into a pure `target-error` predicate** (`target × stat × dir-empty? × force? → message|nil`) so the branch logic is unit-testable with a clean truth table and no filesystem; `validate-target!` becomes a thin effectful wrapper that takes `force?`.
7. **Link containment guard (added after codex P1s; user chose the "simple guard").** `--force` newly allows a *populated* target, which can contain links that make a template write escape the target (verified: a symlink is followed; a hard link shares an inode with an outside file — both bypass the lexical `unsafe-dst?` check). let-go's `os/stat` follows symlinks and exposes no link count, so `validate-target!` shells out to a single `find <target> ( -type l -o -type f -links +1 )` scan and **refuses `--force` when the target tree contains any symlink or hard-linked file**. The scan **fails closed** (a nonzero `find` exit → treated as unsafe) and prefixes a relative target with `./` so a dash-led `--dir` is scanned rather than misparsed as a find expression. Coarse but safe. `contains-links?` is public + unit-tested (symlink, hard link, fail-closed).

### Testing strategy

- Pure unit truth-table over `target-error` (no filesystem).
- Behavioral test at the `generate!` level: generating into a pre-populated dir preserves an unrelated file and overwrites a colliding one (locks in write-on-top).
- End-to-end against the built binary: non-empty `--dir` fails without `--force`, succeeds with it, unrelated file preserved, colliding file overwritten.

### Note (out of scope)

Frame's `lgx.edn` pins tiny-cli at `:git/tag "v0.2.3"`, a tag that currently exists only in the local tiny-cli repo (not pushed to the remote). Frame builds/tests fine locally today. This is a pre-existing release caveat, unrelated to `--force`, and is not addressed here.

---

## File Structure

- Modify `main.lg` — add `--force` to `new`'s `:opts`.
- Modify `src/frame/new.lg` — add pure `target-error`; `validate-target!` takes `force?`; `run*` passes it through.
- Modify `test/frame/new_test.lg` — truth-table unit tests for `target-error`.
- Modify `test/frame/generate_test.lg` — behavioral write-on-top test.
- Modify `README.md` — `--force` options row + example.

---

## Task 1: `--force` flag, `target-error` predicate, and gate refactor

**Files:**
- Modify: `main.lg`
- Modify: `src/frame/new.lg`
- Test: `test/frame/new_test.lg`

> Deviation: Task 1 is the whole production change; Task 2 only adds a behavioral test and Tasks 3–4 are docs/verify. Running review-with-codex once over Tasks 1–2 (code + tests complete) rather than per-task.
>
> Deviation (codex P1s + user decision): the codex reviews found — and I reproduced — three containment escapes `--force` enabled: (a) a pre-existing symlink in the target, (b) a pre-existing hard link to an outside file (`spit` truncates the shared inode), and (c) a dash-led `--dir` that made the `find` scan silently error and pass. Added a link-containment guard (`contains-links?` via one `find ( -type l -o -type f -links +1 )` scan; fails closed; `./`-prefixes relative paths) that refuses `--force` when the target holds any symlink or hard-linked file — an in-spirit extension of the user's "simple guard" choice. Commits `63ddfd5` (initial symlink guard) + `a5e9c20` (hard-link/fail-closed/dash-led hardening). Verified via the binary that all three vectors are blocked and outside files are untouched, and the normal populated-dir happy path still succeeds. See design decision 7.

- [x] **Step 1: Write failing unit tests**
  In `new_test.lg`, add a `deftest` calling `new/target-error` (a new public fn) as a pure truth table — no filesystem:
  - `(new/target-error "t" nil true false)` → nil (absent target).
  - `(new/target-error "t" nil true true)` → nil.
  - `(new/target-error "t" {:dir? false} true false)` → non-nil, message matches `#"not a directory"`.
  - `(new/target-error "t" {:dir? false} true true)` → non-nil (file target still errors under force).
  - `(new/target-error "t" {:dir? true} true false)` → nil (empty dir).
  - `(new/target-error "t" {:dir? true} false false)` → non-nil, message matches `#"--force"`.
  - `(new/target-error "t" {:dir? true} false true)` → nil (non-empty + force).

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL (`target-error` is undefined / unresolved var).

- [x] **Step 3: Implement `target-error` + refactor `validate-target!`**
  In `src/frame/new.lg`, add the pure predicate and rewrite `validate-target!` to take `force?`. Use these shapes so the tests match exactly:
  ```clojure
  (defn target-error
    "Message describing why target can't be generated into, or nil when writable.
     stat is (os/stat target) — nil when absent; dir-empty? whether an existing
     target dir has no entries; force? the --force flag."
    [target stat dir-empty? force?]
    (cond
      (nil? stat) nil
      (not (:dir? stat)) (str "target exists and is not a directory: " target)
      (and (not dir-empty?) (not force?))
      (str "target directory already exists and is not empty: " target
           " (use --force to write into it)")
      :else nil))

  (defn- validate-target! [target force?]
    (let [stat (os/stat target)
          dir-empty? (or (nil? stat)
                         (not (:dir? stat))
                         (empty? (os/ls target)))]
      (when-let [msg (target-error target stat dir-empty? force?)]
        (cli-error msg))
      target))
  ```
  The `or` short-circuits so `os/ls` runs only on an existing directory.

- [x] **Step 4: Wire `--force` through `run*` and `main.lg`**
  - In `run*`, bind `force? (boolean (:force? opts))` and call `(validate-target! target force?)`.
  - In `main.lg`, add to `new`'s `:opts`:
    ```clojure
    {:key :force?
     :long "force"
     :doc "Generate into the target directory even if it already contains files (overwrites on collision)."}
    ```

- [x] **Step 5: Run checks to verify they pass**
  Run: `lgx fmt check && lgx lint && lgx test`
  Expected: PASS, 0 lint warnings.

- [x] **Step 6: Commit**
  `git commit -am "feat: add --force to generate into an existing directory"`

## Task 2: Behavioral write-on-top test

**Files:**
- Test: `test/frame/generate_test.lg`

- [x] **Step 1: Write the behavioral test**
  In `generate_test.lg`, add a `deftest` (reuse the file's `tmp-dir` helper). Build a minimal template with a colliding file, pre-populate the target with one unrelated and one colliding file, run `generate!`, and assert:
  ```clojure
  ;; template: template/README.md -> "from-template"; frame.edn {:vars []}
  ;; target pre-populated: keep.txt -> "keep-me", README.md -> "old-content"
  (generate/generate! tpl target {"project-name" "x"} cfg)
  (is (= "keep-me" (slurp (str target "/keep.txt"))))        ; unrelated preserved
  (is (= "from-template" (slurp (str target "/README.md")))) ; colliding overwritten
  ```
  Clean up with `(os/sh "rm" "-rf" tpl target)`. (This exercises `generate!` directly, which does not gate on non-empty — it documents the write-on-top semantics `--force` unlocks.)

- [x] **Step 2: Run tests to verify pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 3: Commit**
  `git commit -am "test: cover write-on-top generation into an existing dir"`

## Task 3: Document `--force`

**Files:**
- Modify: `README.md`

- [x] **Step 1: Add the options row and an example**
  In the `frame new` Options table (around `README.md:22`), add:
  ```
  | `--force` | Generate into the target directory even when it already contains files. Existing files with the same path are overwritten; other files are left untouched; the directory is never removed. |
  ```
  Add an example under the Examples block, e.g.:
  ```bash
  # Write into an existing directory, overwriting only colliding files
  frame new ./my-template my-app --defaults --force
  ```
  Use /writing-clearly.

- [x] **Step 2: Commit**
  `git commit -am "docs: document frame new --force"`

## Task 4: Verify end to end

**Files:** none (manual verification).

- [x] **Step 1: Build the binary**
  Run: `lgx build`
  Expected: builds `bin/frame`.

- [x] **Step 2: Drive the flag against an existing non-empty dir**
  Pre-populate a temp dir with both an *unrelated* file and one that *collides* with a template output (README.md), so the e2e checks preservation AND overwrite:
  ```bash
  DIR=$(mktemp -d)
  printf keep > "$DIR/keep.txt"
  printf old-content > "$DIR/README.md"
  TMPL=test/frame/fixtures/demo-template
  bin/frame new "$TMPL" myproj --defaults --dir "$DIR";          echo "exit=$?"   # expect frame: ... not empty, exit 1
  bin/frame new "$TMPL" myproj --defaults --dir "$DIR" --force;  echo "exit=$?"   # expect success, exit 0
  cat "$DIR/keep.txt"    # expect: keep  (unrelated file preserved)
  cat "$DIR/README.md"   # expect the template's README content, NOT "old-content" (colliding file overwritten)
  find "$DIR" -type f | head    # expect template files present alongside keep.txt
  rm -rf "$DIR"
  ```
  Confirm: without `--force` it errors "not empty" (exit 1) and writes nothing; with `--force` it succeeds, `keep.txt` survives, README.md is overwritten, and template files are generated on top.

- [x] **Step 3: Full suite + help**
  Run: `lgx test` (full suite green) and `bin/frame new --help` (confirm `--force` appears in the options).

- [x] **Step 4: Commit any fixups**
  If verification surfaced fixes, commit them; otherwise no commit.
