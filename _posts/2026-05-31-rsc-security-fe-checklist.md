---
layout: post
title: "RSC 보안 사태 이후, FE가 먼저 볼 것 — CVE·패치·영향 범위"
date: 2026-06-01 12:26:00 +0900
tags: [react, rsc, security, nextjs, frontend, cve]
learn_source: /Users/musinsa/docs/learning-backlog/items/2026-05-27-react-monthly-watch.md
---

> RSC 보안 이슈는 “리액트가 싫어서”가 아니다.  
> **서버 경계가 있는 스택**에서는 FE도 **영향 범위·패치·거버넌스**를 같이 봐야 한다.

[GeekNews #29900](https://news.hada.io/topic?id=29900)은 React 비판 글을 모아 둔 큐레이션이다. “요즘 리액트 좋아하는 사람 있나” 같은 여론도 있지만, **이 글의 목적은 비판을 보태는 데 있지 않다.**  
[12월 RCE 공지](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)와 [12월 후속 DoS·유출](https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components)을 바탕으로, **월요일 아침에 바로 꺼내 볼 점검 목록**으로 정리해 본다.

## TL;DR

- **CVE-2025-55182 (CVSS 10.0):** RSC·Server Function 경로의 **인증 없는 RCE**. 패치: React 19.0.1 / 19.1.2 / 19.2.1 이상, **Next는 프레임워크별 패치 버전**을 따른다 (14.x → `14.2.35` 등).
- **12/11 추가:** DoS **CVE-2025-55184·67779** (7.5), 소스 유출 **CVE-2025-55183** (5.3). “한 번 패치하면 끝”이 아니었고, **1/26 CVE-2026-23864** DoS까지 이어졌다.
- **Server Function을 안 써도** RSC를 **지원만 하면** 영향 받을 수 있다 — 공식 문구 기준.
- FE 할 일: **의존성·Next 버전·Server Action/RSC 라우트 목록**을 확인해 보안/플랫폼과 패치 계획을 잡기. **프레임워크 교체 선언이 이 글의 결론은 아니다.**

---

## 타임라인 (공식 포스트 기준)

| 시점 | 내용 |
|------|------|
| 2025-12-03 | RCE **CVE-2025-55182** 공개, 긴급 패치 |
| 2025-12-11 | 패치 검증 과정에서 **DoS·소스코드 유출** 추가 공개 |
| 2026-01-26 | DoS **CVE-2026-23864** 추가 (공식 follow-up) |

연말 한 주 사이에 **RCE → DoS → 유출 → 재패치**가 이어지면서, “프론트 CVE”를 넘어서 **서버 가용성·코드 노출**까지 FE 일정 안으로 들어왔다.

---

## 본론 1: RCE — 왜 “프론트”가 서버를 걱정하게 됐나

React Server Functions는 클라이언트 요청을 서버 함수 호출로 바꾼다.  
**역직렬화(deserialization)** 단계의 결함으로, 악의적 HTTP 요청이 Server Function 엔드포인트에서 **RCE**로 이어질 수 있었다 ([공식 12/03](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)).

여기서 핵심 문장은 이것이다.

> Server Function 엔드포인트를 **직접 구현하지 않아도**, 앱이 **React Server Components를 지원하면** 취약할 수 있다.

즉 “우리는 Server Action 안 쓴다”만으로는 끝나지 않을 수 있다. 먼저 **프레임워크·번들러가 RSC 스택을 포함하는지**를 확인해야 한다. Next, react-router(RSC), waku, `@vitejs/plugin-rsc` 등이 공식 affected 목록에 있다.

**Flight 프로토콜**은 혁신 설명용 키워드로 충분하다. exploit 레시피·PoC 재현은 이 글 범위 밖이다.  
핵심은 **서버 경계가 넓어질수록, FE도 배포 단위·라우트·의존성 트리를 모른 채 “보안은 백엔드 일”이라고만 둘 수 없어진다**는 점이다.

---

## 본론 2: 패치 일주일 뒤 — DoS와 소스코드 유출

[12/11 공지](https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components)는 **이전 패치만으로는 충분하지 않았다**는 메시지에 가깝다.

### DoS (High, CVSS 7.5)

- **CVE-2025-55184**, **CVE-2025-67779** — 역직렬화·처리 과정에서 **서버 멈춤·CPU 고갈** 가능.
- 당일 패치 뒤에도 **누락 케이스**가 확인되어 **67779**가 추가됐다. “패치 버전이면 안전하다”는 가정을 **검증 없이 두면 위험하다.**

### 소스코드 유출 (Medium, CVSS 5.3)

- **CVE-2025-55183** — 특정 Server Function 조건에서 **함수 구현(소스)이 응답에 실릴** 수 있음.
- 공식: **하드코딩된 시크릿**이 위험. `process.env.*` 런타임 값은 해당 범위 밖이라고 명시.

RCE만큼 자극적으로 보이지는 않아도, **배포 파이프라인·장애·비즈니스 로직 노출**은 FE 운영 범주다.

### 2026-01 follow-up

공식 12/03 글 업데이트에 **CVE-2026-23864** DoS가 추가됐다. **보안 릴리즈는 한 번 읽고 끝나는 이벤트가 아니다.** 월간 관측 루틴에 넣어둘 이유가 충분하다.

---

## 본론 3: 여론(GN)과 기술 공지 사이

[GN #29900](https://news.hada.io/topic?id=29900)은 성능·하이드레이션·RSC·Next 거버넌스 비판을 모은 **큐레이션**이다.  
“React가 기본값이 되며 선택이 관성으로 굳었다”는 지적과, “그래도 생태계가 살아남았다”는 반론이 같이 있다.

이 글에서 가져올 것은 **감정선 한 가지**다.

- **도구 선택은 문제 정의보다 앞서면 안 된다** ([problem-first](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)와 동형).
- **최근 CVE는 “맹신의 대가”를 설명하는 근거**로 쓰되, **결론을 “당장 다른 스택으로”로 옮기지는 않는다.**

운영 중인 Next/React 스택에서는 **패치·범위·거버넌스** 점검이 먼저고, 스택 교체는 **별도 의사결정**으로 다루는 편이 맞다.

---

## FE 실무 체크리스트

팀과 레포 상황마다 다르다. 아래는 바로 써볼 수 있는 **질문 목록**이다.

### 1) 우리 앱이 해당인가

- [ ] `react-server-dom-webpack` / `parcel` / `turbopack` 등 **RSC 패키지**가 lockfile·번들에 있는가
- [ ] **Next App Router** (또는 RSC 지원 프레임워크)를 쓰는 앱이 있는가
- [ ] **Server Actions**·`"use server"`·Server Component만 있는 라우트 — **목록 1페이지**로 적을 수 있는가

**React 18.x만 쓰고 RSC 스택이 없다**고 판단되면, 공식 기준에서 영향은 상대적으로 작을 수 있다. 다만 **Next 버전은 프레임워크 권고 라인을 따르는 편이 안전**하다.

### 2) 패치

- [ ] [React 12/03 업데이트 지침](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)의 **Next 패치 라인** 확인 (예: 14.x → **14.2.35**)
- [ ] monorepo면 **앱별 `package.json` Next 버전**이 한 줄로 맞는지
- [ ] 12/11·1/26 follow-up 이후 **추가 bump** 필요 여부 — 보안 채널·릴리즈 노트 재확인

### 3) 코드·운영 습관

- [ ] Server Function·Action 코드에 **하드코딩 시크릿** 없는지 (유출 CVE 맥락)
- [ ] CVE 대응 PR에 **영향 라우트·스모크 테스트** 한 줄이라도 있는지
- [ ] “패치 완료” 선언 전 **스테이징에서 Server 경로** 한 번 호출

### 4) 거버넌스 (짧게)

- [ ] 프레임워크·호스팅 **보안 공지 구독** (React blog, Next blog, Vercel security)
- [ ] FE 리드가 **의존성 bump 티켓** 소유자를 알고 있는지

---

## 우리 맥락에서 (예시 프레임, 일반화)

| 질문 | 왜 보는가 |
|------|-----------|
| App Router 앱이 있는가 | RSC 인프라·Next 패치 라인 |
| SSR/동적 렌더를 쓰지 않아도 되는가 | 정책상 Static 위주면 **노출 surface**가 다를 수 있음 — 그래도 **프레임워크 CVE는 별도** |
| Pages Router만인 앱 | RSC 스택 미사용 가능성 ↑ — **lockfile으로 확인** |

특정 회사나 앱 이름은 빼고, **레포를 열었을 때 바로 던질 질문**만 남겼다.

---

## 마치며

RSC 보안 사태가 곧바로 **“리액트를 그만 쓰자”**는 결론으로 이어지지는 않는다.  
다만 **“우리는 해당 없음”**이라는 말도 **lockfile·라우트·Next 버전 확인 없이** 하기는 어렵다.

개발할 때 손에 들고 볼 목록은 대략 다음과 같다.

1. **RSC/Server Function 지원 여부**  
2. **프레임워크·React 패치 버전** (공식 지침 기준)  
3. **12/03 이후 follow-up CVE**까지 포함한 재점검  
4. **도구 교체는** 문제 정의·제약이 정리된 뒤 — 맹신이 아니라 **선택**의 문제

더 깊게는 아래 공식 글을 순서대로 보면 된다.

- [Critical Security Vulnerability in React Server Components (2025-12-03)](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
- [Denial of Service and Source Code Exposure (2025-12-11)](https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components)
- [GeekNews #29900 — React 비판 큐레이션 (배경)](https://news.hada.io/topic?id=29900)
