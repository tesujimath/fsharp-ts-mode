# PRD: mistty backend for `fsharp-ts-repl`

Status: ready-for-agent

## Summary

Add an alternative REPL backend so F# Interactive (`dotnet fsi`) runs in a real
terminal via [mistty](https://github.com/szermatt/mistty) instead of comint. The
motivating benefit is **interactive TAB completion** in the REPL: comint is
line-oriented and never forwards `TAB` to fsi, so fsi's own readline completion
is unreachable. A terminal backend forwards keystrokes, unlocking fsi's
interactive line editing (completion, in-session history).

The backend is selected by a new customizable variable
`fsharp-ts-repl-backend`, with options `comint` (default, unchanged behaviour)
and `mistty`.

## Goals

- New `fsharp-ts-repl-backend` defcustom (`comint` | `mistty`), default `comint`.
- Under the mistty backend: a working interactive REPL with fsi TAB completion,
  and **all existing source-buffer sending commands** (`send-region`,
  `send-definition`, `send-buffer`, `load-file`, `send-project-references`,
  `require`, etc.) continue to work.
- comint backend behaviour is **byte-for-byte unchanged** for existing users.
- mistty remains an **optional** dependency: users who don't select the mistty
  backend need not install it, and core editing/the comint REPL keep working
  without it.

## Non-goals (interactive-first scope)

The following comint-specific REPL-buffer features are **not** reimplemented on
mistty — fsi's terminal handles its own rendering/history/colours instead. They
are documented as comint-only differences:

- Input fontification (`comint-fontify-input-mode` / `fsharp-ts-repl-fontify-input`).
- Our output font-lock keywords (`fsharp-ts-repl-font-lock-keywords`) — fsi emits
  its own ANSI colours.
- comint prompt-regexp / read-only prompt; prettify-symbols in the REPL buffer.
- **Persistent (cross-restart) history** — deferred to a follow-up (issue 07);
  in-session ↑/↓ history still works via fsi readline.

## Empirically verified (during design)

- ✅ TAB completion works in `dotnet fsi` running under mistty.
- ✅ fsi does **not** support bracketed paste (`mistty-bracketed-paste` stays
  `nil`); so paste-bracketing is not an option and is not used.
- ✅ Raw multi-line send preserves indentation and produces `val` echoes via
  fsi's `- ` continuation prompt (matches comint's behaviour).
- ✅ fsi's readline is reached via `dotnet fsi` **without** `--readline-` (the
  comint default disables it on purpose); mistty wants it **on**.

## Key design decisions

1. **Config & state.** `fsharp-ts-repl-backend` defcustom (`:safe`,
   dir-locals-settable). Buffer-local `fsharp-ts-repl--backend` recorded when a
   REPL starts, mirroring the existing `fsharp-ts-repl--flavor` /
   `fsharp-ts-repl--command-line`.
2. **Factoring.** Lightweight `pcase` dispatch over a few primitives
   (`--start-command`, `--running-p`, `--send`, `--kill`); a separate
   `fsharp-ts-repl-mistty-mode` deriving from `mistty-mode`. No
   `cl-defgeneric`/struct machinery.
3. **Dispatch basis.** Existing buffers dispatch on their **buffer-local**
   backend; the global var is consulted only for fresh starts. (Needed because
   `comint-check-proc` / `get-buffer-process` do not work on mistty buffers —
   mistty's process lives in a hidden term buffer, reached via
   `mistty-live-buffer-p` / `mistty-proc`.)
4. **Send.** Raw send via `mistty-send-string`, reusing the shared `;;`
   terminator logic; multi-line relies on fsi's continuation prompt. (Temp-file
   `#load` is a documented fallback, not implemented.)
5. **Readline as a backend concern.** Drop `--readline-` from the default
   `fsharp-ts-repl-program-args` (→ `("fsi")`); each backend **enforces** its
   requirement: comint ensures `--readline-`, mistty ensures `--readline+` /
   strips `--readline-`.
6. **Keymap.** Don't fight mistty's prompt overlay keymap: leave
   `TAB`/`RET`/`C-c C-c` (completion/submit/interrupt) to mistty; expose F# REPL
   extras via the menu; bind `switch-to-source` to **`C-c z`** (plain `z`,
   dodging mistty's `C-c C-z`).
7. **Lifecycle.** interrupt → `interrupt-process` on `mistty-proc`; restart →
   `mistty-exec` re-exec; clear → `mistty-clear` (trim-style, accepted).
8. **Flavor × backend.** All four combinations allowed; readline enforced
   generically; the restart-on-mismatch prompt generalized to the
   `(flavor, backend)` pair.
9. **Packaging.** Separate `fsharp-ts-repl-mistty.el` (hard-requires `mistty` +
   `fsharp-ts-repl`); core gates + lazy-loads it (`user-error` if mistty
   missing); mistty stays **out of `Package-Requires`**, added as an **Eldev
   dev/test dependency** so CI can compile/test it; `declare-function` stubs in
   core.

## Backend differences (summary)

| Concern            | comint                         | mistty                                  |
|--------------------|--------------------------------|-----------------------------------------|
| TAB completion     | no (line-oriented)             | **yes** (fsi readline)                  |
| readline flag      | `--readline-` (enforced)       | `--readline+` (enforced)                |
| process liveness   | `comint-check-proc`            | `mistty-live-buffer-p` / `mistty-proc`  |
| send               | `comint-send-string`           | `mistty-send-string` (raw, `;;`)        |
| switch-to-source   | `C-c C-z`                      | `C-c z` (menu also)                     |
| interrupt          | `interrupt-process`            | `interrupt-process` on `mistty-proc`    |
| clear              | erase + fresh prompt           | `mistty-clear` (trim scrollback)        |
| input fontify      | `comint-fontify-input-mode`    | n/a (terminal)                          |
| output highlight   | our font-lock keywords         | fsi's own ANSI colours                  |
| persistent history | comint ring → file             | deferred (issue 07); in-session only    |

## Implementation issues

- `01-backend-config-and-dispatch.md` — config var, buffer-local state, dispatch
  primitives (comint implemented; mistty branch stubbed/gated).
- `02-readline-backend-enforcement.md` — make readline a backend concern.
- `03-mistty-module-and-mode.md` — `fsharp-ts-repl-mistty.el`: major mode +
  mistty implementations + core gate + Eldev dev-dep.
- `04-flavor-backend-matrix-and-restart-prompt.md` — combinations + generalized
  mismatch prompt.
- `05-tests.md` — pure-logic buttercup tests for the backend.
- `06-docs-and-changelog.md` — CHANGELOG, README, mkdocs, docstrings.
- `07-persistent-history-followup.md` — **deferred** follow-up (separate PR).

## References

- Source: `fsharp-ts-repl.el`.
- mistty API used: `mistty-mode`, `mistty-exec`, `mistty-send-string`,
  `mistty-send-command`, `mistty-live-buffer-p`, `mistty-proc`, `mistty-clear`,
  `mistty-mode-map` / `mistty-prompt-map`.
- Design discussion: this directory's grilling session (2026-06-25).
