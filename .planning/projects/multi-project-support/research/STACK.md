# Stack Research

**Domain:** Multi-Project CLI Tool Support
**Researched:** 2026-02-04
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| File-based context tracking | N/A | Active project marker | Industry standard (kubeconfig, direnv, asdf). Simple text files for git-friendly tracking. Zero dependencies. |
| Directory-per-project isolation | N/A | Project workspace separation | Universal pattern (npm workspaces, Terraform, monorepos). Clear boundaries, independent state management. |
| Hierarchical configuration | N/A | Global + per-project config | Standard pattern (direnv source_up, git configs). Allows defaults with overrides. |
| Plain text markers | N/A | Active context persistence | Used by kubectl (~/.kube/config), asdf (.tool-versions). Git-trackable, human-readable. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Node.js fs module | Built-in | File operations | All file read/write operations. Already available, zero dependencies. |
| Node.js path module | Built-in | Path resolution | Cross-platform path handling for project directories. |
| Git integration | CLI | Project name detection | Reading current branch for default project naming. Already integrated in GSD. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| None required | Pure Node.js | GSD's zero-dependency philosophy maintained. |

## Installation

```bash
# No new dependencies required
# Pattern uses only Node.js built-ins
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| `.active` file with project name | Environment variables (KUBECONFIG style) | If projects span multiple repos or need system-wide context. GSD is repo-scoped, so file-based is better. |
| `projects/<name>/` subdirectories | Flat structure with prefixes | If number of projects is very small (<3). Multi-project use cases need clear isolation. |
| Plain text `.active` file | JSON config with metadata | If you need to track additional context (timestamps, user, etc.). Keep it simple for now. |
| Branch name as default | Hardcoded "default" only | If teams don't use feature branches. Most modern workflows do. |
| Shared codebase map at root | Per-project codebase maps | If projects have different file scopes. Repo structure is shared, so centralize. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| SQLite or database | Adds dependency, overkill for simple context | Plain text files |
| Complex symlink patterns | Fragile across platforms, git issues | Direct path resolution |
| Workspace isolation via separate repos | Breaks "single `.planning/` folder" goal | Project subdirectories |
| Binary formats (.DS_Store style) | Not git-friendly, not human-readable | Plain text marker files |
| Global state in ~/.config | Makes `.planning/` non-portable | Keep state in `.planning/` |

## Stack Patterns by Variant

### Pattern 1: Kubernetes Config Model (RECOMMENDED)

**Structure:**
```
.planning/
├── .active              # Plain text: "auth-refactor"
├── config.json          # Global defaults
├── codebase/            # Shared repo map
└── projects/
    ├── default/
    │   ├── config.json  # Per-project overrides
    │   ├── PROJECT.md
    │   └── ...
    └── auth-refactor/
        └── ...
```

**Used by:**
- kubectl (~/.kube/config with current-context)
- Terraform CLI (workspaces with state tracking)

**Why it works:**
- Single source of truth for active context
- Git-trackable for team coordination
- Simple text file, no parsing complexity
- Natural to read/write with fs.readFileSync/writeFileSync

**Implementation:**
```javascript
// Read active project
const active = fs.readFileSync('.planning/.active', 'utf-8').trim();

// Switch project
fs.writeFileSync('.planning/.active', projectName);

// Get project path
const projectPath = path.join('.planning/projects', active);
```

### Pattern 2: Directory Auto-Detection (SECONDARY)

**Structure:** Same as Pattern 1, but with direnv-style auto-switching

**Used by:**
- direnv (.envrc with auto-load on cd)
- asdf (.tool-versions with per-directory versions)

**Why NOT recommended for GSD:**
- Requires shell integration or file watching
- Adds complexity to zero-dependency model
- GSD agents work across project boundaries
- Explicit switching is more predictable

**When to consider:**
- If adding shell completion later
- For local development convenience
- As opt-in feature, not default

### Pattern 3: Monorepo Workspace Pattern (PARTIAL ADOPTION)

**Structure:** npm/yarn/pnpm workspaces approach

**Used by:**
- npm workspaces (package.json with "workspaces" array)
- Nx (nx.json with project configurations)
- Turborepo (turbo.json with pipeline definitions)

**Relevant elements for GSD:**
- ✓ Projects as subdirectories
- ✓ Shared configuration at root
- ✓ Per-project configuration overrides
- ✗ Dependency hoisting (not applicable)
- ✗ Task orchestration (GSD already does this)
- ✗ Build caching (not applicable to planning)

**What to adopt:**
```javascript
// Config merging pattern from monorepo tools
const globalConfig = require('.planning/config.json');
const projectConfig = require(`.planning/projects/${active}/config.json`);
const mergedConfig = { ...globalConfig, ...projectConfig };
```

## Detailed Recommendations for GSD

### 1. Active Project Tracking

**Format:** Plain text file `.planning/.active`
```
auth-refactor
```

**Why:**
- Simplest possible format
- Git tracks changes (see who switched when)
- Human-readable for debugging
- Shell-scriptable (cat .planning/.active)
- Cross-platform compatible

**Precedent:**
- kubectl uses current-context in kubeconfig
- Git uses HEAD file to track active branch
- asdf uses .tool-versions for active versions

### 2. Project Directory Structure

**Format:** `projects/<name>/` subdirectories

```
.planning/
└── projects/
    ├── default/         # Auto-migrated existing projects
    ├── feature-auth/    # Named from git branch
    └── spike-perf/      # Manual override
```

**Why:**
- Clear isolation (each project is self-contained)
- Prevents file conflicts between projects
- Natural mapping for git branch workflows
- Easy to archive (delete directory)
- Supports parallel work by multiple team members

**Precedent:**
- Git worktrees: `project-main/`, `project-feature/`
- Terraform: workspace directories
- npm workspaces: packages/\*/

### 3. Configuration Hierarchy

**Format:** JSON config at two levels

```javascript
// .planning/config.json (global)
{
  "ai_runtime": "claude-code",
  "commit_docs": true,
  "research_depth": "medium"
}

// .planning/projects/auth-refactor/config.json (override)
{
  "research_depth": "deep"  // Only overrides this key
}
```

**Why:**
- Most settings are repo-wide (runtime, commit behavior)
- Some projects need deeper research, custom templates
- Follows git config pattern (system/global/local)
- Easy to understand precedence

**Precedent:**
- Git: /etc/gitconfig → ~/.gitconfig → .git/config
- npm: npmrc at multiple levels
- direnv: source_up for parent inheritance

### 4. Shared vs. Per-Project Artifacts

**Shared at root:**
```
.planning/
├── codebase/           # Repo structure doesn't change per project
├── config.json         # Global defaults
└── .active             # Active project marker
```

**Per-project:**
```
.planning/projects/auth-refactor/
├── PROJECT.md          # Project-specific goal
├── ROADMAP.md          # Different milestones
├── STATE.md            # Independent progress
├── research/           # Project-specific research
├── phases/             # Project-specific execution
└── config.json         # Optional overrides
```

**Why:**
- Codebase map is expensive to generate, repo-scoped
- Each project has different goals/milestones
- Research may focus on different domains
- Phase execution is independent

**Precedent:**
- Monorepos: shared node_modules, per-package sources
- Terraform: shared provider config, per-workspace state
- Git worktrees: shared .git, per-worktree branches

### 5. Project Naming Strategy

**Default:** Use git branch name
```bash
git rev-parse --abbrev-ref HEAD
# → "feature/auth-refactor"
# Transform to: "auth-refactor"
```

**Override:** Allow explicit naming
```bash
/gsd:new-project spike-performance
# Creates: .planning/projects/spike-performance/
```

**Why:**
- Natural mapping (branch = unit of work)
- Auto-naming reduces friction
- Override supports non-branch workflows
- Normalized names avoid path issues

**Precedent:**
- Git worktrees: default to branch name
- Heroku: app name from git remote
- Docker: container name from image

### 6. Migration Strategy

**Auto-detect flat structure:**
```javascript
// Check if old structure exists
if (fs.existsSync('.planning/PROJECT.md') &&
    !fs.existsSync('.planning/projects')) {
  // Migrate to projects/default/
  migrateToMultiProject();
}
```

**Migration steps:**
1. Create `projects/default/`
2. Move all project files (PROJECT.md, ROADMAP.md, etc.)
3. Keep codebase/ at root
4. Create `.active` with "default"
5. Leave config.json at root (becomes global)

**Why:**
- Backwards compatible (old repos work automatically)
- No breaking changes to existing workflows
- Clear upgrade path
- Git history preserved

**Precedent:**
- npm v7 workspace migration (auto-detects lerna.json)
- Git v2 hash algorithm migration (transparent)
- Python 2→3 (2to3 auto-converter)

## Version Compatibility

| Component | Compatible With | Notes |
|-----------|-----------------|-------|
| Node.js fs.readFileSync | All Node.js versions | Built-in, no version concerns |
| `.planning/` structure | Git 2.0+ | Standard directory tracking |
| Plain text `.active` | All platforms | UTF-8 encoding, LF line endings |
| JSON config files | Node.js built-in JSON.parse | Standard JSON, no special features |

## File Format Specifications

### .active File Format

```
auth-refactor
```

**Rules:**
- Single line containing project name
- No trailing newline (trim on read)
- Valid project name (alphanumeric, hyphens, underscores)
- Must match directory name in `projects/`

**Error handling:**
```javascript
// Validate on read
const active = fs.readFileSync('.planning/.active', 'utf-8').trim();
const projectPath = path.join('.planning/projects', active);

if (!fs.existsSync(projectPath)) {
  throw new Error(`Active project "${active}" not found in projects/`);
}
```

### Project Directory Naming Rules

**Valid:**
- `default` (migration target)
- `feature-auth` (kebab-case)
- `spike_performance` (snake_case)
- `v2-rewrite` (with numbers)

**Invalid:**
- `feature/auth` (no slashes - path confusion)
- `.hidden` (no leading dots - hidden file)
- `with spaces` (no spaces - shell issues)
- `../escape` (no path traversal)

**Normalization:**
```javascript
function normalizeProjectName(name) {
  return name
    .toLowerCase()
    .replace(/[^a-z0-9-_]/g, '-')  // Replace invalid chars
    .replace(/^-+|-+$/g, '');       // Trim leading/trailing dashes
}

// "feature/auth-refactor" → "feature-auth-refactor"
// "My Project!" → "my-project"
```

### Configuration Merge Strategy

```javascript
function getProjectConfig(projectName) {
  const globalPath = '.planning/config.json';
  const projectPath = `.planning/projects/${projectName}/config.json`;

  const global = fs.existsSync(globalPath)
    ? JSON.parse(fs.readFileSync(globalPath, 'utf-8'))
    : {};

  const project = fs.existsSync(projectPath)
    ? JSON.parse(fs.readFileSync(projectPath, 'utf-8'))
    : {};

  // Shallow merge (project overrides global)
  return { ...global, ...project };
}
```

**Deep merge NOT recommended:**
- Adds complexity
- Harder to reason about precedence
- Most config values are scalar (strings, booleans)
- If needed later, use lodash.merge pattern

## Command Patterns

### Project Switching

**Pattern:** Single command updates `.active` file

```bash
/gsd:switch-project auth-refactor
```

**Implementation:**
```javascript
function switchProject(projectName) {
  const projectPath = path.join('.planning/projects', projectName);

  if (!fs.existsSync(projectPath)) {
    const available = fs.readdirSync('.planning/projects');
    throw new Error(
      `Project "${projectName}" not found. Available: ${available.join(', ')}`
    );
  }

  fs.writeFileSync('.planning/.active', projectName);
  console.log(`Switched to project: ${projectName}`);
}
```

**Precedent:**
- kubectl config use-context <name>
- git worktree add <path> <branch>
- terraform workspace select <name>

### Project Listing

**Pattern:** Read projects/ directory, annotate with metadata

```bash
/gsd:list-projects
```

**Output:**
```
  default (15 files, last: 2025-01-15)
* auth-refactor (8 files, last: 2026-02-04)  ← active
  spike-perf (2 files, last: 2026-01-20)
```

**Implementation:**
```javascript
function listProjects() {
  const active = fs.readFileSync('.planning/.active', 'utf-8').trim();
  const projects = fs.readdirSync('.planning/projects');

  return projects.map(name => {
    const projectPath = path.join('.planning/projects', name);
    const stats = fs.statSync(projectPath);
    const fileCount = countMarkdownFiles(projectPath);

    return {
      name,
      active: name === active,
      fileCount,
      lastModified: stats.mtime
    };
  });
}
```

**Precedent:**
- git worktree list
- terraform workspace list
- kubectl config get-contexts

### Project Creation

**Pattern:** Create directory, copy templates, optionally set active

```bash
/gsd:new-project spike-performance
```

**Implementation:**
```javascript
function createProject(name, options = {}) {
  const normalized = normalizeProjectName(name || getCurrentBranch());
  const projectPath = path.join('.planning/projects', normalized);

  if (fs.existsSync(projectPath)) {
    throw new Error(`Project "${normalized}" already exists`);
  }

  // Create directory structure
  fs.mkdirSync(projectPath, { recursive: true });
  fs.mkdirSync(path.join(projectPath, 'research'));
  fs.mkdirSync(path.join(projectPath, 'phases'));

  // Copy templates if needed
  if (options.fromTemplate) {
    copyTemplates(projectPath);
  }

  // Switch to new project if requested
  if (options.switchTo) {
    fs.writeFileSync('.planning/.active', normalized);
  }

  console.log(`Created project: ${normalized}`);
}
```

**Precedent:**
- git worktree add
- terraform workspace new
- npm init (for new packages)

### Project Archival

**Pattern:** Delete directory (git history preserves if needed)

```bash
/gsd:archive-project spike-perf
```

**Implementation:**
```javascript
function archiveProject(projectName) {
  const active = fs.readFileSync('.planning/.active', 'utf-8').trim();

  if (projectName === active) {
    throw new Error('Cannot archive active project. Switch first.');
  }

  const projectPath = path.join('.planning/projects', projectName);

  if (!fs.existsSync(projectPath)) {
    throw new Error(`Project "${projectName}" not found`);
  }

  // Delete directory
  fs.rmSync(projectPath, { recursive: true, force: true });
  console.log(`Archived project: ${projectName}`);
}
```

**Why delete instead of move:**
- Git history preserves deleted files
- Simpler than archival directory
- Can restore with git revert
- Matches git branch -D behavior

**Precedent:**
- git worktree remove
- terraform workspace delete
- docker rm (containers)

## Error Handling Patterns

### No Active Project

**Scenario:** `.active` file missing or points to deleted project

**Solution:** Prompt user to select project

```javascript
function getActiveProject() {
  if (!fs.existsSync('.planning/.active')) {
    const projects = fs.readdirSync('.planning/projects');

    if (projects.length === 0) {
      throw new Error('No projects found. Create one with /gsd:new-project');
    }

    if (projects.length === 1) {
      // Auto-select only project
      fs.writeFileSync('.planning/.active', projects[0]);
      return projects[0];
    }

    // Prompt user
    throw new Error(
      `No active project. Choose one:\n${projects.map(p => `  /gsd:switch-project ${p}`).join('\n')}`
    );
  }

  const active = fs.readFileSync('.planning/.active', 'utf-8').trim();
  const projectPath = path.join('.planning/projects', active);

  if (!fs.existsSync(projectPath)) {
    throw new Error(
      `Active project "${active}" not found. Update with /gsd:switch-project`
    );
  }

  return active;
}
```

### Migration Detection

**Scenario:** Old flat structure with new multi-project command

**Solution:** Auto-migrate on first command

```javascript
function ensureMultiProjectStructure() {
  const hasOldStructure = fs.existsSync('.planning/PROJECT.md');
  const hasNewStructure = fs.existsSync('.planning/projects');

  if (hasOldStructure && !hasNewStructure) {
    console.log('Migrating to multi-project structure...');

    // Create projects/default/
    fs.mkdirSync('.planning/projects/default', { recursive: true });

    // Move project files
    const projectFiles = [
      'PROJECT.md', 'REQUIREMENTS.md', 'ROADMAP.md', 'STATE.md',
      'research/', 'phases/'
    ];

    projectFiles.forEach(file => {
      const oldPath = path.join('.planning', file);
      const newPath = path.join('.planning/projects/default', file);

      if (fs.existsSync(oldPath)) {
        fs.renameSync(oldPath, newPath);
      }
    });

    // Set active project
    fs.writeFileSync('.planning/.active', 'default');

    console.log('Migration complete. Active project: default');
  }
}
```

### Config Validation

**Scenario:** Invalid JSON in config files

**Solution:** Graceful fallback to defaults

```javascript
function loadConfig(configPath) {
  if (!fs.existsSync(configPath)) {
    return {};
  }

  try {
    const content = fs.readFileSync(configPath, 'utf-8');
    return JSON.parse(content);
  } catch (error) {
    console.warn(`Invalid config at ${configPath}: ${error.message}`);
    console.warn('Using defaults');
    return {};
  }
}
```

## Integration Points

### Git Branch Detection

```javascript
function getCurrentBranch() {
  try {
    // Read .git/HEAD
    const head = fs.readFileSync('.git/HEAD', 'utf-8').trim();

    // Parse ref: refs/heads/feature/auth-refactor
    if (head.startsWith('ref:')) {
      const branch = head.split('/').slice(2).join('/');
      return normalizeProjectName(branch);
    }

    // Detached HEAD - use commit hash prefix
    return head.substring(0, 7);
  } catch {
    return 'default';
  }
}
```

### Agent Command Resolution

**Current:** Hardcoded `.planning/` paths
```javascript
const projectPath = '.planning/PROJECT.md';
```

**New:** Dynamic project-aware paths
```javascript
function getProjectPath(filename) {
  const active = getActiveProject();
  return path.join('.planning/projects', active, filename);
}

const projectPath = getProjectPath('PROJECT.md');
```

**Commands affected:**
- All `/gsd:*` commands that read/write project files
- Template rendering (needs project context)
- Phase execution (reads from project phases/)

### Template System

**Current:** Templates reference `.planning/` directly

**New:** Templates use project-aware paths

```markdown
<!-- OLD -->
See `.planning/ROADMAP.md` for details

<!-- NEW -->
See `.planning/projects/${PROJECT_NAME}/ROADMAP.md` for details

<!-- BETTER: Use variable -->
See `${PROJECT_ROOT}/ROADMAP.md` for details
```

**Implementation:**
```javascript
function renderTemplate(templatePath, projectName) {
  const content = fs.readFileSync(templatePath, 'utf-8');
  const projectRoot = `.planning/projects/${projectName}`;

  return content
    .replace(/\$\{PROJECT_NAME\}/g, projectName)
    .replace(/\$\{PROJECT_ROOT\}/g, projectRoot);
}
```

## Testing Strategy

### Unit Tests (if adding later)

```javascript
// Test project switching
test('switchProject updates .active file', () => {
  switchProject('test-project');
  const active = fs.readFileSync('.planning/.active', 'utf-8').trim();
  expect(active).toBe('test-project');
});

// Test config merging
test('project config overrides global config', () => {
  const config = getProjectConfig('test-project');
  expect(config.research_depth).toBe('deep');  // overridden
  expect(config.ai_runtime).toBe('claude-code');  // inherited
});

// Test migration
test('auto-migrates flat structure', () => {
  // Setup old structure
  fs.writeFileSync('.planning/PROJECT.md', 'Test');

  // Run migration
  ensureMultiProjectStructure();

  // Verify new structure
  expect(fs.existsSync('.planning/projects/default/PROJECT.md')).toBe(true);
  expect(fs.readFileSync('.planning/.active', 'utf-8').trim()).toBe('default');
});
```

### Manual Testing Checklist

- [ ] Create new project with `/gsd:new-project test`
- [ ] List projects with `/gsd:list-projects`
- [ ] Switch between projects with `/gsd:switch-project`
- [ ] Run existing commands (new-project, map-codebase)
- [ ] Verify config overrides work
- [ ] Test migration from flat structure
- [ ] Archive old project with `/gsd:archive-project`
- [ ] Verify git status shows clean changes

## Performance Considerations

### File System Operations

**Concern:** Reading `.active` on every command

**Impact:** Negligible - single file read, <1ms

**Optimization:** Cache in memory if called repeatedly in same session

```javascript
let cachedActive = null;

function getActiveProject(useCache = false) {
  if (useCache && cachedActive !== null) {
    return cachedActive;
  }

  cachedActive = fs.readFileSync('.planning/.active', 'utf-8').trim();
  return cachedActive;
}
```

### Project Listing

**Concern:** Reading all project directories for listing

**Impact:** Low - typically <10 projects, stat operations are fast

**Optimization:** Not needed unless >100 projects (unlikely)

### Migration

**Concern:** Moving files during migration

**Impact:** One-time operation, acceptable to block

**Optimization:** Already using fs.renameSync (atomic on same filesystem)

## Sources

### Tools & Patterns
- [Yarn Workspaces](https://classic.yarnpkg.com/blog/2017/08/02/introducing-workspaces/) - Monorepo workspace patterns
- [Terraform Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces) - Multi-environment state management
- [pnpm Workspaces](https://pnpm.io/workspaces) - Modern workspace protocol
- [Git Worktree Documentation](https://git-scm.com/docs/git-worktree) - Multiple working directory management
- [Mastering Git Worktree](https://mskadu.medium.com/mastering-git-worktree-a-developers-guide-to-multiple-working-directories-c30f834f79a5) - Best practices for worktree layout

### Context Switching
- [kubectl config](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/) - Context management patterns
- [kubectl config current-context](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_current-context/) - Active context tracking
- [Understanding Kubernetes Contexts](https://medium.com/@ravipatel.it/understanding-kubernetes-contexts-and-kubeconfig-file-5c3d346c629e) - Kubeconfig file format
- [Kubeconfig File Explained](https://devopscube.com/kubernetes-kubeconfig-file/) - YAML configuration structure

### Dotfiles & Configuration
- [The Ultimate Guide to Mastering Dotfiles](https://www.daytona.io/dotfiles/ultimate-guide-to-dotfiles) - Configuration file best practices
- [How to Manage Dotfiles With Git](https://www.control-escape.com/linux/dotfiles/) - Version control for configs
- [Managing Your Dotfiles](https://effective-shell.com/part-5-building-your-toolkit/managing-your-dotfiles/) - Hierarchical configuration patterns

### Version Management
- [asdf Version Manager](https://asdf-vm.com/) - .tool-versions file format
- [Switching to asdf](https://jinyuz.dev/posts/switching-from-env-to-asdf/) - Multi-tool version management
- [asdf Beginners Guide](https://blog.techatpower.com/asdf-a-beginners-guide-to-tool-version-management-81ee84f57c7b) - Local vs global versions

### Directory Context
- [direnv Documentation](https://direnv.net/) - Auto-loading environment variables
- [5 Ways to Manage Environment Variables with direnv](https://www.sixfeetup.com/blog/direnv-manage-environment-variables) - Multi-level configuration
- [Discovering direnv](https://rednafi.com/misc/direnv/) - Project-specific environments
- [direnv stdlib](https://direnv.net/man/direnv-stdlib.1.html) - source_up inheritance patterns

### Monorepo Tools
- [Nx vs Turborepo](https://www.wisp.blog/blog/nx-vs-turborepo-a-comprehensive-guide-to-monorepo-tools) - Monorepo tool comparison
- [Monorepo Explained](https://monorepo.tools/) - Comprehensive patterns overview
- [Architecting a Modern Monorepo](https://blog.theodo.com/2022/02/architecting-a-modern-monorepo/) - Nx and Turborepo architecture
- [Monorepo Demystified](https://dev.to/werliton/monorepo-demystified-turborepo-vs-lerna-vs-nx-which-one-should-you-choose-3aeh) - Tool selection criteria

---
*Stack research for: Multi-Project CLI Tool Support*
*Researched: 2026-02-04*
