---
layout: post
title: "리뷰 큐를 기다리는 동안 — 아래에서 실행해 본 습관과 Approve 병목"
date: 2026-06-02 11:00:00 +0900
permalink: /2026-06-02-review-queue-self-check-when-slack-is-quiet/
tags: [review, productivity, metrics, ci, cursor, engineering, frontend, collaboration]
---

> "생성은 빨라졌는데 merge는 왜 그대로일까.  
> 가끔 **병목은 키보드가 아니라 Approve 큐**에 있는 것 같다."

팀 문화는 회의에서만 만들어지지 않는다.  
리뷰 스레드 한 줄, PR 본문 한 단락이 **먼저** 쌓일 때도 있다. 나는 최근 그쪽——**실무 리뷰**——부터 손대 보기 시작했다.

팀에서 리뷰 요청 스레드에 올리는 **메시지 포맷**을 조금 손봤다. 위에서 내려온 규칙이 아니라, [리뷰 스레드 체크리스트](https://changbaebang.github.io/2026-05-21-review-thread-operating-checklist/)에서 쓰던 `결론 → 근거 → 확인 포인트` 순서를 **내 리뷰에 매일 적용**해 본 것이다.

Cursor를 쓰면서 달라진 것도 **리뷰량**보다 **문장을 쓰는 방식** 쪽에 가깝다. diff를 읽고 나서, 근거 몇 개와 QA 한 줄을 초안으로 받아 다듬어 보낸다. 의외로 그렇게 정리한 코멘트가 대화를 덜 돌아가게 만든다. 도구가 리뷰를 대신한 건 아니고, **리뷰 문장**이 조금 정돈된 느낌이다.

그런데 팀 전체 속도를 숫자로 보면, 내 쪽에서 실행해 본 것과는 별개로 **open PR의 Approve 대기**는 그대로인 경우가 많았다.  
아래에서 **리뷰 문장**을 조금씩 맞춰 가는 일과, 위에서 **Approve 큐**가 막히는 일은 **같은 팀 안의 다른 층**처럼 느껴졌다.  
이 글은 누군가를 탓하려는 글이 아니다. **frontend monorepo** 하나와 **팀 Slack 채널**을 샘플로, 두 층을 데이터와 익명 패턴으로 적어 본 메모다.

관련: [AI 생산성 지표 체크리스트](https://changbaebang.github.io/2026-06-01-ai-productivity-metrics-fe-checklist/) · [테스트와 CI가 진짜 생산성](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)

## TL;DR

- merge PR **첫 사람 리뷰 중앙값 ~0.7h** — 한번 잡히면 꽤 빠른 편. **병목은 큐(Approve 대기)** 쪽에 가깝다.
- 최근 14일 **open PR 86%가 Approve 없음** (21건 중 18건, 샘플 monorepo).
- GitHub **Approve 상위 5명이 ~62%** — 리뷰어 풀이 좁을 수 있다.
- **실무 리뷰**에서 `결론 → 근거 → 확인 포인트` 포맷을 **아래에서** 매일 실행해 봤다 — 위에서 규칙이 내려오기 전에.
- Slack `리뷰 완료` 메시지는 **한 workflow에 몰릴 수 있다** — 팀 지표로 쓰면 왜곡될 수 있다.
- 리뷰가 **늘 보장되지는 않을 때**, author 쪽 **self-check·CI**가 속도에 더 도움이 될 수 있다.

---

## 1. 실무에서부터 — 아래에서 위로 퍼지는 리뷰 문장

먼저 개인 쪽부터. "우리 팀 리뷰 문화를 바꾸자"고 선언한 적은 없다.  
대신 **내가 리뷰할 때마다** 아래 세 가지를 붙여 보기 시작했다. 회의 안건이 아니라, **스레드와 PR 위에서** 반복되는 습관이다.

- PR 본문에서 `의도 / 리뷰 포인트 / 롤백` ([주니어 첫 레버](https://changbaebang.github.io/2026-05-27-junior-first-leverage/)와 같은 뼈대)을 먼저 읽는다.
- GitHub 코멘트는 그대로 두고, Slack에는 `리뷰 완료 — Approve` 또는 `Comment`를 **첫 줄**에 적는다.
- Cursor로 diff 근거 3~5개, 비블로커 확인 포인트를 **초안 → 손보기**한다. (전부 AI가 쓴 건 아니고, 내가 고친 문장만 남긴다.)

문화는 가끔 이렇게 퍼진다.  
위에서 문서를 내리기 전에, **아래에서 같은 문장을 여러 번 쓰면** "아, 우리 팀은 이렇게 말하는구나"가 스레드에 남는다. 스레드에서 "그래서 머지해도 돼?"가 줄어든 주가 있어서, 나는 계속 쓰고 있다.

5~6월 **팀 채널**을 보면 `리뷰 완료` 포맷 메시지가 **우연히도** 내 workflow 쪽에 많이 모여 있다. 팀 전체가 따라온 건 아니다. 다른 분들은 GitHub만 쓰시거나, 포맷 없이 짧게 남기시는 경우도 많다.  
그래도 **한 사람이 꾸준히** 같은 형식으로 리뷰를 남기면, 채널에는 그 습관이 **흔적**으로 남는다. 그게 아래에서 위로 올라가는 문화 전파 중 **가장 작은 단위**가 아닐까 싶다.

다만 겸손하게 구분해야 한다.

- **내 리뷰 습관** ≠ 팀 전체 리뷰 **용량**
- **Slack에 보이는 리뷰** ≠ **GitHub Approve 분포**

팀 merge 속도를 보려면, 채널 문장 수보다 **open PR에 Approve가 붙었는지**, **누가 Approve 했는지**가 더 직접적이다.  
아래에서 열심히 **문장**을 맞춰도, 위의 **Approve 큐**는 별도로 풀어야 한다.

---

## 2. 숫자로 본 Approve 큐 — 느린 게 "리뷰 자체"만은 아닐 수 있다

2026년 5~6월, **하나의 frontend monorepo** GitHub API 기준으로 대략 집계했다. (조직·채널·PR 번호는 생략)

### open PR

| 지표 | 값 | 읽는 법 |
|------|-----|--------|
| 최근 14일 open PR | 21건 | — |
| Approve 없음 | 18건 (86%) | merge 게이트 대기 |
| Approve 1건 이상 | 3건 (14%) | merge 가능 후보 |

생성·푸시·CI는 도는데 **Approve 큐**가 비어 있지 않으면, main까지는 시간이 걸린다.

### merge된 PR

| 지표 | 값 |
|------|-----|
| **첫 사람 리뷰까지 중앙값** | **~0.7h** |

리뷰어가 diff를 열면, 보통 1시간 안팎에 첫 활동이 나온다.  
그래서 "리뷰가 원래 느리다"기보다 **누가·언제 큐에서 꺼내느냐** 쪽을 같이 봐야 할 것 같다.

### Approve 분포

| 지표 | 값 |
|------|-----|
| Approve 상위 5명 비율 | **~62%** |

[횡단 PR 리뷰](https://changbaebang.github.io/2026-05-17-reviewer-cross-app-pr/)에서도 말했듯, monorepo에서는 **도메인을 아는 사람**과 **경로를 아는 사람**이 다를 수 있다. Approve가 소수에게 몰리면, 그분들의 일정이 **팀 merge rate**에 그대로 묶인다.

---

## 3. 익명 패턴 — merge key가 한쪽으로 기울 때

아래는 **합성·익명**이다. 특정 PR·실명·Slack URL 없음.

### 패턴 A — architecture / package boundary PR

- monorepo 횡단 변경 (navigation·shared package 경계)
- Slack: `@도메인오너` + `@리뷰어B` co-tag
- 리뷰어B: co-tag 스레드에 **~1분** 안 "지금 본다"
- 도메인 오너: 같은 스레드 **~45분** 뒤 "지금 보고 있습니다"
- GitHub: 도메인 오너 **첫 COMMENT ~22h**, Approve 없음
- 다른 리뷰어: **~3h** 안 Approve, Comment 다수

co-tag된 두 사람인데, **Slack·GitHub 모두에서 게이트처럼 보이는 쪽**이 한쪽뿐일 수 있다.  
Approve가 도메인 오너에게만 의미 있게 묶여 있으면, 그 사람의 backlog가 **팀 merge backlog**처럼 느껴진다.

### 패턴 B — 같은 주, 다른 PR

- `@도메인오너` 멘션
- 도메인 오너: 샘플 기간 Slack·GitHub **무응답**
- 다른 리뷰어: **~80초** 안 Slack 응답

"리뷰가 없다"기보다 **특정 handle에 라우팅이 고정**돼 있을 때, 그 handle이 비면 전체가 잠깐 멈춘다.

### 패턴 C — 도메인 오너 본인 PR

- open 상태로 **Approve·리뷰가 오래 비는** 경우 (샘플에서 수 주 규모)

merge key를 쥔 사람도 **받을 리뷰 큐**에 갇힐 수 있다. 게이트가 한쪽으로만 기울면 **양방향**으로 답답해진다.

### 멘션 vs parent post

5~6월 **팀 채널**에서 **특정 handle `@` 멘션**은 **36건+**.  
같은 기간 그 handle의 **parent 글(리뷰 요청 스레드 개설)** 은 **0건**.

채널에 요청을 올리는 사람과, 멘션만 받는 사람 사이에 **역할 비대칭**이 생길 수 있다. 개인 성향 이야기로만 풀기엔, 구조가 한쪽으로 기울어 있는 느낌이다.

---

## 4. 시니어·도메인 오너 — 역할을 나누면 부담도 나뉜다

architecture PR에서 도메인 오너가 마지막 판단을 맡는 건 자연스럽다. **더 많이 아는 사람**이 boundary를 봐 주는 구조니까.

다만 **merge 게이트**까지 한 사람에게 묶이면, "꼼꼼히 본다"가 팀 전체 **open PR 86% Approve 대기**로 이어질 수 있다. 여기서 필요한 건 평가가 아니라 **역할 나누기**에 가깝다.

| 역할 | 할 일 |
|------|--------|
| 도메인 오너 | architecture **Comment**, boundary 판단 |
| named reviewer 2 | **Approve**, diff·회귀·테스트플랜 |
| backup | 24h 무응답 시 다음 reviewer |

"더 빨리"보다 **"병목이 되지 않게"** — Approve를 한 사람에게만 두지 않거나, SLA와 backup을 두는 쪽이 현실적이다.

참고: GitHub [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) — CODEOWNERS는 **라우팅**에 가깝고, 한 사람의 캘린더가 merge 한도는 아니다.

---

## 5. 리뷰가 늘 활발하지 않을 때 — self-check가 속도에 도움이 되는 이유

팀 리뷰가 **항상 바로** 이어지지 않으면, author 입장에서는 이렇게 정리할 수 있다.

> 큐를 기다리는 동안 **스스로 한 번 더 검증**해 두면, 리뷰 1라운드에 끝날 확률이 올라간다.

리뷰를 없애자는 뜻은 아니다. [생산성 레버](https://changbaebang.github.io/2026-05-15-productivity-leverage-points-ai-tools/)에서 말한 **리뷰 가능성(reviewability)** 과 같은 축이다.

### Approve 대기 중 author 쪽 최소선

1. **CI green** — lint, typecheck, unit ([test-ci 글](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)의 5게이트)
2. **PR 3단락** — 의도 / 리뷰 포인트 / 롤백
3. **에이전트 pre-review** — "이 diff에서 깨질 surface 3개" 초안
4. **스크린·스모크** — Chromatic / 핵심 플로우 1~2개
5. **비블로커 QA 목록** — 리뷰어가 물을 질문을 author가 먼저 적어 둠

리뷰가 0.7h 만에 시작돼도, **준비 없는 PR**이면 2~3라운드로 늘어난다.  
Approve 큐가 길 때 self-check는 **팀 버프**라기보다 **author 쪽 습관**에 가깝다.

### AI도 같은 방식으로 — 아래에서 먼저

리뷰 코멘트 초안을 AI에게 맡기는 것도 **팀 합의 전에**, 내 workflow에서 먼저 써 봤다.  
"이제부터 다들 이렇게 하자"가 아니라, **내 리뷰가 조금 나아지면** 그게 스레드에 쌓이는 쪽을 택했다.

- diff → 근거 번호 리뷰 코멘트 초안
- PR 본문 → edge case 질문 목록
- pre-merge → 회귀 surface 체크리스트

author·reviewer 둘 다 쓰는 게 정답인지는 아직 모르겠다. 다만 **첫 라운드 품질**이 올라가면 왕복이 줄어든다.  
팀 Approve가 여유롭지 않을수록, **author-side 검증**과 **리뷰 문장 정리**를 아래에서 꾸준히 해두는 편이, 체감상 가장 손에 잡히는 레버였다.

---

## 6. 부드럽게 고칠 수 있는 운영 (제안)

### 멘션

- 팀 전체 blanket 멘션은 줄이고
- **named reviewer 2 + domain owner Comment**로 역할을 나누고
- 24h 무응답 시 backup reviewer가 Approve할 수 있게 (팀 convention 또는 GitHub rules)

### 지표 — Slack 말고 이쪽

| 지표 | 왜 |
|------|-----|
| open PR **Approve 대기 시간** p50/p90 | 큐 병목 |
| **reviewer당 weekly Approve** 분포 | 소수 집중 |
| merge PR **review round count** | self-check 효과 |
| `@` 멘션 **건수 vs parent 요청 글** | 라우팅 기울기 |

Tab·LOC 대신 cycle time 쪽은 [metrics 글](https://changbaebang.github.io/2026-06-01-ai-productivity-metrics-fe-checklist/) 참고.

### 대형 PR

- package / navigation / shared boundary:
  - domain owner: **Comment**
  - Approve: **한 사람 sole approver가 아니게** (리뷰어 2명 등)
- [merge queue threshold](https://changbaebang.github.io/2026-04-29-merge-queue-threshold/)처럼 크기·경계별 규칙

---

## 마무리

나는 팀 회의에서 "리뷰 더 하자"고 말한 적은 없다.  
대신 **팀 채널에 올리는 리뷰 메시지**와 GitHub 코멘트를, **실무에서 매일** 맞춰 가며 실행해 왔다. 위에서 규칙이 내려오기 전에, 아래에서 **같은 문장을 반복**하는 쪽——그게 내가 할 수 있는 문화 전파였다.

이 글도 그 연장선이다. 스레드에서 쓰던 생각을 **블로그로 한 번 더 정리**해 두면, 나중에 팀에서 "우리 리뷰 포맷 뭐였지?" 할 때 **아래에서 올라온 자료**가 하나 더 생긴다.

숫자를 다시 보면, **층이 둘**이다.

**아래 (실무·습관)**  
- 리뷰 문장 포맷, self-check, AI로 초안 정리  
- 스레드가 덜 돌아가게 만드는 **대화 품질**

**위 (구조·큐)**  
- open PR **86% Approve 없음**
- merge PR **첫 리뷰 ~0.7h** — 큐만 열리면 빠름
- Approve **~62%가 5명**
- architecture PR **co-tag인데 응답·Approve 비대칭**

아래에서 열심히 해도, 위가 막혀 있으면 **merge wall-clock**은 그대로일 수 있다.  
반대로 위만 고쳐도, PR 본문과 리뷰 문장이 비어 있으면 **왕복**은 줄지 않는다.

리뷰 culture를 키우는 것과 **self-check·CI 하한선**을 높이는 것은 대립하지 않는다.  
**아래에서 문장을 맞추고**, **위에서 Approve 큐를 넓히는**——둘 다 천천히, 같이 가야 merge까지의 시간이 줄어든다.

---

## 레퍼런스

- [AI 코딩 생산성, FE 팀이 대신 보는 지표](https://changbaebang.github.io/2026-06-01-ai-productivity-metrics-fe-checklist/)
- [테스트와 CI가 진짜 생산성](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)
- [리뷰 스레드 운영 체크리스트](https://changbaebang.github.io/2026-05-21-review-thread-operating-checklist/)
- [생산성 레버 — 리뷰 가능성](https://changbaebang.github.io/2026-05-15-productivity-leverage-points-ai-tools/)
- GitHub — [About pull request reviews](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews)
- GitHub — [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
