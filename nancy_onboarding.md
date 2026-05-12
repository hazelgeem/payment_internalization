---
name: Nancy 온보딩 — checkout prototype 공동 편집 가이드 (Claude Code 워크플로)
description: Nancy가 로컬에 repo를 clone하고 Claude Code로 checkout prototype을 편집·푸시하는 단계별 가이드
type: onboarding
last_updated: 2026-05-08
status: active
---

## 안녕하세요 Nancy!

이 문서는 Nancy가 [checkout prototype](https://hazelgeem.github.io/payment_internalization/)을 직접 편집·반영하기 위한 가이드예요. **Claude Code로 로컬에서 편집 → staging URL에서 검증 → 프로덕션 반영** 흐름입니다.

워크플로 한 줄 요약:
**Claude Code에서 `checkout.html` 편집 → `/push-prototype` (staging 배포) → staging URL 확인 → `/promote-prototype` (프로덕션 반영)**

두 개의 URL이 있어요:
- **Staging (검증용)**: https://hazelgeem.github.io/payment_internalization/staging/checkout.html
- **Production (라이브)**: https://hazelgeem.github.io/payment_internalization/checkout.html

`/push-prototype`은 staging만 갱신해요. staging에서 깨진 게 발견되면 라이브엔 영향 X, 추가로 수정한 뒤 다시 push해서 재확인하면 됩니다. 확인 끝나면 `/promote-prototype`으로 한 번에 라이브 반영.

(중요한 변경은 미리 Hazel과 슬랙으로 한번 합의하고 진행하는 걸 권장해요. 형상관리는 GitHub이 다 해주니까 잘못해도 되돌릴 수 있어요.)

---

## 1. 사전 준비 (1회만)

### 1-1. GitHub 계정 + Hazel 초대 수락

1. GitHub 계정이 없다면 https://github.com/signup 에서 가입. 이메일은 **`nancy.lee@bunjang.co.kr`** 사용.
2. Hazel이 Collaborator 초대를 보내면, GitHub 가입 이메일로 초대 메일 도착 → "View invitation" → "Accept invitation".
3. 또는 https://github.com/hazelgeem/payment_internalization 직접 방문해서 상단의 노란색 배너에서 수락.

### 1-2. 도구 설치

Mac 기준으로 다음 3가지가 필요해요. 이미 있는 건 건너뛰면 됩니다.

**1) Claude Code** — 이미 사용 중이면 OK.

**2) Git** — 터미널 열고 `git --version` 쳐봤을 때 버전 번호가 뜨면 이미 설치된 것.
   - 안 뜨면 `xcode-select --install` 실행 → 안내 따라 설치.

**3) GitHub CLI (`gh`)** — Hazel 계정 인증용은 아니지만, clone을 HTTPS로 편하게 하려면 필요.
   - 설치: `brew install gh` (Homebrew 없으면 https://brew.sh 먼저 설치)
   - 인증: 터미널에서 `gh auth login` → "GitHub.com" → "HTTPS" → "Login with a web browser" → 브라우저에서 본인 GitHub 계정으로 로그인.

### 1-3. Repo 클론 (한 번만)

본인이 작업할 폴더 위치를 정하고 (예: `~/Documents/work/`), 거기서 터미널을 열어 다음 실행:

```bash
cd ~/Documents/work
gh repo clone hazelgeem/payment_internalization
cd payment_internalization
```

(`gh repo clone` 대신 `git clone https://github.com/hazelgeem/payment_internalization.git` 도 동일하게 동작합니다.)

### 1-4. Git 작성자 정보 등록

이 repo에서만 본인 정보를 쓸 수 있도록:

```bash
git config user.email "nancy.lee@bunjang.co.kr"
git config user.name "Nancy Lee"
```

이 설정은 `/push-prototype` 스킬이 CHANGELOG의 `author` 필드를 자동 기입하는 데 사용돼요.

---

## 2. 편집 흐름 (매번 반복)

### 2-1. Claude Code 열고 repo 디렉터리로 이동

```bash
cd ~/Documents/work/payment_internalization
claude
```

(Claude Code가 어떤 식으로 열리든, 작업 디렉터리가 `payment_internalization`이면 OK.)

### 2-2. 최신 상태로 동기화

본격적으로 편집하기 전에:

```bash
git pull origin main
```

Hazel이 푸시한 변경이 있다면 그걸 먼저 받아요. 충돌이 안 나면 그냥 OK.

### 2-3. checkout.html 편집

Claude Code에 자연어로 요청하면 됩니다. 예시:
- "checkout.html에서 결제 버튼을 카드 영역 밖으로 빼줘"
- "주문서 헤더 폰트를 16px로 키워줘"
- "체크아웃 우측 패널의 Scenarios chips에 'Coupon applied' 시나리오 추가"

**큰 변경은 슬랙으로 Hazel과 먼저 합의** 후 진행하는 걸 권장해요. (정책/스펙 변경, 새 컴포넌트, 결제 수단 변경 등)

### 2-4. Staging에 먼저 배포

Claude Code 안에서 그대로:

```
/push-prototype
```

실행하면 스킬이 자동으로:
1. `staging` 브랜치로 이동 + 원격 동기화 체크 (필요시 `git pull --rebase`)
2. 버전 자동 bump (`v1.14` → `v1.15`)
3. 변경 내용 정리해서 CHANGELOG 항목 초안 보여줌 — 작성자는 `nancy.lee@bunjang.co.kr` 자동 기입
4. **확인 받으면** 그때 commit + push (staging 브랜치로)
5. 완료 후 staging URL 안내

### 2-5. Staging URL에서 검증

1-2분 후 다음 URL 열기:
**https://hazelgeem.github.io/payment_internalization/staging/checkout.html**

확인 포인트:
- 의도한 변경이 보이는지
- 디자인 시스템(폰트, 색상, 컴포넌트)이 안 깨졌는지
- 다른 영역이 의도치 않게 영향받지 않았는지

이상하면 다시 코드 수정 후 `/push-prototype` 재실행 → 같은 URL에서 재확인. 프로덕션엔 영향 X.

**큰 변경이면 슬랙으로 Hazel에게 "staging 한번 봐줘" 가볍게 ping** 권장.

### 2-6. 프로덕션 반영

검증 OK면 Claude Code에서:

```
/promote-prototype
```

실행하면 스킬이:
1. staging의 commit들을 main으로 가져옴 (fast-forward merge)
2. promote 전에 어떤 commit이 들어가는지 미리 보여주고 확인 받음
3. 확인 후 push → 1-2분 후 프로덕션 URL에 반영
4. 완료 후 프로덕션 URL 안내

이때 라이브 사이트 https://hazelgeem.github.io/payment_internalization/checkout.html 에서 본인 변경 확인 + CHANGELOG 모달에서 이메일 노출 확인.

---

## 2-7. Nancy의 편집 범위 (꼭 읽어주세요)

프로토타입에는 상단에 **Spec / UI 토글**이 있어요. 같은 화면을 두 가지 모드로 보여주는데:
- **UI 모드** (기본): 실제 사용자가 보는 화면 — 디자인, 색상, 폰트, 레이아웃
- **Spec 모드**: 컴포넌트별 데이터/규격을 표시 — PM이 개발팀과 합의하는 "스펙" 정의

이 둘은 같은 HTML에서 공존해요. **Nancy는 UI(디자인) 영역만 수정**, **Spec 영역은 Hazel(PM)이 소유**. 이유는 Spec이 개발팀과의 계약이라 디자이너가 임의로 못 바꿔야 해서예요.

### Nancy가 직접 수정해도 되는 것 (UI 영역)
- 색상, 폰트, 여백, 라운드, 그림자, 호버 효과
- UI에 보이는 텍스트(문구) 다듬기
- 장식용 아이콘/배지/이미지 추가·삭제
- 시각적 정렬·순서 조정 (단, "어떤 컴포넌트가 존재하는지" 자체는 그대로 유지)

### Nancy가 직접 수정하면 안 되는 것 (Spec 영역)
- Spec 모드에서만 보이는 텍스트 (코드상으론 `data-spec="..."` 속성)
- 새 기능 버튼/입력 필드 추가·삭제
- 화면 구조 변경 (예: 결제 단계 순서 변경, 새 화면 추가)
- 다른 파일 (`.github/workflows/`, `.claude/skills/`, `.gitignore` 등)

### 애매하면?
"이게 디자인 변경인지 스펙 변경인지 모르겠다" 싶으면 **Claude Code가 알아서 멈추고 알려줘요** (CLAUDE.md 룰 기반). 그때는 슬랙으로 Hazel에게 한 줄 ping → 합의 후 진행.

큰 변경(새 컴포넌트, 결제 흐름 변경 등)은 Spec 영역 영향이 있을 가능성이 높으니 **무조건 Hazel과 슬랙으로 먼저 합의** 권장.

---

## 3. CHANGELOG 작성 규칙 (스킬이 자동으로 챙겨주지만 참고)

스킬이 작성한 초안을 검토할 때 다음 기준으로 수정·승인해주세요:

- **항목**: 사용자가 보는 관점에서 **짧은 한국어 한 줄**. 예: `"결제 버튼 위치 조정"`, `"주문서 헤더 폰트 통일"`.
- **제외 대상**: 버그 수정, 작은 UI 다듬기, 변수명 정리, 단순 텍스트 다듬기, 리팩터링 — 이런 건 stakeholder가 안 봐도 되는 내용이라 CHANGELOG에서 빠져요.
- **포함 대상**: 정책/스펙 변경, 신규 기능, 컴포넌트 추가, 결제 수단 변경 등 "유저 또는 운영 관점에서 의미 있는 변화".
- 작성자(`author`), 날짜(`date`), 버전(`version`)은 스킬이 자동으로 채워줌.

---

## 4. 자주 하는 실수 / 주의사항

### 편집 후 push를 까먹음
로컬에서 편집만 하고 `/push-prototype`을 안 돌리면 staging URL에도 반영이 안 돼요. 매번 push까지 해야 끝.

### Staging만 보고 promote를 까먹음
**가장 흔한 실수.** `/push-prototype`은 staging URL만 갱신해요. 라이브 사이트(`/checkout.html`, `/staging/` 없는 URL)에 반영하려면 `/promote-prototype` 별도 실행 필요. staging URL 확인 후 OK면 promote 꼭 하기.

### Pull 먼저 안 하고 편집 시작
스킬이 push 단계에서 자동으로 pull --rebase를 해주긴 하지만, 충돌(conflict)이 나면 자동 해결 안 함. 작업 시작 전에 `git pull origin main` 한 번 돌리는 습관을 들이면 충돌 가능성이 줄어요.

### 큰 변경을 합의 없이 push
정책/스펙 변경 (예: 약관 동의 항목 추가), 새 컴포넌트/페이지 추가, 결제 수단 변경 같은 건 Hazel과 슬랙으로 먼저 합의 후 진행하는 게 좋아요.

### Conflict가 발생했을 때
스킬이 `rebase --abort` 하고 멈춤. → Hazel에게 슬랙으로 ping. 같이 해결.

---

## 5. 만약 뭔가 잘못된 것 같으면

- **편집했는데 commit 전에 되돌리고 싶을 때**: `git checkout -- checkout.html` (이러면 마지막 commit 상태로 복귀)
- **이미 push했는데 잘못된 걸 발견했을 때**: 다시 편집해서 새 push로 수정. 더 큰 문제면 Hazel에게 ping → revert 가능 (GitHub에 모든 이력 남음).
- **헷갈리거나 막히면**: Hazel 한테 슬랙으로 ping 🙏

---

## 6. 빠른 참고 링크

- Repo: https://github.com/hazelgeem/payment_internalization
- **Staging URL** (검증용): https://hazelgeem.github.io/payment_internalization/staging/checkout.html
- **Production URL** (라이브): https://hazelgeem.github.io/payment_internalization/checkout.html
- 내가 만든 staging commit 목록: https://github.com/hazelgeem/payment_internalization/commits/staging?author=nancylee-bunjang
- 내가 만든 main commit 목록: https://github.com/hazelgeem/payment_internalization/commits/main?author=nancylee-bunjang
- 스킬 정의 (참고용, 수정할 일은 거의 없음): `.claude/skills/push-prototype/SKILL.md`, `.claude/skills/promote-prototype/SKILL.md`
