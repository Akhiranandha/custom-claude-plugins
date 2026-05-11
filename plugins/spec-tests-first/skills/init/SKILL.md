---
name: init
description: Pre-cycle skill for spec-tests-first. Use when the user invokes /sdd:init to set up STF on a repository — auto-detects whether to run Case A (fresh repo, fresh project), Case B (existing codebase, no specs), or Case C (existing repo with flat specs requiring migration). Seeds CLAUDE.md (## Test commands + ## Test layout), scaffolds docs/codebase-map.md from existing source code using per-profile glob patterns, and for Case C migrates flat docs/specs/*.md files into the v2 subdirectory layout (US-N → AC-N.M conversion with TODO discriminating-signal flags for error-path ACs). Idempotent — re-running skips already-migrated artifacts.
argument-hint: <project-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:init — Pre-cycle: Bring a Repo into SDD

**Announce at start:** Say to the user: "I'm using /sdd:init to scan `$1` and decide which of three init paths fits (fresh / existing-codebase / migrate-flat-specs), then run only what's needed." Then proceed.

You are running the pre-cycle initializer for project **$1**. Inputs: the working directory state. Outputs: CLAUDE.md config (test commands + test layout), seeded docs/codebase-map.md, and — only in the migration case — restructured docs/specs/<feature>/ subdirectories with per-feature spec-status.md.

## Iron Law

> **Idempotent or nothing. Every action checks if the artifact already exists; if it does, skip with a printed reason. /sdd:init re-runs must be safe — never overwrite spec.md, spec-status.md, or completed CLAUDE.md sections.**

The whole point of init is to bring an existing project into SDD without destroying anything. Overwriting a live spec-status.md (which may contain in-progress phase tracking) is the worst-case failure mode.

## Pre-checks

1. **Working tree clean.** Run `git status --porcelain`. If dirty AND the changes are unrelated to init artifacts (`docs/specs/`, `docs/codebase-map.md`, `CLAUDE.md`), AskUserQuestion: stash WIP (recommended) / proceed anyway / cancel. On stash → `git stash push -u -m "sdd:init pre-start stash"`. On cancel → stop.
2. **Project name provided.** `$1` is the project's friendly name (used in next-step pointer + commit messages). If `$1` is empty, AskUserQuestion to capture one (default: the repo's root directory name).

## Step 1 — Auto-detect the init case

Determine which case applies by scanning the working directory.

### Detection signals

- **Flat specs present** = at least one `docs/specs/*.md` file (NOT inside a subdirectory). Use Glob: `docs/specs/*.md`.
- **Subdirectory specs present** = at least one `docs/specs/*/spec.md`. Use Glob: `docs/specs/*/spec.md`.
- **Source manifests present** = at least one of these files anywhere up to 3 levels deep: `package.json`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `setup.py`, `Gemfile`, `composer.json`, `*.csproj`.
- **CLAUDE.md `## Test commands` present** = boolean, from grep.

### Case decision tree

```
Flat specs present?
├─ YES → Case C (migration)
└─ NO  → Source manifests present?
         ├─ YES → Case B (existing codebase, no specs)
         └─ NO  → Case A (fresh repo, fresh project)
```

### Mixed-state handling

If BOTH flat specs AND subdirectory specs are present (someone migrated by hand for some features), this is **Case C — partial**. Process only the flat files; existing subdirectory specs are preserved untouched.

If subdirectory specs are present AND no flat specs, the project is **already initialized**. Print a summary of detected features and Phase progress (read each `spec-status.md`), then stop without changes.

### Print the plan to the user

After detection, print:

```
/sdd:init for project `<name>`.

Scanning...
  Source manifests detected: <comma-separated list with their subdirectories>
  docs/specs/ contains: <C_flat> flat files, <C_sub> subdirectories
  CLAUDE.md ## Test commands present: <yes / no>

Plan: Case <A | B | C> — <one-line description>

Continue?
  (a, recommended) Yes — run Case <X>
  (b) Cancel
```

AskUserQuestion. On (a): proceed to Step 2 (case-specific path). On (b): stop.

## Case A — Fresh repo, fresh project

**Triggered when:** no source manifests detected AND no `docs/specs/` directory or it's empty.

### A.1 — Confirm intent

AskUserQuestion:

> Detected an empty project — no source code, no existing specs.
> Is this the intended state for `<name>`?
> - (a, recommended) Yes — set up CLAUDE.md + empty codebase-map.md and exit cleanly
> - (b) No — let me cancel and check what's missing

### A.2 — Git pre-checks

Same shape as `/sdd:build`'s pre-checks 2 and 3:

- If not a git repo, offer `git init` (recommended), proceed without git, or cancel.
- If no `.gitignore` at the project root, scaffold a minimal language-agnostic one:

```
# Local scratch
*.log
*.tmp
.env
.env.*

# OS junk
.DS_Store
Thumbs.db
desktop.ini

# IDE
.vscode/
.idea/
*.swp

# Claude Code session artifacts
chat-session/
```

### A.3 — Resolve initial profile (optional)

AskUserQuestion:

> Want to pick a test framework + layout profile now, or leave that for the first `/sdd:spec` to detect?
> - (a) Pick now — I'll save it to CLAUDE.md
> - (b, recommended) Leave for later — /sdd:spec will detect on first invocation

On (a): present the 12 built-in profiles + `custom`. Save to `CLAUDE.md ## Test commands` + `## Test layout` using the single-service or multi-service shape from `/sdd:build`'s Step 2/3.

On (b): skip — `/sdd:spec` will handle this when invoked.

### A.4 — Scaffold empty codebase-map.md

Write `docs/codebase-map.md` (only if it doesn't exist):

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

| File | Role |
|------|------|

## Key invariants

<!-- TODO: add invariants as the project takes shape -->
```

### A.5 — Print next-step pointer

```
/sdd:init complete — project `<name>` ready for SDD.

What's set up:
  - .gitignore scaffolded (if missing)
  - CLAUDE.md ## Test commands + ## Test layout (if you picked A.3 option (a))
  - docs/codebase-map.md (empty, ready for first build)

Next: /sdd:spec <feature> — write your first feature spec
```

Stop. Skip downstream sections.

<!-- Slices 4-10 append below this point -->
