---
layout: post
title: "「하네스 다 했어요」는 아직 시작 — Claude Code가 대규모 코드베이스에서 말하는 것"
date: 2026-05-24 22:00:00 +0900
tags: [ai, claude, harness, cursor, engineering, productivity, agent-ops]
---

> Claude로 한 번 문서 뽑고 “하네스 세팅 끝”이라고 말하는 순간,  
> **실제로는 하네스 유지보수가 시작되지 않은 것**일 수 있다.

이 글은 [Anthropic — *How Claude Code works in large codebases*](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start) (2026-05-14)을 읽고, **번역본이 아니라 내 해석**을 붙인 메모다. 원문의 표·세부 사례는 링크를 본다.

요즘 흔한 장면:

1. Claude/Cursor로 `CLAUDE.md`·스킬·훅을 **한 번** 생성한다.
2. “우리 팀도 AI 하네스 도입 완료”라고 말한다.
3. 코드는 매일 바뀌는데, 문서·훅·스킬은 **그날 그대로**다.

Anthropic 글은 이 패턴을 정면으로 깨뜨린다. 핵심은 한 줄:

**모델만큼, 아니 그 이상으로 “하네스”가 성능을 결정한다. 그리고 하네스는 한 번 세팅하는 게 아니라 유지보수하는 것이다.**

## TL;DR

- Claude Code는 RAG 인덱스가 아니라 **살아 있는 코드베이스를 agentic search**한다 — 대신 **어디를 볼지** 알려줄 길잡이가 필요하다.
- 하네스 5층: `CLAUDE.md` → hooks → skills → plugins → MCP (+ LSP, subagents).
- **stop hook으로 CLAUDE.md를 세션마다 갱신**하라는 말이 있다 — “한 번 만들고 끝”과 정반대.
- 모델이 업그레이드되면 **예전 모델 한계를 메우던 규칙이 오히려 발목**이 될 수 있다 — 3~6개월마다 검토.
- 전사 도입에는 **DRI / agent manager / 크로스펑셔널 WG** — 지식이 부족이 되지 않게.

---

## 1. “모델이 전부”가 아니다

벤치마크·데모 task에서 잘 나온다고, 레거시·모노레포에서 바로 잘 되는 건 아니다.

Anthropic은 Claude Code가 **인간 엔지니어처럼** 파일을 읽고 grep하고 참조를 따라간다고 설명한다. RAG로 잘라 넣은 **과거 스냅샷**과 달리, 로컬의 **현재 코드**를 본다.

대신 tradeoff가 있다:

> 코드베이스 setup에 투자한 팀이 더 좋은 결과를 본다.

즉, “도구만 깔면 된다”가 아니라 **코드베이스 + 하네스가 도구의 일부**다.

---

## 2. 하네스 5층 — 순서가 있다

| 층 | 역할 | 내가 읽은 포인트 |
|----|------|------------------|
| **CLAUDE.md** | 매 세션 자동 로드 — root는 큰 그림, 하위는 로컬 규칙 | root에 다 넣으면 **매 세션 노이즈** |
| **Hooks** | 규칙 강제 + **자기 개선** | stop hook → 세션 끝에 CLAUDE.md 업데이트 **제안** |
| **Skills** | 필요할 때만 전문 지식 | 전부 CLAUDE.md에 넣지 말 것 |
| **Plugins** | 잘 되는 설정을 **패키지로 배포** | “잘 아는 사람만”이 아니라 조직 표준 |
| **MCP** | 티켓·분석 등 외부 연결 | — |

추가로 **LSP**(정의로 이동·심볼 추적)와 **subagents**(탐색과 편집 분리)를 강조한다. 텍스트 grep만으로는 C/C++·대규모 다국어 repo에서 길을 잃기 쉽다.

---

## 3. 코드베이스를 “읽기 쉽게” — FE도 해당

대규모 전용이 아니다. 원칙은 그대로다.

- **root가 아니라 작업 중인 하위 디렉터리**에서 시작 — monorepo에서 root만 보면 컨텍스트 낭비.
- **테스트·lint는 해당 서비스 범위** — 전체 스위트 돌리다 timeout·무관 로그.
- `.ignore` / `permissions.deny`로 생성물·서드파티 **노이즈 제거**.
- 폴더 구조만으로 부족하면 **codebase map**(한 줄 설명 목차).

나는 여기서 my-cursor를 떠올렸다. `~/.cursor/commands/cb/*.md`가 **repo 밖 personal harness**이고, 팀 레포 `.claude`와 **역할이 겹치지 않게** 나누는 게 중요하다. [skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)에서 쓴 것처럼, **실전에서 틀린 걸 스킬 정의에 다시 써 넣는 루프**가 하네스의 본체다.

---

## 4. 내가 이 글에서 가장 강하게 가져가고 싶은 문단

### “하네스는 살아 있다”

Anthropic은 **모델 intelligence가 바뀔 때마다 CLAUDE.md를 actively maintain**하라고 명시한다.

- 예전 모델이 못 하던 걸 **한 파일씩만 고치게** 하던 규칙 → 새 모델에겐 **발목**.
- Perforce `p4 edit` 훅처럼 **도구 한계를 메우던 hook** → 네이티브 지원 생기면 **overhead**.

**3~6개월마다**, 또는 **주요 모델 업데이트 직후** — “이 규칙/훅/스킬 아직 필요한가?”를 지운다.

이게 “Claude 한 번 돌려서 문서 만들고 끝”과 정면으로 충돌한다.  
**코드가 변한 만큼 하네스도 PR처럼 변해야** repo에 실효성이 남는다.

### “누가 주인인가”

대규모 조직 이야기지만, 개인·소규모 팀에도 통한다.

- **DRI** 한 명: settings, plugin marketplace, CLAUDE.md convention, **keep them current**.
- bottom-up만 하면 **tribal knowledge** — A팀 스킬은 B팀 모름.
- eng + security + governance **WG** — 우리에겐 “리뷰·CI·식별자” 정도로 축소 가능.

“에이전트 매니저”라는 역할 이름까지 나온다. PM+엔지니어 hybrid.  
**하네스를 제품처럼 운영**하라는 뜻에 가깝다.

---

## 5. “하네스 다 했어요” 체크 — 세게

**8개 전부 “예”가 아니면 “다 했어요”라고 말하지 말 것.**  
하나라도 아니오면, Anthropic이 말하는 하네스 운영과는 아직 거리가 있다.

- [ ] 지난 **14일** 안에 `CLAUDE.md` / 스킬 / 훅을 **실제 작업 경험** 때문에 고친 적이 있는가 — *초기 생성 PR만 있으면 No*
- [ ] stop hook **또는** 고정 루틴(세션 끝 3분)으로 배운 걸 문서에 반영하는가 — *지난 5세션 중 2회 이상 실행했는가*
- [ ] root `CLAUDE.md`가 **포인터 수준**(핵심 gotcha·링크만)인가 — *만능 백과사전이면 No*
- [ ] 테스트·lint가 **작업 디렉터리 범위**로 scoped 되어 있고, *전체 suite timeout으로 세션을 낭비한 적이 없는가*
- [ ] 지난 **분기**에 예전 모델·도구 한계를 메우던 규칙·훅을 **삭제한 기록**이 있는가 — *추가만 하고 제거 안 하면 No*
- [ ] 잘 되는 설정이 repo·plugin·`/cb:*` 등 **공유 경로**에 있고, *clone/install만으로 2주 뒤의 나(또는 동료)가 그대로 쓸 수 있는가*
- [ ] 하네스 변경이 코드처럼 **git diff로 추적**되는가 — *로컬만 바꾸고 커밋 안 하면 No*
- [ ] “하네스 품질”과 “모델/플랜 tier”를 **별 문제**로 보고 있는가 — *tier 올리면 끝,이면 No*

> **자가 진단:** 8개 중 6개 이상 No → “하네스 다 했어요”가 아니라 **“문서 한 번 뽑았어요”**에 가깝다.

---

## 6. blog·my-cursor와 연결

| Anthropic 말 | 내 운영 |
|--------------|---------|
| CLAUDE.md 계층 | 레포 규칙 + personal `~/.cursor/commands/cb` |
| Skills on-demand | `/cb:blog-radar`, `/cb:learn-radar` 등 **스킬 = 코드** |
| Hooks self-improve | E2E 실패 → 스킬 수정 루프 ([skill-is-code](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)) |
| Plugin 배포 | `my-cursor` + `install.sh` — **git이 SSOT** |
| 주기적 정리 | 모델 업데이트 때 **완화·삭제**할 규칙 목록 만들기 |

측정 이야기는 [Twelve Ways 큐레이션](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)과 이어진다.  
“하네스 도입률”을 성과로 재지 말고, **규칙이 살아 움직이는지**를 보라.

---

## 마치며

Anthropic 글은 “Claude Code 잘 쓰는 법 12가지”가 아니다.  
**대규모 코드베이스에서 AI를 ‘도구’가 아니 ‘환경’으로 두는 법**이다.

그 환경은 README 한 번 PR로 끝나지 않는다.  
코드 diff만큼 **하네스 diff**가 쌓여야, “우리 팀 AI 잘 쓴다”가 헛소리가 아니다.

**한 줄 결:** *하네스는 설치물이 아니라, 코드와 같이 버전 관리되는 운영 자산이다.*

---

## 읽을 거리

- **원문 (필수):** [How Claude Code works in large codebases](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start) — Anthropic (2026-05-14)
- [AI 스킬은 코드다 — 쓰면서 고쳐야 산다](https://changbaebang.github.io/2026-05-21-skill-is-code-iterate-it/)
- [AI 보조 코딩 생산성 — 잘못 재는 12가지 (읽을거리)](https://changbaebang.github.io/2026-05-23-twelve-ways-wrong-ai-coding-curation/)

*이 포스트는 Anthropic 원문의 번역·재게시가 아닙니다. 요약과 실무 해석을 더했습니다.*
