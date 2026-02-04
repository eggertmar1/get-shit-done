# Architecture Research

**Domain:** Multi-project CLI tool directory structures
**Researched:** 2026-02-04
**Confidence:** HIGH

## Standard Architecture

### System Overview

```
.planning/                          ← Root configuration directory
├── .active                         ← Active project marker (file-based state)
├── config.json                     ← Global defaults (all projects)
├── codebase/                       ← Shared repo-level analysis
│   ├── STACK.md
│   ├── INTEGRATIONS.md
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── CONVENTIONS.md
│   ├── TESTING.md
│   └── CONCERNS.md
└── projects/                       ← Project namespace
    ├── default/                    ← Auto-migrated from flat structure
    │   ├── config.json             ← Project-specific overrides
    │   ├── PROJECT.md
    │   ├── REQUIREMENTS.md
    │   ├── ROADMAP.md
    │   ├── STATE.md
    │   ├── research/               ← Project-specific research
    │   └── phases/                 ← Phase working directories
    │       ├── 01-foundation/
    │       │   ├── 01-01-PLAN.md
    │       │   ├── 01-01-CONTEXT.md
    │       │   └── 01-01-SUMMARY.md
    │       └── 02-features/
    └── feature-auth/               ← Named project
        ├── config.json
        ├── PROJECT.md
        └── ...
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| `.active` | Track current project context | Plain text file containing project name |
| `config.json` (root) | Global defaults for all projects | JSON with mode, depth, workflow settings |
| `config.json` (project) | Project-specific overrides | Extends/overrides global config |
| `codebase/` | Shared repo-level analysis | Markdown files populated by `/gsd:map-codebase` |
| `projects/<name>/` | Isolated project context | Contains all project-specific artifacts |

## Recommended Project Structure

**Flat Structure (Current - Single Project):**
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

**Nested Structure (Target - Multi-Project):**
```
.planning/
├── .active
├── config.json
├── codebase/
└── projects/
    ├── default/
    │   ├── config.json
    │   ├── PROJECT.md
    │   ├── REQUIREMENTS.md
    │   ├── ROADMAP.md
    │   ├── STATE.md
    │   ├── research/
    │   └── phases/
    └── <project-name>/
        └── ...
```

### Structure Rationale

- **`.active` file:** Simple, git-trackable state without environment variables. Branch-based workflows (git worktree) can have different active projects per worktree.
- **Shared `codebase/`:** Repository structure doesn't change per project. All projects analyze the same code, avoiding duplication.
- **`projects/` namespace:** Clear separation between global config and project contexts. Predictable paths (`projects/<name>/PROJECT.md`).
- **`default/` migration target:** Existing flat structures auto-migrate to `projects/default/` for backwards compatibility.
- **Per-project `config.json`:** Override global settings without forking entire configuration.

## Architectural Patterns

### Pattern 1: Configuration Cascade

**What:** Three-level configuration hierarchy where settings cascade from global → project → runtime.

**When to use:** All multi-project systems need configuration inheritance to balance consistency with customization.

**Trade-offs:**
- Pro: Centralized defaults with project-specific overrides
- Pro: Changes to global config affect all projects
- Con: Debugging can require checking multiple config files
- Con: Precedence rules must be clearly documented

**Example:**
```typescript
// Global: .planning/config.json
{
  "mode": "yolo",
  "depth": "quick",
  "model_profile": "balanced"
}

// Project: .planning/projects/auth-refactor/config.json
{
  "mode": "plan",  // Override global
  "depth": "deep"   // Override global
  // model_profile: "balanced" inherited
}

// Resolution logic
const projectConfig = {
  ...globalConfig,
  ...projectConfigOverrides
}
```

**Real-world examples:**
- **pnpm workspaces:** Root `.npmrc` applies to all packages, package-level `.npmrc` overrides root ([pnpm settings](https://pnpm.io/settings))
- **ESLint monorepos:** Root `.eslintrc` with `root: true`, sub-folders extend base config ([ESLint in a Monorepo](https://gregory-gerard.dev/articles/eslint-in-a-monorepo))
- **TypeScript monorepos:** Base `tsconfig.json` extended by package-level configs with path overrides ([Managing TypeScript Packages](https://nx.dev/blog/managing-ts-packages-in-monorepos))
- **VS Code workspaces:** User → Workspace → Folder settings hierarchy ([User and workspace settings](https://code.visualstudio.com/docs/configure/settings))

### Pattern 2: Active Project Marker

**What:** A `.active` file containing the current project name, used by all commands to resolve project paths.

**When to use:** CLI tools that need to track context across command invocations without environment variables.

**Trade-offs:**
- Pro: Git-trackable (shared across team via commit)
- Pro: Per-worktree (git worktree allows different active projects per working directory)
- Pro: Simple to implement (read/write text file)
- Con: Requires explicit switching (not automatic like branch detection)
- Con: File must exist or commands need fallback behavior

**Example:**
```bash
# Read active project
ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null || echo "default")

# Resolve project path
PROJECT_PATH=".planning/projects/$ACTIVE_PROJECT"

# All commands use resolved path
cat "$PROJECT_PATH/PROJECT.md"
```

**Alternative: Environment variable approach:**
```bash
# User must export in shell
export GSD_ACTIVE_PROJECT="feature-auth"

# Commands read from env
PROJECT_PATH=".planning/projects/${GSD_ACTIVE_PROJECT:-default}"
```

**Why file-based wins for GSD:**
- Git worktree workflows: Each worktree can have different `.active` file
- Team coordination: Committing `.active` shares context
- No shell session state: Works across terminal sessions
- Simpler UX: No env var setup required

**Real-world examples:**
- **Terraform workspaces:** Store active workspace in `.terraform/environment` file ([Terraform workspaces](https://developer.hashicorp.com/terraform/cli/workspaces))
- **Git worktree:** Tracks active worktree in `.git/worktrees/<name>/` metadata ([Git worktree layout](https://gist.github.com/sellout/3361145fac9bf2dfdc6a9bc18dcdff36))
- **Node.js nvm:** `.nvmrc` file specifies Node version for directory ([dotfile management](https://data-wise.github.io/flow-cli/guides/DOTFILE-MANAGEMENT/))

### Pattern 3: Shared vs Per-Project Resources

**What:** Repository-level resources live at root (shared), project-specific resources live in project namespace (isolated).

**When to use:** Multi-project systems where some context is inherently shared (repo structure) and some is project-specific (planning artifacts).

**Trade-offs:**
- Pro: Avoids duplication of repo-level analysis
- Pro: Clear ownership boundaries
- Pro: Projects can't accidentally modify shared resources
- Con: Must decide what's shared vs isolated upfront
- Con: Shared resources can become stale if not refreshed

**Example:**
```
Shared (repository scope):
  .planning/codebase/          ← One repo = one structure
    ├── STACK.md               ← Technologies don't change per project
    ├── ARCHITECTURE.md        ← System design is shared
    └── STRUCTURE.md           ← File layout is shared

Per-Project (project scope):
  .planning/projects/auth-refactor/
    ├── PROJECT.md             ← Each project has unique goals
    ├── ROADMAP.md             ← Each project has unique phases
    ├── research/              ← Project-specific domain research
    └── phases/                ← Project-specific execution
```

**Decision framework:**

| Resource Type | Scope | Rationale |
|--------------|-------|-----------|
| Codebase analysis | Shared | Repo structure is constant |
| Tech stack documentation | Shared | Technologies used in repo are shared |
| Project goals (PROJECT.md) | Per-project | Each initiative has unique objectives |
| Roadmap & phases | Per-project | Work breakdown is project-specific |
| Research findings | Per-project | Domain research varies by feature |
| Config defaults | Shared | Consistency across projects |
| Config overrides | Per-project | Flexibility for special cases |

**Real-world examples:**
- **Docker Compose:** `docker-compose.yml` per project, shared volumes via `external: true` ([Docker volumes](https://docs.docker.com/reference/compose-file/volumes/))
- **pnpm workspaces:** Root `package.json` for workspace config, per-package `package.json` for dependencies ([pnpm workspaces](https://pnpm.io/workspaces))
- **Terraform:** Shared backend config, per-environment state files ([Terraform environments](https://spacelift.io/blog/terraform-environments))

## Data Flow

### Command Execution Flow

```
User runs: /gsd:plan-phase 1
    ↓
[Command] Read .planning/.active → "feature-auth"
    ↓
[Command] Resolve path: .planning/projects/feature-auth/
    ↓
[Command] Load context:
    - .planning/config.json (global)
    - .planning/projects/feature-auth/config.json (project)
    - .planning/projects/feature-auth/PROJECT.md
    - .planning/projects/feature-auth/ROADMAP.md
    - .planning/codebase/ (shared)
    ↓
[Command] Spawn gsd-planner agent with resolved paths
    ↓
[Agent] Write .planning/projects/feature-auth/phases/01-foundation/01-01-PLAN.md
    ↓
[Agent] Update .planning/projects/feature-auth/STATE.md
```

### Project Switching Flow

```
User runs: /gsd:switch-project auth-refactor
    ↓
[Command] Check if .planning/projects/auth-refactor/ exists
    ↓
    YES → Write "auth-refactor" to .planning/.active
    ↓
    NO → Error: "Project 'auth-refactor' not found. Run /gsd:list-projects"
    ↓
[Command] Confirm: "Switched to project 'auth-refactor'"
```

### Migration Flow (Flat → Nested)

```
User runs: /gsd:new-project (or any command on flat structure)
    ↓
[Command] Detect flat structure:
    - .planning/PROJECT.md exists (root level)
    - .planning/projects/ does NOT exist
    ↓
[Command] Auto-migrate:
    1. Create .planning/projects/default/
    2. Move artifacts:
       - PROJECT.md → projects/default/PROJECT.md
       - REQUIREMENTS.md → projects/default/REQUIREMENTS.md
       - ROADMAP.md → projects/default/ROADMAP.md
       - STATE.md → projects/default/STATE.md
       - research/ → projects/default/research/
       - phases/ → projects/default/phases/
    3. Keep codebase/ at root (shared)
    4. Keep config.json at root (becomes global)
    5. Write "default" to .planning/.active
    ↓
[Command] Commit migration:
    git add .planning/
    git commit -m "refactor: migrate to multi-project structure"
    ↓
[Command] Continue with original command
```

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1-3 projects | Flat structure acceptable, but nested prevents future pain |
| 4-10 projects | Nested essential, shared resources avoid duplication |
| 10+ projects | Consider project archival, list-projects filtering, cleanup automation |

### Scaling Priorities

1. **First bottleneck (4+ projects):** Flat structure becomes confusing. Users can't remember which files belong to which project. **Solution:** Nested structure with clear project namespaces.

2. **Second bottleneck (10+ projects):** `.active` switching becomes tedious. Users forget which project is active. **Solution:** Add project indicator to CLI prompt/status bar, auto-detect from git branch name.

3. **Third bottleneck (20+ projects):** Stale projects accumulate. **Solution:** Archive mechanism (move to `.planning/archive/<name>` or delete), list-projects shows last activity date.

## Anti-Patterns

### Anti-Pattern 1: Nested Projects

**What people do:** Allow projects inside projects (`projects/parent/child/`).

**Why it's wrong:**
- Infinite nesting complexity
- Config inheritance becomes ambiguous (parent → child or global → child?)
- Path resolution logic explodes
- User mental model breaks (is child independent or part of parent?)

**Do this instead:** Flat project namespace. Use naming conventions for grouping: `auth-login`, `auth-registration`, `auth-password-reset`.

### Anti-Pattern 2: Shared Phases Directory

**What people do:** Put phases at root level with project prefixes: `phases/auth-01-foundation/`, `phases/api-01-setup/`.

**Why it's wrong:**
- Violates encapsulation (phases are project-specific)
- Phase numbering collisions across projects
- Can't archive a project cleanly (phases scattered)
- Breaks "delete projects/<name> = delete project" invariant

**Do this instead:** Phases live in project namespace: `projects/auth/phases/01-foundation/`.

### Anti-Pattern 3: Per-Project Codebase Analysis

**What people do:** Copy `codebase/` into each project: `projects/auth/codebase/`, `projects/api/codebase/`.

**Why it's wrong:**
- Massive duplication (7 markdown files × N projects)
- Inconsistency (projects see different repo structure)
- Stale analysis (updating one project doesn't update others)
- Wasted tokens (agents load duplicate codebase context)

**Do this instead:** Shared `codebase/` at root. One `/gsd:map-codebase` updates all projects.

### Anti-Pattern 4: Environment Variables for Active Project

**What people do:** Use `export GSD_ACTIVE_PROJECT=auth` instead of `.active` file.

**Why it's wrong:**
- Breaks git worktree workflows (env vars are shell-session-scoped, not directory-scoped)
- Not git-trackable (team can't see which project others are working on)
- Requires shell setup (zshrc/bashrc configuration)
- Fragile (forget to set = cryptic errors)

**Do this instead:** File-based `.active` marker. Per-worktree, git-trackable, zero setup.

### Anti-Pattern 5: No Migration Path

**What people do:** Release multi-project feature without auto-migration, force users to manually restructure.

**Why it's wrong:**
- High friction adoption (users delay upgrading)
- Data loss risk (manual moves go wrong)
- Documentation burden (migration guide becomes critical path)
- Support burden (users need hand-holding)

**Do this instead:** Transparent auto-migration on first command after upgrade. Detect flat structure, migrate to `projects/default/`, commit, continue. User never edits files manually.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Git | Direct file operations | `.active` is git-trackable, project paths resolve normally |
| Git worktree | Per-worktree `.active` | Each worktree can have different active project |
| CI/CD | Explicit project selection | Set `.active` before GSD commands or pass `--project` flag |
| IDE status bars | Read `.active` for display | Show current project in statusline |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Commands ↔ Agents | Pass resolved project paths | Commands resolve, agents receive absolute paths |
| Global config ↔ Project config | Merge on load | `{...globalConfig, ...projectConfig}` |
| Shared codebase ↔ Projects | Read-only reference | Projects read, only `/gsd:map-codebase` writes |

## Migration Strategies

### Strategy 1: Transparent Auto-Migration (Recommended)

**Approach:** Detect flat structure on any command, auto-migrate to nested, continue execution.

**Steps:**
1. Command detects `.planning/PROJECT.md` at root + no `.planning/projects/`
2. Display: "Migrating to multi-project structure..."
3. Create `.planning/projects/default/`
4. Move artifacts (PROJECT.md, ROADMAP.md, etc.) to `projects/default/`
5. Keep `codebase/` and `config.json` at root
6. Write "default" to `.planning/.active`
7. Commit migration with clear message
8. Continue with original command

**Benefits:**
- Zero user action required
- Safe (git history preserves original)
- Rollback possible (git revert migration commit)
- Backwards compatible (existing scripts work after migration)

**Implementation:**
```bash
# In every command, before reading PROJECT.md:
if [ -f .planning/PROJECT.md ] && [ ! -d .planning/projects ]; then
  echo "━━━ Migrating to multi-project structure..."
  mkdir -p .planning/projects/default

  # Move project-specific artifacts
  mv .planning/PROJECT.md .planning/projects/default/
  mv .planning/REQUIREMENTS.md .planning/projects/default/ 2>/dev/null || true
  mv .planning/ROADMAP.md .planning/projects/default/ 2>/dev/null || true
  mv .planning/STATE.md .planning/projects/default/ 2>/dev/null || true
  mv .planning/research .planning/projects/default/ 2>/dev/null || true
  mv .planning/phases .planning/projects/default/ 2>/dev/null || true
  mv .planning/todos .planning/projects/default/ 2>/dev/null || true

  # Keep codebase/ and config.json at root (shared)

  # Set active project
  echo "default" > .planning/.active

  # Commit migration
  git add .planning/
  git commit -m "refactor: migrate to multi-project structure

Auto-migrated existing project to projects/default/. Codebase analysis
remains shared at .planning/codebase/. Active project: default."

  echo "✓ Migration complete. Continuing with command..."
fi
```

### Strategy 2: Opt-In Migration (Alternative)

**Approach:** Detect flat structure, prompt user to migrate or continue in legacy mode.

**Steps:**
1. Command detects flat structure
2. Display warning: "Multi-project support available. Migrate now?"
3. Options: "Migrate" / "Continue in single-project mode"
4. If migrate: Same as Strategy 1
5. If continue: Commands use legacy path resolution

**Benefits:**
- User control (no surprises)
- Gradual adoption (migrate when ready)

**Drawbacks:**
- Dual code paths (legacy + nested)
- Documentation burden (two modes to explain)
- Support burden (users confused about which mode)
- Technical debt (legacy path resolution forever)

**Verdict:** Avoid unless absolutely necessary. Transparent auto-migration is cleaner.

### Strategy 3: Breaking Change with Migration Script (Not Recommended)

**Approach:** Release v2.0 with breaking changes, provide separate migration script.

**Steps:**
1. Release notes: "Breaking: Run `gsd migrate` before upgrading"
2. User runs migration script manually
3. v2.0 commands fail on flat structure

**Drawbacks:**
- High friction (users must act before upgrading)
- Data loss risk (users forget to migrate)
- Support burden (users need help with script)

**Verdict:** Only acceptable if auto-migration is technically impossible.

### Backwards Compatibility Patterns

**Path Resolution Abstraction:**
```bash
# Function to resolve project path (handles both flat and nested)
resolve_project_path() {
  if [ -d .planning/projects ]; then
    # Nested structure
    ACTIVE=$(cat .planning/.active 2>/dev/null || echo "default")
    echo ".planning/projects/$ACTIVE"
  else
    # Legacy flat structure
    echo ".planning"
  fi
}

# All commands use function
PROJECT_PATH=$(resolve_project_path)
cat "$PROJECT_PATH/PROJECT.md"
```

**Config Merging:**
```bash
# Load global config
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo '{}')

# Load project config (if nested structure)
if [ -d .planning/projects ]; then
  ACTIVE=$(cat .planning/.active)
  PROJECT_CONFIG=$(cat .planning/projects/$ACTIVE/config.json 2>/dev/null || echo '{}')
else
  PROJECT_CONFIG='{}'
fi

# Merge (project overrides global)
MERGED_CONFIG=$(echo "$GLOBAL_CONFIG $PROJECT_CONFIG" | jq -s '.[0] * .[1]')
```

## Sources

**Configuration Hierarchy:**
- [pnpm Workspace Settings](https://pnpm.io/workspaces) - Workspace configuration cascade
- [ESLint in Monorepos](https://gregory-gerard.dev/articles/eslint-in-a-monorepo) - Root + project config patterns
- [Managing TypeScript Packages in Monorepos](https://nx.dev/blog/managing-ts-packages-in-monorepos) - Config extension patterns
- [VS Code User and Workspace Settings](https://code.visualstudio.com/docs/configure/settings) - Settings hierarchy

**Active Project Tracking:**
- [Terraform Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces) - File-based workspace tracking
- [Git Worktree Best Practices](https://gist.github.com/ChristopherA/4643b2f5e024578606b9cd5d2e6815cc) - Directory-per-branch patterns
- [Flow CLI Dotfile Management](https://data-wise.github.io/flow-cli/guides/DOTFILE-MANAGEMENT/) - Config file patterns

**Shared vs Per-Project Resources:**
- [Docker Compose Volumes](https://docs.docker.com/reference/compose-file/volumes/) - Shared vs project-specific volumes
- [Terraform Multiple Environments](https://spacelift.io/blog/terraform-environments) - Directory-per-environment approach
- [Complete Monorepo Guide](https://jsdev.space/complete-monorepo-guide/) - Workspace resource organization

**Migration Strategies:**
- [Migrating to Monorepo: Step-by-Step Guide](https://graphite.com/guides/migrating-to-monorepo-a-step-by-step-guide) - Migration planning
- [Backward Compatible Database Changes](https://planetscale.com/blog/backward-compatible-databases-changes) - Expand-migrate-contract pattern
- [Migrating Git to Monorepo Without Losing History](https://developers.netlify.com/guides/migrating-git-from-multirepo-to-monorepo-without-losing-history/) - History preservation

**Multi-Project Patterns:**
- [Claude Code Multi-Directory Support](https://apidog.com/blog/claude-code-multi-directory-support/) - Modern CLI multi-project workflows
- [GitHub Copilot CLI Enhanced Context Management](https://github.blog/changelog/2026-01-14-github-copilot-cli-enhanced-agents-context-management-and-new-ways-to-install/) - Project context switching
- [Git Worktree Tutorial](https://www.datacamp.com/tutorial/git-worktree-tutorial) - Multiple working directories

---
*Architecture research for: Multi-project CLI tool directory structures*
*Researched: 2026-02-04*
