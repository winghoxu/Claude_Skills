---
name: preview
description: Push the current branch to trigger a preview deployment, open or update a PR, and print the per-branch preview URL. Works with Vercel, Netlify, Cloudflare Workers/Pages, Railway, and Render. Works from `main`, detached HEAD, or any feature branch (auto-creates a branch when needed). Use when you want a shareable preview of in-progress work.
---

# /preview

End-to-end "preview" runbook. By invoking `/preview`, you have authorized branch creation, push, and PR creation. Do NOT re-prompt for those — but DO confirm before any operation outside this runbook (force push, hard reset, amending merged commits, etc.).

The deliverable is the per-branch preview URL, **as fast as possible**. This skill deliberately skips local quality gates. A failed preview deployment is an acceptable outcome; the goal is to see *something* deployed (or know it failed) quickly, not wait on local builds.

This skill **does not deploy directly** — it relies on the connected CI/CD platform, which auto-builds every push to a non-`main`/`master` branch.

If a step fails, stop and surface the exact error. Never bypass with `--no-verify` or `--force`.

---

## Step 1 — detect platform

Scan the repo root for platform config files to identify the deployment provider:

```bash
ls wrangler.toml wrangler.json wrangler.jsonc 2>/dev/null && echo "cloudflare-workers"
ls .cloudflare/ 2>/dev/null && echo "cloudflare-pages"
ls vercel.json .vercel/ 2>/dev/null && echo "vercel"
ls netlify.toml netlify.json 2>/dev/null && echo "netlify"
ls railway.json railway.toml 2>/dev/null && echo "railway"
ls render.yaml 2>/dev/null && echo "render"
```

Also check `package.json` for deploy scripts or platform-specific dependencies if config files are absent. Set `PLATFORM` to whichever matches (default to `unknown` if none found). Note it — you'll use it in Step 3.

---

## Step 2 — preflight

```bash
git rev-parse --abbrev-ref HEAD            # current branch
git status --porcelain                      # working-tree state
git log origin/main..HEAD --oneline 2>/dev/null || git log origin/master..HEAD --oneline 2>/dev/null
```

- **If branch is `main`, `master`, or `HEAD` (detached)**: auto-create a feature branch from the current commit and continue — do not prompt. Derive the name from the worktree directory:
  ```bash
  RAW=$(basename "$(git rev-parse --show-toplevel)")
  SLUG=$(echo "$RAW" | sed -E 's/[^a-zA-Z0-9]+/-/g; s/^-+//; s/-+$//' | tr '[:upper:]' '[:lower:]')
  ```
  If a branch named `$SLUG` already exists locally or on the remote, fall back to `<slug>-<short-sha>` (e.g. `my-project-a1b2c3d`). Then `git checkout -b "$SLUG"`. Refresh the branch variable from `git rev-parse --abbrev-ref HEAD` before continuing.

- **If 0 commits ahead AND working tree is clean**: stop. There is nothing to preview.

- **If working tree is dirty**: auto-commit and continue — do not prompt. Stage tracked modifications (`git add -u`) and untracked files individually by path (never `git add .`). Generate a one-line commit message — `feat: <topic> — <details>` / `fix: <topic> — <details>` (check `git log -5` for repo style). Use a HEREDOC with `Co-Authored-By: Claude` trailer. Then continue.

---

## Step 3 — push and open or update the PR

Skip local build/lint/typecheck. If something is broken, CI will catch it.

```bash
gh pr view --json number,state,url 2>/dev/null
```

- **PR exists and is OPEN**: `git push` to update it.
- **No PR**: `git push -u origin HEAD`, then `gh pr create` with a HEREDOC body (Summary + Test plan). Title: `feat: <topic> — <details>` / `fix: <topic> — <details>`.
- **PR is MERGED**: stop. The branch is already shipped.
- **PR is CLOSED (not merged)**: ask the user whether to reopen or push to a fresh branch.

---

## Step 4 — compute the preview URL and poll until deploy finishes

### URL derivation by platform

Compute the branch slug:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
SLUG=$(echo "$BRANCH" | sed -E 's/[^a-zA-Z0-9]+/-/g; s/^-+//; s/-+$//' | tr '[:upper:]' '[:lower:]')
```

Then derive the preview URL based on `PLATFORM`:

**Vercel** — extract from the PR check output (Vercel posts a deployment URL as a check). Fall back to:
```
https://<repo-name>-git-<slug>-<github-username>.vercel.app
```
To get the actual URL, inspect `gh pr checks --json name,targetUrl` for an entry matching `vercel`.

**Netlify** — URL is always:
```
https://deploy-preview-<PR_NUMBER>--<site-name>.netlify.app
```
Get `<site-name>` from `netlify.toml`'s `[context]` block or from `gh pr checks` where the check name includes `netlify`.

**Cloudflare Workers** — URL pattern:
```
https://<slug>-<worker-name>.<account-subdomain>.workers.dev
```
Get `<worker-name>` from `wrangler.toml` (`name = "..."` field). Get `<account-subdomain>` from `gh pr checks` target URL or from `wrangler.toml`.

**Cloudflare Pages** — URL pattern:
```
https://<slug>.<project-name>.pages.dev
```
Get `<project-name>` from `.cloudflare/` config or the `gh pr checks` target URL.

**Railway / Render / unknown** — don't try to guess the URL. Instead extract it directly from `gh pr checks --json name,targetUrl` for any check whose name matches `(?i)railway|render|deploy|preview`.

If the slug exceeds ~50 chars, warn that the URL may be truncated by the platform.

### Poll the deployment check

```bash
for i in $(seq 1 18); do
  RESULT=$(gh pr checks --json name,state,targetUrl \
    --jq '[.[] | select(.name | test("(?i)vercel|netlify|cloudflare|workers|pages|railway|render|deploy"))] | 
      if length == 0 then "PENDING|" 
      elif (.[0].state | test("(?i)success|completed|pass")) then "SUCCESS|\(.[0].targetUrl)" 
      elif (.[0].state | test("(?i)fail|error|cancel")) then "FAILURE|\(.[0].targetUrl)"
      else "PENDING|" end')
  STATUS=$(echo "$RESULT" | cut -d'|' -f1)
  CHECK_URL=$(echo "$RESULT" | cut -d'|' -f2)
  echo "[$i] $STATUS"
  if [ "$STATUS" = "SUCCESS" ] || [ "$STATUS" = "FAILURE" ]; then break; fi
  sleep 10
done
```

If `gh pr checks` doesn't expose the deployment check yet (it can lag), fall back to `curl` on the derived URL:
```bash
curl -sI -o /dev/null -w '%{http_code}' "$PREVIEW_URL"
```
200/302 = live; 404/502/523 = not yet.

Stop polling on **SUCCESS**, **FAILURE**, or **TIMEOUT** (>3 min). On timeout, hand back the URL anyway with a note that the deploy is still running.

---

## Step 5 — report

In this order:

1. Branch name + last commit pushed (`git log -1 --oneline`)
2. PR number + URL (created or updated)
3. Deployment check status: ✅ live / ❌ failed / ⏳ still building
4. **Preview URL** — rendered as a markdown link, not a bare URL or code block: `[Open preview](<url>)`
5. If the build failed: a one-line nudge to check the failed check logs (`gh run view <id> --log-failed`) or run your project's local build command to reproduce.
6. If platform is `unknown`: note which config files were missing and which platform you guessed.

Keep it short. The point is a URL, not a wall of text.

---

## Hard rules

- Never `git push --force`.
- Never use `--no-verify` to skip hooks.
- Never deploy directly from this skill — let the platform's CI handle it.
- Never merge the PR from `/preview`.
- Never push directly to `main` or `master` — if invoked on those branches, branch off first (Step 2).
