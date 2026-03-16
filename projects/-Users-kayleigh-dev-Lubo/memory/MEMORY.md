# Lubo Project Memory

## What is Lubo
- **Routine-first personal knowledge system** — "love your routines"
- Automates recurring workflows: daily focus, weekly reviews, backlog pruning
- Name: Irish *lúb* (loop) + Slavic *lubo* (beloved) = "beloved routines"
- Status: Ideation → PoC (March 2026)

## Product Docs (Notion)
- Main page: `31c179c60963818dbb46db250d93b2f1`
- PoC Spec: `31c179c609638105ba8df292291cf27f`
- Investment Memo: `31c179c60963813e82bdf9cb7e649b8f`
- Competitor Landscape: `31e179c6096381fb8bf3e110edf80ce3`
- Dead Startups Research: `31e179c60963817cb1d3cba71d4e6d2d`

## Architecture Decisions
- **Frontend:** Next.js (App Router), React, Tailwind, Zustand
- **Backend:** Supabase (Postgres + RLS + Auth) + Next.js API routes (replaces Edge Functions)
- **AI:** Claude API with streaming (Sonnet)
- **Integrations:** Google Calendar + Linear (v1 task integration)
- **Auth:** Supabase magic link (passwordless)
- **Data model:** Typed graph (projects, tasks, notes, meetings, sources) with JSONB metadata
- **Single project** (not monorepo) — everything in one Next.js app + `supabase/` dir

## v1 Validation Target
- Daily Routine: triage yesterday → Big 3 → inbox sweep
- Success: 10 consecutive completions, <5 min each, faster than manual
- Weekly Review and Backlog Pruning are planned but NOT v1

## Key Design Choices
- Routines are first-class objects (trigger, scope, steps, adaptation)
- **Chat-driven routine UX:** Split panel — chat (left) navigates the routine, routine panel (right) shows live state
- **Dual interaction:** User can click buttons on task cards OR type in chat; both update same state
- AI uses **Claude tool use** (function calling) to update routine state during chat
- Adaptation layer tracks user behavior and auto-adjusts (rule-based in v1)
- External integrations are read-primary; write-back only during routine execution
- **Visual style:** Warm / friendly — soft colors, rounded corners, gentle gradients
- **Graph model:** Hybrid — single `nodes` table with promoted columns + JSONB `data`

## Deployment
- **App:** https://lubo.vercel.app (Vercel, auto-deploys from `main`)
- **Database:** Supabase project `iejvipsxbjxbemxoalje`
- **NEVER deploy to prod manually** (`vercel --prod`) — let Vercel auto-deploy on merge to main. Use preview deployments from PR branches for testing.

## Tooling
- **Package manager:** Use `npm` (NOT `bun` — bun is not installed)

## Local Dev Gotchas
- **Dev server in worktrees:** Use `nohup npm run dev > /tmp/lubo-dev-server.log 2>&1 &` — NOT `run_in_background`. Claude Code's background task runner kills child processes when tasks complete/timeout.
- **Env vars:** `.env.local` uses `NEXT_PUBLIC_SUPABASE_ANON_KEY` but the app client expects `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`. Both must be set. When creating worktrees, copy `.env.local` from main repo.
- **DB NOT NULL columns:** `nodes.tags` (text[]), `nodes.source_ext` (jsonb), `nodes.data` (jsonb) are NOT NULL with defaults. The `nodeToDbRow()` helper strips null values for these so DB defaults apply. Don't send `null` for these columns in inserts.

## E2E Testing
- **Always write e2e tests** when making changes — covers auth flow, chat interactions, routine flow
- **Playwright setup:** Install in `/tmp/pw-runner/` (separate from project to avoid vitest conflicts). Run scripts with `node /tmp/pw-runner/test.mjs`
- **Auth flow:** Send magic link → fetch from Mailpit API (`http://127.0.0.1:54324/api/v1/messages`) → extract token URL → navigate to it
- **Prerequisites:** OrbStack running, `supabase start`, `supabase db reset` (run from worktree to get branch migrations), dev server running
- **Dashboard URL:** `/dashboard` (NOT `/today` — TodayPage renders at `/(app)/dashboard/page.tsx`)

## CopilotKit Gotchas
- **AnthropicAdapter baseURL:** Must pass `new Anthropic({ baseURL: "https://api.anthropic.com/v1" })` explicitly — SDK defaults to `/` without `/v1`, which breaks `@ai-sdk/anthropic`
- **Model ID:** Use `claude-sonnet-4-20250514` (NOT `claude-sonnet-4-5-...`)
- **`available` prop bug:** Do NOT use `available: "disabled"/"enabled"` on `useCopilotAction` — it changes the internal action type classification and throws "Action configuration changed between renders". Instead, guard handlers with a session-active check.

## Vercel CLI
- [Vercel env var management](feedback_vercel_env.md) — how to add env vars for production and preview

## Repo State
- Path: `/Users/kayleigh/dev/Lubo` (same as `/Users/kayleigh/Dev/Lubo` — macOS case-insensitive)
- Git repo, main branch tracks origin/main
