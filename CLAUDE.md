# Repo guidance for Claude Code

This repo (`hazelgeem/payment_internalization`) holds the checkout prototype (`checkout.html`). It is co-managed by two people with **different editing scopes**. Identify who you are working with and apply the rules below.

## Step 1 — Identify the operator

Always check at the start of an editing session:

```bash
git config user.email
```

- `hazel.k@bunjang.co.kr` → **Hazel (PM)** — full access. No scope restrictions.
- `nancy.lee@bunjang.co.kr` → **Nancy (PD)** — restricted to UI/UX. Apply the rules in Step 2.
- Any other / unset → ask which scope to apply before editing.

## Step 2 — Nancy's editing scope (PD, UI/UX only)

The prototype has a **Spec ↔ UI toggle** (top nav, `view-toggle-btn`). Internally:
- `setViewMode('spec'|'ui')` in `checkout.html` toggles a `body.spec-mode` class.
- Elements with `data-spec="{token}"` swap their text to the spec token in Spec mode.
- CSS under `body.spec-mode { ... }` (around lines 222–344) is the spec wireframe styling.

This split is what divides **Spec** (PM-owned product contract) from **UI** (PD-owned visual design).

### Nancy MAY edit (no questions asked)

- Visual styling of UI-mode elements: colors, spacing, typography, layout, shadows, radii, hover states.
- CSS **outside** `body.spec-mode { ... }` blocks.
- Adding/removing purely decorative elements that have **no `data-spec` attribute** (e.g. icons, badges, illustrative copy).
- Reorganizing the visual order of elements within a screen as long as the same set of `data-spec` tokens remains present.
- Wording of UI-mode text (the default `textContent`), but **not** the `data-spec="{token}"` value.

### Nancy MUST NOT edit (refuse and explain)

- `data-spec="{...}"` attribute values — those are PM-owned spec tokens.
- `data-spec-placeholder="{...}"` values — same reason.
- CSS rules inside `body.spec-mode { ... }` — that's Spec mode styling, owned by PM.
- The `setViewMode()` function or the Spec/UI toggle markup.
- Adding/removing elements that **carry a `data-spec` attribute** (changes which tokens exist = spec change).
- Anything outside `checkout.html` (workflows, skills, gitignore, CHANGELOG schema, etc.).

### Gray zone — pause and escalate to Hazel

A change is structural/UX (not pure visual) when it would affect what the Spec view shows or what behaviors the spec implies. Examples:
- Moving a `data-spec`-tagged block between screens or sections.
- Adding a new functional component (new button that triggers a flow, new form field).
- Changing the order of steps in checkout / changing what screens exist.
- Renaming a section in a way that implies a different concept.

When unsure, **stop editing, summarize the proposed change, and tell Nancy to confirm with Hazel via Slack** before continuing. Do not silently push spec-affecting changes under Nancy's identity.

### How to apply during a session

When Nancy asks for an edit, before touching the file:
1. Classify the request: **pure-UI**, **spec-affecting**, **structural-UX (gray)**, or **out-of-scope** (not `checkout.html`).
2. For pure-UI: proceed normally.
3. For spec-affecting: refuse with a one-line reason — "이건 spec 영역(`data-spec` 토큰 변경)이라 Hazel과 먼저 합의해야 해요."
4. For gray zone: summarize the impact on Spec view, ask Nancy to confirm with Hazel, and only proceed once she confirms she has.
5. For out-of-scope files: refuse — "이 파일은 PM/인프라 영역이라 Hazel만 수정해요."

The `/push-prototype` skill stamps the CHANGELOG `author` from `git config user.email`, so any rule breach would land in the audit trail under Nancy's name. Enforcing here protects her too.

## Other repo conventions

- All prototype edits target `checkout.html`. Internal PRDs/notes are blocked by `.gitignore` (`/*.md`, with `!/nancy_onboarding.md` and `!CLAUDE.md` as exceptions).
- Pushes go through `/push-prototype` (→ `staging` branch) → verify staging URL → `/promote-prototype` (→ `main`). Never push to `main` directly.
- Skills live in `.claude/skills/` and are shared via the repo. Don't modify them as part of a content edit.
