# Project Research Summary

**Project:** Multi-Project Support for GSD CLI
**Domain:** CLI tool workspace/context management
**Researched:** 2026-02-04
**Confidence:** HIGH

## Executive Summary

GSD needs multi-project support to enable developers to work on parallel initiatives within a single repository without context conflicts. Research into production CLI tools (kubectl, terraform workspaces, git worktree, pnpm workspaces) reveals a clear architectural pattern: file-based active project tracking, directory-per-project isolation, and hierarchical configuration. The recommended approach uses a plain text `.active` file, nested project directories under `.planning/projects/`, and a shared codebase map to avoid duplication.

The critical implementation decision is whether to gitignore the `.active` file or commit it. Research on merge conflict patterns strongly suggests gitignoring it to prevent team coordination issues. Key technical risks include breaking existing single-project users during migration (requires transparent auto-migration), path corruption from hardcoded file assumptions (requires centralized path resolver), and wrong-context execution (requires prominent active project indicators). The zero-dependency philosophy must be maintained - all patterns use Node.js built-ins only.

This is a well-documented problem space with established patterns from mature CLI tools. Implementation is straightforward if migration and path abstraction are handled correctly in Phase 1 (Foundation). The main risk is insufficient testing of migration paths and cross-branch scenarios.

## Key Findings

### Recommended Stack

**Use file-based patterns with Node.js built-ins only.** Multi-project CLI tools universally adopt plain text markers for active context, directory isolation for state management, and hierarchical configuration. GSD should follow this standard with zero new dependencies.

**Core technologies:**
- **Plain text `.active` file**: Persistent active project marker (like kubeconfig's current-context) - Simple, git-aware if needed, zero parsing complexity
- **Directory-per-project isolation**: `projects/<name>/` subdirectories (like npm workspaces) - Clear boundaries, independent state, simple path resolution
- **Hierarchical configuration**: Global `.planning/config.json` + per-project overrides (like git config hierarchy) - Shared defaults with project flexibility
- **Node.js fs/path modules**: All file operations - No dependencies, cross-platform compatible

**Critical decision point**: The `.active` file should be **gitignored** (not committed) based on research into merge conflict patterns. Two developers working on different projects will create constant conflicts if `.active` is tracked. Instead, each developer maintains their own active context locally, and commands prompt for project selection when `.active` is missing.

### Expected Features

**Must have (table stakes):**
- Project creation with optional naming (`/gsd:new-project [name]`) - defaults to git branch name
- Project switching (`/gsd:switch-project <name>`) - explicit context change
- List all projects (`/gsd:list-projects`) - show available projects with status
- Active project tracking (`.active` file + CLI indicators) - user always knows context
- Isolated project directories (`projects/<name>/`) - full state separation
- Auto-migration from flat structure to `projects/default/` - backwards compatibility
- Archive/delete project functionality - cleanup old work
- Shared codebase map at root - avoid duplication across projects

**Should have (competitive):**
- Git branch auto-detection for default project names - reduces friction
- Project status metadata (last active, phase progress) - enhanced `/gsd:list-projects`
- Per-project config overrides - model profile, research depth customization
- Git worktree integration - detect worktrees, suggest project-per-worktree

**Defer (v2+):**
- Project templates - quick-start common patterns
- Fuzzy project search - interactive filtering for 10+ projects
- Project tagging/grouping - organize by team/epic/client
- Project analytics dashboard - aggregate metrics across projects

### Architecture Approach

**Adopt the Kubernetes config model**: `.active` file pointing to current project + nested project directories + hierarchical configuration. This pattern is proven across kubectl, terraform, git worktree, and monorepo tools. Keep codebase analysis shared at root (repository structure is constant), isolate planning artifacts per-project (goals/roadmaps vary).

**Major components:**
1. **Active Project Tracker** - `.active` file (gitignored) containing project name, validated on every command
2. **Path Resolver** - Centralized module translating logical paths to physical (`PROJECT.md` → `.planning/projects/<active>/PROJECT.md`)
3. **Config Merger** - Loads global config, overlays project-specific overrides, provides unified config object
4. **Migration Handler** - Auto-detects flat structure, moves artifacts to `projects/default/`, commits, continues execution
5. **Project Validator** - Enforces naming rules (alphanumeric/hyphens/underscores only), prevents traversal attacks

**Key architectural decision**: The path resolver must be implemented FIRST (Phase 1) before any project operations. All commands must go through the resolver - no hardcoded `.planning/PROJECT.md` paths. This prevents state corruption when multi-project support is added.

### Critical Pitfalls

1. **Breaking existing users during migration** - Transparent auto-migration is non-negotiable. Detect flat structure on any command, migrate to `projects/default/`, set `.active`, commit, continue. Never force manual migration. Test with real existing single-project repos.

2. **`.active` file causing merge conflicts** - If `.active` is committed, two developers on different projects will conflict on every merge. Solution: Add `.active` to `.gitignore`, prompt users to select project when missing. Each developer maintains local context independently.

3. **Wrong context execution (running on wrong project)** - Users lose track of active project, especially after git branch switches. Solution: Display `[Project: <name>]` in ALL command outputs first line, require confirmation for destructive operations, validate project exists before execution.

4. **State corruption from hardcoded paths** - Commands assume `.planning/PROJECT.md` instead of project-aware paths. Solution: Centralized path resolver module, lint rule preventing hardcoded paths, abstraction tests verifying all operations use resolver.

5. **Performance degradation with many projects** - Naive iteration through all project directories slows commands like `list-projects`. Solution: Lightweight metadata index (`projects.json`) updated on operations, lazy loading of project details, archive directory for completed projects.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation & Migration (CRITICAL PATH)
**Rationale:** Must establish path abstraction and migration before any project operations. Migration breakage is #1 risk from PITFALLS.md. Path resolver prevents state corruption.

**Delivers:**
- Path resolver module (centralized `getProjectPath()` function)
- Auto-migration logic (flat → `projects/default/`)
- Active project tracking (`.active` file handling, gitignore setup)
- Project name validation (prevent traversal, enforce naming rules)
- Backwards compatibility layer (support both flat and nested structures)

**Addresses:**
- Pitfall #1: Breaking existing users (auto-migration prevents breakage)
- Pitfall #4: State corruption from path assumptions (resolver abstracts all paths)
- Pitfall #6: Inconsistent naming validation (centralized validator)

**Must include:** Comprehensive migration tests with real single-project repos, cross-branch scenario testing, path resolver coverage for all file operations.

**Research needed:** NO - This is infrastructure setup, patterns are well-documented (kubectl, terraform, git worktree all solved this).

### Phase 2: Core Project Operations
**Rationale:** Once foundation is stable, implement user-facing project management. Features are table stakes from FEATURES.md.

**Delivers:**
- `/gsd:new-project [name]` - create project with auto-naming from git branch
- `/gsd:switch-project <name>` - change active context
- `/gsd:list-projects` - show available projects with basic status
- Project existence validation on all commands
- Active project display in command outputs (`[Project: X]`)

**Addresses:**
- Features: Project creation, switching, listing (all table stakes)
- Pitfall #3: Wrong context execution (always show active project)
- Pitfall #8: Missing project errors (validate before operations)

**Uses:** Path resolver from Phase 1, validator from Phase 1

**Research needed:** NO - Standard CRUD operations on directories, git branch integration is straightforward.

### Phase 3: Configuration & Context Intelligence
**Rationale:** After basic operations work, add configuration flexibility and git integration intelligence.

**Delivers:**
- Per-project `config.json` overrides (model profile, research depth)
- Config merging (global + project overrides)
- Git branch auto-detection for project naming
- Cross-branch project validation (warn if active project missing in new branch)

**Addresses:**
- Features: Per-project config, git branch integration (should-haves)
- Pitfall #7: Codebase map assumptions (document scope, add commit hash tracking if needed)

**Uses:** Config cascade pattern from ARCHITECTURE.md

**Research needed:** NO - Configuration merge is standard (git config, npm rc files, ESLint extends pattern).

### Phase 4: Project Lifecycle & Polish
**Rationale:** Complete the user experience with lifecycle management and performance optimization.

**Delivers:**
- `/gsd:archive-project <name>` - move to `.planning/archive/` directory
- `/gsd:delete-project <name>` - permanent deletion with confirmation
- Project metadata index (`projects.json`) for performance
- Enhanced `/gsd:list-projects` with last active, phase progress
- Confirmation prompts for destructive operations

**Addresses:**
- Features: Archive/delete (table stakes), project status metadata (should-have)
- Pitfall #5: Performance degradation (metadata index, lazy loading)
- Pitfall #9: Archive/delete ambiguity (separate commands, strong confirmation)

**Research needed:** NO - Standard file operations and UX patterns.

### Phase 5: Integration & Testing
**Rationale:** Verify multi-project works across all existing GSD commands and runtimes.

**Delivers:**
- Update all agent spawning to use path resolver
- Update template system for project-aware paths
- Update statusline.js for active project display
- Cross-runtime testing (Claude Code, OpenCode, Gemini CLI)
- Git worktree scenario testing

**Addresses:**
- Integration points from ARCHITECTURE.md
- Pitfall #2: `.active` merge conflicts (verify gitignore approach)

**Research needed:** NO - Integration testing and verification.

### Phase Ordering Rationale

- **Foundation first is non-negotiable**: Path abstraction and migration must precede all project operations. Otherwise, commands will have inconsistent path handling and existing users will break.

- **Operations before polish**: Get basic project management working before adding lifecycle features. Users need create/switch/list immediately, archive/delete can wait.

- **Configuration after operations**: Config overrides only matter once users have multiple projects. Deliver core functionality first.

- **Integration last**: Verify multi-project works across entire GSD system after core features are stable.

This ordering follows the "expand-migrate-contract" pattern from database migrations: Phase 1 expands (adds support for both structures), Phase 2-3 migrate (users adopt new structure), Phase 5 can later contract (remove legacy support if desired).

### Research Flags

**Phases needing NO additional research** (patterns well-documented):
- **Phase 1 (Foundation):** File operations, path resolution, migration patterns - all standard Node.js, proven by kubectl/terraform
- **Phase 2 (Operations):** CRUD on directories, git integration - straightforward implementation
- **Phase 3 (Configuration):** Config merging, hierarchical settings - established patterns (git, npm, eslint)
- **Phase 4 (Lifecycle):** File archival, metadata tracking - basic file operations
- **Phase 5 (Integration):** Testing and verification - no new concepts

**Overall assessment:** This entire project needs NO phase-specific research. The domain is well-understood with mature reference implementations. All research has been completed upfront. Focus execution on careful implementation and thorough testing, especially migration paths.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Zero dependencies, Node.js built-ins only. Pattern validated across kubectl, terraform, git worktree. File-based tracking is universal standard. |
| Features | HIGH | Comprehensive competitive analysis (git worktree, tmuxinator, pnpm workspaces, GitKraken). Table stakes clearly identified. MVP scope well-defined. |
| Architecture | HIGH | Kubernetes config model is proven pattern. Clear separation (shared codebase, isolated projects). Migration strategy tested in npm workspaces, Terraform. |
| Pitfalls | HIGH | Verified with real-world CLI tools and incident reports. Merge conflict patterns documented in GitHub/Atlassian guides. Performance traps from monorepo tools. |

**Overall confidence:** HIGH

All research sources are authoritative (official documentation for kubectl, terraform, git) or based on production CLI tools with millions of users. The problem space is mature with established patterns. No novel technical challenges - this is systems integration of proven patterns.

### Gaps to Address

**No significant gaps identified.** Research covered all critical areas with high-quality sources.

**Minor validation points during implementation:**
- `.active` gitignore decision: Currently recommended based on merge conflict research. Validate with team workflows during Phase 2. If teams prefer committed `.active` for coordination, can add `.gitattributes merge=ours` strategy.
- Migration commit message: Test whether auto-commit during migration is acceptable to users or requires opt-in confirmation.
- Project metadata format: `projects.json` structure can be refined based on Phase 4 performance testing.

These are implementation details, not research gaps. The architectural approach is sound.

## Roadmap Success Criteria

Based on research, the roadmap will succeed if:

1. **Zero breaking changes for existing users** - Auto-migration makes multi-project transparent
2. **Zero new dependencies** - Node.js built-ins only, maintaining GSD philosophy
3. **Clear active context** - Users always know which project is active
4. **Fast operations** - Project operations <100ms up to 20 projects
5. **Git-friendly** - No merge conflicts, works with worktrees, git-trackable state

## Sources

### Stack Research
- **Workspace Patterns:** Yarn Workspaces, Terraform CLI, pnpm Workspaces, Git Worktree (all official docs)
- **Context Switching:** kubectl config, kubeconfig file format, Kubernetes contexts (Kubernetes official docs)
- **Version Management:** asdf .tool-versions, direnv auto-loading (official tool docs)
- **Monorepo Tools:** Nx vs Turborepo, monorepo architecture guides

### Features Research
- **Multi-Project Tools:** Gemini CLI issue #4935, VS Code multi-root workspaces, IntelliJ IDEA workspaces
- **Git Workflows:** Git worktree docs, GitKraken workspaces, git worktree manager (gwq)
- **Session Management:** tmuxinator, tmux-sessionx (GitHub repos with 5k+ stars)
- **Context Switching:** Jellyfish, Graphite, Hatica productivity research on context switching impact

### Architecture Research
- **Configuration Hierarchy:** pnpm settings, ESLint in monorepos, TypeScript monorepo configs, VS Code settings
- **Active Project Tracking:** Terraform workspaces, git worktree best practices, dotfile management patterns
- **Migration Strategies:** Migrating to monorepo guides, backward-compatible database changes, git history preservation

### Pitfalls Research
- **Breaking Changes:** Packit 1.0 migration, PHP deprecation patterns, Databricks CLI migration, AWS CLI v2 migration
- **Merge Conflicts:** GitHub official docs, Atlassian git tutorials, git worktree state management
- **Context Errors:** kubectl context switching, gcloud configurations, Terraform workspace performance
- **State Management:** MCP vs CLI tools comparison, Terraform state best practices
- **Performance:** Monorepo tools comparison (Nx, Lerna, Turborepo, Bazel), Terraform workspace scalability

All sources are authoritative (official documentation) or production-tested (CLI tools with millions of users). Research synthesis cross-references 40+ sources across 4 research files.

---
*Research completed: 2026-02-04*
*Ready for roadmap: YES*
*Confidence: HIGH across all areas*
*Additional research needed: NONE - proceed directly to requirements and roadmap creation*
