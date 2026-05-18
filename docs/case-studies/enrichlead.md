# Case Study: Enrichlead

> 분류: L2 (보안 경계 실패) — 부수적으로 L1 신뢰성 실패 동반
> 핵심 실패 모드: Client-side only authorization, exposed secrets,
>   AI를 통한 사후 수정의 cascade 실패
> 본 프로젝트 영향: AC-2.x (인증/권한), AC-4.x (멤버 권한),
>   .claude/CLAUDE.md red line 다수

## 1. 개요

Enrichlead는 영업 리드 생성을 위한 SaaS 제품으로, 창업자 Leo Acevedo
(@leojr94_)가 2025년 3월 X(구 Twitter)에서 "Cursor AI로 0줄의 직접
코드 작성 없이 구축했다"고 공개적으로 자랑하면서 알려졌다. 게시
2일 만에 본인이 직접 "공격받고 있다"는 후속 글을 올렸고, 일주일
이내 서비스가 완전히 폐쇄됐다.

CVE가 할당되지는 않았으나 (단일 벤더 SaaS의 자체 폐쇄), 본 사례는
바이브코딩으로 production SaaS를 만들 때 발생할 수 있는 복합적
실패 모드의 표준적 사례로 인용된다.

- 사건 시점: 2025년 3월 (Acevedo의 첫 게시 ~ 폐쇄)
- 분류: 자가 공개 incident, 별도 CVE 미할당
- 도구: Cursor AI (보고된 바)
- 결과: 서비스 완전 폐쇄

## 2. 실패 모드 상세

### 2.1 자가 보고된 증상

Acevedo는 폐쇄 직전 X에 다음과 같이 게시했다 (요지):

> "공격받고 있다. 이상한 일들이 일어나고 있고, API 키 사용량이 최대치를
> 찍었고, 사람들이 구독을 우회하고 있고, DB에 임의로 데이터를
> 생성하고 있다."

이 짧은 증언만으로 다음 4개 실패 모드가 동시에 식별된다.

| 증상 | 추정 원인 |
|---|---|
| API 키 사용량 최대치 | API 키가 frontend 코드에 노출됨 |
| 구독 우회 | 권한/구독 체크가 client-side에서만 수행됨 |
| DB 임의 데이터 생성 | server-side 인증 부재, rate limiting 부재 |
| "이상한 일들" | 입력 검증 부재로 인한 데이터 무결성 손상 |

### 2.2 아키텍처적 배경

Lovable의 CVE-2025-48757이 BaaS 플랫폼 default의 실패였다면,
Enrichlead는 그보다 더 원초적인 실패다. **AI(Cursor)는 명시적으로
"보안을 적용하라"고 요청받지 않는 한, 인증·권한·입력검증·rate
limit·secret management 등 어느 하나도 자율적으로 적용하지 않는다.**

결과적으로 AI는 다음 패턴의 코드를 생성한다.

- 권한 체크: `if (user.isPremium) { showFeature() }` 같은 client-side
  JavaScript 분기. 브라우저 dev tool로 `user.isPremium = true` 한 줄로
  우회 가능.
- API 키 관리: OpenAI 등 외부 서비스 키를 frontend 환경변수
  (예: `NEXT_PUBLIC_*` 또는 fetch 호출 헤더에 hardcode) 에 넣어 노출.
- 인증: localStorage 또는 cookie에 "logged in" 플래그만 저장, 서버 측
  세션 검증 부재.
- DB 쓰기: 인증된 사용자라는 가정 하에 모든 POST/PUT 요청을 수락.

### 2.3 AI를 통한 사후 수정의 cascade 실패

본 사례에서 가장 주목할 만한 것은, Acevedo가 사후에 Cursor로 보안
문제를 패치하려 시도했을 때 발생한 현상이다. 공개된 그의 표현은
"Cursor가 다른 부분을 계속 망가뜨린다(Cursor keeps breaking other
parts of the code)"였다.

이는 다음 구조적 문제의 표현이다.

- **컨텍스트 부재**: AI는 전체 코드베이스의 의존성과 invariant를
  완전히 이해하지 못한 채 patch를 생성한다.
- **수정의 회귀**: 보안 패치 한 곳이 다른 happy path를 무너뜨리고,
  그것을 고치는 패치가 또 다른 보안 invariant를 깬다.
- **15,000줄 audit 불가**: 직접 작성하지 않은 코드를 사람이 사후에
  검토하기엔 분량과 이해도 양쪽에서 한계.

결과적으로 patch 시도 자체가 실패의 추가 layer가 되어 서비스
폐쇄에 이르렀다.

## 3. SE 원칙 관점의 분석

| 위반 원칙 | 본 사례에서의 양상 |
|---|---|
| Server-side Enforcement | 권한·구독·인증을 client-side에 의존 |
| Secret Management | API 키를 frontend bundle에 노출 |
| Defense in Depth | 단일 layer(client)에 모든 통제 집중 |
| Input Validation | DB 쓰기 시 유효성 검증 부재 |
| Rate Limiting | API key 단위 호출 제한 부재 |
| Reviewability | 직접 작성하지 않은 코드의 audit 불가 |
| Regression Testing | patch가 다른 부분을 회귀시킴, 자동 회귀 테스트 부재 |

## 4. AI 코드 생성의 구조적 원인

Enrichlead 사례가 보여주는 본질은 다음 한 줄로 요약된다.

**AI는 명시적으로 요청된 기능을 만족하는 코드를 생성하며, 명시적으로
요청되지 않은 보안·신뢰성·운영 invariant는 누락한다.**

이 패턴이 Enrichlead에서 다섯 영역에 동시에 적용됐기 때문에, 단일
취약점이 아닌 systemic failure가 되었다. Lovable 사례가 "default가
잘못된 한 영역"의 실패였다면, Enrichlead는 "default가 적용된 적
없는 다섯 영역"의 동시 실패다.

또한 사후 수정 단계의 cascade 실패는 다음 메타 교훈을 제공한다.

- **반복 가능한 프로세스 없이 만들어진 코드는 반복 가능한 프로세스로
  고칠 수 없다.** Acevedo는 코드를 생성한 도구로 코드를 고치려 했으나,
  생성 단계에서 invariant가 명세되지 않았으므로 수정 단계에서도
  invariant를 보존할 수 없었다.

## 5. GroupVault 프로젝트에 미치는 영향

GroupVault는 멤버십 tier(free/paid feature 분리), 그룹 단위 접근
통제, 외부 서비스 API 키 사용 가능성을 모두 포함하므로, Enrichlead의
실패 모드 5개가 모두 직접적으로 재현 가능한 위험이다.

### 5.1 영향받는 commit unit

- **Commit Unit 2 (인증 + 세션)**: server-side 세션 검증, JWT 또는
  Supabase Auth의 적절한 사용
- **Commit Unit 4 (멤버 권한)**: 모든 권한 체크는 server endpoint
  또는 RLS에 mirror, client-side는 UX용만
- **Commit Unit 5 (외부 통합 / API 사용)**: 외부 서비스 키는 server
  환경변수, 절대 frontend bundle 포함 금지
- **Commit Unit 6 (rate limiting + 입력 검증)**: 모든 mutation
  endpoint에 rate limit, Zod 등으로 입력 schema 검증

### 5.2 적용된 mitigation

본 사례에 대응하여 본 프로젝트는 다음을 강제한다.

1. `.claude/CLAUDE.md` red line:
   - "모든 권한·구독 체크는 server endpoint에 반드시 존재해야 하며,
     client-side는 UX hint로만 사용"
   - "외부 서비스 API 키는 `NEXT_PUBLIC_*` 환경변수로 노출 금지,
     server-only 환경변수만 사용"
   - "Supabase service_role 키는 server 코드에서만 import"
2. Validator invariant:
   - 모든 PR/commit에서 `NEXT_PUBLIC_` 패턴 검색하여 secret-like 이름이
     발견되면 needs_work
   - frontend 코드에서 권한 분기(`isPremium`, `role === 'admin'` 등)
     발견 시 대응하는 server-side 체크가 존재하는지 확인
3. 자동 테스트:
   - 비프리미엄 멤버 토큰으로 paid feature endpoint 호출 → 403 확인
   - 인증 헤더 없이 mutation endpoint 호출 → 401 확인
   - 동일 IP/세션에서 rate limit 초과 호출 → 429 확인
4. 회귀 방지 (cascade 실패 대응):
   - 모든 commit unit은 이전 unit의 테스트가 통과해야 진행
   - validator는 patch가 다른 commit unit의 AC를 깨지 않는지 명시적
     확인

### 5.3 acceptance criteria anchor

다음 AC가 본 사례를 직접 anchor로 한다.

- AC-2.1: 인증되지 않은 요청은 모든 mutation endpoint에서 401
- AC-2.3: 세션 토큰은 server에서 검증, client localStorage 플래그는
  신뢰하지 않음
- AC-4.1: free tier 토큰으로 paid feature endpoint 호출 시 403
- AC-4.2: 다른 그룹의 자료에 대한 write 시도 시 403
- AC-5.1: frontend bundle에 외부 서비스 API 키 포함 금지 (빌드 산출물
  스캔으로 검증)
- AC-6.1: 모든 mutation endpoint에 rate limit 적용
- AC-6.2: 모든 입력은 schema 검증 통과 후에만 처리

## 6. 보고서 lessons 연결

본 사례는 보고서의 다음 lessons learned에 직접 인용된다.

- **"사례 기반 acceptance criteria의 효과"** — Lovable이 단일 layer의
  실패였다면 Enrichlead는 다층 동시 실패. 따라서 AC가 다층(인증·권한·
  secret·rate limit·입력검증)으로 작성되어야 함을 보여줌.
- **"AI 생성 코드의 명시적 요청 부재의 위험"** — Lovable은 "default
  잘못된 것"의 실패, Enrichlead는 "명시되지 않은 것"의 실패.
  프로세스가 acceptance criteria를 통해 명시성을 강제하는 것이 핵심.
- **"반복 가능한 프로세스의 필요성"** — Acevedo의 cascade 실패는 본
  프로젝트의 plan.md / log.md / reviews/ 구조가 왜 필요한지의 negative
  예시. invariant가 외재화되어 있어야 patch가 invariant를 보존할 수 있음.
- **"executor의 commit 권한 박탈"** — Acevedo는 AI에게 직접 commit
  권한을 위임한 셈. 본 프로젝트의 executor는 stage만 가능하고 validator만
  commit. 이 분리가 cascade 회귀를 차단하는 메커니즘.

## 7. Lovable 사례와의 비교

| 항목 | Lovable CVE-2025-48757 | Enrichlead |
|---|---|---|
| 분류 | L2 (보안) | L2 (보안) + 운영 신뢰성 |
| 실패 layer | DB (RLS 누락) | Application 전체 (다층) |
| 도구 | Lovable 플랫폼 | Cursor AI |
| 책임 소재 | 플랫폼 default + 개발자 audit 부재 | 개발자의 명시 요청 부재 |
| CVE | 할당됨 | 미할당 (자체 폐쇄) |
| 결과 | 패치 + 운영 지속 | 서비스 완전 폐쇄 |
| 핵심 교훈 | secure-by-default의 중요성 | 명시적 요구사항의 중요성 |

두 사례는 바이브코딩 SaaS의 실패가 서로 다른 layer에서 발생할 수
있음을 보여주며, 본 프로젝트의 acceptance criteria는 두 layer를
모두 cover하도록 설계된다.

## 8. 참고 자료

### 1차 자료
- Leo Acevedo (@leojr94_) 본인의 X 게시물 시퀀스:
  - https://x.com/leojr94_/status/1900767509621674109 (최초 자랑 게시)
  - https://x.com/leojr94_/status/1900842648119718236
  - https://x.com/leojr94_/status/1901560276488511759 (공격 신고)
  - https://x.com/leojr94_/status/1901802091795927281
  - https://x.com/leojr94_/status/1901979660948267360

### 2차 분석
- Pivot to AI, "Guys, I'm under attack: AI vibe coding in the wild"
  (2025-03-18): https://pivot-to-ai.com/2025/03/18/guys-im-under-attack-ai-vibe-coding-in-the-wild/
- Vibe Graveyard, "Zero hand-written code SaaS app shut down within
  a week": https://vibegraveyard.ai/story/enrichlead-vibe-coded-saas-shutdown/
- Simon Roses Femerling (VULNEX), "Anatomy of a Vibe Coding Breach"
  (2026-04, Part 3 of Vibe Coding Security series):
  https://simonroses.com/2026/04/anatomy-of-a-vibe-coding-breach-lessons-from-2026s-worst-incidents-part-3/

---

*문서 작성일: 2026-05-18*
*최종 수정일: 2026-05-18*