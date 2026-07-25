# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is the **`ai-guidelines`** source repo — a collection of markdown guidelines that gets installed into downstream WordPress/WooCommerce projects as a git submodule. It is **not** a WordPress plugin.

When you edit files here, you are editing the rules that future Claude sessions follow in *other* repos. Changes propagate to consumers via `git submodule update`.

There is no build, no tests, no lint — the deliverable is the markdown itself.

## Two CLAUDE.md Files (Don't Confuse Them)

- **This file** (`/CLAUDE.md`) — guidance for Claude when *editing this repo*.
- **`.claude/CLAUDE.md`** — the entry point Claude reads in *consumer projects* after installation. It is the user-facing contract. Edit it deliberately.

## Repository Layout

```
.
├── README.md              Public install instructions (GitHub)
├── setup.sh               Submodule installer run by consumers
├── CLAUDE.md              This file
└── .claude/
    ├── CLAUDE.md          Consumer entry point — the "always loaded" rules
    └── guidelines/
        ├── index.md       Task → file routing table
        └── NN-*.md        Modular guides loaded on demand
```

## The Modular Loading Contract

The whole point of this repo is that `.claude/CLAUDE.md` stays small and routes Claude to one numbered guideline per task. Do not inline long content into `.claude/CLAUDE.md`.

Guideline files use a **numeric prefix** that implies ordering and rough topic grouping:

| Range | Topic |
|-------|-------|
| `00-` | Workflows (new plugin creation) |
| `01-02` | Setup, architecture |
| `03` | Modern PHP |
| `04-06` | Security, WooCommerce, performance |
| `07` | JavaScript |
| `08-11` | Docs, testing, static analysis, checklist |
| `12-13` | Advanced patterns, automation |
| `14-16` | E2E testing, Interactivity API, Playground |
| `17` | Quality gates, CI/CD |

When adding a new guideline, pick the next unused number in the right range. Do not renumber existing files — downstream CLAUDE.md and cross-references will break.

## When Adding or Renaming a Guideline

Three files must stay in sync. Update all of them in the same commit:

1. **`.claude/guidelines/NN-your-file.md`** — the new guideline itself.
2. **`.claude/guidelines/index.md`** — add a row to the "By Task Type" table **and** the "By File Count" table (include approximate line count).
3. **`.claude/CLAUDE.md`** — add a row to the Quick Start table if the task is common enough to route directly from the entry point.

Also update `README.md`'s Structure/Task Reference sections if the file is prominent enough to appear there.

## Core Rules Consumers Rely On

These are hard-coded across the guidelines. Changing them means a breaking change for every downstream project:

- PHP 8.0+ with `declare(strict_types=1)` in every file
- Namespace: `NVM\{Plugin}\`
- Hook prefix: `nvm/{plugin-slug}/`
- Author: `nevma` / `https://nevma.gr`
- No jQuery on frontend (vanilla `fetch()`); jQuery allowed in admin

If you touch these, update every guideline that mentions them, not just one.

## Specialized Agents

The guidelines reference three auto-triggered agents by name. These agents live in the user's global Claude config, not in this repo, but the names are part of the public contract:

- `wordpress-security-auditor` — triggered by AJAX/REST/forms/SQL
- `wordpress-performance-optimizer` — triggered by DB queries, loops, reports
- `wp-unit-test-writer` — triggered by new service classes, handlers, enums

Do not rename these in guidelines without confirming the agent definitions match.

## Editing Style

- Guidelines are read by Claude, not humans browsing docs. Prefer terse bullet lists and copy-pasteable code blocks over prose.
- Every code example must itself follow the Core Rules above — examples are the primary teaching signal.
- Keep file line counts close to what `index.md` advertises. If a file grows substantially, update its row in the "By File Count" table.
