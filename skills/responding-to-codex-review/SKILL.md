---
name: responding-to-codex-review
description: Use when Codex (the auto-triggered PR reviewer) has posted comments on a pull request and you need to respond — react thumbs up/down to each comment, auto-implement the ones you agree with, resolve them on GitHub, and summarize on the linked Linear ticket
---

# Responding to a Codex Code Review

## Overview

Codex is configured as an **auto-triggered code reviewer**: every time a PR is opened (or new commits are pushed), Codex reviews the diff and posts inline review comments on the GitHub PR. This skill handles the response loop.

The user triggers this skill after Codex has responded (e.g., "respond to Codex", "address the Codex review", "Codex replied on PR 142"). Then, **for every Codex comment**, Claude Code:

1. **Thumbs up (👍) or thumbs down (👎)** the comment, depending on whether CC agrees with Codex.
2. **Agree → auto-implement the fix with no further buy-in**, push it, and mark the comment **resolved** on GitHub + note it on the linked Linear ticket.
3. **Disagree → 👎 + reply explaining why**, leave the thread **open**, and surface it to the user at the end.

> **Authorized autonomy.** Auto-implementing agreed-with Codex feedback is an **explicit, standing exception** to the global "state a plan and wait for approval" rule. The user has pre-approved implementing anything CC genuinely agrees with on a Codex review. Do **not** stop to ask for buy-in on agreed items — implement, push, resolve. (You still never merge the PR without approval.)

## When to Use

- Codex has posted review comments on an open PR and the user asks you to respond / address them.
- A re-review fires after you push fixes (the loop below repeats until no actionable Codex comments remain).

## When NOT to Use

- For your own pre-emptive review before Codex runs — that's Step 9 of `starting-linear-ticket` (`superpowers:code-reviewer`).
- Human reviewer comments — those still require the normal discuss-then-act flow, not blanket auto-implementation.

## Prerequisites

- The PR exists and Codex has reviewed it.
- You are working from the PR branch's worktree (so fixes land on the PR branch, **never main**). If you're not, check out that worktree first.
- `gh` is authenticated; the Linear MCP is connected.

## Process

```
identify PR ──> fetch Codex threads ──> triage each (agree/disagree)
   │                                              │
   │                          ┌───────────────────┴───────────────────┐
   │                       AGREE                                    DISAGREE
   │                  👍 + implement (TDD)                    👎 + reply, leave open
   │                          │                                       │
   └──> push fixes once ──> resolve agreed threads ──> Linear summary ──> report
                                   │
                          (push may re-trigger Codex → loop)
```

### Step 1 — Identify the PR, repo, and Linear ticket

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)   # owner/name
PR=$(gh pr view --json number -q .number)                     # current branch's PR
gh pr view "$PR" --json url,headRefName,body -q '.body'        # body holds the Linear id
```

Extract the Linear identifier (e.g., `PROJ-142`) from the PR body's `## Linear` section. Keep `REPO` and `PR` for later commands.

### Step 2 — Fetch Codex review threads

Pull every review thread with its comments, IDs, and resolution state:

```bash
gh api graphql -f query='
query($owner:String!,$name:String!,$pr:Int!){
  repository(owner:$owner,name:$name){
    pullRequest(number:$pr){
      reviewThreads(first:100){
        nodes{
          id isResolved isOutdated
          comments(first:30){
            nodes{ databaseId body path line url author{ login } reactions(first:20){ nodes{ content user{ login } } } }
          }
        }
      }
    }
  }
}' -F owner="${REPO%/*}" -F name="${REPO#*/}" -F pr="$PR"
```

- `id` = thread node id → used to **resolve** the thread.
- `comments[].databaseId` = REST comment id → used to **react** and **reply**.
- **Detect the Codex author dynamically.** Find comment authors whose `login` matches Codex's bot (case-insensitive `codex` or `chatgpt`; commonly `chatgpt-codex-connector[bot]`). Do **not** hardcode the login — if it's ambiguous, ask the user which author is Codex once, then proceed.
- **Know your own login** (you react/reply as the authenticated `gh` user): `ME=$(gh api user --jq .login)`.
- **Build the actionable set** = Codex-authored threads that are **not yet handled**. Skip a thread when any of these is true:
  - it's authored by a human (not Codex), or
  - it's already `isResolved`, or
  - **you've already handled it** — the Codex comment already carries a reaction from `$ME`, **or** the thread already contains a reply authored by `$ME`.
- This last check is essential: disagreed threads are intentionally left **unresolved** (Step 5), so without skipping already-handled threads the re-review loop (Step 6) would re-fetch the same disagreement every pass and **never go clean**.

### Step 3 — Triage each Codex comment (agree vs disagree)

For each Codex comment, decide with real judgment — do **not** rubber-stamp:

- **Agree** when the comment identifies a genuine bug, correctness/security issue, missing edge case, or a clear, valid improvement. When Codex claims a bug, **verify it first** (read the code; use `superpowers:systematic-debugging` if non-obvious) before agreeing — agreement means you've confirmed it's real.
- **Disagree** when Codex is wrong, the comment is based on a misread of the code, it conflicts with an intentional decision, it's a false positive, or the suggestion is net-negative. A plausible-sounding comment that's actually incorrect should get a 👎.

Record each as AGREE or DISAGREE before touching anything.

### Step 4 — For each AGREE: react 👍, then implement

React first:

```bash
gh api --method POST "repos/$REPO/pulls/comments/<databaseId>/reactions" -f content='+1'
```

Then **implement the fix** on the PR branch — no buy-in needed:
- Follow TDD (`superpowers:test-driven-development`): for a bug fix, add/adjust a failing test, then fix. Keep the suite green.
- Make the smallest correct change that addresses the comment.
- Batch all agreed fixes together before pushing (Step 6).

### Step 5 — For each DISAGREE: react 👎, reply, leave open

```bash
gh api --method POST "repos/$REPO/pulls/comments/<databaseId>/reactions" -f content='-1'
gh api --method POST "repos/$REPO/pulls/$PR/comments/<databaseId>/replies" \
  -f body='Respectfully disagree: <specific, concrete reason — cite the code/line or the intentional decision>.'
```

Do **not** resolve disagreed threads — leave them open so the user can override. Collect them for the final report.

### Step 6 — Commit and push the batched fixes

**Commit first.** `git push` only sends *commits* — uncommitted edits stay local, so the PR branch wouldn't actually contain your fixes while Step 7 resolves the threads as "fixed". Always commit before pushing:

```bash
git add -A
git status                                   # confirm the intended fixes are staged
git commit -m "Address Codex review: <short summary of the fixes>"
git push                                     # on the PR branch / in the worktree — NEVER main
```

Pushing new commits **may re-trigger Codex**. That's expected: after it re-reviews, run this skill again on the actionable set (Step 2 — which excludes threads you've already handled, so standing disagreements don't keep the loop alive). Continue until Codex has no remaining un-addressed comments.

### Step 7 — Resolve the agreed threads on GitHub

Only after the fix is pushed, resolve each AGREE thread:

```bash
gh api graphql -f query='
mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' \
  -f id='<thread node id>'
```

### Step 8 — Summarize on the linked Linear ticket

Post one comment on the Linear ticket (from Step 1) using the configured Linear MCP comment tool (e.g., `mcp__linear-server__create_comment` — match your project's Linear MCP server name):

```
Codex review addressed on PR #<PR>:
- Implemented & resolved (N): <one line each — what was fixed>
- Pushed back on (M): <one line each — why disagreed>
<PR URL>
```

If a Linear identifier wasn't found in the PR body, skip Linear and note that in the final report.

### Step 9 — Report to the user

Give a concise summary:
- **Implemented & resolved** (count + bullets) — pushed to the PR branch.
- **Disagreed / left open** (count + bullets with reasons) — **call these out explicitly** so the user can override any of your push-backs.
- Whether the push likely re-triggered Codex (loop status).

Never merge the PR as part of this skill — merging always waits for explicit user approval.

## Command Reference

| Action | Command |
|--------|---------|
| Repo slug | `gh repo view --json nameWithOwner -q .nameWithOwner` |
| PR number | `gh pr view --json number -q .number` |
| Fetch threads + IDs | `gh api graphql` query in Step 2 |
| 👍 / 👎 a comment | `gh api --method POST repos/$REPO/pulls/comments/<databaseId>/reactions -f content='+1'` (or `'-1'`) |
| Reply in a thread | `gh api --method POST repos/$REPO/pulls/$PR/comments/<databaseId>/replies -f body='...'` |
| Your own login | `gh api user --jq .login` |
| Commit + push fixes | `git add -A && git commit -m '...' && git push` (commit before push) |
| Resolve a thread | `gh api graphql` `resolveReviewThread` mutation in Step 7 |
| Linear summary | configured Linear MCP create-comment tool |

Reaction `content` values are GitHub's enum: `+1` = 👍, `-1` = 👎.

## Important / Guardrails

- **Auto-implement agreed items without asking** — this is the user's standing authorization. Hesitating to confirm defeats the purpose.
- **But verify before you agree.** Agreement is a judgment that the comment is correct — confirm claimed bugs against the actual code first. A wrong 👍 leads to a wrong auto-implementation.
- **Never resolve a thread you disagreed with.** Disagreements stay open for the user.
- **Never push to main; never merge.** Fixes go to the PR branch; merge waits for the user.
- **One reaction per comment** — react exactly once (👍 xor 👎) per Codex comment.
- **Loop on re-review** — pushing fixes re-triggers Codex; repeat until clean.

## Common Mistakes

### Rubber-stamping every Codex comment with 👍
- **Problem:** Codex can be confidently wrong; blanket agreement auto-implements bad changes.
- **Fix:** Triage with judgment (Step 3); verify claimed bugs before agreeing; 👎 false positives.

### Resolving threads before the fix is pushed
- **Problem:** Thread shows resolved but the fix isn't on the branch yet.
- **Fix:** Implement → commit → push (Step 6) → only then resolve (Step 7).

### Pushing without committing
- **Problem:** `git push` sends only commits, so uncommitted fixes never reach the PR branch — yet the threads get resolved as "fixed".
- **Fix:** Always `git add` + `git commit` (with a `git status` check) before `git push` (Step 6).

### Re-handling threads you already addressed
- **Problem:** Disagreed threads stay unresolved by design, so a naive "loop until no open threads" re-fetches them every re-review and never terminates.
- **Fix:** Skip threads that already carry your reaction or reply when building the actionable set (Step 2).

### Stopping to ask permission for agreed fixes
- **Problem:** Defeats the authorized-autonomy design; wastes the user's time.
- **Fix:** Implement agreed items immediately; only surface disagreements and the final summary.

### Hardcoding the Codex bot login
- **Problem:** The bot login can change; acting on the wrong author or missing Codex threads.
- **Fix:** Detect the author dynamically (Step 2); confirm once with the user if ambiguous.
