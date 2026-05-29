---
layout: post
title: "CSS 성능 글 4편 읽고 FE가 남긴 체크리스트 — Style·DOM·Web Vitals"
date: 2026-05-29 15:02:00 +0900
tags: [css, performance, web-vitals, frontend, rendering]
---

> CSS는 화면을 꾸미는 언어이기도 하지만,  
> 브라우저 입장에서는 Style → Layout → Paint → Composite 비용을 정하는 레이어다.

[web.dev/css (ko)](https://web.dev/css?hl=ko) 성능 관련 글 4편을 읽고, 개발할 때 손에 들고 볼 체크리스트로 정리했다.

## TL;DR

- CSS 성능은 선택자가 예쁜지보다, Style 재계산이 얼마나 넓은지와 DOM이 얼마나 큰지에서 갈린다.
- 화면 밖 구간은 `content-visibility`로 렌더를 미룰 수 있지만, 측정·fallback·접근성은 같이 봐야 한다.
- LCP·CLS·INP는 CSS 선택의 결과 지표다. “느낌상 빨라졌다” 전에 한 번 측정해 보면 충분하다.
- 긴 페이지, 리스트, 필터 UI는 DOM과 Style, INP를 동시에 건드린다.

---

## 4편을 한 줄로 묶으면

| 글 | 파이프라인에서 하는 일 |
|----|------------------------|
| Style calculations | Style 단계 — 누구에게 규칙이 적용되는지, 재계산 범위 |
| DOM size & interactivity | Layout + 메인 스레드 — 트리 크기, 입력 지연 |
| content-visibility | Paint 전 — 안 보이는 구간 렌더 지연 |
| CSS for Web Vitals | 검증 — LCP·CLS·INP와 CSS의 연결 |

읽는 순서는 위 표와 같다. 원인(Style/DOM) → 완화(content-visibility) → 측정(Web Vitals).

---

## 각 글에서 남긴 것

### 1) Style 재계산 범위 줄이기

전역·깊은 선택자, 과한 `:hover` 규칙은 Style 단계에서 더 많은 노드를 훑게 만든다.  
컴포넌트 스코프, 단순 선택자, 불필요한 전역 규칙 정리는 미관 문제가 아니라 성능 습관에 가깝다.

→ [Reduce the scope and complexity of style calculations](https://web.dev/articles/reduce-the-scope-and-complexity-of-style-calculations?hl=ko)

### 2) DOM 크기와 상호작용(INP)

DOM이 크면 Style/Layout뿐 아니라 이벤트 처리, 스크롤, 입력 반응도 같이 무거워진다.  
wrapper div, 긴 리스트, 한 번에 mount되는 subtree는 구조 문제이면서 INP 문제이기도 하다.

→ [How large DOM sizes affect interactivity](https://web.dev/articles/dom-size-and-interactivity?hl=ko)

### 3) content-visibility — off-screen 렌더 지연

viewport 밖 콘텐츠의 렌더·페인트를 미루면 초기 로드나 스크롤 진입 체감이 나아질 수 있다.  
다만 레이아웃/contain, 스크린리더, 검색·SEO 영향은 페이지마다 달라서, 속성 한 줄로 끝나지는 않는다.

→ [content-visibility: boost rendering performance](https://web.dev/articles/content-visibility?hl=ko)

### 4) CSS와 Web Vitals

- **LCP:** 히어로·첫 meaningful paint에 걸리는 CSS(폰트, 이미지 크기, critical path)
- **CLS:** 레이아웃이 밀리는 CSS(예약 공간 없는 미디어, 늦게 붙는 UI)
- **INP:** 클릭 후 메인 스레드가 Style/Layout에 묶이면 입력이 늦게 느껴진다

→ [CSS for Web Vitals](https://web.dev/articles/css-web-vitals?hl=ko)

---

## 개발할 때의 체크리스트

새 화면을 만들거나 기존 UI를 손볼 때, 위 4편을 읽고 나면 대략 이런 질문 목록이 남는다.

### Style

- [ ] 깊은 선택자(`div div div …`)나 과한 전역 `:hover`가 없는지
- [ ] 상태 토글 시 영향 받는 subtree가 최소인지 (class 한두 개로 끝나는지)
- [ ] utility 위주라도 불필요한 전역 `@layer` 오염은 없는지

### DOM

- [ ] 리스트·모달·탭에 의미 없는 wrapper를 줄일 여지가 있는지
- [ ] 긴 스크롤 페이지라면 가상화·페이지네이션을 검토할 만한지

### Paint / Layout

- [ ] 애니메이션은 `transform` / `opacity` 위주인지
- [ ] off-screen 블록 한 곳에 `content-visibility: auto` PoC를 해볼 만한지

### 측정

- [ ] Lighthouse로 LCP·CLS·INP(또는 TBT) 스냅샷 한 번
- [ ] DevTools Performance로 필터·스크롤·탭 전환 1회 녹화
- [ ] “빨라졌다”고 말하기 전에 숫자 한 줄 남기기

화면 유형에 따라 강조점만 조금 달라진다.

| 화면 유형 | Style | DOM | content-visibility | Vitals |
|-----------|-------|-----|------------------|--------|
| 긴 상품 리스트 | item 공통 class | 카드 수·depth | 리스트 섹션 단위 | LCP(첫 카드), CLS(이미지) |
| 필터·정렬 UI | 상태 class 범위 | overlay subtree | (보통 해당 없음) | INP(클릭→반응) |
| 마케팅 롱폼 | 전역 규칙 | section 수 | below-fold section | LCP(히어로), CLS |

PoC는 한 페이지, 한 변경, 한 번 측정이면 충분하다.

---

## 마치며

web.dev CSS 성능 글 4편을 통째로 외울 필요는 없다.  
무언가를 개발할 때 손에 들고 볼 체크리스트는 대략 이렇게 생겼다 — Style과 DOM을 먼저 보고, off-screen이 있으면 `content-visibility`를 검토하고, 마지막에 Vitals로 한 번 확인한다.

더 깊게 들어가고 싶다면 [web.dev/css (ko)](https://web.dev/css?hl=ko)에서 각 글을 이어서 보면 된다.
