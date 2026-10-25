# `frame check --json` emits `vars` strings that `frame new --var` cannot replay when a value contains a comma

**Status: open**

## Problem

`frame check --json` gives every combination a `vars` string (`key=value` pairs joined by `,`) that is meant to be passed straight to `frame new --defaults --var <vars>`. That round trip breaks when a value contains a comma.

Repro: a string var with `:default "Doe, Jane"` produces `"author=Doe, Jane"` with `"ok": true`. Replaying it fails with `frame: invalid --var (expected key=value):  Jane`. The same applies to an enum option containing a comma.

The cause is in the `--var` syntax, not in `check`. `src/frame/new.lg:76` (`parse-cli-vars`) splits the whole flag on every `,` and has no escape. `applied-pairs` and `format-json` in `src/frame/check.lg` join values with a bare `,` without checking for this. The `--var` limitation predates `frame check`; `check` is the first place that generates `--var` strings mechanically, so it is where the gap shows up.

It is narrow in practice. A template needs a comma inside a string default or an enum option. Typing such a value by hand has always failed, and the clojure-stack-lite template has none. The human output (space-separated pairs) and the `answers` object in the JSON are unaffected; only `vars` and the error `example` string are.

## Fix

Two options, in rising cost:

1. Make `check` report values it cannot represent. While building `vars`, a combination whose applied value contains `,` is recorded as an error (grouped by message like the others), for example `var author: value "Doe, Jane" contains a comma and cannot be passed with --var`. This stops `ok: true` from advertising an unusable matrix entry. About ten lines in `check.lg` plus a test.
2. Add escaping to `--var`, for example `\,` in `parse-var-flags` / `parse-cli-vars` and the matching escape in `applied-pairs`. This fixes the root cause but changes a documented CLI format, so it needs a README update and a decision about backslash handling.

Notes for whoever picks this up:

- Option 1 can ship first and does not block option 2.
- Keep `answers` in the JSON as it is; it is lossless. Consumers that need exact values can read it instead of `vars`.
- Test with a string default and an enum option that each contain a comma, and assert `check` flags both.

## Origin

Found by the Codex review of the `frame check` commit (`58495fb`) while executing `docs/plans/2026-10-25-2155-conditional-vars-check-next-steps.md`, 2026-10-25. The review had the symptom and the cause right. It was left out of that plan because it is a gap in the existing `--var` format and fixing it changes the CLI contract, which the plan did not cover.
