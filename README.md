# GroupVault

> 학기 과제: 프로세스에 입각한 바이브코딩의 효과 분석
> [ 홍익대학교 공과대학 ] [ 컴퓨터공학과 ] / [ 2026년 1학기 ]

## 1. 프로젝트 개요

본 프로젝트는 **"바이브코딩으로 작성된 SaaS의 보고된 실패 사례"** 를
프로세스의 입력(reference)으로 활용하여, 동일 실패 모드를 사전에
방지하는 review cycle 기반 멀티 에이전트 개발 프로세스의 효과를
검증하는 것을 목표로 한다.

만들 SW는 소규모 그룹(동아리, 스터디그룹)을 위한 자료 공유 및
멤버 관리 SaaS이며, 일반적인 인증·권한·구독·자료 접근 제어 기능을
포함하므로 보고된 실패 사례의 핵심 vulnerability class를 그대로
검증할 수 있는 도메인이다.

## 2. 문제의식

2026년 상반기 보고된 바이브코딩 SaaS 실패 사례는 두 갈래로 나뉜다.

- **L1 (기능적 정확성 실패)**: 코드는 컴파일되고 happy path에서
  동작하나, 의도된 의미와 다르게 작동 (예: 권한 로직이 반대로 구현)
- **L2 (보안 경계 실패)**: 기능은 정상 작동하나, 의도된 접근 경계를
  지키지 못함 (예: RLS 미적용, client-side only auth)

본 프로젝트는 두 layer 모두를 명시적 acceptance criteria와
agent-level constraint로 다루며, validator가 사례 기반 invariant를
강제하도록 설계한다.

## 3. 참조 사례

본 프로젝트의 acceptance criteria 및 .claude/CLAUDE.md red line은
다음 사례에 anchor되어 있다. 각 사례의 상세 분석은
`docs/case-studies/` 참조.

| 사례 | 분류 | 핵심 실패 모드 |
|---|---|---|
| Lovable CVE-2025-48757 | L2 | DB Row-Level Security 정책 누락 |
| Enrichlead | L2 | client-side only authorization |
| Lovable inverted logic | L1 | access control 로직 반전 |
| Replit AI agent DB wipe | L1 | destructive op의 권한 분리 부재 |
| Moltbook (Wiz disclosure) | L2 | 공개 read/write 권한 default |

## 4. 만들 SW: GroupVault

소규모 그룹용 자료 공유 + 멤버 관리 SaaS.

### 주요 기능
- 그룹 생성 / 가입 / 탈퇴
- 역할: admin / member / guest (RBAC)
- 그룹 내 자료 업로드, 공유, 검색
- 멤버십 tier (free / paid feature 분리)
- 활동 로그

### 기술 스택
- Frontend: Next.js + TypeScript
- Backend: Supabase (PostgreSQL + Auth + RLS)
- 배포: Vercel
- 멀티 에이전트: Claude Code (planner / executor / validator)

### 비기능 요구사항 (사례 기반)
- 모든 DB 테이블에 적절한 RLS policy enabled
- 모든 권한 체크가 server-side에서 enforce됨 (client-side는 UX용만)
- destructive operation은 validator 명시 승인 필요
- 권한 체크 로직은 positive + negative test 모두 통과해야 함

## 5. 프로세스 아키텍처

### 5.1 에이전트 역할 분리 (Process-level RBAC)

| 에이전트 | 권한 | 금지 |
|---|---|---|
| Planner | plan.md 작성/수정 | commit, executor 호출 |
| Executor | 파일 stage (git add) | commit, plan.md 수정 |
| Validator | commit, log.md append, reviews/ 작성 | plan.md 수정 |

### 5.2 상태 외재화

- `plan.md`: 현재 commit unit 명세, AC, 사례 anchor
- `log.md`: append-only 실행 로그, ISO timestamp
- `reviews/commit-N-attempt-M.md`: validator의 review 결과

### 5.3 품질 게이트

- 2-strike Andon cord: validator가 needs_work를 2회 연속 발행 시
  자동 halt → planner 재호출
- 모든 commit은 validator만 수행 (separation of duty)

## 6. 보고서 lessons learned 주제 (작성 예정)

1. 사례 기반 acceptance criteria의 효과
2. Agent-level red line (사전 정의)의 효과
3. 2-layer RBAC (코드 + 프로세스)의 defense in depth
4. append-only log.md의 감사 추적성
5. validator의 isolated context — bias 감소 vs 토큰 비용
6. executor의 commit 권한 박탈의 scope creep 방지 효과
7. 선언적 권한과 시스템적 강제의 차이

## 7. 일정

본 과제는 [ 2026-05-18 ]에 착수하여 [ 2026-06-08 ]에 제출 예정이다.
첫 2주는 다음과 같이 진행한다.

- D1~D2: 사례 분석, red line 정의, threat model
- D3~D5: 요구사항 명세, plan.md 작성, 인프라 셋업
- D6~D11: review cycle로 commit unit 6개 진행
- D12: 통합 테스트, security invariant 자동 검증
- D13: 보고서 작성, lessons learned 정량화
- D14: polish, 제출

## 8. 디렉토리 구조
```
.
├── README.md
├── plan.md                 # planner가 작성
├── log.md                  # append-only 실행 로그
├── .claude/
│   └── CLAUDE.md           # agent red line
├── docs/
│   ├── case-studies/       # 참조 사례 분석
│   │   ├── lovable-cve-2025-48757.md
│   │   ├── enrichlead.md
│   │   ├── lovable-inverted-logic.md
│   │   └── ...
│   ├── threat-model.md
│   ├── requirements.md
│   └── report.md           # 최종 보고서
├── reviews/                # validator 산출물
│   ├── commit-1-attempt-1.md
│   └── ...
├── src/                    # 본 SaaS 소스
└── tests/
```
## 9. 산출물

- 본 SaaS의 동작 캡처
- 프로세스 적용 증빙: 본 repo의 commit history, plan.md, log.md, reviews/
- 보고서 PDF: `docs/report.md`를 최종 PDF로 변환하여 제출

## 10. 라이선스

MIT

---

*본 README는 2026-05-18 작성되었으며, 프로젝트 진행에 따라 갱신된다.*
