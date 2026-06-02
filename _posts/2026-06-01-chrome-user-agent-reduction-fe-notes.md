---
layout: post
title: "Chrome User-Agent 변경 안내 — 무엇이, 언제, 어떻게 바뀌었는지"
date: 2026-06-02 10:30:00 +0900
tags: [chrome, user-agent, privacy-sandbox, frontend, web-platform, client-hints]
---

> 이 글은 공식 문서를 옮기는 글이 아니라, **팀에 전달하기 좋은 안내**에 가깝다.  
> “UA가 없어지나?” “이미 바뀐 건가?” “우리 코드는 괜찮나?” — 질문 순서대로 정리했다.

많은 분이 기억하는 이야기는 대략 이렇다. **구글이 User-Agent를 바꾼다.** 그런데 DevTools를 열어 보면 `navigator.userAgent`는 **여전히 길고**, 형식도 **예전과 비슷**하다. 그래서 “결국 안 바꿨네?”라고 느끼기 쉽다.

실제로 Chrome이 한 일은 **헤더를 없애는 것**이 아니라, **문자열 안의 정보를 단계적으로 줄이는 것**이다. 줄인 뒤에도 **껍데기 문자열은 남겨 두었고**, 더 자세한 정보가 필요할 때는 **다른 통로(User-Agent Client Hints)**를 쓰도록 안내하고 있다.

아래는 [Privacy Sandbox](https://privacysandbox.google.com/protections/user-agent), [Chromium rollout](https://www.chromium.org/updates/ua-reduction/), [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/User-agent_reduction)을 바탕으로 한 **안내용 요약**이다.

---

## 한눈에 보는 핵심 (먼저 읽어도 됩니다)

### User-Agent의 역할 — 예전과 지금

| 구분 | **예전 (많이 쓰이던 방식)** | **지금 (Chrome 기준)** |
|------|---------------------------|------------------------|
| **역할** | 요청마다 **긴 문자열**로 브라우저·OS·기기 정보를 **일괄 전달** | 같은 헤더/JS API는 남지만, **민감한 세부 정보는 기본으로 비움** |
| **FE가 기대하던 것** | UA 파싱만으로 **모델명·OS 패치·풀 브라우저 버전** 확보 | 그 필드는 **의미가 바뀌었거나 고정값** — **파싱만으로는 부족**할 수 있음 |
| **상세 정보가 필요할 때** | UA 문자열만 보면 됨 | **UA-CH**로 “필요한 힌트만” 요청 (`Sec-CH-UA-*`, `navigator.userAgentData`) |
| **형식** | `Mozilla/5.0 …` | **형식은 유지** — 기존 코드가 당장 깨지지 않게 |
| **일정** | “언젠가 바뀐다” | Chrome **113(2023년 5월)부터 reduced UA가 기본** (rollout 완료) |

### 세 줄로 전달할 때

> 1. **User-Agent는 없어지지 않았다.** 줄였고(reduction), Chrome에서는 **이미 적용이 끝났다.**  
> 2. **UA의 역할은 “모든 정보의 기본 통로”에서 “거친 요약 + 호환용 껍데기”로 옮겨갔다.**  
> 3. **모델·OS 버전·풀 브라우저 버전**이 필요하면 **UA 파싱이 아니라 UA-CH**를 보면 된다.

### “안 바뀐 것 같다”고 느껴지는 이유

- 문자열 **형태**는 그대로라서, DevTools만 보면 변화가 작아 보인다.
- Google이 말한 것은 **remove가 아니라 freeze·unify**에 가깝다 ([Blink Intent](https://groups.google.com/a/chromium.org/g/blink-dev/c/-2JIRNMWJ7s)).
- **인앱 WebView·앱이 붙인 커스텀 UA**는 브라우저 reduction과 **별개**로 남는 경우가 많다.

---

## User-Agent는 원래 무엇을 했나

HTTP 요청마다 브라우저가 `User-Agent` 헤더(와 `navigator.userAgent`)로 **“나는 이런 클라이언트다”**라고 알려주었다.

원래 의도에 가깝게 쓰이면 **콘텐츠 협상** — 이 브라우저·이 OS에 맞는 HTML/CSS/JS를 주자 — 였다.  
시간이 지나면서는 **기기별 UI 분기**, **버전별 polyfill**, **로그·분석**, **봇 구분** 등으로 쓰이면서 문자열이 길어졌고, 그 안의 정보가 **수동 지문(fingerprinting)**에도 쓰일 수 있게 되었다.

Chrome·업계의 방향은 “통로를 없애자”보다 **“기본으로 나가는 정보를 줄이고, 필요할 때만 명시적으로 받자”**에 가깝다.

---

## 단계적으로 어떻게 바뀌었나 (Chrome)

아래는 **Chrome만** 기준이다. Safari·Firefox는 정책·시기가 다를 수 있다.

### 1단계 — 경고와 시범 (2021~2022)

- DevTools에서 `navigator.userAgent` 등 접근에 **경고**를 붙이기 시작했다.
- Origin Trial로 **축소된 UA**를 미리 맛볼 수 있게 했다.
- 동시에 **User-Agent Client Hints(UA-CH)** API를 정착시켰다. “나중에 줄일 때 쓸 대체 통로”를 먼저 깔아 둔 시기다.

### 2단계 — 브라우저 버전 숫자만 먼저 줄임 (Chrome 101, 2022년 4월)

UA 안의 Chrome 버전 표기에서 **minor·patch가 `0.0.0`으로 고정**되기 시작했다.

- 예: `Chrome/90.0.4430.85` → `Chrome/90.0.0.0`
- **major 버전**은 여전히 읽을 수 있다.
- “패치 번호로 capability를 나누던” 로직은 여기서부터 **의미가 약해진다.**

### 3단계 — 데스크톱 OS·플랫폼 정보 축소 (Chrome 107, 2022년 10월)

**데스크톱** 요청의 UA에서 플랫폼·OS 관련 **세부 버전**이 줄어든다.

- Windows·macOS·Linux·ChromeOS 등에서 **고정·축약된 플랫폼 문자열**이 쓰인다.
- 예: macOS는 `Macintosh; Intel Mac OS X 10_15_7`처럼 **실제 OS 버전과 다를 수 있는** 고정값이 될 수 있다 ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/User-agent_reduction)).

### 4단계 — Android 모바일·태블릿 (Chrome 110, 2023년 2월)

**Android 모바일·태블릿** UA에서 **기기 모델명·Android OS 버전**이 축소된다.

- 흔히 보이는 형태: `Android 10; K` — **실제 기기 OS·모델과 1:1이 아닐 수 있음**
- “Galaxy S24인지 Pixel인지 UA만으로 구분”하던 코드는 **이 단계부터 신뢰하기 어렵다.**

### 5단계 — 전면 적용 완료 (Chrome 113, 2023년 5월)

**모든 Chrome 페이지 로드**에 reduced UA가 기본으로 적용되었다.  
일시적으로 **예전 UA를 쓸 수 있던 연장 trial**도 여기서 끝났다.

- 공식 문서 표현: **“User-Agent reduction is now complete.”**
- 따라서 “곧 바뀐다”가 아니라, **이미 바뀐 뒤를 유지·점검하는 단계**에 가깝다.

### 단계 요약표

| 단계 | Chrome | 바뀐 내용 (요지) |
|------|--------|------------------|
| 준비 | 92~100 | 경고, trial, UA-CH 도입 |
| ① | 101 | Chrome **minor/patch → 0.0.0** |
| ② | 107 | **Desktop** OS/플랫폼 세부 정보 축소 |
| ③ | 110 | **Android** 모델·OS 버전 축소 |
| ④ | 113 | **전체 기본 적용** (완료) |

상세 일정·예시 문자열: [chromium.org/updates/ua-reduction](https://www.chromium.org/updates/ua-reduction/)

### 축소 후 UA 예시 (Android Chrome)

```text
Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Mobile Safari/537.36
```

읽을 때 참고:

- `Android 10; K` — **플랫폼·모델 자리가 고정·축약**되었을 수 있음
- `143.0.0.0` — **143**만 보통 의미 있고, `.0.0.0`은 고정 패턴

---

## 지금 User-Agent의 역할은 무엇인가

여기가 **안내의 중심**이다. “UA가 죽었다”가 아니라 **역할이 나뉘었다**고 보면 편하다.

### User-Agent 문자열이 **여전히 하는 일**

- **호환용 식별자** — 예전 코드·서드파티·로그 파이프라인이 **형식을 기대**하기 때문에, `Mozilla/5.0 …` 형태는 유지된다.
- **거친 분류** — 대략적인 브라우저 계열, **모바일 여부**, OS **이름** 정도는 reduced UA에도 남는 경우가 많다.
- **앱·봇이 붙인 추가 정보** — WebView나 앱이 **자체 토큰**을 UA에 실어 보내는 패턴은 reduction과 **별도**로 남을 수 있다 (Chrome FAQ에서도 “봇·앱의 선택”으로 구분).

### User-Agent 문자열이 **더 이상 맡기 어려운 일** (Chrome 일반 웹)

- **정확한 기기 모델명** (예: 특정 Pixel·Galaxy 구분)
- **정확한 OS / 플랫폼 패치 버전**
- **Chrome minor·patch·build**로 한정한 capability 판별

이런 정보가 **업무상 꼭 필요**하면, UA 파싱을 계속 믿기보다 **UA-CH로 opt-in**하는 쪽이 공식 문서가 권하는 방향이다.

### 그다음 통로: User-Agent Client Hints (UA-CH)

UA-CH는 **“필요한 클라이언트 정보만 요청해서 받는”** 방식이다.

**기본으로 자주 오는 것 (low-entropy)**  
대부분의 요청에 이미 붙는다.

| 헤더 / API | 내용 |
|------------|------|
| `Sec-CH-UA` | 브라우저 브랜드·major |
| `Sec-CH-UA-Mobile` | 모바일 여부 |
| `Sec-CH-UA-Platform` | OS 이름 |

**추가로 요청하는 것 (high-entropy)**  
서버가 `Accept-CH`로 요청하거나, JS에서 `getHighEntropyValues`로 받는다.

| 예시 | 내용 |
|------|------|
| `Sec-CH-UA-Model` | 기기 모델 |
| `Sec-CH-UA-Platform-Version` | OS/플랫폼 버전 |
| `Sec-CH-UA-Full-Version` | 풀 브라우저 버전 (필요 시) |

클라이언트 예시:

```javascript
navigator.userAgentData?.getHighEntropyValues(['model', 'platformVersion'])
  .then((hints) => {
    // 필요한 필드만 사용
  });
```

**역할 분담을 한 문장으로:**

> **UA 문자열** = “항상 가는, 짧아진 요약 + 레거시 호환”  
> **UA-CH** = “정말 필요할 때만 받는 상세 정보”

마이그레이션 절차·첫 요청 이슈(`Critical-CH` 등)는 [web.dev — UA-CH로 이전 (ko)](https://web.dev/articles/migrate-to-ua-ch?hl=ko)를 보면 된다.

---

## 직접 확인해 보기 (안내용 링크)

| 무엇을 보나 | 링크 |
|-------------|------|
| 내 브라우저의 reduced UA | https://goo.gle/reduced-ua-demo |
| 내 브라우저의 UA-CH (헤더·JS) | https://goo.gle/ua-ch-demo |
| 로컬에서 reduced 강제 | Chrome 주소창 `chrome://flags/#reduce-user-agent` |

---

## 프론트엔드 팀에 드리는 안내

급하게 “전부 고쳐라”가 아니라, **어디를 보면 되는지**만 정리한다.

### 1. 코드에서 UA를 쓰는 곳 찾기

- 클라이언트: `navigator.userAgent`, `navigator.platform`, `navigator.appVersion`
- 서버(SSR): `User-Agent` 요청 헤더
- 라이브러리: `ua-parser-js` 등 **문자열 파싱** 유틸

공식 문서도 첫 단계를 **“어디서 UA 데이터를 쓰는지 감사”**로 안내한다.

### 2. 용도별로 나눠 보기

| 용도 | 안내 |
|------|------|
| **모바일 vs 데스크톱** coarse 분기 | UA만 의존하기보다 **media query·Client Hints·뷰포트**와 함께 보는 편이 안전한 경우가 많다 |
| **특정 기기 모델·OS 패치** | Chrome 일반 웹에서는 UA 파싱 **신뢰도 하락** → **UA-CH** 또는 다른 제품 신호 검토 |
| **Chrome minor/patch 버전** | reduced UA에서는 **0.0.0 고정** → **major + feature detection** |
| **인앱 WebView·앱 버전** | 앱이 붙인 **커스텀 UA 토큰**은 reduction과 **별도 계약** — “UA 전부 obsolete”로 묶지 말 것 |
| **광고·어트리뷰션 SDK** | SDK가 UA-CH를 쓰는지, UA 파싱만 쓰는지 **한 번만 맞춰 보기** |

### 3. 서버·인프라와 같이 볼 때

SSR 첫 HTML에서 **모델명이 꼭 필요**하면, 브라우저가 보내는 `Sec-CH-UA-*`를 **백엔드·CDN이 받도록** 설정되어 있는지 확인한다.  
FE만으로 끝나지 않는 경우가 많다.

### 4. 당장 하지 않아도 되는 것

- “곧 User-Agent 헤더가 **완전히 삭제**된다”는 전제로 대규모 제거 작업을 시작할 필요는 없다. 스펙·Intent 모두 **단기간 완전 제거는 어렵다**는 전제에 가깝다.
- **프레임워크 교체**가 이 안내의 결론이 아니다. **문자열은 남았는데 필드 의미가 바뀌었다**는 인식을 팀에 공유하는 것이 1차 목표다.

---

## 다른 브라우저·앱에 대해

- **Chrome**: 위 단계가 **완료된 상태**로 이해하면 된다.
- **Safari**: 과거 UA freeze 시도·일부 조정 이력이 있어, **Chrome과 동일한 타임라인**으로 보지 않는 것이 좋다.
- **앱 WebView**: 네이티브 앱·하이브리드는 **자체 UA 규칙**이 reduction 뉴스와 **겹치지 않을 수 있다.** 앱 팀과 **UA 계약**을 따로 확인하는 편이 낫다.

---

## 더 읽고 싶을 때 (공식 순서)

1. [Privacy Sandbox — User-Agent reduction](https://privacysandbox.google.com/protections/user-agent) — 개요·완료 상태  
2. [Chromium — User-Agent Reduction](https://www.chromium.org/updates/ua-reduction/) — 단계별 rollout  
3. [MDN — User-Agent reduction](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/User-agent_reduction) — Before/After·고정값 설명  
4. [web.dev — UA-CH로 이전 (ko)](https://web.dev/articles/migrate-to-ua-ch?hl=ko) — 코드·서버 마이그레이션  
5. [WICG — User-Agent Client Hints](https://wicg.github.io/ua-client-hints/) — 스펙·장기 방향  

---

## 마무리

“구글이 UA를 바꾼다”는 말은 **헤더 삭제**가 아니라, **기본으로 나가는 정보를 줄이고**, 필요하면 **UA-CH로 받으라**는 안내에 가깝다. Chrome에서는 **2023년 중반(113)부터 그 안내가 기본 동작**이다.

DevTools에서 UA가 **예전처럼 길어 보이는 것**과, 그 안의 **모델·OS 버전·패치 번호를 믿어도 되는지**는 **다른 이야기**다.  
팀에는 **역할이 나뉘었다**는 점만 분명히 전달해 두면, 이후 코드 리뷰·로그·분기 설계에서 헷갈림이 많이 줄어든다.
