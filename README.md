# Plugins Marketplace

A Claude Code plugin marketplace by [Akhiranandha](mailto:kodamakhira624@gmail.com).

## Available plugins

- **[spec-tests-first](plugins/spec-tests-first/)** — Spec-driven development with tests-first (TDD) discipline: spec → tests → build → validate → ship.
- **[spec-lean](plugins/spec-lean/)** — Spec-driven development without automated tests, story-driven (US-N IDs) with manual Done-when checks: spec → build → validate → ship.

## Adding this marketplace

```
/plugin marketplace add <repo-url>
```

Then install individual plugins:

```
/plugin install spec-tests-first@sdd-marketplace
/plugin install spec-lean@sdd-marketplace
```
