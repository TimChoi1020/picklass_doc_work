---
title: Actor & 조직 모델
version: v1.0
updated: 2026-07-17
owner_service: common
master_origin: v0.21 §4
depends_on:
  - ./01_서비스개요_전체구조.md
---

> 📄 본 문서는 통합 기획서 v0.21(→ `_archive/`)에서 분리된 문서이다. 본문 내 **§번호는 구 통합 기획서 기준**이며, §번호 → 신규 문서 매핑은 [README](../README.md)의 매핑표를 참조한다.

## 4. Actor & 조직 모델

> **v0.6 전면 재편:** B2B 고객을 학원뿐 아니라 기업·학교까지 포괄하기 위해, 조직 계층을 **3-Tier 고정 구조**로 정의하고 Person Actor를 **8종**으로 재정의한다. 계정은 **단일 User + 복수 Membership** 모델로 통일한다.

### 4.1 Actor Taxonomy 개관

Picklass의 사용자 체계는 **4개 축**으로 정의된다.

| 축 | 분류 | 예시 |
|---|---|---|
| **A. Entity vs Person** | 조직(엔터티) / 사람(역할) | Institution = Entity / 강사 = Person |
| **B. 조직 계층 (Depth)** | Partner(L0) / Group(L1) / Institution(L2) — 3단 고정 | 파트너사 → 프랜차이즈 본부 → 가맹학원 |
| **C. 채널** | B2B (파트너/본부/기관) / B2C (개인) | 기관 소속 학생 / 개인 구독자 |
| **D. 역할(Role)** | 8종 Person Role | 시스템관리자 ~ Consumer |

### 4.2 조직 계층 (Organization Hierarchy)

#### 4.2.1 3-Tier 고정 구조

조직은 최대 3단 고정 계층으로 구성된다. 4단 이상 세분화가 필요한 경우 Institution 내부의 논리적 분류(팀·반·부서)로 처리하며, 조직 엔터티 자체는 확장하지 않는다.

```
[L0] Partner (파트너사)           ◇ 선택적
  │   여러 Group 또는 Institution 횡단 관리
  │
  ├─[L1] Group (본부 / 지주사)    ◇ 선택적
  │    산하 Institution 등록·플랜 배분·통계
  │
  └─[L2] Institution (학원/계열사/학교) ■ 필수
       실제 수업·결제·액세스코드 발급 단위
```

#### 4.2.2 필수/선택 레벨 규칙

| 고객 유형 | L0 Partner | L1 Group | L2 Institution |
|---|---|---|---|
| 단독 소규모 학원 | — | — | ■ |
| 프랜차이즈 학원 | 선택 | ■ | ■ |
| 단일 법인 기업 | — | — | ■ |
| 대기업 그룹 | 선택 | ■ | ■ |
| 파트너 중개 (단일 Institution) | ■ | — | ■ |
| 파트너 중개 (복수 Group) | ■ | ■ | ■ |

**핵심 원칙**
- **Institution(L2)은 항상 필수** — 실제 학습·결제·액세스코드 발급의 단위
- Partner는 **Group을 건너뛰고 Institution 직접 관리 가능** (작은 단일 계약을 위한 유연성)
- Partner 단일로는 운영 불가 — 반드시 산하에 Group 또는 Institution 1개 이상

#### 4.2.3 Sector별 매핑 (Academy / Enterprise / K12)

Institution.sector 속성에 따라 **UI 레이블이 동적 전환**된다. 내부 DB 필드와 코드 상수는 영어 기반 일관 유지.

| 내부 개념 | Academy (학원) | Enterprise (기업) | K12 (학교) |
|---|---|---|---|
| **L1 Group** | 프랜차이즈 본부 | 지주사 / 본사 | 교육청 / 재단 |
| **L2 Institution** | 가맹학원 (단독 학원 포함) | 계열사 / 지사 | 학교 |
| **Group Admin** | 본부 관리자 | **HR 총괄** | 교육청 관리자 |
| **Institution Admin (공통 용어)** | **학원관리자** | HR 담당자 | 학교 관리자 |
| **Instructor** | 강사 | 사내강사 · 외부강사 | 교사 |
| **Learner** | 학생 | 임직원 | 학생 |
| **Guardian** | 학부모 | (필요 시 사용 — 가족돌봄 교육 등) | 학부모 |

> **용어 확정(v0.6)**
> - "기관장" 용어 폐기 → **"학원관리자"로 통일** (내부 코드: `InstitutionAdmin`)
> - L1 Group Admin의 기업 UI 기본값 = **"HR 총괄"**
> - Guardian은 주로 Academy·K12용이며, Enterprise에서도 필요 시 사용 허용

#### 4.2.4 Sector 속성과 UI 레이블 동적 전환

```typescript
enum Sector {
  ACADEMY = 'academy',
  ENTERPRISE = 'enterprise',
  K12 = 'k12',
}

// UI 레이블 매핑 예시 (i18n 상수)
const LABELS = {
  academy:    { L1: '프랜차이즈 본부', L2: '가맹학원', learner: '학생', instructor: '강사' },
  enterprise: { L1: '지주사',         L2: '계열사',   learner: '임직원', instructor: '사내강사' },
  k12:        { L1: '교육청',         L2: '학교',     learner: '학생',  instructor: '교사' },
};
```

Institution.sector는 L2에서 지정하며, 상위 Group의 sector와 불일치할 경우 경고(혼합 섹터 운영은 허용하되 UI 통일을 위해 Group sector가 기본값).

### 4.3 Organization Entities 정의

| Entity | 계층 | 계정 | 필수 여부 | 주 관리 Role |
|---|---|---|---|---|
| **Platform** | — | 시스템 기본 | — | 시스템관리자 |
| **Partner** | L0 | ❌ (대표자가 role 보유) | 선택 | Partner Admin |
| **Group** | L1 | ❌ | 선택 | Group Admin |
| **Institution** | L2 | ❌ | **필수** | 학원관리자 |
| **Plan / Product** | — | ❌ | 기본 제공 | 시스템관리자 (§7.7.1) |

**Organization 테이블 통합 모델 (§16 연동)**

```typescript
interface Organization {
  id: string;
  level: 'Partner' | 'Group' | 'Institution';
  parentId: string | null;           // 상위 조직 참조
  name: string;
  sector?: Sector;                   // Institution에서 필수, Group은 옵션
  status: 'active' | 'inactive' | 'suspended' | 'withdrawn';
  contractInfo?: object;
  createdAt: Date;
}
```

계층 무결성: Partner → Group → Institution 순서만 허용. Partner의 parentId는 null, Group의 parentId는 Partner 또는 null, Institution의 parentId는 Group/Partner/null 중 하나.

### 4.4 Person Actors 정의 (8종) *(v0.17 보강 — B2B 제휴 데이터 처리 + L2+ 이메일 정책)*

| # | Role (내부 코드) | UI 명칭 (Academy / Enterprise / K12) | 주 소속 | 핵심 책임 |
|---|---|---|---|---|
| 1 | **PlatformAdmin** | 시스템관리자 | Platform | 전체 운영, 감사 로그 |
| 2 | **PartnerAdmin** | 파트너 담당자 | Partner (L0) | 산하 Group/Institution 계약 |
| 3 | **GroupAdmin** | 본부 관리자 / **HR 총괄** / 교육청 관리자 | Group (L1) | 산하 Institution 등록, 플랜 배분, 통계 |
| 4 | **InstitutionAdmin** | **학원관리자** / HR 담당자 / 학교 관리자 | Institution (L2) | 강사·학습자 초대, 액세스코드 발급, 수강 관리 |
| 5 | **Instructor** | 강사 / 사내강사 / 교사 | Institution (L2) | 과정·레슨 제작, 배정 학습자 모니터링 |
| 6 | **Learner** | 학생 / 임직원 | Institution (B2B) or — (B2C) | 학습 진행 |
| 7 | **Guardian** | 학부모 | Learner 계정 연결 | 자녀 진도 조회, 결제 — **Phase 1은 속성만** |
| 8 | **Consumer** | 개인 구독자 | 없음 (B2C) | 개인 학습, 구독 결제 |

#### 4.4.1 B2B 제휴 데이터 처리 정책 *(v0.17 신설)*

B2B 제휴 모드(§16)에서는 **파트너사가 자체 회원 시스템을 운영**하므로, picklass는 학습 진행에 필요한 최소 데이터만 외부에서 수신·저장한다.

| 항목 | 정책 |
|---|---|
| 데이터 흐름 | **파트너사 → picklass** 단방향 (학습 진입 시점에 외부 토큰과 함께 전달) |
| 저장 범위 | **학습 진행에 필요한 항목만 저장** — 학생 식별자(외부 ID), 표시명, 레벨, 수강 과정 매핑. 결제·연락처·민감 정보는 저장 X |
| 동기화 주기 | 토큰 verify 시점(§14.10.5) 또는 파트너사가 명시적으로 푸시할 때만 갱신 |
| 데이터 보존 | 파트너 계약 종료 후 **30일 내 자동 삭제** (학습 로그는 익명화 후 보존) |
| 회원 인증 | picklass 자체 비밀번호 미발급 — **외부 토큰 기반 SSO** (§14.10.5 GET /external/verify) |

#### 4.4.2 이메일 아이디 정책 *(v0.17 신설)*

| 대상 | 정책 |
|---|---|
| **Institution(L2) 이상 관리자** (PlatformAdmin / PartnerAdmin / GroupAdmin / InstitutionAdmin / Instructor) | **확인된 이메일 아이디 사용 필수** — 회원가입 시 이메일 인증 완료해야 활성화. 이메일이 곧 로그인 ID 역할 |
| Learner (B2B) | 기관 발급 액세스코드 + 임의 ID(이메일 미인증 허용) |
| Learner (B2C / Consumer) | 이메일 인증 권장 (필수는 아님, 단 결제 발생 시 인증 강제) |
| Guardian | Learner 속성으로 등록 시 이메일 미인증 허용. 독립 Membership 승격 시(Phase 2+) 인증 필수 |
| **B2B 제휴 진입자** | picklass 이메일 인증 불필요 — 파트너사 인증 신뢰 (§4.4.1) |

이 정책은 **운영 책임 격차**(관리자 = 신원 책임 高 / 학습자 = 학습 자유도 高)를 반영한다.

**Guardian 운영 규칙 (v0.6)**
- **Phase 1**: Learner 계정의 **보호자 속성**(이름·이메일·연락처)로 존재. 독립 로그인 없음.
- **Phase 2+**: 독립 Membership 도입, 자녀 진도 읽기 전용 + B2C 결제 주체 역할
- **Enterprise 섹터**: 기본 미사용이나 "가족돌봄 교육" 등 복지 프로그램 요구 시 활성화 (옵션)

### 4.5 Role 권한 매트릭스 (관리 Scope × 기능)

| 기능 \ Role | Platform | Partner | Group | 학원관리자 | Instructor | Learner | Guardian | Consumer |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Platform 설정 | ✅ | — | — | — | — | — | — | — |
| Partner 계약·수수료 | ✅ | 자기 Partner | — | — | — | — | — | — |
| Group 생성·편집 | ✅ | ✅ 산하 | 자기 Group | — | — | — | — | — |
| Institution 등록 | ✅ | ✅ 산하 | ✅ 산하 | 초기 세팅만 | — | — | — | — |
| 플랜 구독·결제 | ✅ | 산하 대리 | 산하 대리 | ✅ 자기 기관 | — | — | 자녀 대리 | ✅ |
| 액세스코드 발급 | ✅ | 산하 일괄 | 산하 일괄 | ✅ 자기 기관 | — | — | — | — |
| 강사·학습자 초대 | ✅ | 산하 | 산하 | ✅ 자기 기관 | — | — | — | — |
| 과정·레슨 제작 | 조회 | 조회 | 조회 | ✅ 자기 기관 | ✅ 배정 | — | — | — |
| 학습 진행 | — | — | — | 조회 | 조회(배정) | ✅ 본인 | 자녀 조회 | ✅ 본인 |
| 통계·리포트 | ✅ 전체 | 산하 | 산하 | ✅ 자기 기관 | 자기 담당 | 본인 | 자녀 | 본인 |

**권한 상속 원칙**: 상위 Organization Role은 하위의 모든 권한을 **위임 실행 가능**. 단, 감사 로그(§22.4)에 "위임 실행자" 기록 필수.

### 4.6 계정 모델 — 단일 User + 복수 Membership

한 사람 = 한 계정, 그 위에 **여러 역할 컨텍스트(Membership)**를 두는 모델을 채택한다.

```
User (alice@example.com)
  │
  ├─ Membership #1: Institution X에서 "Instructor"       ← 주 업무
  ├─ Membership #2: Institution Y에서 "Learner"         ← 교사 연수
  ├─ Membership #3: Partner P에서 "PartnerAdmin"        ← 파트너 담당자 겸직
  │
  ├─ GuardianLink #1: 자녀 Learner Z 연결                ← 보호자 역할
  │
  └─ Subscription #1: B2C "Consumer" (Speaking 단독 플랜)
```

**Membership 스키마 (개념)**

```typescript
interface Membership {
  id: string;
  userId: string;
  orgType: 'Partner' | 'Group' | 'Institution';
  orgId: string;
  role: 'PartnerAdmin' | 'GroupAdmin' | 'InstitutionAdmin'
      | 'Instructor' | 'Learner';
  status: 'active' | 'inactive' | 'suspended' | 'withdrawn';
  // seatSource: ❌ v0.17 폐기 — 청구는 활성 카운트 + 사용량 기반으로 단순화
  joinedAt: Date;
  revokedAt?: Date;
}
```

> ❌ **v0.17 삭제 — 좌석 배분(seatSource) 혼합 모델**: v0.6에서 채택했던 seatSource(`institution-paid` / `group-allocated` / `partner-allocated`) 분류 모델은 v0.17에서 폐기되었다. 좌석 단위 추적은 더 이상 핵심 청구 모델이 아니며, B2B 제휴(§16) 운영 모델에서는 제휴사가 자체 결제·과금을 관리하므로 picklass 측 seatSource 추적은 불필요하다. **Membership.seatSource 필드 폐기**, 청구는 **Membership 활성 카운트 + 사용량 기반**으로 단순화.

### 4.7 B2B/B2C 채널 분리 원칙

| 채널 | 포함 Actor | 진입 도메인 | 결제 주체 | 소속 Organization |
|---|---|---|---|---|
| B2B-파트너 | PartnerAdmin | admin (제한) | Partner 수수료·정산 | L0 Partner |
| B2B-본부/지주 | GroupAdmin | studio → admin (제한) | Group 일괄 or 산하 개별 | L1 Group |
| B2B-기관 | InstitutionAdmin / Instructor / Learner(B2B) / Guardian(B2B) | studio · www → tutoring | Institution | L2 Institution |
| B2C | Consumer / Guardian(B2C) | www → tutoring/speaking | 개인 | 없음 |
| Platform | PlatformAdmin | admin | — | Platform |

### 4.8 Organization/사용자 상태 체계

| 상태 | 코드 | User 의미 | Organization 의미 |
|---|---|---|---|
| 활성 | active | 정상 활동 | 정상 운영 |
| 휴회 | inactive | 일시적 비활성 (재활성 가능) | 일시적 운영 중단 |
| 정지 | suspended | 관리자 조치 정지 | 계약 위반·미납 등 임시 정지 |
| 탈퇴 | withdrawn | 영구 탈퇴 (복원 불가) | 계약 종료·해지 |

**상태 전환 규칙**
- 활성 ↔ 휴회/정지: 가역
- 탈퇴: 불가역 (데이터 아카이빙)
- Organization 상태 변경 시 **산하 Membership·하위 조직 일괄 영향** — 정책에 따라 연쇄 처리

### 4.9 Speaking 단독 사용자 시나리오

Speaking Tutor를 B2C 경로로 이용하는 Consumer의 주요 페르소나 4종 (기존 §4.6 유지):

1. **초급 학습자(A1–A2)** — 발화 습관 형성, 즉각 교정 중심
2. **중·고급 학습자(B1–C2)** — 논리 전개/유창성 개선
3. **시험 준비생** — 토익 스피킹, 오픽 대비 (Role-Play, QAR 집중)
4. **비즈니스 영어 학습자** — 업무 상황극·Key Expressions 중심

> B2B Learner(기관 소속)의 Speaking 이용은 §3.3.1(튜터링↔스피킹 연동) 및 §3.3.2 경로 A/B 참조.

### 4.10 경계 케이스 (Edge Cases)

| 케이스 | 처리 방식 |
|---|---|
| **강사의 자기 학습** | 같은 Institution에 Learner Membership 추가(활성 Membership 카운트에 반영) 또는 B2C Subscription 병행. Instructor role 단독으로는 학습 기록 미생성(미리보기 샌드박스만 제공) |
| **학부모 결제 (B2C)** | Guardian이 자녀 Consumer 계정을 대신 결제. Phase 1은 Consumer 계정 하위 "결제 담당자" 속성으로 처리, Phase 2에 Guardian 독립 Membership화 |
| **복수 소속 (강사+학생 겸직)** | 단일 User + 2개 Membership. 로그인 후 컨텍스트 선택 UI |
| **B2B→B2C 전환 (졸업 후 개인 학습)** | B2B Membership 만료 → Consumer Subscription 신규. 학습 이력 본인 계정에 계승 |
| **파트너가 복수 Group 관리** | 단일 PartnerAdmin 계정이 복수 L1 Group을 산하로 관리. 대시보드에서 Group 선택 후 드릴다운 |
| **파트너가 Group 없이 Institution 직접 관리** | Partner → Institution 직접 연결 (L1 생략). 중소형 단일 계약에 적용 |
| **Group의 좌석 일괄 배분** *(❌ v0.17 폐기 — Membership 활성 카운트 + 사용량 기반으로 단순화)* | 본부가 1,000석 구매 → 산하 10개 기관에 100석씩 자동/수동 배분 (개념 유지, seatSource 추적은 미사용) |
| **단독 소규모 학원** | L0/L1 없이 L2만 활성화. sector=`academy` |
| **단일 법인 기업교육** | L0/L1 없이 L2만 활성화. sector=`enterprise`, Guardian 미사용 |
| **대기업 그룹의 교차 계열사 이동** | User 동일, Membership 2개(이전 계열사 withdrawn, 신규 계열사 active). 학습 기록 계승 |
| **교사 연수 (학원이 학원에 위탁)** | 위탁 학원(B)이 피연수 학원(A)의 강사를 학생으로 받음. 강사가 A의 Instructor + B의 Learner 2개 Membership 보유 |

---


