# Phase 02: Project Commands - Research

**Researched:** 2026-02-04
**Domain:** CLI project management, state persistence, interactive prompts
**Confidence:** MEDIUM

## Summary

Phase 02 implements four commands for multi-project management: create, switch, list, and archive. The standard approach uses file-based state persistence with a `.active` file (gitignored) to track the current project, combined with directory-based project discovery from `.planning/projects/`.

Interactive prompting should use established Node.js libraries rather than custom implementations. Table formatting for list display has mature libraries with extensive customization. Git branch detection requires third-party packages (not in Node.js standard library).

**Key architectural insight:** The path resolution system from Phase 01 provides the foundation - commands build on existing detection logic rather than reimplementing it.

**Primary recommendation:** Use simple file I/O for state management (read/write `.active` file), delegate interactive prompts to Inquirer.js, and use cli-table3 for table formatting. Avoid custom state management abstractions.

## Standard Stack

The established libraries/tools for CLI project management:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js fs | Built-in | File I/O for .active and project listing | Native, zero dependencies, sufficient for file-based state |
| Bash scripts | Built-in | Path resolution, git operations | Already established in Phase 01, consistent with existing commands |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| inquirer.js | 8.x+ | Interactive prompts | Project selection when .active missing |
| @inquirer/prompts | Latest | Modern Inquirer API | Alternative to classic Inquirer (recommended by maintainers) |
| cli-table3 | 0.6.x | Table formatting | list-projects display with columns |
| current-git-branch | 1.1.x+ | Git branch detection | new-project command default naming |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Inquirer.js | Enquirer | Lighter weight but less ecosystem support |
| cli-table3 | table (npm) | More features but heavier, overkill for simple tables |
| File-based state | Database (SQLite) | Adds complexity, unnecessary for <100 projects |

**Installation:**
```bash
# Not needed - using bash scripts + built-in Node.js fs
# Optional if TypeScript implementation needed:
npm install inquirer cli-table3 current-git-branch
```

**Note:** GSD system is file-based (commands defined in `.md` files), not a Node.js package. These libraries are reference for if/when commands need programmatic implementations.

## Architecture Patterns

### Recommended Project Structure
```
.planning/
├── .active              # Current project name (gitignored, machine-local)
├── .gitignore           # Ignores .active
├── codebase/            # Shared across projects
├── config.json          # Global defaults
└── projects/
    ├── main-app/        # Project directories
    │   ├── PROJECT.md
    │   ├── ROADMAP.md
    │   └── phases/
    └── api-refactor/
```

### Pattern 1: File-Based Active Project Tracking
**What:** Single `.active` file contains the active project name as plain text
**When to use:** All multi-project operations need to know which project is active

**Example:**
```bash
# Source: Standard dotfile pattern + GSD path-resolution.md
# Read active project
if [ -f .planning/.active ]; then
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')
else
  # Prompt user to select project
  echo "No active project set"
fi

# Write active project (when switching)
echo "new-project-name" > .planning/.active
```

**Key characteristics:**
- Plain text, single line
- Whitespace trimmed when reading
- Gitignored to prevent merge conflicts
- Local to each developer's machine

### Pattern 2: Directory-Based Project Discovery
**What:** List projects by scanning `.planning/projects/` directory
**When to use:** Listing available projects, validating project existence

**Example:**
```bash
# Source: Standard UNIX directory listing pattern
# List all projects
ls -1 .planning/projects/

# Check if project exists
if [ -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Project exists"
fi

# Count projects
PROJECT_COUNT=$(ls -1d .planning/projects/*/ 2>/dev/null | wc -l | tr -d ' ')
```

### Pattern 3: Progress Calculation from Phase Tracking
**What:** Calculate completion percentage from checked/total requirements across phases
**When to use:** list-projects display showing progress per project

**Example:**
```bash
# Source: Project management progress calculation pattern
# Count completed requirements (checked boxes in ROADMAP.md)
COMPLETED=$(grep -c "^\- \[x\]" .planning/projects/$PROJECT/ROADMAP.md 2>/dev/null || echo "0")
# Count total requirements
TOTAL=$(grep -c "^\- \[[ x]\]" .planning/projects/$PROJECT/ROADMAP.md 2>/dev/null || echo "0")
# Calculate percentage
if [ "$TOTAL" -gt 0 ]; then
  PERCENT=$((100 * COMPLETED / TOTAL))
else
  PERCENT=0
fi
```

**Alternative:** Parse STATE.md for current phase + plan position, calculate based on phase count

### Pattern 4: Interactive Prompt for Missing State
**What:** When `.active` missing or invalid, prompt user to select project
**When to use:** Error handling in all GSD commands when no active project

**Example:**
```bash
# Source: CLI best practice - prompt for missing flags/args
# Detect missing .active
if [ ! -f .planning/.active ] || [ -z "$(cat .planning/.active | tr -d '[:space:]')" ]; then
  # List available projects
  echo "No active project set. Available projects:"
  ls -1 .planning/projects/ | nl

  # Prompt for selection (in real implementation, use Inquirer.js)
  read -p "Select project number: " PROJECT_NUM
  PROJECT_NAME=$(ls -1 .planning/projects/ | sed -n "${PROJECT_NUM}p")

  # Set as active
  echo "$PROJECT_NAME" > .planning/.active
fi
```

**Note:** For Claude Code commands, use AskUserQuestion tool for interactive prompts

### Pattern 5: Git Branch Name Detection
**What:** Get current git branch as default project name
**When to use:** new-project command when user doesn't provide [name]

**Example:**
```bash
# Source: Git integration pattern - multiple fallback approaches
# Method 1: Symbolic reference (most reliable)
BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)

# Method 2: Fallback - describe with tags
if [ -z "$BRANCH_NAME" ]; then
  BRANCH_NAME=$(git describe --tags --exact-match 2>/dev/null || git rev-parse --short HEAD 2>/dev/null)
fi

# Validation: Branch name exists and is not "HEAD" (detached state)
if [ -z "$BRANCH_NAME" ] || [ "$BRANCH_NAME" == "HEAD" ]; then
  echo "Error: Not on a git branch (detached HEAD)"
  # Prompt user for manual name
fi
```

**Pitfall:** Detached HEAD state (when not on a branch) should be detected and handled

### Pattern 6: Archive via Directory Rename
**What:** Archive project by moving directory to `.planning/projects/.archive/`
**When to use:** archive-project command to hide projects from list without deletion

**Example:**
```bash
# Source: Standard UNIX archive pattern with dot-prefix hiding
# Create archive directory if needed
mkdir -p .planning/projects/.archive

# Move project (preserving git history with git mv)
git mv .planning/projects/$PROJECT_NAME .planning/projects/.archive/$PROJECT_NAME

# Commit
git commit -m "archive: $PROJECT_NAME"

# If archived project was active, clear .active
if [ "$(cat .planning/.active 2>/dev/null)" == "$PROJECT_NAME" ]; then
  rm .planning/.active
fi
```

**Why .archive/ directory:**
- Keeps archived projects in git history
- Dot-prefix hides from casual directory listings
- Easy to restore: `git mv .archive/project projects/project`
- Alternative to deletion (non-destructive)

### Anti-Patterns to Avoid

- **Complex state management:** Don't build a state management library for simple read/write operations - file I/O is sufficient
- **Cached project lists:** Don't cache `.planning/projects/` listing - directory reads are fast (<1ms), cache adds stale data risk
- **Database for project state:** Don't use SQLite/JSON database - file-based discovery is simpler and git-friendly
- **Active project in config.json:** Don't store `.active` in config.json - config should be committed, active project is local-only

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Interactive prompts | Custom readline loops | Inquirer.js or @inquirer/prompts | Handles keyboard navigation, validation, multi-select, consistent UX |
| Table formatting | String concatenation with padding | cli-table3 | Unicode borders, column alignment, word wrapping, color support |
| Git branch detection | Parsing .git/HEAD manually | git symbolic-ref command or current-git-branch package | Handles edge cases (detached HEAD, packed-refs, worktrees) |
| Progress percentage display | Custom ASCII bars | Built-in table cells with emoji/text | Simple percentage in table cell sufficient for list view |

**Key insight:** CLI interaction patterns (prompts, tables) have mature libraries. The complexity is in edge cases (keyboard handling, terminal width, color detection) not the happy path.

## Common Pitfalls

### Pitfall 1: .active File Merged into Version Control
**What goes wrong:** Users commit `.active` file, causing merge conflicts when team members work on different projects
**Why it happens:** Forgetting to gitignore `.active` before creating it, or gitignore added after first commit
**How to avoid:** ALWAYS verify `.active` is gitignored BEFORE writing it (Phase 01-02 already established this pattern)
**Warning signs:** Git shows `.planning/.active` as untracked or modified file

```bash
# Prevention: Check gitignore BEFORE creating .active
if ! git check-ignore -q .planning/.active 2>/dev/null; then
  echo "⚠️ Warning: .active file is not gitignored"
  # Add to gitignore first
  echo ".active" >> .planning/.gitignore
fi
```

### Pitfall 2: Empty or Whitespace-Only .active File
**What goes wrong:** Commands fail with cryptic errors when `.active` contains only whitespace (newline, spaces)
**Why it happens:** Writing empty string or newline-only content, accidental file truncation
**How to avoid:** Always trim whitespace when reading, validate non-empty when writing
**Warning signs:** "Project '' not found" error, commands fail but `.active` file exists

```bash
# Prevention: Trim when reading
ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
if [ -z "$ACTIVE_PROJECT" ]; then
  # Treat empty as missing
fi

# Validation when writing
if [ -n "$NEW_PROJECT" ]; then
  echo "$NEW_PROJECT" > .planning/.active
else
  echo "Error: Project name cannot be empty"
fi
```

### Pitfall 3: Detached HEAD State for Git Branch Default
**What goes wrong:** `git symbolic-ref HEAD` fails when not on a branch (detached HEAD), causing new-project to fail without clear error
**Why it happens:** User is on a specific commit, tag, or remote branch without local tracking
**How to avoid:** Detect detached HEAD state and prompt user for manual project name
**Warning signs:** Git shows "HEAD detached at <commit>" in status

```bash
# Detection
BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)
if [ -z "$BRANCH_NAME" ]; then
  # Detached HEAD - cannot use branch as default
  echo "Not on a git branch. Please provide project name:"
  # Prompt user
fi
```

### Pitfall 4: Active Project Deletion Without Updating .active
**What goes wrong:** `.active` points to archived/deleted project, all GSD commands fail
**Why it happens:** archive-project command doesn't clear `.active` when archiving the active project
**How to avoid:** When archiving a project, check if it's the active project and clear `.active` if so
**Warning signs:** "Active project 'X' not found" error after archiving

```bash
# Prevention in archive-project
CURRENT_ACTIVE=$(cat .planning/.active 2>/dev/null)
if [ "$CURRENT_ACTIVE" == "$PROJECT_TO_ARCHIVE" ]; then
  echo "Archiving active project. Clearing .active file..."
  rm .planning/.active
  echo "Run /gsd:switch-project to select another project"
fi
```

### Pitfall 5: Progress Calculation with No ROADMAP.md
**What goes wrong:** list-projects fails or shows incorrect progress for projects without a roadmap
**Why it happens:** New projects created but not yet planned (via /gsd:new-project in nested structure)
**How to avoid:** Gracefully handle missing ROADMAP.md, display "Phase -, 0%" per CONTEXT.md decision
**Warning signs:** grep errors, division by zero errors

```bash
# Prevention
if [ -f ".planning/projects/$PROJECT/ROADMAP.md" ]; then
  # Calculate progress from ROADMAP
  TOTAL=$(grep -c "^\- \[[ x]\]" .planning/projects/$PROJECT/ROADMAP.md)
  COMPLETED=$(grep -c "^\- \[x\]" .planning/projects/$PROJECT/ROADMAP.md)
  PERCENT=$((100 * COMPLETED / TOTAL))
  PHASE="2" # Parse from STATE.md or ROADMAP
else
  # No roadmap yet
  PHASE="-"
  PERCENT=0
fi
```

### Pitfall 6: Race Conditions with Concurrent Project Switching
**What goes wrong:** Multiple Claude sessions writing `.active` simultaneously could corrupt file or cause inconsistent state
**Why it happens:** No file locking on `.active` writes, concurrent `/gsd:switch-project` commands
**How to avoid:** Atomic file writes (write to temp file, then move), but realistically this is rare (single-user tool)
**Warning signs:** `.active` contains multiple lines or corrupted content

**Decision:** Accept this risk as extremely low probability - GSD is single-user tool, concurrent sessions unlikely. If it becomes an issue, use atomic writes:

```bash
# Atomic write pattern (if needed later)
TEMP_FILE=".planning/.active.tmp.$$"
echo "$PROJECT_NAME" > "$TEMP_FILE"
mv "$TEMP_FILE" .planning/.active
```

## Code Examples

Verified patterns from official sources and GSD Phase 01 artifacts:

### Project Existence Validation
```bash
# Source: GSD path-resolution.md + standard UNIX pattern
PROJECT_NAME="$1"

# Validate project exists
if [ ! -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Error: Project '$PROJECT_NAME' not found"
  echo ""
  echo "Available projects:"
  ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
  exit 1
fi
```

### List Projects with Metadata
```bash
# Source: Combining path-resolution + progress tracking patterns
echo "Projects:"
echo ""

# Iterate over projects (exclude dotfile directories like .archive)
for PROJECT_DIR in .planning/projects/*/; do
  PROJECT_NAME=$(basename "$PROJECT_DIR")

  # Skip dotfile directories (e.g., .archive)
  if [[ "$PROJECT_NAME" == .* ]]; then
    continue
  fi

  # Get current phase from STATE.md
  if [ -f "$PROJECT_DIR/STATE.md" ]; then
    PHASE=$(grep "^## Current Position" -A 3 "$PROJECT_DIR/STATE.md" | grep "Phase:" | sed 's/.*Phase: //' | cut -d' ' -f1)
  else
    PHASE="-"
  fi

  # Calculate progress (simplified - see Pattern 3 for full implementation)
  if [ -f "$PROJECT_DIR/ROADMAP.md" ]; then
    TOTAL=$(grep -c "^- \[" "$PROJECT_DIR/ROADMAP.md" 2>/dev/null || echo "1")
    COMPLETED=$(grep -c "^- \[x\]" "$PROJECT_DIR/ROADMAP.md" 2>/dev/null || echo "0")
    PERCENT=$((100 * COMPLETED / TOTAL))
  else
    PERCENT=0
  fi

  # Check if active
  ACTIVE=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
  MARKER=""
  if [ "$ACTIVE" == "$PROJECT_NAME" ]; then
    MARKER="*"
  fi

  # Display (table formatting would use cli-table3 in real implementation)
  printf "  %s %-20s | Phase %-2s | %3d%%\n" "$MARKER" "$PROJECT_NAME" "$PHASE" "$PERCENT"
done
```

### Switch Project with Validation
```bash
# Source: GSD path-resolution.md error handling + state persistence pattern
PROJECT_NAME="$1"

# Verify multi-project structure exists
if [ ! -d .planning/projects/ ]; then
  echo "Error: Multi-project structure not initialized"
  echo "Run /gsd:new-project first"
  exit 1
fi

# Validate project exists
if [ ! -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Error: Project '$PROJECT_NAME' not found"
  echo ""
  echo "Available projects:"
  ls -1 .planning/projects/ | grep -v "^\."
  exit 1
fi

# Update .active file
echo "$PROJECT_NAME" > .planning/.active

echo "Switched to project: $PROJECT_NAME"
```

### Archive Project with Safety Checks
```bash
# Source: Archive pattern + active project validation
PROJECT_NAME="$1"

# Validate project exists
if [ ! -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Error: Project '$PROJECT_NAME' not found"
  exit 1
fi

# Create archive directory
mkdir -p .planning/projects/.archive

# Check if archiving active project
CURRENT_ACTIVE=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
if [ "$CURRENT_ACTIVE" == "$PROJECT_NAME" ]; then
  echo "⚠️  Archiving active project"
  rm .planning/.active
fi

# Move project (using git mv to preserve history)
git mv ".planning/projects/$PROJECT_NAME" ".planning/projects/.archive/$PROJECT_NAME"

# Commit
git add .planning/.active 2>/dev/null || true  # May not exist if was active
git commit -m "archive: $PROJECT_NAME"

echo "Archived: $PROJECT_NAME"
if [ "$CURRENT_ACTIVE" == "$PROJECT_NAME" ]; then
  echo ""
  echo "No active project set. Run /gsd:switch-project to select another."
fi
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Complex CLI frameworks (Commander, yargs) | Simple bash scripts with @inquirer for prompts | Ongoing in 2026 | Bash is sufficient for file operations, only need JS libraries for interactive UX |
| Cached directory listings | Direct filesystem reads | Always | Modern filesystems are fast enough that caching adds unnecessary complexity |
| JSON files for state | Plain text .active file | Dotfile pattern (decades old) | Single value doesn't need JSON overhead |
| Inquirer v8 (callbacks) | @inquirer/prompts (promises) | v9+ (2023+) | Modern async/await API is more ergonomic |

**Deprecated/outdated:**
- Inquirer classic API (still works but deprecated in favor of @inquirer/prompts)
- cli-table and cli-table2 (superseded by cli-table3 with Unicode support)

**Current best practice (2026):**
- File-based state for simplicity
- Bash for file operations
- JavaScript libraries only for complex UX (prompts, tables)
- Git integration for all file moves (preserves history)

## Open Questions

Things that couldn't be fully resolved:

1. **Progress calculation algorithm**
   - What we know: Can be calculated from completed/total requirements in ROADMAP, or from current phase position
   - What's unclear: Which method is more accurate? Phase-based assumes equal phase sizes (not realistic)
   - Recommendation: Use requirement-based calculation (ROADMAP checkboxes). Document that it's coarse-grained. Phase 04 can implement if detailed progress needed.

2. **Table column widths and formatting**
   - What we know: cli-table3 handles column sizing, but exact widths are Claude's discretion per CONTEXT.md
   - What's unclear: Optimal column widths for project names (could be long), how to truncate
   - Recommendation: Start with auto-width (cli-table3 default), add truncation if user feedback indicates need

3. **Empty state messaging**
   - What we know: Should guide user when no projects exist (Claude's discretion per CONTEXT.md)
   - What's unclear: Friendly vs minimal style preference
   - Recommendation: Test both styles in implementation, choose based on consistency with other GSD commands

4. **Concurrent project switching handling**
   - What we know: Race conditions possible if multiple Claude sessions switch projects simultaneously
   - What's unclear: Is atomic file writing worth the complexity for single-user tool?
   - Recommendation: Accept the risk for now (extremely low probability). If it becomes a real issue, add atomic writes (write to temp file + move)

## Sources

### Primary (HIGH confidence)
- GSD path-resolution.md - Path resolution algorithm and error handling patterns (existing artifact from Phase 01)
- GSD shared-paths.md - Path classification (shared vs project-specific) (existing artifact from Phase 01)
- [CLI Interface Guidelines](https://clig.dev/) - Interactive prompt best practices
- [Node.js CLI Apps Best Practices](https://github.com/lirantal/nodejs-cli-apps-best-practices) - State management and prompting patterns

### Secondary (MEDIUM confidence)
- [cli-table3 npm documentation](https://www.npmjs.com/package/cli-table3) - Table formatting library (verified: latest version 0.6.x, active maintenance)
- [Inquirer.js GitHub](https://github.com/SBoudrias/Inquirer.js) - Interactive prompts library (verified: modern @inquirer/prompts API recommended)
- [current-git-branch npm](https://www.npmjs.com/package/current-git-branch) - Git branch detection (verified: latest version 1.1.x)
- [Project progress calculation methods](https://twproject.com/blog/project-progress-calculation/) - Multiple calculation approaches for phase-based progress

### Tertiary (LOW confidence, marked for validation)
- WebSearch results on "multi-project CLI state management" - Focused more on frontend state (Redux, Zustand) than CLI tools, limited direct applicability
- WebSearch results on "CLI frameworks 2026" - General framework comparisons, not specific to project management patterns

### GSD-Specific (HIGH confidence)
- Phase 01-01 PLAN: Path resolution implementation (existing)
- Phase 01-02 PLAN: Migration workflow implementation (existing)
- commands/gsd/new-project.md: Migration detection and nested project creation (existing)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Bash + file I/O is established pattern in Phase 01, Node.js libraries well-documented
- Architecture: HIGH - File-based patterns are proven in GSD system, path resolution from Phase 01 provides foundation
- Pitfalls: MEDIUM - Most pitfalls inferred from general CLI development and path-resolution.md error handling, not specifically tested in GSD context
- Progress calculation: LOW - Multiple approaches possible, no single "standard" for multi-project CLI tools

**Research date:** 2026-02-04
**Valid until:** 30 days (stable domain - CLI patterns don't change rapidly)
**Validation needed:** Progress calculation algorithm choice should be validated with user preference or testing in Phase 04 integration
