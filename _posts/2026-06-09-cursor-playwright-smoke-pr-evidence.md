---
layout: post
title: "스킬은 외우지 않는다 — Cursor 자연어로 Playwright smoke PR을 만든 방법"
date: 2026-06-09 10:54:00 +0900
permalink: /2026-06-08-cursor-playwright-smoke-pr-evidence/
tags: [cursor, e2e, playwright, skills, workflow, qa, frontend, verification, agent-ops]
series: [skill-is-code, browser-mcp]
---

> Cursor 채팅에는 자연어로 말한다.  
> 실제로 돌아가는 건 **미리 쌓아 둔 스킬·명령·가드레일**이다.

공유 네비 패키지를 옮긴 뒤, 배포 QA에서 **Header·Footer가 살아 있는지** 보여줘야 했다. Playwright smoke를 **E2E 처리 PR**로 올렸고, 리뷰어가 스스로 확인할 수 있게 PR 댓글에 **결과 표**와 **화면 캡처**를 붙였다.

이 글은 당일 실황 회고가 아니다. **어떤 스킬을 준비했는지**, **무엇으로 스펙을 만들었는지**, **채팅 한 줄이 어떻게 그 스킬을 불렀는지**를 정리한다. `/cb:*` 이름을 외울 필요는 없다. **스킬을 알맞게 두고, 말로 작업을 이어가는 것**이 목표다.

[앞글 — 로컬 vs 배포 QA, Browser MCP E2E의 판단](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/)은 “**어디서** 검증할지”였다. 이번 글은 “**배포 QA에서 smoke를 어떻게 만들고, PR에 사람이 읽을 증빙을 어떻게 붙였는지**”다.

## TL;DR

- `8 passed` 한 줄만으로는 리뷰어가 “무엇을 봤는지” 알기 어렵다.
- Playwright smoke + PR **케이스×viewport 표** + **스크린샷** — git에는 spec만, PNG는 release asset.
- 준비물: [`e2e-design`](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-design.md), [`work-start`](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-start.md) / [`work-closeout`](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-closeout.md), [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/) 루프.
- 1차: Modern nav — **8 passed / 6 skipped** (~9s). 2차: Legacy 8앱 — 헬퍼 추출 후 **24 passed / 8 skipped** (~13s).
- 자연어 → 규칙·`/cb:` → `commands/cb/*.md` → `SKILL.md` → `gh`·Playwright.

---

## 1. “통과”만으로는 부족했다

monorepo에서 공유 패키지를 옮기면, diff만으로 “화면 괜찮다”고 말하기 어렵다. Header·Footer는 **앱마다 소비 경로가 다르고**, QA에는 **아직 변경 브랜치가 안 올라온 빌드**가 돌아갈 수 있다.

리뷰어에게 필요한 건 세 가지였다.

| 필요 | 이유 |
|------|------|
| **재실행 명령** | “믿어 주세요”가 아니라 `pnpm test:e2e -- …` 한 줄 |
| **케이스별 결과** | desktop/mobile, 앱별 PASS·skip |
| **눈으로 보는 증거** | selector·레이아웃이 맞는지 |

그래서 CI gate를 새로 달기보다, **로컬·수동 smoke** + **PR 증빙 패턴**을 택했다. 기존 PDP QA spec과 같다 — gate 없음, 담당자가 돌린다.

---

## 2. E2E 처리 PR — 무엇을 올렸나

내부 monorepo에 올린 E2E smoke PR만 요약한다. 번호·티켓 키는 생략하고, **1차 → 2차** 순서로 적었다.

### 2.1 1차 — Modern nav smoke

| 항목 | 내용 |
|------|------|
| 파일 | `e2e/home/nav/tests/01-modern-nav-smoke.spec.ts` |
| 대상 | QA staging 홈 — GNB, Desktop Footer, preuser/category QR, Mobile TabBar, 티켓 앱 Footer |
| 포인트 | 미배포 QA에서도 되는 **fallback selector** (`testId` OR `getByRole('link', …)`) |
| 결과 | **8 passed / 6 skipped** |
| 스킵 | Footer·QR은 desktop, TabBar·preuser 목록은 mobile — viewport 분기 |

### 2.2 2차 — Legacy 8앱 (1차 패턴 재사용)

| 항목 | 내용 |
|------|------|
| 파일 | `e2e/_shared/nav-smoke.helpers.ts`, `e2e/legacy-nav/tests/02-legacy-nav-smoke.spec.ts` |
| 대상 | Pages Router **8앱** × Header + Desktop Footer |
| 포인트 | 1차에서 헬퍼·캡처 **추출** → `for (const app of APPS)` 반복 |
| URL | 상대 경로(baseURL) + auth·ticket **서브도메인 절대 URL** |
| 결과 | **24 passed / 8 skipped** (mobile Desktop Footer 8건 skip) |

plan에 적힌 `data-testid="global-nav-bar"`는 Legacy GNB에 없었다. **`getByRole('link', …)` fallback**을 썼다. 문서 selector보다 **QA DOM**이 우선이다.

---

## 3. 기술 스택

### 3.1 Playwright

```bash
pnpm test:e2e -- e2e/home/nav/tests/01-modern-nav-smoke.spec.ts
pnpm test:e2e -- e2e/legacy-nav/tests/02-legacy-nav-smoke.spec.ts
```

- `@playwright/test` — `test.describe`, `test.skip`, `expect`
- 프로젝트: `chromium-desktop`, `webkit-mobile` (스크린샷 파일명 suffix)
- baseURL: QA staging — 대부분 `page.goto('/')`
- auth·ticket: `https://qa-auth.example.com/login` 같은 **절대 URL**

### 3.2 Resilient selector

```typescript
const header = page
  .getByTestId('gnb-content-container')
  .or(page.getByRole('link', { name: '브랜드명' }).first());
await expect(header).toBeVisible({ timeout: 30_000 });
```

Footer도 같다. 컴포넌트 이름은 앱마다 다르지만, **개인정보·고객센터 링크**처럼 잘 안 바뀌는 텍스트로 OR 체인을 만든다.

### 3.3 Viewport별 skip

```typescript
test('Desktop Footer', async ({ page }) => {
  test.skip(isMobileViewport(page), 'Footer smoke는 Desktop viewport');
  // ...
});
```

desktop·mobile을 한 파일에 돌리면 **의도된 skip**이 늘어난다. PR 표에 skip을 **실패가 아닌 범위**로 적어 두는 이유다.

### 3.4 스크린샷 — 캡처는 하고, git에는 안 넣음

실행 시 `tests/e2e/screenshots/`에 PNG를 남기고, 경로는 **gitignore**. PR diff에는 spec·helpers만.

리뷰용 이미지는 **draft GitHub Release asset** + PR 코멘트 링크. 1차 PR에서 PNG를 잠깐 커밋했다가 **revert**했다. “Files changed에서 보기”와 “repo에 바이너리 안 남기기” 사이에서, 최종은 **release + 로컬 폴더 + PR 코멘트** 세 겹이다.

### 3.5 공통 헬퍼 (2차)

`nav-smoke.helpers.ts`에 `isMobileViewport`, `captureScreenshot`, `waitForLegacyHeader`, `expectDesktopFooter`를 모았다. 2차 spec은 **데이터 배열만** 늘리면 된다.

---

## 4. PR 증빙 — 리뷰어가 읽는 표

### 4.1 1차 (예시)

| 케이스 | desktop | mobile |
|--------|---------|--------|
| 홈 GNB | ✅ | ✅ |
| 홈 Footer | ✅ | skip |
| preuser QR | ✅ | skip |
| category QR | ✅ | skip |
| preuser Mobile 목록 | skip | ✅ |
| 홈 TabBar | skip | ✅ |
| ticket Footer | ✅ | skip |

`8 passed / 6 skipped` + 실행 명령 한 줄이면, skip과 fail을 구분할 수 있다.

### 4.2 2차 (예시)

| 앱 | Header d | Header m | Footer d |
|----|----------|----------|----------|
| order | ✅ | ✅ | ✅ |
| auth | ✅ | ✅ | ✅ |
| … | … | … | … |

8앱 × Header(2 viewport) + Footer(desktop) = **24 passed**, mobile Footer **8 skipped**.

### 4.3 스크린샷

2차는 desktop Header/Footer를 **2×2 표**로 묶어 release URL을 넣었다. 댓글 인라인 업로드 API 제한이 있어 download 링크를 썼다.

**원칙:** 증빙은 PR **대화**에, 소스는 **spec**만.

---

## 5. 준비해 둔 스킬·명령

명령·스킬 **원문**은 [changbaebang/my-cursor](https://github.com/changbaebang/my-cursor)에 있다. 회사 호스트·티켓 키는 **익명화**했고, 로컬 `~/.cursor`는 그대로 쓴다. ([SANITIZATION.md](https://github.com/changbaebang/my-cursor/blob/main/SANITIZATION.md))

### 5.1 이번에 거친 순서

```text
[자연어] "nav QA smoke playwright로, PR에 표랑 캡처도"
    ├─► work-start      브랜치·범위
    ├─► e2e-design      페이지·스킵·QA URL (플랜 md)
    ├─► (에이전트)      spec·helpers 작성
    ├─► pnpm test:e2e   실행·스크린샷
    ├─► gh pr comment   표·release 링크
    └─► work-closeout   PR Test plan·마무리
```

Browser MCP ([`e2e-local-page`](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-local-page.md) · [`e2e-deployed-check`](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-deployed-check.md))는 **클릭·reload·Interaction log**가 필요한 회귀용이다. 이번 smoke는 **visible·역할·텍스트**면 충분해서 Playwright가 맞았다. 환경 선택은 [앞글 §2](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/) 표를 따랐다.

### 5.2 e2e-design — 실행 전 범위 고정

- **커맨드:** [e2e-design.md](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-design.md)
- **스킬:** [e2e-test-design/SKILL.md](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-test-design/SKILL.md)

PR diff → P0 surface → 스킵(S*) → QA preflight(200/404·sample id). smoke에서는 SC-* 클릭 시나리오 대신 **GNB·Footer·QR·TabBar**와 8앱 진입 URL만 골랐다. mypage·Webview·로그인 QR은 **1차 범위 밖**으로 플랜에 적어 두었다.

### 5.3 work-start · work-closeout

| | 원문 | 이번 |
|---|------|------|
| 시작 | [work-start](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-start.md) | 브랜치·grep |
| 마감 | [work-closeout](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-closeout.md) | `test:e2e` 결과를 Test plan에, unpushed 없음 |

이름을 외울 필요는 없다. “브랜치 만들고 범위 확인해줘”면 에이전트가 해당 md를 읽는다.

### 5.4 패턴이 스킬이 되는 순간

nav smoke용 별도 SKILL.md는 없었다. 1차 spec이 **템플릿**, 2차에서 helpers로 **추출**했다. [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)에서 말한 것처럼, **이어지는 PR**이 스킬 파일이 된다.

---

## 6. 자연어가 스킬을 부르는 방식

`/cb:e2e-design`을 매번 치지 않았다. 채팅에는 이렇게만 말했다.

> 배포된 QA에서 nav smoke playwright 추가해줘. 리뷰어가 볼 표랑 스크린샷도 PR에 남겨줘.

```mermaid
flowchart TD
  U[자연어] --> R[Rules + 맥락]
  R --> C{/cb: 또는 스킬 이름?}
  C -->|있음| CMD[commands/cb/*.md]
  C -->|없음| description 매칭
  CMD --> SK[SKILL.md]
  SK --> T[gh · Shell · Playwright]
  T --> O[spec · PR 코멘트]
```

| 층 | 역할 |
|----|------|
| 채팅 | 오늘 뭘 할지 |
| `commands/cb/*.md` | 짧은 런북 |
| `skills/*/SKILL.md` | Guardrails·템플릿 |
| repo `e2e/`, PR | 산출물 |

`disable-model-invocation: true`는 에이전트가 **임의로** E2E 스킬을 끼어들지 않게 한다. “배포 QA에서 E2E해줘”라고 하면 description을 보고 Browser MCP vs Playwright를 고르거나, 이번처럼 smoke spec을 만든다.

**외우지 않는다:** `/cb:` 이름 20개. **준비한다:** 설계·로컬·배포·시작·마감 **역할**, 플랜 md, 스크린샷·release **패턴**, 실패 한 줄 → SKILL diff.

---

## 7. Browser MCP vs Playwright smoke

| | Browser MCP | Playwright smoke |
|---|-------------|------------------|
| 증명 | 클릭·reload·API log | visible·역할·캡처 |
| 환경 | 로컬 dev 또는 QA 탭 | QA baseURL (로컬에서 실행) |
| 로그인 | 수동 인계 | 이번 범위는 비로그인 |
| 리뷰 | 설명이 길어지기 쉬움 | **표 + 캡처** |

둘 다 E2E지만 **oracle**이 다르다. 한 PR에 섞지 않았다.

---

## 8. spec 발췌와 flake 줄이기

```typescript
test('홈 GNB 노출', async ({ page }, testInfo) => {
  await page.goto('/');
  await waitForGnb(page);
  await capture(page, '01-home-gnb', testInfo.project.name);
});

test('홈 Desktop Footer', async ({ page }, testInfo) => {
  test.skip(isMobileViewport(page), 'Footer smoke는 Desktop');
  await page.goto('/');
  await waitForGnb(page);
  await expectDesktopFooter(page);
  await capture(page, '02-home-footer-desktop', testInfo.project.name);
});
```

- Footer 아코디언·TabBar negative assert는 1차 제외
- `waitUntil: 'domcontentloaded'` — networkidle 대기 시간 폭증 방지
- timeout 30s — QA cold start 여유

---

## 9. 다음에도 쓸 기록

| 상황 | 남긴 문장 |
|------|-----------|
| 미배포 QA | `gnb-content-container` 없음 → role fallback |
| plan testId | Legacy에 없음 → `getByRole` |
| PNG 커밋 | revert → release asset |
| 댓글 인라인 이미지 | API 제한 → release URL |
| Mobile Footer | Desktop 전용 → `test.skip` + 표에 명시 |

스킬 원문(Guardrails·Interaction log·플랜 템플릿)은 [my-cursor](https://github.com/changbaebang/my-cursor) — [e2e-local-page-check](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-local-page-check/SKILL.md), [e2e-deployed-browser-check](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-deployed-browser-check/SKILL.md), [e2e-test-design](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-test-design/SKILL.md).

---

## 10. 가져갈 체크리스트

**설계:** PR diff → surface 목록 · QA preflight · 로그인·Webview는 범위 밖 명시

**구현:** `getByTestId().or(getByRole())` · viewport skip을 표에 · 스크린샷 gitignore · 이어지는 PR은 helpers 먼저

**PR:** 실행 명령 · passed/skipped · 케이스 표 · PNG는 release · spec만 merge

**스킬:** oracle 먼저 고르기 · `disable-model-invocation` · 실패 한 줄은 SKILL에 ([skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/))

---

## 마치며

스킬 이름을 외운 게 아니다. **무엇을 증명할지**, **리뷰어가 어떻게 알아볼지**, **Browser MCP와 Playwright 중 무엇을 쓸지**를 미리 쌓아 두고, 채팅에서는 “smoke 추가하고 PR에 표랑 캡처 남겨줘”만 이어갔다.

1차가 **템플릿**, 2차가 **추출**, PR 댓글이 **사람용 UI**였다.

**한 줄 결:** *외워야 하는 건 명령어가 아니라, 자연어가 붙을 **스킬·패턴·증빙 형식**이다.*

### “일정 도구 있으세요?”

이 글을 다듬던 날, 동료가 물었다. **에픽 일정이 거의 실시간으로 갱신되는데, 따로 쓰는 도구가 있냐**고. 답은 단순하다. **별도 대시보드나 플러그인은 없다.**

하는 일을 잘게 나누어 **스킬·명령**으로 쌓아 두고, 일하는 방식이 바뀔 때마다 그걸 고친다. 그다음 채팅에 “마일스톤 끝나는 시점에 일정 체크해줘”처럼 말한다. 빠르고 디테일하게 보인 건, AI가 혼자 잘해서라기보다 **미리 정해 둔 분해·체크 항목·가드레일**이 붙은 결과에 가깝다.

Cursor나 Claude는 **은탄환이 아니다**. 무엇을 언제 확인할지, PR에 무엇을 남길지, smoke 범위를 어디까지 둘지 — **그 판단과 우선순위는 여전히 사람 쪽**에 있다. AI는 스킬을 실행하고 초안을 채우는 쪽에 가깝고, 스킬 내용을 고치는 것도 결국 내가 한다. [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)에서 말한 루프가 그대로다.

---

## 읽을 거리

### 이 블로그

- [로컬 vs 배포 QA — Browser MCP E2E](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/)
- [AI 스킬은 코드다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)
- [my-cursor 아키텍처 회고](https://changbaebang.github.io/2026-05-20-my-cursor-architecture-retro/)

### 명령·스킬 원문 ([my-cursor](https://github.com/changbaebang/my-cursor))

아래 링크는 **발행 시점 스냅샷**이다. 실제 `~/.cursor`와 public mirror는 작업이 쌓일 때마다 바뀐다 — 실패 한 줄, 체크리스트 한 줄, 가드레일 한 줄이 그때그때 추가·수정된다. 고정된 제품이 아니라 **계속 다듬는 작업 노트**에 가깝다.

| 구분 | 링크 |
|------|------|
| E2E 설계 | [e2e-design](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-design.md) · [e2e-test-design](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-test-design/SKILL.md) |
| 로컬 Browser MCP | [e2e-local-page](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-local-page.md) · [e2e-local-page-check](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-local-page-check/SKILL.md) |
| 배포 Browser MCP | [e2e-deployed-check](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/e2e-deployed-check.md) · [e2e-deployed-browser-check](https://github.com/changbaebang/my-cursor/blob/main/skills/e2e-deployed-browser-check/SKILL.md) |
| 작업 시작·마감 | [work-start](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-start.md) · [work-closeout](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/work-closeout.md) |
| 익명화 안내 | [SANITIZATION.md](https://github.com/changbaebang/my-cursor/blob/main/SANITIZATION.md) |

### Playwright

- [Best Practices](https://playwright.dev/docs/best-practices)
- [Test projects](https://playwright.dev/docs/test-projects)
