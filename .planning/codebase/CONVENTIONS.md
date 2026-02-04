# Coding Conventions

**Analysis Date:** 2026-02-04

## Naming Patterns

**Files:**
- Node.js scripts: kebab-case with `.js` extension
  - `bin/install.js` - installer entry point
  - `hooks/gsd-statusline.js` - hook script
  - `scripts/build-hooks.js` - build script
- Markdown command/agent files: kebab-case with `.md` extension
  - `commands/gsd/help.md` - command definitions
  - `agents/gsd-codebase-mapper.md` - agent specifications
- Configuration files: lowercase with dots
  - `package.json` - NPM manifest
  - `.prettierrc`, `.eslintrc` - linting (not present in this project)

**Functions:**
- camelCase for all function names
- Examples: `getDirName()`, `getGlobalDir()`, `parseConfigDirArg()`, `expandTilde()`, `buildHookCommand()`
- Use clear action verbs: get, parse, build, convert, copy, verify, clean

**Variables:**
- camelCase for all variables and constants
- Examples: `selectedRuntimes`, `cacheDir`, `configPath`, `hasGlobal`, `isOpencode`
- Descriptive naming preferred over abbreviated names
- Boolean variables prefixed with `is`, `has`, `should`: `isGlobal`, `hasGlobal`, `forceStatusline`

**Types/Classes:**
- PascalCase for object keys and structured data types
- Examples: `{ name: 'gsd:help', description: '...', tools: [...] }`
- Map keys use lowercase: `claudeToOpencodeTools = { AskUserQuestion: 'question' }`

## Code Style

**Formatting:**
- No automatic formatter configured (no .prettierrc or ESLint setup)
- Manual formatting follows JavaScript standard conventions
- 2-space indentation (Node.js standard)
- Line length: no hard limit observed, but code generally stays under 100 chars
- Trailing newline in JSON files (seen in package.json, settings files)

**Linting:**
- No linter configuration found (.eslintrc*, biome.json, etc.)
- Code quality maintained through review and testing

**Semicolons:**
- Semicolons used consistently at end of statements
- No ASI (Automatic Semicolon Insertion) patterns observed

## Import Organization

**Node.js Requires:**
- Group 1: Node.js built-in modules (fs, path, os, readline, child_process)
- Group 2: npm packages (if any)
- Group 3: Local/relative imports

**Pattern in `bin/install.js`:**
```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');
const readline = require('readline');

const pkg = require('../package.json');
```

**Pattern in hooks:**
```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');
const { spawn } = require('child_process');
```

**No path aliases used** - Direct relative paths throughout codebase

## Error Handling

**Try-Catch Patterns:**
- Used for non-critical errors to fail silently
- Example in `gsd-statusline.js`: catches JSON parse errors, continues execution
- Pattern: Try operation, catch error, assign default/fallback value

```javascript
try {
  const todos = JSON.parse(fs.readFileSync(path.join(todosDir, files[0].name), 'utf8'));
  const inProgress = todos.find(t => t.status === 'in_progress');
  if (inProgress) task = inProgress.activeForm || '';
} catch (e) {
  // Silently fail - don't break statusline
}
```

**Exit Codes:**
- `process.exit(0)` - successful exit
- `process.exit(1)` - error exit with console.error() message

**Error Messages:**
- Prefixed with colored status indicators: `${yellow}⚠${reset}`, `${green}✓${reset}`
- User-facing messages use console.log() with color codes
- Development warnings use console.warn()

## Logging

**Framework:** Node.js console object (console.log, console.error)

**Patterns:**
- Status indicators with ANSI color codes
- `console.log()` for normal output, info, and warnings
- `console.error()` for actual errors
- No timestamp logging; assumes Claude IDE handles session logging
- Silent failures common in hooks (graceful degradation)

**Color Codes Used:**
```javascript
const cyan = '\x1b[36m';      // Primary info
const green = '\x1b[32m';     // Success/checkmarks
const yellow = '\x1b[33m';    // Warnings
const dim = '\x1b[2m';        // Secondary info
const reset = '\x1b[0m';      // Reset color
```

**Examples:**
```javascript
console.log(`  ${cyan}1${reset}) Global ${dim}(${pathExamples})${reset}`);
console.log(`  ${green}✓${reset} Installed ${count} commands to command/`);
console.error(`  ${yellow}✗${reset} Failed to install ${description}`);
```

## Comments

**When to Comment:**
- Complex logic requiring explanation: color mapping, path normalization, format conversion
- Non-obvious algorithms: version checking, context window calculations
- Cross-platform concerns: Windows vs Unix path handling, console output

**JSDoc/TSDoc:**
- Used for exported functions, especially with complex parameters
- Documents parameter types, return types, and behavior
- Example from `bin/install.js`:

```javascript
/**
 * Get the global config directory for OpenCode
 * OpenCode follows XDG Base Directory spec and uses ~/.config/opencode/
 * Priority: OPENCODE_CONFIG_DIR > dirname(OPENCODE_CONFIG) > XDG_CONFIG_HOME/opencode > ~/.config/opencode
 */
function getOpencodeGlobalDir() {
  // Implementation with numbered priority comments
}

/**
 * Convert Claude Code frontmatter to opencode format
 * - Converts 'allowed-tools:' array to 'permission:' object
 * @param {string} content - Markdown file content with YAML frontmatter
 * @returns {string} - Content with converted frontmatter
 */
function convertClaudeToOpencodeFrontmatter(content) {
  // Implementation
}
```

**Inline Comments:**
- Explain WHY, not WHAT (code should be clear)
- Label with numbers when showing multi-step logic (see `getOpencodeGlobalDir()`)
- Example:
  ```javascript
  // Use forward slashes for cross-platform compatibility
  const hooksPath = configDir.replace(/\\/g, '/') + '/hooks/' + hookName;
  ```

**TODO/FIXME:**
- Format: `// TODO: description (optional issue reference)`
- Observed in codebase, but not heavily used
- Example: See agent documentation patterns

## Function Design

**Size:**
- Large functions acceptable when they group related operations
- Example: `install()` function spans ~200 lines because it orchestrates installation steps
- Related helper functions extracted for readability: `copyWithPathReplacement()`, `verifyInstalled()`

**Parameters:**
- Use default parameters for optional values
- Example: `function getGlobalDir(runtime, explicitDir = null)`
- Maximum 3-4 parameters; bundle related params into objects for larger functions

**Return Values:**
- Functions return null/undefined for missing values
- Objects returned with consistent structure:
  ```javascript
  return { settingsPath, settings, statuslineCommand, runtime };
  ```
- No early returns if single return satisfies logic

## Module Design

**Exports:**
- Node.js modules export functions directly via assignment
- No module.exports pattern observed (mostly scripts with direct execution)
- Scripts use functions at module scope for organization

**Barrel Files:**
- Not used in this project
- Each module is single-responsibility

**File Organization:**
- Constants at top (color codes, mappings)
- Helper functions before main functions
- Main entry logic at bottom of file
- Clear separation between phases (e.g., argument parsing → helper functions → main logic)

## Markdown Command/Agent Structure

**Frontmatter:**
- YAML format with standard fields
- Fields: `name`, `description`, `tools`, `color` (optional)

```yaml
---
name: gsd-codebase-mapper
description: Explores codebase and writes structured analysis documents.
tools: Read, Bash, Grep, Glob, Write
color: cyan
---
```

**Body:**
- XML-like role and context sections
- Markdown for readable documentation
- Code blocks with language annotation
- `<objective>`, `<reference>`, `<process>`, `<philosophy>` tags for structuring content

**Tool References:**
- Tools specified in frontmatter tools field
- Format: comma-separated tool names matching Claude Code tool names
- Examples: `Read`, `Bash`, `Grep`, `Glob`, `Write`, `Edit`

## Common Patterns

**Validation Pattern:**
- Check existence/format before processing
- Example: Parse command-line args, validate non-empty, show help if invalid

**Recursive File Operations:**
- Pattern: Process directory with `fs.readdirSync()`, recurse on subdirectories
- See: `copyFlattenedCommands()`, `copyWithPathReplacement()` in `bin/install.js`

**Settings Management:**
- Read JSON settings file, apply modifications, write back with pretty-print
- Preserve existing settings, add/update GSD-specific fields
- Example: `cleanupOrphanedHooks()` modifies settings.json without removing user content

**Path Normalization:**
- Expand `~` to home directory: `expandTilde()`
- Handle both Windows and Unix paths
- Cross-platform: Use forward slashes in path strings for Node.js

---

*Convention analysis: 2026-02-04*
*Update when coding patterns change or new conventions are adopted*
