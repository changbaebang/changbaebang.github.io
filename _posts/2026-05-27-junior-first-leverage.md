---
layout: post
title: "주니어의 첫 레버 — 이번 주는 PR 3단락만 통일해도 충분하다"
date: 2026-05-27 13:05:00 +0900
permalink: /2026-05-27-junior-first-leverage/
tags: [junior, mentorship, pr, code-review, productivity, collaboration, ai]
---

> "처음부터 잘 쓰는 사람은 없다.  
> 대신 **항상 같은 순서로 설명하는 사람**은 빨리른다."

주니어에게 **AI 에이전트**를 가르칠 때, 나는 기능 목록부터 꺼내지 않는다.  
먼저 권하는 건 딱 하나다. PR 본문 **3단락** — `의도 / 리뷰어가 볼 곳 / 롤백`.

[횡단 PR](https://changbaebang.github.io/2026-05-18-reviewer-cross-app-pr/)이나 [생산성 레버](https://changbaebang.github.io/2026-05-16-productivity-leverage-points-ai-tools/) 글에서 이미 썼던 뼈대를, 이번엔 **한 주짜리 습관**으로만 줄였다.  
요즘 GeekNews에서도 비슷한 말이 반복된다. diff보다 [의도를 먼저 맞추는 리뷰](https://news.hada.io/topic?id=27316), [리뷰 습관이 곧 에이전트 습관](https://news.hada.io/topic?id=23221) 같은 이야기다.

## TL;DR

- 주니어의 첫 레버는 거대한 자동화가 아니라 **PR 설명 구조**다.
- `의도 / 리뷰어가 볼 곳 / 롤백`만 통일해도 팀 전체가 이득을 본다.
- 에이전트는 초안을, 검증과 합의는 사람이 한다.
- 파이프라인 채널에 올리는 monorepo PR도 마찬가지다. diff보다 **의도**가 먼저다.

---

구현 실력과 상관없이 **오늘부터** 쓸 수 있다.  
리뷰 스레드가 짧아지고, 슬랙에 “이게 뭐 하는 PR이에요?”가 덜 올라온다. 그래서 첫 레버로 잡는다.

## 주니어에게 특히 통하는 이유

설명 **순서**가 고정되면, 코드가 복잡해도 리뷰어가 맥락을 덜 잃는다.  
질문도 “이거 맞나요?”에서 “이 **의도**를 충족하는지 봐 주세요”로 바뀐다.  
한 달 뒤 비슷한 이슈가 나면, 그때 쓴 PR 본문이 의사결정 메모가 된다.

## 팀에서 본 장면 (익명)

아래는 여러 번 본 일을 **합쳐서 익명화**한 것이다. PR 번호·채널 ID·회사 URL은 넣지 않았다.

### 링크만 올린 경우

`#fe-pipeline`에 이런 메시지가 자주 보인다.

```text
리뷰 부탁드립니다 🙏
https://github.com/…/pull/NNNN
```

본문이 비어 있는 횡단 PR이면, 리뷰어는 어쩔 수 없이 diff부터 연다.  
shop·content·order가 한 PR에 섞이면, 슬랙에 **“어디를 같게 맞춘 거예요?”**가 먼저 온다. 리뷰는 늦어지고, 피로만 쌓인다.

### 같은 크기인데, 3단락만 채운 경우

같은 채널에 올렸는데 읽는 속도가 다르다.

```markdown
## 의도
- checkout·wallet에서 eligibility 문구를 같게 맞춘다.
- content 앱에서는 배너를 의도적으로 끈다 (PM 스레드 합의).

## 리뷰어가 볼 곳
- packages/payments-ui/usePartnerSummary.ts — surface별 캐시 키
- apps/checkout-shell/.../SubmitBar.tsx — optional payload 필드

## 롤백
- FF_PARTNER_PHASE2를 끄면 구 경로로 복귀. 배포 후 GNB·submit을 한 번 확인.
```

**의도**를 먼저 읽고 diff에 들어간다.  
인라인 코멘트도 “이 줄 맞나?”보다 “의도대로 구현됐나?”에 모인다.

### 에이전트는 여기서만 쓴다

“파일을 못 찾겠어요”라고 할 때, 튜토리얼보다 이렇게 시킨다.

```text
이 PR diff 기준으로 리뷰어가 볼 파일 3개만 추려 주고,
각 파일에서 판단 포인트를 한 줄씩 써서 3단락 초안을 만들어 줘.
```

초안은 에이전트가 쓰고, **의도 문장이 맞는지·롤백이 되는지**는 사람이 확인한다.  
[코드 리뷰를 잘하는 사람이 에이전트도 잘 다룬다](https://news.hada.io/topic?id=23221)는 말이, 현장에서는 이렇게 느껴진다.

---

## 복붙 템플릿

```md
## 의도
- 이번 변경으로 같게 맞추는 것:
- 의도적으로 제외한 것:

## 리뷰어가 볼 곳
- `path/to/fileA` — 무엇을 판단할지
- `path/to/fileB` — 무엇을 판단할지

## 롤백
- 어떤 토글/리버트 경로로 되돌릴지
- 롤백 후 확인할 화면/로그
```

## 자주 막히는 곳

- 의도는 아는데, **어느 파일**을 보여줄지 모른다
- 파일은 찾았는데, **어느 줄·어느 판단**을 물어봐야 할지 모른다
- 롤백에 `revert` 한마디만 적고 끝낸다

그때는 이렇게 부탁해도 된다.

```text
이 변경의 핵심 파일 3개를 추려 주고,
각 파일에서 리뷰어가 봐야 할 판단 포인트를 한 줄씩 써 줘.
```

초안은 에이전트, 최종 책임은 사람. 이 순서만 지켜도 된다.

## 이번 주, 하나씩만

1. monorepo PR마다 3단락 뼈대 넣기 — `#fe-pipeline`에 링크 올리기 **전에**
2. 리뷰 요청 전, **의도** 한 문장을 소리 내어 읽어 보기
3. 롤백 경로가 **정말** 되는지 한 번만 확인하기

하루에 하나만 해도 된다.  
“코드를 더 빨리”보다 “협업을 덜 틀리게”가 먼저 오른다.

## 마치며

주니어 생산성은 어려운 기술에서 시작하지 않는다.  
**같은 구조로 설명하는 습관**에서 시작한다.

**한 줄 결:** *첫 레버는 거대한 자동화가 아니다. PR 3단락을 꾸준히 쓰는 작은 반복이다.*

---

## 레퍼런스

### 내 글

- [리뷰어가 먼저 읽는 횡단 PR](https://changbaebang.github.io/2026-05-18-reviewer-cross-app-pr/)
- [생산성 레버 — 리뷰 가능성](https://changbaebang.github.io/2026-05-16-productivity-leverage-points-ai-tools/)
- [문제 퍼스트](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)

### GeekNews

- [AI 시대에 코드 리뷰, 어떻게 해야할까?](https://news.hada.io/topic?id=27316)
- [2026년 LLM 코딩 워크플로우](https://news.hada.io/topic?id=25755)
- [코드 리뷰를 잘하면 AI 에이전트도 잘 다룰 수 있음](https://news.hada.io/topic?id=23221)
- [AI 코딩의 함정](https://news.hada.io/topic?id=23335)

### 외부

- [CodeRabbit — AI vs human PR 이슈](https://coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report)
