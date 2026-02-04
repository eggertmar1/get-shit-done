<overview>
Path resolution algorithm for routing GSD file operations between flat (legacy) and nested (multi-project) `.planning/` structures.
</overview>

<core_principle>

**Detect structure on every call, route accordingly.**

Always check whether `.planning/projects/` exists before resolving paths. Shared files (like `codebase/`) stay at root; project-specific files (like `PROJECT.md`) route through active project in nested structures.

</core_principle>

<when_to_use>

Use this resolver in:
- All GSD workflows that read/write `.planning/` files
- Commands that access PROJECT.md, STATE.md, ROADMAP.md, phases/
- Agents that create research/, todos/, or other project artifacts

Do NOT use for:
- Files outside `.planning/` directory
- Temporary files or system paths
- Git operations (use git commands directly)

</when_to_use>

<detection_logic>

## Structure Detection

Determine if repository uses flat (legacy) or nested (multi-project) structure:

```bash
# Check for nested structure
if [ -d .planning/projects/ ]; then
  # NESTED - multi-project structure active
  # Need to read .active file to determine project
else
  # FLAT - legacy single-project structure
  # Access files directly at .planning/ root
fi
```

**Structure types:**

| Type   | Indicator                      | Behavior                              |
|--------|--------------------------------|---------------------------------------|
| Flat   | No `.planning/projects/` dir   | All paths resolve to `.planning/`     |
| Nested | `.planning/projects/` exists   | Project paths resolve via `.active`   |

**Important:** Mixed states should never occur - migration is atomic per repository.

</detection_logic>

<resolution_algorithm>

## Path Resolution Algorithm

Pseudocode for resolving any `.planning/` relative path:

```
function resolvePlanningPath(relativePath):
  planningRoot = cwd + '/.planning'

  # STEP 1: Check if path is shared (always at root)
  if relativePath starts with 'codebase/' or equals 'config.json':
    return planningRoot + '/' + relativePath

  # STEP 2: Detect structure type
  projectsDir = planningRoot + '/projects'

  if exists(projectsDir):
    # NESTED STRUCTURE - route through active project

    # Read active project
    activeFile = planningRoot + '/.active'
    if not exists(activeFile):
      throw NO_ACTIVE_PROJECT

    activeProject = read(activeFile).trim()
    if empty(activeProject):
      throw EMPTY_ACTIVE_FILE

    # Verify project exists
    projectDir = projectsDir + '/' + activeProject
    if not exists(projectDir):
      throw PROJECT_NOT_FOUND(activeProject)

    # Resolve to project-specific path
    return projectDir + '/' + relativePath

  else:
    # FLAT STRUCTURE - direct path
    return planningRoot + '/' + relativePath
```

**Shared vs Project-Specific Paths:**

See @shared-paths.md for complete list of which paths are shared vs project-specific.

</resolution_algorithm>

<error_handling>

## Error States

Handle these errors when resolving paths:

### NO_ACTIVE_PROJECT

**Trigger:** `.planning/projects/` exists but `.active` file missing

**Resolution:**
1. List available projects from `.planning/projects/`
2. Prompt user: "No active project set. Select project:"
3. Write selected project to `.planning/.active`
4. Retry path resolution

**Example:**
```bash
# Detect error
if [ -d .planning/projects/ ] && [ ! -f .planning/.active ]; then
  echo "No active project set."
  echo "Available projects:"
  ls -1 .planning/projects/
  echo ""
  echo "Select project or run: gsd:switch-project <name>"
  exit 1
fi
```

### EMPTY_ACTIVE_FILE

**Trigger:** `.active` file exists but contains only whitespace

**Resolution:** Same as NO_ACTIVE_PROJECT - treat empty as missing

### PROJECT_NOT_FOUND

**Trigger:** `.active` references a project that doesn't exist in `projects/`

**Resolution:**
1. Show error: "Active project '{name}' not found"
2. List available projects
3. Prompt user to select valid project
4. Update `.active` with new selection

**Example:**
```bash
ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
if [ ! -d ".planning/projects/$ACTIVE_PROJECT" ]; then
  echo "Error: Active project '$ACTIVE_PROJECT' not found."
  echo "Available projects:"
  ls -1 .planning/projects/
  exit 1
fi
```

### PLANNING_NOT_INITIALIZED

**Trigger:** `.planning/` directory doesn't exist

**Resolution:** Direct user to `/gsd:new-project` to initialize

**Note:** This check happens BEFORE path resolution in command entry points.

</error_handling>

<usage_patterns>

## Usage in Commands and Workflows

### Pattern 1: Read Project-Specific File

```bash
# In workflow/command <process> section

# Resolve path
if [ -d .planning/projects/ ]; then
  # Nested structure
  if [ ! -f .planning/.active ]; then
    echo "Error: No active project set"
    exit 1
  fi
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_FILE=".planning/projects/$ACTIVE_PROJECT/PROJECT.md"
else
  # Flat structure
  PROJECT_FILE=".planning/PROJECT.md"
fi

# Read file
cat "$PROJECT_FILE"
```

### Pattern 2: Write Project-Specific File

```bash
# Resolve STATE.md path
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  STATE_FILE=".planning/projects/$ACTIVE_PROJECT/STATE.md"
else
  STATE_FILE=".planning/STATE.md"
fi

# Write file
echo "Updated content" > "$STATE_FILE"
```

### Pattern 3: Access Shared File (Always Root)

```bash
# Codebase map is always shared - no resolution needed
CODEBASE_DIR=".planning/codebase"

# Read shared file
cat "$CODEBASE_DIR/STACK.md"
```

### Pattern 4: List Phase Plans

```bash
# Resolve phases directory
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PHASES_DIR=".planning/projects/$ACTIVE_PROJECT/phases"
else
  PHASES_DIR=".planning/phases"
fi

# List plans
ls -1 "$PHASES_DIR"/*/*-PLAN.md 2>/dev/null
```

### Pattern 5: Create New Artifact Directory

```bash
# Resolve base path for new directory (e.g., todos/)
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  BASE_PATH=".planning/projects/$ACTIVE_PROJECT"
else
  BASE_PATH=".planning"
fi

# Create artifact directory
mkdir -p "$BASE_PATH/todos/pending"
```

</usage_patterns>

<transition_behavior>

## Backwards Compatibility During Transition

### Flat Structure (Legacy)

**Behavior:** All paths work exactly as before - no changes to command behavior

**Files accessed directly:**
- `.planning/PROJECT.md`
- `.planning/STATE.md`
- `.planning/ROADMAP.md`
- `.planning/phases/`
- `.planning/research/`
- `.planning/codebase/` (will become shared in nested)

**No `.active` file:** Not needed in flat structure

### Nested Structure (Multi-Project)

**Behavior:** Project-specific paths route through `.active` file

**Shared paths (at root):**
- `.planning/codebase/` (repo-level analysis)
- `.planning/config.json` (global defaults)
- `.planning/.gitignore`
- `.planning/projects/` (container)

**Project-specific paths (under projects/<name>/):**
- `PROJECT.md`, `STATE.md`, `ROADMAP.md`
- `phases/`, `research/`, `todos/`
- `config.json` (per-project overrides)

**`.active` file required:** Commands fail with NO_ACTIVE_PROJECT if missing

### Migration Moment

**Before migration:**
```
.planning/
├── PROJECT.md          # flat structure
├── STATE.md
└── phases/
```

**After migration:**
```
.planning/
├── .active             # contains "my-project"
├── codebase/           # shared (stays at root)
├── config.json         # global config (stays at root)
└── projects/
    └── my-project/
        ├── PROJECT.md  # moved from root
        ├── STATE.md
        └── phases/
```

**Migration is atomic:** No mixed state exists - either flat or nested, never both.

**Trigger point:** Migration only occurs when user runs `/gsd:new-project` (handled in Phase 2).

</transition_behavior>

<integration_checklist>

## Adding Path Resolution to New Commands

When creating or updating a command that accesses `.planning/` files:

- [ ] Add `@path-resolution.md` to command's `<execution_context>`
- [ ] Identify which files command reads/writes
- [ ] Classify each file as shared or project-specific (see @shared-paths.md)
- [ ] Replace hardcoded `.planning/` paths with resolution logic
- [ ] Add error handling for NO_ACTIVE_PROJECT case
- [ ] Test in both flat and nested structures
- [ ] Document which paths command accesses in command specification

**Template for command updates:** See Phase 4 plan for batch integration approach.

</integration_checklist>

<examples>

## Complete Examples

### Example 1: Resume Project Workflow

```bash
# Old approach (flat only)
cat .planning/STATE.md

# New approach (flat + nested)
if [ -d .planning/projects/ ]; then
  if [ ! -f .planning/.active ]; then
    echo "No active project. Available projects:"
    ls -1 .planning/projects/
    exit 1
  fi
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  cat ".planning/projects/$ACTIVE_PROJECT/STATE.md"
else
  cat .planning/STATE.md
fi
```

### Example 2: Create Phase Directory

```bash
# Resolve project base path
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  BASE_PATH=".planning/projects/$ACTIVE_PROJECT"
else
  BASE_PATH=".planning"
fi

# Create phase directory
PHASE_DIR="$BASE_PATH/phases/04-checkout"
mkdir -p "$PHASE_DIR"

echo "Phase directory created: $PHASE_DIR"
```

### Example 3: Read Codebase Map (Shared)

```bash
# Codebase is always shared - no resolution needed
CODEBASE_DIR=".planning/codebase"

if [ -d "$CODEBASE_DIR" ]; then
  echo "Tech stack:"
  cat "$CODEBASE_DIR/STACK.md"
else
  echo "No codebase map. Run: gsd:map-codebase"
fi
```

</examples>

<performance_notes>

## Performance Characteristics

**Detection overhead:** Each resolution checks `fs.existsSync()` - typically <1ms

**Caching consideration:** Path resolver is stateless by design
- Structure type (flat/nested) could theoretically be cached
- NOT recommended: cache adds complexity, stale data risk
- Structure changes are rare (one-time migration)
- Detection cost is negligible compared to file I/O

**Optimization if needed:**
- If performance becomes issue (unlikely), add cache invalidated after any write
- Start stateless, measure before optimizing

</performance_notes>

<related_references>

## Related Documentation

- **@shared-paths.md** - Complete list of shared vs project-specific paths
- **@git-integration.md** - How commits work in multi-project structure
- **/gsd:new-project** - Command that triggers migration (Phase 2)
- **/gsd:switch-project** - Command to change active project (Phase 2)

</related_references>

---

*Reference created: Phase 01 Plan 01*
*Valid for: All GSD commands and workflows*
*Update frequency: When new path categories added*
