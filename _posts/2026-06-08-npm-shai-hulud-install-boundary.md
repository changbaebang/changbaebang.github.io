---
layout: post
title: "빨리 고쳤는데, 왜 위험했는지 — Mini Shai-Hulud와 npm 설치 경계"
date: 2026-06-08 14:35:00 +0900
tags: [security, npm, supply-chain, frontend, lockfile, registry, pnpm]
---

> 배포 일정 앞에서 공지가 왔고, lockfile PR은 빠르게 merge됐다.  
> 그다음 주엔 “그때 뭐가 문제였지?”라는 질문만 남는 경우가 많다.

2026년 5월 **Mini Shai-Hulud** npm 공급망 사건 때, 많은 FE 팀이 비슷한 하루를 보냈다. 보안 advisory가 오고, 영향 패키지 목록이 공유되고, monorepo마다 lockfile을 손봤다. **다행히 배포 일정까지는 막히지 않았다.**

몇 주 뒤 회고 자리에서는 종종 이렇게 말한다.

- “lockfile 지우고 다시 깔라 해서 했어요.”
- “TanStack 버전 올렸어요.”
- “**근데 install할 때 뭐가 위험한 거였죠?**”

이 글은 긴박한 실황 회고가 아니다. **왜 위험했는지**를 차분히 풀고, **빠르게 고친 것**과 **절차 있게 고치는 것**, **재발 방지**를 어떻게 나눌지 정리한다. 사실 관계와 대응 프레임은 [Tenable](https://www.tenable.com/blog/mini-shai-hulud-frequently-asked-questions), [OWASP NPM Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html), [TanStack postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem) 등 공개 글을 참고했다.

## TL;DR

- **Mini Shai-Hulud**는 `@tanstack/*` 등 **믿던 패키지**에 악성 버전을 올리고, **`postinstall` 같은 lifecycle script**로 토큰·시크릿을 노린 공급망 웜이다([CVE-2026-45321](https://nvd.nist.gov/vuln/detail/CVE-2026-45321)).
- 위험의 본체는 “이상한 패키지 이름”이 아니라, **`npm install` 한 번이 외부 코드 실행과 캐시 복제를 허용**한다는 점이다.
- **빨리 merge한 PR**만으로는 부족하다. **(1) 영향 범위 확인 → (2) pin만 surgical bump → (3) store·mirror 정리 → (4) typecheck로 검증** 순서가 재사용 가능한 **절차**가 된다.
- **재발 방지**는 registry 직통 축소, **lockfile + `npm ci`**, **스크립트 allowlist**, **신규 버전 cooldown**처럼 평소에 깔아 두는 **층**이다([ArmorCode](https://www.armorcode.com/blog/defending-against-npm-supply-chain-attacks-a-practical-guide), [OpenReplay](https://blog.openreplay.com/npm-security-best-practices/)).

---

## 1. 무슨 일이었나

| 시기 | 요약 |
|------|------|
| 2025-09~ | Shai-Hulud — npm 대량 오염 ([Snyk 정리](https://snyk.io/articles/npm-security-best-practices-shai-hulud-attack/)) |
| 2026-05-11 | `@tanstack/*` 42패키지·84 악성 버전 — 수 분 내 publish ([NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-45321), [TanStack postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)) |
| 이후 | 많은 조직에서 lock bump, cache 정리, `ignore-scripts` 등 2차 runbook 적용 |

공개 분석이 말하는 공격 체인은 비슷하다. CI **`pull_request_target`·cache poisoning·OIDC 탈취** → trusted identity로 npm publish → **install script**로 확산([Mend.io](https://www.mend.io/blog/mini-shai-hulud-is-back-172-npm-and-pypi-packages-compromised-in-latest-wave/), [Orca](https://orca.security/resources/blog/tanstack-npm-supply-chain-worm/), [Socket](https://socket.dev/blog/tanstack-npm-packages-compromised-mini-shai-hulud-supply-chain-attack)).

일부 버전에는 **SLSA provenance**도 붙었다. “출처 증명이 있다”고 해서 안전한 건 아니다. **파이프라인이 이미 오염**되면 attestation은 *누가 빌드했는가*만 말해 준다([Tenable FAQ](https://www.tenable.com/blog/mini-shai-hulud-frequently-asked-questions)).

---

## 2. 왜 위험했나 — FE가 자주 놓치는 세 가지

“악성 TanStack 버전”은 **표면**이다. 아래가 **구조**다.

### (1) `install` = 코드 실행 기회

악성물은 앱을 import하기 **전**, `preinstall` / `postinstall`에서 돌아간다([CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-mini-shai-hulud-supply-chain-20260503-csa/)).  
로컬 `~/.npmrc`, CI env, GitHub token 같은 **개발·배포 경계 안**을 노린다.

그래서 “앱 코드엔 TanStack API만 쓴다”는 말과 **공급망 위험**은 별개다. **설치 한 번**이면 충분할 수 있다.

### (2) lockfile = 악성 **exact version** 고정핀

`pnpm-lock.yaml` / `package-lock.json`은 semver range가 아니라 **특정 tarball**을 가리킨다.

- advisory에 있는 **버전 번호**가 lock에 남아 있으면, `node_modules`만 지워도 **같은 pin**으로 다시 받는다.
- 사설 **npm mirror**는 upstream이 quarantine해도 **캐시된 tarball**을 잠깐 더 줄 수 있다([Safeguard — pass-through gap](https://safeguard.sh/resources/blog/private-registry-firewall-hardening-2026)).

“registry에서 직접 안 받는다” ≠ “안전하다”. **pin · 로컬 store · org mirror**는 같이 봐야 한다.

### (3) `npm audit`·스캔만으로는 당일을 막기 어렵다

[Marisi Romanillos](https://marisiromanillos.com/blog/mitigating-npm-supply-chain-attacks)가 짚듯, audit은 **advisory가 올라온 뒤**에 강하다. 당일 zero-day malicious tarball에는 **평소 깔아 둔 경계**(cooldown, proxy policy, frozen lockfile)가 맞다.

---

## 3. “후다다닥 고침”과 “절차 있게 고침”

배포 일정이 있으면 **빠르게 PR을 merge**하는 건 맞다. 문제는 **같은 조작을 매번 새로 짜는 것**이다.

### 빠른 대응에서 자주 보이는 패턴

| 패턴 | 단기 | 장기 |
|------|------|------|
| lockfile **통째 삭제** 후 재생성 | install은 될 수 있음 | transitive 대량 이동 → **타입·테스트 2차 장애** |
| `cache clean`만 | 한 겹 정리 | **pnpm store**·mirror는 그대로 |
| “일단 merge” | 배포는 맞춤 | **어떤 버전을 왜 바꿨는지** 기록 없음 |
| 영향 없는 레포까지 동일 runbook | 통제감 | 불필요 diff·리뷰 피로 |

긴박함과 **무질서**는 다르다. 전자는 일정, 후자는 **다음 사고 때 또 처음부터**다.

### 절차 있게 고칠 때의 순서

[OWASP](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html)와 공개 runbook을 FE monorepo에 맞추면 아래 순서를 그대로 재사용할 수 있다.

```text
1. SSOT — advisory의 패키지@버전 목록 (표 한 장)
2. 영향 검색 — lockfile / `pnpm why` / `npm ls` (transitive 포함)
3. 없으면 — lock 무변경, “해당 없음” 기록
4. 있으면 — 해당 exact version만 safe version으로 bump (lock diff 최소)
5. node_modules 삭제 + pnpm store / npm cache 정리
6. (조직 정책) mirror 차단·purge 요청 — pin과 병행
7. frozen install + typecheck + smoke — “설치 성공” ≠ 완료
8. PR 본문 — 패키지, from→to, 검증 명령 3줄
```

**8번**이 다음 사고 때 “왜 위험했는지”를 팀이 기억하게 만든다. [스레드 합의를 티켓 한 문단으로](https://changbaebang.github.io/2026-05-28-thread-to-ticket/)에서 말한 “한 문단 기록”과 같은 층이다.

### pin · store · mirror

| 겹 | 역할 |
|----|------|
| **lockfile** | *무엇을* 설치할지 — bad pin이면 재감염 |
| **로컬 store** | pnpm/npm이 tarball을 *어디에* 보관하는지 |
| **org mirror** | 조직이 upstream을 *어떻게* 복제·캐시하는지 |

runbook에 `rm -rf node_modules && npm cache clean`만 있으면 **store·mirror** 겹이 빠지기 쉽다. 절차 문서에는 **세 줄 모두** 적어 두는 편이 낫다.

---

## 4. 재발 방지 — 사고 **다음 날**부터

“한 번 고치고 끝”이 아니라, **다음 advisory가 왔을 때 1~3절을 다시 읽지 않아도** 되게 만드는 층이다. [ArmorCode](https://www.armorcode.com/blog/defending-against-npm-supply-chain-attacks-a-practical-guide)의 defense-in-depth를 FE 팀 단위로 옮기면 아래와 같다.

### 평소 (레포·로컬)

| 조치 | 왜 |
|------|-----|
| lockfile **항상 commit** | 공급망 **계약서** ([Snyk](https://snyk.io/articles/npm-security-best-practices-shai-hulud-attack/)) |
| CI는 **`npm ci` / frozen lockfile** | install이 lock을 몰래 바꾸지 않게 |
| `.npmrc` registry **한 줄 문서화** | public 직통 vs proxy — 온보딩 |
| lifecycle script **기본 차단 + allowlist** | [Safeguard — install scripts](https://safeguard.sh/resources/blog/mitigate-npm-install-scripts-without-breaking-builds) |
| advisory 대응 PR **템플릿** (패키지·버전·검증) | 빠르게 해도 **절차는 같음** |

### 조직 (플랫폼·보안이 있을 때)

| 조치 | 왜 |
|------|-----|
| **Private registry proxy** — public 직접 egress 차단 | dependency confusion·중앙 정책 ([OWASP](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html)) |
| **min-release-age** / mirror **cooldown** (24~72h) | publish 직후 attack window 완충 ([OpenReplay](https://blog.openreplay.com/npm-security-best-practices/)) |
| mirror **blocklist·악성 feed** | pass-through gap 줄이기 ([Safeguard 2026](https://safeguard.sh/resources/blog/private-registry-firewall-hardening-2026), [Socket firewall](https://docs.socket.dev/docs/socket-firewall-enterprise-registry-mode-upstream-deployment-guide)) |
| publish **OIDC trusted publishing** | 장기 PAT 탈취 면적 축소 |
| runbook에 **mirror purge** 절차 | 2차 조치 때 FE만 cache clean하지 않게 |

### registry.npmjs.org **직접** — 기본값 재검토

예전 기본값:

```text
registry=https://registry.npmjs.org/
npm install → resolve & fetch
```

2026년에 가깝게 권장되는 방향:

```text
registry=https://npm.internal.example.com/   # proxy
min-release-age=…                            # 선택 (npm 11+)
ignore-scripts=true + allowlist              # 선택
npm ci in CI                                 # 필수에 가깝게
```

“직접 가져오기”는 **편함**과 **공격 창을 그대로 받기**의 교환이다. 재발 방지는 **매 사고마다 영웅이 나오는 것**이 아니라, **평소 경계가 얇게 겹치는 것**이다.

---

## 5. 다음 advisory가 왔을 때 — FE 체크리스트

배포 일정이 있어도 **순서는 바꾸지 않는** 쪽이 낫다.

**범위**

- [ ] SSOT 표와 lockfile **exact version** 대조
- [ ] monorepo **workspace**·transitive까지
- [ ] **해당 없음** 레포는 손대지 않기 — diff 최소

**수정**

- [ ] 악성 version만 bump — lock **통째 삭제는 최후**
- [ ] `node_modules` + **store/cache** + (필요 시) mirror

**검증**

- [ ] frozen install
- [ ] typecheck · smoke
- [ ] PR에 **from → to · 검증 명령**

**사후 (같은 주)**

- [ ] “왜 위험했는지” wiki·블로그 **한 문단**
- [ ] 깨진 **`ignore-scripts` allowlist** 보완
- [ ] cooldown·proxy **미적용**이면 티켓 하나

---

## 6. AI·에이전트 (짧게)

[CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-shai-hulud-ai-supply-chain-20260517-csa-st/)는 Mini Shai-Hulud가 **CI·dev credential**을 노린다고 본다. 에이전트가 lock bump PR을 빠르게 만들어 줄 때는, [하네스](https://changbaebang.github.io/2026-05-24-claude-code-harness-living-docs/)에 **“advisory PR = 버전 diff + 검증 명령 필수”** 한 줄을 두면 **절차**가 지켜진다. 빠름과 무질서를 구분하는 최소 장치다.

---

## 7. 마치며

Mini Shai-Hulud 때 많은 팀이 **잘 대응했다.** 배포도 맞췄다. 그건 인정할 일이다.

다만 “고쳤다”와 “**왜 위험했는지 안다**”는 다르다. 전자만 반복하면 다음 공지에도 **lock 지우기·cache clean**만 또 외워 둔다.

- 위험의 중심은 **registry에서 tarball을 받는 순간**과 **install script**다.
- 대응의 중심은 **속도 + 같은 순서 + 최소 diff + 기록**이다.
- 재발 방지는 **사고 당일의 영웅**이 아니라 **평소의 proxy·lockfile·cooldown·allowlist**다.

**한 줄 결:** *빨리 merge한 PR은 그날을 살리고, **절차와 경계**는 다음 advisory를 설명 가능하게 만든다.*

---

## 참고

### 사건·CVE

- [CVE-2026-45321 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2026-45321)
- [TanStack npm supply-chain postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)
- [Mini Shai-Hulud FAQ (Tenable)](https://www.tenable.com/blog/mini-shai-hulud-frequently-asked-questions)
- [TanStack compromise (Orca)](https://orca.security/resources/blog/tanstack-npm-supply-chain-worm/)
- [172 packages wave (Mend.io)](https://www.mend.io/blog/mini-shai-hulud-is-back-172-npm-and-pypi-packages-compromised-in-latest-wave/)
- [TanStack (Socket)](https://socket.dev/blog/tanstack-npm-packages-compromised-mini-shai-hulud-supply-chain-attack)

### 실무·가이드

- [OWASP NPM Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html)
- [Snyk — npm security after Shai-Hulud](https://snyk.io/articles/npm-security-best-practices-shai-hulud-attack/)
- [OpenReplay — npm security best practices](https://blog.openreplay.com/npm-security-best-practices/)
- [ArmorCode — defending npm supply chain](https://www.armorcode.com/blog/defending-against-npm-supply-chain-attacks-a-practical-guide)
- [Marisi Romanillos — mitigating from the trenches](https://marisiromanillos.com/blog/mitigating-npm-supply-chain-attacks)
- [Safeguard — install scripts](https://safeguard.sh/resources/blog/mitigate-npm-install-scripts-without-breaking-builds)
- [Safeguard — private registry hardening 2026](https://safeguard.sh/resources/blog/private-registry-firewall-hardening-2026)
- [CSA — Shai-Hulud & AI supply chain](https://labs.cloudsecurityalliance.org/research/csa-research-note-shai-hulud-ai-supply-chain-20260517-csa-st/)

### 이 블로그

- [RSC 보안 FE 체크리스트](https://changbaebang.github.io/2026-05-31-rsc-security-fe-checklist/)
- [스레드 합의 → 티켓 한 문단](https://changbaebang.github.io/2026-05-28-thread-to-ticket/)
