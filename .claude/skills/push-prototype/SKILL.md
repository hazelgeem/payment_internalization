---
name: push-prototype
description: Push the Payment Internalization checkout prototype to the **staging** branch on GitHub. Use ONLY when the user explicitly types "/push-prototype" or asks to "upload / push / deploy the prototype to GitHub". Never auto-push on regular edits. The skill bumps the version, drafts a changelog entry, shows it for confirmation, then commits and pushes to `staging` (NOT main). Production is reached separately via `/promote-prototype` after the user verifies the staging URL.
---

# /push-prototype — Release the checkout prototype to STAGING

The repo `hazelgeem/payment_internalization` is co-managed by Hazel (`hazel.k@bunjang.co.kr`) and Nancy Lee (PD, `nancy.lee@bunjang.co.kr`). Both clone locally, edit `checkout.html` via Claude Code, and push to the `staging` branch using this skill. Production (`main`) is reached separately via `/promote-prototype` after the user verifies the staging URL in the browser.

This skill auto-detects who is running it from `git config user.email` and stamps the changelog entry accordingly.

## Core rules

1. **Never push automatically.** Even if the user says "perfect" or "looks good" after an edit, that's acceptance of the edit — not a push request. Wait for an explicit push request.
2. **Push target is always `staging`.** This skill never touches `main`. Production is only updated via `/promote-prototype`.
3. **Version bumps only happen on push.** Don't edit the `CHANGELOG` array or the `proto-nav-version-tag` text during normal editing. Only this skill touches them.
4. **Every push adds exactly one changelog entry** at the top of the `CHANGELOG` array. Group all accumulated changes since the last push into that single entry. Each entry must include an `author` field.
5. **Always pull `staging` before pushing.** The other person may have pushed since the last sync.

## Repo facts

- **GitHub repo**: `hazelgeem/payment_internalization`
- **Staging URL**: https://hazelgeem.github.io/payment_internalization/staging/checkout.html
- **Production URL** (only reached via promote): https://hazelgeem.github.io/payment_internalization/checkout.html
- **File being edited**: `checkout.html` (at repo root)
- The local working tree is wherever the user has cloned the repo. Never hardcode paths — derive the repo root from `git rev-parse --show-toplevel`.

## Prerequisites

Verify once per session before the first push:

```bash
git rev-parse --show-toplevel    # must be inside a git repo
git config user.email             # must be set; used as changelog author
git remote get-url origin         # should point to hazelgeem/payment_internalization
```

If `user.email` is missing, stop and ask the user to set it:
```bash
git config user.email "<their bunjang email>"
git config user.name "<their name>"
```

## Execution steps

Run all commands from the repo root (`cd "$(git rev-parse --show-toplevel)"`).

### 1. Ensure local is on the `staging` branch

```bash
git fetch origin
CURRENT=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT" != "staging" ]; then
  # Stash any in-progress changes so they ride along to staging
  git stash push -u -m "push-prototype-autostash" 2>/dev/null || true
  if git show-ref --verify --quiet refs/heads/staging; then
    git checkout staging
  else
    git checkout -b staging origin/staging
  fi
  git stash pop 2>/dev/null || true
fi
```

If the user was working on `main` (or any other branch) with local edits to `checkout.html`, those edits move to `staging` via the stash. After this step, the user is on `staging` with their edits intact.

### 2. Pre-flight: fetch and check drift against `origin/staging`

```bash
git fetch origin staging
BEHIND=$(git rev-list --count HEAD..origin/staging)
echo "behind=$BEHIND"
```

- **`BEHIND > 0`** (other person pushed to staging since last sync): pull first.
  ```bash
  git pull --rebase origin staging
  ```
  If rebase has conflicts, abort with `git rebase --abort` and ask the user to resolve manually before retrying. Do not attempt automated conflict resolution on `checkout.html`.

- **`BEHIND == 0`**: proceed.

### 3. Determine the version number

Read the current version tag from `checkout.html`:

```html
<button class="proto-nav-version-tag" onclick="openChangelog()" title="View changelog">v1.14</button>
```

Bump to the next patch version (e.g., `v1.14` → `v1.15`). Use minor bumps (`v1.14` → `v2.0`) only when the user explicitly asks for a major release.

### 4. Determine the author

```bash
AUTHOR_EMAIL=$(git config user.email)
```

Use this value for the `author` field of the new CHANGELOG entry.

### 5. Draft the changelog entry

Summarize all changes made to `checkout.html` since the last push. Sources of truth for the diff:
- The conversation context (what the user asked, what edits were made)
- If unclear, diff against `origin/staging`: `git diff origin/staging -- checkout.html`

Entry format (matches existing entries in `CHANGELOG`):

```js
{
  version: 'v1.15',
  date: 'YYYY-MM-DD',                  // today's date in Asia/Seoul
  author: 'hazel.k@bunjang.co.kr',     // from git config user.email
  changes: [
    '한국어로 작성된 한 줄 요약 (bullet)',
    '또 다른 변경사항',
  ]
}
```

The `author` field is mandatory for all new entries.

**Writing style for changelog bullets:**
- Korean, short, user-facing. Describe what the USER sees, not the implementation.
- Good: `"주문서 동의항목 3개 → 4개로 개편"` (정책/스펙 변화)
- Bad: `"Edited HTML to remove togglePaypalFeeInfo() button binding"` (구현 디테일)
- New features / policy changes / spec changes: include them. Start with the feature area, e.g., "Proxy 카드에 ~ 추가", "주문서 ~ 정책 변경".

**What to EXCLUDE from the changelog (do NOT add these as bullets):**
- 버그 수정 (bug fixes — broken UI, blank screens, JS errors, navigation issues)
- 작은 UI 수정 (small UI tweaks — heading size adjustments, color tweaks, header height/padding normalization, font weight changes, alignment fixes, button styling)
- 변수명/식별자 정리 (variable name cleanup)
- 단순 텍스트 다듬기 (copy/wording tweaks that don't change the underlying policy)
- 리팩터링 (refactoring with no user-facing impact)

The changelog audience is stakeholders reviewing what *spec/policy* moved between versions. If a change is purely visual or fixes something broken, leave it out.

If after exclusions there are no meaningful bullets left, ask the user: "v1.X 이후 변경된 내용이 모두 버그 수정/UI 다듬기라 changelog에 올릴 항목이 없는데, 그래도 staging에 푸시할까?" — let them decide.

### 6. Show the draft for confirmation

Before touching the file or pushing, display the planned version + author + changelog entry and ask for confirmation:

> **v1.15 (2026-05-12) · hazel.k@bunjang.co.kr** 로 staging에 푸시할게. 변경사항:
> - bullet 1
> - bullet 2
>
> 이대로 올릴까? (push 후 staging URL에서 확인하고, OK면 `/promote-prototype`으로 프로덕션 반영)

If the user wants edits, revise and re-show. Only proceed on explicit go-ahead.

### 7. Apply the updates to checkout.html

Two edits, in order:

**A. Bump the nav version tag** (find and update):
```html
<button class="proto-nav-version-tag" onclick="openChangelog()" title="View changelog">v1.14</button>
```
→
```html
<button class="proto-nav-version-tag" onclick="openChangelog()" title="View changelog">v1.15</button>
```

Also update the `Updated YYYY-MM-DD` span next to it.

**B. Prepend the new entry to the `CHANGELOG` array** (in the `<script>` section). Insert the new object as the first array element. Keep existing entries unchanged.

### 8. Commit and push to staging

```bash
git add checkout.html
git commit -m "v1.15: <one-line English summary>"
git push origin staging
```

Commit message format: `vX.Y: <one-line summary of the release>` — English, scannable.

If the push is rejected (non-fast-forward), it means someone pushed in the seconds between step 2 and step 8. Run `git pull --rebase origin staging` and retry the push.

### 9. Report back with staging URL + next step

After the push, tell the user:

> ✅ **v1.15** → staging에 배포됨 (commit `abc1234`).
>
> 1-2분 후 검증: https://hazelgeem.github.io/payment_internalization/staging/checkout.html
>
> 확인하고 문제없으면 `/promote-prototype` 으로 프로덕션 반영해줘.

Get the short SHA with `git rev-parse --short HEAD`.

## When NOT to trigger this skill

- Normal editing conversations — even if the user says "good, that works" after a change
- When the user asks about the prototype but doesn't mention pushing/uploading/deploying
- When the user asks to preview or test locally
- When the request is to edit a different file (PRD, TODO, etc.) that also happens to live in the `payment_internalization` folder

Only trigger on explicit verbs like "push", "upload", "deploy", "github에 올려", "github 반영", or the literal `/push-prototype` command.

## Edge cases

- **No changes to checkout.html since last staging push**: If `git diff origin/staging -- checkout.html` is empty, stop — don't create an empty changelog entry.
- **Conflict on rebase**: `git rebase --abort` and surface to the user. Do not attempt automated resolution on `checkout.html`.
- **Author field missing on historical entries**: Don't backfill. The renderer treats `author` as optional.
- **Working dir not a git repo**: Stop and tell the user — this skill only works against a cloned repo.
- **User wants to push directly to production**: Refuse and explain the staging flow. Push to staging first, verify, then `/promote-prototype`. If they insist on a hotfix bypass, they need to run git commands manually — this skill does not push to main.
