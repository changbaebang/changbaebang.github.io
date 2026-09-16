---
layout: post
title: "CLI 세 개를 npm에 올리기 전에 확인한 것들"
date: 2026-09-16 08:30:00 +0900
categories: [Engineering, Tooling]
tags: [npm, Supply Chain Security, CLI, Tailwind, Next.js, React, pnpm]
excerpt: "작은 CLI를 만들어 npm에 올리는 일은 코드 작성보다 패키징, 검증, 배포 경로를 함께 설계하는 일에 가까워졌다. 2026년의 npm 배포는 토큰을 줄이고 OIDC로 옮기며, 받는 쪽은 냉각 기간과 설치 제한으로 새 버전을 천천히 받아야 한다."
---

> 개인 저장소 세 개를 만든 이야기다. 도구를 실제 프로젝트에 돌려 본 부분은
> 공개 저장소에 남겨도 되는 결론만 적었다.

작은 CLI를 세 개 만들었다. 최근 작업에서 실제로 마주친 문제를 반복해서 확인하다가, 매번 코드와 맥락을 AI에게 다시 읽히는 대신 명확한 판정은 도구로 빼는 편이 낫겠다고 생각했다. 제품 안에 AI가 들어간 도구는 아니고, 프론트엔드 프로젝트에서 자주 놓치는 문제를 정적으로 찾는 Node.js 기반 패키지다.

| 저장소 | 하는 일 |
| --- | --- |
| [tw-ghost](https://github.com/changbaebang/tw-ghost) | Tailwind 설정에서 CSS를 만들지 않는 클래스를 찾는다 |
| [unprovided](https://github.com/changbaebang/unprovided) | Provider 없이 마운트되는 React Context 소비 훅을 페이지 단위로 찾는다 |
| [ssr-leak](https://github.com/changbaebang/ssr-leak) | Next.js SSR에서 요청 사이에 공유될 수 있는 모듈 스코프 상태를 찾는다 |

처음에는 "문제 하나당 CLI 하나" 정도의 가벼운 실험이었다. 코드를 AI로 계속 돌려볼 수도 있다. 하지만 같은 종류의 검사를 반복한다면, 매번 토큰을 쓰는 대화보다 명령 하나로 재현되는 도구가 더 낫다. 그런데 세 개를 npm 패키지로 만들고 배포 준비까지 해 보니, 정작 오래 걸린 것은 핵심 알고리즘보다 패키징과 검증이었다. 어떤 파일이 tarball에 들어가는지, `bin`이 실제로 실행되는지, ESM과 CJS 소비자가 타입을 읽는지, 배포 워크플로에 남은 토큰은 없는지, 받는 쪽은 새 버전을 얼마나 빨리 설치할지 같은 문제들이다.

2026년에 npm 패키지를 올린다는 것은 이제 `npm publish` 한 번으로 끝나는 일이 아니다. 올리는 쪽은 토큰을 줄이고 provenance를 남기는 방향으로 가야 하고, 받는 쪽은 갓 올라온 패키지를 바로 믿지 않아야 한다. 세 CLI를 만들며 정리한 체크리스트를 이 글에 남긴다.

## 세 CLI가 잡으려던 문제

셋 다 실제 프로젝트에서 사람이 반복해서 확인하던 문제에서 출발했다. 린터가 일반적으로 잘 잡아 주지 못하거나, 잡아도 프로젝트의 구조를 알아야 판단할 수 있는 것들이다.

**tw-ghost**는 Tailwind 클래스가 실제 CSS를 만드는지 본다. Tailwind v3에서 테마 스케일을 프로젝트 토큰으로 통째로 교체하면 `text-sm`, `z-10` 같은 표준 클래스가 겉보기에는 멀쩡하지만 CSS를 만들지 않을 수 있다. 브라우저는 모르는 클래스를 조용히 무시하므로 화면만 미묘하게 틀어진다. tw-ghost는 프로젝트 설정과 기본 Tailwind 설정으로 CSS를 각각 생성한 뒤, 기본 Tailwind에서는 나오지만 현재 프로젝트에서는 나오지 않는 클래스만 골라낸다.

**unprovided**는 React Context 소비자가 Provider 없이 렌더될 수 있는 페이지를 찾는다. `createContext('en')`처럼 기본값 자체가 정책인 경우도 있으므로, 모든 Provider 누락을 오류로 보면 안 된다. App Router의 `layout` 체인과 페이지 import 그래프를 이용하되, 트리 안에서의 정확한 렌더 순서까지는 추론하지 않는 선에서 경계를 잡았다.

**ssr-leak**는 서버 요청 사이에 공유될 수 있는 모듈 스코프 상태를 찾는다. 서버에서 모듈은 한 번 로드되고 여러 요청이 공유한다. `axios.defaults.headers`에 요청별 토큰을 쓰거나, 모듈 레벨 변수에 사용자 정보를 넣으면 한 요청의 값이 다른 요청에 보일 수 있다. 이 문제는 로컬에서 혼자 누를 때는 잘 안 보이고, 동시 요청이 생겼을 때 드러난다.

도구의 규칙은 다르지만, npm 패키지로 만드는 과정에서 공통으로 필요한 일은 비슷했다.

## 패키지로 만들 때 먼저 본 것

코드가 테스트를 통과해도 패키지가 제대로 동작한다는 뜻은 아니다. 저장소 안에서 `pnpm test`가 초록색인 것과, 사용자가 `npm install`한 뒤 `node_modules/.bin/<name>`을 실행할 수 있는 것은 다른 문제다.

먼저 `package.json`을 배포 단위로 보았다.

- `name`, `version`, `description`, `license`, `repository`, `homepage`, `bugs`, `keywords`, `engines`를 채운다.
- `repository.url`은 npm trusted publishing 설정과 대조되므로 실제 GitHub 저장소와 정확히 맞춘다.
- `files`는 화이트리스트로 둔다. `dist`, README, LICENSE, CHANGELOG 정도만 넣고 테스트와 픽스처는 빼는 쪽이 안전하다.
- `bin`은 실제 빌드 산출물을 가리키고, 파일 첫 줄에는 shebang이 있어야 한다.
- `exports`는 ESM과 CJS 조건을 분리한다. `import`에는 `.d.ts`, `require`에는 `.d.cts` 타입을 연결한다.
- `publishConfig.registry`는 `https://registry.npmjs.org/`로 고정한다. 로컬 `.npmrc`가 프록시 레지스트리를 가리키는 환경에서도 공개 npm으로 배포할 패키지는 목적지를 명시하는 편이 낫다.

그다음은 tarball이다. `npm pack --dry-run`으로 파일 목록을 보는 것만으로는 부족했다. 실제 tarball을 빈 프로젝트에 설치하고, 다음을 확인해야 한다.

```bash
npm install ../package-name-0.1.0.tgz
node_modules/.bin/package-name --help
node -e "import('package-name').then(console.log)"
node -e "console.log(require('package-name'))"
```

이 단계에서 shebang 누락, `bin` 경로 오타, `exports` 조건 누락, CJS 타입 누락 같은 문제가 나온다. 런타임은 되는데 타입만 깨지는 경우도 있어서 `publint`나 `@arethetypeswrong/cli` 같은 도구까지 함께 돌리는 편이 좋다.

## 검사 도구의 기본값

세 CLI 모두 첫 구현에서 같은 실수를 했다. 입력 glob이 아무 파일도 찾지 못하면 "0개 검사, 문제 없음"으로 끝났다. CI에서 경로 하나를 잘못 쓰면 영원히 초록불이 되는 설정이다.

그래서 기본값을 바꿨다. 검사 대상이 비어 있으면 실패하고, 정말 빈 입력이 정상인 곳에서만 `--allow-empty`를 주게 했다. 검사 도구는 사용자가 잘못 실행했을 때 조용히 성공하면 안 된다. 오탐도 문제지만, 아무것도 보지 않고 성공하는 것은 더 위험하다.

공개 저장소로 나가기 전에 식별자도 확인했다. 실제 프로젝트에 도구를 돌려 보는 과정에서는 픽스처나 README 예시에 조직 이름, 내부 레지스트리 주소, 티켓 키, 절대 경로가 섞이기 쉽다. 단순하게라도 `grep -riE '<식별자 목록>|/Users/'` 같은 검사를 마무리 조건에 넣어 두면 공개하면 안 되는 흔적을 줄일 수 있다.

README를 두 언어로 둘 때는 구조 대조도 필요했다. 영어 README만 고치고 한국어 README의 옵션 표가 뒤처지는 일이 생긴다. 마지막에는 섹션 헤더 수, 옵션 표 행 수, `--help`에 있는 옵션이 양쪽 README에 모두 등장하는지를 스크립트로 비교했다. 번역 품질까지 보지는 못해도 빠진 옵션은 잡을 수 있다.

## npm 배포는 토큰을 줄이는 쪽으로 간다

예전 npm 배포 글을 따라 하면 `NPM_TOKEN`을 GitHub Actions secret에 넣고 `npm publish`를 실행하는 흐름이 자주 나온다. 지금 새 패키지를 만든다면 그 방식을 기본값으로 두지 않는 편이 낫다.

2025년 하반기 이후 npm은 토큰 정책을 크게 바꿨다. 클래식 토큰은 폐지되었고, granular 토큰은 수명과 2FA 조건이 더 엄격해졌다. npm은 GitHub Actions나 GitLab CI/CD에서 OIDC로 신원을 증명하고 배포하는 trusted publishing을 정식 지원한다. 이 방식에서는 장기 토큰을 CI에 저장하지 않는다. 배포 시점의 워크플로 신원을 npm이 확인하고, provenance 증명도 기본으로 붙는다.

GitHub Actions 기준으로 필요한 것은 대략 이렇다.

1. npmjs.com에서 패키지의 trusted publisher에 저장소와 workflow 파일명을 등록한다.
2. 워크플로에 `permissions: id-token: write`를 준다.
3. npm CLI 11.5.1 이상과 Node.js 22.14.0 이상을 쓴다.
4. 배포 단계에서 `npm publish --access public`을 실행한다. trusted publishing에서는 provenance가 기본으로 생성된다.

첫 배포에서는 조심할 점이 있다. 패키지가 아직 npm에 존재하지 않으면 패키지 설정 화면에서 trusted publisher를 바로 등록할 수 없는 경우가 있다. 이때는 한 번만 제한된 granular token으로 첫 버전을 올리고, 곧바로 trusted publisher를 등록한 뒤 토큰을 지우는 순서가 현실적이다. 그래서 첫 release workflow를 작성할 때도 "토큰으로 계속 배포한다"가 아니라 "첫 배포 이후 OIDC로 옮긴다"를 기준으로 두는 편이 안전하다. 새 패키지를 여럿 관리한다면 npm CLI의 trusted publishing 관련 명령으로 설정을 묶어서 관리하는 방식도 볼 만하다.

최근 npm은 staged publishing도 지원한다. CI가 바로 공개 버전을 밀어 넣는 대신, tarball을 staging queue에 올리고 사람이 승인한 뒤 공개하는 방식이다. 모든 작은 패키지에 꼭 필요한 것은 아니지만, 배포 권한을 가진 워크플로가 곧바로 공개 릴리스를 만들지 못하게 하는 선택지가 생긴 것은 중요하다.

## 배포 트리거는 단순하게

세 저장소의 배포 트리거는 태그로 둘 생각이다.

```bash
git tag v0.1.0
git push origin v0.1.0
```

main에 머지된다고 바로 배포되지 않고, 버전 태그가 올라갈 때만 release workflow가 돈다. 이때 `package.json`의 `version`과 태그가 맞는지 확인하고, `prepublishOnly`에서 빌드와 테스트를 한 번 더 실행한다.

로컬에서 `npm publish`를 직접 치는 흐름은 되도록 피한다. 로컬 인증 상태와 `.npmrc`가 어떤지 매번 확인하기 어렵기 때문이다. 공개 배포는 CI에서 하고, 첫 배포 이후의 CI 인증은 OIDC로 옮기는 것을 목표로 둔다. 배포 대상 registry는 `publishConfig.registry`로 고정한다. 단순한 규칙이지만 실수를 줄이는 데 효과가 크다.

## 받는 쪽은 새 버전을 천천히 받아야 한다

올리는 쪽이 조심해도 충분하지 않다. 내가 의존하는 패키지 하나가 감염되면 내 CI가 그것을 설치한다. 최근 npm 보안 사고들이 남긴 교훈은 단순하다. 새 버전이라고 바로 받지 말고, 설치 시점의 실행 권한을 줄여야 한다.

가장 실용적인 첫 번째 층은 냉각 기간이다. pnpm은 `minimumReleaseAge`로 새로 공개된 지 얼마 안 된 버전을 설치하지 않게 할 수 있고, pnpm 11부터는 기본값이 1440분, 즉 하루다. 감염된 버전은 대개 빠르게 보고되고 내려가므로 하루만 기다려도 상당수 사고를 피할 수 있다. 내부 패키지나 즉시 받아야 하는 패키지는 `minimumReleaseAgeExclude`로 예외를 둘 수 있다.

두 번째 층은 설치 스크립트 제한이다. 많은 공급망 공격은 `postinstall` 같은 lifecycle script에서 실행된다. pnpm은 허용한 패키지만 build script를 실행하게 할 수 있고, npm 환경에서는 `ignore-scripts=true`를 기본으로 둔 뒤 필요한 패키지만 별도로 처리하는 방식을 검토할 수 있다.

세 번째 층은 출처 제한이다. pnpm의 `blockExoticSubdeps`처럼 전이 의존성이 git이나 tarball URL을 끌고 오지 못하게 막는 설정이 있다. npm도 2026년에 git, file, remote, directory 출처를 제한하는 `--allow-*` 계열 옵션을 추가했다. 특히 git 의존성은 install script를 꺼도 다른 실행 경로가 생길 수 있으므로, 필요하지 않다면 닫아 두는 편이 낫다.

네 번째 층은 프록시 레지스트리와 lockfile이다. 공개 npm을 직접 보지 않고 Nexus, Artifactory, Verdaccio 같은 프록시를 거치면 캐시, 차단 목록, 승인 지연을 둘 수 있다. lockfile은 설치 결과를 고정하고 리뷰 가능한 변경으로 만든다. 새 의존성이 들어오는 일을 코드 변경처럼 다룰 수 있어야 한다.

JSR 같은 대체 레지스트리도 선택지다. TypeScript 소스를 중심으로 설계되어 있고 OIDC 배포가 자연스럽다. 다만 npm 생태계의 크기와 호환성을 한 번에 대체하기는 어렵다. 새 패키지를 npm과 JSR에 같이 올리는 정도가 현실적인 출발점이다.

정리하면 받는 쪽의 기본값은 이렇다.

- 갓 올라온 버전은 하루 정도 기다린다.
- 설치 스크립트는 필요한 패키지만 허용한다.
- git, tarball, file 같은 출처는 기본적으로 막는다.
- lockfile 변경을 리뷰한다.
- 가능하면 프록시 레지스트리를 통과시킨다.

## 작은 패키지도 배포 경로가 제품이다

CLI 세 개를 만들며 배운 것은 코드보다 주변 경로가 더 자주 깨진다는 점이었다. 알고리즘은 테스트로 어느 정도 잡히지만, 패키지 사용자는 `exports` 조건, `bin` 경로, tarball 파일 목록, README 옵션, 배포 인증 방식에서 더 빨리 막힌다.

AI에게 코드를 맡겨 보는 것은 여전히 유용하다. 다만 반복 가능한 문제라면 매번 대화로 싸우기보다, 애초에 싸울 일을 줄이는 도구로 만드는 쪽이 더 오래 간다. 손자병법의 "싸우지 않고 이기는 것"이라는 말처럼, 코드 리뷰에서 이기는 가장 좋은 방법은 리뷰어를 설득하는 긴 설명이 아니라 같은 실수를 다시 만들기 어렵게 하는 실행 가능한 검사일 때가 있다.

그래서 이제 작은 npm 패키지를 만들 때도 다음 질문을 먼저 하게 된다.

- tarball에 무엇이 들어가는가?
- 빈 프로젝트에 설치했을 때 바로 실행되는가?
- ESM, CJS, 타입 소비자가 모두 같은 패키지를 읽을 수 있는가?
- CI에 장기 토큰이 남아 있는가?
- 새 버전을 받는 쪽이 하루 정도 멈출 수 있는가?
- 설치 중 실행되는 코드는 누가 허용했는가?

npm에 올리는 일은 여전히 쉽다. 하지만 쉽게 올릴 수 있다는 것과 바로 믿어도 된다는 것은 다르다. 2026년에 작은 CLI를 공개한다면, 코드와 함께 배포 경로까지 같이 설계해야 한다.

도구를 어디까지 패키지로 만들고 어디부터 워크플로로 둘지는 예전에 [스킬, 에이전트, 서비스의 경계](https://changbaebang.github.io/2026-09-14-skills-agents-and-services-boundaries/)에서도 비슷하게 정리한 적이 있다.

## 읽을 거리

- [npm trusted publishing with OIDC is generally available](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/)
- [Trusted publishing for npm packages](https://docs.npmjs.com/trusted-publishers/)
- [npm classic tokens revoked, session-based auth and CLI token management now available](https://github.blog/changelog/2025-12-09-npm-classic-tokens-revoked-session-based-auth-and-cli-token-management-now-available/)
- [npm bulk trusted publishing config and script security now generally available](https://github.blog/changelog/2026-02-18-npm-bulk-trusted-publishing-config-and-script-security-now-generally-available/)
- [Staged publishing and new install-time controls for npm](https://github.blog/changelog/2026-05-22-staged-publishing-and-new-install-time-controls-for-npm)
- [pnpm -- Mitigating supply chain attacks](https://pnpm.io/supply-chain-security)
- [pnpm 10.16 Adds New Setting for Delayed Dependency Updates](https://socket.dev/blog/pnpm-10-16-adds-new-setting-for-delayed-dependency-updates)
