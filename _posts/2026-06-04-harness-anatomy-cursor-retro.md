---
layout: post
title: "Harness 글을 읽고 — 무엇을 고칠지, Cursor는 무엇을 덜어주는지"
date: 2026-06-04 17:25:00 +0900
tags: [cursor, harness, agent, langchain, claude, developer-experience]
---

> [LangChain — The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)를 읽고 남긴 메모다.  
> 잘 쓰는 사람 자랑이 아니라, **읽고 조심스럽게 정리한 내용**과 **도구는 사람에 맞게**라는 생각만 적는다.

## 읽을 글 (소개)

**[The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)** · LangChain · 약 12분

핵심만 옮기면 이렇다.

- **Agent = Model + Harness** — 똑똑한 모델만으로는 “일하는 에이전트”가 아니다.
- **Harness** = 프롬프트, 도구, 파일시스템, 샌드박스, 검증 루프, 훅, 오케스트레이션 같은 **모델 밖 전부**.
- 글은 도구(MCP·Skills)만 말하지 않고, **작업 흔적이 남는지**, **실패하면 다시 도는지**, **컨텍스트가 썩지 않는지**, **언제 끝내는지**까지 이어서 쓴다.

원문이 낫다. 여기서는 그다음 **내 상황**만 적는다.

---

## 읽고 나서 — 겸손하게 적어 보는 것

LangChain 프레임으로 **내 최근 Cursor 설정**과 **레포에 있는 팀 규칙**을 나눠 보면 아래와 같다. 정답표는 아니다.

| Harness 쪽 | 내가 하는 것 | 느낌 |
|------------|--------------|------|
| 가이드·규칙 | Cursor 규칙, 레포 가이드, 슬래시로 정리한 런북 | 나쁘지 않음 |
| 도구·연동 | MCP, 스킬, 필요할 때만 쓰는 슬래시 커맨드 | 나쁘지 않음 |
| 파일·기억 | 블로그 초안, 학습 메모, 개인 저장소 백업 | 더 정리할 여지 |
| 검증 | PR 리뷰 **순서**, diff 체크리스트, E2E 단계 | 루틴을 돌릴 때는 있음 |
| 센서·훅 | 조건부 가드 **아주 소수** | **얇음** |
| 장시간·종료 | 마감 루틴, “여기서 멈춤” 보류 정책 | 사람이 끊는 편 |

**고민 1.** LangChain이 말하는 **“편집 후 자동 검증”**은 아직 약하다. CI·리뷰 루틴에 많이 맡기고, 훅은 거의 안 돌린다.  
**고민 2.** 규칙·스킬·MCP가 흩어져 있어서, 가끔 **“지금 뭐가 켜져 있지?”**를 잃는다. 한 장짜리 지도 정도는 있으면 좋겠다.  
**고민 3.** 생산성 **숫자**는 잘 모르겠다. harness 이야기는 흥미로운데, 잰 뒤에 깊게 파기는 꺼려진다.

반대로 **잘 된 것**도 있다. 스킬을 매번 다 넣지 않는다. 리뷰·마감 **순서**가 있다. 팀 레포 표준과 **개인 Cursor 설정** 사이에 **경계**를 두려고 한다.  
이 정도만 적어 두고, 더 자동화할지는 **천천히** 보려 한다.

---

## 그런데 Cursor가 가볍게 해 준 것

[harness 글](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)을 읽으면 “처음부터 다 짜야 하나?” 부담이 큰데, Cursor를 쓰는 입장에서는 **꽤 많이 깔려 있다**.

| 글이 말하는 부담 | Cursor 쪽 |
|------------------|-----------|
| 채팅 루프·메시지 상태 | IDE 안 **채팅·에이전트** |
| 파일 읽기·쓰기·터미널 | **Composer·도구** |
| MCP | **MCP 연동** |
| 훅·가드 슬롯 | **훅 설정** |
| 브라우저로 확인 | **브라우저 MCP** |
| 긴 일 쪼개기 | **서브에이전트** |

그래서 내가 신경 쓰는 건 **“인프라를 새로 짓기”**보다 **“레포에서 뭐가 표준인지, 실패할 때 어디를 볼지”**에 가깝다.  
**뼈대는 IDE가 주고**, 남는 건 **얇은 런북**이라고 느꼈다.

---

## 도구는 사람·일에 맞게

승자를 가리는 것보다, **일하는 방식**이 다르다고 본다.

### Cursor가 잘 맞는 편 (나 포함)

- **에디터 안에서** 파일·diff·PR·탐색을 오간다.
- 리뷰·스테이징·로컬 확인을 **한 화면 흐름**으로 묶고 싶다.
- harness를 **규칙 + MCP + 가끔 슬래시 커맨드**로 얹는 게 편하다.

### Claude Code가 잘 맞을 수 있는 편

- **터미널·레포 전체**를 에이전트에 맡기는 **리듬**이 익숙하다.
- 레포 루트 가이드·훅·플러그인으로 **코드베이스에 harness를 심는** 그림이 자연스럽다.
- IDE보다 **CLI 오케스트레이션**이 중심이다.

### 그 외

- 이미 **JetBrains·Vim·다른 에이전트**에 맞춰 둔 사람에게까지 **같은 제품**을 권하는 건 부담일 수 있다.
- 중요한 건 제품 이름이 아니라 **같은 질문**을 공유하는 것이다.  
  → 실패할 때 멈추나? 작업 흔적은 어디에 남나? 끝 조건이 있나?

나는 지금 Cursor 쪽에 런북을 더 쌓아 둔 것뿐이고, **Cursor가 정답**이라고 말하고 싶지는 않다.

---

## 한 가지 더 — harness를 어디까지 맞출지

커뮤니티나 문서에서 **“이 플러그인만 깔면 된다”**는 말을 가끔 본다. 그때마다 [harness 글](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)이 강조하는 **검증·종료·작업 흔적**까지 **같이 안내되는지**가 더 궁금하다.

에디터 테마·PR 템플릿처럼, harness도 **팀 바닥**(레포 가이드, CI)과 **개인 층**(규칙, MCP, 마감 순서)을 나누는 쪽이 맞다고 본다. 표준 패키지가 있으면 참고하되, **본인이 쓰는 도구**에 맞게 얇게 얹는 편이 낫다. 나는 Cursor, 다른 분은 Claude Code — **둘 다 harness**일 수 있다.

---

## 마무리

1. [Harness anatomy](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness) — **모델 밖**을 읽을 거리.  
2. 나 — **검증 순서·살아 있는 문서**는 있고, **자동 센서·한 장짜리 지도**는 고민.  
3. Cursor — **인프라 부담**을 꽤 덜어 준다.  
4. 도구 — **Cursor / Claude Code / 기타**는 사람·일에 맞게. 설치 목록만으로는 부족하다.  
5. 표준 패키지 — **팀 바닥**과 **개인 층**을 나눠 두는 쪽이 편하다.

다음에 손대면 훅 하나, harness 맵 한 페이지 정도면 될 것 같다.  
그 전에 원문 한 번 읽고 **“나는 지금 모델 밖에 뭘 두고 있지?”**만 물어보면 충분하다.

---

## 참고

- [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [How Claude Code works in large codebases](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start)
- [my-cursor를 어떻게 구성했는가](https://changbaebang.github.io/2026-05-20-my-cursor-architecture-retro/) — 개인 Cursor 설정·레이어를 더 길게 쓴 글
