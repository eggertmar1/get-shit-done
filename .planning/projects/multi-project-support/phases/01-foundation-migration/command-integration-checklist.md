# Command Integration Checklist

**Purpose:** Track which GSD commands, workflows, and agents need path resolution integration for multi-project support.

**Status:** Created in Phase 1, executed in Phase 4 (Integration)

**Reference:** @get-shit-done/references/path-resolution.md, @get-shit-done/references/shared-paths.md

**Integration Priority:**
- **HIGH** - Used frequently, blocks core workflows
- **MEDIUM** - Used regularly, important for user experience
- **LOW** - Used occasionally, can defer

---

## Workflows

Workflows that read/write `.planning/` paths and need path resolution updates.

### [x] execute-plan.md

**Priority:** HIGH

**Files Read:**
- `config.json` (shared - for model_profile, commit_docs)
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/XX-name/*-PLAN.md` (project-specific)
- `phases/XX-name/*-SUMMARY.md` (project-specific)

**Files Write:**
- `STATE.md` (project-specific)
- `phases/XX-name/{phase}-{plan}-SUMMARY.md` (project-specific)
- `current-agent-id.txt` (project-specific)
- `agent-history.json` (project-specific)

**Integration Approach:**
1. Add path resolution to all STATE.md, ROADMAP.md references
2. Add path resolution to phases/ directory operations
3. Keep config.json resolution as shared (with per-project override merge)
4. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Every plan execution

---

### [x] resume-project.md

**Priority:** HIGH

**Files Read:**
- `STATE.md` (project-specific)
- `PROJECT.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/*/.continue-here*.md` (project-specific)
- `phases/*/*-PLAN.md` (project-specific)
- `phases/XX-name/*-CONTEXT.md` (project-specific)
- `current-agent-id.txt` (project-specific)

**Files Write:**
- None (read-only workflow)

**Integration Approach:**
1. Add path resolution for all project-specific file checks
2. Detect structure type early and route all reads
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Every resume operation

---

### [x] map-codebase.md

**Priority:** MEDIUM

**Files Read:**
- `config.json` (shared - for model_profile)
- `codebase/` (shared - checking if exists)

**Files Write:**
- `codebase/STACK.md` (shared)
- `codebase/ARCHITECTURE.md` (shared)
- `codebase/COMPONENTS.md` (shared)
- `codebase/DATA_FLOW.md` (shared)
- `codebase/DEPENDENCIES.md` (shared)
- `codebase/TESTING.md` (shared)
- `codebase/DEPLOYMENT.md` (shared)

**Integration Approach:**
1. codebase/ is always shared - no path resolution needed
2. Always write to `.planning/codebase/` regardless of structure
3. config.json resolution for global settings

**Usage Frequency:** Once per project, occasionally refreshed

---

### [x] complete-milestone.md

**Priority:** MEDIUM

**Files Read:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/` (project-specific - scanning for SUMMARY.md files)
- `PROJECT.md` (project-specific)
- `config.json` (shared)

**Files Write:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)

**Integration Approach:**
1. Add path resolution for all project-specific paths
2. Phase scanning needs to route through active project
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** After each phase completion

---

### [x] transition.md

**Priority:** MEDIUM

**Files Read:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/XX-name/*-SUMMARY.md` (project-specific)

**Files Write:**
- `STATE.md` (project-specific)

**Integration Approach:**
1. Add path resolution for STATE.md and ROADMAP.md
2. Phase directory operations need resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** After each plan completion

---

### [x] execute-phase.md

**Priority:** HIGH

**Files Read:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/XX-name/` (project-specific - listing plans)

**Files Write:**
- None (orchestrates other workflows)

**Integration Approach:**
1. Add path resolution for reading state
2. Phase directory scanning needs resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Every phase execution

---

### [x] discovery-phase.md

**Priority:** MEDIUM

**Files Read:**
- `PROJECT.md` (project-specific)
- `config.json` (shared)

**Files Write:**
- `research/` artifacts (project-specific)

**Integration Approach:**
1. Add path resolution for PROJECT.md
2. Add path resolution for research/ directory creation
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Early in project lifecycle

---

### [x] discuss-phase.md

**Priority:** MEDIUM

**Files Read:**
- `PROJECT.md` (project-specific)
- `phases/XX-name/XX-RESEARCH.md` (project-specific)

**Files Write:**
- `phases/XX-name/XX-CONTEXT.md` (project-specific)

**Integration Approach:**
1. Add path resolution for PROJECT.md
2. Add path resolution for phases/ directory operations
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** During planning sessions

---

### [x] verify-phase.md

**Priority:** LOW

**Files Read:**
- `ROADMAP.md` (project-specific)
- `phases/XX-name/*-PLAN.md` (project-specific)

**Files Write:**
- None (verification only)

**Integration Approach:**
1. Add path resolution for ROADMAP.md
2. Add path resolution for phases/ directory access
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Occasional verification

---

### [x] verify-work.md

**Priority:** LOW

**Files Read:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/` (project-specific - scanning plans/summaries)

**Files Write:**
- None (verification only)

**Integration Approach:**
1. Add path resolution for all state/roadmap reads
2. Phase directory scanning needs resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Occasional verification

---

### [x] diagnose-issues.md

**Priority:** LOW

**Files Read:**
- `STATE.md` (project-specific)
- `phases/` (project-specific)

**Files Write:**
- None (diagnostic only)

**Integration Approach:**
1. Add path resolution for STATE.md
2. Add path resolution for phases/ scanning
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** When debugging issues

---

### [x] list-phase-assumptions.md

**Priority:** LOW

**Files Read:**
- `phases/XX-name/*-PLAN.md` (project-specific)

**Files Write:**
- None (read-only)

**Integration Approach:**
1. Add path resolution for phases/ directory
2. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Rare

---

## Agents

Agent specifications that reference `.planning/` paths.

### [x] gsd-executor.md

**Priority:** HIGH

**Files Read:**
- `STATE.md` (project-specific)
- `config.json` (shared)
- `phases/XX-name/{phase}-{plan}-PLAN.md` (project-specific)

**Files Write:**
- `phases/XX-name/{phase}-{plan}-SUMMARY.md` (project-specific)
- `STATE.md` (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. Add path resolution for all file operations
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Every plan execution

---

### [x] gsd-planner.md

**Priority:** HIGH

**Files Read:**
- `PROJECT.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/XX-name/XX-CONTEXT.md` (project-specific)
- `phases/XX-name/XX-RESEARCH.md` (project-specific)

**Files Write:**
- `phases/XX-name/{phase}-{plan}-PLAN.md` (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. Add path resolution for reading context/research
3. Add path resolution for writing PLAN.md
4. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Every plan creation

---

### [x] gsd-roadmapper.md

**Priority:** HIGH

**Files Read:**
- `PROJECT.md` (project-specific)
- `REQUIREMENTS.md` (project-specific)

**Files Write:**
- `ROADMAP.md` (project-specific)
- `STATE.md` (project-specific)
- `phases/` directories (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. Add path resolution for all reads/writes
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Project initialization

---

### [x] gsd-phase-researcher.md

**Priority:** MEDIUM

**Files Read:**
- `PROJECT.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `codebase/` (shared)

**Files Write:**
- `phases/XX-name/XX-RESEARCH.md` (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. Project/roadmap need resolution
3. codebase/ is always shared
4. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Per-phase research

---

### [x] gsd-project-researcher.md

**Priority:** MEDIUM

**Files Read:**
- `PROJECT.md` (project-specific)
- `codebase/` (shared)

**Files Write:**
- `research/` artifacts (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. PROJECT.md needs resolution
3. research/ directory needs resolution
4. codebase/ is always shared
5. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Early project lifecycle

---

### [x] gsd-research-synthesizer.md

**Priority:** MEDIUM

**Files Read:**
- `research/` artifacts (project-specific)
- `phases/XX-name/*-RESEARCH.md` (project-specific)

**Files Write:**
- Synthesized research documents (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. All research paths need resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Research synthesis

---

### [x] gsd-verifier.md

**Priority:** LOW

**Files Read:**
- `STATE.md` (project-specific)
- `ROADMAP.md` (project-specific)
- `phases/XX-name/*-SUMMARY.md` (project-specific)

**Files Write:**
- None (verification only)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. All reads need path resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Verification operations

---

### [x] gsd-plan-checker.md

**Priority:** LOW

**Files Read:**
- `phases/XX-name/*-PLAN.md` (project-specific)
- `ROADMAP.md` (project-specific)

**Files Write:**
- None (validation only)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. All reads need path resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Plan validation

---

### [x] gsd-integration-checker.md

**Priority:** LOW

**Files Read:**
- `phases/` scanning (project-specific)
- `STATE.md` (project-specific)

**Files Write:**
- None (checking only)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. Phase scanning needs resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Integration checks

---

### [x] gsd-codebase-mapper.md

**Priority:** MEDIUM

**Files Read:**
- `codebase/` (shared - checking if exists)

**Files Write:**
- `codebase/*` files (shared)

**Integration Approach:**
1. No path resolution needed - codebase is always shared
2. Always write to `.planning/codebase/` root

**Usage Frequency:** Codebase mapping

---

### [x] gsd-debugger.md

**Priority:** LOW

**Files Read:**
- `STATE.md` (project-specific)
- `phases/` (project-specific)

**Files Write:**
- Diagnostic reports (project-specific)

**Integration Approach:**
1. Add @path-resolution.md to execution_context
2. All reads need path resolution
3. Add NO_ACTIVE_PROJECT error handling

**Usage Frequency:** Debugging sessions

---

## Commands

User-facing commands that need integration (Phase 2 will create new commands).

### New Commands (Created in Phase 2)

These commands will be created with path resolution built-in:

- `/gsd:new-project [name]` - Creates project in `projects/<name>/`, triggers migration if needed
- `/gsd:switch-project <name>` - Changes `.active` file
- `/gsd:list-projects` - Lists `projects/` directory
- `/gsd:archive-project [name]` - Deletes project folder

### Existing Commands (May Need Updates - TBD in Phase 2)

If any existing slash commands directly access `.planning/` paths, they'll be identified and updated in Phase 2.

---

## Integration Template

When updating a workflow/agent for path resolution:

```markdown
<execution_context>
@get-shit-done/references/path-resolution.md
@get-shit-done/references/shared-paths.md
[...existing context...]
</execution_context>

<process>
# Add at start of process section:

**Step 1: Resolve planning paths**

```bash
# Detect structure and resolve project-specific paths
if [ -d .planning/projects/ ]; then
  # Nested structure - need active project
  if [ ! -f .planning/.active ]; then
    echo "Error: No active project set."
    echo "Available projects:"
    ls -1 .planning/projects/
    echo ""
    echo "Select project with: /gsd:switch-project <name>"
    exit 1
  fi

  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')

  if [ ! -d ".planning/projects/$ACTIVE_PROJECT" ]; then
    echo "Error: Active project '$ACTIVE_PROJECT' not found."
    echo "Available projects:"
    ls -1 .planning/projects/
    exit 1
  fi

  # Set resolved paths
  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"
  STATE_FILE="$PROJECT_BASE/STATE.md"
  ROADMAP_FILE="$PROJECT_BASE/ROADMAP.md"
  PROJECT_FILE="$PROJECT_BASE/PROJECT.md"
  PHASES_DIR="$PROJECT_BASE/phases"
else
  # Flat structure - direct paths
  PROJECT_BASE=".planning"
  STATE_FILE="$PROJECT_BASE/STATE.md"
  ROADMAP_FILE="$PROJECT_BASE/ROADMAP.md"
  PROJECT_FILE="$PROJECT_BASE/PROJECT.md"
  PHASES_DIR="$PROJECT_BASE/phases"
fi

# Shared paths (always at root)
CODEBASE_DIR=".planning/codebase"
GLOBAL_CONFIG=".planning/config.json"
```

# Then use resolved paths throughout:
cat "$STATE_FILE"
cat "$PROJECT_FILE"
ls "$PHASES_DIR"/*/*-PLAN.md
```
</process>
```

---

## Validation Criteria

After integrating path resolution into a workflow/agent:

- [ ] Added @path-resolution.md to execution_context
- [ ] All hardcoded `.planning/` paths replaced with resolved variables
- [ ] Shared paths (codebase/, config.json) resolve to root
- [ ] Project-specific paths route through active project in nested structure
- [ ] NO_ACTIVE_PROJECT error handled with helpful message
- [ ] PROJECT_NOT_FOUND error handled
- [ ] Tested in both flat and nested structures
- [ ] Documentation updated to reflect path handling

---

## Completion Tracking

**Total items:** 24 (12 workflows + 11 agents + 1 command section)

**Completed:** 24

**Progress:** [██████████] 100%

Integration complete! All workflows, agents, and commands have been updated with path resolution support.

**Summary:**
- ✓ 12 workflows integrated (9 in Plan 02, 3 in Plan 01)
- ✓ 11 agents integrated (8 in Plan 02, 3 in Plan 01)
- ✓ 19+ command orchestrators integrated (Plan 02)

**Cross-runtime compatibility:** All integration snippets use POSIX-compliant bash (no bashisms). Works across Claude Code, OpenCode, and Gemini runtimes.

---

*Created: Phase 01 Plan 01*
*Last Updated: 2026-02-07*
*Owner: Phase 4 Integration Work - COMPLETE*
