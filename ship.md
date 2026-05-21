---
name: ship
description: Ship the current branch — verify types/build/tests pass, open or update a PR, and merge to main. Stops after merge. Use when done iterating on a branch and ready to merge. Works with any Node/Deno/Python/Go/Rust project — auto-detects available quality gates and build tools.
---

# /ship

End-to-end "merge" runbook. By invoking `/ship`, you have authorized the operations in steps 3–4 (push, merge). Do NOT re-prompt for those — but DO confirm before any operation outside this runbook (force push, hard reset, amending merged commits, etc.).

This skill **does not deploy directly**. Merging to `main` triggers whatever CI/CD pipeline is connected to the repo (Cloudflare Workers Builds, Vercel, Netlify, Railway, GitHub Actions deploy job, etc.). Never run a deploy CLI command from `/ship`.

If any step fails, stop and surface the exact error. Never bypass with `--no-verify`, `--force`, or by skipping a check.

---

## Step 1 — preflight

```bash
git rev-parse --abbrev-ref HEAD
git status
git log origin/main..HEAD --oneline 2>/dev/null || git log origin/master..HEAD --oneline 2>/dev/null
```

- **If on `main` or `master`**: stop. `/ship` runs on a feature branch.
- **If working tree is dirty**: auto-commit and continue — do not prompt. Stage tracked modifications (`git add -u`) and untracked files individually by path (never `git add .`). Generate a one-line commit message in repo style — `feat: <topic> — <details>` / `fix: <topic> — <details>` (check `git log -5` for examples). Use a HEREDOC with `Co-Authored-By: Claude` trailer. Then continue.
- **If 0 commits ahead of origin/main (or master)**: stop. Nothing to ship.

---

## Step 2 — detect and run quality gates

First, identify what's available. Read `package.json` (if present) and scan for common config files:

```bash
# Detect package manager
ls package-lock.json yarn.lock pnpm-lock.yaml bun.lockb 2>/dev/null

# Check available npm scripts
cat package.json | grep -E '"(build|typecheck|type-check|tsc|test|lint|check)"' 2>/dev/null

# Check for language-specific tooling
ls tsconfig.json tsconfig.*.json 2>/dev/null   # TypeScript
ls Cargo.toml 2>/dev/null                       # Rust
ls go.mod 2>/dev/null                           # Go
ls pyproject.toml setup.py requirements*.txt 2>/dev/null  # Python
ls Makefile 2>/dev/null                         # Make-based
```

Then run gates in this order — **stop on any hard failure**:

### TypeScript / type check (blocking)
If `tsconfig.json` exists and a typecheck script is present (`typecheck`, `type-check`, `tsc`), run it. If no script but `tsconfig.json` exists, run `npx tsc --noEmit`. Must pass.

### Build (blocking)
If a `build` script is in `package.json`, run it. For Rust: `cargo build --release`. For Go: `go build ./...`. For Python: skip unless a `build` script is present. Must pass.

### Tests (blocking)
If a `test` script is in `package.json` and it's not a placeholder (i.e., not `echo "Error: no test specified"`), run it. For Rust: `cargo test`. For Go: `go test ./...`. For Python: `pytest` if present. Must pass.

### Lint (advisory)
If a `lint` script is in `package.json`, run it:
1. Run `npm run lint` (or equivalent).
2. If it fails, run the auto-fix variant (`lint:fix`, `eslint . --fix`, `prettier --write .`) once. Stage with `git add -u` and commit as `style: auto-fix lint` if that fully clears it.
3. If errors remain, count how many are **new on this branch** vs already on `main`. Fix only the new ones. Do NOT block the ship on pre-existing lint errors — surface the count to the user at the end.
4. Never silence errors with `eslint-disable` or `// @ts-ignore` comments just to pass.

If no quality gates are found (no `package.json`, no `Makefile`, no language tooling), note this in the final report and continue — the PR can still be merged.

---

## Step 3 — push and open or update the PR

```bash
gh pr view --json number,state,url,mergeable 2>/dev/null
```

- **PR exists and is OPEN**: `git push` to update it.
- **No PR**: `git push -u origin HEAD`, then `gh pr create` with a HEREDOC body (Summary + Test plan). Title: `feat: <topic> — <details>` / `fix: <topic> — <details>`.
- **PR is MERGED or CLOSED**: stop and ask the user — the branch may already be shipped or intentionally abandoned.

Wait for required CI checks:

```bash
gh pr checks --watch
```

If any check fails, stop and report. Do not merge a red PR.

---

## Step 4 — merge (resolving conflicts if needed)

```bash
gh pr merge --squash --delete-branch
```

If `gh pr merge` fails due to merge conflicts:

1. `git fetch origin main` (or `master`)
2. `git merge origin/main` — use merge, not rebase, unless the user explicitly asked for rebase.
3. Open each conflicted file with Read, then Edit to apply the **union of intent** from both sides. Never wholesale discard either side without understanding why both changed.
4. Re-run Step 2's quality gates.
5. `git add <resolved files>` and `git commit` (no `--no-verify`).
6. `git push`, re-run `gh pr checks --watch`, then retry the merge.

If the merge fails for another reason (review required, branch protection, draft state), surface the exact `gh` error — don't paper over it.

---

## Step 5 — clean up local dev processes

Kill any dev server (node, vite, webpack, wrangler, deno, uvicorn, air, etc.) whose working directory is **this worktree only** — other agents may be running servers from sibling worktrees, so scope by `cwd`.

```bash
WORKTREE=$(pwd)

# Find processes rooted in this worktree
PIDS=$(lsof -a -d cwd -c node -c workerd -c deno -c python -c uvicorn -c air 2>/dev/null \
  | awk -v w="$WORKTREE" '$NF == w {print $2}' | sort -u)

FREED_PORTS=""
if [ -n "$PIDS" ]; then
  for PID in $PIDS; do
    PORTS=$(lsof -aPi -p $PID 2>/dev/null \
      | awk '/LISTEN/ {split($9,a,":"); print a[length(a)]}' | sort -u)
    FREED_PORTS="$FREED_PORTS $PORTS"
  done
  echo "$PIDS" | xargs kill 2>/dev/null
  sleep 1
  REMAINING=$(echo "$PIDS" | xargs -I{} sh -c 'kill -0 {} 2>/dev/null && echo {}')
  [ -n "$REMAINING" ] && echo "$REMAINING" | xargs kill -9 2>/dev/null
fi

echo "freed ports: $FREED_PORTS"
```

If `kill` fails or processes refuse to die, surface the PIDs and ports to the user — don't retry silently.

---

## Step 6 — report

In this order:

1. **PR number + URL** (now merged)
2. **Quality gates**: which ran, which passed, which were skipped (and why)
3. **Lint advisory**: pre-existing error count if any were not fixed, so you know what's already on `main`
4. **Cleanup**: which ports were freed; note any still held by other agents
5. **CI/CD reminder**: note which platform was detected in Step 1 of `/preview` (if known) and that it will now run against `main`. If this branch added migration files, database schema changes, or env var requirements, call them out explicitly so you know what's about to land in production
6. **Follow-up actions**: secrets to set, feature flags to flip, manual steps the CI pipeline can't handle

Keep it short. The point is confirming the merge landed cleanly, not a wall of text.

---

## Hard rules

- Never `git push --force` to `main` or `master`.
- Never use `--no-verify` to skip hooks.
- Never `git reset --hard` or `git checkout .` on the user's branch without explicit confirmation.
- Never run a deploy CLI command (`wrangler deploy`, `vercel deploy`, `netlify deploy`, etc.) as part of `/ship`.
- Never edit `.git/config` or change the merge style (squash) without the user asking.
