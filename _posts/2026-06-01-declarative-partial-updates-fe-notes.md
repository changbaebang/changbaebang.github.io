---
layout: post
title: "선언적이라는 말이 두 겹이다 — React·RSC와 Declarative Partial Updates"
date: 2026-06-01 17:03:00 +0900
tags: [chrome, html, performance, frontend, web-platform, streaming, react]
---

> React도 선언적이다.  
> 그런데 Chrome이 말하는 **Declarative Partial Updates**의 “선언적”은 **다른 층**이다.

[Chrome DevBlog — Declarative partial updates (ko)](https://developer.chrome.com/blog/declarative-partial-updates?hl=ko)를 읽다가, API보다 먼저 이 질문이 남았다. **우리가 이미 쓰는 “선언적”과 무엇이 다른가.**  
이 글은 그 차이를 기준으로, FE가 어디를 보면 좋은지 정리해 본다.

최근 [CSS 성능 체크리스트](https://changbaebang.github.io/2026-05-28-css-rendering-pipeline-practical-notes/)에서 DOM·스트리밍을 봤다면, 여기서는 **같은 불편을 React 밖에서 푸는지**를 본다.

## TL;DR

- **HTTP/2·HTTP/3** 이후 전송은 **스트림**으로 보는 일이 익숙해졌다. 이제 **마크업·HTML** 쪽이 그에 맞는 규칙을 준비하는 단계에 가깝다.
- **React의 선언적:** `state` → UI. **런타임**이 reconcile·hydrate 한다.
- **RSC:** 그 선언을 **서버까지** 늘린다. 조각은 **Flight·프레임워크**가 이어 붙인다.
- **Declarative Partial Updates:** **스트림·문서 자리**를 마크업으로 선언한다. **HTML 파싱·패칭**이 실행 주체다.
- 겹치는 문제: 느린 조각, 순서, 부분만 먼저 보여 주기. **갈라지는 곳:** React 안이냐, HTML 플랫폼 안이냐.
- **지금:** Chrome 148+ 실험. PoC·관측용. “React 대체”가 아니라 **층이 다른 도구**다.

---

## 1) React가 말하는 선언적

클라이언트 React에서 “선언적”은 보통 이렇게 쓰인다.

- UI는 **상태의 함수**다.
- DOM을 직접 손대기보다, **렌더 결과**를 맡긴다.
- 명령형에 가깝다면 `querySelector`로 노드를 찾아 `appendChild`하는 쪽이다.

Suspense, lazy, code split도 같은 방향이다. **느린 조각 때문에 전체를 막지 않게** React 런타임 안에서 순서와 로딩을 다룬다.  
여기까지는 익숙한 이야기다.

---

## 2) RSC — 선언이 서버까지 넓어질 때

[RSC](https://react.dev/blog)는 그 선언을 **서버 컴포넌트 트리**까지 확장한다.  
서버가 먼저 그릴 수 있는 것을 보내고, 클라이언트가 나머지를 이어받는다. **부분만 먼저, 나중에 끼우기**라는 문제의식은 React 쪽에서도 커졌다.

다만 실행은 여전히 **React + 프레임워크** 안이다.

- 직렬화·역직렬화는 **Flight** 같은 프로토콜
- 경계는 **번들·패치·보안** 이슈와 맞닿는다 ([RSC 보안 정리](https://changbaebang.github.io/2026-05-31-rsc-security-fe-checklist/))

즉 RSC는 **“React가 원래 하던 선언적 UI”를 서버 쪽으로 넓힌 층**에 가깝다.  
HTML 문서의 **도착 순서**를 플랫폼이 직접 다루는 것과는 겹치지만, **같은 메커니즘은 아니다.**

---

## 3) Declarative Partial Updates — 문서가 말하는 선언적

Chrome 제안은 **컴포넌트·상태**가 아니라 **placeholder와 늦게 오는 HTML**을 선언한다.

```html
<?marker name="photos">

<!-- 스트림 뒤쪽에서 -->

<template for="photos">
  <img src="..." alt="...">
</template>
```

- **선언:** `photos`라는 이름의 자리에, 나중에 이 HTML을 넣어라.
- **실행:** 브라우저의 파싱·치환 규칙 (프레임워크 reconcile이 아니다)

| | 무엇을 선언하나 | 누가 실행하나 |
|--|----------------|---------------|
| 클라이언트 React | state → UI | React 런타임 |
| RSC | 서버·클라이언트 컴포넌트 트리 | React + Flight·Next 등 |
| Partial Updates | marker → HTML 조각 | HTML·브라우저 |

세 줄로 압축하면:

> React의 선언적은 **상태에서 UI로**.  
> RSC는 그걸 **서버까지**.  
> Partial Updates는 **스트림에서 문서 자리로**.

---

## 4) 같은 불편, 다른 층 — out-of-order

공식 글이 푸는 문제는 React가 겪던 것과 **비슷하다.**

1. **준비된 것부터** — DB가 느린 블록 때문에 앞 HTML 전체를 막아 두지 않기. (Suspense·스트리밍과 같은 방향)
2. **로드 순서** — 메가 메뉴를 본문 뒤로 보내고, 도착할 때 끼우기.
3. **Island에 가까운 골격** — hydration 전에 자리만 잡아 두기.

차이는 **누가 맞추느냐**다.

- React/RSC: 컴포넌트 트리·런타임·번들
- Partial Updates: `<?marker?>`, `<?start?>`/`<?end?>`, `<template for>`

`<?start?>` / `<?end?>` 사이에 Loading을 두는 것도 **선언적**이지만, **타이밍 설계**는 여전히 개발자 몫이다. React에서 로딩 UI를 설계하는 것과 같은 종류의 일이다.

### PoC 전에 볼 제한

- `<template>`은 **같은 부모 안**의 instruction만 갱신 (보안)
- `innerHTML` / `setHTML`과 **스트리밍** 삽입은 fragment 동작이 다름
- placeholder를 스트리밍 중에 옮기면 **엉뚱한 자리**에 계속 들어갈 수 있음

→ [공식 Restrictions](https://developer.chrome.com/blog/declarative-partial-updates?hl=ko)

---

## 5) 같은 불편, 다른 층 — HTML 삽입·스트리밍 API

React SPA는 **첫 HTML 로드**만 스트리밍 이점이 있고, 이후 갱신은 JS로 **HTML 덩어리**를 받아 `innerHTML`·`setHTML` 등에 넣는 경우가 많다.  
API마다 **덮기·append·sanitize·script** 규칙이 달라 외우기 어렵다.

Partial Updates 쪽 제안은 **동작 이름을 통일**하고 **stream**을 붙인다.

| 동작 | Static | Streaming |
|------|--------|-----------|
| 내용 교체 | `setHTML` | `streamHTML` |
| 요소 교체 | `replaceWithHTML` | `streamReplaceWithHTML` |
| 앞/뒤/자식 | `beforeHTML`, `appendHTML` … | `streamBeforeHTML`, `streamAppendHTML` … |

React가 reconcile로 맞추던 **“조각을 문서에 넣기”**를 **플랫폼 API**로 다시 쓰려는 셈이다.  
`Unsafe`는 “쓰지 마라”가 아니라 **sanitizer·runScripts**를 드러낸 이름이다.

Polyfill `html-setters-polyfill`은 형태 위주이고, 스트리밍은 버퍼 후 적용한다. **RSC용 polyfill**을 쓰는 것과 같이, 실험 단계라는 전제는 같다.

---

## 6) 플랫폼은 가는데, 도구는 따라오나

HTTP/2, HTTP/3가 널리 쓰이면서 **전송**은 예전보다 **스트림**으로 보는 일이 많아졌다. 연결 하나에 여러 조각이 오고, 순서와 타이밍을 **전송 계층**에서 다룬다.

그런데 **앱·마크업**은 한동안 그에 맞춰지지 못한 채, **한 덩어리 HTML + JS로 조립**에 가까웠다. React는 그 세계를 **state·VDOM**으로 받아들였고, 스트리밍 UI 문제는 Suspense·RSC처럼 **런타임 안**에서 풀어 왔다. **문서 자리·도착 순서**를 HTML이 직접 선언하는 모델은, 아직 기본 전제가 아니다.

**Declarative Partial Updates**는 그 간극을 **순수 마크업 쪽**에서 메우려는 시도에 가깝다. React가 “늦었다”기보다, **다른 층에서 먼저 준비하는** 그림이다. RSC는 프레임워크가 따라가는 중이고, marker·template은 **브라우저·HTML**이 따라가는 중이다.

**최신 웹 = 브라우저만 올리면 된다**고 느끼기 어려운 이유도, 그 사이에 **도구 층**이 끼어 있기 때문이다.

CSS도 그렇다. container queries, `@property`, relative color syntax처럼 **브라우저에 생긴 뒤**, Tailwind 같은 **빌드 타임·zero-runtime CSS**가 문법·매핑을 넣어 줘야 그 스택 안에서 쓰기 편해진다. 안 넣으면 **순수 CSS로 빠지거나**, **메이저 업데이트**를 기다린다.

React·Next도 비슷하다.

```
브라우저 (HTML/CSS 신규 API)
    ↑ 릴리스·패치
프레임워크·도구 (Next, Tailwind, …)
    ↑ 릴리스·마이그레이션
앱 코드
```

[GeekNews #29900](https://news.hada.io/topic?id=29900) 같은 “React 피로” 큐레이션도, **맹신**만의 문제가 아니라 **플랫폼은 앞서 가는데 기본 스택이 따라오는 속도**에 대한 불만이 섞여 있다. 이 글은 그 감정을 **스택 교체**로 풀지 않고, **어느 층을 볼지**로 풀려는 쪽이다.

Tailwind를 올려야 최신 CSS를 쓰는 것처럼, React·Next를 올려야 RSC·패치·생태계를 맞추는 것과 겹친다.  
Partial Updates는 **플랫폼 층이 하나 더 드러나는** 그림이라, “업데이트를 안 하면 최신 웹을 못 쓴다”는 말이 더 분명해진다. 다만 marker 층은 **React 없이도** PoC할 수 있다는 점이 다르다.

---

## 7) FE 체크리스트 — “어느 층의 선언”으로 풀까

### 문제 정의

- [ ] 병목이 **상태·컴포넌트**인가 → React 선언(기존) / RSC
- [ ] 병목이 **HTML 도착 순서·스트림**인가 → Partial Updates 후보
- [ ] 병목이 **DOM 크기·Style**인가 → [CSS 체크리스트](https://changbaebang.github.io/2026-05-28-css-rendering-pipeline-practical-notes/) (다른 층)

### Partial Updates를 볼 때

- [ ] `<?marker?>`, `<template for>`로 **자리·조각**을 표현할 수 있는 화면인가
- [ ] `chrome://flags/#enable-experimental-web-platform-features` (Chrome 148+)
- [ ] polyfill 한계·**a11y**(치환 후 DOM 순서)·**Unsafe** 입력 신뢰도

### React/RSC와 같이 쓸 때

- [ ] **RSC·Next 패치**와 **플랫폼 HTML 실험**을 같은 티켓에 섞지 않았는가
- [ ] “React가 선언적이니까 이 API는 안 봐도 된다” / “이 API가 있으니 RSC는 필요 없다” — **둘 다 성급한 결론**인지 한 번 점검

---

## 마치며

전송은 이미 **스트림**으로 보는데, **문서**는 아직 그에 맞게 말하는 법이 부족했다. Partial Updates는 그쪽을 **마크업**으로 채우려는 시도다.

**선언적**이라는 말이 겹쳐서 헷갈리기 쉽다.

- **React:** 상태로 UI를 말한다.
- **RSC:** 그 말을 서버까지 넓힌다. 런타임·Flight는 그대로다.
- **Declarative Partial Updates:** 스트림 안 **문서 자리**를 HTML로 말한다.

셋 다 “부분만 먼저, 나중에 끼우기”와 친하지만, **서로 바꿔 끼울 관계는 아니다.**  
순서·스트리밍이 병목이면 플랫폼 층을 보고, 컴포넌트·서버 경계가 병목이면 React·RSC 층을 보면 된다.

개발할 때 손에 들고 볼 질문 하나로 줄이면:

> **이 화면의 선언은 state인가, Flight인가, marker인가?**

더 깊게는 아래 원문을 보면 된다.

→ [Declarative partial updates (ko)](https://developer.chrome.com/blog/declarative-partial-updates?hl=ko)
