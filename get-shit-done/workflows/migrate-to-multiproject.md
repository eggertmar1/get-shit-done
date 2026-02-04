<purpose>
Migrate existing flat `.planning/` structure to nested multi-project structure.

This workflow runs when user explicitly triggers migration via `/gsd:new-project` on a repo with flat `.planning/` structure. It preserves git history, gitignores local state files, and handles both project-specific and shared files correctly.

Output: Nested `.planning/projects/<name>/` structure with all files moved via git mv.
</purpose>

<philosophy>
**User-initiated migration:**
- Never auto-migrate. User must explicitly run `/gsd:new-project` and confirm.
- Show what will happen before doing anything.
- Ask for project name (don't assume "default").

**Git history preservation:**
- Use `git mv` for all file moves (preserves blame/log).
- Commit in logical groups with descriptive messages.
- Never use fs rename + delete (loses history).

**Best-effort error handling:**
- If migration fails partway, document what succeeded and what failed.
- Don't attempt rollback (too complex for rare failure case).
- User can recover via git reset if needed.

**Critical ordering:**
- Add `.active` to gitignore BEFORE creating the file.
- Create structure, move files, commit, THEN create .active file.
</philosophy>

<when_this_runs>
## When This Workflow Runs

User runs `/gsd:new-project` on repo with existing flat `.planning/` structure and confirms they want to migrate to multi-project.

**Preconditions:**
- `.planning/PROJECT.md` exists (flat structure)
- `.planning/projects/` does NOT exist (not yet migrated)
- User confirms migration (chose option A)
- User provides name for the migrated project
</when_this_runs>

<process>

<step name="pre_migration_checks">
## Pre-Migration Checks

Before moving any files, verify preconditions:

```bash
# Verify flat structure exists
test -f .planning/PROJECT.md && ! test -d .planning/projects
```

**Check git status:**

```bash
git status --porcelain .planning/
```

If working directory has uncommitted changes in `.planning/`:

```
⚠️ Uncommitted changes in .planning/

Git status shows:
[output]

To safely migrate, either:
1. Commit these changes first
2. Continue anyway (migration will preserve uncommitted files)

Proceed? (commit first / continue anyway / cancel)
```

Wait for user response. If "commit first", guide user to commit, then continue. If "cancel", exit workflow.

</step>

<step name="gitignore_active_file">
## Step 1: Gitignore .active FIRST

**CRITICAL:** Add .active to gitignore BEFORE creating any structure or files.

```bash
# Check if .planning/.gitignore exists
if [ ! -f .planning/.gitignore ]; then
  echo ".active" > .planning/.gitignore
else
  # Check if .active already in gitignore
  if ! grep -q "^\.active$" .planning/.gitignore; then
    echo ".active" >> .planning/.gitignore
  fi
fi
```

**Commit gitignore change:**

```bash
git add .planning/.gitignore
git commit -m "chore: gitignore .active file for multi-project support"
```

**Why this order matters:**
- If .active is created before gitignore, git stages it
- Staged files cause merge conflicts across team members
- Gitignore only affects untracked files, not already-tracked ones
- Must gitignore FIRST, then create file

</step>

<step name="create_structure">
## Step 2: Create Project Structure

Create the nested directory for the migrated project:

```bash
mkdir -p .planning/projects/{PROJECT_NAME}
```

Replace `{PROJECT_NAME}` with the name user provided.

**Verify:**
```bash
test -d .planning/projects/{PROJECT_NAME}
```

</step>

<step name="move_project_files">
## Step 3: Move Project-Specific Files

Use `git mv` for all moves to preserve git history (blame, log --follow).

**Project-specific files to move:**

```bash
# Core planning docs
git mv .planning/PROJECT.md .planning/projects/{PROJECT_NAME}/PROJECT.md
git mv .planning/REQUIREMENTS.md .planning/projects/{PROJECT_NAME}/REQUIREMENTS.md
git mv .planning/ROADMAP.md .planning/projects/{PROJECT_NAME}/ROADMAP.md
git mv .planning/STATE.md .planning/projects/{PROJECT_NAME}/STATE.md

# Directory trees
git mv .planning/phases .planning/projects/{PROJECT_NAME}/phases
git mv .planning/research .planning/projects/{PROJECT_NAME}/research
git mv .planning/todos .planning/projects/{PROJECT_NAME}/todos
```

**Error handling:**

Some files may not exist (e.g., `research/` or `todos/` might be missing). Handle each `git mv` individually:

```bash
# Example error-tolerant move
if [ -f .planning/PROJECT.md ]; then
  git mv .planning/PROJECT.md .planning/projects/{PROJECT_NAME}/PROJECT.md
else
  echo "⚠️ PROJECT.md not found (skipped)"
fi

if [ -d .planning/phases ]; then
  git mv .planning/phases .planning/projects/{PROJECT_NAME}/phases
else
  echo "⚠️ phases/ not found (skipped)"
fi
```

Track what succeeded and what failed. If any moves fail:

```
⚠️ Some files failed to move:
- phases/: [error message]

Successfully moved:
- PROJECT.md
- ROADMAP.md
- STATE.md

You can manually fix failed moves using:
git mv .planning/phases .planning/projects/{PROJECT_NAME}/phases
```

Don't attempt rollback. Let user recover manually or continue with partial migration.

</step>

<step name="handle_config">
## Step 4: Handle config.json

Config.json has dual purpose after migration:
- Original at `.planning/config.json` becomes **global defaults**
- Copy at `.planning/projects/{PROJECT_NAME}/config.json` becomes **per-project overrides**

```bash
# Copy (not move) config.json
if [ -f .planning/config.json ]; then
  cp .planning/config.json .planning/projects/{PROJECT_NAME}/config.json
  git add .planning/projects/{PROJECT_NAME}/config.json
fi
```

**Why copy, not move:**
- Root config.json becomes global defaults for all projects
- Per-project config can override specific settings
- Both files needed in nested structure

</step>

<step name="verify_shared_files">
## Step 5: Verify Shared Files Stay at Root

These files remain at `.planning/` root (shared across all projects):

- **codebase/** - Repo-level analysis, not project-specific
- **config.json** - Global defaults (copied to project, original stays)

**Verify they weren't moved:**

```bash
test -d .planning/codebase && echo "✓ codebase/ at root (shared)"
test -f .planning/config.json && echo "✓ config.json at root (global)"
```

If codebase/ or config.json are missing from root, this indicates an error earlier in the workflow.

</step>

<step name="commit_migration">
## Step 6: Commit Migration

Stage all moved files and commit the restructure:

```bash
# Stage the new structure
git add .planning/projects/{PROJECT_NAME}/

# Commit with descriptive message
git commit -m "$(cat <<'EOF'
refactor: migrate to multi-project structure

Migrated existing planning files to projects/{PROJECT_NAME}/

Project-specific files moved:
- PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md
- phases/, research/, todos/
- config.json (copied for per-project overrides)

Shared files remain at .planning/ root:
- codebase/ (repo-level analysis)
- config.json (global defaults)

Active project will be set to: {PROJECT_NAME}
EOF
)"
```

Replace `{PROJECT_NAME}` with actual project name.

**Verify commit:**

```bash
git log -1 --stat
```

Should show files moved from `.planning/` to `.planning/projects/{PROJECT_NAME}/`.

</step>

<step name="set_active_project">
## Step 7: Set Active Project

Create `.active` file (after gitignore is committed):

```bash
echo "{PROJECT_NAME}" > .planning/.active
```

**Verify:**

```bash
# File exists
test -f .planning/.active

# Contains project name
cat .planning/.active

# Is gitignored (won't be committed)
git check-ignore .planning/.active
```

All three checks should pass. If `git check-ignore` fails, the gitignore step didn't work correctly.

</step>

<step name="post_migration_verification">
## Post-Migration Verification

Verify the migration completed successfully:

```bash
# Structure exists
test -d .planning/projects/{PROJECT_NAME}
echo "✓ Project directory exists"

# Key files migrated
test -f .planning/projects/{PROJECT_NAME}/PROJECT.md
echo "✓ PROJECT.md migrated"

# Active file exists
test -f .planning/.active
echo "✓ Active project set"

# Shared files still at root
test -d .planning/codebase
echo "✓ Codebase at root (shared)"

# Active file is gitignored
git check-ignore .planning/.active
echo "✓ Active file gitignored"
```

**If any verification fails:**

Report which check failed and provide recovery guidance:

```
✗ Migration verification failed: PROJECT.md not found

This means the file wasn't moved. Check:
- Was PROJECT.md in .planning/ before migration?
- Did git mv command succeed?

To fix manually:
git mv .planning/PROJECT.md .planning/projects/{PROJECT_NAME}/PROJECT.md
```

</step>

</process>

<error_scenarios>
## Error Handling Scenarios

### Scenario 1: git mv Fails (Permission Denied)

**Error:** `Permission denied` during file move

**Response:**
```
✗ Failed to move [file]: Permission denied

Check file permissions:
ls -l .planning/[file]

To fix:
1. Ensure you have write permissions
2. Close any programs that might have the file open
3. Retry manually: git mv .planning/[file] .planning/projects/{NAME}/[file]
```

### Scenario 2: Disk Full During Migration

**Error:** `No space left on device`

**Response:**
```
✗ Migration failed: No space left on device

Partial migration state:
- Moved: [list of successfully moved files]
- Not moved: [list of files still at root]

To recover:
1. Free up disk space
2. Complete migration manually using remaining git mv commands
3. OR reset: git reset --hard HEAD~1 (undoes migration)
```

### Scenario 3: Project Name Directory Already Exists

**Error:** `mkdir: .planning/projects/{NAME}: File exists`

**Response:**
```
✗ Directory .planning/projects/{NAME} already exists

This might mean:
1. Previous migration attempt failed partway
2. Project name conflicts with existing directory

Options:
1. Choose different project name
2. Delete existing directory (CAREFUL - check what's in it first)
3. Investigate: ls -la .planning/projects/{NAME}
```

### Scenario 4: .active File Already Tracked in Git

**Error:** `git check-ignore .planning/.active` returns non-zero (file is tracked)

**Response:**
```
⚠️ .active file is tracked in git

This will cause merge conflicts. To fix:

1. Remove from git tracking:
   git rm --cached .planning/.active

2. Verify it's in .gitignore:
   grep "^\.active$" .planning/.gitignore

3. Commit the removal:
   git commit -m "chore: untrack .active file"

Then proceed with migration.
```

</error_scenarios>

<migration_success>
## Migration Success

When all steps complete successfully:

```
✓ Migration complete!

Structure:
  .planning/
  ├── .active                    ← "{PROJECT_NAME}" (gitignored)
  ├── .gitignore                 ← includes .active
  ├── config.json                ← global defaults
  ├── codebase/                  ← shared across projects
  └── projects/
      └── {PROJECT_NAME}/
          ├── PROJECT.md         ← migrated
          ├── REQUIREMENTS.md    ← migrated
          ├── ROADMAP.md         ← migrated
          ├── STATE.md           ← migrated
          ├── config.json        ← per-project overrides
          ├── phases/            ← migrated
          ├── research/          ← migrated
          └── todos/             ← migrated

Active project: {PROJECT_NAME}

All GSD commands will now operate on projects/{PROJECT_NAME}/
To work on a different project, use: /gsd:switch-project <name>
```

</migration_success>

<success_criteria>
- [ ] .active added to .gitignore and committed
- [ ] projects/{NAME}/ directory created
- [ ] All project-specific files moved via git mv
- [ ] config.json copied (not moved) to project directory
- [ ] Shared files (codebase/, config.json) remain at root
- [ ] Migration committed with descriptive message
- [ ] .active file created with project name
- [ ] Post-migration verification passes
- [ ] User informed of success
</success_criteria>
