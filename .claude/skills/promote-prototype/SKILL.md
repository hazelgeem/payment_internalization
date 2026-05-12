---
name: promote-prototype
description: Promote the checkout prototype from `staging` to production (`main`) on GitHub. Use ONLY when the user explicitly types "/promote-prototype" or asks to "promote / 프로덕션 반영 / staging에서 main으로". Performs a fast-forward-only merge from `origin/staging` into `main` and pushes — the user must have already verified the staging URL. Aborts if main has diverged from staging (someone bypassed staging).
---

# /promote-prototype — Promote staging to production

After `/push-prototype` deploys changes to the staging URL, the user verifies them in the browser. Once verified, this skill promotes the same commits to `main`, which auto-deploys to the production URL.

This skill performs a **fast-forward-only merge** from `origin/staging` into `main`. If `main` has commits not present in `staging` (i.e., someone pushed directly to `main`), the skill aborts — that's a signal that someone bypassed the staging flow and the user must resolve it manually.

## Core rules

1. **Trigger only on explicit promote request.** Phrases like "프로덕션 반영해줘", "main으로 올려", "promote", or `/promote-prototype`. Never auto-promote after `/push-prototype`.
2. **Fast-forward only.** If `main` has diverged from `staging`, abort. Do not attempt merge commits or rebase rewrites of `main`.
3. **Show diff summary before promoting.** Display the commits and CHANGELOG versions being promoted. Get explicit go-ahead.
4. **No version bumps or CHANGELOG edits.** Those happened at `/push-prototype` time. Promote is a pure git merge.

## Repo facts

- **Production URL**: https://hazelgeem.github.io/payment_internalization/checkout.html
- **Staging URL**: https://hazelgeem.github.io/payment_internalization/staging/checkout.html
- **Source branch**: `staging`
- **Target branch**: `main`

## Prerequisites

```bash
git rev-parse --show-toplevel    # must be inside a git repo
git remote get-url origin         # should point to hazelgeem/payment_internalization
```

Run all commands from the repo root (`cd "$(git rev-parse --show-toplevel)"`).

## Execution steps

### 1. Fetch latest refs and gather diff summary

```bash
git fetch origin main staging
```

Then collect what would be promoted:

```bash
# Commits in staging not in main
COMMITS=$(git log --oneline origin/main..origin/staging)
# Number of commits to promote
N=$(git rev-list --count origin/main..origin/staging)
# CHANGELOG versions being added (peek at staging file)
NEW_VERSIONS=$(git show origin/staging:checkout.html | grep -oE "version: 'v[0-9]+\.[0-9]+'" | head -10)
```

If `N == 0`, stop:
> staging이 main과 동일해 — promote할 변경이 없어.

If `git rev-list --count origin/staging..origin/main` is **greater than 0** (main is ahead of staging), abort immediately:
> ⚠️ main이 staging보다 앞서 있어. 누군가 staging 우회해서 main에 직접 push했을 가능성이 있어. 다음 중 하나로 해결해줘:
> 1. 그 main 변경을 staging에도 반영해서 다시 정렬: `git fetch && git checkout staging && git merge origin/main && git push origin staging` (이후 다시 promote)
> 2. 그 변경이 실수였다면 main에서 revert 후 재시도
>
> promote 진행하지 않음.

### 2. Show diff summary and confirm

Display to the user:

> 다음 commit들을 staging → main으로 promote할게:
>
> ```
> <abc1234> v1.15: Shipping Agent redesign
> <def5678> v1.16: Add coupon validation
> ```
>
> 영향받는 버전: v1.15, v1.16
>
> staging URL에서 확인했지? 프로덕션 반영할까?

Only proceed on explicit go-ahead. If the user wants to see more detail, run `git diff origin/main..origin/staging -- checkout.html` for the file diff.

### 3. Fast-forward merge and push

```bash
# Switch to main locally
git checkout main
git pull --ff-only origin main

# Fast-forward merge from staging
git merge --ff-only origin/staging

# Push to remote
git push origin main
```

If `git merge --ff-only` fails (very rare since the divergence check in step 1 should catch this), abort and ask the user. Do not force-push.

### 4. Return to staging branch

After successful promote, leave the user back on `staging` since that's their normal working branch:

```bash
git checkout staging
git pull --ff-only origin staging
```

(If staging was fast-forwarded as part of someone else's earlier push, this catches up. Otherwise no-op.)

### 5. Report back with production URL

> ✅ **v1.16 (commit `def5678`)** → main에 promote됨.
>
> 1-2분 후 프로덕션 반영 확인: https://hazelgeem.github.io/payment_internalization/checkout.html
>
> staging 브랜치는 그대로 유지 — 다음 작업도 여기서 시작.

Get the short SHA with `git rev-parse --short origin/main`.

## When NOT to trigger this skill

- Immediately after `/push-prototype` finishes — the user must verify staging URL first
- Normal editing or conversation about the prototype
- When the user mentions "push" or "upload" without specifying promote/main/production (use `/push-prototype` instead)
- When asked to revert or roll back (not this skill's job — use `git revert` manually)

Only trigger on explicit verbs like "promote", "프로덕션 반영", "main으로 올려", "프로덕션에 반영", or the literal `/promote-prototype` command.

## Edge cases

- **staging branch doesn't exist locally**: `git fetch && git checkout -b staging origin/staging` (or `git checkout staging` if remote-tracking already set). Then re-run.
- **main is ahead of staging** (someone bypassed): abort per step 1.
- **No new commits on staging**: stop, "promote할 변경 없음".
- **Push to main rejected** (race condition): the merge succeeded locally but origin/main moved. Tell user to retry — likely the other person promoted at the same moment.
