---
layout: post
title: "content-visibility: auto를 적용했더니 position: fixed가 고정되지 않았다"
date: 2026-07-16 17:31:00 +0900
tags: [css, content-visibility, position-fixed, performance, frontend]
---

> 실제 커머스 프론트엔드의 성능 실험을 바탕으로 썼다. 조직과 저장소를 식별할 수 있는 정보는 걷어내고 재현 가능한 구조만 남겼다.

> 앞서 쓴 [동시에 처리한다는 것](https://changbaebang.github.io/2026-07-15-parallel-work-serial-judgment/)에서 짧게 언급했던 성능 실험이다. 제품 코드에 반영하지 않고 멈춘 이유까지 기술적으로 펼쳐본다.

화면 밖 콘텐츠의 렌더링 비용을 줄이려고 `content-visibility: auto`를 적용했다. 적용 자체는 간단했다. 그런데 모바일 화면 하단에 고정돼 있어야 할 버튼이 더 이상 뷰포트에 붙어 있지 않았다.

처음에는 관련 없어 보였다. 버튼에는 `position: fixed`가 그대로 있었고 버튼 스타일도 건드리지 않았다. 원인은 버튼이 아니라, 그 조상에 새로 생긴 **containment와 containing block**이었다.

## TL;DR

- `content-visibility: auto`는 화면 밖 콘텐츠를 건너뛰는 힌트만 주는 속성이 아니다. **`layout`, `style`, `paint` containment**도 함께 적용한다.
- layout과 paint containment가 적용된 박스는 `position: fixed` 자손의 새로운 containing block을 만든다.
- 그 결과 `fixed` 요소가 뷰포트가 아니라 해당 조상 박스를 기준으로 배치될 수 있다.
- 해결은 `fixed` 요소를 containment 밖으로 빼거나, `content-visibility` 적용 경계를 더 안쪽으로 좁히는 것이다.
- 다만 버그를 고칠 수 있다는 사실과 성능 최적화를 제품에 넣을 가치가 있다는 사실은 별개다. 이 실험은 수정 후에도 효과가 작아 최종적으로 철회했다.

## 시작은 화면 밖 렌더링 줄이기였다

대상 화면은 모바일에서 세로로 긴 폼이었다. 첫 화면 아래에 요약과 여러 입력 영역이 이어졌고, 하단에는 항상 노출되는 고정 버튼이 있었다.

구조를 단순화하면 다음과 비슷했다.

```tsx
<aside className="deferred">
  <Summary />
  <AdditionalSections />
  <FixedActionButton />
</aside>
```

```css
.deferred {
  content-visibility: auto;
  contain-intrinsic-size: auto 500px;
}

.fixedActionButton {
  position: fixed;
  right: 0;
  bottom: 0;
  left: 0;
}
```

`content-visibility: auto`를 사용하면 브라우저가 현재 화면에 필요하지 않은 콘텐츠의 레이아웃과 페인트를 건너뛸 수 있다. 긴 목록이나 폴드 아래의 큰 섹션에는 매력적인 선택이다.

문제는 이 속성을 `aside` 전체에 적용했다는 점이었다. `aside`에는 미뤄도 되는 콘텐츠뿐 아니라 고정 버튼도 들어 있었다.

## 증상: `fixed`인데 뷰포트에 고정되지 않았다

변경 후 모바일 화면을 열어보니 하단 버튼이 뷰포트에 고정되지 않았다. 스크롤을 움직이면 버튼은 뷰포트 하단을 따라오는 대신 `aside` 영역에 묶인 것처럼 움직였다.

개발자 도구에서 확인해도 버튼의 계산된 스타일은 여전히 `position: fixed`였다. `bottom: 0`도 정상이다. 그래서 버튼 자체만 보면 원인을 찾기 어렵다.

`position: fixed`의 기준이 무엇인지까지 봐야 했다.

## 원인: `auto`가 layout containment를 켠다

MDN에 따르면 `content-visibility: auto`는 요소에 `layout`, `style`, `paint` containment를 적용한다. 중요한 점은 요소의 콘텐츠가 실제로 건너뛰어지지 않는 순간에도 이 containment가 유지된다는 것이다. 화면 안으로 들어왔다고 원래 레이아웃 규칙으로 돌아가는 것이 아니다.

CSS Containment 명세는 layout containment box가 절대 위치와 고정 위치 요소의 containing block을 만든다고 정의한다. paint containment도 마찬가지다.

고정 위치 요소의 containing block을 만드는 조상이 따로 없다면 `position: fixed` 요소는 일반적으로 뷰포트를 기준으로 배치된다. 하지만 그런 조상이 생기면 이야기가 달라진다.

```text
변경 전
viewport
  └─ aside
       └─ fixed button  ── 기준: viewport

변경 후
viewport
  └─ aside (content-visibility: auto)
       └─ fixed button  ── 기준: aside가 만든 containing block
```

버튼 스타일을 건드리지 않았는데 버튼이 깨진 이유다. 성능 최적화를 위해 조상에 추가한 한 줄이 자손의 좌표계를 바꿨다.

## 최소 재현

아래 정도로도 현상을 확인할 수 있다.

```html
<main class="container">
  <div class="spacer">긴 콘텐츠</div>
  <button class="fixed">항상 화면 아래에 있어야 하는 버튼</button>
</main>
```

```css
.container {
  content-visibility: auto;
  min-height: 150vh;
}

.fixed {
  position: fixed;
  right: 16px;
  bottom: 16px;
}
```

`content-visibility`를 제거했을 때와 비교하면 `fixed` 요소의 기준이 달라지는 것을 볼 수 있다. 같은 문제는 `contain: layout`, `contain: paint`, `transform` 등 containing block을 만드는 다른 속성을 조상에 적용했을 때도 나타날 수 있다.

## 해결: 최적화 경계를 더 안쪽으로 옮겼다

고정 버튼에는 화면 밖 렌더링 최적화가 필요하지 않았다. 미루고 싶었던 것은 그 위의 요약과 폼 섹션뿐이었다. 그래서 containment 경계를 좁혔다.

```tsx
<aside>
  <div className="deferred">
    <Summary />
    <AdditionalSections />
  </div>

  <FixedActionButton />
</aside>
```

```css
.deferred {
  content-visibility: auto;
  contain-intrinsic-size: auto 500px;
}
```

이제 `fixed` 버튼은 containment 밖에 있고, 미뤄도 되는 콘텐츠만 그 안에 남는다.

핵심은 `z-index`를 더 높이거나 `fixed` 스타일을 다시 선언하는 것이 아니다. **좌표계를 바꾼 조상 관계를 끊는 것**이다.

버튼을 `portal`로 렌더링하는 방법도 있지만, 이 문제 하나를 피하려고 렌더링 위치를 바꾸는 것은 더 큰 구조 변경이 될 수 있다. 원래부터 모달이나 전역 오버레이처럼 `portal`이 자연스러운 UI가 아니라면 먼저 containment의 범위를 좁히는 편이 단순하다.

## 고쳤지만 제품에는 넣지 않았다

버튼 문제를 고친 뒤 같은 조건으로 다시 측정했다. 회귀는 없어졌지만 성능 차이도 측정 편차를 벗어나지 못했다.

원인을 다시 나눠보니 첫 화면 지연의 약 4분의 3은 데이터 API 응답을 기다리는 시간이었다. `content-visibility`는 DOM을 없애거나 React 렌더링과 하이드레이션을 생략하지 않는다. 현재 화면 밖 콘텐츠의 레이아웃과 페인트 일부를 미룰 뿐이다. 게다가 대상도 거대한 피드가 아니라 비교적 작은 폼 섹션이었다.

얻는 이익은 작지만 containment의 동작을 계속 기억해야 하는 유지 비용은 생겼다. 결국 이 변경은 제품에 넣지 않았다.

이 판단이 중요했다. **버그를 수정했다는 이유만으로 실험을 살려둘 필요는 없다.** 실험의 원래 목표는 CSS를 적용하는 것이 아니라 사용자가 체감할 만큼 첫 화면을 빠르게 만드는 것이었다.

## 적용 전에 볼 체크리스트

`content-visibility: auto`를 적용하기 전에는 자손까지 함께 확인해야 한다.

- 자손에 `position: fixed` 또는 `position: absolute`가 있는가
- box-shadow, focus ring, 드롭다운처럼 박스 밖으로 그려지는 UI가 있는가
- portal을 사용하지 않는 팝오버나 오버레이가 있는가
- `contain-intrinsic-size`의 예약 크기가 실제 크기와 너무 다르지 않은가
- 실제로 건너뛸 레이아웃과 페인트 작업이 충분할 만큼 콘텐츠가 긴가
- React 렌더링, 하이드레이션, API 대기가 진짜 병목은 아닌가
- 같은 조건의 A/B에서 차이가 측정 편차보다 큰가

특히 첫 번째 질문은 스타일을 적용하는 요소만 보고서는 답할 수 없다. containment는 자손의 배치 규칙까지 바꾼다.

## 마치며

`content-visibility: auto`는 사용하기 쉬워 보인다. CSS 한두 줄로 화면 밖 렌더링을 미룰 수 있다. 하지만 브라우저가 그 최적화를 안전하게 수행하려면 해당 하위 트리를 외부 레이아웃과 분리해야 한다. 그 격리가 `fixed` 자손의 기준점까지 바꾼다.

이번에 남은 교훈은 두 가지다.

1. 성능 관련 CSS 속성은 적용된 요소 자체만 바꾸는 것이 아니다. **자손의 좌표계와 페인트 경계까지 확인한다.**
2. 회귀를 고칠 수 있어도 효과가 작다면 적용하지 않는다. **수정 가능성과 제품 가치는 별도로 판단한다.**

## 참고 자료

- [CSS Containment Module Level 2](https://drafts.csswg.org/css-contain/)
- [MDN: content-visibility](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/content-visibility)
- [MDN: Layout and the containing block](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Display/Containing_block)
