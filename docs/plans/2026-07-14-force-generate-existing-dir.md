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

**Caveat 1 (accepted):** if a pre-existing entry's *type* conflicts with what the template needs — e.g. the target already has a *file* named `src` but the template writes `src/core.clj` (needs `src` to be a directory) — the underlying `mkdir`/`spit` raises a normal write error. Because `generate!` writes file-by-file with no pre-check against pre-existing target files, that can leave a **partial** result. Rare edge, left as ordinary error behavior (YAGNI).

**Caveat 2 (residual TOCTOU, mitigated not eliminated):** the containment scan runs, then variable prompts may block for input, then generation writes. A concurrent process with write access to the target could swap a plain file for a symlink/hard link/FIFO during the prompt window. Mitigation: the scan is **re-run immediately before `generate!`**, shrinking the exploitable window from indefinite prompt time to the microsecond scan→write gap. It cannot be closed entirely — let-go's fs primitives lack `O_NOFOLLOW`/lstat/atomic-safe-write — so a same-machine attacker who already has write access to your target and can win a microsecond race is out of scope.

### Key decisions

1. **`--force` is a long-only boolean flag** (mirrors `--defaults`; no `-f`).
2. **It relaxes only the "non-empty directory" check.** A target that exists as a *file* (not a directory) still hard-errors — you can't write a tree into a file.
3. **Colliding files are overwritten silently** — no per-file prompt, no `.bak`. Existing `spit` behavior; matches "write on top". Non-templated files survive; nothing is deleted.
4. **Better error hint** when blocked: `target directory already exists and is not empty: <path> (use --force to write into it)`.
5. **`print-summary` wording unchanged** ("Created N files…").
6. **Refactor the gate into a pure `target-error` predicate** (`target × stat × dir-empty? × force? → message|nil`) so the branch logic is unit-testable with a clean truth table and no filesystem; `validate-target!` becomes a thin effectful wrapper that takes `force?`.
7. **Containment guard (added after codex reviews; user chose the "simple guard").** `--force` newly allows a *populated* target, whose contents can defeat containment (all verified): a **symlink** is followed outside the target; a **hard link** shares an inode with an outside file (`spit` truncates it); a **FIFO** makes `spit` hang forever; and a dash-led `--dir` made the scan silently error and pass. Rather than patch types one at a time, the guard generalizes: a populated target is safe only if it holds **nothing but plain regular files and directories**. `validate-target!` shells out to one `find <target> ( ! -type d ! -type f -o -type f -links +1 )` scan and **refuses `--force`** when it finds any special entry (symlink/FIFO/socket/device) or hard-linked regular file. The scan **fails closed** (nonzero `find` exit → unsafe) and `./`-prefixes a relative target so a dash-led `--dir` is scanned, not misparsed. `contains-unsafe-entry?` is public + unit-tested (plain, symlink, hard link, FIFO, fail-closed).

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
> Deviation (codex reviews + user decision): three review rounds found — and I reproduced — four ways `--force` could escape or hang: (a) a pre-existing symlink in the target, (b) a hard link to an outside file (`spit` truncates the shared inode), (c) a dash-led `--dir` that made the `find` scan silently error and pass, and (d) a FIFO that would make `spit` hang forever. Instead of a per-type patch chain, generalized to a "simple guard" that refuses `--force` when the target holds anything but plain files/dirs — an in-spirit extension of the user's choice. Commits: `63ddfd5` (symlink guard) → `a5e9c20` (hard-link/fail-closed/dash-led) → `07693a0` (generalize to all special files incl. FIFO; `contains-links?` renamed `contains-unsafe-entry?`). Verified via the binary that every vector is refused (no hang, outside files untouched) and the normal populated-dir happy path still succeeds. See design decision 7.

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

---

## Completion Summary

**Status: COMPLETE.** All commits are local on `master` (not pushed).

### What was implemented
`frame new --force` generates into an existing non-empty directory, writing template files on top: colliding files are overwritten, untouched files are preserved, the directory is never removed.

- `b0e3ebf` — `--force` flag + pure `target-error` gate + `validate-target!(force?)` + unit truth-table.
- `514167b` — behavioral write-on-top test at the `generate!` level.
- `63ddfd5` → `a5e9c20` → `07693a0` — containment guard, hardened over three review rounds (see below).
- `2a48f9c` — re-check containment right before the write (TOCTOU window mitigation).
- `f598d87`, `fde1e88`, `63039ef` — README + plan docs.

### The security thread (four codex rounds)
`--force` newly lets a *populated* target influence where writes land. Each review round found — and I reproduced — a way to escape or hang, fixed in turn, then generalized:
1. **Symlink** in target → template write followed it outside the target.
2. **Hard link** to an outside file → `spit` truncated the shared inode.
3. **Dash-led `--dir`** → the `find` scan silently errored and passed (fixed: fail-closed + `./` prefix).
4. **FIFO** at a template path → `spit` would hang forever.

Final guard (`contains-unsafe-entry?`): one `find <target> ( ! -type d ! -type f -o -type f -links +1 )` scan refuses `--force` when the target holds anything but plain files/dirs (any special entry or hard-linked file); fails closed; path-safe. Verified via the binary that all four vectors are refused (no hang, outside files untouched) and the normal populated-dir happy path still succeeds.

### Verification
- `lgx test`: 159 tests / 296 assertions, 0 failures. Lint + fmt clean.
- e2e (built binary): refuse-without-force (exit 1) / success-with-force (exit 0) / unrelated-preserved / colliding-overwritten / symlink+hardlink+FIFO+dash-led all refused / `--help` shows `--force`.

### Deviations & open items
- **Scope grew** well beyond the planned "small contained change": a full containment guard (4 fix commits) driven by codex review. Done in-spirit with the user's "simple guard" choice (refuse suspicious targets), but the user should sanity-check the added surface.
- **Residual TOCTOU (mitigated, not eliminated):** a same-machine attacker with write access to the target who wins a microsecond scan→write race could still escape. Cannot be closed with let-go's fs primitives (no `O_NOFOLLOW`/lstat/atomic-safe write). Documented; **I stopped the review-hardening loop here** rather than chase unfixable variants.
- **Not pushed:** all commits local, per the standing "don't push" context; frame still pins tiny-cli at the local-only `v0.2.3` tag.

### What the plan could have specified better
The plan treated `--force` as a pure gate-relaxation and never considered that allowing a *populated, possibly-adversarial* target is a security boundary. A one-line "what can a hostile target directory do?" threat check in the design would have surfaced the symlink/hard-link/FIFO/TOCTOU class up front instead of across four review rounds — worth adding to the plan template for any feature that widens what untrusted inputs (remote templates, existing dirs) can influence.
