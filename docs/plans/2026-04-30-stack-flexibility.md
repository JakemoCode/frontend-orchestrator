# Stack Flexibility — Make Plugin Defaults Overridable

**Status:** Idea, not started
**Date:** 2026-04-30
**Goal:** Keep React + TypeScript + CSS Modules + tokens.css + Apollo GraphQL + RTL/Vitest as defaults, but let users override the stack without forking the plugin.

## Problem

Stack assumptions are hardcoded as prose across command files, subagent prompts, and standards docs. A user on Vue/Svelte, or even a React user using styled-components or TanStack Query, can't drive the orchestration without editing plugin files.

The runner (`runner/`) is already stack-agnostic — this is a prompt/template problem, not a code problem.

## Current stack lock-in (29 references)

Concentrated in:
- `commands/build-component.md` — RTL tests, `src/styles/tokens.css`
- `commands/build-pipeline-open-prs.md` — PR checklist hardcodes "CSS Modules only / Tokens from tokens.css only"
- `commands/build-pipeline.md`, `commands/build-page.md`, `commands/visual-qa.md`, `commands/design-audit.md` — "RTL suite" gates
- `commands/ui-interview.md` — file paths (`.tsx`, `src/graphql/...`), test runner = vitest, token file = `src/styles/tokens.css`
- `subagents/component-builder.md`, `subagents/e2e-writer.md`, `subagents/design-auditor.md`, `subagents/wave-executor.md` — `.tsx` paths, RTL/typescript code blocks
- `standards/design-and-a11y.md`, `standards/ux-quality.md` — implicit stack in gate language
- Project-level `CLAUDE.md` PR template — same checklist

Full grep results captured in conversation 2026-04-30; rerun:
```
grep -rEn "css module|tokens\.css|apollo|graphql|\.tsx|typescript|RTL|vitest|useQuery|useMutation" \
  commands/ subagents/ standards/
```

## Two approaches

### Option A — Lightweight `STACK.md` override (~half day)

Add a single `STACK.md` at the plugin root with the defaults documented. Every command/subagent prepends a one-liner: *"Unless STACK.md specifies otherwise, assume…"*.

- **Pro:** Minimal change. Lets variant React stacks (styled-components, TanStack Query, Jest instead of Vitest) work today.
- **Con:** Won't gracefully handle non-React frameworks. PR checklist still says "CSS Modules" unless user edits it.
- **Good for:** 90% of likely users (React variants).

### Option B — Stack profile system (~1–2 days)

Single `stack.config.{md,yaml}` declares:
- Language (TS/JS)
- Framework (React/Vue/Svelte/Solid)
- Styling (CSS Modules + tokens / Tailwind / styled-components / vanilla-extract)
- Data layer (Apollo / TanStack Query / SWR / fetch)
- Test runner + library (Vitest+RTL / Jest+RTL / Playwright Component)
- File path conventions (component path template, test path template, token file)
- PR gate strings (templated)

Ship 2–3 prebuilt profiles:
- `react-ts-apollo` (current default)
- `react-ts-tanstack`
- `vue-ts-pinia` (or similar)

Every command/subagent reads the active profile via a shared "stack profile preamble" template. `/ui-interview` either picks a profile or writes a custom one.

- **Pro:** Real flexibility. Non-React users can use the orchestration machinery.
- **Con:** More files to keep coherent. Standards docs need parameterization (e.g., "no inline styles, tokens from `<configured-token-source>`").

## Recommended path

1. Start with **Option A** — ship `STACK.md` and preamble snippet. Validates whether anyone actually needs flexibility before paying for Option B.
2. If users request non-React support or multiple profiles, evolve to **Option B**. The `STACK.md` content becomes the seed for `react-ts-apollo` profile.

## File-level change list (Option A)

1. **Create** `STACK.md` at plugin root — documents current defaults as the canonical source.
2. **Create** `commands/_stack-preamble.md` (or inline snippet) — one-paragraph block: "Read STACK.md if it exists; otherwise assume [defaults]."
3. **Edit** each command in `commands/` (8 files) and subagent in `subagents/` (5 files) to reference the preamble instead of hardcoding stack terms. Replace:
   - `RTL tests` → `<test-library> tests`
   - `src/styles/tokens.css` → `<token-file>`
   - `CSS Modules only / Tokens from tokens.css only` → `<styling-gate>`
   - `.tsx` path templates → `<component-file-extension>` from STACK.md
4. **Edit** `standards/design-and-a11y.md` and `standards/ux-quality.md` to phrase gates in terms of the configured stack rather than CSS Modules specifically.
5. **Edit** project `CLAUDE.md` PR template to source from STACK.md.
6. **Update** `README.md` to document override mechanism.

## File-level change list (Option B, if pursued)

Everything in Option A, plus:
- `profiles/` directory with `react-ts-apollo.md`, `react-ts-tanstack.md`, etc.
- `stack.config.md` schema doc
- `/ui-interview` extended to ask "pick profile or customize"
- Standards docs templated with placeholders, not hardcoded gates

## Risk notes

- **PR checklist drift:** if PR template lives in project CLAUDE.md (outside plugin), users may see stale gates. Either move template into plugin or provide a snippet to copy.
- **Subagent prompts cache stack assumptions:** when editing, search for `.tsx`, `Apollo`, `useQuery`, `useMutation` examples — the example code blocks need parameterization or labeling as "example only".
- **Runner tests** (`runner/tests/`) reference `.tsx` paths in fixtures — verify they're treated as opaque strings, not parsed.

## Out of scope

- Translating all docs to support frameworks the maintainer doesn't use. Document the extension point; let users contribute profiles.
