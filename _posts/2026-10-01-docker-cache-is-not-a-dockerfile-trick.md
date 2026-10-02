---
layout: post
title: "Docker 캐시는 Dockerfile을 잘게 나누는 기술이 아니다"
date: 2026-10-02 11:09:00 +0900
excerpt: "프론트엔드 컨테이너 빌드에서 레이어를 많이 만드는 것보다 중요한 것은, 비싼 단계의 입력과 캐시가 실제로 다음 CI runner까지 살아 있는지를 아는 일이다. Docker 레이어, 패키지 store, 태스크 결과 캐시를 구분해야 어디에 투자할지 결정할 수 있다."
tags: [docker, buildkit, ci, frontend, cache, devops]
---

> 특정 제품이나 CI 설정이 아니라, 프론트엔드 컨테이너 빌드의 캐시를 판단하는 기준만 정리했다.

프론트엔드 CI가 느릴 때 Dockerfile부터 펼쳐 보는 일이 많다. `COPY`를 나누고, multi-stage build를 추가하고, 레이어를 더 만들면 빨라질 것처럼 보인다.

그런데 Docker 캐시는 레이어 개수를 보상하지 않는다. **같은 입력으로 비싼 작업을 다시 하지 않게 만들었는지**를 보상한다.

그리고 CI에서 그 캐시가 다음 runner까지 전달되지 않는다면, 잘 나눈 Dockerfile은 로컬에서는 빨라도 매번 새로 시작하는 CI에서는 거의 효과가 없을 수 있다.

이 글은 CI 시간을 층별로 나눠 보는 두 질문에서 이어진다. 최신 조합 검증의 책임은 [머지큐를 도입하기 전에 main부터 정의해야 했다](https://changbaebang.github.io/2026-10-01-merge-queue-needs-a-main-policy/)에서, 의존성 설치와 태스크 캐시의 구분은 [pnpm인가 Yarn인가보다 먼저 물을 것](https://changbaebang.github.io/2026-10-01-pnpm-or-yarn-is-not-the-first-ci-question/)에서 먼저 정리했다.

## 한 줄 요약

- Docker 레이어는 많이 나누는 것이 아니라, 자주 바뀌지 않는 입력과 비싼 작업을 앞에 두기 위해 나눈다.
- 프론트엔드에서는 대개 의존성 설치와 build를 분리하는 정도가 출발점으로 충분하다.
- ephemeral runner 환경에서는 Docker의 로컬 레이어 캐시만으로 부족할 수 있다. BuildKit 외부 캐시의 복원·저장 시간까지 측정해야 한다.
- Docker 레이어, 패키지 store, 태스크 결과 캐시는 서로 다른 것을 재사용한다.

## 레이어는 앞에서 깨지면 뒤도 함께 깨진다

Docker는 Dockerfile의 명령을 순서대로 실행하고, 각 명령의 결과를 레이어로 쌓는다. 어떤 레이어의 명령이나 그 레이어가 `COPY`한 입력이 바뀌면, 그 뒤 레이어는 다시 실행된다. [Docker build cache](https://docs.docker.com/build/cache)

그래서 다음 Dockerfile은 소스 한 줄을 바꿔도 의존성 설치부터 다시 실행할 가능성이 높다.

```dockerfile
FROM node:22-alpine
WORKDIR /app

COPY . .
RUN corepack enable && pnpm install --frozen-lockfile
RUN pnpm build
```

소스 코드와 lockfile이 하나의 `COPY . .`에 함께 들어가 있기 때문이다. 소스가 바뀌면 `COPY` 레이어가 달라지고, 그 아래의 install과 build 캐시도 함께 무효화된다.

가장 기본적인 분리는 이렇다.

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app

# 의존성의 입력은 코드보다 덜 바뀐다.
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# 소스 변경은 install 레이어를 무효화하지 않는다.
COPY . .
RUN pnpm build
```

위 코드는 단일 애플리케이션의 구조만 보여 주는 예시다. 모노레포에서는 root manifest·lockfile뿐 아니라 build 대상 workspace와 그 의존 workspace의 manifest도 install 단계의 입력으로 포함해야 한다. 필요 이상의 소스를 먼저 복사하지 않되, 필요한 workspace manifest를 빼서 설치가 깨지는 일도 피해야 한다.

Docker 공식 문서도 자주 바뀌지 않는 명령을 앞에, 자주 바뀌는 입력을 뒤에 두어 불필요한 cache invalidation을 피하라고 권한다. [Docker cache optimization](https://docs.docker.com/build/cache/optimize/)

## 프론트엔드 Dockerfile은 어디까지 나누면 되는가

레이어를 더 많이 만들수록 빨라지는 것은 아니다. 레이어 경계에는 관리 비용과 오해할 여지가 생긴다.

예를 들어 프론트엔드 소스 자체는 자주 바뀌고, build는 어차피 해당 변경을 반영해야 한다. 소스를 여러 디렉터리로 잘게 복사해 레이어를 만들더라도 build가 전체 소스에 의존한다면 마지막 단계는 다시 실행된다. 이때 레이어 분할은 Dockerfile만 복잡하게 할 수 있다.

대개 다음 네 단계부터 확인하면 충분하다.

1. **base**: Node와 OS 패키지처럼 드물게 바뀌는 기반 이미지
2. **dependencies**: manifest와 lockfile을 입력으로 하는 의존성 설치
3. **build**: 애플리케이션 소스와 빌드 산출물
4. **runtime**: 실행에 필요한 정적 파일이나 서버 산출물만 담은 최종 이미지

multi-stage build는 build cache를 마술처럼 빠르게 만드는 기능이 아니다. runtime image에 개발 의존성과 빌드 도구를 남기지 않아 이미지 크기와 공격 표면을 줄이는 구조다. 캐시 효율은 각 단계의 입력과 캐시 전달 방식에 달려 있다.

## CI에서는 “캐시가 있나”보다 “다음 runner가 받나”가 중요하다

로컬에서는 이전 빌드의 레이어가 Docker daemon에 남아 있다. 같은 머신에서 다시 빌드하면 dependency 단계가 즉시 cache hit가 날 수 있다.

하지만 CI runner가 매 작업마다 새로 만들어지고, 이전 builder의 상태를 공유하지 않는다면 이야기가 달라진다. Dockerfile을 잘 나눠도 다음 runner는 캐시를 찾을 곳이 없다.

BuildKit은 이 문제를 위해 캐시를 registry, 로컬 디렉터리, CI 제공 backend 같은 외부 위치에 export하고 이후 빌드에서 import할 수 있게 한다. Docker 문서도 runner 상태가 유지되지 않는 CI/CD 환경에서는 외부 캐시가 사실상 중요하다고 설명한다. [Docker cache backends](https://docs.docker.com/build/cache/backends/)

여기서도 “외부 캐시를 켜자”로 끝나면 안 된다. 다음을 측정해야 한다.

| 질문 | 확인할 값 |
| --- | --- |
| 레이어가 실제로 재사용되는가? | 단계별 cache hit / miss |
| 캐시 전달이 이득인가? | export·import 시간과 절감된 build 시간 |
| 캐시가 너무 커지지 않는가? | 저장소 용량, 네트워크 전송량, GC 정책 |
| 접근 경계가 안전한가? | cache write 권한, secret 처리, 신뢰하지 않는 job 격리 |
| runner가 달라져도 재사용되는가? | CPU 아키텍처, builder driver, image manifest 호환성 |

캐시를 저장하고 복원하는 데 90초가 걸려 build 40초를 아끼는 구조라면, cache hit 자체는 성공해도 전체 CI는 빨라지지 않는다.

## 캐시 네 종류를 섞지 말 것

프론트엔드 CI에서 서로 이름이 비슷한 캐시가 겹친다.

| 캐시 | 재사용하는 것 | 대표적인 질문 |
| --- | --- | --- |
| 패키지 store | 내려받은 의존성 파일 | install 단계의 다운로드를 줄였는가? |
| Docker / BuildKit 레이어 | Dockerfile 명령의 결과 | image build 단계를 줄였는가? |
| 태스크 캐시 | build·typecheck·test 산출물 | 같은 입력의 task를 다시 실행하지 않는가? |
| 아티팩트 캐시 | 배포·테스트에 넘길 결과물 | 이후 job이 결과를 다시 만들지 않는가? |

같은 `build`라는 단어가 들어가도 재사용 단위는 다르다. Docker 레이어 cache hit가 났다고 태스크 cache가 맞았다는 뜻은 아니고, 태스크 결과를 재사용한다고 Docker image를 다시 만들 필요가 없다는 뜻도 아니다.

따라서 개선 순서도 달라진다.

- 의존성 다운로드가 병목이면 패키지 store와 metadata cache를 본다.
- 컨테이너 조립이 병목이면 Dockerfile 입력 순서와 BuildKit 외부 캐시를 본다.
- build·typecheck·test 계산이 병목이면 태스크 캐시와 변경 범위 기반 실행을 본다.
- runner 대기가 병목이면 캐시가 아니라 실행 용량과 큐 정책을 본다.

## 레이어를 늘리기 전에 볼 체크리스트

Dockerfile을 고치기 전에 다음 정도는 확인할 수 있어야 한다.

- [ ] CI가 매번 새 runner인지, builder cache가 유지되는지 안다.
- [ ] dependency install과 application build 시간을 따로 기록한다.
- [ ] `COPY`하는 파일 범위가 불필요하게 넓지 않다. `.dockerignore`도 함께 점검한다.
- [ ] 외부 캐시를 쓴다면 import·export 시간과 hit rate를 같이 본다.
- [ ] secret은 `COPY`나 `ARG`가 아니라 build secret으로 전달한다.
- [ ] 캐시가 실패해도 빌드 결과의 정확성이 아니라 속도만 영향을 받는 구조인지 확인한다.

마지막 항목은 특히 중요하다. 캐시는 없어져도 다시 빌드하면 되는 성능 최적화여야 한다. 캐시 유무에 따라 산출물의 정확성이 달라지면, 그것은 캐시 설계가 아니라 빌드 입력 관리 문제다.

## 마치며

프론트엔드 컨테이너 빌드에서 레이어를 몇 개로 나눌지에는 정답이 없다. install이 거의 즉시 끝나는 프로젝트에서 레이어를 복잡하게 나누는 것은 이득이 작다. 반대로 의존성 설치와 이미지 빌드가 길고, 외부 캐시가 안정적으로 전달되는 환경이라면 작은 Dockerfile 구조 변경도 반복 비용을 크게 줄일 수 있다.

중요한 것은 레이어 수가 아니라, 비싼 작업의 입력이 무엇인지와 다음 runner가 그 결과를 실제로 재사용하는지다.

**Docker 캐시는 Dockerfile 꾸미기가 아니라, 변경 빈도와 실행 비용과 캐시 전달 경로를 함께 설계하는 일이다.**
