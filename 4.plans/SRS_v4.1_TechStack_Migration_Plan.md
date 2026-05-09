# SRS v4.1 Tech Stack Migration Plan

**Document ID:** PLAN-MIGRATION-001  
**Base Document:** SRS-001 Rev 0.35 (2026-05-07)  
**Target Revision:** SRS v4.1  
**Date:** 2026-05-09  
**Purpose:** SRS v0.35의 엔터프라이즈 분산 아키텍처를 MVP 관점의 Next.js 단일 풀스택 프레임워크로 전환하기 위한 변경 계획 및 기능 커버리지·사용자 가치 영향 분석

---

## 1. Target Tech Stack (변경 목표 기술 스택)

| ID | 제약 | 설명 |
|---|---|---|
| C-TEC-001 | Next.js (App Router) 단일 풀스택 | 프론트엔드·백엔드를 단일 프레임워크로 통합. 별도 서버 분리 금지 |
| C-TEC-002 | Server Actions / Route Handlers | DB 접근·API 호출 등 서버 측 로직을 Next.js 내부에서 구현. 별도 백엔드 서버 없음 |
| C-TEC-003 | Prisma + SQLite(로컬) / Supabase PostgreSQL(배포) | 로컬 개발 시 SQLite, 배포 시 Supabase(PostgreSQL)로 DB 단일화 |
| C-TEC-004 | Tailwind CSS + shadcn/ui | 일관된 디자인 시스템 강제 |
| C-TEC-005 | Vercel AI SDK | LLM 오케스트레이션을 Next.js 내부에서 직접 구현 (별도 Python 서버 없음) |
| C-TEC-006 | Google Gemini API | 기본 LLM. 환경 변수로 모델 교체 가능하도록 SDK 표준 인터페이스 준수 |
| C-TEC-007 | Vercel 배포 | CI/CD 설정 없이 Git Push 자동 배포. 인프라 관리 단일화 |

---

## 2. Migration Scope Summary (변경 범위 요약)

### 2.1 변경 규모 개요

| 구분 | 항목 수 | 변경 유형 |
|---|---|---|
| 아키텍처 전면 재구성 | Component Diagram, Interaction Sequences 전체 | 구조 변경 |
| Constraints 신규 추가 | C-TEC-001 ~ C-TEC-007 (7건) | 신규 |
| Functional Requirements 수정 | 약 15건 (구현 방식 변경) | 수정 |
| Functional Requirements 축소 | 약 5건 (MVP 시뮬레이션 전환) | 축소 |
| Non-Functional Requirements 수정 | 약 20건 (성능·인프라 기준 완화) | 수정 |
| Non-Functional Requirements 삭제 | 약 5건 (스택 무관 항목) | 삭제 |
| Entity/Data Model | 통합·단순화 | 수정 |
| API Endpoint List | 엔드포인트 유지, 구현 방식 변경 | 수정 |
| LLM 통합 요구사항 | 신규 정의 필요 | 신규 |

---

## 3. Detailed Migration Plan (상세 변경 계획)

### 3.1 아키텍처 변경 (§3 System Context and Interfaces)

#### 3.1.1 Component Diagram 전면 재작성 (§3.2.5)

**현재 (AS-IS):**

```
Client Layer (Tier-1/2/3 별도 앱)
    → API Gateway Layer (인증·Rate Limit·라우팅)
        → Core Service Layer (SVC1~SVC17, 16개 독립 서비스)
            → Data Layer (PostgreSQL + Fraud DB + Off-Chain Registry DB + Redis)
                → Blockchain Layer (3개 스마트 컨트랙트)
                    → NRM Adapter Registry (5개 어댑터)
                        → Notification Channel Adapters (5개 어댑터)
                            → External Integration Layer (8개 외부 시스템)
                                → Simulation Layer (Mock RPC, Stub DB, Testnet)
```

**변경 후 (TO-BE):**

```
Next.js App Router (단일 애플리케이션)
├── app/
│   ├── (public)/           ← Tier-1 고객용 페이지
│   │   ├── fraud-lookup/   ← 사기 주소 조회·신고
│   │   ├── safe-name/      ← Safe-Name 등록·리졸브·송금
│   │   └── warranty/       ← Warranty 위젯 (오프체인 시뮬레이션)
│   ├── (dashboard)/        ← Tier-2 이용기관용 페이지
│   │   ├── hotline/        ← 핫라인 대시보드
│   │   └── fraud-agent/    ← Fraud Intelligence Agent
│   ├── (admin)/            ← Tier-3 운영기관 관리 콘솔 (라우트 그룹)
│   │   ├── oc-1/           ← 사기 정보 관리
│   │   ├── oc-2/           ← Safe-Name 관리
│   │   ├── oc-3/           ← 핫라인·SLA 관리
│   │   ├── oc-4/           ← Warranty·보증 관리
│   │   └── oc-5/           ← 시스템 운영
│   └── api/v1/             ← Route Handlers (REST API 엔드포인트)
│       ├── simulate/
│       ├── fraud/
│       ├── resolve/
│       ├── safename/
│       ├── warranty/
│       ├── kyc/
│       ├── transfer/
│       ├── disposable/
│       ├── notification/
│       └── admin/
├── lib/
│   ├── services/           ← 비즈니스 로직 (기존 SVC1~SVC17 → 모듈)
│   │   ├── zero-fp.ts
│   │   ├── hotline.ts
│   │   ├── fraud-report.ts
│   │   ├── safe-name.ts
│   │   ├── warranty.ts
│   │   ├── fraud-agent.ts
│   │   ├── notification.ts
│   │   ├── dispute.ts
│   │   ├── nrm.ts
│   │   ├── kyc.ts
│   │   ├── compatibility.ts
│   │   ├── auto-select.ts
│   │   ├── disposable-address.ts
│   │   ├── anchor.ts
│   │   ├── admin.ts
│   │   └── simulation.ts
│   ├── adapters/            ← 외부 시스템 어댑터 + Mock
│   │   ├── naming/          ← NRM 어댑터 (ENS, Unstoppable 등)
│   │   ├── notification/    ← 알림 채널 어댑터 (Slack, Email 등)
│   │   ├── fraud-source/    ← 외부 사기 정보 소스 어댑터
│   │   └── kyc/             ← KYC 제공자 어댑터
│   ├── ai/                  ← Vercel AI SDK + Gemini 통합
│   └── db/
│       └── prisma/          ← Prisma 스키마 + 클라이언트
├── prisma/
│   └── schema.prisma        ← 단일 DB 스키마 (SQLite/PostgreSQL 호환)
└── middleware.ts             ← Next.js Middleware (인증·Rate Limit)
```

**변경 사항 상세:**

| # | 현재 SRS 컴포넌트 | 변경 후 | 변경 근거 |
|---|---|---|---|
| 1 | API Gateway Layer (독립 레이어) | Next.js `middleware.ts` | C-TEC-001: 별도 서버 분리 금지. Middleware에서 JWT 검증, API Key 확인, Rate Limit 처리 |
| 2 | SVC1~SVC17 (16개 독립 Core Service) | `lib/services/*.ts` (16개 모듈) | C-TEC-002: 서비스 간 통신이 함수 import/호출로 대체. 네트워크 오버헤드 제거 |
| 3 | Primary DB + Fraud DB + Registry DB + Redis (4종 DB) | Prisma + 단일 Supabase PostgreSQL | C-TEC-003: 모든 테이블을 단일 DB에 통합. 인덱스 전략으로 조회 성능 확보 |
| 4 | Redis Caching Layer | Vercel KV(선택) 또는 인메모리 Map 캐시 | C-TEC-003: 별도 Redis 인프라 불필요. MVP 트래픽 수준에서 인메모리 캐시 충분 |
| 5 | Micro Frontend 5종 (독립 배포) | Next.js Route Group `(admin)/oc-*` | C-TEC-001: 단일 앱 내 라우트 분리. 역할 기반 접근 제어는 Middleware에서 처리 |
| 6 | Blockchain Layer (3개 스마트 컨트랙트) | 오프체인 DB 시뮬레이션 + 별도 Hardhat 프로젝트(선택) | C-TEC-007: Vercel에서 스마트 컨트랙트 배포 불가. Simulation Mode에서 DB 기반 시뮬레이션 |
| 7 | NRM Adapter Registry (플러그인 아키텍처) | DB 기반 어댑터 설정 + 코드 내 사전 등록 어댑터 풀 | C-TEC-001: 동적 코드 로딩 대신 DB Config + Strategy 패턴으로 구현 |
| 8 | Notification Channel Adapters (플러그인) | 동일 패턴: DB 설정 + 사전 등록 채널 핸들러 | C-TEC-001: 신규 채널 추가 시 코드 배포 필요하나 MVP 범위에서 충분 |
| 9 | Simulation Layer (Hardhat/Anvil, Stub, Testnet) | 환경 변수(`SIMULATION_MODE=true`) + Mock 서비스 모듈 | C-TEC-001: Mock을 서비스 모듈 내 분기로 구현. 별도 인프라 불필요 |
| 10 | External Integration Layer | `lib/adapters/*` (어댑터 모듈) | C-TEC-002: Route Handler 또는 Server Action에서 직접 외부 API 호출 |

#### 3.1.2 External Systems 변경 (§3.1)

**추가:**

| 시스템 | 유형 | 역할 | 제약 |
|---|---|---|---|
| Vercel Platform | 배포 인프라 | 애플리케이션 호스팅, Edge Network, Cron Jobs, KV Store | Serverless Function Timeout: Pro 60초, Cron 실행 시간 60초 |
| Supabase | 외부 DBaaS | PostgreSQL 호스팅, 인증(선택), 실시간 구독(선택) | Free Tier: 500MB DB, Pro: 8GB. 연결 풀 제한 있음 |
| Google Gemini API | 외부 REST | LLM 추론 (Vercel AI SDK 경유) | Rate Limit·과금 정책 확인 필요, 환경 변수로 모델 교체 가능 |

**삭제/변경:**

| 현재 시스템 | 변경 | 사유 |
|---|---|---|
| Mock RPC (Hardhat/Anvil) | 삭제 → 코드 내 Mock 함수로 대체 | 별도 Mock 서버 불필요 |
| PagerDuty | 유지 (Webhook 호출) | Route Handler에서 Webhook POST 가능 |
| Datadog APM | 삭제 → Vercel Analytics + 자체 로그로 대체 | Vercel 배포 환경에서 Datadog Agent 설치 불가 |
| Grafana / CloudWatch | 삭제 → Supabase Dashboard + 자체 모니터링 페이지로 대체 | 인프라 단순화 |
| AWS RDS | 삭제 → Supabase PostgreSQL로 대체 | C-TEC-003 |

#### 3.1.3 Interaction Sequences 변경 (§3.4)

모든 Sequence Diagram에서 다음 패턴으로 변경:

**변경 패턴:**

- `participant API as Zero-FP API Engine` → `participant API as Route Handler (/api/v1/simulate)`
- `participant SimCtrl as Simulation Controller` → `participant SimCtrl as SimulationService (lib/services/simulation.ts)`
- `participant Cache as Caching Layer` → `participant Cache as In-Memory Cache (lib/cache.ts)`
- `participant MockRPC as Mock RPC (Hardhat/Anvil)` → `participant MockData as Mock Module (lib/adapters/mock-rpc.ts)`
- 서비스 간 REST 호출 화살표 → 함수 직접 호출 화살표 (네트워크 지연 제거)
- `API->>SimCtrl: 현재 모드 확인` → `API->>SimCtrl: getMode()` (동기 함수 호출)

**§3.4.4 Safe-Name 송금 플로우 변경 핵심:**

현재 플로우의 `participant` 12개(Sender, App, OffChainDB, AutoSelect, KYCSvc, CompatSvc, FraudDB, DispAddr, Chain, Relayer, NotifyGW, Recipient)가 내부적으로는 단일 프로세스 내 함수 호출 체인으로 변경됨. 그러나 **사용자 관점의 인터랙션 흐름(입력→검증→확인→실행→통지)은 동일하게 유지**.

---

### 3.2 Constraints 변경 (§1 Constraints)

#### 3.2.1 신규 추가

| ID | 제약/가정 | 유형 | 검증 방안 | 시한 |
|---|---|---|---|---|
| **C-TEC-001** | 모든 서비스는 Next.js (App Router) 기반 단일 풀스택 프레임워크로 구현한다 | 정책 | 코드 리뷰 + 아키텍처 검증 | 전 기간 |
| **C-TEC-002** | 서버 측 로직은 Next.js Server Actions 또는 Route Handlers로 구현한다 | 정책 | 코드 리뷰 | 전 기간 |
| **C-TEC-003** | DB는 Prisma + 로컬 SQLite(개발) / Supabase PostgreSQL(배포)를 사용한다 | 정책 | Prisma 스키마 호환성 테스트 | MVP 착수 전 |
| **C-TEC-004** | UI는 Tailwind CSS + shadcn/ui를 사용한다 | 정책 | 코드 리뷰 + 디자인 시스템 검증 | 전 기간 |
| **C-TEC-005** | LLM 오케스트레이션은 Vercel AI SDK로 Next.js 내부에서 구현한다 | 정책 | 통합 테스트 | MVP 개발 중 |
| **C-TEC-006** | LLM은 Google Gemini API 기본 사용. 환경 변수로 모델 교체 가능 | 정책 | 모델 교체 E2E 테스트 | MVP 개발 중 |
| **C-TEC-007** | 배포는 Vercel 플랫폼. Git Push 자동 배포 | 정책 | 배포 파이프라인 검증 | MVP 착수 전 |
| **C-TEC-008** | (신규) Vercel Serverless Function Timeout은 Pro 플랜 기준 최대 60초이다. 이를 초과하는 배치 작업은 분할 실행 또는 Supabase Edge Functions로 오프로드한다 | 제약 | Timeout 초과 작업 식별 + 분할 설계 | MVP 개발 중 |
| **C-TEC-009** | (신규) Vercel Cron Jobs 실행 시간은 최대 60초이다. 대용량 배치(사기 DB 수집, Merkle Root 앵커링, GC 등)는 60초 이내 완료 가능한 단위로 분할한다 | 제약 | Cron Job 실행 시간 측정 | MVP 개발 중 |
| **C-TEC-010** | (신규) 로컬 SQLite → Supabase PostgreSQL 전환은 Prisma 마이그레이션으로 수행하며, Simulation→Production 전환 체크리스트에 DB 마이그레이션을 포함한다 | 정책 | 마이그레이션 자동화 테스트 | MVP 전환 전 |

#### 3.2.2 기존 Constraints 수정

| ID | 현재 | 변경 | 사유 |
|---|---|---|---|
| CON-1 | 자체 Caching 아키텍처(Redis 전제) | Caching은 인메모리 Map 캐시 또는 Vercel KV로 구현하며, 트래픽 90% 캐시 적중 목표는 유지 | Redis 인프라 제거 |
| CON-8 | 외부 인프라(RPC) 비용 위험 | Simulation Mode에서 RPC 비용 = $0. Production 전환 시 RPC 비용을 Vercel 인프라 비용과 통합 관리 | 인프라 단일화 |
| CON-15 | 오프체인 레지스트리 운영 비용 월 $1,500 이내 | Supabase Free/Pro 플랜 기준으로 재산정 (Pro: 월 $25). 전체 인프라 비용 목표를 월 $500 이하로 변경 | 비용 구조 변경 |
| CON-16 | Mock/Stub으로 대체. 테스트넷 사용 | Mock을 Next.js 코드 내 모듈로 구현. 별도 Mock 서버(Hardhat/Anvil) 불필요. 테스트넷 연동은 ethers.js로 Route Handler에서 직접 호출 | 인프라 단순화 |
| CON-23 | HD Wallet Seed를 AWS KMS / HashiCorp Vault에 저장 | MVP Simulation Mode에서는 환경 변수 기반 Mock Seed 사용. Production 전환 시 Vault/KMS 연동을 별도 추진 | Vercel 환경 제약 |

---

### 3.3 Functional Requirements 변경 (§4.1)

#### 3.3.1 구현 방식 변경 (기능 동일, 구현체 변경)

아래 요구사항은 **기능 자체는 동일하게 유지**하되, 구현 기술 참조를 변경한다.

| REQ ID | 변경 내용 | 영향 범위 |
|---|---|---|
| REQ-FUNC-001 | "백엔드 API 엔진이 포크 환경 시뮬레이션" → "Route Handler(`/api/v1/simulate`)가 시뮬레이션 로직 실행". Simulation Mode 시 Mock 모듈에서 응답 반환 | 구현 기술만 변경. 입출력 동일 |
| REQ-FUNC-004 | "자체 Caching Layer" → "인메모리 캐시 또는 Vercel KV". Mock RPC → Mock 모듈 | 구현 기술만 변경 |
| REQ-FUNC-006 | "Slack 실시간 알림" → "설정된 선호 채널로 알림 (Route Handler에서 Webhook/API 호출)" | 구현 기술만 변경 |
| REQ-FUNC-014 | "오프체인 레지스트리 DB에 즉시 기록" → "Prisma를 통해 Supabase PostgreSQL에 기록" | DB 기술만 변경 |
| REQ-FUNC-024 | "Agent가 외부 소스에서 데이터 수집 → Staging DB 적재" → "Vercel Cron Job이 외부 API 호출 → Prisma로 staging 테이블 적재" | 실행 환경 변경 |
| REQ-FUNC-035 | "NRM Adapter Registry에 등록된 어댑터로 라우팅" → "DB의 naming_adapter 테이블 조회 + 코드 내 어댑터 Strategy 패턴으로 라우팅" | 플러그인 → Strategy 패턴 |
| REQ-FUNC-041 | "코드 배포 없이 설정 기반 무중단 등록" → "DB 설정 변경 + 사전 등록된 어댑터 풀에서 활성화. 완전히 새로운 어댑터 유형 추가 시 코드 배포 필요" | 유연성 일부 제한 (MVP 허용) |
| REQ-FUNC-045 | "L2 체인에 Merkle Root 앵커링" → "Vercel Cron Job에서 ethers.js로 L2 TX 제출. Cron 60초 제한 내 완료 가능 (단일 TX)" | 실행 환경 변경 |
| REQ-FUNC-046 | "Channel Adapter 패턴 플러그인 방식" → "DB 설정 + 코드 내 사전 등록 채널 핸들러 (Slack, Email, KakaoTalk, SMS). 신규 채널 유형 추가 시 코드 배포 필요" | 유연성 일부 제한 (MVP 허용) |
| REQ-FUNC-048 | "5개 마이크로 프론트엔드 모듈, 독립 배포 가능" → "단일 Next.js 앱 내 라우트 그룹(`/admin/oc-*`). 독립 배포 불가, 동일 앱 내 라우트 전환" | 배포 독립성 삭제. 기능 동일 |
| REQ-FUNC-049 | "Admin Console에서 모드 전환" → "시스템 운영 콘솔(`/admin/oc-5`)에서 모드 전환. 환경 변수 + DB Config 조합" | 구현 기술만 변경 |
| REQ-FUNC-055 | "외부 KYC 제공자와 연동" → "Simulation Mode에서 Mock KYC 모듈. Production 전환 시 외부 KYC API 연동 (Route Handler에서 호출)" | 구현 기술만 변경 |

#### 3.3.2 기능 축소 / 시뮬레이션 전환

아래 요구사항은 **MVP에서 기능을 오프체인 시뮬레이션으로 대체**한다.

| REQ ID | 현재 기능 | MVP 변경 | 가치 영향 | Production 전환 시 |
|---|---|---|---|---|
| REQ-FUNC-018 | Warranty 팝업 → 스마트 컨트랙트 연동 | 팝업 UI는 동일 유지. 내부적으로 DB 레코드 생성 (스마트 컨트랙트 Mock) | 사용자 UX 무변경. 온체인 검증 불가 (시뮬레이션임을 명시) | 별도 Hardhat 프로젝트에서 스마트 컨트랙트 배포 후 주소 연동 |
| REQ-FUNC-019 | NFT 민팅 → 스마트 컨트랙트 실행 | DB에 WARRANTY_POLICY 레코드 생성 + 시뮬레이션 NFT 메타데이터 저장. 실제 온체인 민팅 없음 | 사용자에게 "시뮬레이션 환경" 안내 표시. 보증서 정보 확인 가능 | 테스트넷/메인넷 NFT 민팅 연동 |
| REQ-FUNC-020 | 보상금 자동 릴리즈 → 스마트 컨트랙트 실행 | 클레임 접수 → DB 상태 변경 (claimed) + 보상 금액 기록. 실제 온체인 지급 없음 | 클레임 워크플로우 검증 가능. 실제 지급은 Production에서 | 스마트 컨트랙트 자동 릴리즈 연동 |
| REQ-FUNC-065 | 일회용 주소 → HD Wallet BIP-44 파생 | 랜덤 UUID 기반 Mock 주소 생성 + DB 기록. HD Wallet 파생 없음 | 송금자 UX 무변경 (일회용 주소는 내부 처리). 주소 파생 보안성 미검증 | HD Wallet + KMS 연동 별도 서비스 추가 |
| REQ-FUNC-066 | 자금 포워딩 → Relayer 상시 리스닝 | DB 상태 전환 (`active` → `used`) + Cron Job으로 주기적 상태 갱신. 실제 온체인 포워딩 없음 | 포워딩 워크플로우 검증 가능. 실제 자금 이동 없음 | 별도 Relayer 서비스 (상시 프로세스) 추가 필요 |
| REQ-FUNC-028~030 | On-Premise ZK 모듈 (폐쇄망, 1000 TPS) | MVP 범위 유지 (Should). 구현 없이 "Production 별도 추진"으로 명시 | F7 전체가 Should이므로 MVP 가치 영향 없음 | 별도 인프라 + ZK 라이브러리 통합 |

#### 3.3.3 LLM 통합 요구사항 신규 정의

현재 SRS v0.35에는 LLM/AI 관련 기능 요구사항이 전혀 없다. Vercel AI SDK + Gemini API(C-TEC-005, C-TEC-006)의 활용 범위를 정의해야 한다.

**LLM 적용 후보 기능 (SRS v4.1에서 정의 필요):**

| 후보 ID | 적용 영역 | LLM 역할 | 우선순위 | 구현 방식 |
|---|---|---|---|---|
| LLM-001 | F3. 사기 주소 신고 접수 | 신고 내용 자동 분류 (사기 유형 판별), 스팸 신고 필터링 강화 | Could | Server Action에서 Gemini 호출 → 신고 description 분석 → 자동 분류 태그 |
| LLM-002 | F6. Fraud Intelligence Agent | 수집된 사기 정보 요약·위험도 자동 평가, 소스 간 교차 검증 자동화 | Could | Cron Job에서 Gemini 호출 → Staging 데이터 분석 → 위험 등급 추천 |
| LLM-003 | OC-1 사기 정보 관리 콘솔 | 운영 담당자용 AI 어시스턴트 (사기 패턴 분석, 승인 판단 보조) | Could | 관리 콘솔 페이지에 AI 채팅 위젯 → Vercel AI SDK 스트리밍 |
| LLM-004 | F3-A. 이의 신청 심사 | 이의 신청 증빙 자동 검토, 심사 의견 초안 생성 | Won't (MVP) | 향후 추진. 법적 리스크로 MVP 배제 |

> **⚠️ 참고:** LLM 적용 범위는 프로젝트 오너의 의사결정이 필요한 항목이다. 위 후보를 기반으로 MVP에 포함할 LLM 기능을 확정한 후 SRS v4.1에 Functional Requirements로 추가해야 한다.

---

### 3.4 Non-Functional Requirements 변경 (§4.2)

#### 3.4.1 성능 (Performance) 변경

| REQ ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-001 | Zero-FP API p95 ≤ 100ms | **Simulation Mode: p95 ≤ 500ms, Production: p95 ≤ 300ms (목표)** | Vercel Serverless Cold Start(수백 ms) 존재. 100ms는 Edge Runtime에서도 DB 조회 포함 시 비현실적 |
| REQ-NF-002 | 사기 주소 조회 p95 ≤ 2,000ms | p95 ≤ 2,000ms (유지) | DB 인덱스 + 인메모리 캐시로 달성 가능 |
| REQ-NF-003 | Safe-Name 리졸브 p95 ≤ 500ms | p95 ≤ 500ms (유지) | Prisma 쿼리 + 캐시로 달성 가능 |
| REQ-NF-005 | 알림 발송 p95 ≤ 2,000ms | p95 ≤ 5,000ms | 외부 알림 API 호출 지연 + Serverless 오버헤드 |
| REQ-NF-007 | 10,000 TPS (Zero-FP API) | **MVP: 100 TPS. Production: 1,000 TPS (Vercel Pro 기준)** | Vercel Serverless 동시 실행 한도. MVP Simulation Mode에서 10,000 TPS 불필요 |
| REQ-NF-008 | 동시 접속 500 유저 (피크 1,000) | MVP: 동시 접속 100 유저 (피크 200) | MVP 사용자 규모에 맞춤 |
| REQ-NF-009 | 부하 테스트 출시 전 1회 + 분기 1회, k6 + Grafana | **Vercel Analytics 기반 모니터링 + 자체 부하 테스트 스크립트** | Grafana 인프라 삭제 |
| REQ-NF-037 | Safe-Name 등록 p95 ≤ 3,000ms | p95 ≤ 3,000ms (유지) | Prisma 쿼리로 충분 |
| REQ-NF-039 | NRM 리졸브 p95 ≤ 1,000ms | Simulation Mode: p95 ≤ 1,000ms (Mock). Production: p95 ≤ 2,000ms (외부 API 의존) | 외부 네이밍 서비스 API 지연 반영 |
| REQ-NF-048 | Auto-Select p95 ≤ 200ms | p95 ≤ 500ms | Serverless 오버헤드 반영 |
| REQ-NF-053 | 일회용 주소 생성 p95 ≤ 200ms | Simulation Mode: p95 ≤ 300ms (Mock 주소). Production: 별도 정의 | Mock 주소 생성은 빠르나 Serverless 오버헤드 |

#### 3.4.2 신뢰성 (Reliability) 변경

| REQ ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-010 | 월간 API 가용성 ≥ 99.99% | **MVP: ≥ 99.9%. Production: ≥ 99.95% (Vercel Pro SLA 기준)** | Vercel Pro SLA는 99.99%를 공식 보장하지 않음 |
| REQ-NF-017 | 데이터 백업 일 1회 (RPO ≤ 24h), AWS RDS 스냅샷 + S3 | **Supabase 자동 백업 (Pro: 일 1회, RPO ≤ 24h) + Point-in-Time Recovery** | 인프라 변경 |

#### 3.4.3 비용 (Cost) 변경

| REQ ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-023 | 외부 RPC 비용 월 ≤ $5,000 (Simulation: $0) | **Simulation Mode: $0. Production: 월 ≤ $500 (L2 우선 + 캐시 전략)** | 인프라 비용 구조 변경 |
| REQ-NF-024 | 전체 MVP 월 인프라 비용 ≤ $15,000 (Simulation: ≤ $5,000) | **MVP: ≤ $100 (Vercel Pro $20 + Supabase Pro $25 + 도메인 등). Production: ≤ $500** | Vercel + Supabase 기반 비용 구조로 대폭 절감 |

#### 3.4.4 유지보수성 (Maintainability) 변경

| REQ ID | 현재 기준 | 변경 기준 | 변경 사유 |
|---|---|---|---|
| REQ-NF-029 | Datadog / CloudWatch 로그 수집 | **Vercel Logs + Supabase Logs + 자체 AUDIT_LOG 테이블 (Prisma)** | 인프라 변경 |
| REQ-NF-030 | Grafana / Mixpanel 운영 대시보드 | **OC-5 시스템 운영 콘솔 내 자체 대시보드 페이지 (Next.js + Recharts/shadcn Charts)** | 외부 도구 의존 제거 |

#### 3.4.5 삭제 대상 NFR

| REQ ID | 현재 내용 | 삭제 사유 |
|---|---|---|
| REQ-NF-054 | 자금 포워딩 완료 시간 p95 ≤ 5분(L2) | MVP에서 실제 포워딩 없음 (DB 시뮬레이션). "MVP 시뮬레이션 시 해당 없음, Production 전환 시 재정의"로 변경 |
| REQ-NF-055 | 자금 포워딩 성공률 ≥ 99.9% | 동일 사유 |
| REQ-NF-056 | 포워딩 가스비 ≤ $0.01 | 동일 사유 |
| REQ-NF-058 | HD Wallet Seed KMS 감사 로그 100% | MVP에서 KMS 미사용. Production 전환 시 재정의 |
| REQ-NF-059 | 파생 개인키 Zero-Copy Wipe | MVP에서 HD Wallet 미사용. Production 전환 시 재정의 |

> **참고:** 위 항목은 "삭제"가 아니라 **"MVP 해당 없음(N/A) — Production 전환 시 재정의"** 으로 표기하여 추적성을 유지한다.

---

### 3.5 Entity & Data Model 변경 (§6.2)

#### 3.5.1 통합 DB 스키마 (Prisma)

현재 4종 분리 DB(Primary PostgreSQL, Fraud Address DB, Off-Chain Registry DB, Redis)를 **단일 Prisma 스키마**로 통합한다.

**변경 원칙:**

1. 모든 엔터티를 단일 `schema.prisma`에 정의
2. SQLite 호환성 유지 (로컬 개발용): JSON 필드 → String으로 저장, `@db.Text` 대신 `String` 사용
3. 인덱스 전략으로 조회 성능 확보 (FRAUD_ADDRESS.address, SAFE_NAME.human_name 등)
4. Redis 캐시 역할은 서비스 레이어의 인메모리 Map으로 대체

**주요 변경:**

| 엔터티 | 변경 사항 |
|---|---|
| 전체 | `string` PK → Prisma `@id @default(cuid())` 또는 `@default(uuid())` |
| 전체 | `json` 타입 필드 → SQLite 호환을 위해 `String`으로 저장, 서비스 레이어에서 `JSON.parse/stringify` |
| FRAUD_ADDRESS | 별도 DB → 동일 DB 내 테이블. `@@index([address])`, `@@index([chain, address])` 추가 |
| SAFE_NAME | Off-Chain Registry DB → 동일 DB 내 테이블. `@@unique([human_name])` 유지 |
| SIMULATION_CONFIG | 단일 레코드 테이블로 유지. 환경 변수 `SIMULATION_MODE`와 연동 |

#### 3.5.2 Prisma 스키마 호환 고려사항

| 항목 | SQLite (로컬) | PostgreSQL (Supabase) | 대응 |
|---|---|---|---|
| JSON 필드 | 미지원 (String으로 저장) | 네이티브 JSON 지원 | 서비스 레이어에서 직렬화/역직렬화 |
| Enum | 미지원 | 네이티브 Enum 지원 | String 필드 + 애플리케이션 레벨 검증 |
| DateTime | TEXT 저장 | TIMESTAMPTZ | Prisma가 자동 변환 |
| Full-Text Search | 미지원 | 지원 | MVP에서 LIKE 검색 사용, 필요 시 Supabase FTS |

---

### 3.6 API Endpoint 변경 (§6.1)

#### 3.6.1 구현 방식 변경

| 현재 | 변경 후 |
|---|---|
| 독립 마이크로서비스의 REST 엔드포인트 | Next.js Route Handler (`/app/api/v1/*/route.ts`) |
| API Gateway 레이어에서 인증·Rate Limit | Next.js `middleware.ts`에서 JWT/API Key 검증 + Rate Limit |
| Internal Service Token 인증 (A22, A23, A25) | 동일 프로세스 내 함수 호출이므로 **내부 인증 불요**. API로 외부 노출하지 않음 |

#### 3.6.2 내부 전용 엔드포인트 처리

| 엔드포인트 | 현재 인증 | 변경 |
|---|---|---|
| A22 `/api/v1/transfer/notify` | Internal Service Token | **API 노출 삭제** → `lib/services/notification.ts`의 `sendTransferNotification()` 함수 직접 호출 |
| A23 `/api/v1/disposable/generate` | Internal Service Token | **API 노출 삭제** → `lib/services/disposable-address.ts`의 `generateDisposableAddress()` 함수 직접 호출 |

---

### 3.7 Simulation Mode 전환 체크리스트 변경 (§6.3.7)

현재 체크리스트에 다음 항목을 추가/변경:

| # | 현재 체크 항목 | 변경/추가 |
|---|---|---|
| ① | 외부 API 키 설정 확인 | 유지 + **Gemini API Key 설정 확인** 추가 |
| ② | 메인넷 RPC 연결 테스트 | 유지 |
| ③ | 보증풀 잔고 확인 | 유지 (Production 시 스마트 컨트랙트 연동 필요) |
| ④ | (신규) **Supabase PostgreSQL 연결 확인** | 로컬 SQLite → Supabase 전환 검증 |
| ⑤ | 알림 채널 연동 테스트 | 유지 |
| ⑥ | 온체인 컨트랙트 배포 확인 | 유지 |
| ⑦ | (신규) **Prisma 마이그레이션 완료 확인** | 스키마 동기화 검증 |
| ⑧ | (신규) **Vercel 환경 변수 설정 확인** | `SIMULATION_MODE=false`, DB URL, API Keys 등 |
| ⑨ | (신규) **Vercel 도메인·SSL 설정 확인** | 프로덕션 도메인 바인딩 검증 |

---

## 4. MVP 핵심 사용자 경험 (가치 전달) 영향 분석

### 4.1 분석 프레임워크

SRS §1.1의 7대 Pain Point(CORE-1~3, CJM-1~2, EXT-1~2)와 §1.2의 In-Scope 12개 범위를 기준으로, 기술 스택 변경이 **최종 사용자가 체감하는 가치**를 훼손하는지 검증한다.

**검증 기준:** "화면에 보이는 것과 사용자 인터랙션이 동일한가?" — 내부 구현 기술의 변경은 허용하되, 사용자에게 노출되는 UI·UX·응답·결과가 변하면 가치 훼손으로 판정.

### 4.2 Pain Point별 가치 전달 영향 평가

#### CORE-1: 과도한 오탐지로 VASP VIP 정상 출금 차단 및 CS 마비

| 가치 요소 | 현재 SRS 전달 방식 | 스택 변경 후 | 영향 |
|---|---|---|---|
| Zero-FP API 검증 → 오탐지율 ≤ 0.01% | 마이크로서비스 API 엔진 | Route Handler + Mock (Simulation) | **✅ 무영향.** 입출력 동일. 오탐지 알고리즘은 스택 무관 |
| 핫라인 10분 SLA 락 해제 | 대시보드 + Slack 알림 | Next.js 대시보드 + 멀티채널 알림 | **✅ 무영향.** CISO가 보는 대시보드 UX 동일 |
| 8분 미처리 시 PagerDuty 에스컬레이션 | PagerDuty Webhook | Route Handler에서 PagerDuty Webhook 호출 | **✅ 무영향.** 외부 Webhook 호출 방식 동일 |

**CORE-1 판정: ✅ 가치 훼손 없음**

---

#### CORE-2: 사람이 인식 불가능한 온체인 주소 구조 — 오송금·피싱 취약

| 가치 요소 | 현재 SRS 전달 방식 | 스택 변경 후 | 영향 |
|---|---|---|---|
| Safe-Name "이름 + 금액" 간소화 이체 | 오프체인 DB 리졸브 + Auto-Select | Prisma DB 리졸브 + 동일 비즈니스 로직 | **✅ 무영향.** "lee.safe" 입력 → 주소 리졸브 → 체인 자동 결정 UX 동일 |
| KYC Tier 표시 + Verified 배지 | KYC Service + DB 조회 | 동일 로직을 `lib/services/kyc.ts`에서 실행 | **✅ 무영향.** 사용자에게 보이는 배지·등급 표시 동일 |
| 사기 주소 Hard Block (강제 차단) | FraudDB 조회 + 차단 화면 | Prisma 쿼리 + 동일 차단 화면 (Tailwind + shadcn) | **✅ 무영향.** 차단 화면·사유 표시 동일 |
| 일회용 주소 (송금자 UX 투명) | HD Wallet 파생 + Relayer 포워딩 | Mock 주소 생성 + DB 상태 전환 | **⚠️ 부분 영향.** 송금자 UX는 동일(일회용 주소 미노출). 그러나 **실제 온체인 보안 효과(주소 추적 방지)는 Simulation Mode에서 검증 불가**. Production 전환 시 HD Wallet + Relayer 추가 필요 |
| NRM 외부 네이밍 통합 리졸브 | Adapter Registry + 외부 API 호출 | DB 설정 + Strategy 패턴 + Mock (Simulation) | **✅ 무영향 (Simulation).** Mock 응답으로 UX 검증 가능. Production에서 실제 외부 API 연동 |

**CORE-2 판정: ✅ 가치 훼손 없음** (일회용 주소의 실제 보안 효과는 Production에서 검증)

---

#### CORE-3: 퍼블릭 SaaS 망 의존으로 TradFi 인가 탈락 위기

| 가치 요소 | 현재 SRS 전달 방식 | 스택 변경 후 | 영향 |
|---|---|---|---|
| On-Premise ZK 모듈 (100% 망분리) | 폐쇄망 전용 모듈 배포 | Should 우선순위 → "Production 별도 추진" 유지 | **✅ 무영향.** 현재도 Should이며 MVP에서 구현 예정 없음 |

**CORE-3 판정: ✅ 가치 훼손 없음** (MVP 범위 외)

---

#### CJM-1: 구제/보상 수단이 없는 100% 면책 조항

| 가치 요소 | 현재 SRS 전달 방식 | 스택 변경 후 | 영향 |
|---|---|---|---|
| Warranty 보증 팝업 (최대 $30K) | 스마트 컨트랙트 기반 보증풀 | 오프체인 DB 기반 시뮬레이션 | **⚠️ 부분 영향.** 팝업 UI·구독 UX는 동일. 그러나 **보증풀 잔고가 온체인에 투명하게 공개되는 신뢰성 요소**가 Simulation Mode에서는 제공되지 않음. 시뮬레이션 환경임을 사용자에게 명시해야 함 |
| NFT 보험 증서 발급 | 온체인 NFT 민팅 | DB 레코드 + 시뮬레이션 메타데이터 | **⚠️ 부분 영향.** 보증서 정보 확인은 가능하나 온체인 NFT의 불변성·증명력은 없음 |
| 보상금 자동 릴리즈 (24시간 SLA) | 스마트 컨트랙트 자동 실행 | DB 상태 변경 (수동 프로세스) | **⚠️ 부분 영향.** 클레임 워크플로우는 동일하나 자동 지급의 기술적 보증(스마트 컨트랙트 무신뢰 실행)은 없음 |

**CJM-1 판정: ⚠️ 경미한 영향** — 사용자 UX 흐름은 동일하나, 온체인 신뢰성 요소(투명 잔고, NFT 증명, 자동 지급)가 Simulation Mode에서 부재. **Simulation 환경 안내 문구 필수**.

**완화 조치:**
1. Warranty 관련 모든 화면에 "현재 시뮬레이션 환경입니다. 실제 온체인 보증은 정식 출시 후 제공됩니다" 배너 표시
2. 보증서 정보를 DB에 기록하되, 향후 온체인 전환 시 동일 데이터를 NFT 메타데이터로 마이그레이션하는 경로를 설계에 반영
3. REQ-NF-025(보증풀 잔고 투명성)를 "MVP: 자체 대시보드 공개, Production: 온체인 + Dune Analytics"로 2단계 전환

---

#### CJM-2: 사기 주소 신고 접점 부재

| 가치 요소 | 현재 SRS 전달 방식 | 스택 변경 후 | 영향 |
|---|---|---|---|
| 사기 주소 신고 플랫폼 (Web) | 웹 프론트엔드 + API | Next.js 페이지 + Route Handler | **✅ 무영향.** 신고 양식·접수 확인·알림 UX 완전 동일 |
| 사기 여부 사전 조회 | Fraud DB 조회 + 결과 표시 | Prisma 쿼리 + 동일 결과 표시 | **✅ 무영향** |
| 이의 신청·심사 프로세스 | 워크플로우 + 컴플라이언스 대시보드 | 동일 워크플로우 + OC-4 라우트 그룹 | **✅ 무영향** |
| 신고자 리워드 (포인트/배지) | 알림 + DB 기록 | 동일 | **✅ 무영향** |

**CJM-2 판정: ✅ 가치 훼손 없음**

---

#### EXT-1: 지속되는 해킹 공포 / 무보증 트라우마

| 가치 요소 | 스택 변경 후 | 영향 |
|---|---|---|
| Warranty 보증 구독 가능 | 오프체인 시뮬레이션 (CJM-1과 동일) | **⚠️ 경미한 영향** (CJM-1 판정과 동일) |
| 사기 주소 사전 조회로 안심 | ✅ 무영향 | **✅ 무영향** |
| Safe-Name 이름 기반 송금으로 오송금 방지 | ✅ 무영향 | **✅ 무영향** |

**EXT-1 판정: ⚠️ 경미한 영향** (Warranty 온체인 신뢰 요소만 해당)

---

#### EXT-2: 기관의 외부 사기정보 수집 역량 부재

| 가치 요소 | 스택 변경 후 | 영향 |
|---|---|---|
| 멀티소스 사기 정보 통합 Agent | Vercel Cron Job + Prisma Staging 테이블 | **✅ 무영향.** 수집·정규화·승인 워크플로우 동일 |
| Fraud Agent 대시보드 | Next.js 페이지 + shadcn/ui | **✅ 무영향.** CISO가 보는 필터·이력·추이 UX 동일 |
| 승인 대기열 (OC-1 콘솔) | 라우트 그룹 기반 관리 페이지 | **✅ 무영향.** 건별·일괄 승인/거부 워크플로우 동일 |

**EXT-2 판정: ✅ 가치 훼손 없음**

---

### 4.3 가치 영향 종합 매트릭스

| Pain Point | 가치 훼손 여부 | 영향 수준 | 완화 가능 여부 |
|---|---|---|---|
| CORE-1 (오탐지 CS 마비) | ✅ 없음 | — | — |
| CORE-2 (주소 인식 불가) | ✅ 없음 | — | — |
| CORE-3 (TradFi 망분리) | ✅ 없음 (MVP 범위 외) | — | — |
| CJM-1 (구제/보상 부재) | ⚠️ 경미 | 온체인 신뢰 요소 부재 (Simulation) | ✅ 시뮬레이션 안내 + 2단계 전환 설계 |
| CJM-2 (신고 접점 부재) | ✅ 없음 | — | — |
| EXT-1 (해킹 공포) | ⚠️ 경미 | CJM-1과 동일 | ✅ 동일 완화 조치 |
| EXT-2 (사기정보 수집 역량) | ✅ 없음 | — | — |

### 4.4 가치 영향 결론

> **기술 스택 변경으로 인한 MVP 핵심 사용자 경험 훼손은 7대 Pain Point 중 0건(완전 훼손)이며, 2건(CJM-1, EXT-1)에서 경미한 영향이 발생한다.**
>
> 경미한 영향은 모두 **Warranty 관련 온체인 신뢰 요소**(보증풀 온체인 투명성, NFT 증명력, 스마트 컨트랙트 자동 지급)에 한정되며, 이는 현재 SRS v0.35에서도 이미 Simulation Mode(CON-16)에서 테스트넷 시뮬레이션으로 운영하도록 설계된 영역이다.
>
> 따라서 **기술 스택 변경은 MVP Simulation Mode의 본래 설계 의도와 정합하며, 핵심 사용자 가치 전달을 훼손하지 않는다.**
>
> 단, **Production 전환 시 아래 3개 영역은 반드시 별도 인프라·서비스 추가가 필요**하며, SRS v4.1에 해당 전환 경로를 명시적으로 기록해야 한다:
> 1. **Warranty 스마트 컨트랙트** — Hardhat 프로젝트 + 온체인 배포
> 2. **HD Wallet + Forwarding Relayer** — 상시 프로세스 (Vercel 외부)
> 3. **KMS (키 관리 시스템)** — AWS KMS 또는 HashiCorp Vault

---

## 5. In-Scope (IS) 항목별 커버리지 최종 확인

| IS # | 범위 | 스택 변경 후 커버리지 | 비고 |
|---|---|---|---|
| IS-1 | Zero-FP 검증 API 엔진 | ✅ 100% | Route Handler + Mock으로 완전 구현 |
| IS-2 | VASP 핫라인 대시보드 | ✅ 100% | Next.js 페이지 + SSE/폴링으로 실시간 갱신 |
| IS-3 | 사기 주소 신고·조회 플랫폼 | ✅ 100% | Tailwind + shadcn/ui로 구현 |
| IS-4 | Safe-Name 오프체인 레지스트리 + DNS 비용 모델 | ✅ 100% | Prisma DB로 완전 구현 |
| IS-5 | Warranty 보증 프론트엔드 위젯 | ⚠️ 90% | UI 100%. 스마트 컨트랙트 → DB 시뮬레이션 |
| IS-6 | 상위 3~5개 메이저 체인 지원 | ✅ 100% (Simulation) | Mock 체인 데이터. Production에서 실제 RPC 연동 |
| IS-7 | 이의 신청·심사·해제 프로세스 | ✅ 100% | 비즈니스 로직 + DB 워크플로우 |
| IS-8 | NRM 이종 네이밍 통합 리졸브 | ⚠️ 95% | Mock 리졸브. 플러그인 동적 로딩 → Strategy 패턴으로 변경 |
| IS-10 | 통합 알림 게이트웨이 | ⚠️ 95% | 사전 등록 채널 핸들러. 완전 동적 플러그인 → Config 기반 활성화 |
| IS-11 | 운영기관 관리 콘솔 5종 | ⚠️ 95% | 라우트 그룹. 독립 배포 불가 → 동일 앱 내 분리 |
| IS-12 | MVP Simulation Mode | ✅ 100% | 환경 변수 + DB Config + Mock 모듈 |

**총 커버리지: 12개 In-Scope 항목 중 100% 커버 7건, 90~95% 커버 5건. 기능적 커버리지 부족(< 80%)으로 인한 MVP 목표 미달 항목: 0건.**

---

## 6. 작업 순서 (Migration Work Breakdown)

### Phase 1: SRS 문서 변경 (본 Plan 기반)

| # | 작업 | 변경 대상 섹션 | 산출물 |
|---|---|---|---|
| 1-1 | Constraints C-TEC-001~010 추가 | §1 Constraints | SRS v4.1 §1 |
| 1-2 | External Systems 추가/삭제/변경 | §3.1 | SRS v4.1 §3.1 |
| 1-3 | Component Diagram 전면 재작성 | §3.2.5 | SRS v4.1 §3.2.5 |
| 1-4 | Client Applications 구현 기술 변경 | §3.2.1 ~ §3.2.3 | SRS v4.1 §3.2 |
| 1-5 | Interaction Sequences 업데이트 | §3.4.1 ~ §3.4.4 | SRS v4.1 §3.4 |
| 1-6 | API Overview 구현 방식 변경 | §3.3 | SRS v4.1 §3.3 |
| 1-7 | Functional Requirements 수정 (15건) | §4.1 F1~F10 | SRS v4.1 §4.1 |
| 1-8 | Functional Requirements 축소 (5건) | §4.1 F4-F, F5 | SRS v4.1 §4.1 |
| 1-9 | LLM 통합 요구사항 정의 (오너 확인 후) | §4.1 신규 섹션 | SRS v4.1 §4.1 F11 |
| 1-10 | Non-Functional Requirements 변경 (20건) | §4.2 | SRS v4.1 §4.2 |
| 1-11 | Entity/Data Model Prisma 통합 | §6.2 | SRS v4.1 §6.2 |
| 1-12 | API Endpoint List 변경 | §6.1 | SRS v4.1 §6.1 |
| 1-13 | Simulation Mode 전환 체크리스트 변경 | §6.3.7 | SRS v4.1 §6.3.7 |
| 1-14 | Traceability Matrix 갱신 | §5 | SRS v4.1 §5 |
| 1-15 | Validation Plan 갱신 | §6.4 | SRS v4.1 §6.4 |

### Phase 2: 아키텍처 설계 (SRS v4.1 기반)

| # | 작업 | 산출물 |
|---|---|---|
| 2-1 | Next.js 프로젝트 구조 설계 (폴더·라우트·모듈 구조) | Architecture Design Document |
| 2-2 | Prisma 스키마 설계 (단일 통합) | `schema.prisma` |
| 2-3 | 미들웨어 인증·Rate Limit 설계 | Middleware Spec |
| 2-4 | Simulation Mode Mock 모듈 설계 | Mock Module Spec |
| 2-5 | LLM 통합 아키텍처 설계 (Vercel AI SDK + Gemini) | AI Integration Spec |

### Phase 3: 구현 (Phase 2 기반)

구현 순서는 별도 Sprint Plan에서 정의.

---

## 7. Risk & Mitigation (위험 및 완화)

| # | 위험 | 영향 | 발생 확률 | 완화 방안 |
|---|---|---|---|---|
| R-1 | Vercel Serverless Timeout(60초)으로 복잡한 API 응답 불가 | 일부 API 실패 | 중 | 복잡 로직 분할, 배경 처리(Vercel Background Functions — Beta) 검토 |
| R-2 | Supabase Free Tier DB 용량(500MB) 초과 | 서비스 중단 | 중 | 시드 데이터 최적화, 조기 Pro 전환($25/월) |
| R-3 | SQLite → PostgreSQL 마이그레이션 시 비호환 | 데이터 유실 | 저 | Prisma 스키마 양쪽 호환 설계 + 마이그레이션 테스트 자동화 |
| R-4 | Production 전환 시 별도 인프라(Relayer, KMS, 스마트 컨트랙트) 추가 비용·공수 과소 추정 | 일정 지연 | 고 | Production 전환 PoC를 MVP 개발 중 병행 수행. 전환 공수를 별도 추정 |
| R-5 | LLM 적용 범위 미확정으로 개발 착수 지연 | 일정 지연 | 중 | LLM 기능을 Could 우선순위로 설정, MVP Core 기능과 분리 개발 |
| R-6 | Vercel 요금 예상 초과 (높은 트래픽 시) | 비용 초과 | 저 | 모니터링 + 캐시 전략 + Rate Limit 강화 |

---

## 8. Decision Log (의사결정 기록)

| # | 결정 사항 | 근거 | 대안 | 결정일 |
|---|---|---|---|---|
| D-1 | 마이크로서비스 → Next.js 모놀리스 전환 | C-TEC-001. MVP 단계에서 인프라 복잡도 최소화 | 별도 백엔드 서버 유지 → 거부 (C-TEC-002 위반) | 2026-05-09 |
| D-2 | Redis 삭제 → 인메모리 캐시 / Vercel KV | C-TEC-003. MVP 트래픽에서 별도 Redis 불필요 | Redis 유지 → 거부 (인프라 복잡도) | 2026-05-09 |
| D-3 | Micro Frontend → 라우트 그룹 | C-TEC-001. 단일 앱 내 분리로 충분 | Module Federation → 거부 (과도 엔지니어링) | 2026-05-09 |
| D-4 | 스마트 컨트랙트 → DB 시뮬레이션 (MVP) | C-TEC-007. Vercel에서 컨트랙트 배포 불가 | Hardhat 별도 서버 → 보류 (Production 전환 시) | 2026-05-09 |
| D-5 | HD Wallet + Relayer → Mock 주소 + DB 상태 전환 (MVP) | C-TEC-001, C-TEC-007. 상시 프로세스 Serverless 비호환 | 별도 Relayer 서버 → 보류 (Production 전환 시) | 2026-05-09 |
| D-6 | 성능 NFR 완화 (100ms → 500ms, 10K TPS → 100 TPS) | Vercel Serverless 물리적 제약. MVP 트래픽 규모에 맞춤 | NFR 유지 + Edge Runtime → 보류 (추후 최적화) | 2026-05-09 |
| D-7 | Datadog/Grafana → Vercel Analytics + 자체 대시보드 | C-TEC-007. Vercel 환경에서 Datadog Agent 미지원 | Datadog Serverless Plugin → 비용·복잡도 대비 보류 | 2026-05-09 |

---

**— End of Plan Document —**
