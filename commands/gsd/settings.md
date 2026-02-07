---
name: gsd:settings
description: Configure GSD workflow toggles and model profile
allowed-tools:
  - Read
  - Write
  - AskUserQuestion
---

<objective>
Allow users to toggle workflow agents on/off and select model profile via interactive settings. In multi-project mode, users can edit global config or per-project overrides.

Updates `.planning/config.json` (global) or `.planning/projects/<name>/config.json` (project) based on scope selection.
</objective>

<execution_context>
@get-shit-done/references/config-resolution.md
</execution_context>

<process>

## 1. Validate Environment

```bash
ls .planning/config.json 2>/dev/null
```

**If not found:** Error - run `/gsd:new-project` first.

**Detect multi-project mode:**

```bash
MULTI_PROJECT=false
if [ -d .planning/projects/ ]; then
  MULTI_PROJECT=true
  ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
fi
```

## 1.5. Determine Scope

**Scope selection logic:**

- **If not multi-project:** scope = global (only option)
- **If multi-project with `--global` flag:** scope = global
- **If multi-project with `--project` flag:** scope = project (error if no active project)
- **If multi-project with no flags (auto-detect):**
  - If active project exists: default to **project** scope
  - If no active project: default to **global** scope

**Display scope banner:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Editing: Project Config
 Project: critical-feature
 (overrides global for this project only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

or:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Editing: Global Config
 (applies to all projects)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Determine config file path:**

```bash
if [ "$SCOPE" = "global" ]; then
  CONFIG_FILE=".planning/config.json"
  SCOPE_DISPLAY="Global Config (applies to all projects)"
else
  # Project scope
  if [ -z "$ACTIVE_PROJECT" ]; then
    echo "Error: No active project set. Use --global or switch to a project."
    exit 1
  fi
  CONFIG_FILE=".planning/projects/$ACTIVE_PROJECT/config.json"
  SCOPE_DISPLAY="Project Config: $ACTIVE_PROJECT (overrides global)"
fi
```

## 2. Read Current Config

**Use merged config resolution to show current effective values:**

```bash
# Read both global and project configs
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
PROJECT_CONFIG="{}"

if [ "$MULTI_PROJECT" = "true" ] && [ -n "$ACTIVE_PROJECT" ]; then
  PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"
  if [ -f "$PROJECT_CONFIG_PATH" ]; then
    PROJECT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
  fi
fi

# Shallow merge to get effective config
MERGED_CONFIG=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Extract current values (use flat key access)
MODEL_PROFILE=$(echo "$MERGED_CONFIG" | jq -r '.model_profile // "balanced"')
RESEARCH=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.research"] // true')
PLAN_CHECK=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.plan_check"] // true')
VERIFIER=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.verifier"] // true')
BRANCHING=$(echo "$MERGED_CONFIG" | jq -r '.["git.branching_strategy"] // "none"')
```

**Track provenance** (which values are from global vs project):

```bash
# For display purposes, determine source of each value
function get_source() {
  local KEY="$1"
  if [ "$MULTI_PROJECT" = "true" ] && [ -n "$PROJECT_CONFIG" ]; then
    VALUE=$(echo "$PROJECT_CONFIG" | jq -r ".\"$KEY\"" 2>/dev/null)
    if [ "$VALUE" != "null" ]; then
      echo "project override"
      return
    fi
  fi
  echo "global"
}
```

Parse current values:
- `model_profile` — which model each agent uses (default: `balanced`)
- `workflow.research` — spawn researcher during plan-phase (default: `true`)
- `workflow.plan_check` — spawn plan checker during plan-phase (default: `true`)
- `workflow.verifier` — spawn verifier during execute-phase (default: `true`)
- `git.branching_strategy` — branching approach (default: `"none"`)

## 3. Present Settings

Use AskUserQuestion with current values shown:

```
AskUserQuestion([
  {
    question: "Which model profile for agents?",
    header: "Model",
    multiSelect: false,
    options: [
      { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
      { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
      { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
    ]
  },
  {
    question: "Spawn Plan Researcher? (researches domain before planning)",
    header: "Research",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Research phase goals before planning" },
      { label: "No", description: "Skip research, plan directly" }
    ]
  },
  {
    question: "Spawn Plan Checker? (verifies plans before execution)",
    header: "Plan Check",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify plans meet phase goals" },
      { label: "No", description: "Skip plan verification" }
    ]
  },
  {
    question: "Spawn Execution Verifier? (verifies phase completion)",
    header: "Verifier",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify must-haves after execution" },
      { label: "No", description: "Skip post-execution verification" }
    ]
  },
  {
    question: "Git branching strategy?",
    header: "Branching",
    multiSelect: false,
    options: [
      { label: "None (Recommended)", description: "Commit directly to current branch" },
      { label: "Per Phase", description: "Create branch for each phase (gsd/phase-{N}-{name})" },
      { label: "Per Milestone", description: "Create branch for entire milestone (gsd/{version}-{name})" }
    ]
  }
])
```

**Pre-select based on current config values.**

## 4. Update Config

**Write to correct config file based on scope:**

```bash
# Read current config from target file
if [ "$SCOPE" = "global" ]; then
  CURRENT_CONFIG=$(cat .planning/config.json)
else
  # Project scope
  PROJECT_CONFIG_PATH=".planning/projects/$ACTIVE_PROJECT/config.json"

  # Lazy creation: create file if doesn't exist
  if [ ! -f "$PROJECT_CONFIG_PATH" ]; then
    echo "{}" > "$PROJECT_CONFIG_PATH"
    echo "Created project config (first override)"
  fi

  CURRENT_CONFIG=$(cat "$PROJECT_CONFIG_PATH")
fi

# Update config with new values (use flat dot-notation keys)
UPDATED_CONFIG=$(echo "$CURRENT_CONFIG" | jq \
  --arg profile "$MODEL_PROFILE_NEW" \
  --argjson research "$RESEARCH_NEW" \
  --argjson plan_check "$PLAN_CHECK_NEW" \
  --argjson verifier "$VERIFIER_NEW" \
  --arg branching "$BRANCHING_NEW" \
  '.model_profile = $profile |
   .["workflow.research"] = $research |
   .["workflow.plan_check"] = $plan_check |
   .["workflow.verifier"] = $verifier |
   .["git.branching_strategy"] = $branching')

# Write back to config file
echo "$UPDATED_CONFIG" > "$CONFIG_FILE"
```

**Important for project scope:** Only write the keys being changed (sparse config). Don't copy entire global config. The jq command above updates existing keys or adds them if missing, preserving any other project overrides.

**Flat key format:** Use dot-notation keys like `"workflow.research"` and `"git.branching_strategy"` to ensure proper shallow merge behavior when configs are resolved.

## 5. Confirm Changes

**Display with scope and provenance information:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► SETTINGS UPDATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scope: {Global / Project: <name>}

| Setting              | Value | Source |
|----------------------|-------|--------|
| Model Profile        | {quality/balanced/budget} | {global/project} |
| Plan Researcher      | {On/Off} | {global/project} |
| Plan Checker         | {On/Off} | {global/project} |
| Execution Verifier   | {On/Off} | {global/project} |
| Git Branching        | {None/Per Phase/Per Milestone} | {global/project} |

These settings apply to future /gsd:plan-phase and /gsd:execute-phase runs.

Quick commands:
- /gsd:settings --global — edit global config
- /gsd:settings --project — edit project config
- /gsd:set-profile <profile> — switch model profile
- /gsd:plan-phase --research — force research
- /gsd:plan-phase --skip-research — skip research
- /gsd:plan-phase --skip-verify — skip plan check
```

**Note about scope switching:**

If in project scope, remind user:
```
Tip: These overrides apply only to this project.
Use /gsd:settings --global to change defaults for all projects.
```

If in global scope with multi-project active, remind user:
```
Tip: These are global defaults.
Use /gsd:settings --project to override for current project only.
```

</process>

<success_criteria>
- [ ] Environment validated (.planning/config.json exists)
- [ ] Multi-project mode detected
- [ ] Correct config scope identified (global or project)
- [ ] Scope displayed clearly to user
- [ ] Current merged config read with provenance tracking
- [ ] User presented with 5 settings (profile + 3 workflow toggles + git branching)
- [ ] Config updated with flat dot-notation keys in correct scope file
- [ ] Project config created lazily on first project-scope edit
- [ ] Changes confirmed to user with scope and source information
- [ ] Scope switching tips provided
</success_criteria>
