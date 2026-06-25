# 01 — Backend config variable and dispatch scaffolding

Status: ready-for-agent

## Summary

Introduce the `fsharp-ts-repl-backend` customizable variable, the buffer-local
state that records what a running REPL was launched as, and the backend-dispatch
primitives that all higher-level commands route through. In this issue the
comint branch is fully implemented; the mistty branch is stubbed (signals a
clear "not yet implemented / requires the mistty backend module" error) and is
filled in by issue 03.

## Why

`fsharp-ts-repl.el` currently calls comint directly (`comint-check-proc`,
`get-buffer-process`, `comint-send-string`, `make-comint-in-buffer`) in ~8
places. To support a second backend we need a thin abstraction those call sites
go through, and we must record the backend per REPL buffer so dispatch is
correct even if the global setting changes while a REPL is live.

## Scope

In `fsharp-ts-repl.el`:

- Add defcustom `fsharp-ts-repl-backend`:
  - `:type '(choice (const :tag "comint" comint) (const :tag "mistty" mistty))`
  - default `'comint`, `:safe` predicate `(lambda (v) (memq v '(comint mistty)))`,
    `:package-version '(fsharp-ts-mode . "0.1.0")`.
  - Docstring: explain comint (line-oriented, default) vs mistty (terminal,
    enables fsi TAB completion; requires the `mistty` package). Mention it can be
    set per-project via `.dir-locals.el`.
- Add buffer-local `fsharp-ts-repl--backend` (mirrors `fsharp-ts-repl--flavor`).
  Record it in `fsharp-ts-repl--start-command` alongside flavor/command-line.
- Introduce dispatch primitives that branch per backend:
  - `fsharp-ts-repl--backend-for (bufname)` — returns the effective backend: the
    buffer-local `fsharp-ts-repl--backend` if the buffer exists, else the global
    `fsharp-ts-repl-backend`.
  - `fsharp-ts-repl--running-p (bufname)` — comint: `comint-check-proc`; mistty:
    delegated to the module (issue 03).
  - `fsharp-ts-repl--send (bufname text)` — comint: existing `--input-sender`
    logic via `comint-send-string`; mistty: delegated (issue 03). Keeps the
    shared `;;`-terminator handling (`fsharp-ts-repl--input-sender`) backend-
    neutral so both backends reuse it.
  - `fsharp-ts-repl--kill (bufname)` — extend the existing function to dispatch.
  - `fsharp-ts-repl--start-command` — dispatch the actual start per backend
    (comint: `make-comint-in-buffer` as today; mistty: delegated).
- Replace the direct comint call sites throughout the file
  (`fsharp-ts-repl-start`, `--ensure-running`, `--process`/sending commands,
  `switch-to-repl`, `clear-buffer`, `interrupt`, `restart`, menus’ `:enable`)
  with the new primitives.
- mistty branch of each primitive: for now, `(fsharp-ts-repl--require-mistty)`
  which does `(unless (require 'mistty nil t) (user-error "The mistty backend
  requires the `mistty' package (install from MELPA)"))` then
  `(require 'fsharp-ts-repl-mistty)` and calls the module function. Until issue
  03 lands, this errors cleanly. Add `declare-function` stubs for the
  `fsharp-ts-repl-mistty--*` functions.

## Acceptance criteria

- With `fsharp-ts-repl-backend` unset/`comint`, all existing REPL behaviour and
  tests pass unchanged.
- All direct `comint-check-proc` / `get-buffer-process` call sites in the file
  now go through the new primitives.
- Selecting `'mistty` before issue 03 lands yields a clear `user-error`, never a
  void-function backtrace.
- `eldev byte-compile --warnings-as-errors` is clean (declare-function stubs in
  place).

## Dependencies

None. Precedes 02, 03, 04.

## Out of scope

- mistty implementations (issue 03), readline enforcement (issue 02),
  flavor/backend mismatch prompt (issue 04).
