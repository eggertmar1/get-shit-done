# Phase 4: Integration - Research

**Researched:** 2026-02-07
**Domain:** Command integration, cross-runtime compatibility, gitignore configuration
**Confidence:** HIGH

## Summary

Phase 4 integrates multi-project path resolution into all existing GSD commands, orchestrators, agents, and workflows. The foundation has already been established: Phase 1 created path resolution references, Phase 2 created project management commands with validation patterns, and Phase 3 implemented configuration resolution. This phase applies these patterns consistently across the entire codebase.

The standard approach is straightforward: add reference imports to execution contexts, copy proven validation snippets, replace hardcoded `.planning/` paths with `$PROJECT_BASE/` variables, and handle error states gracefully. The checklist from Phase 1 identifies all 24 components needing updates (12 workflows, 11 agents, 1 command section).

A critical finding: `.active` is currently tracked by git but should be gitignored to prevent merge conflicts. This must be fixed before Phase 4 completion.

**Primary recommendation:** Use the established validation pattern from `active-project-validation.md` for all command updates. Verify each component by checking that it reads/writes to the correct project directory. Add `.active` to `.gitignore` before any other work. Test in both flat and nested structures to ensure backwards compatibility.

## Standard Stack

### Core
| Component | Version | Purpose | Why Standard |
|-----------|---------|---------|--------------|
| Bash scripts | Built-in | Path resolution and validation | Established in Phase 1, zero dependencies, works across all runtimes |
| Markdown references | N/A | Reference documentation | GSD's native documentation pattern |
| @-notation imports | N/A | Reference inclusion in execution contexts | GSD's established context pattern |

### Supporting
| Tool | Version | Purpose | When to Use |
|------|---------|---------|-------------|
| jq | 1.6+ | JSON manipulation in bash scripts | Config merging, validation |
| git | 2.x+ | Version control operations | Already required by GSD |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Bash validation | Node.js validation | Adds runtime dependency, breaks zero-dependency principle |
| Manual path construction | Helper library | Over-engineering, pattern is simple enough |
| Per-file gitignore | `.gitignore` at root | Scattered configuration vs centralized |

**Installation:**
```bash
# No installation needed - all tools are built-in or already required
```

## Architecture Patterns

### Recommended Integration Structure
```
Phase 4 Integration Layers:
1. Reference Layer    — path-resolution.md, active-project-validation.md, config-resolution.md
2. Command Layer      — commands/gsd/*.md updated with validation
3. Workflow Layer     — get-shit-done/workflows/*.md with path resolution
4. Agent Layer        — agents/*.md with execution context references
5. Gitignore Layer    — .planning/.gitignore ensures .active is local-only
```

### Pattern 1: Command Integration
**What:** Add validation snippet to command <process> section, replace paths with PROJECT_BASE
**When to use:** Any command that reads/writes project-specific files

**Example:**
```markdown
<execution_context>
@get-shit-done/references/path-resolution.md
@get-shit-done/references/active-project-validation.md
</execution_context>

<process>
## 1. Validate Active Project

```bash
# Check if multi-project structure exists
if [ -d .planning/projects/ ]; then
  # Multi-project mode - need active project
  if [ ! -f .planning/.active ]; then
    echo "No active project set."
    echo "Available projects:"
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    echo "Select a project with: /gsd:switch-project <name>"
    exit 1
  fi

  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')

  if [ -z "$ACTIVE_PROJECT" ] || [ ! -d ".planning/projects/$ACTIVE_PROJECT" ]; then
    echo "Error: Active project invalid or not found."
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    exit 1
  fi

  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"
else
  # Flat structure - use root
  PROJECT_BASE=".planning"
fi
```

## 2. Use PROJECT_BASE Throughout

```bash
# OLD: cat .planning/STATE.md
# NEW:
cat "$PROJECT_BASE/STATE.md"

# OLD: ls .planning/phases/
# NEW:
ls "$PROJECT_BASE/phases/"
```
</process>
```

**Source:** Phase 2 Plan 03 - active-project-validation.md

### Pattern 2: Workflow Integration
**What:** Import path resolution reference, detect structure early, resolve all paths
**When to use:** Workflows that orchestrate multiple file operations

**Example:**
```markdown
<execution_context>
@get-shit-done/references/path-resolution.md
@get-shit-done/references/shared-paths.md
[...existing context...]
</execution_context>

<process>
**Step 1: Resolve paths**

Detect structure and set variables once at workflow start:

```bash
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"
else
  PROJECT_BASE=".planning"
fi

# Shared paths (always at root)
CODEBASE_DIR=".planning/codebase"
GLOBAL_CONFIG=".planning/config.json"

# Project-specific paths
STATE_FILE="$PROJECT_BASE/STATE.md"
ROADMAP_FILE="$PROJECT_BASE/ROADMAP.md"
PHASES_DIR="$PROJECT_BASE/phases"
```

**Step 2+:** Use variables throughout workflow
</process>
```

**Source:** Phase 1 - path-resolution.md usage patterns

### Pattern 3: Agent Integration
**What:** Add execution context references, agent uses resolver in commands
**When to use:** Agents that create artifacts in project directories

**Example:**
```markdown
<execution_context>
@get-shit-done/references/path-resolution.md
@get-shit-done/references/shared-paths.md
@.planning/PROJECT.md
@.planning/ROADMAP.md
[...other context...]
</execution_context>

<process>
# Agent spawns with execution context loaded
# All @ references are automatically resolved via path-resolution.md
# Agent writes to PROJECT_BASE paths passed by orchestrator
</process>
```

**Source:** Existing agent pattern from GSD system

### Pattern 4: Gitignore Configuration
**What:** Ensure `.active` and other local-only files are gitignored
**When to use:** Phase 4 setup, before any integration work

**Example:**
```bash
# Create .planning/.gitignore if missing
if [ ! -f .planning/.gitignore ]; then
  cat > .planning/.gitignore << 'EOF'
# Local-only project state
.active

# OS artifacts
.DS_Store

# Temporary files
*.tmp
EOF
  git add .planning/.gitignore
  git commit -m "chore: gitignore .active for multi-project support"
fi

# If .active is tracked, untrack it
if git ls-files --error-unmatch .planning/.active 2>/dev/null; then
  git rm --cached .planning/.active
  git commit -m "chore: untrack .active (gitignored)"
fi
```

**Source:** Git best practices, Phase 1 shared-paths.md

### Anti-Patterns to Avoid

- **Per-component path resolution:** Don't implement different resolution logic in each file - use the reference pattern consistently
- **Skipping validation:** Every command that needs project context must validate active project, not assume it exists
- **Hardcoding paths after integration:** After adding PROJECT_BASE, don't mix with hardcoded `.planning/` paths
- **Forgetting shared paths:** codebase/ and config.json always resolve to root, never through PROJECT_BASE

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Path resolution logic | Custom path builder | @path-resolution.md pattern | Already tested, handles flat + nested structures |
| Active project validation | Custom validation | @active-project-validation.md snippet | Consistent error messages, proven pattern |
| Config merging | Custom JSON merge | jq shallow merge | Handles dot-notation keys correctly |
| Gitignore management | Manual git commands | Verified gitignore pattern | Prevents edge cases (already tracked files) |

**Key insight:** Phase 1-3 created all the patterns needed for integration. Don't create new approaches - copy the existing proven patterns.

## Common Pitfalls

### Pitfall 1: Forgetting Backwards Compatibility
**What goes wrong:** Command only works in nested structure, breaks flat repos
**Why it happens:** Testing only in multi-project mode
**How to avoid:**
- Always wrap multi-project logic in `if [ -d .planning/projects/ ]`
- Test in both flat structure (no projects/ dir) and nested structure
- Use PROJECT_BASE variable that works in both modes
**Warning signs:** Command works after `new-project` but fails in fresh clone

### Pitfall 2: .active File Race Conditions
**What goes wrong:** Multiple operations read stale `.active` content
**Why it happens:** Command switches project mid-execution, another command reads old state
**How to avoid:**
- Read `.active` once at start of command, cache in variable
- Don't write to `.active` in commands that also use it
- Only switch-project command should write `.active`
**Warning signs:** "Project not found" errors after successful switch

### Pitfall 3: Mixed Path Resolution
**What goes wrong:** Some paths use PROJECT_BASE, others hardcoded
**Why it happens:** Incomplete integration, missing some file operations
**How to avoid:**
- Search for all `.planning/` references in file
- Replace all project-specific paths with PROJECT_BASE
- Keep only codebase/ and config.json at root
- Use checklist to verify each path is resolved
**Warning signs:** Files created in wrong location, operations succeed but write to flat structure

### Pitfall 4: Gitignore Added After .active Tracked
**What goes wrong:** `.active` appears in git status despite being gitignored
**Why it happens:** Git tracks files added before gitignore, ignore rules don't apply retroactively
**How to avoid:**
- Check if .active is tracked: `git ls-files .planning/.active`
- If tracked: `git rm --cached .planning/.active`
- Commit removal before adding to gitignore
- Verify: `git check-ignore -v .planning/.active` should show rule
**Warning signs:** `.active` in git status, merge conflicts on branch switches

### Pitfall 5: Runtime-Specific Bash Differences
**What goes wrong:** Command works in Claude Code but fails in OpenCode/Gemini
**Why it happens:** Different bash versions, shell configurations, or tool availability
**How to avoid:**
- Use POSIX-compliant bash syntax (no bashisms like `[[` vs `[`)
- Check tool availability: `command -v jq >/dev/null 2>&1 || echo "jq not found"`
- Test read operations: `cat file 2>/dev/null || echo "{}"`
- Avoid runtime-specific environment variables
**Warning signs:** Works locally but fails in CI, or works in one runtime but not others

## Code Examples

Verified patterns from Phase 1-3 implementation:

### Reading Project File with Validation
```bash
# Source: active-project-validation.md + path-resolution.md
# Full validation + read pattern

# Validate and resolve
if [ -d .planning/projects/ ]; then
  if [ ! -f .planning/.active ]; then
    echo "No active project set. Run: /gsd:switch-project <name>"
    exit 1
  fi
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"
else
  PROJECT_BASE=".planning"
fi

# Read project file
if [ -f "$PROJECT_BASE/STATE.md" ]; then
  CONTENT=$(cat "$PROJECT_BASE/STATE.md")
else
  echo "STATE.md not found in project"
  exit 1
fi
```

### Writing to Phase Directory
```bash
# Source: Phase 1 path-resolution.md usage patterns
# Create phase artifact in correct location

# Resolve base path
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"
else
  PROJECT_BASE=".planning"
fi

# Create phase directory and file
PHASE_DIR="$PROJECT_BASE/phases/05-deployment"
mkdir -p "$PHASE_DIR"
echo "Phase 5 research content" > "$PHASE_DIR/05-RESEARCH.md"

echo "Created: $PHASE_DIR/05-RESEARCH.md"
```

### Accessing Shared Codebase Map
```bash
# Source: shared-paths.md
# Shared files always at root, no PROJECT_BASE needed

# Read shared codebase analysis
if [ -d .planning/codebase/ ]; then
  STACK=$(cat .planning/codebase/STACK.md)
  ARCH=$(cat .planning/codebase/ARCHITECTURE.md)
else
  echo "No codebase map. Run: /gsd:map-codebase"
  exit 1
fi

# Shared paths never use PROJECT_BASE
```

### Config Resolution with Merge
```bash
# Source: Phase 3 config-resolution.md
# Merge global and project configs

# Read global config
GLOBAL_CONFIG=$(cat .planning/config.json 2>/dev/null || echo "{}")

# Read project config if in multi-project mode
PROJECT_CONFIG="{}"
if [ -d .planning/projects/ ]; then
  ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
  if [ -n "$ACTIVE_PROJECT" ] && [ -f ".planning/projects/$ACTIVE_PROJECT/config.json" ]; then
    PROJECT_CONFIG=$(cat ".planning/projects/$ACTIVE_PROJECT/config.json")
  fi
fi

# Shallow merge (project overrides global)
MERGED_CONFIG=$(jq -s '.[0] * .[1]' <(echo "$GLOBAL_CONFIG") <(echo "$PROJECT_CONFIG"))

# Extract values using flat key notation
MODE=$(echo "$MERGED_CONFIG" | jq -r '.mode // "yolo"')
RESEARCH=$(echo "$MERGED_CONFIG" | jq -r '.["workflow.research"] // true')
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hardcoded `.planning/` paths | Dynamic PROJECT_BASE resolution | Phase 1 (2026-02-04) | Enables multi-project support |
| Global-only config | Global + per-project overrides | Phase 3 (2026-02-07) | Project-specific settings without duplication |
| No active project tracking | `.active` file (gitignored) | Phase 2 (2026-02-06) | Machine-local project selection |
| Manual path checks | Validation reference pattern | Phase 2 (2026-02-06) | Consistent error handling |

**Deprecated/outdated:**
- Direct `.planning/PROJECT.md` references without resolution - Use `$PROJECT_BASE/PROJECT.md`
- Assuming flat structure - Always check for `projects/` directory
- Single config.json - Use global config with project overrides

## Runtime Compatibility

### Multi-Runtime Support
GSD works across three runtimes with different characteristics:

| Runtime | Config Location | Context Window | Key Differences |
|---------|----------------|----------------|-----------------|
| Claude Code | `~/.claude/` | 200,000 tokens | Proprietary, semantic project graph |
| OpenCode | `~/.config/opencode/` or `$XDG_CONFIG_HOME/opencode/` | Varies by model | Open-source, 75+ LLM providers |
| Gemini CLI | `~/.gemini/` | 1,000,000 tokens | Open-source (Apache 2.0), massive context |

**Source:** GSD README.md, .planning/codebase/STACK.md, [Claude Code vs Gemini CLI comparison](https://shipyard.build/blog/claude-code-vs-gemini-cli/)

### Runtime-Agnostic Patterns

All Phase 4 integration must work across runtimes:

1. **Config detection:** Use environment variables to find config directories
   - Claude: `CLAUDE_CONFIG_DIR` → `~/.claude/`
   - OpenCode: `OPENCODE_CONFIG_DIR` → `XDG_CONFIG_HOME/opencode/` → `~/.config/opencode/`
   - Gemini: `GEMINI_CONFIG_DIR` → `~/.gemini/`

2. **Bash compatibility:** All runtimes execute bash commands
   - Use POSIX-compliant syntax (works on all systems)
   - No runtime-specific shell features
   - Test on macOS (darwin), Linux, Windows (Git Bash)

3. **File operations:** Standard fs operations work everywhere
   - Read/write/mkdir/ls all work identically
   - Git operations work identically
   - Path resolution is pure bash logic

4. **No runtime detection needed:** Path resolution doesn't depend on which runtime executes it
   - Same `.planning/` structure across all runtimes
   - Same validation patterns work everywhere
   - Same error messages shown to users

### Verification Strategy

**Per-runtime testing:** Not needed for path resolution (runtime-agnostic)

**Structure testing:** Required for both flat and nested structures:
- Test with no `projects/` directory (flat/legacy)
- Test with `projects/` directory and `.active` file (nested/multi-project)
- Test error states (missing `.active`, invalid project name)

## Open Questions

### Question 1: Should commands auto-migrate on first use?
**What we know:** Phase 2 handles migration only on explicit `/gsd:new-project` command
**What's unclear:** Should other commands detect flat structure and offer migration?
**Recommendation:** Keep explicit migration only. Don't surprise users with automatic structure changes. Document migration requirement clearly in error messages.

### Question 2: How to handle .active in multi-user scenarios?
**What we know:** `.active` is gitignored, machine-local
**What's unclear:** What if two users run GSD simultaneously on same machine (different shells)?
**Recommendation:** Current single-file approach is fine for typical use. If concurrent usage becomes issue, add lockfile pattern. Not worth complexity pre-emptively.

### Question 3: Should verification be automated or manual?
**What we know:** CONTEXT.md allows Claude to choose per command
**What's unclear:** Which commands benefit from automated script vs manual walkthrough?
**Recommendation:**
- High-priority workflows (execute-plan, resume-project): Automated script
- Medium-priority commands: Manual walkthrough with checklist
- Low-priority/rarely-used: Code inspection only
- Let executor decide based on complexity

## Integration Checklist Status

From Phase 1 command-integration-checklist.md:

**Total components:** 24 (12 workflows + 11 agents + 1 command section)

**Currently integrated:** 3 commands (new-project, switch-project, list-projects, archive-project)

**Status:**
- ✓ Foundation established (Phase 1)
- ✓ Project commands created (Phase 2)
- ✓ Config resolution implemented (Phase 3)
- ⚠️ .active NOT gitignored (must fix)
- ○ 0/24 existing components integrated (Phase 4 work)

**Priorities for Phase 4:**
1. HIGH: Gitignore .active file (CRITICAL - prevents merge conflicts)
2. HIGH: execute-plan.md, resume-project.md, execute-phase.md (core workflows)
3. HIGH: gsd-executor.md, gsd-planner.md, gsd-roadmapper.md (core agents)
4. MEDIUM: Other workflows and agents from checklist
5. LOW: Verification-only workflows (verify-phase.md, diagnose-issues.md)

## Verification Approach

Per CONTEXT.md decisions, executor chooses verification method per command:

### Automated Verification Script
**When to use:** High-priority workflows with complex file operations

**Approach:**
```bash
# verification-script.sh
# For each command:
# 1. Create test flat structure
# 2. Run command, verify writes to .planning/
# 3. Migrate to nested structure
# 4. Run command, verify writes to projects/<active>/
# 5. Test with missing .active, verify error handling
```

### Manual Walkthrough
**When to use:** Medium-priority commands with straightforward paths

**Approach:**
1. Read command file
2. Identify all `.planning/` references
3. Verify each is wrapped in structure detection
4. Verify PROJECT_BASE is used consistently
5. Verify error handling for missing .active
6. Mark complete in checklist

### Code Inspection
**When to use:** Low-priority commands, rarely-used features

**Approach:**
1. Search for hardcoded `.planning/` strings
2. Verify @ references include path-resolution.md
3. Spot-check one or two key file operations
4. Trust pattern application, don't test every path

## Sources

### Primary (HIGH confidence)
- `/Users/eggert/Documents/get-shit-done/get-shit-done/references/path-resolution.md` - Path resolution algorithm (Phase 1)
- `/Users/eggert/Documents/get-shit-done/get-shit-done/references/shared-paths.md` - Shared vs project-specific paths (Phase 1)
- `/Users/eggert/Documents/get-shit-done/get-shit-done/references/active-project-validation.md` - Validation pattern (Phase 2)
- `/Users/eggert/Documents/get-shit-done/get-shit-done/references/config-resolution.md` - Config merging (Phase 3)
- `.planning/projects/multi-project-support/phases/01-foundation-migration/command-integration-checklist.md` - Integration scope
- `.planning/projects/multi-project-support/phases/02-project-commands/02-RESEARCH.md` - Project command patterns
- `.planning/codebase/STACK.md` - Runtime environment analysis
- Current git status - .active is tracked (verified via `git ls-files`)

### Secondary (MEDIUM confidence)
- [Claude Code vs Gemini CLI comparison](https://shipyard.build/blog/claude-code-vs-gemini-cli/) - Runtime differences
- [AI Coding Tools Guide](https://senrecep.medium.com/ai-coding-tools-the-complete-guide-to-claude-code-opencode-modern-development-eb9da4477dc1) - Runtime characteristics
- [Top CLI Coding Agents 2026](https://pinggy.io/blog/top_cli_based_ai_coding_agents/) - Runtime comparison

### Tertiary (LOW confidence)
- None - all findings verified with local codebase or official sources

## Metadata

**Confidence breakdown:**
- Integration patterns: HIGH - Established in Phases 1-3, proven working
- Runtime compatibility: HIGH - Verified via codebase analysis and documentation
- Gitignore status: HIGH - Verified via git commands (CRITICAL: .active is tracked)
- Verification approach: MEDIUM - Framework clear, specifics are executor discretion

**Research date:** 2026-02-07
**Valid until:** 90 days (stable domain - path resolution patterns unlikely to change)

**Critical finding:** `.active` file is currently tracked by git. This MUST be fixed before integration work begins to prevent merge conflicts in multi-user scenarios.

**Phase 4 readiness:** All foundations established. Can proceed immediately after gitignore fix.
