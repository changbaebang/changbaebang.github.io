---
layout: post
title: "Harness engineering을 읽었을 때 — 계층·중앙·도구 의존은 왜 틀린가"
date: 2026-05-25 16:30:00 +0900
permalink: /2026-05-25-fowler-harness-roles-not-layers/
tags: [harness, harness-engineering, ai, coding-agent, martin-fowler, engineering, workflow, problem-first]
---

> Fowler가 말하는 harness는 **조직도가 아니다.**  
> **계층으로 쌓거나, 중앙이 독점하거나, 모델·툴에 매달리면** — 그 순간 harness engineering이 아니라 **이름만 남는다.**

[Birgitta Böckeler의 Harness engineering](https://martinfowler.com/articles/harness-engineering.html)(Martin Fowler 사이트)을 읽었다. coding agent 앞의 **신뢰 장벽**을 낮추는 **guide · sensor · human steering** — 이 프레임은 실무와 잘 맞는다.

동시에, 그 글이 유행한 뒤 조직에서 보이는 움직임 가운데 **글과 반대로 가는 패턴**도 분명했다. 은탄환을 논할 필요까지는 없다. **제어 구조가 잘못됐을 때** Fowler의 논리로 “이건 harness engineering이 아니다”라고 말할 수 있어야 한다.

이 글은 Fowler 글을 짧게 정리하고, 그 기준으로 **세 가지 잘못된 패턴**을 짚는다.

1. harness가 **계층**으로 정리될 때  
2. harness가 **중앙**에 모일 때  
3. harness가 **특정 모델·툴**에 묶일 때  

---

## TL;DR

- Fowler: coding agent **바깥 outer harness** — guide(피드포워드) + sensor(피드백), 인간은 **steering**.
- **계층화** — “토대 없으면 위층 노이즈”가 **진척 착시**가 되고 behaviour 검증을 뒤로 미룬다.
- **중앙집중** — 플랫폼이 harness **전부**를 소유하면 팀의 steering이 죽는다.
- **모델·툴 의존** — inferential만 늘리거나 벤더 harness에 매달리면 **computational sensor**와 **문제 정의**가 비어 있다.
- 건강한 신호: **이번 실패 하나** + guide/sensor 역할 + oracle 한 줄 + 인간 판단 위치.

---

## Fowler가 말하는 harness

LLM은 비결정적이고 맥락을 모른다. 코딩 에이전트 사용자에게 **신뢰 장벽(trust barrier)** 은 자연스럽다.

Fowler는 harness를 **coding agent** 맥락으로 좁힌다. 넓게는 Agent = Model + Harness 이지만, 우리가 설계하는 것은 **outer harness**다.

| 역할 | 시점 | 예 |
|------|------|-----|
| **Guide** | 행동 전 | AGENTS.md, Skills, bootstrap |
| **Sensor** | 행동 후 | lint, test, ArchUnit, (신중히) AI 리뷰 |
| **Human** | 같은 실패가 반복될 때 | harness **steering** — guide/sensor를 고쳐 재발을 줄임 |

**Computational** sensor(lint, type, fast test)는 매 변경에 싸게 돌리고, **inferential**(LLM judge)은 비싸서 보완에 쓴다. **언제** 도는지는 커밋 전 / 파이프라인 / 상시 드리프트 — 이건 **층이 아니라 시간축**이다.

**무엇을** 맞추는지는 maintainability, architecture fitness, **behaviour** — 역시 **층이 아니라 축**이다.

Fowler가 분명히 말하는 두 가지:

> 좋은 harness는 인간 입력을 **없애는** 게 아니라, **가장 중요한 곳으로 보내는** 것이다.

> Behaviour harness — AI가 만든 테스트만으로는 **아직 부족**하다.

아래 세 패턴은 이 문장들에 **반한다**.

---

## 잘못된 패턴 1: 계층화

조직에서 harness를 **스택**으로 그린다. 맥락 층, SSOT 층, 입력 층, 출력·점수·샘플러 층 — 라벨은 다르지만 그림은 같다. “아래가 없으면 위는 노이즈”까지 붙는다.

**맥락 없이 게이트만 두지 말라**는 방향은 Fowler와 맞다.  
**문제는 계층을 조직의 기본 언어로 쓰는 순간**이다.

Fowler의 사고도는 위→아래 **층**이 아니라, **한 번의 변경**을 도는 루프다.

```text
의도(oracle) → Guide → Agent → Sensor → (같은 실패 반복) → Human steering → Guide …
```

- **의도** — behaviour에서 “맞다”의 기준  
- **Guide / Sensor** — 같은 루프 안의 **역할**  
- **Human** — 층 위 승인자가 아니라 **루프를 고치는 사람**

계층화가 틀리게 느껴지는 이유:

1. **진척이 층 번호로 바뀐다** — “위층 올렸다”가 곧 성숙도처럼 보인다.  
2. **행동 검증이 숨는다** — maintainability sensor만으로 “하네스 완료”라고 말하기 쉽다. Fowler가 가장 어렵다고 한 **behaviour**는 회의 표에 없다.  
3. **실무 질문이 망가진다** — “지금 몇 층인가?”는 답이 없다. “이번 PR에 guide가 비었는가, sensor가 비었는가?”가 답이 있다.

Fowler 글에 OpenAI의 **layered architecture** 사례가 나온다. 그걸 **“우리도 층부터 쌓아야 한다”**는 로드맵으로 바꾸는 건, **steering loop**(반복 실패 → guide/sensor 수정)가 아니라 **건축 프로그램**이다.

**Fowler 기준:** harness engineering은 **변경 한 건마다 돌아가는 제어 루프**다. 계층이 먼저이면 harness가 아니라 **조직도 정리**에 가깝다.

---

## 잘못된 패턴 2: 중앙집중

harness **소유권**이 중앙으로 몰릴 때다.

- 플랫폼·TF·“하네스 담당”이 **표준 스택 전체**를 정의한다.  
- “입력은 자유, 출력·판정은 공통”이라 말하지만, 팀 일정은 **중앙 로드맵 맞추기**로 채워진다.  
- 점수·게이트·샘플러·공통 PoC가 **먼저** 오고, 팀의 핵심 흐름·oracle은 **나중**에 온다.

Fowler는 **harness template**(서비스 토폴로지별 guide/sensor 묶음)을 말한다. 성숙한 조직은 이미 서비스 템플릿이 있다. Fowler도 **버전·동기화·기여** 문제를 짚는다 — 템플릿은 **출발점**이지 **중앙 통제의 끝**이 아니다.

| 중앙집중에서 흔한 일 | Fowler 쪽 |
|---------------------|-----------|
| 분기 로드맵에 도구 N개 | **같은 실패가 여러 번**일 때만 harness 수정 |
| “아직 아래 층”이라며 대기 | 팀이 **지금** 쓰는 guide/sensor부터 steering |
| 공통 점수가 목표 | 점수는 보조이고, **oracle**이 behaviour를 잡는다 |
| 중앙이 도메인 의도까지 씀 | **조직 정렬·trade-off**는 인간 harness — 완전 위임 불가 |

현장에서도 보인다. **출력 쪽 검증**(공통 워크플로·PoC)과 **입력 쪽 구현 라인**(팀 에이전트·스킬)을 나눠 말하려 해도, 대화는 금방 **“회사 하네스 프로그램” 한 줄**로 접힌다. 그때 팀의 일은 **steering**이 아니라 **대기**가 된다.

**Fowler 기준:** harness는 **분산된 steering**이다. 중앙은 템플릿·공통 sensor **후보**를 줄 수 있지만, **의도·trade-off·behaviour oracle**을 대신 잡으면 harness engineering이 아니라 **플랫폼 통제**다.

---

## 잘못된 패턴 3: 모델·툴 의존

harness 논의가 **문제·역할**이 아니라 **모델·제품**으로 붙을 때다.

프론트엔드 채널에서 이런 말이 오갔다 — 인물은 빼고 **패턴만** 옮긴다.

> 할루시네이션 제어도 harness, 오케스트레이션도 harness, AGENTS.md도 harness.  
> Claude Code·Cursor·Codex **연결 서비스**도 harness — LLM 빼고 전부.

Fowler도 넓은 정의(모델 제외 전부)를 **인정**한다. 그래서 비유의 한계를 스스로 말한다.  
조직이 그 넓은 정의를 **기본 언어**로 쓰면 harness는 **우산어**가 되고, 다음 질문은 “어떤 **툴**을 더 달까”가 된다.

[문제 퍼스트](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)와 같다. **Codex vs Claude**가 먼저 나오고 **무엇이 맞는지**는 나중에 나온다.

| 툴·모델 의존 | Fowler와 어긋남 |
|--------------|------------------|
| Inferential만 늘림 | computational sensor를 **매 변경**에 싸게 돌려야 함 |
| 벤더 harness = 우리 harness | outer harness는 **우리 실패**에 맞게 steering |
| 도구는 있는데 oracle 없음 | correctness는 human이 **원하는 것**을 말하지 않으면 sensor 밖 |

**Fowler 기준:** harness engineering은 **모델 쇼핑**이 아니다. **guide/sensor 시스템**을 키우는 일이고, 비싼 inferential은 computational이 잡은 뒤 보완한다. 툴 이름이 먼저면 harness가 아니라 **구매·정치**에 가깝다.

---

## 세 패턴이 겹칠 때

보통 동시에 온다.

- **계층**으로 “아직 아래 층”이라며 팀 작업을 멈춘다.  
- **중앙**이 위층 도구·점수·PoC 로드맵을 밀어 넣는다.  
- **툴** 비교가 회의 시간을 잡아먹는다.

그러면 **E2E·시각 회귀·샘플 타겟**은 있는데 **“이게 맞다”는 oracle**은 없다. PR에는 diff만 있고 **승인 fixture**는 없다.  
Fowler가 **behaviour harness**라고 부르는 구간이 비어 있다. 층이 낮아서가 아니다.

[테스트·CI](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/) 글과도 같다 — 생성은 빠르고 **완료 정의**가 없으면 harness가 많을수록 **속도만 난다**.

---

## Fowler 기준으로 건강한 harness

| 신호 | 의미 |
|------|------|
| 줄이는 **실패 한 가지**가 문장으로 있다 | 우산어가 아님 |
| **Guide**인지 **Sensor**인지 말할 수 있다 | 역할 분명 |
| **커밋 전 / CI / 드리프트** 중 어디인지 안다 | 층이 아니라 시간 |
| behaviour **oracle 한 줄** | 점수·샘플러보다 우선 |
| 인간 검토 **위치**가 분명 | steering이 살아 있음 |

**착수 전 5문장** ([problem-first](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)와 짝):

1. 이번에 줄이는 실패 한 가지: `___`  
2. Guide인가 Sensor인가: `___`  
3. 언제 도는가 (커밋 전 / CI / 상시): `___`  
4. behaviour oracle 한 줄: `___`  
5. 인간이 판단할 지점: `___`  

다섯 줄이 비면 — 층 번호, 중앙 로드맵, 툴 이름만 있는 상태다.

**좁히는 한 문장:**

> 이번 변경의 harness는 “모델 제외 전부”가 아니라,  
> **이 실패를 한 번 덜 내는 guide 또는 sensor 하나**다.

---

## 바로 적용 체크리스트

- [ ] “하네스 도입” 대신 **실패 한 가지**부터 쓴다  
- [ ] 로드맵에 도구를 더할 때 **Guide/Sensor/시간**을 적는다  
- [ ] 중앙 항목은 **템플릿·후보**인지, **팀 steering 대체**인지 구분한다  
- [ ] 회의에서 **모델·벤더**보다 **oracle 한 줄**을 먼저 고정한다  
- [ ] behaviour가 비어 있으면 **위층 도구 추가를 hold**한다  

---

## 읽을 거리

- [Harness engineering (Fowler / Böckeler)](https://martinfowler.com/articles/harness-engineering.html) — 기준 문서  
- [문제 퍼스트 — AI 시대에 툴보다 먼저 정의할 것](https://changbaebang.github.io/2026-05-25-problem-first-not-tool-first/)  
- [테스트와 CI가 진짜 생산성인 이유](https://changbaebang.github.io/2026-05-24-test-ci-real-productivity/)  
- [「하네스 다 했어요」는 아직 시작](https://changbaebang.github.io/2026-05-24-claude-code-harness-living-docs/)  

---

## 마치며

Harness engineering은 **은탄환이 아니다.** 신뢰 장벽을 낮추는 **제어 설계**다.

Fowler를 읽은 뒤에도 조직이 **계층·중앙·툴** 쪽으로 가면, 글을 따른 게 아니라 **이름만 가져온 것**이다.

- **계층**으로 쌓지 말고 **루프**를 돌려라.  
- **중앙**이 판단을 대신하지 말고 **steering**을 팀에 남겨라.  
- **모델·툴**보다 **oracle·computational sensor**를 먼저 채워라.

**한 줄 결:** *Harness가 계층·중앙·벤더에 묶이면, Fowler가 말한 harness engineering이 아니다.*

*구조와 패턴에 대한 글이다. 특정인을 겨냥하지 않는다.*
