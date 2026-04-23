# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **documentation-only repository**. It contains language-specific guides that instruct AI coding agents how to write idiomatic, production-grade code in downstream projects. There is no source code, no build system, no test suite, and no package manifest in this repo. The `.gitignore` is a generic Node template and does not imply Node tooling.

Do not try to `make`, `go build`, `pytest`, `dart test`, etc. here — those commands appear *inside* the guides as instructions for projects that consume these guides, not as commands to run against this repository.

## Layout

Three sibling directories, one per target language, each with the same two-file shape:

```
dart/     AGENTS.md   guideline.md
go/       AGENTS.md   guidline.md    # note: typo, missing 'e' — do not "fix"
python/   AGENTS.md   guideline.md
```

The `go/` filename is `guidline.md` (missing the `e`). Leave it alone unless the user explicitly asks to rename it — external references may depend on the current path.

## The two document types — know which to edit

Each language directory has two complementary files. Treat them as different products:

- **`AGENTS.md`** — the short, directive, agent-facing "rules of the road". Terse. Checklist-heavy. Lists non-negotiables, required commands, and "reject-on-sight" patterns. This is what agents in downstream projects are expected to read before writing code.
- **`guideline.md`** (or `guidline.md`) — the long reference manual. Numbered sections, rationale, code examples, decision heuristics, deeper patterns. This is where the *why* lives.

When a change arrives, decide up front which file it belongs in:

- A new rule or forbidden pattern? → `AGENTS.md` (and, if it warrants rationale, a matching section in `guideline.md`).
- A worked example, pattern catalogue entry, or explanation of trade-offs? → `guideline.md`.
- A command added to the "daily commands" list? → both, and they must agree.

If you edit one, check the other for drift. The two files are meant to stay consistent; divergence is a bug.

## Cross-language consistency

The three language guides are intentionally parallel in structure and tone (TDD-first, typed failures at boundaries, measure-before-optimizing, composition over inheritance, etc.). When adding or revising a section:

- Match the tone and section style of the existing file — terse imperative in `AGENTS.md`, numbered prose in `guideline.md`.
- If a cross-cutting principle (e.g. "measure before optimizing") already exists in the other language guides, align the wording and placement rather than inventing a new phrasing.
- Language-specific idioms are the point — do not flatten Dart's `const`-everything or Go's "accept interfaces, return structs" into generic advice. Keep the specificity.

## Editing conventions

- The guides are the product. Prose quality matters: short sentences, active voice, concrete examples, no filler.
- Code fences in the guides are illustrative, not executable — but they must still be syntactically valid and idiomatic in the target language.
- Do not add generic software-engineering platitudes ("write clean code", "use meaningful names"). The guides only earn their keep by being specific and opinionated.
- Do not introduce a README, contributing guide, or meta-documentation unless the user asks. The repo is deliberately flat.

## When the user asks to "add a language"

Create a sibling directory with the same two-file structure (`AGENTS.md` + `guideline.md`), matching the section layout and tone of the existing three. Mirror the standard sections (stack defaults, project layout, non-negotiables, testing, performance, security, before-submitting checklist) rather than inventing a new skeleton.
