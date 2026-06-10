---
layout: post
title: "새 웹 기능과 옛 WebView — HTML-in-Canvas가 보여준 것"
date: 2026-06-10 10:00:00 +0900
tags: [chrome, canvas, html-in-canvas, webview, frontend, web-platform, performance]
---

> [Chrome at I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)에는 **WebMCP**, **HTML-in-Canvas**, **Partial Updates** 같은 것들이 올라왔다.  
> 상황이 맞으면 **가져올 건 가져와야** 한다. 다만 커머스 현장에는 **오래된 WebView·인앱 브라우저**가 붙어 있고, polyfill로 따라잡기 어렵거나 **비슷한 화면**을 맞추기 힘든 API도 있다.  
> 이 글은 HTML-in-Canvas **소개**가 아니라, [UA 후속](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)에서 이어지는 생각이다. **legacy WebView 때문에 로드맵 자체를 멈추지 말고**, 제품에 **어느 단계로** 넣을지 정하고, **컴포넌트가 스스로** “지금 이 환경에서 돌아가는가”를 보고 보여줄지 말지 결정하는 쪽이 낫다.

---

## TL;DR

- I/O 신기능은 **읽고, 맞는 것은 단계적으로 도입**한다. 전부 무시할 이유는 없다.
- 막는 건 “WebView다”가 아니라, **제품 단계**(기본 / 향상 / 실험)를 정하지 않은 채 **전 화면에 한꺼번에** 얹으려는 것.
- **구형 WebView**는 “최신 기능을 **어디까지** 켤지”를 정할 때 **숫자**로 보면 된다. 로드맵 전체를 거기에 **묶어 두지는** 않는다.
- **컴포넌트**가 capability를 probe하고, 되면 enhanced, 안 되면 baseline(또는 아예 안 그리기)을 **스스로** 고르는 구조가 어색함을 줄인다.
- HTML-in-Canvas는 **일반 PLP·결제의 기본 경로**엔 아직 안 맞고, WebGL 캠페인 같은 **좁은 실험** 후보로 두면 된다.

---

## I/O는 그대로 봐야 한다

플랫폼 쪽은 계속 움직인다. WebMCP처럼 **에이전트·결제 흐름**과 맞는 것도 있고, Partial Updates처럼 **느린 구간**을 나누는 것도 있다. “WebView가 옛날이니까 I/O는 스킵”은 **오래 버티기 어렵다.**

다만 현장에서 자주 겪는 간격은 이거다.

```text
I/O 발표     →  최신 Chrome에서 잘 돌아감
우리 트래픽  →  앱 WebView APK 제각각, iOS, 인앱 브라우저
흔한 반응    →  “적용?” / “못 하니까 포기?”  ← 둘 다 극단에 가깝다
```

가운데는 이렇게 두는 편이 낫다.

- **도입 여부** — 이 기능이 우리 **목적**(구매, 검색, 캠페인)에 맞는가  
- **도입 단계** — 기본 화면인가, 일부 섹션인가, PoC인가  
- **런타임 판단** — 지금 이 기기·이 WebView에서 **실제로** 되는가  

HTML-in-Canvas는 그 간격이 **특히 크게** 드러나는 사례다. [오리진 트라이얼 (ko)](https://developer.chrome.com/blog/html-in-canvas-origin-trial?hl=ko) · [overview](https://html-in-canvas.dev/docs/overview/)

---

## legacy WebView에 발목 잡히지 않기

[UA 후속](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)에서 말한 것과 같다. WebView를 **“구형 = 기능 포기”** 한 줄로 묶지 않는다. 동시에 **“웹뷰 아니면 최신”**도 아니다.

| 흔한 함정 | 조금 나은 방향 |
|-----------|----------------|
| WebView라서 **신기능 검토 자체를 안 함** | 기능별로 **어느 tier**에 넣을지만 미리 정해 둠 |
| WebView라서 **전부 구형 UI** | 같은 앱 안에서도 **APK마다** 다름 — probe로 본다 |
| 데스크톱 데모 보고 **내일 전체 적용** | **단계·섹션** 단위 롤아웃 |
| UA 한 줄로 on/off | **런타임 capability** + (필요하면) 성능 예산 |

**product**(앱 버전, 브릿지, 채널)와 **platform**(API 존재, INP, 메모리)을 섞지 않는 것도 그대로다.

구형 WebView는 **“안 된다”**가 아니라 **“이 tier는 이 비율만”**으로 읽으면, 로드맵과 통계가 맞아 떨어진다.

---

## polyfill이 안 되는 API는 다르게 받아들이기

DOM·CSS 쪽은 progressive enhancement가 **“조금 덜 예쁜 Plan B”** 로 통하는 경우가 많다.

`layoutsubtree`·`drawElementImage` 같은 **엔진 안쪽** API는 JS로 **비슷하게** 맞추기 어렵다. 그래서 미지원일 때 목표는 **같은 화면의 축소판**이 아니라, **같은 목적을 담은 다른 tier** 쪽에 가깝다.

HTML-in-Canvas 예:

- **되면:** WebGL 옆에 DOM 가격 태그  
- **안 되면:** 익숙한 DOM 상세 — 3D를 CSS로 **억지로 흉내** 내기보다, **상품·CTA는 그대로**  

기획·마케팅이 “전 채널 똑같은 3D”를 요구하면 FE만 곤란해진다. **단계를 나눠 두면** iOS·구형 앱도 **미지원이 아니라 정상 경로**로 설명할 수 있다.

---

## 제품 단계 — 어디에 넣을지만 먼저 정하기

WebView **때문에** 단계를 정하는 게 아니라, **제품에** 단계를 정하고 WebView는 **커버리지 숫자**로 본다.

```text
기본     — 모든 런타임. 매출·결제·검색. DOM·데이터 정합.
향상     — 넓은 지원. view transition, soft nav 등 부담 작은 플랫폼 기능.
실험     — 좁은 지원. 오리진 트라이얼, HTML-in-Canvas, 캠페인 한 섹션.
```

- **실험**을 **기본의 축소 복제**로 만들지 않는다.  
- **기본**은 목적이 완결된 화면. **실험**은 보너스이거나 별도 랜딩.  
- [Baseline Checker](https://developer.chrome.com/blog/chrome-at-io26?hl=ko) + UV로 “향상·실험을 켤 **비율**”만 보면 된다.

HTML-in-Canvas는 **실험** 칸 후보에 가깝다. WebMCP처럼 **기본·향상** tier에 더 가깝게 볼 수 있는 I/O 주제도 있지만 — 기능마다 다르다.

---

## 컴포넌트가 스스로 결정하게

말하고 싶은 구조는 이거다. **페이지 전체**가 `if (isWebview())` 로 갈리기보다, **컴포넌트**가 “지금 나, 돌아가?”를 본다.

```text
CampaignHero
  ├─ canUseHtmlInCanvas() && perfBudgetOk  →  ImmersiveHero (실험)
  └─ else                                   →  StandardHero (기본)

ProductCard
  └─ 항상 DOM baseline — canvas 경로 없음
```

원칙만 적으면:

1. **capability는 컴포넌트 가까이** — `in` / `@supports` / trial / (필요 시) 간단한 perf probe  
2. **보여줄지 말지는 컴포넌트가 판단** — 부모가 “웹뷰니까 전부 구형”으로 일괄 off하지 않음  
3. **baseline은 같은 컴포넌트 트리 안에 둔다** — 빈 칸·하얀 화면 대신 **항상 채워진 기본 UI**  
4. **실험 variant는 optional** — 안 되면 **조용히** baseline만 (사용자에게 “미지원” 배너는 선택)

이렇게 두면 I/O에서 본 기능을 **한 섹션·한 컴포넌트**부터 넣어도, 나머지 화면은 legacy WebView에 **발목 잡히지** 않는다.

```tsx
// 패턴 예시 (의사 코드)
function CampaignHero(props: HeroProps) {
  const mode = useCapability({
    htmlInCanvas: () => 'layoutsubtree' in HTMLCanvasElement.prototype,
    perf: () => !prefersReducedMotion && meetsInpBudget('hero-canvas'),
  });

  if (mode.htmlInCanvas && mode.perf) {
    return <ImmersiveCanvasHero {...props} />;
  }
  return <StandardDomHero {...props} />;
}
```

`useCapability` 같은 건 팀마다 이름만 다를 뿐, **probe + 정책을 한곳**에 모으는 쪽이 유지보수에 낫다.

---

## HTML-in-Canvas — 사례로만

스펙 자세한 건 [참고](#읽는-순서)로 넘긴다.

**한 줄:** canvas/WebGL 앱에 DOM 조각을 붙이는 **실험적** 브릿지. **쇼핑몰 전체**를 옮기라는 뜻은 아니다.

**일반 커머스 기본 경로**에 아직 안 맞는 이유는 짧게:

- iOS·많은 인앱 브라우저 — Chromium 계열 실험  
- 결제·PG — [cross-origin painting](https://html-in-canvas.dev/docs/overview/)과 안 맞음  
- `paint` — [메인 스레드·스크롤 한계](https://developer.chrome.com/blog/html-in-canvas-origin-trial?hl=ko)  
- 상품·SEO — **시맨틱 DOM·피드**가 본진  

**볼 만한 때:** WebGL 캠페인, 3D 룩북 등 **별도 섹션** + `CampaignHero` 식 **컴포넌트 단위** + kill switch.

“영원히 안 쓴다”기보다 **“기본 레이어엔 안 올린다”** 정도로 두면 된다.

---

## 운영할 때 같이 보면 좋은 것

**앱·WebView (Android)**  
System WebView 업데이트 비중, minSdk — **지원 %**에 영향. 웹만 고치면 안 올라가는 구간이 있다.

**제품·마케팅**  
“전 사용자 3D” 대신 “지원 환경에서 3D, 그 외 **같은 상품 일반 상세**” — **단계**를 문서에 써 두면 덜 어색하다.

**신기능 입고 (가볍게)**  
1. polyfill 가능?  
2. **어느 tier**? (기본 / 향상 / 실험)  
3. UV % 측정했는가?  
4. **컴포넌트** baseline은 완결되는가?  
5. PoC → 측정 → **섹션** 롤아웃  

**성능**  
API가 있어도 INP·`onpaint` 빈도가 예산을 넘기면 **컴포넌트가 스스로** baseline으로 돌아가게 두는 편이 낫다.

---

## 맺음

I/O는 **계속 본다.** 맞는 기능은 **가져온다.** 다만 **legacy WebView** 때문에 “전부 안 함 / 전부 억지로 비슷하게”로 기울지 않는다.

**제품에 단계**를 정하고, **컴포넌트**가 런타임에 돌아가는지 보고 enhanced와 baseline 중 **스스로** 고르게 두면, 신기능도 넣을 수 있고 구형 환경도 **정상 경로**로 남는다.

HTML-in-Canvas는 그 연습에 쓸 만한 **극단적인 사례**다. 지금 당장은 **CampaignHero 같은 실험 칸** 정도로 두고, 리스트·결제·검색의 **본체는 DOM**으로 가져가면 된다.

[UA 후속](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)이 채널 이분법을 깨는 **첫 걸음**이었다면, 여기서는 **단계 + 컴포넌트 자율**이 **다음 걸음**에 가깝다.

---

## 참고

### 이 글과 함께 보면 좋은 것 (맥락)

| | 문서 | 용도 |
|---|------|------|
| 선행 | [WebView와 UA (발행)](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/) | 이 글의 **전제** — WebView·capability를 채널 한 줄로 보지 말자 |
| 배경 | [Chrome at I/O 2026 (ko)](https://developer.chrome.com/blog/chrome-at-io26?hl=ko) | HTML-in-Canvas가 **어디서** 소개됐는지 |
| 인접 주제 | [Partial Updates (발행)](https://changbaebang.github.io/2026-06-01-declarative-partial-updates-fe-notes/) | 같은 I/O의 **다른 층** 선언적 API |

### HTML-in-Canvas를 더 깊게 볼 때 (읽는 순서) {#읽는-순서}

이 글은 API **소개·튜토리얼**이 아니다. 사례를 이해한 뒤 스펙·논쟁까지 보려면 **아래 순서**가 부담이 덜하다.

| 순서 | 문서 | 이 단계에서 보는 것 |
|------|------|---------------------|
| 1 | [HTML-in-Canvas origin trial (ko)](https://developer.chrome.com/blog/html-in-canvas-origin-trial?hl=ko) | **왜** 나왔는지, 2D/WebGL **사용 흐름**, 공식 한계 |
| 2 | [Spec overview](https://html-in-canvas.dev/docs/overview/) | `layoutsubtree` · `paint` · privacy painting **요약** |
| 3 | [Design decisions](https://html-in-canvas.dev/docs/design-decisions/) | **왜** direct child만, transform 무시 등 설계 이유 |
| 4 | [WICG · Examples](https://github.com/WICG/html-in-canvas/) | 라이브 데모·explainer 원문 |
| 5 | [Mozilla standards-position #1076](https://github.com/mozilla/standards-positions/issues/1076) | 다른 엔진·표준화 **우려** (실험 단계라는 맥락) |
