# Phase 1: Foundation & Migration - Context

**Gathered:** 2026-02-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Build the infrastructure that makes multi-project work: path resolution to route file operations to the active project, auto-migration from flat `.planning/` structure to nested `projects/<name>/` structure, and active project tracking via `.active` file. This phase does NOT include the user-facing commands (create, switch, list, archive) — those are Phase 2.

</domain>

<decisions>
## Implementation Decisions

### Error Handling
- If `.active` references a project that doesn't exist → prompt user to select from available projects
- If `.planning/projects/` directory is missing entirely → guided recovery (explain situation, offer to initialize structure)
- When migration fails partway through → best effort (keep what worked, report what failed, let user fix manually)

### Path Resolution
- When no `.active` file exists but `projects/` has content → prompt user to select which project to activate
- Use existing GSD validation patterns for project directory structure (no new validation layer)

### Migration Behavior
- Migration triggers only when user runs `/gsd:new-project` (not on any GSD command)
- Always confirm before migration — show what will happen, ask before moving files
- Ask user what to name the migrated project (don't assume "default")
- Auto-commit the restructure with descriptive message after migration

### Claude's Discretion
- Error message verbosity — pick appropriate level per situation
- Path resolver architecture — reference doc vs inline in workflows
- How to distinguish shared paths (codebase/) from project-specific paths

</decisions>

<specifics>
## Specific Ideas

- Migration is a one-time operation that happens when someone wants multi-project for the first time
- The path resolver should integrate with existing GSD validation patterns, not create a parallel system

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation-migration*
*Context gathered: 2026-02-04*
