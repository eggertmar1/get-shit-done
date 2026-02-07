# Phase 4: Integration - Context

**Gathered:** 2026-02-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Verify and update all existing GSD commands, orchestrators, and agents so they correctly operate within the multi-project directory structure. Ensure `.active` is gitignored and functionality works across Claude Code, OpenCode, and Gemini runtimes. No new features — this phase makes existing features work with the structure established in Phases 1-3.

</domain>

<decisions>
## Implementation Decisions

### Verification approach
- A command is verified when it reads/writes files in the correct active project directory (not root `.planning/`)
- Claude chooses the verification method per command — manual walkthrough, automated script, or code inspection as appropriate
- No need for full round-trip or downstream consumption testing — correct path resolution is sufficient

### Command update scope
- If a command already works correctly with multi-project paths, Claude decides whether to leave it alone or standardize it
- Case-by-case judgment: don't fix what isn't broken, but standardize where it prevents future issues

### Plan structure
- The existing 2-plan split (04-01: orchestrator updates, 04-02: cross-runtime + gitignore) can be reorganized by the planner if research reveals a better structure

### Claude's Discretion
- Verification method per command (manual, automated, inspection)
- Whether to update already-working commands for consistency
- Plan split / reorganization based on research findings
- Runtime-specific adaptations if needed
- Gitignore strategy (per-repo vs global, additional local-only files)

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 04-integration*
*Context gathered: 2026-02-07*
