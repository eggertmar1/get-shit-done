# Phase 03: Configuration & Context - Research

**Researched:** 2026-02-07
**Domain:** Hierarchical configuration systems, JSON config override patterns, git branch detection
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Configurable settings:**
- Claude decides which settings are overridable per-project vs global-only, based on each setting's nature
- Claude decides whether any project-only settings are needed (settings that don't exist at global level)
- No rigid "all or subset" rule — let the setting's purpose dictate whether per-project override makes sense

### Claude's Discretion

- Override behavior: how per-project configs merge with global (deep merge vs shallow, conflict resolution)
- Which specific settings are overridable per-project
- Whether project-only settings exist (e.g., project-specific model profile)
- `/gsd:settings` UX in multi-project context (edit project config, global config, or merged view)
- Whether project-level config.json is created on project init or only when user overrides something
- Branch-to-project mapping: how git branch auto-detection works for naming, including edge cases (detached HEAD, main branch, non-standard names)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

<research_summary>
## Summary

Researched hierarchical configuration override systems to understand best practices for implementing per-project config overrides on top of global defaults. The standard approach across modern tools (git, npm, VSCode) follows a consistent pattern: **local-first precedence with shallow merge semantics**. Configuration is read from multiple scopes (global → project) with later scopes completely replacing earlier values for matching keys.

The critical insight: **shallow merge is standard for configuration hierarchies**. Deep merging creates ambiguity about which level controls nested properties and makes it unclear what the "active" configuration actually is. Shallow merge provides clear precedence: if a key exists in project config, project config wins entirely for that key.

For GSD's needs, this means: project config.json contains only the keys being overridden. Global config.json provides defaults. At runtime, merge with project values replacing global values completely (not recursively merging nested objects).

**Primary recommendation:** Use shallow merge for config hierarchy (project values completely replace global values for matching keys), make settings overridable by nature (workflow toggles yes, shared codebase paths no), and provide clear UX showing which config scope is being edited.
</research_summary>

<standard_stack>
## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| JSON | Native | Config file format | Human-readable, parseable by all tools, existing GSD format |
| Bash / jq | Built-in | Config parsing and merging | Already used in GSD, sufficient for shallow merge |
| git symbolic-ref | Built-in | Branch name detection | Most reliable git branch detection, Phase 02 already established |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| JSON Schema | 2020-12 | Config validation (optional) | If GSD adds config validation in future |
| cat/grep/sed | Built-in | Config value extraction | Simple config reads without full JSON parsing |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Shallow merge | Deep merge | Deep merge adds complexity determining which scope controls nested values, ambiguous for users |
| JSON files | YAML/TOML | GSD already uses JSON, no benefit to switching formats |
| jq for merge | Node.js script | Node.js more complex, jq sufficient for shallow merge |

**Installation:**
```bash
# No installation needed - all built-in tools
# jq typically pre-installed on macOS/Linux, available via package managers
```
</standard_stack>

<architecture_patterns>
## Architecture Patterns

### Recommended Config Structure
```
.planning/
├── config.json              # Global defaults for all projects
└── projects/
    └── <project-name>/
        └── config.json      # Per-project overrides (optional file)
```

### Pattern 1: Local-First Precedence (Industry Standard)
**What:** Later configuration scopes override earlier ones completely for matching keys
**When to use:** All hierarchical configuration systems
**Examples from industry:**

**Git config (3 scopes):**
```bash
# System < Global < Local
# Local setting completely replaces global setting
git config --local user.email "project@example.com"
# user.email is now "project@example.com", not merged with global
```

**npm config (4 scopes):**
```bash
# Default < Global < User < Project
# Project .npmrc completely replaces global value for each key
# Example: project registry setting replaces global registry
```

**VSCode settings (2 scopes):**
```json
// User settings.json
{ "editor.fontSize": 14, "editor.tabSize": 4 }

// Workspace .vscode/settings.json
{ "editor.tabSize": 2 }

// Result: fontSize from user (14), tabSize from workspace (2)
// Workspace doesn't merge with user, it replaces for that key
```

### Pattern 2: Shallow Merge Semantics
**What:** Top-level keys are replaced entirely, not recursively merged
**When to use:** Configuration hierarchies where clear precedence matters

**Example:**
```json
// Global config.json
{
  "mode": "yolo",
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}

// Project config.json (overrides)
{
  "workflow": {
    "research": false
  }
}

// SHALLOW MERGE RESULT (standard approach):
{
  "mode": "yolo",               // From global (key not in project)
  "workflow": {                 // From project (replaces entire workflow object)
    "research": false
  }
}
// plan_check and verifier are GONE (not merged) - this is the pitfall!

// BETTER: Store workflow toggles as separate keys
{
  "mode": "yolo",
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true
}
// Now project can override individual toggles without losing others
```

**Key insight:** Shallow merge works well for flat structures. For nested objects, either deep merge (adds complexity) or flatten the structure (simpler, clearer).

### Pattern 3: Sparse Project Configs (VSCode Pattern)
**What:** Project config contains ONLY overrides, not full config copy
**When to use:** Any hierarchical config system
**Why:** Clear what's customized, easier to maintain, smaller diffs

**Example:**
```json
// Global config.json (100 lines, all settings)
{
  "mode": "yolo",
  "depth": "quick",
  "parallelization": true,
  "commit_docs": true,
  "model_profile": "balanced",
  "workflow": { ... },
  "git": { ... }
}

// Project config.json (5 lines, just overrides)
{
  "model_profile": "quality",
  "commit_docs": false
}

// At runtime: merge these two, project wins for matching keys
```

**Benefits:**
- Clear intent: "this project needs quality profile and private docs"
- Small diffs in git
- Easy to remove override (delete the key)
- Doesn't break when global config adds new settings

### Pattern 4: Config Scope Resolution (Multi-Level)
**What:** Algorithm for resolving which config value to use
**When to use:** Reading any config setting in GSD commands

**Pseudocode:**
```
function getConfig(key):
  projectConfig = readProjectConfig()  # .planning/projects/<active>/config.json
  globalConfig = readGlobalConfig()    # .planning/config.json

  # Shallow merge: project values replace global
  mergedConfig = { ...globalConfig, ...projectConfig }

  return mergedConfig[key]

function readProjectConfig():
  if not exists(.planning/.active):
    return {}  # No active project, only global config applies

  activeProject = read(.planning/.active).trim()
  projectConfigPath = .planning/projects/<activeProject>/config.json

  if exists(projectConfigPath):
    return parseJSON(projectConfigPath)
  else:
    return {}  # No project overrides, use global config

function readGlobalConfig():
  return parseJSON(.planning/config.json)
```

**In bash:**
```bash
# Read global config (always exists)
GLOBAL_CONFIG=$(cat .planning/config.json)

# Read project config (may not exist)
PROJECT_CONFIG="{}"
if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"
  if [ -f "$PROJECT_CONFIG_PATH" ]; then
    PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
  fi
fi

# Merge (jq shallow merge: project overrides global)
MERGED_CONFIG=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Extract value
MODEL_PROFILE=$(echo "$MERGED_CONFIG" | jq -r '.model_profile')
```

### Pattern 5: Lazy Project Config Creation
**What:** Don't create project config.json until user overrides something
**When to use:** Keeps project directories clean for projects using all defaults
**Why:** Sparse is better - only create files when needed

**Flow:**
```
1. /gsd:new-project creates project directory WITHOUT config.json
2. Project uses global config (via resolution algorithm)
3. User runs /gsd:settings and changes a setting
4. IF user is editing project scope:
   - Create .planning/projects/<active>/config.json with override
5. IF user is editing global scope:
   - Update .planning/config.json
```

### Pattern 6: Git Branch Detection (From Phase 02)
**What:** Use git symbolic-ref for reliable branch name detection
**When to use:** Default project naming in /gsd:new-project
**Already implemented:** Phase 02-01 established this pattern

**Reference:**
```bash
# Most reliable method (Phase 02-01 decision)
BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)

# Handle detached HEAD
if [ -z "$BRANCH_NAME" ] || [ "$BRANCH_NAME" == "HEAD" ]; then
  # Fallback: prompt user for manual name
  BRANCH_NAME=""
fi
```

**This phase doesn't change branch detection** - just documents it as context for config merger.

### Anti-Patterns to Avoid

- **Deep merge without clear rules:** Users confused about which scope controls nested properties
- **Global config with project-specific values:** Antipattern - project paths should never be in global config
- **Requiring project config.json on init:** Creates unnecessary files for projects using defaults
- **Copying global config to project config:** Large diffs, maintenance burden, unclear what's overridden
- **Config caching:** Config files are small (<1KB), read on every command - caching adds stale state risk
</architecture_patterns>

<dont_hand_roll>
## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JSON parsing in bash | String manipulation, regex parsing | jq (JSON processor) | jq handles edge cases (nested quotes, unicode, escaping) that regex misses |
| JSON merging | Manual key-by-key copy loops | jq shallow merge: `jq -s '.[0] * .[1]'` | jq's `*` operator does shallow merge correctly, handles all JSON types |
| Config validation | Custom schema checker | JSON Schema (if needed) | Industry standard, tooling support, clear error messages |
| Deep merge algorithm | Recursive merge implementation | Keep structure flat OR use jq `reduce` with clear rules | Deep merge is complex (arrays? nulls? conflicts?) - avoid if possible |

**Key insight:** Config operations look simple but have edge cases. JSON contains strings with special characters, nested objects, arrays, nulls. jq handles all of this correctly. Hand-rolled bash string manipulation will have bugs.
</dont_hand_roll>

<common_pitfalls>
## Common Pitfalls

### Pitfall 1: Shallow Merge Loses Nested Keys
**What goes wrong:** Project overrides `workflow.research = false`, but this replaces the entire `workflow` object, losing `plan_check` and `verifier` settings
**Why it happens:** Shallow merge replaces values for top-level keys, not recursive merge
**How to avoid:**
- Option A: Flatten config structure (`workflow.research`, `workflow.plan_check` as separate top-level keys)
- Option B: Use deep merge (more complex, requires clear rules for arrays and conflicts)
- Option C: Require complete objects in overrides (forces explicit choice)
**Warning signs:** User says "I changed one workflow setting and the others disappeared"

**Recommendation for GSD:** Flatten workflow settings:
```json
// INSTEAD OF (nested - shallow merge breaks):
{
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}

// USE (flat - shallow merge works):
{
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true
}
```

### Pitfall 2: Config Scope Ambiguity in /gsd:settings
**What goes wrong:** User runs `/gsd:settings`, changes model_profile, unclear if this updated global or project config
**Why it happens:** No indication of which scope is being edited
**How to avoid:** Make scope selection explicit:
- Show current scope: "Editing: Project (my-feature) config" vs "Editing: Global config"
- Offer choice: "Apply to: [This project only] [All projects (global)]"
- Show merged view but track which scope each value comes from
**Warning signs:** User says "I changed settings but other project still has old value" (expected global edit, got project edit)

### Pitfall 3: Non-Overridable Settings in Project Config
**What goes wrong:** User puts `codebase` paths in project config, expecting project-specific codebase maps
**Why it happens:** No clear documentation of what's overridable vs global-only
**How to avoid:** Document setting scopes clearly:
- **Global-only:** Shared resources (codebase maps, git integration settings)
- **Overridable:** Workflow toggles, model profile, commit_docs behavior
- **Project-only:** Project-specific metadata (if any)
**Warning signs:** User reports "project config setting is ignored"

**Recommendation for GSD:**

| Setting | Scope | Reason |
|---------|-------|--------|
| `mode`, `depth`, `parallelization` | Overridable | Workflow style is project-dependent |
| `commit_docs` | Overridable | Some projects are public (no commit), others private (commit) |
| `model_profile` | Overridable | High-priority projects may need quality profile |
| `workflow.*` toggles | Overridable | Disable research for simple projects, enable for complex ones |
| `git.branching_strategy` | Overridable | Solo projects use "none", team projects use "phase" |
| Codebase paths | Global-only | Shared analysis across all projects in repo |
| `.active` file | N/A (not in config) | Machine-local state, not configuration |

### Pitfall 4: Missing Config Files Break Commands
**What goes wrong:** Project config.json doesn't exist, config reader crashes instead of falling back to global
**Why it happens:** Not handling missing file case in resolution logic
**How to avoid:** Graceful fallback:
```bash
# WRONG:
PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")  # Fails if file doesn't exist

# RIGHT:
if [ -f "$PROJECT_CONFIG_PATH" ]; then
  PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
else
  PROJECT_CONFIG="{}"  # Empty JSON object (no overrides)
fi
```
**Warning signs:** "No such file" errors when project config doesn't exist

### Pitfall 5: Forgetting to Document Merged Config Source
**What goes wrong:** User sees config value, doesn't know if it's from global or project
**Why it happens:** Showing merged config without provenance
**How to avoid:** In settings UI or debug output, show source:
```
Current configuration (merged):
- mode: "yolo" (global)
- model_profile: "quality" (project override)
- workflow.research: true (global)
```
**Warning signs:** User confusion about "where is this setting coming from?"

### Pitfall 6: Updating Wrong Config Scope
**What goes wrong:** User intends to update global config, command updates project config
**Why it happens:** Implicit scope selection based on whether .active exists
**How to avoid:** Explicit scope selection:
- `/gsd:settings --global` - edit global config
- `/gsd:settings` - edit project config if active project, else global
- `/gsd:settings --project <name>` - edit specific project config
**Warning signs:** Unexpected config changes, settings not applying to other projects
</common_pitfalls>

<code_examples>
## Code Examples

### Config Resolution (Shallow Merge)
```bash
# Source: jq manual + VSCode config precedence pattern
# Resolve merged config with project overrides

# Function: Get merged config
function get_merged_config() {
  GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")

  # Try to get project config
  PROJECT_CONFIG="{}"
  if [ -f .planning/.active ]; then
    ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
    if [ -n "$ACTIVE_PROJECT" ]; then
      PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"
      if [ -f "$PROJECT_CONFIG_PATH" ]; then
        PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
      fi
    fi
  fi

  # Shallow merge: project overrides global
  # jq's `*` operator merges objects (right side wins for conflicts)
  MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))
  echo "$MERGED"
}

# Usage: Get specific config value
MERGED_CONFIG=$(get_merged_config)
MODEL_PROFILE=$(echo "$MERGED_CONFIG" | jq -r '.model_profile // "balanced"')
COMMIT_DOCS=$(echo "$MERGED_CONFIG" | jq -r '.commit_docs // true')

echo "Using model profile: $MODEL_PROFILE"
```

### Config Update (Specific Scope)
```bash
# Source: npm config set pattern
# Update config at specific scope (global or project)

function update_config() {
  SCOPE=$1  # "global" or "project"
  KEY=$2
  VALUE=$3

  if [ "$SCOPE" = "global" ]; then
    CONFIG_PATH=".planning/config.json"
  else
    # Project scope
    if [ ! -f .planning/.active ]; then
      echo "Error: No active project"
      return 1
    fi
    ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
    CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"

    # Create project config if doesn't exist
    if [ ! -f "$CONFIG_PATH" ]; then
      echo "{}" > "$CONFIG_PATH"
    fi
  fi

  # Update config (jq updates in place)
  CURRENT=$(cat "$CONFIG_PATH")
  UPDATED=$(echo "$CURRENT" | jq --arg key "$KEY" --arg val "$VALUE" '.[$key] = $val')
  echo "$UPDATED" > "$CONFIG_PATH"

  echo "Updated $SCOPE config: $KEY = $VALUE"
}

# Usage
update_config "project" "model_profile" "quality"
update_config "global" "commit_docs" "true"
```

### Flattened Config Structure (Avoiding Shallow Merge Pitfall)
```json
// Source: Best practice for shallow merge compatibility
// Flatten nested structures to avoid losing keys

// BEFORE (problematic with shallow merge):
{
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  },
  "git": {
    "branching_strategy": "none",
    "phase_branch_template": "gsd/phase-{phase}-{slug}"
  }
}

// AFTER (shallow merge safe):
{
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true,
  "git.branching_strategy": "none",
  "git.phase_branch_template": "gsd/phase-{phase}-{slug}"
}

// Now project can override individual keys:
// Project config.json
{
  "workflow.research": false,  // Overrides this one setting
  "git.branching_strategy": "phase"  // Overrides this one setting
}
// Other settings remain from global (not lost in shallow merge)
```

**Alternative:** Keep nested structure but use deep merge:
```bash
# Deep merge with jq (more complex)
# Recursively merge nested objects
jq -s 'def deepmerge(a;b):
  reduce (a | to_entries)[] as {$key,$value} (b;
    .[$key] = if ($value | type == "object") and (.[$key] | type == "object")
              then deepmerge($value; .[$key])
              else $value end);
  deepmerge(.[0]; .[1])' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG")
```

**Recommendation:** Start with flat structure (simpler), add deep merge later if really needed.

### Config Provenance (Show Sources)
```bash
# Source: Git config --show-origin pattern
# Show which scope each config value comes from

function show_config_with_sources() {
  GLOBAL_CONFIG=$(cat .planning/config.json)

  # Get project config if exists
  PROJECT_CONFIG="{}"
  PROJECT_CONFIG_EXISTS=false
  if [ -f .planning/.active ]; then
    ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
    PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"
    if [ -f "$PROJECT_CONFIG_PATH" ]; then
      PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
      PROJECT_CONFIG_EXISTS=true
    fi
  fi

  echo "Configuration (merged view):"
  echo ""

  # Get all keys from merged config
  MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

  # For each key, determine source
  echo "$MERGED" | jq -r 'to_entries[] | "\(.key) = \(.value)"' | while read -r line; do
    KEY=$(echo "$line" | cut -d'=' -f1 | tr -d ' ')
    VALUE=$(echo "$line" | cut -d'=' -f2-)

    # Check if key is in project config
    if [ "$PROJECT_CONFIG_EXISTS" = true ]; then
      PROJECT_HAS_KEY=$(echo "$PROJECT_CONFIG" | jq --arg key "$KEY" 'has($key)')
      if [ "$PROJECT_HAS_KEY" = "true" ]; then
        echo "  $KEY = $VALUE (project override)"
        continue
      fi
    fi

    # Otherwise from global
    echo "  $KEY = $VALUE (global)"
  done
}

# Usage
show_config_with_sources
```

### Settings UX with Scope Selection
```bash
# Source: Git config --global pattern
# Settings command with explicit scope selection

function gsd_settings() {
  # Parse flags
  SCOPE="auto"  # auto = project if active, else global
  while [[ "$1" =~ ^-- ]]; do
    case "$1" in
      --global)
        SCOPE="global"
        shift
        ;;
      --project)
        SCOPE="project"
        shift
        ;;
      *)
        echo "Unknown flag: $1"
        return 1
        ;;
    esac
  done

  # Determine target scope
  if [ "$SCOPE" = "auto" ]; then
    if [ -f .planning/.active ] && [ -n "$(cat .planning/.active | tr -d '[:space:]')" ]; then
      SCOPE="project"
      ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
    else
      SCOPE="global"
    fi
  fi

  # Show scope to user
  if [ "$SCOPE" = "global" ]; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo " Editing: Global Config"
    echo " (applies to all projects)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  else
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo " Editing: Project Config"
    echo " Project: $ACTIVE_PROJECT"
    echo " (overrides global for this project only)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  fi

  # Continue with settings prompts...
  # Use AskUserQuestion for interactive selection
  # Write to appropriate config.json based on SCOPE
}

# Usage examples:
# /gsd:settings              # Edit project config (if active), else global
# /gsd:settings --global     # Always edit global
# /gsd:settings --project    # Always edit project (error if no active)
```
</code_examples>

<sota_updates>
## State of the Art (2025-2026)

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Deep merge by default | Shallow merge with explicit scope | 2020+ (Terraform, Pulumi) | Clearer precedence, less ambiguity about which scope controls what |
| All settings overridable | Scope-aware settings (global-only, overridable, local-only) | 2023+ | Prevents nonsensical overrides (e.g., shared resources in project scope) |
| Config caching | Read config fresh each time | Ongoing | Small files (<1KB), read is fast, avoids stale cache issues |
| JSON-only | Multi-format (JSON, YAML, TOML) | 2022+ | GSD already uses JSON, no need to change |

**Current best practices (2026):**
- **Shallow merge by default** with deep merge opt-in for specific keys (Terraform `merge_strategy`)
- **Provenance tracking** showing which scope each value came from (git config --show-origin)
- **Flat config structures** where possible to avoid merge ambiguity
- **Lazy config creation** - don't create project config until user overrides something
- **Explicit scope selection** in settings UX (--global vs --project flags)

**Deprecated/outdated:**
- Deep merge without clear rules about arrays/nulls/conflicts
- Hidden config scope (user doesn't know if editing global or project)
- Copying global config to project config (creates maintenance burden)

**Emerging patterns:**
- **JSON Schema validation** for config files (catches typos, wrong types)
- **Config migration scripts** for breaking changes to config format
- **Config templating** for common project types (not needed for GSD yet)
</sota_updates>

<open_questions>
## Open Questions

Things that couldn't be fully resolved:

1. **Deep merge vs shallow merge for GSD**
   - What we know: Shallow merge is simpler and standard across git/npm/VSCode
   - What's unclear: Will nested config structures (like `workflow` object) cause user confusion with shallow merge?
   - Recommendation: Start with **shallow merge** but **flatten config structure** to avoid losing keys. Example: `workflow.research` instead of `workflow: { research: true }`. This gives shallow merge benefits without the pitfall of losing sibling keys.

2. **Which settings are truly overridable?**
   - What we know: Workflow toggles and model_profile make sense to override per-project
   - What's unclear: Should git.branching_strategy be overridable? (probably yes - some projects need branches, others don't)
   - Recommendation: Make **most settings overridable by default** (workflow style is project-dependent), only restrict settings that are technically global-only (e.g., codebase paths which are shared resources). Document clearly in reference doc.

3. **/gsd:settings UX: which scope by default?**
   - What we know: Git uses flags (--global vs --local), VSCode has separate UI for user vs workspace
   - What's unclear: Should GSD default to project scope (when active) or always ask?
   - Recommendation: **Auto-detect scope** (project if active, global if no active project), but **show scope clearly** at top of settings UI and offer way to switch: "Editing project config. Switch to global config?" link.

4. **Project config.json creation timing**
   - What we know: Lazy creation (only when overriding) keeps directories clean
   - What's unclear: Does this confuse users ("where do I put overrides?")?
   - Recommendation: **Lazy creation** (don't create until user overrides something). In /gsd:settings UI, show "Project config: not created yet (using global defaults)" to make it clear.

5. **Config validation**
   - What we know: JSON Schema exists for validation
   - What's unclear: Is validation worth the complexity for GSD's simple config?
   - Recommendation: **Skip validation for now**. GSD's config is simple (~10 keys), typos will surface as "setting ignored" which is acceptable. Add JSON Schema later if config grows complex.
</open_questions>

<sources>
## Sources

### Primary (HIGH confidence)
- [Git Config Scopes Documentation](https://medium.com/@yadavprakhar1809/understanding-the-three-levels-of-git-config-local-global-and-system-e95c26aac8ee) - Git's three-level hierarchy (system, global, local) with precedence
- [npm Config Documentation](https://docs.npmjs.com/cli/v9/configuring-npm/npmrc/) - npm's four-level config hierarchy and precedence rules
- [VSCode User and Workspace Settings](https://code.visualstudio.com/docs/configure/settings) - Two-tier config with workspace overriding user settings
- [Configuration in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-8.0) - Hierarchical config with last-wins override pattern
- [Git symbolic-ref Documentation](https://git-scm.com/docs/git-symbolic-ref) - Official git documentation for branch detection
- GSD Phase 02 Research (existing) - Git branch detection patterns established in Phase 02-01

### Secondary (MEDIUM confidence)
- [Shallow vs Deep Merge in JavaScript](https://lakum.in/blog/shallow-merge-vs-deep-merge-in-javascript/) - Explains difference and use cases
- [Terragrunt Deep Merge](https://docs.gruntwork.io/guides/stay-up-to-date/terraform/how-to-dry-your-reference-architecture/deployment-walkthrough/optional-even-dryer-configuration/) - Example of deep merge in config system with explicit opt-in
- [JSON Schema](https://json-schema.org/) - JSON validation standard (2020-12 latest)
- [Configuration Management Best Practices](https://www.infoq.com/articles/5-config-mgmt-best-practices/) - Avoid implementation-specific access patterns, centralize config

### Tertiary (LOW confidence - context only)
- [Hierarchical Config in Pulumi](https://www.pulumi.com/blog/2022-03-10-hierarchical-config/) - Infrastructure-as-code config hierarchies (different domain but similar patterns)
- [Design System Overrides](https://medium.com/cva-design/when-is-it-okay-to-override-a-design-system-816e89a56f9a) - Document overrides, treat as temporary (philosophy applies to config)

### GSD-Specific (HIGH confidence)
- `.planning/config.json` (existing) - Current GSD config structure with mode, depth, parallelization, commit_docs, model_profile, workflow toggles
- `get-shit-done/references/planning-config.md` (existing) - Documents current config schema and behavior
- `get-shit-done/references/model-profiles.md` (existing) - Model profile definitions per agent
- `commands/gsd/settings.md` (existing) - Current settings command (single-scope, needs multi-project update)
</sources>

<metadata>
## Metadata

**Research scope:**
- Core technology: JSON config files, bash + jq for parsing/merging, git symbolic-ref for branch detection
- Ecosystem: Config hierarchy patterns from git, npm, VSCode (industry standards)
- Patterns: Local-first precedence, shallow merge semantics, sparse project configs, lazy config creation
- Pitfalls: Shallow merge loses nested keys, scope ambiguity in UX, non-overridable settings in wrong scope

**Confidence breakdown:**
- Standard stack: HIGH - JSON and jq are proven, git symbolic-ref established in Phase 02
- Architecture: HIGH - Local-first precedence is universal pattern across git/npm/VSCode, shallow merge is standard
- Override semantics: MEDIUM - Shallow vs deep merge tradeoffs are clear, but optimal choice for GSD requires implementation testing
- Settings scope categorization: MEDIUM - General principles clear (workflow = overridable, shared resources = global-only), but specific GSD settings need case-by-case analysis
- UX patterns: HIGH - Git/VSCode patterns are well-established and user-familiar

**Research date:** 2026-02-07
**Valid until:** 2026-03-07 (30 days - config management patterns are mature and stable)

**Key decisions for planner:**
1. **Use shallow merge** with flattened config structure (avoid nested objects that lose keys)
2. **Make settings overridable by nature** (workflow/profile = yes, codebase paths = no)
3. **Lazy project config creation** (don't create until user overrides)
4. **Explicit scope in /gsd:settings UX** (show "Editing: Project config" vs "Editing: Global config")
5. **No config validation** for now (simple config, validation is overkill)

---

*Phase: 03-configuration-context*
*Research completed: 2026-02-07*
*Ready for planning: yes*
</metadata>
