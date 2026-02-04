# Codebase Concerns

**Analysis Date:** 2026-02-04

## Tech Debt

**Fragile JSON Parsing in Workflows:**
- Issue: Multiple workflows use grep/sed patterns to extract JSON values instead of proper JSON parsing
- Files: `agents/gsd-executor.md:47`, `agents/gsd-planner.md:1042`, `get-shit-done/workflows/execute-phase.md:20,62,76-77`, `commands/gsd/execute-phase.md:45,100`
- Impact: Configuration settings may silently fall back to defaults if JSON formatting varies. Examples of fragile patterns:
  ```bash
  # Fails if JSON is minified, has different spacing, or uses unquoted values
  MODEL_PROFILE=$(cat .planning/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
  COMMIT_PLANNING_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
  ```
- Fix approach: Replace with `jq` for robust JSON parsing, or create a Node.js helper script for JSON extraction that workflows can call

**Hardcoded Paths Scattered Throughout Codebase:**
- Issue: Directory paths (`.planning/`, `~/.claude/`, `~/.opencode/`, `~/.gemini/`, etc.) are hardcoded in many files
- Files: `hooks/gsd-statusline.js:49,71`, `hooks/gsd-check-update.js:12,16-17`, multiple agent and workflow files, `bin/install.js` path constants
- Impact: Directory structure changes would require updates in many locations; increases maintenance burden
- Fix approach: Centralize path constants in shared configuration module or environment variables

**Inconsistent Error Handling Patterns:**
- Issue: Error handling is inconsistent across the codebase:
  - Some functions have comprehensive try-catch blocks
  - Others rely on optional chaining or existence checks
  - Some fail silently (good for robustness), others propagate errors
- Files: `bin/install.js`, `hooks/gsd-statusline.js`, `hooks/gsd-check-update.js`, workflow files
- Impact: Unpredictable behavior when edge cases occur; difficult to debug production issues
- Fix approach: Establish documented error handling standards across all files

## Known Bugs

**File System Race Conditions in statusline.js (FIXED in v1.11.1+):**
- Symptoms: Statusline crashes with ENOENT or permission errors when files are accessed while being modified
- Files: `hooks/gsd-statusline.js:51-54`
- Trigger: Rapid creation/deletion of files in `~/.claude/todos/` while statusline is reading them
- Workaround: Fixed - directory reading operations now wrapped in try-catch (line 51-66)
- Status: ✅ FIXED - Added proper error handling per `FIXES_APPLIED.md`

**Git Operations Violate Commit Rules (FIXED in v1.11.1+):**
- Symptoms: Used `git add -u` instead of individual file staging
- Files: `commands/gsd/execute-phase.md:94`
- Trigger: When orchestrator made corrections to phase execution
- Status: ✅ FIXED - Replaced with individual file staging pattern per `FIXES_APPLIED.md`

**Hex Color Validation Missing (FIXED in v1.11.1+):**
- Symptoms: Invalid hex color values in frontmatter accepted without validation
- Files: `bin/install.js:437-441` (before fix)
- Trigger: User providing malformed color codes like `#ZZZ`
- Status: ✅ FIXED - Added regex validation for hex format per `FIXES_APPLIED.md`

## Security Considerations

**Subprocess Command Execution via execSync:**
- Risk: `gsd-check-update.js` uses `execSync('npm view ...')` which could be vulnerable to command injection if paths contain special characters
- Files: `hooks/gsd-check-update.js:45`
- Current mitigation: Uses static command string with no user input interpolation; timeout enforced; stdio redirected
- Recommendations: Continue avoiding user input in shell commands; maintain strict whitelist of allowed executables

**File Path Expansion with Tilde:**
- Risk: `~` expansion in paths passed to Node.js (shell doesn't expand automatically)
- Files: `bin/install.js:105-109` (expandTilde function), multiple calls throughout
- Current mitigation: Custom expandTilde function properly uses `os.homedir()` instead of shell expansion
- Recommendations: Ensure all path handling uses expandTilde consistently; never pass unexpanded `~` to file system operations

**JSON Parsing from Config Files:**
- Risk: Grep/sed patterns are fragile and could be tricked by malformed input; improper escaping could cause issues
- Files: Multiple workflow files reading `.planning/config.json`
- Current mitigation: Fallback to defaults if parsing fails (safe but hides errors)
- Recommendations: Use proper JSON parser (jq or Node.js) for all config reading

**Tool Name Validation in Installer:**
- Risk: Tool names converted for Gemini/OpenCode without validation; could accept malicious tool names
- Files: `bin/install.js:395-420` (convertGeminiToolName), `bin/install.js:440-542` (convertClaudeToOpencodeFrontmatter)
- Current mitigation: Only known tool names are mapped; unknown names silently skipped
- Recommendations: Consider validation whitelist of allowed tool names

## Performance Bottlenecks

**Large Files with Complex Logic:**
- Problem: Several files approach or exceed complexity limits for a single document
- Files:
  - `bin/install.js:1529` lines - Very large installer with many code paths
  - `agents/gsd-planner.md:1418` lines - Complex planner with multiple decision trees
  - `get-shit-done/workflows/execute-plan.md:1844` lines - Longest workflow file
- Cause: Centralized logic that handles multiple runtimes (Claude, OpenCode, Gemini) and features
- Improvement path: Consider breaking large files into smaller modules or splitting runtime-specific logic

**Background Process for Update Check:**
- Problem: `gsd-check-update.js` spawns a background process that makes an npm registry call
- Files: `hooks/gsd-check-update.js`
- Current behavior: Network call could be slow (mitigated by 10s timeout), but runs in background so not blocking
- Improvement path: Implement caching with expiration; consider using faster registry mirrors

## Fragile Areas

**YAML Parsing in Frontmatter Conversion:**
- Files: `bin/install.js:440-542` (convertClaudeToOpencodeFrontmatter), `bin/install.js:325-437` (convertClaudeToGeminiFrontmatter)
- Why fragile: Simple line-by-line YAML parsing without proper YAML library; assumes specific formatting
- Safe modification: Add extensive test cases for malformed YAML; consider using a proper YAML parser library
- Test coverage: No explicit tests visible for frontmatter conversion edge cases

**JSON Config Reading Across Workflows:**
- Files: Multiple workflow files with grep/sed JSON extraction
- Why fragile: Pattern matching assumes specific JSON formatting; will fail silently on variations
- Safe modification: Centralize JSON reading in a helper function or use jq exclusively
- Test coverage: Gaps in config parsing edge cases (minified JSON, escaped quotes, etc.)

**State File Management:**
- Files: `agents/gsd-executor.md`, `agents/gsd-planner.md`, workflow files reading `.planning/STATE.md`
- Why fragile: State files are critical but have no explicit schema validation; missing files handled with fallback logic
- Safe modification: Define explicit STATE.md format; add schema validation on read
- Test coverage: Edge cases like corrupted state files not thoroughly tested

**Multi-Runtime Support in Installer:**
- Files: `bin/install.js` handles Claude Code, OpenCode, and Gemini with different file structures
- Why fragile: Multiple conversion functions with different logic paths; edge cases where conversions might conflict
- Safe modification: Extract runtime-specific logic into separate modules; add comprehensive integration tests
- Test coverage: Need tests for all runtime combinations and cross-runtime scenarios

## Scaling Limits

**No Explicit Rate Limiting on npm Registry Calls:**
- Current capacity: Single npm view call per session start
- Limit: If many users trigger GSD sessions simultaneously, could hit npm registry rate limits
- Scaling path: Implement local cache with TTL; batch requests; consider using faster registries

**Concurrent Agent Spawning:**
- Current capacity: Config allows `max_concurrent_agents: 3` (from `get-shit-done/templates/config.json:18`)
- Limit: No apparent enforcement mechanism visible in agent spawning logic
- Scaling path: Implement queue-based agent spawning with concurrency limiter; monitor resource usage

## Dependencies at Risk

**No npm Dependencies in package.json:**
- Risk: Project has zero production dependencies (only esbuild for dev)
- Impact: Fewer external vulnerabilities but all features implemented in-house (larger attack surface for features like JSON parsing)
- Migration plan: Consider adding jq as required CLI tool or node-json-parser for robustness

**shell-based Workflows with Bash:**
- Risk: Heavy reliance on bash for critical workflows; different behavior across OSes (Linux/Mac/Windows)
- Files: Multiple workflow files using bash-specific features
- Impact: Could have cross-platform issues despite Node.js wrapper
- Improvement path: Gradual migration of critical shell scripts to JavaScript/TypeScript

## Missing Critical Features

**No Explicit JSON Schema for Config Files:**
- Problem: `.planning/config.json` structure is documented but not validated
- Blocks: Invalid configs could cause silent failures or unexpected behavior
- Recommendation: Add JSON schema file or runtime validation schema

**No Update Mechanism for GSD Itself in Long Sessions:**
- Problem: Update check runs once at session start; updates within session not detected
- Blocks: Users won't know about critical security fixes until session restart
- Recommendation: Consider periodic background update checks or user notification system

## Test Coverage Gaps

**Frontmatter Conversion:**
- What's not tested: OpenCode/Gemini frontmatter conversion edge cases
- Files: `bin/install.js:325-542` (conversion functions)
- Risk: Malformed generated files could cause issues for users
- Priority: High - affects user-facing output

**JSON Config Parsing:**
- What's not tested: Config parsing with various JSON formatting (minified, with escape sequences, empty values)
- Files: Multiple workflow files
- Risk: Silent failures when config is malformed
- Priority: High - affects workflow behavior

**File System Operations:**
- What's not tested: Race conditions, permission errors, symlinks, special characters in paths
- Files: `bin/install.js`, `hooks/gsd-statusline.js`, `hooks/gsd-check-update.js`
- Risk: Crashes in production, especially in multi-user environments
- Priority: Medium - partially mitigated by try-catch blocks

**Cross-Platform Compatibility:**
- What's not tested: Windows-specific path handling, different line endings, shell differences
- Files: Multiple bash workflows, path handling code
- Risk: Behavior differences across operating systems
- Priority: Medium - project supports multiple platforms

**Multi-Runtime Installation:**
- What's not tested: Installing for multiple runtimes simultaneously (Claude + OpenCode + Gemini)
- Files: `bin/install.js` with `--all` flag
- Risk: Conflicts, partially installed state, or incomplete configurations
- Priority: Medium - new feature from v1.10.0

---

*Concerns audit: 2026-02-04*
