# Phase 1: Foundation & Migration - Research

**Researched:** 2026-02-04
**Domain:** File system migration, path resolution, git-safe state tracking
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Error Handling:**
- If `.active` references a project that doesn't exist → prompt user to select from available projects
- If `.planning/projects/` directory is missing entirely → guided recovery (explain situation, offer to initialize structure)
- When migration fails partway through → best effort (keep what worked, report what failed, let user fix manually)

**Path Resolution:**
- When no `.active` file exists but `projects/` has content → prompt user to select which project to activate
- Use existing GSD validation patterns for project directory structure (no new validation layer)

**Migration Behavior:**
- Migration triggers only when user runs `/gsd:new-project` (not on any GSD command)
- Always confirm before migration — show what will happen, ask before moving files
- Ask user what to name the migrated project (don't assume "default")
- Auto-commit the restructure with descriptive message after migration

### Claude's Discretion
- Error message verbosity — pick appropriate level per situation
- Path resolver architecture — reference doc vs inline in workflows
- How to distinguish shared paths (codebase/) from project-specific paths

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

<research_summary>
## Summary

Researched file system migration patterns, path resolution strategies, and git-safe state tracking for implementing multi-project support in GSD. The standard approach uses Node.js native filesystem APIs (fs.promises) with an expand-migrate-contract pattern for backwards compatibility. Key finding: migration must preserve git history using `git mv` commands, and local state files (like `.active`) must be gitignored to prevent merge conflicts.

The critical pattern is **detection-before-action**: always check current structure before any file operation, route through a path resolver function that handles both old (flat) and new (nested) structures during transition, then migrate only when explicitly triggered by user action.

**Primary recommendation:** Build a synchronous path resolver that detects structure type on each call (zero state, always fresh), use `git mv` for migration to preserve history, and add `.active` to `.gitignore` immediately during migration.
</research_summary>

<standard_stack>
## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| fs (Node.js) | Built-in | File system operations | Native, zero dependencies, promises API stable since Node 14 |
| path (Node.js) | Built-in | Path resolution | Cross-platform path handling, standard for all Node.js projects |
| git CLI | System | File moves with history | Preserves git history better than manual file copy+delete |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| process.cwd() | Built-in | Working directory | Resolve relative paths in commands |
| fs.promises.access() | Built-in | File existence checks | Non-throwing existence validation |
| fs.promises.readdir() | Built-in | Directory listing | List projects, detect structure |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| git CLI | fs-extra library | fs-extra doesn't preserve git history for moves |
| Synchronous checks | Cache structure type | Cache adds state management complexity, stale data risk |
| Single resolver function | Class-based resolver | Class adds OOP overhead for stateless operations |

**Installation:**
```bash
# No installation needed - all built-in Node.js APIs
# Git must be available on system PATH (already required by GSD)
```
</standard_stack>

<architecture_patterns>
## Architecture Patterns

### Recommended Project Structure
```
.planning/
├── .active                    # gitignored, tracks current project
├── config.json                # global config
├── codebase/                  # shared across all projects
│   ├── STACK.md
│   ├── ARCHITECTURE.md
│   └── ...
└── projects/                  # new nested structure
    └── <project-name>/
        ├── PROJECT.md
        ├── ROADMAP.md
        ├── STATE.md
        ├── config.json        # per-project overrides
        ├── research/
        └── phases/
```

### Pattern 1: Detection-Based Path Resolution
**What:** Check structure on every call, no cached state
**When to use:** All file operations in commands/agents
**Example:**
```javascript
// Stateless resolver - detects structure every call
function resolvePlanningPath(relativePath) {
  const planningRoot = path.join(process.cwd(), '.planning');

  // Check if nested structure exists
  const projectsDir = path.join(planningRoot, 'projects');
  if (fs.existsSync(projectsDir)) {
    // New structure - read active project
    const activeFile = path.join(planningRoot, '.active');
    if (fs.existsSync(activeFile)) {
      const activeProject = fs.readFileSync(activeFile, 'utf-8').trim();
      return path.join(projectsDir, activeProject, relativePath);
    } else {
      // No active project - caller must handle (prompt user)
      throw new Error('NO_ACTIVE_PROJECT');
    }
  } else {
    // Old flat structure - direct path
    return path.join(planningRoot, relativePath);
  }
}

// Usage in commands
const projectPath = resolvePlanningPath('PROJECT.md');
const projectContent = fs.readFileSync(projectPath, 'utf-8');
```

### Pattern 2: Expand-Migrate-Contract for Backwards Compatibility
**What:** Three-phase migration preserving backwards compatibility
**When to use:** One-time structural migrations
**Example:**
```bash
# EXPAND: Create new structure alongside old
mkdir -p .planning/projects/my-project

# MIGRATE: Move files with git (preserves history)
git mv .planning/PROJECT.md .planning/projects/my-project/PROJECT.md
git mv .planning/ROADMAP.md .planning/projects/my-project/ROADMAP.md
git mv .planning/phases .planning/projects/my-project/phases

# Leave shared files at root
# .planning/codebase/ stays at root (shared)
# .planning/config.json stays at root (becomes global)

# CONTRACT: Clean up (if needed)
# In this case, old paths are now occupied by new structure
```

### Pattern 3: Git-Safe Local State Tracking
**What:** Use gitignored files for machine-local state
**When to use:** State that varies per user/machine (active project)
**Example:**
```bash
# Add .active to .gitignore FIRST (before creating file)
echo ".active" >> .planning/.gitignore

# Write active project
echo "my-project" > .planning/.active

# Commit the gitignore update
git add .planning/.gitignore
git commit -m "chore: gitignore .active file"

# .active file now exists but won't be tracked
```

### Anti-Patterns to Avoid
- **Caching structure type in memory:** Structure can change between command calls, always detect fresh
- **Moving files without git mv:** Loses git history, blame/log becomes useless
- **Creating .active without gitignoring first:** Causes merge conflicts when team members have different active projects
- **Migrating on any command:** User loses control, surprise file moves feel like bugs
</architecture_patterns>

<dont_hand_roll>
## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| File moving with history | fs.rename() then delete old | git mv via child_process | Git tracks content similarity across moves, enables git log --follow |
| Path resolution | String concatenation | path.join() | Handles Windows/Unix differences, trailing slashes, .. navigation |
| File existence | try/catch fs.readFile | fs.promises.access() or fs.existsSync | access() doesn't throw on ENOENT, clearer intent |
| Directory listing | Recursive fs.readdir implementation | fs.promises.readdir with withFileTypes | Built-in, handles symlinks correctly |
| Project selection UI | Custom readline prompts | Existing AskUserQuestion tool | GSD already has this, consistent UX |

**Key insight:** File system operations have decades of edge cases (permissions, symlinks, race conditions, platform differences). Node.js built-ins and git CLI handle these correctly. Custom implementations miss edge cases that surface as "works on my machine" bugs.
</dont_hand_roll>

<common_pitfalls>
## Common Pitfalls

### Pitfall 1: Migration Without User Consent
**What goes wrong:** Files suddenly move, user confused about structure change
**Why it happens:** Auto-migration on any command seems convenient but violates user expectations
**How to avoid:** Only migrate when user explicitly runs `/gsd:new-project`, show preview of what will move, require confirmation
**Warning signs:** User reports "files moved themselves", git history shows unexpected restructure commits

### Pitfall 2: Gitignore Added AFTER File Creation
**What goes wrong:** .active file gets tracked in git, merge conflicts on every branch switch
**Why it happens:** Creating file first seems logical, but git stages it before gitignore takes effect
**How to avoid:** Add to .gitignore first, commit gitignore change, THEN create .active file
**Warning signs:** Git status shows .active as tracked, team members see merge conflicts on .active

### Pitfall 3: Synchronous Blocking Operations in Production
**What goes wrong:** Commands freeze during large directory operations
**Why it happens:** Using fs.readdirSync, fs.readFileSync in main flow blocks event loop
**How to avoid:** Use fs.promises APIs with await for all file operations except one-time setup checks
**Warning signs:** CLI becomes unresponsive during file operations, can't cancel with Ctrl+C

### Pitfall 4: Forgetting Shared vs Project-Specific Paths
**What goes wrong:** Codebase maps end up per-project (wrong) or PROJECT.md shared across projects (wrong)
**Why it happens:** Path resolver doesn't distinguish between shared and project paths
**How to avoid:** Explicit handling: codebase/ always resolves to root, everything else resolves to project
**Warning signs:** Codebase map shows different content per project, or all projects share same STATE.md

### Pitfall 5: Partial Migration Leaves Inconsistent State
**What goes wrong:** Some files migrated, others left behind, neither structure works
**Why it happens:** Migration script fails midway (permission error, disk full), no rollback
**How to avoid:** Best-effort approach - document what succeeded, what failed, let user recover manually. Don't attempt atomic rollback (adds complexity)
**Warning signs:** User reports "some files in old location, some in new", git status shows half-completed moves
</common_pitfalls>

<code_examples>
## Code Examples

### Basic Path Resolver (Detection-Based)
```javascript
// Source: Node.js fs documentation + backwards compatibility patterns
const fs = require('fs');
const path = require('path');

function resolvePlanningPath(relativePath) {
  const planningRoot = path.join(process.cwd(), '.planning');

  // Shared paths always at root
  const sharedPaths = ['codebase', 'config.json'];
  if (sharedPaths.some(p => relativePath.startsWith(p))) {
    return path.join(planningRoot, relativePath);
  }

  // Check for nested structure
  const projectsDir = path.join(planningRoot, 'projects');
  if (fs.existsSync(projectsDir)) {
    // New structure - need active project
    const activeFile = path.join(planningRoot, '.active');
    if (!fs.existsSync(activeFile)) {
      throw new Error('NO_ACTIVE_PROJECT');
    }
    const activeProject = fs.readFileSync(activeFile, 'utf-8').trim();
    if (!activeProject) {
      throw new Error('EMPTY_ACTIVE_FILE');
    }
    return path.join(projectsDir, activeProject, relativePath);
  } else {
    // Old flat structure
    return path.join(planningRoot, relativePath);
  }
}

// Usage
try {
  const projectPath = resolvePlanningPath('PROJECT.md');
  const content = fs.readFileSync(projectPath, 'utf-8');
} catch (err) {
  if (err.message === 'NO_ACTIVE_PROJECT') {
    // Prompt user to select project
  } else {
    throw err;
  }
}
```

### Git-Preserving Migration
```bash
# Source: Git best practices for file moves
# Run these commands in sequence, committing after each logical group

# Step 1: Create target structure
mkdir -p .planning/projects/my-project

# Step 2: Move project-specific files with git mv (preserves history)
git mv .planning/PROJECT.md .planning/projects/my-project/PROJECT.md
git mv .planning/ROADMAP.md .planning/projects/my-project/ROADMAP.md
git mv .planning/STATE.md .planning/projects/my-project/STATE.md
git mv .planning/phases .planning/projects/my-project/phases
git mv .planning/research .planning/projects/my-project/research

# Step 3: Move per-project config (create new, copy content)
cp .planning/config.json .planning/projects/my-project/config.json
# Original config.json becomes global config (stays at root)

# Step 4: Commit the restructure
git commit -m "refactor: migrate to multi-project structure

Moved project files to projects/my-project/:
- PROJECT.md, ROADMAP.md, STATE.md
- phases/, research/
- config.json (per-project)

Shared files remain at root:
- codebase/ (shared across projects)
- config.json (global defaults)
"

# Step 5: Set up gitignored state tracking
echo ".active" >> .planning/.gitignore
git add .planning/.gitignore
git commit -m "chore: gitignore .active file"

# Step 6: Create active project marker (NOT committed)
echo "my-project" > .planning/.active
```

### Listing Available Projects
```javascript
// Source: Node.js fs.promises documentation
const fs = require('fs').promises;
const path = require('path');

async function listProjects() {
  const projectsDir = path.join(process.cwd(), '.planning', 'projects');

  try {
    await fs.access(projectsDir);
  } catch {
    // No projects directory - old structure
    return [];
  }

  const entries = await fs.readdir(projectsDir, { withFileTypes: true });
  return entries
    .filter(entry => entry.isDirectory())
    .map(entry => entry.name);
}

// Usage
const projects = await listProjects();
if (projects.length === 0) {
  console.log('No multi-project structure detected');
} else {
  console.log('Available projects:', projects.join(', '));
}
```

### Active Project Management
```javascript
// Source: Git-safe local state patterns
const fs = require('fs').promises;
const path = require('path');

async function getActiveProject() {
  const activeFile = path.join(process.cwd(), '.planning', '.active');

  try {
    const content = await fs.readFile(activeFile, 'utf-8');
    return content.trim();
  } catch (err) {
    if (err.code === 'ENOENT') {
      return null; // No active project
    }
    throw err;
  }
}

async function setActiveProject(projectName) {
  const activeFile = path.join(process.cwd(), '.planning', '.active');

  // Verify project exists
  const projectDir = path.join(process.cwd(), '.planning', 'projects', projectName);
  try {
    await fs.access(projectDir);
  } catch {
    throw new Error(`Project not found: ${projectName}`);
  }

  // Write active project
  await fs.writeFile(activeFile, projectName + '\n', 'utf-8');
}
```
</code_examples>

<sota_updates>
## State of the Art (2025-2026)

What's changed recently:

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| fs callbacks | fs.promises with async/await | Node 14+ (2020) | Cleaner error handling, no callback hell |
| Manual recursive readdir | fs.promises.readdir with recursive option | Node 18.17 (2023) | Simpler directory traversal |
| External glob libraries | fs.promises.glob() | Node 22 (2024) | Pattern matching built-in, zero dependencies |
| Synchronous operations in CLI | Async operations with proper await | 2022+ | Non-blocking, better UX for long operations |

**New tools/patterns to consider:**
- **fs.promises.glob()**: Node 22 added native glob support, eliminates need for fast-glob/globby for pattern matching
- **withFileTypes option**: fs.readdir now returns Dirent objects, more efficient than stat-ing each entry
- **fsPromises.cp()**: Node 16.7+ has native recursive directory copy, faster than shell cp

**Deprecated/outdated:**
- **fs-extra library**: Most features now in Node.js core (copy, move, mkdirp), only needed for legacy Node versions
- **Synchronous fs operations**: Still available but discouraged in production code, blocks event loop
- **Manual gitignore parsing**: Use git check-ignore instead of parsing .gitignore manually
</sota_updates>

<open_questions>
## Open Questions

Things that couldn't be fully resolved:

1. **Handling concurrent project switching**
   - What we know: .active file is local-only, fast to read/write
   - What's unclear: What if two terminal windows try to switch projects simultaneously?
   - Recommendation: Document that switching projects affects all GSD commands globally (like git checkout affects all terminals). Not a problem in practice since users typically work in one project at a time.

2. **Migration rollback strategy**
   - What we know: Best-effort approach is industry standard for file migrations
   - What's unclear: Should we attempt atomic rollback on failure?
   - Recommendation: No atomic rollback - adds significant complexity for rare failure case. Instead: clear error messages showing what succeeded, what failed, let user use git to recover.

3. **Path resolver performance with many projects**
   - What we know: Detecting structure on every call adds fs.existsSync() overhead
   - What's unclear: Does this become slow with 10+ projects and frequent file operations?
   - Recommendation: Start with stateless detection. If performance becomes issue (unlikely - existsSync is <1ms), add simple cache invalidated after any write operation.
</open_questions>

<sources>
## Sources

### Primary (HIGH confidence)
- [Node.js File System Documentation](https://nodejs.org/api/fs.html) - fs.promises, path.join, access, readdir APIs
- [Node.js Working with Folders](https://nodejs.org/en/learn/manipulating-files/working-with-folders-in-nodejs) - Directory operations best practices
- [Git Move Files: Practical Renames, Refactors, and History Preservation in 2026](https://thelinuxcode.com/git-move-files-practical-renames-refactors-and-history-preservation-in-2026/) - git mv preserves history
- [Git - preserve history when moving files](https://linuxctl.com/p/git-preserve-history-when-moving-files/) - Pure move commits for best history tracking

### Secondary (MEDIUM confidence)
- [Backward compatible database changes](https://planetscale.com/blog/backward-compatible-databases-changes) - Expand-migrate-contract pattern (verified applicable to file migrations)
- [Database Design Patterns for Ensuring Backward Compatibility](https://www.pingcap.com/article/database-design-patterns-for-ensuring-backward-compatibility/) - Dual-write pattern adaptation
- [How to Use .gitignore to Ignore Files and Folders in Git](https://www.codecademy.com/article/how-to-use-gitignore) - Gitignore patterns for preventing conflicts
- [Better local require() paths for Node.js](https://gist.github.com/branneman/8048520) - Path resolution patterns

### Tertiary (LOW confidence - needs validation)
- None - all findings verified with official documentation
</sources>

<metadata>
## Metadata

**Research scope:**
- Core technology: Node.js fs/path built-ins, git CLI
- Ecosystem: Native APIs only, zero dependencies
- Patterns: Detection-based resolution, expand-migrate-contract, git-safe state
- Pitfalls: Migration consent, gitignore timing, blocking operations, path categorization

**Confidence breakdown:**
- Standard stack: HIGH - Node.js built-ins, stable since Node 14+
- Architecture: HIGH - Patterns from official docs and migration best practices
- Pitfalls: HIGH - Documented in git/fs docs and backwards compatibility literature
- Code examples: HIGH - From Node.js official documentation

**Research date:** 2026-02-04
**Valid until:** 2026-03-04 (30 days - Node.js APIs stable, patterns mature)
</metadata>

---

*Phase: 01-foundation-migration*
*Research completed: 2026-02-04*
*Ready for planning: yes*
