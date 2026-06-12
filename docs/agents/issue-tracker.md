# Issue tracker: GitHub + Linear

This project uses two trackers with distinct scopes:

- **GitHub Issues** — bug reports and implementation tasks on the Bun fork.
  Use the `gh` CLI against your fork of `oven-sh/bun`.
- **Linear** — higher-level project tracking (milestones, epics, cross-cutting concerns).

## GitHub conventions

- **Create**: `gh issue create --title "..." --body "..."` (heredoc for multi-line bodies)
- **Read**: `gh issue view <number> --comments`
- **List**: `gh issue list --state open --json number,title,body,labels,comments`
- **Label**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v` inside `bun/` — `gh` picks it up automatically.

## Linear conventions

Use the Linear web app or CLI for epic/milestone tracking. When a skill says
"publish to the issue tracker", use GitHub for implementation-level tasks and Linear
for anything that spans multiple implementation tasks or represents a project milestone.

## When a skill says "publish to the issue tracker"

Default to GitHub Issues. Use Linear when the item is a milestone, epic, or
cross-cutting concern that spans multiple Bun-fork tasks.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments` for GitHub items.
For Linear items, open the Linear app or use the Linear CLI.
