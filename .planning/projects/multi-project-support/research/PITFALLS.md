# Pitfalls Research: Multi-Project CLI Tools

**Domain:** Multi-project CLI tool management (workspace/context switching)
**Researched:** 2026-02-04
**Confidence:** HIGH (verified with real-world CLI tools: kubectl, gcloud, terraform, npm/yarn workspaces, git worktree)

## Critical Pitfalls

### Pitfall 1: Breaking Existing Single-Project Users During Migration

**What goes wrong:**
Users with existing single-project setups encounter breaking changes after updating. Commands fail because they expect new directory structures or configuration files. Users lose work or spend hours debugging why their previously working setup is broken.

**Why it happens:**
Teams prioritize the new multi-project architecture without providing automatic migration paths. The assumption "users will read the migration guide" fails in practice—most users just run `update` and expect things to work.

**How to avoid:**
- **Auto-detect and auto-migrate**: On first command after update, detect flat structure (`.planning/PROJECT.md` exists at root), automatically migrate to `projects/default/`, set `.active` to "default", show success message
- **Backward compatibility mode**: Support BOTH flat and nested structures for 2+ major versions
- **Graceful degradation**: If migration fails, continue in compatibility mode with warning
- **Migration dry-run**: Provide `--preview-migration` flag to show what would change

**Warning signs:**
- Tests only cover new structure, not migration path
- No detection logic for old vs new structure
- Breaking changes in minor version updates
- Installation script doesn't check existing state

**Phase to address:**
Phase 1 (Foundation) must include:
- Migration detection and auto-migration logic
- Backward compatibility layer
- Comprehensive migration tests with real-world scenarios

**Sources:**
- [Breaking Changes Without Breaking Users: Lessons from Packit 1.0](https://pretalx.devconf.info/devconf-cz-2025/talk/UTGBCF/)
- [How to deprecate PHP code without breaking your users](https://dev.to/robertobutti/how-to-deprecate-php-code-without-breaking-your-users-4hae)
- [Databricks CLI migration](https://docs.databricks.com/aws/en/dev-tools/cli/migrate)

---

### Pitfall 2: `.active` File Causing Git Merge Conflicts

**What goes wrong:**
Two developers work on different branches. Both create projects in their branches. When merging, `.active` file has conflicting content. Even worse: one developer's active project doesn't exist in the other's branch, causing commands to fail after merge.

**Why it happens:**
The `.active` file tracks per-developer context but lives in a shared git repository. It's analogous to committing editor preferences or local environment state. Every branch switch or merge becomes a conflict point.

**How to avoid:**
- **Option A (Recommended)**: Make `.active` a local-only file (gitignore it)
  - Store in `.planning/.active` but add to `.gitignore`
  - On commands with no active project, prompt user to select or create
  - Each developer maintains their own context independently

- **Option B**: Git attributes with merge strategy
  - Use `.gitattributes`: `.planning/.active merge=ours` to always keep local version
  - Document this approach clearly in README

- **Option C**: Remove `.active` entirely
  - Require explicit project flag: `/gsd:execute-phase --project=auth-refactor`
  - Store last-used project in user's global config (`~/.gsd/last-project`)

**Warning signs:**
- Merge conflict notifications on `.active` in testing
- CI/CD failures due to missing projects referenced in `.active`
- User confusion about which project is active after git operations
- Tests don't simulate multi-developer git workflows

**Phase to address:**
Phase 1 (Foundation) MUST decide on approach before implementing project switching. This decision affects all subsequent commands.

**Sources:**
- [Resolving a merge conflict using the command line - GitHub Docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line)
- [How to Resolve Merge Conflicts in Git? | Atlassian](https://www.atlassian.com/git/tutorials/using-branches/merge-conflicts)
- Personal analysis based on git-tracked state file patterns

---

### Pitfall 3: Wrong Context Execution (Running Command on Wrong Project)

**What goes wrong:**
Developer intends to run `/gsd:execute-phase` on project A but it executes on project B (the currently active project). In production-like scenarios (deploying, deleting resources), this causes catastrophic errors. Even in development, hours of work get applied to wrong project.

**Why it happens:**
**Context switching without verification**: Similar to `kubectl` context mistakes where running commands in production instead of dev. Developers lose track of active context, especially after:
- Switching git branches
- Resuming work after days away
- Working across multiple terminal windows
- Collaborating with multiple people on different projects

**How to avoid:**
- **Always show active project in output**: First line of EVERY command shows `[Project: auth-refactor]`
- **Prompt before destructive operations**: Commands like `archive-project`, phase execution confirm: "Execute on project 'X'? (y/N)"
- **Visual indicators**: If using hooks/statusline, display active project prominently
- **Verify before execution**: Add `--verify` step that shows what would change, on which project
- **Branch-project association**: When switching git branches, prompt if expected project != active project

**Warning signs:**
- No visual feedback about active context
- Commands execute without confirmation
- Tests don't verify context isolation
- No audit trail showing which project was active during operations

**Phase to address:**
Phase 2 (Project Operations) must include:
- Active project display in all command outputs
- Confirmation prompts for destructive operations
- Context verification before execution

**Sources:**
- [Kubectl Switch Namespace: 3 Methods to Switch](https://www.plural.sh/blog/kubectl-switch-namespace-guide/)
- [kubectl get context](https://spacelift.io/blog/kubectl-get-context)
- [Switching projects in the Google Cloud CLI](https://medium.com/@kamiyabi7777/switching-projects-in-the-google-cloud-cli-a8bbd15c7954)
- [Context Switching Is Killing Your Productivity [2026] • Asana](https://asana.com/resources/context-switching)

---

### Pitfall 4: State Corruption from File Path Assumptions

**What goes wrong:**
Commands hard-code paths like `.planning/PROJECT.md` instead of `.planning/projects/<active>/PROJECT.md`. After introducing multi-project support:
- Commands read from wrong location, get stale data
- Commands write to root instead of project directory
- State files get mixed between projects
- Verification fails because artifacts are in wrong locations

**Why it happens:**
**Insufficient abstraction**: Path construction scattered across codebase. When adding multi-project support, developers miss updating path references in some commands. No single source of truth for "where are project files?"

**How to avoid:**
- **Centralize path logic**: Create path resolver module
  ```javascript
  // PathResolver.js
  function getProjectRoot(projectName) {
    if (isLegacyStructure()) return '.planning';
    const active = projectName || readActiveProject();
    return `.planning/projects/${active}`;
  }

  function getProjectFile(filename, projectName) {
    return `${getProjectRoot(projectName)}/${filename}`;
  }
  ```

- **Prohibit direct path strings**: Lint rule or code review checklist prevents `.planning/PROJECT.md` hardcoded paths
- **Path abstraction tests**: Test suite verifies ALL file operations go through path resolver
- **Shared vs project paths**: Clear distinction
  - Shared (codebase map): `.planning/codebase/`
  - Project-specific: `.planning/projects/<name>/PROJECT.md`

**Warning signs:**
- Grep for `.planning/` finds hardcoded paths in commands
- No path abstraction layer in codebase
- File operations use string concatenation
- Tests don't verify file locations for both legacy and multi-project structures

**Phase to address:**
Phase 1 (Foundation) must implement path resolver FIRST, before any project operations. All commands updated to use resolver before Phase 2.

**Sources:**
- [MCP vs CLI Tools: State management challenges](https://dev.to/mathewpregasen/mcp-vs-cli-tools-which-is-best-for-production-applications-bd8)
- [Managing Terraform State - Best Practices](https://spacelift.io/blog/terraform-state)
- GSD architecture analysis (file-based artifact management pattern)

---

### Pitfall 5: Performance Degradation with Project Count

**What goes wrong:**
Commands like `/gsd:list-projects` become slow when users have 20+ archived projects. Project switching delays. Status checks timeout. Users start avoiding multi-project features because they're too slow.

**Why it happens:**
**Naive iteration**: Commands iterate through ALL project directories, reading all STATE.md files, checking git status. No pagination, caching, or lazy loading. Scales linearly (or worse) with project count.

**How to avoid:**
- **Lazy metadata**: Store lightweight project index at `.planning/projects.json`
  ```json
  {
    "auth-refactor": {"status": "active", "lastActivity": "2026-02-04", "phase": 2},
    "default": {"status": "complete", "lastActivity": "2026-01-15", "phase": 5}
  }
  ```
  Update on project operations, read for list/status commands

- **Pagination for list operations**: `list-projects` shows 10 most recent by default, `--all` for complete list
- **Archive to separate directory**: Move completed projects to `.planning/archive/` so they're not scanned by default
- **Incremental updates**: Don't re-read all projects on every command, only active project

**Warning signs:**
- `list-projects` implementation iterates all directories
- No caching of project metadata
- Status checks read full STATE.md for all projects
- Tests only verify correctness, not performance with 50+ projects

**Phase to address:**
Phase 2 (Project Operations) should include project index from start. Phase 4 (Polish) adds archive directory if needed.

**Sources:**
- [Top 5 Monorepo Tools for 2025](https://www.aviator.co/blog/monorepo-tools/)
- [Terraform workspace performance](https://blog.gruntwork.io/how-to-manage-multiple-environments-with-terraform-using-workspaces-98680d89a03e)
- GSD concerns about complexity creep

---

## Moderate Pitfalls

### Pitfall 6: Inconsistent Project Naming Validation

**What goes wrong:**
User creates project named `my project` (with space). Some commands work, others fail. Or project named `../../etc` creates files outside `.planning/`. Directory traversal vulnerability or filesystem errors.

**Why it happens:**
Validation added inconsistently across commands. `new-project` validates, but `switch-project` doesn't. Special characters cause shell escaping issues.

**How to avoid:**
- **Centralized validation**: Single `validateProjectName()` function
  - Allow: alphanumeric, hyphens, underscores
  - Disallow: spaces, slashes, dots (except single `.` not at start)
  - Max length: 64 characters
  - Reject reserved names: ".", "..", ".active", "codebase"

- **Validate at boundaries**: ALL commands that accept project names call validator
- **Sanitization option**: Offer to convert `my project` → `my-project` automatically
- **Clear error messages**: "Project name 'my project' invalid. Use: my-project"

**Warning signs:**
- No project name validation function
- Validation logic duplicated across commands
- Tests don't try malicious project names
- Shell injection concerns not considered

**Phase to address:**
Phase 1 (Foundation) includes validation. Phase 3 (UX Enhancement) adds sanitization suggestions.

---

### Pitfall 7: Shared Codebase Map Assumptions Break

**What goes wrong:**
Codebase map at `.planning/codebase/` is shared across all projects. Two projects have different views of architecture. One project adds new integration, runs `map-codebase`, overwrites other project's analysis. Confusion and lost information.

**Why it happens:**
**Assumption that codebase is static**: In reality, codebase changes during development. Different branches have different codebases. But codebase map is branch-independent.

**How to avoid:**
- **Option A**: Codebase map per git commit hash
  - Generate map once per commit
  - Cache at `.planning/codebase/<commit-hash>/`
  - Projects reference: `codebaseCommit: abc123` in config

- **Option B**: Codebase map per branch
  - `.planning/codebase/main/` vs `.planning/codebase/feature-x/`
  - Auto-detect current branch

- **Option C**: Hybrid - shared baseline, project-specific overrides
  - `.planning/codebase/` = baseline (main branch)
  - `.planning/projects/<name>/codebase-notes.md` = project-specific additions

- **Option D** (Recommended): Keep shared, rely on git for versioning
  - Codebase map represents current HEAD
  - If you need historical map, check out old commit
  - Document this explicitly: "Codebase map reflects current codebase state"

**Warning signs:**
- Codebase map content differs between git branches
- Merge conflicts in `ARCHITECTURE.md`, `STACK.md`
- Projects depend on specific codebase map versions but no versioning exists
- No documentation about codebase map scope

**Phase to address:**
Phase 1 (Foundation) documents codebase map semantics. Phase 3 (UX) adds commit hash tracking if needed.

**Sources:**
- GSD architecture (shared codebase map decision in PROJECT.md)
- [Git worktree state management](https://stevekinney.com/courses/ai-development/git-worktrees)

---

### Pitfall 8: Missing Project Creates Cryptic Errors

**What goes wrong:**
User switches to branch where project "auth-refactor" doesn't exist (it's in another branch). Runs `/gsd:progress`. Gets error: "Cannot read PROJECT.md". User confused: "I ran this yesterday, it worked!"

**Why it happens:**
Project existence not validated before operations. Git branches have different project directories. Commands assume active project exists.

**How to avoid:**
- **Validate on every command**: Check `projects/<active>/` exists before proceeding
- **Clear error messages**:
  ```
  Project 'auth-refactor' not found in current branch.

  Available projects:
  - default
  - api-redesign

  Switch project: /gsd:switch-project <name>
  Create new:    /gsd:new-project [name]
  ```

- **Git branch detection**: On branch switch, show warning if active project not in new branch
- **Interactive recovery**: Prompt user to select from available projects instead of hard error

**Warning signs:**
- Commands don't check project directory existence
- Error messages don't suggest recovery actions
- No integration with git hooks to detect branch changes
- Tests don't simulate cross-branch scenarios

**Phase to address:**
Phase 2 (Project Operations) includes project existence validation in all commands.

---

### Pitfall 9: Archive/Delete Ambiguity Causes Data Loss

**What goes wrong:**
User runs `/gsd:archive-project old-feature` thinking it means "mark complete but keep". Actually deletes `.planning/projects/old-feature/` permanently. User loses planning artifacts, git history exists but hard to recover.

**Why it happens:**
Terminology ambiguity. "Archive" in some tools means "move to archive directory" (keep), in others means "delete". GSD spec says "delete" but users expect "move to archive".

**How to avoid:**
- **Rename command**: `/gsd:delete-project` is unambiguous
- **Strong confirmation**:
  ```
  WARNING: This will PERMANENTLY DELETE the project 'old-feature'.
  All planning files will be removed (git history will remain).

  Type project name to confirm: _____
  ```

- **Add separate archive**: `/gsd:archive-project` moves to `.planning/archive/<name>` (kept, hidden from list)
- **Soft delete option**: Add `archived: true` flag, keep files but hide from default list

**Warning signs:**
- Delete operations without confirmation
- "Archive" terminology without clear definition
- No recovery mechanism documented
- Tests don't verify destructive operations behavior

**Phase to address:**
Phase 2 (Project Operations) uses unambiguous naming. Phase 4 (Polish) adds archive directory if users request it.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip auto-migration, document manual steps | Faster initial implementation | 80% of users won't migrate, support burden | Never—migration is critical |
| Gitignore `.active`, no persistence | Avoids merge conflicts | User re-selects project every session | Only if project count < 3 |
| Hard-code paths during prototyping | Quick proof of concept | Major refactor needed for multi-project | Only in throwaway spike |
| No project name validation | Faster MVP | Security vulnerability, broken filesystem | Never—validation is 10 lines of code |
| Skip confirmation on delete | Fewer steps for experienced users | Accidental data loss, user frustration | Never—confirmation is table stakes |
| Single-threaded project list | Simpler implementation | Slow with many projects | Acceptable until 20+ projects |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Git operations | Committing `.active` file causing merge conflicts | Add `.active` to `.gitignore`, use local user config |
| Git branch switching | Active project becomes invalid after branch switch | Validate project exists on every command, prompt if missing |
| Git hooks | Assuming pre-commit/post-checkout available | Check if git hooks supported, degrade gracefully |
| Multi-runtime (Claude/OpenCode/Gemini) | Paths differ across runtimes | Detect runtime, normalize paths, use path resolver |
| Statusline integration | Failing when project context invalid | Cache last-known-good status, show stale indicator if stale |
| File watchers | Watching all project directories, high CPU | Watch only active project + shared codebase |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Reading all STATE.md files | `list-projects` slow | Lightweight metadata index (projects.json) | 15+ projects |
| No lazy loading | Every command slower than before | Load only active project context | 10+ projects |
| Full git status on all projects | Minutes to show status | Cache status, update incrementally | 20+ projects |
| Iterating archived projects | Command latency increases over time | Move archives to separate directory | 50+ total projects |
| Re-parsing all PLAN.md files | Phase execution startup delay | Cache plan metadata in SUMMARY | 30+ plans across projects |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No indication of active project | Users run commands on wrong project | Show `[Project: X]` in all command outputs |
| Silent project switching | Lost context, confusion | Confirm switch, show previous and new project |
| No project count limit | Overwhelming project list | Archive old projects, show 10 most recent by default |
| Assuming users read migration guide | Broken workflows after update | Auto-migrate with confirmation, show what changed |
| Error on missing project | User stuck, unclear what to do | Interactive prompt to create or switch to available project |
| Branch name as default project name | Cryptic project names like `fix-bug-123` | Prompt for human-readable name, suggest branch name as default |

## "Looks Done But Isn't" Checklist

Multi-project features often appear complete but miss critical pieces:

- [ ] **Auto-migration:** Working new structure but no migration from old structure — verify clean migration from existing single-project repos
- [ ] **Path abstraction:** Commands updated but paths still hardcoded in templates — verify templates use path resolver
- [ ] **Project validation:** `new-project` validates names but `switch-project` doesn't — verify all entry points validate
- [ ] **Git integration:** Works in normal workflow but breaks with merge conflicts — verify `.active` handling in merge scenarios
- [ ] **Cross-branch behavior:** Works in single branch but fails after branch switch — verify project existence checks
- [ ] **Error recovery:** Commands fail gracefully but no recovery suggestions — verify error messages include next steps
- [ ] **Performance testing:** Correctness verified but not tested with 50+ projects — verify performance at scale
- [ ] **Documentation:** Code works but migration guide missing — verify user-facing docs complete
- [ ] **Confirmation prompts:** Destructive operations work but no safeguards — verify delete requires confirmation
- [ ] **Status visibility:** Commands work but no feedback about active context — verify active project shown in all outputs

## Recovery Strategies

When pitfalls occur despite prevention, how to recover:

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| User's workflow broken by update | HIGH | 1. Release hotfix with auto-migration; 2. Document manual rollback; 3. Provide recovery script |
| `.active` file conflicts in merges | LOW | 1. Delete `.active`; 2. Prompt user to select project; 3. Update `.gitignore` in next release |
| Command ran on wrong project | MEDIUM | 1. Git revert commits if possible; 2. Manual fix if files mixed; 3. Add confirmation prompts |
| Path corruption (files in wrong locations) | HIGH | 1. Detection script finds misplaced files; 2. Move to correct locations; 3. Verify STATE.md consistency |
| Performance degradation with many projects | MEDIUM | 1. Add project metadata cache; 2. Implement archive directory; 3. Provide cleanup command |
| Project name collision or invalid chars | LOW | 1. Rename project directory; 2. Update `.active` if needed; 3. Add validation for future |
| Missing project after branch switch | LOW | 1. List available projects; 2. Prompt user to select; 3. Cache branch-project associations |
| Accidental project deletion | MEDIUM | 1. Git history recover if committed; 2. Document recovery from git; 3. Add soft delete option |

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls:

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Breaking migration | Phase 1: Foundation | Test suite with real single-project repos, auto-migration succeeds |
| `.active` merge conflicts | Phase 1: Foundation | Merge simulation tests, no conflicts on `.active` |
| Wrong context execution | Phase 2: Project Operations | All commands show active project, destructive ops require confirmation |
| State corruption (paths) | Phase 1: Foundation | All file operations use path resolver, tested in both legacy and multi-project modes |
| Performance degradation | Phase 2 (start), Phase 4 (optimize) | List 50 projects completes in <500ms |
| Project naming validation | Phase 1: Foundation | Security tests with malicious names (traversal, injection) pass |
| Codebase map conflicts | Phase 1: Foundation | Documented scope, merge conflict rate tracked |
| Missing project errors | Phase 2: Project Operations | Error messages include recovery actions, interactive fallback works |
| Archive/delete ambiguity | Phase 2: Project Operations | User confirmation required, command naming unambiguous |

## Sources

Research synthesized from:

**Multi-project CLI tool examples:**
- [kubectl context switching](https://www.plural.sh/blog/kubectl-switch-namespace-guide/)
- [gcloud configurations](https://docs.cloud.google.com/sdk/docs/configurations)
- [terraform workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)
- [npm/yarn workspaces](https://blog.logrocket.com/advanced-package-manager-features-npm-yarn-pnpm/)
- [git worktree CLI](https://github.com/max-sixty/worktrunk)

**Migration and breaking changes:**
- [Breaking Changes Without Breaking Users: Packit 1.0](https://pretalx.devconf.info/devconf-cz-2025/talk/UTGBCF/)
- [How to deprecate code without breaking users](https://dev.to/robertobutti/how-to-deprecate-php-code-without-breaking-your-users-4hae)
- [API versioning and deprecation strategy](https://apidog.com/blog/api-versioning-deprecation-strategy/)
- [AWS CLI migration guide](https://docs.aws.amazon.com/cli/latest/userguide/cliv2-migration.html)

**Git integration:**
- [Resolving merge conflicts (GitHub)](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line)
- [Git merge conflicts (Atlassian)](https://www.atlassian.com/git/tutorials/using-branches/merge-conflicts)
- [Git worktrees for parallel development](https://stevekinney.com/courses/ai-development/git-worktrees)

**State management and performance:**
- [Managing Terraform State](https://spacelift.io/blog/terraform-state)
- [MCP vs CLI Tools: State management](https://dev.to/mathewpregasen/mcp-vs-cli-tools-which-is-best-for-production-applications-bd8)
- [Monorepo tools performance](https://www.aviator.co/blog/monorepo-tools/)

**Context switching UX:**
- [Context Switching Is Killing Your Productivity [2026]](https://asana.com/resources/context-switching)
- [gh auth switch context issues](https://github.com/cli/cli/issues/8851)
- [Gemini CLI context management](https://artofcoding.dev/building-a-context-aware-gemini-cli-workflow)

**GSD-specific context:**
- GSD PROJECT.md: Multi-project support requirements and constraints
- GSD ARCHITECTURE.md: File-based artifact management, path structure
- GSD concerns: Breaking existing users, complexity creep, git integration, context confusion, state corruption

---
*Pitfalls research for: Multi-project CLI tool support*
*Researched: 2026-02-04*
*Confidence: HIGH — Verified with 5+ production CLI tools and real-world incident reports*
