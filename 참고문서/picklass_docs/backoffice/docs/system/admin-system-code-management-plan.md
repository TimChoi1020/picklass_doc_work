# 코드 관리 구현 플랜

**페이지:** `/admin/system`
**작성일:** 2026-03-17

---

## 1. 현재 상태 분석

### 1.1 DB (code_groups / code_items)

seed.ts 기준 **8개 코드 그룹**이 이미 DB에 존재:

| # | 그룹 코드 | 그룹명 | 항목 수 | DB 참조 컬럼 |
|---|-----------|--------|---------|-------------|
| 1 | `USER_ROLE` | 사용자 역할 | 4 | `users.role_code`, `access_codes.role_code` |
| 2 | `USER_STATUS` | 사용자 상태 | 4 | `users.status_code` |
| 3 | `INSTITUTION_TYPE` | 기관 유형 | 5 | `institutions.institution_type_code` |
| 4 | `OPERATION_TYPE` | 운영 형태 | 2 | `institutions.operation_type_code` |
| 5 | `PLAN_TYPE` | 요금제 플랜 | 4 | `institutions.plan_code` |
| 6 | `BILLING_CYCLE` | 청구 주기 | 2 | `institutions.billing_cycle_code` |
| 7 | `CONTRACT_STATUS` | 계약 상태 | 5 | `institutions.contract_status_code` |
| 8 | `ACCESS_CODE_STATUS` | 액세스코드 상태 | 3 | `access_codes.status_code` |

> **미등록 그룹 (신규 추가 필요):**
> - `INSTITUTION_STATUS` — 기관 회원 상태 (`institutions.member_status_code`)
> - `ACCESS_CODE_DURATION` — 액세스코드 사용기간 (현재 constants.ts에만 존재)
> - `LEVEL_SYSTEM` — CEFR 18단계 레벨 (현재 constants.ts에만 존재)

### 1.2 Backend

- **CodeController** (`apps/admin/backend/src/code/code.controller.ts`): 읽기 전용 2개 엔드포인트
  - `GET /codes` — 전체 그룹+항목 조회
  - `GET /codes/:groupCode` — 특정 그룹 항목 조회
- **CodeService** (`packages/core/src/code/code.service.ts`): `findAll()`, `findByGroupCode()` 만 구현
- **쓰기(CUD) 기능 미구현** — 항목 추가/수정/삭제/순서변경 API 없음

### 1.3 Frontend

- **현재 page.tsx**: constants.ts 하드코딩 데이터로 useState 초기화 → DB 연동 없음
- **기존 API 유틸**: `src/lib/api/code.ts`에 `getAllCodes()`, `getCodesByGroup()` 존재 (읽기 전용)
- **Tabs 컴포넌트**: `src/components/ui/tabs.tsx` (shadcn/ui, line variant 지원) 이미 존재

### 1.4 공유 타입 (packages/types)

```typescript
// 현재 정의
interface CodeItemResponse { code, name, sortOrder, extraData }
interface CodeGroupResponse { code, name, items: CodeItemResponse[] }

// description 필드 누락 — CodeItem 모델에는 description 컬럼이 있으나 응답 타입에 미포함
```

---

## 2. 구현 범위

### 탭 구성 (총 4개 탭)

관리 항목이 많으므로 아래와 같이 **업무 도메인 기준 4개 탭**으로 분류:

| 탭 | 포함 코드 그룹 | 이유 |
|----|---------------|------|
| **사용자/역할** | `USER_ROLE`, `USER_STATUS`, `INSTITUTION_STATUS`(신규) | 사용자·기관 회원 관련 |
| **기관/계약** | `INSTITUTION_TYPE`, `OPERATION_TYPE`, `PLAN_TYPE`, `BILLING_CYCLE`, `CONTRACT_STATUS` | 기관 등록·계약 관련 |
| **액세스코드** | `ACCESS_CODE_STATUS`, `ACCESS_CODE_DURATION`(신규) | 액세스코드 운영 관련 |
| **레벨시스템** | `LEVEL_SYSTEM`(신규) | CEFR 18단계 레벨 |

---

## 3. 단계별 구현 계획

### Phase 1: Backend — CRUD API 추가

#### 3.1.1 packages/types 수정

`packages/types/src/index.ts`에 추가:

```typescript
// CodeItem 응답에 description 추가
export interface CodeItemResponse {
  id: number;           // 신규: 수정/삭제 시 식별자
  code: string;
  name: string;
  description: string | null;  // 신규
  sortOrder: number;
  extraData: Record<string, unknown> | null;
}

// 항목 생성/수정 DTO
export interface UpsertCodeItemDto {
  id?: number;                  // 있으면 수정, 없으면 생성
  code: string;
  name: string;
  description?: string;
  sortOrder: number;
  extraData?: Record<string, unknown>;
}

// 그룹 단위 일괄 저장 DTO
export interface SaveCodeGroupItemsDto {
  items: UpsertCodeItemDto[];
}
```

#### 3.1.2 packages/core CodeService 확장

`packages/core/src/code/code.service.ts`에 메서드 추가:

| 메서드 | 기능 | 핵심 로직 |
|--------|------|----------|
| `saveGroupItems(groupCode, items)` | 그룹 항목 일괄 저장 | 트랜잭션 내에서: 1) 기존 항목 중 요청에 없는 것 soft-delete(`isActive=false`), 2) id가 있는 항목 update, 3) id가 없는 항목 create |
| `createItem(groupCode, item)` | 단일 항목 추가 | groupCode로 그룹 조회 → 항목 생성 |
| `updateItem(itemId, data)` | 단일 항목 수정 | id로 직접 수정 |
| `deleteItem(itemId)` | 단일 항목 삭제 | soft-delete (isActive=false) |

> **결정 사항: 일괄 저장 vs 개별 CRUD**
>
> 개발 가이드에서 두 가지 방식을 모두 제시했으나, **일괄 저장(PUT) 방식을 주 방식으로 채택**.
> - 이유: 순서 변경, 다수 항목 동시 편집이 빈번하므로 한번에 저장이 UX 적합
> - 개별 CRUD는 단일 항목 삭제 시에만 보조적으로 사용

#### 3.1.3 Backend Controller 확장

`apps/admin/backend/src/code/code.controller.ts`에 엔드포인트 추가:

```
PUT    /codes/:groupCode/items    — 그룹 항목 일괄 저장 (순서 포함)
DELETE /codes/:groupCode/items/:itemId — 단일 항목 삭제
```

#### 3.1.4 Seed 데이터 보강

`prisma/seed.ts`에 누락 그룹 3개 추가:

```typescript
// 신규 추가
{ code: 'INSTITUTION_STATUS', name: '기관 회원 상태', description: '활성, 휴회, 정지, 탈퇴' }
{ code: 'ACCESS_CODE_DURATION', name: '액세스코드 사용기간', description: '1개월, 3개월, 6개월, 1년' }
{ code: 'LEVEL_SYSTEM', name: '레벨시스템', description: 'CEFR 기반 18단계 레벨' }

// ACCESS_CODE_DURATION items — extraData에 days 저장
{ code: '1month', name: '1개월', sortOrder: 1, extraData: { days: 30, description: '1개월 사용 기간' } }
{ code: '3month', name: '3개월', sortOrder: 2, extraData: { days: 90, description: '3개월 사용 기간' } }
{ code: '6month', name: '6개월', sortOrder: 3, extraData: { days: 180, description: '6개월 사용 기간' } }
{ code: '1year',  name: '1년',   sortOrder: 4, extraData: { days: 365, description: '1년 사용 기간' } }

// LEVEL_SYSTEM items — extraData에 cefrLevel, category 저장
{ code: 'level_1',  name: '레벨 1',  sortOrder: 1,  extraData: { cefrLevel: 'A1-', category: 'Starter' } }
{ code: 'level_2',  name: '레벨 2',  sortOrder: 2,  extraData: { cefrLevel: 'A1',  category: 'Starter' } }
// ... (총 18개)

// INSTITUTION_STATUS items
{ code: 'active',    name: '활성', sortOrder: 1, extraData: { description: '활성 기관 상태' } }
{ code: 'inactive',  name: '휴회', sortOrder: 2, extraData: { description: '휴회 상태' } }
{ code: 'suspended', name: '정지', sortOrder: 3, extraData: { description: '일시적 정지 상태' } }
{ code: 'withdrawn', name: '탈퇴', sortOrder: 4, extraData: { description: '기관 탈퇴 상태' } }
```

---

### Phase 2: Frontend — 페이지 리팩토링

#### 3.2.1 API 유틸 확장

`apps/admin/frontend/src/lib/api/code.ts`에 추가:

```typescript
// 그룹 항목 일괄 저장
export async function saveCodeGroupItems(groupCode: string, items: UpsertCodeItemDto[]): Promise<CodeItemResponse[]>

// 단일 항목 삭제
export async function deleteCodeItem(groupCode: string, itemId: number): Promise<void>
```

#### 3.2.2 페이지 구조 변경

**파일:** `apps/admin/frontend/src/app/(admin)/admin/system/page.tsx`

**변경 전:**
```
시스템 관리 (h2)
└── 코드관리 (h3)
    ├── [card] 사용자상태 (하드코딩)
    ├── [card] 레벨시스템 (하드코딩)
    ├── [card] 액세스코드 상태 (하드코딩)
    └── [card] 액세스코드 사용기간 (하드코딩)
```

**변경 후:**
```
시스템 관리 (h2)
└── 코드관리 (h3)
    └── [Tabs] (shadcn/ui, variant="line")
        ├── [탭1: 사용자/역할]
        │   ├── [card] 사용자 역할 (USER_ROLE)
        │   ├── [card] 사용자 상태 (USER_STATUS)
        │   └── [card] 기관 회원 상태 (INSTITUTION_STATUS)
        │
        ├── [탭2: 기관/계약]
        │   ├── [card] 기관 유형 (INSTITUTION_TYPE)
        │   ├── [card] 운영 형태 (OPERATION_TYPE)
        │   ├── [card] 요금제 (PLAN_TYPE)
        │   ├── [card] 청구 주기 (BILLING_CYCLE)
        │   └── [card] 계약 상태 (CONTRACT_STATUS)
        │
        ├── [탭3: 액세스코드]
        │   ├── [card] 액세스코드 상태 (ACCESS_CODE_STATUS)
        │   └── [card] 사용기간 (ACCESS_CODE_DURATION)
        │
        └── [탭4: 레벨시스템]
            └── [card] CEFR 레벨 (LEVEL_SYSTEM)
```

#### 3.2.3 공통 테이블 컴포넌트 추출

각 코드 그룹마다 반복되는 인라인 편집 테이블 패턴을 **재사용 컴포넌트**로 추출:

**새 파일:** `apps/admin/frontend/src/app/(admin)/admin/system/_components/code-group-table.tsx`

```typescript
interface CodeGroupTableProps {
  groupCode: string;                    // 코드 그룹 코드
  title: string;                        // 카드 헤더 타이틀
  columns: ColumnDef[];                 // 컬럼 정의 (name, code, extraData 필드들)
  items: WithId<CodeItemResponse>[];    // 현재 데이터
  onItemsChange: (items) => void;       // 상태 변경 콜백
  onSave: () => void;                   // 전체 저장 콜백
  newItemDefaults: Partial<CodeItemResponse>; // 새 행 추가 시 기본값
}
```

**컬럼 정의 유연성:**
```typescript
interface ColumnDef {
  key: string;            // 'code' | 'name' | 'description' | 'extraData.days' 등
  label: string;          // 컬럼 헤더 텍스트
  type: 'text' | 'number' | 'textarea' | 'select';
  width?: string;         // CSS 너비
  options?: { value: string; label: string }[];  // select 타입일 때
  editable?: boolean;     // 기본 true
}
```

이 컴포넌트가 기존 page.tsx의 반복 패턴(인라인 편집, ✓저장, -삭제, ↑↓순서변경, +추가)을 **모두 캡슐화**.

#### 3.2.4 데이터 로딩 전략

```
페이지 마운트
  → GET /codes (전체 코드 그룹 + 항목 한번에 조회)
  → 응답을 groupCode 기준으로 분류하여 각 탭에 배포
  → WithId 패턴으로 _uid 부여 (기존 패턴 유지)

저장 시
  → PUT /codes/:groupCode/items (해당 그룹만 일괄 저장)
  → 성공 시 toast 메시지 (alert() 제거)
  → 응답 데이터로 state 갱신 (서버에서 발급한 id 반영)

삭제 시
  → FormModal 확인 후
  → 신규 항목(_uid만 있고 서버 id 없음): 클라이언트에서 직접 제거
  → 기존 항목(서버 id 있음): DELETE /codes/:groupCode/items/:itemId → state 갱신
```

#### 3.2.5 레벨시스템 탭 특수 처리

레벨시스템은 다른 코드 그룹과 구조가 다름 (extraData에 cefrLevel, category 저장):

- **컬럼 구성:** 레벨번호(sortOrder) | CEFR 레벨(extraData.cefrLevel) | 카테고리(extraData.category, select)
- **카테고리 옵션:** `Starter`, `Beginner`, `Intermediate`, `Upper-Intermediate`, `Advanced`, `Proficient`
- **레벨 번호 중복 체크:** 저장 전 클라이언트에서 검증
- **자동 번호 부여:** 새 행 추가 시 `max(level) + 1`

#### 3.2.6 constants.ts 정리

DB에서 코드를 동적으로 로드하므로, constants.ts의 하드코딩 상수들의 역할 변경:

- `USER_STATUSES`, `ACCESS_CODE_STATUSES`, `ACCESS_CODE_DURATIONS`, `LEVEL_SYSTEM` → **fallback 용도로만 유지** (API 실패 시)
- `INSTITUTION_STATUSES` → DB `INSTITUTION_STATUS` 그룹과 통합
- 향후 다른 페이지도 DB 코드를 사용하도록 점진 마이그레이션

---

## 4. 파일 변경 목록

| 구분 | 파일 경로 | 작업 |
|------|----------|------|
| **타입** | `packages/types/src/index.ts` | CodeItemResponse에 id/description 추가, UpsertCodeItemDto·SaveCodeGroupItemsDto 추가 |
| **Core** | `packages/core/src/code/code.service.ts` | saveGroupItems, deleteItem 메서드 추가 |
| **Backend** | `apps/admin/backend/src/code/code.controller.ts` | PUT/DELETE 엔드포인트 추가 |
| **Seed** | `prisma/seed.ts` | INSTITUTION_STATUS, ACCESS_CODE_DURATION, LEVEL_SYSTEM 그룹+항목 추가 |
| **API 유틸** | `apps/admin/frontend/src/lib/api/code.ts` | saveCodeGroupItems, deleteCodeItem 함수 추가 |
| **컴포넌트(신규)** | `apps/admin/frontend/src/app/(admin)/admin/system/_components/code-group-table.tsx` | 공통 인라인 편집 테이블 컴포넌트 |
| **페이지** | `apps/admin/frontend/src/app/(admin)/admin/system/page.tsx` | 탭 UI + DB 연동으로 전면 리팩토링 |

---

## 5. 유의사항

### 5.1 코드값 변경 위험

DB의 코드값(`active`, `inactive` 등)을 다른 테이블에서 직접 참조 중. 코드값 변경은 곧 **데이터 정합성 파괴** 가능.

→ **대응:** code 필드는 기존 항목에 대해 **읽기 전용(disabled)**으로 처리. 새 항목만 code 편집 가능.
→ 추후 코드값 변경이 필요하면 마이그레이션 스크립트와 함께 처리.

### 5.2 extraData JSON 구조 일관성

코드 그룹마다 extraData에 저장하는 필드가 다름:
- `USER_ROLE`: `{ color, badge_bg }`
- `USER_STATUS`: `{ color, icon }`
- `ACCESS_CODE_DURATION`: `{ days, description }`
- `LEVEL_SYSTEM`: `{ cefrLevel, category }`

→ **대응:** CodeGroupTable 컴포넌트의 ColumnDef에서 `extraData.필드명` 형태의 dot notation으로 접근할 수 있게 설계.

### 5.3 마지막 1개 삭제 방지

기존 정책: 각 그룹에서 마지막 1개 항목은 삭제 불가 (삭제 버튼 숨김) → 유지.

### 5.4 패키지 빌드 순서

CLAUDE.md 규칙: `packages/core` 수정 후 `pnpm run build` + 백엔드 재시작 필수.
→ 구현 순서: types → core → backend → frontend

### 5.5 기존 코딩 규칙 준수

- `any` 타입, `@ts-ignore` 금지
- API 호출 시 `fetchApi` 유틸리티 사용 필수
- `moveUp`/`moveDown`은 `@/lib/utils`에서 import
- 삭제 확인은 `FormModal` 사용 (`confirm()` 금지)
- `key={item._uid}` (index key 금지)

---

## 6. 구현 순서 요약

```
Step 1: packages/types — DTO/응답 타입 추가
Step 2: packages/core — CodeService CRUD 메서드 추가
Step 3: apps/admin/backend — Controller 엔드포인트 추가
Step 4: prisma/seed.ts — 누락 그룹 3개 시딩 추가
Step 5: pnpm run build (packages) + seed 실행
Step 6: apps/admin/frontend/src/lib/api/code.ts — 쓰기 API 함수 추가
Step 7: _components/code-group-table.tsx — 공통 컴포넌트 생성
Step 8: system/page.tsx — 탭 UI + DB 연동 리팩토링
Step 9: 검증 — 빌드 + 동작 테스트
```
