# SRS v0.33 → v0.34 기술 스택 마이그레이션 플랜

**Document ID:** PLAN-MIGRATION-001  
**Date:** 2026-05-03  
**Base Document:** SRS v0.33 (SRS-001 Rev 0.33)  
**Target Document:** SRS v0.34 (기술 스택 정합성 보완)  
**Author:** AI Architect (Claude)  
**Status:** Draft — 검토 대기

---

## 1. 목적 및 배경

### 1.1 목적

본 문서는 SRS v0.33에 정의된 온체인 사기 방지 플랫폼(On-Chain Fraud Shield Platform)의 기술 아키텍처를, 아래 확정된 기술 스택 제약사항(C-TEC-001~007)에 정합하도록 마이그레이션하는 작업 계획을 정의한다. 동시에 마이그레이션 후에도 MVP 핵심 사용자 경험(가치 전달)이 훼손되지 않음을 검증한다.

### 1.2 확정 기술 스택 (Assumptions & Constraints)

| ID | 영역 | 제약사항 |
|---|---|---|
| C-TEC-001 | 프레임워크 | 모든 서비스는 **Next.js (App Router)** 기반 단일 풀스택 프레임워크로 구현한다. 프론트엔드와 백엔드를 별도 분리하지 않는다. |
| C-TEC-002 | 서버 로직 | 서버 측 로직(DB 접근, API 호출 등)은 Next.js의 **Server Actions** 또는 **Route Handlers**를 사용하여 별도 백엔드 서버 없이 구현한다. |
| C-TEC-003 | 데이터베이스 | **Prisma + 로컬 SQLite**(개발)를 기본으로 하고, 배포 시 **Supabase(PostgreSQL)**를 사용하여 인프라 설정 복잡도를 최소화한다. |
| C-TEC-004 | UI/스타일링 | **Tailwind CSS + shadcn/ui**를 사용하여 AI가 일관된 디자인 코드를 생성하도록 강제한다. |
| C-TEC-005 | LLM 오케스트레이션 | 별도 Python 서버 없이 **Vercel AI SDK**를 사용하여 Next.js 내부에서 직접 구현한다. |
| C-TEC-006 | LLM 모델 | **Google Gemini API**를 기본으로 사용하며, 환경 변수 설정만으로 모델 교체가 가능하도록 SDK 표준 인터페이스를 준수한다. |
| C-TEC-007 | 배포/인프라 | **Vercel 플랫폼**으로 단일화하며, CI/CD 설정 없이 Git Push만으로 배포를 자동화한다. |

### 1.3 마이그레이션 원칙

1. **가치 보존 원칙** — MVP 핵심 사용자 경험(7대 Pain 해결)은 절대 훼손하지 않는다.
2. **구조 단순화 원칙** — 마이크로서비스를 모놀리식으로 전환하되, 코드 내부의 논리적 모듈 분리는 유지한다.
3. **점진적 확장 원칙** — MVP에서 과도한 인프라 요구사항은 현실적 수치로 조정하고, Scale-up 경로를 문서화한다.
4. **AI 네이티브 원칙** — Vercel AI SDK + Gemini를 활용 가능한 기능에 AI 통합 명세를 추가한다.

---

## 2. 변경 영역 총괄 매트릭스

아래 매트릭스는 SRS v0.33의 각 섹션별 변경 유형과 영향도를 정리한다.

| # | SRS 섹션 | 변경 유형 | 영향도 | 관련 C-TEC |
|---|---|---|---|---|
| M-01 | §1.2 Constraints | 추가 | 중 | C-TEC-001~007 전체 |
| M-02 | §3.2.5 Component Diagram | 전면 재작성 | 상 | C-TEC-001, 002 |
| M-03 | §3.2.1~3.2.3 Client Applications | 재구성 | 상 | C-TEC-001, 004 |
| M-04 | §3.3 API Overview | 수정 | 중 | C-TEC-002 |
| M-05 | §3.4 Interaction Sequences | 수정 | 중 | C-TEC-001, 002 |
| M-06 | §4.2 Non-Functional Requirements | 수치 조정 + 측정 경로 변경 | 상 | C-TEC-003, 007 |
| M-07 | §6.1 API Endpoint List | 수정 | 중 | C-TEC-002 |
| M-08 | §6.2 Entity & Data Model | 수정 | 중 | C-TEC-003 |
| M-09 | F7 (TradFi ZK 모듈) | 범위 변경 | 중 | C-TEC-001 |
| M-10 | F9 (Admin Console 5종) | 재정의 | 상 | C-TEC-001, 004 |
| M-11 | 신규: AI/LLM 통합 명세 | 신규 추가 | 중 | C-TEC-005, 006 |
| M-12 | §1.3 용어 정의 | 추가 | 하 | 전체 |

---

## 3. 상세 변경 계획

### M-01. §1.2 Constraints — 기술 스택 제약사항 추가

**작업 내용:** §1.2 Constraints 테이블에 C-TEC-001~007을 신규 행으로 추가한다.

**변경 전:** 기술 스택에 대한 명시적 제약사항 없음 (암묵적으로 마이크로서비스 + AWS 가정)

**변경 후:** C-TEC-001~007을 유형 "기술 제약"으로 추가하고, 검증 방안·시한을 명시한다.

| ID | 제약/가정 | 유형 | 검증 방안 | 시한 |
|---|---|---|---|---|
| C-TEC-001 | Next.js (App Router) 단일 풀스택 | 기술 제약 | 아키텍처 PoC | MVP 착수 전 |
| C-TEC-002 | Server Actions / Route Handlers | 기술 제약 | API 응답 시간 벤치마크 | MVP 착수 전 |
| C-TEC-003 | Prisma + SQLite(dev) / Supabase PostgreSQL(prod) | 기술 제약 | 스키마 마이그레이션 테스트 | MVP 착수 전 |
| C-TEC-004 | Tailwind CSS + shadcn/ui | 기술 제약 | UI 컴포넌트 라이브러리 PoC | MVP 착수 전 |
| C-TEC-005 | Vercel AI SDK (LLM 오케스트레이션) | 기술 제약 | AI 기능 PoC (Agent 분류, 심사 보조) | MVP 개발 중 |
| C-TEC-006 | Google Gemini API (기본 LLM) | 기술 제약 | Gemini API 연동 + 모델 교체 테스트 | MVP 개발 중 |
| C-TEC-007 | Vercel 배포 + Git Push 자동화 | 기술 제약 | 배포 파이프라인 검증 | MVP 착수 전 |

**연쇄 영향:** CON-8(RPC 비용)의 "노드 제공자 고정비 상한선 계약"은 Vercel 환경에서 불필요하므로 "Vercel Function 실행 비용 모니터링"으로 변경한다.

---

### M-02. §3.2.5 Component Diagram — 전면 재작성

**작업 내용:** 마이크로서비스 기반 Component Diagram을 Next.js 모놀리식 아키텍처로 재작성한다.

**현재 구조 (v0.33):**
```
Client Layer (11개 앱) → API Gateway → 13개 Core Services → Data Layer (3개 DB + Redis) → Blockchain Layer
```

**변경 후 구조 (v0.34):**
```
Next.js App (App Router)
├── 프론트엔드 (Tailwind + shadcn/ui)
│   ├── /(public)         — Tier-1 고객용 페이지 (사기 조회/신고, Safe-Name, Warranty)
│   ├── /(dashboard)      — Tier-2 이용기관용 대시보드 (핫라인, Fraud Agent)
│   └── /(admin)          — Tier-3 운영기관용 콘솔 (5개 Route Group)
│       ├── /admin/fraud-intel      — OC-1 사기 정보 관리
│       ├── /admin/safename         — OC-2 Safe-Name 관리
│       ├── /admin/hotline          — OC-3 핫라인·SLA 관리
│       ├── /admin/warranty         — OC-4 Warranty·보증 관리
│       └── /admin/system           — OC-5 시스템 운영
│
├── 서버 로직 (Route Handlers + Server Actions)
│   ├── /api/v1/simulate            — Zero-FP 검증 엔진
│   ├── /api/v1/override            — 핫라인 락 해제
│   ├── /api/v1/fraud/*             — 사기 신고/조회/이의
│   ├── /api/v1/safename/*          — Safe-Name CRUD + 리졸브
│   ├── /api/v1/resolve/*           — NRM 통합 리졸브
│   ├── /api/v1/warranty/*          — Warranty 민팅/클레임
│   ├── /api/v1/kyc/*               — KYC 검증
│   ├── /api/v1/transfer/*          — Pre-Transfer 검증
│   ├── /api/v1/notification/*      — 알림 설정
│   ├── /api/v1/admin/*             — 운영 API
│   ├── /api/v1/agent/*             — Fraud Agent API
│   ├── /api/v1/ai/*                — AI/LLM 엔드포인트 (신규)
│   └── /api/cron/*                 — Vercel Cron Jobs
│       ├── /api/cron/anchor        — 일일 Merkle Root 앵커링
│       ├── /api/cron/collect       — 사기 정보 정기 수집
│       ├── /api/cron/health        — NRM 어댑터 Health Check
│       └── /api/cron/expiry        — Safe-Name 만료 처리
│
├── 서비스 모듈 (lib/)
│   ├── lib/services/               — 비즈니스 로직 (기존 SVC1~15의 논리적 모듈화)
│   │   ├── zero-fp.ts
│   │   ├── hotline.ts
│   │   ├── fraud-report.ts
│   │   ├── safename-registry.ts
│   │   ├── warranty.ts
│   │   ├── fraud-agent.ts
│   │   ├── notification-gateway.ts
│   │   ├── dispute.ts
│   │   ├── nrm.ts                  — Name Resolution Middleware
│   │   ├── onchain-anchor.ts
│   │   ├── simulation-controller.ts
│   │   ├── kyc-verification.ts
│   │   └── chain-asset-compat.ts
│   ├── lib/adapters/               — 외부 연동 어댑터
│   │   ├── nrm/                    — NRM 네이밍 어댑터 (ENS, Unstoppable 등)
│   │   ├── notification/           — 알림 채널 어댑터 (Slack, Email 등)
│   │   ├── fraud-source/           — 사기 정보 소스 어댑터
│   │   ├── kyc-provider/           — KYC 제공자 어댑터
│   │   └── blockchain/             — 블록체인 연동 (ethers.js)
│   ├── lib/ai/                     — Vercel AI SDK 통합 (신규)
│   │   ├── fraud-classifier.ts     — 사기 정보 자동 분류
│   │   ├── dispute-assistant.ts    — 이의 심사 보조
│   │   └── alert-composer.ts       — 알림 메시지 생성
│   └── lib/simulation/             — Simulation Mode 목/스텁
│
├── 데이터 레이어
│   ├── prisma/schema.prisma        — 단일 Prisma 스키마
│   └── (in-memory cache)           — Map 기반 LRU 캐시 (MVP)
│
└── 외부 연동
    ├── Supabase (PostgreSQL)       — 운영 DB
    ├── Vercel Cron                  — 스케줄링
    ├── Blockchain (ethers.js)       — 온체인 앵커링 + Warranty SC
    └── External APIs                — Chainalysis, OFAC, ENS 등
```

**Mermaid Diagram 교체:** 기존 v0.33 §3.2.5의 Mermaid graph를 위 구조에 맞춰 전면 재작성한다. 주요 변경점은 다음과 같다.

1. "API Gateway Layer" 제거 → Next.js Middleware로 대체 (인증·Rate Limit)
2. "Core Service Layer" 13개 서비스 → `lib/services/` 내부 모듈로 전환
3. "Data Layer" 3개 DB + Redis → Prisma 단일 스키마 + 인메모리 캐시
4. "NRM Adapter Registry" / "Notification Channel Adapters" → `lib/adapters/` 디렉토리
5. "Simulation Layer" → `lib/simulation/` + 환경 변수 분기

---

### M-03. §3.2.1~3.2.3 Client Applications — 재구성

**작업 내용:** 별도 앱으로 기술된 Client Applications을 단일 Next.js App 내 라우트 그룹으로 재정의한다.

**변경 요약:**

| 현재 (v0.33) | 변경 후 (v0.34) |
|---|---|
| 사기 주소 신고·조회 플랫폼 (Web) — 별도 앱 | Next.js Route Group: `/(public)/fraud` |
| Safe-Name 서비스 (Web/API) — 별도 앱 | Next.js Route Group: `/(public)/safename` |
| Warranty 위젯 (Invisible UI 팝업) — 별도 위젯 | Next.js 동적 컴포넌트: `components/warranty-widget.tsx` (shadcn Dialog 기반) |
| VASP 연동 SDK/API — 별도 SDK | Route Handlers (`/api/v1/*`) + npm 패키지 (향후) |
| B2B 핫라인 대시보드 (Web) — 별도 앱 | Next.js Route Group: `/(dashboard)/hotline` |
| Fraud Agent 대시보드 (Web) — 별도 앱 | Next.js Route Group: `/(dashboard)/fraud-agent` |
| OC-1~5 관리 콘솔 (Micro Frontend 5개) — 독립 배포 | Next.js Route Group: `/(admin)/[콘솔별 경로]` — 단일 배포, 역할 기반 접근 제어 |

**REQ-FUNC-048 변경:** "독립 배포 가능한 마이크로 프론트엔드 모듈" → "단일 Next.js App 내 역할 기반 접근 제어(RBAC) Route Group으로 구현한다. 각 콘솔은 논리적으로 독립된 Route Group이며, Middleware에서 역할(O1/O2/O3)에 따라 접근을 제어한다."

**UI 기술 명세 추가:** "모든 UI 컴포넌트는 Tailwind CSS + shadcn/ui를 사용하여 구현하며, 일관된 디자인 시스템을 유지한다."

---

### M-04. §3.3 API Overview — 수정

**작업 내용:** API 구현 방식 설명을 Next.js Route Handlers 기반으로 변경한다.

**변경 사항:**

- "내부 REST" 유형 설명에 "(Next.js Route Handlers)" 부기 추가
- API Gateway 관련 제약("API Gateway 로그" 등) → "Next.js Middleware 로그"로 변경
- Rate Limit 구현 방식: "API Gateway" → "Next.js Middleware + Upstash Rate Limit 또는 인메모리 토큰 버킷"
- 인증 방식: "API Key (VASP)" → "Next.js Middleware에서 API Key 검증" (구현 위치만 변경, 로직 동일)

**API 엔드포인트 경로:** 기존 `/api/v1/*` 경로를 그대로 유지한다. Next.js App Router의 `app/api/v1/` 디렉토리에 1:1 매핑된다.

**신규 API 추가:**

| Endpoint | Method | 설명 | 관련 C-TEC |
|---|---|---|---|
| `/api/v1/ai/classify-fraud` | POST | AI 기반 사기 정보 자동 분류 (Vercel AI SDK + Gemini) | C-TEC-005, 006 |
| `/api/v1/ai/dispute-assist` | POST | AI 기반 이의 심사 1차 판정 보조 | C-TEC-005, 006 |
| `/api/cron/anchor` | POST | Vercel Cron — 일일 Merkle Root 앵커링 | C-TEC-007 |
| `/api/cron/collect` | POST | Vercel Cron — 사기 정보 정기 수집 | C-TEC-007 |
| `/api/cron/health` | POST | Vercel Cron — NRM 어댑터 Health Check | C-TEC-007 |
| `/api/cron/expiry` | POST | Vercel Cron — Safe-Name 만료 처리 | C-TEC-007 |

---

### M-05. §3.4 Interaction Sequences — 수정

**작업 내용:** 시퀀스 다이어그램 내 participant를 Next.js 아키텍처에 맞게 변경한다.

**변경 규칙:**
- `API` / `API Engine` → `Route Handler` (예: `POST /api/v1/simulate`)
- `SimCtrl` (Simulation Controller) → `SimCtrl (lib/services)` — 서비스 모듈 내부 함수
- `Cache` (Caching Layer - Redis) → `Cache (in-memory LRU)`
- `FraudDB` / `OffChainDB` 등 → `Prisma (DB)` 단일 표기
- `NotifyGW` (Unified Notification Gateway) → `NotifyGW (lib/services)` — 내부 모듈

시퀀스의 **논리적 흐름은 동일하게 유지**한다. 변경은 participant 이름과 구현 위치 표기에 국한된다.

---

### M-06. §4.2 Non-Functional Requirements — 수치 조정 + 측정 경로 변경

#### M-06a. 성능 수치 현실화

| NFR ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-001 | Zero-FP API p95 ≤ 100ms | p95 ≤ **500ms** | Vercel Serverless cold start (~250ms). Edge Runtime 사용 시 100ms 근접 가능하나, MVP에서는 500ms로 완화. Production Scale-up 시 Edge Runtime 전환 경로 명시 |
| REQ-NF-007 | 10,000 TPS | **100 TPS** (MVP) | Vercel 동시 실행 한도. MVP Simulation Mode에서 이 수준이면 충분. Scale-up 시 Vercel Enterprise 또는 별도 인프라 전환 경로 명시 |
| REQ-NF-008 | 동시 접속 500 유저 (피크 1,000) | 동시 접속 **100 유저** (피크 **300**) | MVP 초기 유저 규모 현실화 |
| REQ-NF-010 | 월간 API 가용성 ≥ 99.99% | ≥ **99.9%** | Vercel Pro SLA 기준. 99.99%는 Vercel Enterprise 필요 |

#### M-06b. 비용 NFR 현실화

| NFR ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-023 | 외부 RPC 비용 ≤ $5,000/월 | ≤ **$500/월** (Production) / **$0** (Simulation) | 무상 소스 우선 정책 + Vercel 환경 비용 절감 |
| REQ-NF-024 | 전체 MVP 월 인프라 비용 ≤ $15,000 (Sim ≤ $5,000) | ≤ **$500/월** (Sim) / ≤ **$2,000/월** (Production MVP) | Vercel Pro + Supabase Pro 기준 현실적 비용. AWS 인프라 대비 대폭 절감 |

#### M-06c. 측정 경로 전면 변경

| 현재 측정 경로 | 변경 후 | 적용 범위 |
|---|---|---|
| Datadog APM: `api.*.latency_p95` | **Vercel Analytics** + **Vercel Speed Insights** | REQ-NF-001~009 전체 |
| Datadog Custom Metric | **Supabase Dashboard** + **앱 내 커스텀 로깅** (Prisma 쿼리 로그) | REQ-NF-011, 014, 018 |
| Datadog Uptime Monitor | **Vercel Uptime Monitoring** (또는 외부 무상: UptimeRobot) | REQ-NF-010 |
| AWS Cost Explorer | **Vercel Usage Dashboard** + **Supabase Usage** | REQ-NF-023, 024 |
| AWS RDS 자동 스냅샷 + S3 | **Supabase 자동 백업** (Point-in-Time Recovery) | REQ-NF-017 |
| Jira SLA 보드 | **앱 내 SLA 대시보드** (Admin Console OC-3) 또는 Linear/Notion | REQ-NF-012, 015 |
| Grafana / Mixpanel | **Vercel Analytics** + **PostHog (무상 티어)** | REQ-NF-030 |
| k6 부하 테스트 | **유지** (외부 도구) | REQ-NF-009 |
| RUM: `widget.popup.load_p95` | **Vercel Speed Insights** (Web Vitals 기반) | REQ-NF-004 |

---

### M-07. §6.1 API Endpoint List — 수정

**작업 내용:**
- 모든 엔드포인트의 "인증" 열에서 "API Gateway"를 "Next.js Middleware"로 변경
- 신규 Cron 엔드포인트 및 AI 엔드포인트 추가 (M-04에서 정의한 항목)
- Rate Limit 구현을 "Upstash Rate Limit (Redis) 또는 인메모리 토큰 버킷"으로 명시

기존 엔드포인트 경로(`/api/v1/*`)는 **변경 없이 유지**한다.

---

### M-08. §6.2 Entity & Data Model — 수정

**작업 내용:**
- "PostgreSQL" 언급을 "Prisma ORM을 통해 접근하며, 개발 환경에서는 SQLite, 운영 환경에서는 Supabase PostgreSQL을 사용한다"로 변경
- 3개 분리 DB(Primary DB, Fraud Address DB, Off-Chain Name Registry DB)를 **단일 Prisma 스키마**로 통합. 테이블은 논리적으로 그룹화하되 물리적으로 동일 DB에 존재
- Redis 관련 언급 제거 → "인메모리 캐시(MVP) 또는 Vercel KV(확장 시)"로 변경
- 모든 엔터티의 PK 타입을 Prisma 호환 형식으로 검토 (string PK 유지 가능, UUID 사용 권장)

**Prisma 스키마 논리적 그룹:**

| 그룹 | 포함 엔터티 |
|---|---|
| Core | VASP, USER, OPERATOR |
| Fraud | FRAUD_ADDRESS, FRAUD_REPORT, FRAUD_DISPUTE, FRAUD_STAGING |
| Safe-Name | SAFE_NAME, NAMING_ADAPTER |
| Warranty | WARRANTY_POLICY, WARRANTY_CLAIM |
| KYC & Compatibility | KYC_VERIFICATION_LOG, CHAIN_ASSET_REGISTRY, TRANSFER_VERIFICATION_LOG |
| Notification | NOTIFICATION_PREFERENCE, NOTIFICATION_LOG |
| System | SIMULATION_CONFIG, HOTLINE_TICKET, AUDIT_LOG |

---

### M-09. F7 (TradFi ZK 모듈) — 범위 변경

**작업 내용:** F7(On-Premise ZK 인프라)은 Next.js 풀스택과 완전히 이질적인 별도 배포물이다. MVP에서의 처리 방침을 명확히 한다.

**변경 방안 (택 1, 권장: Option A):**

- **Option A (권장): Out-of-Scope로 이동** — F7을 IS(In-Scope)에서 제거하고 OS(Out-of-Scope)로 이동. REQ-FUNC-028~030을 "Post-MVP"로 재분류. 사유: C-TEC-001(단일 풀스택) 원칙과 근본적으로 충돌하며, 별도 Go/Rust 바이너리 형태의 폐쇄망 배포물을 Next.js 내에서 구현할 수 없음.

- **Option B: 별도 패키지로 명시적 분리** — F7은 "Next.js 플랫폼과 독립적인 별도 배포 패키지"로 명시하고, SRS 내에서 인터페이스 명세만 유지. 구현은 별도 프로젝트로 진행.

**Stakeholder 영향:** C3(TradFi IT팀장)의 요구가 MVP에서 지원되지 않음. §2.2에 "C3 관련 요구는 Post-MVP에서 별도 패키지로 대응"을 명시.

---

### M-10. F9 (Admin Console 5종) — 재정의

**작업 내용:** "독립 배포 가능한 마이크로 프론트엔드(Micro Frontend)"를 "단일 Next.js App 내 역할 기반 접근 제어(RBAC) Route Group"으로 재정의한다.

**변경 전 (v0.33):**
> 각 콘솔은 독립 배포 가능한 마이크로 프론트엔드(Micro Frontend) 모듈로 설계한다.

**변경 후 (v0.34):**
> 각 콘솔은 단일 Next.js App 내 Route Group으로 구현하며, Next.js Middleware에서 운영자 역할(O1/O2/O3)에 따라 접근을 제어한다. 모든 콘솔은 Tailwind CSS + shadcn/ui 공통 디자인 시스템을 공유한다. 향후 트래픽 증가 또는 조직 분리 필요 시 Route Group 단위로 독립 앱으로 분리 가능한 구조를 유지한다.

**REQ-FUNC-048 Acceptance Criteria 변경:**
- "독립 배포 가능(타 콘솔 무영향)" → "Route Group 단위 코드 분리 유지, 공통 Middleware 기반 RBAC 적용"
- "콘솔 간 전환 ≤ 1초" → 유지 (Next.js App Router의 클라이언트 내비게이션으로 충족)
- "각 콘솔 로드 ≤ 3초" → 유지

---

### M-11. 신규: AI/LLM 통합 명세

**작업 내용:** Vercel AI SDK + Google Gemini API 활용 기능을 SRS에 명세한다.

**신규 섹션: §4.1.X AI/LLM 통합 기능**

| ID | 요구사항 | Priority | 관련 기능 | 구현 방식 |
|---|---|---|---|---|
| REQ-FUNC-AI-001 | Fraud Intelligence Agent가 수집한 사기 정보를 Gemini API로 자동 분류하고 위험도를 추천한다. 자동 승인 룰셋(3소스 교차)에 해당하지 않는 단일 소스 건에 대해 AI 기반 위험도 점수를 산출하여 승인 대기열에 표시한다. | Should | F6, F6-B | Vercel AI SDK → Gemini API. Route Handler `/api/v1/ai/classify-fraud`에서 호출. |
| REQ-FUNC-AI-002 | 이의 신청(Dispute) 건에 대해 AI가 1차 판정 보조 의견을 생성한다. 증빙 해시, 소유권 서명, 기존 신고 이력을 종합하여 "승인 권고 / 기각 권고 / 수동 검토 필요" 3단계 의견을 운영자(O2)에게 제시한다. 최종 판정은 반드시 인간 운영자가 수행한다. | Should | F3-A | Vercel AI SDK → Gemini API. Server Action에서 호출, 결과를 OC-4 콘솔에 표시. |
| REQ-FUNC-AI-003 | 통합 알림 게이트웨이에서 발송하는 알림 메시지의 본문을 AI가 상황에 맞게 자동 생성한다. 알림 유형(긴급 경보, 일반, 마케팅)과 수신자 프로필에 따라 톤·내용을 조절한다. | Could | F8 | Vercel AI SDK → Gemini API. 알림 발송 전 메시지 생성. |

**AI 통합 제약사항:**
- 모든 AI 판정은 "보조(assistant)" 역할이며, 최종 의사결정은 인간이 수행한다.
- Simulation Mode에서는 Mock AI 응답(고정 JSON)으로 대체하여 API 비용을 절감한다.
- AI 모델은 환경 변수(`AI_MODEL_PROVIDER`, `AI_MODEL_NAME`)로 교체 가능하다.
- AI 응답 시간 SLA: p95 ≤ 3,000ms (LLM 호출 특성 반영).

---

### M-12. §1.3 용어 정의 — 추가

| 용어 | 정의 |
|---|---|
| Next.js App Router | Next.js 13+의 파일 시스템 기반 라우팅 방식. Server Components, Route Handlers, Server Actions를 지원 |
| Route Handler | Next.js App Router에서 HTTP API 엔드포인트를 정의하는 서버 사이드 핸들러 (`route.ts`) |
| Server Action | Next.js에서 클라이언트 컴포넌트가 서버 사이드 함수를 직접 호출할 수 있는 RPC 방식의 서버 함수 |
| Prisma | Node.js/TypeScript용 ORM. 스키마 정의, 마이그레이션, 타입 안전한 쿼리를 지원 |
| Supabase | PostgreSQL 기반 오픈소스 BaaS(Backend-as-a-Service). 인증, 스토리지, 실시간 구독 제공 |
| shadcn/ui | Tailwind CSS 기반의 Re-usable UI 컴포넌트 라이브러리. 복사-붙여넣기 방식으로 프로젝트에 통합 |
| Vercel AI SDK | Vercel에서 제공하는 AI 애플리케이션 개발 SDK. 스트리밍, 도구 호출, 멀티 프로바이더 지원 |
| Vercel Cron Jobs | Vercel 서버리스 환경에서 정기 실행되는 스케줄링 함수. `vercel.json`의 cron 설정으로 정의 |
| Route Group | Next.js App Router에서 URL 경로에 영향을 주지 않고 라우트를 논리적으로 그룹화하는 폴더 구조 `(groupName)` |
| RBAC | Role-Based Access Control — 역할 기반 접근 제어 |
| LRU Cache | Least Recently Used Cache — 가장 오래 미사용된 항목을 먼저 제거하는 인메모리 캐시 전략 |

---

## 4. MVP 핵심 사용자 경험 (가치 전달) 훼손 여부 검토

### 4.1 검토 방법론

SRS v0.33 §1.1에 정의된 7대 Pain Point(CORE-1~2, CJM-1~2, EXT-1~2, CORE-3)와 "한 줄 비전"을 기준으로, 기술 스택 전환 후에도 각 Pain에 대한 솔루션이 동등하게 전달되는지를 검증한다.

**한 줄 비전:** "수만 건의 오송금 민원과 오탐지를 해결하고, 사람이 읽을 수 있는 이름 기반 안전 거래와 실시간 사기 주소 필터링, 그리고 에러 시 100% 현금 보상을 보장하는 0.1초 온체인 사기 방지 플랫폼"

### 4.2 Pain Point별 가치 전달 검증 결과

#### CORE-1: 과도한 오탐지로 VASP의 VIP 정상 출금 차단 및 CS 마비

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | Zero-FP 엔진 → 오탐지율 ≤ 0.01% + 10분 SLA 핫라인 |
| 기술 전환 영향 | **훼손 없음.** Zero-FP 로직은 서버 사이드 비즈니스 로직이며, Next.js Route Handler에서 동일하게 동작한다. 포크 시뮬레이션 알고리즘 자체는 프레임워크 무관. |
| 유의사항 | 응답시간 SLA가 100ms → 500ms로 완화되었으나, "0.1초"를 비전에 명시한 부분과 괴리. MVP에서는 "0.5초 이내 검증"으로 비전 문구를 조정하거나, Edge Runtime 도입으로 100ms를 목표하는 Scale-up 경로를 명시해야 한다. |
| **판정** | **✅ 가치 보존 (비전 문구 미세 조정 권장)** |

#### CORE-2: 사람이 인식 불가능한 온체인 주소 구조 — 오송금·피싱 취약

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | Safe-Name 이름 등록 + 이름 기반 송금 + NRM 통합 리졸브 |
| 기술 전환 영향 | **훼손 없음.** 오프체인 레지스트리는 Prisma + Supabase에서 완벽 지원. NRM 어댑터 패턴은 `lib/adapters/nrm/` 디렉토리에서 Strategy 패턴으로 구현. DNS식 비용 모델·생명주기 관리는 Vercel Cron으로 스케줄링. |
| 유의사항 | NRM Adapter Registry의 "코드 배포 없이 설정 기반 등록"은 DB 기반 어댑터 설정으로 구현 가능. 단, 어댑터 코드 자체는 배포 필요 → "어댑터 설정(엔드포인트, TLD 등)은 배포 없이 변경 가능, 신규 어댑터 로직 추가는 코드 배포 필요"로 표현을 정밀화. |
| **판정** | **✅ 가치 보존** |

#### CORE-3: 퍼블릭 SaaS 망 의존으로 TradFi 인가 탈락 위기

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | On-Premise ZK 모듈 → 100% 망분리 가능 |
| 기술 전환 영향 | **F7이 MVP Out-of-Scope로 이동 시 해당 Pain 미해결.** 단, 이는 기술 스택 전환의 결과가 아니라 MVP 범위 조정의 결과. C3(TradFi IT팀장)은 원래 "Should" 우선순위 기능의 대상이었으며, MVP 핵심 타겟 고객(C1, C2, E1)의 경험에는 무영향. |
| 유의사항 | §2.2에 "C3 관련 요구는 Post-MVP 별도 패키지로 대응" 명시 필요. CORE-3 Pain의 해결 시점을 Post-MVP로 명확히 표기. |
| **판정** | **⚠️ 부분 보류 (MVP 타겟 고객 무영향, Post-MVP 경로 명시)** |

#### CJM-1: 구제/보상 수단이 없는 100% 면책 조항

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | Warranty 보증 → 에러 시 최대 $30K 현금 보상 |
| 기술 전환 영향 | **훼손 없음.** Warranty 스마트 컨트랙트는 ethers.js로 Next.js Server Action에서 직접 호출. 컨트랙트 배포·민팅·클레임 로직은 프레임워크 무관. NFT 민팅, 보상 릴리즈 모두 Route Handler에서 처리 가능. |
| 유의사항 | Invisible UI 팝업(Warranty Widget)은 shadcn/ui Dialog 컴포넌트로 구현. "백그라운드 렌더링" 방식은 Next.js 동적 임포트(`next/dynamic`) + 클라이언트 컴포넌트로 대체. |
| **판정** | **✅ 가치 보존** |

#### CJM-2: 사기 주소 신고 접점 부재

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | B2C 웹 기반 사기 주소 신고·조회 플랫폼 |
| 기술 전환 영향 | **훼손 없음.** 웹 기반 신고·조회 플랫폼은 Next.js의 핵심 강점 영역. shadcn/ui로 폼·대시보드를 빠르게 구현 가능. Prisma를 통한 Fraud DB CRUD는 프레임워크 전환에 의해 오히려 개발 속도가 향상될 수 있다. |
| **판정** | **✅ 가치 보존** |

#### EXT-1: 지속되는 해킹 공포 / 무보증 트라우마

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | 사기 주소 사전 조회 → 피해율 감소 + Warranty 보상 보증으로 신뢰 구축 |
| 기술 전환 영향 | **훼손 없음.** CJM-1 + CJM-2의 조합이며, 양쪽 모두 가치 보존 확인됨. |
| **판정** | **✅ 가치 보존** |

#### EXT-2: 기관의 외부 사기정보 수집 역량 부재

| 항목 | 검증 결과 |
|---|---|
| 핵심 가치 | Fraud Intelligence Agent → 멀티소스 사기 정보 통합 수집·조회 |
| 기술 전환 영향 | **훼손 없음.** Agent의 정기 수집은 Vercel Cron Jobs로 구현. 외부 API 호출(Etherscan, MistTrack 등)은 Route Handler에서 fetch로 동일하게 수행. Staging DB + 승인 워크플로우는 Prisma 모델로 구현. AI 기반 자동 분류(M-11)가 추가되어 오히려 가치가 강화된다. |
| **판정** | **✅ 가치 보존 (AI 추가로 가치 강화)** |

### 4.3 v0.33 신규 기능 (KYC·Pre-Transfer·Compatibility Gate) 가치 전달 검증

| 기능 | 검증 결과 | 판정 |
|---|---|---|
| KYC Verification Tier (4단계) | Prisma 모델로 KYC 상태 관리, 외부 KYC API는 `lib/adapters/kyc-provider/`에서 호출. Simulation Mode 시 Mock 응답. **동일 가치 전달.** | ✅ |
| Pre-Transfer Recipient Verification | Route Handler에서 KYC + 호환성 + 사기 DB 교차 조회를 순차 수행. 로직은 프레임워크 무관. **동일 가치 전달.** | ✅ |
| Asset-Chain Compatibility Gate | CHAIN_ASSET_REGISTRY 테이블을 Prisma로 관리. 호환성 검증 로직은 순수 비즈니스 로직. **동일 가치 전달.** | ✅ |
| Verified Badge | SAFE_NAME 테이블의 `verified_badge` 필드. UI 표시는 shadcn Badge 컴포넌트. **동일 가치 전달.** | ✅ |
| Enhanced Verification ($1,000+) | 금액 임계값 기반 분기 로직. 프레임워크 무관. **동일 가치 전달.** | ✅ |

### 4.4 가치 훼손 리스크 요약

| 리스크 | 심각도 | 완화 방안 |
|---|---|---|
| 비전 문구 "0.1초"와 실제 SLA 500ms 괴리 | 중 | (A) 비전 문구를 "0.5초 이내"로 조정, 또는 (B) Edge Runtime 도입으로 100ms 목표 유지 + Scale-up 경로 명시 |
| F7(TradFi ZK) MVP 미지원 → CORE-3 미해결 | 중 | Post-MVP 별도 패키지 로드맵 명시. MVP 타겟 고객(C1, C2, E1) 무영향 확인 |
| 마이크로 프론트엔드 독립 배포 불가 | 하 | Route Group 기반 논리 분리 유지. 향후 분리 가능한 구조 설계. MVP에서 독립 배포 필요성 낮음 (운영팀 소규모) |
| Vercel Cron 실행 한도 (Hobby: 일 1회, Pro: 일 40회) | 중 | Vercel Pro 사용 확인. NRM Health Check(5분 주기)는 외부 Cron(cron-job.org 무상) 또는 Upstash QStash로 보완 |
| 10,000 TPS → 100 TPS 완화 | 하 | MVP Simulation Mode에서 100 TPS 충분. Production 전환 시 Scale-up 경로(Vercel Enterprise, Edge Functions, 또는 별도 인프라) 명시 |

### 4.5 검증 결론

> **기술 스택 전환 후에도 MVP의 7대 Pain Point 중 6개(CORE-1, CORE-2, CJM-1, CJM-2, EXT-1, EXT-2)에 대한 핵심 사용자 경험은 완전히 보존된다.** CORE-3(TradFi 망분리)은 F7의 MVP 범위 조정에 따라 Post-MVP로 이연되나, 이는 MVP 핵심 타겟 고객(C1 VASP CISO, C2 일반 유저, E1 크립토 포비아)의 경험과 무관하다.
>
> 또한 AI/LLM 통합(M-11)의 추가로 Fraud Agent 자동 분류, 이의 심사 보조 기능이 강화되어, 전체적으로 **가치 전달이 보존되거나 강화**된 것으로 판단한다.

---

## 5. 작업 순서 및 의존성

아래 작업 순서는 SRS 문서 수정의 논리적 의존성을 반영한다.

### Phase 1: 기반 변경 (선행 작업)

| 순서 | 작업 ID | 작업 내용 | 의존성 | 예상 분량 |
|---|---|---|---|---|
| 1 | M-12 | §1.3 용어 정의 추가 | 없음 | 소 |
| 2 | M-01 | §1.2 Constraints에 C-TEC-001~007 추가 | 없음 | 소 |
| 3 | M-09 | F7 범위 변경 (Out-of-Scope 이동) 결정 및 반영 | M-01 | 소 |

### Phase 2: 아키텍처 재구성 (핵심 변경)

| 순서 | 작업 ID | 작업 내용 | 의존성 | 예상 분량 |
|---|---|---|---|---|
| 4 | M-02 | §3.2.5 Component Diagram 전면 재작성 | M-01 | 대 |
| 5 | M-03 | §3.2.1~3.2.3 Client Applications 재구성 | M-02 | 중 |
| 6 | M-10 | F9 Admin Console 재정의 (REQ-FUNC-048 변경) | M-02, M-03 | 중 |

### Phase 3: API·데이터 정합성

| 순서 | 작업 ID | 작업 내용 | 의존성 | 예상 분량 |
|---|---|---|---|---|
| 7 | M-04 | §3.3 API Overview 수정 + 신규 API 추가 | M-02 | 중 |
| 8 | M-07 | §6.1 API Endpoint List 수정 | M-04 | 중 |
| 9 | M-08 | §6.2 Entity & Data Model 수정 (Prisma 기준) | M-01 | 중 |
| 10 | M-05 | §3.4 Interaction Sequences participant 변경 | M-02, M-04 | 중 |

### Phase 4: NFR·AI·마무리

| 순서 | 작업 ID | 작업 내용 | 의존성 | 예상 분량 |
|---|---|---|---|---|
| 11 | M-06 | §4.2 NFR 수치 조정 + 측정 경로 전면 변경 | M-01, M-02 | 대 |
| 12 | M-11 | AI/LLM 통합 명세 신규 추가 | M-04 | 중 |
| 13 | — | §5 Traceability Matrix 갱신 (신규 REQ 반영) | M-11 | 소 |
| 14 | — | §6.3 Behavioral Diagrams 내 participant 정합성 확인 | M-05 | 소 |
| 15 | — | 문서 전체 Cross-Reference 검증 + 변경 이력 기재 | 전체 | 소 |

---

## 6. 리스크 및 완화 계획

| # | 리스크 | 영향도 | 발생 가능성 | 완화 방안 |
|---|---|---|---|---|
| R-01 | Vercel Serverless cold start로 Zero-FP 응답시간 SLA 미달 | 중 | 높음 | (1) Edge Runtime 활용 (2) warm-up Cron 설정 (3) SLA를 500ms로 현실화한 상태이므로 Risk 수용 가능 |
| R-02 | Vercel Cron 실행 한도 초과 (5분 주기 Health Check 등) | 중 | 중 | Upstash QStash 또는 외부 무상 Cron 서비스 활용. Vercel Pro 기준 일 40회이므로, 5분 주기(일 288회)는 외부 서비스 필요 |
| R-03 | Supabase Free/Pro 플랜 DB 용량 제한 | 중 | 낮음 | MVP 데이터 규모(시드 데이터 ~11,000건)에서는 Free 플랜으로도 충분. Production 전환 시 Pro 플랜 ($25/월) |
| R-04 | Prisma + SQLite ↔ PostgreSQL 간 SQL 방언 차이 | 낮음 | 중 | Prisma ORM이 추상화하므로 대부분 무영향. JSON 필드 타입 등 일부 차이는 Prisma 스키마 레벨에서 처리 |
| R-05 | AI/LLM 기능의 응답 품질 불확실성 | 낮음 | 중 | AI 판정은 "보조" 역할로 한정. 자동 승인 룰셋(3소스 교차)이 주요 로직이며 AI는 단일 소스 건에만 적용 |
| R-06 | 단일 앱 모놀리스의 빌드 시간 증가 | 낮음 | 중 | Next.js Turbopack 사용. Route Group 기반 코드 분할로 번들 크기 관리 |

---

## 7. 마이그레이션 검증 체크리스트

SRS v0.34 작성 완료 후 아래 체크리스트로 정합성을 최종 검증한다.

| # | 검증 항목 | 검증 방법 | 합격 기준 |
|---|---|---|---|
| V-01 | 모든 In-Scope 기능(IS-1~12)에 대응하는 REQ-FUNC가 존재하는가 | Traceability Matrix 교차 확인 | 누락 0건 (F7 Out-of-Scope 이동 시 제외) |
| V-02 | Component Diagram이 C-TEC-001~007과 정합하는가 | 다이어그램 내 "별도 백엔드 서버", "API Gateway", "Redis" 등 금지 용어 부재 확인 | 금지 용어 0건 |
| V-03 | 모든 NFR 측정 경로에서 "Datadog", "AWS", "Grafana" 참조가 제거되었는가 | 전문 검색 | 잔존 참조 0건 |
| V-04 | API Endpoint 경로가 Next.js App Router 파일 구조와 1:1 매핑 가능한가 | 경로별 `app/api/` 디렉토리 매핑 검증 | 매핑 불가 엔드포인트 0건 |
| V-05 | Entity & Data Model이 단일 Prisma 스키마로 표현 가능한가 | 스키마 초안 작성 + `prisma validate` | 유효성 검증 통과 |
| V-06 | 7대 Pain Point 해결 가치가 보존되는가 | §4 검증 결과 재확인 | 6/7 보존 + 1건 Post-MVP 경로 명시 |
| V-07 | AI/LLM 통합 기능이 Vercel AI SDK + Gemini로 구현 가능한가 | SDK 문서 대조 + 프로토타입 | 구현 불가 기능 0건 |
| V-08 | 시퀀스 다이어그램의 participant가 v0.34 아키텍처와 일치하는가 | 전 시퀀스 다이어그램 검수 | 불일치 0건 |

---

## 8. 참고: 기술 스택별 MVP 구현 매핑 요약

아래는 SRS의 주요 기능이 확정 기술 스택의 어떤 요소로 구현되는지를 한눈에 보여주는 매핑이다.

| SRS 기능 | Next.js 구현 요소 | DB 모델 | 외부 연동 | AI 활용 |
|---|---|---|---|---|
| F1. Zero-FP 엔진 | Route Handler (`/api/v1/simulate`) | FRAUD_ADDRESS, VASP | RPC (ethers.js) / Mock | — |
| F2. 핫라인 대시보드 | Route Group `/(dashboard)/hotline` + Route Handler | HOTLINE_TICKET | PagerDuty | — |
| F3. 사기 신고/조회 | Route Group `/(public)/fraud` + Route Handler | FRAUD_ADDRESS, FRAUD_REPORT | — | — |
| F3-A. 이의 심사 | Route Group `/(admin)/warranty` + Server Action | FRAUD_DISPUTE | — | **REQ-FUNC-AI-002** |
| F4. Safe-Name | Route Group `/(public)/safename` + Route Handler | SAFE_NAME | Blockchain (ethers.js) | — |
| F4-A. NRM | Route Handler (`/api/v1/resolve/unified`) + `lib/adapters/nrm/` | NAMING_ADAPTER | ENS, Unstoppable 등 | — |
| F4-B. 오프체인 레지스트리 | Server Action + Vercel Cron | SAFE_NAME | — | — |
| F4-D. KYC/Compat | Route Handler + `lib/adapters/kyc-provider/` | KYC_VERIFICATION_LOG, CHAIN_ASSET_REGISTRY | 외부 KYC API | — |
| F5. Warranty | 클라이언트 컴포넌트 (shadcn Dialog) + Route Handler | WARRANTY_POLICY, WARRANTY_CLAIM | Blockchain (ethers.js) | — |
| F6. Fraud Agent | Route Handler + Vercel Cron (`/api/cron/collect`) | FRAUD_STAGING, FRAUD_ADDRESS | Etherscan, MistTrack 등 | **REQ-FUNC-AI-001** |
| F7. TradFi ZK | **(Post-MVP 이관)** | — | — | — |
| F8. 알림 GW | `lib/services/notification-gateway.ts` + `lib/adapters/notification/` | NOTIFICATION_PREFERENCE | Slack, Email, KakaoTalk, SMS | **REQ-FUNC-AI-003** |
| F9. Admin Console | Route Group `/(admin)/*` | 전체 | — | — |
| F10. Simulation Mode | 환경 변수 + `lib/simulation/` + Prisma Seed | SIMULATION_CONFIG | Mock/Stub | — |

---

**— End of Document —**
