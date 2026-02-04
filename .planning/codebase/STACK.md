# Technology Stack

**Analysis Date:** 2026-02-04

## Languages

**Primary:**
- JavaScript (Node.js) - CLI tool, installation scripts, hooks, build system
- Markdown - Agent prompts, command specifications, documentation, templates

**No Secondary Languages:**
- Codebase is pure JavaScript/Node.js with no TypeScript, Python, or compiled languages

## Runtime

**Environment:**
- Node.js >=16.7.0 (specified in `package.json` engines)
- Cross-platform: macOS, Windows, Linux

**Package Manager:**
- npm (Node Package Manager)
- Lockfile: `package-lock.json` present (version 3 lockfile format)

## Frameworks

**Build/Development:**
- esbuild ^0.24.0 - Bundling and transpilation for hooks
  - Config: `scripts/build-hooks.js` - Simple file copy build system
  - Primary use: Copy JavaScript hooks to `hooks/dist/` for distribution

**Core Architecture:**
- No web framework (CLI-based tool)
- No React, Vue, or frontend framework
- No backend framework (Express, Koa, etc.)

## Key Dependencies

**Production Dependencies:**
- None listed in `package.json` - Zero runtime dependencies
- All functionality uses Node.js built-in modules

**Development Dependencies:**
- esbuild ^0.24.0 - Build/packaging only

**Built-in Modules Used:**
- `fs` - File system operations
- `path` - File path manipulation
- `os` - OS utilities (home directory, platform detection)
- `readline` - Interactive CLI prompts
- `child_process` - Subprocess execution (execSync, spawn)

## Configuration

**Environment Variables:**
- `CLAUDE_CONFIG_DIR` - Override Claude Code global config location (defaults to `~/.claude`)
- `OPENCODE_CONFIG_DIR` - Override OpenCode global config location
- `OPENCODE_CONFIG` - Alternative OpenCode config path
- `XDG_CONFIG_HOME` - XDG Base Directory Specification support (OpenCode only)
- `GEMINI_CONFIG_DIR` - Override Gemini global config location (defaults to `~/.gemini`)

**Installation Configuration:**
- Command-line flags control install behavior:
  - `--global` / `-g` - Install to runtime's global config directory
  - `--local` / `-l` - Install to current project's local directory
  - `--claude`, `--opencode`, `--gemini` - Target specific runtime
  - `--all` - Install for all runtimes
  - `--config-dir <path>` / `-c <path>` - Custom config directory
  - `--uninstall` / `-u` - Remove GSD installation

**Build Configuration:**
- `scripts/build-hooks.js` - Simple Node.js script, no build tool config needed
- Pre-publish hook runs build automatically

## Installation & Distribution

**Package Distribution:**
- Published to npm registry as `get-shit-done-cc`
- Current version: 1.11.1
- Repository: https://github.com/glittercowboy/get-shit-done
- Installation method: `npx get-shit-done-cc`

**Entry Point:**
- `bin/install.js` - Main CLI installer script
  - Installed as global command `get-shit-done-cc` via npm bin directive
  - Handles all runtime setup (Claude Code, OpenCode, Gemini)

**Packaged Files:**
- `bin/` - Installation scripts
- `commands/gsd/` - Command definitions and workflows
- `get-shit-done/` - Core templates and references
- `agents/` - Agent prompt definitions
- `hooks/dist/` - Compiled hooks
- `scripts/` - Build utilities

## Runtime Hooks

**Installed Hooks:**
- `gsd-statusline.js` - Claude Code statusline integration
  - Location: `hooks/gsd-statusline.js`
  - Reads from stdin: model info, context window, workspace data
  - Outputs: formatted status bar with context usage, task info
  - Color-coded context window visualization
  - Updates via GSD update indicator

- `gsd-check-update.js` - Version checking integration
  - Location: `hooks/gsd-check-update.js`
  - Spawns background process to check npm registry
  - Compares installed vs latest version from npm
  - Caches result: `~/.claude/cache/gsd-update-check.json`
  - Called on session start

## External Service Integrations

**npm Registry:**
- Queries: `npm view get-shit-done-cc version`
- Purpose: Check for available updates
- Implementation: `gsd-check-update.js` uses execSync subprocess

**Version Management:**
- Reads from `package.json` during install
- Writes VERSION file to installed location
- Format: Semantic versioning (e.g., "1.11.1")
- Supports local and global install version detection

## Platform Support

**Operating Systems:**
- macOS - Full support
- Windows - Full support (PowerShell/cmd.exe compatible)
- Linux - Full support (XDG Base Directory compliant)

**Special Handling:**
- Windows: Forward slash conversion for cross-platform paths
- Linux: XDG Base Directory Specification support for OpenCode
- Tilde expansion: Manual path expansion for all platforms (shells don't expand in env vars passed to Node)

## Script Execution

**Update Check Process:**
- Spawned as background process (detached)
- Windows hidden window (`windowsHide: true`)
- 10-second timeout for npm registry query
- Graceful failure if npm is unavailable
- Results cached for performance

**Installation Process:**
- Interactive prompts for runtime selection
- Fallback to defaults in non-TTY environments
- Path validation and expansion
- Directory creation with recursive option
- File copying and verification
- Permission configuration for AI runtimes

---

*Stack analysis: 2026-02-04*
