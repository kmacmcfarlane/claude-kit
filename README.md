# claude-kit

A personal toolkit for building software with [Claude Code](https://docs.anthropic.com/en/docs/claude-code). This repo is the map: it describes how the pieces fit together and what each one owns. Setup and usage instructions live in the component repos linked below.

> This repo was previously named `kmac-claude-kit`. GitHub redirects the old URL, but `claude-kit` is the current name.

## Components

| Repo | Owns |
|---|---|
| [claude-sandbox](https://github.com/kmacmcfarlane/claude-sandbox) | Running Claude Code in a Docker container with filesystem isolation and opt-in host access (Docker socket, AWS, git, SSH). Provides interactive and ralph (autonomous loop) modes, and the `init` / `init-ralph` commands that bootstrap a project. **Start here** — installation and CLI reference are in its README. |
| [claude-templates](https://github.com/kmacmcfarlane/claude-templates) | Project scaffolding for new repos — application skeleton, Docker Compose, subagent definitions, and the project-specific workflow and practice docs. |
| [claude-plugins](https://github.com/kmacmcfarlane/claude-plugins) | The plugin marketplace and the **source of truth for skills** — backlog management, project scaffolding, upstream sync, and framework-specific helpers. Its README has the marketplace and install commands. |

> **Note:** A legacy `claude-skills` repo exists but is **deprecated** in favor of the claude-plugins marketplace. Do not sync to it.

## How they fit together

```
claude-sandbox          Runs Claude Code in a container, mounts your project
      │
      ├── init / init-ralph  ──►  seeds .claude-sandbox/ into the project
      │
      ▼
your project repo
      │
      ├── .claude-sandbox/     Sandbox config + agent workflow + tooling
      │     ├── config.yaml          Host access, model, image settings
      │     ├── env                  Secrets (Discord webhook, etc.)
      │     ├── agent/               Workflow docs, prompts, PRD, backlog
      │     └── scripts/             backlog + worktree tooling
      │
      ├── .claude/             Subagent definitions, skills, permission policy
      │     └── skills/              ◄── installed from claude-plugins
      │
      └── (application code)   ◄── scaffolded from claude-templates
```

Three rules make this composable:

- **`init-ralph` is idempotent.** It never overwrites an existing `config.yaml`, `env`, agent doc, or script — re-running fills only what's missing and reports what it skipped.
- **Templates win over defaults.** A template lays down its own project-specific workflow and practice docs first; a later `init-ralph` keeps those and adds only the generic pieces the template didn't provide.
- **Skills flow one way.** claude-plugins is upstream of every project. Skills are installed from the marketplace, never hand-copied between projects, and improvements go back up via `/update-kit`.

## Agent pipeline

Stories progress through a multi-agent pipeline driven by the orchestrator prompt in `.claude-sandbox/agent/`:

```
todo → in_progress → review → testing → uat → done
         │              │         │        │
    implement      code review    QA    user reviews
```

Specialized subagents (implementation, code review, QA, architecture review, debugging, security audit) handle each stage. The current roster lives in the template's `.claude/agents/` and in the claude-kit plugin.

Subagents *decide* transitions; the orchestrator *writes* them. A reviewer's verdict determines whether a story advances or bounces back, but no subagent touches the backlog, CHANGELOG, commits, or merges — those are exclusively the orchestrator's. That split is what keeps the state machine consistent when agents run in parallel.

After QA approval, stories enter `uat` with code already merged to main. The user reviews and either approves (`done`) or provides feedback (`uat_feedback`) for a rework cycle.

**Model tiering:** the orchestrator picks a model per agent from story complexity — lighter models for low-complexity work, stronger ones where deeper analysis pays off.

## Tooling

Seeded into `.claude-sandbox/scripts/` by `claude-sandbox init-ralph`:

- **Backlog CLI** — all backlog operations go through it; the YAML is never edited directly. Round-trip YAML preservation, schema validation, atomic writes, and file locking for concurrent access. Also drives deterministic work selection, so an autonomous loop always picks the same next story.
- **Worktree manager** — per-story git worktrees with isolated Docker Compose stacks, so multiple agents can work in parallel without colliding.
- **Merge helper** — auto-resolves the mechanical merge conflicts (CHANGELOG, backlog) that concurrent stories inevitably produce. Anything non-trivial goes through the normal rework flow.

## Workflow

### Starting a new project

Use the `/new-project-from-template` skill, or do it by hand: copy a template from claude-templates, `git init`, then run `claude-sandbox init-ralph` to seed the sandbox config and agent scaffolding. Fill in the PRD, groom the backlog, and start a session.

### Day-to-day development

- **Interactive** — a sandboxed Claude Code session against the project.
- **Autonomous** — ralph mode runs fresh-context loops that pull stories off the backlog on their own. Each cycle takes exactly one story through the full pipeline.
- **Parallel** — several agents at once via worktrees, each with its own checkout and Compose stack.
- **Notifications** — a Discord MCP server pings you when the agent starts a story, needs input, finishes, or gets blocked.

### Backlog grooming

`/backlog-grooming` runs a conversational UAT review: triage stories awaiting acceptance, file bugs and feature requests, adjust priorities and dependencies. Mutations are batched and confirmed before anything is written.

### Syncing changes upstream

When a project's workflow files drift ahead of the template, `/update-kit` syncs the improvements back. It scans for differences, classifies each change as generic or project-specific, and syncs the generic ones with genericization verification.

### Skills

The full skill catalog, along with marketplace and install instructions, lives in the [claude-plugins](https://github.com/kmacmcfarlane/claude-plugins) README. New skills are bootstrapped with `/create-skill` and belong in that repo.

## License

This project is licensed under the [GPL-3.0](LICENSE).
