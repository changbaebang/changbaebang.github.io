---
layout: post
title: "같은 코드가 iOS 에서는 되고 Android 에서는 조용했다 — 웹뷰 URL 인터셉트가 두 곳으로 갈라져 있을 때"
date: 2026-09-15 11:50:00 +0900
excerpt: "웹뷰 안에서 상품 상세로 돌아가는 버튼이 iOS 에서는 동작했고 Android 에서는 아무 반응이 없었는데, 프런트엔드 코드는 양쪽에 똑같은 이동을 요청하고 있었다. 브라우저에서 UA 를 바꿔 테스트해도 잡히지 않는 이 차이는 앱 저장소를 직접 읽고 나서야 사실이 됐다."
categories: [Engineering, Frontend]
tags: [WebView, Android, iOS, Deep Link, Hybrid App, Debugging, Root Cause Analysis]
---

> 내부 private repo 작업을 바탕으로 썼다.
> 외부 접근이 불가한 저장소라 링크 없이, 조직·시스템·사람을 식별할 수 있는 정보는 걷어내고 구조만 남긴다.

배포 전날 QA 환경에서 Android 실기기로 마지막 확인을 하다가, 「이전 페이지로 돌아가기」 버튼이 아무 반응이 없는 것을 봤다. 같은 빌드를 iOS 로 확인한 동료와 기획자는 "잘 된다" 고 했다. 주문서나 마이페이지에서 들어온 경우는 Android 도 잘 돌아갔다. 상품 상세에서 들어온 경우만, Android 만, 조용했다.

프런트엔드 코드는 어느 지면에서 왔든 같은 일을 한다. 들어올 때 받아둔 복귀 주소로 `window.location.replace(returnTo)` 를 호출한다. 플랫폼 분기가 없으니 코드가 다르게 동작할 이유가 없다. 그렇다면 차이는 코드 밖에 있다.

## TL;DR

- 웹뷰 안에서 `https://…/products/{id}` 로 이동하는 요청은 **iOS 와 Android 앱 모두 가로챈다.** "Android 만 인터셉트한다" 는 첫 추측은 증상은 설명했지만 원인은 절반만 설명했다.
- iOS 는 가로챈 자리에서 곧바로 새 웹뷰를 띄운다. 판단과 처리가 한 함수다. Android 는 **"현재 웹뷰의 이동을 막을지"** 와 **"새 웹뷰를 띄울지"** 를 두 곳에서 따로 판단하고, 두 곳이 **다른 URL 을 기준**으로 본다. 그 사이에 로그인 복귀용 가드가 끼어 있어 우리 시나리오에서는 막기만 하고 띄우지 않았다.
- 브라우저에서 User-Agent 를 앱처럼 바꿔 테스트하면 `isWebview()` 분기까지만 재현된다. 앱의 URL 인터셉트는 브라우저에 없는 계층이라 원리상 잡히지 않는다.
- "iOS 는 된다" 는 안심의 근거가 아니었다. iOS 는 웹이 의도한 방식(현재 웹뷰 이동)이 아니라 앱 라우터의 별도 경로(새 웹뷰)로 성공한 것이었다.
- 앱 저장소를 읽는 비용은 추측을 문서에 적어두는 것보다 쌌다.

## 흐름과 증상

기능은 세 단계다. 상품 상세·주문서·마이페이지의 넛지에서 카드 혜택 랜딩으로 가고, 랜딩에서 브릿지 페이지로 가고, 브릿지가 카드사로 보낸다. 사용자가 발급을 마치고 앱으로 돌아오면 브릿지가 그대로 떠 있고, 「이전 페이지로 돌아가기」 를 누르면 처음 들어왔던 지면으로 복귀한다. 복귀 주소는 넛지가 쿼리로 실어 보내고 랜딩이 브릿지까지 그대로 전달한다.

버튼 핸들러는 (당시 기준으로) 이랬다.

```ts
const handleBack = () => {
  if (!returnToUrl) {
    redirect({ path: homePath });
    return;
  }
  redirect({ path: returnToUrl }); // 내부적으로 window.location.replace
};
```

증상을 표로 놓으면 원인이 코드가 아니라는 게 분명해진다.

| 진입 지면 | 복귀 주소 | iOS | Android |
|---|---|---|---|
| 주문서 | `https://…/order/checkout?…` | 복귀 | 복귀 |
| 마이페이지 | `https://…/my-page/…` | 복귀 | 복귀 |
| 상품 상세 | `https://…/products/{id}` | 상품 상세가 뜸 | **무반응** |

같은 핸들러, 같은 `replace`. 주소의 경로만 다르다. 그러니 `/products/` 라는 경로를 특별 취급하는 무언가가 Android 웹뷰 바깥에 있다.

## 첫 추측: "Android 만 가로챈다"

당장 배포 전이라 먼저 고쳤다. 저장소 관례를 보니 웹뷰에서 상품 상세로 가는 링크는 예외 없이 `app://product/{id}` 딥링크였다. 상품 카드 훅, 링크 빌더, 열여덟 군데가 전부 그랬고 웹뷰 안에서 `https` 로 상품 상세를 여는 코드는 한 군데도 없었다. 브릿지 버튼이 그 관례를 어긴 첫 사례였던 것이다.

```ts
if (isInWebview) {
  const productId = parseProductIdFromReturnTo(returnToUrl);
  if (productId) {
    window.location.href = `app://product/${productId}`;
    return;
  }
}
redirect({ path: returnToUrl });
```

Android 실기기에서 상품 상세가 열렸고, iOS 도 동료 기기로 확인했다. 배포 준비는 끝났다.

그런데 "왜" 는 남았다. 그 시점의 내 설명은 이랬다. *Android 앱의 `WebViewClient.shouldOverrideUrlLoading` 이 `/products/` 경로를 가로채서 `true` 를 반환하고 아무것도 하지 않는다. iOS 의 `WKNavigationDelegate` 는 허용한다.* 그럴듯했고, 증상과 맞았다. 그런데 Android 쪽 증상은 일부 설명했지만, iOS 가 허용한다는 부분은 틀렸고 Android 에서 대체 동작이 빠지는 이유도 설명하지 못하고 있었다.

앱 저장소는 읽을 수 있는 권한이 있었다. 배포가 끝난 뒤 열어봤다.

## Android: 판단이 두 곳에 있고, 기준 URL 이 다르다

Android 웹뷰 클라이언트의 `shouldOverrideUrlLoading` 은 이렇게 끝난다. (식별자는 바꿨고, 이 글과 무관한 앞선 분기는 뺐다.)

```kotlin
override fun shouldOverrideUrlLoading(view: WebView?, request: WebResourceRequest?): Boolean {
    // 리다이렉트 허용 규칙, 새 페이지 강제 모드 등 앞선 분기는 생략
    val uri = request?.url ?: return false
    updateUri(uri) // ViewModel 로 흘려보낸다
    return !UriTypes.isWebType(uri) ||
        uriFilter.shouldOverrideUrlLoading(view?.originalUrl.orEmpty())(uri)
}
```

두 가지가 눈에 들어온다. 반환값은 **현재 웹뷰의 이동을 막을지** 만 결정한다. 그리고 그 판단의 기준은 `view.originalUrl`, 즉 **지금 떠 있는 페이지** 의 URL 이다. 우리 경우 브릿지 페이지다.

그럼 새 웹뷰는 누가 띄우나. `updateUri(uri)` 로 넘어간 URL 을 ViewModel 이 받아 처리한다.

```kotlin
private fun handleWebUrl(type: UriTypes.Web) {
    // ... 로그인 URL 등 앞선 분기 생략
    if (uriFilter.isNewWebViewOpen(originalUrl = originalUri.toString(), currentLoadUrl = type.uri.toString())) {
        WebForegroundEffects.NavigateWebView(type.uri.toString()).offer()
        return
    }
    if (uriFilter.isKakaoMapDetailViewUrl(type.uri)) { /* ... */ return }
    if (uriFilter.isHomeUrl(type.uri)) { /* ... */ return }
}
```

여기서 기준은 `originalUri` 다. 클라이언트의 `view.originalUrl` 과 이름이 비슷하지만 다른 값이다. `savedStateHandle` 에서 읽는, **이 웹뷰 액티비티가 처음 열렸을 때의 URL** 이다.

둘 다 같은 필터 함수 `isNewWebViewOpen(originalUrl, currentLoadUrl)` 을 호출한다. 이 함수는 상품 상세·브랜드·티켓 등 "새 웹뷰로 띄워야 하는 경로" 목록에 `currentLoadUrl` 이 걸리면 참을 돌려준다. 단, 그 앞에 유효성 검사가 있다.

```kotlin
private fun isValidUrlForNewWebViewOpen(originalUrl: String, currentLoadUrl: String): Boolean =
    currentLoadUrl.isNotEmpty() &&
        originalUrl.isNotEmpty() &&
        !isAuthDomain(currentLoadUrl) &&
        !isShopMainUrl(currentLoadUrl) &&
        !originalUrl.isSamePath(currentLoadUrl)   // ← 여기
```

마지막 줄의 주석은 이랬다. *비로그인 상태에서 상품 상세 → 하트 클릭 → 로그인 → 성공하면 상품 상세를 새로 띄우지 않고 이전 액티비티가 처리하도록 한다.* 로그인하고 같은 상품 상세로 돌아올 때 화면이 한 장 더 쌓이는 걸 막는 가드다. 합리적인 규칙이다.

이제 우리 시나리오를 넣어보자. 넛지는 웹뷰에서 `window.location.href` 로 **같은 웹뷰 안에서** 랜딩으로 간다. 랜딩도 같은 방식으로 브릿지로 간다. 그러니 이 웹뷰 액티비티는 상품 상세 URL 로 시작해서 페이지만 세 번 바뀐 상태다.

| | 기준 URL (`originalUrl`) | 요청 URL | `isNewWebViewOpen` | 결과 |
|---|---|---|---|---|
| 클라이언트 | 브릿지 (현재 페이지) | 상품 상세 | 참 | **현재 웹뷰 이동 차단** |
| ViewModel | 상품 상세 (액티비티 최초 URL) | 상품 상세 | `isSamePath` 에 걸려 거짓 | **새 웹뷰 안 띄움** |

막기는 했는데 대체 동작이 없다. 사용자에게는 무반응이다.

주문서와 마이페이지가 됐던 이유도 같은 표로 설명된다. `/order/checkout` 과 `/my-page/…` 는 "새 웹뷰로 띄울 경로" 목록에 없어서 클라이언트가 거짓을 돌려주고, 웹뷰가 그냥 이동한다. 애초에 인터셉트 대상이 아니었다.

딥링크로 고친 것이 맞았던 이유도 같은 코드에 있다. `app://product/{id}` 는 `isWebType` 이 아니라서 클라이언트는 즉시 참을 돌려주고, ViewModel 은 `handleWebUrl` 이 아니라 `handleDeeplink` 로 간다. `isSamePath` 가드를 아예 거치지 않는다.

## iOS: 판단과 처리가 한 곳에 있다

iOS 쪽은 `decidePolicyFor navigationAction` 이 라우터에 URL 을 넘기고, 라우터가 참을 돌려주면 `.cancel`, 아니면 `.allow` 다. 라우터 안에서 `^/products/[0-9]+` 는 이렇게 처리된다.

```swift
if path.range(of: "^\\/products\\/[0-9]+", options: .regularExpression) != nil {
    let urlString = "https://\(host)\(path)\(queryString)"
    navigator?.navigateToWeb(string: urlString)   // 새 웹뷰를 띄운다
    return true                                    // 현재 웹뷰 이동은 취소
}
```

가로채는 건 Android 와 같다. 차이는 가로챈 그 자리에서 새 화면을 띄우고 나서 취소한다는 것이다. "막을까" 와 "띄울까" 가 한 함수의 한 분기라서, 하나만 되고 하나는 안 되는 상태가 만들어지지 않는다.

그래서 iOS 에서 "잘 됐다" 는 것도 우리 코드가 의도한 결과가 아니었다. 우리는 현재 웹뷰를 상품 상세로 이동시키려 했고, iOS 앱은 그걸 취소하고 새 웹뷰를 띄웠다. 결과 화면이 비슷해서 구분이 안 됐을 뿐이다. 수정 전 iOS 는 우리 코드가 맞아서가 아니라 앱 라우터가 다른 경로로 받아줘서 성공한 것이었다.

## 왜 사전에 못 잡았나

셋 다 내 몫이다.

**브라우저 테스트의 한계를 알면서도 그 안에서 끝냈다.** 배포 전 검증은 Playwright 로 브라우저를 열고 User-Agent 를 앱처럼 바꿔서 했다. `isWebview()` 가 참이 되니 웹뷰 분기가 실행되고, 버튼을 누르면 `replace` 가 호출되는 것까지 확인했다. 그런데 브라우저에는 `shouldOverrideUrlLoading` 이 없다. 앱이 URL 을 가로채는 계층은 UA 문자열과 무관하고, 브라우저에서는 존재 자체가 없다. [WebView 와 UA — 채널보다 브라우저 스펙을 보는 쪽이 낫다](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/) 에서 UA 로 채널을 가르는 코드의 취약함을 적었는데, 테스트에서도 같은 함정이었다. UA 는 앱을 흉내 내지만 앱의 네비게이션 정책은 흉내 내지 못한다.

**실기기 확인을 지면 단위로 다 돌리지 않았다.** Android 실기기로 주문서와 마이페이지 복귀를 확인하고 "웹뷰 복귀는 된다" 고 정리했다. 진입 지면이 셋이고 플랫폼이 둘이니 여섯 칸인데 네 칸만 채웠다. 남은 두 칸 중 하나가 문제였다.

**iOS 확인을 안심의 근거로 썼다.** 기획자가 iOS 에서 세 경로를 확인한 영상이 먼저 도착했고, 나는 그걸로 "복귀 로직은 맞다" 고 판단했다. 위에서 본 대로 iOS 는 우리 로직이 아니라 앱의 라우터 덕에 됐다. 한 플랫폼의 성공은 코드의 정확성을 보증하지 않는다. 그 플랫폼이 관대할 뿐일 수 있다.

## 추측을 확정으로 바꾸는 비용

수정 직후 내가 적어둔 원인은 "Android 앱이 `/products/` 를 가로채고 처리하지 않는다" 였다. 배포 티켓에 그렇게 남길 뻔했다. 틀린 말은 아니지만 두 가지가 빠져 있었다. iOS 도 가로챈다는 것, 그리고 Android 가 "처리하지 않는" 이유가 특정 가드의 오탐이라는 것.

앱 저장소는 코드 검색으로 `shouldOverrideUrlLoading` 을 찾고, 필터 클래스와 ViewModel 을 따라가고, iOS 라우터를 대조하는 데 십여 분이 걸렸다. [그럴듯한 설명이 측정을 대신할 때](https://changbaebang.github.io/2026-08-13-plausible-story-instead-of-measurement/) 에서 "설명이 그럴듯할수록 확인을 건너뛰기 쉽다" 고 썼는데, 이번에도 증상과 맞아떨어지는 설명이 손에 있으니 확인을 미뤘다. 십여 분과 "반만 맞은 원인이 기록에 남는 것" 을 저울에 올리면 답은 정해져 있다.

확정된 원인은 후속 작업의 방향도 바꾼다. "Android 가 `/products/` 를 안 받는다" 면 앱팀에 "받아 달라" 고 요청하게 된다. "두 판단의 기준 URL 이 다르고 같은-경로 가드가 오탐한다" 면 요청이 구체적이 된다. 딥링크로 새 화면을 띄우면 브릿지 웹뷰가 뒤에 남는 문제가 아직 있는데, 이 요청을 할 때 함께 전달할 것이 생겼다.

## 배운 것

1. **하이브리드 앱에서 웹의 이동 요청은 제안이다.** `location.replace` 는 브라우저에서는 명령이지만 웹뷰에서는 앱이 거부할 수 있는 요청이다. 웹뷰 안에서 어떤 경로로 갈 때는 저장소에 그 경로를 여는 관례가 있는지 먼저 본다. 열여덟 군데가 딥링크인데 한 군데만 `https` 면 그 한 군데가 틀렸을 가능성이 높다.
2. **판단이 두 곳으로 갈라진 구조는 "하나만 실행된 상태" 를 만든다.** 막는 쪽과 대체하는 쪽이 다른 기준을 보면, 둘 다 자기 규칙대로 맞게 동작하면서 전체는 무반응이 될 수 있다. 앱 코드를 고칠 입장은 아니지만, 이런 구조 앞에서 웹이 할 일은 두 판단이 같은 답을 내는 입력(딥링크)을 주는 것이다.
3. **UA 스푸핑은 분기를 재현하고 정책은 재현하지 않는다.** 웹뷰 관련 변경의 최종 확인은 실기기여야 하고, 진입 지면 × 플랫폼 표를 미리 그려서 빈칸을 남기지 않는다.
4. **한 플랫폼의 성공을 다른 플랫폼의 증거로 쓰지 않는다.** 특히 "된다" 가 우리 코드 때문인지 플랫폼이 관대해서인지 구분되지 않을 때는.
5. **읽을 수 있는 저장소는 읽는다.** 추측이 증상과 맞아도 확정이 아니다. 접근 권한이 있고 십여 분이면 되는 확인을 "다른 팀 코드" 라는 이유로 미루지 않는다.

## 남은 것

딥링크는 "이동" 이지 "복귀" 가 아니다. 상품 상세가 새 화면으로 쌓이고 브릿지 웹뷰는 뒤에 남는다. 뒤로가기를 한 번 더 누르면 브릿지가 다시 보인다. 웹에서 현재 웹뷰를 닫을 방법은 없으니 "딥링크로 열고 현재 웹뷰를 닫는" 동작은 앱 쪽에 요청해야 한다. 이번 분석 덕에 그 요청에 "왜 `https` 복귀가 안 됐는지" 를 코드 위치까지 붙일 수 있게 됐다.

## 읽을 거리

- [WebView 와 UA — 채널보다 브라우저 스펙을 보는 쪽이 낫다](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/) — UA 로 채널을 가르는 코드의 한계. 이번 글은 테스트 쪽에서 같은 한계를 만났다.
- [그럴듯한 설명이 측정을 대신할 때](https://changbaebang.github.io/2026-08-13-plausible-story-instead-of-measurement/) — 증상과 맞는 설명이 확인을 밀어내는 패턴.
- [`WebViewClient.shouldOverrideUrlLoading`](https://developer.android.com/reference/android/webkit/WebViewClient#shouldOverrideUrlLoading(android.webkit.WebView,%20android.webkit.WebResourceRequest)) — 반환값 `true` 는 "이 URL 로딩을 앱이 처리했다" 는 뜻이며, 실제로 처리했는지는 앱의 몫이다.
- [`WKNavigationDelegate.webView(_:decidePolicyFor:decisionHandler:)`](https://developer.apple.com/documentation/webkit/wknavigationdelegate/1455641-webview) — iOS 쪽 대응 API.
- [`WebView.getOriginalUrl()`](https://developer.android.com/reference/android/webkit/WebView#getOriginalUrl()) — "현재 페이지" 의 원래 URL. 액티비티가 처음 열린 URL 과는 다른 값이다.
