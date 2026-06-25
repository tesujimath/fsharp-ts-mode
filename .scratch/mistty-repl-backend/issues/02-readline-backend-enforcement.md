# 02 — Make readline a backend responsibility

Status: ready-for-agent

## Summary

Move the readline on/off decision out of user-facing config and into the
backend. comint requires fsi readline **off** (`--readline-`); mistty requires
it **on** (that is where TAB completion lives). Each backend enforces its
requirement when building the launch command.

## Why

Today `fsharp-ts-repl-program-args` defaults to `("fsi" "--readline-")`, and the
docstring states the flag exists because readline conflicts with comint. That
bakes a comint-specific necessity into shared config that the mistty path would
otherwise have to fight. readline is dictated by the backend, not a user
preference.

## Scope

In `fsharp-ts-repl.el`:

- Change the default of `fsharp-ts-repl-program-args` from `("fsi" "--readline-")`
  to `("fsi")` (the backend-neutral base, plus whatever extra flags the user
  wants, e.g. `--use:`). Update its docstring (remove the comint/readline
  rationale; point at the backend for readline handling).
- In command construction (`fsharp-ts-repl--command` / `--start-command`), apply
  per-backend **enforcement** to the final argument list:
  - comint: ensure `--readline-` is present and `--readline+` is absent.
  - mistty: ensure `--readline+` is present and `--readline-` is absent.
  - Enforcement must be robust to a user who hand-set the "wrong" flag in
    `fsharp-ts-repl-program-args` (strip the wrong one, add the right one; no
    duplicates).
  - Apply generically regardless of flavor (`dotnet` and `fsharpi` share the fsi
    flag vocabulary). Note: the `fsharpi` flavor currently passes no args; after
    this change it should also receive the enforced readline flag.
- Update the existing `"REPL flavor"` test in `fsharp-ts-repl-test.el`: the
  comint command must still contain `--readline-` (now via enforcement, not the
  default args). (Mistty-side assertions live in issue 05.)

## Acceptance criteria

- `fsharp-ts-repl--command`/start for the comint backend yields a command list
  containing `--readline-` and not `--readline+`, for both flavors.
- The mistty backend yields `--readline+` and not `--readline-`.
- A user-set `fsharp-ts-repl-program-args` containing the wrong readline flag is
  corrected by enforcement (verified by issue 05).
- Existing tests updated and green.

## Dependencies

Builds on 01 (backend concept). The mistty assertions depend on 03 being present
for a live check but the enforcement logic itself is backend-parameterized and
testable without the module.

## Out of scope

- The mistty module/mode (issue 03).
