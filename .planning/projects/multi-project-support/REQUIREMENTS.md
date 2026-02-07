# Requirements: Multi-Project Support

**Defined:** 2026-02-04
**Core Value:** Multiple people can work on different things simultaneously in the same repo, with full visibility into each other's planning documents.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [x] **FOUND-01**: `/gsd:new-project` accepts optional `[name]` argument for project naming
- [x] **FOUND-02**: Project name defaults to current git branch if not specified
- [x] **FOUND-03**: User can switch active project with `/gsd:switch-project <name>`
- [x] **FOUND-04**: User can list all projects with `/gsd:list-projects`
- [x] **FOUND-05**: User can delete a project with `/gsd:archive-project [name]`
- [x] **FOUND-06**: Active project is tracked in `.planning/.active` file
- [x] **FOUND-07**: Running GSD commands with no active project prompts user to select

### Structure

- [ ] **STRUCT-01**: Each project has isolated directory at `.planning/projects/<name>/`
- [ ] **STRUCT-02**: Project directories contain PROJECT.md, ROADMAP.md, STATE.md, config.json, research/, phases/
- [ ] **STRUCT-03**: Codebase map is shared at `.planning/codebase/` (not per-project)
- [ ] **STRUCT-04**: Existing flat `.planning/` structure can be migrated via `/gsd:new-project`
- [ ] **STRUCT-05**: Migration preserves all existing files and git history

### Configuration

- [x] **CONFIG-01**: Global config lives at `.planning/config.json`
- [x] **CONFIG-02**: Per-project config at `projects/<name>/config.json` overrides global
- [x] **CONFIG-03**: Config merge is shallow (project values replace global values)

### Integration

- [ ] **INTEG-01**: All existing `/gsd:*` commands work with active project context
- [ ] **INTEG-02**: `.active` file is gitignored to prevent merge conflicts
- [ ] **INTEG-03**: Works across Claude Code, OpenCode, and Gemini runtimes

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Enhanced Status

- **STATUS-01**: `/gsd:list-projects` shows last active timestamp
- **STATUS-02**: `/gsd:list-projects` shows phase progress (e.g., "Phase 3/8")
- **STATUS-03**: `/gsd:list-projects` shows git status (ahead/behind, uncommitted)

### UX Improvements

- **UX-01**: Fuzzy project search for `/gsd:switch-project`
- **UX-02**: Project templates for quick-start common types
- **UX-03**: Git worktree integration (detect worktrees, suggest project per worktree)

### Team Features

- **TEAM-01**: Project tagging/grouping (by team, epic, client)
- **TEAM-02**: Project hand-off documentation generation

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Real-time collaboration | GSD is async via git by design. Conflicts handled by git merge. |
| Project permissions/locking | Git already handles this via branch protection and PR workflow. |
| Cross-project dependencies | Breaks project isolation. If projects depend on each other, they should be phases in one roadmap. |
| Automatic project switching | Conflicts with GSD's explicit context management philosophy. Fragile heuristics. |
| Nested projects | Exponential complexity. Command routing becomes ambiguous. Use phases/milestones instead. |
| Project merge/split | State reconciliation is too complex. Create new project, archive old ones. |
| Global task queue | Context switching killer. Defeats project isolation. |
| Sync to remote service | Git is already the remote. No vendor lock-in. |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | Phase 2 | Complete |
| FOUND-02 | Phase 2 | Complete |
| FOUND-03 | Phase 2 | Complete |
| FOUND-04 | Phase 2 | Complete |
| FOUND-05 | Phase 2 | Complete |
| FOUND-06 | Phase 1 | Complete |
| FOUND-07 | Phase 2 | Complete |
| STRUCT-01 | Phase 1 | Complete |
| STRUCT-02 | Phase 1 | Complete |
| STRUCT-03 | Phase 1 | Complete |
| STRUCT-04 | Phase 1 | Complete |
| STRUCT-05 | Phase 1 | Complete |
| CONFIG-01 | Phase 3 | Complete |
| CONFIG-02 | Phase 3 | Complete |
| CONFIG-03 | Phase 3 | Complete |
| INTEG-01 | Phase 4 | Pending |
| INTEG-02 | Phase 4 | Pending |
| INTEG-03 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0

---
*Requirements defined: 2026-02-04*
*Last updated: 2026-02-07 after Phase 3 completion*
