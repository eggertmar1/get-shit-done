---
name: gsd:archive-project
description: Archive a project to remove it from the active project list
allowed-tools:
  - Read
  - Bash
  - Write
  - AskUserQuestion
---

<objective>
Archive a project by moving it to `.planning/projects/.archive/`. Archived projects are hidden from `/gsd:list-projects` but preserved in git history.

**Usage:** `/gsd:archive-project [name]`

If [name] not provided, archives the active project.

Archive = removal from view. Projects can be restored by moving them back from .archive/.
</objective>

<execution_context>
@~/.claude/get-shit-done/references/path-resolution.md
</execution_context>

<process>

## Step 1: Validate Multi-Project Structure

```bash
if [ ! -d .planning/projects/ ]; then
  echo "Error: Multi-project structure not initialized"
  echo ""
  echo "Nothing to archive in single-project structure."
  exit 1
fi
```

## Step 2: Determine Target Project

If [name] argument provided:
- Use that as PROJECT_NAME

If no argument:
- Get active project from .planning/.active
- If no active project, show error and list available projects

```bash
if [ -z "$1" ]; then
  # No argument - use active project
  if [ ! -f .planning/.active ]; then
    echo "Error: No project specified and no active project set"
    echo ""
    echo "Usage: /gsd:archive-project <name>"
    echo ""
    echo "Available projects:"
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    exit 1
  fi
  PROJECT_NAME=$(cat .planning/.active | tr -d '[:space:]')
else
  PROJECT_NAME="$1"
fi
```

## Step 3: Validate Project Exists

```bash
if [ ! -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Error: Project '$PROJECT_NAME' not found"
  echo ""
  echo "Available projects:"
  ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
  exit 1
fi
```

## Step 4: Confirm Archive

Use AskUserQuestion:
- header: "Archive Project"
- question: "Archive project '{PROJECT_NAME}'? This hides it from project list but preserves it in .archive/"
- options:
  - "Yes, archive it" - Proceed with archive
  - "No, cancel" - Abort

## Step 5: Create Archive Directory

```bash
mkdir -p .planning/projects/.archive
```

## Step 6: Check if Archiving Active Project

```bash
CURRENT_ACTIVE=$(cat .planning/.active 2>/dev/null | tr -d '[:space:]')
IS_ACTIVE_PROJECT="no"
if [ "$CURRENT_ACTIVE" == "$PROJECT_NAME" ]; then
  IS_ACTIVE_PROJECT="yes"
fi
```

## Step 7: Move Project to Archive

Use git mv to preserve history:

```bash
git mv ".planning/projects/$PROJECT_NAME" ".planning/projects/.archive/$PROJECT_NAME"
```

## Step 8: Handle Active Project Removal

If archiving the active project:

```bash
if [ "$IS_ACTIVE_PROJECT" == "yes" ]; then
  # Clear .active file
  rm .planning/.active

  echo "Cleared active project (was archiving active project)"
fi
```

## Step 9: Commit Archive

```bash
git add .planning/.active 2>/dev/null || true  # May not exist
git commit -m "archive: $PROJECT_NAME"
```

## Step 10: Show Confirmation

```
Archived: {PROJECT_NAME}

Project moved to: .planning/projects/.archive/{PROJECT_NAME}/

To restore: git mv .planning/projects/.archive/{PROJECT_NAME} .planning/projects/{PROJECT_NAME}
```

If was active project, also show:
```
No active project set. Run /gsd:switch-project to select another.
```

</process>

<output>
- Project moved to `.planning/projects/.archive/`
- If archiving active project: `.planning/.active` cleared
- Git commit created for archive operation
</output>

<success_criteria>
- [ ] Project moved to .archive/ directory via git mv
- [ ] Git history preserved (git mv, not rm + add)
- [ ] Active project cleared if archiving active project
- [ ] User sees confirmation with restore instructions
- [ ] Invalid project shows helpful error
</success_criteria>
