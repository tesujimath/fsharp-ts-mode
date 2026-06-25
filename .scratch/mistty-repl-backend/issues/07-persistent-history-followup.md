# 07 — Persistent REPL history under the mistty backend (DEFERRED follow-up)

Status: ready-for-agent

> **Deferred — do not start until the initial mistty backend (issues 01–06) is
> merged.** This is a separable follow-up shipped as its own PR. Listed here so
> the design captured during grilling isn't lost.

## Summary

Give the mistty backend cross-restart input history, comparable to comint's
`comint-input-ring` persistence. Out of the box, mistty+fsi only has fsi
readline's **in-session** ↑/↓ history, which is lost on restart.

## Why

Persistent history is the one genuinely useful comint feature dropped under the
interactive-first scope. fsi has no "load history file" directive and its
in-session ring can only be populated by *executing* commands, so persistence
must be implemented Emacs-side.

## Design (from grilling)

- **Capture on submit:** advise/hook the submission path
  (`mistty-send-command` while on our prompt) to read the just-submitted input
  from the prompt region (mistty exposes prompt boundaries) and push it into a
  ring.
- **Reuse comint's ring machinery standalone:** `comint-read-input-ring` /
  `comint-write-input-ring` operate over a ring var + file and do not require
  `comint-mode`. Point them at the **same `fsharp-ts-repl-history-file`** so
  history is **shared across both backends**.
- **Recall:** fsi owns ↑ on the prompt, so don't fight it — add a command (bound
  to a free `C-c` key in `fsharp-ts-repl-mistty-mode`) that does `completing-read`
  over the persisted ring and inserts the chosen entry into the prompt via
  `mistty-send-string` (without submitting). Coexists with fsi's in-session ↑.

## Known risk

The capture hook is the fragile part: reliably grabbing the submitted input
(multi-line, distinguishing real submits) by advising a third-party command.
Prototype this first and confirm robustness before wiring recall.

## Acceptance criteria (when undertaken)

- Submitting commands in a mistty REPL appends them to
  `fsharp-ts-repl-history-file`.
- A new mistty (or comint) REPL can recall history persisted by a previous
  session of either backend.
- fsi's in-session ↑/↓ continues to work unchanged.

## Dependencies

Requires the mistty backend (01–06) merged.
