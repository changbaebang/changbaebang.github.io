---
layout: post
title: "async function*는 왜 쓰는가 — 청크로 오는 세상을 메시지 단위 반복자로 바꾸는 법"
date: 2026-11-13 17:30:00 +0900
excerpt: "async function*를 처음 봤을 때 *가 함수 포인터처럼 보였고, 호출해도 본문이 실행되지 않았다. 동작을 외우기 전에 물었다. 스트림을 읽는 코드를 왜 제너레이터로 쓰나. 기능별로 나눠 쓰는 쪽은 반복자로만 받는 구조, 지역 변수가 되는 상태, 소비 속도가 스트림까지 전달되는 당김, 그리고 끝까지 따라오는 정리를 직접 돌려 보며 정리한다."
categories: [Engineering]
tags: [JavaScript, Generator, Async Generator, Iterator, Streams, Backpressure, Frontend]
---

[첫 글](https://changbaebang.github.io/2026-11-12-readablestream-fetch-body/)에서 스트림을 읽는 코드를 `async function*` 하나에 가뒀다. 쓰는 쪽은 `for await`로 이벤트를 하나씩 받았다. [두 번째 글](https://changbaebang.github.io/2026-11-12-event-loop-who-wakes-await/)에서는 멈춘 함수가 어디 있고 누가 깨우는지를 봤고, [세 번째 글](https://changbaebang.github.io/2026-11-12-microtask-does-not-yield-to-rendering/)의 끝에서는 제너레이터가 양보를 대신 해 주지는 않는다고 미뤄 두었다.

처음 `async function*`를 봤을 때는 `*`가 함수 포인터처럼 보였고, 호출해도 본문이 실행되지 않았다. `next()`, `yield`, `return()` 같은 동작은 찾아보면 나온다. 그런데 동작을 알아도 남는 질문이 있었다. **스트림을 읽는 코드를 왜 굳이 제너레이터로 쓰나.** 콜백으로도 되는 일이다.

## 따라 나온 질문들

```text
1. 스트림에서 오는 것과 쓰는 쪽이 원하는 것은 왜 다른가?
2. 콜백으로 쓰면 무엇이 불편한가?
3. 기능별로 나눠 놓으면 쓰는 쪽은 어떻게 보이나?
4. 소비하는 속도는 어디까지 전달되나?
5. 중간에 그만두면 정리는 누가 하나?
6. 그 동작들은 실제로 어떻게 움직이나?
7. 이 구조가 해 주지 않는 일은 무엇인가?
```

## 오는 것과 원하는 것이 다르다

1번 질문이다. `fetch`의 `response.body`에서 나오는 것은 **바이트 조각(청크)**이다. 쓰는 쪽이 원하는 것은 **이벤트 하나**다. 둘의 경계는 맞지 않는다.

```text
서버가 보낸 것:   data: {"a":1}\n\n  data: {"b":2}\n\n
받은 청크:        'data: {"a":1}\n\ndata: {"b'   |   '":2}\n\n'
```

한 청크에 이벤트가 여러 개 들 수도, 한 이벤트가 둘로 쪼개질 수도 있다. 그래서 중간에 **남은 조각을 들고 있는** 일이 필요하다. 청크 단위의 세상을 이벤트 단위의 세상으로 바꿔 주는 어댑터가 있어야 한다.

이 어댑터를 어떤 모양으로 만들지가 이 글의 질문이다.

## 콜백으로 쓰면, 상태가 바깥에 남는다

2번 질문이다. 같은 일을 콜백으로 쓰면 이렇게 된다. 청크가 올 때마다 불리는 함수가 있고, 남은 조각과 "끝났는가"를 함수 **바깥**에 둔다.

```js
function createParser(onEvent, onDone) {
  let buffer = ''          // 남은 조각: 호출 사이에 살아남아야 해서 바깥에 있다
  let finished = false     // 끝 표시를 받았는가: 이후 청크를 무시하려고 바깥에 있다
  return {
    onChunk(chunk) {
      if (finished) return
      buffer += chunk
      let i
      while ((i = buffer.indexOf('\n\n')) !== -1) {
        const block = buffer.slice(0, i); buffer = buffer.slice(i + 2)
        const data = toData(block)
        if (!data) continue
        if (data === '[DONE]') { finished = true; onDone(); return }
        onEvent(JSON.parse(data))
      }
    },
  }
}
```

같은 일을 `async function*`로 쓰면 이렇게 된다.

```js
async function* parse(chunks) {
  let buffer = ''                          // 그냥 지역 변수다
  for await (const chunk of chunks) {
    buffer += chunk
    let i
    while ((i = buffer.indexOf('\n\n')) !== -1) {
      const block = buffer.slice(0, i); buffer = buffer.slice(i + 2)
      const data = toData(block)
      if (!data) continue
      if (data === '[DONE]') return        // finished 플래그가 필요 없다. 끝내면 된다
      yield JSON.parse(data)
    }
  }
}
```

조각을 어떻게 끊어 보내도 두 쪽의 결과가 같은지 확인했다. 청크를 `'data: {"a":1}\n'`, `'\ndata: {"b"'`, `':2}\n\nda'`처럼 일부러 어색한 자리에서 나눠 넣었다.

```text
callback version : [{"a":1},{"b":2},{"c":3},"DONE"]
generator version: [{"a":1},{"b":2},{"c":3},"DONE"] | same result: true
```

결과는 같다. 차이는 **상태가 어디에 있는가**다. 콜백은 호출과 호출 사이에 값을 남기려고 `buffer`와 `finished`를 바깥으로 빼야 하고, "지금 어느 단계인가"를 변수로 관리해야 한다. 제너레이터는 `yield`에서 멈춰도 지역 변수가 그대로 남는다. 상태 기계를 손으로 짜지 않고, **코드 흐름이 곧 상태**가 된다. 파싱이 단계가 늘어날수록(헤더, 본문, 끝 표시) 이 차이는 커진다.

## 기능별로 나누고, 쓰는 쪽은 반복자로만 본다

3번 질문이다. 이 구조의 쓸모는 여기서 나온다. 내부 구현을 기능 단위로 쪼개 제너레이터로 만들면, 쓰는 쪽에서는 **하나의 반복자**로 보인다.

```js
async function* chunks(stream)  { /* ReadableStream → 문자열 조각, 연결은 여기서 닫는다 */ }
async function* messages(chunks) { /* 조각 → 메시지. 남은 조각은 여기서 들고 있는다 */ }
async function* events(messages) { /* 메시지 → 이벤트 객체. 끝 표시는 여기서 처리한다 */ }

for await (const event of events(messages(chunks(stream)))) {
  render(event)             // 쓰는 쪽은 바이트도, 남은 조각도, 끝 표시도 모른다
}
```

각 단계는 앞 단계의 반복자를 받아 자기 일만 하고 또 하나의 반복자를 내놓는다. 단계를 바꾸거나 끼워 넣어도 쓰는 쪽 코드는 그대로다. 디코딩을 바꾸든(`TextDecoderStream`), 메시지 형식을 바꾸든(SSE에서 줄 단위 JSON으로), 쓰는 쪽은 `for await`뿐이다.

이 구조가 동작을 보여 주는 데서 끝나지 않는 이유는, 모양을 나눈 것 말고도 두 가지가 **같이 따라오기** 때문이다. 다음 두 절이 그 둘이다. 하나는 소비하는 속도가 전달되는 것이고, 하나는 정리가 따라오는 것이다.

## 소비하는 속도가 스트림까지 전달된다

4번 질문이다. `for await`는 값이 필요할 때 `next()`를 부르고, 그 요청은 사슬을 타고 거슬러 올라가 마지막에는 `reader.read()`까지 닿는다. 소비하는 쪽이 느리면 `next()`를 늦게 부르고, 그러면 `read()`도 늦게 불린다. **읽는 속도가 소비하는 속도에 묶인다.**

이 효과를 직접 재 봤다. 생산자는 호출될 때마다 청크를 하나 만드는 `ReadableStream`이고, 소비자는 항목당 50ms가 걸리는 느린 쪽이다. 소비자가 8개를 처리하는 동안 생산자에게 몇 번 요청이 갔는지 셌다.

| 읽는 방식 | 소비자가 처리한 수 | 생산자에게 간 요청 | 메모리에 쌓인 것 |
| --------- | ------------------ | ------------------ | ---------------- |
| `for await` + 제너레이터 (당김) | 8 | **9** | 거의 없음 |
| 읽을 수 있는 대로 읽어 큐에 쌓기 (밀어 넣기) | 8 | **143,096** | 143,087 |

당김 구조에서는 소비한 8개보다 하나 많은 9번만 요청됐다. 읽을 수 있는 대로 읽는 쪽은 소비자가 8개를 처리하는 사이 14만 개를 쌓았다. 이 값은 Node.js(v22)의 웹 스트림으로 약 0.4초 동안 돌린 결과이고, 일부러 가능한 한 빨리 읽는 극단적인 읽기 루프를 비교용으로 썼다. 기기마다 숫자는 다르다. 요점은 숫자가 아니라, **읽는 일이 소비하는 일과 묶여 있느냐**다.

콜백(`onmessage`처럼 오는 대로 부르는 구조)은 소비하는 쪽이 느려도 오는 속도를 늦출 방법이 없다. 늦추려면 직접 멈추고 재개하는 장치를 짜야 한다. 반복자는 그것을 구조로 해결한다. 소비자가 `next()`를 부를 때만 다음이 일어난다.

## 정리도 끝까지 따라온다

5번 질문이다. 쓰는 쪽이 중간에 그만두면 어떻게 되나. 단계 세 개를 이은 파이프라인에서 소비자가 1000개 중 3개만 받고 `break`했다.

```js
for await (const e of parse(split(source(1000)))) { out.push(e.n); if (out.length === 3) break }
```

```text
consumer got [1,2,3] | cleanup ran in each stage: source > split > parse
```

`break`가 가장 바깥 반복자에 `return()`을 부르고, 그 요청이 안쪽으로 전달돼 **세 단계의 `finally`가 모두 실행됐다.** 연결을 닫는 일을 가장 안쪽 단계의 `finally`에 한 번 써 두면, 쓰는 쪽이 정리를 따로 쓰지 않아도 어떻게 끝나든 그 코드가 돈다. 첫 글에서 `finally`에 `reader.cancel()`을 둔 이유다.

`for...of`도 `break`하면 `return()`을 부른다. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)은 `return()`이 "호출자가 반복을 마쳤음을 반복자에게 알리고, 필요한 정리(예: 데이터베이스 연결 닫기)를 하게 하는" 메서드라고 설명한다.

## 그 동작들은 이렇게 움직인다

6번 질문이다. 위의 구조가 가능한 것은 제너레이터가 이렇게 동작하기 때문이다.

### 호출해도 본문은 실행되지 않는다

`*`는 포인터가 아니라 **제너레이터 함수**라는 표시다. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function%2A)은 제너레이터 함수를 부를 때마다 "반복자 프로토콜을 따르는 새 Generator 객체를 돌려준다"고 설명하고, 그 실행이 "함수 본문의 맨 처음에서 멈춘 상태로 시작한다"고 적는다.

```js
function* g() { log('[body] started'); yield 1; log('[body] resumed'); yield 2 }

log('before calling g()')
const it = g()
log('after calling g() — body has not run')
it.next()   // [body] started
it.next()   // [body] resumed
```

```text
before calling g()
after calling g() — body has not run
first next()
  [body] started
second next()
  [body] resumed
```

호출은 반복자 객체만 만들고, **`next()`를 부를 때마다 다음 `yield`까지만** 실행한다. 돌아오는 값은 `{ value, done }`이다. 소비하는 쪽이 당겨야 움직인다는 것, 앞 절의 속도 전달이 이 성질에서 나온다. 멈춘 상태가 어디 있는지는 두 번째 글의 `await`와 같은 이야기로, 엔진마다 구현은 다르지만 개념적으로는 반복자 객체가 들고 있고 그 객체를 가리키는 쪽이 있는 한 남는다.

### 안으로 넣고, 끝내고, 던지고

멈춘 제너레이터를 움직이는 방법은 `next()` 외에도 있다.

```js
function* g() {
  try { const x = yield 'a'; log('got', x); yield 'b' }
  finally { log('finally ran') }
}
const it = g()
it.next()           // 'a'에서 멈춘다
it.next('hi')       // 값을 안으로 넣는다: x === 'hi'
it.return(99)       // 'b'에서 return 99가 끼어든 것처럼 끝낸다 → finally 실행
```

```text
  got hi
next("hi") -> {"value":"b","done":false}
  finally ran
return(99) -> {"value":99,"done":true}
```

MDN 표현으로 `return(값)`은 멈춘 자리에 `return` 문이 끼어든 것처럼, `throw(오류)`는 `throw` 문이 끼어든 것처럼 동작한다. 앞 절에서 `finally`가 실행된 이유가 이것이다.

### async function*

`async function*`에는 `await`가 더해진다. 기다리는 일은 `await`, 값을 내보내는 일은 `yield`다.

| | 반환 | 값의 수 | 소비 |
| -- | ---- | ------- | ---- |
| `function` | 값 | 1 | 바로 사용 |
| `async function` | `Promise` | 1 (나중에) | `await` |
| `function*` | 이터레이터 | 여러 개 | `for...of`, `next()` |
| `async function*` | 비동기 이터레이터 | 여러 개 (나중에, 하나씩) | `for await...of` |

`async function*`의 `next()`는 `{ value, done }`을 담은 **Promise**를 돌려준다. 결과를 기다리지 않고 네 번 불러 보면 요청이 순서대로 처리돼 순서대로 풀린다.

```text
[
  {"value":"first","done":false},
  {"value":"second","done":false},
  {"value":"end","done":true},
  {"done":true}
]
```

ECMAScript 명세의 비동기 제너레이터 객체는 `[[AsyncGeneratorQueue]]`라는 슬롯을 갖는다. 명세는 이것을 "비동기 제너레이터를 재개하라는 요청 기록들의 목록"이라고 정의한다. 또 `async function*` 안의 `yield promise`는 Promise를 기다린 값을 내보낸다.

## 이 구조가 해 주지 않는 일

7번 질문이다. 장점을 봤으니 한계도 같이 본다. 직접 확인한 것이 셋이다.

### 1. await에서 기다리는 중이면 return()도 기다린다

`for await`의 `break`는 제너레이터가 `yield`에서 멈춘 상태에서 일어나서 `finally`로 곧바로 들어간다. 그런데 제너레이터가 **`await`에서 기다리는 중**일 때 밖에서 `return()`을 부르면 요청이 큐에 줄을 서서, 하던 `await`가 끝난 뒤에야 처리된다.

```js
async function* g() {
  try { await sleep(200); yield 'x' }      // 200ms 동안 기다리는 중
  finally { log('finally ran at', ms()) }
}
const it = g(); it.next()
await sleep(20)
it.return('stop')                          // 약 28ms에 호출
```

```text
return() called at ~ 28 ms
finally ran at 216 ms
```

`throw()`도 같았다. 28ms에 불렀는데 154ms에 받았다. 그래서 첫 글의 스트림 코드에는 `AbortController`가 따로 필요하다. `await reader.read()`에서 기다리는 동안에는 `return()`으로 끝내 달라고 해도 다음 조각이 올 때까지 기다린다. 즉시 끊으려면 `signal`로 `fetch`와 `read()`를 직접 중단해야 한다.

### 2. yield*로 감싼 재시도는 이미 낸 값을 다시 낸다

다른 제너레이터의 값을 이어서 내보낼 때는 `yield*`로 위임한다. 재시도로 감싸기 쉬운데 조건을 두지 않으면 함정이 있다.

```js
async function* withRetry() {
  for (let attempt = 0; attempt < 2; attempt++) {
    try { yield* open(attempt === 0); return }   // 첫 시도는 event-2 뒤에 끊긴다
    catch (e) { if (attempt === 1) throw e }
  }
}
```

```text
event-1, event-2, event-1, event-2, event-3
```

첫 시도에서 낸 `event-1`, `event-2`가 재시도 때 또 나왔다. 화면에서는 같은 글자가 두 번 이어 붙는 버그가 된다. 재시도는 아직 아무것도 내보내지 않은 연결 직후의 실패(예: 409)로 한정해야 한다.

### 3. 제너레이터는 이벤트 루프에 양보하지 않는다

세 번째 글에서 미뤄 둔 질문이다. 한 조각에 약 20ms가 걸리는 일을 10조각으로 나눈 동기 제너레이터를 그냥 `for...of`로 돌렸다.

```text
plain for...of: total 200ms, the 0ms timer fired at 201ms (after all chunks)
```

`yield`로 멈췄다 이어 가도 루프 전체가 **하나의 태스크** 안에서 돈다. 0ms 타이머는 10조각이 모두 끝난 201ms에야 실행됐다. 제너레이터는 "어디서 멈출지"를 코드로 표현할 뿐, 그 멈춤이 이벤트 루프에게 양보하는 것은 아니다. 조각 사이에 소비하는 쪽이 태스크를 거치게 해야 한다. 그렇게 했더니 타이머가 41ms에 실행됐다. 조각 하나가 1ms인 일 100개를 8ms 예산으로 돌렸을 때는 12번 양보했고 타이머는 8ms에 실행됐다. 이 실험은 Node.js의 `setImmediate`로 했고, 브라우저에서는 `setTimeout(0)`이나 `scheduler.yield()`로 바꾼다.

제너레이터가 이 일에서 해 주는 것은 쪼개는 **지점과 그 사이의 상태**를 코드 흐름 그대로 들고 있는 것까지다. 양보는 여전히 쓰는 쪽의 몫이다.

## 스트림은 이제 for await로 바로 돌 수 있다

그러면 `ReadableStream`을 제너레이터로 감싸지 않고 바로 `for await`로 돌 수 있나. 첫 글에서 나는 "브라우저 지원이 고르지 않다"고 썼다. 다시 확인했다. Playwright가 쓰는 세 엔진에서 직접 물어봤다.

| 엔진 | `for await (const c of response.body)` | `ReadableStream.from()` | `scheduler.yield()` |
| ---- | -------------------------------------- | ----------------------- | ------------------- |
| Chromium 158 | 된다 | 없다 | 있다 |
| Firefox 159 | 된다 | 있다 | 있다 |
| WebKit 27.2 | 된다 | 있다 | 없다 |

`for await`로 도는 것은 세 엔진 모두에서 됐다. 첫 글에서 지원이 고르지 않다고 쓴 것은 이 확인 이전의 내가 틀린 것이다. 반대로 `ReadableStream.from()`(반복자에서 스트림 만들기)은 [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/from_static)이 "Limited availability"로 표시하고, 직접 돌려 본 결과도 Chromium에서 없었다. `scheduler.yield()`도 MDN 표시와 같게 나왔다. 다만 Playwright의 WebKit은 실제 Safari가 아니라서, 실제 Safari에서는 확인하지 못했다.

취소는 세 엔진에서 직접 돌려 봤다. `cancel()` 훅을 단 스트림을 조각 하나 읽은 뒤 이렇게 끝냈다.

| 어떻게 끝냈나 | 스트림의 `cancel()` |
| ------------- | ------------------- |
| `for await`에서 `break` | 불렸다 |
| `reader.cancel()` | 불렸다 |
| `reader.releaseLock()`만 | 불리지 않았다 |

Chromium, Firefox, WebKit 모두 같았다. 첫 글에서 `releaseLock()`을 `cancel()`로 바꾼 이유가 그대로 확인된 셈이다. 그러면 첫 글에서 `getReader()`와 `while`을 쓴 이유는 지원이 아니다. 끝 표시 없이 `done`이 오는 경우를 따로 구분하려고 `done`을 코드에서 직접 보는 쪽이 분명해서였다. `for await`를 쓰면 루프가 끝난 뒤에 "끝 표시를 받았는가"를 따로 확인하는 식으로 같은 구분을 한다. 어느 쪽이 낫다는 정답은 없다.

## 다음에 제너레이터로 스트림을 다룰 때 먼저 볼 것

```text
1. 오는 단위와 원하는 단위가 다른가
   → 다르면 그 사이를 이어 주는 어댑터가 필요하다. 남은 조각은 지역 변수로 들고 있는다.

2. 기능별로 나눠 반복자로 이었는가
   → 쓰는 쪽은 for await 하나만 본다. 단계를 바꿔도 쓰는 쪽 코드는 그대로다.

3. 읽는 속도가 소비하는 속도에 묶여 있는가
   → 읽을 수 있는 대로 읽어 쌓고 있다면 느린 소비자 뒤에서 메모리가 쌓인다.

4. 정리를 가장 안쪽 finally에 한 번만 썼는가
   → break, 예외, return() 어느 경우에도 모든 단계의 finally가 돈다.

5. 중간에 끊어야 할 때 제너레이터가 어디서 멈춰 있는가
   → await에서 기다리는 중이면 return()도 기다린다. signal로 직접 중단한다.

6. 재시도나 양보를 제너레이터에 기대고 있는가
   → yield*는 이미 낸 값을 다시 낸다. 양보는 소비하는 쪽이 태스크를 거쳐야 한다.
```

## 마치며

`*` 하나가 낯설어서 시작한 질문이었지만, 동작을 외우는 것은 답이 되지 않았다. 답은 "이 도구가 어떤 문제를 푸는가"에 있었다. 청크로 오는 세상과 메시지로 읽고 싶은 세상 사이에는 늘 어댑터가 필요하고, 제너레이터는 그 어댑터를 **상태를 지역 변수로 두고, 소비하는 속도에 맞춰 당기고, 정리를 한 곳에 모으는** 모양으로 만들어 준다. 기능별로 나눠 이으면 쓰는 쪽은 반복자 하나만 본다.

세 편을 이으면 하나의 그림이 된다. 첫 글은 `await`가 헤더에서 풀리고 `body`가 스트림이라는 것, 두 번째 글은 멈춘 함수가 힙에 있고 바깥의 신호가 깨운다는 것, 세 번째 글은 마이크로태스크가 끝날 때까지 양보하지 않는다는 것이었다. 이 글은 그 위에서 "멈추고 이어 가는 지점을 코드가 정하고, 그 위에 읽는 코드를 반복자로 쌓는다"는 쓰임을 다뤘다.

그리고 이 구조가 해 주지 않는 일도 분명했다. 제너레이터가 양보를 대신 해 주지 않았고, `return()`이 `await`를 끊어 주지도 않았고, `yield*`는 재시도를 안전하게 만들어 주지 않았다. 편하게 읽히는 코드 아래에서 무엇이 언제 멈추고 누가 깨우는지는 여전히 코드를 쓰는 사람이 알아야 하는 일이다.

## 참고

- MDN, [function*](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function%2A), [async function*](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function%2A)
- MDN, [Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols), [for await...of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)
- MDN, [ReadableStream.from()](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/from_static), [Scheduler.yield()](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield)
- ECMA-262, [ECMAScript Language Specification](https://tc39.es/ecma262/) (`[[AsyncGeneratorQueue]]`)
