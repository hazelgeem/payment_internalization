---
name: push-prototype
description: Push the Payment Internalization checkout prototype to GitHub, bumping the version and recording a changelog entry. Use ONLY when the user explicitly types "/push-prototype" or asks to "upload / push / deploy the prototype to GitHub". Never auto-push on regular edits — prototype changes stay local until this skill is invoked. The skill drafts a changelog entry (version bump + bullet list of changes since last push), shows it for confirmation, then commits and pushes via git.
---

# /push-prototype — Release the checkout prototype to GitHub

The repo `hazelgeem/payment_internalization` is co-managed by Hazel (`hazel.k@bunjang.co.kr`) and Nancy Lee (PD, `nancy.lee@bunjang.co.kr`). Both clone locally, edit `checkout.html` via Claude Code, and push directly to `main` using this skill. There is no PR review — formality is replaced by git history (revertable if anything goes wrong).

The skill auto-detects who is running it from `git config user.email` and stamps the changelog entry accordingly.

## Core rules

1. **Never push to GitHub automatically.** Even if the user says "perfect" or "looks good" after an edit, that's acceptance of the edit — not a push request. Wait for an explicit push request.
2. **Version bumps only happen on push.** Don't edit the `CHANGELOG` array or the `proto-nav-version-tag` text during normal editing. Only this skill touches them.
3. **Every push adds exactly one changelog entry** at the top of the `CHANGELOG` array. Group all accumulated changes since the last push into that single entry. Each entry must include an `author` field.
4. **Always pull before pushing.** The other person may have committed since the last sync. Pulling first prevents conflicts and silent overwrites.

## Repo facts

- **GitHub repo**: `hazelgeem/payment_internalization`
- **Live URL**: https://hazelgeem.github.io/payment_internalization/
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

### 1. Pre-flight: fetch and check for drift

```bash
git fetch origin main
BEHIND=$(git rev-list --count HEAD..origin/main)
AHEAD=$(git rev-list --count origin/main..HEAD)
echo "behind=$BEHIND ahead=$AHEAD"
```

- **`BEHIND > 0`** (other person committed since last sync): pull first.
  ```bash
  git pull --rebase origin main
  ```
  If rebase has conflicts, abort with `git rebase --abort` and ask the user to resolve manually before retrying. Do not attempt automated conflict resolution on `checkout.html`.

- **`BEHIND == 0`**: proceed.

### 2. Determine the version number

Read the current version tag from `checkout.html`:

```html
<button class="proto-nav-version-tag" onclick="openChangelog()" title="View changelog">v1.14</button>
```

Bump to the next patch version (e.g., `v1.14` → `v1.15`). Use minor bumps (`v1.14` → `v2.0`) only when the user explicitly asks for a major release.

### 3. Determine the author

```bash
AUTHOR_EMAIL=$(git config user.email)
```

Use this value for the `author` field of the new CHANGELOG entry.

### 4. Draft the changelog entry

Summarize all changes made to `checkout.html` since the last push. Sources of truth for the diff:
- The conversation context (what the user asked, what edits were made)
- If unclear, diff against `origin/main`: `git diff origin/main -- checkout.html`

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
- 변수명/식별자 정리 (variable name cleanup — e.g., Spec 모드 변수 통일, 인터널 라벨 일관성)
- 단순 텍스트 다듬기 (copy/wording tweaks that don't change the underlying policy)
- 리팩터링 (refactoring with no user-facing impact)

The changelog audience is stakeholders reviewing what *spec/policy* moved between versions, not what visual polish was applied. If a change is purely visual or fixes something broken, leave it out — even if the user spent significant time on it during the session.

If after applying these exclusions there are no meaningful bullets left, stop and ask the user: "v1.X 이후 변경된 내용이 모두 버그 수정/UI 다듬기라 changelog에 올릴 항목이 없는데, 그래도 푸시할까?" — let them decide.

### 5. Show the draft for confirmation

Before touching the file or pushing, display the planned version + author + changelog entry and ask for confirmation:

> **v1.15 (2026-05-08) · hazel.k@bunjang.co.kr** 로 푸시할게. 변경사항:
> - bullet 1
> - bullet 2
>
> 이대로 올릴까?

If the user wants edits, revise and re-show. Only proceed on explicit go-ahead.

### 6. Apply the updates to checkout.html

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

### 7. Commit and push

```bash
git add checkout.html
git commit -m "v1.15: <one-line English summary>"
git push origin main
```

Commit message format: `vX.Y: <one-line summary of the release>` — English, scannable (e.g., `v1.15: Add 결제 버튼 위치 조정`).

If the push is rejected (non-fast-forward), it means someone committed in the seconds between step 1 and step 7. Run `git pull --rebase origin main` and retry the push.

### 8. Report back

After the push, tell the user:
- The new version (e.g., `v1.15`)
- The commit SHA (`git rev-parse --short HEAD`)
- The live URL: https://hazelgeem.github.io/payment_internalization/ — typically updates within 1-2 minutes.

## When NOT to trigger this skill

- Normal editing conversations — even if the user says "good, that works" after a change
- When the user asks about the prototype but doesn't mention pushing/uploading/deploying
- When the user asks to preview or test locally
- When the request is to edit a different file (PRD, TODO, etc.) that also happens to live in the `payment_internalization` folder

Only trigger on explicit verbs like "push", "upload", "deploy", "github에 올려", "github 반영", or the literal `/push-prototype` command.

## Edge cases

- **No changes to checkout.html since last push**: If `git diff origin/main -- checkout.html` is empty, stop — don't create an empty changelog entry.
- **Conflict on rebase**: `git rebase --abort` and surface the situation to the user. Do not attempt automated resolution on `checkout.html`.
- **Author field missing on historical entries**: Don't backfill. The renderer treats `author` as optional (only displays the email when present). Just include it on every new entry going forward.
- **Working dir not a git repo**: Stop and tell the user — this skill only works against a cloned repo.
