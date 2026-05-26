---
layout: post
title: "비싼 실수는 AI가 아니라 검증 부재 — geohot의 Sloptember를 읽고"
date: 2026-05-26 17:30:00 +0900
permalink: /2026-05-26-geohot-slop-verification-not-agents/
tags: [ai, agents, quality, verification, geohot, curation, engineering]
---

> 10배 아웃풋은 싸지 않다.  
> **10배 검증**이 붙을 때만 싸다.

George Hotz(geohot)가 [The Eternal Sloptember](https://geohot.github.io/blog/jekyll/update/2026/05/24/the-eternal-sloptember.html)에서 쓴 글이 [GeekNews](https://news.hada.io/topic?id=29852)에 올랐다. 문장은 거칠지만, 실무에서는 꽤 잘 맞는 지적이 있다.

> *I'm calling it now, the adoption of AI agents into software development will be one of the most costly mistakes in the field's history.*

이 글은 번역이 아니다. geohot을 인용하면서, 내가 이미 써 온 [문제 퍼스트](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/), [테스트·CI](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/), [Fowler harness](https://changbaebang.github.io/2026-05-25-fowler-harness-roles-not-layers/)와 같은 축으로 읽어 본 각색이다.

나는 에이전트를 끄자는 쪽이 아니다. **검증 없이 밀어붙이는 쪽**이 문제라고 본다.

## TL;DR

- geohot: 에이전트는 프로그래밍을 하지 않는다. 프로그래밍 **분포**를 흉내 낼 뿐이고, 그 결과는 점점 **망가진 채로 그럴듯**해진다.
- 진행은 앞쪽 80%에 몰리고, 마지막 다듬기는 **슬롯머신** 같다 — 끝까지 잘 안 간다.
- 진짜 위험은 **스스로 검증하지 않는 채** 10배를 내는 사람과, 그걸 KPI로 밀어 넣는 조직이다.
- Karpathy나 Anthropic이 말하는 “리뷰하면서 쓴다”는 전제가 있을 때만 반론이 성립한다.
- *AI psychosis*는 정신과 진단명이 아니라, **과몰입·무검증·출력만 키우기**에 가깝다.

---

## geohot이 말한 것

tinygrad 일부, USB↔PCIe 칩 리버싱까지 **6개월 직접** 써 본 뒤 쓴 글이다. “한 번도 안 써 봤다”는 식의 비판과는 결이 다르다.

핵심만 옮기면 이렇다.

- **Agents cannot program** — 통계 모델이 코드 **분포**를 mimic할 뿐이다.
- 출력은 망가져 있는데, **점점 알아채기 어렵다**.
- 진행은 앞에 몰리고, 마지막은 **슬롯머신 레버** — *It never quite gets there.*
- 잘하는 사람은 slop를 slop으로 알아보고 **한 줄씩** 읽는다. 못하는 사람은 에이전트로 **10배**를 쏟는다.
- 시대는 *golden era for buckets of slop, dark age for gems* — 쓰레기는 넘치고, 진짜 좋은 건 드물어진다.
- 실패한 테스트를 주석 처리하고 “다 통과”라고 말하는 식의 RLVR도 비판한다.

Apple 엔지니어에게 AI를 밀어 넣는다는 얘기도 나온다. 여기서는 geohot **인용**으로만 두고, 특정 회사 이야기로 넓히지는 않는다.

---

## 동의하는 부분

### 비싼 실수는 “도구 이름”이 아니라 구조

실무에서 비용이 터지는 순간은 보통 비슷하다.

- PR은 늘었는데 **성공 기준**은 없다.
- 테스트는 green인데 **의미 있는 assertion**은 없다.
- “에이전트가 했어요”가 **검증**을 대신한다.

[테스트·CI 글](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)에서 말한 것과 같다. 생성만 빨라지고 **완료 정의**가 없으면, 재작업이 곧 비용이다.

### 80%와 슬롯머신

> *The agent frontloads all the progress, then gives you a slot machine lever to pull to hope the finishing work gets done. It never quite gets there.*

이건 “AI를 쓰지 마라”가 아니라 **마지막 20%의 문제**다. 시니어는 slop를 알아보고 되돌린다. 조직이 **출력만** 재면 평균 품질은 내려간다.

[Twelve Ways 큐레이션](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)이 짚은 것처럼, 생성량·PR 수만 보면 10배 아웃풋과 10배 slop를 같이 키울 수 있다.

### 테스트를 주석 처리할 때

RLVR이나 reasoning 논쟁은 기술 이야기지만, 실무 번역은 단순하다.

**실패를 숨기는 건 검증이 아니다.**

[Fowler harness 글](https://changbaebang.github.io/2026-05-25-fowler-harness-roles-not-layers/)에서 말한 것처럼, 점수와 층만 올리고 **oracle**(이게 맞다는 기준)이 없을 때 나는 증상이다.

---

## 동의하지 않는 부분

### “절대 못 한다”까지는 아니다

geohot은 LeCun·Marcus 쪽에 가깝다고 밝혔다. 나는 **절대주의**까지는 가지 않는다.

보일러플레이트, 반복 마이그레이션, 테스트 뼈대 같은 **좁은 범위**에서는 이미 쓸 만하다. 문제는 범위를 말하지 않은 채 “에이전트 = 만능”으로 조직이 밀 때다.

### Karpathy·Anthropic과의 온도 차

맥락상 Karpathy는 에이전트 쪽으로 무게를 옮겼고, Anthropic도 “엔지니어는 **리뷰**한다”는 이야기가 나온다. geohot이 **완전히 틀렸다**는 뜻은 아니다.

**리뷰·CI·한 줄씩 읽기**가 전제일 때만 그 그림이 된다. 전제가 없으면 geohot이 말한 “하위 성과자 10배” 시나리오가 그대로다.

---

## AI psychosis — 무섭한 단어, 풀어 쓰면

geohot 글 마지막은 이렇게 끝난다.

> *The real story of this era will be who manages to avoid harming themselves in their AI psychosis.*

한국어로는 “정신증”에 가깝게 들려서 무섭게 느껴지는 게 당연하다. 다만 geohot이 쓴 *psychosis*는 **병원 진단명이 아니다.** “AI 에이전트 붐에 말려 들어가 **스스로와 팀을 해치는 패턴**”을 가리키는 **은유**에 가깝다.

- *harm themselves* — 번아웃만이 아니다. **검증 없이 slop를 쌓고**, 나중에 되돌리기 어려운 부채와 신뢰 손실을 만드는 것.
- *the real story* — 누가 모델을 이기느냐가 아니라, **누가 미친 듯이 밀다가 안 망하느냐**.

geohot도 에이전트를 6개월 써 봤고, Google 대체·프로토타입에는 유용하다고 말한다. “AI 쓰면 정신병”이 아니라, **판단 없이 밀 때** 위험하다는 쪽이다.

### 같은 단어, 다른 뜻

업계에서는 [AI psychosis](https://www.vellum.ai/blog/ai-psychosis-is-real)라는 말이 두 갈래로 쓰인다.

**A. 생산성 과몰입** — Karpathy가 말한 “끝없이 가능성을 파악하려는 상태”. 에이전트에 오래 붙어 있고, 토큰이 남으면 아깝다는 심리도 섞인다. 그런데 **엄청 ship**하기도 한다.

**B. 임상·경계 붕괴** — 챗봇과의 관계에서 현실 감각이 흐려지는 극단적 사례. 코딩 에이전트 논쟁과는 **별개**다.

이 글은 A와 **조직 slop**만 다룬다. geohot 마지막 한 줄만 보고 B까지 단정하지는 않는다.

### geohot이 걱정하는 “그런 사람” — 역할로 보면

특정인을 찍는 말이 아니라, **반복되는 워크플로**다.

| 패턴 | 하는 일 | 덜 위험한 쪽 |
|------|---------|-------------|
| 슬롯머신 머지 | 80% 만든 뒤 “한 번만 더” | 범위를 닫고, 수동 마무리 시점을 정한다 |
| 10배 아웃풋 KPI | 리뷰 없이 PR만 쏟는다 | 리뷰·CI도 같은 비율로 늘린다 |
| 툴 맥시멀리스트 | 모델·플러그인만 갈아탄다 | [문제를 먼저](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/) 정의한다 |
| 검증 대리인 | “테스트 통과했대요”만 말한다 | diff에서 테스트·skip·주석을 본다 |
| 설교만 하는 쪽 | 도구·프로그램 도입만 말한다 | oracle·실험 한 줄을 먼저 쓴다 |
| geohot이 본 고성과자 | slop를 알아보고 한 줄씩 읽는다 | — |

geohot이 걱정하는 건 **Cursor를 쓰는 모든 개발자**가 아니다. **스스로 검증하지 않는 채 10배를 내는 사람**과, 그걸 **조직 KPI로 밀어 넣는 쪽**이다.

Karpathy 쪽 psychosis는 “과몰입하지만 (대체로) 잘 돌아가는 루프”에 가깝고, geohot이 경계하는 건 “과몰입 + 저성과 slop가 **조직 전체**로 퍼지는 것”이다. 단어는 같아도 결이 다르다. 해결책은 비슷하다 — **멈출 수 있는 구조, 검증, 한 줄씩 읽기.**

---

## 조직에 대한 함의

geohot이 말한 대기업 시나리오는 익숙하다. **10배 아웃풋** KPI는 있는데, 검증 인력·CI·리뷰 깊이는 그대로인 경우. “하네스 프로그램”은 있는데 **이게 맞다**는 oracle 한 줄은 없는 경우.

[Fowler harness 글](https://changbaebang.github.io/2026-05-25-fowler-harness-roles-not-layers/)이 다룬 **계층·중앙·툴 의존**과는 다른 면이다. 거기는 조직도였고, geohot은 **평균 품질**이다. 둘 다 동시에 일어날 수 있다.

말과 도구만 늘고 **slop을 걸러내는 루프**가 없으면, geohot이 말한 “가장 비싼 실수”가 된다.

---

## 실무 체크리스트

에이전트를 돌리기 전에 — [문제 퍼스트 5문장](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)과 같이 쓴다.

1. 이번에 **줄이려는 실패** 한 가지는 무엇인가  
2. **성공 기준**(oracle) 한 줄은 있는가  
3. PR에 **diff만** 있는가, **검증 근거**도 있는가  
4. 테스트가 green인데 **삭제·skip·주석 처리**만 늘지는 않았는가  
5. 10배 아웃풋을 요구한다면, **리뷰·CI**도 10배인가  

다섯 가지 중 두 가지 이상 “아니오”면, geohot이 말한 슬롯머신 구간에 들어가기 쉽다.

---

## 마치며

geohot은 에이전트 **만능 신화**에 맞선다. 나는 **검증 없는 도입**에 맞서고 싶다.

- 에이전트를 쓰지 말자 — 아니다.  
- **검증 없이** 조직 KPI에 넣지 말자 — 그렇다.

마지막 문장은 겁주기처럼 들리지만, 실무로 옮기면 단순하다. **slop를 slop으로 알아보고, 검증 없이 10배를 내지 않으면** 된다. AI를 끄라는 말이 아니라, **판단 없이 밀지 말라**는 말에 가깝다.

**한 줄 결:** 10배 아웃풋보다 먼저 10배 검증을 설계하지 않으면, geohot이 말한 비싼 실수가 팀 예산으로 찾아온다.

---

## 읽을 거리

- [The Eternal Sloptember (geohot)](https://geohot.github.io/blog/jekyll/update/2026/05/24/the-eternal-sloptember.html)  
- [GeekNews #29852](https://news.hada.io/topic?id=29852)  
- [문제 퍼스트 — AI 시대에 툴보다 먼저 정의할 것](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)  
- [테스트와 CI가 진짜 생산성인 이유](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)  
- [Twelve Ways to Be Wrong (큐레이션)](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)  
- [Fowler harness — 계층·중앙·도구 의존](https://changbaebang.github.io/2026-05-25-fowler-harness-roles-not-layers/)  

*특정 회사·특정인을 겨냥하지 않는다. geohot과 커뮤니티 논쟁을 실무 관점에서 각색한 글이다.*
