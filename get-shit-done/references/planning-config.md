<planning_config>

Configuration options for `.planning/` directory behavior.

<config_schema>

## Flattened Dot-Notation Format

GSD uses a flattened config structure with dot-notation keys instead of nested objects. This prevents shallow merge issues when per-project configs override global defaults.

**Complete config schema:**

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

**All available options:**

| Option | Default | Description |
|--------|---------|-------------|
| `mode` | `"yolo"` | Workflow execution style: `"yolo"`, `"careful"`, or `"strategic"` |
| `depth` | `"quick"` | Planning depth: `"quick"`, `"standard"`, or `"comprehensive"` |
| `parallelization` | `true` | Execute plans in parallel when possible |
| `commit_docs` | `true` | Whether to commit planning artifacts to git |
| `model_profile` | `"balanced"` | Model selection: `"quality"`, `"balanced"`, or `"budget"` |
| `workflow.research` | `true` | Spawn researcher during plan-phase |
| `workflow.plan_check` | `true` | Spawn plan checker during plan-phase |
| `workflow.verifier` | `true` | Spawn verifier during execute-phase |
| `planning.search_gitignored` | `false` | Add `--no-ignore` to broad rg searches (global-only) |
| `git.branching_strategy` | `"none"` | Git branching approach: `"none"`, `"phase"`, or `"milestone"` |
| `git.phase_branch_template` | `"gsd/phase-{phase}-{slug}"` | Branch template for phase strategy |
| `git.milestone_branch_template` | `"gsd/{milestone}-{slug}"` | Branch template for milestone strategy |

**Backwards compatibility:** Existing configs with nested format (e.g., `"workflow": {"research": true}`) still work, but flat format is recommended for new configs and required for proper per-project overrides.

</config_schema>

<multi_project_config>

## Multi-Project Configuration

GSD supports hierarchical configuration with global defaults and per-project overrides.

**Two config scopes:**

1. **Global config** — `.planning/config.json` (provides defaults for all projects)
2. **Project config** — `.planning/projects/<name>/config.json` (sparse overrides)

**Resolution:** Shallow merge with project config values winning for matching keys.

**Example:**

Global config (`.planning/config.json`):
```json
{
  "mode": "yolo",
  "model_profile": "balanced",
  "workflow.research": true,
  "workflow.plan_check": true,
  "workflow.verifier": true
}
```

Project config (`.planning/projects/critical-feature/config.json`):
```json
{
  "mode": "careful",
  "model_profile": "quality"
}
```

Effective config for `critical-feature` project:
```json
{
  "mode": "careful",             // ← from project (overridden)
  "model_profile": "quality",    // ← from project (overridden)
  "workflow.research": true,     // ← from global (inherited)
  "workflow.plan_check": true,   // ← from global (inherited)
  "workflow.verifier": true      // ← from global (inherited)
}
```

**Key benefits of flat format:** Overriding `"mode"` in project config doesn't affect `"workflow.research"` or other settings. Each dot-notation key is independent.

**For complete resolution algorithm:** See @config-resolution.md

**Global-only setting:** `planning.search_gitignored` cannot be overridden per-project (affects repo-level search behavior).

</multi_project_config>

<commit_docs_behavior>

**When `commit_docs: true` (default):**
- Planning files committed normally
- SUMMARY.md, STATE.md, ROADMAP.md tracked in git
- Full history of planning decisions preserved

**When `commit_docs: false`:**
- Skip all `git add`/`git commit` for `.planning/` files
- User must add `.planning/` to `.gitignore`
- Useful for: OSS contributions, client projects, keeping planning private

**Checking the config (multi-project resolution):**

```bash
# Multi-project config resolution
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
PROJECT_CONFIG="{}"
if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
    PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
  fi
fi

# Shallow merge and extract commit_docs
MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))
COMMIT_DOCS=$(echo "$MERGED" | jq -r '.commit_docs // true')

# Auto-detect gitignored (overrides config)
git check-ignore -q .planning 2>/dev/null && COMMIT_DOCS=false
```

**Auto-detection:** If `.planning/` is gitignored, `commit_docs` is automatically `false` regardless of config.json. This prevents git errors when users have `.planning/` in `.gitignore`.

**Conditional git operations:**

```bash
if [ "$COMMIT_DOCS" = "true" ]; then
  git add .planning/STATE.md
  git commit -m "docs: update state"
fi
```

</commit_docs_behavior>

<search_behavior>

**When `search_gitignored: false` (default):**
- Standard rg behavior (respects .gitignore)
- Direct path searches work: `rg "pattern" .planning/` finds files
- Broad searches skip gitignored: `rg "pattern"` skips `.planning/`

**When `search_gitignored: true`:**
- Add `--no-ignore` to broad rg searches that should include `.planning/`
- Only needed when searching entire repo and expecting `.planning/` matches

**Note:** Most GSD operations use direct file reads or explicit paths, which work regardless of gitignore status.

</search_behavior>

<setup_uncommitted_mode>

To use uncommitted mode:

1. **Set config:**
   ```json
   "planning": {
     "commit_docs": false,
     "search_gitignored": true
   }
   ```

2. **Add to .gitignore:**
   ```
   .planning/
   ```

3. **Existing tracked files:** If `.planning/` was previously tracked:
   ```bash
   git rm -r --cached .planning/
   git commit -m "chore: stop tracking planning docs"
   ```

</setup_uncommitted_mode>

<branching_strategy_behavior>

**Branching Strategies:**

| Strategy | When branch created | Branch scope | Merge point |
|----------|---------------------|--------------|-------------|
| `none` | Never | N/A | N/A |
| `phase` | At `execute-phase` start | Single phase | User merges after phase |
| `milestone` | At first `execute-phase` of milestone | Entire milestone | At `complete-milestone` |

**When `git.branching_strategy: "none"` (default):**
- All work commits to current branch
- Standard GSD behavior

**When `git.branching_strategy: "phase"`:**
- `execute-phase` creates/switches to a branch before execution
- Branch name from `phase_branch_template` (e.g., `gsd/phase-03-authentication`)
- All plan commits go to that branch
- User merges branches manually after phase completion
- `complete-milestone` offers to merge all phase branches

**When `git.branching_strategy: "milestone"`:**
- First `execute-phase` of milestone creates the milestone branch
- Branch name from `milestone_branch_template` (e.g., `gsd/v1.0-mvp`)
- All phases in milestone commit to same branch
- `complete-milestone` offers to merge milestone branch to main

**Template variables:**

| Variable | Available in | Description |
|----------|--------------|-------------|
| `{phase}` | phase_branch_template | Zero-padded phase number (e.g., "03") |
| `{slug}` | Both | Lowercase, hyphenated name |
| `{milestone}` | milestone_branch_template | Milestone version (e.g., "v1.0") |

**Checking the config (multi-project resolution):**

```bash
# Multi-project config resolution
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")
PROJECT_CONFIG="{}"
if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
    PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
  fi
fi

# Shallow merge (use flat key access with bracket syntax for dot-notation)
MERGED=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Extract branching settings
BRANCHING_STRATEGY=$(echo "$MERGED" | jq -r '.["git.branching_strategy"] // "none"')
PHASE_BRANCH_TEMPLATE=$(echo "$MERGED" | jq -r '.["git.phase_branch_template"] // "gsd/phase-{phase}-{slug}"')
MILESTONE_BRANCH_TEMPLATE=$(echo "$MERGED" | jq -r '.["git.milestone_branch_template"] // "gsd/{milestone}-{slug}"')
```

**Branch creation:**

```bash
# For phase strategy
if [ "$BRANCHING_STRATEGY" = "phase" ]; then
  PHASE_SLUG=$(echo "$PHASE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
  BRANCH_NAME=$(echo "$PHASE_BRANCH_TEMPLATE" | sed "s/{phase}/$PADDED_PHASE/g" | sed "s/{slug}/$PHASE_SLUG/g")
  git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
fi

# For milestone strategy
if [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  MILESTONE_SLUG=$(echo "$MILESTONE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
  BRANCH_NAME=$(echo "$MILESTONE_BRANCH_TEMPLATE" | sed "s/{milestone}/$MILESTONE_VERSION/g" | sed "s/{slug}/$MILESTONE_SLUG/g")
  git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
fi
```

**Merge options at complete-milestone:**

| Option | Git command | Result |
|--------|-------------|--------|
| Squash merge (recommended) | `git merge --squash` | Single clean commit per branch |
| Merge with history | `git merge --no-ff` | Preserves all individual commits |
| Delete without merging | `git branch -D` | Discard branch work |
| Keep branches | (none) | Manual handling later |

Squash merge is recommended — keeps main branch history clean while preserving the full development history in the branch (until deleted).

**Use cases:**

| Strategy | Best for |
|----------|----------|
| `none` | Solo development, simple projects |
| `phase` | Code review per phase, granular rollback, team collaboration |
| `milestone` | Release branches, staging environments, PR per version |

</branching_strategy_behavior>

</planning_config>
