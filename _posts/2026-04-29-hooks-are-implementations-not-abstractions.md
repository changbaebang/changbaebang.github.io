---
layout: post
title: "hook 은 추상화가 아니다 — lifecycle 의 한 슬롯이다"
date: 2026-04-29 10:00:00 +0900
author: Changbae Bang
tags: [FE, React, hook, 추상화, 회고]
---

# 들어가면서 — 한 줄

오늘 한 결제 흐름의 hook 을 한참 들여다봤다. 그 hook 한 개를 읽기 위해 hook 일곱 개를 같이 펼쳐 두어야 했다. 그 자리에서 한 줄을 적어 두기로 했다.

> **hook 은 도메인 추상화가 아니다. lifecycle 의 한 슬롯이다.**

이 한 줄을 풀어 적어 두는 게 이번 글이다. *"hook 은 함수의 한 종류일 뿐이고, 그 함수에는 React 라이프사이클이라는 환경 의존성이 깊게 박혀 있다"* 라는 정의로부터 이번 사례의 패악을 다시 보면, 코드가 왜 이렇게 자라는지 한 박자 또렷해진다.

# 이번 사례 — 다섯 줄, 일곱 층

`handleCheckout` 한 칸은 다섯 줄이다.

```ts
const handleCheckout = withAuth(async () => {
  trackClickCheckout();
  await withGate(() => {
    submitCheckout();
  })();
});
```

이 다섯 줄을 한 번 읽기 위해 머릿속에 동시에 들어와야 하는 가정 일곱 가지.

1. `withAuth(cb)` 는 hook 이 반환한 **콜백 래퍼**다. 안의 callback 은 기본 인증 통과 후에야 실행된다.
2. 그 callback 안에서 `withGate(cb)` 가 또 한 번 콜백 래퍼다. **HOF 위의 HOF.**
3. `withGate(cb)` 의 결과는 함수다. 그래서 `()` 로 한 번 더 *즉시 호출* 한다.
4. 그 결과는 promise 다. 그래서 `await` 가 붙는다.
5. 안의 callback 은 `() => { submitCheckout(); }` 다. — 그런데 `submitCheckout` 은 그 자체가 함수다. *함수를 한 겹 더 감쌌다.*
6. `submitCheckout` 안의 `runOrder()` 는 sync 호출이다. — **밖의 `await` 는 거짓말이다.** 동기화를 보장하지 못한다.
7. 같은 함수 안의 `trackClickCheckout` 은 *로깅 트래커* 용이다. 같은 파일 윗쪽에는 *분석 트래커* 용 `trackClickCheckout` 이 동명으로 또 있다. **이름이 두 개의 일을 한다.**

다섯 줄을 읽기 위해 일곱 층을 동시에 들고 있어야 한다.
*다섯 줄에 일곱 가지 가정이 박힌 코드는, 일곱 가지 가정 중 하나가 어긋나는 순간 다섯 줄 전체가 깨진다.*

# 왜 이렇게 자라는가 — hook 의 추상화 착시

이 코드는 처음에 한 줄이었을 것이다.

```ts
const handleCheckout = () => submitCheckout();
```

여기에 *"기본 인증을 먼저 받아야 합니다"* 가 들어왔다. 누군가 hook 을 만들었다.

```ts
const { withAuth } = useWithAuth({ ... });
const handleCheckout = withAuth(submitCheckout);
```

그 다음 *"민감 상품이면 추가 게이트도"* 가 들어왔다. 또 다른 hook 이 만들어졌다.

```ts
const { withGate } = useWithGate({ ... });
const handleCheckout = withAuth(() => withGate(submitCheckout)());
```

그 다음 *"클릭 시점에 로그를 남겨야 합니다"* 가 들어왔다. 그리고 그 로그가 두 종류 (분석 트래커 / 로깅 트래커) 로 갈라졌다. 그래서 다섯 줄짜리 양파가 됐다.

이 양파는 **각 단계가 hook 으로 만들어졌다는 사실 자체가, 단계 사이의 경계를 흐려 놓은** 결과다. hook 의 외형을 띠는 순간, 우리는 이 단계가 *추상의 단위* 라고 착각한다. 그래서 hook 이 hook 을 호출하고, 그 hook 이 또 hook 을 호출하는 그래프가 자란다.

> **추상화의 단위는 함수 / 모듈 / 도메인이다. hook 의 단위는 React 라이프사이클의 한 슬롯이다.**

이 둘이 같은 줄에 있을 때만 hook 은 잘 작동한다. 다른 줄이 되는 순간, hook 은 *추상의 단위* 가 아니라 *구현의 단위* — 즉 *lifecycle 위에 박힌 코드 한 칸* 이 된다.

# React 팀이 같은 줄에 그어 둔 선

이건 우리 팀만의 의견이 아니다. React 공식 문서가 같은 자리에 선을 그어 둔다.

react.dev 의 *Reusing Logic with Custom Hooks* 의 한 문장.

> *"Custom Hooks let you share **stateful logic** but **not state itself**."*

*stateful logic* 의 재사용 — 이게 hook 의 자리다. *state 자체* 의 공유는 아니라고 명시한다. 뒤에 다시 보겠지만, 이번 사례에서 본 가장 위태로운 hook 은 정확히 이 선을 넘은 hook 이다.

같은 페이지의 또 다른 한 문장.

> *"A custom Hook is a JavaScript function whose name starts with 'use' and that may call other Hooks."*

**그저 JS 함수다.** 모듈도, 도메인의 단위도 아니다.

여기에 한 줄을 더하면 hook 의 환경 의존성이 또렷해진다. Dan Abramov, *Why Do React Hooks Rely on Call Order?*.

> *"Hooks rely on a stable call order between renders."*

hook 은 자기 정체성을 *render 마다 매겨지는 호출 순서 인덱스* 로 갖는다. 그래서 hook 의 시그니처만 보고 그 동작을 추측할 수 없다. **시그니처가 약속한 것과, hook 이 실제로 하는 일 사이의 거리** — 이번 사례에서 본 그 거리 — 의 근본 원인이 여기 있다.

react.dev 의 챕터 구조도 같은 선을 그어 둔다.

- `useState` / `useReducer` / `useContext` → **"Managing State"** 챕터
- `useEffect` / `useRef` / `useImperativeHandle` → **"Escape Hatches"** 챕터

> *"Effects are an escape hatch from the React paradigm."*

React 팀 자신이 hook 의 일부를 *비상구* 로 분류했다. 일상의 모듈을 *비상구로 짓지 말라* 는 뜻이다.

마지막으로 방향성. **React Compiler** 는 `useMemo` / `useCallback` 을 컴파일러가 가져간다. **React Server Components** 는 *기본은 hook 없는 함수* 를 표준으로 만든다. 둘 다 같은 방향이다.

> hook 은 점점 *lifecycle 의 자리에 머무는 도구* 가 되어 가고, 그 외의 자리는 함수와 타입과 컴파일러가 가져간다.

이번 글의 척추 한 줄이 React 팀이 코드와 문서 양쪽에서 같이 그어 두는 선과 같은 자리에 있다.

# 같은 컴포넌트, 같은 hook 두 번

이 결은 hook-in-hook 그래프 안에서 한 번 더 드러난다. `useCheckout` 본체와 그 안에서 호출하는 `useGateGuard` 가 **같은 hook 두 개를 각각 호출**한다.

```ts
export const useCheckout = () => {
  // ...
  const { logged } = useUser();              // 본체
  const { isGated } = useGatedItem();        // 본체
  const { withGate } = useGateGuard();
  // ...
};

export const useGateGuard = () => {
  const { logged } = useUser();              // guard 안
  const { isGated } = useGatedItem();        // guard 안
  const { withGate } = useWithGate({ logged, isGated });
  return { withGate };
};
```

같은 렌더 사이클에서 `useUser`, `useGatedItem` 이 한 번씩 더 호출된다. 둘 다 React Query 결과를 들고 있어 호출이 두 번이라고 fetch 가 두 번 가지는 않는다.
**그러나 의존 그래프는 두 배가 된다.** 그리고 두 호출의 반환값이 *반드시 같다는 보장* 은 React 가 해 주지 않는다 — 한 batch 안에서는 같지만, 그 보장을 코드에서 명시하지 않는다.

이건 *hook 을 추상의 단위로 다뤘기 때문에* 생긴 그림이다.
함수의 단위로 다뤘다면 — 즉 `useGateGuard` 가 자기 의존성을 *받았다면* — 같은 hook 이 두 번 호출되는 그림은 그려지지 않는다.

```ts
// 이런 모양이었어야 한다
const useGateGuard = (deps: { logged: boolean; isGated: boolean }) => {
  const { withGate } = useWithGate(deps);
  return { withGate };
};

// 호출자
const { logged } = useUser();
const { isGated } = useGatedItem();
const { withGate } = useGateGuard({ logged, isGated });
```

차이가 미세해 보이지만 결정적이다.
**전자는 `useGateGuard` 가 자기 의존성을 *스스로 가져온다.*** 그래서 외부에서 같은 의존성을 또 가져오면 두 번이 된다.
**후자는 의존성을 *받는다.*** 호출자가 한 번만 가져와서 흘려 준다.

전자는 hook 을 *모듈* 처럼 다룬 결과다. 후자는 hook 을 *함수* 로 다룬 결과다. **같은 의존성은 hook 트리의 한 칸에서만 가져온다** — 이게 hook 을 함수처럼 다루는 가장 작은 규칙이다.

# `usePricing` — *state 자체* 의 공유라는 선을 넘은 hook

같은 결의 더 깊은 사례는 가격 영역의 `usePricing` 이다. 이 hook 의 모양이 *react.dev 가 그어 둔 선* — *stateful logic 의 공유 ⊆, state 자체의 공유 ⊄* — 을 어떻게 넘는지 가장 또렷하게 보여 준다.

```ts
export const usePricing = (itemId: number) => {
  // ...
  const { requestParams, setRequestParams, increaseConsumerCount, decreaseConsumerCount } = usePricingStore();

  // NOTE: 여러 컴포넌트(영역 A, 영역 B 등)에서 이 훅을 동시에 호출하더라도
  //       전역 스토어를 공유하므로 상태가 동기화되며, React Query 에 의해 API 중복 호출이 방지됩니다.

  useEffect(() => {
    increaseConsumerCount();
    return () => decreaseConsumerCount();
  }, []);

  const query = useQuery({
    queryKey: [
      'pricing',
      itemId,
      requestParams?.flagA,
      requestParams?.flagB,
      requestParams?.flagC,
      requestParams?.flagD,
      requestParams?.selectionSet,
      // ...8개 더
    ],
    // ...
  });
  // ...
};
```

이 hook 의 외형은 *"itemId 를 받아 가격을 돌려주는 hook"* 이다. 안을 열면 다른 모양이다.

- **외형은 hook**, 실체는 *전역 싱글톤* 이다. 모든 호출자는 같은 store 를 공유한다 — *state 자체* 의 공유다.
- **lifecycle 로 reference counting** 을 한다. mount 시 `increaseConsumerCount`, unmount 시 `decreaseConsumerCount`. 마지막 consumer 가 빠지면 store 를 비운다 — *메모리 매니저가 hook 안에 들어와 있다.*
- **React Query 의 `queryKey` 가 store 의 `requestParams` 거의 전부** 를 받아쓴다. 한 컴포넌트가 `setRequestParams` 하면 그 변경이 *모든 호출자의 `queryKey` 를 바꾼다.* — 한 컴포넌트의 동작이 다른 컴포넌트의 캐시 엔트리를 흔든다.
- 더 나아가 *캐시 키를 수동으로 흐트리기 위한 sentinel* 까지 박혀 있다. 코드 위 주석에 명시:
  > *"queryKey 의 selectionSet 이 기존 값과 달라지므로 **별도 캐시 엔트리·재요청이 의도적으로 발생한다.**"*

이걸 hook 한 개의 시그니처 — `usePricing(itemId) → PricingResult` — 만 보고 짐작할 수 있는 사람은 없다.

이 hook 을 부르는 호출자는 *itemId 만 넘기면 알아서 돌려 준다* 고 생각한다. 안을 열면 그게 거짓말이다. 호출자는 store 를 공유하고, 다른 호출자의 `setRequestParams` 에 영향을 받고, 마지막으로 빠지는 자신이 store 를 비운다.

react.dev 의 한 줄 — *"share stateful logic, **not state itself**"* — 을 정확히 넘은 자리다. 그리고 *"hook 은 lifecycle 의 한 슬롯"* 이라는 정의에서 *"reference counting 을 하는 메모리 매니저"* 까지 한 번 더 넘은 자리이기도 하다.

> **hook 의 시그니처가 약속한 것** 과 **hook 이 실제로 하는 일** 의 거리가 이 hook 이 만들어낸 빚이다.

이 빚을 갚는 곳은 보통 호출자 컴포넌트가 아니라 — *나란히 호출되는 다른 컴포넌트* 다. 한 쪽이 setter 를 호출했을 때, 옆 쪽 컴포넌트의 query 가 다시 도는 사이드이펙트가 같은 페이지 안에서 일어난다.

# hook 이 잘하는 일과, hook 으로 하지 말아야 할 일

hook 이 잘 하는 일은 분명히 있다. **React 라이프사이클의 한 슬롯에 정확히 한 번** 일어나는 것 — `useEffect`, `useState`, `useRef`, `useMemo`. 그리고 그것을 *조합* 하는 것 — 그 조합이 한 컴포넌트의 한 라이프사이클 안에서만 의미가 있을 때.

react.dev 가 *좋은 custom hook 사례* 로 드는 셋을 보면 답이 또렷하다.

| 사례 | 묶이는 lifecycle | 도메인 결정 |
|---|---|---|
| `useOnlineStatus` | `online` / `offline` 이벤트 구독·해제 | (없음) |
| `useChatRoom` | 마운트/언마운트에 connect/disconnect | (없음) |
| `useFetch` | 요청 lifecycle, abort | (없음) |

셋 다 *lifecycle 그 자체* 를 묶는다. 도메인 결정은 hook 안에 없다.

hook 으로 하지 말아야 할 일은 그 반대편에 있다.

- **모듈 경계를 hook 으로 긋지 않는다.** 모듈은 함수와 타입의 묶음이다. hook 은 *그 위에서* 한 칸 더 묶을 때만 hook 이다.
- **도메인 로직을 hook 안에 박지 않는다.** 도메인 로직은 React 가 없어도 같은 답을 내야 한다. hook 은 React 의존이라 같은 답을 보장 못 한다.
- **같은 의존성은 hook 트리의 한 칸에서만 가져온다.** 자식 hook 이 자기 의존성을 *스스로 가져오기 시작* 하면, 부모와 두 번이 된다. 자식은 *받는다.*
- **`state 자체` 를 hook 안에 박지 않는다.** *react.dev 의 `share stateful logic, not state itself`* 가 hook 한 칸의 척추다. 전역 store / lifecycle reference counting 이 hook 안에 들어오는 순간, 그 hook 은 더 이상 *외형의 약속* 을 지키지 않는다는 걸 인지한다. *"의도적으로"* 라는 주석으로 변명할수록, 그 hook 은 다른 사람이 못 읽는 hook 이 된다.

# 그래서 — 다섯 줄을 다시 쓴다면

처음의 다섯 줄을 다시 쓰면 이렇다.

```ts
const handleCheckout = async () => {
  trackCheckoutAttempt(); // 로깅 트래커. 이름이 다르면 동명 충돌이 사라진다.

  const authOk = await ensureAuth();
  if (!authOk) return;

  const gateOk = await ensureGate();
  if (!gateOk) return;

  if (!validatePaymentMethodForm(selectedPaymentMethod)) return;
  await runOrder();
};
```

세 가지가 없어진다.

1. **HOF 의 HOF 의 HOF** — 콜백 wrapper 가 모두 사라지고, *조건 분기* 로 바뀐다. 한 가정에 한 줄이 붙는다.
2. **거짓말한 await** — 진짜 비동기인 `runOrder` 에만 `await` 가 붙는다.
3. **이름 충돌** — `trackCheckoutAttempt` (로깅 트래커) / `trackCheckoutClicked` (분석 트래커) — 이름이 두 일을 같이 하지 않는다.

`ensureAuth`, `ensureGate` 는 *함수* 다. hook 이 아니다. hook 은 그 위에 한 칸만, *라이프사이클 이벤트를 묶는 자리에서만* 쓰인다.

# 마치며 — 한 줄

오늘 적어 두고 싶은 한 줄.

> **hook 은 도메인 추상화가 아니다. lifecycle 의 한 슬롯이다.**
> **추상화는 함수와 타입으로 한다. hook 은 그 슬롯을 묶는 자리에서만, 한 칸 더 쓴다.**

다섯 줄에 일곱 층이 박힌 코드는, *그 일곱 층을 hook 이 추상해 줄 거라고 잠깐 믿었던* 자국이다. 다음에 같은 자리를 만나면, hook 의 외형이 보이는 순간 한 번 멈춰 보고 싶다.

> *이건 도메인의 단위인가, lifecycle 의 단위인가?*
> *이 hook 이 가진 state 는 stateful logic 인가, state 자체인가?*

전자가 *lifecycle 의 단위* 가 아니거나, 후자가 *state 자체* 면 — 그 자리는 hook 이 아니라 함수와 타입의 자리다.
