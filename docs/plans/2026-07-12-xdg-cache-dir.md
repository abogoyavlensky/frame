# XDG Cache Directory Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move frame's downloaded-template cache out of `~/.frame` and into the XDG cache directory (default `~/.cache/frame`), centralizing base-directory resolution in a new `frame.paths` namespace so a future aliases-config location can reuse the same scheme.

**Tech Stack:** frame (let-go/lgx); a new `src/frame/paths.lg` + test, edits to `src/frame/source.lg`, and `README.md`.

---

## Design

### Motivation
Today all of frame's on-disk state lives under `~/.frame` (via `FRAME_HOME`, default `~/.frame`). That pollutes `$HOME` and conflates two kinds of state with different lifecycles: the **template cache** (disposable, re-clonable, content-addressed by SHA) and a **future aliases config** (precious, user-authored, must survive a cache wipe). Splitting by XDG category makes the semantics explicit — cache is safe to delete, config is not.

This plan moves the template cache to the XDG **cache** home and pre-decides (but does not build) the config location, so aliases can slot in later with a symmetric ~3-line addition.

### Resolution rules
A new namespace `frame.paths` owns base-directory resolution. `FRAME_HOME` remains a single "everything under one dir" override; otherwise XDG defaults apply.

For the template cache:

| `FRAME_HOME` | `XDG_CACHE_HOME` | Templates land in |
|---|---|---|
| set (non-blank) | (ignored) | `$FRAME_HOME/templates/<host>/<owner>/<repo>/<sha>/` — identical to today |
| unset/blank | set (non-blank) | `$XDG_CACHE_HOME/frame/templates/...` |
| unset/blank | unset/blank | `~/.cache/frame/templates/...` — new default |

Consequences:
- **Existing `FRAME_HOME` users are unaffected** — the layout under `$FRAME_HOME` is byte-for-byte the same.
- Only users relying on the implicit `~/.frame` default move to `~/.cache/frame`. This is cheap: the cache is content-addressed and re-clonable, so the worst case is one re-clone.
- The `FRAME_HOME` override appends **no** `/frame` segment (templates go directly to `$FRAME_HOME/templates`, preserving the current exact layout). Only the XDG default appends `/frame`, because `~/.cache` is shared across applications.
- **Per the XDG Base Directory spec, a *relative* `XDG_CACHE_HOME` is invalid and ignored** — it falls back to `$HOME/.cache`, exactly as if unset. `app-dir` treats an xdg value as usable only when it is non-blank **and** absolute (starts with `/`). This rule applies to the XDG variable only; `FRAME_HOME` is frame's own override and is used verbatim (its current behavior is preserved).

### Future config location (documented, NOT implemented here)
When aliases land, `frame.paths` will gain a symmetric `config-dir`:

| `FRAME_HOME` | `XDG_CONFIG_HOME` | Config lives in |
|---|---|---|
| set (non-blank) | (ignored) | `$FRAME_HOME/config.edn` |
| unset/blank | set (non-blank) | `$XDG_CONFIG_HOME/frame/config.edn` |
| unset/blank | unset/blank | `~/.config/frame/config.edn` |

This is documented in `frame.paths` as a comment and here in the plan so the future addition follows the identical pattern. **Do not implement `config-dir` in this plan** (YAGNI — no caller yet).

### Key decisions
- **New `frame.paths` namespace** rather than inlining in `source.lg`: the `FRAME_HOME`/XDG base logic is exactly what the future aliases-config reuses, so it is the natural shared boundary. `source.lg` keeps owning the source-specific `/templates/<host>/<owner>/<repo>/<sha>` suffix.
- **Pure resolver `app-dir` is public and takes explicit env values** (no `os/getenv` inside), so it is unit-testable with no process-env mutation — matching how `parse-git-url` and `parse-semver` are pure-and-tested. The thin `cache-dir` wrapper reads `os/getenv` and delegates; it is trivial wiring and left untested, exactly as the old `frame-home` env wrapper was.
- **Don't ship `config-dir` yet** (YAGNI) — document the scheme instead.
- **XDG on all platforms**, including macOS (no `~/Library` special-casing). Simpler; `FRAME_HOME` / `XDG_CACHE_HOME` remain the escape hatch.
- **No migration / auto-cleanup** of old `~/.frame`. It gets orphaned and re-clones on next use; documented rather than handled by risky cleanup code.

### `app-dir` contract (tasks must agree on this exactly)
```clojure
(defn app-dir
  "Resolve frame's base directory for one XDG category.
   frame-home, when non-blank, overrides everything (used as-is, no suffix).
   Otherwise use xdg-home when it is non-blank AND absolute (starts with \"/\");
   a blank or relative xdg-home is ignored per the XDG spec and falls back to
   <home>/<fallback>. Then append \"/frame\"."
  [{:keys [frame-home xdg-home home fallback]}]
  ...)
;; Examples (cache category, fallback \".cache\"):
;;   {:frame-home \"/tmp/fh\"        ...}                       => \"/tmp/fh\"
;;   {:frame-home \"\"  :xdg-home \"/x/cache\" ...}             => \"/x/cache/frame\"
;;   {:frame-home \"\"  :xdg-home \"\" :home \"/h\" :fallback \".cache\"} => \"/h/.cache/frame\"
;;   {:frame-home \"\"  :xdg-home \"rel/dir\" :home \"/h\" :fallback \".cache\"} => \"/h/.cache/frame\"  (relative xdg ignored)
;;   {:frame-home nil :xdg-home nil :home \"/h\" :fallback \".cache\"} => \"/h/.cache/frame\"  (nil == unset)
```
`cache-dir` wires it: `(app-dir {:frame-home (os/getenv "FRAME_HOME") :xdg-home (os/getenv "XDG_CACHE_HOME") :home (os/getenv "HOME") :fallback ".cache"})`.

`source/template-dir` then builds `(str (paths/cache-dir) "/templates/" host "/" owner "/" repo "/" sha)`.

### Testing strategy
- `test/frame/paths_test.lg` — unit-test `app-dir` for all three branches (override set; xdg set; both blank → home fallback). Pure, no env mutation.
- Existing `test/frame/source_test.lg` is unaffected: its tests exercise `git-url?`, `parse-git-url`, and local-path resolution only — none touch the cache path.
- Full `lgx check` (fmt + lint + test) must stay green.
- One manual network verification that a real clone lands under `~/.cache/frame/templates` (default) and under `$FRAME_HOME/templates` when the override is set.

## File Structure

- Create: `src/frame/paths.lg` — namespace `frame.paths`. Public pure `app-dir`; thin `cache-dir` reading env. Comment documenting the future `config-dir` scheme.
- Create: `test/frame/paths_test.lg` — namespace `frame.paths-test`. Unit tests for `app-dir`.
- Modify: `src/frame/source.lg` — require `frame.paths`; `template-dir` uses `(paths/cache-dir)`; delete the private `frame-home`; refresh the header comment (path + the stale "shallow-cloned" wording).
- Modify: `README.md` — rewrite the caching bullet (~line 207) and the inline example comment (~line 53) for the XDG cache dir and the `FRAME_HOME` / `XDG_CACHE_HOME` overrides.

---

## Task 1: `frame.paths` namespace

**Files:**
- Create: `src/frame/paths.lg`
- Test: `test/frame/paths_test.lg`

- [x] **Step 1: Write the failing test**
  Create `test/frame/paths_test.lg` (namespace `frame.paths-test`, requiring `clojure.test` and `frame.paths`, following the style of `test/frame/source_test.lg`). Test `app-dir` with these cases, asserting the exact strings from the contract in the Design:
  - override present: `(app-dir {:frame-home "/tmp/fh" :xdg-home "/x/cache" :home "/h" :fallback ".cache"})` => `"/tmp/fh"` (override wins even when xdg is set).
  - xdg present + absolute, override blank: `(app-dir {:frame-home "" :xdg-home "/x/cache" :home "/h" :fallback ".cache"})` => `"/x/cache/frame"`.
  - both blank: `(app-dir {:frame-home "" :xdg-home "" :home "/h" :fallback ".cache"})` => `"/h/.cache/frame"`.
  - **relative xdg ignored** (XDG spec): `(app-dir {:frame-home "" :xdg-home "rel/dir" :home "/h" :fallback ".cache"})` => `"/h/.cache/frame"`.
  - **nil env values** (`os/getenv` returns nil when unset): `(app-dir {:frame-home nil :xdg-home nil :home "/h" :fallback ".cache"})` => `"/h/.cache/frame"`.

- [x] **Step 2: Run test to verify it fails**
  Run: `lgx test test/frame/paths_test.lg`
  Expected: FAIL (namespace `frame.paths` not found).

- [x] **Step 3: Write minimal implementation**
  Create `src/frame/paths.lg` (namespace `frame.paths`, requiring `[string :as str]`). Implement public `app-dir` per the contract in the Design: when `frame-home` is non-blank (use `str/blank?`) return it as-is; otherwise compute the base as `xdg-home` when non-blank else `(str home "/" fallback)`, and return `(str base "/frame")`. Add public `cache-dir` wiring `os/getenv` for `FRAME_HOME` / `XDG_CACHE_HOME` / `HOME` with fallback `".cache"`. Add a header comment describing the cache-vs-config split and the future `config-dir` scheme (from the Design table) so aliases can be added symmetrically — but do NOT add `config-dir`.

- [x] **Step 4: Run test to verify it passes**
  Run: `lgx test test/frame/paths_test.lg`
  Expected: PASS.

- [x] **Step 5: Commit**
  `git commit -m "feat: add frame.paths for XDG-aware base dir resolution"`

## Task 2: Wire the template cache to `frame.paths`

**Files:**
- Modify: `src/frame/source.lg`
- Test: `test/frame/source_test.lg` (existing; should stay green with no edits)

> Deviation (Step 2): `lgx test` accepts only one file, so the two files were run as separate `lgx test <file>` invocations instead of one combined command.
> Verified (Step 4): all three env cases confirmed with tiny-tui — clone landed under `$HOME/.cache/frame/templates/...` (default), `$FRAME_HOME/templates/...` (override), and `$XDG_CACHE_HOME/frame/templates/...` (custom XDG). The trailing `frame.edn not found` is expected (tiny-tui is not a frame template); the clone/cache path is what this step checks.

- [x] **Step 1: Update `source.lg`**
  In `src/frame/source.lg`:
  - Add `[frame.paths :as paths]` to the `:require`.
  - Delete the private `frame-home` defn.
  - Change `template-dir` to build `(str (paths/cache-dir) "/templates/" host "/" owner "/" repo "/" sha)`.
  - Refresh the header comment (lines ~4-13): replace the `$FRAME_HOME/templates/... (FRAME_HOME defaults to ~/.frame)` description with the XDG cache location (`$XDG_CACHE_HOME/frame/templates/...`, default `~/.cache/frame`, `FRAME_HOME` overrides the whole base), and drop the inaccurate "shallow-cloned" wording (the clone is a full `git clone`, not `--depth 1`).

- [x] **Step 2: Run the source + paths tests**
  Run: `lgx test test/frame/source_test.lg test/frame/paths_test.lg`
  Expected: PASS (existing source tests are pure/local-path and unaffected).

- [x] **Step 3: Run full checks**
  Run: `lgx check`
  Expected: green (fmt + lint + all tests).

- [x] **Step 4: Manual verification (needs network)**
  Build and clone a small template into a temp cache, confirming the path layout:
  - Default XDG: `HOME=$(mktemp -d) env -u FRAME_HOME -u XDG_CACHE_HOME bin/frame new --defaults --dir "$(mktemp -d)/out" https://github.com/abogoyavlensky/tiny-tui app`, then confirm `~/.cache/frame/templates/github.com/abogoyavlensky/tiny-tui/<sha>/` exists under that temp `HOME` (i.e. `$HOME/.cache/frame/templates/...`).
  - Override: set `FRAME_HOME=$(mktemp -d)` and repeat; confirm the clone lands under `$FRAME_HOME/templates/github.com/abogoyavlensky/tiny-tui/<sha>/`.
  - Custom absolute XDG: set `XDG_CACHE_HOME=$(mktemp -d)` (FRAME_HOME unset) and repeat; confirm the clone lands under `$XDG_CACHE_HOME/frame/templates/...`.
  Run `lgx build` first if `bin/frame` is stale. (Manual — skip if no network; note it in the task summary.)

- [x] **Step 5: Commit**
  `git commit -m "feat: cache templates under XDG cache dir via frame.paths"`

## Task 3: Documentation

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the caching docs**
  In `README.md`:
  - The "Sources and caching" bullet (~line 207): replace "caches it under `~/.frame/templates/...`, reusing the cache on later runs. Set `FRAME_HOME` to override `~/.frame`." with wording for the XDG cache dir: caches under `~/.cache/frame/templates/<host>/<owner>/<repo>/<sha>/` (honoring `XDG_CACHE_HOME`), reused on later runs; `FRAME_HOME` overrides the whole base (templates then live under `$FRAME_HOME/templates/...`). Mention the cache is safe to delete (it re-clones on next use).
  - The inline example comment (~line 53): change `# From a git URL (cloned and cached under ~/.frame)` to `# From a git URL (cloned and cached under ~/.cache/frame)`.
  Use /writing-clearly for the prose.

- [ ] **Step 2: Commit**
  `git commit -m "docs: document XDG cache dir for downloaded templates"`

---

## Notes
- DRY/YAGNI: `config-dir` and the aliases feature are intentionally out of scope — only the cache moves. The `frame.paths` seam is what makes the later addition small.
- No migration code: old `~/.frame` caches are orphaned and harmless; a re-clone repopulates the new location on demand.
