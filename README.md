# kmac-claude-kit

A personal toolkit for building software with [Claude Code](https://docs.anthropic.com/en/docs/claude-code). This repo describes how the pieces fit together.

## Components

| Repo | Purpose |
|---|---|
| [claude-sandbox](https://github.com/kmacmcfarlane/claude-sandbox) | Sandboxed Docker container for running Claude Code sessions safely. Mounts the project directory and host Docker socket, provides Go, Node.js, Python, and build tools, and supports interactive and ralph (autonomous loop) modes. |
| [claude-templates](https://github.com/kmacmcfarlane/claude-templates) | Project templates for quick-starting new repos. Each template includes a full agent workflow (orchestrator, subagent definitions, backlog CLI, worktree tooling, practices docs), Docker Compose, and MCP servers for Discord and gopls. |
| [claude-skills](https://github.com/kmacmcfarlane/claude-skills) | Reusable Claude Code skills (slash commands) for backlog management, project scaffolding, and upstream sync. |

## How they relate

```
kmac-claude-kit (you are here)
│
├── claude-sandbox        Start a sandboxed Claude Code session
│     │
│     └── mounts your project repo
│           │
│           ├── .claude/
│           │     ├── agents/          Subagent definitions (5 agents)
│           │     ├── skills/          ← copy from claude-skills
│           │     └── settings.json    Permission policy
│           │
│           └── (project scaffolded from claude-templates)
│                 ├── CLAUDE.md         Always-loaded agent context
│                 ├── .mcp.json         MCP servers (Discord, gopls)
│                 ├── agent/
│                 │     ├── AGENT_FLOW.md        Orchestrator contract
│                 │     ├── PROMPT.md            Orchestrator prompt
│                 │     ├── PROMPT_AUTO.md       Autonomous mode policy
│                 │     ├── PROMPT_INTERACTIVE.md Interactive mode policy
│                 │     ├── TEST_PRACTICES.md    Testing standards
│                 │     ├── DEVELOPMENT_PRACTICES.md Engineering standards
│                 │     ├── LSP_TOOLS.md         gopls/LSP tool reference
│                 │     ├── BUG_REPORTING.md     Bug report quality guide
│                 │     ├── PRD.md               Product requirements (per-project)
│                 │     ├── backlog.yaml          Story tracker
│                 │     ├── backlog_done.yaml     Completed stories archive
│                 │     └── ideas/               Agent-suggested improvements
│                 ├── scripts/
│                 │     ├── backlog/    backlog.py CLI (CRUD, work selection)
│                 │     ├── worktree/   worktree.py + merge_helper.py
│                 │     └── compose-project-name.sh
│                 ├── docs/            Architecture, database, API docs
│                 └── ...
│
├── claude-templates      Template source (copy into new repos)
│     └── local-web-app/  Go + Vue 3 + Docker Compose
│
└── claude-skills         Skill source (copy or symlink into projects)
      └── skills/
            ├── backlog-yaml/      Backlog CLI reference skill
            ├── backlog-entry/     Interactive ticket creation
            ├── backlog-grooming/  UAT review and grooming sessions
            ├── update-kit/        Sync changes back to upstream repos
            ├── create-skill/      Bootstrap new skills
            ├── goa/               Goa v3 API framework
            ├── playwright/        E2E test authoring
            └── musubi-tuner/      LoRA training with kohya
```

## Agent Pipeline

Stories progress through a multi-agent pipeline orchestrated by `PROMPT.md`:

```
todo → in_progress → review → testing → uat → done
         │              │         │        │
   fullstack-dev   code-reviewer  QA    user reviews
```

| Agent | Role | Invoked when |
|---|---|---|
| **Fullstack Developer** | Implements features/bugs, writes unit + E2E tests | `todo` or `in_progress` |
| **Code Reviewer** | Reviews code quality, security, architecture | `review` |
| **QA Expert** | Runs E2E suite, verifies acceptance criteria, runtime error sweep | `testing` |
| **Debugger** | Diagnoses hard bugs, test failures | On demand |
| **Security Auditor** | Security assessments | On demand |

The orchestrator owns all status transitions, CHANGELOG updates, commits, and merges. Subagents report structured verdicts only.

After QA approval, stories enter `uat` (user acceptance testing) with code merged to main. The user reviews and either approves (`done`) or provides feedback (`uat_feedback`) for a rework cycle.

### Model tiering

The orchestrator selects models per agent based on story complexity:
- **low** complexity → `sonnet` (fast, cost-effective)
- **medium/high** complexity → `opus` (deeper analysis)

## Tooling

### Backlog CLI (`scripts/backlog/backlog.py`)

All backlog operations go through this CLI — never edit YAML directly. It provides round-trip YAML preservation, schema validation, atomic writes, and file locking for concurrent access.

```bash
backlog.py next-work --format json              # Deterministic work selection
backlog.py query --status todo --fields id,title # Filter stories
backlog.py set S-052 status review              # Update status
backlog.py next-id B                            # Get next bug ID
cat ticket.yaml | backlog.py add                # Add new stories
backlog.py validate --strict                    # Schema validation
```

### Worktree manager (`scripts/worktree/worktree.py`)

Enables parallel agent execution via per-story git worktrees with Docker compose isolation:

```bash
worktree.py create S-042                  # Create isolated worktree
worktree.py detect-stale                  # Find orphaned worktrees
STORY_ID=S-042 make test-backend          # Story-scoped compose stack
```

### Merge helper (`scripts/worktree/merge_helper.py`)

Auto-resolves trivial merge conflicts (CHANGELOG, backlog.yaml) when concurrent stories merge to main. Non-trivial conflicts go through the normal rework flow.

## Workflow

### Starting a new project

1. Copy the `local-web-app` template from [claude-templates](https://github.com/kmacmcfarlane/claude-templates) into a new repo.
2. Replace placeholder values (project name in `backlog.yaml`, compose project names).
3. Copy skills from [claude-skills](https://github.com/kmacmcfarlane/claude-skills) into `.claude/skills/`.
4. Write your PRD in `agent/PRD.md` and add stories to `agent/backlog.yaml` (via `backlog.py add`).
5. Run `make claude` to start a sandboxed session, or `make ralph` for autonomous loops.

### Day-to-day development

- **Interactive**: `make claude` launches a [claude-sandbox](https://github.com/kmacmcfarlane/claude-sandbox) container with Claude Code.
- **Autonomous**: `make ralph` or `make ralph-auto` runs fresh-context loops that pick up stories from the backlog automatically. Each cycle processes exactly one story through the full pipeline.
- **Parallel**: Multiple agents can work concurrently using worktrees — each gets an isolated git checkout and Docker compose stack.
- **Notifications**: Discord MCP server pings you when the agent starts a story, needs input, finishes, or gets blocked.

### Backlog grooming

Use the `/backlog-grooming` skill for conversational UAT review sessions:
- Triage `uat` stories (approve, provide feedback, or skip)
- File new bugs and feature requests
- Adjust priorities and dependencies
- All mutations are batched and confirmed before execution

### Creating new skills

Use the `/create-skill` skill from [claude-skills](https://github.com/kmacmcfarlane/claude-skills):

```
/create-skill A skill that runs the test suite and summarizes failures
```

### Syncing changes upstream

When a project evolves its workflow files beyond the template, use `/update-kit` to sync improvements back:

```
/update-kit all
```

This dynamically scans for differences between the project and upstream repos, classifies changes as generic or project-specific, builds a task-based plan, and syncs with genericization verification.

## Skills Reference

| Skill | Description |
|---|---|
| `/backlog-yaml` | Backlog CLI reference — auto-activates when working with stories |
| `/backlog-entry` | Interactive ticket creation with validation and batch support |
| `/backlog-grooming` | UAT review, bug reporting, priority management sessions |
| `/update-kit` | Sync workflow files and skills back to upstream repos |
| `/create-skill` | Bootstrap new skills with best-practices template |
| `/goa` | Goa v3 API framework design and code generation |
| `/playwright` | Playwright E2E test authoring and configuration |
| `/musubi-tuner` | LoRA training with kohya's musubi-tuner |

## License

This project is licensed under the [GPL-3.0](LICENSE).
