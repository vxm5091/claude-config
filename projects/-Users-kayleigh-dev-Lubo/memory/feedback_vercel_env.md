---
name: Vercel env var management
description: How to add environment variables to Vercel using the CLI — syntax for production vs preview deployments
type: feedback
---

Use `vercel env add` to set environment variables. The CLI is installed and authenticated as `kmigdol`.

**Production:**
```bash
vercel env add VAR_NAME production --value "value" < /dev/null
```

**Preview (all branches):**
```bash
vercel env add VAR_NAME preview "" --value "value"
```

The empty string `""` as the branch argument means "all preview branches". Without it, the CLI prompts interactively and fails in non-interactive mode.

**Verify:**
```bash
vercel env ls | grep VAR_NAME
```
