# 05 — Tests for the mistty backend

Status: ready-for-agent

## Summary

Add pure-logic buttercup tests for the backend dispatch and readline
enforcement, matching the existing test suite's convention (no live `dotnet fsi`
process is ever launched). Live behaviour (TAB completion, send round-trip) is
covered by manual verification, captured in the PR description.

## Why

The existing suite (`test/fsharp-ts-repl-test.el`) is entirely pure-logic:
command building, buffer naming, directive formatting, JSON parsing, mocked
sends. The mistty backend should be tested the same way; spawning fsi/mistty in
CI is out of scope (and would require the .NET SDK).

## Scope

New file `test/fsharp-ts-repl-mistty-test.el` (separate file, consistent with
the per-module test files for eglot/lens/info):

- `fsharp-ts-repl-backend` defaults to `'comint`.
- **Readline enforcement** (the core of issue 02):
  - comint command list contains `--readline-`, not `--readline+`.
  - mistty command list contains `--readline+`, not `--readline-`.
  - When `fsharp-ts-repl-program-args` is hand-set with the wrong flag,
    enforcement strips it and adds the right one (no duplicates), for each
    backend.
- **Dispatch routing** via buttercup `spy-on` (no live process):
  - With a buffer whose buffer-local `fsharp-ts-repl--backend` is `mistty`,
    `fsharp-ts-repl--running-p` calls `mistty-live-buffer-p`; with `comint` it
    calls `comint-check-proc`.
  - `fsharp-ts-repl--send` routes to `mistty-send-string` vs `comint-send-string`
    accordingly.
  - `fsharp-ts-repl--backend-for` returns the buffer-local backend for an
    existing buffer and the global var otherwise.
- The shared `;;` terminator logic is applied on the mistty send path (assert the
  string handed to `mistty-send-string` is `;;`-terminated, via spy).

Update `test/fsharp-ts-repl-test.el`: the `"REPL flavor"` test's expected comint
command (now `--readline-` via enforcement; default args no longer carry it).

Build: relies on mistty being an Eldev test dependency (issue 03) so
`(require 'fsharp-ts-repl-mistty)` and `spy-on` of mistty functions resolve.

## Acceptance criteria

- `eldev test` is green, including the new file.
- `eldev test -p "readline"` (or similar) exercises enforcement for both
  backends.
- No test launches a real fsi/mistty process.

## Dependencies

Requires 02 and 03 (and 01). The flavor-test update overlaps issue 02 — keep them
consistent.

## Out of scope

- Live integration tests; docs (issue 06).
