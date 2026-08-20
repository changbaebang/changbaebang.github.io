---
layout: post
title: "내 사용 통계가 내 폴더 이름에 달려 있었다"
date: 2026-08-20 12:35:00 +0900
excerpt: "AI 사용 기록을 분류하는 스크립트가 요청이 아니라 그 요청이 놓인 환경을 채점하고 있었다. 경로, 링크, 에디터가 붙인 컨텍스트, 그리고 도구 이름 자체가 카테고리를 결정했다. 무엇을 채점하면 안 되는지 정리한다."
tags: [ai, agents, skills, measurement, regex, testing, open-source, developer-experience]
---

공개 저장소 [`Agent Skill Garden`](https://github.com/changbaebang/agent-skill-garden)에는 내 AI 사용 기록을 분석해 주는 스킬이 있다. 어떤 작업을 반복하는데 스킬이 없는지, 있는데 안 불리는지를 찾아 준다.

그걸 돌리다가 **스킬이 아니라 측정기가 고장 나 있는 걸** 발견했다. [PR로 고쳤다](https://github.com/changbaebang/agent-skill-garden/pull/3).

## 45와 18

어떤 스킬의 수요가 **45건**으로 나왔다. 그런데 그 스킬은 한 번도 발동하지 않았다.

"수요 45건인데 0발동"이면 결론은 뻔하다. 트리거 문구가 실제 표현과 어긋난 것이니 문구를 고쳐야 한다. 다만 45라는 숫자가 체감과 달랐다. 그렇게 자주 쓴 기억이 없었다.

원문을 열어 보니 이런 것들이 섞여 있었다.

```text
<ide_opened_file>The user opened the file
/home/me/Codes/shop/src/checkout.ts in the IDE.</ide_opened_file>
```

내가 친 말이 아니다. **에디터가 붙인 컨텍스트**다. 파일을 열었다는 사실을 알려 주는 블록인데, 이게 "사람이 친 요청"으로 세어지고 있었다.

걷어내니 45건이 **18건**이 됐다. 남은 18건은 대부분 한 달 전 것이었다. 실제 상황은 "수요가 많은데 안 불린다"가 아니라 **"수요가 이미 사라졌다"**였다. 정반대다.

## 여기서 멈췄으면 잘못 고칠 뻔했다

처음 쓴 수정은 단순했다. 시스템 주입을 걸러내는 필터가 이미 있었으니, 거기에 에디터 태그 세 개를 추가하는 것이다.

```python
SYSTEM_PROMPT = re.compile(
    r"^\s*(?:<heartbeat>|<recommended_plugins>|<task-notification>|<command-|...)",
    re.I,
)
```

PR을 올리고 나서 "너무 구체적인 처리 아니냐"는 지적을 받았다. 맞는 말이었다. **이 목록은 이미 낡아 있었기 때문에 문제가 된 것**이다. 거기에 항목을 세 개 더 얹으면 다음에 새 태그가 생겼을 때 똑같이 뒤처진다. 증상을 고치고 원인을 남기는 수정이다.

그래서 태그를 전부 걷어낸 상태에서, 사람이 직접 친 프롬프트만 가지고 다시 재봤다.

## 태그가 문제가 아니었다

```text
"이 파일 좀 봐줘 /home/me/Codes/shop/src/a.ts"
  → implementation

"이 초안 어디 있어? ~/Documents/Codex/notes.md"
  → skill-maintenance

"https://example.com/blog/local-server-setup 이거 읽어봐"
  → local-development
```

전부 사람이 친 말이고, 주입 태그는 하나도 없다. 그런데 분류가 다 틀렸다.

매칭된 문자열을 찍어 보니 첫 줄은 `['Code']`였다. 분류 규칙 중에 이런 게 있다.

```python
("implementation", re.compile(r"구현|수정|리팩터|refactor|bug|버그|fix|code|코드", re.I)),
```

내 소스 경로가 `/home/me/Codes/…`이다. 규칙의 `code`가 경로의 `Codes` 안에서 걸린 것이다. **파일을 봐 달라고만 해도 코딩 작업으로 집계된다.** 두 번째 줄은 더 나쁘다. 경로의 `Codex`가 실제 요청어인 `초안`을 이겼다.

여기서 진짜 문제가 보인다. **결과가 내가 소스 폴더를 뭐라고 이름 지었는지에 달려 있다.** `Codes`를 쓰면 코딩 비중이 올라가고 `dev`나 `workspace`를 쓰면 안 올라간다. 같은 도구가 같은 사용 패턴에 대해 **머신마다 다른 통계**를 낸다는 뜻이다.

## 도구 이름이 1번 규칙이었다

더 있었다. 규칙 목록의 맨 앞은 이랬다.

```python
("skill-maintenance", re.compile(r"스킬|skill|agent.?skill|claude|codex|cursor", re.I)),
```

그리고 분류 함수는 **첫 매칭에서 즉시 반환**한다.

```python
for name, pattern in CATEGORY_RULES:
    if pattern.search(prompt):
        return name
```

이 기록은 전부 Claude Code·Codex·Cursor 안에서 만들어진 것이다. 그러니 그 단어들은 **모든 종류의 요청에 등장한다.**

```text
"claude code 로 PR 리뷰 해줘"
  매칭된 규칙: skill-maintenance, code-review, git-collaboration, implementation
  보고된 값:   skill-maintenance
```

리뷰 요청인데 스킬 정비로 집계된다. 환경 이름이 작업 내용을 이기는 구조다.

## 그리고 `ci`는 `decision` 안에 있다

짧은 라틴 토큰에 단어 경계가 없었다.

```text
"prefix 를 바꿔줘"         → implementation         "prefix" 안의 fix
"이 decision 을 기록해줘"   → testing-verification   "decision" 안의 ci
"specific 한 조건 알려줘"   → testing-verification   "specific" 안의 ci
```

`ci`는 decision, specific, precision, efficiency, social 안에 다 들어 있다. 영어 단어가 섞인 요청은 내용과 무관하게 검증 작업으로 빠지기 쉽다.

## 원칙 하나로 정리했다

넷을 따로 고치는 대신 한 문장으로 묶었다.

> **사람이 작성하지 않은 문자열은 채점하지 않는다.**

에디터가 붙인 블록도, 붙여넣은 경로도, 링크도 사람이 작성한 산문이 아니다. 채점 전에 가려낸다.

```python
def prose(text: str) -> str:
    """The part a person composed: injected blocks and pasted locators removed."""
    return PASTED_LOCATOR.sub(" ", CONTEXT_BLOCK.sub(" ", text))
```

태그 이름은 **한 개도 열거하지 않는다.** 블록의 모양으로 판단하니 새 태그가 생겨도 그대로 동작한다. 여기에 라틴 토큰 경계를 주고, 규칙에서 도구 이름을 뺐다.

한 가지는 좁게 잡았다. **마스킹은 분류에만 적용한다.** 스킬 탐지는 원문을 읽어야 하기 때문이다. `/blog-draft` 같은 슬래시 호출과 `$work-closeout` 같은 표기가 경로 마스킹에 함께 지워지면 안 된다. 그것도 테스트로 고정했다.

## 좋아 보이지 않는 결과도 남겼다

고치고 나니 `other`로 떨어지는 프롬프트가 늘었다. 분류율이 내려갔고, 도구가 덜 똑똑해 보인다.

그중 하나는 이랬다.

```text
"https://github.com/example/repo/pull/1 리뷰 확인해줘"
  이전 → git-collaboration
  이후 → other
```

이전 값은 URL 안의 `github`이 만든 것이다. 그런데 확인해 보니 review 규칙에는 `코드 리뷰`·`pr 리뷰`·`재리뷰`·`review`는 있어도 **`리뷰` 단독이 없었다.** 붙여넣은 링크가 그 구멍을 대신 메우고 있었던 것이다.

맞는 값이 나오고 있으니 규칙도 맞는 줄 알았던 것이다. 규칙에 `리뷰`를 넣어 실제로 덮게 하고, 이 사실을 PR 본문에 그대로 적었다.

## 배운 것

**1. 도구가 환경을 재고 있으면 그 값은 남과 비교할 수 없다.** 폴더 이름이 통계를 바꾼다면 두 사람의 수치도, 같은 사람의 두 시점도 비교가 안 된다. 재현되지 않는 측정은 측정이 아니다.

**2. 목록을 늘리는 수정은 목록이 낡는 문제를 못 고친다.** 애초에 그 필터가 낡아서 생긴 일인데 항목을 더 얹으면 같은 일이 반복된다. 열거를 없앨 수 있는지부터 본다.

**3. 우연히 맞은 값이 진짜 구멍을 가린다.** `리뷰`가 규칙에 없다는 걸 몰랐던 이유는, 링크가 대신 맞춰 주고 있었기 때문이다. 오탐을 걷어내면 분류율은 떨어지지만 고칠 자리가 보인다.

**4. 부분 문자열 매칭은 토큰이 짧을수록 위험하다.** `ci`, `fix`, `code` 같은 2~4글자 영문에는 단어 경계를 붙인다. 없으면 평범한 문장이 걸린다.

**5. 마스킹은 최소 범위로.** 분류에는 필요하지만 탐지에는 해롭다. 같은 텍스트라도 무엇을 위해 읽는지에 따라 전처리가 달라야 한다.

---

[브랜치 정리 스킬](https://changbaebang.github.io/2026-08-18-branch-cleanup-what-it-costs/)을 만들 때는 잘못된 명령이 5,972라는 숫자를 내놨다. 그때는 **틀린 질문에 정확히 답한** 경우였고, 이번엔 **답이 환경에 오염된** 경우다. 둘 다 화면에는 그럴듯한 값이 떠 있었다.

같은 저장소에 [E2E 검증 스킬을 나눠 넣을 때](https://changbaebang.github.io/2026-08-19-e2e-design-and-execution/)도 결국 같은 이야기를 했다. 통과를 주장하려면 무엇을 근거로 통과라고 하는지가 먼저 있어야 한다. 측정 도구라고 예외는 아니었다.
