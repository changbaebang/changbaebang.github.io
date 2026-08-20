---
layout: post
title: "Agent Skill Garden에 프론트엔드 리뷰 팩을 추가했다"
date: 2026-08-20 15:25:00 +0900
excerpt: "PR의 현재 diff를 읽고 필요한 리뷰 관점만 선택하는 프론트엔드 리뷰 팩을 공개 저장소에 추가했다. 오케스트레이터와 React·TypeScript·Next.js·저장소 위생 리뷰가 어떻게 구성되는지 소개한다."
tags: [ai, agents, skills, code-review, pull-request, react, typescript, nextjs, workflow, open-source]
---

공개 저장소 [`Agent Skill Garden`](https://github.com/changbaebang/agent-skill-garden)에 라우팅 기반 프론트엔드 리뷰 팩을 추가했다. [PR #5](https://github.com/changbaebang/agent-skill-garden/pull/5)로 만들었고, 현재 `main`에 병합됐다.

하나의 긴 프롬프트에 모든 리뷰 규칙을 넣는 대신, PR의 현재 상태와 diff를 먼저 읽고 필요한 리뷰만 선택하는 구성이다.

## 구성

중심에는 `pull-request-review`가 있다. 이 스킬은 직접 모든 결함을 찾으려 하지 않고 다음 작업을 조율한다.

- PR의 base, head, 상태와 변경 파일 확인
- 저장소 지침과 기존 리뷰 스레드 확인
- diff에 맞는 전문 리뷰 선택
- 리뷰 결과와 심각도 통합
- 재리뷰 시 이전 head와 현재 head 비교
- 승인이나 수정 요청을 게시하기 전 권한 확인

실제 검토는 목적별 스킬이 나눠 맡는다.

| 스킬 | 역할 |
|---|---|
| `critical-review` | 배포를 막아야 할 장애·보안·데이터 결함 확인 |
| `side-effect-check` | 공유 상태·캐시·인증과 소비처 영향 추적 |
| `react-review` | 렌더·상태·Effect·Hook과 비동기 생명주기 확인 |
| `typescript-review` | `any`, 타입 단언, nullability와 외부 데이터 경계 확인 |
| `nextjs-review` | 라우트 의도, Server/Client 경계, redirect, hydration, cache 확인 |
| `hygiene-review` | 의존성, manifest, lockfile, export와 디버그 잔여물 확인 |

`critical-review`는 모든 PR에서 수행한다. 나머지는 파일 확장자보다 실제로 변경된 동작과 경계를 기준으로 선택한다.

예를 들어 공유 Provider가 바뀌면 `react-review`, `typescript-review`, `side-effect-check`를 함께 적용한다. 의존성과 lockfile만 바뀌었다면 `critical-review`와 `hygiene-review`만 적용한다.

## 공통 리뷰 규칙

리뷰 종류가 달라도 지적을 만드는 기준은 같다.

`any`, `useEffect`, `use client`, 큰 lockfile diff는 그 자체로 결함이 아니다. 확인을 시작할 위치를 알려주는 신호다. 구체적인 실행 조건과 영향을 받는 경로, 실제 결과, 현재 변경이 원인이라는 근거가 확인될 때만 리뷰 지적으로 남긴다.

재리뷰에서는 이전에 리뷰한 head와 현재 head를 비교한다. 기존 지적이 현재 코드에서 해결됐는지 먼저 확인하고, 새로운 지적은 이전 리뷰 이후 바뀐 코드에서만 만든다. 수정되지 않은 코드에서 지적이 계속 늘어나는 일을 줄이기 위한 규칙이다.

리뷰 결과를 만드는 것과 GitHub에 승인·수정 요청을 게시하는 것도 분리했다. 기본 동작은 읽기 전용이며, 게시에는 명시적인 요청이 필요하다.

이 기준은 [Google Engineering Practices의 코드 리뷰 가이드](https://google.github.io/eng-practices/review/reviewer/looking-for.html), [PR-Agent의 공개 리뷰 프롬프트](https://github.com/qodo-ai/pr-agent/blob/main/pr_agent/settings/pr_reviewer_prompts.toml), [React](https://react.dev/learn/you-might-not-need-an-effect)·[TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)·[Next.js](https://nextjs.org/docs/app/guides/production-checklist) 공식 문서와 비교해 정리했다. 외부 프롬프트를 복사하지 않고 공통으로 확인되는 판단 기준만 반영했다.

## 설치

리뷰 팩은 함께 설치한다.

```bash
./scripts/install.sh --target all --scope project --root path/to/project \
  --skill pull-request-review \
  --skill critical-review \
  --skill side-effect-check \
  --skill react-review \
  --skill typescript-review \
  --skill nextjs-review \
  --skill hygiene-review
```

먼저 dry run을 확인한 뒤 같은 명령에 `--apply`를 붙이면 Claude Code와 Codex가 공유하는 프로젝트 스킬로 연결된다.

## 현재 상태

이번 PR로 다섯 개의 새 스킬과 한국어·영어 사용 문서, 자연어 라우팅 평가 사례가 추가됐다.

- 공개 스킬 20개 검증
- 자연어 라우팅 사례 25개 검증
- 단위 테스트 14개 통과
- 공개 안전성 검사 통과
- 일곱 개 리뷰 스킬의 Claude Code·Codex 설치와 재설치 확인
- GitHub Actions 통과

[블로그 스킬을 공개했을 때](https://changbaebang.github.io/2026-08-19-blog-skill-without-publishing-my-voice/)와 마찬가지로, 특정 채널이나 봇, 개인 경로와 전용 승인 정책은 넣지 않았다. 공개 저장소에는 여러 환경에서 다시 사용할 수 있는 리뷰 선택과 검증 절차만 남겼다.

아직 실제 사용 결과를 회고할 단계는 아니다. 이번 글에서는 [`Agent Skill Garden`](https://changbaebang.github.io/2026-08-17-my-cursor-to-agent-skill-garden/)에 추가한 리뷰 팩의 구성과 현재 상태만 기록해 둔다.
