# Phase 3: Configuration & Context - Context

**Gathered:** 2026-02-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Enable per-project configuration overrides on top of global defaults, plus git branch intelligence for project naming. This phase builds the config merger — it does NOT add new settings or modify existing commands (that's Phase 4).

</domain>

<decisions>
## Implementation Decisions

### Configurable settings
- Claude decides which settings are overridable per-project vs global-only, based on each setting's nature
- Claude decides whether any project-only settings are needed (settings that don't exist at global level)
- No rigid "all or subset" rule — let the setting's purpose dictate whether per-project override makes sense

### Claude's Discretion
- Override behavior: how per-project configs merge with global (deep merge vs shallow, conflict resolution)
- Which specific settings are overridable per-project
- Whether project-only settings exist (e.g., project-specific model profile)
- `/gsd:settings` UX in multi-project context (edit project config, global config, or merged view)
- Whether project-level config.json is created on project init or only when user overrides something
- Branch-to-project mapping: how git branch auto-detection works for naming, including edge cases (detached HEAD, main branch, non-standard names)

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. User deferred all implementation choices to Claude's judgment across all areas.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-configuration-context*
*Context gathered: 2026-02-07*
