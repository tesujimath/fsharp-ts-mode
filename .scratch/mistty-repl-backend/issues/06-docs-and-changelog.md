# 06 — Documentation and changelog

Status: ready-for-agent

## Summary

Document the mistty backend: its benefit (TAB completion), how to enable it, the
mistty install requirement, and the comint-only feature differences.

## Scope

- **CHANGELOG.md** — under `## main (unreleased)` → `### New features`, add an
  entry for the `fsharp-ts-repl-backend` option and the mistty backend (mention
  fsi TAB completion as the headline benefit and that mistty must be installed).
- **README.md** — note the mistty backend in the Features list; add a short
  configuration note: install `mistty` (MELPA) and `(setq fsharp-ts-repl-backend
  'mistty)`. Mention TAB completion in the REPL.
- **docs/** (mkdocs REPL page) — document:
  - `fsharp-ts-repl-backend` (`comint` default vs `mistty`).
  - A comint-vs-mistty tradeoff table (TAB completion, readline, history,
    fontification, output colours).
  - mistty install requirement and the graceful error if missing.
  - The **comint-only** caveats under mistty: input fontification, our output
    font-lock keywords, comint prompt-regexp/read-only, prettify-symbols in the
    REPL, and persistent history (note in-session ↑/↓ still works; cross-restart
    persistence is a planned follow-up).
- **Docstrings:**
  - `fsharp-ts-repl-backend` (from issue 01) — comint vs mistty, install note.
  - Add "comint backend only" notes to `fsharp-ts-repl-fontify-input`,
    `fsharp-ts-repl-history-file`, `fsharp-ts-repl-history-size`.

## Acceptance criteria

- `eldev lint` (package-lint/checkdoc) passes with the new/updated docstrings.
- README, CHANGELOG, and the mkdocs REPL page all mention the backend and the
  mistty requirement.
- The comint-only differences are documented in one discoverable place.

## Dependencies

Requires the feature (01–04) to be implemented; can be written in parallel with
05.

## Out of scope

- Persistent history (issue 07) — reference it as "planned" only.
