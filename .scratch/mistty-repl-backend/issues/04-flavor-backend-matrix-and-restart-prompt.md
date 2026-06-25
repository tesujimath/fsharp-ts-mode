# 04 — Flavor × backend combinations and restart-on-mismatch prompt

Status: ready-for-agent

## Summary

Treat `fsharp-ts-repl-flavor` and `fsharp-ts-repl-backend` as orthogonal axes
(all four combinations valid), and generalize the existing "an existing REPL is
running; restart it as …?" prompt so it also fires when the *backend* changes,
not only the flavor.

## Why

`fsharp-ts-repl-switch-to-repl` already detects a running REPL whose flavor
differs from the desired `fsharp-ts-repl-flavor` and offers to restart it — this
is how a changed setting takes effect. The same affordance should apply when the
user flips `fsharp-ts-repl-backend`.

## Scope

In `fsharp-ts-repl.el`, `fsharp-ts-repl-switch-to-repl`:

- Compare the running REPL's recorded `(fsharp-ts-repl--flavor,
  fsharp-ts-repl--backend)` pair against the desired `(fsharp-ts-repl-flavor,
  fsharp-ts-repl-backend)`.
- If either differs (and the running values are known), prompt to restart with
  the requested settings, e.g.:
  `"An existing %s/%s REPL is running for this project; restart it as %s/%s? "`
  with flavor/backend in each slot.
- On confirmation, kill and restart so the new backend/flavor takes effect
  (reusing the existing kill + `fsharp-ts-repl-start` flow, which now records the
  new backend).
- Ensure all four `(flavor, backend)` combinations launch correctly (readline
  enforced per backend by issue 02).

## Acceptance criteria

- Changing `fsharp-ts-repl-backend` and invoking `switch-to-repl` offers to
  restart a running REPL of the other backend; declining leaves it running.
- Changing flavor still behaves as before.
- `dotnet`+`mistty`, `dotnet`+`comint`, `fsharpi`+`mistty`, `fsharpi`+`comint`
  all start.

## Dependencies

Requires 01, 03. Pairs with 02.

## Out of scope

- Tests (issue 05), docs (issue 06).
