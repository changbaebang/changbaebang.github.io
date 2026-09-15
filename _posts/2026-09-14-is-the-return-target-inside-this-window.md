---
layout: post
title: "복귀 대상이 이 창 안에 있는가 — 세 지면, 두 채널, 하나의 '돌아가기' 버튼"
date: 2026-09-15 11:57:00 +0900
excerpt: "'이전 페이지로 돌아가기' 한 버튼이 새 탭에서는 닫고, 같은 탭에서는 이동하고, 웹뷰에서는 절대 닫지 않아야 했는데, 웹뷰인지로 가르면 주문서를 잃었고 새 탭인지는 런타임에서 알 수 없었다. 기준을 '복귀 대상이 이 창 안에 있는가'로 바꾸고, 런타임이 모르는 사실은 여는 쪽이 선언하게 했다."
categories: [Engineering, Frontend]
tags: [Navigation, WebView, window.open, Open Redirect, URL, Hybrid App, Design]
---

> 내부 private repo 작업을 바탕으로 썼다.
> 외부 접근이 불가한 저장소라 링크 없이, 조직·시스템·사람을 식별할 수 있는 정보는 걷어내고 구조만 남긴다.

[넛지에서 랜딩까지](https://changbaebang.github.io/2026-04-02-event-with-dom-attributes/) 에서 넛지 → 랜딩 → 외부 신청 URL 로 이어지는 흐름에서 "어느 지면의 어떤 넛지였는지" 를 어떻게 남기는지 적었다. 그 글은 앞으로 가는 방향의 이야기였다. 이 글은 반대 방향, 돌아오는 이야기다.

제휴 카드 발급이 2분 배치에서 실시간으로 바뀌면서 요구사항이 하나 생겼다. 카드사에서 발급을 마치고 돌아온 사용자가 브릿지 페이지의 「이전 페이지로 돌아가기」 를 누르면, **처음 들어왔던 지면으로 돌아가서 새로고침되어야 한다.** 주문서에서 왔으면 주문서로 돌아가 방금 발급된 카드가 결제 수단에 보여야 하고, 상품 상세에서 왔으면 상품 상세로 돌아가 카드 할인가가 갱신되어야 한다.

버튼 하나에 "돌아가기" 라는 이름이 붙어 있지만, 이 버튼이 해야 할 일은 어디서 어떻게 들어왔느냐에 따라 네 가지다.

## TL;DR

- 「돌아가기」 의 동작을 가르는 기준은 웹뷰인지가 아니라 **복귀 대상이 이 창 안에 있는가** 다. 같은 창 안이면 이동, 창 밖이면 닫기.
- 스크립트가 연 새 탭은 `opener` 를 끊어두면 런타임에서 "내가 새 탭인지" 알 방법이 없다. 그래서 **여는 쪽이 URL 에 선언**한다. 받는 쪽은 그 선언을 믿되, 웹뷰에서는 믿지 않는다.
- 웹뷰는 절대 닫지 않는다. 웹뷰를 닫으면 복귀 대상까지 함께 사라진다.
- `window.close()` 는 거부될 수 있다. 허용되면 `window.closed` 가 먼저 `true` 가 되고 실제 닫기는 뒤에 실행되므로, 다음 틱에도 `closed` 가 `false` 면 거부된 것으로 보고 이동으로 폴백한다.
- 복귀 주소는 외부에서 오는 값이므로 검증한다. 허용하지 않는 역슬래시·제어 문자를 먼저 거르고, URL 로 파싱한 뒤 오리진을 비교한다. 운영은 홈과 같은 오리진만 통과한다.

## 네 가지 경우

진입 지면은 셋(주문서·상품 상세·마이페이지), 채널은 둘(웹·앱 웹뷰)이다. 넛지가 랜딩을 여는 방식이 지면과 채널마다 다르고, 그에 따라 브릿지가 돌아갈 방법도 달라진다.

| 진입 | 랜딩을 여는 방식 | 브릿지에서 「돌아가기」 | 이유 |
|---|---|---|---|
| 웹 · 주문서 「발급받기」 | `location.href` (같은 탭) | 복귀 주소로 **이동** | 주문서가 이 탭에 있었고 지금은 없다. 닫으면 주문서를 잃는다 |
| 웹 · 상품 상세 넛지 | `window.open(url, '_blank')` (새 탭) | 이 탭을 **닫기** | 상품 상세는 원래 탭에 그대로 있다. 닫으면 돌아간다 |
| 웹뷰 · 주문서·상품 상세 | `location.href` (같은 웹뷰) | 복귀 주소로 **이동** | 웹뷰 안에서 페이지만 바뀌었다. 웹뷰를 닫으면 전부 사라진다 |
| 웹뷰 · 마이페이지 메뉴 | 앱이 **새 웹뷰**를 띄움 | **닫기 페이지**로 이동 | 마이페이지는 네이티브 화면이라 이 웹뷰 밖에 있다. 웹에서 웹뷰를 직접 닫을 수 없으니 앱이 닫아주는 페이지로 간다 |

표를 세로로 읽으면 규칙이 하나로 모인다. 복귀 대상이 **이 브라우징 컨텍스트 안에** 있으면 이동하고, **밖에** 있으면 (직접이든 앱을 통해서든) 이 컨텍스트를 닫는다.

## 처음 코드는 `isWebview()` 로 갈랐다

첫 구현은 이랬다.

```ts
if (isWebview()) {
  redirect({ path: returnToUrl });   // 웹뷰: 이동
} else {
  window.close();                    // 웹: 닫기
}
```

"웹은 새 탭으로 열리니까 닫고, 웹뷰는 같은 웹뷰니까 이동한다" 는 전제다. 상품 상세 넛지만 보면 맞다. 상품 상세는 웹에서 `window.open` 으로 랜딩을 연다.

그런데 주문서의 「발급받기」 버튼은 웹에서도 `location.href` 로 같은 탭을 보낸다. 결제 수단 옆에 붙은 작은 링크라 새 탭을 열지 않는 게 자연스러웠고, 그건 그 지면의 맞는 판단이다. 이 경로에서 위 코드가 실행되면 `close()` 가 주문서가 있던 탭을 닫는다. 사용자는 카드를 발급받고 돌아왔더니 주문서가 사라진 것을 본다.

`isWebview()` 는 채널을 알려줄 뿐 "이 탭이 어떻게 열렸는지" 는 알려주지 않는다. 기준을 잘못 골랐다.

## "새 탭인지" 는 런타임에서 알 수 없다

그러면 브릿지가 스스로 "나는 새 탭인가" 를 판단하면 되지 않을까. `window.opener` 가 있으면 새 탭이고 없으면 같은 탭이다.

여기서 걸린다. 넛지가 새 탭을 열 때는 이렇게 연다.

```ts
const w = window.open(plccLandingUrl, '_blank');
if (w) {
  w.opener = null;
}
```

`opener` 를 끊는 건 이유가 있다. 열린 탭이 `window.opener.location` 으로 원래 탭을 다른 곳으로 보낼 수 있는 [역 탭내빙(reverse tabnabbing)](https://owasp.org/www-community/attacks/Reverse_Tabnabbing) 을 막기 위해서다. 랜딩은 우리 도메인이지만 그 뒤에 카드사 도메인이 이어지므로 끊어두는 게 맞다. 최신 브라우저는 `_blank` 에 `noopener` 를 기본으로 적용하기도 한다.

그 결과 브릿지 입장에서는 `window.opener === null` 이 "새 탭" 과 "같은 탭" 양쪽에서 똑같이 참이다. `history.length` 도 도움이 안 된다. 새 탭이라도 랜딩 → 브릿지 → 카드사 → 브릿지를 거치며 히스토리가 쌓인다.

런타임에서 판별할 수 없는 사실은 **알고 있는 쪽이 말해주는** 수밖에 없다. 새 탭을 여는 코드는 자기가 새 탭을 연다는 것을 안다. 그러니 URL 에 적는다.

```ts
const markOpenedInNewTab = (href: string) => {
  const url = new URL(href);
  url.searchParams.set(OPENED_IN_NEW_TAB_QUERY_KEY, '1');
  return url.toString();
};

if (isWebview()) {
  window.location.href = landingUrl;                       // 선언 없음
} else {
  const w = window.open(markOpenedInNewTab(landingUrl), '_blank');
  if (w) w.opener = null;
}
```

랜딩은 이 값을 복귀 주소와 함께 브릿지까지 그대로 전달한다. 주문서의 「발급받기」 는 같은 탭으로 보내니 이 값을 붙이지 않고, 브릿지는 값이 없으면 이동한다.

## 받는 쪽: 선언을 믿되, 웹뷰에서는 믿지 않는다

브릿지의 버튼 핸들러는 이렇게 됐다.

```ts
const createHandleBack = (returnToUrl?: string, openedInNewTab?: boolean) => () => {
  const isInWebview = isWebview();

  if (!openedInNewTab || isInWebview) {
    if (!returnToUrl) {
      window.location.href = isInWebview ? 'app://home' : homePath;
      return;
    }
    redirect({ path: returnToUrl, isWebview: isInWebview });
    return;
  }

  window.close();
  setTimeout(() => {
    if (!window.closed) {
      redirect({ path: returnToUrl ?? homePath, isWebview: false });
    }
  }, 0);
};
```

세 가지 결정이 들어 있다.

**`openedInNewTab` 은 웹뷰에서 무시한다.** 이 값은 웹의 `window.open` 경로만 싣는다. 웹뷰에서 이 값이 보인다면 누군가 링크를 만들었거나 옛 링크가 공유된 것이다. 웹뷰에서 `close()` 를 호출하면 (된다 해도) 복귀 대상까지 닫힌다. 그러니 웹뷰는 값이 무엇이든 이동한다. 선언을 믿는 것과, 그 선언이 나올 수 없는 맥락에서 무시하는 것은 양립한다.

**`close()` 뒤에 폴백을 둔다.** 스크립트가 열지 않은 탭(정확히는 [script-closable](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-window-close) 이 아닌 탭)에서 `window.close()` 는 조용히 무시된다. 그러면 화면이 그대로 멈춘 것처럼 보인다. 표준에 따르면 닫기가 허용될 때 먼저 `is closing` 상태가 설정되고 실제 닫기 작업은 큐에 들어가며, `window.closed` 는 이 상태를 보고 바로 `true` 를 돌려준다. 그래서 `close()` 직후에 무조건 `redirect` 를 부르면 닫히는 탭에서 불필요한 페이지 요청이 나가고, 반대로 `closed` 를 보지 않으면 거부된 경우를 알 수 없다. 다음 틱까지 `window.closed === false` 라면 닫기 요청이 받아들여지지 않은 것으로 보고, 그때만 이동한다.

**복귀 주소가 없으면 홈이다.** 넛지 없이 브릿지 URL 로 직접 들어온 경우다. 이동할 곳이 없으니 닫지도 않는다.

## 네 번째 경우: 창 밖에 있는데 내가 닫을 수 없을 때

마이페이지의 「제휴 카드」 메뉴는 네이티브 화면에서 `app://web/{url}` 딥링크로 **새 웹뷰**를 띄운다. 그 웹뷰가 랜딩 → 브릿지로 이어진다. 복귀 대상인 마이페이지는 이 웹뷰 밖, 네이티브에 있다.

규칙대로면 "닫기" 다. 그런데 웹에서 `window.close()` 로 앱의 웹뷰를 닫을 수는 없다. 대신 앱과 약속된 "닫기 페이지" 가 있다. 그 페이지로 이동하면 앱이 웹뷰를 닫는다. 그래서 이 진입점은 복귀 주소 자체를 닫기 페이지로 넣는다.

```ts
const returnToForLanding = isWebview() ? toAbsoluteUrl(WEBVIEW_CLOSE_PATH) : returnAbsoluteUrl;
landingUrl.searchParams.set(RETURN_TO_QUERY_KEY, returnToForLanding);
```

브릿지 코드는 바뀌지 않는다. 브릿지는 여전히 "웹뷰면 복귀 주소로 이동" 을 한다. 이동한 곳이 닫기 페이지일 뿐이다. "창 밖이면 닫는다" 는 규칙을 브릿지가 아니라 진입점이 복귀 주소를 고르는 것으로 구현했다.

같은 마이페이지라도 상단 배너는 같은 웹뷰 안에서 랜딩으로 가므로 복귀 주소가 마이페이지 웹 URL 이고 브릿지는 거기로 이동한다. QA 에서 "배너와 메뉴가 다르게 동작한다" 는 질문이 왔는데, 둘은 복귀 대상의 위치가 다르니 동작이 다른 게 맞다. 규칙이 하나면 이런 질문에 답이 바로 나온다.

## 복귀 주소는 외부 입력이다

`returnTo` 는 URL 쿼리로 세 페이지를 건너온다. 누구나 만들 수 있는 값이고, 브릿지는 로그인 뒤에 뜨는 페이지다. 검증 없이 `location.replace(returnTo)` 를 하면 브릿지가 오픈 리다이렉트의 한 홉이 된다.

정책은 좁게 잡았다. 복귀 대상은 언제나 우리 지면이므로, **운영에서는 홈과 같은 오리진** 하나만 허용한다. 서브도메인을 열면 우리가 운영하는 모든 서브도메인이 정당한 복귀 대상이 되고, 그중 하나가 자체 오픈 리다이렉트를 가지면 브릿지가 그걸 세탁해준다. 호스트만 비교하지 않고 오리진(스킴·호스트·포트)을 비교하는 이유는 `https://www.example.com:4443/…` 처럼 포트만 다른 값이 호스트 비교를 통과하기 때문이다. 비운영은 로컬 앱마다 포트가 달라 서브도메인과 임의 포트를 연다.

```ts
export const isAllowedReturnOrigin = (url: URL) => {
  if (url.origin.toLowerCase() === RETURN_ROOT_ORIGIN) return true;
  return appEnv !== 'production' && url.hostname.toLowerCase().endsWith(NON_PRODUCTION_HOST_SUFFIX);
};
```

상대 경로도 받는데, 여기서 리뷰어가 별건을 짚었다. `startsWith('/')` 이고 `startsWith('//')` 가 아니면 내부 경로로 봤는데, WHATWG URL 파서는 http(s) 같은 special scheme 에서 `\` 를 `/` 와 동일하게 다루고 탭·개행·CR 은 파싱 전에 제거한다.

```ts
new URL('/\\evil.com', 'https://www.example.com');    // → https://evil.com/
new URL('/\t/evil.com', 'https://www.example.com');   // → https://evil.com/
```

`/\evil.com` 은 `/` 로 시작하고 `//` 로 시작하지 않으니 검사를 통과하고, 브라우저는 그걸 `//evil.com` 으로 읽는다. 정상 경로에 이 문자들이 들어갈 일은 없으므로 상대 경로는 이 문자들을 먼저 거르고, 절대 URL 은 파싱한 뒤 오리진을 비교한다.

```ts
const PATH_AUTHORITY_ESCAPE_PATTERN = /[\\\t\n\r]/;

export const isInternalPath = (value?: string | null): value is string => {
  if (!value?.startsWith('/')) return false;
  if (value.startsWith('//')) return false;
  return !PATH_AUTHORITY_ESCAPE_PATTERN.test(value);
};
```

같은 복귀 주소 검증을 사용하는 다른 흐름(간편결제 등록 브릿지)에도 동일한 취약 패턴이 있었다. 같은 보안 경계에서 발생하는 문제라 함께 막았고, 그 함수에는 기존 테스트가 없어 열다섯 개를 추가했다.

## 배운 것

1. **분기 기준은 "무엇이 다른가" 가 아니라 "무엇 때문에 달라야 하는가" 에서 나온다.** 웹과 웹뷰가 다른 건 사실이지만, 「돌아가기」 가 달라야 하는 이유는 채널이 아니라 복귀 대상의 위치였다. 기준을 "복귀 대상이 이 창 안에 있는가" 로 바꾸자 네 경우가 한 규칙이 됐고, 마이페이지 메뉴처럼 나중에 나온 경우도 규칙 안에서 풀렸다.
2. **런타임에서 알 수 없는 사실은 아는 쪽이 선언한다.** `opener` 를 끊는 건 보안상 맞는 결정이고, 그 결정은 "새 탭인지" 라는 정보를 지운다. 지워진 정보는 복원하려 하지 말고 지우기 전에 다른 채널로 보낸다.
3. **선언을 믿는 것과 맥락으로 거르는 것은 함께 간다.** `openedInNewTab` 은 웹의 `window.open` 에서만 나올 수 있다. 웹뷰에서 보이면 신뢰할 수 없는 입력이다 — 오래된 링크일 수도, 잘못 조립된 URL 일 수도 있다. 값의 출처가 정해져 있으면 그 출처 밖에서는 무시하는 게 맞다.
4. **닫는 동작은 되돌릴 수 없으니 실패 경로부터 본다.** `close()` 는 거부될 수 있고, 허용돼도 실제 닫기는 이후에 실행된다. "닫히지 않았을 때" 를 다음 틱에서 `closed` 로 확인하고 이동으로 폴백한다.
5. **경로를 건너다니는 값은 검증 지점이 하나여야 한다.** `returnTo` 는 넛지가 만들고 랜딩이 전달하고 브릿지가 쓴다. 검증은 쓰는 곳 하나에 두고, 만드는 곳과 전달하는 곳은 신뢰하지 않는다.

## 남은 것

웹뷰에서 상품 상세로 "이동" 하는 경우는 이 글의 규칙대로 구현했는데도 Android 에서 무반응이었다. 앱이 https 상품 상세 이동을 가로채는 구조 때문이었고, 딥링크로 바꿔 해결했다. 그 이야기는 [같은 코드가 iOS 에서는 되고 Android 에서는 조용했다](https://changbaebang.github.io/2026-09-14-same-code-android-silent-webview-intercept/) 에 적었다. 딥링크는 새 화면을 여는 것이라 브릿지 웹뷰가 뒤에 남는데, "딥링크로 열고 현재 웹뷰를 닫는" 조합은 앱 쪽 협의가 필요해 후속으로 남겼다. 규칙으로 보면 이것도 "복귀 대상이 창 밖에 있으니 닫는다" 의 한 변형이다.

## 읽을 거리

- [같은 코드가 iOS 에서는 되고 Android 에서는 조용했다](https://changbaebang.github.io/2026-09-14-same-code-android-silent-webview-intercept/) — 이 규칙대로 구현하고도 Android 에서 무반응이었던 이유.
- [넛지에서 랜딩까지: pathname 과 data-* 로 위치를 남기기로 한 이유](https://changbaebang.github.io/2026-04-02-event-with-dom-attributes/) — 같은 흐름의 앞 방향. 넛지 맥락을 어떻게 실어 보내는가.
- [Reverse Tabnabbing — OWASP](https://owasp.org/www-community/attacks/Reverse_Tabnabbing) — `opener` 를 끊어야 하는 이유.
- [`window.close()` — HTML Living Standard](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-window-close) — script-closable 조건, `is closing` 상태와 `closed` getter.
- [URL Standard — special scheme 의 `\` 처리](https://url.spec.whatwg.org/#special-scheme) — `/\evil.com` 이 `//evil.com` 이 되는 근거.
- [Unvalidated Redirects and Forwards — OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) — 복귀 주소 검증의 일반 원칙.
