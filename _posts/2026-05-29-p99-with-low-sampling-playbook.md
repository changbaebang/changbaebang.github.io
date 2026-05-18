---
layout: post
title: "샘플링 1% 미만에서 p99를 읽는 법 — 장바구니 tail 지표가 튀었을 때"
date: 2026-05-18 09:00:00 +0900
tags: [datadog, p99, latency, sampling, frontend, cart, reliability, observability]
---

> "p99는 숫자가 아니라 태도다.  
> 평균이 멀쩡한 날에도, 가장 느린 1%를 제품의 진실로 다루겠다는 태도."

어느 주, 모니터링을 보다가 장바구니 p99가 **비정상적으로 높게** 잡혀 있었다.  
그 시기에 **주요 페이지 p75/p95/p99를 매주 상위 보고**하는 흐름도 맞물려 있어서, tail 지표를 어떻게 읽고 대응할지 정리할 필요가 생겼다.

이 글은 그때 내가 실제로 정리한 메모다.  
회사명, 제품명, 팀명은 모두 뺐다. 대신 비슷한 상황을 겪는 팀이 바로 적용할 수 있도록 **운영 플레이북**으로 쓴다.

# 먼저 현실 인정: 샘플링 1% 미만이면 p99는 쉽게 흔들린다

샘플링이 낮을수록 tail 지표는 노이즈에 취약하다.  
특히 트래픽이 낮은 시간대/리소스 단위/엔드포인트 단위로 잘라 보면 더 그렇다.

여기서 많이 하는 오해가 있다.

- "샘플링 낮으니 p99는 버려야 한다"
- "p99가 튀었으니 바로 백엔드 장애다"

둘 다 반만 맞다.

핵심은 **지표 계층을 분리**하는 것이다.

1. **전수 기반 지표(요청 수, 에러율, 전체 지연 분포)**  
2. **샘플 기반 추적(원인 분석용 trace/span)**  

전수 지표로 "진짜로 느려졌는지"를 먼저 확인하고, 샘플 추적으로 "왜 느려졌는지"를 좁힌다.

# p99가 비정상적으로 튀었을 때 30분 플레이북

## 1) 숫자부터 의심하지 말고, 단위를 확인한다

- 페이지 기준 p99인지
- 리소스(특정 API path) 기준 p99인지
- 집계 창(window)이 5분/15분/1시간 중 무엇인지

같은 p99라도 집계 단위가 다르면 의미가 완전히 달라진다.

## 2) 같은 시간대의 p50/p75/p95를 함께 본다

- p50 정상 + p99만 급등: tail 이벤트 가능성
- p75/p95 동반 상승: 전체 구간 성능 저하 가능성

**"p99만 튄다"**는 말만으로는 조치 우선순위를 정할 수 없다.

## 3) timeout/cancel/error를 분리해서 본다

장바구니에서 특히 중요한 건 "느림"보다 "사용자가 포기하게 되는 실패"다.

- timeout (deadline 초과)
- cancel (사용자 이탈/중복 클릭/라우팅 전환)
- transport error (네트워크/연결)
- server error (5xx)

이걸 한 바구니로 보면 원인이 섞여서 액션이 늦어진다.

## 4) 샘플 trace는 "증거"가 아니라 "가설 후보"로 쓴다

샘플링이 낮을 때 trace 1~2개로 단정하면 거의 항상 삐끗한다.  
trace는 방향만 준다. 확정은 지표와 로그로 한다.

# 프론트엔드가 바로 점검할 타임아웃/에러 처리 7개

## 1) 요청별 deadline이 있는가

`fetch`/client wrapper에 글로벌 timeout 하나만 두면 안 된다.  
장바구니 요약, 가격 계산, 쿠폰 검증은 기대 시간이 다르다.

## 2) Abort 이후 후속 상태 업데이트를 막는가

요청은 취소됐는데 stale response가 늦게 도착해 UI를 덮어쓰는 문제는 여전히 잦다.

## 3) retry 정책이 멱등성(idempotency)과 맞는가

- GET: 제한적 retry + jitter 가능
- POST/결제성 요청: 기본 no-retry, 필요 시 idempotency key 전제

## 4) fallback UI가 "로딩 무한"을 만들지 않는가

Skeleton/Spinner가 무한 유지되면, 사용자 입장에서는 100초 timeout과 같다.

## 5) 사용자 메시지가 실패 유형을 구분하는가

"잠시 후 다시 시도" 하나로 통일하면 운영 힌트를 잃는다.

## 6) timeout을 별도 이벤트로 남기는가

`api_timeout`, `api_cancel`, `api_5xx`는 분리되어야 주간 리포트에서 액션이 나온다.

## 7) route 전환 시 in-flight request를 정리하는가

장바구니/결제처럼 상태 변이가 많은 화면에서 누락되면 p99 tail이 과장된다.

# 샘플링이 낮아도 Datadog에서 할 수 있는 것

## A. "전수 관찰"과 "원인 추적"을 대시보드에서 분리

- 보드 1: SLI 보드 (p75/p95/p99 + timeout rate + error rate)
- 보드 2: Trace 보드 (상위 느린 리소스, 대표 span, 최근 배포 상관)

두 보드를 섞으면 회의에서 논점이 자주 엇갈린다.

## B. 핵심 리소스만 샘플링 정책을 올린다

모든 서비스 샘플링을 올리면 비용만 오른다.  
대신 장바구니/체크아웃 핵심 리소스만 타겟팅해 retention을 높인다.

## C. 극단적인 p99 숫자 하나로 끝내지 않는다

다음 3개를 함께 걸어야 운영이 돌아간다.

1. p99 latency
2. timeout ratio
3. conversion-impact proxy (예: checkout step drop)

tail 지표는 제품 지표와 연결할 때 우선순위가 선다.

# 주간 보고 체계( p75/p95/p99 )를 이렇게 바꾸면 덜 괴롭다

매주 p99 상위 보고가 예정되어 있다면, 형식을 먼저 고정하는 게 낫다.

각 페이지마다 5줄만 남긴다.

1. 이번 주 p99 / 지난 주 p99
2. timeout rate 증감
3. top 3 slow resource
4. 이번 주 조치
5. 다음 주 검증 계획

이렇게 해야 "숫자 낭독"이 아니라 "조치 추적"이 된다.

# 이 글의 결론: p99는 측정이 아니라 운영 문제다

샘플링이 낮다는 사실은 핑계도, 공포도 아니다.  
관찰 계층을 나누고, timeout/cancel을 분리하고, 주간 리듬을 고정하면 p99는 충분히 다룰 수 있다.

장바구니 p99가 한 번 크게 튀었을 때의 본질은 "데이터가 나쁘다"가 아니다.  
**느린 1%를 어떻게 제품 책임으로 번역할지**를 묻는 신호다.

**한 줄 결:** *샘플링이 낮아도 p99는 운영할 수 있다. 단, trace를 진실로 믿지 말고 지표-에러-행동을 한 세트로 묶어야 한다.*

---

## 참고 링크

- Datadog trace sampling/ingestion 가이드: [Trace Sampling Use Cases](https://docs.datadoghq.com/tracing/guide/ingestion_sampling_use_cases), [Ingestion volume control](https://docs.datadoghq.com/tracing/guide/trace_ingestion_volume_control/), [Trace Sampling and Storage](https://docs.datadoghq.com/tracing/faq/trace_sampling_and_storage)
- Google의 tail latency 고전: [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/)
- SRE 관측 기본 원칙: [Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

