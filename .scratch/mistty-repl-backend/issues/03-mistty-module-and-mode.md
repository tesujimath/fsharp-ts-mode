# 03 — `fsharp-ts-repl-mistty.el`: major mode and backend implementations

Status: ready-for-agent

## Summary

Create the `fsharp-ts-repl-mistty.el` module holding all mistty-specific code:
the `fsharp-ts-repl-mistty-mode` major mode and the mistty implementations of the
dispatch primitives. Wire the core gate so the module is lazy-loaded only when
the mistty backend is used, and add mistty as an Eldev dev/test dependency so CI
can compile and test it.

## Why

Keeps mistty code in its own file for clarity while leaving mistty an optional
dependency for users (out of `Package-Requires`). Core dispatch (issue 01)
delegates to the functions defined here.

## Scope

New file `fsharp-ts-repl-mistty.el`:

- Header `lexical-binding: t`; `(require 'mistty)` and `(require 'fsharp-ts-repl)`
  (hard requires — this file is only loaded when the mistty backend is used).
- `fsharp-ts-repl-mistty-mode`, `define-derived-mode` from `mistty-mode`:
  - Keymap: bind `C-c z` → `fsharp-ts-repl-switch-to-source`. Do **not** bind
    `TAB`, `RET`, or `C-c C-c` (leave mistty's completion/submit/interrupt).
    Reuse the same F# REPL menu items as `fsharp-ts-repl-mode` (Switch to Source,
    Interrupt, Restart, Clear Buffer, Customize). Note keys mistty does not bind
    fall through to this map even on the prompt; keys it does bind win on the
    prompt (by design — do not mutate mistty's global maps).
  - Set `mode-name`/record `fsharp-ts-repl--flavor` and
    `fsharp-ts-repl--backend` like the comint path.
- Implement the mistty branch of the primitives (named e.g.
  `fsharp-ts-repl-mistty--start`, `--running-p`, `--send`, `--kill`, `--clear`,
  `--interrupt`), called from core via the gate in issue 01:
  - **start / restart:** if the buffer already exists and is in
    `fsharp-ts-repl-mistty-mode`, `(with-current-buffer buf (mistty-exec
    command))` (mistty-exec replaces a running program in place — this covers
    restart). Otherwise create the buffer, enable `fsharp-ts-repl-mistty-mode`,
    then `(mistty-exec command)`. Display/return the buffer as the comint path
    does. Record flavor/backend/command-line buffer-locally.
  - **running-p:** `mistty-live-buffer-p` on the buffer (handles mistty's hidden
    term-buffer process model; `comint-check-proc` does not work here).
  - **send:** `(with-current-buffer buf (mistty-send-string text))` where `text`
    already has the shared `;;` terminator applied by the backend-neutral
    `fsharp-ts-repl--input-sender` logic; submit with a trailing newline /
    `mistty-send-command` as appropriate. Raw send (no bracketed paste — fsi does
    not support it).
  - **interrupt:** `interrupt-process` on the buffer-local `mistty-proc`.
  - **clear:** `mistty-clear` (trim-style; accepted difference from comint's
    full wipe).
  - **kill:** tear down the mistty process/buffer cleanly (kill `mistty-proc`
    with query-on-exit cleared, consistent with the comint `--kill`).
- `(provide 'fsharp-ts-repl-mistty)`.

Core (`fsharp-ts-repl.el`): the mistty branch gate from issue 01 now resolves to
real functions. Confirm `declare-function` stubs match.

Build/packaging:

- Add mistty as an Eldev dependency for compile + test (e.g.
  `(eldev-add-extra-dependencies 'test 'mistty)` and ensure byte-compilation of
  the new file has mistty available). Do **not** add mistty to
  `Package-Requires`.
- `.github/workflows/ci.yml`: confirm the eldev steps pull the dependency (Eldev
  resolves from the configured MELPA archive; no workflow change expected, but
  verify the new file byte-compiles under `--warnings-as-errors`).

## Acceptance criteria

- With `(setq fsharp-ts-repl-backend 'mistty)` and mistty installed, `M-x
  fsharp-ts-repl-start` launches `dotnet fsi` in a `fsharp-ts-repl-mistty-mode`
  buffer; `TAB` completes; `C-c z` returns to the source buffer; the F# REPL
  menu works.
- Source-buffer commands (`send-region`/`send-definition`/`send-buffer`/
  `load-file`/`send-project-references`) work against the mistty REPL.
- restart re-execs in place; interrupt stops a running eval; clear trims
  scrollback.
- With mistty **not** installed, selecting the backend yields the issue-01
  `user-error`, and core/comint paths are unaffected.
- `eldev byte-compile --warnings-as-errors` and `eldev test` are green.

## Dependencies

Requires 01 (dispatch + gate). Pairs with 02 (readline enforcement) for the
mistty `--readline+` requirement.

## Out of scope

- flavor/backend mismatch prompt (issue 04), tests (issue 05), docs (issue 06),
  persistent history (issue 07).
