# claude-config — Project Instructions

Global Claude Code configuration: instructions (`Claude.MD`), custom skills (`skills/`), and workflow automation. This repo is `vxm5091/claude-config`.

## Linear

- **Team:** `GetBuddy` (key `GET`)
- **Project:** `Claude Config` — **all** tickets for this repo go here.

Every change to this repo is tracked as a ticket in the **Claude Config** project, built on a branch in a worktree, and shipped via a PR whose body references the ticket (`## Linear` section with `GET-<number>`). No PR without a project-linked ticket. See the global `Claude.MD` → "Task Tracking with Linear" and "Git Workflow".

## Installing skills

Skills in `skills/` are symlinked into `~/.claude/skills/` (repo stays the source of truth — edits take effect globally with no re-sync). When adding a new skill, symlink it after its PR merges:

```bash
ln -s "/Users/vlad/claude-config/skills/<name>" ~/.claude/skills/<name>
```

## Codex review

PRs in this repo are auto-reviewed by Codex. When Codex replies, use the `responding-to-codex-review` skill (👍/👎 each comment, auto-implement agreed items with no buy-in, resolve on GitHub + summarize on the Linear ticket).
