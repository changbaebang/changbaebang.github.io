---
layout: post
title: "AI 보조 코딩 생산성 — 잘못 재는 12가지 (읽을거리)"
date: 2026-05-23 21:00:00 +0900
tags: [ai, productivity, metrics, curation, reading, engineering]
---

> 번역 글이 아니다. **읽어볼 가치가 있는 글을 알리는 메모**다.  
> 본문·근거·인용은 모두 원문에 있다.

Greg Wilson의 [Twelve Ways to Be Wrong About AI-Assisted Coding](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/) (2026-05-20, Third Bit)을 읽었다.

AI 코딩 **사용법**이 아니라, ROI·생산성 **측정법**이 얼마나 허술한지 12가지로 정리한 글이다.  
agile/TDD 측정 비판과도 같은 결이라, “도구가 좋다/나쁘다” 논쟁보다 **무엇을 재면 틀리는가**에 집중한다.

나는 LOC·PR 수·“더 생산적이다” 설문을 보면 불안해진다.  
실무에서는 생성 속도보다 **검증·리뷰·복구 루프**가 팀 속도를 좌우하기 때문이다. 이 글은 그 직감을 방법론 언어로 정리해 준다.

**원문:** [Twelve Ways to Be Wrong About AI-Assisted Coding](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/)

---

## 12가지 — 한 줄씩만 (요약)

아래는 각 섹션의 요지만 적었다. 해설·논문 인용·反론은 **원문**을 본다.

1. **생성 LOC** — 코드 양은 verbosity일 뿐, 품질·생산성의 프록시가 아니다.
2. **인위적 과제 시간** — 90분 toy task는 레거시·모호한 티켓·회의가 있는 실제 개발과 다르다.
3. **대조군 없는 전후 비교** — 도입 전후만 보면 채용·CI 개선 등 다른 변수와 분리할 수 없다.
4. **“더 생산적” 설문** — Hawthorne·novelty·social desirability로 자가 보고는 흔들린다.
5. **커밋·PR·티켓 수** — Goodhart: 지표가 목표가 되면 숫자만 오른다.
6. **쉬운 절반만 측정** — 생성은 빠른데 리뷰·보안·부채 비용은 안 본다.
7. **채택률 = 성공** — 설치·사용 ≠ 유용·정확.
8. **자원자 vs 비자원자** — early adopter 특성과 도구 효과가 섞인다 (selection bias).
9. **개인만 측정** — 한 사람의 코딩 속도 ↑ ≠ 팀의 ticket→prod cycle ↓.
10. **짧은 novelty 기간만** — 4주 부스트와 장기 복잡도·부채는 다르다.
11. **Tab 수락률 = 품질** — “그럴듯함”과 correctness/security는 다르다.
12. **AI vs 아무것도 없음** — IDE·문서·동료와 비교하지 않으면 baseline이 약하다.

---

## 왜 북마크했는가

- “AI 도입 성과 보고”를 받을 때 **어디가 허술한지** 짚을 수 있는 체크리스트다.
- [AI 스킬은 코드다 — 쓰면서 고쳐야 산다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/) — 하네스·검증 루프
- 번역·해설을 붙이기보다, **원문 링크 하나로 팀에 넘기기**에 적합하다.

---

## 마치며

이 글은 AI 코딩을 반대하거나 찬양하는 글이 아니다.  
**측정을 엄밀하게 하라**는 글이다.

한국어로 전문 번역본을 대신 기대하기보다, 위 12줄로 “읽을지 말지” 판단하고 원문으로 가면 충분하다.

**한 줄 결:** *AI ROI 논쟁 전에, 무엇을 재고 있는지부터 의심하라.*

---

## 읽을 거리

- **원문 (필수):** [Twelve Ways to Be Wrong About AI-Assisted Coding](https://third-bit.com/2026/05/20/twelve-ways-to-be-wrong/) — Greg Wilson, Third Bit
- [AI 스킬은 코드다 — 쓰면서 고쳐야 산다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)
- [리뷰 스레드 운영 체크리스트](https://changbaebang.github.io/2026-05-21-review-thread-operating-checklist/)

*이 포스트는 원문의 번역·재게시가 아닙니다. 요약과 링크만 제공합니다.*
