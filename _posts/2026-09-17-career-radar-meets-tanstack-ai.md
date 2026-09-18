---
layout: post
title: "OpenAI에서 OpenRouter로, 그리고 TanStack AI로 — Career Radar가 다음에 갈 자리"
date: 2026-09-18 15:21:00 +0900
categories: [Engineering, AI]
tags: [Career Radar, TanStack AI, Open Source, MCP, AI]
excerpt: "OpenAI 크레딧 없이 시작해 OpenRouter 무료 모델로 검증을 끝냈고, 제공자를 더 늘리려다 멈춘 자리에서 TanStack AI를 만났다. 그 저장소의 어댑터 계층이 내가 직접 만든 계층과 같은 자리였고, 거기에 없는 제공자 둘은 내가 조사해 둔 것이었다. 힘이 빠졌다고 쓴 지 하루 만에 다음 자리가 보인 이야기다."
---

Career Radar는 OpenAI로 시작했다. 첫 마일스톤의 유일한 실제 호출은 크레딧 오류로 끝났고, 그 뒤로 한동안 합성 입력으로만 검증했다. 크레딧 없이 실제 모델 경로를 보려고 찾은 것이 OpenRouter였다. 무료 모델 넷 중 하나가 답했고, 하루 50회 한도에 잘린 실행을 나흘에 걸쳐 이어 붙여 기준선을 만들었다. 다음은 제공자를 늘리는 것이었다. Upstage와 Liner가 가입 크레딧을 주니, OpenAI 호환 공통 어댑터 하나만 있으면 하루에 전체를 돌릴 수 있었다. 그런데 [회고](https://changbaebang.github.io/2026-09-17-career-radar-retrospective/)에서 만들지 않기로 했다. 앱이 일반 채팅보다 낫다는 확신이 없었고, 솔직히 힘도 빠져 있었다.

그 주에 다른 창이 하나 열려 있었다. [TanStack AI](https://github.com/TanStack/ai)에 낸 PR 두 개. 하나는 메모리 어댑터의 계약 테스트 묶음을 패키지 서브패스로 내보내는 것으로 [병합됐고](https://github.com/TanStack/ai/pull/1388), 하나는 문서의 import 경로 한 줄로 [열려 있다](https://github.com/TanStack/ai/pull/1415). 둘 다 작다. 그런데 병합 알림을 받고 그 저장소를 다시 읽다가, 내 앱과 겹치는 자리가 생각보다 정확하다는 것을 알았다. 만들지 않기로 한 공통 어댑터가 거기서는 함수 하나였고, 거기에 없는 제공자 둘이 내가 막 조사해 둔 것이었다. 이 글은 그 겹침과, 하루 만에 달라진 마음에 대한 것이다.

## TanStack AI가 무엇인가

저장소의 한 줄 설명은 이렇다. 타입 안전하고 제공자에 무관한 TypeScript AI SDK로, 스트리밍 채팅·도구 호출·에이전트·멀티모달을 OpenAI·Anthropic·Gemini와 React·Vue·Svelte·Solid에서 쓴다. 패키지가 예순 개 남짓이다. 제공자 어댑터(`ai-openai`, `ai-anthropic`, `ai-gemini`, `ai-openrouter`, `ai-groq`, `ai-ollama` 등), 프레임워크 바인딩과 UI, 호스트 측 MCP 클라이언트(`ai-mcp`), 메모리(`ai-memory`), 샌드박스 여러 종, devtools. README에는 2026년 JS 오픈소스 어워드의 AI 프로젝트 배지가 있고, 에이전트용 스킬을 플러그인으로 설치하는 안내가 첫 화면에 있다.

내 앱과 닿는 부분은 셋이다. `chat({ outputSchema })`로 [구조화 출력](https://tanstack.com/ai/latest/docs/structured-outputs/one-shot)을 받으면 반환 타입이 스키마에서 따라온다. [`openaiCompatible({ baseURL, apiKey, models })`](https://tanstack.com/ai/latest/docs/adapters/openai-compatible)는 OpenAI Chat Completions 형식을 말하는 어떤 제공자든 붙이고, 모델마다 `createModel(name, { features: ["reasoning", "structured_outputs"] })`로 기능을 타입에 선언한다. 그리고 [내장 미들웨어](https://tanstack.com/ai/latest/docs/advanced/built-in-middleware)에 OpenTelemetry 스팬과 GenAI 메트릭을 내는 `otelMiddleware`, 스트림 텍스트를 가리는 `contentGuardMiddleware`가 있다.

## 겹치는 자리

Career Radar의 `server/src/ai/`는 네 파일이다. OpenAI Responses 어댑터, OpenRouter chat/completions 어댑터, 제공자 선택, 호출별 텔레메트리. 열흘 동안 리뷰가 잡은 결함의 절반 가까이가 이 네 파일에서 나왔다. SDK 타임아웃이 본문을 못 막는 것, 제공자 이름이 무료를 보장하지 않는 것, 오류 본문이 응답에 새는 것, 상위 엔드포인트를 기록하지 않던 것.

| 내가 직접 만든 것 | TanStack AI |
| --- | --- |
| OpenAI·OpenRouter 어댑터 둘 | `ai-openai`, `ai-openrouter` |
| 엄격한 JSON 스키마 생성 계약 | `chat({ outputSchema })` |
| 만들려다 멈춘 Upstage·Liner 공통 어댑터 | `openaiCompatible(...)` 한 줄 |
| 호출별 텔레메트리(토큰·finish·상위 제공자) | `otelMiddleware`와 미들웨어 체인 |
| 실패 분류(스키마·거절·잘림·타임아웃) | 있는지, 얼마나 나뉘는지 확인 필요 |

겹치지 않는 것도 분명하다. Career Radar는 MCP **서버**인데 `ai-mcp`는 호스트 측 **클라이언트**라 방향이 반대다. 결정적 정책, 인용 검증기, 골든셋 러너, 실행 흔적은 SDK가 할 일이 아니고 앱의 것이다.

## 넣을까, 붙일까, 만들까

방향은 셋이다. 하나는 Career Radar를 저쪽에 넣는 것. 예제 앱으로, "MCP 서버가 구조화 출력을 받아 결정적 정책을 통과시키는" 형태의 실물로. 이건 내가 정할 수 없고 메인테이너의 의향이 먼저다. 다른 하나는 저쪽을 이쪽에 붙이는 것. `server/src/ai/` 네 파일을 SDK 호출로 바꾸고, 생성 계약과 정책과 검증기는 그대로 두는 것.

두 번째가 먼저인 이유가 있다. 회고에서 "OpenRouter 다음에 Upstage와 Liner로 검증하려면 OpenAI 호환 공통 어댑터가 필요한데 만들지 않기로 했다"고 썼다. 그 어댑터가 저쪽에서는 이미 함수 하나다. 내가 멈춘 자리가 저쪽에서는 출발선이다. 그리고 [무료 모델 편](https://changbaebang.github.io/2026-09-14-openrouter-free-models-lessons/)에 적은 여덟 가지 중 몇은 그대로 확인 항목이 된다. 호출별 abort signal이 응답 본문까지 끊는지. OpenRouter 어댑터가 실제로 답한 상위 엔드포인트를 결과에 남기는지. 제공자 라우팅 옵션을 넘길 수 있는지. `finish_reason`이 `length`인 잘림과 스키마 불일치를 구분하는지. openai-compatible 문서에 미지원 파라미터를 400으로 거절하는 제공자가 있다는 주의가 있는지. 각각 "지원된다", "문서가 없다", "안 된다" 중 하나로 갈릴 것이고, 뒤의 둘은 그대로 이슈나 PR이다.

그리고 세 번째 갈래가 있다. 저쪽에 없는 것을 만들어 넣는 것. 어댑터 패키지가 예순 개 남짓인데 Upstage도 Liner도 없다. 커뮤니티 어댑터 목록에도 없다. 저장소에는 [커뮤니티 어댑터 가이드](https://tanstack.com/ai/latest/docs/community-adapters/guide)가 있어 내 이름으로 배포하고 문서 한 장을 올리는 길이 열려 있고, 저장소 안의 어댑터로 가려면 `ai-groq`가 그대로 본이 된다. OpenAI 호환 base 클래스를 상속해 이름과 base URL을 주고, 모델마다 문맥 길이와 출력 상한과 가격과 기능을 메타데이터로 적고, 제공자별 옵션을 타입으로 선언하는 소스 열 개 남짓. Groq 어댑터에는 usage가 표준 위치가 아닌 곳에 실려 오는 것을 표준 위치로 옮겨 주는 주석이 있는데, 그런 quirk를 찾는 것이 Career Radar가 할 일이다. Upstage의 문맥 512K, 출력 128K, strict JSON 스키마, 추론 강도 옵션, 가격은 이미 조사해 두었고 가입 크레딧이 검증 예산이다. 이쪽이 셋 중 가장 작고, 가장 분명하게 남의 저장소에 남는다.

회고에 힘이 빠졌다고 썼다. 보여 주고 싶었는데 봐주는 사람이 없었다. 그런데 이 방향은 동기가 다르다. 앱은 증명의 대상이 아니라 재현 도구가 되고, 부딪힌 것은 내 저장소의 기록이 아니라 남의 저장소의 확인 항목이 된다. 병합된 PR 하나가 알려 준 것은 그쪽에는 읽는 사람이 있다는 것이었고, 그 사실 하나로 다시 힘이 났다. 열흘 동안 만든 것이 쓸모없어진 게 아니라 쓸 자리를 찾은 것이었다.

## 아직 정하지 않았다

결론은 없다. 정한 것은 순서뿐이다. 먼저 OpenRouter 어댑터 하나만 SDK로 갈아 끼우는 spike를 브랜치에서 돌린다. 골든셋 dry run이 그대로 통과해야 하고, 무료 모델로 세 건만 실제로 돌려 위 다섯 항목을 판별한다. 그 결과가 이슈 목록이 되면 붙이는 쪽으로 가고, 그 다음이 Upstage 어댑터다. 처음부터 저장소 안으로 갈지 커뮤니티 어댑터로 시작할지는 이슈 하나로 먼저 묻는다. 예제로 넣는 이야기는 그 뒤다. 걸리는 것이 없으면 그것대로 짧은 글 하나로 끝난다.
