---
layout: post
title: "프론트엔드 알람 대응 30분 점검표"
date: 2026-05-19 09:30:00 +0900
tags: [frontend, reliability, timeout, error-handling, cart, observability, checklist]
---

> "어느 주, 모니터링을 보다가 장바구니 p99가 비정상적으로 높게 잡혀 있었다.  
> 그때부터 알람 직후 30분을 따로 점검하기 시작했다."

[tail 지표를 읽는 법](https://changbaebang.github.io/2026-05-17-p99-with-low-sampling-playbook/)을 정리한 뒤,  
[횡단 PR에서 의도를 먼저 맞추는 법](https://changbaebang.github.io/2026-05-17-reviewer-cross-app-pr/)까지 썼다.

그다음에 자연스럽게 남는 질문이 있다.  
**"그래서 코드에서는 뭘 보나?"**

이 글은 그 답이다.  
장바구니·결제처럼 민감한 화면에서 p99나 timeout 비율이 튀었다는 신호가 왔을 때, 프론트가 **우선 확인하는 30분 루틴**을 정리했다. 회사명·제품명은 넣지 않았다.

# 먼저 기준을 맞춘다 (객관 근거 3개)

## 1) p99가 중요한 이유

Google의 고전인 *The Tail at Scale*이 말한 핵심은 단순하다.  
분산 시스템에서는 평균이 멀쩡해도 tail(상위 지연)이 사용자 경험을 지배한다.  
즉 p99는 "과민 지표"가 아니라, 대규모 서비스에서 **실사용자 불편을 먼저 잡는 지표**에 가깝다.

## 2) 샘플링 환경에서의 해석

Datadog 문서 기준으로, APM trace ingestion 샘플링을 낮춰도 trace 기반 메트릭 해석 규칙이 있다.  
다만 실무에서는 **전수 지표(요청·에러·지연)** 와 **샘플 trace(원인 후보)** 를 분리해서 봐야 오판이 줄어든다.  
특히 OTel SDK/Collector 단계 샘플링을 함께 쓰는 팀이라면 더더욱 "무엇이 전수이고 무엇이 표본인지"를 명확히 적어 두는 게 좋다.

## 3) p99 하나로 끝내지 않는다

SRE의 Golden Signals(지연·트래픽·에러·포화) 관점에서 보면, p99는 한 축일 뿐이다.  
p99 + timeout ratio + 사용자 이탈 지표를 묶어야 **조치 우선순위**가 선다.

# 30분의 순서

## 0~5분: 숫자의 단위부터 고정

- 페이지 p99인지, 특정 API resource p99인지
- 집계 창(5분/15분/1시간)
- 같은 구간의 p50·p95·timeout rate·트래픽량

**한 줄 메모:** "느림"인지 "실패"인지부터 갈라 둔다.

## 5~15분: 클라이언트 이벤트를 쪼개서 본다

한 덩어리 `api_error`로만 보면 수정 지점이 안 보인다. 최소한 이렇게 나눈다.

| 구분 | 의미 | 프론트에서 흔한 원인 |
|------|------|----------------------|
| `timeout` | deadline 초과 | 글로벌 timeout, 무한 로딩 |
| `abort` | 요청 취소 | 라우트 전환, 중복 클릭, Strict Mode |
| `transport` | 연결·네트워크 | 오프라인, CORS, 연결 끊김 |
| `5xx` | 서버 오류 | 백엔드 이슈 (프론트는 표현·재시도만) |

장바구니는 **timeout과 abort가 섞이면** p99가 과장되기 쉽다.

## 15~25분: 코드에서 보는 7곳

에이전트에게 "장바구니 API 호출 전부 찾아줘"보다, **항목별로** 본다.

### 1) 요청별 deadline

글로벌 30초 하나로 묶여 있지 않은지.  
요약·가격·쿠폰·재고는 기대 시간이 다르다.

### 2) AbortSignal과 stale 응답

취소된 요청의 응답이 늦게 와서 state를 덮어쓰지 않는지.

```ts
// 패턴 1: 요청 id / generation으로 stale 응답 무시
let gen = 0
async function loadCart() {
  const myGen = ++gen
  const res = await fetchCart({ signal })
  if (myGen !== gen) return // stale
  setState(res)
}
```

```ts
// 패턴 2: TimeoutError / AbortError를 분리해 기록
try {
  await fetch(url, { signal: AbortSignal.timeout(5000) })
} catch (e) {
  if (e instanceof DOMException && e.name === 'TimeoutError') track('api_timeout')
  else if (e instanceof DOMException && e.name === 'AbortError') track('api_abort')
  else track('api_transport')
}
```

### 3) retry와 멱등성

POST·결제성 호출에 무분별 retry가 없는지. GET만 제한적 retry.  
결제·주문 생성처럼 중복 비용이 큰 요청은 idempotency key 전제가 없으면 재시도를 보수적으로 둔다.

### 4) 무한 로딩 UI

`isLoading`이 false로 안 내려가는 분기, `finally` 누락, error 시에도 spinner 유지.

### 5) 라우트 전환 시 in-flight 정리

장바구니 → 결제 이동 시 이전 요청이 살아 있지 않은지.

### 6) 병렬 호출 폭주

마운트 시 N개 API가 동시에 나가 tail을 키우지 않는지. 순서·배치·dedupe.

### 7) 관측 이벤트 이름

`api_timeout` / `api_abort` / `api_5xx`가 분리되어 있는지.  
합쳐져 있으면 다음 주 보고에서 또 헤맨다.

## 25~30분: 한 줄 결론을 PR·티켓에 남긴다

리뷰 글에서 쓰던 **의도 3단락**을 짧게 쓴다.

```md
## 확인 결과 (가설)
- timeout 비율 상승 구간: (리소스/화면)
- 1차 원인 후보: (예: 라우트 전환 후 stale 응답)
- 다음 액션: (예: generation guard / deadline 분리)
- 비목표: (이번 PR에서 안 건드리는 것)
```

숫자만 슬랙에 던지지 말고, **다음 사람이 이어갈 문장**을 남긴다.

# 자주 하는 실수

- trace 1개로 단정한다 → 샘플링 낮을 때 특히 위험
- 백엔드만 태그한다 → abort·무한 로딩은 프론트만의 tail
- timeout을 전부 늘린다 → p99는 잠깐 조용해지고, 사용자 대기만 길어진다

# 최근 실무 동향에서 배운 것 (짧게)

1. **샘플링 정책 분리**  
   전 구간 비용 절감을 위해 head sampling을 쓰더라도, 오류/고지연 trace는 tail sampling 정책으로 별도 보존하는 방식이 늘고 있다.

2. **에러 이름 표준화**  
   `timeout`, `abort`, `network`, `5xx`를 제품 이벤트와 로그 모두에서 같은 이름으로 맞추는 팀이 운영 속도가 빠르다.

3. **주간 보고 포맷 고정**  
   p75/p95/p99 숫자 나열이 아니라 "이번 주 조치 / 다음 주 검증"까지 쓰는 템플릿이 회고와 재발 방지에 유리하다.

# 마치며

알람은 "누가 잘못했나"가 아니라 **"어디를 먼저 볼지"**를 묻는다.  
프론트 30분 루틴이 있으면, p99 글에서 말한 **지표-에러-행동** 세트가 코드 쪽까지 이어진다.

**한 줄 결:** *알람 직후 30분은 구현 전에 관측을 쪼개는 시간이다. timeout·abort·로딩 정지를 나누면, tail은 줄이기 쉬워진다.*

---

## 참고 링크

- Datadog APM sampling/metrics  
  - [Ingestion volume control](https://docs.datadoghq.com/tracing/guide/trace_ingestion_volume_control/)  
  - [Trace Sampling and Storage](https://docs.datadoghq.com/tracing/faq/trace_sampling_and_storage)  
  - [APM Metrics](https://docs.datadoghq.com/tracing/metrics)
- Tail latency / SRE  
  - [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/)  
  - [Monitoring Distributed Systems (Google SRE)](https://sre.google/sre-book/monitoring-distributed-systems/)
- Web/API 및 재시도 설계  
  - [AbortSignal (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)  
  - [Idempotent requests (Stripe)](https://docs.stripe.com/api/idempotent_requests)

