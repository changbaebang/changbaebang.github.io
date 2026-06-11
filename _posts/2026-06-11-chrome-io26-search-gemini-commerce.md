---
layout: post
title: "키워드 검색 이후 — AI Mode·Gemini와 커머스가 받아들일 것"
date: 2026-06-11 15:35:00 +0900
permalink: /2026-06-10-chrome-io26-search-gemini-commerce/
tags: [chrome, ai, search, gemini, commerce, frontend, ucp]
series: [chrome-io26]
---

> [Chrome at I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)의 한 축은 **Gemini in Chrome** — auto browse, 화면 선택, 음성, Android 확장이다.  
> 검색 쪽은 **AI Mode**가 이미 한 세대 앞서 나 있다. 이 글은 **검색·Gemini·커머스**만 다룬다.

[WebMCP 1편](https://changbaebang.github.io/2026-06-10-chrome-io26-webmcp-declarative/)이 **탭 안 행동 계약**을, [HTML-in-Canvas 3편](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)이 **고충실도·capability**를 다뤘다면, 여기서는 AI Mode·Shopping Graph·UCP가 만드는 **발견·비교·결제 표면**을 다룬다.

---

## TL;DR

- “키워드 검색 종말”보다, **쿼리 하나 → 블루링크 목록**이 기본인 패턴이 줄어든다.  
- **AI Overviews** → **AI Mode** → **Gemini in Chrome**은 같은 방향의 **다른 표면**이다.  
- **query fan-out** — 한 질문을 여러 검색으로 쪼개 실행; 쇼핑은 **Shopping Graph** + 자연어.  
- 커머스는 **피드·구조화 데이터 + [UCP](https://ucp.dev/)** (AI 표면 안 거래)와 **WebMCP** (자사몰 탭 안 행동) **이중 레일**.  
- **우리 페이지로 보내기**만이 목표였던 시대는 줄어든다 — **UI 없이도 구매 결정·실행이 가능한 상태**를 만드는 쪽이 생존에 가깝다.

---

## 타임라인 — 검색이 어디까지 왔나

Google 공식 발표만 놓고 보면 대략 이렇다.

| 시기 | 무엇 | 문서 |
|------|------|------|
| 2024~ | **AI Overviews** — SERP 위 생성 요약 + 링크 | Search 팀 발표 다수 |
| 2025 I/O | **AI Mode** 공개 — fan-out·Deep Search·agentic 예약/티켓 | [AI in Search (2025)](https://blog.google/products-and-platforms/products/search/google-search-ai-mode-update/) |
| 2025.09 | AI Mode **시각·쇼핑** — 자연어로 쇼핑, Shopping Graph | [Search AI updates Sep 2025](https://blog.google/products-and-platforms/products/search/search-ai-updates-september-2025/) |
| 2026 I/O | **Gemini in Chrome** 확장 — auto browse, Android, 화면 선택, Skills | [Chrome at I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko) |

**AI Overviews**와 **AI Mode**는 같지 않다.

- Overviews: 기존 검색 결과 **위**에 요약이 붙는 형태. “10%대 사용량 증가” 등 [I/O 2025 글](https://blog.google/products-and-platforms/products/search/google-search-ai-mode-update/)이 인용하는 실험도 이 축.  
- AI Mode: **별도 탭·모드** — 더 긴 추론, 후속 질문, fan-out, 에이전트적 작업.

키워드가 **사라진다**기보다, **첫 접점**이 “10 blue links”에서 **요약·대화·에이전트**로 옮겨간다고 보는 편이 정확하다.

---

## query fan-out — “한 번 검색”이 아니다

[I/O 2025 — AI in Search](https://blog.google/products-and-platforms/products/search/google-search-ai-mode-update/) 설명:

> AI Mode는 **query fan-out** 기법을 사용한다. 질문을 하위 주제로 나누고 **많은 쿼리를 동시에** 실행한다.

사용자는 문장 **하나**를 넣는다.

- “주말 나시빌에서 친구랑 갈 만한 음식점, 우리는 음악이랑 맛집 좋아해”

시스템은 내부적으로:

- 지역·날짜·음악 이벤트·레스토랑 랭킹·영업시간 … **여러 검색**을 병렬로 돌릴 수 있다.  
- **Deep Search**는 fan-out을 더 키워 **수백 검색**·인용 보고서를 만든다.

FE·커머스 관점에서:

- “상위 1페이지 키워드”만 최적화하는 모델은 **부족**해진다.  
- **긴 꼬리 의도·비교·제약**이 한 번에 들어온다 — 사이트 콘텐츠가 **인용·링크 후보**로 읽혀야 한다.  
- 사이트 **내부 검색**도 같은 압력을 받는다. 카테고리·필터 UI만으로는 “barrel jeans that aren’t too baggy”류 질문에 **약**할 수 있다.

---

## 쇼핑 — 필터 UI와 다른 축

[2025년 9월 업데이트](https://blog.google/products-and-platforms/products/search/search-ai-updates-september-2025/):

- 사용자는 필터 칩을 누르기보다 **친구에게 말하듯** 쇼핑한다.  
- 예: “너무 baggy하지 않은 barrel jeans” → **시각적 쇼핑 결과**  
- 후속: “발목 길이 더 길게”처럼 **대화로 다듬기**  
- **Shopping Graph** — 500억+ 리스팅, 리뷰·딜·컬러·재고; 시간당 20억+ 리스팅 갱신(공식 블로그 수치)

[I/O 2025 쇼핑 파트](https://blog.google/products-and-platforms/products/search/google-search-ai-mode-update/) (요지):

- Gemini + Shopping Graph로 **영감·고려사항·상품 좁히기**  
- **가상 피팅** — 단일 이미지로 의류 수십억 건 try-on  
- **agentic checkout** — 가격 조건 맞을 때 Google Pay로 대신 구매, **사용자 감독** 하에

여기서 중요한 구분:

| 축 | 무엇을 최적화하나 |
|----|-------------------|
| **자사몰 필터·정렬 UI** | 이미 **의도가 좁혀진** 사용자 — 사람 경로 |
| **AI Mode 쇼핑** | **의도가 아직 넓은** 사용자 — 그래프·자연어·멀티모달 |
| **둘의 공통분모** | **정확한 상품 데이터** — 제목·속성·이미지·가격·재고·리뷰 |

“PLP 필터를 잘 만든다”와 “AI Mode에 잡힌다”는 **겹치지만 동일하지 않다**. 후자는 **Merchant Center·스키마·그래프 freshness** 쪽 비중이 크다.

---

## Gemini in Chrome — 검색 AI와 한 묶음

[I/O 2026](https://developer.chrome.com/blog/chrome-at-io26?hl=ko) Gemini 항목 요약:

| 기능 | 하는 일 |
|------|---------|
| **auto browse (desktop·Android)** | 예약·주차·**재고 있는 상품** 찾기 등 다단계 작업 |
| **화면 선택** | 페이지에서 영역 지정 → 비교·질문·Nano Banana 편집 |
| **Skills** | 자주 쓰는 프롬프트·멀티탭 워크플로 **원클릭** |
| **음성 입력** | 폼·댓글 — Gemini가 맥락에 맞게 정리 |
| **Android (2026.06~)** | 요약, Keep/Calendar/Gmail, **Personal Intelligence**(옵트인) |

검색 AI Mode와 **겹치는 지점**:

- 사용자 대신 **탐색·비교·폼 작성**  
- 쇼핑·티켓·예약 등 **행동**까지  

**갈라지는 지점**:

- AI Mode — **Google Search·Shopping Graph** 중심  
- Gemini in Chrome — **지금 열린 탭·사이트** 중심 → [WebMCP 1편](https://changbaebang.github.io/2026-06-10-chrome-io26-webmcp-declarative/)과 직결  

사이트에 WebMCP가 없으면 auto browse는 **actuation**(추측 클릭)에 머문다. 커머스는 “검색 유입”과 “탭 안 자동화” **둘 다** 대비하는 편이 낫다.

---

## UCP — AI 표면 안에서의 거래

[Universal Commerce Protocol](https://ucp.dev/)은 Google이 밀어 올리는 **오픈 커머스 표준**이다. [Merchant 가이드](https://developers.google.com/merchant/ucp) 요지:

**무엇을 푸는가**

- AI Mode in Google Search·Gemini 웹(앱 예정)에서 **에이전트 커머스**  
- Merchant Center **기존 피드** + checkout 연동  
- **Merchant of Record는 판매자 유지**

**역량 (UCP 사이트)**

- Catalog Search / Lookup  
- Cart Building  
- Identity Linking  
- Checkout  
- Order Management  
- (확장) 숙박·음식 주문 등 vertical 시나리오  

**생태계 연동**

- **AP2** (Agent Payments Protocol)  
- **A2A** (Agent2Agent)  
- **MCP** — 에이전트 프레임워크와 **호환** (WebMCP·Model Context Protocol과 **이름만 겹칠 뿐 역할 다름**)

FE가 당장 UCP 엔드포인트를 구현하지 않아도, 읽어 둘 메시지:

1. **발견과 결제가 한 표면에** 붙는다 — “클릭 후 이동”만이 아님  
2. **가격·재고·배송·반품** 구조화가 틀리면 AI 경로에서 **바로 신뢰 손실**  
3. 다품목 장바구니·로열티 연동·A/S는 **로드맵** — 지금 가이드는 “direct buying” 시작점  

---

## 커머스 여정 — 세 단계로 나누기

### 1) 발견 (Discovery)

**플랫폼:** AI Mode, Shopping Graph, (선택) UCP catalog  

**준비물:**

- Merchant Center 피드 품질 — GTIN, 가격, availability, 이미지  
- Schema.org Product/Offer — 사이트 HTML과 **피드 정합**  
- 리뷰·딜·컬러웨이 — 그래프 “freshness”  

**FE 역할:**

- 상품 상세·리스트의 **시맨틱 마크업**·메타 일관성  
- **CWV** — 링크로 들어온 사용자 이탈은 여전히 손해  
- 사이트맵·canonical — fan-out이 여러 URL을 볼 때 **중복** 최소화  

### 2) 설명 (Consideration)

**플랫폼:** AI Mode 대화, 가상 피팅, 멀티모달 Lens/Live  

**준비물:**

- 비교 가능한 **스펙·소재·핏** 텍스트  
- 고해상도·다각도 이미지 — 시각 검색·try-on 입력  
- FAQ·사이즈 가이드 — “인용 가능한” 단락 구조  

**FE 역할:**

- 이미지 `alt`·구조화된 스펙 테이블  
- 접근성 있는 비교 UI (사람 경로)  
- [HTML-in-Canvas 3편](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)의 premium 시각화는 **보조** — 기본 정보는 DOM·텍스트로  

### 3) 행동 (Action)

**경로 A — AI 표면 안:** UCP checkout, Google Pay, AP2  

**경로 B — 자사몰:** 링크 클릭 → checkout · **WebMCP** (`add_to_cart`, `apply_coupon` …)  

**경로 C — 브라우저 에이전트:** Gemini auto browse + WebMCP 또는 actuation  

같은 SKU도 **전환 경로가 갈라진다**. Analytics·재고·프로모션 정책을 **경로별**로 생각해야 한다.

---

## 이중 레일 — 실무적으로

### 레일 A: 데이터·콘텐츠 (AI가 우리를 **찾고 인용**하게)

- 피드·structured data 감사  
- 가격·재고 **실시간성** — auto browse·agentic checkout은 틀리면 바로 불만  
- 브랜드 스토리·에디토리얼 — fan-out **긴 질문**에 걸릴 콘텐츠  

### 레일 B: 계약·신뢰 (AI·에이전트가 우리를 **안전하게 실행**하게)

- [WebMCP 1편](https://changbaebang.github.io/2026-06-10-chrome-io26-webmcp-declarative/) — 스키마·확인 UI·`agentInvoked`  
- UCP — 백엔드·Merchant·결제 (FE는 스펙·오류 UX)  
- 약관·반품·개인정보 — **자동 구매** 경로에서 더 중요  

“SEO만” 또는 “MCP만”은 한쪽이 비면 플랫폼이 **한 번 더 추측**한다.

---

## 페이지 진입만이 목줄은 아니다

커머스는 오래도록 같은 목표에 매달렸다. **우리 URL로 들어오게 하고**, **우리 UI에서 행동**하게 하는 것. SERP·광고·푸시·딥링크는 다 그 목줄의 연장이었다. 나는 검색·Gemini·UCP·채팅 안 결제 같은 흐름이 **커머스 생태계 전체**에 큰 영향을 줄 거라 본다.

다만 “쇼핑몰 UI가 사라진다”기보다, **진입의 정의가 갈라진다**에 가깝다.

| 갈래 | 여전히 중요한가 | 무엇을 잡나 |
|------|----------------|-------------|
| **자사몰 UI** | 예 — 옵션이 많고, 브랜드·신뢰가 중요한 구매 | 랜딩 → 전환, 회원·장바구니 |
| **AI·에이전트 표면** | 점점 더 — 의도가 넓을 때, 비교·단순 재구매 | **후보 등록·인용·결제 가능 상태** |

예전에는 “사용자가 **우리 화면을 봐야** 산다”가 전제였다. 이제는 **구매 결정 상태**가 PLP 몇 번 클릭이 아니라, 대화 한 턴·조건 충족·agentic checkout으로 만들어질 수 있다. **우리 페이지에 들어오지 않아도** SKU가 후보·장바구니·결제까지 갈 수 있다.

그래서 중요해지는 건 두 가지다.

1. **결정이 아직 안 난 사용자** — AI 답변·쇼핑 그래프·fan-out **후보 풀**에 우리 SKU가 **인용·비교 가능한 데이터**로 올라가기 (레일 A)  
2. **결정이 거의 난 사용자** — UI 없이도 **실행·결제**가 되게 하기 (레일 B, UCP·WebMCP)

뒤처짐은 “사이트 트래픽이 줄었다”만이 아니다. **후보 풀에서 빠지는 것** — 순위가 아니라 **언급·링크·checkout eligibility**로 재분배되는 쪽이 더 아프다. 사람 UI는 남는다. 다만 **첫 접점·비교·단순 재구매**를 UI 밖에만 맡기면, 파이프 **상단**을 잃기 쉽다.

같은 SKU를 **여러 표면**에 맞게 보내는 능력 — 데이터로 발견되고, 계약으로 실행되는 상태 — 이 생존 조건에 가깝다.

---

## 키워드 검색과 자사 검색 — 같이 볼 것

| | Google AI Mode | 자사몰 검색 |
|--|----------------|-------------|
| 입력 | 자연어·멀티모달 | 키워드·필터·최근 본 |
| 엔진 | Shopping Graph·fan-out | Elasticsearch·자체 인덱스 |
| 목표 | 발견·비교·(UCP) 구매 | 전환·장바구니 |

자사 검색도 **대화형·의도 분해** 압력을 받는다. 다만 **재고·프로모션·회원가**는 자사 데이터가 맞아야 한다. 외부 그래프와 **정합**이 깨지면 AI 유입 사용자가 **더 크게** 실망한다.

---

## 리스크·관측 포인트

**리스크**

- 재고·가격 불일치 — agentic checkout·auto browse에서 **즉시 신뢰 붕괴**  
- AI 답변 안 **인용 누락** — 트래픽이 “순위”가 아니라 **언급·링크**로 재분배; 후보 풀에서 **도태**  
- UCP·WebMCP **이중 구현** 부담 — 우선순위·소유(Merchant vs FE) 정리 필요  

**관측 (가능한 범위에서)**

- AI 유입 세션의 **랜딩 URL·이탈·전환** (UTM·referrer 한계 인정)  
- 자사 도메인 **인용·링크** (서치 콘솔·브랜드 쿼리)  
- auto browse 실패 로그 — actuation vs tool 호출 비율 (장기적으로 WebMCP ROI)  

---

## 맺음

검색은 **키워드 박스**에서 **의도·맥락·행동**으로 넓어졌다. Gemini in Chrome은 그걸 **탭 안**으로 가져온다. 커머스 FE는:

- **리스트·필터·검색 UI** — 사람 경로, 여전히 핵심  
- **피드·스키마·UCP** — AI 표면 발견·결제  
- **WebMCP** — 들어온 뒤 에이전트·auto browse  

세 축이 **한 제품**을 향할 때, “랭킹만 보던 팀”에서 “**데이터 + 계약**도 보는 팀”으로 일이 옮겨간다. **페이지로 끌어들이기**만이 아니라, **UI 밖에서도 우리 SKU가 결정·실행 가능한 상태**를 만드는 쪽으로.

[WebMCP 1편](https://changbaebang.github.io/2026-06-10-chrome-io26-webmcp-declarative/)이 **탭 안 계약**, [HTML-in-Canvas 3편](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)이 **capability·고충실도**를 다뤘다면, 이 글은 그 사이 — **검색·Gemini·UCP로 들어오는 유입** — 을 메운다.

---

## 같은 쪽, 다른 자리

이 글은 Google 쪽을 중심으로 썼다. 다만 **검색·쇼핑·행동을 대화로 옮기려는 시도**는 Google만의 이야기는 아니다. 우열이 아니라 **어디에 붙이느냐**가 다르다.

| | 붙는 위치 | 하는 일 (요지) |
|--|-----------|----------------|
| **Google** | **웹 브라우저 옆** — Chrome·Search·Android에 Gemini | 이미 열린 **탭·SERP** 위에서 auto browse, AI Mode 쇼핑, (UCP) 결제 |
| **OpenAI** | **앱 안** — 사용자 **명령·대화 흐름**에 삽입 | 채팅 안 검색·브라우징, 파트너 몰·Instant Checkout 등 **대화 속 행동** |

Google은 “사이트에 **들어온 뒤**”와 “검색에서 **발견**”을 **브라우저·검색 그래프**에 묶는다. ChatGPT는 “질문하다 **곧바로 행동**”을 **앱 한 화면**에 묶는다. 표면은 갈라져도, 앞에서 본 **피드·스키마·가격·재고**와 **실행 계약**(UCP, WebMCP, 파트너 checkout)에 대한 압력은 비슷하다.

---

## 읽을 거리

### Chrome I/O 2026 시리즈

- [WebMCP와 선언적 API — 1편](https://changbaebang.github.io/2026-06-10-chrome-io26-webmcp-declarative/)
- [새 웹 기능과 옛 WebView — HTML-in-Canvas · 3편](https://changbaebang.github.io/2026-06-09-chrome-io26-html-in-canvas/)

### 이 블로그

- [Google I/O 2026, FE가 봐야 할 6가지](https://changbaebang.github.io/2026-05-22-io2026-fe-what-matters/)
- [에이전트 인터페이스 FE 체크리스트](https://changbaebang.github.io/2026-06-04-agent-interface-for-frontend/)
- [Universal Cart FE Playbook](https://changbaebang.github.io/2026-05-22-universal-cart-fe-playbook/)

### 같은 방향 — 다른 붙는 위치 (참고)

- [Buy it in ChatGPT — Instant Checkout (OpenAI)](https://openai.com/index/buy-it-in-chatgpt/) — 앱·대화 흐름 안 결제 시도 (Google UCP와 **축이 다름**, 우열 비교 아님)

### 공식

- [Chrome at I/O 2026 (ko)](https://developer.chrome.com/blog/chrome-at-io26?hl=ko)
- [AI Mode — I/O 2025](https://blog.google/products-and-platforms/products/search/google-search-ai-mode-update/)
- [AI Mode 시각·쇼핑 (2025.09)](https://blog.google/products-and-platforms/products/search/search-ai-updates-september-2025/)
- [Universal Commerce Protocol](https://ucp.dev/) · [Google Merchant UCP](https://developers.google.com/merchant/ucp)
