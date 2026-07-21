# 기관 관리 - 구현 계획 (Implementation Plan)

> **문서 목적**: spec.md 기준 GAP 분석 및 레이어별 구현 계획
> **기준 문서**: `docs/institute/spec.md`
> **최종 수정**: 2026-03-11 (UI 전면 재작성 후 추가 갭 분석 반영)

---

## 1. GAP 분석 (현재 구현 vs spec.md)

### 1.1 데이터 모델 GAP

| 항목 | spec.md 요구 | 현재 구현 | 변경 필요 |
|------|-------------|----------|---------|
| 기관 관리자 이메일 | `adminEmail` (필수, unique) | `contactEmail` (optional) | ✅ 신규 필드 추가 |
| 요금제 | `plan`: Starter/Pro/Enterprise | `planCode`: lite/pro/enterprise/affiliate | ✅ 값 매핑 변경 |
| 단가 | `unitPrice`: String ("1.0만원") | `pricePerStudent`: Decimal (숫자) | ✅ 포맷 변환 레이어 |
| 계약 기간 | `contractPeriod`: 단일 날짜 | `contractStartDate` + `contractEndDate` (두 날짜) | ✅ `contractEndDate` 매핑 |
| 기관 상태 | `status`: 활성/휴회 (2-state) | `contractStatusCode`: 다중 코드 | ✅ 매핑 레이어 추가 |
| 현재 학생 수 | `currentStudents` (읽기 전용) | `currentStudents` (동일) | ✅ 읽기 전용 강제화 |
| 최대 학생 수 | `maxStudents` | `maxStudents` (동일) | - 변경 없음 |

### 1.2 통계 카드 GAP

| spec.md 요구 | 현재 구현 | 변경 필요 |
|------------|----------|---------|
| 전체 기관 수 | ✅ `totalInstitutions` | - 동일 |
| 활성 기관 | ✅ `activeInstitutions` | - 동일 |
| 휴회 기관 | ❌ 없음 | ✅ 추가 |
| 총 학생 수 | ❌ 없음 (`recentUsers` 대체) | ✅ `SUM(currentStudents)` 추가 |
| 전체 수용 인원 | ❌ 없음 | ✅ `SUM(maxStudents)` 추가 |

### 1.3 검색 필터 GAP

| spec.md 요구 | 현재 구현 | 변경 필요 |
|------------|----------|---------|
| 기관명 검색 | ✅ `name` | - 동일 |
| 관리자 이메일 검색 | ❌ 없음 | ✅ `adminEmail` 필터 추가 |
| 요금제 필터 | ✅ `planCode` | 값 매핑 변경 |
| 상태 필터 | ✅ `contractStatusCode` | ✅ `status` (활성/휴회) 필터로 변경 |
| 계약 기간 범위 | ❌ 없음 | ✅ `contractPeriodFrom/To` 추가 |

### 1.4 UI 구조 GAP

| spec.md 요구 | 현재 구현 | 변경 필요 |
|------------|----------|---------|
| 기관 목록: `/admin/institute` | ✅ 동일 | - 동일 |
| 기관 등록: `/admin/institute/register` (별도 페이지) | ❌ 모달 방식 | ✅ 별도 페이지로 전환 |
| 기관 수정: `/admin/institute/[id]/edit` (별도 페이지) | ❌ 모달 방식 | ✅ 별도 페이지로 전환 |
| 기관명 수정 불가 | ❌ 수정 가능 | ✅ 읽기 전용 강제화 |

---

## 2. 구현 전략

### 2.1 Prisma Schema 전략

**최소한의 스키마 변경** 원칙 적용:
- 기존 필드 유지 (다른 기능과의 호환성)
- 신규 필드(`adminEmail`, `unitPrice`) 추가
- 기존 `contractStatusCode` → 서비스 레이어에서 활성/휴회 2-state로 매핑

**Plan 코드 매핑 전략**:
- DB에는 기존 `planCode` 유지 (`lite`, `pro`, `enterprise`, `affiliate`)
- 응답 레이어에서 spec.md의 `plan` 값 ('Starter', 'Pro', 'Enterprise')으로 변환
- 등록/수정 시 역변환

```
spec plan → DB planCode
'Starter'    → 'lite'
'Pro'        → 'pro'
'Enterprise' → 'enterprise'
```

**Status 매핑 전략**:
- DB의 `contractStatusCode` 값 중 `'active'` → '활성', 나머지 → '휴회'
- 등록/수정 시: '활성' → `'active'`, '휴회' → `'inactive'`

### 2.2 URL 구조 결정

spec.md는 `/admin/institute/[name]/edit`를 제시하지만, **`[id]` 사용을 권장**:
- 기관명에 공백/특수문자 포함 가능성
- Backend API가 id 기반 동작
- name은 immutable이므로 id와 1:1 매핑 가능

**결정**: 프론트엔드 URL은 `/admin/institute/[id]/edit` 사용

---

## 3. 레이어별 구현 계획

### 3.1 Layer 1: Prisma Schema 변경

**파일**: `prisma/schema.prisma`

#### 추가할 필드

```prisma
model Institution {
  // ... 기존 필드 유지 ...

  // [신규] spec.md 요구 필드
  adminEmail  String  @map("admin_email") @db.VarChar(255)  // 기관 관리자 이메일 (필수)
  unitPrice   String  @default("") @map("unit_price") @db.VarChar(50)  // 단가 문자열 (예: "1.0만원")

  // ... 기존 필드 유지 ...
}
```

#### 마이그레이션 고려사항

`adminEmail`은 NOT NULL 필드이므로 기존 데이터 처리 필요:
1. 마이그레이션 파일에서 DEFAULT 값 설정 후 추가
2. 기존 레코드: `contactEmail` 값을 `adminEmail`로 복사, 없으면 임시값 설정
3. 이후 UNIQUE 제약 추가

```sql
-- 마이그레이션 순서
ALTER TABLE institutions ADD COLUMN admin_email VARCHAR(255) NOT NULL DEFAULT '';
ALTER TABLE institutions ADD COLUMN unit_price VARCHAR(50) NOT NULL DEFAULT '';
UPDATE institutions SET admin_email = COALESCE(contact_email, CONCAT(id, '@placeholder.com'));
-- UNIQUE 제약은 데이터 정리 후 별도로 추가
```

#### 변경 후 재빌드

```bash
pnpm run build  # packages/core 빌드
# backend 재시작
```

---

### 3.2 Layer 2: packages/types 변경

**파일**: `packages/types/src/index.ts`

#### 변경 사항

**① `CreateInstitutionDto` 교체** (4-section → spec.md 단순 모델)

```typescript
// 기존 CreateInstitutionDto 유지 (하위 호환) + 신규 추가
export interface CreateInstitutionSimpleDto {
  name: string;               // 2-100자, 등록 후 변경 불가
  adminEmail: string;         // 유효한 이메일, unique
  plan: 'Starter' | 'Pro' | 'Enterprise';
  unitPrice: string;          // 예: "1.0만원", "협의 중"
  maxStudents: number;        // 1 이상
  contractPeriod: string;     // YYYY-MM-DD 미래 날짜
  status: '활성' | '휴회';
}

export interface UpdateInstitutionSimpleDto {
  // name 제외 (변경 불가)
  adminEmail?: string;
  plan?: 'Starter' | 'Pro' | 'Enterprise';
  unitPrice?: string;
  maxStudents?: number;
  contractPeriod?: string;
  status?: '활성' | '휴회';
}
```

**② `InstitutionQueryDto` 확장**

```typescript
export interface InstitutionQueryDto extends PaginationQuery {
  name?: string;
  adminEmail?: string;           // [신규]
  plan?: string;                 // 'Starter' | 'Pro' | 'Enterprise'
  status?: '활성' | '휴회';      // [신규] contractStatusCode 대체
  contractPeriodFrom?: string;   // [신규] YYYY-MM-DD
  contractPeriodTo?: string;     // [신규] YYYY-MM-DD
  // 하위 호환을 위해 기존 필드 유지
  planCode?: string;
  contractStatusCode?: string;
}
```

**③ `InstitutionSimpleResponse` 추가** (spec.md 기준 응답)

```typescript
export interface InstitutionSimpleResponse {
  id: string;
  name: string;
  adminEmail: string;
  plan: 'Starter' | 'Pro' | 'Enterprise';
  unitPrice: string;
  currentStudents: number;       // 읽기 전용
  maxStudents: number;
  contractPeriod: string;        // YYYY-MM-DD
  status: '활성' | '휴회';
  createdAt: string;
  updatedAt: string;
}
```

**④ `InstitutionDashboardResponse` 교체**

```typescript
// 기존 유지 + 신규 추가
export interface InstitutionStatsResponse {
  totalInstitutions: number;     // 전체 기관 수
  activeInstitutions: number;    // 활성 기관
  inactiveInstitutions: number;  // 휴회 기관 [신규]
  totalStudents: number;         // 총 학생 수 [신규] SUM(currentStudents)
  totalCapacity: number;         // 전체 수용 인원 [신규] SUM(maxStudents)
}
```

---

### 3.3 Layer 3: packages/core 서비스 변경

**파일**: `packages/core/src/institution/institution.service.ts`

#### 변경 사항

**① `findAll` - 검색 필터 확장**

```typescript
// 추가할 where 조건
...(query.adminEmail && {
  adminEmail: { contains: query.adminEmail, mode: 'insensitive' },
}),
...(query.plan && { planCode: planToCode(query.plan) }),  // plan → planCode 변환
...(query.status && {
  contractStatusCode: query.status === '활성' ? 'active' : { not: 'active' },
}),
...(query.contractPeriodFrom && {
  contractEndDate: { gte: new Date(query.contractPeriodFrom) },
}),
...(query.contractPeriodTo && {
  contractEndDate: { lte: new Date(query.contractPeriodTo) },
}),
```

**② `create` - spec 필드 처리**

```typescript
async createSimple(dto: CreateInstitutionSimpleDto): Promise<InstitutionSimpleResponse> {
  // 1. name 중복 확인
  // 2. adminEmail 중복 확인
  // 3. plan → planCode 변환
  // 4. status → contractStatusCode 변환
  // 5. contractPeriod → contractEndDate (contractStartDate = 오늘)
  // 6. unitPrice → DB unitPrice 저장
  // 7. 나머지 필수 필드 기본값 설정
}
```

**③ `update` - name 변경 차단**

```typescript
async updateSimple(id: string, dto: UpdateInstitutionSimpleDto): Promise<InstitutionSimpleResponse> {
  // name 필드 무시 (spec: 변경 불가)
  // maxStudents < currentStudents 검증
  // ... 나머지 필드 업데이트
}
```

**④ `getStats` - 5개 통계 카드**

```typescript
async getStats(): Promise<InstitutionStatsResponse> {
  const [total, active, aggregate] = await Promise.all([
    this.prisma.institution.count({ where: { deletedAt: null } }),
    this.prisma.institution.count({
      where: { deletedAt: null, contractStatusCode: 'active' }
    }),
    this.prisma.institution.aggregate({
      where: { deletedAt: null },
      _sum: { currentStudents: true, maxStudents: true }
    }),
  ]);

  return {
    totalInstitutions: total,
    activeInstitutions: active,
    inactiveInstitutions: total - active,
    totalStudents: aggregate._sum.currentStudents ?? 0,
    totalCapacity: aggregate._sum.maxStudents ?? 0,
  };
}
```

**⑤ 내부 변환 헬퍼 추가**

```typescript
// planCode ↔ plan 변환
const CODE_TO_PLAN: Record<string, 'Starter' | 'Pro' | 'Enterprise'> = {
  lite: 'Starter',
  pro: 'Pro',
  enterprise: 'Enterprise',
};
const PLAN_TO_CODE: Record<string, string> = {
  Starter: 'lite',
  Pro: 'pro',
  Enterprise: 'enterprise',
};

// unitPrice 포맷: Decimal → String
function formatUnitPrice(price: Decimal | null, unitPrice: string): string {
  if (unitPrice) return unitPrice;  // DB에 문자열로 저장된 경우
  if (!price) return '협의 중';
  const wan = Number(price) / 10000;
  return `${wan.toFixed(1)}만원`;
}

// toSimpleResponse 변환 메서드 추가
private toSimpleResponse(institution): InstitutionSimpleResponse {
  return {
    id: institution.id,
    name: institution.name,
    adminEmail: institution.adminEmail,
    plan: CODE_TO_PLAN[institution.planCode] ?? 'Starter',
    unitPrice: institution.unitPrice || formatUnitPrice(institution.pricePerStudent),
    currentStudents: institution.currentStudents,
    maxStudents: institution.maxStudents,
    contractPeriod: institution.contractEndDate.toISOString().split('T')[0],
    status: institution.contractStatusCode === 'active' ? '활성' : '휴회',
    createdAt: institution.createdAt.toISOString(),
    updatedAt: institution.updatedAt.toISOString(),
  };
}
```

---

### 3.4 Layer 4: Backend Controller 변경

**파일**: `apps/admin/backend/src/institution/institution.controller.ts`

#### 변경 사항

**① 통계 엔드포인트 추가/변경**

```typescript
// 기존 /institutions/dashboard 유지
// 신규 추가
@Get('stats')
async getStats(): Promise<InstitutionStatsResponse> {
  return this.institutionService.getStats();
}
```

**② 기존 엔드포인트 쿼리 파라미터 확장**

```typescript
@Get()
async findAll(@Query() query: InstitutionQueryDto) {
  // adminEmail, plan, status, contractPeriodFrom, contractPeriodTo 파라미터 추가
  // 기존 planCode, contractStatusCode는 하위 호환 유지
}
```

**③ 등록/수정 엔드포인트에 spec DTO 적용**

```typescript
@Post()
async create(@Body() dto: CreateInstitutionSimpleDto) { ... }

@Put(':id')
async update(@Param('id') id: string, @Body() dto: UpdateInstitutionSimpleDto) { ... }
```

---

### 3.5 Layer 5: Frontend 변경

**기준 경로**: `apps/admin/frontend/src/app/(admin)/admin/institute/`

#### 5.1 디렉터리 구조

```
institute/
├── page.tsx                    # [변경] 기관 목록 (통계 카드 + 검색 + 테이블)
├── register/
│   └── page.tsx                # [신규] 기관 등록 페이지
└── [id]/
    └── edit/
        └── page.tsx            # [신규] 기관 수정 페이지 (id 기반)
```

#### 5.2 `page.tsx` - 기관 목록 페이지 변경

**현재**: 모달 기반 (등록/수정 버튼 → 모달 오픈)
**변경**: 라우터 기반 (등록 버튼 → `/register`, 수정 버튼 → `/[id]/edit`)

**주요 변경점**:
- 통계 카드: 5개 (`totalInstitutions`, `activeInstitutions`, `inactiveInstitutions`, `totalStudents`, `totalCapacity`)
- 검색 필터: `adminEmail`, `status`(활성/휴회), `contractPeriodFrom/To` 추가
- 등록 버튼: `router.push('/admin/institute/register')`
- 수정 버튼: `router.push('/admin/institute/${inst.id}/edit')`
- API 엔드포인트: `/institutions/stats` 추가 호출

**테이블 컬럼** (spec.md 기준):

| 컬럼 | 내용 |
|------|------|
| 기관명 | `name` |
| 관리자이메일 | `adminEmail` (마스킹 표시: `ad**@domain.com`) |
| 요금제 | `plan` (Starter/Pro/Enterprise) |
| 단가 | `unitPrice` ("1.0만원") |
| 현재/최대 학생 수 | `currentStudents/maxStudents` |
| 계약기간 | `contractPeriod` (YYYY-MM-DD) |
| 상태 | `status` 배지 (활성: 초록, 휴회: 주황) |
| 작업 | 수정 버튼 |

#### 5.3 `register/page.tsx` - 기관 등록 페이지 (신규)

**폼 필드** (spec.md 기준 단순 폼):

```
기관명 *           text input        (2-100자, 등록 후 변경 불가 안내)
관리자 이메일 *    email input       (unique 검증)
요금제 *           select            (Starter / Pro / Enterprise)
단가 *             text input        (예: "1.0만원", "협의 중")
최대 학생 수 *     number input      (1 이상)
계약 기간 *        date picker       (미래 날짜)
상태 *             select            (활성 / 휴회)
```

**동작**:
- 제출 → `POST /institutions` (CreateInstitutionSimpleDto)
- 성공 → `/admin/institute`로 이동 + 성공 토스트
- 취소 → `/admin/institute`로 이동

#### 5.4 `[id]/edit/page.tsx` - 기관 수정 페이지 (신규)

**폼 필드**:

```
기관명             text input (읽기 전용, disabled)
관리자 이메일 *    email input
요금제 *           select
단가 *             text input
최대 학생 수 *     number input (현재 학생 수 이상 검증)
계약 기간 *        date picker
상태 *             select
현재 학생 수       text (읽기 전용, 표시만)
```

**동작**:
- 마운트 시 `GET /institutions/:id` → 폼에 기존 값 세팅
- 제출 → `PUT /institutions/:id` (UpdateInstitutionSimpleDto)
- 성공 → `/admin/institute`로 이동 + 성공 토스트

#### 5.5 API Client 변경

**파일**: `apps/admin/frontend/src/lib/api/institution.ts` (신규 또는 기존 파일 수정)

```typescript
// 추가할 함수들
export async function fetchInstitutionStats(): Promise<InstitutionStatsResponse>
export async function createInstitution(dto: CreateInstitutionSimpleDto): Promise<InstitutionSimpleResponse>
export async function updateInstitution(id: string, dto: UpdateInstitutionSimpleDto): Promise<InstitutionSimpleResponse>
export async function fetchInstitutionById(id: string): Promise<InstitutionSimpleResponse>
export async function fetchInstitutions(query: InstitutionQueryDto): Promise<InstitutionListResponse>
```

---

## 4. 구현 순서

```
1. Prisma Schema 수정 (adminEmail, unitPrice 필드 추가)
   → pnpm prisma migrate dev

2. packages/types 수정
   → CreateInstitutionSimpleDto, UpdateInstitutionSimpleDto 추가
   → InstitutionQueryDto 확장
   → InstitutionSimpleResponse, InstitutionStatsResponse 추가

3. packages/core 수정
   → institution.service.ts: 헬퍼 함수, getStats, createSimple, updateSimple 추가
   → findAll 필터 확장
   → pnpm run build

4. Backend Controller 수정
   → GET /institutions/stats 엔드포인트 추가
   → 기존 엔드포인트 DTO 교체
   → 백엔드 재시작

5. Frontend 수정
   5-1. API client 함수 추가 (lib/api/institution.ts)
   5-2. 기관 목록 page.tsx (통계카드 5개 + 검색필터 확장 + 라우터 네비게이션)
   5-3. register/page.tsx 신규 생성
   5-4. [id]/edit/page.tsx 신규 생성
```

---

## 5. 유효성 검사 규칙

### 5.1 등록 시 검증

| 필드 | 규칙 |
|------|------|
| name | 2-100자, 특수문자 제한, unique |
| adminEmail | 이메일 형식, unique |
| plan | 'Starter' \| 'Pro' \| 'Enterprise' |
| unitPrice | 1자 이상 (빈 값 불가) |
| maxStudents | 정수, 1 이상 |
| contractPeriod | 미래 날짜 (오늘 이후) |
| status | '활성' \| '휴회' |

### 5.2 수정 시 검증

| 필드 | 규칙 |
|------|------|
| name | 읽기 전용 (서버에서 무시) |
| adminEmail | 이메일 형식, unique (본인 제외) |
| maxStudents | `currentStudents` 이상 |
| contractPeriod | 미래 날짜 |

### 5.3 에러 메시지

| 상황 | 메시지 |
|------|--------|
| 기관명 중복 | "이미 등록된 기관명입니다." |
| 이메일 중복 | "이미 등록된 관리자 이메일입니다." |
| maxStudents < currentStudents | "최대 학생 수는 현재 학생 수({n}명) 이상이어야 합니다." |
| 과거 계약 기간 | "계약 기간은 오늘 이후 날짜로 설정해주세요." |

---

## 6. 하위 호환성 고려사항

- 기존 `CreateInstitutionDto` (4-section) 타입은 유지 → billing 기능 등 다른 용도로 참조 가능성
- 기존 `GET /institutions/dashboard` 엔드포인트 유지
- 신규 `GET /institutions/stats` 엔드포인트 추가
- 기존 `planCode` 필드 유지 (DB 스키마 변경 최소화)

---

## 7. 관련 파일 목록

| 파일 | 변경 유형 | 설명 |
|------|----------|------|
| `prisma/schema.prisma` | 수정 | `adminEmail`, `unitPrice` 필드 추가 |
| `packages/types/src/index.ts` | 수정 | DTO/Response 타입 추가·확장 |
| `packages/core/src/institution/institution.service.ts` | 수정 | getStats, createSimple, updateSimple, 필터 확장 |
| `apps/admin/backend/src/institution/institution.controller.ts` | 수정 | /stats 엔드포인트 추가, 파라미터 확장 |
| `apps/admin/frontend/src/app/(admin)/admin/institute/page.tsx` | 수정 | 통계 카드 5개, 검색 필터 확장, 라우터 네비게이션 |
| `apps/admin/frontend/src/app/(admin)/admin/institute/register/page.tsx` | 신규 | 기관 등록 페이지 |
| `apps/admin/frontend/src/app/(admin)/admin/institute/[id]/edit/page.tsx` | 신규 | 기관 수정 페이지 |
| `apps/admin/frontend/src/lib/api/institution.ts` | 신규/수정 | API 클라이언트 함수 |

---

## 8. 참고

- 기능 정의 및 정책: `docs/institute/spec.md`
- 기존 복잡 Billing 모델: `docs/billing/billing.md`
- 프로젝트 구조: 루트 `CLAUDE.md`

---

## 10. UI 전면 재작성 후 추가 개발 계획 (2026-03-11)

> **배경**: picklass-background-react(reference) 프로젝트와 100% 동일한 UI 구현을 위해
> institute 관련 모든 Frontend 페이지가 4섹션 폼 구조로 전면 재작성됨.
> 그 결과 UI에는 표시되지만 Backend로 전송/수신되지 않는 필드들이 다수 발생함.

---

### 10.1 현재 상태 요약

#### Frontend 현황

| 파일 | 상태 | 비고 |
|------|------|------|
| `institute/page.tsx` | ✅ 완성 (UI) | 2-테이블 통계, 9컬럼 목록. 단, `adminEmail` 미노출 문제 有 |
| `institute/register/page.tsx` | ⚠️ UI 완성 / API 불완전 | 4섹션 폼 표시, 제출 시 7개 필드만 전송 |
| `institute/[id]/edit/page.tsx` | ⚠️ UI 완성 / API 불완전 | 4섹션 폼 표시, 수정 시 6개 필드만 전송 |

#### 현재 등록/수정 시 실제로 전송되는 필드

```typescript
// register/page.tsx → createInstitution()
{
  name,          // 기관명
  adminEmail,    // 아이디(이메일)
  plan,          // 요금제 (Starter/Pro/Enterprise)
  unitPrice,     // 단가 (자동계산 문자열)
  maxStudents,   // 최대 허용 학생수
  contractPeriod, // 계약 종료일
  status: '활성'  // 고정값
}

// edit/page.tsx → updateInstitution()
{
  adminEmail,
  plan,
  unitPrice,
  maxStudents,
  contractPeriod,
  status         // 회원 상태
}
```

#### UI에 표시되지만 전송되지 않는 필드 (4섹션 폼)

| 섹션 | UI 필드명 | form 변수명 | 백엔드 필드명 (InstitutionResponse) |
|------|-----------|-------------|-------------------------------------|
| 1. 가입정보 | 기관 유형 | `institutionType` | `institutionTypeCode` |
| 1. 가입정보 | 담당자 성명 | `partnerManager` | `contactName` |
| 1. 가입정보 | 담당자 연락처 | `partnerManagerPhone` | `contactPhone` |
| 2. 부가정보 | 지점 수 | `branchCount` | `branchCount` |
| 2. 부가정보 | 운영 형태 | `operationType` | `operationTypeCode` |
| 2. 부가정보 | 현재 수강생 규모 | `currentStudents` | `currentStudents` (readonly) |
| 3. 라이선스 | 허용 관리계정수 | `maxAdminAccounts` | `maxAdminAccounts` |
| 3. 라이선스 | 청구 주기 | `billingCycle` | `billingCycleCode` |
| 3. 라이선스 | 연납 할인율 | `annualDiscount` | `annualDiscountRate` |
| 3. 라이선스 | API 연동비 금액 | `apiCostAmount` | `apiIntegrationFee` |
| 3. 라이선스 | 기술지원비 | `techSupportCost` | `techSupportFee` |
| 3. 라이선스 | 콘텐츠 IP 전환비 | `contentIPCost` | `contentIpFee` |
| 3. 라이선스 | 기타 유의사항 | `notes` | `notes` |
| 4. 계약정보 | 계약 상태 | `contractStatus` | `contractStatusCode` |
| 4. 계약정보 | 계약 시작일 | `contractStartDate` | `contractStartDate` |
| 4. 계약정보 | 자동갱신 여부 | `autoRenewal` | `isAutoRenewal` |
| 4. 계약정보 | 갱신 조건 | `renewalCondition` | `renewalConditions` |

---

### 10.2 추가 개발 갭 분석

#### [GAP-1] 목록 페이지 - `adminEmail` 미노출

**현상**: `institute/page.tsx`의 "기관장 아이디" 컬럼이 `inst.adminEmail`을 참조하지만,
`InstitutionListResponse.data`는 `InstitutionResponse[]` 타입이고 이 타입에는
`adminEmail` 필드가 없음 (`contactEmail`로 존재). 현재 `as unknown as` 캐스팅으로 우회 중.

**해결 방향**:
- Option A: `InstitutionResponse`에 `adminEmail` 필드 추가 (DB `admin_email` → 응답)
- Option B: 목록 응답 타입을 `InstitutionSimpleResponse`로 전환하고 리스트 API가 이를 반환하도록 수정
- **권장**: Option A (기존 타입 유지, 필드 추가만)

---

#### [GAP-2] 등록/수정 - 미전송 필드 17개

**현상**: 4섹션 UI 폼에 17개의 추가 필드가 있으나 API 전송 DTO에 포함되지 않음.

**해결 방향**: `CreateInstitutionSimpleDto` / `UpdateInstitutionSimpleDto` 확장 + 백엔드 처리

---

## 11. Section 3 재설계: 라이선스 & 요금제 상세 편집 (2026-03-14)

### 11.1 변경 개요

**기존:** Section 3는 읽기 전용 정보 표시
**신규:** Section 3는 7행 편집 가능한 레이아웃

#### 7행 구조

| 행 번호 | 필드명 | 타입 | 편집가능 | 자동채움 | 값 예시 |
|--------|--------|------|--------|--------|--------|
| Row 1 | 플랜 | Select | ✅ | - | Starter/Pro/Enterprise |
| Row 2 | 월비용(원)/연납할인 | Input/Input | ✅ | ✅ | 100,000 / 5% |
| Row 3 | 학생기본제공/추가학생당단가(원) | Input/Input | ✅ | ✅ | 10명 / 5,000 |
| Row 4 | 최대학생수/최대관리계정수 | Input/Input | ✅ | ✅ | 50 / 2 |
| Row 5 | API연동(선택)/API연동비용(원) | Select/Input | ✅ | ✅ | 미연동 / 0 |
| Row 6 | 기술지원비(원)/콘텐츠IP전환비(원) | Input/Input | ✅ | - | 0 / 0 |
| Row 7 | 기타 | Textarea | ✅ | - | (유의사항 작성) |

### 11.2 필드 정의

#### Form State 추가 필드
```typescript
interface InstitutionFormData {
  // ... 기존 필드 ...
  
  // Section 3: 라이선스 & 요금제 (신규/변경)
  plan: 'Starter' | 'Pro' | 'Enterprise';           // Row 1
  monthlyPrice: number;                             // Row 2 - 1 (원 단위)
  annualDiscount: string;                           // Row 2 - 2 (예: "5%")
  baseStudents: string;                             // Row 3 - 1 (기본 제공 명)
  pricePerStudent: number;                          // Row 3 - 2 (원 단위)
  maxStudents: string;                              // Row 4 - 1 (최대 학생수)
  maxAdminAccounts: string;                         // Row 4 - 2 (최대 관리계정수)
  apiIntegration: 'Y' | 'N';                        // Row 5 - 1 (API 연동 여부)
  apiCostAmount: string;                            // Row 5 - 2 (원 단위)
  techSupportCost: string;                          // Row 6 - 1 (원 단위)
  contentIPCost: string;                            // Row 6 - 2 (원 단위)
  notes: string;                                    // Row 7 (기타)
}
```

### 11.3 자동 채움 로직 (handleChange)

**플랜 선택 시 트리거:**
```typescript
if (name === 'plan') {
  const planOption = DEFAULT_PLANS.find((p) => p.name === value);
  if (planOption) {
    setForm((prev) => ({
      ...prev,
      [name]: value,
      monthlyPrice: planOption.monthlyPrice,
      annualDiscount: String(planOption.annualDiscount),
      baseStudents: String(planOption.baseStudents),
      pricePerStudent: planOption.pricePerStudent,
      maxStudents: String(planOption.maxStudents),
      maxAdminAccounts: String(planOption.maxAdminAccounts),
      apiIntegration: planOption.apiIntegration,
    }));
  }
}
```

### 11.4 필드 분리 정책

**중요:** Row 3과 Row 4의 필드 분리
- `baseStudents` (학생기본제공): 플랜에서 자동 채움, 사용자 편집 가능
- `maxStudents` (최대학생수): 사용자 직접 입력, 독립적 관리

### 11.5 금액 입력 정책

**원 단위 입력 표준화:**
- 모든 금액은 숫자로만 입력 (콤마 제거)
- DB 저장: 원 단위 숫자 (예: 100000)
- 필드명에 명확한 단위 표시: "(원)"

### 11.6 개발자 체크리스트

#### Frontend
- [ ] DEFAULT_PLANS 상수 import 확인
- [ ] Form state baseStudents, maxStudents, apiCostAmount 등 추가
- [ ] handleChange에 plan 선택 시 자동 채움 로직 구현
- [ ] 숫자 필드 parseInt 처리
- [ ] 7행 레이아웃 구성

#### Backend
- [ ] monthlyPrice, pricePerStudent, baseStudents 등 필드 Db 마이그레이션
- [ ] POST/PATCH API DTO에 필드 추가
- [ ] GET API 응답에 모든 필드 포함
- [ ] 숫자 필드 타입 검증

#### 통합 테스트
- [ ] 플랜 선택 → 자동 채움 확인
- [ ] 금액 원 단위 저장 확인
- [ ] 조회 시 모든 필드 정상 반환 확인

#### [GAP-3] 수정 페이지 - 기존 데이터 로드 불완전

**현상**: `edit/page.tsx`에서 `getInstitution(id)` 호출 후 일부 필드만 폼에 세팅됨.
`InstitutionSimpleResponse`에 없는 필드들은 빈 값으로 표시됨.

```typescript
// 현재 edit/page.tsx에서 실제 세팅되는 필드만
setForm((p) => ({
  ...p,
  adminEmail: data.adminEmail ?? '',
  institutionName: data.name ?? '',
  plan: data.plan ?? '',
  unitPrice: ...,
  baseStudents: String(data.maxStudents ?? ''),
  contractEndDate: data.contractPeriod ?? '',
  status: data.status ?? '',
}));
// 나머지 16개 필드는 항상 빈 값
```

**해결 방향**: 상세 조회 API(`GET /institutions/:id`)가 `InstitutionResponse` (전체 필드) 반환하도록 하고,
edit 페이지에서 모든 필드를 폼에 매핑하는 로직 추가

---

#### [GAP-4] adminEmail 중복확인 - 우회 구현

**현상**: `checkDuplicateAdminEmail()`이 `GET /institutions?adminEmail=...` 호출 후
결과 건수로 중복 여부를 추정하는 방식. 이메일 필터 기능이 백엔드에 구현되어 있지 않으면 동작 안 함.

**해결 방향**: 백엔드에 `GET /institutions/check-email?email=...` 엔드포인트 추가 또는
`findAll` 쿼리의 `adminEmail` 필터 동작 확인

---

### 10.3 레이어별 추가 개발 계획

#### Layer 1: packages/types 확장

**파일**: `packages/types/src/index.ts`

```typescript
// CreateInstitutionSimpleDto 확장 (기존 7개 → 24개)
export interface CreateInstitutionSimpleDto {
  // 기존 필드 유지
  name: string;
  adminEmail: string;
  plan: 'Starter' | 'Pro' | 'Enterprise';
  unitPrice: string;
  maxStudents: number;
  contractPeriod: string;
  status: '활성' | '휴회';

  // [신규] 섹션 1 - 가입정보
  tempPassword?: string;
  institutionType?: string;        // '개인 학원' | '프랜차이즈' | '어학원' | '공교육' | '기업교육'
  partnerManager?: string;
  partnerManagerPhone?: string;

  // [신규] 섹션 2 - 부가정보
  branchCount?: number;
  operationType?: string;          // '직영' | '가맹'
  currentStudents?: number;

  // [신규] 섹션 3 - 라이선스
  maxAdminAccounts?: number;
  billingCycle?: string;           // '월납' | '연납'
  annualDiscount?: number;
  apiCostAmount?: number;
  techSupportCost?: number;
  contentIPCost?: number;
  notes?: string;

  // [신규] 섹션 4 - 계약정보
  contractStatus?: string;         // '협의중' | '계약완료' | '활성' | '만료' | '해지'
  contractStartDate?: string;
  autoRenewal?: string;            // 'Y' | 'N'
  renewalCondition?: string;
}

// UpdateInstitutionSimpleDto 확장 (위 CreateDto에서 name/tempPassword 제외한 모든 필드를 optional로)
export interface UpdateInstitutionSimpleDto {
  adminEmail?: string;
  plan?: 'Starter' | 'Pro' | 'Enterprise';
  unitPrice?: string;
  maxStudents?: number;
  contractPeriod?: string;
  status?: '활성' | '휴회';
  institutionType?: string;
  partnerManager?: string;
  partnerManagerPhone?: string;
  branchCount?: number;
  operationType?: string;
  maxAdminAccounts?: number;
  billingCycle?: string;
  annualDiscount?: number;
  apiCostAmount?: number;
  techSupportCost?: number;
  contentIPCost?: number;
  notes?: string;
  contractStatus?: string;
  contractStartDate?: string;
  autoRenewal?: string;
  renewalCondition?: string;
}

// InstitutionResponse 확장 - adminEmail 필드 추가
export interface InstitutionResponse {
  // ... 기존 필드 유지 ...
  adminEmail: string;   // [신규] admin_email DB 필드에서 매핑
}

// InstitutionSimpleResponse 확장 - 상세 조회에서 전체 폼 데이터 반환
export interface InstitutionSimpleResponse {
  // ... 기존 필드 유지 ...
  institutionType: string;
  partnerManager: string;
  partnerManagerPhone: string;
  branchCount: number;
  operationType: string;
  maxAdminAccounts: number;
  billingCycle: string;
  annualDiscount: number;
  apiCostAmount: number;
  techSupportCost: number;
  contentIPCost: number;
  notes: string | null;
  contractStatus: string;
  contractStartDate: string;
  autoRenewal: string;
  renewalCondition: string | null;
}
```

---

#### Layer 2: Prisma Schema DB 컬럼 현황 분석

**파일**: `prisma/schema.prisma` — `Institution` 모델 기준 전체 컬럼 현황

> ✅ **결론: Prisma 스키마에 모든 필드가 이미 존재함. DB 마이그레이션 불필요.**
> 4섹션 UI 폼에서 사용하는 모든 컬럼이 현재 `institutions` 테이블에 정의되어 있음.

##### 섹션 1 — 가입정보

| DB 컬럼 | Prisma 필드명 | 타입 | 필수여부 | UI 폼 변수명 | 현재 DTO 포함 | 상태 |
|---------|--------------|------|----------|-------------|--------------|------|
| `id` | `id` | UUID (PK) | 필수 | - (자동생성) | - | ✅ 존재 |
| `name` | `name` | VarChar(200) | 필수 | `institutionName` | ✅ `CreateDto.name` | ✅ 존재 |
| `admin_email` | `adminEmail` | VarChar(255) | 필수 (default: "") | `adminEmail` | ✅ `CreateDto.adminEmail` | ✅ 존재 |
| `institution_type_code` | `institutionTypeCode` | VarChar(50) | 필수 | `institutionType` | ❌ 미전송 | ✅ 존재 |
| `contact_name` | `contactName` | VarChar(100) | 필수 | `partnerManager` | ❌ 미전송 | ✅ 존재 |
| `contact_phone` | `contactPhone` | VarChar(20) | 필수 | `partnerManagerPhone` | ❌ 미전송 | ✅ 존재 |
| `contact_email` | `contactEmail` | VarChar(255)? | 선택 | (폼 없음) | - | ✅ 존재 |

##### 섹션 2 — 부가정보

| DB 컬럼 | Prisma 필드명 | 타입 | 필수여부 | UI 폼 변수명 | 현재 DTO 포함 | 상태 |
|---------|--------------|------|----------|-------------|--------------|------|
| `branch_count` | `branchCount` | Int (default: 0) | 필수 | `branchCount` | ❌ 미전송 | ✅ 존재 |
| `operation_type_code` | `operationTypeCode` | VarChar(50) | 필수 | `operationType` | ❌ 미전송 | ✅ 존재 |
| `current_students` | `currentStudents` | Int (default: 0) | readonly | `currentStudents` | ❌ (읽기 전용) | ✅ 존재 |

##### 섹션 3 — 라이선스 & 요금제

| DB 컬럼 | Prisma 필드명 | 타입 | 필수여부 | UI 폼 변수명 | 현재 DTO 포함 | 상태 |
|---------|--------------|------|----------|-------------|--------------|------|
| `plan_code` | `planCode` | VarChar(50) | 필수 | `plan` | ✅ `CreateDto.plan` (변환됨) | ✅ 존재 |
| `unit_price` | `unitPrice` | VarChar(50) (default: "") | 필수 | `unitPrice` | ✅ `CreateDto.unitPrice` | ✅ 존재 |
| `price_per_student` | `pricePerStudent` | Decimal(10,2)? | 선택 | (폼 없음, 자동계산) | - | ✅ 존재 |
| `max_students` | `maxStudents` | Int | 필수 | `baseStudents` | ✅ `CreateDto.maxStudents` | ✅ 존재 |
| `max_admin_accounts` | `maxAdminAccounts` | Int (default: 1) | 필수 | `maxAdminAccounts` | ❌ 미전송 | ✅ 존재 |
| `billing_cycle_code` | `billingCycleCode` | VarChar(50) | 필수 | `billingCycle` | ❌ 미전송 | ✅ 존재 |
| `annual_discount_rate` | `annualDiscountRate` | Decimal(5,2) (default: 0) | 필수 | `annualDiscount` | ❌ 미전송 | ✅ 존재 |
| `api_integration_fee` | `apiIntegrationFee` | Decimal(12,2) (default: 0) | 필수 | `apiCostAmount` | ❌ 미전송 | ✅ 존재 |
| `tech_support_fee` | `techSupportFee` | Decimal(12,2) (default: 0) | 필수 | `techSupportCost` | ❌ 미전송 | ✅ 존재 |
| `content_ip_fee` | `contentIpFee` | Decimal(12,2) (default: 0) | 필수 | `contentIPCost` | ❌ 미전송 | ✅ 존재 |
| `notes` | `notes` | Text? | 선택 | `notes` | ❌ 미전송 | ✅ 존재 |

##### 섹션 4 — 계약정보

| DB 컬럼 | Prisma 필드명 | 타입 | 필수여부 | UI 폼 변수명 | 현재 DTO 포함 | 상태 |
|---------|--------------|------|----------|-------------|--------------|------|
| `contract_status_code` | `contractStatusCode` | VarChar(50) | 필수 | `contractStatus` | ❌ 미전송 (`status`로 우회) | ✅ 존재 |
| `contract_start_date` | `contractStartDate` | Date | 필수 | `contractStartDate` | ❌ 미전송 (서버에서 오늘로 설정) | ✅ 존재 |
| `contract_end_date` | `contractEndDate` | Date | 필수 | `contractEndDate` | ✅ `CreateDto.contractPeriod` (변환됨) | ✅ 존재 |
| `is_auto_renewal` | `isAutoRenewal` | Boolean (default: false) | 필수 | `autoRenewal` | ❌ 미전송 | ✅ 존재 |
| `renewal_conditions` | `renewalConditions` | Text? | 선택 | `renewalCondition` | ❌ 미전송 | ✅ 존재 |

##### 시스템 필드

| DB 컬럼 | Prisma 필드명 | 타입 | 비고 |
|---------|--------------|------|------|
| `created_at` | `createdAt` | Timestamptz | 자동생성 |
| `updated_at` | `updatedAt` | Timestamptz | 자동업데이트 |
| `deleted_at` | `deletedAt` | Timestamptz? | 소프트 삭제 |

##### 요약: 미전송 필드 현황 (DB 컬럼은 존재하나 DTO에 없는 필드)

| 필드 수 | 항목 |
|---------|------|
| ✅ DTO 전송 중 (7개) | `name`, `adminEmail`, `planCode`(via plan), `unitPrice`, `maxStudents`, `contractEndDate`(via contractPeriod), `contractStatusCode`(via status) |
| ❌ DTO 미전송 (15개) | `institutionTypeCode`, `contactName`, `contactPhone`, `branchCount`, `operationTypeCode`, `maxAdminAccounts`, `billingCycleCode`, `annualDiscountRate`, `apiIntegrationFee`, `techSupportFee`, `contentIpFee`, `notes`, `contractStartDate`, `isAutoRenewal`, `renewalConditions` |
| ➕ DB 기본값으로 처리 | `currentStudents`(0), `isAutoRenewal`(false), `annualDiscountRate`(0), `apiIntegrationFee`(0), `techSupportFee`(0), `contentIpFee`(0) |

> **결론**: DB 마이그레이션은 불필요. 필요한 작업은 DTO 확장 + 서비스 레이어 매핑 + Frontend 전송 로직뿐.

---

#### Layer 3: packages/core 서비스 확장

**파일**: `packages/core/src/institution/institution.service.ts`

**① `createSimple()` 확장** - 신규 필드 처리

```typescript
async createSimple(dto: CreateInstitutionSimpleDto): Promise<InstitutionSimpleResponse> {
  // 기존 로직 + 신규 필드 매핑
  return this.prisma.institution.create({
    data: {
      // 기존 매핑 유지
      name: dto.name,
      adminEmail: dto.adminEmail,
      planCode: PLAN_TO_CODE[dto.plan],
      unitPrice: dto.unitPrice,
      maxStudents: dto.maxStudents,
      contractEndDate: dto.contractPeriod ? new Date(dto.contractPeriod) : null,
      contractStatusCode: dto.status === '활성' ? 'active' : 'inactive',

      // [신규] 4섹션 폼 추가 필드
      institutionTypeCode: dto.institutionType ?? '',
      contactName: dto.partnerManager ?? '',
      contactPhone: dto.partnerManagerPhone ?? '',
      branchCount: dto.branchCount ?? 0,
      operationTypeCode: dto.operationType ?? '',
      maxAdminAccounts: dto.maxAdminAccounts ?? 0,
      billingCycleCode: dto.billingCycle ?? '',
      annualDiscountRate: dto.annualDiscount ?? 0,
      apiIntegrationFee: dto.apiCostAmount ?? 0,
      techSupportFee: dto.techSupportCost ?? 0,
      contentIpFee: dto.contentIPCost ?? 0,
      notes: dto.notes ?? null,
      contractStatusCode: dto.contractStatus
        ? contractStatusToCode(dto.contractStatus)
        : 'active',
      contractStartDate: dto.contractStartDate ? new Date(dto.contractStartDate) : new Date(),
      isAutoRenewal: dto.autoRenewal === 'Y',
      renewalConditions: dto.renewalCondition ?? null,
    }
  });
}
```

**② `updateSimple()` 확장** - 신규 필드 처리 (동일 패턴)

**③ `toSimpleResponse()` 확장** - 신규 필드 매핑 추가

**④ `findAll()` 응답에 `adminEmail` 포함** - `InstitutionResponse` 변환 시 `adminEmail` 추가

**⑤ `contractStatusToCode()` 헬퍼 추가**

```typescript
const CONTRACT_STATUS_TO_CODE: Record<string, string> = {
  '협의중': 'negotiating',
  '계약완료': 'contracted',
  '활성': 'active',
  '만료': 'expired',
  '해지': 'terminated',
};
```

---

#### Layer 4: Backend Controller 확장

**파일**: `apps/admin/backend/src/institution/institution.controller.ts`

**① `adminEmail` 필터 파라미터 활성화**

현재 `GET /institutions?adminEmail=...` 쿼리 처리 로직이 서비스에 구현되어 있는지 확인.
미구현 시 `findAll()` where 조건에 추가:

```typescript
...(query.adminEmail && {
  adminEmail: { contains: query.adminEmail, mode: 'insensitive' },
}),
```

**② 이메일 중복 확인 전용 엔드포인트 추가** (선택사항)

```typescript
@Get('check-email')
async checkEmail(@Query('email') email: string) {
  const exists = await this.institutionService.existsByAdminEmail(email);
  return { available: !exists, message: exists ? '이미 사용 중인 이메일입니다.' : '사용 가능한 이메일입니다.' };
}
```

---

#### Layer 5: Frontend 수정

**① `register/page.tsx` - 전체 필드 전송**

현재 7개 필드만 전송하는 `createInstitution()` 호출을 확장된 DTO로 교체:

```typescript
await createInstitution({
  name: form.institutionName,
  adminEmail: form.adminEmail,
  plan: form.plan as 'Starter' | 'Pro' | 'Enterprise',
  unitPrice: form.unitPrice || '협의 중',
  maxStudents: parseInt(form.baseStudents) || 0,
  contractPeriod: form.contractEndDate || '',
  status: (form.status as '활성' | '휴회') || '활성',
  // [신규 추가]
  institutionType: form.institutionType,
  partnerManager: form.partnerManager,
  partnerManagerPhone: form.partnerManagerPhone,
  branchCount: parseInt(form.branchCount) || 0,
  operationType: form.operationType,
  maxAdminAccounts: parseInt(form.maxAdminAccounts) || 0,
  billingCycle: form.billingCycle,
  annualDiscount: parseFloat(form.annualDiscount) || 0,
  apiCostAmount: parseInt(form.apiCostAmount) || 0,
  techSupportCost: parseInt(form.techSupportCost) || 0,
  contentIPCost: parseInt(form.contentIPCost) || 0,
  notes: form.notes,
  contractStatus: form.contractStatus,
  contractStartDate: form.contractStartDate,
  autoRenewal: form.autoRenewal,
  renewalCondition: form.renewalCondition,
});
```

**② `edit/page.tsx` - 기존 데이터 전체 로드**

`getInstitution(id)` 응답에서 모든 폼 필드 세팅:

```typescript
setForm((p) => ({
  ...p,
  adminEmail: data.adminEmail ?? '',
  institutionName: data.name ?? '',
  institutionType: data.institutionType ?? '',
  partnerManager: data.partnerManager ?? '',
  partnerManagerPhone: data.partnerManagerPhone ?? '',
  branchCount: String(data.branchCount ?? ''),
  operationType: data.operationType ?? '',
  currentStudents: String(data.currentStudents ?? ''),
  plan: data.plan ?? '',
  unitPrice: data.unitPrice ?? '',
  baseStudents: String(data.maxStudents ?? ''),
  maxAdminAccounts: String(data.maxAdminAccounts ?? ''),
  billingCycle: data.billingCycle ?? '',
  annualDiscount: String(data.annualDiscount ?? ''),
  apiCostAmount: String(data.apiCostAmount ?? ''),
  techSupportCost: String(data.techSupportCost ?? ''),
  contentIPCost: String(data.contentIPCost ?? ''),
  notes: data.notes ?? '',
  contractStatus: data.contractStatus ?? '',
  contractStartDate: data.contractStartDate ?? '',
  contractEndDate: data.contractPeriod ?? '',
  autoRenewal: data.autoRenewal ?? '',
  renewalCondition: data.renewalCondition ?? '',
  status: data.status ?? '',
}));
```

**③ 목록 페이지 - `adminEmail` 컬럼 데이터 정합성**

`InstitutionResponse`에 `adminEmail` 추가 후 `as unknown as` 캐스팅 제거:

```typescript
// 현재 (타입 캐스팅 우회)
setInstitutions(response.data as unknown as InstitutionSimpleResponse[]);

// 변경 후 (정상 타입 사용)
setInstitutions(response.data);  // InstitutionResponse[] - adminEmail 포함
```

---

### 10.4 추가 구현 우선순위

| 우선순위 | 항목 | 이유 |
|---------|------|------|
| 🔴 필수 | [GAP-1] 목록 `adminEmail` 노출 | 현재 "기관장 아이디" 컬럼이 빈 값 |
| 🔴 필수 | [GAP-2] 등록/수정 미전송 필드 연결 | UI 입력이 저장 안 됨 |
| 🔴 필수 | [GAP-3] 수정 페이지 데이터 로드 완전화 | 폼 초기값이 빈 값 |
| 🟡 권장 | [GAP-4] 중복확인 엔드포인트 전용화 | 현재 우회 방식은 필터 미구현 시 항상 "사용 가능"으로 오판 |
| 🟢 선택 | Prisma 필드 존재 여부 검증 | 기존 마이그레이션 실제 적용 여부 확인 |

---

### 10.5 추가 구현 순서

```
1. Prisma DB 확인
   → adminEmail, unitPrice 컬럼 실제 존재 여부 확인
   → 누락 시 migrate dev 실행

2. packages/types 확장
   → InstitutionResponse에 adminEmail 추가
   → CreateInstitutionSimpleDto / UpdateInstitutionSimpleDto에 17개 필드 추가
   → InstitutionSimpleResponse에 17개 응답 필드 추가
   → pnpm run build (packages/types, packages/core)

3. packages/core 서비스 확장
   → createSimple() / updateSimple()에 17개 필드 처리 추가
   → toSimpleResponse()에 17개 필드 매핑 추가
   → findAll() 응답에 adminEmail 포함
   → contractStatusToCode() 헬퍼 추가
   → pnpm run build

4. Backend Controller 확장
   → adminEmail 검색 필터 확인 및 활성화
   → (선택) /check-email 엔드포인트 추가
   → 백엔드 재시작

5. Frontend 수정
   5-1. register/page.tsx → createInstitution() 호출에 17개 필드 추가
   5-2. edit/page.tsx → getInstitution() 응답에서 17개 필드 폼 세팅
   5-3. edit/page.tsx → updateInstitution() 호출에 17개 필드 추가
   5-4. institute/page.tsx → as unknown as 캐스팅 제거
```

---

## 9. 변경 이력

### 2026-03-11 — 초기 구현 (spec.md 기반 전면 재구성)

| 레이어 | 변경 내용 |
|--------|---------|
| **Prisma Schema** | `adminEmail` VARCHAR(255), `unitPrice` VARCHAR(50) 필드 추가 |
| **DB Migration** | `20260311000000_add_admin_email_unit_price` 적용 |
| **packages/types** | `InstitutionSimpleResponse`, `InstitutionStatsResponse`, `CreateInstitutionSimpleDto`, `UpdateInstitutionSimpleDto` 추가; `InstitutionQueryDto` 확장 (adminEmail, plan, status, contractPeriodFrom/To) |
| **packages/core** | `getStats()`, `createSimple()`, `updateSimple()`, `toSimpleResponse()`, `formatUnitPrice()` 추가; `findAll()` 필터 확장; planCode ↔ plan 매핑 헬퍼 추가 |
| **Backend** | `GET /institutions/stats` 엔드포인트 추가; `POST/PUT` DTO를 Simple 모델로 교체 |
| **Frontend API** | `getInstitutionStats()` 추가; `getInstitution`, `createInstitution`, `updateInstitution` 타입 Simple 모델로 변경 |
| **Frontend 목록** | 통계 카드 5개 (전체/활성/휴회/총학생/수용인원); 검색필터 4개 + 계약기간 범위; 등록/수정 모달 → 라우터 네비게이션 전환 |
| **Frontend 신규** | `/admin/institute/register` 등록 페이지, `/admin/institute/[id]/edit` 수정 페이지 생성 |

### 2026-03-11 — UI 레이아웃 버그 수정

**문제**: 검색 필터 항목 추가(6개)로 인해 `SearchFilter`가 단일 flex-row에서 overflow되어 레이아웃 틀어짐. 통계 카드 5개가 `grid-cols-2`에서 마지막 카드 홀로 배치.

**수정 내용**:

| 파일 | 수정 내용 |
|------|---------|
| `components/common/search-filter.tsx` | `flex flex-col md:flex-row` → `flex flex-wrap` 변경; 각 필드 `sm:max-w-[200px]` 제한 추가; `className` prop 추가 |
| `apps/admin/.../institute/page.tsx` | 통계 카드 그리드 `grid-cols-2 lg:grid-cols-5` → `grid-cols-2 sm:grid-cols-3 lg:grid-cols-5`; 계약기간 날짜 필드 SearchFilter에서 분리하여 별도 행으로 렌더링 (`~` 구분자 포함) |

**영향 범위**: `SearchFilter`를 사용하는 모든 페이지 (users, accesscode 등) - `flex-wrap` 추가는 하위 호환적이며 기존 동작 유지
