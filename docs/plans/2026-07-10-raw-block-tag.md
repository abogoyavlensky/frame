# Raw Block Tag Implementation Plan

> **For agentic workers:** Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `{% raw %}...{% endraw %}` block tag that emits its content verbatim, so templates can contain other tools' template syntax (e.g. GitHub Actions `${{ secrets.X }}`) without frame trying to render it.

**Tech Stack:** frame (let-go/lgx), existing template engine in `src/frame/template/`.

---

## Design

### Approach

`raw` is handled entirely in the **lexer's linear scan** (`src/frame/template/lexer.lg`),
not the parser. The protected span may contain foreign templater syntax that is not
even lexable (an unterminated `{{`, a stray `{%`), so it must never reach
tokenization — Liquid implements `raw` the same way. Parser (`parser.lg`), renderer
(`render.lg`), and filters stay untouched except for one parser error message.

### Mechanics

1. **Scan (`scan` in lexer.lg):** when a lexed tag's trimmed inner text is `raw`,
   switch to a forward search for the next `{% ... %}` whose trimmed inner is
   `endraw`. The search is spacing-tolerant (`{%endraw%}`, `{% endraw  %}` all
   terminate). Search loop: find the next `{%`; find its `%}`; if the trimmed inner
   is `endraw`, stop; otherwise resume the search **just past the `{%`** (advance
   by 2, not past the `%}`), so junk `{%` inside the content cannot hide the real
   terminator (e.g. content `{% {% endraw %}` still terminates at the inner
   `{% endraw %}`). Emit three tokens:
   - a `raw` marker tag token (`{:type :tag :value "raw" :marker true :line n}`),
   - one verbatim `:text` token for the span between the delimiters,
   - an `endraw` marker tag token (also `:marker true`).
   Line numbers are tracked across the whole consumed region (`count-newlines`).
   No `endraw` found → throw `{:reason :syntax :line <line of the raw tag>}` with
   message `unclosed 'raw' tag`. An empty span emits no text token between markers.

2. **Whitespace rule applies to the markers:** marker tokens flow through the
   existing line-ownership pass (phase 2) like any tag, so a standalone
   `{% raw %}` or `{% endraw %}` line vanishes without a phantom blank line, and
   an inline marker is removed in place — the protected content itself lands
   exactly as written. After ownership, `tokenize` **drops tokens flagged
   `:marker`**, so the parser never sees them. The drop happens in both modes
   (default and `:inline`).

3. **Stray `{% endraw %}`** (no opening `raw`) lexes as a normal tag (no
   `:marker` flag) and reaches the parser: `parse-body`'s catch-all reports it.
   Add an explicit branch so the message is `'endraw' without matching 'raw'`
   (`{:reason :syntax :line n}`) instead of the generic `unknown tag: 'endraw'`.
   A bare `{% raw %}` never reaches the parser (the lexer either pairs it or
   throws), so only `endraw` needs the explicit message.

4. **Path segments** get raw for free — `scan` runs before the `:inline` mode
   branch in `tokenize`, and marker-dropping applies in both modes.

### Key decisions

- **First `endraw` always terminates; no nesting** — Liquid semantics. A raw
  block therefore cannot contain the literal text `{% endraw %}`. Documented
  limitation.
- **Markers obey the standard whitespace rule** rather than preserving every
  byte between the delimiters — consistent with every other frame tag;
  `{% raw %}` on its own line leaves no trace in the output.
- Known edge, unchanged: `{% raw %}{% endraw %}` adjacent on one line does not
  own the line (identical to today's adjacent-tag behavior such as
  `{% endif %}{% if x %}`).

### Error handling

- Unclosed `{% raw %}` → `{:reason :syntax :line <raw tag line>}`, message
  `unclosed 'raw' tag` (thrown by the lexer).
- `{% endraw %}` without `raw` → `{:reason :syntax :line n}`, message
  `'endraw' without matching 'raw'` (thrown by the parser).
- Content inside a raw block can never raise render errors — it is a text token.

### Testing

TDD against the existing test layout: lexer behavior in
`test/frame/template/lexer_test.lg`, rendered behavior in
`test/frame/template/render_test.lg`, one end-to-end generation test in
`test/frame/generate_test.lg` using the existing temp-template pattern.

## File Structure

- Modify: `src/frame/template/lexer.lg` — raw-mode span consumption in `scan`;
  marker-dropping in `tokenize`
- Modify: `src/frame/template/parser.lg` — explicit `endraw` error message in
  `parse-body`
- Modify: `README.md` — "Raw blocks" subsection under Template syntax
- Test: `test/frame/template/lexer_test.lg`, `test/frame/template/render_test.lg`,
  `test/frame/generate_test.lg`

## Tasks

### Task 1: lexer — raw span consumption and marker dropping

**Files:**
- Modify: `src/frame/template/lexer.lg`
- Test: `test/frame/template/lexer_test.lg`

- [ ] **Step 1: Write failing lexer tests**
  Following the existing test style in `lexer_test.lg`:
  - `{% raw %}{{ x }} and {% if y %}{% endraw %}` tokenizes to exactly one
    `:text` token with value `{{ x }} and {% if y %}` (markers dropped, nothing
    lexed inside).
  - Spacing variants terminate: `{%endraw%}` and `{% endraw   %}`.
  - Junk `{%` inside the span does not hide the terminator: content
    `a {% b {{ endraw %}`-style cases; e.g. source
    `{% raw %}a {% b {% endraw %}` yields text `a {% b `.
  - Unterminated raw throws `{:reason :syntax}` with the line of the `raw` tag
    (put the raw tag on line 3 of a multi-line source and assert `:line 3`).
  - Line numbers after a multi-line raw span stay correct: a token following the
    span carries the right `:line`.
  - Standalone `{% raw %}` / `{% endraw %}` lines own their lines: source
    `"A\n{% raw %}\n{{ x }}\n{% endraw %}\nB\n"` tokenizes to text
    `"A\n"`, text `"{{ x }}\n"`, text `"B\n"` (or equivalent token split).
  - `:inline` mode: `(tokenize s {:mode :inline})` on `"{% raw %}{{ x }}{% endraw %}"`
    yields one text token `{{ x }}` (markers dropped, no ownership pass).

- [ ] **Step 2: Run tests to verify they fail**
  Run: `lgx test`
  Expected: new assertions FAIL (`unknown tag` / wrong tokens).

- [ ] **Step 3: Implement in lexer.lg**
  In `scan`, after computing `inner` for a `:tag` token: if `(= inner "raw")`,
  enter the endraw search described in the Design (loop with `str/index-of` on
  `"{%"` / `"%}"`, trimmed-inner comparison, advance-by-2 on non-match). Emit the
  three tokens with `:marker true` on both tag tokens and correct `:line` values;
  continue the main loop after the `endraw`'s `%}`. Throw `unclosed 'raw' tag`
  with the raw tag's line when the search exhausts the input. In `tokenize`,
  filter out `:marker` tokens after the ownership pass (default mode) and after
  `scan` (`:inline` mode).

- [ ] **Step 4: Run tests to verify they pass**
  Run: `lgx test`
  Expected: PASS.

- [ ] **Step 5: Commit**
  `git commit -m "feat: raw block tag in template lexer"`

### Task 2: parser — explicit stray-endraw error

**Files:**
- Modify: `src/frame/template/parser.lg`
- Test: `test/frame/template/parser_test.lg`

- [ ] **Step 1: Write failing test**
  Parsing tokens of `"text {% endraw %} more"` throws `{:reason :syntax}` with
  message containing `'endraw' without matching 'raw'`.

- [ ] **Step 2: Run test to verify it fails**
  Run: `lgx test`
  Expected: FAIL (current message is `unknown tag: 'endraw'`).

- [ ] **Step 3: Implement**
  In `parse-body`'s tag `cond`, before the `block-keywords` / unknown-tag
  branches, add: `(= kw "endraw")` → throw `{:reason :syntax :line (:line tok)}`
  with message `'endraw' without matching 'raw'`.

- [ ] **Step 4: Run tests, full check**
  Run: `lgx check`
  Expected: PASS.

- [ ] **Step 5: Commit**
  `git commit -m "feat: clear error for endraw without raw"`

### Task 3: render + generate coverage

**Files:**
- Test: `test/frame/template/render_test.lg`, `test/frame/generate_test.lg`

- [ ] **Step 1: Write render tests**
  In `render_test.lg` (existing style, via `render-string`):
  - `{% raw %}${{ secrets.TOKEN }}{% endraw %}` with empty vars renders
    `${{ secrets.TOKEN }}` — no unresolved-var error.
  - Raw block mixed with live tags: `Hi {{ name }} {% raw %}{{ name }}{% endraw %}`
    with `{"name" "Ann"}` renders `Hi Ann {{ name }}`.
  - Standalone raw/endraw lines leave no phantom blank lines (multi-line
    fixture mirroring the whitespace-rule tests already in the file).

- [ ] **Step 2: Write one end-to-end generate test**
  In `generate_test.lg`, using the existing temp-template pattern
  (`tmp-dir`, `spit` a `frame.edn` + `template/` file): a template file whose
  content contains a raw block with `{{ not-a-var }}` generates successfully and
  the output file contains the literal text.

- [ ] **Step 3: Run full check**
  Run: `lgx check`
  Expected: PASS.

- [ ] **Step 4: Commit**
  `git commit -m "test: render and generation coverage for raw blocks"`

### Task 4: README documentation

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add "Raw blocks" subsection**
  Under "Template syntax", after the case-blocks section: what `{% raw %}` does,
  the GitHub Actions `${{ secrets.X }}` motivating example, the whitespace rule
  applying to the marker tags, the no-nesting/first-endraw-wins limitation, and
  the relation to `:raw` globs (globs protect whole files verbatim; the tag
  protects spans inside an otherwise-rendered file). Use /writing-clearly.

- [ ] **Step 2: Commit**
  `git commit -m "docs: raw block tag"`
