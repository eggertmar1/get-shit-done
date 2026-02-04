<overview>
Classification of `.planning/` paths as shared (repo-level) vs project-specific (per-workstream) for multi-project support.
</overview>

<core_principle>

**Shared files stay at root, project files nest under projects/<name>/.**

Shared files apply to the entire repository and don't vary per project. Project-specific files are unique to each workstream and isolate planning context.

</core_principle>

<quick_reference>

## Path Classification Table

| Path | Type | Location | Rationale |
|------|------|----------|-----------|
| `codebase/` | Shared | `.planning/codebase/` | Repo structure doesn't change per project |
| `config.json` | Both | Root = global, Project = overrides | Global defaults + per-project customization |
| `.gitignore` | Shared | `.planning/.gitignore` | Git rules apply to entire `.planning/` |
| `.active` | Shared | `.planning/.active` | Tracks active project (gitignored, local-only) |
| `projects/` | Shared | `.planning/projects/` | Container for all projects |
| `PROJECT.md` | Project | `projects/<name>/PROJECT.md` | Each project has unique vision |
| `REQUIREMENTS.md` | Project | `projects/<name>/REQUIREMENTS.md` | Each project has scoped requirements |
| `ROADMAP.md` | Project | `projects/<name>/ROADMAP.md` | Each project has independent phases |
| `STATE.md` | Project | `projects/<name>/STATE.md` | Each project tracks its own progress |
| `phases/` | Project | `projects/<name>/phases/` | Phase work is project-specific |
| `research/` | Project | `projects/<name>/research/` | Domain research scoped to project |
| `todos/` | Project | `projects/<name>/todos/` | Captured ideas unique to project |

</quick_reference>

<shared_paths>

## Shared Paths (Repo-Level)

These paths always resolve to `.planning/` root, regardless of active project.

### codebase/

**Location:** `.planning/codebase/`

**Purpose:** Repository structure analysis and architectural documentation

**Contents:**
- `STACK.md` - Technology stack across entire repo
- `ARCHITECTURE.md` - System-wide architecture patterns
- `COMPONENTS.md` - Shared component inventory
- `DATA_FLOW.md` - Data flow across system
- `DEPENDENCIES.md` - External dependencies
- `TESTING.md` - Testing infrastructure
- `DEPLOYMENT.md` - Deployment configuration

**Why shared:** Codebase structure is repository-level - it doesn't change based on which project is active. Multiple projects work on the same codebase.

**Migration behavior:** Stays at root during migration

---

### config.json (Global)

**Location:** `.planning/config.json`

**Purpose:** Global GSD configuration defaults

**Contents:**
```json
{
  "mode": "yolo",
  "depth": "quick",
  "parallelization": true,
  "commit_docs": true,
  "model_profile": "balanced",
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}
```

**Why shared:** Default settings apply across all projects unless overridden

**Migration behavior:** Original `config.json` becomes global config (stays at root)

**Note:** Projects can override via per-project `config.json` (see project-specific section)

---

### .gitignore

**Location:** `.planning/.gitignore`

**Purpose:** Git ignore rules for `.planning/` directory

**Contents:**
```
.active
.DS_Store
*.tmp
```

**Why shared:** Git rules apply to entire `.planning/` directory tree

**Migration behavior:** Stays at root during migration, updated to add `.active`

---

### .active

**Location:** `.planning/.active`

**Purpose:** Tracks currently active project (gitignored, machine-local)

**Contents:** Single line with project name (e.g., `auth-refactor`)

**Why shared:** Active project selection is per-user/per-machine, not per-project

**Gitignored:** YES - prevents merge conflicts when team members have different active projects

**Migration behavior:** Created during migration, immediately gitignored

---

### projects/

**Location:** `.planning/projects/`

**Purpose:** Container directory for all project subdirectories

**Contents:** Subdirectories for each project (e.g., `default/`, `auth-refactor/`)

**Why shared:** Directory structure organizing all projects

**Migration behavior:** Created during migration

</shared_paths>

<project_specific_paths>

## Project-Specific Paths (Per-Workstream)

These paths resolve to `.planning/projects/<name>/` when nested structure active.

### PROJECT.md

**Location:** `projects/<name>/PROJECT.md`

**Purpose:** Project vision, scope, requirements, and context

**Contents:**
- What This Is
- Core Value
- Requirements (Validated, Active, Out of Scope)
- Context (structure diagrams, commands affected)
- Constraints
- Key Decisions

**Why project-specific:** Each workstream has unique goals and scope

**Migration behavior:** Moved from `.planning/PROJECT.md` to `projects/<name>/PROJECT.md`

---

### REQUIREMENTS.md

**Location:** `projects/<name>/REQUIREMENTS.md`

**Purpose:** Detailed requirements discovery and scoping

**Contents:**
- User stories
- Feature requirements
- Technical constraints
- Dependencies

**Why project-specific:** Requirements are scoped to project goals

**Migration behavior:** Moved from `.planning/REQUIREMENTS.md` if exists

---

### ROADMAP.md

**Location:** `projects/<name>/ROADMAP.md`

**Purpose:** Phase breakdown and project structure

**Contents:**
- Phase list with objectives
- Phase dependencies
- Success criteria per phase

**Why project-specific:** Each project has independent phase structure

**Migration behavior:** Moved from `.planning/ROADMAP.md` to `projects/<name>/ROADMAP.md`

---

### STATE.md

**Location:** `projects/<name>/STATE.md`

**Purpose:** Project memory, position tracking, and accumulated context

**Contents:**
- Current Position (phase, plan, progress)
- Performance Metrics (velocity, duration)
- Accumulated Context (decisions, todos, blockers)
- Session Continuity (last session, resume point)

**Why project-specific:** Each project tracks its own progress independently

**Migration behavior:** Moved from `.planning/STATE.md` to `projects/<name>/STATE.md`

---

### config.json (Per-Project)

**Location:** `projects/<name>/config.json`

**Purpose:** Project-specific configuration overrides

**Contents:** Same structure as global config, overrides specific fields

**Example:**
```json
{
  "mode": "careful",
  "model_profile": "thorough"
}
```

**Why project-specific:** Some projects need different settings (e.g., critical path needs "careful" mode)

**Migration behavior:** Copy of original `.planning/config.json` becomes per-project config

**Resolution:** Project config merged over global config (project settings win)

---

### phases/

**Location:** `projects/<name>/phases/`

**Purpose:** Phase-specific planning artifacts

**Contents:**
- Phase directories (e.g., `01-foundation/`, `02-auth/`)
- PLAN.md files
- SUMMARY.md files
- RESEARCH.md, CONTEXT.md files
- Other phase artifacts

**Why project-specific:** Phase work is unique to each project's workstream

**Migration behavior:** Moved from `.planning/phases/` to `projects/<name>/phases/`

---

### research/

**Location:** `projects/<name>/research/`

**Purpose:** Domain research and discovery documents

**Contents:**
- Technology research
- Pattern investigations
- Proof of concept findings

**Why project-specific:** Research is scoped to project domain

**Migration behavior:** Moved from `.planning/research/` if exists

---

### todos/

**Location:** `projects/<name>/todos/`

**Purpose:** Captured ideas and future work

**Contents:**
- `pending/` - Ideas not yet planned
- `backlog/` - Deprioritized work
- Other todo categories

**Why project-specific:** Todos are scoped to project goals

**Migration behavior:** Moved from `.planning/todos/` if exists

</project_specific_paths>

<rationale_deep_dive>

## Why This Classification?

### Shared: Codebase Analysis

**Rationale:** Multiple projects work on the same repository. The codebase structure (files, architecture, stack) doesn't change based on which project is active.

**Example:** Two projects:
- Project A: Adding authentication
- Project B: Refactoring API

Both projects work on the same codebase. They should see the same `STACK.md`, `ARCHITECTURE.md`, etc.

**Anti-pattern:** If codebase maps were per-project, each project would have duplicate or divergent documentation of the same code.

---

### Shared: Global Config

**Rationale:** Most configuration is consistent across projects (mode, model profile, workflow flags). Per-project config would be mostly duplicates.

**Solution:** Global config provides defaults, per-project config overrides specific settings when needed.

**Example:**
- Global: `{"mode": "yolo", "model_profile": "balanced"}`
- Project A: Uses defaults
- Project B: `{"mode": "careful"}` (overrides mode, inherits model_profile)

---

### Project-Specific: Planning Documents

**Rationale:** Each project has unique vision, requirements, phases, and progress. These must be isolated to prevent interference between workstreams.

**Example:**
- Project A: Authentication refactor (Phase 1: JWT, Phase 2: OAuth)
- Project B: API redesign (Phase 1: Schema, Phase 2: Endpoints)

If `ROADMAP.md` were shared, both projects would conflict. Isolation prevents crosstalk.

---

### Project-Specific: State Tracking

**Rationale:** Each project progresses independently. STATE.md tracks position, velocity, decisions - all unique to the workstream.

**Example:**
- Project A: Phase 2, Plan 3, 67% complete
- Project B: Phase 1, Plan 1, 15% complete

Separate STATE.md files prevent one project's progress from overwriting another's.

</rationale_deep_dive>

<migration_impact>

## Migration Path Classification

What happens to each path during migration from flat to nested structure:

### Stays at Root (Shared)

- `codebase/` → `.planning/codebase/`
- `config.json` → `.planning/config.json` (becomes global)
- `.gitignore` → `.planning/.gitignore` (updated to ignore `.active`)

### Moves to Project (Project-Specific)

- `PROJECT.md` → `projects/<name>/PROJECT.md`
- `REQUIREMENTS.md` → `projects/<name>/REQUIREMENTS.md`
- `ROADMAP.md` → `projects/<name>/ROADMAP.md`
- `STATE.md` → `projects/<name>/STATE.md`
- `phases/` → `projects/<name>/phases/`
- `research/` → `projects/<name>/research/`
- `todos/` → `projects/<name>/todos/`

### Created New (New Structure)

- `projects/` → `.planning/projects/` (directory created)
- `.active` → `.planning/.active` (file created, gitignored)
- `projects/<name>/config.json` → Copy of original config.json

### Git Operations

All moves use `git mv` to preserve history:
```bash
git mv .planning/PROJECT.md .planning/projects/<name>/PROJECT.md
git mv .planning/phases .planning/projects/<name>/phases
```

See migration workflow (Phase 2) for complete procedure.

</migration_impact>

<usage_in_resolver>

## How Path Resolver Uses This

The path resolution algorithm (see @path-resolution.md) checks path classification:

```bash
# Step 1: Check if shared path
if [[ "$PATH" == codebase/* ]] || [[ "$PATH" == "config.json" ]]; then
  # Shared - always at root
  RESOLVED=".planning/$PATH"
else
  # Project-specific - route through active project
  if [ -d .planning/projects/ ]; then
    ACTIVE=$(cat .planning/.active | tr -d '[:space:]')
    RESOLVED=".planning/projects/$ACTIVE/$PATH"
  else
    RESOLVED=".planning/$PATH"
  fi
fi
```

**Key insight:** Shared path check happens FIRST, before structure detection. This ensures codebase/ always resolves to root even in nested structures.

</usage_in_resolver>

<edge_cases>

## Edge Cases and Special Considerations

### config.json Resolution

**Challenge:** config.json exists in two places (global + per-project)

**Resolution strategy:**
1. Read global config: `.planning/config.json`
2. If nested structure, read project config: `projects/<name>/config.json`
3. Merge project config over global (project settings override)

**Example:**
```bash
# Read global config
GLOBAL_CONFIG=$(cat .planning/config.json)

# Read project config if nested
if [ -d .planning/projects/ ]; then
  ACTIVE=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE/config.json" 2>/dev/null || echo "{}")
  # Merge configs (implementation depends on language)
fi
```

---

### .gitignore Scope

**Location:** `.planning/.gitignore`

**Scope:** Applies to entire `.planning/` tree, including all projects

**Contents:** Machine-local files that shouldn't be committed
- `.active` (per-user active project)
- `.DS_Store` (macOS)
- `*.tmp` (temporary files)

**Note:** Per-project gitignore NOT supported - use single shared .gitignore

---

### New Path Categories

**When adding new paths to .planning/:**

1. Classify as shared or project-specific using rationale:
   - Shared: Applies to entire repo, doesn't vary per project
   - Project-specific: Unique to each workstream

2. Update this reference document

3. Update path resolver if needed (only if new shared path category)

4. Document in migration workflow (Phase 2)

</edge_cases>

<validation_checklist>

## Validating Path Classification

When uncertain if a path should be shared or project-specific:

- [ ] **Repo-level?** Does this apply to the entire repository regardless of active project?
  - YES → Shared
  - NO → Continue

- [ ] **Varies per workstream?** Does each project need its own version of this?
  - YES → Project-specific
  - NO → Continue

- [ ] **Configuration?** Is this a setting or config value?
  - Global default → Shared (global config.json)
  - Project-specific override → Project-specific (per-project config.json)

- [ ] **Codebase analysis?** Is this documenting the actual code structure?
  - YES → Shared (goes in codebase/)
  - NO → Continue

- [ ] **Planning artifact?** Is this part of the GSD planning workflow?
  - YES → Project-specific (phases/, research/, todos/)
  - NO → Continue

**When still uncertain:** Default to project-specific. Shared paths require strong justification (must be truly repo-level).

</validation_checklist>

<related_references>

## Related Documentation

- **@path-resolution.md** - Algorithm for resolving paths using this classification
- **@git-integration.md** - How git commits work in multi-project structure
- **Migration workflow** - Phase 2 handles actual file moves

</related_references>

---

*Reference created: Phase 01 Plan 01*
*Valid for: All GSD path-dependent operations*
*Update frequency: When new path categories added*
