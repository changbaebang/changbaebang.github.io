---
layout: post
title: "WebView와 UA — 채널보다 브라우저 스펙을 보는 쪽이 낫다"
date: 2026-06-08 13:26:00 +0900
tags: [chrome, user-agent, webview, frontend, web-platform, client-hints]
---

> [앞글 — Chrome User-Agent 변경 안내](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)은 **브라우저 UA가 어떻게 줄었는지**였다.  
> 이번 글은 WebView·모바일 웹을 다루면서, UA를 어떻게 써 왔는지 — 그리고 **“웹뷰냐 아니냐”만으로 나누는 것**의 한계를 적는다. 개인 메모에 가깝다.

---

## 배경

Chrome **reduction은 이미 기본**이다. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/User-agent_reduction)도 **브라우저 버전 감지보다 feature detection**을, 그다음 **progressive enhancement**를 권한다.

현장에서는 여전히 이런 말이 나온다.

- “WebView니까 `navigator.userAgent`만 보면 된다.”
- “앱 토큰 + 브라우저 UA를 **한 덩어리**로 파싱하면 된다.”

저도 한동안 **채널(웹뷰 / 브라우저)** 로 먼저 나누는 쪽에 기울었다. 다시 보니, **플랫폼·기능**까지 그 이분법에 맡기기는 **어렵다**고 느꼈다.

---

## WebView도 “브라우저 하나”가 아니다

**WebView도 브라우저**다. iOS **WKWebView**는 WebKit 계열, Android **System WebView**는 Chromium인데, **앱·기기마다 탑재 버전이 다르다.**

Android 쪽 공식 가이드([Jetpack Webkit](https://developer.android.com/develop/ui/views/layout/webapps/jetpack-webkit-overview))도 비슷한 말을 한다. OS 버전과 별개로 **설치된 WebView APK**에 따라 쓸 수 있는 API가 달라지니, **`WebViewFeature.isFeatureSupported()`로 기능 지원을 본다**고 적혀 있다. “Android WebView = 한 스펙”이 아니라는 뜻이다.

[Chrome](https://developer.chrome.com/blog/user-agent-reduction-android-model-and-version/)은 Android UA에서 **기기 모델·OS minor**가 고정값으로 줄어드는 변경을 WebView·Chrome 모두에 적용해 왔다. “앱 안이니까 UA는 예전 그대로”라고 가정하기 **점점 어렵다.**

그래서 **`isWebview()` 한 방**으로 sticky, API 지원, reduction 대응까지 가르는 건, 생각보다 **맞지 않는 경우**가 많다. 필요한 건 종종 **“지금 이 런타임에서 이게 되는가?”** — [MDN의 UA sniffing 가이드](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent)가 말하는 **feature detection** 쪽에 가깝다.

---

## UA를 “많이” 쓰는 흔한 형태

하이브리드·WebView 상품에서 자주 보이는 패턴이다. 특정 회사만의 이야기는 아니다.

| 패턴 | 왜 쓰나 | reduction·스펙 쪽 |
|------|---------|-------------------|
| WebView 여부만 보고 분기 | 빠르다 | WebView **안에서도** 엔진·버전이 다름 |
| 앱 버전을 UA **커스텀 토큰**에서 | 브릿지 없을 때 | **제품 신호**로는 자연스럽다. CSS/API와는 별개 |
| `ua-parser`로 Safari·Chrome minor | 예전 웹 코드 재사용 | 문자열 **의미가 줄어든** 필드가 있음 |
| `isMobile`을 UA만으로 | SSR | **media query**와 같이 쓰는 편이 안전 ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent)) |
| WebView에 Safari 전용 우회 그대로 | “앱만 본다” | Android WebView와 **엔진이 다름** |

WebView가 문제가 아니라, **채널로 플랫폼을 대신**하거나 **UA 한 줄로 스펙을 대신**하려 할 때 부담이 커진다.

---

## “웹뷰 vs 브라우저” — 쓸 때와 아닐 때

이분법이 **여전히 도움이 되는 곳**은 **제품·채널** 쪽이다.

- 네이티브 **브릿지**, 딥링크, 앱 스킴  
- **앱 버전**과 묶인 기능 on/off  
- 분석을 **앱 WebView / 모바일 웹**으로 나누는 정책  

반면 **플랫폼·스펙** — Safari 우회, API 지원, Client Hints — 에서는 [Microsoft Edge 팀](https://learn.microsoft.com/en-us/microsoft-edge/web-platform/user-agent-guidance)도 **가능하면 브라우저를 감지하지 말고 기능을 감지하라**고 한다. UA-CH가 필요해도 **feature detection과 함께** 쓰라는 권고다.

[Chrome UA-CH 문서](https://developer.chrome.com/docs/privacy-security/user-agent-client-hints) 역시, UA 파싱을 옮기기 **전에** progressive enhancement·feature detection·responsive design으로 **대체할 수 있는지** 다시 보라고 한다. Client Hints로 **기능 분기를 바꾸는 것**도 [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent)은 여전히 비추천한다 — **감지 수단만 바뀌었을 뿐**이기 때문이다.

| 알고 싶은 것 | 흔한 선택 | 더 맞기 쉬운 선택 |
|--------------|-----------|-------------------|
| sticky / layout | webview? | **feature + CSS** |
| 앱 최소 버전 | Chrome minor | **앱 토큰·브릿지** |
| Safari 결제 UI | webview? | **기능·엔진** (WKWebView ≠ Safari 전부) |
| OS·모델 상세 | UA 파싱 | **UA-CH** ([web.dev 마이그레이션](https://web.dev/articles/migrate-to-ua-ch)) |

**웹뷰냐**보다 **무엇을 알고 싶은지**에 맞는 신호를 고르는 편이, 공식 가이드와도 방향이 맞는다.

---

## 살짝만 — 제가 본 형태 (일반화)

커머스 **앱 WebView** FE를 보며 느낀 것만 짧게. 감사 보고서는 아니다.

- 앱 UA **`APP…(serviceVersion=…)`** — 웹뷰·앱 배포 판별용 **제품 계약**으로는 자연스럽다.  
- 같은 레포 어딘가에서 WebView 화면도 **`ua-parser-js`**로 브라우저·OS minor를 읽는다.  
- `Mobile Safari` 분기, OS 버전 regex — **채널과 무관하게** UA 파서에 기대는 코드.

[web.dev](https://web.dev/articles/migrate-to-ua-ch)가 말하듯, 먼저 **`navigator.userAgent`가 어디서 쓰이는지** 찾아보는 게 1단계다.  
저는 그다음 **“이건 product냐 platform이냐”**만 구분해 보았다. WebView **종류·WebView APK 버전**까지는 아직 코드에 잘 안 드러난다 — 그게 **준비가 덜 된** 느낌의 출발점이었다.

---

## 준비 — 크게 하지 않아도 된다

전면 UA-CH 마이그레이션이 1순위는 아니다. **신호만 나눠 보기.**

```text
product   → 앱 토큰, 브릿지, 앱 버전 (채널·배포)
platform  → feature detection, progressive enhancement
telemetry → 로그용 UA (기능 분기와 분리)
```

`isWebview`는 **product**에 두고, **platform** 분기에는 넣지 않는 쪽을 시험해 본다.

```bash
rg 'navigator\.userAgent|UAParser|ua-parser' --glob '*.{ts,tsx}'
```

각 hit에 **앱 기능인가 / 브라우저 capability인가**, **minor·모델을 기대하는가**만 적어 보면 된다.

당장 **모든 파일에 `if (isWebview)`를** 더 붙이거나, UI를 **두 벌**로 쪼개지 않아도 된다. [앞글](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)처럼 “전부 고쳐라”가 아니라 **어디를 볼지** 정하는 정도면 충분하다.

---

## 마무리

[앞글](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)이 **정책 타임라인**이었다면,  
이번 글은 **WebView FE가 헷갈리기 쉬운 축**을 정리한 것이다.

- WebView = **브라우저 하나**가 아니다 (Android는 [Jetpack Webkit](https://developer.android.com/develop/ui/views/layout/webapps/jetpack-webkit-overview)도 **기능 단위**를 말한다).  
- **`isWebview`만으로 CSS·API를 가르기** — product에는 쓸 만하고, platform에는 **부족할 수 있다.**
- [MDN·Chrome·Microsoft](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent)가 공통으로 말하는 방향: **feature detection 우선**, UA는 **줄이고**, 꼭 필요할 때 **Client Hints**.

“웹뷰인지 구분 없이 UA만 본다”도 아쉽고,  
“웹뷰냐 아니냐만 구분하면 된다”도 **충분하지 않을 수 있다.**

grep 한 번, **product / platform** 한 줄 — 그 정도가 지금의 **준비**로 느껴진다.

---

## 참고

### 앞글·데모

- [Chrome User-Agent 변경 안내 (앞글)](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)
- [Reduced UA demo](https://goo.gle/reduced-ua-demo) · [UA-CH demo](https://goo.gle/ua-ch-demo)

### UA reduction · Client Hints

- [MDN — User-Agent reduction](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/User-agent_reduction) — feature detection, progressive enhancement, Client Hints  
- [MDN — Browser detection using the user agent (UA sniffing)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent) — “알고 싶은 건 브라우저가 아니라 기능”; desktop/mobile 같은 브라우저도 다를 수 있음  
- [web.dev — Migrate to User-Agent Client Hints](https://web.dev/articles/migrate-to-ua-ch) — `navigator.userAgent` 사용처 감사  
- [Chrome — User-Agent Client Hints](https://developer.chrome.com/docs/privacy-security/user-agent-client-hints) — UA 대신 feature detection·responsive design 검토  
- [Chrome — Android model/version reduction](https://developer.chrome.com/blog/user-agent-reduction-android-model-and-version/) — WebView·Android UA 고정값  
- [Microsoft Edge — User agent guidance](https://learn.microsoft.com/en-us/microsoft-edge/web-platform/user-agent-guidance) — feature detection 우선, UA-CH와 병행  

### WebView · capability (플랫폼 쪽)

- [Android Developers — Jetpack Webkit overview](https://developer.android.com/develop/ui/views/layout/webapps/jetpack-webkit-overview) — WebView APK별 기능, `WebViewFeature.isFeatureSupported()`  
- [Microsoft — How to Detect Features Instead of Browsers](https://learn.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/samples/hh273397(v=vs.85)) — desktop/mobile 같은 브라우저도 기능이 다를 수 있음  
