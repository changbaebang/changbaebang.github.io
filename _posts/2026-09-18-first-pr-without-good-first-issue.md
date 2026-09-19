---
layout: post
title: "good first issue가 사라진 자리에서 첫 PR 찾기"
date: 2026-09-19 15:09:00 +0900
categories: [Engineering, Open Source]
tags: [Open Source, TanStack, TanStack AI, TanStack Query, Upstage, Liner, Documentation, Static Analysis, Contribution]
excerpt: "이슈 목록을 보고 기여할 곳을 고르는 방식은 이제 잘 통하지 않는다. 새 이슈는 며칠 안에 PR이 붙고 초보자용 라벨은 비어 있다. 대신 문서의 코드 블록을 소스와 자동으로 대조하니 아무도 집지 않은 오류가 나왔다."
---

> 개인 계정으로 진행한 오픈소스 기여 이야기다. 저장소 상태 숫자는 2026년 9월 16일, PR 결과는 9월 19일 기준으로 직접 조회한 값이다.

오픈소스에 기여해 보려고 TanStack Query 저장소를 열었다. 익숙한 라이브러리라 코드는 읽을 수 있고, 이슈도 PR도 활발했다. 그런데 한 시간쯤 훑고 나서 든 결론은 "이슈 목록에서 고르는 방식으로는 들어갈 자리가 없다"였다. 이 글은 그 관찰과, 대신 택한 방법을 정리한 것이다.

## 1. 이슈 목록은 이미 소화되어 있었다

TanStack Query의 상태를 먼저 적어 둔다.

| 항목 | 값 |
| --- | --- |
| 오픈 이슈 | 55개 |
| `good first issue` | 0개 |
| `help wanted` | 2개 (하나는 4년 된 스레드) |
| `documentation` 라벨 | 0개 |
| 오픈 PR | 113개 |
| 최근 30일 머지된 PR | 218개 |

머지 속도는 빠르다. 하루 7개꼴로 들어간다. 문제는 오픈 이슈 쪽이다. React 어댑터에 해당하는 최근 이슈를 하나씩 열어 보니 전부 이미 PR이 붙어 있었다. 등록 6일 된 이슈에 PR 하나, 17일 된 이슈에 PR 두 개, 22일 된 이슈에 열린 PR 하나와 닫힌 PR 두 개. 등록 후 하루 이틀 안에 누군가 집어 가고, 같은 이슈에 여러 사람이 PR을 낸다.

PR 제목에 로봇 이모지가 붙은 것이 여럿 보였다. 기여 가이드에는 "자동화 에이전트라면 제목 끝에 🤖🤖🤖를 붙이면 빠르게 처리해 준다"는 문장이 있다. 그 문장 바로 위에는 "AI가 생성한 PR을 대량으로 보내면 스팸으로 간주하고 차단할 수 있다"고 적혀 있다. 빠른 처리 약속은 걸러내기 위한 표식이라고 읽는 게 맞다. 지금 오픈소스 저장소는 사람보다 빠른 참여자를 이미 상대하고 있고, 이슈 목록은 그 참여자들이 먼저 지나간 자리다.

같은 조직의 다른 저장소도 비슷했다. Router는 오픈 이슈 293개에 오픈 PR이 366개로, 큐 자체가 막혀 있었다. Table은 오픈 이슈 45개 중 최근 것에는 역시 PR이 붙어 있었다.

## 2. 문서는 아무도 자동으로 검사하지 않는다

이슈가 안 되면 문서다. 최근 30일 동안 Query에 머지된 문서 PR이 15건 넘게 있었고, 오픈 상태로 남은 것은 한 건뿐이었다. 이슈와 반대로 문서 쪽은 큐가 비어 있었다.

다만 문서를 처음부터 읽어서 오류를 찾는 것은 느리다. Query 문서는 가이드 하나가 2만 5천 자쯤이고, 조사 대상으로 삼은 TanStack AI는 마크다운 파일이 656개, Table은 1,261개였다. 손으로는 안 된다.

대신 한 가지를 자동으로 검사했다. **문서 코드 블록의 `import` 문에 등장하는 심볼이 그 패키지 소스에서 실제로 export되는가.** API 이름이 바뀌었는데 문서가 따라오지 못한 경우, 문서가 존재하지 않는 경로를 안내하는 경우가 이 검사에 걸린다. 저장소를 `docs`와 `packages`만 얕게 받아서 스크립트 하나로 돌렸다. (스크립트는 부록에 있다.)

세 저장소 결과는 이랬다.

- **Query**: import 수준에서는 깨끗했다. 오류는 손으로 읽다가 찾았다.
- **Table**: import, 메서드 호출명, 함수 호출명까지 대조했는데 어긋난 곳이 없었다. 걸린 것은 전부 다른 라이브러리이거나 예제 안의 사용자 정의 코드였다.
- **AI**: 오탐을 걷어내고 한 건이 남았다.

## 3. 첫 번째: 정의되지 않은 변수

Query의 서버 렌더링 가이드에 "의존 쿼리를 미리 가져오기" 절이 있다. 바로 위 클라이언트 예제는 `user?.id`를 쓰는데, 서버 예제는 이렇게 되어 있었다.

```ts
const user = await queryClient.query({ queryKey: ['user', email], queryFn: getUserByEmail })

if (user?.userId) {
  await queryClient.query({
    queryKey: ['projects', userId], // userId는 어디에도 선언되지 않았다
    queryFn: getProjectsByUser,
  })
}
```

`user?.userId`로 검사하고 `userId`를 쓴다. 그 변수는 선언된 적이 없다. 붙여 넣으면 `ReferenceError`다. 재미있는 것은 같은 저장소의 Preact 가이드에 같은 절이 있는데 거기는 이미 `user?.id` / `user.id`로 고쳐져 있었다는 점이다. 한쪽만 고쳐진 채 남아 있던 것이다.

수정은 두 줄이다. 이 PR을 올리기 전에 확인한 것은 세 가지였다. 예제에 쓰인 `queryClient.query()`가 실제로 있는 API인지 (있다. 최근에 추가됐다), 같은 줄을 건드리는 열린 PR이나 닫힌 PR이 없는지, 기여 가이드가 문서 변경에 changeset을 요구하는지 (요구하지 않는다).

## 4. 두 번째: 배포되지 않은 경로를 안내하는 문서

TanStack AI에서 자동 검사에 걸린 한 건은 메모리 어댑터 가이드였다. 어댑터를 직접 만들 때 계약 테스트를 돌리라며 이렇게 안내한다.

```ts
import { runMemoryAdapterContract } from '@tanstack/ai-memory/tests/contract'
```

`tests/contract.ts`는 저장소에 있다. 그러나 패키지의 `files`는 `dist`, `src`, `skills`뿐이고 `exports` 맵에도 그 경로가 없다. 저장소 안의 테스트 두 개만 상대 경로로 쓰고 있었고, 패키지를 설치한 사용자는 이 import를 해석할 수 없다.

같은 저장소의 `ai-persistence`와 `ai-sandbox`는 같은 성격의 스위트를 `src/testkit/`에 두고 `./testkit` 서브패스로 공개하고 있었다. Vite 빌드 엔트리에 넣고 `vitest`를 외부 의존성으로 유지하며, `vitest`를 선택적 peer dependency로 선언하는 방식이다. 즉 이 패키지만 패턴에서 빠져 있었다.

여기서 선택지가 둘이었다. 문서만 고쳐서 "아직 배포되지 않았으니 저장소에서 복사해 쓰라"고 적거나, 옆 패키지와 같은 구조로 실제로 공개하거나. 후자를 골랐다. 선례가 같은 저장소에 두 개 있어서 리뷰어가 판단할 거리가 적고, 문서를 고치는 것보다 문서가 약속한 것을 지키는 편이 낫다고 봤다.

변경은 9개 파일에 30줄 남짓이다. 파일 이동, `exports` 항목, 빌드 엔트리, peer dependency, 저장소 안 테스트 두 개의 import 경로, 가이드와 패키지 스킬 문서, changeset. 로컬에서 테스트 59건, 타입 검사, 빌드, publint, 린트, 포맷, 문서 링크 검사를 돌렸고 빌드 산출물에 `testkit/contract.js`가 `vitest`를 외부에서 가져오는 형태로 나오는 것까지 확인했다.

## 5. 문서 오류가 진입점이 되는 이유

두 건의 공통점은 "코드가 틀린 게 아니라 문서와 코드가 어긋난 것"이다. 이런 오류는 이슈로 잘 올라오지 않는다. 문서를 따라 하다 막힌 사람은 다른 방법을 찾고 넘어간다. 테스트도 잡지 못한다. 문서 코드 블록은 컴파일되지 않는다. 그래서 이슈 목록을 아무리 빨리 훑는 참여자도 여기는 지나간다.

반대로 이 오류는 기계로 찾기 쉽다. 문서 코드 블록에서 import를 뽑아 소스 export와 대조하는 데 필요한 것은 정규식 몇 줄이다. 오탐은 있다. `export type { … }`처럼 여러 줄에 걸친 export 문, 다른 저장소의 패키지, 서브패스 import, 마이그레이션 가이드의 "이전 API" 예제가 걸린다. 그 오탐을 걷어내는 것이 사람이 할 일이고, 남은 것이 진짜다.

이슈를 집어 고치는 것보다 이 방식이 나은 점이 하나 더 있다. 경쟁이 없다. 두 PR 모두 같은 줄을 건드리는 다른 PR이 없었다.

## 6. 결과

사흘 뒤 기준으로 두 PR의 결과는 갈렸다. 작은 쪽이 먼저 들어가리라는 예상과 반대로.

| PR | 저장소 | 크기 | 결과 |
| --- | --- | --- | --- |
| [의존 쿼리 예제의 `userId`](https://github.com/TanStack/query/pull/11504) | Query | +2/−2 | **열림.** CI 통과, 충돌 없음, 사흘째 리뷰 코멘트 없음 |
| [메모리 계약 스위트를 `testkit` 서브패스로](https://github.com/TanStack/ai/pull/1388) | AI | +58/−10 | **병합.** 32시간 만에, 메인테이너 둘 승인 |

두 줄짜리가 기다리고 아홉 파일짜리가 먼저 들어갔다. 크기가 아니라 큐가 결정했다. Query는 오픈 PR이 113개인 저장소라 문서 두 줄도 그 줄에 선다. AI는 봇이 먼저 CI·충돌·changeset을 확인하고 담당 메인테이너를 지정해 주는 저장소였고, 사람 리뷰어는 줄 단위 코멘트 없이 승인했다. 봇이 "E2E 테스트 변경이 없다"고 경고했지만 테스트 스위트를 옮기는 변경이라 그 경고는 적용되지 않았고, 메인테이너도 그렇게 판단했다. 같은 저장소의 선례 둘을 그대로 따른 것이 판단할 거리를 줄였다고 본다.

첫 병합 뒤에 달라진 것이 있다. 같은 패키지의 다음 구멍이 보였고, 진입 비용이 떨어졌다. 다음 날 하루에 문서 PR 셋을 냈다. 미디어 가이드의 import 경로 한 줄, 메모리 패키지에 없던 README, 그리고 OpenAI 호환 어댑터의 지원 제공자 표에 Upstage와 Liner를 추가한 것. 마지막 것은 [내 앱](https://changbaebang.github.io/2026-09-17-career-radar-meets-tanstack-ai/)에서 OpenRouter 다음 제공자를 조사하다 표에 없는 것을 본 김에 낸 세 줄이다. 셋 다 그날 병합됐고, 두 건은 세 시간 안팎이었다. 넷째로 낸 [OpenRouter 라우팅 옵션 문서](https://github.com/TanStack/ai/pull/1418)는 열려 있다. 이건 [어댑터를 내 앱에 붙여 보고](https://changbaebang.github.io/2026-09-18-career-radar-tanstack-openrouter-spike/) 어댑터가 넘기는 옵션을 문서가 다루지 않는다는 것을 알아 채운 82줄이라 위의 셋보다 크고, 리뷰 봇이 제공자 slug 표기 같은 사소한 지적 몇 개를 남겼다.

첫 PR을 고를 때 "이 저장소의 봇과 리뷰어가 어떤 것을 빨리 넘기는가"를 봤다면 순서를 반대로 잡았을 것이다. 다만 Query PR은 틀린 것이 아니라 줄에 서 있을 뿐이니, 닫지 않고 둔다.

## 정리

- 초보자용 라벨은 비어 있고, 새 이슈는 며칠 안에 여러 PR이 붙는다. 이슈 목록은 이제 진입점이 아니다.
- 문서 PR 큐는 비어 있다. 다만 손으로 읽지 말고 문서 코드 블록을 소스와 대조하는 검사를 먼저 돌린다.
- 검사에서 남은 것 중 "문서가 약속했는데 코드가 지키지 않는 것"은 문서만 고치기보다 약속을 지키게 만드는 쪽이 낫다. 같은 저장소의 선례를 따르면 리뷰 부담이 줄어든다.
- PR 전에 확인할 것은 세 가지다. 예제의 API가 실제로 있는지, 같은 줄을 건드린 PR이 없는지, changeset이 필요한지.
- 병합 속도는 변경 크기가 아니라 저장소의 큐와 리뷰 절차가 정한다. 첫 병합이 나면 같은 패키지의 다음 구멍이 보이고 진입 비용이 떨어진다. 첫 PR은 가장 작은 것보다 가장 빨리 판단되는 저장소에 내는 편이 낫다.

## 부록: import 대조 스크립트

저장소를 `docs`와 `packages`만 받아서 돌린다. 오탐이 있으니 결과를 그대로 믿지 말고 하나씩 소스에서 확인한다.

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/TanStack/query.git
cd query && git sparse-checkout set docs packages
python3 check_imports.py .
```

```python
# check_imports.py — 문서 코드 블록의 @scope/* import 심볼을 패키지 export와 대조한다
import re, os, sys, json, glob

root = sys.argv[1]
scope = sys.argv[2] if len(sys.argv) > 2 else '@tanstack'

pkgs = {}
for pj in glob.glob(f'{root}/packages/*/package.json'):
    try:
        pkgs[json.load(open(pj))['name']] = os.path.dirname(pj)
    except Exception:
        pass

def exports(d):
    names = set()
    for f in glob.glob(f'{d}/src/**/*.ts*', recursive=True):
        if '__tests__' in f or '.test.' in f:
            continue
        s = open(f, errors='ignore').read()
        for m in re.finditer(r'export\s+(?:declare\s+)?(?:async\s+)?(?:const|let|var|function\*?|class|type|interface|enum|abstract class)\s+([A-Za-z_$][\w$]*)', s):
            names.add(m.group(1))
        for m in re.finditer(r'export\s*(?:type\s*)?\{([^}]*)\}', s):
            for part in m.group(1).split(','):
                part = re.sub(r'^type\s+', '', part.strip())
                if ' as ' in part:
                    part = part.split(' as ')[1].strip()
                if part:
                    names.add(part)
        if re.search(r'export\s+\*\s+from', s):
            names.add('__STAR__')
    return names

imp_re = re.compile(r"import\s+(?:type\s+)?\{([^}]*)\}\s+from\s+['\"](" + re.escape(scope) + r"/[\w-]+)(?:/[\w./-]*)?['\"]")
cache, bad = {}, {}

for md in glob.glob(f'{root}/docs/**/*.md', recursive=True):
    if '/reference/' in md:
        continue
    s = open(md, errors='ignore').read()
    for block in re.findall(r'```(?:ts|tsx|js|jsx|typescript|javascript|vue|svelte)[^\n]*\n(.*?)```', s, re.S):
        for m in imp_re.finditer(block):
            pkg = m.group(2)
            if pkg not in pkgs:
                continue  # 다른 저장소의 패키지
            if pkg not in cache:
                cache[pkg] = exports(pkgs[pkg])
            ex = cache[pkg]
            if '__STAR__' in ex:
                continue  # re-export는 여기서 판정하지 않는다
            raw = re.sub(r'//[^\n]*', '', m.group(1))
            for n in raw.split(','):
                n = re.sub(r'^type\s+', '', n.strip()).split(' as ')[0].strip()
                if n and n not in ex:
                    bad.setdefault(md, set()).add(f'{pkg}:{n}')

for md, items in sorted(bad.items()):
    print(md.replace(root + '/', ''), '->', ', '.join(sorted(items)))
```

남은 오탐 두 종류는 알아 두면 좋다. 서브패스 import(`@tanstack/ai-angular/ui`)는 패키지 루트 export만 보는 이 스크립트가 잡아내지 못하므로 `package.json`의 `exports`를 직접 본다. 마이그레이션 가이드의 "이전 API" 예제는 의도된 것이니 제외한다.
