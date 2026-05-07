# spec-lean — Spec-Driven Development for Claude Code

A Claude Code plugin that runs a Spec-Driven Development cycle as a flat set of six skills (one per phase for the four phases, plus `update` and `run` helpers), and a read-only `/spec-lean:status` slash command. The cycle is **story-driven**: each spec is anchored on User Stories (`US-N` IDs), not test files. Each phase is self-contained — there's no orchestrator/worker indirection, and the plugin does not depend on any user-level skills. The only external dependencies are two marketplace plugins used by the ship phase.

## The cycle

```
/spec-lean:spec <feature>     →  Phase 1: Write spec interactively (6-section template, US-N IDs)
/spec-lean:build <feature>    →  Phase 2: Implement story-by-story, mark each US `done`
/spec-lean:validate <feature> →  Phase 3: Walk each story's Done-when checks (auto + manual)
/spec-lean:ship <feature>     →  Phase 4: SDD-aware pre-commit review → commit/push/PR → /code-review on PR → merge

/spec-lean:update <feature>   →  Iteration: amend spec, mark affected stories `in-progress`
/spec-lean:run <feature>      →  Orchestrator: runs all phases with checkpoints between

/spec-lean:status [<feature>] →  Inspect: aggregate done/in-progress/blocked rollup, or per-story drill-down (read-only)
```

`/spec-lean:status` is a slash command, not a skill — it never writes files, never invokes other phases, and never proposes next actions. Use it to check progress at any point in the cycle without disturbing it. With no argument it scans every `docs/specs/*/spec-status.md` and prints one row per spec; with a feature name it drills into that spec and prints one row per US-ID.

## Spec file convention

All specs live at `docs/specs/<feature>/spec.md` in the target project (not the plugin). Each spec folder also accumulates:

- `spec.md` — the spec itself (6 sections: Goal, Requirements, User Stories with US-N IDs, Technical details, Out of scope, Edge cases / open questions)
- `spec.md.prev` — snapshot from the last `/spec-lean:update` (used for diff)
- `spec-status.md` — live status per US-ID (`done` / `in-progress` / `blocked` / `not-started`)

In addition, the project has **one shared** `docs/codebase-map.md` — a project-wide table of source files with one-line role descriptions, append/merge-updated by `/spec-lean:build` after each spec.

## User Stories and Done-when checks

Each story in Section 3 has a stable `US-N` ID and may include an optional `Done when:` block listing concrete checks:

```markdown
- **US-2**: As a user, I want to be told when I enter a negative amount, so that bad data isn't stored.
  - Done when:
    - _(automated)_ `tracker add -5` — exits 2, stderr contains "amount must be positive"
    - _(manual)_ The CLI help text mentions the validation rule
```

`/spec-lean:validate` walks each story:

- **Stories with Done-when checks** — runs automated checks via Bash; presents manual checks for y/n confirmation.
- **Stories without Done-when checks** — falls back to a single y/n confirmation against the story narrative.

Done-when blocks are optional but recommended — without them, `/spec-lean:validate` can only ask the user to manually confirm the story works.

## Dependencies

### External plugins

Install these once before using `/spec-lean:ship` (the only phase that talks to the outside world):

```
/plugin install commit-commands@claude-plugins-official
/plugin install code-review@claude-plugins-official
```

- `commit-commands` provides `commit-commands:commit` (used when there's no remote) and `commit-commands:commit-push-pr` (used when there is one).
- `code-review` provides the `/code-review` slash command (run on the PR after push).

`/spec-lean:spec`, `/spec-lean:build`, `/spec-lean:validate`, `/spec-lean:update`, and `/spec-lean:run` have **no external skill dependencies** — they're entirely self-contained.

### System binaries

- `git` — required by `/spec-lean:build` (stash, branch, rollback) and `/spec-lean:ship` (commit, remote detection).
- `gh` (GitHub CLI), authenticated — required by `/spec-lean:ship` only when a remote is configured (push, PR, merge, code-review-on-PR).

The plugin itself is language-agnostic.

## Key design decisions

- **One skill per phase.** No orchestrator/worker split. Each phase's skill contains all the logic for that phase. Easier to audit, fewer cross-references, no convention drift between layers.
- **Story-driven.** The trackable unit is the User Story. `/spec-lean:build` implements story-by-story; `/spec-lean:validate` confirms each story works.
- **Done-when checks fold validation into the spec.** Each story's acceptance lives in its `Done when:` block — no separate Validation Steps section.
- **`.gitignore` scaffolded at build time.** `/spec-lean:build` adds a language-aware `.gitignore` if missing, including any runtime data file the spec names in Section 4. Catches the artifact-leak problem before commit.
- **Two-stage review on ship.** SDD-aware pre-commit review on the local diff (BLOCK/CAUTION/GO verdict, blocks bad commits) → commit/push/PR → `/code-review` on the PR (multi-agent, deeper). Both gates can refuse to proceed.
- **Iterative back-edge.** `/spec-lean:update` diffs `US-N` IDs and marks affected stories `in-progress`, never wiping `done` stories that didn't change.
- **Multi-feature handling.** Sub-capabilities of one feature go in one spec as separate `US-N` IDs; only genuinely independent features get separate specs. Specs are implemented sequentially.
- **Git is required for safety rails.** `/spec-lean:build` offers `git init` if missing; without git there's no stash, branch, or rollback.
- **No-remote handling.** `/spec-lean:ship` detects missing remote and offers commit-only / configure-then-proceed / cancel.
