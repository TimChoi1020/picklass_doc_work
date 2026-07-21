---
title: 백오피스 (Admin) 기획서
version: v1.4
updated: 2026-07-19
owner_service: backoffice
master_origin: v0.21 §7
depends_on:
  - ../00_공통/02_Actor_조직모델.md
  - ../00_공통/03_비즈니스모델_도메인정책.md
  - ../Speaking/기획서_Speaking.md
  - ./기획서_셀프서브온보딩.md
---

> 📄 본 문서는 통합 기획서 v0.21(→ `_archive/`)에서 분리된 문서이다. 본문 내 **§번호는 구 통합 기획서 기준**이며, §번호 → 신규 문서 매핑은 [README](../README.md)의 매핑표를 참조한다.

## 7. 백오피스 (Admin)

### 7.1 페이지 맵 및 IA *(⚠️ v1.2 재정의 — 5대 메뉴 IA 확장, A1)*

실제 운영 IA는 아래 5대 대메뉴 체계로 확장되었다.

```
/admin
├── 조직관리          (Partner/Group/Institution 3-Tier, 조직상세 탭)
├── 사용자관리        (회원, 권한, 로그인 이력)
├── 콘텐츠 플랫폼      (과정목록·액세스코드, 과정 카테고리, 지문 운영 §7.11)
├── 통계/리포트        (출석률 §7.12, 사용량)
├── 시스템            (상수, API 로그) ← system_admin 전용
└── 파트너연동         (B2B 제휴 관리·API키, → 07_B2B 소유) ← system+partner_admin
```

> 📎 구 페이지맵(dashboard/institute/users/accesscode/ai-modules/billing/system)에서 확장. 시스템·API로그·플랫폼 현황·B2B 학습데이터는 **system_admin 전용**(§7.9.1 Tier1).

### 7.2 대시보드

- 등록 기관 수, 활성 사용자, 월 매출
- 플랜별 기관 분포
- **AI 모듈 사용량** (LLM 토큰/STT·TTS 분)
- **음성 세션 사용량** (Speaking 전용 그래프)

### 7.3 Organization 관리 (Partner / Group / Institution)

> **v0.6 재편:** 기존 "기관관리(Institute)"를 **Organization 3-Tier 관리**로 확장. Academy / Enterprise / K12 섹터 공통 UI에서 L0–L2 전 레벨의 조직 등록·편집·상태 관리를 제공한다.

#### 7.3.1 Organization 목록 및 계층 뷰

- 트리 뷰: Partner → Group → Institution 계층 구조 시각화
- 필터: 섹터(Academy/Enterprise/K12) · 레벨(L0/L1/L2) · 상태
- 검색: 조직명·담당자·연락처·계약번호

#### 7.3.2 Partner 등록 (L0, 선택)

- 파트너사명, 대표 담당자, 계약 유형(중개·리셀·화이트라벨)
- 수수료·정산 정책
- 관리 범위(산하 Group/Institution 설정 권한)

#### 7.3.3 Group 등록 (L1, 선택)

- Group명, sector(Academy/Enterprise/K12), 상위 Partner(있을 경우)
- 학원: 프랜차이즈 본부 / 기업: 지주사·본사 / K12: 교육청
- GroupAdmin(본부 관리자 / **HR 총괄** / 교육청 관리자) 초기 담당자
- Group 단위 플랜 일괄 구매 여부(선택)

#### 7.3.4 Institution 등록 (L2, 필수)

기존 4섹션 구조 유지, sector·parentId 필드 추가:

1. **가입정보** — 아이디(이메일), 초기 비밀번호, **Institution명**, **sector**, **parentId(Partner/Group)**, 담당자
2. **부가정보** — 지점 수, 운영 형태(직영/가맹/직접운영), 학습자 규모
3. **라이선스·요금제** — 플랜, 월비용, 기본좌석수, 단가, API 연동 *(seatSource 기본값은 v0.17에서 폐기)*
4. **계약정보** — 계약상태, 시작/종료일, 자동갱신, 갱신조건

**플랜 선택 시 자동 채움**: monthlyPrice, annualDiscount, baseStudents, pricePerStudent, maxAdminAccounts, apiIntegration이 DEFAULT_PLANS 값으로 초기화되며, 이후 수동 편집 가능.

#### 7.3.5 조직 상태 연쇄 처리

상위 Organization 상태 변경 시 산하 조직과 Membership에 대한 영향 범위를 선택:
- **연쇄 적용(Cascade)** — 산하 전체 동일 상태로 전환
- **독립 유지(Isolate)** — 산하 상태 그대로, 상위만 변경
- **경고만(Warn)** — 상태 변경하되 산하는 별도 재검토 큐로 이동

#### 7.3.6 조직 상세 탭 IA *(v1.2 신설 — A13)*

| 조직 레벨 | 상세 탭 |
|---|---|
| **Partner** | 기본정보 · **레벨 매핑**(§7.8A) |
| **Group** | 기본정보 |
| **Institution** | 기본정보 · **수업설정**(§7.3.7) |

#### 7.3.7 기관 수업설정 정책 *(v1.2 신설 — A8·A9, 설정 SSoT)*

기관(institution) 단위로 학습 옵션을 설정하며, Studio·Tutoring이 이 설정을 참조·강제한다.

| 설정 | 내용 |
|---|---|
| 전체 수업 최대시간 | 세션 상한 |
| 이수조건 기본값 | 발화량·레슨이수율·누적 수업시간 (**체크한 조건만 AND 적용, 미체크는 null=미적용**, A9) |
| 나만의 수업(자유학습) 허용 | ON/OFF (미허용 시 학습자 생성 차단) |
| FRT 모듈 최대시간 | 5~30분 |
| 허용 마이크모드 | PTT/Hybrid/Always-on 중 **최소 1개** |
| 학습자 변경 허용 | 학습자의 옵션 변경 허용 여부 |

> 📎 이 설정은 **Backoffice가 쓰기 소유(SSoT)**, [Studio §8.11](../Studio/기획서_Studio.md)은 읽기·강제, [Tutoring §11.8.6](../Tutoring/기획서_Tutoring.md)은 학습자 게이팅. 옵션 값 도메인(마이크모드·FRT 종료조건 등)은 [Speaking §14](../Speaking/기획서_Speaking.md) 소유.

### 7.3A 기존 명칭 (Institute) 호환 (Deprecated)

기존 `/admin/institute` 경로는 L2 Institution 전용 뷰로 리다이렉트. 새로운 `/admin/organizations`가 3-Tier 통합 관리 진입점.

### 7.4 사용자관리 (Users) *(⚠️ v0.10 정합화)*

- 역할별 통계 카드 (학원관리자/강사/학생)
- 테이블 컬럼: 역할·소속·사용자명·아이디·액세스코드·상태·활성일·작업
- **등록 방식**: 관리자는 institute에서, 강사/학생은 액세스코드 사전 생성
- **아이디 동시 생성 옵션**: `{prefix}{seq}@{domain}.pick` 형식

#### 7.4.1 사용자 일괄 생성 워크플로 *(✅ 구현 완료 — 2026-04-24)*

> 기관 단위로 강사/학생 계정을 **대량 생성하여 즉시 로그인 가능한 상태**로 배포하는 기능. v0.10에서 정식 워크플로로 등재.
>
> **참조**: `picklass-backoffice/docs/users/20260424_사용자일괄생성.md`

**진입 경로**

```
/admin/users
   └─ "사용자 일괄 생성" 버튼 클릭
        ↓
   /admin/users/access-code (페이지 제목: "아이디 일괄 생성")
        ↓
   사용자 유형 (강사/학생) + 소속 기관 + 생성 개수(1~1000) + 고유코드(4자리)
        ↓
   "생성" 클릭 → 액세스코드 + 사용자 계정 동시 생성
        ↓
   "초기 비밀번호는 아이디와 동일합니다" alert
        ↓
   /admin/users 리다이렉트
```

**자동 생성 규칙**

| 항목 | 규칙 |
|---|---|
| `userId` | `{고유코드소문자}{3자리일련번호}@{고유코드소문자}.pick` 형식 (예: `abcd001@abcd.pick`) |
| `password` | `userId와 동일` (임시, `isTempPassword: true`) |
| 일련번호 시작값 | 동일 도메인 기존 사용자 수 + 1 (재실행 시 충돌 방지) |
| `registrationExpiry` | 생성일 + **365일** 고정 |
| `usagePeriodDays` | **365** 고정 |
| 사용자 statusCode | **`active`** (즉시 활성, 별도 활성화 플로우 불필요) |
| 액세스코드 statusCode | **`active`** |
| 액세스코드 동시 생성 | **항상 함께 생성** (체크박스 옵션 제거됨) |

**기술 부채 / 알려진 제한**

| 항목 | 상태 |
|---|---|
| UI는 1,000개 허용, 서비스 내부 `Math.min(..., 100)` 클램핑 | ⚠️ 100개 상한 버그 — `access-code.service.ts:53` 수정 필요 |
| `isTempPassword: true` 강제 변경 로직 | 📋 미구현 — 첫 로그인 시 비밀번호 변경 강제 라우팅 후속 작업 |
| `generated_password` 평문 DB 저장 | ⚠️ 보안 검토 필요 — 배포 확인 후 null 처리 검토 |
| 성공 메시지 `alert()` | 📋 toast UI로 교체 예정 |

**API 계약**

```typescript
// POST /access-codes
interface CreateAccessCodeDto {
  roleCode: 'teacher' | 'student';
  institutionId: string;            // UUID
  count: number;                    // 1~1000 (서비스 내부는 100 클램핑 — 버그)
  registrationExpiry: string;       // ISO date (생성일 +365일 고정)
  usagePeriodDays: number;          // 365 고정
  statusCode?: string;              // 'active' 고정
  createUserSimultaneously?: boolean; // true 고정
  userIdPrefix?: string;            // 고유코드 소문자
  userIdDomain?: string;            // '{고유코드소문자}.pick'
}

interface AccessCodeResponse {
  id, code, institutionId, institutionName, roleCode,
  userId, courseId, statusCode,
  registrationExpiry, usagePeriodDays,
  generatedUserId: string;          // 평문 (배포용)
  generatedPassword: string;        // 평문 (배포용)
  activatedAt, createdAt;
}
```

### 7.5 액세스코드 관리

- **형식**: 6자리 (A-Z [I, O 제외] + 2-9 [0, 1 제외] = 32종 조합)
- **혼동 방지**: I↔1, O↔0 혼동 문자 제거
- **고유성**: 시스템 내 중복 생성 절대 불가
- **상태**: 활성 / 예약됨 / 사용됨 / 만료됨
- **초기 상태**: 생성 직후 `미사용` 고정 (등록 대기)
- **유효기간**: 1M / 3M / 6M / 12M (발급일 + 일수)
- **생성 단위**: **1~1,000개 일괄 생성** (v0.7 확정)
- **학생용**: 제공 과정 선택 필수 (강사는 과정 선택 미표시)
- **재발급**: 만료·오류 코드는 폐기 후 신규 생성

> 📎 **일괄 생성 상한 변경 이력**: PDF 초안 v1.0(2026-03-05)은 100개 상한, v0.7(2026-04-21)에서 **1,000개로 확정**. 대량 배포 기관(프랜차이즈 본부·기업 지주사) 시나리오 대응을 위한 확장.

**Speaking 독립 앱 확장 (v0.5.3)**
- **과정 단위 인증코드**: Speaking 단독 앱 경로 A에서 사용. 인증코드 1건 = 과정 1건 수강권. 사용자가 입력하면 해당 과정이 "수강 중인 과정" 목록에 추가되며, 과정 내 모든 레슨을 추가 인증 없이 수강 가능.
- **사용 기록 단위**: 인증코드 ↔ 사용자 ↔ 과정 3축으로 매핑되며, 동일 과정 재사용(연장) 시 신규 코드 발급 필요.

> ✅ **배포 횟수·수강 인원 산출 정의 (v1.2, A12)**: **배포 횟수 = 과정에 연결된 액세스코드 전체 수** / **수강 인원 = 그중 활성(active) 코드 수**. 구 정의 "액세스코드 발급 수"에서 정정.

### 7.6 수업 모듈 관리 (AI Modules)

#### 7.6.1 Reading/Writing 모듈 마스터
| 모듈ID | 이름 | 상태 | 포함 플랜 | 월 한도 |
|---|---|---|---|---|
| ai-generate | AI 생성 | 활성 | Pro/Ent | 무제한 |
| tts | TTS | 활성 | Pro/Ent | 10,000자 |
| strategic-reading | 전략적 읽기 | 활성 | Pro/Ent | 무제한 |
| word-analysis | 단어 분석 | 활성 | 모든 플랜 | 무제한 |
| stt | STT | 개발중 | - | - |

#### 7.6.2 Speaking 9모듈 마스터 (§14 상세 연결) *(⚠️ v1.3 — 6→9모듈 재정의)*
- Learn & Study (LRN), Vocabulary Listening & Meaning (VLM), **Shadowing Drill (SHD)**, **Sentence Fill-in Blank (SFB)**, **Sentence Making Drill (SMK)**, **Role-Play Basic (RPB)**, **Role-Play Free (RPF)**, Free Talking (FRT), One Minute Presentation (OMP). *구 EDR(3-stage)→SHD·SFB·SMK, RPL→RPB·RPF 분리 및 그 이전 이력은 [04_통합모듈시스템 §13.1.6](../00_공통/04_통합모듈시스템.md) 폐기·재분리 표 참조.*

#### 7.6.3 플랜 포함 여부 · 월 한도 · 초과 과금
- 기본 포함: 플랜 월정액에 포함
- 초과 사용: 모듈별 초과 요금 자동 계산 (Usage 테이블 기록)

#### 7.6.4 음성 세션별 사용량 집계
- 세션 단위: STT 초, TTS 글자, LLM 토큰 3축 집계
- 기관별·사용자별·일별 대시보드

### 7.7 Billing (청구 관리)

- 요금제별 가입 현황 / 월 매출 추이 (최근 3개월)
- 미납 처리 워크플로 (미납해결 버튼)
- 플랜 변경 시 역사적 가격 보존(기존 계약 유지)
- 페이지네이션: 20행/페이지

### 7.7.1 플랜 상품 구성 옵션 (Product Configuration)

> **v0.5.3 신설**: 플랜 단위로 활성/비활성화하는 기능 토글을 중앙에서 관리한다. 여기에서 설정한 옵션은 해당 플랜을 보유한 모든 사용자·기관에 일괄 반영된다.

| 옵션 코드 | 옵션명 | 설명 | 적용 플랜(예) |
|---|---|---|---|
| `opt.personal-interest-speaking` | **개인 관심사 수업** | Speaking 독립 앱 경로 B(개인 관심사 기반 세션) 활성화. 옵션 ON인 플랜 사용자에게만 UI 노출. | Pro·Enterprise 또는 별도 요금제 |
| `opt.speaking-unlimited` | Speaking 무제한 세션 | 월 세션 한도 해제 | Enterprise |
| `opt.ai-topic-recommender` | AI 추천 주제 엔진 | 최신 뉴스·이슈 기반 주제 자동 생성 (경로 B 전제 기능) | Pro·Enterprise |
| ... | (향후 추가) | | |

**운영 규칙**
- 옵션 토글은 Admin 시스템관리자만 편집 가능
- 기존 플랜 구독자에 대해 옵션을 회수할 경우: 다음 갱신 주기부터 적용 (현재 주기는 유지)
- 옵션 조합의 비정상 상태(예: 경로 B 옵션 ON인데 AI 추천 엔진 OFF) 자동 검증

### 7.7.2 ❌ 좌석 배분 정책 *(v0.17 삭제)*

> ❌ **v0.17 폐기 — 좌석 배분(seatSource) 정책**: v0.6에서 신설했던 좌석 단위 추적·배분 모델은 v0.17에서 폐기되었다. 청구 모델은 **활성 Membership 카운트 + 사용량 기반**으로 단순화되었으며, B2B 제휴 운영(§16)에서는 제휴사가 자체 결제·과금을 관리하므로 picklass 측 좌석 추적이 불필요하다. `seat_allocations` 테이블 폐기, `Membership.seatSource` 필드 폐기.

### 7.8 시스템관리

4개 섹션:
1. **사용자 상태** (active/inactive/suspended/withdrawn)
2. **액세스코드 상태·유효기간**
3. **CEFR 18단계 레벨 시스템**
4. **레벨 ↔ WPM 매핑표** (Speaking 연동)

#### CEFR 18단계 ↔ WPM 매핑

| 카테고리 | 레벨 | CEFR | WPM (L1=50→L18=148 선형보간) |
|---|---|---|---|
| Starter | 1–3 | A1−, A1, A1+ | 50, 56, 62 |
| Beginner | 4–6 | A2−, A2, A2+ | 67, 73, 79 |
| Intermediate | 7–9 | B1−, B1, B1+ | 85, 90, 96 |
| Upper-Intermediate | 10–12 | B2−, B2, B2+ | 102, 108, 113 |
| Advanced | 13–15 | C1−, C1, C1+ | 119, 125, 131 |
| Proficient | 16–18 | C2−, C2, C2+ | 136, 142, 148 |

> 🔁 **v1.1 개정 (WPM 앵커 단일 통일, 2026-07-17)**: 기존 L1=80 앵커(`80+(N−1)×4`)를 **P-ALT v8.1 개정에 맞춰 L1=50으로 통일**(L18=148 고정, 18레벨 선형보간). 위 정수값은 반올림 예시이며 **정확한 반올림 규칙은 코드 소유**. 근거: [06 §15.3.4](../00_공통/06_진단평가_PPD.md).

### 7.8A 파트너 레벨 매핑 *(v1.2 신설 — A11)*

파트너 자체 레벨 체계를 픽클래스 18레벨로 매핑한다.

| 항목 | 정책 |
|---|---|
| 매핑 방식 | 파트너 레벨 → 픽클래스 18레벨 **N:1** |
| 설정 대상 | `type='partner'`에만 설정, 산하 그룹·기관은 **상속**(기관→그룹→파트너 순 해소) |
| 폴백 | 빈 값이면 픽클래스 레벨명 그대로 표시 |
| 용도 | **클라이언트 표시용**(레벨 배지·리포트). 표시 적용은 각 서비스(Speaking/Tutoring) 소유 |

### 7.9 권한 및 감사 로그

- 모든 기관·액세스코드·상태 변경 기록 보관
- 변경자·시간·이전/이후 값 3종 필드
- 보안 감시 로그는 일정 기간 보관

#### 7.9.1 2-Tier 권한 모델 *(v1.2 신설 — A2·A3·A4)*

| Tier | 방식 | 예시 |
|---|---|---|
| **Tier 1 — 메뉴 접근 차단** | 메뉴 자체 미노출 | 시스템·API로그·플랫폼현황·B2B 학습데이터 = **system_admin ONLY** |
| **Tier 2 — 데이터 범위 제한** | 메뉴는 노출, 데이터만 제한(pre-fill+disable) | 조직·통계 화면의 조직 필터 |

- **역할별 데이터 스코프 고정** (A3): partner_admin=파트너🔒 / group_admin=파트너·그룹🔒 / academy_admin=전체🔒. **백엔드에서도 동일 scope 강제**(프론트 우회 차단). "숨기지 않고 고정값으로 잠금" 원칙.
- **PARTNER_ABOVE 권한군** (A4): 파트너연동·과정 카테고리 메뉴는 **system_admin + partner_admin만** 접근(group/academy 제외).
- 권한 기준은 이메일 하드코딩(`restrictedToEmails`)에서 **역할 기반(`allowedRoles`)으로 전환**한다.

> 📎 조직 3-Tier·역할코드 체계(system/partner/group/academy_admin)의 정의 SSoT는 [02_Actor_조직모델](../00_공통/02_Actor_조직모델.md). 본 절은 backoffice 적용 규칙.

### 7.10 기관 브랜딩 (Institution Branding) *(v0.17 신설)*

> 📋 **기획 단계** — 거래 학원·기업이 **자기 기관의 로고·테마 컬러·스킨**을 학습자 화면에 적용할 수 있도록 하는 UI 커스터마이징 기능. B2B 영업의 핵심 요구사항으로 **"우리 회사 영어 교육 플랫폼"으로 보이게 하는 화이트 라벨 효과**를 제공한다.

#### 7.10.1 적용 범위

| 적용 대상 | 학습자 화면 | 강사 화면 | 관리자 화면 |
|---|---|---|---|
| **기관 로고** | ✅ 우상단 + 로딩 화면 | ✅ 헤더 | ✅ 헤더 |
| **테마 컬러 (primary·accent)** | ✅ 버튼·진행바·하이라이트 | ✅ 일부 | ✅ 일부 |
| **스킨 (배경·카드 스타일)** | ✅ 학습자 메인·세션 화면 | ⚠️ 선택적 | ❌ (관리 일관성) |
| **도메인 (서브도메인 별칭)** | ⚠️ 옵션 (`{기관}.tutoring.picklass.com`) | ⚠️ 옵션 | ❌ |
| **이메일 발송 from 주소** | ✅ "{기관명} 영어 학습" | ✅ | — |
| **수료증·리포트 PDF 헤더** | ✅ 기관 로고+이름 | ✅ | ✅ |

> ⚠️ Pickle Agent 캐릭터·UI 패턴(모바일 3분할 등) 등 **picklass 핵심 학습 UX**는 브랜딩 변경 대상에서 제외. 학습 효과·접근성 일관성을 위함.

#### 7.10.2 브랜딩 설정 권한

| 권한 | 설정 가능 항목 |
|---|---|
| PlatformAdmin | 전체 |
| GroupAdmin | 산하 Institution 일괄 적용 |
| **InstitutionAdmin** | **자기 기관 로고·테마·이메일 from·PDF 헤더** |
| Instructor | (없음 — 조회만) |

> 📎 위 표의 역할명은 UI/PascalCase 표기이며, DB 역할코드는 snake_case이다(PlatformAdmin=system_admin, GroupAdmin=group_admin, InstitutionAdmin=academy_admin, Instructor=teacher). 매핑은 [02_Actor_조직모델](../00_공통/02_Actor_조직모델.md) 참조.

#### 7.10.3 데이터 모델 (요약 — 골격)

```sql
table institutions (
  id pk,
  name varchar(255),
  ...
  brand_config jsonb null,    -- v0.17 신설
  ...
);

-- brand_config 구조 (예시)
{
  "logo_url": "https://cdn.../org_123/logo.svg",
  "logo_dark_url": "https://cdn.../org_123/logo-dark.svg",
  "favicon_url": "https://cdn.../org_123/favicon.ico",
  "theme": {
    "primary_color": "#0066CC",
    "accent_color": "#FF6B35",
    "background_style": "subtle-pattern" | "solid" | "gradient"
  },
  "skin_id": "default" | "warm" | "cool" | "minimal",
  "subdomain_alias": "abc-academy",   // 옵션
  "email_from_name": "ABC 어학원 영어 학습",
  "email_from_address": "noreply@abc-academy.com",  // SPF/DKIM 검증 후
  "pdf_header": {
    "logo_url": "...",
    "institution_name": "ABC 어학원",
    "footer_text": "..."
  }
}
```

#### 7.10.4 적용 정책

| 정책 | 내용 |
|---|---|
| **렌더링 방식** | 학습자 진입 시 액세스코드 또는 도메인으로 기관 식별 → `brand_config` 동적 주입 (CSS 변수 + 이미지 URL 교체) |
| **캐싱** | 브랜드 자산은 CDN 캐싱(1시간), 변경 시 즉시 invalidate |
| **승인** | 로고 업로드 시 자동 검수(파일 크기·형식·해상도) + 부적절 콘텐츠 자동 차단 |
| **B2C 영역** | `www.picklass.com` 등 picklass 자체 도메인은 brand_config 미적용 |
| **이메일 도메인 검증** | 사용자 정의 from 주소 사용 시 SPF·DKIM 레코드 검증 필수 |

#### 7.10.5 B2B 제휴와의 관계 (§16 연동)

B2B 제휴 모드에서는 일반적으로 제휴사가 **자체 사이트에 픽클래스 학습 모듈을 임베드**하므로, 외부 사이트 자체가 브랜딩 컨테이너 역할을 한다. 이 경우 picklass 측 brand_config는 보조적으로만 사용된다.

| 운영 형태 | brand_config 역할 |
|---|---|
| 픽클래스 자체 운영 (B2C·일반 B2B 직판) | **주요** — 학습자 화면 전체 브랜딩 |
| Tutoring 통합 임베드 (§3.3.1) | **보조** — picklass 도메인 진입 시에만 적용 |
| 외부 임베드 (§3.3.4, §16) | **보조** — 임베드 위젯 내부 영역만 적용 (외부 사이트 헤더는 제휴사가 브랜딩) |

### 7.11 콘텐츠 플랫폼 — 과정 카테고리 & 지문 운영 *(v1.2 신설)*

#### 7.11.1 과정 카테고리 관리 *(A5·A6·A7)*

| 항목 | 정책 |
|---|---|
| **L1/L2 2-depth** | 대/중 2단계 분류. **파트너 단일 소유**(platform 공통 카테고리 없음). 용도=학습자 탐색·B2B 리포트 집계·라이브러리 구조화 |
| **생성·수정·삭제·소유** | **Backoffice 전담**(강사 Studio는 배정·표시만 — [Studio §8.12](../Studio/기획서_Studio.md)) |
| **배정 규칙** (A6) | 과정은 **L2에만 배정**(L1 단독 금지). **교차 파트너 배정 금지**(카테고리 소유 파트너=과정 소유 파트너). 강사·backoffice 양 경로 동일 규칙 |
| **독립 기관** (A7) | 파트너 미소속 기관은 카테고리 필터 미표시 |

#### 7.11.2 지문(Speaking 핵심데이터) 운영 관리 *(A14)*

지문 `core_expressions`·`core_dialog`의 소급 생성·모니터링은 운영 도구 성격이므로 **backoffice로 일원화**한다. **생성 로직 자체는 [Speaking](../Speaking/기획서_Speaking.md) 소유**, backoffice는 운영 관리 지점만.

### 7.12 통계/리포트 — 출석률 *(v1.2 신설 — A10, 집계 SSoT)*

| 항목 | 정의 |
|---|---|
| **출석** | 해당 날(KST) **1회 이상 로그인** |
| **출석률** | 로그인 일수 / 조회기간 평일 수 × 100 |
| 평일 | 월~금 |
| 조회기간 | 수강기간 ∩ 조회월 |

> 📎 구 `module_histories.started_at` 기준에서 **로그인 이력 기준**으로 변경한 결정. [Tutoring §11.10.1](../Tutoring/기획서_Tutoring.md) CMP_ATTEND가 이 정의를 참조.

---


