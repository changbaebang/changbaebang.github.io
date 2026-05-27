---
layout: post
title: "비밀번호 이후의 로그인: WebAuthn Immediate UI를 붙일 때 놓치기 쉬운 것들"
date: 2026-05-27 10:41:00 +0900
tags: [webauthn, passkey, auth, frontend, chrome, security]
---

> 이 글의 요점은 단순하다.  
> 로그인 UX의 경쟁력은 이제 \"입력을 잘 받는 폼\"보다 \"입력을 덜 하게 만드는 인증\"에서 나온다.

[Chrome DevBlog](https://developer.chrome.com/blog/webauthn-immediate-ui?hl=ko)와 [구현 가이드](https://developer.chrome.com/docs/identity/immediate-ui-mode?hl=ko)를 바탕으로, 번역 재작성 대신 \"제품에 붙일 때 실패하는 지점\"만 정리한다.

비밀번호 인증은 줄어드는 방향이 맞다. Microsoft도 개인 계정에서 passwordless 전환을 공식 가이드로 안내한다([공식 문서](https://support.microsoft.com/ko-kr/accounts-billing/security/how-to-go-passwordless-with-your-microsoft-account)).  
업계 메시지는 꽤 분명하다. \"아이디/비밀번호를 더 잘 입력하게 하는 UX\"보다 \"비밀번호 자체를 덜 쓰는 인증 UX\"로 이동 중이라는 것이다.

## TL;DR

- 대상 브라우저는 **Chrome 149+**를 기준으로 잡는다.
- `mediation: 'immediate'`는 OT 방식이고, stable은 **`uiMode: 'immediate'`**다.
- 이 기능이 동작하려면 사용자의 브라우저/기기에 **패스키 또는 저장 비밀번호가 사전 연결**돼 있어야 한다.
- **WebView는 제약이 크다.** 앱 내 WebView 로그인은 별도 검증 없이 바로 확대하지 않는다.
- 적용 포인트는 API 한 줄이 아니라 **사전 등록(credential 생성) + 로그인 호출 + fallback UX**다.

---

## 인증의 방향이 왜 바뀌는가

사용자는 이미 매일 기기 잠금(지문/얼굴/PIN)으로 신원을 확인한다. 로그인도 같은 방향으로 수렴하는 것이 자연스럽다.

여기서 중요한 기술적 구분이 있다.

- 생체정보 원본을 서비스 서버로 보내 인증하는 모델이 아니다.
- 기기 내부 인증을 통과하면, 기기/플랫폼에 저장된 자격증명(패스키) 사용 권한이 열리는 모델이다.
- 즉 \"생체정보 수집\"보다 \"로컬 보안 경계에서 키 사용 승인\"에 가깝다.

이 구분을 이해하면 보안팀, 기획, 고객 커뮤니케이션 문구가 훨씬 명확해진다.

---

## 왜 이게 편한가 (그리고 왜 오해가 생기나)

사용자 입장에서 편한 이유는 간단하다. 긴 비밀번호를 기억하고 입력하는 대신, 이미 매일 쓰는 기기 잠금(지문/얼굴/PIN)으로 인증할 수 있기 때문이다.

다만 표현은 정확히 구분해야 한다.

- 맞는 말: 모바일에서 지문/얼굴로 인증하는 흐름은 이미 보편적이다.
- 보완할 점: 서버에 지문 원본이 전달되는 게 아니라, **기기 내부의 인증수단으로 패스키(개인키 사용 권한)를 해제**하는 방식이다.
- 즉 \"제3에 생체정보를 저장해서 인증\"이라기보다, \"기기(또는 플랫폼 계정 동기화 저장소)에 있는 자격증명을 기기 생체인증으로 사용 승인\"에 가깝다.

---

## 먼저 제약사항부터

### 1) 브라우저/버전 제약

- 본문 기준 레퍼런스는 Chrome 문서의 **Chrome 149+** 흐름이다.
- 지원 여부는 런타임에서 `PublicKeyCredential.getClientCapabilities()`로 확인한다.
- 미지원 환경(Safari/Firefox/구버전 Chrome)은 기존 로그인 흐름으로 즉시 fallback 한다.

### 2) 사전 연결(등록) 전제

- immediate UI는 \"없는 인증수단을 만들어 주는 기능\"이 아니다.
- 사용자가 이전에 패스키를 만들었거나, 브라우저 비밀번호 매니저에 자격증명이 있어야 의미가 있다.
- 즉, 롤아웃은 보통 `등록률(credential 보유율)`을 먼저 보고 시작한다.

### 3) WebView 제약

- WebView는 브라우저와 동일한 동작을 보장하지 않는다.
- 인증 저장소 접근, 사용자 제스처, 계정 선택 UI 노출에서 차이가 날 수 있다.
- 모바일 앱 WebView는 \"지원 가정\" 대신 **실측 매트릭스**로 판단해야 한다.

---

## API 변경 핵심 (OT → stable)

Origin Trial 시절(레거시):

```javascript
// ❌ Chrome 149+에서 immediate 트리거 아님
navigator.credentials.get({
  mediation: 'immediate',
  // ...
});
```

Chrome 149+:

```javascript
navigator.credentials.get({
  publicKey: { /* ... */ },
  uiMode: 'immediate',
});
```

OT 코드가 남아 있으면 프로덕션에서 조용히 실패할 수 있으니 diff를 먼저 제거한다.

---

## 구현 순서 (실무 버전)

### 1) 인증수단 연결(등록) API 먼저 준비

로그인 API보다 먼저 \"연결\" 경로를 만든다.

```javascript
// 예시: 등록(create) 흐름
const credential = await navigator.credentials.create({
  publicKey: publicKeyCreationOptionsFromServer,
});
// 서버에 attestation/credential 저장
```

- 등록이 끝나야 `navigator.credentials.get()`에서 고를 credential이 생긴다.
- 제품 UX에서는 \"기기 인증수단 연결하기\" 단계를 명시해 둔다.

### 2) 로그인 시 capability 확인 후 immediate 호출

```javascript
const caps = await PublicKeyCredential.getClientCapabilities?.();
const supportsImmediate = caps?.immediateGet === true;

if (!supportsImmediate) {
  return openDefaultLoginFlow();
}

try {
  const cred = await navigator.credentials.get({
    uiMode: 'immediate',
    password: true,
    publicKey: publicKeyRequestOptionsFromServer,
  });
  return verifyCredentialOnServer(cred);
} catch (error) {
  return openDefaultLoginFlow();
}
```

### 3) 오류/예외는 즉시 fallback

- `NotAllowedError`는 흔한 정상 케이스(선택 안 함/없음)로 취급하고 기본 로그인 UI를 연다.
- 네트워크/서버 검증 실패는 분리 로그를 남긴다.
- 에러 문구보다 \"다른 로그인 수단으로 계속하기\" CTA가 더 중요하다.

---

## 테스트 매트릭스 (최소)

- Chrome 149+ / credential 있음 / 없음
- Chrome 149+ incognito
- Safari / Firefox fallback
- Android/iOS 인앱 WebView (지원 여부 확인)

---

## 관측 포인트

- immediate 시도 → 성공/취소/실패 이벤트 로깅 (PII 없이)
- 등록 완료율(credential 생성 성공률)
- WebView vs 브라우저 성공률 분리
- A/B: 로그인 완료율, 이탈률, fallback 진입률

---

## 도입 전략: 어디부터 붙일까

- 전면 치환보다 **Chrome 149+ + 웹 브라우저 진입 로그인**부터 시작한다.
- 성공률이 나오는 구간(credential 보유 사용자)에서 먼저 체감을 만든다.
- WebView는 별도 트랙으로 측정하고, 결과가 쌓일 때까지 기본 로그인을 유지한다.
- KPI는 \"신기능 노출 수\"보다 \"완료율/이탈률/fallback 진입률\"로 본다.

---

## 마치며

Immediate UI는 멋진 데모 기능이 아니라, 로그인 비용을 줄이는 실무 기능이다.  
다만 전제(버전, 사전 등록, WebView 현실)를 모르면 기능을 붙여도 성과가 잘 안 나온다.

**한 줄 결:** *지금은 비밀번호를 더 잘 받는 시대가 아니라, 비밀번호를 덜 쓰게 만드는 인증을 설계하는 시대다.*

---

## 참고

- [Streamlined sign-in: Immediate UI mode (ko)](https://developer.chrome.com/blog/webauthn-immediate-ui?hl=ko)
- [Use Immediate UI mode in passkey authentication (ko)](https://developer.chrome.com/docs/identity/immediate-ui-mode?hl=ko)
- [Use passkeys for passwordless sign-in](https://developer.chrome.com/docs/identity/passkeys)
- [Microsoft 계정 passwordless 안내 (ko)](https://support.microsoft.com/ko-kr/accounts-billing/security/how-to-go-passwordless-with-your-microsoft-account)
