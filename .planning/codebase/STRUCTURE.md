# Codebase Structure

**Analysis Date:** 2026-02-04

## Directory Layout

```
get-shit-done/
├── bin/
│   └── install.js              # NPM entrypoint, multi-runtime installer
├── agents/
│   ├── gsd-codebase-mapper.md  # Analyzes codebase, writes STACK/INTEGRATIONS/ARCHITECTURE/etc
│   ├── gsd-executor.md          # Executes PLAN.md, writes SUMMARY.md
│   ├── gsd-planner.md           # Creates PLAN.md from phase requirements
│   ├── gsd-debugger.md          # Debugs execution failures
│   ├── gsd-verifier.md          # Verifies plan execution quality
│   ├── gsd-project-researcher.md # Deep discovery for new projects
│   ├── gsd-phase-researcher.md  # Research before planning
│   ├── gsd-roadmapper.md        # Creates ROADMAP.md from requirements
│   ├── gsd-plan-checker.md      # Validates plans before execution
│   ├── gsd-integration-checker.md # Analyzes decisions, locks choices
│   └── gsd-research-synthesizer.md # Synthesizes research findings
├── commands/gsd/
│   ├── new-project.md           # Initialize project (.planning/ structure)
│   ├── plan-phase.md            # Plan a phase (spawn planner)
│   ├── execute-phase.md         # Execute phase plans (wave-based)
│   ├── verify-work.md           # Verify execution, create gap-closure plans
│   ├── map-codebase.md          # Analyze codebase, spawn 4 mapper agents
│   ├── discuss-phase.md         # Lock user decisions before planning
│   ├── research-phase.md        # Research before planning
│   ├── add-phase.md             # Add new phase to ROADMAP
│   ├── progress.md              # Show project status
│   ├── audit-milestone.md       # Review milestone completion
│   ├── complete-milestone.md    # Mark milestone complete
│   └── 19 other commands        # See full list in directory
├── get-shit-done/
│   ├── references/              # Implementation guidance
│   │   ├── git-integration.md       # How to safely git (commit, stage, push)
│   │   ├── questioning.md           # How to ask discovery questions
│   │   ├── planning-config.md       # Config schema for .planning/config.json
│   │   ├── model-profiles.md        # Which models for which agents
│   │   ├── ui-brand.md              # UI/output formatting standards
│   │   └── 4 other references
│   ├── workflows/               # Execution contexts for agents
│   │   ├── execute-plan.md          # How executor should run PLAN.md
│   │   ├── execute-phase.md         # How phase execution flows
│   │   ├── verify-phase.md          # How to verify phase completion
│   │   └── 9 other workflows
│   └── templates/               # Output structure templates
│       ├── codebase/
│       │   ├── stack.md             # Technology stack template
│       │   ├── integrations.md      # External integrations template
│       │   ├── architecture.md      # Architecture analysis template
│       │   ├── structure.md         # File structure template
│       │   ├── conventions.md       # Coding patterns template
│       │   ├── testing.md           # Testing patterns template
│       │   └── concerns.md          # Technical debt template
│       ├── project.md               # PROJECT.md template
│       ├── requirements.md          # REQUIREMENTS.md template
│       ├── roadmap.md               # ROADMAP.md template
│       ├── phase-prompt.md          # PLAN.md template
│       ├── state.md                 # STATE.md template
│       ├── research.md              # RESEARCH.md template
│       └── 8 other templates
├── hooks/
│   ├── gsd-statusline.js        # IDE status bar integration
│   └── gsd-check-update.js      # Update checker
├── scripts/
│   └── build-hooks.js           # Builds hooks for distribution
├── .planning/
│   └── codebase/                # Populated by /gsd:map-codebase
│       ├── STACK.md
│       ├── INTEGRATIONS.md
│       ├── ARCHITECTURE.md
│       ├── STRUCTURE.md
│       ├── CONVENTIONS.md
│       ├── TESTING.md
│       └── CONCERNS.md
├── assets/                      # Documentation assets
├── .github/                     # GitHub workflows
├── package.json                 # npm metadata (no runtime deps)
├── README.md                    # Main documentation
├── GSD-STYLE.md                 # System design philosophy
├── CHANGELOG.md                 # Version history
└── CONTRIBUTING.md              # Contributor guide
```

## Directory Purposes

**`bin/`** — Installation and setup
- Purpose: Enable installation across runtimes (Claude Code, OpenCode, Gemini)
- Key file: `bin/install.js` (52KB, comprehensive multi-runtime logic)
- Responsibility:
  - Detect target runtime
  - Copy agents/commands/templates to runtime config directory
  - Translate frontmatter (Claude → OpenCode → Gemini)
  - Configure git hooks for statusline

**`agents/`** — Specialized worker agents
- Purpose: Perform focused work with fresh context (50-70% budget)
- Contains: 11 markdown files, each defining an agent role
- Key agents:
  - `gsd-executor.md`: Executes PLAN.md, writes SUMMARY.md
  - `gsd-planner.md`: Creates phase PLAN.md files
  - `gsd-codebase-mapper.md`: Analyzes code, writes STACK/INTEGRATIONS/etc
  - `gsd-verifier.md`: Checks execution quality
  - `gsd-debugger.md`: Fixes execution failures

**`commands/gsd/`** — User-facing orchestrators
- Purpose: Parse user input, spawn agents, coordinate workflows
- Contains: 29 markdown files (one per command)
- Pattern: Each is a thin orchestrator with process steps
- Key commands:
  - `new-project.md`: Initialize project
  - `plan-phase.md`: Plan a phase
  - `execute-phase.md`: Execute phase plans
  - `map-codebase.md`: Analyze codebase
  - `verify-work.md`: Verify and fix failures

**`get-shit-done/references/`** — Implementation guidance
- Purpose: Provide decision frameworks, schemas, best practices
- Contains: 9 markdown reference documents
- Key references:
  - `git-integration.md`: Safe git operations (commit, stage, push rules)
  - `questioning.md`: Discovery question patterns
  - `planning-config.md`: Configuration schema
  - `model-profiles.md`: Model selection (opus/sonnet/haiku per agent)

**`get-shit-done/workflows/`** — Subprocess execution contexts
- Purpose: Detail complex procedures referenced by agents
- Contains: 12 markdown workflow documents
- Key workflows:
  - `execute-plan.md`: Step-by-step how to execute PLAN.md (loaded by executor)
  - `execute-phase.md`: How orchestrator groups and executes waves
  - `verify-phase.md`: Verification procedure

**`get-shit-done/templates/`** — Output structure templates
- Purpose: Ensure artifact consistency across projects
- Contains: 23 markdown templates in subdirectories
- `codebase/` subdirectory: 7 templates for codebase analysis
  - Each template defines structure, sections, example content
  - Agents fill placeholders, never invent formats
- Root templates: PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, etc.

**`hooks/`** — Runtime integrations
- Purpose: Enable IDE/CLI environment features
- Files:
  - `gsd-statusline.js`: Displays project status in IDE
  - `gsd-check-update.js`: Checks for new versions
- Used by: Runtime environments after install

**`scripts/`** — Build utilities
- Purpose: Build distribution artifacts
- File: `build-hooks.js` (compiles hooks from source)
- Triggers: `npm run build:hooks` during publish

**`.planning/codebase/`** — Codebase analysis artifacts
- Purpose: Store analysis from /gsd:map-codebase
- Contains: 7 markdown files (populated by mapper agents)
- Consumed by: `/gsd:plan-phase` (planner reads during planning)
- Updated: Via `/gsd:map-codebase` command

## Key File Locations

**Entry Points:**

- `bin/install.js` — NPM package entry point (via `npx get-shit-done-cc`)
- `package.json` — Defines bin link, no runtime dependencies
- `commands/gsd/*.md` — User commands (dispatched by runtime)

**Agent Specifications:**

- `agents/gsd-executor.md` (22KB) — Execution implementation
- `agents/gsd-planner.md` (43KB) — Planning implementation
- `agents/gsd-codebase-mapper.md` (15KB) — Analysis implementation

**Core Templates:**

- `get-shit-done/templates/project.md` — PROJECT.md structure
- `get-shit-done/templates/phase-prompt.md` — PLAN.md structure
- `get-shit-done/templates/state.md` — STATE.md structure (with lifecycle)

**Codebase Analysis:**

- `get-shit-done/templates/codebase/stack.md` — Technology template
- `get-shit-done/templates/codebase/architecture.md` — Architecture template
- `get-shit-done/templates/codebase/structure.md` — File structure template
- `get-shit-done/templates/codebase/conventions.md` — Coding patterns template
- `get-shit-done/templates/codebase/testing.md` — Testing patterns template

**Configuration:**

- `get-shit-done/references/planning-config.md` — .planning/config.json schema
- `get-shit-done/references/model-profiles.md` — Agent model assignments

## Naming Conventions

**Files:**

- **Agent files:** `gsd-[function].md` (kebab-case, descriptive verb)
  - Examples: `gsd-executor.md`, `gsd-planner.md`, `gsd-codebase-mapper.md`

- **Command files:** `[command-name].md` (kebab-case, user-facing name)
  - Examples: `new-project.md`, `execute-phase.md`, `map-codebase.md`

- **Template files:** `[artifact-name].md` (lowercase, artifact-specific)
  - Examples: `project.md`, `state.md`, `phase-prompt.md`

- **Workflow files:** `[workflow-name].md` (kebab-case, descriptive)
  - Examples: `execute-plan.md`, `verify-phase.md`, `discovery-phase.md`

- **Reference files:** `[reference-topic].md` (kebab-case, topic-specific)
  - Examples: `git-integration.md`, `questioning.md`, `model-profiles.md`

**Directories:**

- **Runtime config:** `.claude/`, `.opencode/`, `.gemini/` (dot-prefixed)
  - In each: `get-shit-done/`, `hooks/dist/`, `settings.json`

- **Project planning:** `.planning/` (dot-prefixed, hidden by default)
  - Subdirs: `phases/`, `research/`, `todos/`, `codebase/`

- **Phase directories:** `phases/[XX]-[name]/` (zero-padded number, kebab-case name)
  - Examples: `01-foundation/`, `02-features/`, `03.1-hotfix/`

- **Internal structure:** `get-shit-done/` (project root, contains all GSD content)
  - Subdirs: `agents/`, `commands/`, `templates/`, `workflows/`, `references/`

## Where to Add New Code

**New Agent:**
1. Create `agents/gsd-[function-name].md`
2. Follow structure in `agents/gsd-executor.md`:
   - YAML frontmatter (name, description, tools, color)
   - `<role>` section
   - `<context_handling>` (if needed)
   - `<process>` with numbered steps
3. Register in `package.json` files array (if applicable)

**New Command:**
1. Create `commands/gsd/[command-name].md`
2. Follow structure in `commands/gsd/execute-phase.md`:
   - YAML frontmatter
   - `<objective>`
   - `<execution_context>` (@references)
   - `<context>` (loads PROJECT.md, STATE.md, etc.)
   - `<process>` with numbered steps
3. Agent discovery handles registration automatically

**New Workflow:**
1. Create `get-shit-done/workflows/[workflow-name].md`
2. Detail step-by-step procedure
3. Reference from command/agent via `@~/.claude/get-shit-done/workflows/[name].md`

**New Template:**
1. For codebase analysis: Create `get-shit-done/templates/codebase/[analysis].md`
2. For project artifacts: Create `get-shit-done/templates/[artifact].md`
3. Fill in sections, use placeholders: `[YYYY-MM-DD]`, `[Placeholder text]`
4. Load in agent via template structure reference

**New Reference:**
1. Create `get-shit-done/references/[topic].md`
2. Provide guidance, schemas, decision frameworks
3. Reference from command/agent via `@~/.claude/get-shit-done/references/[topic].md`

## Special Directories

**`.planning/`** — Project artifact storage
- Purpose: Hold project state, plans, summaries
- Generated: Yes (created by /gsd:new-project)
- Committed: Yes (essential to project continuity)
- Structure:
  - `PROJECT.md` — Core vision (immutable once locked)
  - `ROADMAP.md` — Phase breakdown
  - `STATE.md` — Current position, velocity metrics
  - `config.json` — Workflow preferences
  - `codebase/` — Analysis documents
  - `phases/` — Phase directories with PLAN/CONTEXT/SUMMARY
  - `research/` — Optional research findings

**`.claude/`, `.opencode/`, `.gemini/`** — Runtime config
- Purpose: Store runtime-specific configuration
- Generated: Yes (by install.js)
- Committed: No (runtime-specific, ignored via .gitignore)
- Contents:
  - `get-shit-done/` (agents, commands, templates, workflows, references)
  - `hooks/dist/` (compiled hooks)
  - `settings.json` (runtime settings)

**`phases/[XX]-[name]/`** — Phase working directory
- Purpose: Contain all artifacts for one phase
- Structure:
  - `XX-YY-PLAN.md` — Executable task breakdown
  - `XX-YY-CONTEXT.md` — Locked user decisions
  - `XX-YY-SUMMARY.md` — Execution results
- Multiple plans per phase supported (YY increments)

**`codebase/`** — Codebase analysis cache
- Purpose: Store technology, architecture, quality analysis
- Contents: 7 markdown files (STACK, INTEGRATIONS, ARCHITECTURE, etc.)
- Refresh: Via /gsd:map-codebase
- Consumed by: gsd-planner during plan creation

**`research/`** — Domain research
- Purpose: Store research findings before planning
- Contents: RESEARCH.md (from /gsd:research-phase)
- Optional: Only created if research phase runs

**`todos/`** — Captured ideas
- Purpose: Store ideas captured via /gsd:add-todo
- Subdirs: `pending/`, `completed/`
- Used by: STATE.md references pending todos

---

*Structure analysis: 2026-02-04*
