<overview>
Hierarchical configuration resolution algorithm for multi-project GSD. Merges global defaults with per-project overrides using shallow merge semantics (project values completely replace global values for matching keys).
</overview>

<core_principle>

**Read global, overlay project, project wins.**

Global config at `.planning/config.json` provides defaults for all settings. Per-project config at `.planning/projects/<name>/config.json` contains only overrides (sparse). Merge is shallow: top-level keys from project config completely replace global values for matching keys.

This approach provides sensible defaults while allowing projects to customize settings like workflow mode, model profile, or branching strategy without duplicating the entire config.

</core_principle>

<config_hierarchy>

## Two Configuration Scopes

### 1. Global Config

**Location:** `.planning/config.json`

**Purpose:** Default settings for all projects in the repository

**Lifecycle:** Always exists (created during first project initialization)

**Contains:** Complete configuration with all settings defined

**Example:**
```json
{
  "mode": "yolo",
  "depth": "quick",
  "parallelization": true,
  "commit_docs": true,
  "model_profile": "balanced",
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true,
  "planning.search_gitignored": false,
  "git.branching_strategy": "none",
  "git.phase_branch_template": "gsd/phase-{phase}-{slug}",
  "git.milestone_branch_template": "gsd/{milestone}-{slug}"
}
```

---

### 2. Project Config

**Location:** `.planning/projects/<active>/config.json`

**Purpose:** Project-specific overrides for customization

**Lifecycle:** Created lazily on first override (most projects won't have one)

**Contains:** Only the settings being overridden (sparse)

**Example:**
```json
{
  "mode": "careful",
  "model_profile": "quality",
  "git.branching_strategy": "phase"
}
```

This project config overrides 3 settings while inheriting the rest from global.

---

## Resolution Order

1. **Global** — Read `.planning/config.json` (base layer)
2. **Project** — Read `.planning/projects/<active>/config.json` if exists (overlay)
3. **Merge** — Shallow merge with project values winning for matching keys

**Result:** Effective configuration combining both sources, with project taking precedence.

</config_hierarchy>

<resolution_algorithm>

## Config Resolution Algorithm

### Pseudocode

```
function resolveConfig():
  # Step 1: Read global config (always exists)
  globalConfig = readJSON('.planning/config.json') || {}

  # Step 2: Determine active project
  if not exists('.planning/.active'):
    # No active project (flat structure or multi-project without active)
    return globalConfig

  activeProject = readFile('.planning/.active').trim()
  if empty(activeProject):
    return globalConfig

  # Step 3: Read project config (may not exist - lazy creation)
  projectConfigPath = '.planning/projects/' + activeProject + '/config.json'
  projectConfig = readJSON(projectConfigPath) || {}

  # Step 4: Shallow merge (project wins)
  mergedConfig = shallowMerge(globalConfig, projectConfig)

  return mergedConfig
```

---

### Bash Implementation

Complete bash implementation for resolving merged config:

```bash
# Read global config (always exists, fallback to {} if missing)
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")

# Initialize project config as empty
PROJECT_CONFIG="{}"

# Read .active file to find active project (use path-resolution.md pattern)
if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')

  # Read project config if exists (fallback to {} if missing)
  if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
    PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
  fi
fi

# Shallow merge with jq: global as base, project overlay
# The * operator performs object merge (project values override global)
MERGED_CONFIG=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Extract specific value with jq (using bracket syntax for dot-notation keys)
MODE=$(echo "$MERGED_CONFIG" | jq -r '.mode // "yolo"')
RESEARCH=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.research"] // true')
BRANCHING=$(echo "$MERGED_CONFIG" | jq -r '.["git.branching_strategy"] // "none"')
```

**Key points:**

- Use `jq -s '.[0] * .[1]'` for shallow merge (slurp both configs into array, merge with `*` operator)
- Access dot-notation keys with bracket syntax: `.["workflow.research"]` not `.workflow.research`
- Always provide fallback defaults with `// operator` in case key doesn't exist
- Empty `{}` for missing project config ensures merge still works

---

### Reading a Single Config Value

Quick helper for reading one config value:

```bash
function read_config_value() {
  local KEY="$1"
  local DEFAULT="$2"

  # Resolve merged config
  GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
  PROJECT_CONFIG="{}"

  if [ -f .planning/.active ]; then
    ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
    if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
      PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
    fi
  fi

  MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

  # Extract value (handle keys with dots using bracket syntax)
  if [[ "$KEY" == *.* ]]; then
    echo "$MERGED" | jq -r ".\"$KEY\" // $DEFAULT"
  else
    echo "$MERGED" | jq -r ".$KEY // $DEFAULT"
  fi
}

# Usage
MODE=$(read_config_value "mode" "\"yolo\"")
RESEARCH=$(read_config_value "workflow.research" "true")
```

</resolution_algorithm>

<overridable_settings>

## Setting Classification

Every config setting is classified as either **overridable** (can be customized per-project) or **global-only** (must be consistent across all projects).

| Setting Key | Default | Overridable | Rationale |
|-------------|---------|-------------|-----------|
| `mode` | `"yolo"` | Yes | Workflow style is project-dependent (careful for critical systems, yolo for exploration) |
| `depth` | `"quick"` | Yes | Complex projects may need comprehensive planning while simple ones need quick |
| `parallelization` | `true` | Yes | Some projects may need sequential execution for dependencies |
| `commit_docs` | `true` | Yes | Public/OSS projects may keep planning private via gitignore |
| `model_profile` | `"balanced"` | Yes | High-priority projects may warrant quality profile despite cost |
| `workflow.research` | `true` | Yes | Simple bug fixes can skip research, complex features need it |
| `workflow.plan_check` | `true` | Yes | Fast-iteration projects may skip plan checking |
| `workflow.verifier` | `true` | Yes | Quick fixes may skip verification, critical paths need it |
| `git.branching_strategy` | `"none"` | Yes | Team projects need phase branches for review, solo projects don't |
| `git.phase_branch_template` | `"gsd/phase-{phase}-{slug}"` | Yes | Projects may need custom branch naming for CI/CD integration |
| `git.milestone_branch_template` | `"gsd/{milestone}-{slug}"` | Yes | Projects may need custom branch naming for release workflows |
| `planning.search_gitignored` | `false` | No (global-only) | Search scope is repo-level, not project-level (affects ripgrep behavior globally) |

**Overridable settings** can be customized per-project via `/gsd:settings --project`.

**Global-only settings** must be edited via `/gsd:settings --global` and apply to all projects.

### Why planning.search_gitignored is Global-Only

The `planning.search_gitignored` setting controls whether ripgrep searches (used throughout GSD) add the `--no-ignore` flag. This affects how the tool searches the entire repository structure, not just the planning artifacts. Making this per-project would create confusing behavior where the same search command behaves differently based on active project context.

</overridable_settings>

<lazy_creation>

## Lazy Project Config Creation

Project config files are NOT created during `/gsd:new-project`. They're only created when a user first overrides a setting via `/gsd:settings` with project scope.

**Rationale:** Most projects use global defaults. Creating `config.json` in every project directory would clutter the structure with duplicate default configs.

**Lifecycle:**

1. **New project created** — No `config.json` file exists yet
2. **Settings command reads config** — Missing project config treated as `{}` (empty object)
3. **User overrides first setting** — `/gsd:settings --project` creates the file
4. **Subsequent overrides** — File exists, update it

**Reading behavior:**

```bash
# When reading project config
PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"

if [ -f "$PROJECT_CONFIG_PATH" ]; then
  PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
else
  # File doesn't exist yet - treat as empty (no overrides)
  PROJECT_CONFIG="{}"
fi
```

**Writing behavior:**

```bash
# When writing project config
PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"

# Create file if doesn't exist (lazy creation happens here)
if [ ! -f "$PROJECT_CONFIG_PATH" ]; then
  echo "Creating project config (first override)"
fi

# Write sparse config (only overridden keys)
echo "$PROJECT_CONFIG" > "$PROJECT_CONFIG_PATH"
```

**Benefits:**

- Clean project directories (no unnecessary files)
- Clear signal of which projects have customizations
- Reduces git diff noise (no duplicate defaults)
- Lazy creation defers file system operations until needed

</lazy_creation>

<flat_config_structure>

## Flattened Dot-Notation Keys

**CRITICAL:** GSD config uses a flattened structure with dot-notation keys instead of nested objects.

### Why Flat Format?

**Problem with nested format:**

```json
{
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}
```

With shallow merge, if project config overrides `workflow.research`:

```json
{
  "workflow": {
    "research": false
  }
}
```

The entire `workflow` object from global config is **replaced**, losing `plan_check` and `verifier` settings. This forces every project override to duplicate all sibling keys.

**Solution: Flat dot-notation:**

```json
{
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true
}
```

Each setting is an independent top-level key. Overriding `"workflow.research"` in project config doesn't affect `"workflow.plan_check"` or `"workflow.verifier"` — they remain inherited from global config.

---

### Config Format Comparison

**Old nested format (problematic):**

```json
{
  "mode": "yolo",
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  },
  "git": {
    "branching_strategy": "none",
    "phase_branch_template": "gsd/phase-{phase}-{slug}",
    "milestone_branch_template": "gsd/{milestone}-{slug}"
  }
}
```

**New flat format (recommended):**

```json
{
  "mode": "yolo",
  "depth": "quick",
  "parallelization": true,
  "commit_docs": true,
  "model_profile": "balanced",
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true,
  "planning.search_gitignored": false,
  "git.branching_strategy": "none",
  "git.phase_branch_template": "gsd/phase-{phase}-{slug}",
  "git.milestone_branch_template": "gsd/{milestone}-{slug}"
}
```

---

### Accessing Flat Keys with jq

**Correct** (bracket syntax for keys with dots):

```bash
RESEARCH=$(echo "$CONFIG" | jq -r '.["workflow.research"] // true')
BRANCHING=$(echo "$CONFIG" | jq -r '.["git.branching_strategy"] // "none"')
```

**Incorrect** (treats dots as nested object access):

```bash
# This looks for CONFIG.workflow.research (nested), not CONFIG["workflow.research"]
RESEARCH=$(echo "$CONFIG" | jq -r '.workflow.research // true')
```

---

### Migration and Backwards Compatibility

**Existing configs with nested format still work** because jq's `*` merge operator handles object merging recursively. However, this maintains the shallow merge problem for nested keys.

**Migration path:**

- Phase 3 (this phase): Document flat format as the standard
- Phase 4 integration: Update `/gsd:settings` command to write flat format
- Existing nested configs: Continue working but not recommended for new projects

**For new projects:** Always use flat dot-notation format.

**For existing projects:** No immediate migration required, but flat format prevents override issues.

</flat_config_structure>

<usage_patterns>

## Common Usage Patterns

### Pattern 1: Read a Single Config Value

Quick one-liner for reading a config value with default fallback:

```bash
# Read mode setting (default: "yolo")
MODE=$(jq -s '.[0] * .[1]' \
  <(cat .planning/config.json 2>/dev/null || echo "{}") \
  <(cat ".planning/projects/$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')/config.json" 2>/dev/null || echo "{}") \
  | jq -r '.mode // "yolo"')

# Read workflow.research setting (default: true)
RESEARCH=$(jq -s '.[0] * .[1]' \
  <(cat .planning/config.json 2>/dev/null || echo "{}") \
  <(cat ".planning/projects/$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')/config.json" 2>/dev/null || echo "{}") \
  | jq -r '.["workflow.research"] // true')
```

---

### Pattern 2: Read Merged Config in Workflow/Command

Full merged config for commands that need multiple settings:

```bash
# In workflow <process> section

# Step 1: Resolve merged config
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
PROJECT_CONFIG="{}"

if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
    PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
  fi
fi

MERGED_CONFIG=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Step 2: Extract needed values
MODE=$(echo "$MERGED_CONFIG" | jq -r '.mode // "yolo"')
MODEL_PROFILE=$(echo "$MERGED_CONFIG" | jq -r '.model_profile // "balanced"')
RESEARCH=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.research"] // true')
PLAN_CHECK=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.plan_check"] // true')
VERIFIER=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.verifier"] // true')

# Step 3: Use settings
if [ "$RESEARCH" = "true" ]; then
  echo "Spawning researcher..."
fi
```

---

### Pattern 3: Write Config Value to Specific Scope

Update a config value in either global or project scope:

```bash
# Determine scope (global or project)
SCOPE="$1"  # "global" or "project"
KEY="$2"    # e.g., "mode" or "workflow.research"
VALUE="$3"  # e.g., "careful" or "false"

if [ "$SCOPE" = "global" ]; then
  CONFIG_FILE=".planning/config.json"
else
  # Project scope
  if [ ! -f .planning/.active ]; then
    echo "Error: No active project"
    exit 1
  fi

  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  CONFIG_FILE=".planning/projects/$ACTIVE_PROJECT/config.json"

  # Lazy creation: create file if doesn't exist
  if [ ! -f "$CONFIG_FILE" ]; then
    echo "{}" > "$CONFIG_FILE"
  fi
fi

# Read current config
CURRENT_CONFIG=$(cat "$CONFIG_FILE")

# Update specific key (handle dot-notation keys)
if [[ "$KEY" == *.* ]]; then
  UPDATED_CONFIG=$(echo "$CURRENT_CONFIG" | jq ".\"$KEY\" = $VALUE")
else
  UPDATED_CONFIG=$(echo "$CURRENT_CONFIG" | jq ".$KEY = $VALUE")
fi

# Write back
echo "$UPDATED_CONFIG" > "$CONFIG_FILE"

echo "Updated $KEY = $VALUE in $SCOPE config"
```

**Important:** For project scope, only write the keys being overridden (sparse config), not a full copy of global config.

</usage_patterns>

<provenance_display>

## Config Provenance Display

Show where each config value comes from (similar to `git config --show-origin`).

Useful for `/gsd:settings` command to show users which values are overridden vs inherited:

```bash
# Read both configs separately (don't merge yet)
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
PROJECT_CONFIG="{}"
PROJECT_EXISTS=false

if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"

  if [ -f "$PROJECT_CONFIG_PATH" ]; then
    PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
    PROJECT_EXISTS=true
  fi
fi

# Function to get value source
function get_config_source() {
  local KEY="$1"

  # Check if key exists in project config
  if [ "$PROJECT_EXISTS" = "true" ]; then
    PROJECT_VALUE=$(echo "$PROJECT_CONFIG" | jq -r ".\"$KEY\"" 2>/dev/null)
    if [ "$PROJECT_VALUE" != "null" ]; then
      echo "project"
      return
    fi
  fi

  echo "global"
}

# Display config with provenance
echo "Current Configuration:"
echo ""
echo "| Setting | Value | Source |"
echo "|---------|-------|--------|"

for KEY in "mode" "model_profile" "workflow.research" "workflow.plan_check" "workflow.verifier" "git.branching_strategy"; do
  # Get merged value
  MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))
  VALUE=$(echo "$MERGED" | jq -r ".\"$KEY\"")

  # Get source
  SOURCE=$(get_config_source "$KEY")

  echo "| $KEY | $VALUE | $SOURCE |"
done
```

**Example output:**

```
Current Configuration:

| Setting | Value | Source |
|---------|-------|--------|
| mode | careful | project |
| model_profile | quality | project |
| workflow.research | true | global |
| workflow.plan_check | true | global |
| workflow.verifier | true | global |
| git.branching_strategy | phase | project |
```

This clearly shows which settings are overridden (project) vs inherited (global).

</provenance_display>

<related_references>

## Related Documentation

- **@path-resolution.md** — Path resolver for reading `.active` file and determining project paths
- **@shared-paths.md** — Classification of which paths are shared (like global config) vs project-specific (like project config overrides)
- **@planning-config.md** — Complete config schema with all available settings and their defaults
- **/gsd:settings** — User-facing command that implements config editing with scope selection

</related_references>

---

*Reference created: Phase 03 Plan 01*
*Valid for: All GSD commands and workflows reading configuration*
*Update frequency: When new config settings added or resolution algorithm changes*
