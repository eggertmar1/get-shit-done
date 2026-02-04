# External Integrations

**Analysis Date:** 2026-02-04

## APIs & External Services

**npm Registry:**
- Service: npm (Node Package Package Registry)
- What it's used for: Check for available updates to GSD package
  - Client: Node.js `execSync` subprocess
  - Command: `npm view get-shit-done-cc version`
  - Timeout: 10 seconds
  - Implementation: `hooks/gsd-check-update.js` line 45

**GitHub:**
- Service: GitHub repository hosting
- What it's used for: Source code distribution and community contributions
  - Repository URL: https://github.com/glittercowboy/get-shit-done
  - Bug reports: https://github.com/glittercowboy/get-shit-done/issues
  - Used for: Issues, pull requests, releases

## AI Runtimes (Integration Targets)

**Claude Code:**
- Integration point: Markdown commands in `.claude/commands/gsd/`
- Config directory: `~/.claude/` (global) or `./.claude/` (local)
- Env var override: `CLAUDE_CONFIG_DIR`
- Hooks location: `{config}/hooks/gsd-*.js`
- Settings file: `{config}/settings.json`
- Features:
  - Read/write permissions configured for GSD doc paths
  - External directory permissions for safe file access
  - Statusline hook for context window visualization
  - Session-start hook for update checking

**OpenCode:**
- Integration point: Markdown commands in `.opencode/commands/gsd/`
- Config directory: Follows XDG Base Directory Specification
  - Priority 1: `$OPENCODE_CONFIG_DIR` (env var)
  - Priority 2: Directory of `$OPENCODE_CONFIG` (env var)
  - Priority 3: `$XDG_CONFIG_HOME/opencode`
  - Priority 4: `~/.config/opencode` (XDG default)
- Env var overrides: `OPENCODE_CONFIG_DIR`, `OPENCODE_CONFIG`, `XDG_CONFIG_HOME`
- Features: Same as Claude Code

**Gemini:**
- Integration point: Markdown commands in `.gemini/commands/gsd/`
- Config directory: `~/.gemini/` (global) or `./.gemini/` (local)
- Env var override: `GEMINI_CONFIG_DIR`
- Features: Same as Claude Code

## Data Storage

**Local File System:**
- All data stored locally, no cloud services
- Config directories: Runtime-specific (Claude, OpenCode, Gemini)
- Cache location: `~/.claude/cache/gsd-update-check.json`
- Todo tracking: `~/.claude/todos/` (session-based JSON files)
- VERSION file: Tracks installed GSD version in config directory

**Session Management:**
- Session IDs from Claude Code context
- Todo files: `{session_id}-agent-{date}.json`
- Statusline reads latest todo file for current task
- No remote session storage

## Caching

**Update Check Cache:**
- Location: `~/.claude/cache/gsd-update-check.json`
- Content:
  ```json
  {
    "update_available": boolean,
    "installed": "version",
    "latest": "version",
    "checked": unix_timestamp
  }
  ```
- Lifespan: Session-based (checked once per session start)
- Purpose: Avoid repeated npm registry queries

## Authentication & Identity

**Auth Provider:**
- Not applicable - GSD itself has no authentication
- Installation: Requires file system write permissions to config directories
- Permission model: File system ACLs only

**Runtime Authentication:**
- Claude Code: Uses Claude's internal authentication
- OpenCode: Uses OpenCode's internal authentication
- Gemini: Uses Gemini's internal authentication
- GSD does not manage credentials

## Monitoring & Observability

**Error Tracking:**
- Not integrated with any external service
- Local console output for errors during installation
- Silent failures in hooks (prevents statusline crashes)

**Logs:**
- Console output to stdout/stderr during installation
- GSD statusline shows colored progress indicators
- Cache files for update check status
- No external log aggregation

**Debugging:**
- File system inspection for state (VERSION, cache, todos)
- Local JSON inspection for status and caching

## CI/CD & Deployment

**Hosting:**
- npm registry (public package)
- GitHub (source repository)

**CI Pipeline:**
- GitHub Actions (assumed from `.github/` directory)
- Pre-publish: `npm run build:hooks` (esbuild hook compilation)

**Package Publication:**
- npm registry: https://www.npmjs.com/package/get-shit-done-cc
- Published artifacts:
  - bin/ (CLI installer)
  - commands/gsd/ (command specs)
  - get-shit-done/ (templates and references)
  - agents/ (agent prompts)
  - hooks/dist/ (compiled hooks)
  - scripts/ (build utilities)

**Version Control:**
- Repository: github.com/glittercowboy/get-shit-done
- Git hooks: Referenced but not implemented in core GSD (for user projects)

## Webhooks & Callbacks

**Incoming:**
- None - GSD is passive, CLI-driven

**Outgoing:**
- None - GSD only reads from npm and file system

**Hook System:**
- Claude Code/OpenCode/Gemini hooks are **integration points**, not webhooks
- `gsd-statusline.js` - Called by runtime on statusline render
- `gsd-check-update.js` - Called by SessionStart hook
- Hooks read from stdin (runtime data), write to stdout
- No bidirectional communication protocol

## Installation & Configuration Flow

**Installation Process:**
1. User runs: `npx get-shit-done-cc`
2. Interactive prompt for runtime (Claude/OpenCode/Gemini)
3. Interactive prompt for location (global or local)
4. Files copied to target config directory
5. `settings.json` configured with read/external-directory permissions
6. Statusline hook command configured in runtime settings
7. VERSION file written for future update checks

**Config Directory Structure (post-install):**
```
~/.claude/
├── commands/gsd/          # All GSD commands as markdown files
├── get-shit-done/         # Workflows, templates, references
│   ├── workflows/         # Execution workflows
│   ├── templates/         # Project/phase templates
│   ├── references/        # Style guides, patterns
│   └── VERSION            # Installed GSD version
├── hooks/                 # Hook scripts
│   ├── gsd-statusline.js
│   └── gsd-check-update.js
├── todos/                 # Session task tracking
├── cache/
│   └── gsd-update-check.json
└── settings.json          # Runtime configuration
```

## Environment Detection

**Path Expansion:**
- `~` → `os.homedir()` (manual expansion)
- Handles shell limitations in subprocess execution

**Platform Detection:**
- `process.platform` for OS-specific behavior
- Forward slash normalization for Windows paths
- XDG Base Directory support for Linux

**TTY Detection:**
- `process.stdin.isTTY` for interactive vs non-interactive mode
- Non-interactive: defaults to Claude Code global install

---

*Integration audit: 2026-02-04*
