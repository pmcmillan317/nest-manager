# Subagent reference

Condensed from the Claude Code "Create custom subagents" docs. Behavior changes between releases, so check the official page when something here disagrees with what you see.

## Contents

- What a subagent is
- Built-in subagents
- Scope and priority
- File format
- CLI-defined subagents
- Frontmatter fields
- Reload behavior
- Plugin limits

## What a subagent is

A specialized assistant with its own context window, system prompt, tool access, and permissions. It does the noisy work (search results, logs, file contents) and returns only a summary. Its requests count toward the same usage limits as the main conversation.

Claude decides when to delegate by matching the task against each subagent's `description`, so write that field to say when to use it.

Subagents receive only their own system prompt plus basic environment details such as the working directory. They do not get the Claude Code system prompt. `CLAUDE.md` files and the git status snapshot are loaded, unless the agent sets `omitClaudeMd`.

A subagent starts in the main conversation's working directory. `cd` inside it does not persist between Bash calls and does not affect the main conversation.

## Built-in subagents

- **Explore**: fast, read-only (Write and Edit denied), for file discovery and code search. Starts without `CLAUDE.md` and git status to stay cheap. Claude passes a thoroughness level: quick, medium, or very thorough.
- **Plan**: read-only research for planning. Also starts without `CLAUDE.md`.
- **General-purpose**: handles multi-step tasks that need exploration plus action.

A user or project agent named `Explore` overrides the built-in and keeps its own `model` field.

To restrict them: add the agent type to `permissions.deny`, deny the `Agent` tool entirely to stop all delegation, or set `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS=1` to drop only Explore and Plan.

## Scope and priority

When names collide, the higher row wins.

| Location | Scope | Priority |
|---|---|---|
| Managed settings | Organization-wide | 1 (highest) |
| `--agents` CLI flag | Current session only | 2 |
| `.claude/agents/` | Current project (commit it for the team) | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin `agents/` directory | Where the plugin is enabled | 5 (lowest) |

Project agents are found by walking up from the working directory to the repo root, and the definition closest to the working directory wins. Both agents directories are scanned recursively, so subfolders are fine, and identity comes only from the `name` field.

## File format

Markdown with YAML frontmatter. The body is the system prompt.

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. Give specific, actionable feedback.
```

Only `name` and `description` are required. Omit `tools` to inherit everything, or list only what the agent needs.

## CLI-defined subagents

For one-off sessions or scripts. Nothing is saved to disk.

```bash
claude --agents '{
  "reviewer": {
    "description": "Reviews diffs. Use after code changes.",
    "prompt": "You are a senior reviewer...",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "haiku"
  }
}'
```

Each top-level key is the agent name (do not start it with `-`). In non-interactive mode, `--agents` also accepts a path to a JSON file with the same object. `color` and `experimental` are ignored there.

## Frontmatter fields

Documented names: `name`, `description`, `tools`, `disallowedTools`, `model`, `permissionMode`, `mcpServers`, `hooks`, `maxTurns`, `autoCompactWindow`, `skills`, `initialPrompt`, `memory`, `effort`, `background`, `omitClaudeMd`, `isolation`.

The ones this skill relies on:

- `tools`: allowlist. Read-only agents get `Read, Grep, Glob`.
- `model`: an alias (`haiku`, `sonnet`, `opus`) or a full model ID.
- `isolation: worktree`: runs the agent in its own git worktree. Its Bash commands are checked so they cannot reach back into the main checkout.
- `omitClaudeMd`: start without the user, project, and local `CLAUDE.md` files, for a smaller context.

For what the others do, read the official frontmatter reference.

## Reload behavior

Edits to files in `~/.claude/agents/` and `.claude/agents/` are picked up within a few seconds. Restart is needed when:

- The agents directory did not exist at session start.
- The agent lives in a directory added with `--add-dir` or `/add-dir`.
- The session was started with `--disable-slash-commands`.

## Plugin limits

Plugin agents ignore `hooks`, `mcpServers`, and `permissionMode` for security. To use them, copy the agent file into `.claude/agents/` or `~/.claude/agents/`. Plugin agents in subfolders get a scoped name, so `agents/review/security.md` in plugin `my-plugin` registers as `my-plugin:review:security`.

## Token budget

If the combined descriptions of non-built-in subagents exceed 15,000 tokens, Claude Code warns at startup. Trim descriptions and move detail into the system prompt, which loads only when the agent runs.
