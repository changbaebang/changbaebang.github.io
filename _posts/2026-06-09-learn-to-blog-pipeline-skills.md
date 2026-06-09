---
layout: post
title: "공부가 멈춘 게 아니다 — 배울 것과 쓸 것을 나누며 읽은 것을 글로"
date: 2026-06-09 16:23:00 +0900
permalink: /2026-06-09-learn-to-blog-pipeline-skills/
tags: [learning, blog, cursor, workflow, skills, agent-ops, retrospective]
series: [skill-is-code]
---

> 배움은 의지 문제가 아니라 **마찰 문제**였다.  
> “쓸 글”과 “배울 것”을 나누고, 스킬로 연결하니 **끝까지 이어지기** 쉬워졌다.

한동안 공부를 안 한 것처럼 느꼈다.  
GeekNews 탭은 늘 열려 있었고, YouTube는 “나중에” 큐에만 쌓였다. 읽은 것도 있었는데 **끝**이 없었다. `done`이 없으니 죄책감만 남는다.

AI 도구를 쓰면서 역설이 커졌다. **글 생성**은 빨라졌는데, **읽기 → 정리 → 발행**은 오히려 더 흐려질 수 있었다. 북마크와 초안이 한곳에 섞이면, 매번 “오늘은 뭐부터?”를 처음부터 다시 결정해야 했다.

이 글은 생산성 자랑이 아니다. **폴더 질문을 나누고**, `/cb:learn-radar` · `/cb:blog-radar` · `/cb:blog`로 **읽기와 발행 사이를 이어 붙인** 방법을 정리한다. 명령 이름을 외울 필요는 없다. [앞글 — Playwright smoke와 스킬](https://changbaebang.github.io/2026-06-08-cursor-playwright-smoke-pr-evidence/)이 “일할 때 스킬을 부르는 법”이었다면, 이번 글은 “**읽은 것이 글로 남는** 법”이다.

바꾼 건 의지력이 아니라 **폴더 두 개와 명령 세 개**다.

## TL;DR

- 공부를 안 한 게 아니라, **읽고 끝나는 루프**에 갇혀 있었다.
- `learning-backlog`(배울 것)와 `blog-drafts`(쓸 것)를 **분리**하고, learn-radar · blog-radar · blog 명령으로 이었다.
- 학습 항목 → `done` → `promote` → 초안 → `publish-check` → `_posts` → **초안 삭제** — 이 순서가 파이프라인이다.
- 같은 루프로 [Twelve Ways 큐레이션](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)·[Chrome UA 2편](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)·[harness 회고](https://changbaebang.github.io/2026-06-04-harness-anatomy-cursor-retro/)처럼 **읽은 것이 글로 남기 쉬워졌다**.

---

## 1. 왜 공부가 멈춘 것처럼 보였나

멈춘 게 아니라 **전환 비용**이 컸다.

| 증상 | 원인 |
|------|------|
| 링크만 쌓임 | “봤다”는 기록이 없음 |
| 글 쓰려면 또 처음부터 | 학습 메모와 초안이 섞임 |
| AI로 초안은 빠른데 발행은 느림 | 식별자·SEO·퇴고·날짜를 매번 새로 |
| 같은 주제를 두 번 씀 | 발행 SSOT와 초안 목록이 따로 놀음 |

특히 마지막이 크다. harness를 여러 번 썼다고 느낄 때도, 실제로는 **읽은 출처·각도·독자**가 달랐다. 문제는 그 차이가 **파일 이름**이 아니라 **머릿속**에만 있어서, 다음 주에 또 겹치기 쉬웠다는 점이다.

---

## 2. 바꾼 것: 질문이 다른 SSOT 두 개

| | 경로 | 매일 묻는 질문 |
|---|------|----------------|
| **배울 것** | `~/docs/learning-backlog/items/` | “이걸 **봤나**?” |
| **쓸 것** | `~/docs/blog-drafts/` | “이걸 **쓸 건가**?” |
| **발행** | GitHub `_posts` | “**올렸나**?” |

`blog-radar`는 “오늘 **쓸** 주제 후보”를 고른다.  
`learn-radar`는 “먼저 **배울** 것”을 고른다.  
둘을 섞지 않는 게 핵심이다. 읽을거리 스캔과 쓸거리 결정은 **같은 뇌 비용**을 쓰지만, 산출물이 다르다.

발행 후에는 `blog-drafts`의 해당 초안을 **삭제**한다. `_posts`와 live URL만 남기면, “이미 썼나?”를 다시 묻지 않아도 된다. [my-cursor 회고 §7](https://changbaebang.github.io/2026-05-20-my-cursor-architecture-retro/)에서 말한 **출처가 두 군데면 충돌한다**는 원칙과 같다.

---

## 3. 스킬로 묶은 루프

```text
/cb:learn-radar scan      → 후보 목록
/cb:learn-radar pick      → 이번 주 1~2개
(직접 읽기·시청)
/cb:learn-radar done      → takeaway 기록
/cb:learn-radar promote   → blog-drafts skeleton
(본문 퇴고)
/cb:blog publish-check    → 식별자·SEO·날짜
_posts commit → blog-drafts 해당 파일 삭제
```

**사건·뉴스**는 learn이 아니라 `blog-radar`나 바로 `draft`로 들어올 때도 있다. [Mini Shai-Hulud 글](https://changbaebang.github.io/2026-06-07-npm-shai-hulud-install-boundary/)은 “먼저 배울 논문”이 아니라 advisory가 온 **쓸 주제**에 가깝다. 파이프라인은 하나지만 **입구**는 둘이다.

### 실제로 돌아간 사이클

**① Third Bit → 큐레이션 → 각색**

- learn item: [Twelve Ways 원문 정독](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/)
- `done` 메모: Goodhart, 쉬운 절반만 측정
- 발행: [큐레이션](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/) → 이어서 [FE 지표 체크리스트](https://changbaebang.github.io/2026-06-01-ai-productivity-metrics-fe-checklist/)

같은 출처에서 **소개 글**과 **실무 각색**이 갈라졌다. `promote`는 skeleton만 만들고, 각도는 `done` 이후에 정한다.

**② Chrome UA → 2편 시리즈**

- learn item: Privacy Sandbox UA reduction
- 1편: [브라우저가 무엇을 줄였는지](https://changbaebang.github.io/2026-06-01-chrome-user-agent-reduction-fe-notes/)
- 코드 grep이 남아 2편: [WebView와 UA 감사](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)

학습 항목에 `follow_up_slug`를 남겨 두면, “1편만 쓰고 끝”하지 않게 된다.

**③ LangChain harness → Cursor 회고**

- learn item → [Harness anatomy 읽고](https://changbaebang.github.io/2026-06-04-harness-anatomy-cursor-retro/)
- harness **제품** 설명이 아니라 “내 설정과 대조표”만 적었다. 같은 달 harness 글이 많아도, **입구가 learn item**이면 각도가 겹치지 않게 잡히기 쉽다.

**④ WebAuthn — learn 없이도 되지만 같은 퇴고 축**

- [Immediate UI 체크리스트](https://changbaebang.github.io/2026-05-26-webauthn-immediate-ui-fe-checklist/)는 공식 문서를 “붙일 때 실패하는 지점”으로 바꾼 글이다. `promote` · `publish-check` 순서는 같다.

**원칙:** 스킬은 코드다([skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)). 틀리면 스킬을 고친다. “의지”가 아니라 **diff**가 쌓여야 한다.

---

## 4. “많이 올린다”의 정체

겉보기엔 6월 초에 글이 잦아 보였다 — Chrome UA 2편, harness 회고, Shai-Hulud, [agent interface](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/), [Playwright smoke](https://changbaebang.github.io/2026-06-08-cursor-playwright-smoke-pr-evidence/) 등. **의지나 습관**보다 **마찰이 낮아진** 결과에 가깝다.

측정도 LOC가 아니다.

- `learning-backlog`에서 `status: done` 비율
- `promote` → `_posts` conversion
- 발행 후 초안이 **삭제됐는지** (SSOT 중복 여부)

[Twelve Ways](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)에서 말한 것처럼, “많이 썼다”는 지표는 쉽게 속인다. **읽은 것이 글로 남았는지**가 더 솔직하다.

하네스를 **제품**으로 또 쓰지 않으려는 이유도 여기 있다. [하네스 글](https://changbaebang.github.io/2026-05-24-claude-code-harness-living-docs/)은 “에이전트 통제” 설명이고, 이 글은 **읽기·쓰기 루프** 설명이다. 같은 단어, 다른 층이다.

---

## 5. 다른 사람이 가져갈 최소 세트

팀 monorepo 없이도 **개인 `~/docs` + `~/.cursor/commands/cb`**만으로 시작할 수 있다. 명령 원문은 [my-cursor](https://github.com/changbaebang/my-cursor) — [learn-radar](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/learn-radar.md), [blog-radar](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/blog-radar.md), [blog](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/blog.md). (발행 시점 스냅샷이며, 로컬은 계속 수정된다.)

1. **learn item** 파일 하나 = `source` + `status` + `After learning` 세 줄
2. **blog-radar / learn-radar 분리** — “읽을거리”와 “쓸거리” 혼합 금지
3. **`promote`는 skeleton** — 본문은 학습·회고 후에
4. **발행 후 초안 삭제** — `_posts` + live URL만 SSOT
5. **`publish-check` 한 번** — 회사 식별·SEO·permalink 날짜. `blog-drafts/PUBLISH-CHECKLIST.md` 한 장이면 충분하다

완벽한 자동화가 먼저가 아니다. **폴더 질문이 바뀌는 것**만으로도, 읽기·쓰기·발행 같은 잡무가 **덜 헤매이게** 된다.

---

## 마치며

공부를 재개한 게 아니라, **끝나는 지점**을 만들었다.

- 배울 것과 쓸 것의 **폴더 분리**
- `done` → `promote` → `publish-check` → **초안 삭제**의 **고정 순서**
- 틀리면 고치는 **스킬 diff**

**한 줄 결:** *의지력으로 공부를 살리기보다, 읽은 것이 글로 남을 때까지 마찰을 줄이는 쪽이 낫다.*

---

## 읽을 거리

### 이 블로그

- [AI 스킬은 코드다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)
- [Twelve Ways 큐레이션](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)
- [my-cursor 아키텍처 회고](https://changbaebang.github.io/2026-05-20-my-cursor-architecture-retro/)
- [Playwright smoke — 스킬과 자연어](https://changbaebang.github.io/2026-06-08-cursor-playwright-smoke-pr-evidence/)

### 명령 원문 ([my-cursor](https://github.com/changbaebang/my-cursor))

| 구분 | 링크 |
|------|------|
| 학습 | [learn-radar](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/learn-radar.md) |
| 글 주제 | [blog-radar](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/blog-radar.md) |
| 발행 | [blog](https://github.com/changbaebang/my-cursor/blob/main/commands/cb/blog.md) |
