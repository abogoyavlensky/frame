# Min Frame Version & Unknown-Key Warnings Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let templates declare the minimum frame version they need via an optional `:min-frame-version` key in `frame.edn`, and warn on unrecognized top-level keys.

**Tech Stack:** let-go (lg), lgx tasks, clojure.test

---

## Design

### Problem

Templates are resolved at generation time from git URLs, decoupled from the installed frame binary. A template that uses a newer frame feature fails on an older binary with a confusing downstream error — or, because `read-config` picks only the keys it knows, silently produces wrong output. Template authors need a way to say "don't even try below version X", and users need to see typos and future keys instead of having them silently ignored.

### Solution

Two additions to `frame.edn` handling:

1. **`:min-frame-version`** — an optional top-level key holding an exact semver string (`"0.3.0"`). During `read-config`, right after parsing and *before any var validation*, frame compares it numerically against its own version and throws a `:config` error if the running frame is too old. Checking first matters: a template that needs a newer frame likely also uses config features the old frame rejects (e.g. a new var `:type`), and the user should see "upgrade frame", not `invalid :type`. This keeps the existing invariant that every config failure throws before any prompting.

2. **Unknown-key warnings** — `read-config` collects top-level keys outside the known set (`:description`, `:root`, `:vars`, `:computed`, `:raw`, `:min-frame-version`) and returns them as a `:warnings` vector of strings in the normalized config map (empty when clean). `read-config` stays side-effect-free; `frame.new` prints each warning to stderr as `frame: warning: <msg>` right after reading the config — the orchestration layer already owns all output. Warnings never block generation. Scope is top-level keys only; keys inside var entries stay unchecked.

### Key decisions

- **Single source of truth for frame's version:** a new `frame.version` namespace with `(def current "0.1.0")`. Both `main.lg` (the CLI `:version`) and the config check reference it, so they cannot drift.
- **Strict format:** `:min-frame-version` must be a string matching `^\d+\.\d+\.\d+$`. Anything else — `"1.2"`, `"abc"`, a non-string — is a `:config` error.
- **Numeric comparison:** component-wise on the three integers, so `"0.10.0"` > `"0.9.0"`. No ranges or constraint operators — a floor is all a CLI upgrade path needs (YAGNI).
- **Pure, testable helpers:** the parse/compare/check functions take both versions explicitly so tests never depend on the actual current version. `read-config` supplies `frame.version/current`.
- **Error message names both versions and the fix:** `template requires frame >= 0.3.0, but this is frame 0.1.0 (upgrade frame)`.

### Testing strategy

Pure tests for semver parsing/comparison (ordering, numeric-vs-lexicographic, malformed input) plus `read-edn`-style integration tests through `read-config`: absent key (no error, no warnings), satisfied version (`"0.0.1"`), unsatisfied (`"999.0.0"` → `:config` error), malformed values (`:config` error), unknown keys populate `:warnings`, and the version check firing before var validation.

## File Structure

- Create: `src/frame/version.lg` — single def `current` holding frame's version string. No logic.
- Modify: `src/frame/config.lg` — semver helpers, min-version check, known-key set, `:warnings` in the returned map.
- Modify: `src/frame/new.lg` — print `:warnings` to stderr after `read-config`.
- Modify: `main.lg` — replace the `:version "0.1.0"` literal with `frame.version/current`.
- Modify: `test/frame/config_test.lg` — tests for all new behavior.
- Modify: `README.md` — document `:min-frame-version` and the warning behavior.

## Tasks

### Task 1: Shared version namespace

**Files:**
- Create: `src/frame/version.lg`
- Modify: `main.lg`

- [x] **Step 1: Create `src/frame/version.lg`**
  Namespace `frame.version` containing exactly one def:
  ```clojure
  (def current "0.1.0")
  ```
  A short comment noting this is the single source of truth for frame's version, referenced by the CLI and the `:min-frame-version` config check.

- [x] **Step 2: Use it in `main.lg`**
  Require `frame.version` and replace the `:version "0.1.0"` literal in the `app` map with `version/current`.

- [x] **Step 3: Verify the CLI still reports its version**
  Run: `lgx build && bin/frame --help`
  Expected: help output unchanged, version shown as `0.1.0`.

- [x] **Step 4: Commit**
  `git commit -m "refactor: move frame version to shared frame.version namespace"`

### Task 2: Semver parse, compare, and check helpers

**Files:**
- Modify: `src/frame/config.lg`
- Test: `test/frame/config_test.lg`

- [x] **Step 1: Write failing tests for the helpers**
  In `config_test.lg`, a new section testing two public functions in `frame.config`:
  - `parse-semver` — `"0.3.0"` → `[0 3 0]`; returns `nil` for `"1.2"`, `"1.2.3.4"`, `"abc"`, `"1.2.x"`, `""`, and non-strings.
  - `check-min-version!` — takes `(min-str current-str)`; the shared contract both tasks rely on:
    - `nil` min → no-op (key absent).
    - min ≤ current → returns without throwing (`"0.0.1"` vs `"0.1.0"`, equal versions).
    - min > current → throws ex-info with `{:reason :config}` and a message containing both versions (`"0.2.0"` vs `"0.1.0"`).
    - numeric, not lexicographic: min `"0.9.0"` vs current `"0.10.0"` does NOT throw.
    - malformed min (`"1.2"`, `"abc"`, `1.2` as a number) → throws `{:reason :config}` naming the expected format.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — the new functions don't exist yet.

- [x] **Step 3: Implement the helpers in `config.lg`**
  - `parse-semver`: match `^(\d+)\.(\d+)\.(\d+)$`, return a vector of three ints or `nil`. Guard non-strings.
  - `check-min-version!`: no-op on `nil`; malformed → `config-error` like `":min-frame-version must be an exact semver string like \"0.3.0\", got <val>"`; compare parsed vectors component-wise (plain vector comparison of int vectors works if let-go supports it; otherwise compare the three components explicitly); too-old → `config-error` with message `template requires frame >= <min>, but this is frame <current> (upgrade frame)`.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: add semver parse and min-version check helpers"`

> Deviation: codex review caught that `parse-long` returns nil on overflow, letting a huge component (e.g. `"999999999999999999999.0.0"`) bypass the check; `parse-semver` now rejects any nil component (fixup commit `a1f6f04`).

### Task 3: Wire the check and unknown-key warnings into read-config

**Files:**
- Modify: `src/frame/config.lg`
- Test: `test/frame/config_test.lg`

- [x] **Step 1: Write failing tests through `read-config`**
  Using the existing `read-cfg`/`read-edn` helpers:
  - No `:min-frame-version` → config reads fine, `(:warnings r)` is `[]`.
  - `{:min-frame-version "0.0.1" :vars []}` → reads fine.
  - `{:min-frame-version "999.0.0" :vars []}` → `{:reason :config}` error, message contains `999.0.0` and the current version.
  - `{:min-frame-version "1.2" :vars []}` and `{:min-frame-version 1.2 :vars []}` → `{:reason :config}` error.
  - Version check precedes var validation: `{:min-frame-version "999.0.0" :vars [{:key :db :prompt "P" :type :future-type}]}` → the error message is about the version, not the var type.
  - `{:vars [] :extra-key 1 :typo "x"}` → reads fine, `:warnings` has one entry per unknown key, each naming the key.
  - Non-keyword top-level keys (EDN allows them): `{:vars [] "stray" 1 42 "x"}` → reads fine, one warning per key — must not throw.
  - `reads-full-config` config (all known keys plus `:min-frame-version "0.0.1"`) → `:warnings` is `[]`.

- [x] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: FAIL — no version check in `read-config`, no `:warnings` key.

- [x] **Step 3: Implement in `read-config`**
  - Define `known-keys` as `#{:description :root :vars :computed :raw :min-frame-version}` (private).
  - Require `frame.version`.
  - After the map check and before building the result map: `(check-min-version! (:min-frame-version raw) version/current)`.
  - Build `:warnings`: for each top-level key of `raw` not in `known-keys`, a string like `unrecognized key :extra-key in frame.edn`. Keys may be any EDN value, not just keywords — render and sort them via `pr-str` (never `name`, which throws on non-Named keys).
  - Add `:warnings` to the returned normalized map. Update the ns header comment describing the returned shape.

- [x] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS, including all pre-existing config tests (they must not break from the added `:warnings` key).

- [x] **Step 5: Commit**
  `git commit -m "feat: enforce :min-frame-version and collect unknown-key warnings in read-config"`

### Task 4: Print warnings in frame new

**Files:**
- Modify: `src/frame/new.lg`

- [x] **Step 1: Print warnings after reading config**
  In `run*`, immediately after `(config/read-config template-dir)`, print each warning to stderr:
  `frame: warning: <msg>` via `(write! *err* ...)`, matching the existing error-output style in `cmd-new!`. Warnings print before the project-name prompt so they are visible even if the user cancels.

- [x] **Step 2: Verify end-to-end by hand**
  Run: `lgx build`, then against a scratch template containing an unknown key and `:min-frame-version "0.0.1"`:
  - `bin/frame new --defaults <scratch-template> demo-app` (in a temp dir) → generation succeeds, stderr shows one `frame: warning: unrecognized key ...` line per unknown key.
  - Change the scratch template to `:min-frame-version "999.0.0"` → exits 1 with `frame: template requires frame >= 999.0.0, but this is frame 0.1.0 (upgrade frame)`.
  Clean up the scratch dirs afterwards.

- [x] **Step 3: Commit**
  `git commit -m "feat: print frame.edn warnings on stderr in frame new"`

### Task 5: Documentation and final checks

**Files:**
- Modify: `README.md`

- [x] **Step 1: Document the new behavior**
  In the `### frame.edn` section (use /writing-clearly if available):
  - Add `:min-frame-version "0.3.0"` to the example config with an `; optional` comment.
  - Add a bullet: optional exact semver string; frame refuses to generate (before any prompting) when the installed frame is older; malformed values are config errors.
  - Note that unrecognized top-level keys in `frame.edn` produce a warning on stderr but do not block generation.
  - Extend the existing "reported before any prompting" error list with a too-old frame version.

- [x] **Step 2: Run full checks**
  Run: `lgx check`
  Expected: fmt, lint, and tests all pass. Fix any formatting with `lgx fmt`.

- [x] **Step 3: Commit**
  `git commit -m "docs: document :min-frame-version and unknown-key warnings"`

---

## Completion Summary

**Status: completed** (all 5 tasks, 2026-07-12).

Implemented as designed: `frame.version/current` is the single version source (used by the CLI and the config check); `parse-semver` / `check-min-version!` in `frame.config` enforce an optional exact-semver `:min-frame-version` immediately after parsing and before var validation; `read-config` returns a `:warnings` vector for unrecognized top-level keys, which `frame new` prints to stderr as `frame: warning: ...` before any prompting; README documents both behaviors.

Verified: `lgx check` fully green (fmt, lint 0 warnings, 147 tests / 274 assertions), plus an end-to-end drive of the built binary — generation with a satisfied version and an unknown key (warning printed, project rendered), and rejection with `:min-frame-version "0.1.1"` (exit 1, `template requires frame >= 0.1.1, but this is frame 0.1.0 (upgrade frame)`).

Deviations (all from codex review checkpoints, none changed the design):
- `parse-semver` rejects components that overflow `parse-long` (would otherwise compare as nil and silently pass) — fixup `a1f6f04`.
- README example uses `:min-frame-version "0.1.0"` instead of the plan's `"0.3.0"` so the copyable example works against the current release — fixup `f70e548`.

What the plan could have specified better: the README example version should have been pinned to the current release from the start; everything else held up.
