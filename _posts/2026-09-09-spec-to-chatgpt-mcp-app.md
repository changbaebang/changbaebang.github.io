---
layout: post
title: "Codex에게 앱을 만들어 달라고 하기 전에 명세부터 썼다"
date: 2026-09-09 16:00:00 +0900
excerpt: "커리어 도구 아이디어를 바로 코딩하지 않고, 판단 기준과 금지 사항을 Markdown 명세로 먼저 고정했다. Codex로 MCP 서버와 React 위젯을 만들고 ChatGPT의 실제 실행 환경까지 연결하면서, 명세와 로컬 테스트만으로는 잡히지 않던 빈 화면도 발견했다."
categories: [Engineering, AI]
tags: [Codex, ChatGPT, MCP, React, TypeScript, Product Specification]
---

앱을 하나 만들었다고 하기에는 아직 이르다. 이력서를 분석하지도 않고, 채용 공고를 평가하지도 않는다. 지금 되는 것은 ChatGPT에서 도구 하나를 호출하고, 로컬 MCP 서버가 반환한 상태를 React 카드로 보여주는 것까지다.

그런데 이 작은 단계까지 오는 과정은 생각보다 글로 남길 만했다.

아이디어를 코드로 옮기기 전에 무엇을 판단할 도구인지 정의했고, 하지 않을 일을 정했고, 그 내용을 Codex가 실행할 수 있는 Markdown 명세로 바꿨다. 로컬에서 빌드와 테스트를 통과한 뒤에는 Secure MCP Tunnel로 ChatGPT에 연결했다. 도구 호출은 성공했는데 위젯은 비어 있었고, 실제 ChatGPT 환경에서만 보이는 번들 문제도 하나 찾았다.

이 글은 완성된 서비스 소개가 아니라 **아이디어를 에이전트가 구현하고 사람이 검증할 수 있는 작업 단위로 바꾼 기록**이다.

## 또 하나의 이력서 매칭 앱은 만들고 싶지 않았다

출발점은 흔하다. 이력서와 채용 공고를 넣으면 둘이 얼마나 잘 맞는지 알려주는 도구다.

문제는 이 설명만으로는 이미 있는 많은 서비스와 크게 다르지 않다는 점이다. 키워드를 맞춰주고, ATS 점수를 보여주고, 공고에 맞게 이력서를 고쳐주는 제품은 많다. 점수 하나를 크게 보여주는 순간 사용자는 그것을 채용 가능성처럼 받아들이기도 쉽다.

내가 먼저 알고 싶었던 것은 “이 공고에 맞게 이력서를 어떻게 고칠까?”보다 “이 공고에 애초에 지원할 가치가 있을까?”에 가까웠다.

그래서 Career Radar의 첫 판단을 세 가지로 정했다.

- `REALISTIC`: 현재 이력서로도 큰 왜곡 없이 적합성을 설명할 수 있다.
- `STRETCH`: 핵심 경력은 일부 맞지만 분명한 도전 요소가 있다.
- `PASS`: 중심 요구사항과 거리가 멀거나 명확한 차단 조건이 있다.

여기에 `resume contortion`이라는 기준을 하나 더 넣었다. 한 공고에 맞추기 위해 이력서를 얼마나 억지로 비틀어야 하는지를 보는 개념이다. 없는 경험을 비슷한 단어로 포장해야만 맞는 것처럼 보인다면, 높은 점수를 만드는 것보다 그 공고를 거르는 편이 낫다.

이 정의가 없으면 AI는 그럴듯하게 낙관적인 답을 만들기 쉽다. 이전에 [잘못 이해한 요구도 AI는 빠르게 구현한다](/2026-07-22-ai-fast-wrong-requirements/)고 썼는데, 이번에는 코드보다 먼저 그 문제를 막아보기로 했다.

## 요구 사항 문서에 기능보다 금지 사항을 많이 적었다

프로젝트 명세는 [`docs/PROJECT_SPEC.md`](https://github.com/changbaebang/career-radar/blob/main/docs/PROJECT_SPEC.md)에 넣었다. 제품 설명만 적은 문서는 아니다. Codex가 저장소를 열고 바로 작업을 시작할 수 있도록 다음 내용을 한 파일에 모았다.

- `REALISTIC / STRETCH / PASS` 판정 기준
- Candidate, Job, Assessment, Application 데이터 모델
- 여섯 개의 MCP 도구 설계
- React + Node + TypeScript 저장소 구조
- OpenAI Responses API와 구조화된 출력(structured output)의 역할
- SQLite 기반 로컬 저장 방향
- false-REALISTIC, hard-blocker recall, evidence grounding 평가 기준
- Milestone 0부터 공개 버전까지의 순서
- 첫 번째와 두 번째 Codex 프롬프트
- 아직 하지 말아야 할 것

특히 마지막 항목을 의도적으로 강하게 썼다.

```text
Do not implement resume parsing, job analysis, search,
persistence, or application tracking yet.

Stop after Milestone 0 is complete.
```

처음부터 채용 공고 검색을 붙이면 겉보기에는 서비스처럼 보인다. 하지만 핵심 판정이 틀리면 검색 결과가 많아질수록 잘못된 추천도 더 빨리 늘어난다. 그래서 첫 단계에서는 `이력서 → JD → 근거 기반 판정`조차 만들지 않았다. 먼저 MCP 서버와 UI가 실제 ChatGPT에서 왕복하는지만 확인하기로 했다.

명세와 별도로 저장소 루트에 `AGENTS.md`도 뒀다. 둘의 역할은 조금 다르다.

- `PROJECT_SPEC.md`: 무엇을 왜 만들지 정의한다.
- `AGENTS.md`: 이 저장소에서 에이전트가 어떻게 행동할지 제한한다.

예를 들어 `AGENTS.md`에는 미래 마일스톤을 임의로 구현하지 말 것, 후보자의 경력을 만들지 말 것, 모델 출력을 Zod로 검증할 것, 코드로 확정할 수 있는 규칙을 LLM에 맡기지 말 것, 작업을 마치기 전에 lint·typecheck·test를 실행할 것을 적었다.

프롬프트 한 번에 모든 맥락을 길게 설명하는 대신, 저장소 자체가 작업 규칙을 갖게 한 셈이다.

## 첫 번째 구현 목표는 일부러 작게 잡았다

Codex에 준 첫 작업의 목표는 Milestone 0 하나였다.

결과물은 다음 정도다.

```text
career-radar/
  server/           # MCP 서버와 /health 엔드포인트
  web/              # Vite로 번들링하는 React 위젯
  packages/shared/  # 공유 Zod 스키마와 TypeScript 타입
  docs/             # 프로젝트 명세와 기술 결정 기록
  data/             # 이후 로컬 데이터가 들어갈 자리
```

서버에는 읽기 전용 `career_radar_status` 도구 하나만 있다. 호출하면 현재 마일스톤, 준비 상태, 지원 기능 목록을 `structuredContent`로 반환한다. UI는 그 결과를 받아 작은 상태 카드로 렌더링한다.

실제 커리어 판단 기능은 없지만 이 단계에서 확인할 수 있는 것은 분명하다.

1. ChatGPT가 MCP 도구를 발견할 수 있는가.
2. 로컬 서버가 `structuredContent`를 반환하는가.
3. 도구와 UI 리소스가 연결되는가.
4. React 위젯이 ChatGPT iframe 안에서 실행되는가.
5. UI가 없어도 도구의 텍스트 결과만으로 상태를 이해할 수 있는가.

OpenAI의 현재 플러그인 UI 문서 역시 MCP 도구가 커스텀 UI 없이도 유용해야 한다고 설명한다. UI는 `_meta.ui.resourceUri`로 연결하고 MCP Apps 브리지를 통해 호스트와 통신한다. 구현할 때는 예전 Apps SDK에 대한 기억에 기대지 않고 [현재 MCP 서버 UI 문서](https://developers.openai.com/plugins/build/chatgpt-ui)를 다시 확인했다.

## 로컬 검증은 모두 통과했다

구현 후에는 네 가지 명령을 기준으로 확인했다.

```bash
pnpm build
pnpm lint
pnpm typecheck
pnpm test
```

공유 Zod 스키마 테스트와 서버 테스트가 통과했고, `/health`와 `/mcp` 엔드포인트도 응답했다. 여기까지만 보면 Milestone 0은 끝난 것처럼 보였다.

하지만 로컬 검증이 통과했다는 것은 로컬에서 확인한 범위가 통과했다는 뜻이다. [일곱 번 다, 확인한 대상이 틀렸다](/2026-08-11-verified-the-wrong-thing/)에서 남긴 문장을 이번에도 다시 만나게 됐다.

## Secure MCP Tunnel로 ChatGPT에 연결했다

로컬 MCP 서버를 ChatGPT에서 호출하려면 ChatGPT가 접근할 수 있는 경로가 필요하다. 이번에는 로컬 포트를 공개 URL로 여는 대신 OpenAI의 [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)을 사용했다.

구조는 단순하다.

```text
ChatGPT
   ↓
OpenAI-hosted tunnel endpoint
   ↓ outbound HTTPS
tunnel-client
   ↓
http://localhost:8000/mcp
```

로컬 네트워크에 인바운드 포트를 열지 않고, `tunnel-client`가 OpenAI에서 전달된 MCP 요청을 받아 로컬 서버로 넘긴다. ChatGPT에서는 개발자 모드를 활성화하고 이 터널을 사용하는 Career Radar 플러그인을 만들었다.

그리고 새 대화에서 이렇게 요청했다.

```text
Career Radar 플러그인의 career_radar_status 도구를 호출해서
현재 연결 상태를 보여줘.
```

도구 호출은 성공했다. ChatGPT는 `ready`, `Milestone 0`, 세 가지 지원 기능을 정상적으로 읽었다.

그런데 React 카드가 나와야 할 자리는 커다란 빈 상자였다.

## 도구는 성공했는데 위젯은 비어 있었다

이 상태를 “연결 완료”라고 써도 될까?

서버 관점에서는 완료다. MCP 도구가 호출됐고 `structuredContent`도 반환됐다. ChatGPT도 내용을 문장으로 설명했다. 하지만 사용자가 보는 위젯은 비어 있었다. 프로젝트 목표에 React UI가 포함되어 있으므로 완료가 아니었다.

iframe의 본문을 확인하니 텍스트가 하나도 없었다. 데이터 수신 문제라면 최소한 `Waiting for Career Radar status…`가 보여야 한다. 그것조차 없다는 것은 React가 상태를 못 받은 게 아니라 **번들 자체가 실행되지 않았다**는 뜻이었다.

처음에는 UI 리소스에 넣은 `<script type="module">`을 일반 `<script>`로 바꿨다. 번들은 Vite가 만든 단일 파일이라 외부 `import`가 없었다. 그러자 빈 화면 대신 ChatGPT가 `Runtime error`를 보여줬다. 문제를 고친 것은 아니지만 실패가 보이기 시작했다.

생성된 번들을 검색하니 브라우저 코드 안에 다음 참조가 남아 있었다.

```js
process.env.NODE_ENV
```

ChatGPT의 sandbox iframe은 Node.js 환경이 아니다. 전역 `process`가 없으니 React가 마운트되기 전에 스크립트가 종료된다. 로컬 빌드가 성공하는 것과 브라우저 런타임에서 실행되는 것은 다른 문제였다.

Vite 설정에서 브라우저 번들에 프로덕션 값을 명시했다.

```ts
export default defineConfig({
  plugins: [react()],
  define: {
    "process.env.NODE_ENV": JSON.stringify("production"),
  },
  // ...
});
```

다시 빌드하자 `process.env.NODE_ENV` 참조가 사라졌고 번들 크기도 약 1.04 MB에서 396 KB로 줄었다. React 개발용 코드가 함께 빠진 결과였다.

여기서 한 번 더 해야 할 일이 있었다. 기존 대화와 플러그인 연결에는 이전 UI 리소스 URI가 남아 있었다. 캐시 영향을 피하려고 URI를 `status-v3.html`로 올리고, 터널을 다시 시작하고, 플러그인 메타데이터를 새로고침했다.

그 뒤 새 대화에서 다시 호출했다.

```text
MILESTONE 0
Career Radar
Scaffold ready

✓ Read-only MCP status tool
✓ React widget resource
✓ Shared Zod schema
```

이번에는 같은 내용이 ChatGPT의 설명과 React 카드 양쪽에 모두 나타났다.

## API 키는 코드가 아니라 실행 환경에 남겼다

로컬 서버를 터널에 연결하려면 OpenAI API 키가 필요하다. 하지만 이 키는 Career Radar의 React 위젯에 들어갈 이유가 없다. 브라우저로 전달될 이유도 없고, 저장소에 커밋될 이유도 없다.

로컬에서는 Git에서 제외된 `.env.local`에 두고 터널 프로세스의 환경 변수로만 전달했다. GitHub Actions에서 실제로 배포나 터널 작업을 실행하게 되는 시점이 오면 그때 GitHub Actions의 secret을 사용하면 된다. 지금 쓰지 않는 CI용 secret을 미리 만드는 것도 하지 않았다.

계정을 바꾸게 되더라도 프로젝트를 다시 만들 필요는 없다. 새 계정에서 API 키와 터널을 만들고 ChatGPT 플러그인 연결만 갱신하면 된다. 코드와 명세는 GitHub 저장소에 있고, 계정별 비밀값은 실행 환경에만 있기 때문이다.

## 지금 만든 것은 제품이 아니라 다음 구현을 올릴 기반이다

현재 Career Radar에 이력서를 넣어도 아무 일도 일어나지 않는다. 채용 공고를 주고 `REALISTIC / STRETCH / PASS` 판정을 요청할 수도 없다. 이 부분을 흐리면 작은 연결 데모를 완성된 AI 앱처럼 포장하게 된다.

Milestone 0에서 확인한 것은 더 제한적이다.

- Codex가 명세와 저장소 규칙을 따라 범위를 지켰다.
- Node MCP 서버와 React 위젯 구조가 실제 ChatGPT에 연결된다.
- `structuredContent`가 UI와 대화 양쪽에서 사용된다.
- 로컬 테스트와 실환경 검증은 서로 다른 실패를 잡는다.
- 키를 저장소와 UI에서 분리한 채 개발할 수 있다.

다음 Milestone 1부터 비로소 Career Radar다운 기능이 시작된다.

1. 후보자 이력서를 입력한다.
2. 하나의 채용 공고를 입력한다.
3. 공고 요구 사항을 실제 경력 문장과 연결한다.
4. `hard blocker`와 `resume contortion`을 분리한다.
5. `REALISTIC / STRETCH / PASS`를 근거와 함께 보여준다.
6. 판정 동작을 테스트와 평가 케이스(eval case)로 남긴다.

여기서도 채용 공고 웹 검색과 자동 이력서 재작성은 만들지 않는다. 단일 공고 판정이 충분히 믿을 만해진 뒤에 확장할 일이다.

## 남는 문장

Codex를 쓰면 코드를 빠르게 만들 수 있다. 하지만 빠르게 만드는 것과 만들 대상을 잘 정하는 것은 다른 일이다.

이번에 시간을 더 쓴 곳은 코드가 아니라 경계였다. `REALISTIC`이 무엇인지, 무엇을 확률처럼 말하면 안 되는지, 첫 마일스톤에서 무엇을 만들지 않을지, 어떤 검증이 끝나야 완료라고 부를지를 먼저 적었다.

그리고 마지막 빈 화면은 그 경계가 한 단계 더 필요하다는 것을 보여줬다.

> 빌드가 통과한 앱과 사용자가 실제로 본 앱은 같은 결과물이 아니다.

아직 Career Radar는 커리어를 판단하지 못한다. 대신 이제 다음 기능을 쌓아도 되는 기반이 실제 ChatGPT 안에서 작동한다. 지금 단계에서 완성했다고 말할 수 있는 것은 그 정도이고, 다음 작업을 시작하기에는 그 정도면 충분하다.

프로젝트 저장소: [changbaebang/career-radar](https://github.com/changbaebang/career-radar)
