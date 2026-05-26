---
layout: post
title: "로컬에서 확인할 것, QA에서 확인할 것 — Browser MCP E2E의 판단"
date: 2026-05-26 18:10:00 +0900
permalink: /2026-05-26-e2e-local-vs-deployed-browser-mcp/
tags: [ai, e2e, cursor, mcp, browser, qa, workflow, frontend, verification]
series: [problem-first, skill-is-code, browser-mcp]
---

> Browser MCP의 가치는 클릭 자동화가 아니다.  
> **어느 환경에서 무엇을 증명할지** 고르는 데 있다.

최근 글들은 AI 균형, 문제 정의, 하네스, 검증 부재 쪽이었다.  
이번 주제는 결이 다르다. **로컬 E2E**와 **배포 QA E2E** 중 어디서 돌릴지를 먼저 정하고, Cursor **Browser MCP**로 플랜을 따라가는 실무다.

같은 PR, 같은 기능이라도 **깨지는 이유**가 환경마다 다르다. 그걸 한 루프에 섞으면 “에이전트가 이상하다”가 아니라 **판단이 흐려진다**.

## TL;DR

- 로컬 E2E와 배포 QA E2E는 **같은 테스트가 아니다** — 증명하는 것이 다르다.
- 로컬: 브랜치 코드·UI·네트워크·빠른 회귀. QA: 로그인·통합·실데이터·배포 SHA.
- Browser MCP는 **실행 수단**이고, 스킬/커맨드 이름은 **환경 경계**를 박아 둔다.
- API 200만으로 PASS 하지 않는다 — 클릭·reload·Interaction log까지 닫아야 한다.
- 시리즈: [문제 퍼스트](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/) → **이 글(어디서)** → [테스트·CI](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)

---

## 1) 한 PR, 두 개의 세계

에픽 하나를 마무리할 때 흔한 그림이다.

| | 로컬 dev (HTTPS + 인증서) | 배포 QA (스테이징 / 개발 배포) |
|---|---------------------------|----------------------|
| **코드** | 내가 checkout한 브랜치 | 방금 배포된 SHA |
| **데이터** | QA API를 치지만 샘플이 비거나 `id=1`이 404인 경우 많음 | 팀이 쓰는 실제 샘플 URL·목록 |
| **인증** | 로컬 호스트와 QA 호스트 **쿠키가 안 맞을 수 있음** | SSO·2FA·세션 그대로 |
| **속도** | 페이지 하나씩 빠르게 | 배포·로그인 대기, 시나리오 단위 |
| **적합한 질문** | “내 변경이 UI/API를 깨뜨렸나?” | “배포본이 사용자 경로를 통과하나?” |

[skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)에서 쿠폰 컬렉션 `id=1`이 404였던 이야기는 **배포 QA 쪽 oracle** 이슈다.  
로컬에서 페이지가 뜬다고 해서 QA 샘플 ID가 맞다는 뜻은 아니다.

반대로 QA에서 PASS여도, **아직 merge 안 된 브랜치** 버그는 로컬에서만 잡힌다.

---

## 2) 판단 트리 — 먼저 환경, 그다음 브라우저

도구를 고르기 전에 환경을 고른다. [문제 퍼스트](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)의 “성공 기준”이 여기서 **어느 URL·어떤 계정·어떤 샘플 ID**인지로 내려온다.

```text
검증할 항목이 정해졌나? (플랜·Checks·Login 열)
  └─ No  → 플랜부터. Browser MCP는 그다음.
  └─ Yes
       ├─ 아직 배포 전 / 내 브랜치 코드만 보면 됨
       │     → 로컬: 앱 1개만, 페이지 1개씩
       ├─ 배포 완료 + 사용자 로그인 경로 + 통합 플로우
       │     → 배포 QA: 시나리오(SC-*) 순차
       ├─ 로컬 목록 0건·샘플 ID 없음
       │     → QA에서 샘플 확보 후, 필요하면 로컬 재시도
       └─ 로그인·2FA·캡차
             → 사람이 먼저. 에이전트는 그다음 클릭 루프
```

**로컬을 택할 때**

- dev 서버 하나만 띄운다 (포트 충돌·잘못된 앱 방지).
- 플랜의 `Login` 열에 맞게 **매 페이지마다** 로그인 상태를 기록한다.
- `Checks`에 있는 클릭을 **실제로** 한다. 스냅샷만 보고 PASS 금지.

**배포 QA를 택할 때**

- 배포 SHA·환경·시각을 먼저 남긴다. 배포 전 실행은 시간 낭비다.
- 시나리오는 `시작 URL → 액션 → 기대 상태` 한 덩어리씩.
- blocker면 같은 클릭만 반복하지 말고 스냅샷·네트워크·콘솔을 남긴다.

---

## 3) Browser MCP — “미래”가 아니라 오늘의 루프

[브라우저 MCP가 오면](https://changbaebang.github.io/2026-05-22-browser-mcp-future-web-dev/) 글은 인터페이스와 패턴을 봤다.  
이 글에서 말하는 Browser MCP는 **Cursor IDE에 붙은 브라우저 자동화** — `navigate` → `snapshot` → `click` → `network_requests` → 필요 시 reload.

[Chrome DevTools MCP](https://developer.chrome.com/blog/chrome-devtools-mcp)처럼 “에이전트가 Chrome을 직접 디버깅하는” 축과 겹치지만, 여기서는 **제품 플로우를 플랜대로 닫는** 용도에 가깝다.

전형적인 한 페이지 루프:

1. 대상 URL 진입 (로컬이면 `dev-host:port`, QA면 배포 URL)
2. 스냅샷으로 클릭 대상 ref 확보 (가리면 scroll)
3. 클릭 **한 번씩**, mutation 끝날 때까지 기다림
4. 네트워크에서 API status 확인 (예: 401이면 Login FAIL)
5. **reload 또는 동일 URL 재진입** — 토글 상태가 GET만으로 유지되는지

이 루프는 [geohot · 검증](https://changbaebang.github.io/2026-05-26-geohot-slop-verification-not-agents/)에서 말한 “10배 아웃풋에는 10배 검증”의 **수동+에이전트 층**에 가깝다. CI가 대체하지 못하는 **눈·손·판단** 구간이다.

---

## 4) 사람과 에이전트 역할 나누기

함께 E2E를 돌릴 때 실제로 나뉘는 일이다.

| 역할 | 사람 | 에이전트 (Browser MCP) |
|------|------|-------------------------|
| 로그인·2FA·캡차 | ✅ 수동 | ❌ 멈추고 요청 |
| “이상한데” 감지 | ✅ 맥락·제품 감각 | △ 스냅샷/네트워크 근거 제시 |
| 플랜 따라 페이지·시나리오 순회 | △ 우선순위 | ✅ |
| 클릭·reload·Interaction log | △ 샘플 확인 | ✅ 반복 |
| PASS/FAIL·blocker 요약 | ✅ 최종 판단 | ✅ 초안 |

에이전트 브라우저 세션과 내가 쓰는 Chrome이 **다른 프로필**일 수 있다.  
“내 브라우저에서는 되는데 MCP에서는 로그인 페이지”는 버그가 아니라 **세션 경계** 신호다. 로그인 URL을 플랜에 박아 두는 이유다.

---

## 5) 스킬 이름이 곧 경계

커맨드를 둘로 나눴다. 이름에 환경과 범위가 들어간다. 아래는 **실제 운영 스킬**(`e2e-local-page-check`, `e2e-deployed-browser-check`)에서 뽑은 **발행용 발췌(B)** — 호스트·repo·API 경로만 익명화했다.

| 커맨드 | 스킬 | 의미 |
|--------|------|------|
| `/cb:e2e-local-page` | `e2e-local-page-check` | 로컬·**페이지 1개**·앱 1개 |
| `/cb:e2e-deployed-check` | `e2e-deployed-browser-check` | **배포 후** QA URL·시나리오 흐름 |

- **`local-*`**: 빠른 oracle, 좁은 blast radius
- **`deployed-*`**: 배포·통합·로그인 전제
- description에 **하지 않는 것**을 적어 둔다 (`disable-model-invocation`, “배포 전 금지”, “클릭 없이 PASS 금지”)

이름이 다르면 에이전트가 “로컬인데 QA URL을 연다” 같은 **환경 혼선**이 줄어든다. [AI 스킬은 코드다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)에서 말한 것처럼, 실패 한 줄이 스킬 description에 들어가면 다음 회차 비용이 내려간다.

### front matter (발췌)

로컬:

```yaml
---
name: e2e-local-page-check
description: |
  Single-page local E2E on HTTPS dev host (mkcert). One app only.
  Verify login before each page. Do not PASS without click + reload.
  Use e2e-deployed-browser-check after deploy for SC-* flows.
disable-model-invocation: true
---
```

배포 QA:

```yaml
---
name: e2e-deployed-browser-check
description: |
  Scenario E2E on deployed staging/dev URLs via Cursor browser.
  Run only after deploy is confirmed. Manual login/2FA takeover.
  Do not repeat the same failing click more than 4 times — file a blocker.
disable-model-invocation: true
---
```

`disable-model-invocation`은 “이 스킬은 에이전트가 임의로 끼어들 skill이 아니다”는 신호다. `/cb:e2e-*`로 **명시 호출**할 때만 쓴다.

### Guardrails (발췌)

**로컬 (`e2e-local-page-check`)**

- dev 서버 **앱 1개**만. 페이지는 **한 번에 하나**.
- 플랜 `Login`이 Y인데 guest면 토글 테스트하지 말고 로그인부터.
- 인증서 오류 → mkcert 스킬 후 dev 서버 재기동.
- IDE Browser가 dev 호스트에 `ERR_CONNECTION_REFUSED`면: 로그인은 **배포 QA URL**, 검증은 로컬 포트 — 세션 경계를 플랜에 적어 둔다.
- **클릭 없이 PASS 금지** — snapshot만으로 끝내지 않는다.
- **reload/재진입 없이 PASS 금지** — mutation 200만으로 끝내지 않는다.
- 목록형 페이지: QA `id=1` often 404 → 플랜·협업 채널·배포 URL에서 **실제 샘플 ID** 확보.
- 이 라운드는 **검증·기록만** (코드 수정은 별 PR).

**배포 (`e2e-deployed-browser-check`)**

- **배포 전 실행 금지** — SHA·환경·시각을 먼저 기록.
- Browser MCP는 **배포 URL**에서 (로컬 dev 호스트는 연결 실패가 잦음).
- 동일 실패 클릭 **4회 이상 반복 금지** → blocker + 티켓/PR 코멘트.
- destructive 액션은 사용자 승인 없이 진행하지 않는다.
- 플랜에서 스킵(`S*`) 표시된 시나리오는 실행하지 않는다.

### Interaction log (발췌)

토글(찜)·즐겨찾기 회귀는 스킬에 이렇게 박아 둔다.

| 대상 | 액션 | 기대 UI | 기대 Network |
|------|------|---------|--------------|
| 엔티티 토글 | off → on → **reload** → off | on/off 아이콘 유지·해제 | mutation 200 · reload 시 GET만 |
| 리스트 아이템 토글 | off → on → **reload** → off | 카드·카운트 동기화 | item mutation 200 |
| 플로팅 액션 | 클릭 | on/off | mutation 200 |

- **401** → Login FAIL (클릭 결과 무효)
- **5xx / mutation throw** → FAIL + 스냅샷·응답 요약
- 리스트 0건 → 엔티티 토글만 수행, 플랜에 `item toggle N/A (no items)`

클릭마다 남기는 표:

```markdown
| Step | Action | UI after | API | Result |
|------|--------|----------|-----|--------|
| 1 | entity toggle on | active | POST 200 | PASS |
| 2 | entity toggle off | inactive | DELETE 200 | PASS |
| 3 | reload | state kept | GET 200 | PASS |
```

PASS 조건: **Interaction log 전 step PASS** + Login verified.

### 페이지·시나리오 기록 (발췌)

로컬 한 페이지:

```markdown
### SH-1 /catalog/brand/{id}
- Env: local dev (shop only)
- Login (plan): Y
- Login (verified): logged-in | guest | FAIL (401 on toggle API)
- Result: PASS | FAIL
- Interactions: entity toggle on/off/reload PASS; item toggle N/A (0 items)
```

배포 시나리오 묶음:

```markdown
## Deployed E2E result (staging)
- Build/Commit: …

| Scenario | Login verified | Result | Evidence |
|----------|----------------|--------|----------|
| SC-01 | logged-in | PASS | entity/item toggle + reload OK |

### Blockers
- 없음
```

전문 SKILL.md는 팀 호스트·포트·API prefix가 들어가서 블로그에는 넣지 않았다. 구조와 guardrails만 복사해도 **환경 선택 → Browser MCP 실행 → log로 닫기** 루프는 재현 가능하다.

---

## 6) 실패에서 남긴 규칙 세 가지

**① 샘플 ID는 플랜의 일부다**  
“페이지 URL”만 적어 두면 에이전트는 `id=1`을 택한다. 404는 환경 탓이 아니라 **oracle 미정의**다.

**② API 200 ≠ PASS**  
응답이 200이어도 UI가 안 바뀌거나, reload 후 상태가 풀리면 FAIL. [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)의 두 번째 교훈과 같다.

**③ 로컬과 QA를 한 번에 섞지 않는다**  
로컬에서 고친 뒤 QA에서 시나리오를 닫는 **순서**는 있어도, “로컬 PASS니까 QA 스킵”은 없다. 반대도 마찬가지.

---

## 7) 하네스·CI와의 위치

[Fowler harness](https://changbaebang.github.io/2026-05-25-fowler-harness-roles-not-layers/)로 말하면:

- **입력 하네스**: E2E 플랜, Login/Checks 열, 환경 선택(로컬 vs QA)
- **출력 하네스**: Browser MCP Interaction log, PASS/FAIL 표
- **CI**: merge 이후 자동 회귀 — [테스트·CI](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)가 담당

Browser E2E는 CI를 대체하지 않는다. **배포 전·에픽 마감 전** 사람과 에이전트가 닫는 층이다.

---

## 8) 독자용 체크리스트 (복붙)

```markdown
## E2E 시작 전 2분

- [ ] 검증 목록·성공 기준이 플랜에 있는가?
- [ ] 이번 라운드는 로컬(브랜치)인가, 배포 QA(SHA)인가?
- [ ] Login Y면 로그인 확인 방법·URL이 적혀 있는가?
- [ ] 샘플 ID/목록이 QA에 존재하는가? (id=1 가정 금지)
- [ ] Checks의 클릭·reload까지 Interaction log에 남길 것인가?
- [ ] blocker 시 스냅샷+네트워크를 남기고 같은 클릭만 반복하지 않을 것인가?
```

---

## 마무리

[browser-mcp](https://changbaebang.github.io/2026-05-22-browser-mcp-future-web-dev/) 글이 “무엇이 바뀌는가”였다면, 이 글은 **오늘 어디서 돌리는가**다.

로컬과 QA를 나누면 에이전트는 빨라지는 게 아니라 **덜 헷갈린다**.  
Browser MCP는 만능 자동화가 아니라, **환경을 고른 뒤** 플랜을 닫는 도구다.

다음에 같은 PR을 또 볼 때 첫 질문은 “어떤 모델이냐”가 아니라 이렇게 묻는다.

> **지금 증명하려는 것은 로컬 코드인가, 배포본인가?**

---

## 레퍼런스

- [Chrome DevTools (MCP) for your AI agent](https://developer.chrome.com/blog/chrome-devtools-mcp) — 브라우저·에이전트 검증 축
- [Playwright — Best Practices](https://playwright.dev/docs/best-practices) — 격리·assert·flake (Browser MCP 루프와 대응되는 개념)
- 시리즈: [problem-first](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/) · [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/) · [test-ci](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)
