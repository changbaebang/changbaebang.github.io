---
layout: post
title: "while(true)는 돌고 있지 않았다 — 호출 스택, 힙, 태스크 큐, 마이크로태스크 큐, 그리고 누가 await를 깨우는가"
date: 2026-11-13 13:10:00 +0900
excerpt: "스트림을 읽는 while(true) 루프가 왜 UI를 막지 않는지 따라가다, await에서 멈춘 함수가 어디로 가는지, 기다리는 Promise는 큐에 있는지, 그 함수를 누가 깨우는지까지 왔다. 호출 스택, 힙, 태스크 큐, 마이크로태스크 큐가 각자 무엇을 맡는지 문서로 확인하며 정리한다."
categories: [Engineering]
tags: [JavaScript, Event Loop, Async, Promise, Browser, Node.js, Frontend]
---

[지난 글](https://changbaebang.github.io/2026-11-12-readablestream-fetch-body/)에서 스트림을 읽는 코드를 이렇게 정리했다.

```ts
while (true) {
  const { value, done } = await reader.read()
  if (done) break
  // 조각 처리
}
```

`while (true)`다. 무한 루프다. 그런데 이 루프가 도는 동안에도 버튼은 눌리고, 화면은 그려지고, 스크롤도 된다. 처음엔 "이거 스피닝 아닌가?"부터 걸렸다.

## 따라 나온 질문들

```text
1. await에서 함수는 어디로 가나? 호출 스택에서 빠지면 지역 변수도 사라지지 않나?
2. 기다리는 Promise는 큐에 들어가 있나?
3. 그럼 그 함수는 누가 깨우나?
4. 태스크와 마이크로태스크는 어떤 순서로 도나?
5. setTimeout의 콜백과 await로 기다리는 타이머는 무엇이 다른가?
6. 화면은 언제 그려지고, 언제 멈추나?
7. 워커도 안 만들었는데 왜 이게 되나?
```

C++과 Java에 익숙한 감각으로 읽으면 1번에서 바로 막힌다. 함수가 스택에서 빠지면 지역 변수도 같이 사라진다. 그런데 이 루프는 스택에서 빠졌다가 나중에 다시 이어서 돈다. 하나씩 확인한다.

## 등장하는 것들

| 이름 | 하는 일 |
| ---- | ------- |
| 호출 스택 | 지금 실행 중인 코드. JavaScript는 한 번에 하나만 실행한다 |
| 힙 | 객체와 클로저, 그리고 멈춘 함수의 상태가 머무는 곳 |
| 브라우저(Web API) | 네트워크, 타이머, 입력처럼 JavaScript 밖에서 기다리는 일을 맡는다 |
| 태스크 큐 | 바깥에서 온 일이 줄 선다. 응답 도착, 타이머 만료, 클릭 |
| 마이크로태스크 큐 | Promise의 후속 작업과 `await` 뒤의 코드가 줄 선다 |
| 이벤트 루프 | 스택이 비면 다음 일을 꺼내는 반복 장치 |

## 이벤트 루프의 규칙

4번 질문의 답부터 보면 나머지가 읽기 쉽다. HTML 표준의 [처리 모델](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)을 줄이면 이렇다. 태스크 하나를 실행하고 나면 [마이크로태스크 체크포인트](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)를 수행한다. [MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)의 표현으로는 "마이크로태스크 큐가 빌 때까지 마이크로태스크를 실행하고, 그다음 브라우저가 렌더링을 갱신할 수 있다".

```text
① 태스크 큐에서 태스크 하나를 꺼내 실행한다
② 마이크로태스크 큐가 빌 때까지 전부 실행한다
   (실행 중에 새로 생긴 마이크로태스크도 이번에 함께 처리된다)
③ 필요하면 화면을 그린다
④ ①로 돌아간다. 할 일이 없으면 기다린다
```

순서를 확인하는 고전적인 예다.

```js
console.log('1')
setTimeout(() => console.log('2'), 0)
Promise.resolve().then(() => console.log('3'))
queueMicrotask(() => console.log('4'))
console.log('5')
// 1 5 3 4 2
```

`1`과 `5`는 지금 실행 중인 코드라 바로 나온다. 그다음 ②에서 마이크로태스크 `3`, `4`를 전부 비운다. `setTimeout`의 콜백 `2`는 타이머 태스크라, 다음 바퀴의 ①에서야 실행된다. 지연을 `0`으로 줘도 마찬가지다.

## await에서 함수는 어디로 가나

1번 질문이다.

`async` 함수는 끝까지 실행돼야 돌아오는 함수가 아니다. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)은 이렇게 설명한다. "첫 `await` 식까지의 코드(그것을 포함해)는 동기로 실행된다." 그리고 `await`를 만나면 [그 값에 기대는 나머지 코드가 멈춘다](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await).

```ts
async function receive() {
  setup()                                   // A. 호출한 그 자리에서 동기로 실행
  const { value } = await reader.read()     // B. 여기서 멈춘다. 호출한 쪽에는 Promise가 돌아간다
  handle(value)                             // C. 나중에, 새 호출 스택에서 이어서 실행
}

receive()       // await를 붙이지 않아도 된다
nextThing()     // receive가 B에서 멈추면 바로 여기로 온다
```

그래서 `await` 없이 호출하고 지나가도 스택이 막히지 않는다. 함수는 B에서 Promise를 돌려주고 스택에서 빠진다.

그럼 지역 변수는 어디 있나. 멈춘 함수의 재개 지점과 지역 변수는 **호출 스택이 아닌 곳에 따로 보관된다.** 엔진마다 구현은 다르지만, 개념적으로는 힙에 있는 객체 하나다. 그 객체를 가리키는 쪽이 있는 한 사라지지 않는다.

C++20 코루틴과 같은 구조다. [cppreference](https://en.cppreference.com/cpp/language/coroutines)에 따르면 코루틴은 실행을 시작할 때 "코루틴 상태 객체를 `operator new`로 할당"하고, `co_await`에서 멈췄다가 핸들로 다시 이어 간다. JavaScript에서는 이 일을 엔진이 하고, 정리는 가비지 컬렉터가 한다.

| | 일반 함수 | `async` 함수 |
| -- | --------- | ------------- |
| 돌아오는 때 | 본문을 끝까지 실행한 뒤 | 첫 `await`에서 Promise를 돌려주며 |
| 지역 변수 | 스택 프레임과 함께 사라진다 | 멈춘 상태로 따로 보관된다 |
| 이어서 실행 | 없다 | 나중에 새 호출 스택에서 |

## 기다리는 Promise는 큐에 있나

2번 질문이다. 내가 가장 오래 헷갈린 부분이다. 답은 **아니다.**

큐에는 "지금 실행할 수 있는 일"만 들어간다. 아직 결과가 없는(pending) Promise는 실행할 수 있는 일이 아니다. 기다리는 동안 각자 있는 곳은 이렇다.

| 것 | 있는 곳 |
| -- | ------- |
| 일 자체 (통신, 타이머 재기) | 브라우저. JavaScript 밖 |
| pending Promise와 멈춘 함수의 상태 | 힙 |

그동안 태스크 큐와 마이크로태스크 큐에는 이 일과 관련된 것이 아무것도 없다.

Promise 자체가 줄을 서는 것도 아니다. Promise는 힙에 있는 객체이고, "결과가 나오면 이어서 할 일" 목록을 들고 있을 뿐이다. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)은 이렇게 쓴다. "기다리던 값이 resolve되면, 멈춘 코드를 이어서 실행하는 마이크로태스크가 하나 예약된다." 큐에 무언가가 생기는 것은 resolve되는 그 순간이다.

그리고 실행은 언제나 호출 스택 위에서 일어난다. 힙은 보관하는 곳이지 실행하는 곳이 아니다. 책장에 꽂힌 책과 책상에 펼친 책의 차이다.

## 누가 깨우나

3번 질문이다. 깨우는 것은 힙 안의 무엇도 아니고, **바깥에서 온 신호**다.

`reader.read()`를 기다리는 동안 서버에서 바이트가 오면 이런 사슬이 움직인다.

```text
서버가 바이트를 보낸다
  → OS가 받고, 브라우저의 네트워크 계층이 처리한다
  → 메인 스레드의 태스크 큐에 "도착" 태스크가 들어간다     ← 여기서 처음 큐에 무언가가 생긴다
  → 그 태스크가 실행되면서 read()의 Promise를 resolve한다
  → 멈춘 함수를 이어서 실행하는 마이크로태스크가 예약된다   ← 마이크로태스크 큐에 생긴다
  → 스택이 비면 그 마이크로태스크가 실행되고, await 뒤의 코드부터 이어서 돈다
  → 다음 await에서 다시 멈춘다
```

HTML 표준에도 이런 일을 위한 태스크 소스가 따로 있다. 이름이 [networking task source](https://html.spec.whatwg.org/multipage/webappapis.html#networking-task-source)이고, "네트워크 활동에 반응해 일어나는 기능"에 쓴다고 적혀 있다.

정리하면 **태스크는 깨우는 신호이고, 마이크로태스크는 깨어난 코드가 실행되는 자리다.** 처음 호출한 함수의 스택은 첫 `await`에서 이미 끝났다. 이어지는 실행은 매번 새 호출 스택에서 시작한다. 그래서 처음 호출한 쪽의 `try/catch`로는 나중에 나는 에러를 잡을 수 없다. 비동기 함수 안에 `try/catch`가 있어야 하는 이유다.

이제 처음의 질문으로 돌아가면, `while (true)`는 계속 돌며 확인하는 루프가 아니었다. 조각이 올 때마다 이벤트 루프가 깨워 한 바퀴씩 도는 루프였다. 기다리는 동안 CPU는 놀고, 그 틈에 클릭과 렌더링이 들어온다.

## 타이머는 조금 다르다

5번 질문이다.

| | 우리 코드가 실행되는 곳 |
| -- | --------------------- |
| `setTimeout(cb, ms)` | 만료 태스크가 `cb`를 직접 실행한다. 마이크로태스크를 거치지 않는다 |
| `await new Promise((r) => setTimeout(r, ms))` | 만료 태스크는 `r()`로 resolve만 한다. `await` 뒤의 코드는 마이크로태스크에서 실행된다 |

타이머를 거는 순간 큐에 들어가는 것도 아니다. 브라우저가 시간을 재다가 **만료됐을 때** 태스크를 넣는다. 그래서 `setTimeout(cb, 100)`은 "정확히 100ms 뒤"가 아니라 "최소 100ms 뒤"다. [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout)도 "실제 지연은 설정보다 길 수 있다"고 하고, 중첩된 호출이 5번을 넘으면 브라우저가 최소 4ms를 강제한다고 적어 두었다.

## 화면은 언제 그려지고, 언제 멈추나

6번 질문이다. 규칙의 ③을 보면 답이 나온다. 화면은 마이크로태스크 큐가 **다 비고 나서야** 그려질 수 있다.

UI가 멈추는 경우는 둘이다.

- **한 구간의 동기 코드가 길 때.** 큰 계산, 큰 렌더링. 그동안 다음 태스크도 렌더링도 오지 않는다.
- **마이크로태스크가 끝나지 않을 때.** 마이크로태스크가 계속 새 마이크로태스크를 만들면 ②가 끝나지 않고, ③에 닿지 못한다. 무거운 루프를 `async`로 감싸기만 해서는 UI가 풀리지 않는 이유다. `await`가 이미 resolve된 값을 기다려도 마이크로태스크 하나를 거칠 뿐, 태스크로 돌아가지는 않는다.

양보하려면 태스크를 한 번 거치게 해야 한다.

```ts
await new Promise((r) => setTimeout(r, 0))   // 흔한 방법
await scheduler.yield()                       // 새 API
```

[`scheduler.yield()`](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield)는 이 글을 쓰는 지금 MDN에서 Chrome, Edge, Firefox는 지원하고 Safari는 지원하지 않는 "Limited availability"로 표시돼 있다. 쓰려면 없을 때의 대안을 함께 둔다.

## 워커 없이 왜 되나

7번 질문이다. 기다리는 일은 JavaScript가 하지 않기 때문이다.

네트워크 통신, 타이머 재기, 입력 받기는 브라우저가 JavaScript 밖에서 한다. JavaScript는 결과가 오면 짧게 일하고 다시 비켜 준다. 그래서 스레드를 만들 필요도, 락을 걸 필요도 없다. 한 번에 하나만 실행되니 상태를 바꾸는 도중에 다른 코드가 끼어들 일도 없다.

[Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)가 필요한 것은 **JavaScript가 직접 오래 계산해야 할 때**다. 기다림이 아니라 계산이 문제일 때다.

대신 치르는 값이 있다. 동기 구간 안에서는 아무도 끼어들지 않지만, `await`와 `await` 사이에는 다른 코드가 돌 수 있다. `await` 앞에서 확인한 상태가 뒤에서는 바뀌어 있을 수 있다. 늦게 온 응답이 지금 화면을 덮어쓰는 버그가 대개 여기서 나온다.

## Node.js는 조금 다르다

같은 원리지만 차이가 있다. Node.js의 이벤트 루프는 단계가 더 나뉘어 있고, Promise와 따로 도는 `process.nextTick` 큐가 있다. [Node.js API 문서](https://nodejs.org/api/process.html)는 이렇게 적는다. "next tick 큐를 비울 때마다 마이크로태스크 큐도 곧바로 이어서 비운다." 그리고 "`process.nextTick()` 콜백은 항상 `queueMicrotask()` 콜백보다 먼저 실행된다."

브라우저에서 확인한 순서를 그대로 Node.js에 옮겨 생각하면 틀릴 수 있다.

서버에서는 이 구조의 장단점이 더 크게 드러난다. 요청을 받아 DB나 다른 API를 기다리는 일이 대부분인 서버에는 잘 맞는다. 기다리는 동안 같은 스레드가 다른 요청을 처리하니, 요청마다 스레드를 두지 않아도 된다. 반대로 한 요청이 동기로 오래 계산하면, 그동안 **모든 사용자의 요청이 함께 멈춘다.** 브라우저에서는 화면 하나가 멈추는 일이 서버에서는 서비스 전체의 지연이 된다. 무거운 계산은 [`worker_threads`](https://nodejs.org/api/worker_threads.html)로 옮기거나 다른 서비스로 뺀다. `await` 사이에 다른 요청의 코드가 끼어든다는 점도 같아서, 요청들이 함께 쓰는 상태는 브라우저에서보다 더 조심해야 한다.

## 다음에 비동기 코드를 읽을 때 먼저 볼 것

```text
1. 이 await에서 함수는 멈추고, 호출한 쪽은 이미 다음 줄로 갔다
   → 에러는 이 함수 안에서 잡아야 한다. 호출한 쪽의 try/catch는 닿지 않는다.

2. 지금 기다리는 것은 무엇이 깨우는가
   → 네트워크, 타이머, 사용자 입력처럼 바깥 신호가 태스크를 넣어야 이어진다.
     아무도 resolve하지 않는 Promise를 기다리면 영영 이어지지 않는다.

3. await 앞뒤로 상태가 바뀔 수 있는가
   → 사이에 다른 코드가 돈다. 결과를 쓰기 전에 아직 유효한지 확인한다.

4. 마이크로태스크만으로 오래 도는 구간이 있는가
   → 렌더링이 오지 않는다. 태스크를 한 번 거치게 양보한다.

5. 계산이 무거운가, 기다림이 긴가
   → 기다림이면 그대로 두고, 계산이면 워커를 생각한다.
```

## 마치며

`while (true)`가 무한 루프처럼 보여서 시작한 질문이었다. 따라가 보니 그 루프는 거의 언제나 멈춰 있었다. 멈춘 함수는 스택에도 큐에도 없이 힙에 보관돼 있었고, 그것을 깨우는 것은 바깥에서 온 신호 하나였다. 그 신호가 태스크로 들어와 Promise를 풀고, 풀린 자리에서 마이크로태스크가 코드를 이어 갔다.

C++이나 Java에서는 동시에 하려면 스레드를 만들었다. JavaScript는 기다리는 동안 비켜 주는 방식으로 같은 일을 한다. 스레드를 관리하지 않아도 되는 대신, 비켜 준 사이에 무엇이 바뀌었을지는 늘 내가 생각해야 한다. 코드는 한 줄로 이어져 보여도, 실제로는 `await`마다 끊겼다가 다른 스택에서 다시 이어진다.

`await` 하나를 지날 때마다 함수는 잠깐 사라졌다가 돌아온다. 그걸 알고 나니 무한 루프가 무섭지 않아졌다.

## 참고

- WHATWG, [HTML Standard: Event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- MDN, [JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model), [Using microtasks in JavaScript with queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- MDN, [async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function), [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- MDN, [setTimeout()](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout), [Scheduler.yield()](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield), [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- Node.js, [process.nextTick()와 queueMicrotask()](https://nodejs.org/api/process.html), [The Node.js Event Loop](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick), [Worker threads](https://nodejs.org/api/worker_threads.html)
- cppreference, [Coroutines (C++20)](https://en.cppreference.com/cpp/language/coroutines)
