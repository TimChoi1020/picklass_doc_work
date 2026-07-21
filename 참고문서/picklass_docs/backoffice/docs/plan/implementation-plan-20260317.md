# 플랜관리 구현 계획서

**작성일:** 2026-03-17
**경로:** `/admin/plans`

---

## 1. 현재 상태 분석

### 1.1 문서 분석 요약

두 개의 기획 문서가 존재합니다:

| 문서 | 핵심 내용 |
|------|-----------|
| `plans.md` (03-14) | 플랜 개념 정의, DEFAULT_PLANS 상수 기반, Plan 인터페이스, 기관 연동 시나리오, 향후 DB 마이그레이션 언급 |
| `admin-plan-20260316.md` (03-16) | 구체적 UI 구현 명세 — 2행 테이블, Fragment 패턴, FormModal 삭제, moveUp/moveDown, 코딩 룰 |

### 1.2 이미 구현된 것

- **프론트엔드 UI** ([plans/page.tsx](../../apps/admin/frontend/src/app/(admin)/admin/plans/page.tsx)): 2행 테이블 레이아웃이 `admin-plan-20260316.md` 명세대로 구현 완료
  - Fragment 기반 2행 렌더링 ✅
  - FormModal 삭제 확인 ✅
  - moveUp/moveDown 순서 변경 ✅
  - 개별 저장 버튼(✓) → `alert()` 호출만 (미연동) ⚠️
  - 행 추가(+) ✅
- **시드 데이터**: `PLAN_TYPE` 코드 그룹 존재 (Lite, Pro, Enterprise, 제휴 — 4개 항목)
- **DB 스키마**: `Institution.planCode` 필드가 `code_items.code`를 참조하는 구조
- **백엔드**: `PLAN_TO_CODE` / `CODE_TO_PLAN` 매핑으로 프론트 Plan명 ↔ DB planCode 변환 중

### 1.3 핵심 문제점

| # | 문제 | 영향 |
|---|------|------|
| 1 | **데이터 이중화**: 프론트 `DEFAULT_PLANS` 상수와 DB `PLAN_TYPE` 코드 그룹이 별도로 존재하며 데이터가 불일치 | 프론트 Starter ≠ DB Lite, 가격/할인 정보가 상수에만 존재 |
| 2 | **저장 미연동**: 플랜 편집 UI가 있지만 저장 시 `alert()`만 호출, 실제 영속 없음 | 새로고침 시 편집 내용 소멸 |
| 3 | **플랜 상세정보 DB 미저장**: 월비용, 기본학생수, 추가단가 등이 DB `PLAN_TYPE.extraData`에 부분적으로만 존재 | 플랜 선택 시 기관 필드 자동채움이 상수에 의존 |
| 4 | **기관 등록/수정 페이지**: 여전히 `DEFAULT_PLANS` 상수에서 플랜 목록과 기본값을 가져옴 | 코드 관리 시스템과 단절 |

### 1.4 데이터 불일치 상세

| 항목 | `DEFAULT_PLANS` (상수) | `PLAN_TYPE` (DB 시드) |
|------|----------------------|---------------------|
| 플랜 이름 | Starter, Pro, Enterprise | Lite, Pro, Enterprise, 제휴 |
| 플랜 코드 | (없음, name 기반) | lite, pro, enterprise, affiliate |
| 가격 정보 | monthlyPrice, pricePerStudent 등 상세 | extraData에 price_per_student, description만 |
| 기관 연결 | `PLAN_TO_CODE['Starter'] = 'lite'` | `planCode = 'lite'` |

---

## 2. 구현 전략

### 2.1 방향: `PLAN_TYPE` 코드 그룹의 `extraData` 확장

**별도 `plans` 테이블을 만들지 않고**, 기존 코드 관리 시스템(`code_groups` / `code_items`)의 `PLAN_TYPE` 그룹을 확장합니다.

**근거:**
- 이미 `PLAN_TYPE` 코드 그룹과 시드 데이터가 존재
- `Institution.planCode`가 코드 기반으로 동작 중
- 코드 관리 시스템(`/admin/system`)과 일관된 패턴
- `extraData` JSONB 필드로 플랜별 상세 정보를 유연하게 저장 가능

### 2.2 `PLAN_TYPE` extraData 확장 구조

```typescript
// code_items 의 extraData 구조 (PLAN_TYPE 그룹)
interface PlanExtraData {
  monthlyPrice: number;       // 월 비용 (원), 0 = 협의
  annualDiscount: string;     // 연납 할인율 (예: '10%')
  baseStudents: number;       // 기본 제공 학생 수
  pricePerStudent: number;    // 추가 학생당 단가 (원), 0 = 협의
  maxStudents: number;        // 최대 학생 수, 0 = 무제한
  maxAdminAccounts: number;   // 최대 관리 계정 수
  apiIntegration: string;     // 'Y' | 'N'
  apiCost: number;            // API 연동 비용 (원/월), 0 = 무료/협의
  unitPrice: string;          // 표시용 단가 (예: '1만원/명', '협의')
  description: string;        // 플랜 설명
  enabled: boolean;           // 사용 여부
}
```

---

## 3. 단계별 구현 계획

### 3.1 Step 1 — 시드 데이터 확장

**파일:** `prisma/seed.ts`

`PLAN_TYPE` 항목의 `extraData`를 플랜 상세정보로 확장합니다.

```
PLAN_TYPE:
  lite → { monthlyPrice: 100000, annualDiscount: '5%', baseStudents: 10,
           pricePerStudent: 5000, maxStudents: 50, maxAdminAccounts: 2,
           apiIntegration: 'N', apiCost: 0, unitPrice: '5,000원/명',
           description: '월 10만원', enabled: true }
  pro  → { monthlyPrice: 1000000, annualDiscount: '10%', baseStudents: 50,
           pricePerStudent: 10000, maxStudents: 500, maxAdminAccounts: 5,
           apiIntegration: 'Y', apiCost: 30000, unitPrice: '10,000원/명',
           description: '월 100만원, 구간할인', enabled: true }
  enterprise → { monthlyPrice: 0, annualDiscount: '0%', baseStudents: 100,
                  pricePerStudent: 0, maxStudents: 0, maxAdminAccounts: 10,
                  apiIntegration: 'Y', apiCost: 0, unitPrice: '협의',
                  description: '제휴 협의', enabled: true }
  affiliate → { monthlyPrice: 0, annualDiscount: '0%', baseStudents: 0,
                 pricePerStudent: 0, maxStudents: 0, maxAdminAccounts: 0,
                 apiIntegration: 'N', apiCost: 0, unitPrice: '협의',
                 description: '제휴 협의', enabled: true }
```

**플랜명 변경:** 기존 Starter → DB의 `lite` 코드와 일치시키기 위해:
- 시드의 `name`을 `Lite`로 유지 (이미 DB 기준)
- 백엔드 `PLAN_TO_CODE` 매핑에 `Lite: 'lite'` 추가 (기존 `Starter: 'lite'` 유지 or 제거 — 하위호환)

### 3.2 Step 2 — `/admin/system` 코드관리에 플랜 컬럼 추가

**파일:** `apps/admin/frontend/src/app/(admin)/admin/system/page.tsx`

기관/계약 탭의 `PLAN_TYPE` 그룹에 상세 컬럼을 정의합니다.

```
기관/계약 탭 → PLAN_TYPE 그룹:
  컬럼: 플랜명(name) | 코드(code, readOnly) | 월비용(extraData.monthlyPrice) |
        기본학생(extraData.baseStudents) | 추가단가(extraData.pricePerStudent) |
        사용여부(extraData.enabled, select)
```

이렇게 하면 `/admin/system`에서도 플랜의 기본 정보를 관리할 수 있습니다.

### 3.3 Step 3 — `/admin/plans` 페이지 DB 연동

**파일:** `apps/admin/frontend/src/app/(admin)/admin/plans/page.tsx`

현재 `DEFAULT_PLANS` 상수 기반 → `getCodesByGroup('PLAN_TYPE')` API 호출로 전환합니다.

주요 변경:
1. **데이터 로드**: `useEffect`에서 `getCodesByGroup('PLAN_TYPE')` 호출
2. **타입 변환**: `CodeItemResponse` → 기존 UI 호환 형태로 매핑 (`extraData`에서 필드 추출)
3. **저장**: 개별 ✓ 버튼 → `saveCodeGroupItems('PLAN_TYPE', items)` 일괄 저장으로 변경
4. **삭제**: `deleteCodeItem('PLAN_TYPE', itemId)` 연동
5. **추가**: 새 행 추가 시 `_isNew: true` 플래그, 저장 시 서버에 생성

**UI 구조 유지:**
- 기존 2행 Fragment 테이블 구조 그대로
- 메인 행: 플랜명 | 월비용 | 연납할인 | 사용여부 | 관리
- 상세 행: 기본학생수 | 추가단가 | 최대학생수 | 최대관리계정수 | API연동 | API비용

**저장 방식 변경:**
- 현재: 개별 행 ✓ 버튼 (미구현)
- 변경: 상단 "전체 저장" 버튼 (code-group-table 패턴과 동일)
- 또는: 개별 저장 + 전체 저장 병행 (기획 확인 필요)

### 3.4 Step 4 — 기관 등록/수정 페이지 플랜 연동

**파일:**
- `apps/admin/frontend/src/app/(admin)/admin/institute/register/page.tsx`
- `apps/admin/frontend/src/app/(admin)/admin/institute/[id]/edit/page.tsx`

주요 변경:
1. `DEFAULT_PLANS` import 제거
2. `getCodesByGroup('PLAN_TYPE')` 로 플랜 목록 로드
3. 플랜 선택 시 `extraData`에서 기본값 자동 채움:
   ```
   플랜 선택 → extraData에서 monthlyPrice, annualDiscount, baseStudents,
              pricePerStudent, maxAdminAccounts, apiIntegration 읽어서 폼에 설정
   ```
4. 플랜 드롭다운의 `value`를 `plan.code` (DB 코드) 기반으로 변경

### 3.5 Step 5 — 백엔드 매핑 정리

**파일:** `packages/core/src/institution/institution.service.ts`

- `PLAN_TO_CODE` / `CODE_TO_PLAN` 하드코딩 매핑 제거
- 프론트에서 직접 `planCode` (DB 코드)를 전달하는 구조로 변경
- 기관 응답의 `plan` 필드도 코드 기반으로 통일하거나, 필요시 `CodeItem.name`을 조회하여 표시명 반환

### 3.6 Step 6 — `DEFAULT_PLANS` 상수 제거

**파일:** `apps/admin/frontend/src/lib/constants.ts`

모든 참조 제거 확인 후 `Plan` 인터페이스와 `DEFAULT_PLANS` 상수를 삭제합니다.

**참조 파일 목록 (현재):**
- `plans/page.tsx` — Step 3에서 제거
- `institute/register/page.tsx` — Step 4에서 제거
- `institute/[id]/edit/page.tsx` — Step 4에서 제거
- `institute/page.tsx` — 확인 필요
- `billing/page.tsx` — 확인 필요
- `pricing-section.tsx` — 확인 필요 (랜딩 페이지)
- `studio/*` 3개 파일 — 확인 필요

> ⚠️ 모든 참조가 DB 연동으로 전환되기 전까지는 `DEFAULT_PLANS`를 유지하되, 전환 완료 후 삭제합니다.

---

## 4. 파일 변경 요약

| 단계 | 파일 | 작업 |
|------|------|------|
| 1 | `prisma/seed.ts` | `PLAN_TYPE` extraData 확장 |
| 2 | `system/page.tsx` | PLAN_TYPE 컬럼 정의 추가 |
| 3 | `plans/page.tsx` | DEFAULT_PLANS → API 연동, 저장/삭제 구현 |
| 4 | `institute/register/page.tsx` | DEFAULT_PLANS → API 연동 |
| 4 | `institute/[id]/edit/page.tsx` | DEFAULT_PLANS → API 연동 |
| 5 | `institution.service.ts` | PLAN_TO_CODE/CODE_TO_PLAN 매핑 정리 |
| 6 | `constants.ts` | DEFAULT_PLANS, Plan 인터페이스 제거 |

---

## 5. 기획 확인 필요 사항

| # | 항목 | 설명 |
|---|------|------|
| 1 | **플랜명 통일** | 현재 프론트 `Starter` vs DB `Lite` — 어느 쪽으로 통일할지 |
| 2 | **저장 방식** | 개별 행 저장(✓) vs 전체 저장 버튼 — 문서에는 ✓이 있으나 code-group-table은 전체 저장 패턴 |
| 3 | **unitPrice 필드** | UI에서 편집 가능으로 할지, 자동 계산으로 할지 (`admin-plan-20260316.md` Known Issue #3) |
| 4 | **제휴 플랜** | DB에 `affiliate` 항목 존재, DEFAULT_PLANS에는 없음 — 포함 여부 |
| 5 | **billing/studio 참조** | `DEFAULT_PLANS`를 참조하는 다른 페이지(billing, studio, pricing-section)도 동시에 전환할지 |

---

## 6. 의존성 / 빌드 순서

```
prisma/seed.ts (시드 실행)
  → packages/types (타입 변경 없음, CodeItemResponse 재사용)
  → packages/core (institution.service.ts 매핑 정리)
  → apps/admin/backend (변경 없음, 기존 code API 재사용)
  → apps/admin/frontend (plans, institute, system 페이지 수정)
```

기존 코드 관리 API (`GET /codes/PLAN_TYPE`, `PUT /codes/PLAN_TYPE/items`)를 그대로 활용하므로 **백엔드 신규 API 개발이 불필요**합니다.
