# Multi-Project Support for GSD

## What This Is

A feature that restructures GSD's `.planning/` folder to support multiple concurrent projects. Teams can share one `.planning/` folder in their repo, with each person or workstream getting isolated project context while being able to review each other's planning docs.

## Core Value

Multiple people can work on different things simultaneously in the same repo, with full visibility into each other's planning documents.

## Requirements

### Validated

- ✓ Orchestrator-agent model for coordinating distributed work — existing
- ✓ Phase-based project planning and execution — existing
- ✓ File-based artifact management in `.planning/` — existing
- ✓ Template-driven output consistency — existing
- ✓ Git integration for tracking planning artifacts — existing
- ✓ Multi-runtime support (Claude Code, OpenCode, Gemini) — existing
- ✓ Codebase mapping via `map-codebase` — existing
- ✓ Research, requirements, roadmap workflow — existing

### Active

- [ ] Restructure `.planning/` to support multiple projects as subdirectories
- [ ] Project naming defaults to git branch, with override option
- [ ] Store active project in `.planning/.active` file
- [ ] `/gsd:new-project [name]` creates project in `projects/<name>/`
- [ ] `/gsd:switch-project <name>` changes active project context
- [ ] `/gsd:list-projects` shows all projects with status
- [ ] `/gsd:archive-project [name]` deletes project folder
- [ ] Auto-migrate existing flat `.planning/` to `projects/default/`
- [ ] Shared codebase map at `.planning/codebase/`
- [ ] Global config with per-project overrides
- [ ] Prompt to select project when running GSD commands with no active project

### Out of Scope

- Real-time collaboration features — GSD is async via git
- Project permissions/locking — git handles this
- Cross-project dependencies — each project is independent
- Project templates — use existing GSD workflow

## Context

**Existing Structure:**
```
.planning/
├── PROJECT.md
├── REQUIREMENTS.md
├── ROADMAP.md
├── STATE.md
├── config.json
├── codebase/
├── research/
└── phases/
```

**Target Structure:**
```
.planning/
├── .active              ← "auth-refactor"
├── config.json          ← global defaults
├── codebase/            ← shared (repo-level)
└── projects/
    ├── default/         ← auto-migrated
    │   ├── config.json  ← per-project overrides
    │   ├── PROJECT.md
    │   ├── REQUIREMENTS.md
    │   ├── ROADMAP.md
    │   ├── STATE.md
    │   ├── research/
    │   └── phases/
    └── auth-refactor/
        ├── PROJECT.md
        └── ...
```

**Commands Affected:**
- All existing `/gsd:*` commands need to read/write from active project path
- New commands: `switch-project`, `list-projects`, `archive-project`
- Modified command: `new-project` gains project naming

**Migration Strategy:**
- Detect flat structure (PROJECT.md at root)
- Move all files to `projects/default/`
- Set `.active` to "default"
- Preserve codebase map at root level

## Constraints

- **Backwards Compatible**: Existing single-project repos must continue working
- **Zero Dependencies**: GSD has no runtime dependencies, keep it that way
- **Multi-Runtime**: Changes must work across Claude Code, OpenCode, Gemini
- **Git-Friendly**: Structure should produce clean diffs, no binary files

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Projects in `projects/` subdirectory | Keeps root clean, clear separation | — Pending |
| `.active` file for tracking | Simple, git-trackable, no env vars needed | — Pending |
| Branch name as default project name | Natural mapping for feature branches | — Pending |
| Shared codebase map | Repo structure doesn't change per project | — Pending |
| Archive = delete | Simplest approach, git history preserves if needed | — Pending |
| Prompt on no active project | Better UX than error or silent default | — Pending |

---
*Last updated: 2026-02-04 after initialization*
