---
layout: post
title: "releaseLock 한 줄을 cancel로 바꿨더니 409가 사라졌다 — fetch의 body는 처음부터 스트림이었다"
date: 2026-11-13 11:30:00 +0900
excerpt: "스트리밍 응답을 읽다 그만둘 때 releaseLock()을 cancel()로 바꾼 한 줄의 이유를 따라가다 보니, await fetch가 언제 풀리는지, response.body가 무엇인지, 예전 XMLHttpRequest의 progress와 버퍼는 무엇이 달랐는지까지 왔다. fetch를 조사하고, XHR과 쓰기와 보안을 비교하고, 지금 Next.js 서버와 브라우저에서 fetch 스트림을 쓸 때 볼 것을 정리한다."
categories: [Engineering]
tags: [JavaScript, Fetch, Streams, XMLHttpRequest, Security, Next.js, Frontend]
---

채팅 답변을 스트리밍하는 코드를 다시 읽다가 한 줄에서 멈췄다.

```diff
-    reader.releaseLock()
+    reader.cancel().catch(() => {})
```

답변을 중간에 그만 읽어도 연결이 남아서, 바로 보낸 다음 메시지가 409로 거절됐다고 했다. 그래서 읽던 것을 내려놓기만 하는 대신 스트림을 취소하게 바꿨다고.

맞는 수정이었다. 그런데 "왜 `releaseLock()`으로는 연결이 안 닫히지?"를 설명하려니 질문이 이어졌다.

## 따라 나온 질문들

```text
1. await fetch(...)는 응답이 다 와야 풀리나?
2. response.body는 무엇이고, releaseLock()으로는 왜 연결이 남나?
3. 예전 XMLHttpRequest는 이걸 어떻게 했나? progress로 오는 건 무엇이고 body와는 뭐가 다른가?
4. 브라우저에서 쓰기는 얼마나 편해졌고, 보안은 무엇이 달라졌나?
5. 지금 Next.js 서버와 브라우저에서 fetch로 스트림을 다룰 때는 무엇을 봐야 하나?
```

fetch부터 들여다보고, 예전에 쓰던 XHR과 비교하고, 지금 쓰는 환경으로 돌아온다.

## fetch를 다시 보다

### await fetch는 헤더에서 풀린다

```ts
const response = await fetch(url, { method: 'POST', body })
//                ^ 응답 헤더가 도착하면 풀린다. 본문은 아직 오는 중일 수 있다
```

`await fetch(...)`는 **헤더(상태 코드 포함)가 도착하면** 풀린다. 그래서 `response.ok`, `response.status`, `response.headers`는 바로 읽을 수 있다. `response.body`는 객체만 만들어진 채 내용은 아직 오는 중일 수 있다.

HTTP 응답은 TCP 바이트 스트림 위에 순서대로 온다. 빈 줄까지 오면 헤더를 파싱할 수 있고, 그 뒤의 바이트가 본문이다. 작은 JSON은 헤더와 본문이 같은 패킷에 실려 와서 두 시점이 겹쳐 보일 뿐이다.

| 읽는 방법 | 풀리는 시점 |
| --------- | ----------- |
| `await fetch()` | 헤더 도착 |
| `await response.json()` / `.text()` / `.blob()` | 본문이 모두 도착 |
| `await reader.read()` | 본문 청크 하나가 도착 |

### body는 항상 ReadableStream이다

응답이 JSON이든 HTML이든 SSE든, `response.body`는 [`ReadableStream<Uint8Array>`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream)다. 본문이 없는 응답(204, `HEAD`)만 `null`이다. `response.json()`도 이 스트림을 끝까지 읽어 합쳐 주는 편의 메서드일 뿐이다.

```text
[소스]  ──enqueue──▶  [내부 큐]  ◀──read()──  [소비자]
(네트워크)             (아직 안 꺼낸 조각)       (reader)
```

알아 둘 것은 셋이다.

- **한 번에 한 reader만 읽는다.** `getReader()`를 부르면 그 reader가 이 스트림을 맡는다. 그 뒤에 다른 reader를 얻거나 `response.json()`을 부르면 이미 읽는 중이라며 거절된다. 한 번 꺼낸 조각은 다시 나오지 않으니, 같은 응답을 두 번 써야 하면 읽기 전에 `response.clone()`이나 `tee()`로 갈래를 만든다. 명세는 이 상태를 잠금(lock)이라 부르고, 그래서 맡은 것을 내려놓는 메서드 이름이 `releaseLock()`이다.
- **안 꺼내면 더 안 받는다.** 소비자가 조각을 꺼내 가지 않으면 큐가 차고, 브라우저는 서버에서 더 받지 않는다. 받는 쪽 속도에 맞춰 흐름이 늦춰지는 셈이다. 영어로는 backpressure라고 부른다.
- **도착하는 건 글자가 아니라 바이트다.** 조각은 `Uint8Array`, 숫자 배열이다. 글자로 쓰려면 이 숫자를 UTF-8로 읽어야(디코딩) 하는데, 한글 한 글자는 3바이트라 조각 경계에서 잘려 올 수 있다. '가'는 `EA B0 80`인데, 앞 조각이 `EA B0`로 끝나고 뒤 조각이 `80`으로 시작하면, 조각마다 따로 읽을 때 깨진 글자가 나온다. 둘을 이어 읽어야 '가'가 된다. 그래서 잘린 바이트를 다음 조각과 이어 읽어 주는 `TextDecoderStream`을 끼운다.

```ts
const reader = response.body
  .pipeThrough(new TextDecoderStream())   // 바이트 → 문자열 (잘린 한글도 이어 붙인다)
  .getReader()
```

`TextDecoderStream`은 입구와 출구를 가진 변환기(`TransformStream`)이고, `pipeThrough`는 앞 스트림을 그 입구에 꽂고 출구를 돌려준다.

### 청크는 메시지가 아니다

네트워크 청크의 경계와 메시지(SSE 이벤트, JSON 줄)의 경계는 일치하지 않는다.

```text
서버가 보낸 것:   data: {"a":1}\n\n  data: {"b":2}\n\n
받은 청크:        'data: {"a":1}\n\ndata: {"b'   |   '":2}\n\n'
```

한 청크에 메시지가 여러 개 들 수도, 하나가 둘로 쪼개질 수도 있다. 그래서 문자열을 버퍼에 쌓고 구분자로 완성된 메시지만 잘라 낸다.

### 끝은 done이 아니라 끝 표시로 안다

`reader.read()`가 `done: true`를 주면 스트림이 끝난 것이다. 다만 정상 종료와 중간에 끊긴 것을 구분하지 못한다. TCP는 둘 다 "닫힘"이다. 그래서 프로토콜에 끝 표시(예: `data: [DONE]`)를 두고, 그것 없이 `done`이 오면 비정상으로 본다.

처음엔 "끝 표시가 왔는데 왜 에러를 내지?"라고 잘못 읽었다. 끝 표시를 받으면 함수가 먼저 빠져나가서 `done` 검사에 닿지 않는다. `done`에 닿았다는 것 자체가 끝 표시를 못 받았다는 뜻이다.

### releaseLock은 내려놓기만 한다

처음의 한 줄로 돌아온다.

| | 하는 일 | 연결 |
| -- | ------- | ---- |
| `reader.releaseLock()` | 읽던 것을 내려놓기만 한다 | **남아 있다.** 서버는 계속 보낸다 |
| `reader.cancel()` | 스트림을 취소한다 | **닫힌다.** `pipeThrough`로 이어진 앞쪽, 결국 `response.body`까지 전달된다 |

`releaseLock()`은 "이 reader는 이제 안 읽는다"일 뿐이다. 스트림도 연결도 살아 있어서, 서버는 그 대화가 아직 답변 중이라고 본다. 그 상태에서 다음 메시지를 보내면 거절당한다. `cancel()`은 파이프를 거꾸로 타고 연결까지 닫는다. 이미 끊긴 연결에서는 reject될 수 있어서 `.catch(() => {})`를 붙인다.

사용자가 멈춘 경우는 `AbortController.abort()`로 끊고, 이때 나오는 `AbortError`는 실패로 보여 주지 않는다.

## 예전에는 XMLHttpRequest였다

`fetch` 전에는 [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)로 요청을 보냈다. 지금도 업로드 진행률 때문에 쓴다. 처음엔 "XHR도 `progress`로 데이터가 조금씩 오니까 결국 비슷한 것 아닌가" 했다. 들여다보니 달랐다.

### progress로 오는 건 데이터가 아니라 숫자다

```ts
xhr.onprogress = (event) => {
  event.loaded       // 지금까지 받은 바이트 수
  event.total        // 전체 바이트 수 (Content-Length가 있으면)
  xhr.responseText   // 데이터는 여기, 지금까지 받은 "전체" 텍스트
}
```

[`progress` 이벤트](https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent)가 알려 주는 것은 `loaded`, `total` 같은 **숫자**다. 데이터 조각을 건네주지 않는다. 데이터는 XHR이 안에 들고 있는 **버퍼**에 쌓이고, 그 버퍼를 `response`나 `responseText`로 들여다볼 뿐이다.

그 버퍼도 응답 형식에 따라 보이는 시점이 다르다.

| `responseType` | 받는 도중 | 다 받은 뒤 |
| -------------- | --------- | ---------- |
| `''`, `'text'` | `responseText`가 점점 길어진다 | 전체 텍스트 |
| `'json'`, `'arraybuffer'`, `'blob'` | `null` | 완성된 값 |

그래서 XHR로 스트리밍을 하려면 텍스트 응답에서 `responseText`의 늘어난 부분만 잘라 쓰는 우회밖에 없다.

### 같은 "조금씩"이지만 구조가 다르다

| | XHR의 응답 | fetch의 `response.body` |
| -- | ---------- | ----------------------- |
| 데이터가 있는 곳 | XHR 안의 버퍼 하나 | 스트림의 내부 큐 |
| 받는 방식 | 이벤트로 "늘었다"는 알림을 받고 버퍼를 다시 들여다본다 | 내가 `read()`로 조각을 꺼내 온다 |
| 읽은 부분 | 버퍼에 계속 남는다. 긴 응답일수록 메모리가 는다 | 꺼낸 조각은 버려진다 |
| 바이너리 | 다 받을 때까지 못 본다 | 조각 단위로 받는다 |
| 알림 시점 | 브라우저가 정한다. 명세상 대략 50ms마다 묶어서 알린다 | 조각이 큐에 들어오는 대로 꺼낼 수 있다 |
| 받는 속도 조절 | 없다. 받는 대로 버퍼에 쌓인다 | 있다. 안 꺼내면 더 받지 않는다 |
| 중간에 그만두기 | `xhr.abort()` | `reader.cancel()` 또는 `AbortController` |

한 줄로 하면 XHR은 **"쌓이는 버퍼를 지켜보는"** 구조이고, fetch는 **"흘러오는 조각을 꺼내 가는"** 구조다. 스트리밍 응답을 다루기에는 후자가 맞다.

### 같은 일을 두 방식으로 써 보면

XHR은 서버에서 쓸 수 없으니 완전히 같은 비교는 아니다. 그래도 브라우저 안에서만 놓고 보면 얼마나 편해졌는지가 보인다. JSON 하나를 받으면서 HTTP 오류, 네트워크 오류, 제한 시간, 사용자 취소를 모두 처리하는 경우다.

XHR은 이벤트마다 처리기를 달고, Promise로 직접 감싸야 `await`로 쓸 수 있다.

```ts
new Promise((resolve, reject) => {
  const xhr = new XMLHttpRequest()
  xhr.open('GET', url)
  xhr.responseType = 'json'
  xhr.timeout = 5000
  xhr.onload = () => (xhr.status >= 200 && xhr.status < 300 ? resolve(xhr.response) : reject(new Error(`HTTP ${xhr.status}`)))
  xhr.onerror = () => reject(new Error('Network error'))
  xhr.ontimeout = () => reject(new Error('Timeout'))
  xhr.onabort = () => reject(new DOMException('Aborted', 'AbortError'))
  signal.addEventListener('abort', () => xhr.abort(), { once: true })
  xhr.send()
})
```

fetch는 같은 일을 이렇게 쓴다.

```ts
const response = await fetch(url, { signal: AbortSignal.timeout(5000) })
if (!response.ok) throw new Error(`HTTP ${response.status}`)
const data = await response.json()
```

제한 시간과 사용자 취소를 함께 걸려면 `AbortSignal.any([signal, AbortSignal.timeout(5000)])`로 묶는다. 비교적 새 API라 지원 범위는 확인하고 쓴다.

편해진 지점을 꼽으면 이렇다.

| | XHR | fetch |
| -- | --- | ----- |
| `await`로 쓰기 | Promise로 직접 감싼다 | 바로 된다 |
| 오류의 갈래 | `onerror`, `ontimeout`, `onabort`, `status` 검사가 흩어진다 | 네트워크 오류와 취소는 reject, HTTP 오류는 `response.ok` 하나 |
| 취소와 제한 시간 | `abort()`와 `timeout`이 따로 | 둘 다 `AbortSignal` 하나로 |
| 요청을 이어서 보내기 | 콜백 안에서 다음 요청 (중첩) | `await`를 줄 세우면 된다 |
| 요청, 응답, 헤더 | `xhr` 하나가 다 들고 있다 | `Request`, `Response`, `Headers`를 따로 만들고 넘기고 재사용 |
| 스트리밍 | `responseText`에서 늘어난 부분을 잘라 쓴다 | `body`에서 조각을 꺼낸다. 변환은 `pipeThrough`로 잇는다 |
| 같은 코드를 서버에서 | 불가 | 그대로 된다 |

가장 크게 느껴지는 건 오류 처리가 한 갈래로 모인다는 점이다. 대신 fetch는 4xx, 5xx를 실패로 보지 않으니 `response.ok` 확인을 잊으면 안 된다.

### 진행률은 반대로 XHR이 낫다

| | XHR | fetch |
| -- | --- | ----- |
| 다운로드 진행률 | `onprogress` | 이벤트 없음. `body`를 읽으며 받은 바이트를 직접 센다 |
| **업로드 진행률** | **`xhr.upload.onprogress`** | **없음** |

fetch로 다운로드 진행률을 만들 때는 조심할 점이 있다. 응답이 압축돼 오면 `Content-Length`는 압축된 크기인데, `body`에서 읽는 바이트는 풀린 크기라 둘이 맞지 않을 수 있다.

업로드 진행률은 여전히 XHR이다. fetch도 요청 본문을 스트림으로 보내는 기능(`duplex: 'half'`)이 있지만 브라우저 지원이 제한적이라 확인하고 써야 한다. XHR을 Promise로 감싸고 `AbortSignal`을 받게 하면, 쓰는 쪽에서는 fetch와 같은 방식으로 다룰 수 있다.

### 보안에서 달라지는 것

먼저 오해 하나. fetch가 XHR보다 "더 열려 있거나" "더 막혀 있는" 것은 아니다. 둘 다 같은 브라우저 보안 모델 위에 있다.

**같은 것**

- **동일 출처 정책과 [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS).** 다른 출처의 응답을 읽으려면 서버가 허락해야 한다. fetch의 `mode: 'no-cors'`도 우회가 아니다. 요청은 보내지만 응답은 읽을 수 없는(opaque) 상태로 돌아온다.
- **금지된 헤더.** `Cookie`, `Host`, `Origin` 같은 헤더는 둘 다 직접 정할 수 없다.
- **읽을 수 있는 응답 헤더.** 다른 출처 응답에서는 서버가 `Access-Control-Expose-Headers`로 허락한 것만 보인다. `Set-Cookie`는 어느 쪽으로도 읽을 수 없다.
- **CSP.** 둘 다 `connect-src`의 적용을 받는다.

**달라진 것**

| | XHR | fetch |
| -- | --- | ----- |
| 쿠키 보내기 | 같은 출처는 보냄. 다른 출처는 `withCredentials = true` | `credentials`로 고른다. 기본은 `'same-origin'`, 다른 출처까지는 `'include'`, 아예 안 보내려면 `'omit'` |
| 리다이렉트 | 브라우저가 알아서 따라간다. 막을 수 없다 | `redirect`를 `'follow'`, `'error'`, `'manual'` 중에서 정한다. 예상하지 못한 리다이렉트를 오류로 만들 수 있다 |
| 리퍼러 | 요청마다 정할 수 없다 | `referrerPolicy`, `referrer`로 요청마다 줄일 수 있다 |
| 응답 무결성 | 없다 | `integrity`로 응답의 해시를 검증한다(서브리소스 무결성) |
| 취소된 요청 | `abort()` | `AbortSignal`. 서버로 넘겨 위쪽 요청까지 끊을 수 있다 |

fetch 쪽은 **기본값을 보수적으로 두고 요청마다 범위를 좁힐 수 있는** 옵션이 늘었다고 보면 된다. 쿠키를 아예 안 보내는 `'omit'`, 리다이렉트를 오류로 만드는 `'error'`가 그렇다.

**가장 큰 차이는 코드가 실행되는 곳이다**

비밀 키가 필요한 API를 브라우저 코드에서 바로 부르면, 그 키는 브라우저로 내려간다. 번들에 넣든 응답으로 받든, 사용자가 개발자 도구로 볼 수 있다. 예전에도 서버를 한 번 거쳐 키를 숨길 수는 있었다. 다만 그 서버는 PHP나 Java처럼 다른 언어로 된 다른 코드였다. fetch는 같은 JavaScript, 같은 API로 서버에서도 쓸 수 있어서, 키를 서버에 두고 결과만 흘려보내는 구조를 한 프로젝트 안에서 같은 방식으로 만든다(아래 Next.js 절). 스트림도 `body`를 그대로 넘기면 되므로 이 구조를 쓰는 데 비용이 거의 없다.

브라우저에 남는 인증 정보를 어디에 둘지는 fetch냐 XHR이냐와 별개의 문제다. `localStorage`에 둔 토큰은 XSS에 노출되고, `httpOnly` 쿠키는 스크립트가 읽지 못하는 대신 CSRF를 따로 막아야 한다.

### 그 밖의 차이

| | XHR | fetch |
| -- | --- | ----- |
| 페이지를 떠나도 보내기 | 어렵다 | `keepalive: true` |
| 동기 요청 | 가능 (메인 스레드에서는 사용 중단 권고) | 불가. 항상 비동기 |
| 쓸 수 있는 곳 | 브라우저(창, Web Worker). Service Worker에서는 불가. Node.js에 기본으로 없다 | 브라우저 전부, Node.js 18 이후, Deno, Bun, 엣지 런타임 |

### 조각을 끼워 넣던 시절부터

XHR이 처음 널리 쓰였을 때 가장 흔한 모양은 이랬다. 서버가 HTML 조각을 만들어 주고, XHR로 받아 `innerHTML`로 정해진 자리에 끼워 넣는다. 페이지를 새로 불러오지 않고 화면 일부만 바뀌는 게 그때는 혁신이었다. 그 뒤로 누가 화면을 그리는지와 보안의 기준이 몇 번 바뀌었다.

| 시기 | 누가 그리나 | 오가는 것 | 보안에서 신경 쓴 것 |
| ---- | ----------- | --------- | ------------------- |
| 페이지 단위 | 서버가 페이지 전체를 그린다 | 클릭마다 새 HTML 페이지 | 서버가 출력을 이스케이프하는가 |
| 조각 끼워 넣기 | 서버가 조각을 그리고, 브라우저가 자리에 넣는다 | XHR로 HTML 조각 | 받은 HTML을 `innerHTML`로 그대로 넣으면 XSS. 이스케이프를 빠뜨리기 쉬웠다 |
| 클라이언트 렌더링 | 브라우저가 JS로 전부 그린다. 첫 화면이 비고 JS가 커진다 | JSON 데이터 | 프레임워크가 기본으로 이스케이프한다. 대신 브라우저가 API를 직접 부르니 키와 권한이 브라우저 쪽에 드러나기 쉽다 |
| 서버와 클라이언트가 나눠 그리기 | 서버가 먼저 그려 조각으로 흘려보내고, 상호작용이 필요한 곳만 브라우저가 그린다 | 응답 하나의 스트림(HTML과 RSC 페이로드) | 데이터와 키는 기본으로 서버에 남는다. 새로 신경 쓸 곳은 서버에서 브라우저로 넘기는 값이다 |

정리되는 방향은 이렇다.

- **화면은 서버와 브라우저가 나눠 그리고, 그 경계를 개발자가 정한다.** 다 서버에서 그리던 시절과 다 브라우저에서 그리던 시절을 지나, 이제는 컴포넌트마다 어디서 그릴지 고른다.
- **조각을 끼워 넣는 일은 여전히 있지만, 손으로 하지 않는다.** 서버가 조각을 보내고 브라우저가 자리에 넣는다는 모양은 예전과 닮았다. 다만 조각은 응답 하나의 스트림으로 이어서 흘러오고, 끼워 넣는 일은 프레임워크가 한다. HTML 문자열을 직접 꽂지 않으니 `innerHTML` 시절의 XSS 위험도 그만큼 줄었다.
- **보안의 기본값이 안전한 쪽으로 옮겨 왔다.** 출력은 기본으로 이스케이프되고, 비밀 키와 데이터 접근은 기본으로 서버에 남는다. 대신 경계를 넘는 것이 새 확인 대상이다. 서버 컴포넌트가 클라이언트 컴포넌트에 넘기는 props는 RSC 페이로드에 담겨 브라우저로 가므로, 넘기기 전에 필요한 필드만 남겨야 한다. Server Action은 화면을 거치지 않고 POST 요청으로 바로 부를 수 있어서, Next.js 문서는 액션마다 안에서 인증과 권한을 확인하라고 한다.

마지막 단계가 가능해진 데에는 XHR과 fetch의 차이 하나가 깔려 있다. XHR은 브라우저 안의 API다. fetch는 서버에도 있다.

## 지금 fetch를 쓰는 두 곳

같은 fetch와 `ReadableStream`이 서버와 브라우저 양쪽에 있다. Next.js라면 한 프로젝트 안에서 둘 다 쓴다.

### 서버: 대부분은 스트림을 만들지 않는다

Server Component에서 쓰는 fetch는 대개 스트림을 직접 다루지 않는다. 데이터를 기다리는 부분을 `<Suspense>`로 감싸 두면, 준비된 조각부터 나눠 보내는 일은 Next.js가 한다.

**Server Component에서 읽을 때.** 가장 흔한 경우다.

```tsx
async function Messages({ id }: { id: string }) {
  const response = await fetch(`${API}/chats/${id}/messages`)
  const { messages } = await response.json()
  return <MessageList items={messages} />
}

// 부모에서
<Suspense fallback={<Skeleton />}>
  <Messages id={id} />
</Suspense>
```

여기에 `ReadableStream`은 보이지 않는다. `Messages`가 데이터를 기다리는 동안 바깥 부분과 `Skeleton`이 먼저 나가고, 준비되면 그 부분의 HTML과 RSC 페이로드가 이어서 간다. 반대로 `<Suspense>`로 감싸지 않으면, 캐시되지 않은 fetch는 끝날 때까지 페이지 렌더링을 막는다. fetch만 쓰면 알아서 흘러가는 게 아니라, 경계를 어디에 두느냐가 개발자의 몫이다.

그런데 그 아래에서 일어나는 일은 앞에서 본 것과 같은 모양이다. Next.js 문서는 스트리밍을 chunked transfer encoding으로 응답을 나눠 보내는 것이라고 설명한다. 응답 하나가 끝나지 않은 채 조각으로 이어서 오고, 브라우저는 도착한 조각부터 쓴다. 페이지를 옮길 때는 HTML 대신 RSC 페이로드로 화면을 갱신하고, Next.js는 그 페이로드를 미리 받아 캐시해 둔다. 채팅 답변에서 손으로 하던 일을 Next.js와 React가 페이지 단위로 대신 해 주는 셈이다. 스트림이 없어진 게 아니라, 보이지 않는 곳으로 내려갔다.

이 fetch는 Next.js가 확장한 fetch라 캐시 옵션(`cache`, `next.revalidate`)과 같은 요청의 중복 제거가 얹혀 있고, 캐시의 기본값이 버전마다 달랐다. 의도치 않게 캐시되지 않는지 쓰는 버전의 문서로 확인한다.

스트림을 직접 신경 써야 하는 건 Route Handler로 **HTTP 스트림 API를 따로 둘 때**다. 클라이언트 컴포넌트가 채팅 답변을 SSE로 받는 식이다. 이때도 두 경우로 나뉜다.

**다른 API의 스트림을 넘길 때.** 이쪽이 흔하다. 서버가 위쪽 API를 fetch해서 그 body를 그대로 넘긴다.

```ts
const upstream = await fetch(UPSTREAM_URL, { method: 'POST', body, signal: req.signal })
return new Response(upstream.body, { status: upstream.status, headers: { 'Content-Type': 'text/event-stream' } })
```

`upstream.body`가 이미 `ReadableStream`이라, 서버에서 다 받아 모을 필요 없이 파이프만 이어 주면 된다. 키를 서버에 숨기면서 스트림은 그대로 흘려보낼 수 있다. 클라이언트가 끊었을 때 위쪽 요청도 끊기도록 `req.signal`을 넘기는 것이 중요하다. 그렇지 않으면 아무도 안 읽는 답변을 서버가 끝까지 받는다. 브라우저에서 `releaseLock()`만 했을 때와 같은 일이 서버에서 일어난다.

**서버가 직접 만들어 보낼 때.** 이때만 `ReadableStream`을 만든다. 드물고, 라이브러리가 대신 만들어 주는 경우도 많다.

```ts
const stream = new ReadableStream({
  async start(controller) {
    // 만들 때마다 controller.enqueue(바이트)
    // 끝나면 끝 표시를 보내고 controller.close(), 실패하면 controller.error(e)
  },
  cancel() {
    // 클라이언트가 끊으면 여기서 정리할 수 있다
  },
})
return new Response(stream, { headers: { 'Content-Type': 'text/event-stream' } })
```

핸들러가 `Response`를 반환하는 순간 헤더가 먼저 나가고, 본문은 `enqueue`할 때마다 이어서 나간다. HTTP/1.1이면 chunked 인코딩, HTTP/2면 DATA 프레임으로 간다.

Route Handler로 스트림을 내보낼 때, 로컬에서는 잘 흘러오는데 배포하면 한꺼번에 도착할 때가 있다. 그때 볼 곳이다.

| 원인 | 확인 |
| ---- | ---- |
| 응답이 캐시나 정적 최적화에 걸림 | Route Handler가 요청마다 실행되는지 |
| 앞단 프록시가 버퍼링 | `X-Accel-Buffering: no`, 프록시 설정 |
| 압축 미들웨어가 청크를 모음 | `Cache-Control: no-transform`, 압축 설정 |
| 플랫폼 함수가 스트리밍을 지원하지 않음 | 플랫폼 문서의 스트리밍 지원과 제한 시간 |

### 브라우저: 꺼내 가며 그린다

브라우저 쪽은 앞에서 본 그대로다. 읽는 부분(디코딩, 버퍼, 파싱, 끝 표시, 정리)을 한 함수 안에 가두고, 밖에는 이벤트 단위로 내보낸다.

```ts
for await (const event of streamEvents('/api/chat', { input }, { signal })) {
  applyEvent(event)   // 이벤트가 도착하는 대로 상태 갱신
}
```

`streamEvents` 안에서는 `getReader()`로 읽고, 버퍼를 구분자로 자르고, 끝 표시에서 빠져나오고, `finally`에서 `cancel()`한다. 쓰는 쪽은 스트림을 모르고 이벤트만 받는다. `ReadableStream`은 바로 `for await`로 돌 수도 있다. 나중에 Playwright의 Chromium, Firefox, WebKit에서 직접 확인해 보니 세 엔진 모두 됐다(이 글을 처음 올릴 때는 지원이 고르지 않다고 썼는데, 확인하지 않고 쓴 말이었다). 그래도 이렇게 감싼 것은 끝 표시 없는 종료를 구분하려고 `done`을 코드에서 직접 보는 쪽이 분명해서다. 이런 래퍼를 왜 `async function*`로 쓰는지는 [이 글](https://changbaebang.github.io/2026-11-13-why-async-generator/)에서 다룬다.

브라우저에서 특히 볼 것은 두 가지다.

- **루프가 누구의 수명을 따르는가.** 이 루프를 컴포넌트 안에서 돌리면 다른 화면으로 옮기는 순간 함께 끝난다. 답변이 화면을 떠나도 이어져야 한다면 컴포넌트 밖의 스토어에서 돌리고, 컴포넌트는 그 대화의 상태만 구독한다.
- **연결 수.** HTTP/1.1에서는 같은 origin에 동시 연결이 약 6개다. 오래 열린 스트림이 여럿이면 다른 요청이 줄을 선다. HTTP/2는 연결 하나에 여러 요청을 실어 이 한도가 사실상 사라진다.

### 조각을 나르는 길도 바뀌었다

서버와 브라우저 사이에서 그 조각을 실어 나르는 방식도 HTTP 버전마다 다르다.

| | 조각을 나르는 방식 | 연결 | 스트림이 길면 |
| -- | ------------------ | ---- | ------------- |
| HTTP/1.1 | `Transfer-Encoding: chunked`. 길이를 붙인 조각을 이어 보내다가 길이 0인 조각으로 끝낸다 | 연결 하나에 요청 하나씩. 같은 origin에 동시 연결이 약 6개 | 그 연결을 계속 붙잡는다. 다른 요청이 줄을 선다 |
| HTTP/2 | DATA 프레임. chunked 인코딩은 쓸 수 없고, 끝은 프레임의 `END_STREAM` 표시로 알린다 | TCP 연결 하나에 여러 요청을 섞어 싣는다 | 다른 요청이 같은 연결로 함께 간다. 다만 패킷 하나를 잃으면 그 연결의 모든 요청이 함께 기다린다 |
| HTTP/3 | QUIC 스트림 위의 프레임. `Transfer-Encoding`은 쓰지 않는다 | UDP 기반의 QUIC. 요청마다 독립된 스트림 | 패킷을 잃어도 그 요청만 기다린다 |

코드에서 보면 셋 다 같다. 서버는 `ReadableStream`을 `Response`에 넘기고, 브라우저는 `response.body`에서 조각을 꺼낸다. 어느 버전으로 실릴지는 브라우저와 서버가 정한다. 그리고 `read()`로 꺼내는 조각이 HTTP의 chunk나 프레임과 하나씩 맞는다는 보장도 없다. 앞에서 "청크는 메시지가 아니다"라고 한 것이 여기서도 그대로다.

### 어디서 읽을지 고르기

같은 응답도 서버에서 읽느냐 브라우저에서 읽느냐에 따라 개발자가 고민할 것이 달라진다.

| 읽는 곳 | 맞는 경우 | 개발자가 고민할 것 |
| ------- | --------- | ------------------ |
| **Server Component** | 페이지를 열 때 필요한 데이터 | `<Suspense>` 경계를 어디에 둘지. 나눠 보내는 단위가 경계라서, 채팅처럼 글자 단위로 흘려보내는 데는 잘 맞지 않는다. 캐시 기본값 |
| **브라우저** (클라이언트 컴포넌트가 `body`를 읽음) | 사용자가 보낸 뒤 조각 단위로 보여야 할 때. 채팅 답변처럼 | 디코딩, 메시지 자르기, 끝 표시, 중단 버튼(`AbortController`), 루프의 수명(컴포넌트 밖 스토어), 연결 수 |
| **서버에서 넘기기만** (Route Handler가 `upstream.body`를 그대로) | 비밀 키가 필요한 위쪽 API를 브라우저에 스트림으로 보여 줄 때 | 클라이언트가 끊으면 위쪽도 끊기도록 `req.signal` 전달. 배포 환경의 버퍼링 |
| **서버도 읽고 넘기기** | 흘려보내면서 서버도 내용을 봐야 할 때. 다 받은 답변을 저장하거나 검사하는 식 | 스트림은 한 번만 읽히므로 `tee()`로 갈래를 내거나, `TransformStream`으로 지나가는 조각을 본다 |

정하는 순서는 대략 이렇다.

```text
1. 언제 필요한가        → 페이지를 열 때면 Server Component, 사용자가 무엇을 한 뒤면 브라우저
2. 얼마나 잘게 보여야 하나 → 컴포넌트 단위면 <Suspense>, 글자 단위면 브라우저가 body를 읽는다
3. 비밀 키가 필요한가    → 그렇다면 위쪽 호출은 서버에서. 브라우저에는 결과만 흘려보낸다
4. 서버도 내용이 필요한가 → 그렇다면 서버가 읽는다. 넘기기만 하는 파이프로는 부족하다
5. 끊김은 누가 처리하나  → 브라우저면 AbortController, 서버를 거치면 req.signal까지 이어 준다
```

`tee()`에는 주의할 점이 있다. 두 갈래 중 빠른 쪽이 읽는 속도로 원본을 읽고, 느린 쪽이 아직 안 읽은 조각은 쌓아 둔다. 한쪽이 멈추면 그만큼 메모리에 남는다. 저장처럼 느린 갈래가 있다면 지나가는 조각을 모아 두는 `TransformStream`이 더 단순할 때가 많다.

## 다음에 스트림을 다룰 때 먼저 볼 것

```text
1. 끝 표시가 있는가
   → done은 정상 종료와 끊김을 구분하지 못한다. 서버는 close() 전에 끝 표시를 보낸다.

2. 중간에 그만둘 때 연결이 닫히는가
   → 브라우저는 releaseLock()이 아니라 cancel(). 서버는 위쪽 fetch에 req.signal을 넘긴다.

3. 청크를 메시지로 착각하고 있지 않은가
   → 글자는 TextDecoderStream으로 잇고, 메시지는 버퍼에 모아 구분자로 자른다.

4. 진행률이 필요한가
   → 업로드라면 XHR. 다운로드는 body를 읽으며 세되, 압축이면 Content-Length와 다르다.

5. 배포 환경이 스트림을 모으고 있지 않은가
   → 캐시, 프록시 버퍼링, 압축을 확인한다.

6. 비밀 키가 브라우저로 내려가고 있지 않은가
   → 키가 필요한 호출은 서버에서 fetch하고, 브라우저에는 결과만 흘려보낸다.
     요청마다 credentials, redirect, referrerPolicy를 필요한 만큼만 연다.
```

## 마치며

시작은 `releaseLock()`을 `cancel()`로 바꾼 한 줄이었다. "연결을 닫는 쪽으로 바꿨다"로 끝낼 수 있었다. 왜 그게 맞는지 설명하려다 보니, 그 한 줄 아래에 웹이 지나온 길이 깔려 있었다.

XHR은 쌓이는 버퍼를 지켜보게 했고, fetch는 흘러오는 조각을 꺼내 가게 했다. 서버가 보내는 것은 HTML 조각에서 데이터로 바뀌었다가, 다시 렌더링된 조각의 스트림으로 돌아왔다. 그 조각을 나르는 길도 HTTP/1.1의 chunked 응답에서 HTTP/2와 HTTP/3의 프레임으로 바뀌었지만, 코드에서 보이는 `response.body`는 그대로다. 보안은 "받은 HTML을 어떻게 안전하게 끼워 넣나"에서 "무엇을 서버에 남기고 무엇을 경계 너머로 넘기나"로 옮겨 왔다. 그리고 JavaScript는 브라우저에서만 돌던 언어에서 서버와 브라우저 양쪽에서 같은 fetch와 같은 스트림을 쓰는 언어가 됐다.

그래서 지금 개발자가 고르는 것은 API 하나가 아니라 위치다. 이 데이터를 어디서 읽고, 어디서 그리고, 어디서 끊을지. 도구가 많은 걸 대신해 줄수록 그 선택이 드러나는 곳은 오히려 이런 한 줄이다. 연결을 그냥 두느냐 닫느냐 같은.

`releaseLock()`과 `cancel()`의 차이는 메서드 두 개의 차이였다. 그 차이를 설명할 수 있게 되니, 지금 쓰는 프레임워크가 무엇을 대신 해 주고 무엇을 나에게 남겨 두는지가 같이 보였다.

## 참고

- MDN, [Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API), [Using readable streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams), [ReadableStream.tee()](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/tee)
- WHATWG, [Streams Standard](https://streams.spec.whatwg.org/)
- MDN, [Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch), [Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- MDN, [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest), [ProgressEvent](https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent)
- WHATWG, [XMLHttpRequest Standard](https://xhr.spec.whatwg.org/), [Fetch Standard](https://fetch.spec.whatwg.org/)
- IETF, [RFC 9112 HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html) 7.1절 Chunked Transfer Coding, [RFC 9113 HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html), [RFC 9000 QUIC](https://www.rfc-editor.org/rfc/rfc9000.html)
- MDN, [Connection management in HTTP/1.x](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Connection_management_in_HTTP_1.x)
- MDN, [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS), [RequestInit](https://developer.mozilla.org/en-US/docs/Web/API/RequestInit), [Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Subresource_Integrity)
- Next.js, [Fetching Data](https://nextjs.org/docs/app/getting-started/fetching-data), [Streaming](https://nextjs.org/docs/app/guides/streaming), [route.js](https://nextjs.org/docs/app/api-reference/file-conventions/route), [loading.js](https://nextjs.org/docs/app/api-reference/file-conventions/loading), [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components), [Data Security](https://nextjs.org/docs/app/guides/data-security). 버전에 따라 기본 동작이 달라 쓰는 버전의 문서를 본다.
