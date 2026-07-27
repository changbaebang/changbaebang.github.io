---
layout: post
title: "Next.js 보안 패치, 버전만 올리고 끝내지 않기"
date: 2026-07-27 14:21:52 +0900
categories: [Frontend, Security]
tags: [Next.js, Security, Server Actions, App Router]
---

2026년 7월 20일, Next.js 팀은 9개의 취약점을 수정한 보안 릴리스를 공개했다. 권장 버전은 Active LTS인 `16.2.11`과 Maintenance LTS인 `15.5.21`이다.

처음 공지를 보면 할 일은 간단해 보인다. `next` 버전을 올리고 배포하면 된다. 물론 업그레이드가 가장 먼저다. 하지만 버전만 바꾸면 이번에 무엇이 위험했는지, 우리 애플리케이션의 어느 경계가 영향을 받는지, 다음에는 무엇을 테스트해야 하는지가 남지 않는다.

이번 글에서는 [Next.js 공식 보안 공지](https://nextjs.org/blog/july-2026-security-release)의 4개 High, 5개 Medium 취약점을 기능 표면별로 다시 묶어 본다. 목표는 패치를 피할 근거를 찾는 것이 아니다. **먼저 패치하고, 노출된 표면을 증명하고, 같은 종류의 문제가 돌아오지 않도록 경계를 테스트하는 것**이다.

## 먼저 결론: 업그레이드와 영향 분석은 둘 다 필요하다

보안 공지를 읽을 때 흔히 두 극단으로 흐른다.

- CVE가 나왔으니 버전만 올리고 끝낸다.
- 해당 기능을 쓰지 않는 것 같으니 업그레이드를 미룬다.

둘 다 충분하지 않다. 지원되는 패치 버전으로의 업그레이드는 기본 조치다. 영향 분석은 업그레이드 여부를 결정하기 위해서가 아니라 배포 우선순위, 회귀 테스트 범위, 추가 방어 지점을 결정하기 위해 필요하다.

이번 릴리스의 9건은 다음 네 영역으로 나눠 보면 이해하기 쉽다.

| 기능 표면 | 관련 CVE | 확인할 질문 |
|---|---|---|
| Server Actions와 Server Functions | CVE-2026-64641, 64643, 64646, 64649 | 서버 함수가 외부 요청 경계라는 전제로 인증, 입력 크기, 호스트를 검증하는가? |
| Middleware, Proxy, rewrite, redirect | CVE-2026-64642, 64645 | 라우팅 계층 하나에만 권한 검사를 맡기거나 목적지 호스트를 입력으로 조립하는가? |
| Image Optimization | CVE-2026-64644 | 셀프 호스팅 환경에서 외부 SVG를 기본 이미지 로더로 최적화하는가? |
| 서버 `fetch` 캐시 | CVE-2026-64647, 64648 | 본문이 다른 요청의 응답이 같은 캐시 항목으로 취급될 수 있는가? |

## 1. Server Action은 내부 함수가 아니라 요청 경계다

이번 릴리스에서 가장 많은 취약점이 모인 곳은 App Router의 Server Actions와 Server Functions다.

- **CVE-2026-64641 (High)**: Server Action을 겨냥한 조작된 요청이 자원 고갈과 서비스 거부를 일으킬 수 있다.
- **CVE-2026-64646 (Medium)**: Edge Runtime의 Server Action에서 요청 페이로드 크기가 제한되지 않아 메모리를 과도하게 소비할 수 있다.
- **CVE-2026-64649 (High)**: 커스텀 서버에서 Server Action이 전달 또는 리다이렉트되는 과정에 공격자가 호스트 관련 헤더를 통제하면, 서버가 의도하지 않은 호스트로 요청할 수 있다.
- **CVE-2026-64643 (Medium)**: 인증된 페이지에서만 쓴다고 생각한 `use server` 또는 `use cache` 엔드포인트 식별자가 비인증 사용자에게 노출될 수 있다.

여기서 핵심은 “함수 식별자가 노출됐다”는 사실 자체보다, 식별자는 비밀이나 권한 수단이 될 수 없다는 점이다. [CVE-2026-64643 advisory](https://github.com/vercel/next.js/security/advisories/GHSA-955p-x3mx-jcvp)도 `use server`와 `use cache` 경계 안에서 직접 인증하라고 권고한다.

페이지 진입 전에 로그인 여부를 검사했더라도 Server Action은 별도로 호출될 수 있는 서버 경계다. 따라서 다음과 같은 구조가 필요하다.

```ts
'use server';

export async function updateProfile(input: UpdateProfileInput) {
  const session = await requireSession();
  await assertCanUpdateProfile(session.userId, input.profileId);
  const validated = updateProfileSchema.parse(input);

  return profileRepository.update(validated);
}
```

핵심은 UI가 이 함수를 어떻게 호출하는지가 아니라 함수가 호출된 뒤 스스로 인증, 인가, 입력 검증을 수행하는지다.

코드베이스에서는 먼저 후보를 찾는다.

```bash
rg -n "'use server'|\"use server\"|'use cache'|\"use cache\"" .
rg -n "export const runtime\s*=\s*['\"]edge['\"]" .
```

검색 결과가 곧 취약하다는 뜻은 아니다. 각 함수 안에서 인증과 객체 단위 인가가 수행되는지, 큰 본문을 받는 엔드포인트에 애플리케이션 또는 인프라 제한이 있는지 확인하는 출발점이다.

## 2. 라우팅 계층을 최종 보안 경계로 믿지 않는다

두 번째 묶음은 요청이 어디로 가는지를 결정하는 기능이다.

- **CVE-2026-64642 (High)**: App Router, Turbopack, `i18n.locales`에 로케일을 정확히 하나만 둔 조합에서 Middleware 또는 Proxy 검사를 우회할 수 있다.
- **CVE-2026-64645 (High)**: `rewrites()`나 `redirects()`의 외부 목적지 호스트를 요청 입력으로 조립하면, 예상한 접미사를 벗어난 호스트가 만들어질 수 있다. rewrite에서는 SSRF, redirect에서는 open redirect로 이어질 수 있다.

첫 번째 문제는 조건이 좁아 보인다. 그렇다고 Middleware에만 인증을 맡기는 설계가 안전해지는 것은 아니다. Middleware는 빠른 차단과 사용자 경험을 위한 첫 번째 문이 될 수 있지만, 데이터와 변경 작업을 지키는 마지막 문은 해당 Route Handler나 Server Action이어야 한다.

두 번째 문제는 URL을 문자열 결합으로 만들 때 생기는 전형적인 경계 오류다.

```ts
// 피해야 할 형태: 입력이 호스트 구성에 참여한다.
destination: `https://${requestValue}.example.com/:path*`
```

가능한 목적지를 명시적인 allowlist로 매핑하고, 사용자 입력은 호스트가 아닌 제한된 식별자로 취급하는 편이 안전하다.

다음 검색으로 설정과 경계의 후보를 좁힐 수 있다.

```bash
rg -n "middleware|proxy|rewrites|redirects" .
rg -n "i18n|locales|turbopack" next.config.* package.json .
```

이후에는 세 가지를 확인한다.

1. 인증과 인가가 Middleware 이후의 실제 처리 경계에서도 반복되는가?
2. rewrite와 redirect의 scheme, hostname, port가 고정되거나 allowlist로 제한되는가?
3. 요청 헤더와 query/path 값이 목적지 hostname을 구성하지 않는가?

## 3. 이미지 최적화도 서버가 수행하는 파싱 작업이다

**CVE-2026-64644 (Medium)**는 셀프 호스팅한 Next.js가 기본 이미지 로더로 원격 이미지를 최적화하도록 설정된 경우, 악성 SVG가 `/_next/image`에서 과도한 CPU를 쓰게 할 수 있는 문제다. 원격 이미지 최적화는 기본으로 활성화되는 기능은 아니지만, 실제 서비스에서는 흔히 설정한다.

확인할 곳은 `images.remotePatterns`, 이전 설정의 `images.domains`, 커스텀 로더와 배포 방식이다.

```bash
rg -n "remotePatterns|images\.domains|loader|/_next/image" .
```

여기서도 “SVG를 화면에 보여 주는가?”만 물으면 부족하다. 서버가 외부에서 받은 SVG를 가져와 파싱하거나 최적화하는 경로가 있는지, 허용한 원격 호스트에서 사용자가 SVG를 업로드할 수 있는지까지 연결해서 봐야 한다.

패치와 함께 원격 호스트 범위를 최소화하고, SVG가 필요 없다면 입력 단계에서 거부하는 정책을 검토할 수 있다. 셀프 호스팅 환경이라면 프록시의 요청 제한과 프로세스 자원 제한도 방어층이 된다.

## 4. `fetch`의 URL이 같아도 요청이 같다는 뜻은 아니다

마지막 두 건은 서버 측 `fetch` 응답 캐시가 요청 본문을 구분하는 방식과 관련된다.

- **CVE-2026-64648 (Medium)**: 특정 형태로 `Request`와 별도 init을 함께 전달한 서버 `fetch`에서 서로 다른 본문의 응답이 혼동될 수 있다.
- **CVE-2026-64647 (Medium)**: UTF-8이 아닌 charset의 잘못된 바이트 시퀀스를 포함한 본문에서, 같은 URL이지만 본문이 다른 요청의 캐시 응답이 섞일 수 있다.

후자는 단순한 성능 버그가 아니다. [CVE-2026-64647 advisory](https://github.com/vercel/next.js/security/advisories/GHSA-4633-3j49-mh5q)는 다른 POST 요청의 응답 본문이 반환되면서 기밀 데이터가 노출될 수 있다고 설명한다.

코드에서는 서버에서 본문과 함께 `fetch`를 호출하는 지점, 특히 `Request` 객체와 두 번째 init을 함께 사용하는 형태를 찾는다.

```bash
rg -n "fetch\s*\(\s*new Request|method\s*:\s*['\"](POST|PUT|PATCH|DELETE)" .
```

정적 검색만으로 Next.js 내부 캐시 동작까지 판정할 수는 없다. 사용자별 응답이나 민감 데이터를 반환하는 호출을 우선 분류하고, 본문만 다른 두 요청이 절대로 같은 응답을 받지 않는지 회귀 테스트를 추가하는 편이 낫다.

## 실제 대응 순서

이번 공지를 팀에서 처리한다면 다음 순서가 현실적이다.

### 1단계: 지원되는 패치 버전으로 올린다

공식 권장 버전은 `16.2.11` 또는 `15.5.21`이다. 먼저 현재 직접 의존성과 lockfile에 실제로 해석된 버전을 함께 확인한다.

```bash
rg -n '"next"\s*:' -g 'package.json' .
```

지원이 끝난 메이저 버전이라면 “우리 기능은 영향이 적다”로 머무르기보다 지원 브랜치로 이동하는 작업과 임시 완화책을 분리해 계획해야 한다.

### 2단계: 네 기능 표면으로 노출 후보를 만든다

`use server`, `use cache`, Edge Runtime, Middleware/Proxy, i18n, rewrite/redirect, 원격 이미지, 본문이 있는 서버 `fetch`, 커스텀 서버와 셀프 호스팅 여부를 검색한다.

이 목록은 취약점 판정표가 아니라 검토 대기열이다. 코드 설정뿐 아니라 빌드 방식, CDN과 프록시, 런타임 배포 설정도 함께 확인해야 한다.

### 3단계: 경계 중심의 회귀 테스트를 남긴다

- 인증하지 않은 사용자가 Server Action을 직접 호출해도 거부되는가?
- 권한이 없는 객체 ID로 호출하면 객체 단위 인가에서 차단되는가?
- 비정상적으로 큰 Action 입력이 애플리케이션이나 인프라 경계에서 제한되는가?
- rewrite와 redirect의 목적지가 사용자 입력 때문에 다른 호스트로 바뀔 수 없는가?
- 허용된 원격 호스트의 SVG가 이미지 최적화 서버 자원을 고갈시키지 않는가?
- URL은 같고 본문만 다른 서버 `fetch`가 서로의 응답을 받지 않는가?

### 4단계: 배포 뒤에도 확인한다

의존성 PR이 합쳐졌다는 사실보다 운영 산출물에 패치 버전이 들어갔는지가 중요하다. 배포 이미지의 lockfile 또는 SBOM, 런타임 버전 정보를 확인하고, 서버 오류율과 CPU·메모리 지표의 변화를 관찰한다.

## 이번 공지에서 배운 것

처음에는 9개 CVE를 각각 외워야 하나 싶었다. 기능 표면으로 묶어 보니 반복되는 원칙은 세 가지였다.

첫째, 프레임워크가 함수 호출처럼 보이게 추상화해도 Server Action은 네트워크로 닿을 수 있는 서버 경계다. 인증과 인가는 그 경계 안에 있어야 한다.

둘째, Middleware나 URL 설정은 편리한 중앙 제어점이지만 유일한 보안 경계가 되어서는 안 된다. 실제 데이터와 작업을 소유한 계층이 다시 검증해야 한다.

셋째, 캐시는 입력을 정확하게 식별할 때만 안전하다. URL이 같다는 이유로 본문과 사용자 맥락이 다른 요청을 같은 것으로 취급하면 성능 최적화가 데이터 노출 경로가 될 수 있다.

보안 릴리스 대응의 결과물은 버전 변경 한 줄에 그치지 않는다. 어떤 표면이 노출됐는지 설명할 수 있고, 그 설명이 코드 검색과 회귀 테스트로 남을 때 다음 공지를 더 빠르고 정확하게 처리할 수 있다.

이전의 RSC 보안 이슈를 프론트엔드 관점에서 정리한 글은 [RSC 보안 사태 이후, FE가 먼저 볼 것](https://changbaebang.github.io/2026-05-31-rsc-security-fe-checklist/)에서 이어서 볼 수 있다.

## 참고 자료

- [Next.js: July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release)
- [Next.js Security Advisories](https://github.com/vercel/next.js/security/advisories)
- [GHSA-955p-x3mx-jcvp: Unauthenticated disclosure of internal Server Function endpoints](https://github.com/vercel/next.js/security/advisories/GHSA-955p-x3mx-jcvp)
- [GHSA-4633-3j49-mh5q: Cache confusion with invalid UTF-8 byte sequences](https://github.com/vercel/next.js/security/advisories/GHSA-4633-3j49-mh5q)
