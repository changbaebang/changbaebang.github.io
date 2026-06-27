---
layout: post
title: "앱 WebView에서 장바구니가 많을 때 — 로그가 없으면 FE가 보는 것"
date: 2026-06-27 17:56:00 +0900
permalink: /2026-06-27-webview-large-cart-when-logs-are-quiet/
tags: [webview, cart, frontend, performance, android, debugging, commerce]
series: [webview-capability]
---

> 커머스 앱 WebView에서 **장바구니 개수가 많을 때만** 화면이 멈춘다는 VOC가 들어왔다. Sentry·Firebase에는 아무것도 없었다.  
> 이 글은 그때 팀 채널에서 정리했던 **증상·가설·시도**를 회사 식별 없이 **다른 팀도 쓸 수 있는 체크리스트**로 옮긴 것이다.  
> 선행: [WebView와 UA](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/) · [HTML-in-Canvas·capability](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)

[Universal Cart Playbook](https://changbaebang.github.io/2026-05-22-universal-cart-fe-playbook/)이 **장바구니 UX·신뢰**를 다뤘다면, 여기서는 **앱 WebView·대량 SKU**에서 로그가 비었을 때 FE가 **무엇을 보는지**다.

---

## TL;DR

- **저사양 Android WebView + 장바구니 N개 이상**에서만 터치 무응답·UI 정지가 난다면, JS 예외가 아닐 수 있다 — **메인 스레드·렌더·Long Task**를 먼저 본다.
- **에러 로그가 비어 있어도** “정상”이 아니다. baseline 계정·동일 상품·다른 채널(이전 빌드 등)과 **대조**해야 한다.
- **최근 배포 revert**가 항상 답은 아니다. 가설이 맞지 않으면 **KTLO·후속 티켓**으로 정리하고 당일 소모를 끊는 것도 판단이다.
- **타임아웃만 늘리기**(예: 5초→15초)는 증상 완화일 수 있으나 **렌더 병목**과는 별개다 — **두 축을 섞어 보지 말 것**.

---

## 1. 증상 — VOC가 말해 준 것

고객·CS 경로로 들어온 그림은 대략 이렇다.

| 항목 | 관찰 |
|------|------|
| 환경 | **Android 앱 WebView**, 장바구니 **40~100개 이상** |
| 체감 | 화면 터치 **무응답**, UI가 **멈춘 것처럼** 보임 (영상 첨부) |
| 모니터링 | **Sentry·Firebase 크래시 로그 없음** |
| 대조 | 고액·다건 구매 **흔한 계정**은 상대적으로 정상 |
| 대조 2 | **동일 상품·구버전 baseline** WebView에서는 **정상 동작** |

“앱이 터졌다”가 아니라 **“특정 조합에서만 굳는다”**에 가깝다.

---

## 2. 로그가 없을 때의 함정

FE는 습관적으로 **스택 트레이스**를 찾는다. WebView·대량 리스트에서는 그게 안 나올 수 있다.

- **JS throw 없음** — React error boundary에도 안 잡힘  
- **네이티브 크래시 없음** — 앱은 살아 있고 WebView만 멈춘 느낌  
- **재현이 기기·개수·상품 조합에 묶임** — 평소 쓰는 폰·소량 장바구니로는 안 보임  

그래서 첫 질문은 “버그가 어디 있나?”가 아니라:

> **어느 층에서 시간이 쓰이고 있나?** (렌더 · 스크롤 · 터치 핸들러 · API 대기)

[UA·capability 글](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)에서 말했듯, **채널(WebView vs 브라우저)** 을 먼저 쪼개지 않으면 원인 논의가 섞인다.

---

## 3. 최근 변경과 Feature Flag — 가설 목록

증상 시점 전후 **머지된 변경**을 나열하고, 각각에 대해 **FF 없이도 영향 가능한가**를 묻는다.

당시 의심했던 변경 주제만 적으면:

1. 장바구니 **최대 할인·정렬·WebView 동작 통합** 수정  
2. **브랜드 장바구니 쿠폰** 도입  
3. 장바구니 **뱃지·프로모션 UI 클릭 시 쿠폰 선택 + 스크롤** UX  

BE·플랫폼에 넘긴 질문 예:

- 관련 **Feature Flag**는 현재 ON인가, OFF인가?  
- FF와 **무관하게** 위 변경이 WebView 장바구니 경로에 실리는가?  
- Android WebView + **40개 이상** QA·프로파일링 경험이 있는가?  
- **스크롤 컨테이너 변경** 이후, 대량 아이템에서 **체감 지연**을 본 적 있는가?  

가설은 **하나만 고집하지 않는다**. 로그가 없으면 **병렬로 배제**하는 편이 낫다.

---

## 4. 시도한 것 — revert가 답이 아니었을 때

당일 흐름을 단순화하면:

| 시도 | 결과 | 해석 |
|------|------|------|
| 의심 기능 **revert** | VOC **미해소** | 단일 PR 원인 가설 **약함** |
| API **중복 호출 제거** | **효과 없음** → 추가 진행 **보류** | 병목이 API 한 번이 아닐 수 있음 |
| cart-item **타임아웃 5s→15s** | 에러 **빈도** 완화 기대 | **진입이 느릴 때** UX·CS 대응용; 렌더 정지와 **별 축** |
| WebView **대량 렌더·하이드레이션** | **KTLO 티켓**으로 정리, 당일 인시던트 종료 | 당장 핫픽스 없으면 **다음 사이클**로 명시 |

인시던트에서 중요한 건 “끝까지 원인 규명”만이 아니다. **틀린 가설에 시간을 쓰지 않기**, **고객·CS에 말할 수 있는 다음 액션**을 남기는 것이다.

---

## 5. 같은 상황에서 FE가 할 체크

**재현**

- [ ] Android WebView, **N개 이상** 장바구니 (임계값 기록)  
- [ ] 저사양 기기 또는 **고객 단말과 유사** 프로파일  
- [ ] 동일 계정·상품으로 **브라우저 vs WebView** 대조  

**관측**

- [ ] Performance 탭 · **Long Task**  
- [ ] 스크롤·터치 시 **메인 스레드** (Chrome remote debugging)  
- [ ] **baseline·이전 빌드**와 현재 빌드 **동일 시나리오**  

**조직**

- [ ] FF 상태·최근 머지 목록 **한 페이지**로 공유  
- [ ] revert 전 **“무엇을 증명하려는가”** 한 줄  
- [ ] 당일 못 고치면 **KTLO·후속**에 **재현 조건**까지 적기  

---

## 맺음

WebView는 [이미 쓴 글들](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)처럼 **capability·채널** 문제다. 장바구니는 그 위에 올라가는 **커머스의 극단 케이스**다 — 아이템이 많을수록 **DOM·이벤트·할인 계산**이 한 화면에 겹친다.

이번 VOC에서 나는 세 가지를 다시 배웠다.

**첫째, 에러가 없다는 말은 문제가 없다는 뜻이 아니다.** 크래시 로그가 비어 있으면 습관적으로 “JS 버그”를 찾지만, Android WebView·대량 리스트에서는 **예외 없이 메인 스레드만 잡아먹는** 그림이 나올 수 있다. 첫 질문은 “어디가 틀렸나?”가 아니라 **“시간이 어디서 쓰이나?”** — 렌더, 스크롤, 터치, API 대기 — 에 가깝다.

**둘째, “앱이 터졌다”와 “특정 조합에서만 굳는다”는 다른 사건이다.** WebView·N개 이상·저사양·최근 배포가 겹칠 때는 VOC를 **조합 표**로 적고, 브라우저·baseline 빌드·평소 쓰는 계정과 **대조**하지 않으면 논의가 섞인다. [UA·채널을 먼저 쪼개라](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)는 말이 인시던트 대응에서도 그대로 통한다.

**셋째, 인시던트에서 시도마다 목적이 다르다.** revert는 “이 PR이 원인인가”를 **검증**하는 것이고, API 중복 제거는 “네트워크가 병목인가”를 보는 것이며, 타임아웃 연장은 **진입이 느릴 때 에러·CS 빈도**를 줄이는 **완화**에 가깝다. UI가 멈춘 것과 같은 축이 아니다. revert로 안 풀리면 가설을 버리고, 당일 핫픽스가 없으면 **재현 조건까지 적은 KTLO**로 넘기는 것도 — 원인 규명에 실패한 게 아니라 **틀린 가설에 시간을 쓰지 않기로 한 판단**이다.

로그가 조용할수록 FE는 **“예외가 없다”**고 끝내지 말고, **조합(앱·개수·기기·FF)** 과 **대조 실험**을 남겨야 한다. 다음에 비슷한 VOC가 오면 위 체크리스트를 꺼내면 된다. 그게 다음 주의 나·다른 팀의 시간을 산다.

이 글은 **최종 해결책**을 담지 않는다. **로그가 없을 때 무엇을 배우고, 어떻게 정리하고 넘길지** — 그 판단 과정만 남긴다.

---

## 읽을 거리

### WebView·커머스

- [WebView와 UA](https://changbaebang.github.io/2026-06-07-chrome-user-agent-code-audit-fe-notes/)
- [HTML-in-Canvas·legacy WebView](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)
- [Universal Cart FE Playbook](https://changbaebang.github.io/2026-05-22-universal-cart-fe-playbook/)
