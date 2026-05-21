---
layout: post
title: "에이전트는 빨라졌고, 팀은 더 엄격해져야 한다 — 이번 주 발표를 운영 체크리스트로 번역하기"
date: 2026-05-21 11:50:00 +0900
tags: [ai, agents, governance, workflow, mcp, engineering-management]
---

> "이제 성능은 대부분 충분하다.  
> 차이는 **운영 경계와 검증 습관**에서 난다."

이번 주 발표를 보면 공통된 방향이 보인다.  
모델 자체 성능 경쟁은 계속되지만, 실제 현장에서는 "어떻게 안전하게 굴릴 것인가"가 더 큰 이슈가 됐다.

## 이번 주, 무엇이 바뀌었나

### 1) 에이전트가 기본 인터페이스가 됐다
- Google I/O 2026에서 Gemini 3.5 Flash, Antigravity 2.0, Managed Agents가 공개됐다.
- "프롬프트 한 번"이 아니라 "장시간 실행되는 작업 단위"가 표준으로 올라왔다.

참고:
- [Google I/O 2026 키노트](https://blog.google/innovation-and-ai/sundar-pichai-io-2026/)
- [Developer highlights](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)

### 2) 협업 표면이 넓어졌다
- OpenAI는 workspace agents를 소개하며 Slack 같은 협업 표면에서의 실행을 강조했다.
- Codex 모바일/원격 흐름도 나오면서 "자리 앞에서만 개발한다"는 가정이 약해졌다.

참고:
- [Introducing workspace agents in ChatGPT](https://openai.com/index/introducing-workspace-agents-in-chatgpt/)
- [Work with Codex from anywhere](https://openai.com/index/work-with-codex-from-anywhere/)

### 3) 출처 검증이 별도 트랙으로 올라왔다
- OpenAI가 C2PA, SynthID, 검증 도구 프리뷰를 같이 공개했다.
- 생성 품질뿐 아니라 "이 결과물이 어디서 왔는가"가 제품 요구사항이 됐다.

참고:
- [Advancing content provenance](https://openai.com/index/advancing-content-provenance/)

## 왜 지금은 성능보다 운영인가

에이전트가 길게 일할수록, 모델의 똑똑함보다 다음 질문이 먼저 온다.

1. 어디까지 자동 승인할 것인가  
2. 어떤 도구/도메인까지 접근 허용할 것인가  
3. 실패했을 때 누가, 어떤 로그로, 얼마나 빨리 복구할 것인가

이 질문은 모델 벤치마크로 답할 수 없다.  
팀의 운영 정책과 실행 습관으로만 답할 수 있다.

## 팀 운영 체크리스트 (실무 번역본)

### 1) 승인 정책을 위험도 기준으로 분리
- 저위험: read-only 조회, 문서 초안, 테스트 실행
- 중위험: 코드 수정, PR 코멘트, 이슈 생성
- 고위험: 프로덕션 영향 명령, 외부 발신, 권한 변경

핵심은 "모든 승인"이 아니라 "같은 위험은 같은 승인 규칙"이다.

### 2) 실행 경계(샌드박스/네트워크) 명시
- 쓰기 가능한 경로를 좁힌다.
- 허용 도메인과 차단 도메인을 분리한다.
- 경계를 넘는 동작은 승인을 강제한다.

### 3) MCP/툴 권한을 allowlist로 운영
- 연결 가능한 서버 목록을 먼저 정의한다.
- 팀 공용 에이전트에는 최소 권한만 부여한다.
- 개인 실험 권한과 팀 운영 권한을 분리한다.

### 4) 출처/증빙을 결과물에 남긴다
- 이미지/문서 생성물은 provenance 확인 경로를 남긴다.
- 외부 공유 자산은 "생성/편집 이력" 표기 원칙을 둔다.

### 5) 에이전트 로그를 회고 자산으로 남긴다
- 실패 사례를 "개인 실수"로 끝내지 않는다.
- 승인/거부/재시도 패턴을 다음 주 규칙 개선에 반영한다.

## 이번 주에 바로 적용할 3가지

1. PR 템플릿에 `자동 실행 범위 / 수동 승인 지점 / 롤백 경로` 3줄 추가  
2. 팀 MCP allowlist 초안 1페이지 작성  
3. 외부 공유 이미지/자료에 출처 검증 체크 1개 항목 추가

작게 시작해도 충분하다.  
중요한 건 "도구를 잘 쓰는 팀"이 아니라 "실패 비용을 통제하는 팀"이 되는 것이다.

## 마치며

에이전트 시대의 생산성은 속도 하나로 정의되지 않는다.  
**속도 + 통제 + 증빙**이 같이 돌아갈 때만 팀의 실제 생산성이 된다.

**한 줄 결:** *모델은 이미 빠르다. 이제 팀이 빨라질 차례는 운영 설계에서 나온다.*
