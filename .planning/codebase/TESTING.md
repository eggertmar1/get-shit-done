# Testing Patterns

**Analysis Date:** 2026-02-04

## Test Framework

**Status:** Not detected

**Runner:**
- No test framework configured (no jest.config.js, vitest.config.ts, mocha.opts, etc.)
- No test script in package.json
- No test dependencies listed in devDependencies

**Assertion Library:**
- Not applicable

**Test Files:**
- No *.test.js, *.spec.js, *.test.ts, or *.spec.ts files found in codebase
- No __tests__/ directory structure

**Run Commands:**
- No test commands defined in package.json
- No test coverage tooling configured

## Test Philosophy

**Current Approach:** Manual testing and integration testing via CLI

This project is a meta-prompting and development automation system, not a traditional application. Testing is primarily:

1. **CLI Integration Testing:** Functionality verified through actual command-line usage
   - Installation process tested end-to-end with real file system operations
   - Hook integration verified by running in Claude Code environment
   - Command/agent execution tested through actual invocation

2. **Code Review:** Agent and command definitions reviewed for correctness
   - Markdown frontmatter validated manually
   - YAML parsing checked by inspection
   - Tool specifications verified for accuracy

3. **User Validation:** Feedback from real users running GSD workflows

## Code Structure for Maintainability

**Testable Code Patterns:**

While formal tests don't exist, the codebase is structured for potential testing:

**Pure Functions:**
- `expandTilde()` - pure function that could be unit tested
- `convertToolName()` - pure string transformation function
- `processAttribution()` - string processing with regex

**Modular Functions:**
- Helper functions extracted for single responsibility
- Examples:
  - `getGlobalDir()` - resolve config directory paths
  - `getOpencodeGlobalDir()` - specific runtime path resolution
  - `buildHookCommand()` - construct hook paths
  - `readSettings()` / `writeSettings()` - JSON file I/O

**Separation of Concerns:**
- File I/O operations isolated (fs.readFileSync, fs.writeFileSync)
- Path operations grouped (path.join, path.dirname, expandTilde)
- JSON transformations extracted (convertClaudeToOpencodeFrontmatter, etc.)
- CLI interaction separated (readline prompts, process.exit)

## Testing Opportunities

**If tests were to be added, focus areas:**

**Unit Test Candidates:**
- Path utility functions: `expandTilde()`, `getDirName()`, `buildHookCommand()`
- String converters: `convertToolName()`, `processAttribution()`, `stripSubTags()`
- YAML/Frontmatter parsers: `convertClaudeToOpencodeFrontmatter()`, `convertClaudeToGeminiAgent()`
- Settings management: `readSettings()`, `writeSettings()`, `cleanupOrphanedHooks()`

**Integration Test Candidates:**
- Installation flow: Global vs local, single runtime vs multiple runtimes
- Frontmatter conversion: Claude → OpenCode format transformation
- File copying with path replacement: `copyWithPathReplacement()`, `copyFlattenedCommands()`
- Hook registration: Adding/removing hooks from settings.json
- Uninstall process: File cleanup and settings restoration

**Mock Targets (if tests added):**
- `fs` module (readFileSync, writeFileSync, mkdirSync, rmSync)
- `process` module (process.argv, process.cwd(), process.exit)
- `os` module (os.homedir(), os.platform)
- `child_process.spawn` - for background processes
- Command-line prompts (readline.createInterface)

## Error Handling Validation

**Current Validation Approach:**

1. **File System Checks:**
   - `verifyInstalled()` - checks directory exists and contains files
   - `verifyFileInstalled()` - checks file was created
   - Used after each installation step

2. **Silent Failures in Hooks:**
   - Hooks gracefully degrade on errors
   - Example: `gsd-statusline.js` silently continues if JSON parse fails
   - Prevents user experience degradation from missing data

3. **Explicit Exit Codes:**
   - `process.exit(1)` on critical errors (prevents silent failures)
   - `process.exit(0)` on success
   - Examples: invalid arguments, required flags missing

**Pattern in `bin/install.js` (lines 1097-1098):**
```javascript
if (failures.length > 0) {
  console.error(`\n  ${yellow}Installation incomplete!${reset} Failed: ${failures.join(', ')}`);
  process.exit(1);
}
```

## Test Types (if implemented)

**Unit Tests:**
- Test single pure functions in isolation
- Mock file system and process
- Fast execution
- Examples: path converters, string transformations

**Integration Tests:**
- Test multiple modules working together
- Mock file system operations but test actual logic flow
- Examples: full install process, frontmatter conversion pipeline

**CLI Tests:**
- Test command-line argument parsing
- Would require mocking readline for interactive tests
- Examples: `--global vs --local`, `--config-dir` flag handling

**End-to-End Tests:**
- Not currently used
- Would require Docker or isolated test environment
- Would test actual file system operations
- Would verify hook integration in Claude Code

## Code Quality Indicators

**Strengths:**

1. **Defensive Programming:**
   - Null checks before operations
   - Path existence verified before reading/writing
   - Default values for optional parameters
   - Graceful degradation in hooks

2. **Clear Logic Flow:**
   - Top-level logic at end of file
   - Helper functions organized before use
   - Decision trees clearly structured (if-else chains)

3. **Error Messages:**
   - User-friendly with color coding
   - Actionable suggestions (e.g., "Use --force-statusline to replace")
   - No cryptic error codes

**Testing Gaps:**

1. **Cross-Platform Validation:**
   - Windows path handling code (replace backslashes) not tested in CI
   - XDG Base Directory spec parsing for OpenCode not validated

2. **Configuration Parsing:**
   - YAML frontmatter parsing logic complex and error-prone
   - No validation of frontmatter format before conversion
   - Invalid color names silently skipped (intentional but unverified)

3. **Concurrency:**
   - Background processes spawned without wait (via child.unref())
   - Race conditions possible if multiple GSD installs run simultaneously
   - No file locking mechanism

4. **Edge Cases:**
   - Empty config directories not explicitly tested
   - Symlinks and path traversal not handled
   - Very long paths (>260 chars on Windows) not addressed

## Manual Testing Checklist

**Installation:**
- [ ] Global install for Claude Code (`--claude --global`)
- [ ] Global install for OpenCode (`--opencode --global`)
- [ ] Local install for current project (`--local`)
- [ ] Multiple runtimes (`--all --global`)
- [ ] Custom config directory (`--config-dir /path`)
- [ ] Interactive mode (no flags)
- [ ] Uninstall process (`--uninstall`)

**Platform Coverage:**
- [ ] macOS (tested actively)
- [ ] Linux (XDG Base Directory compliance)
- [ ] Windows (path normalization with forward slashes)

**Configuration:**
- [ ] Settings.json creation and update
- [ ] Hook registration in settings
- [ ] Statusline prompt when existing config found
- [ ] Permission configuration for OpenCode

**File Operations:**
- [ ] Commands copied correctly
- [ ] Agents with converted frontmatter
- [ ] Path references updated (~/\.claude/ → actual paths)
- [ ] Attribution removed/replaced based on settings

---

*Testing analysis: 2026-02-04*
*Update when test framework added or testing approach changes*
