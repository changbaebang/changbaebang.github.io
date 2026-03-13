---
layout: post
title: "리뷰는 자동화하고, 판단은 사람이 한다"
date:   2026-03-13 12:00:00 +0900
author: Changbae Bang
tags: [cursor, automation, review, workflow, webstorm, mcp]
---

# 결론부터 말하면
PR 리뷰는 자동화할수록 팀의 피로가 줄어든다.  
그리고 자동화가 늘어날수록 오히려 사람이 더 중요해진다.  
이 말이 약간 모순 같지만, 실제로는 그렇다.

자동화가 가져가는 건 반복이고, 사람이 해야 하는 건 판단이다.  
나는 이걸 직접 체감했다.  
PR을 올리면 본문이 먼저 정리되고, 기본 리뷰가 붙고, 꼼꼼 리뷰가 한 번 더 지나가고, 마지막에 팀 요청까지 이어지는 흐름을 만들었다.  
예전에는 리뷰를 “열심히” 했고, 지금은 리뷰를 “지속 가능하게” 한다.

최근 [Cursor의 JetBrains ACP 지원](https://cursor.com/ko/blog/jetbrains-acp)을 보면서 확신이 더 강해졌다.  
도구 선택의 문제라기보다, 결국 팀의 리뷰 체계를 자동화 가능한 형태로 설계할 수 있느냐가 핵심이라는 쪽으로.

# 내가 만든 흐름 (식별자 제거 버전)
아래는 실제 운영 중인 플로우를 식별자 없이 정리한 것이다.  
채널 ID, 사용자명, 저장소명, 조직명은 모두 제거했다.

```text
0) 오토메이션 목록 조회
   - ownerName 기준으로 내 워크플로우만 필터링

1) PR 오픈 트리거
   - PR 본문 메타데이터/섹션 자동 정리
   - Jira 키, 티어, 영향도, 테스트 근거를 표준 형태로 맞춤

2) 기본 리뷰 트리거
   - TypeScript must-not 규칙
   - React/Next 컨텍스트 규칙
   - PR 템플릿 체크리스트 규칙

3) 꼼꼼 리뷰 트리거
   - Hygiene(의존성/락파일/내보내기 경계/디버그 잔여물)
   - 심각도 중심(릴리즈 블로킹 리스크) 탐지

4) 알림 트리거
   - 팀 리뷰 요청 메시지 자동 생성/전송
   - 본문의 "변경 사항" 섹션만 추출하여 노이즈 최소화
```

이걸 한 줄로 줄이면 이렇다.  
**“PR은 자동으로 정리되고, 리뷰는 계층적으로 쌓이고, 마지막에 사람에게 필요한 정보만 도착한다.”**

# 내가 만든 9개 자동화 (owner 기준)
솔직히 이 파트는 조금 자랑하고 싶다.  
오토메이션 목록에서 내 owner로 필터링하면, 지금 실제로 굴리는 워크플로우가 9개다.

```text
1) 슬랙 리뷰 요청 대응
2) 리뷰 요청 알림 전송
3) TypeScript 기본 리뷰
4) Next.js 기본 리뷰
5) React 기본 리뷰
6) PR Body 자동 정리
7) Hygiene 리뷰 (의존성/락파일/경계/잔여물)
8) PR 템플릿 체크리스트 리뷰
9) 심각도 중심 리뷰 (release-blocking 탐지)
```

여기서 내가 제일 좋아하는 점은 “역할 분리”다.  
하나의 거대한 프롬프트로 모든 걸 하려면 결국 흐려진다.  
반대로 자동화를 잘게 나누면, 각 리뷰가 자기 역할에만 집중한다.

예를 들면:
- PR Body 자동 정리는 문서 품질과 메타데이터 일관성에만 집중
- 기본 리뷰는 must-not과 브랜치별 엄격도에 집중
- Hygiene는 당장 안 터지지만 나중에 비용 큰 지점을 먼저 탐지
- 심각도 리뷰는 정말 막아야 할 리스크만 본다

이 분리가 좋은 이유는 명확하다.  
리뷰가 길어져도 논점이 안 섞이고, 코멘트의 책임 소재도 선명해진다.

# 내 프롬프트 철학 (조금은 자랑)
프롬프트를 길게 쓰는 게 목적은 아니다.  
하지만 현장에서 쓰려면 애매하면 안 된다.

그래서 나는 아래 네 가지를 고정해 둔다.

## 1) Must NOT 먼저
“하면 좋은 것”보다 “하면 안 되는 것”을 먼저 적는다.  
이렇게 해야 리뷰가 취향 싸움으로 안 흐르고 리스크 중심으로 정리된다.

## 2) Branch-aware
본선과 비본선을 분리해서 강도를 다르게 준다.  
운영 현실을 반영해야 룰이 오래 산다.

## 3) Output format 강제
Decision, Checklist, Required Actions, Draft를 강제한다.  
이렇게 하면 코멘트가 바로 실행 가능한 작업으로 떨어진다.

## 4) 조치 가능성 우선
“좋은 말”보다 “다음 행동”이 보이게 쓴다.  
그래야 자동화가 팀 생산성으로 연결된다.

개인적으로 이 지점이 제일 뿌듯하다.  
프롬프트가 그냥 문장이 아니라, 팀의 합의된 운영 방식이 되었기 때문이다.

# 실제 프롬프트 예시 (비식별 발췌)
아래는 내가 실제로 운영 중인 프롬프트에서 핵심만 발췌한 예시다.  
원문은 길고 촘촘하지만, 핵심 구조는 생각보다 명확하다.

## 예시 1) 슬랙 트리거 리뷰 대응
```md
You are a PR reviewer triggered by a Slack message.
Write all review comments in Korean.

Part A: Trigger handling
1) Parse all @mentions.
2) If owner mention is absent => "not target, no action" and stop.
3) Add only in-progress reaction to triggering Slack message.
4) Extract PR URL: https://github.com/{org}/{repo}/pull/{number}
5) If repo mismatch => no action and stop.

Part B: Code review policy
- Must NOT violations => request changes
- Branch-aware strictness
- Output must include:
  - Decision
  - Checklist Result
  - Required Actions
  - Review Comment Draft
- Post review result only on PR (no Slack summary/thread reply)

Part C: Completion handling
- Remove in-progress reaction
- Add done reaction
- If integration can't react automatically, print reaction instructions.
```

이 프롬프트가 좋았던 이유는, “입구 조건”을 먼저 고정했기 때문이다.  
맨션/링크/리포가 맞지 않으면 바로 종료되니, 불필요한 리뷰 실행이 크게 줄었다.

## 예시 2) PR Body 자동 정리
```md
Goal:
- Update PR body metadata/documentation sections only.
- Do NOT perform code review decisioning.
- Always apply concrete section patches in this run.

Scope:
1) 참조 문서
2) Tier (등급 + 근거)
3) 영향도 분석
4) 테스트/검증
5) 변경 사항

Rules:
- idempotent 해야 함
- 중복 bullet 제거
- Jira key는 절대 truncate 금지
- 실패 시 exact reason + remediation 출력

Execution Contract:
- suggestion-only 금지, 반드시 same run에서 apply 시도
- 특정 API flow로 GET -> build -> PATCH 수행
- unrelated section 보존, 기존 체크리스트/댓글 블록 손상 금지
```

이게 특히 좋았던 건 “실행 계약”을 걸어둔 점이다.  
정리 예정안만 내고 끝나는 게 아니라, 같은 런에서 실제 반영까지 가도록 강제했다.

## 예시 3) 기본 리뷰 (TypeScript/React/Next)
```md
When reviewing code, treat the following as must NOT:
- any 사용
- as unknown as T 같은 이중 단언
- 런타임 위험 non-null assertion
- hooks rule 위반
- stale closure 유발 deps 누락
- setState during render without guard
- critical flow의 명백한 로직 결함

Decision:
- production-bound: must NOT => REQUEST_CHANGES
- non-production-bound: WIP/mock 완화 가능 (단 후속 계획 필수)

Evidence Rule:
- 각 지적에는 파일 경로 / 사용자 영향 / blocking 여부 근거 포함
```

이 규칙의 장점은 리뷰 온도를 일정하게 만든다는 점이다.  
누가 리뷰해도 기준이 크게 흔들리지 않는다.

## 예시 4) Hygiene + 심각도 분리
```md
Hygiene bot:
- dependency/lockfile/export boundary/debug residue 점검
- MUST_FIX / SHOULD_FIX / NIT 분류
- speculative concern으로 blocking 금지

Severity bot:
- release-blocking risk only
- no critical bugs found 라는 명시적 결론 요구
- speculative concern은 blocking 금지
- 범위: data loss, crash, auth bypass, major user-flow breakage
```

이 분리가 좋아서 지금까지 살아남았다.  
유지보수 위생과 치명 리스크를 한 프롬프트에 섞으면 결국 둘 다 흐려진다.

## 예시 5) 리뷰 요청 알림 메시지
```md
Goal:
- Generate concise team notification from PR metadata.

Extraction:
- PR body의 "변경 사항" 섹션만 추출
- 노이즈(HTML 주석, 과다 개행, 템플릿 잔여) 제거
- 최대 6줄 요약
- 섹션이 없으면 fallback 문구 사용

Safety:
- 민감정보 출력 금지
- 자동화 내부 로그/에러 스택 노출 금지

Delivery:
- 전송 성공 시 성공 신호만 반환
- 본문 재출력 금지, 중복 전송 방지
```

이게 은근히 강력하다.  
리뷰 요청 메시지의 품질이 일정해지면, 팀의 응답 속도도 같이 안정된다.

# 왜 여기까지 하게 됐는가
솔직히 말하면 시작은 거창하지 않았다.  
반복 작업이 너무 지겨웠다.

PR 본문 매번 손으로 정리하고, 체크리스트 확인하고, 리뷰 요청 메시지 다듬고, 다시 링크 붙이고.  
이걸 몇 달 하다 보니, 코드보다 주변 작업에 에너지를 더 쓰는 날이 많아졌다.  
어떤 날은 코드보다 문서 정리에 더 오래 걸렸다.

이게 쌓이면 묘하게 마음이 가난해진다.  
리뷰를 대충 하자는 마음은 아닌데, 사람은 체력이 떨어지면 정밀도가 떨어진다.  
그래서 자동화를 붙였다.  
“잘하는 사람”이 아니라 “지치지 않는 흐름”을 만들고 싶었다.

# 룰을 어떻게 만들었는가
핵심은 룰을 길게 쓰는 게 아니고, 판단 기준을 흔들리지 않게 쓰는 것이다.

## 1) Must NOT를 먼저 고정
처음에는 “좋은 코드”를 정의하려고 했다.  
근데 이건 팀마다 다르고, 상황마다 바뀐다.

대신 “절대 들어오면 안 되는 것”을 먼저 고정했다.

- `any`, 이중 단언(`as unknown as T`) 같은 타입 우회
- 런타임 리스크가 있는 non-null assertion
- 훅 규칙 위반, stale closure 유발 패턴
- 디버그 잔여물, 고위험 오타, 명백한 크리티컬 플로우 버그

예를 들면 이런 룰이다.

```md
## TypeScript Must NOT
- Do NOT use any
- Do NOT use as unknown as T
- Do NOT use non-null assertion when runtime null is possible

## React Hooks Must NOT
- Do NOT call hooks conditionally
- Do NOT omit required deps causing stale closures
```

이게 좋은 이유는 단순하다.  
리뷰어의 취향을 묻지 않고, 위험을 기준으로 바로 대화할 수 있다.  
“이건 느낌상 별로”가 아니라 “이건 룰 위반이라 런타임 리스크가 있음”으로 정리된다.

이렇게 바꾸니까 리뷰가 훨씬 빨라졌다.  
취향이 아니라 리스크 중심으로 얘기하게 되기 때문이다.

## 2) 브랜치별 강도 차등
모든 브랜치를 동일하게 잡으면 팀이 피곤해진다.  
반대로 너무 느슨하면 본선에서 터진다.

그래서 production-bound와 non-production을 나눴다.

- 본선 계열: must-not 위반은 수정 요청
- 비본선 계열: WIP/mock은 조건부 완화 가능  
  (대신 후속 이슈, 담당자, 반영 시점 명시)

실제 운영할 때는 이렇게 써두면 갈등이 줄어든다.

```md
## Branch-aware strictness
- Production-bound (main/master/release/*): Must NOT 위반 => REQUEST_CHANGES
- Non-production-bound: WIP/mock 이슈 => COMMENT 가능
  단, follow-up issue / owner / target date 필수
```

이 룰이 특히 좋았던 건 “사람 따라 다르게 보이는 문제”를 줄여준 점이다.  
같은 코드여도 상황에 맞는 강도로 처리하니, 팀이 리뷰 결과를 더 납득한다.

이 룰 하나만으로 “왜 이건 코멘트이고, 저건 수정 요청인가”가 설명 가능해졌다.

## 3) 출력 형식을 강제
사람이 읽는 리뷰는 형식이 반이다.  
좋은 지적도 형식이 흐트러지면 액션으로 안 이어진다.

그래서 리뷰 출력은 강제했다.

- Decision (APPROVE / COMMENT / REQUEST_CHANGES)
- Hard/Soft 체크리스트 결과
- Required Actions
- PR에 바로 붙일 코멘트 초안

예시 포맷은 대략 이런 식이다.

```md
## Decision
- COMMENT

## Checklist Result
- [Hard] 중요 플로우 정확성: PASS - 핵심 경로 확인됨
- [Hard] 디버그 잔여물 없음: FAIL - console.log 1건

## Required Actions
- console.log 제거
- 후속 이슈: FE-1234 / owner: @owner / due: before production merge

## Review Comment Draft
- 디버그 로그 제거 후 다시 확인 부탁드립니다.
```

이 포맷이 좋은 이유는, 지적이 액션으로 바로 연결되기 때문이다.  
읽는 사람이 “그래서 뭘 해야 하지?”를 고민하지 않아도 된다.

이 구조를 강제한 뒤부터 “그래서 뭘 고치면 되죠?”라는 질문이 줄었다.

# 식별자 없이 공유 가능한 룰 예시
아래는 실제 룰의 구조를 익명화한 샘플이다.

```md
# Trigger
- PR opened
- PR commented with 특정 키워드

# Scope
- 특정 owner의 워크플로우만 사용
- 특정 저장소/브랜치 컨텍스트에서만 동작

# Review Rules
- Type safety must-not
- Hooks correctness must-not
- Branch-aware strictness
- Hard/Soft gate 판단

# Output
- Decision
- Checklist Result
- Required Actions
- Review Comment Draft
```

그리고 알림 자동화 룰은 의외로 작지만 강하다.

```md
## Notification Rule
- Trigger comment includes 특정 키워드
- Extract only "변경 사항" section from PR body
- Send concise team message with mention policy
- Never include internal IDs/secrets in output
```

이게 좋은 이유는 리뷰의 “마지막 1마일”을 줄여주기 때문이다.  
리뷰 품질이 좋아도 요청 타이밍이 흔들리면 응답이 늦어진다.  
알림을 자동화하면 팀의 반응 속도가 일정해진다.

요지는 화려한 프롬프트가 아니다.  
**팀이 반복해서 부딪히는 문제를 룰로 고정했는가**가 훨씬 중요하다.

# 좋았던 변화
실제로 가장 크게 바뀐 건 “속도”보다 “마음 상태”다.

- PR 본문 품질이 기본선 이상으로 유지된다.
- 리뷰가 개인 컨디션에 덜 흔들린다.
- 누락이 줄고, 후속 액션이 선명해진다.
- 리뷰 요청을 미루는 심리적 저항이 줄어든다.

조금 감성적으로 말하면,  
리뷰가 더 이상 ‘큰일’이 아니라 ‘루틴’이 됐다.  
루틴이 되면 오래 간다.

# 아직 남은 불편함
커서가 웹스톰까지 지원되는 흐름은 확실히 반갑다.  
내 기준에서는 이게 꽤 큰 사건이었다.

다만 MCP를 다루는 감각은 아직 100% 자연스럽지는 않다.  
그래서 지금도 두 IDE를 병행한다.

- 한쪽: 자동화 트리거/연결이 편함
- 한쪽: 손에 익은 편집 리듬이 좋음

완벽하게 하나로 합쳐진 상태는 아니지만,  
지금은 이 하이브리드가 제일 현실적이다.

# 뒤늦게 알게 된 운영 팁
자동화를 몇 번 엎어보고 나서야 알게 된 것들이다.

1. 자동화는 “많이”보다 “순서”가 중요하다.  
2. 트리거 키워드는 짧고 고정된 문구가 낫다.  
3. 리뷰 봇도 맥락(브랜치/목적)을 알아야 쓸모가 있다.  
4. idempotent하게 설계하지 않으면 본문 정리 자동화는 금방 망가진다.  
5. 알림은 길게 쓰지 말고, 다음 행동이 보이게 써야 한다.

# 경험담 몇 가지
처음 PR 본문 자동 정리를 붙였을 때는 솔직히 반신반의했다.  
“이런 거 붙여봐야 결국 사람이 다시 만지지 않나?” 싶었다.

근데 달랐다.  
완벽하지 않아도 초안을 잘 깔아주면, 사람은 고급 판단에 집중할 수 있었다.  
리뷰 퀄리티가 올라가는 건 의외로 이런 데서 시작됐다.

기본 리뷰 룰을 도입할 때는 내부 저항도 조금 있었다.  
“너무 빡세지는 거 아닌가?”라는 걱정이 자연스럽다.

그래서 must-not과 개선 권장을 분리했다.  
고칠 건 확실히 고치고, 취향 영역은 코멘트로 남겼다.  
이 선을 넘지 않으니 팀이 룰을 받아들이기 쉬웠다.

꼼꼼 리뷰(하이진/심각도)는 생각보다 효과가 컸다.  
당장 장애를 내는 코드만 잡는 게 아니라,  
나중에 유지보수 비용이 크게 붙을 지점을 조용히 미리 잡아준다.

그리고 마지막 알림 자동화.  
이건 사소해 보이는데, 사소하지 않다.  
요청 타이밍이 일정해지면 리뷰 응답도 일정해진다.

# 마치며
요즘 나는 리뷰를 “열심히 하자”보다 “흐름으로 만들자”에 더 가깝게 본다.  
그 흐름을 만들어놓으면, 사람은 더 중요한 판단에 시간을 쓸 수 있다.

도구는 계속 바뀔 것이고 IDE도 더 좋아질 것이다.  
하지만 결국 남는 건 팀의 운영 습관이다.

나는 지금 이 루틴으로,  
PR 생성 직후 자동 정리부터 기본 리뷰, 꼼꼼 리뷰, 리뷰 요청까지 이어가고 있고,  
마지막에는 리뷰 승인 혹은 수정 요청을 오토메이션 작업 하고 있다.
