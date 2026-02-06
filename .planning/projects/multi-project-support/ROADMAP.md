# Roadmap: Multi-Project Support

## Overview

This roadmap transforms GSD from single-project to multi-project by restructuring `.planning/` to support parallel workflows. Phase 1 establishes the foundation with transparent auto-migration and path abstraction. Phase 2 delivers project management commands. Phase 3 adds configuration flexibility and git intelligence. Phase 4 verifies system-wide integration across all GSD commands and runtimes.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Foundation & Migration** - Establish multi-project infrastructure with transparent backwards compatibility
- [ ] **Phase 2: Project Commands** - Deliver create, switch, list, archive operations
- [ ] **Phase 3: Configuration & Context** - Enable per-project config overrides and git intelligence
- [ ] **Phase 4: Integration** - Verify system-wide compatibility across all commands and runtimes

## Phase Details

### Phase 1: Foundation & Migration
**Goal**: Existing single-project repos continue working, multi-project structure is ready
**Depends on**: Nothing (first phase)
**Requirements**: STRUCT-01, STRUCT-02, STRUCT-03, STRUCT-04, STRUCT-05, FOUND-06
**Success Criteria** (what must be TRUE):
  1. Running `/gsd:new-project` on flat `.planning/` offers migration with user confirmation
  2. Path resolver correctly routes all file operations to active project directory
  3. Codebase map remains shared at `.planning/codebase/` across all projects
  4. `.active` file tracks current project and is gitignored to prevent merge conflicts
**Plans**: 2 plans

Plans:
- [x] 01-01: Create path resolution reference docs (path-resolution.md, shared-paths.md)
- [x] 01-02: Add migration workflow and detection to /gsd:new-project

### Phase 2: Project Commands
**Goal**: Users can create, switch, list, and delete projects
**Depends on**: Phase 1
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-07
**Success Criteria** (what must be TRUE):
  1. User can create new project with `/gsd:new-project [name]`, defaults to git branch
  2. User can switch between projects with `/gsd:switch-project <name>`
  3. User can see all projects and their status with `/gsd:list-projects`
  4. User can delete unused projects with `/gsd:archive-project [name]`
  5. Running GSD commands without active project prompts for selection
**Plans**: 3 plans

Plans:
- [x] 02-01: Implement new-project command with git branch defaults
- [x] 02-02: Implement switch-project and list-projects commands
- [x] 02-03: Implement archive-project and active project validation

### Phase 3: Configuration & Context
**Goal**: Projects have independent configuration with intelligent git integration
**Depends on**: Phase 2
**Requirements**: CONFIG-01, CONFIG-02, CONFIG-03
**Success Criteria** (what must be TRUE):
  1. Global config at `.planning/config.json` provides defaults for all projects
  2. Per-project config at `projects/<name>/config.json` overrides global settings
  3. Project creation auto-detects git branch name for project naming
**Plans**: 1 plan

Plans:
- [ ] 03-01: Build config merger with hierarchical override system

### Phase 4: Integration
**Goal**: Multi-project works across all GSD commands and runtimes
**Depends on**: Phase 3
**Requirements**: INTEG-01, INTEG-02, INTEG-03
**Success Criteria** (what must be TRUE):
  1. All existing `/gsd:*` commands work correctly with active project context
  2. `.active` file gitignore prevents merge conflicts across branches
  3. Multi-project functionality works in Claude Code, OpenCode, and Gemini runtimes
**Plans**: 2 plans

Plans:
- [ ] 04-01: Update all orchestrators and agents for project-aware paths
- [ ] 04-02: Cross-runtime testing and gitignore configuration

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Migration | 2/2 | Complete | 2026-02-04 |
| 2. Project Commands | 3/3 | Complete | 2026-02-06 |
| 3. Configuration & Context | 0/1 | Not started | - |
| 4. Integration | 0/2 | Not started | - |
