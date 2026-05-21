---
layout: post
title: "MCP가 메인스트림으로 내려온 주간 — 연결성에서 런타임·안전 운영으로"
date: 2026-05-21 16:00:00 +0900
tags: [mcp, ai, agents, integration, workflow, platform]
---

> "연결은 늘 생산성을 올린다.  
> 동시에 연결은 늘 사고 반경도 키운다."

이번 주 발표들을 보면 MCP류 연결 전략이 더 이상 "실험실 기능"이 아니다.  
에이전트가 여러 도구를 오가며 일을 끝내는 흐름이 제품 기본값으로 내려오고 있다.

## TL;DR

- 이번 주는 "연결 기능" 발표 주간이 아니라 **에이전트 운영 인프라** 발표 주간이다.
- 초점이 프롬프트 품질에서 **런타임 신뢰성 + 안전성 테스트**로 이동했다.
- 오늘 남겨둘 기록은 하나다: "에이전트 시대의 차별점은 모델보다 운영 레이어"라는 점.

## 이 글이 필요한 사람

- 이미 MCP/도구 연동을 쓰고 있는데 운영 기준이 아직 없는 팀
- "어떤 모델이 더 좋나"보다 "어떻게 안전하게 굴리나"가 궁금한 팀
- 에이전트를 파일럿에서 운영 단계로 올리려는 팀

## 용어를 먼저 맞추자

이번 주 발표를 읽을 때 용어가 섞이면 핵심이 흐려진다.  
내가 실무에서 구분하는 최소 기준은 아래다.

- **MCP/연결 레이어**: 에이전트가 외부 도구를 호출하는 인터페이스
- **하네스 레이어**: 계획/도구 선택/실행 루프를 조정하는 오케스트레이션
- **런타임 레이어**: 장시간 실행, 재개, 상태 일관성, 격리 같은 운영 기반
- **안전성 레이어**: 설계 검토, 공격 시나리오 테스트, 회귀 방지 체계

이 네 가지를 분리해서 봐야 "연결을 잘했다"와 "운영 가능하다"를 구분할 수 있다.

## 왜 '이번 주'가 분기점인가

### 1) 에이전트 플랫폼이 연결을 전제로 설계됐다
- Google은 Spark/Antigravity 생태계에서 도구 연동과 장시간 실행을 전면에 뒀다.
- OpenAI workspace agents도 "ChatGPT + Slack + 조직 도구" 연결을 핵심 가치로 제시했다.

참고:
- [Google I/O 2026 keynote](https://blog.google/innovation-and-ai/sundar-pichai-io-2026/)
- [Google developer highlights](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)
- [OpenAI workspace agents](https://openai.com/index/introducing-workspace-agents-in-chatgpt/)

### 2) 단순 질의가 아니라 '업무 루프'를 겨냥한다
- 과거: 답변 생성
- 지금: 검색 -> 판단 -> 도구 실행 -> 결과 보고 -> 후속 작업

이 루프가 가능해지면 개인 생산성뿐 아니라 팀 운영 구조 자체가 바뀐다.

### 3) 런타임과 안전성 도구가 공개됐다 (이번 주 핵심 신호)

- Google은 **Agent Executor**를 공개하며, 장시간 에이전트 워크플로에 필요한 내구성/복구/세션 일관성 같은 런타임 특성을 전면에 올렸다.
- Microsoft는 **RAMPART / Clarity**를 오픈소스로 공개하며, 에이전트 안전성을 "일회성 점검"이 아니라 CI에서 반복 가능한 엔지니어링 작업으로 다루기 시작했다.

참고:
- [Agent Executor (Google Cloud)](https://cloud.google.com/blog/products/ai-machine-learning/agent-executor-googles-distributed-agent-runtime)
- [RAMPART & Clarity (Microsoft Security)](https://www.microsoft.com/en-us/security/blog/2026/05/20/introducing-rampart-and-clarity-open-source-tools-to-bring-safety-into-agent-development-workflow/)

## 그래서 지금 꼭 기록할 가치가 있는 이유

이번 주 신호는 "기능 추가"보다 "운영 패러다임 전환"에 가깝다.

1. **연결의 시대 -> 운영의 시대**  
   연결 자체는 이미 가능한 팀이 많다. 이제는 중단/재개/로그/검증 설계가 차이를 만든다.
2. **성능 비교 -> 실패 비용 관리**  
   모델 벤치마크보다 실제 장애 반경과 복구 시간 관리가 더 중요한 단계로 이동했다.
3. **일회성 점검 -> 지속적 안전성 엔지니어링**  
   안전성은 출시 직전 체크리스트가 아니라 CI에 들어가는 회귀 테스트 자산이 되고 있다.

## 연결성의 기회 (팀 입장)

### 1) 컨텍스트 왕복 비용 감소
Slack 스레드, 티켓, PR, 문서를 사람이 손으로 복사/붙여넣기하지 않아도 된다.

### 2) 반복 업무 자동화의 품질 상승
단순 매크로가 아니라, 조건 판단이 들어간 자동화가 가능해진다.

### 3) 의사결정 리드타임 단축
같은 정보로 회의를 시작할 수 있어 정렬 비용이 줄어든다.

## 연결성의 리스크 (팀 입장)

### 1) 권한 과다 부여
"편해서 일단 연결"이 되면 가장 먼저 터지는 건 과권한 문제다.

### 2) 책임 경계 모호화
누가 실행했는지, 어떤 규칙으로 실행됐는지 흐려지면 복구가 느려진다.

### 3) 잘못된 자동화의 빠른 전파
한 번 잘못 연결된 흐름은 사람보다 빨리, 넓게 퍼진다.

## 실제로 겪게 되는 운영 장면 2개

### 장면 A: 장시간 작업이 중간에 끊긴다

- 증상: 30분 이상 실행되는 에이전트 작업이 네트워크/승인 대기로 끊김
- 문제: 어디까지 처리됐는지 불분명해 재실행 비용 증가
- 필요한 것: 재개 가능한 상태 저장, 이벤트 로그, 체크포인트 전략

### 장면 B: 사고 후 재현이 안 된다

- 증상: "어제는 분명 이상 동작했는데 오늘은 재현되지 않음"
- 문제: 원인 규명과 재발 방지가 개인 기억에 의존
- 필요한 것: 공격/오작동 시나리오를 테스트 케이스로 고정하는 체계

## 그래서 필요한 운영 원칙 4개

1. **최소 권한**: 읽기/쓰기/외부 발신 권한을 분리한다.  
2. **명시적 승인**: 고위험 액션은 사람 승인 없이는 진행하지 않는다.  
3. **실행 로그**: 어떤 입력으로 어떤 도구를 호출했는지 남긴다.  
4. **즉시 롤백**: 연결 해제/권한 회수 경로를 문서화한다.

## 이제 추가로 필요한 운영 원칙 2개

5. **런타임 복구성**: 재시작/중단/재개 시 세션과 상태를 어떻게 이어갈지 미리 정한다.  
6. **안전성 회귀 테스트**: 프롬프트 인젝션/오작동 시나리오를 테스트로 고정해 배포마다 재검증한다.

## 이번 주 바로 할 일

- 팀 MCP 서버 목록을 `must / optional / experimental`로 분류  
- 채널/레포별 자동 실행 금지 액션 5개 정의  
- "에이전트가 해도 되는 일"보다 "하면 안 되는 일"을 먼저 합의  
- 실패 사례 1건을 안전성 테스트 케이스로 변환  
- 장시간 실행 작업의 중단/재개 절차를 1페이지 분량으로 문서화

많은 팀이 자동화 설계에서 "할 수 있는 것"부터 적는다.  
실제로는 "막아야 하는 것"부터 적어야 안전하게 빨라진다.

## 참고 링크 (추가)

- [Google I/O 2026 keynote](https://blog.google/innovation-and-ai/sundar-pichai-io-2026/)
- [Google developer highlights](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)
- [Agent Executor (Google Cloud)](https://cloud.google.com/blog/products/ai-machine-learning/agent-executor-googles-distributed-agent-runtime)
- [Agent Executor GitHub (google/ax)](https://github.com/google/ax)
- [OpenAI workspace agents](https://openai.com/index/introducing-workspace-agents-in-chatgpt/)
- [RAMPART & Clarity (Microsoft Security)](https://www.microsoft.com/en-us/security/blog/2026/05/20/introducing-rampart-and-clarity-open-source-tools-to-bring-safety-into-agent-development-workflow/)
- [RAMPART GitHub](https://github.com/microsoft/RAMPART)
- [Clarity GitHub](https://github.com/microsoft/clarity-agent/)

## 마치며

MCP의 본질은 연결 기술이 아니다.  
이번 주를 기점으로 연결성 논의는 런타임/안전성 논의로 확장됐다.

**조직의 경계 설계 + 복구 설계 + 안전성 테스트 설계**를 코드처럼 운영하는 팀이 결국 오래 빠르다.

**한 줄 결:** *연결성은 곧 생산성이지만, 경계 없는 연결성은 곧 장애 반경이다.*
