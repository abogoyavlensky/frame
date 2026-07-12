# Migrate clojure-stack-lite to Frame Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a frame template to `clojure-stack-lite` alongside its existing deps-new template (keeping deps-new as the fallback), verified by frame and deps-new generating **byte-identical** projects across a matrix of option combinations.

**Tech Stack:** frame (let-go/lgx), deps-new (Clojure/tools.build), bash for verification.

---

## Design

### Context

`clojure-stack-lite` (`/Users/andrew/Projects/clojure-stack-lite`, branch `migrate-to-frame`, already checked out) is a **deps-new** template. Its logic is Clojure code in `src/io/github/abogoyavlensky/clojure_stack_lite.clj`:

- **`data-fn`** slurps content chunks from `resources/…/clojure_stack_lite/substitutions/*` and binds them to `{{tag}}` names (each defaulting to `""`), based on the chosen options.
- **`template-fn`** dynamically builds the deps-new `:transform` list — adding/renaming/skipping source directories per option.
- deps-new then copies files, performing **literal `str/replace`** of `{{key}}` tokens (via `tools.build copy-dir :replace`). It is **not** Selmer: `${{ secrets.X }}` survives untouched only because it contains no matching `{{key}}` token.

Four user options (from the project README "Options" section):

| Option | Values | Default |
|---|---|---|
| `db` | `sqlite`, `postgres` | `sqlite` |
| `auth` | boolean | `false` |
| `daisyui` | boolean | `false` |
| `deploy` | `kamal`, `none` | `kamal` |

Plus deps-new built-ins used in output: `developer` (= capitalized OS `$USER`) and `now/year` (current year).

Frame has **no code** — logic is declarative: `frame.edn` vars, inline `{% if %}`/`{% case %}` tags in file contents, and conditional path segments (a segment that renders empty skips the file/subtree). **This migration requires no changes to frame itself** — every deps-new construct maps onto existing frame features (`now-date`/`now-year` built-ins already exist from the prior `clojure-lib` migration).

Two things frame does that force structural choices vs deps-new:
1. Frame **parses** `{{ }}`, so any file containing GitHub-Actions `${{ }}` must be made raw (a `:raw` glob for whole-verbatim files, or `{% raw %}` blocks for mixed files). deps-new left them alone for free.
2. `alpinejs.min.js` contains a `{{`-like sequence — so raw-globbing the JS assets is **mandatory**, not just defensive.

### Tag mapping

| deps-new tag | frame equivalent |
|---|---|
| `{{main/ns}}` (70×) | `{{ project-name }}` |
| `{{main/file}}` (6×) | `{{ project-name \| snake_case }}` |
| `{{developer}}` (2×) | `{{ developer }}` (var) |
| `{{now/year}}` (1×) | `{{ now-year }}` (built-in) |
| substitution tags (13, below) | inline `{% case %}` / `{% if %}` blocks |
| `${{ … }}` (GitHub Actions) | `:raw` glob or `{% raw %}` block |

For a valid frame name (`^[a-z][a-z0-9-]*$`), `main/ns` = the name verbatim (`= {{ project-name }}`) and `main/file` = name with `-`→`_` (`= {{ project-name | snake_case }}`).

### Substitution mapping (deps-new `{{tag}}` → frame inline block)

Each token is replaced **in place** by an inline conditional holding the literal chunk from `substitutions/`. Source chunks are the source of truth — copy their exact bytes (mind leading/trailing newlines; the verification diff is the arbiter). Tags kept inline (not alone on their line) so frame's whitespace rule preserves surrounding bytes exactly.

| deps-new tag | File | frame inline block (condition → chunk) |
|---|---|---|
| `{{auth-deps}}` | `deps.edn` | `{% if auth %}` → `deps_edn_auth_deps.edn` |
| `{{db-driver-deps}}` | `deps.edn` | `{% case db %}` sqlite→`deps_edn_db_driver_deps_sqlite.edn`, postgres→`…_postgres.edn` |
| `{{db-test-deps}}` | `deps.edn` | `{% if db == postgres %}` → `deps_edn_db_test_deps_postgres.edn` |
| `{{clj-repl-cmd}}` | `bb.edn` | `{% case db %}` sqlite→`bb_edn_clj_repl_cmd_sqlite.edn`, postgres→`…_postgres.edn` |
| `{{fetch-assets-urls}}` | `bb.edn` | `{% if daisyui %}` → `bb_edn_daisyui.edn` |
| `{{bb-deploy-kamal}}` | `bb.edn` | `{% if deploy == kamal %}` → `bb_deploy_kamal.edn` |
| `{{db-config}}` | `resources/config.edn` | `{% case db %}` sqlite→`resources_config_edn_sqlite.edn`, postgres→`…_postgres.edn` |
| `{{sql-result-set-config}}` | `src/…/db.clj` | `{% if db == postgres %}` → `src_db_sql_result_set_config_postgres.edn` |
| `{{test-utils-db-setup}}` | `test/…/test_utils.clj` | `{% case db %}` sqlite→`test_utils_db_setup_sqlite.clj`, postgres→`…_postgres.clj` |
| `{{ci-deploy-env-vars}}` | `.github/workflows/deploy.yaml` | `{% if db == postgres %}` → `github_workflows_deploy_ci_deploy_env_vars_postgres.txt` |
| `{{deploy-config-kamal}}` | `.kamal/deploy.yml` | `{% case db %}` sqlite→`kamal-deploy-config-sqlite.txt`, postgres→`…-postgres.txt` |
| `{{deploy-secrets-kamal}}` | `.kamal/secrets` | `{% if db == postgres %}` → `kamal-deploy-secrets-postgres.txt` |
| `{{readme-deploy-kamal}}` | `README.md` | `{% if deploy == kamal %}` → `readme_deploy_kamal.md` |

Note: `resources_config_edn_sqlite.edn` and `kamal-deploy-config-sqlite.txt` themselves contain `{{main/ns}}`/`{{main/file}}` — rewrite those nested tags to frame form too; frame renders in a single pass so they resolve correctly (deps-new resolves them via a second replace pass — verified: output is `jdbc:sqlite:db/<name>.sqlite`).

### Compact template syntax in names

Per the approval: **file/dir names use compact tags — no padding spaces** (e.g. `{%if auth%}`, `{{project-name|snake_case}}`, `{%if db==sqlite%}`). Confirmed against frame's lexer (trims inner) and parser (`bare-end` stops at `=`/`!`/ws/`"`; filters split on `|` then trim). Word-operators still need their boundary spaces: `{%if not auth%}`, `{%if db==postgres and auth%}`. File **contents** use normal readable spacing (spaces inside tags don't affect output).

### `frame.edn` (exact content)

```edn
{:description "A lightweight, modern template to jumpstart your Clojure project"
 :vars [{:key :db
         :prompt "Database"
         :type :enum
         :options ["sqlite" "postgres"]
         :default "sqlite"}
        {:key :auth
         :prompt "Include authentication and registration flow?"
         :type :boolean
         :default false}
        {:key :daisyui
         :prompt "Include DaisyUI component library?"
         :type :boolean
         :default false}
        {:key :deploy
         :prompt "Deployment"
         :type :enum
         :options ["kamal" "none"]
         :default "kamal"}
        {:key :developer
         :prompt "Developer / GitHub username (LICENSE + Docker image)"
         :type :string
         :validate {:pattern "^.+$" :msg "Developer cannot be empty"}}]
 :raw ["resources/public/js/**"
       "resources/public/images/**"
       ".github/actions/**"
       ".github/workflows/checks.yaml"]}
```

- No `:computed`. `developer` is a direct var; `now-year` is a built-in.
- `developer` has **no default** (deps-new derives it from `$USER`, so there is no universal default). Consequence: `--defaults` alone errors on its non-empty validation; callers must add `--var developer=…`. Correct for a public template and required for byte-parity.
- `:raw` globs match the **source** path (`generate.lg:58`), so they must be written against source names — `resources/public/js/**` covers the conditional-named daisyui JS; a `**/*.js` glob would NOT (source name ends in `{%endif%}`).

### `template/` structure

`template/` mirrors the generated project. Verbatim files (no `{{`/`{%`, verified by grep) copy unchanged. Conditional path names use compact syntax.

```
template/
├── .clj-kondo/config.edn                 verbatim
├── .cljfmt.edn .dockerignore .gitignore .mise.toml   verbatim
├── .github/
│   ├── actions/cache-clojure-deps/action.yaml        raw (:raw glob)
│   └── workflows/
│       ├── checks.yaml                               raw (:raw glob)
│       └── {%if deploy==kamal%}deploy.yaml{%endif%}  {% raw %} blocks + {% if db==postgres %}
├── {%if deploy==kamal%}.kamal{%endif%}/
│   ├── deploy.yml                        {{project-name}} + {% case db %} deploy-config
│   └── secrets                           {% if db==postgres %} deploy-secrets
├── {%if db==postgres%}docker-compose.yaml{%endif%}   verbatim (from docker-compose-postgres/)
├── Dockerfile                            {{developer}} {{project-name}}
├── LICENSE                               {{now-year}} {{developer}}
├── README.md                             {{project-name}} + {% if deploy==kamal %} readme-deploy
├── bb.edn                                {% case db %} + {% if daisyui %} + {% if deploy==kamal %}
├── deps.edn                              {% if auth %} + {% case db %} + {% if db==postgres %}
├── dev/user.clj                          verbatim
├── resources/
│   ├── config.dev.edn                    verbatim
│   ├── config.edn                        {{project-name}} + {% case db %} db-config
│   ├── logback.xml                       verbatim
│   ├── migrations/
│   │   ├── {%if db==sqlite%}0001.up.sql{%endif%}             WAL
│   │   ├── {%if db==sqlite and auth%}0002.up.sql{%endif%}    sqlite user table
│   │   ├── {%if db==postgres and auth%}0001.up.sql{%endif%}  postgres user table
│   │   └── {%if db==postgres%}.gitkeep{%endif%}
│   └── public/
│       ├── css/input.css                 {% if daisyui %} plugins block
│       ├── images/**                     raw (icons)
│       ├── js/htmx.min.js js/alpinejs.min.js                raw
│       ├── js/{%if daisyui%}daisyui.js{%endif%}             raw
│       ├── js/{%if daisyui%}daisyui-theme.js{%endif%}       raw
│       └── manifest.json                 {{project-name}}
├── {%if db==sqlite%}db{%endif%}/.gitkeep                     (from db_sqlite/)
├── src/{{project-name|snake_case}}/
│   ├── core.clj  server.clj              {{project-name}}
│   ├── db.clj                            {{project-name}} + {% if db==postgres %} sql-result-set
│   ├── {%if not auth%}handlers.clj{%endif%}   base handlers (src/handlers.clj)
│   ├── {%if auth%}handlers.clj{%endif%}       auth handlers (src_auth/handlers.clj)
│   ├── {%if not auth%}routes.clj{%endif%}     base routes
│   ├── {%if auth%}routes.clj{%endif%}         auth routes
│   ├── {%if not auth%}views.clj{%endif%}      base views
│   ├── {%if auth%}views.clj{%endif%}          auth views
│   └── {%if auth%}auth{%endif%}/
│       ├── handlers.clj queries.clj spec.clj views.clj       {{project-name}}
└── test/{{project-name|snake_case}}/
    ├── home_test.clj                     {{project-name}}
    ├── test_utils.clj                    {{project-name}} + {% case db %} db-setup
    └── {%if auth%}auth_*_test.clj{%endif%}   6 files, {{project-name}}
```

Two files render to `resources/migrations/0001.up.sql` (sqlite-WAL and postgres-auth) but under mutually-exclusive conditions, so at most one survives — no collision. Base vs auth `handlers/routes/views` are near-totally-different files (deps-new ships them as separate `src/` vs `src_auth/` files), so paired conditional-named siblings are used rather than one file with a large inline `{% if %}/{% else %}`.

### Source → template file provenance

deps-new source dirs live under `resources/io/github/abogoyavlensky/clojure_stack_lite/` (abbreviated `R/` below):

- `R/root/**` → `template/**` (the always-present base; strip untracked `.clj-kondo/.cache/**` if present)
- `R/src/*` → `template/src/{{project-name|snake_case}}/*` (base `core/db/handlers/routes/server/views`)
- `R/src_auth/*` → auth versions of `handlers/routes/views` + the `auth/` subdir
- `R/test/*` → `template/test/{{project-name|snake_case}}/*` (base `home_test`, `test_utils`)
- `R/test_auth/*` → the 6 conditional `auth_*_test.clj`
- `R/db_sqlite/.gitkeep` → `template/{%if db==sqlite%}db{%endif%}/.gitkeep`
- `R/docker-compose-postgres/docker-compose.yaml` → `template/{%if db==postgres%}docker-compose.yaml{%endif%}`
- `R/resources_migrations_*/**` → the 4 conditional migration files
- `R/resources_public_css_default/input.css` + `R/resources_public_css_daisyui/input.css` → `template/resources/public/css/input.css` (daisyui delta as inline `{% if daisyui %}`)
- `R/resources_public_js_daisyui/*` → conditional daisyui JS
- `R/github_workflows_deploy_yaml_kamal/deploy.yaml` → conditional `deploy.yaml`
- `R/kamal/*` → `template/{%if deploy==kamal%}.kamal{%endif%}/*`
- `R/substitutions/*` → inline chunk content (never copied as files)

### `deploy.yaml` raw-block structure

The one mixed file: mostly literal (incl. `${{ secrets.* }}`, `${{ github.sha }}`, `${{ cancelled() }}`) with a single live `{% if db == postgres %}` block for the CI env vars. Wrap the literal spans in `{% raw %}…{% endraw %}` and break out only for the conditional. **The postgres branch's content itself contains `${{ secrets.* }}`, so it must ALSO be wrapped in a nested `{% raw %}` block** — frame renders every output tag in a *taken* branch, so an un-raw'd `${{ secrets.DATABASE_URL }}` would be parsed as a frame output tag and error on the unresolved var `secrets.DATABASE_URL`. Shape (exact bytes verified by diff):

```
{% raw %}name: Deploy
… up to and including:
          SESSION_SECRET_KEY: ${{ secrets.SESSION_SECRET_KEY }}{% endraw %}{% if db == postgres %}{% raw %}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          POSTGRES_DB: ${{ secrets.POSTGRES_DB }}
          POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
          POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}{% endraw %}{% endif %}{% raw %}
        run: bb kamal deploy --version=${{ github.sha }}
… through end of file …{% endraw %}
```

The env-vars chunk (wrapped in the nested raw block) is `github_workflows_deploy_ci_deploy_env_vars_postgres.txt`. General rule: **any `{% if %}`/`{% case %}` branch whose chunk contains foreign `${{ }}`/`{{ }}` tags must wrap that chunk in `{% raw %}`.** Audited: only this deploy.yaml postgres branch needs it. (Other kamal/db chunks either contain no braces — `kamal-deploy-secrets-postgres.txt` uses `$VAR`, not `${{ }}` — or contain genuine frame tags `{{main/ns}}`/`{{main/file}}` that must render, not be raw'd.)

### Verification (the real acceptance gate)

Acceptance: deps-new and frame produce **byte-identical** trees for the same inputs, across the matrix below.

Ground truth is a **fresh** deps-new generation — the committed `tmpl-prj/` example is stale (old action pins, missing `cookie-attrs-secure?`) and must not be used for comparison. Determinism:
- deps-new derives `developer` from `$USER`; run it with a fixed `USER` env so the value is stable, and pass frame the matching `--var developer=…`.
- Both run the same day → `now-year` matches.
- Before diffing, remove any untracked `.clj-kondo/.cache/**` under `R/root/` (git-ignored; deps-new would copy it locally but it is not part of the distributed template, and `template/` must not contain it either).

Matrix (name, db, auth, daisyui, deploy) — covers every branch and both name shapes (hyphenated exercises `snake_case`):

| # | name | db | auth | daisyui | deploy |
|---|---|---|---|---|---|
| 1 | my-app | sqlite | false | false | kamal |
| 2 | my-app | postgres | true | true | kamal |
| 3 | myapp | sqlite | true | false | none |
| 4 | my-app | postgres | false | true | none |
| 5 | my-app | sqlite | true | true | kamal |
| 6 | my-app | postgres | false | false | kamal |
| 7 | my-app | sqlite | false | true | none |
| 8 | myapp | postgres | true | false | kamal |

Per-combo commands (from a scratch dir; `CSL=/Users/andrew/Projects/clojure-stack-lite`, `FRAME=/Users/andrew/Projects/frame`):

```bash
# deps-new (run inside $CSL; :local-new alias pins the local template)
USER=frameuser clojure -X:local-new :name my-app \
  :db postgres :auth true :daisyui true :deploy kamal \
  :target-dir '"/abs/out/deps-new-out"'

# frame (after lgx build in $FRAME)
"$FRAME/bin/frame" new --defaults \
  --var db=postgres,auth=true,daisyui=true,deploy=kamal,developer=Frameuser \
  --dir /abs/out/frame-out "$CSL" my-app

diff -r /abs/out/deps-new-out /abs/out/frame-out   # must print nothing
```

(`$USER=frameuser` → deps-new `developer` = `Frameuser`, matching `--var developer=Frameuser`.) For the defaults combo (#1) frame can run `--defaults --var developer=Frameuser` with no other vars. deps-new options are passed as `:key value` exec-args; omit an option to take its default. The diff loop drives fixes until every combo is clean.

### Testing / fallback integrity

- The deps-new files are untouched (except deleting untracked `.cache` junk), so `bb test` in `$CSL` must still pass — proving the fallback is intact.
- No frame unit-test changes: the migration exercises only existing frame features. The byte-diff matrix is the test.

## File Structure

**clojure-stack-lite repo (branch `migrate-to-frame`) — all new except the README note:**

- Create: `frame.edn`
- Create: `template/**` (per the structure above)
- Modify: `README.md` (add a "Create a project with frame" subsection)

**frame repo (branch `master`):**

- Create: `docs/plans/2026-07-12-migrate-clojure-stack-lite-template.md` (this plan)
- No source changes.

All paths below are relative to `/Users/andrew/Projects/clojure-stack-lite` unless noted. `R/` = `resources/io/github/abogoyavlensky/clojure_stack_lite/`.

## Tasks

### Task 1: Scaffold — frame.edn, verbatim files, raw assets

**Files:**
- Create: `frame.edn`
- Create: `template/**` (verbatim + raw files only in this task)

- [ ] **Step 1: Confirm branch**
  Run: `git -C /Users/andrew/Projects/clojure-stack-lite branch --show-current`
  Expected: `migrate-to-frame`.

- [ ] **Step 2: Write `frame.edn`**
  Exact content from the Design section.

- [ ] **Step 3: Copy `R/root/` into `template/`**
  Run: `cp -a resources/io/github/abogoyavlensky/clojure_stack_lite/root/. template/` (from repo root).
  Remove untracked cache if present: `rm -rf template/.clj-kondo/.cache`.
  Verify dotfiles arrived: `ls -A template` shows `.clj-kondo .cljfmt.edn .dockerignore .github .gitignore .mise.toml Dockerfile LICENSE README.md bb.edn deps.edn dev resources`.

- [ ] **Step 4: Confirm the verbatim files are tag-free**
  Run: `grep -rl '{{\|{%' template` — the only hits should be files this plan templates later (`Dockerfile LICENSE README.md bb.edn deps.edn resources/config.edn resources/public/manifest.json`) plus raw assets (`alpinejs.min.js`, `.github/**`). `dev/user.clj resources/config.dev.edn resources/logback.xml .clj-kondo/config.edn .cljfmt.edn .gitignore .dockerignore .mise.toml` must have **no** hits.

- [ ] **Step 5: Commit**
  In `$CSL`: `git add -A && git commit -m "feat: scaffold frame template (frame.edn + verbatim root files)"`

### Task 2: `src/` namespace files (base + auth)

**Files (under `template/src/{{project-name|snake_case}}/`):**
- Create: `core.clj`, `server.clj` — from `R/src/`, rewrite `{{main/ns}}`→`{{ project-name }}`
- Create: `db.clj` — from `R/src/db.clj`, rewrite ns + inline `{{sql-result-set-config}}`
- Create: `{%if not auth%}handlers.clj{%endif%}`, `{%if not auth%}routes.clj{%endif%}`, `{%if not auth%}views.clj{%endif%}` — from `R/src/`
- Create: `{%if auth%}handlers.clj{%endif%}`, `{%if auth%}routes.clj{%endif%}`, `{%if auth%}views.clj{%endif%}` — from `R/src_auth/`
- Create: `{%if auth%}auth{%endif%}/handlers.clj`, `queries.clj`, `spec.clj`, `views.clj` — from `R/src_auth/auth/`

- [ ] **Step 1: Create the templated `src` dir**
  `mkdir -p` the `src/{{project-name|snake_case}}` and `auth` subdirs (literal names with the compact tags).

- [ ] **Step 2: Copy base + auth files and rewrite `{{main/ns}}`→`{{ project-name }}`**
  Every `.clj` here uses `{{main/ns}}` only; replace all occurrences. `views.clj` (base + auth) also contains `src/{{main/file}}/…` path strings inside home-page prose → rewrite `{{main/file}}`→`{{ project-name | snake_case }}`.

- [ ] **Step 3: Inline `{{sql-result-set-config}}` in `db.clj`**
  Replace the token (in `:builder-fn jdbc-rs/as-unqualified-kebab-maps{{sql-result-set-config}}`) with an inline `{% if db == postgres %}<chunk>{% endif %}` where `<chunk>` is the exact bytes of `R/substitutions/src_db_sql_result_set_config_postgres.edn` (`\n   :return-keys true`). Keep the tag inline (immediately after `-maps`).

- [ ] **Step 4: Verify no stray deps-new tags remain**
  Run: `grep -rn '{{' template/src | grep -vE '\{\{ (project-name)( \| snake_case)? \}\}'`
  Expected: no output.

- [ ] **Step 5: Commit**
  `git commit -m "feat: frame template src namespaces (base + auth)"`

### Task 3: `test/` namespace files (base + auth)

**Files (under `template/test/{{project-name|snake_case}}/`):**
- Create: `home_test.clj` — from `R/test/home_test.clj`, rewrite ns
- Create: `test_utils.clj` — from `R/test/test_utils.clj`, rewrite ns + inline `{{test-utils-db-setup}}`
- Create: `{%if auth%}auth_account_test.clj{%endif%}` and the other 5 `auth_*_test.clj` — from `R/test_auth/`, rewrite ns

- [ ] **Step 1: Copy base tests, rewrite `{{main/ns}}`→`{{ project-name }}`**

- [ ] **Step 2: Inline `{{test-utils-db-setup}}` in `test_utils.clj`**
  Replace the standalone-line token with an inline `{% case db %}{% when sqlite %}<sqlite-chunk>{% when postgres %}<postgres-chunk>{% endcase %}`, where the chunks are the exact bytes of `R/substitutions/test_utils_db_setup_sqlite.clj` and `…_postgres.clj`. Place `{% case db %}{% when sqlite %}` immediately before the first `(defn with-truncated-tables` so leading whitespace/blank lines match; keep the tags inline with the content (never alone on a line).

- [ ] **Step 3: Copy the 6 auth tests under conditional names, rewrite ns**
  Each literal filename is `{%if auth%}<name>_test.clj{%endif%}`. All use `{{main/ns}}` only.

- [ ] **Step 4: Verify tags**
  Run: `grep -rn '{{' template/test | grep -vE '\{\{ (project-name)( \| snake_case)? \}\}'`
  Expected: no output.

- [ ] **Step 5: Commit**
  `git commit -m "feat: frame template test namespaces (base + auth)"`

### Task 4: `deps.edn` and `bb.edn` (multi-dimension)

**Files:**
- Modify: `template/deps.edn`, `template/bb.edn`

- [ ] **Step 1: `deps.edn` — three inline substitutions**
  - `{{auth-deps}}` (inline after `manifest-edn {:mvn/version "0.1.1"}`) → `{% if auth %}<deps_edn_auth_deps.edn>{% endif %}`
  - `{{db-driver-deps}}` (its own indented line) → `{% case db %}{% when sqlite %}<sqlite-driver>{% when postgres %}<postgres-driver>{% endcase %}` placed after the line's leading whitespace, tags inline with content
  - `{{db-test-deps}}` (inline after `hickory {:mvn/version "0.7.7"}`) → `{% if db == postgres %}<deps_edn_db_test_deps_postgres.edn>{% endif %}`
  - Rewrite `{{main/ns}}`→`{{ project-name }}` (the `:build` alias `:main-ns {{main/ns}}.core`).

- [ ] **Step 2: `bb.edn` — three inline substitutions**
  - `clj-repl {{clj-repl-cmd}}` → `clj-repl {% case db %}{% when sqlite %}<sqlite-cmd>{% when postgres %}<postgres-cmd>{% endcase %}`
  - `{{fetch-assets-urls}}` (inline in the `fetch-assets` vector, after the alpinejs entry) → `{% if daisyui %}<bb_edn_daisyui.edn>{% endif %}`
  - `{{bb-deploy-kamal}}` (inline after the `build` task's closing `}`) → `{% if deploy == kamal %}<bb_deploy_kamal.edn>{% endif %}`

- [ ] **Step 3: Verify tags**
  Run: `grep -n '{{' template/deps.edn template/bb.edn | grep -vE '\{\{ project-name \}\}'`
  Expected: no output.

- [ ] **Step 4: Commit**
  `git commit -m "feat: frame template deps.edn and bb.edn conditionals"`

### Task 5: DB-conditional resources

**Files:**
- Modify: `template/resources/config.edn`
- Create: 4 migration files under `template/resources/migrations/`
- Create: `template/{%if db==sqlite%}db{%endif%}/.gitkeep`
- Create: `template/{%if db==postgres%}docker-compose.yaml{%endif%}`

- [ ] **Step 1: `config.edn`**
  Rewrite `{{main/ns}}`→`{{ project-name }}` (two keys). Replace `#profile {{db-config}}` with `#profile {% case db %}{% when sqlite %}<sqlite-config>{% when postgres %}<postgres-config>{% endcase %}`, chunks from `R/substitutions/resources_config_edn_{sqlite,postgres}.edn`. The sqlite chunk contains `{{main/ns}}` → rewrite to `{{ project-name }}` inside it.

- [ ] **Step 2: Migration files (conditional names, verbatim content)**
  - `{%if db==sqlite%}0001.up.sql{%endif%}` ← `R/resources_migrations_sqlite/0001.up.sql` (WAL)
  - `{%if db==sqlite and auth%}0002.up.sql{%endif%}` ← `R/resources_migrations_sqlite_auth/0002.up.sql`
  - `{%if db==postgres and auth%}0001.up.sql{%endif%}` ← `R/resources_migrations_postgres_auth/0001.up.sql`
  - `{%if db==postgres%}.gitkeep{%endif%}` ← `R/resources_migrations_postgres/.gitkeep` (empty)

- [ ] **Step 3: `db/.gitkeep` and `docker-compose.yaml`**
  `template/{%if db==sqlite%}db{%endif%}/.gitkeep` (empty, from `R/db_sqlite/.gitkeep`).
  `template/{%if db==postgres%}docker-compose.yaml{%endif%}` verbatim from `R/docker-compose-postgres/docker-compose.yaml`.

- [ ] **Step 4: Commit**
  `git commit -m "feat: frame template db-conditional resources and migrations"`

### Task 6: Deploy files + remaining single-tag content

**Files:**
- Create: `template/{%if deploy==kamal%}.kamal{%endif%}/deploy.yml`, `…/secrets`
- Create: `template/.github/workflows/{%if deploy==kamal%}deploy.yaml{%endif%}`
- Modify: `template/README.md`, `template/Dockerfile`, `template/LICENSE`, `template/resources/public/manifest.json`

- [ ] **Step 1: `.kamal/deploy.yml` + `secrets`**
  `deploy.yml`: rewrite `{{main/ns}}`→`{{ project-name }}`; replace `{{deploy-config-kamal}}` (own line) with inline `{% case db %}{% when sqlite %}<kamal-sqlite>{% when postgres %}<kamal-postgres>{% endcase %}`. The sqlite chunk `kamal-deploy-config-sqlite.txt` contains `{{main/ns}}` and the postgres chunk `kamal-deploy-config-postgres.txt` contains `{{main/file}}` → rewrite those nested tags. `secrets`: replace `{{deploy-secrets-kamal}}` (own line) with `{% if db == postgres %}<kamal-deploy-secrets-postgres.txt>{% endif %}`.

- [ ] **Step 2: `deploy.yaml` with `{% raw %}` blocks**
  Build from `R/github_workflows_deploy_yaml_kamal/deploy.yaml` using the raw-block structure in the Design section: literal spans wrapped in `{% raw %}…{% endraw %}`, the CI env-vars as a live `{% if db == postgres %}{% raw %}…{% endraw %}{% endif %}` block (chunk `github_workflows_deploy_ci_deploy_env_vars_postgres.txt`, itself nested in raw because it contains `${{ secrets.* }}`). The filename is conditional on `deploy==kamal`. Do a focused check right after writing it (before the full matrix): generate combo #6 (postgres, kamal) with frame and confirm it does not error and `deploy.yaml` matches deps-new — this isolates raw-block mistakes.

- [ ] **Step 3: `README.md`, `Dockerfile`, `LICENSE`, `manifest.json`**
  - `README.md`: `{{main/ns}}`→`{{ project-name }}`; `{{readme-deploy-kamal}}` (inline after "…in `resources/public` folder.") → `{% if deploy == kamal %}<readme_deploy_kamal.md>{% endif %}`.
  - `Dockerfile`: `{{developer}}`→`{{ developer }}`, `{{main/ns}}`→`{{ project-name }}`.
  - `LICENSE`: `{{now/year}}`→`{{ now-year }}`, `{{developer}}`→`{{ developer }}`.
  - `manifest.json`: `{{main/ns}}`→`{{ project-name }}` (two keys).

- [ ] **Step 4: Verify no unmapped tags remain anywhere**
  Run: `grep -rn '{{' template | grep -vE '\{\{ (project-name|developer|now-year)( \| snake_case)? \}\}' | grep -v '\${{'`
  Expected: no output (the only bare `{{` left are inside `{% raw %}` blocks as `${{ … }}`, excluded by the `\${{` filter, plus frame vars).

- [ ] **Step 5: Commit**
  `git commit -m "feat: frame template deploy files and remaining content"`

### Task 7: DaisyUI CSS + JS

**Files:**
- Modify: `template/resources/public/css/input.css`
- Rename: daisyui JS to conditional names

- [ ] **Step 1: `input.css`**
  Base content is `@import 'tailwindcss' source("../../../src");\n`. Append the daisyui delta as an inline `{% if daisyui %}` block reproducing `R/resources_public_css_daisyui/input.css` exactly (the extra `@plugin` lines). Diff both source `input.css` files first to isolate the exact delta bytes.

- [ ] **Step 2: DaisyUI JS conditional names**
  Copy `R/resources_public_js_daisyui/daisyui.js` → `template/resources/public/js/{%if daisyui%}daisyui.js{%endif%}` and `daisyui-theme.js` → `template/resources/public/js/{%if daisyui%}daisyui-theme.js{%endif%}`. These are raw (covered by `resources/public/js/**`).

- [ ] **Step 3: Commit**
  `git commit -m "feat: frame template daisyui css and js"`

### Task 8: Verification — byte-identical output across the matrix

**Files:** none (verification + fixes to `template/**` as needed).

- [ ] **Step 1: Build frame**
  Run in `$FRAME`: `lgx build`
  Expected: `bin/frame` produced.

- [ ] **Step 2: Clean untracked deps-new cache**
  Run in `$CSL`: `rm -rf resources/io/github/abogoyavlensky/clojure_stack_lite/root/.clj-kondo/.cache`
  (git-ignored; prevents deps-new from copying it into the comparison output.)

- [ ] **Step 3: Write a verification script**
  A bash loop over the 8 matrix rows. For each: generate deps-new output (`USER=frameuser clojure -X:local-new …` with the row's `:db/:auth/:daisyui/:deploy`, omitting a key to take its default) into `out/<n>/deps-new-out`, generate frame output (`bin/frame new --defaults --var …,developer=Frameuser --dir out/<n>/frame-out $CSL <name>`), then `diff -r`. Also compare tree shape: `diff <(cd deps-new-out && find . | sort) <(cd frame-out && find . | sort)`. Fail loudly on any non-empty diff. Put the script and all output under the scratchpad dir, not the repos.

- [ ] **Step 4: Run and fix until clean**
  Run the script. For each diff, fix the offending `template/` file (usually a whitespace/newline byte in an inline chunk) and re-run. Repeat until all 8 combos diff-clean (content and tree shape). Common culprits: trailing newline of a substitution chunk; a tag accidentally alone on its line consuming a newline; a nested `{{main/ns}}`/`{{main/file}}` not rewritten.

- [ ] **Step 5: Confirm deps-new fallback still passes**
  Run in `$CSL`: `bb test`
  Expected: PASS (template.edn still valid; deps-new files untouched).

- [ ] **Step 6: Commit any template fixes**
  `git commit -m "fix: match deps-new output byte-for-byte"` (skip if Step 4 needed no changes).

### Task 9: clojure-stack-lite README — frame usage note

**Files:**
- Modify: `/Users/andrew/Projects/clojure-stack-lite/README.md`

- [ ] **Step 1: Add "Create a project with frame" subsection**
  Under the existing Usage/Options section, alongside the deps-new instructions (kept as-is, the fallback): install/build frame, then
  `frame new https://github.com/abogoyavlensky/clojure-stack-lite my-app` (interactive) or non-interactive with `--defaults --var db=…,auth=…,daisyui=…,deploy=…,developer=…`. List the same four options + `developer`. Use /writing-clearly.

- [ ] **Step 2: Commit**
  `git commit -m "docs: frame usage instructions"`

---

## Notes for the executor

- **Byte-exactness is the whole game.** When authoring an inline chunk, copy the source `substitutions/*` file's bytes verbatim between the frame tags — do not retype. The diff in Task 8 is the arbiter; trust it over intuition.
- **Keep substitution tags inline** (not alone on their line) so frame's "tag owns the line" rule does not eat a newline the deps-new output keeps. Conversely, conditional path segments and whole-line control tags are fine to be alone — that is how empty-render skipping works.
- **Do not modify the deps-new template** beyond removing the untracked `.clj-kondo/.cache` junk. The fallback must stay intact (`bb test` green).
- If a genuine frame engine limitation surfaces (it should not), stop and surface it rather than distorting the template — parity is the requirement.
