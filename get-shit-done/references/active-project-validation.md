<overview>
Pattern for validating active project state in GSD commands. When a command requires an active project but none is set, prompt user to select one rather than failing with cryptic errors.
</overview>

<when_to_use>

Use this validation at the START of any GSD command that:
- Reads from project-specific files (PROJECT.md, STATE.md, ROADMAP.md, phases/)
- Writes to project-specific locations
- Requires project context to function

Do NOT use for:
- Commands that work on shared files only (codebase/)
- Commands that explicitly handle no-project state (new-project, list-projects)
- Non-GSD operations

</when_to_use>

<validation_pattern>

## Full Validation Snippet

Add this at the start of command <process> section:

```bash
# Check if multi-project structure exists
if [ -d .planning/projects/ ]; then
  # Multi-project mode - need active project

  # Check for .active file
  if [ ! -f .planning/.active ]; then
    echo "No active project set."
    echo ""
    echo "Available projects:"
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    echo ""
    echo "Select a project with: /gsd:switch-project <name>"
    echo "Or run: /gsd:list-projects for details"
    exit 1
  fi

  # Read and validate active project
  ACTIVE_PROJECT=$(cat .planning/.active | tr -d '[:space:]')

  # Check for empty .active file
  if [ -z "$ACTIVE_PROJECT" ]; then
    echo "Active project file is empty."
    echo ""
    echo "Available projects:"
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    echo ""
    echo "Select a project with: /gsd:switch-project <name>"
    exit 1
  fi

  # Verify project directory exists
  if [ ! -d ".planning/projects/$ACTIVE_PROJECT" ]; then
    echo "Error: Active project '$ACTIVE_PROJECT' not found."
    echo ""
    echo "This may happen after archiving or deleting a project."
    echo ""
    echo "Available projects:"
    ls -1 .planning/projects/ | grep -v "^\." | sed 's/^/  - /'
    echo ""
    echo "Select a project with: /gsd:switch-project <name>"
    exit 1
  fi

  # Project validated - set PROJECT_BASE for use in command
  PROJECT_BASE=".planning/projects/$ACTIVE_PROJECT"

else
  # Flat (legacy) structure - use root
  PROJECT_BASE=".planning"
fi

# Now use PROJECT_BASE for all project-specific file access
# Example: $PROJECT_BASE/STATE.md, $PROJECT_BASE/phases/
```

</validation_pattern>

<error_states>

## NO_ACTIVE_PROJECT

**Trigger:** `.active` file missing in multi-project structure

**User sees:**
```
No active project set.

Available projects:
  - my-project
  - feature-auth

Select a project with: /gsd:switch-project <name>
Or run: /gsd:list-projects for details
```

## EMPTY_ACTIVE_FILE

**Trigger:** `.active` exists but contains only whitespace

**User sees:**
```
Active project file is empty.

Available projects:
  - my-project
  - feature-auth

Select a project with: /gsd:switch-project <name>
```

## PROJECT_NOT_FOUND

**Trigger:** `.active` references non-existent project (e.g., after archive)

**User sees:**
```
Error: Active project 'old-project' not found.

This may happen after archiving or deleting a project.

Available projects:
  - my-project
  - feature-auth

Select a project with: /gsd:switch-project <name>
```

</error_states>

<integration_checklist>

## Adding to Existing Commands

When updating commands for Phase 4 integration:

1. Add `@~/.claude/get-shit-done/references/active-project-validation.md` to execution_context
2. Copy validation snippet to START of <process> section
3. Replace hardcoded `.planning/` paths with `$PROJECT_BASE/`
4. Test in both flat and nested structures

**Commands requiring this validation:**
- All workflows that access PROJECT.md, STATE.md, ROADMAP.md
- All agents that write to phases/, research/, todos/
- Any command that needs project context

**Commands that skip this validation:**
- /gsd:new-project (handles its own structure detection)
- /gsd:list-projects (operates on projects/ directory directly)
- /gsd:switch-project (operates on projects/ directory directly)
- /gsd:archive-project (operates on projects/ directory directly)
- /gsd:map-codebase (codebase/ is shared, not project-specific)

</integration_checklist>

<related_references>

- **@path-resolution.md** - Full path resolution algorithm
- **@shared-paths.md** - Which paths are shared vs project-specific
- **/gsd:switch-project** - Command to set active project
- **/gsd:list-projects** - Command to see available projects

</related_references>

---

*Reference created: Phase 02 Plan 03*
*Valid for: All GSD commands requiring project context*
*Integration: Phase 4*
