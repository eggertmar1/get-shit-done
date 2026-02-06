---
name: gsd:switch-project
description: Switch active project context to a different project
allowed-tools:
  - Read
  - Bash
  - Write
  - AskUserQuestion
---

<objective>
Switch the active project to a different project in the multi-project structure.

**Usage:** `/gsd:switch-project <name>`

After switching, all GSD commands operate on the new active project.
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
  echo "This repository uses single-project structure."
  echo "Run /gsd:new-project to migrate to multi-project."
  exit 1
fi
```

If validation fails, exit with helpful message.

## Step 2: Parse Arguments

Extract project name from command arguments.

If no project name provided:
- Use AskUserQuestion to prompt for selection
- List available projects as options

Build options from available projects:

```bash
# List available projects (exclude dot-directories)
ls -1 .planning/projects/ | grep -v "^\."
```

Use AskUserQuestion:
- header: "Select Project"
- question: "Which project do you want to switch to?"
- options: [list of project names from above]

## Step 3: Validate Project Exists

```bash
PROJECT_NAME="$1"  # From argument or user selection

if [ ! -d ".planning/projects/$PROJECT_NAME" ]; then
  echo "Error: Project '$PROJECT_NAME' not found"
  echo ""
  echo "Available projects:"
  ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
  exit 1
fi
```

If project doesn't exist, show error with available projects.

## Step 4: Update Active Project

```bash
echo "$PROJECT_NAME" > .planning/.active
```

## Step 5: Confirm Switch

Show confirmation with project context:

```bash
# Get current phase from STATE.md if exists
PROJECT_DIR=".planning/projects/$PROJECT_NAME"
if [ -f "$PROJECT_DIR/STATE.md" ]; then
  CURRENT_PHASE=$(grep "^Phase:" "$PROJECT_DIR/STATE.md" | head -1 | sed 's/Phase: //')
else
  CURRENT_PHASE="Not started"
fi
```

Display:

```
Switched to project: {PROJECT_NAME}

Current phase: {CURRENT_PHASE}
Project path: .planning/projects/{PROJECT_NAME}/

All GSD commands now operate on this project.
```

</process>

<output>
Updates `.planning/.active` to contain the new project name.
</output>

<success_criteria>
- [ ] Project name validated against existing projects
- [ ] .active file updated with new project name
- [ ] User sees confirmation with current phase context
- [ ] Invalid project shows helpful error with available projects
</success_criteria>
