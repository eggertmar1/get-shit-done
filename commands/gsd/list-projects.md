---
name: gsd:list-projects
description: List all projects in the multi-project structure with their status
allowed-tools:
  - Read
  - Bash
---

<objective>
Display all available projects with their current phase and progress.

**Usage:** `/gsd:list-projects [--verbose]`

Shows a table of projects with:
- Project name
- Current phase
- Progress percentage
- Active project indicator
</objective>

<execution_context>
@~/.claude/get-shit-done/references/path-resolution.md
</execution_context>

<process>

## Step 1: Check for Multi-Project Structure

```bash
if [ ! -d .planning/projects/ ]; then
  echo "No multi-project structure found."
  echo ""
  echo "This repository uses single-project structure."
  echo "Run /gsd:new-project to migrate to multi-project."
  exit 0
fi
```

## Step 2: Check for Empty Projects Directory

```bash
PROJECT_COUNT=$(ls -1d .planning/projects/*/ 2>/dev/null | grep -v '/\.' | wc -l | tr -d ' ')

if [ "$PROJECT_COUNT" -eq 0 ]; then
  echo "No projects found."
  echo ""
  echo "Create a project with: /gsd:new-project [name]"
  exit 0
fi
```

## Step 3: Get Active Project

```bash
ACTIVE_PROJECT=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
```

## Step 4: Build Project List

For each project directory (excluding dot-directories like .archive):

```bash
for PROJECT_DIR in .planning/projects/*/; do
  PROJECT_NAME=$(basename "$PROJECT_DIR")

  # Skip dot-directories (e.g., .archive)
  if [[ "$PROJECT_NAME" == .* ]]; then
    continue
  fi

  # Get current phase from STATE.md
  if [ -f "$PROJECT_DIR/STATE.md" ]; then
    PHASE=$(grep "^Phase:" "$PROJECT_DIR/STATE.md" | head -1 | sed 's/Phase: \([0-9]*\).*/\1/')
    if [ -z "$PHASE" ]; then
      PHASE="-"
    fi
  else
    PHASE="-"
  fi

  # Calculate progress from ROADMAP.md
  if [ -f "$PROJECT_DIR/ROADMAP.md" ]; then
    TOTAL=$(grep -c "^- \[" "$PROJECT_DIR/ROADMAP.md" 2>/dev/null || echo "1")
    COMPLETED=$(grep -c "^- \[x\]" "$PROJECT_DIR/ROADMAP.md" 2>/dev/null || echo "0")
    if [ "$TOTAL" -gt 0 ]; then
      PERCENT=$((100 * COMPLETED / TOTAL))
    else
      PERCENT=0
    fi
  else
    PERCENT=0
  fi

  # Mark active project
  if [ "$ACTIVE_PROJECT" == "$PROJECT_NAME" ]; then
    MARKER="*"
  else
    MARKER=" "
  fi

  # Store for display
  echo "$MARKER|$PROJECT_NAME|$PHASE|$PERCENT"
done
```

## Step 5: Display Table

Format output as table:

```
Projects:

| * | Name            | Phase | Progress |
|---|-----------------|-------|----------|
| * | my-project      | 3     | 45%      |
|   | feature-auth    | 1     | 10%      |
|   | api-refactor    | -     | 0%       |

* = active project

Total: 3 projects
```

The `*` indicates the active project. Table format used even for single project per CONTEXT.md decisions.

## Step 6: Handle --verbose Flag

If `--verbose` flag provided, also show:
- Full project path
- Last activity timestamp (from STATE.md)

```bash
if [ -f "$PROJECT_DIR/STATE.md" ]; then
  LAST_ACTIVITY=$(grep "^Last session:" "$PROJECT_DIR/STATE.md" | head -1 | sed 's/Last session: //')
fi
```

Verbose output adds columns:
- Path: `.planning/projects/{name}/`
- Last Activity: `{timestamp from STATE.md}`

</process>

<output>
Displays formatted table of all projects with status information.
</output>

<success_criteria>
- [ ] Lists all projects (excluding .archive)
- [ ] Shows phase number (or - if no STATE.md)
- [ ] Shows progress percentage (or 0% if no ROADMAP.md)
- [ ] Marks active project
- [ ] Uses table format even for single project
- [ ] Empty state shows helpful guidance
</success_criteria>
