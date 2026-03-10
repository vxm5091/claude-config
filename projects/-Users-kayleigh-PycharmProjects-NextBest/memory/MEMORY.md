# Memory

## Workflow

- **Claude config repo**: `kmigdol/claude-config` at `~/.claude/` — separate git repo for global CLAUDE.md, skills, and project settings. Push changes there (not to NextBest). Note: git tracks the file as `Claude.MD` (not `CLAUDE.md`).

## Patterns

- Always use **pnpm** for frontend (never npm), **uv** for backend/prefect

## Agent Lessons

- Explore agents can make wrong claims about framework conventions (e.g. Next.js middleware). Always verify version-specific behavior.
