---
layout: post
title: "마이크로태스크는 왜 렌더링을 양보하지 않는가 — 브라우저에서는 화면이, 서버에서는 모든 요청이 멈춘다"
date: 2026-11-13 16:00:00 +0900
excerpt: "Promise와 async/await는 비동기 코드를 동기처럼 읽히게 만들었지만, 마이크로태스크 큐가 빌 때까지 멈추지 않는다는 규칙 때문에 성능의 새 함정이 되기도 한다. 브라우저에서는 화면이, Node.js 서버에서는 모든 사용자의 요청이 같은 방식으로 멈춘다. 언제 막히는지, 어떻게 양보하고 옮기고 재는지를 문서로 확인하며 정리한다."
categories: [Engineering]
tags: [JavaScript, Event Loop, Performance, Microtask, Node.js, Web Worker, Frontend]
---

[지난 글](https://changbaebang.github.io/2026-11-12-event-loop-who-wakes-await/)에서 이벤트 루프의 규칙을 이렇게 줄였다.

```text
① 태스크 큐에서 태스크 하나를 꺼내 실행한다
② 마이크로태스크 큐가 빌 때까지 전부 실행한다
③ 필요하면 화면을 그린다
④ ①로 돌아간다
```

②의 "빌 때까지"라는 말은 지나가듯 썼다. 이 글은 거기서 출발한다. 편하게 쓰는 `await`가 이 규칙 때문에 어디서 성능을 해칠 수 있는지, 그리고 JavaScript가 서버에서도 도는 지금은 그 피해가 어떻게 달라지는지를 본다.

## 따라 나온 질문들

```text
1. 마이크로태스크가 끝나지 않으면 정말 화면이 안 그려지나?
2. 개수가 많으면 막히는 건가, 오래 걸리면 막히는 건가?
3. 그 순서는 JavaScript 엔진이 정하나?
4. 서버(Node.js)에서는 같은 일이 어떻게 나타나나?
5. 양보하려면, 옮기려면, 재려면 무엇을 쓰나?
```

## 마이크로태스크는 다 비울 때까지 끼어들지 못한다

1번 질문이다. 규칙 ②의 의미를 풀어 쓰면 이렇다. 마이크로태스크를 하나 실행하다가 새 마이크로태스크가 생기면, 그것도 같은 ②에서 실행한다. 큐가 완전히 빌 때까지 ③(렌더링)과 다음 태스크(클릭, 타이머)는 순서를 얻지 못한다.

[MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)은 이를 "마이크로태스크가 빌 때까지 실행되고, 그다음에야 브라우저가 렌더링을 갱신할 수 있다"고 적는다. 이 규칙은 HTML 표준의 [이벤트 루프 처리 모델](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)에 있다.

그래서 이런 코드는 UI를 얼린다.

```js
function spin() {
  Promise.resolve().then(spin)   // 끝나자마자 다음 마이크로태스크를 만든다
}
spin()
```

`setTimeout(spin, 0)`으로 바꾸면 매번 태스크를 거치고, 그 사이에 렌더링과 클릭이 들어온다. 같은 "비동기"라도 어느 큐로 이어지느냐가 다르다.

### 마이크로태스크는 큐에서 시작하지 않는다

위의 ①②③④는 지난 글에서 이해하기 쉽게 그린 모양이다. 더 정확히는 마이크로태스크가 **루프의 맨 위에서 시작하지 않는다.** HTML 표준의 ["스크립트 실행 뒤 정리"](https://html.spec.whatwg.org/multipage/webappapis.html#clean-up-after-running-script) 알고리즘은 이렇게 적는다. 실행 컨텍스트를 JavaScript 실행 컨텍스트 스택에서 제거하고, "스택이 이제 비었다면 마이크로태스크 체크포인트를 수행한다." 마이크로태스크를 비우는 시점은 **호출 스택이 비는 순간**이다.

그리고 태스크 큐가 비었는데 마이크로태스크만 생기는 일은 일어나지 않는다. 마이크로태스크를 큐에 넣는 알고리즘은 "병렬로(in parallel) 호출되지 않는다"는 단언으로 시작한다. 브라우저의 네트워크 같은 백그라운드가 마이크로태스크를 직접 넣을 수 없고, 태스크를 하나 넣어 그 안에서 JavaScript가 돌게 해야 한다. 마이크로태스크는 언제나 어떤 코드가 실행되던 도중에 생긴다.

이 규칙은 눈에 보이는 차이를 만든다. 같은 버튼에 리스너 두 개를 달고, 첫 리스너에서 마이크로태스크를 예약했다.

```js
b.addEventListener('click', () => { log('listener 1'); Promise.resolve().then(() => log('microtask')) })
b.addEventListener('click', () => { log('listener 2') })
```

| 어떻게 눌렀나 | 실행 순서 |
| ------------- | --------- |
| 실제 클릭 | listener 1 → **microtask** → listener 2 |
| 스크립트에서 `b.click()` | listener 1 → listener 2 → **microtask** |

Chromium, Firefox, WebKit(Playwright)에서 모두 이렇게 나왔다. 실제 클릭은 브라우저가 리스너를 하나씩 부르니 리스너가 끝날 때마다 스택이 비어 마이크로태스크가 끼어든다. `b.click()`은 그것을 부른 코드가 스택에 남아 있어서, 그 코드가 끝날 때까지 스택이 비지 않는다. 표준이 이 알고리즘을 "스크립트가 다른 스크립트를 직접 부르는 경우에는 호출되지 않는다"고 적은 이유다.

## 개수가 아니라 걸린 시간이다

2번 질문이다. 마이크로태스크가 수천 개 있다는 사실만으로는 문제가 아니다. 하나하나가 짧으면 금방 끝난다. 막히는 것은 **합친 시간이 한 프레임이 쓸 수 있는 시간을 넘을 때**다.

기준이 되는 숫자가 둘 있다.

- **프레임 예산:** 화면 주사율에 따라 다르다. 60Hz면 한 프레임이 약 16.7ms이고, 120Hz면 약 8.3ms다. 이 안에 스크립트, 스타일, 레이아웃, 그리기가 들어가야 한다.
- **긴 작업(long task):** [MDN](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceLongTaskTiming)은 메인 스레드를 50ms 이상 막는 작업을 long task라 부르고, 이것이 상호작용 가능 시점을 늦추는 등의 문제를 만든다고 설명한다.

예를 들어 키를 눌렀을 때 핸들러에서 일어나는 일이 이렇다고 하자.

```js
input.addEventListener('keydown', async () => {
  const items = await loadItems()            // 네트워크 응답이 오면 이어서 실행
  for (const item of items) {                // 수천 개
    await process(item)                      // 이미 끝난 Promise여도 마이크로태스크 하나를 거친다
  }
})
```

`process`가 이미 끝난 값을 돌려줘도 `await`는 마이크로태스크 하나를 거친다(MDN의 [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await) 문서가 이 점을 적고 있다). 외부 I/O가 없으면 태스크로 돌아갈 틈이 없어서, 수천 번의 `await`가 마이크로태스크 큐만 왕복하며 메인 스레드를 쓴다. 각 반복이 0.05ms씩만 걸려도 합쳐서 수십 ms가 되면 그 키 입력 뒤의 화면 갱신이 그만큼 밀린다.

직렬로 `await`하는 경우만 그런 것이 아니다. 병렬로 걸어 둔 Promise가 **한 태스크 안에서 한꺼번에 풀릴 때**도 같다. 하나의 Promise에 `.then`을 20만 개 걸어 두고 같은 태스크에서 resolve하면, 그 콜백이 모두 실행된 뒤에야 `setTimeout(0)` 콜백이 실행된다. 이미 끝난 Promise 20만 개를 `Promise.all`에 넣어도 마찬가지다. 직접 Node.js(v22)에서 돌려 본 결과, 앞의 경우 콜백이 모두 도는 데 약 120ms가 걸렸다. 반대로 네트워크 요청 수천 개를 `Promise.all`로 묶었다면 응답이 제각각 다른 태스크로 도착해서 한꺼번에 몰리지 않는다. 몰리는 것은 이미 끝난 값이거나 하나의 Promise를 여럿이 기다리다 같이 풀릴 때다.

`await`가 있다고 해서 UI에 양보하는 것이 아니다. **양보는 태스크를 거칠 때만 일어난다.**

## 그 순서는 누가 정하나

3번 질문이다. "마이크로태스크가 태스크보다 먼저"라는 순서를 JavaScript 엔진이 정한다고 생각하기 쉽다. 그렇지 않다.

ECMAScript 명세는 Promise 후속 작업(job)을 실행하는 시점을 직접 정하지 않고, 호스트 환경에 맡긴다. [명세](https://tc39.es/ecma262/)에는 `HostEnqueuePromiseJob`이라는 호스트 훅이 있고, job을 "ECMAScript 호스트 환경이 일정에 올린다"고 적혀 있다. 브라우저는 HTML 표준의 처리 모델대로, Node.js는 자기 이벤트 루프대로 이것을 구현한다.

이 구분이 서버를 이야기할 때 중요해진다. 같은 JavaScript 코드라도 어디서 도느냐에 따라 순서와 양보할 지점이 달라진다.

## 서버에서는 같은 함정이 이렇게 나타난다

4번 질문이다. [Node.js 문서](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)는 서버가 확장되는 이유를 이렇게 설명한다. Node.js는 적은 수의 스레드로 많은 클라이언트를 처리하고, 그래서 요청 하나가 할 일이 "작아야" 빠르다. 어림 규칙은 이렇다. "각 클라이언트와 관련된 일이 어느 순간이든 '작을' 때 Node.js는 빠르다."

브라우저에서 화면 하나가 멈추던 것이, 서버에서는 **같은 이벤트 루프를 쓰는 모든 요청이 함께 멈추는 것**으로 나타난다.

```js
app.get('/report', async (req, res) => {
  const rows = await db.query(...)       // 여기서는 다른 요청이 처리된다
  const result = heavyCompute(rows)      // 동기로 오래 돈다: 그동안 다른 사용자의 요청은 전부 대기
  res.json(result)
})
```

`await db.query`를 기다리는 동안에는 이벤트 루프가 다른 요청을 처리한다. 하지만 `heavyCompute`가 도는 동안에는 아무도 끼어들지 못한다. 한 사용자의 무거운 요청이 다른 모든 사용자의 응답 시간을 늘린다.

### nextTick은 더 세다

[Node.js 문서](https://nodejs.org/api/process.html)에 따르면 `process.nextTick()`의 콜백은 `queueMicrotask()`의 콜백보다 항상 먼저 실행된다. 그리고 [nextTick 해설](https://nodejs.org/learn/asynchronous-work/understanding-processnexttick)은 재귀적인 `process.nextTick()` 호출로 I/O를 "굶길(starve)" 수 있다고 경고한다.

```js
function loop() {
  process.nextTick(loop)   // I/O 콜백이 차례를 얻지 못한다
}
loop()
```

브라우저에서 마이크로태스크가 렌더링을 굶기는 것과 같은 모양이다. 굶는 대상이 렌더링에서 네트워크와 디스크 I/O로 바뀐다.

### 서버에서 양보하는 곳

Node.js에서도 `setTimeout(0)`은 쓸 수 있다. 다만 서버에는 양보하는 용도로 더 알맞은 `setImmediate()`가 있다. [Node.js 이벤트 루프 문서](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)는 이것이 "현재 poll 단계가 끝난 뒤 스크립트를 실행하도록 설계됐다"고 설명한다. poll 단계(I/O 콜백을 처리하는 단계)가 끝나면 이벤트 루프가 check 단계로 넘어가 `setImmediate()`로 예약된 스크립트를 실행하니, 긴 작업을 나눠 돌릴 때 조각 사이에 I/O가 끼어들 수 있다.

둘의 차이는 어디서 부르느냐에 따라 나타난다. 같은 문서는 `setImmediate()`가 I/O 사이클 안에서 예약되면 타이머가 몇 개 있든 항상 타이머보다 먼저 실행된다고 적는다. 반대로 메인 모듈에서 둘을 부르면 순서가 프로세스의 성능에 좌우된다. 직접 돌려 보니 `fs.readFile`의 콜백 안에서 둘을 예약했을 때 20번 모두 `setImmediate`가 먼저였다. 그래서 요청을 처리하는 콜백 안에서 양보할 때는 `setImmediate()`가 더 예측하기 쉽다.

### 이미 다른 스레드에서 도는 것

Node.js는 이벤트 루프 스레드 하나만 쓰는 것이 아니다. [문서](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)에 따르면 운영체제가 논블로킹 버전을 제공하지 않는 I/O와 CPU를 많이 쓰는 작업은 libuv로 구현된 워커 풀에서 처리된다. 그 덕에 파일 I/O 같은 일이 이벤트 루프를 막지 않는다. 직접 짠 계산 코드는 이 풀에 올라가지 않고 이벤트 루프 위에서 그대로 돈다.

## 양보하고, 옮기고, 재기

5번 질문이다. 두 환경을 나란히 놓으면 선택지가 대응된다.

| | 브라우저 | Node.js 서버 |
| -- | -------- | ------------ |
| **양보하기** (같은 스레드에서 나눠 돌리기) | `scheduler.yield()`, `setTimeout(0)` | `setImmediate()` (`setTimeout(0)`도 가능) |
| **옮기기** (다른 스레드로) | Web Worker | `worker_threads` |
| **재기** | Long Tasks (`PerformanceLongTaskTiming`) | `perf_hooks.monitorEventLoopDelay()` |

Node.js 문서도 계산이 무거울 때의 선택지를 같은 말로 정리한다. "분할(partitioning) 또는 위임(offloading)"이다.

### 양보하기: 일을 쪼개고 사이에 태스크를 거친다

```ts
async function processAll(items: Item[]) {
  let last = performance.now()
  for (const item of items) {
    process(item)
    if (performance.now() - last > 8) {        // 한 번에 8ms 넘게 쓰지 않는다
      await new Promise((r) => setTimeout(r, 0))   // 태스크를 거쳐 렌더링과 입력에 양보
      last = performance.now()
    }
  }
}
```

두 가지를 알아 둔다.

- [`scheduler.yield()`](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield)는 작업 도중 메인 스레드에 양보하고 나중에 이어서 실행하는 새 API다. MDN에 따르면 이어서 실행하는 부분은 우선순위가 높은 태스크로 예약된다. 다만 Chrome, Edge, Firefox는 지원하고 Safari는 지원하지 않는 "Limited availability"로 표시돼 있다. 쓰려면 없을 때 `setTimeout`으로 대신하는 분기를 둔다.
- `setTimeout(0)`을 너무 자주 쓰면 느려진다. [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout)은 중첩된 호출이 5번을 넘으면 브라우저가 최소 4ms를 강제한다고 적는다. 조각을 아주 잘게 내면 일 자체보다 대기 시간이 더 길어진다. 그래서 위 예처럼 시간을 재서 일정 시간마다 한 번씩만 양보한다.

### 옮기기: 계산이 무거우면 스레드를 바꾼다

양보는 같은 스레드를 나눠 쓰는 일이라 전체 계산 시간을 줄이지 못한다. 계산 자체가 무거우면 다른 스레드로 옮긴다.

- **브라우저:** [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)로 옮긴다. 단, 워커 안에서는 DOM을 직접 건드릴 수 없다. MDN이 "워커 안에서 DOM을 직접 조작할 수 없다"고 적고 있어서, 계산 결과만 메시지로 돌려받아 메인 스레드에서 화면에 반영한다.
- **서버:** [`worker_threads`](https://nodejs.org/api/worker_threads.html)나 별도 서비스로 옮긴다.

옮기는 데에도 값이 든다. MDN에 따르면 메인 스레드와 워커 사이에 주고받는 데이터는 공유되지 않고 **복사되거나 이동(transfer)된다.** 크기가 큰 데이터를 자주 주고받으면 그 비용이 커서, 계산이 짧으면 옮기는 비용이 더 클 수 있다.

### 재기: 막혔는지 숫자로 본다

짐작으로 고치지 않으려면 잰다.

- **브라우저:** `PerformanceObserver`를 만들어 `observe()`에 `type: "longtask"`를 넘기면, long task(50ms 이상)마다 `PerformanceLongTaskTiming` 객체를 받는다. 어느 작업이 얼마나 막았는지 볼 수 있다.
- **서버:** [`perf_hooks.monitorEventLoopDelay()`](https://nodejs.org/api/perf_hooks.html)로 이벤트 루프 지연의 분포를 히스토그램으로 잰다. 값은 나노초 단위이고, 평균과 최댓값 같은 통계를 준다. 지연이 길수록 다른 요청이 기다린 시간이 길다는 뜻이다.

## 어떤 도구가 어떤 큐를 타는가

글 1에서 본 구조를 성능 관점으로 다시 묶으면 이렇다.

| 도구 | 이어지는 곳 | UI(또는 I/O)에 양보하나 |
| ---- | ----------- | ----------------------- |
| `Promise.then`, `await` | 마이크로태스크 | 아니다. 큐가 빌 때까지 이어진다 |
| `setTimeout(0)` | 태스크 | 그렇다. 렌더링과 입력이 끼어들 수 있다 |
| `scheduler.yield()` | 태스크(양보 뒤 우선) | 그렇다 |
| `process.nextTick` (Node.js) | nextTick 큐 | 아니다. 마이크로태스크보다 먼저 비워진다 |
| `setImmediate` (Node.js) | 이벤트 루프의 check 단계 | 그렇다. I/O 콜백 다음에 실행된다 |

`function*`는 이 표에 없다. 제너레이터는 큐와 무관하게 동기로 멈췄다 이어 가는 도구라서, 그 자체로는 양보를 만들지 않는다. 다만 `yield` 지점마다 위의 양보 방법과 결합하면 긴 작업을 쪼개는 데 쓸 수 있다. 이 이야기는 [다음 글](https://changbaebang.github.io/2026-11-13-why-async-generator/)에서 한다.

## 다음에 성능이 걱정될 때 먼저 볼 것

```text
1. 이 await 사이에 태스크가 한 번이라도 끼는가
   → 네트워크나 타이머를 기다리지 않으면 마이크로태스크만 왕복한다. 화면과 I/O가 양보를 받지 못한다.

2. 한 번에 이어서 도는 시간의 합이 얼마인가
   → 개수가 아니라 합이다. 브라우저는 프레임 예산, 서버는 다른 요청의 대기 시간.

3. DOM이나 요청 처리와 얽혀 있는가
   → 얽혀 있으면 양보하며 쪼갠다. 얽혀 있지 않은 순수 계산이면 옮긴다.

4. 옮긴다면 주고받는 데이터가 얼마인가
   → 계산이 짧으면 옮기는 비용이 더 클 수 있다.

5. 재 봤는가
   → 브라우저는 long task, 서버는 이벤트 루프 지연. 짐작으로 고치지 않는다.
```

## 마치며

`async/await`는 비동기 코드를 동기처럼 읽게 해 주었다. 읽기는 쉬워졌지만, 실행은 달라지지 않았다. 마이크로태스크는 큐가 빌 때까지 끼어들지 못한다는 규칙이 그대로 있고, 그 규칙은 `await`가 편해 보일수록 눈에 덜 띈다.

JavaScript가 서버에서도 돌게 되면서 이 규칙의 값이 커졌다. 브라우저에서는 한 사용자의 화면이 멈추는 일이 서버에서는 같은 프로세스를 쓰는 모든 사용자의 지연으로 번진다. 같은 코드가 어디서 도느냐에 따라 막히는 대상도, 양보하는 도구도 다르다.

부드러운 화면과 빠른 응답은 엔진이 자동으로 보장해 주지 않는다. 어디서 태스크를 거쳐 숨 쉴 틈을 줄지, 어디서 다른 스레드로 넘길지를 정하는 일은 코드를 쓰는 사람의 몫으로 남아 있다.

## 참고

- WHATWG, [HTML Standard: Event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- ECMA-262, [ECMAScript Language Specification](https://tc39.es/ecma262/) (`HostEnqueuePromiseJob`)
- MDN, [Using microtasks in JavaScript with queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide), [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await), [PerformanceLongTaskTiming](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceLongTaskTiming)
- MDN, [Scheduler.yield()](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield), [setTimeout()](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout), [Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- Node.js, [Don't Block the Event Loop](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop), [The Node.js Event Loop](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick), [Understanding process.nextTick()](https://nodejs.org/learn/asynchronous-work/understanding-processnexttick)
- Node.js, [process.nextTick()와 queueMicrotask()](https://nodejs.org/api/process.html), [perf_hooks](https://nodejs.org/api/perf_hooks.html), [Worker threads](https://nodejs.org/api/worker_threads.html)
