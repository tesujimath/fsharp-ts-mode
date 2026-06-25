# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`fsharp-ts-mode` is a tree-sitter-based Emacs major mode for F#, requiring **Emacs 29.1+** with tree-sitter support. It is a pure Emacs Lisp package distributed on MELPA, inspired by its sibling OCaml mode [neocaml](https://github.com/bbatsov/neocaml).

## Commands

The project uses [Eldev](https://emacs-eldev.github.io/eldev/) as its build tool. The tree-sitter grammars must be installed before byte-compiling or testing.

```sh
# Install the fsharp + fsharp-signature grammars (required first, also in CI)
eldev eval '(fsharp-ts-mode-install-grammars)'

# Byte-compile; CI treats warnings as errors
eldev byte-compile --warnings-as-errors

# Run the full test suite (buttercup)
eldev test

# Lint (package-lint, checkdoc, etc.)
eldev lint
```

Tests use [buttercup](https://github.com/jorgenschaefer/emacs-buttercup). To run a single spec, filter by `describe`/`it` text with a pattern:

```sh
eldev test -p "project name detection"
```

CI (`.github/workflows/ci.yml`) runs compile + test + lint across Emacs 29.4, 30.1, and snapshot.

## Architecture

The package is split into one core file plus optional feature modules, each providing a separate minor mode that hooks into `fsharp-ts-mode`.

- **`fsharp-ts-mode.el`** — the core major mode (largest file). Font-lock, indentation, navigation, imenu, the install-grammars command, project detection, and Fantomas formatting. Everything else depends on this.
- **`fsharp-ts-repl.el`** — comint-based F# Interactive (`dotnet fsi`) REPL plus `fsharp-ts-repl-minor-mode` for sending code from source buffers. Input is highlighted with tree-sitter.
- **`fsharp-ts-dotnet.el`** — `fsharp-ts-dotnet-mode`, keybindings for `dotnet` CLI commands (build, test, run, clean, format, restore, watch) and compilation error parsing.
- **`fsharp-ts-eglot.el`** — enhanced Eglot/FsAutoComplete integration: LSP server auto-download, init options, feature toggles, `.fsproj` manipulation. `(require 'eglot)`.
- **`fsharp-ts-lens.el`** — `fsharp-ts-lens-mode`, inferred type-signature overlays (LineLens). Needs an active eglot/FsAutoComplete connection.
- **`fsharp-ts-info.el`** — `fsharp-ts-info-mode`, persistent doc panel for the symbol at point via the `fsharp/documentation` LSP endpoint. Needs eglot.

The eglot, lens, and info modules are **not** loaded by default — users `(require ...)` them explicitly. Keep that separation: core editing must work without an LSP server.

### The two F# grammars (critical)

F# tree-sitter ships **two independent grammars** with overlapping but distinct node types: `fsharp` (`.fs`/`.fsx`) and `fsharp_signature` (`.fsi`). They do **not** share node names (e.g. a binding is `function_or_value_defn` vs `value_definition`), so a single set of queries cannot serve both. Font-lock is split into shared rules (common nodes only) and grammar-specific rules (`fsharp-ts-mode--font-lock-settings-fsharp` / `--settings-signature`). Grammar-specific rules use `:override t`. When editing font-lock or indentation, always consider both grammars.

### Indentation and the offside rule

F# is whitespace-significant, creating a chicken-and-egg problem: the parser needs correct indentation to build the right tree, but indentation needs the tree. Consequences: re-indenting fully-unindented code does **not** work (ERROR nodes everywhere); round-trip and incremental indentation do. Special cases handled in code: trailing comments attached to the preceding binding, `.fsx` bare expressions forced to column 0, shebang lines excluded via `treesit-parser-set-included-ranges`, and the `no-node` rule for empty lines. Indentation rules are tried in order, first match wins.

**Read [`doc/DESIGN.md`](doc/DESIGN.md) before touching font-lock or indentation** — it documents grammar quirks, supertype query failures, font-lock levels (1–4), and the indentation rule ordering in detail.

## Testing conventions

Test helpers live in `test/fsharp-ts-mode-test-helpers.el`. Use the provided macros rather than hand-rolling buffers:

- `with-fsharp-ts-mode-buffer` / `with-fsharp-ts-signature-buffer` — set up a buffer in the right grammar.
- `when-fontifying-it` / `when-fontifying-signature-it` — face assertions.
- `when-indenting-it` / `when-newline-indenting-it` (+ `-signature` variants) — indentation assertions. Indentation tests rely on the round-trip property (already-correct input is preserved).

Sample fixtures are in `test/resources/` (`sample.fs`, `sample.fsi`, `sample.fsx`).

## Conventions

- `lexical-binding: t` everywhere. `fill-column` 80, no tabs (see `.dir-locals.el`).
- Set `fsharp-ts--debug` to `t` (indentation + `treesit-inspect-mode`) or `'font-lock` (also font-lock, very noisy) when debugging tree-sitter behavior.
- Update `CHANGELOG.md` for user-facing changes. Reference issues in commit messages as `#N` / `[Fix #N]`.
- User-facing docs live in `docs/` (mkdocs-material), deployed to GitHub Pages.

## Agent skills

### Issue tracker

Issues and PRDs are tracked as local markdown files under `.scratch/<feature>/`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default triage vocabulary (needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix), recorded as a `Status:` line in each issue file. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
