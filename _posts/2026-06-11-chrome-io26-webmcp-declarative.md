---
layout: post
title: "WebMCP와 선언적 API — 에이전트 시대에 사이트가 말해야 할 것"
date: 2026-06-11 11:24:00 +0900
permalink: /2026-06-10-chrome-io26-webmcp-declarative/
tags: [chrome, ai, webmcp, frontend, web-platform, declarative]
series: [chrome-io26, agent-interface]
---

> [Chrome at I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)에서 WebMCP가 오리진 트라이얼(Chrome 149)로 올라왔다. Gemini in Chrome도 WebMCP API 지원을 예고했다.  
> 이 글은 I/O 실황 회고가 아니다. **WebMCP**와 **선언적(Declarative) API**, 그리고 **Declarative Partial Updates**와의 관계만 정리한다.

[에이전트 인터페이스 체크리스트](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/)가 **호출 계약**을 다뤘다면, 이번 글은 **사이트가 도구를 어떻게 말하는지**다. [Browser MCP E2E](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/)는 **오늘 쓰는 검증 루프**, [HTML-in-Canvas·WebView](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)는 capability를 **어디서 볼지**, 검색·커머스 2편(초안)은 **발견·결제 표면**을 다룬다.

## TL;DR

- WebMCP는 에이전트 **actuation(추측 클릭)** 대신 사이트가 **도구(contract)** 를 브라우저에 등록하게 한다.
- **Declarative API**는 HTML 폼에 `toolname` 등을 얹는 방식 — 에이전트 전용 화면이 아니라 **사람 UI + 점진적 향상**.
- **Declarative Partial Updates**는 이름만 비슷하고 **층이 다르다** — 행동 계약 vs 문서 패칭.
- 제한: **탭·WebView 필수**, **방문한 뒤에만** 도구 발견, **origin-isolated** 문서.

---

## 1. 에이전틱 웹과 “사이트 쪽 MCP”

I/O 2026은 [agentic web](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)이라는 말로 세 갈래를 묶는다.

1. **에이전트가 사이트와 일한다** — WebMCP, Modern Web Guidance, DevTools for agents  
2. **플랫폼이 UI·성능을 선언적으로 맡긴다** — HTML-in-Canvas, Declarative Partial Updates  
3. **브라우저가 사용자를 돕는다** — Gemini in Chrome, auto browse  

“MCP”라는 이름 때문에 헷갈리기 쉽다.

| 이름 | 무엇인가 |
|------|----------|
| **Model Context Protocol (MCP)** | 에이전트 런타임이 **외부 도구·데이터**에 붙는 **프로토콜** (서버·IDE·로컬) |
| **WebMCP** | **웹 페이지**가 브라우저·에이전트에게 **호출 가능한 도구**를 노출하는 **웹 표준 제안** |
| **UCP의 MCP 언급** | 커머스 프로토콜이 MCP 등과 **연동 가능**하다는 뜻 — WebMCP와 **동의어가 아님** |

WebMCP는 “브라우저 탭 안 사이트”가 자기 행동을 구조화해 주는 층이다. Cursor가 쓰는 MCP 서버와 **짝**이 되는 그림이지만, **구현 위치가 다르다**.

---

## 2. Actuation이 왜 부족한가

[WebMCP 소개(ko)](https://developer.chrome.com/docs/ai/webmcp?hl=ko)는 **Actuation**을 이렇게 정의한다.

> 에이전트가 사람 사용자처럼 마우스 클릭·텍스트 입력을 **시뮬레이션**하는 것.

한 번의 클릭이면 단순하다. **다단계 플로우**에서 문제가 드러난다.

- “검색”인지 “필터 적용”인지 버튼 라벨만으로는 모호할 수 있다.  
- `firstName` / `fullName` / `displayName` 의미는 DOM만 봐서 **추측**이다.  
- 날짜 피커·커스텀 셀렉트·가상 스크롤은 actuation **실패 지점**이 되기 쉽다.  
- SPA는 로딩·토스트·모달마다 에이전트 해석이 달라진다.

I/O 예시도 같다. 멀티시티 여행을 actuation만으로 처리하면 사용자는 폼을 **하나씩 클릭하는 것**을 지켜본다. WebMCP는 백엔드 API를 부르는 **도구**로 줄여 **초 단위**에 가깝게 만들고, 과정은 **사용자가 승인**한다.

WebMCP가 약속하는 축:

- **Discovery** — `checkout`, `filter_results` 같은 도구 등록  
- **JSON Schema** — 입력·출력 형식 명시  
- **State** — “지금 이 맥락에서 무엇이 가능한지” 공유  

도구 실행은 **탭 안에서 보인다**. 브랜드·사람 UI를 유지하면서 자동화를 붙이는 **점진적 향상**(progressive enhancement)에 가깝다.

---

## 3. Imperative API — SPA가 갈 길

Imperative API는 JavaScript로 도구를 등록한다.

| 장점 | 부담 |
|------|------|
| 복잡한 상태 머신을 도구 하나로 **캡슐화** | 도구마다 **페이지 상태와 동기화** |
| 기존 SPA 로직 재사용 | 스키마·에러 메시지를 **에이전트가 읽게** 작성 |
| 민감 동작에 확인·권한 분기 | 테스트 대상이 **UI + contract**로 늘어남 |

커머스 SPA라면 Imperative가 먼저 손이 갈 수 있다.

- `apply_filters` — URL query·facet·API를 한 함수로  
- `add_to_cart` — 옵션·재고·가격 스냅샷 검증 후 실행  
- `start_checkout` — 로그인·배송지 선행 조건을 도구 설명에 명시  

[에이전트 인터페이스 체크리스트](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/)의 Action Boundary·Idempotency·Trust UX는 Imperative에서 특히 중요하다. **도구 하나 = 미니 API**로 보면 된다.

---

## 4. Declarative API — 폼이 곧 도구

Declarative API는 **표준 HTML 폼**에 attribute를 더한다.

- `toolname` — 도구 이름  
- `tooldescription` — 목적 (둘 중 하나라도 없으면 **등록 해제**)  
- `toolparamdescription` — 필드별 스키마 설명 (선택)  
- `toolautosubmit` — 에이전트 호출 시 자동 제출 vs 사용자 Submit  

[공식 예시(ko)](https://developer.chrome.com/docs/ai/webmcp/declarative-api?hl=ko) — 고객 지원 폼:

```html
<form toolname="supportRequestTool" tooldescription="Submit a request for support.">
  <label>First Name <input name="firstName" toolparamdescription="Customer first name"></label>
  <label>Last Name <input name="lastName"></label>
  <select name="select" toolparamdescription="Determines what team this request is routed to.">
    <option value="Customer happiness team">Return my purchase.</option>
    <option value="Distribution team">Check where my package is.</option>
    <option value="Website support team">Get help on the website.</option>
  </select>
  <button type="submit">Submit</button>
</form>
```

브라우저는 JSON Schema가 담긴 tool 정의로 변환한다. 에이전트 호출 시:

1. 폼에 **포커스**  
2. 필드 **자동 채움**  
3. `:tool-form-active` / `:tool-submit-active`로 **시각 피드백**  
4. `toolactivated` / `toolcancel`로 앱 UI 동기화  

제출 경로:

```javascript
document.querySelector("form").addEventListener("submit", (e) => {
  e.preventDefault();
  if (!myFormIsValid()) {
    if (e.agentInvoked) e.respondWith(myFormValidationErrorPromise);
    return;
  }
  if (e.agentInvoked) e.respondWith(Promise.resolve("Search is done!"));
});
```

- `SubmitEvent.agentInvoked` — 에이전트가 건드렸는지  
- `respondWith(Promise)` — 표준 네비게이션 대신 **구조화된 tool output** (`preventDefault` 필수)

**Le Petit Bistro** demo가 Declarative API의 대표 사례다. 문의·예약·단순 검색 폼이 있으면 **attribute부터 PoC**하기 좋다.

### Declarative vs Imperative

| 상황 | 추천 |
|------|------|
| 서버 POST 폼·문의·예약·단순 검색 | **Declarative** |
| 클라이언트 라우팅·장바구니·다단계 checkout | **Imperative** (또는 혼합) |
| 민감 결제 | Declarative + **사용자 Submit 필수** (`toolautosubmit` 끔) |
| 숨겨진 개발자 메뉴 진단 | **Imperative** `run_diagnostics` |

목록 필터는 Imperative, 문의 폼은 Declarative처럼 **행동 단위**마다 나눌 수 있다.

---

## 5. 보안·플랫폼 제약

**Origin isolation** — `document.domain`·`Origin-Agent-Cluster: ?0` 등에서는 WebMCP **비활성**. 레거시 “도메인 맞추기”와 **충돌**할 수 있다.

**Permissions Policy** — `tools` 정책, 기본 `self`. cross-origin iframe은 `allow="tools"` 필요.

**Browsing context** — **JavaScript가 있는 탭·WebView**에서만 호출. headless 크롤러만으로는 불가.

**Discoverability** — 클라이언트는 **사이트를 방문해야** 도구 목록을 안다.

로컬 실험: `chrome://flags/#enable-webmcp-testing`. [Model Context Tool Inspector](https://developer.chrome.com/docs/ai/webmcp?hl=ko)로 자연어 → 도구 매칭·스키마 검증.

---

## 6. “선언적”이라는 말이 두 겹이다

I/O는 WebMCP Declarative와 **Declarative Partial Updates**를 같은 “선언적 플랫폼” 묶음으로 소개한다. [Partial Updates 글](https://changbaebang.github.io/2026-06-01-declarative-partial-updates-fe-notes/)에서 React/RSC와 Chrome “선언적” **층 차이**를 다뤘다. 여기서는 **WebMCP vs Partial Updates**만 맞춘다.

| | WebMCP Declarative | Declarative Partial Updates |
|--|-------------------|----------------------------|
| **선언 대상** | 이 **폼/도구**는 무슨 **행동**인가 | 이 **DOM 구간**은 무엇으로 **채워질** 것인가 |
| **실행 주체** | 브라우저 + 페이지 JS (`respondWith`) | **HTML 파서·패처** |
| **대표 마크업** | `toolname`, `toolparamdescription` | `<?start name="…">`, `<template for="…">` |
| **에이전트** | **직접** — 호출 contract | **간접** — 느린 UI를 빨리 보여 actuation 실패를 줄임 |

[patching explainer](https://github.com/WICG/declarative-partial-updates/blob/main/patching-explainer.md) 요지: HTML 스트리밍은 초기 파싱 후 막히기 쉬웠고, React 등은 out-of-order를 **인라인 script**로 흉내 냈다. 제안은 **선언적 패치** — 마커 구간을 template로 바꾸고 **여러 구간을 인터리브**.

FE가 읽을 때:

- **WebMCP** — “에이전트·auto browse가 **무엇을 호출할지**”  
- **Partial Updates** — “사람·에이전트가 **무엇을 보는지**”  
- 둘 다 **명령형 DOM 스크립트**를 줄이는 방향은 같다.

WebMCP만 넣고 목록 갱신은 `innerHTML`이면 **행동만** 깔끔하고 체감 로딩은 그대다. Partial Updates만 실험하고 contract가 없으면 auto browse는 여전히 **버튼 추측**에 기댄다.

---

## 7. Gemini in Chrome과의 관계

[I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)에서는 WebMCP 오리진 트라이얼(Chrome 149), **Gemini in Chrome**의 WebMCP API 지원 예정, **auto browse** 다단계 자동화를 함께 밝혔다.

```text
사용자 의도 (자연어)
    → Gemini in Chrome / auto browse
        → (도구 있음) WebMCP tool 호출
        → (도구 없음) actuation — 클릭·입력 시뮬레이션
    → 사용자는 탭에서 폼·확인 UI를 봄
```

사이트가 WebMCP를 붙이는 것은 **auto browse 비용·실패율을 낮추는** 쪽에 가깝다. AI Mode·UCP는 **검색·결제 표면**, WebMCP는 **자사몰 탭 안** — 레이어가 다르다.

---

## 8. Cursor Browser MCP — WebMCP 이전에 쓰는 검증

WebMCP·Gemini in Chrome은 **사이트가 도구를 말해 주는** 쪽이다. 그 전에도 FE는 에이전트와 브라우저를 붙여 검증한다. 나는 **Cursor IDE Browser MCP**를 그 층으로 쓴다. [Browser MCP E2E 글](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/)에서 환경(로컬 vs 배포 QA)을, [Playwright smoke 글](https://changbaebang.github.io/2026-06-08-cursor-playwright-smoke-pr-evidence/)에서 PR 증빙을 다뤘다. 세 가지는 **대체 관계가 아니다**.

| | Playwright smoke | Cursor Browser MCP | WebMCP (표준) |
|---|------------------|-------------------|---------------|
| **누가 부르는가** | CI·담당자가 `test:e2e` | Cursor 에이전트 + 사람이 채팅 | Gemini in Chrome·auto browse |
| **잘 맞는 oracle** | visible·역할·스크린샷 | 클릭·reload·**Interaction log** | 등록된 **도구·스키마** |
| **로그인·2FA** | 스펙 밖이거나 fixture | **사람이 먼저**, 이후 클릭 루프 | 탭 안 승인 UI |
| **산출물** | spec + PR 표 | 플랜 md·스냅샷·로그 | `toolname`·JSON Schema |

최근 nav 회귀에서도 **같은 식으로 갈렸다**. Header·Footer **smoke**는 Playwright가 맞았다 — `8 passed` 한 줄과 케이스 표로 리뷰어에게 넘기기 좋다. 반면 **클릭 한 번에 상태가 바뀌는지**, **reload 후에도 남는지**, **API 200인데 UI가 안 바뀌는지**는 Browser MCP 쪽이 낫다. Playwright만으로는 “스냅샷 찍고 끝”에 가기 쉽고, **로그인·SSO·2FA**는 스펙에 넣기 전에 **사람이 탭을 넘겨주는** 패턴이 현실적이다.

로그인 자동화를 “완전 무인”으로 밀기보다, 스킬에 이렇게 박아 두었다.

- 플랜 `Login` 열에 **계정·경로·인계 시점**을 적는다.  
- 사람이 QA에서 로그인한 뒤, 에이전트는 **그 탭에서** SC-* 시나리오를 이어간다.  
- 실패하면 Interaction log·네트워크 한 줄을 스킬에 남긴다 ([skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/) 루프).

WebMCP가 성숙하면 **사이트 contract**가 actuation을 줄인다. 그 전까지는 **Browser MCP = 오늘의 회귀 루프**, **Playwright = 남기는 증거**, **WebMCP = 붙일 다음 층**으로 나누는 편이 덜 헷갈린다.

---

## 9. FE 로드맵 초안

**Now** — Inspector + 문의/예약 폼 Declarative PoC, `agentInvoked` 분기·origin isolation·iframe `tools` 정책 확인.

**Next** — Imperative `apply_filters`·`add_to_cart`, [체크리스트 7항](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/)을 도구 단위로 매핑, 사람 vs 에이전트 경로 **로그 분리**.

**Later** — PLP/상세 느린 슬롯에 Partial Updates, [HTML-in-Canvas](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/) premium path와 **행동 contract는 DOM**에 유지.

**Capability:** UA 문자열 대신 `navigator.modelContext` 등 **런타임 API가 있는지** + 오리진 트라이얼 ([UA 후속](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)).

---

## 마치며

WebMCP는 “에이전트용 API를 하나 더 붙이자”가 아니다. **사이트가 자기 행동을 말하게** 하는 표준이다. Declarative API는 **가장 덜 침습적인 입구** — 폼이 있으면 `toolname`부터.

같은 I/O의 Declarative Partial Updates는 **다른 층**이지만, 방향은 같다.

> **추측·명령형 DOM 대신, 마크업과 브라우저에 선언을 맡긴다.**

다음에는 검색·Gemini·UCP가 **발견과 결제**를 어떻게 바꾸는지 본다.

---

## 읽을 거리

### Chrome I/O 2026 시리즈

- [새 웹 기능과 옛 WebView — HTML-in-Canvas](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)
- 검색·Gemini·커머스 — 2편 (초안, 미발행)

### 이 블로그

- [로컬 vs 배포 QA — Browser MCP E2E](https://changbaebang.github.io/2026-05-26-e2e-local-vs-deployed-browser-mcp/)
- [Playwright smoke — 스킬과 자연어](https://changbaebang.github.io/2026-06-08-cursor-playwright-smoke-pr-evidence/)
- [에이전트 인터페이스 FE 체크리스트](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/)
- [Declarative Partial Updates](https://changbaebang.github.io/2026-06-01-declarative-partial-updates-fe-notes/)
- [Google I/O 2026, FE가 봐야 할 6가지](https://changbaebang.github.io/2026-05-23-io2026-fe-what-matters/)
- [Browser MCP와 웹 개발](https://changbaebang.github.io/2026-05-23-browser-mcp-future-web-dev/)

### 공식

- [Chrome at I/O 2026 (ko)](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)
- [WebMCP (ko)](https://developer.chrome.com/docs/ai/webmcp?hl=ko) · [Declarative API (ko)](https://developer.chrome.com/docs/ai/webmcp/declarative-api?hl=ko)
- [patching explainer](https://github.com/WICG/declarative-partial-updates/blob/main/patching-explainer.md)
