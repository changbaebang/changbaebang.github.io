---
layout: post
title: "이상한 얇은 훅 만들지 말고, 그냥 useEffect 쓰자"
date: 2026-07-03 16:25:00 +0900
permalink: /2026-07-03-thin-hook-useeffect-instead/
tags: [refactoring, clean-code, react, hooks, abstraction]
---

> 웹뷰 ↔ iframe `postMessage` 브릿지 코드를 읽다 든 생각.
> 코드는 익명화했고, 실제 서비스/도메인/기능명은 전부 일반화했다.

## TL;DR

- 커스텀 훅으로 감싸는 게 항상 이득은 아니다.
- `window.addEventListener`를 감싼 얇은 훅 하나가, **값을 더하지도 못하면서** 코드를 더 어렵게 만들고 있었다.
- 심지어 같은 이름의 훅이 시그니처만 다르게 여러 개 있어서, 읽는 사람이 import를 봐야 뭔지 알 수 있었다.
- 결론: 이런 건 그냥 `useEffect`를 직접 쓰는 게 낫다. [Goodbye, Clean Code](https://overreacted.io/goodbye-clean-code/)의 교훈은 "추상화를 아끼라"는 쪽으로도 작동한다.

## 발단: 훅을 따라 들어갔더니

웹뷰 안에서 iframe이 부모 페이지로 `postMessage`를 쏘고, 부모가 그걸 받아 처리하는 흔한 브릿지 패턴이 있었다. 받는 쪽이 이런 훅을 썼다.

```ts
useBridgeMessage<Payload>(({ data: { type, payload } }) => {
  if (type === 'openSheet') { ... }
  if (type === 'closeSheet') { ... }
  if (type === 'navigate') { ... }
});
```

"`useBridgeMessage`가 뭘 해주는 거지?" 하고 정의를 열었더니, 실체는 이게 전부였다.

```ts
export const useBridgeMessage = <T>(handler: (e: MessageEvent<Message<T>>) => void) => {
  useEffect(() => {
    window.addEventListener('message', handler);
    return () => window.removeEventListener('message', handler);
  }, []);
};
```

`addEventListener('message')` + cleanup. **`useEffect` 보일러플레이트 한 조각을 감싼 것.** 타입 필터도 없고, 이벤트를 가공해주지도 않는다. 호출부는 어차피 안에서 `data.type`을 다시 꺼내 `if`로 분기한다.

## 추상화의 언캐니 밸리

이런 래퍼가 애매한 이유는, **너무 얇아서 값을 더하지 못하는데** 훅이라는 간접 계층은 하나 늘리기 때문이다.

- **읽는 비용**: `useBridgeMessage`를 만나면 정의로 점프해서 "아, 그냥 addEventListener구나"를 확인해야 한다. `useEffect`였으면 그 자리에서 다 읽힌다.
- **주는 값**: 리스너 등록/해제 라이프사이클 자동화 — 그런데 이건 `useEffect` 쓰면 원래 공짜로 따라오는 것이다.
- **안 주는 값**: 타입 필터링, 이벤트 정규화, origin 검증… 정작 있으면 좋을 건 하나도 없다.

즉 이 훅은 "추상화라면 제값을 해야 한다"는 기준을 통과하지 못한다. 얇은데 공짜는 아니다.

그리고 더 헷갈렸던 건, 같은 앱 안에서 **이름이 같은데 시그니처가 다른** 훅이 공존했다는 점이다.

| import 출처 | 시그니처 | 하는 일 |
|-------------|----------|---------|
| 앱 로컬 | `(handler)` | 필터 없이 raw 이벤트 통째로 넘김 |
| 공통 패키지 | `(type, handler)` | 특정 type일 때만 payload 전달 |

`useBridgeMessage`라는 이름만 보고는 둘 중 뭔지 알 수 없다. import 줄을 봐야 판별된다. 이름이 같다는 건 보통 "같은 것"이라는 신호인데, 여기선 거짓 신호였다.

## 잠깐, Goodbye Clean Code

이쯤에서 Dan Abramov의 [Goodbye, Clean Code](https://overreacted.io/goodbye-clean-code/)가 떠올랐다. 그 글은 보통 "성급하게 통합(추상화)하지 마라"로 읽힌다. 그런데 이 사례는 그 교훈의 **다른 얼굴**이다.

> 추상화는 공짜가 아니다. 제값을 하지 못하는 추상화는 만들지 않는 게 낫다.

훅으로 감싸는 것도 추상화다. "커스텀 훅 = 깔끔"이라는 조건반사가, 실제로는 `useEffect` 한 줄이면 될 걸 간접 계층으로 덮어버린 것이다. 중복을 성급히 통합하지 않는 것과, 애초에 얇은 래퍼를 만들지 않는 것은 같은 원칙의 양면이다.

그래서 원칙은 하나다. **추상화가 제값을 하지 못하면, 그 자리에 원래 있던 것(`useEffect`)을 그냥 쓴다.**

## 그래서: 그냥 useEffect 쓰자

정직한 선택지는 둘 중 하나다.

**1. 라이프사이클만 필요하면 → `useEffect`를 직접 쓴다.**

```ts
useEffect(() => {
  const handler = (e: MessageEvent) => {
    if (e.data?.type !== 'navigate') return;
    // ...
  };
  window.addEventListener('message', handler);
  return () => window.removeEventListener('message', handler);
}, [/* 이 핸들러가 실제로 참조하는 값들 */]);
```

한 줄 더 길어 보이지만, 점프할 필요 없이 그 자리에서 다 읽힌다. 그리고 각 호출부가 자기 의존성을 직접 관리하니, "얇은 훅이 deps를 대신 삼켜버리는" 함정도 없다.

**2. 진짜로 반복이 아프면 → 제값을 하는 범용 primitive로.**

정말 여러 곳에서 필요하면, 이름 겹치는 반쪽짜리 말고 제대로 된 걸 하나만 둔다. 예를 들어 handler를 ref로 안정화한 `useEventListener('message', handler)` 같은 것. 이건 라이프사이클 + 최신 handler 참조를 **실제로** 보장하므로 추상화가 제값을 한다.

지금처럼 "얇고, 필터도 없고, 이름은 겹치는" 중간 지점이 제일 나쁘다.

## 두 선택지 사이 판단 기준

**`useEffect`를 직접 쓴다** — 호출부가 한두 곳이고, 그 자리에서 무슨 일이 일어나는지 한눈에 보이는 게 중요할 때. `postMessage`를 한 컴포넌트에서만 받거나, origin 검증·타입 분기·DOM 조작이 그 호출부에 묶여 있을 때가 여기에 해당한다. "이 파일만 열면 전부 읽힌다"가 목표면 inline이 이긴다.

**범용 `useEventListener` 같은 primitive를 둔다** — 같은 패턴(등록 → cleanup, handler는 항상 최신)이 세 곳 넘게 반복되고, 호출부마다 달라지는 건 handler 내용뿐일 때. 이때 primitive가 해주는 건 라이프사이클 + ref로 handler 안정화 + deps 실수 방지다. 얇은 훅과 다른 점은, 이름·시그니처·동작이 팀 전체에서 하나로 고정된다는 것.

두 군데쯤에서 비슷하면 아직 primitive는 이르다. 각각 `useEffect`로 두고, 세 번째가 생겼을 때 검토해도 늦지 않다.

## 만들기 전 체크리스트

훅 파일을 새로 만들기 전에, 아래 중 **하나라도** 해당하면 만들 가치가 있다. 전부 아니면 `useEffect`로 충분하다.

| 질문 | 예: 제값을 하는 훅 | 예: 얇은 래퍼 |
|------|-------------------|---------------|
| 라이프사이클 외에 **필터·검증**을 대신 해주나? | `(type, handler)` — type 일치할 때만 호출 | `(handler)` — raw 이벤트 그대로 |
| **정규화·변환**을 한곳에서 하고 있나? | payload shape를 통일해서 넘김 | 호출부마다 `data.type` 분기 |
| **공유 상태·로직**을 묶고 있나? | 여러 type이 같은 DOM 상태를 함께 다룸 | addEventListener만 감쌈 |
| **세 곳 넘게** 같은 보일러플레이트가 반복되나? | `useEventListener` primitive | 한두 곳에서만 씀 |
| 이름이 **팀 전체에서 유일**한가? | 공통 패키지 하나 | 같은 이름 다른 시그니처 공존 |

마지막 줄이 핵심이다. 나머지를 다 통과해도 이름이 겹치면 만들지 말고, 기존 것을 쓰거나 이름부터 바꾼다.

## 정리

1. 커스텀 훅을 만들기 전에 "이게 `useEffect`보다 뭘 더 해주나?"를 묻는다. 대답이 "라이프사이클"뿐이면 그냥 `useEffect`를 쓴다.
2. 얇은 래퍼는 값을 더하지 못하면서 읽는 비용만 늘린다. 점프해서 확인해야 하는 추상화는 의심한다.
3. 같은 이름은 같은 것이라는 신호다. 시그니처가 다른데 이름이 같으면 그 자체로 버그의 씨앗이다.
4. 추상화를 아끼는 것도 "let it go"다. 만들 이유가 없으면 만들지 않는다.
