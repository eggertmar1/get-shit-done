# Architecture

**Analysis Date:** 2026-02-04

## Pattern Overview

**Overall:** Multi-layer orchestration system with specialized agent spawning and distributed context loading.

**Key Characteristics:**
- Orchestrator-Agent model: Thin orchestrators coordinate distributed agents
- Context-per-agent: Each agent gets fresh context (50-70% budget) rather than central context accumulation
- Template-driven: All outputs follow structured templates to ensure consistency
- State machine workflow: Projects move through defined phases with explicit status transitions
- File-based artifact management: All communication happens through markdown files in `.planning/` directory
- Prompt-as-code: Workflow orchestrators are markdown command specifications, not imperative scripts

## Layers

**Command/Orchestrator Layer:**
- Purpose: Parse arguments, coordinate distributed agents, aggregate results
- Location: `commands/gsd/` (29 command specifications)
- Contains: Orchestration workflows in markdown frontmatter + process steps
- Depends on: Agents, workflows, templates, project state
- Used by: User (via CLI), spawned via `gsd:command-name`
- Example entry points: `commands/gsd/execute-phase.md`, `commands/gsd/new-project.md`

**Agent Layer:**
- Purpose: Perform focused work on specific problems (codebase analysis, planning, execution, verification)
- Location: `agents/` (11 specialized agents)
- Contains: Agent prompts (role, context handling, process steps) in markdown
- Depends on: Codebase, project state, templates
- Used by: Orchestrators (spawned via Task tool)
- Examples: `agents/gsd-executor.md`, `agents/gsd-planner.md`, `agents/gsd-codebase-mapper.md`

**Workflow Layer:**
- Purpose: Define end-to-end processes that span multiple agent executions
- Location: `get-shit-done/workflows/` (12 workflow guides)
- Contains: Step-by-step execution contexts referenced by orchestrators
- Depends on: Templates, state schemas, references
- Used by: Orchestrators (loaded via `@` syntax)
- Example: `get-shit-done/workflows/execute-plan.md` defines how execute-phase orchestrator should work

**Template Layer:**
- Purpose: Provide consistent output structure and artifact formats
- Location: `get-shit-done/templates/` (multiple template categories)
- Contains: Markdown templates for project artifacts, phase artifacts, codebase analysis
- Subdirectories:
  - `codebase/` (7 templates): STACK.md, INTEGRATIONS.md, ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, CONCERNS.md
  - Root level: PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, PHASE-PROMPT.md, RESEARCH.md, etc.
- Used by: Agents when creating outputs
- Example: `get-shit-done/templates/codebase/stack.md` defines structure for technology analysis

**Reference Layer:**
- Purpose: Provide implementation guidance and decision frameworks
- Location: `get-shit-done/references/` (9 reference documents)
- Contains: Patterns, best practices, configuration schemas
- Examples:
  - `git-integration.md`: How to handle git operations safely
  - `questioning.md`: How to ask deep discovery questions
  - `planning-config.md`: Configuration schema for `.planning/config.json`
  - `model-profiles.md`: Which models to use for different agent types

**Install/Setup Layer:**
- Purpose: Configure agent system across Claude Code, OpenCode, Gemini runtimes
- Location: `bin/install.js` (main installer)
- Contains: Multi-runtime installation logic, settings management, hook configuration
- Depends on: Runtime detection, file system
- Used by: npm install (via `npx get-shit-done-cc`)
- Special: Handles frontmatter translation (Claude -> OpenCode -> Gemini tool mapping)

**Hooks Layer:**
- Purpose: Enable integrations with IDE/CLI environments
- Location: `hooks/` (2 executable hooks)
- Files:
  - `gsd-statusline.js`: Displays project status in IDE status bar
  - `gsd-check-update.js`: Checks for updates
- Used by: Runtime environments (Claude Code, OpenCode, Gemini)
- Configuration: Stored in `settings.json` for each runtime

## Data Flow

**Project Initialization Flow:**

1. User runs `/gsd:new-project`
2. Orchestrator (`commands/gsd/new-project.md`) coordinates:
   - Brownfield detection (existing code?)
   - Deep questioning (via AskUserQuestion)
   - Research phase (optional, via `/gsd:research-phase`)
   - Spawns `gsd-project-researcher` agent
3. Agent creates:
   - `.planning/PROJECT.md` (core vision, requirements, decisions)
   - `.planning/REQUIREMENTS.md` (scoped requirements)
   - `.planning/ROADMAP.md` (phase breakdown)
   - `.planning/config.json` (workflow preferences)
   - `.planning/STATE.md` (project memory)
4. State persists in git (committed)

**Phase Planning Flow:**

1. User runs `/gsd:plan-phase 1`
2. Orchestrator (`commands/gsd/plan-phase.md`):
   - Reads STATE.md (current context)
   - Loads PROJECT.md, REQUIREMENTS.md, ROADMAP.md
   - Optionally runs `/gsd:discuss-phase` (orchestrator spawns gsd-integration-checker → user decisions)
   - Optionally runs `/gsd:research-phase` if discovery needed
   - Spawns `gsd-planner` agent with full context
3. `gsd-planner` creates:
   - `.planning/phases/XX-name/01-01-PLAN.md` (executable plan)
   - `.planning/phases/XX-name/01-01-CONTEXT.md` (decisions locked during planning)
4. Plans committed to git

**Phase Execution Flow:**

1. User runs `/gsd:execute-phase 1`
2. Orchestrator (`commands/gsd/execute-phase.md`):
   - Discovers all PLAN.md files
   - Reads `wave` metadata from each plan
   - Groups into execution waves (parallel batches)
3. For each wave:
   - Spawns N parallel `gsd-executor` agents (one per plan)
   - Each agent loads full workflow context (from `~/.claude/get-shit-done/workflows/execute-plan.md`)
   - Agent executes tasks, calls git commit, writes SUMMARY.md
4. All agents complete
5. Orchestrator:
   - Verifies all SUMMARYs exist
   - Optionally runs `/gsd:verify-work` (spawns gsd-verifier)
   - Updates STATE.md with completion stats
   - Offers next steps

**Codebase Analysis Flow:**

1. User runs `/gsd:map-codebase`
2. Orchestrator spawns 4 parallel agents:
   - `gsd-codebase-mapper` (tech focus) → writes STACK.md, INTEGRATIONS.md
   - `gsd-codebase-mapper` (arch focus) → writes ARCHITECTURE.md, STRUCTURE.md
   - `gsd-codebase-mapper` (quality focus) → writes CONVENTIONS.md, TESTING.md
   - `gsd-codebase-mapper` (concerns focus) → writes CONCERNS.md
3. Each agent explores codebase independently, writes documents to `.planning/codebase/`
4. Orchestrator verifies all 7 documents exist
5. Codebase docs are consumed by `gsd-planner` during planning

**State Management:**

Central state file: `.planning/STATE.md`
- Read: First operation of every workflow
- Written to: After execution completion, phase completion, milestone completion
- Contains: Phase position, velocity metrics, accumulated decisions, blockers, session continuity
- Lifecycle: Created during init, updated after every significant action

Artifact hierarchy:
```
.planning/
├── PROJECT.md           # Core vision (immutable once locked)
├── REQUIREMENTS.md      # Scoped requirements
├── ROADMAP.md          # Phase structure with status
├── STATE.md            # Project memory (frequently updated)
├── config.json         # Workflow preferences
├── codebase/           # 7 analysis documents (refreshed via map-codebase)
├── research/           # Optional domain research
└── phases/
    ├── 01-foundation/
    │   ├── 01-01-PLAN.md
    │   ├── 01-01-CONTEXT.md
    │   └── 01-01-SUMMARY.md
    └── 02-features/
```

## Key Abstractions

**Orchestrator:**
- Purpose: Coordinate work, manage state transitions, spawn agents
- Files: `commands/gsd/*.md`
- Pattern: Thin orchestration (10-20% context) coordinates distributed agents
- Responsibility: Argument parsing, agent spawning, result aggregation, git staging
- Example: `commands/gsd/execute-phase.md` orchestrates wave-based execution

**Agent:**
- Purpose: Perform focused work with fresh 50-70% context
- Files: `agents/*.md`
- Pattern: Receive focused prompt, return focused artifact
- Responsibility: Read project context, perform work, write output, handle git operations
- Example: `agents/gsd-executor.md` executes PLAN.md and produces SUMMARY.md

**Workflow:**
- Purpose: Provide subprocess execution contexts
- Files: `get-shit-done/workflows/*.md`
- Pattern: Detailed step-by-step instructions referenced by orchestrators
- Used by: Agents that need complex procedure (e.g., executor agent reads `workflows/execute-plan.md`)
- Example: `get-shit-done/workflows/execute-plan.md` defines how to safely execute a PLAN.md

**Phase:**
- Purpose: Logical unit of work producing one or more features
- Container: Directory `phases/XX-name/`
- Contents:
  - `XX-YY-PLAN.md` (executable task breakdown)
  - `XX-YY-CONTEXT.md` (user decisions locked during planning)
  - `XX-YY-SUMMARY.md` (execution results)
- Lifecycle: Created → Planned → Discussed → Executed → Verified → Closed

**Plan:**
- Purpose: Executable specification of phase work
- File: `phases/XX-name/XX-YY-PLAN.md`
- Structure:
  - Frontmatter: wave number, gaps_only flag
  - Objective: What and why
  - Context: File references (@path/to/file)
  - Tasks: 2-3 specific work items with verification criteria
  - Success criteria: Measurable completion definition
- Used by: `gsd-executor` agent
- Principle: Plan IS the prompt (not transformed into one)

**Template:**
- Purpose: Ensure output consistency across projects
- Files: `get-shit-done/templates/**.md`
- Principle: Agents fill placeholders, never invent formats
- Subtypes:
  - Artifact templates (PROJECT.md, REQUIREMENTS.md, ROADMAP.md)
  - Phase templates (PLAN.md, SUMMARY.md, CONTEXT.md)
  - Analysis templates (STACK.md, ARCHITECTURE.md, TESTING.md)

## Entry Points

**User-facing Commands:**

`commands/gsd/*.md` — 29 user commands, each following same pattern:
1. YAML frontmatter (name, description, allowed-tools)
2. `<objective>` section
3. `<execution_context>` (@references)
4. `<context>` (loads from STATE.md, PROJECT.md, etc.)
5. `<process>` (numbered steps with bash examples)

**Installation Entry Point:**

`bin/install.js` — Node.js executable
- Triggers on: `npx get-shit-done-cc`
- Responsibility:
  - Detect available runtimes (Claude Code, OpenCode, Gemini)
  - Prompt for global vs local installation
  - Copy agent/command files to runtime config directory
  - Translate frontmatter for OpenCode (allows-tools → permissions)
  - Configure hooks for statusline and updates

**Agent Spawning:**

Agents spawned via Task tool calls from orchestrators:
```
<spawn_agent>
Agent: gsd-executor
Model: claude-opus (or per config)
Context budget: 50-70%
</spawn_agent>
```

Agents always receive:
- Focused role (what to do)
- Specific context (relevant files only)
- Clear success criteria
- Reference to workflow guides (for complex procedures)

## Error Handling

**Strategy:** Explicit error states, recovery workflows, audit trails.

**Patterns:**

1. **Validation checks:** Commands verify preconditions before spawning agents
   - Example: `gsd:execute-phase` verifies phase directory exists

2. **Graceful degradation:** If optional step fails, continue with reasonable default
   - Example: If research phase skipped, planning proceeds with existing context

3. **Recovery workflows:** Gap closure (via `/gsd:verify-work`)
   - Executor creates SUMMARY.md with failure details
   - Verifier analyzes failures, creates gap-closure plans
   - User can re-execute gaps via `/gsd:execute-phase --gaps-only`

4. **Audit trails:** All git commits preserve decision context
   - Commit messages reference phase/plan number
   - State snapshots in git history

5. **Explicit failures:** Agent processes check for errors, return structured failures
   - Example: If task fails, SUMMARY.md contains `status: failed` with details

## Cross-Cutting Concerns

**Logging:**
- Approach: File-based artifact writing (not console logging)
- What's logged: Every action produces a file (PLAN, SUMMARY, STATE, etc.)
- Result: Full audit trail in git

**Validation:**
- Location: `get-shit-done/references/` (decision frameworks, schemas)
- Patterns:
  - Frontmatter validation (wave numbers, status values)
  - Schema validation (config.json against planning-config.md schema)
  - Decision validation (user locked decisions must appear in tasks)

**Authentication/Authorization:**
- Approach: File-system based
- Mechanism: Runtime detection (which runtimes are installed)
- Managed by: `bin/install.js` (copies files to authorized runtime directories)

**Configuration Management:**
- Central config: `.planning/config.json`
- Schema: Defined in `get-shit-done/references/planning-config.md`
- Contents: model_profile, commit_docs, parallel_waves
- Loaded by: All orchestrators, agents during planning/execution

**Context Engineering:**
- Principle: Aggressive context efficiency
- Mechanism:
  - Orchestrators stay thin (10-20% budget)
  - Agents get focused context (50-70% budget)
  - Templates ensure artifacts follow patterns (reduces explanation overhead)
  - Markdown references (@file) support lazy loading in workflows

**Multi-Runtime Support:**
- Supported runtimes: Claude Code, OpenCode, Gemini CLI
- Mechanism: `bin/install.js` handles frontmatter translation
  - Claude `allowed-tools` → OpenCode `permissions` → Gemini `tools`
  - Runtime-specific paths managed
  - Co-authored-by attribution configurable per runtime

---

*Architecture analysis: 2026-02-04*
