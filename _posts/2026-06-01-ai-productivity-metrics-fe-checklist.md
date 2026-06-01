---
layout: post
title: "AI 코딩 생산성, LOC·설문·Tab 수락률로 재지 마라 — FE 팀이 대신 보는 것"
date: 2026-06-01 15:23:00 +0900
permalink: /2026-06-01-ai-productivity-metrics-fe-checklist/
tags: [ai, productivity, metrics, engineering, frontend, management]
---

> "생성량이 늘었다"는 보고서는 마케팅에 가깝다.  
> **팀이 느끼는 속도**는 검증·리뷰·복구 비용에서 결정된다.

Greg Wilson의 [Twelve Ways to Be Wrong About AI-Assisted Coding](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/)은 AI 코딩 **사용법**이 아니라 **평가법**을 비판한다.  
12가지 함정은 [요약 글](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)에 정리해 두었다. 여기서는 그다음 단계 — **FE 팀 대시보드에 올릴 대체 지표**만 골랐다.

번역 글이 아니다. 실무 체크리스트다.  
toy task·대조군 없는 전후 비교·채택률·자원자 편향 등 나머지 함정은 원문과 큐레이션 글을 본다.

## TL;DR

- LOC·커밋 수·Tab 수락률·“87%가 더 생산적” 설문은 **행동을 왜곡**한다.
- 90분 toy task 벤치마크는 **대규모 레포·모호한 티켓**과 맞지 않는다.
- AI는 **쉬운 절반(생성)**만 빠르게 한다. 리뷰·보안·부채는 숨겨진 비용이다.
- FE 팀은 **시스템 지표**(cycle time, revert, 반복 리뷰)를 본다.

---

## 1) 생성 LOC — 코드 양만 재는 함정

**함정:** AI 도입 후 LOC 40% 증가 = 생산성 40% 증가처럼 보고한다.

**왜 틀리나:** 2000줄을 지우고 200줄로 정리한 개선은 이 지표에서 **마이너스**로 잡힌다.  
코드가 많아질수록 읽기·유지·디버그 부담도 커진다.

**FE 팀이 대신 보는 것:**

- diff **삭제/추가 비율** (리팩터 PR인지 feature PR인지 구분)
- **순복잡도** 또는 lint complexity 경고 추이
- “생성 LOC”가 아니라 **main merge까지의 lead time**

---

## 2) “더 생산적이다” 설문

**함정:** “87% 개발자가 AI로 더 생산적” 같은 자기보고.

**왜 틀리나:** Hawthorne 효과(관찰되면 행동이 바뀜), novelty(신기함), social desirability(상사가 원하는 답)가 섞인다.

**FE 팀이 대신 보는 것:**

- 설문 대신 **행동 로그**: CI 통과율, revert율, hotfix 간격
- 도입 4주 vs 12주 **추이** (신기함 구간과 분리)
- “느낌”이 아니라 **같은 티켓 유형**의 lead time 비교

---

## 3) 커밋·PR·티켓 수

**함정:** McKinsey식 “활동량 = 생산성”.

**왜 틀리나:** Goodhart의 법칙 — 지표가 목표가 되면 **작은 커밋·쪼갠 티켓**으로 숫자만 오른다.

**FE 팀이 대신 보는 것:**

- PR **크기 분포** (너무 작은 PR이 폭증하면 게이밍 신호)
- **배포까지 이어진 merge** (main 배포와 연결된 PR)
- 리뷰 코멘트 **반복 유형** (같은 실수가 줄었는지)

---

## 4) 쉬운 절반만 측정 (생성 ↑, 리뷰 ↓)

**함정:** 생성 시간만 재고, LLM 코드 리뷰·보안·부채는 안 본다.

**왜 틀리나:** 생성물 상당수에 품질·보안 이슈가 있고, 시간이 촉박할수록 **그럴듯하지만 틀린 코드**를 더 수용한다.

**FE 팀이 대신 보는 것:**

- AI 보조 PR의 **리뷰 라운드 수** vs 사람이 직접 작성한 PR
- 시니어 **리뷰 부하** (생성량은 늘었는데 시니어 생산성은 줄었다는 연구 패턴)
- 보안·접근성 **자동 게이트** 통과율

관련: [테스트와 CI가 진짜 생산성인 이유](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/) — 검증 루프를 CI에 고정하는 방법.

---

## 5) Tab 수락률 = 품질

**함정:** “수락률 33%, 만족도 높음” = 도구가 좋다.

**왜 틀리나:** 수락은 **그럴듯함**이지, correctness(정확성)·security(보안)·maintainability(유지보수성)가 아니다.  
마감이 촉박할수록 수락률은 **잘못된 이유로** 올라간다.

**FE 팀이 대신 보는 것:**

- 수락률 대신 **post-merge defect rate**
- 수락된 코드의 **테스트 커버리지 변화량**
- 수락 후 **24시간 안 revert·hotfix** 비율

---

## 6) 개인 속도 vs 팀 cycle time

**함정:** “개발자 A는 30% 빨라졌다” — 팀 배포 속도는 그대로.

**왜 틀리나:** 병목은 코드 작성이 아닐 수 있다. 리뷰·QA·배포·정합성 검증 **쪽**이다.  
생성량만 늘리면 **리뷰 큐**가 병목이 된다.

**FE 팀이 대신 보는 것:**

- ticket → production **end-to-end** 시간
- WIP(진행 중 PR) **한도** 준수 여부
- 리뷰 SLA (첫 응답·merge까지)

---

## FE 팀용 “대신 재기” 표 (한 장)

| 피하기 | 대신 보기 |
|--------|-----------|
| 생성 LOC | main merge lead time, revert율 |
| 설문 만족도 | CI 통과율, hotfix 간격 |
| 커밋/PR 수 | PR 크기 분포, 배포까지 이어진 merge |
| Tab 수락률 | post-merge defect, 24h revert·fix율 |
| 개인 코딩 속도 | ticket → prod cycle time |
| 도구 채택률 | 채택 후 90일 cycle time 변화 |

---

## 마치며

AI 도구 ROI를 묻는 순간, 잘못된 답이 먼저 나온다.  
LOC, 설문, 수락률 — **측정하기 쉬운 것**이지 **중요한 것**이 아니다.

FE 팀이 할 일은 도구를 더 많이 쓰게 만드는 게 아니라,  
**검증·리뷰·복구 루프**가 도구 속도를 따라잡게 만드는 것이다.

**한 줄 결:** *AI 생산성은 “얼마나 많이 생성했나”가 아니라 “얼마나 빨리 안심하고 ship했나”다.*

---

## 읽을 거리

- **원문:** [Twelve Ways to Be Wrong About AI-Assisted Coding](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/) — Greg Wilson, Third Bit (2026-05-20)
- [12가지 함정 요약 (큐레이션)](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)
- [테스트와 CI가 진짜 생산성인 이유](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)

*이 포스트는 원문의 번역·재게시가 아닙니다. 요약·체크리스트와 링크만 제공합니다.*
