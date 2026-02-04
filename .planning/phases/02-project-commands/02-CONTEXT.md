# Phase 2: Project Commands - Context

**Gathered:** 2026-02-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver CLI commands for managing multiple projects: create, switch, list, and archive. Users interact with projects through `/gsd:new-project [name]`, `/gsd:switch-project <name>`, `/gsd:list-projects`, and `/gsd:archive-project [name]`. Running GSD commands without an active project prompts for selection.

</domain>

<decisions>
## Implementation Decisions

### List display
- Show project name + current phase + progress percentage
- Active project marker: Claude's discretion
- Archived projects never appear in list (archive = removal from view)
- Always use table format, even with single project
- Paths shown only with --verbose flag
- Projects with no roadmap show as 0% progress (Phase -, 0%)

### Claude's Discretion
- Active project indicator style (arrow, asterisk, color, etc.)
- Empty state message (friendly guidance vs minimal)
- Sort order for project list
- Exact table column formatting and spacing

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

*Phase: 02-project-commands*
*Context gathered: 2026-02-04*
