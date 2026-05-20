---
layout: post
title: "PR은 쪼개고, 규칙은 ~/.cursor에 — 레거시 API 에픽과 Cursor 스킬"
date: 2026-05-20 15:00:00 +0900
tags: [cursor, monorepo, api-migration, developer-tools, slack, jekyll]
---

> 내부 private monorepo 작업을 바탕으로 썼다. Jira·Slack 채널 ID·private PR URL은 일반화했고, `/cb:*` 스킬 규칙은 거의 그대로 둔다.

## 한 줄 요약

레거시 `heartApi`를 표준 패키지로 옮기는 작업을 **에픽 + 6개 PR**로 쪼갰고, 반복 업무는 **`~/.cursor`의 Markdown 스킬(`/cb:*`)** 로 묶었다. 슬랙 리뷰 요청까지 게이트를 두고 자동화해 봤다.

---

## 1. 왜 한 방에 안 했나

레거시 `HeartService`는 옛 API 허브 경로를 감싼 래퍼였고, 표준은 별도 `@…/apis-heart` 패키지다.

한 PR에 몰면:

- diff가 커져 리뷰·QA 부담이 커지고
- 앱별 회귀 지점이 섞여 롤백이 어렵다.

그래서 **앱·패키지 단위 하위 작업**과 **에픽 브랜치에 쌓는 스택 PR**을 택했다.

| 순서 | 범위 | 규모 |
|------|------|------|
| 1 | shop query/mutation | 작은 PR |
| 2 | shop legacy list | 작은 PR |
| 3 | content 잔여 | 작은 PR |
| 4 | auth 온보딩 | 작은 PR |
| 5 | shared hooks | 중간 PR |
| 6 | 서비스 파일 삭제·export 정리 | 마지막 PR |

마지막 PR에서 `HeartService.ts`를 지우고, 에픽 브랜치를 `main`에 합치는 큰 PR 하나만 남겼다.

---

## 2. 마이그레이션에서 반복된 패턴

### 2.1 API 매핑

| legacy | 표준 |
|--------|------|
| 단건 상태 조회 | `fetchHeartProductsStatus` / `fetchHeartBrand` |
| 사용자 좋아요 목록 | `fetchHeartProductIdList` / `fetchHeartBrandIdList` / … |
| 토글 | `updateSetHeart` / `updateUnsetHeart` |

### 2.2 완료 조건(AC) — grep이 답이다

```bash
rg 'heartApi|from .*/apis.*heart' <scope> || true
# scope 안 0건 (archive 제외)
```

“다 옮겼다”는 말보다 **범위 grep 0건**이 팀 간 합의에 가깝다.

### 2.3 리뷰에서 자주 고친 것

1. **Mutation 실패 시 throw** — envelope만 파싱하면 API 오류가 UI에서 “꺼짐”으로 보일 수 있다.
2. **id list SSOT** — 같은 fetch를 wrapper로 두 번 감싸지 않기.
3. **Union narrowing** — product/brand를 한 타입으로 뭉개지 말고 분기별 fetch.
4. **Optimistic update guard** — 캐시 없을 때 낙관적 업데이트를 넣지 않기 (깜빡임 방지).

---

## 3. 브랜치·PR 전략

```text
main
 └── feat/epic-remove-legacy-heart        ← 에픽 브랜치
      ├── feat/...-shop-query              → PR → epic
      ├── ...
      └── feat/...-remove-service        → 마지막 PR → epic
```

- 하위 PR의 **base는 main이 아니라 에픽 브랜치**.
- 커밋·PR 제목에 **티켓 키**를 붙여 추적성 유지.
- 에픽이 안정되면 **에픽 → main** PR 한 번.

---

## 4. Cursor 개인 스킬 `/cb:*`

팀 레포의 `AGENTS.md`·공용 skills는 **모두에게 맞는 규칙**이다.  
PR 리뷰 톤, Tier, Jira 마무리, 슬랙 멘션 규칙은 **개인·소속 팀 채널**에 가깝다.

그래서 규칙 SSOT를 레포 밖에 뒀다.

```text
~/.cursor/
├── commands/cb/*.md      # 긴 규칙 본문
├── skills/*/SKILL.md     # 슬래시 진입점
└── scripts/cb/           # dedupe, branch helper
```

`disable-model-invocation: true`로 **슬래시나 명시 호출 때만** 로드한다. 매 턴 자동 주입으로 토큰·혼선을 줄인다.

### 4.1 구현 파이프라인

```text
Slack / Confluence / Figma
        ↓
   /cb:intake           → Jira Epic·Story 초안
        ↓
   /cb:work-triage      → 유형·베이스·AC·커밋 단위
        ↓
   /cb:work-start       → 브랜치·grep
        ↓
   (코딩 · PR)
        ↓
   /cb:work-closeout    → Jira Done·다음 티켓
   /cb:slack-review-request → 슬랙 리뷰 요청
   /cb:blog             → 회고 초안 (docs/blog-drafts)
```

### 4.2 PR 리뷰 (권장 순서)

```text
/cb:prd-review → /cb:pr-checklist → /cb:typescript-review → /cb:nextjs-review
→ /cb:react-review → /cb:hygiene-review → /cb:critical-review
```

| 슬래시 | 하는 일 |
|--------|---------|
| `prd-review` | PRD·AC 맞는지 |
| `pr-checklist` | 템플릿·Tier·Gate |
| `typescript-review` | any, assert, ts-ignore |
| `nextjs-review` | App Router·Gateway |
| `react-review` | must-NOT·보안 |
| `hygiene-review` | deps·export·dead code |
| `critical-review` | 릴리즈 블로커만 |
| `pr-body` | PR 본문 메타 |
| `pr-review-notify` | 승인/수정요청 DM·답글 초안 |
| `slack-review-request` | **리뷰 요청** 메시지 |

---

## 5. 슬랙 리뷰 요청 자동화

### 5.1 반복 작업

PR을 올릴 때마다 팀 FE 채널에 리뷰 요청 글을 쓴다.  
매번 `gh pr view`로 reviewer·승인 상태를 확인해야 하고, **이미 승인된 PR에 또 올리면** 피로도만 올라간다.

### 5.2 `/cb:slack-review-request` 요약

1. **먼저** `gh pr view --json …` — 없으면 중단.
2. **Gate**: draft / merged / `APPROVED` → 전송 안 함.
3. **멘션**: `reviewRequests` + `assignees` → 없으면 도메인별 팀 멘션(경로 휴리스틱).
4. 한국어 초안 → **「슬랙에 보내줘」** 할 때만 Slack MCP로 post.
5. **`--test`**: 승인 완료 PR도 연동 테스트만 허용 (본문에 `[TEST]` 명시).

가치는 “보내기”보다 **내면 안 되는 PR을 막는 Gate** 쪽이 컸다.

### 5.3 한계

Cursor IDE는 24시간 Slack 리스너가 아니다.  
다음 단계는 GitHub `review_requested` → Cloud Agent → 슬랙 알림 같은 **이벤트 파이프라인**이다 (아직 설계만).

---

## 6. 배운 것

1. **grep AC**를 티켓마다 박기.
2. **에픽 브랜치 + 작은 PR**이 리뷰·revert 모두 편하다.
3. **규칙을 Markdown SSOT**로 두면 채팅마다 기준을 다시 설명하지 않아도 된다.
4. **자동화는 게이트가 핵심**이다.
5. **개인 스킬은 `~/.cursor`에** — 회사 레포와 분리해도 워크플로는 유지된다.

---

## 부록 — `slack-review-request` 규칙 (공개 가능)

```markdown
# /cb:slack-review-request <pr> [--domain-a|--domain-b] [--test]

## MUST
gh pr view <N> --json number,title,url,state,isDraft,reviewDecision,
  assignees,reviewRequests,reviews,files,...

## Gate
STOP: draft | not OPEN | APPROVED | all human reviewers approved
--test: skip "already approved" only

## Mentions
reviewRequests + assignees (exclude author, bots)
fallback: domain heuristic on changed paths / ticket prefix

## Post
Default: show draft in chat.
User confirms → slack_send_message(channel_id, message)
Dedupe: json file, 24h per PR number
```
