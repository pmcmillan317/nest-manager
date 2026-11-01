---
name: model-routing
description: Cost-aware model routing for coding sessions - Sonnet builds, Haiku subagents search/grep/audit in parallel, Opus advises at plan, repeated-error, and sign-off checkpoints. Use whenever a coding task involves searching or exploring a repo, finding call sites, running tests or diff audits, deciding when to bring in a bigger model, creating or editing .claude/agents subagents, or cutting token use and context bloat - even if the user never mentions models or subagents.
---

# Model routing

Sonnet builds. Haiku swarms. Opus reviews.

The main session's context window is the scarce resource. Every grep dump, test log, and file listing that lands in it stays there and gets re-read on every later turn. Subagents run in their own context and hand back only a summary, so push noisy work to them, and put the cheapest model that can do each job on it.

## The split

| Role | Model | Does | Does not |
|---|---|---|---|
| Main session | Sonnet | Coordinates, writes code, applies fixes | Sweep the repo itself |
| `search` | Haiku | Maps the repo: which files matter, how they connect (read-only) | Edit anything |
| `grep` | Haiku | Finds call sites and usages in its own context (read-only) | Edit anything |
| `audit` | Haiku | Checks the diff and runs tests in a throwaway worktree | Touch the working branch |
| `advisor` | Opus | Reviews the plan, the stuck error, and the final diff | Write the code |

Templates for all four live in `assets/agents/`.

## When Opus steps in

Opus is the expensive one, so use it where a wrong call costs the most. Exactly three moments:

1. **Before a plan locks.** Multi-file changes, new dependencies, anything that changes an interface or data shape. Fixing a bad plan after the code is written costs far more than a review before.
2. **When the same error shows up twice.** A second identical failure means the current approach is probably wrong, and more attempts on it burn tokens.
3. **Before calling it done.** Hand it the synthesized diff, not the transcript.

Send the advisor a tight brief: the goal, the plan or diff, what was already tried, and the one question to answer. Ask for a verdict plus concrete blockers. A whole transcript buries the question and costs Opus context for nothing.

### How to get an Opus advisor

- The source post launches with `claude --model sonnet --advisor opus`. That `--advisor` flag does not appear in `claude --help` on Claude Code v2.1.316, so run `claude --help | grep -i advisor` before relying on it.
- Reliable route that works today: launch with `claude --model sonnet` and use the `advisor` subagent (`model: opus`, read-only) from `assets/agents/advisor.md`.

## Delegating to Haiku

Delegate when the output would be large and mostly thrown away: repo-wide searches, call-site hunts, full test-suite logs, diff audits. Run independent searches in parallel, in the same turn.

Do not delegate when you already know the path. A direct `Read` is cheaper, because each subagent sends its own requests against the same usage limits.

A good delegation prompt names the question, the scope (directories or file types), and the return format, for example "paths with line numbers and a one-line reason each, no file contents pasted". Haiku tends to over-report when the format is open-ended.

## Setting it up

1. Copy the files in `assets/agents/` into `.claude/agents/` (this project) or `~/.claude/agents/` (every project). If the agents directory did not exist when the session started, restart Claude Code once so it gets watched.
2. Add the rule in `assets/claude-md-rule.md` to `CLAUDE.md`.
3. Show the user the diff before writing into their config.

For frontmatter fields, scope priority, `--agents` JSON, and the other gotchas, read `references/subagents.md`.

## Gotchas

- **`audit` runs in a worktree** (`isolation: worktree`), which is a separate checkout. Work that exists only as uncommitted edits may not be visible there. Commit first or paste `git diff` into the audit prompt, and check on first use that it sees what you expect.
- **Descriptions cost context** because they are always loaded. Keep them to a sentence or two and put the detail in the agent's body, which loads only when it runs.
- **Built-in `Explore` already exists** and does read-only search. A custom agent named `Explore` overrides it, so set `model: haiku` on one if you want built-in exploration on the cheap model instead of adding `search`.
- **Agent names must be unique** across the whole `.claude/agents/` tree.

## Left out on purpose

The source post ends with "JEV", described as a router for micro-decisions (tool choice, path selection, retries). It is not a Claude Code feature, and the post gives no install or setup details, so nothing here depends on it. Add a section if the user later supplies what it is.
