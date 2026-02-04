# Feature Research

**Domain:** Multi-Project/Workspace Management for CLI Tools
**Researched:** 2026-02-04
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Project creation | Every multi-project tool allows creating projects | LOW | `/gsd:new-project [name]` — already planned |
| Project switching | Core purpose of multi-project support | LOW | `/gsd:switch-project <name>` — already planned. Essential for workflow. |
| List all projects | Users need to see what projects exist | LOW | `/gsd:list-projects` — already planned. Should show status/context. |
| Isolated project state | Each project must have independent context | MEDIUM | `.planning/projects/<name>/` structure — already planned |
| Active project indicator | User must know which project is active | LOW | `.planning/.active` file + CLI indicator — already planned |
| Project deletion/archiving | Cleanup old/abandoned projects | LOW | `/gsd:archive-project [name]` — already planned |
| Default/fallback project | Handle single-project workflows gracefully | LOW | Migration to `projects/default/` — already planned |
| Git integration | Respect git context (branch, worktree) | MEDIUM | Default project name from branch — already planned |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Auto-detect git branch as project | Zero config: `git checkout feature-x` → project context switches | MEDIUM | Natural mapping. Git worktree support. Reduces cognitive overhead. |
| Cross-project visibility | See all planning docs without switching | LOW | Shared `.planning/` root allows browsing all projects. Git-friendly. |
| Project templates | Quick-start common project types | MEDIUM | Copy template structure to new project. Defer to v1.x. |
| Project status tracking | Rich metadata (created, last active, phase progress) | MEDIUM | Enhance `/gsd:list-projects` with analytics. Could show phase completion %, last modified, git status. |
| Fuzzy project search | Filter projects interactively | LOW | Integration with fzf/similar for `/gsd:switch-project`. Better UX than typing full names. |
| Project tagging/grouping | Organize many projects (team, epic, client) | MEDIUM | Metadata in project config. Useful for teams with 10+ concurrent projects. |
| Shared codebase map | One map for all projects in repo | LOW | `.planning/codebase/` at root level — already planned. Reduces duplication. |
| Per-project config overrides | Customize model profiles, depth per project | LOW | `projects/<name>/config.json` overrides root — already planned |
| Project initialization wizard | Guided setup vs bare-bones creation | MEDIUM | Enhanced `/gsd:new-project` flow. Ask about template, team, lifecycle. |
| Project hand-off docs | Auto-generate project README for context sharing | MEDIUM | Generate from PROJECT.md + STATE.md. Help teams share project ownership. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Real-time collaboration | "Multiple people working at once" | GSD is async via git by design. Conflicts handled by git merge. Adding real-time sync = complexity explosion. | Use git branching. Each person works in separate project, merge via PR. |
| Project permissions/locking | "Prevent others from editing my project" | Git already handles this. Adding lock files = merge conflicts. Race conditions. | Git branch protection, PR workflow, `.gitignore` for personal projects. |
| Cross-project dependencies | "Project A needs Project B done first" | Creates tight coupling. Breaks project isolation. Dependency resolution complexity. | Separate projects are independent. If dependent, should be one roadmap. |
| Global task queue | "See all tasks across all projects" | Context switching killer. Mixing concerns defeats project isolation. | Stay focused on one project. Use `/gsd:progress` within active project. |
| Project sync to remote service | "Backup to cloud" | Git is already the remote. Adding another service = vendor lock-in. | Commit `.planning/` to git (already default). Push to GitHub/GitLab. |
| Automatic project switching | "Auto-switch based on directory" | Fragile heuristics. Surprising behavior. What if nested projects? | Explicit `/gsd:switch-project`. Optional: direnv integration for power users. |
| Nested projects | "Sub-projects within projects" | Exponential complexity. Routing commands becomes ambiguous. | Use phases/milestones within one project. Projects are top-level only. |
| Project merge/split | "Combine or divide projects" | State reconciliation is hard. Phases/requirements don't align cleanly. | Create new project with clear scope. Archive old ones. Git preserves history. |

## Feature Dependencies

```
[Project Creation]
    └──requires──> [Project Switching]
                       └──requires──> [Active Project Tracking]

[List Projects] ──enhances──> [Project Status Tracking]

[Auto-detect Git Branch] ──enhances──> [Project Creation]
                         ──enhances──> [Project Switching]

[Fuzzy Search] ──enhances──> [Project Switching]

[Project Templates] ──requires──> [Project Creation]

[Per-Project Config] ──requires──> [Isolated Project State]

[Cross-Project Dependencies] ──conflicts──> [Project Isolation]

[Automatic Switching] ──conflicts──> [Explicit Context Management]

[Nested Projects] ──conflicts──> [Clean Command Routing]
```

### Dependency Notes

- **Project Switching requires Active Project Tracking:** Can't switch if you don't know what's active. The `.active` file is essential for switching.
- **List Projects enhances Project Status Tracking:** Listing becomes more valuable with metadata (last modified, phase progress, git status).
- **Auto-detect Git Branch enhances Creation & Switching:** Natural workflow: `git checkout new-branch` → `/gsd:new-project` defaults to "new-branch" name.
- **Fuzzy Search enhances Project Switching:** Fast selection when 5+ projects exist. Not essential for MVP.
- **Project Templates require Project Creation:** Templates are just pre-filled structure for new projects.
- **Per-Project Config requires Isolated State:** Overrides only make sense if projects are truly isolated.
- **Cross-Project Dependencies conflict with Isolation:** If projects depend on each other, they should be phases in one project.
- **Automatic Switching conflicts with Explicit Context:** GSD's design philosophy is explicit, predictable behavior.
- **Nested Projects conflict with Command Routing:** Which project receives the command? Adds ambiguity.

## MVP Definition

### Launch With (v1)

Minimum viable product — what's needed to validate the concept.

- [x] Project creation with optional name (`/gsd:new-project [name]`) — defaults to git branch
- [x] Project switching (`/gsd:switch-project <name>`) — explicit context change
- [x] List all projects (`/gsd:list-projects`) — show name + basic status
- [x] Archive project (`/gsd:archive-project [name]`) — delete project folder
- [x] Active project tracking (`.planning/.active` file) — persistent state
- [x] Isolated project directories (`projects/<name>/`) — full separation
- [x] Shared codebase map (`.planning/codebase/`) — avoid duplication
- [x] Auto-migration (flat → `projects/default/`) — backwards compatibility
- [x] Per-project config overrides — model profiles, depth settings
- [x] Prompt on no active project — better UX than error

### Add After Validation (v1.x)

Features to add once core is working.

- [ ] Project status tracking — when projects are frequently switched, show "Last active 2 days ago", "Phase 3/8 complete"
- [ ] Fuzzy project search — when users have 5+ projects, interactive filter becomes valuable
- [ ] Git worktree integration — detect worktrees, offer project per worktree
- [ ] Project templates — when patterns emerge (API project, frontend project), extract templates
- [ ] Project tagging — when teams grow, organize by team/epic/client
- [ ] Enhanced `/gsd:list-projects` — show git status (ahead/behind), uncommitted changes

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] Project hand-off docs — when teams rotate ownership, auto-generate onboarding docs
- [ ] Project initialization wizard — when project setup patterns stabilize, guide users through choices
- [ ] Multi-repo project support — when users want one GSD project spanning multiple repos (monorepo adjacent)
- [ ] Project analytics dashboard — aggregated metrics across all projects (phases completed, tokens used, commits made)
- [ ] direnv integration — auto-load project context based on directory (power user feature)

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Project creation | HIGH | LOW | P1 |
| Project switching | HIGH | LOW | P1 |
| List projects | HIGH | LOW | P1 |
| Active project tracking | HIGH | LOW | P1 |
| Isolated directories | HIGH | MEDIUM | P1 |
| Archive project | MEDIUM | LOW | P1 |
| Shared codebase map | MEDIUM | LOW | P1 |
| Auto-migration | HIGH | MEDIUM | P1 |
| Per-project config | MEDIUM | LOW | P1 |
| Prompt on no active | MEDIUM | LOW | P1 |
| Git branch detection | HIGH | MEDIUM | P1 |
| Project status tracking | MEDIUM | MEDIUM | P2 |
| Fuzzy project search | MEDIUM | LOW | P2 |
| Git worktree support | LOW | HIGH | P2 |
| Project templates | MEDIUM | MEDIUM | P2 |
| Project tagging | LOW | MEDIUM | P3 |
| Hand-off docs | LOW | MEDIUM | P3 |
| Initialization wizard | LOW | MEDIUM | P3 |
| Multi-repo support | LOW | HIGH | P3 |
| Analytics dashboard | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for launch (already in PROJECT.md scope)
- P2: Should have, add when possible (post-launch improvements)
- P3: Nice to have, future consideration (deferred until demand)

## Competitor Feature Analysis

| Feature | Git Worktrees | Tmuxinator | pnpm Workspaces | GitKraken CLI | GSD Approach |
|---------|---------------|------------|-----------------|---------------|--------------|
| Project isolation | Separate working trees per branch | Separate tmux sessions per project | Separate packages per workspace | Cloud-backed workspaces | Separate directories per project in `.planning/projects/` |
| Context switching | `git worktree add` + cd | `tmuxinator start <project>` | `pnpm --filter <workspace>` | `gk work start <name>` | `/gsd:switch-project <name>` |
| List projects | `git worktree list` | `tmuxinator list` | `pnpm list -r` | `gk work list` | `/gsd:list-projects` with status |
| Auto-detect context | Based on current directory | YAML config with project_root | `pnpm-workspace.yaml` | Syncs with GitHub issues | Git branch name → project name |
| Shared resources | Shared .git directory | Shared tmux server | Shared node_modules via symlinks | Shared cloud state | Shared codebase map |
| Persistence | Git history | YAML files in ~/.tmuxinator | pnpm-workspace.yaml | Cloud storage | `.planning/.active` + project folders in git |
| Project creation | `git worktree add -b <branch>` | `tmuxinator new <project>` | Add to pnpm-workspace.yaml | `gk work start <name>` | `/gsd:new-project [name]` |
| Cleanup | `git worktree remove` | `tmuxinator delete <project>` | Remove from config | Complete work item | `/gsd:archive-project [name]` |
| Templates | Branch templates | tmuxinator project templates | Package.json templates | Work item templates | Deferred to v1.x |
| Fuzzy search | No (manual cd) | tmuxinator via shell completion | No | Fuzzy finder for repos | Deferred to v1.x |

### Key Insights

**Git Worktrees** excels at parallel branch work but requires manual directory navigation. GSD borrows isolation concept but abstracts away filesystem complexity.

**Tmuxinator** provides excellent session management with preview and templates. GSD borrows explicit switching (`start <project>`) vs automatic context detection.

**pnpm Workspaces** solves dependency sharing via symlinks. GSD borrows shared resources concept (codebase map) but projects are fully isolated otherwise.

**GitKraken CLI** integrates work items with git workflow. GSD borrows status tracking and cloud-sync concept but keeps it git-only (no vendor lock-in).

**GSD's Differentiation:**
- Planning artifacts (PROJECT.md, ROADMAP.md) are first-class, not just code
- Multi-agent orchestration state is per-project
- Git integration without requiring worktrees (simpler mental model)
- Works across Claude Code, OpenCode, Gemini CLI (not IDE-specific)

## Sources

### Multi-Project Workspace Features
- [feat: Introduce /workspace command for multi-project context switching · Issue #4935 · google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli/issues/4935)
- [Multi Root Workspaces in Visual Studio Code - ISE Developer Blog](https://devblogs.microsoft.com/ise/multi_root_workspaces_in_visual_studio_code/)
- [Top 5 CLI Coding Agents in 2026 - DEV Community](https://dev.to/lightningdev123/top-5-cli-coding-agents-in-2026-3pia)
- [Workspaces in IntelliJ IDEA | The IntelliJ IDEA Blog](https://blog.jetbrains.com/idea/2024/08/workspaces-in-intellij-idea/)
- [Multi-Root Workspaces - Cline](https://docs.cline.bot/features/multiroot-workspace)

### Git Workspace Management
- [Git Workspaces: Multi-Repo Management Made Easy | GitKraken](https://www.gitkraken.com/features/workspaces)
- [Git Worktrees: Git Done Right - DEV Community](https://dev.to/nickytonline/git-worktrees-git-done-right-2p7f)
- [Git - git-worktree Documentation](https://git-scm.com/docs/git-worktree)
- [GitHub - d-kuro/gwq: Git worktree manager with fuzzy finder](https://github.com/d-kuro/gwq)
- [GitKraken CLI: The Ultimate CLI for Git Collaboration](https://www.gitkraken.com/cli)

### Monorepo Workspace Tools
- [Workspace | pnpm](https://pnpm.io/workspaces)
- [Guide to Monorepo Setup: NPM, Yarn, Pnpm & Bun Workspaces](https://jsdev.space/mastering-monorepos/)
- [Mastering pnpm Workspaces: A Complete Guide to Monorepo Management - Glen Thomas](https://blog.glen-thomas.com/software%20engineering/2025/10/02/mastering-pnpm-workspaces-complete-guide-to-monorepo-management.html)
- [How we configured pnpm and Turborepo for our monorepo | Nhost](https://nhost.io/blog/how-we-configured-pnpm-and-turborepo-for-our-monorepo)

### Session Management
- [Managing Development Environments with Tmux and Tmuxinator - Jess Archer](https://jessarcher.com/articles/managing-development-environments-with-tmux-and-tmuxinator/)
- [GitHub - omerxx/tmux-sessionx: A Tmux session manager, with preview, fuzzy finding, and MORE](https://github.com/omerxx/tmux-sessionx)
- [GitHub - tmuxinator/tmuxinator: Manage complex tmux sessions easily](https://github.com/tmuxinator/tmuxinator)
- [Managing multiple projects with tmuxinator – Martin Wood](https://martinwood.org/managing-multiple-projects-with-tmuxinator)

### Build Systems & Task Runners
- [Task-Based Build Systems | Bazel](https://bazel.build/basics/task-based-builds)
- [Comparing Monorepo Tools: Nx, Lerna, Bazel, and Turborepo - Mindful Chase](https://www.mindfulchase.com/deep-dives/monorepo-fundamentals-deep-dives-into-unified-codebases/comparing-monorepo-tools-nx,-lerna,-bazel,-and-turborepo.html)
- [Build Systems: Bazel vs Make · Nalys Technical Blog](https://nalys-taas-projects.gitlab.io/internal/taas_blog/post/bazel_vs_make/)

### Context Switching Best Practices
- [Mitigating Context Switching in Software Development](https://jellyfish.co/library/developer-productivity/context-switching/)
- [Reducing context switching in development workflows](https://graphite.com/guides/reducing-context-switching-development-workflows)
- [Context Switching is Killing Your Productivity | DevOps Culture](https://www.software.com/devops-guides/context-switching)
- [Context Switching: The Silent Killer of Developer Productivity - Hatica](https://www.hatica.io/blog/context-switching-killing-developer-productivity/)
- [Context Switching: Why It Kills Productivity & How to Fix (2026 Guide) | Reclaim](https://reclaim.ai/blog/context-switching)

### Project Management CLI Tools
- [GitHub - EivindArvesen/prm: A minimal project manager for the terminal](https://github.com/EivindArvesen/prm)
- [GitHub - claudiodangelis/banco: A project management tool for the command line](https://github.com/claudiodangelis/banco)
- [Taskwarrior](https://taskwarrior.org/)
- [Ultralist: Amazing task management for tech folks](https://ultralist.io/)

### Environment Management
- [direnv – unclutter your .profile | direnv](https://direnv.net/)
- [A guide to manage your environment variables in a better way using direnv | by Shivam Arora | Medium](https://shivamarora.medium.com/a-guide-to-manage-your-environment-variables-in-a-better-way-using-direnv-2c1cd475c8e)
- [Never worry about environment variables again with direnv - DEV Community](https://dev.to/turck/never-worry-about-environment-variables-again-with-direnv-1h17)
- [My Favorite Tool for Managing Project-Specific Environment Variables | Sean C Davis](https://www.seancdavis.com/posts/favorite-tool-managing-project-specific-environment-variables/)

---
*Feature research for: Multi-Project Support in GSD CLI*
*Researched: 2026-02-04*
