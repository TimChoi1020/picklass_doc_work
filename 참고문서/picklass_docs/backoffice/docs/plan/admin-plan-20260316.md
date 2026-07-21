# 플랜관리 페이지 — admin-plan-20260316

**경로:** `src/app/(admin)/admin/plans/page.tsx`
**작성일:** 2026-03-16

---

## 1. 사용자 흐름 (User Flow)

```
/admin/plans 진입
  └── 플랜 목록 표시 (DEFAULT_PLANS 기반, 초기 3개)
        ├── 플랜 행 편집 (플랜명/월비용/연납할인/사용여부)
        │     └── ✓ 버튼 클릭 → alert('플랜이 저장되었습니다.') [임시]
        ├── 플랜 상세 행 편집 (기본학생수/추가학생단가/최대학생수/최대관리계정수/API연동/API연동비용)
        ├── ↑/↓ 버튼 → 순서 변경 (moveUp/moveDown)
        ├── - 버튼 → 삭제 확인 모달 (FormModal)
        │     ├── 취소 → 모달 닫기
        │     └── 삭제 → 해당 행 제거
        └── + 버튼 → 빈 플랜 행 추가
```

---

## 2. IA 구조 (Information Architecture)

```
플랜관리 (/admin/plans)
└── 카드 (테이블)
    └── 플랜 행 (Fragment, 플랜당 2행)
        ├── 메인 행: 플랜명 | 월비용 | 연납할인 | 사용여부 | 관리(✓/−/↑/↓)
        └── 상세 행(bg-gray-50): 3열 그리드
            ├── 학생기본제공
            ├── 추가학생당단가
            ├── 최대학생수
            ├── 최대관리계정수
            ├── API연동 (Y/N)
            └── API연동비용
```

**현재 DEFAULT_PLANS 데이터:**

| 플랜명 | 월비용 | 연납할인 | 기본학생 | 추가단가 | 최대학생 | 관리계정 | API |
|--------|--------|----------|----------|----------|----------|----------|-----|
| Starter | 100,000 | 5% | 10 | 5,000 | 50 | 2 | N |
| Pro | 1,000,000 | 10% | 50 | 10,000 | 500 | 5 | Y |
| Enterprise | 0(협의) | 0% | 100 | 0(협의) | 999999 | 10 | Y |

---

## 3. 정책 (Policy)

- 플랜이 1개만 남은 경우 삭제 버튼(`-`) 미노출
- 삭제 시 FormModal로 확인 후 처리 (`confirm()` 직접 사용 금지)
- `annualDiscount` 선택지: 0%~100% (5% 단위, 21개) — `Array.from({ length: 21 }, (_, i) => \`${i * 5}%\`)`
- `apiIntegration` 값: `'Y'` / `'N'` (select value 기준)
- `enabled` 값: `'사용'` / `'사용안함'` (select option 기준)
- `unitPrice` 필드는 `Plan` 인터페이스에 존재하나 UI에서 편집 불가 (표시 전용 혹은 미구현)
- 페이지 새로고침 시 `DEFAULT_PLANS` 상수로 초기화됨 (상태 미저장)
- 저장 버튼(✓)은 현재 `alert()`만 표시 — 실제 API 연동 필요 (tech debt)

---

## 4. 개발자 작업 (Developer Tasks)

- [ ] 저장 버튼 API 연동: `PUT /api/plans/:id` or `PATCH /api/admin/plans`
- [ ] 플랜 목록 API 연동: 초기 데이터를 `DEFAULT_PLANS` 상수 대신 `GET /api/plans`로 로드
- [ ] 삭제 API 연동: `DELETE /api/plans/:id`
- [ ] 추가 API 연동: `POST /api/plans`
- [ ] 순서 변경 API 연동: `PATCH /api/plans/reorder`
- [ ] `unitPrice` 필드 처리 결정: UI 편집 추가 또는 자동 계산으로 전환
- [ ] `apiIntegration` 데이터 정합성 확인 (아래 Known Issues 참조)

---

## 5. 코딩 룰 (Coding Rules)

### 패턴: WithId + toRows

편집 가능한 목록은 반드시 `_uid` 기반 식별자를 사용한다.

```typescript
type WithId<T> = T & { _uid: string };

const toRows = <T,>(items: readonly T[]): WithId<T>[] =>
  items.map((item) => ({ ...item, _uid: crypto.randomUUID() }));

const [plans, setPlans] = useState<WithId<Plan>[]>(() => toRows(DEFAULT_PLANS));
```

### 패턴: Fragment key (2행 이상 반복)

플랜당 2개 `<tr>`를 렌더링하므로 `<Fragment key={...}>`를 사용한다. `<>` 사용 금지.

```typescript
import { Fragment, useState } from 'react';

{plans.map((plan, index) => (
  <Fragment key={plan._uid}>
    <tr>...</tr>
    <tr className="bg-gray-50">...</tr>
  </Fragment>
))}
```

### 패턴: 삭제 확인 모달

`confirm()` 직접 사용 금지. `FormModal` + 단일 `deleteConfirm` 상태 사용.

```typescript
const [deleteConfirm, setDeleteConfirm] = useState<{ uid: string; label: string } | null>(null);

// 삭제 버튼
onClick={() => setDeleteConfirm({ uid: plan._uid, label: plan.name })}

// 확인 처리
const handleDeleteConfirm = () => {
  if (!deleteConfirm) return;
  setPlans((p) => p.filter((r) => r._uid !== deleteConfirm.uid));
  setDeleteConfirm(null);
};

// FormModal
<FormModal open={!!deleteConfirm} onOpenChange={(open) => { if (!open) setDeleteConfirm(null); }}
  title="삭제 확인" submitLabel="삭제" cancelLabel="취소" onSubmit={handleDeleteConfirm}>
  <p><strong>{deleteConfirm?.label}</strong> 플랜을 삭제하시겠습니까?</p>
</FormModal>
```

### 패턴: moveUp / moveDown

`src/lib/utils.ts`에서 import. 페이지 내 인라인 정의 금지.

```typescript
import { moveUp, moveDown } from '@/lib/utils';

<button onClick={() => moveUp(index, plans, setPlans)} disabled={index === 0}>↑</button>
<button onClick={() => moveDown(index, plans, setPlans)} disabled={index === plans.length - 1}>↓</button>
```

### 패턴: annualDiscount 선택지 생성

하드코딩 금지. `Array.from`으로 동적 생성.

```typescript
{Array.from({ length: 21 }, (_, i) => `${i * 5}%`).map((d) => (
  <option key={d} value={d}>{d}</option>
))}
```

### 스타일 규칙

- 인라인 스타일 사용 금지, Tailwind 클래스만 사용
- `key={index}` 사용 금지, `key={item._uid}` 사용
- 공통 클래스 변수 상단에 선언:
  ```typescript
  const cellInput = 'w-full p-1 border border-gray-300 rounded-sm box-border text-sm';
  const moveBtn = 'size-[30px] rounded-full flex items-center justify-center bg-gray-400 text-white text-sm border-none cursor-pointer disabled:cursor-not-allowed disabled:opacity-50';
  ```

---

## 6. Known Issues

| # | 이슈 | 심각도 | 상태 |
|---|------|--------|------|
| 1 | 저장 버튼(✓)이 `alert()` 호출만 함 — 실제 저장 없음 | High | 미해결 |
| 2 | `apiIntegration` 데이터 불일치: `DEFAULT_PLANS`의 Starter는 `'N'`, Pro/Enterprise는 `'Y'`이나 기존 docs에 `'미연동'/'연동'` 문자열로 기재됨 | Medium | 코드 기준 `'N'/'Y'`가 정답 |
| 3 | `unitPrice` 필드가 `Plan` 인터페이스에 존재하나 UI 편집 불가 | Low | 기획 결정 필요 |
| 4 | 페이지 새로고침 시 편집 내용 초기화 (DEFAULT_PLANS 상수로 리셋) | High | API 연동 후 해결 |

---

## 7. 의존성 (Dependencies)

| 항목 | 경로 | 용도 |
|------|------|------|
| `DEFAULT_PLANS`, `Plan` | `@/lib/constants` | 초기 플랜 데이터 및 타입 |
| `FormModal` | `@/components/common/form-modal` | 삭제 확인 모달 |
| `moveUp`, `moveDown` | `@/lib/utils` | 행 순서 변경 |
| `Fragment` | `react` | 2행 반복 렌더링에 key 부여 |

---

## 8. DB/API 계약 (DB/API Contract)

### 현재 상태
- 플랜 데이터는 `src/lib/constants.ts`의 `DEFAULT_PLANS` 상수에 하드코딩
- 백엔드 API 미구현

### Plan 인터페이스 (constants.ts 기준)

```typescript
interface Plan {
  name: string;           // 플랜명
  monthlyPrice: number;   // 월 비용 (원)
  annualDiscount: string; // 연납 할인율 (예: '10%')
  baseStudents: number;   // 기본 제공 학생 수
  pricePerStudent: number;// 추가 학생당 단가 (원)
  maxStudents: number;    // 최대 학생 수 (999999 = 무제한)
  maxAdminAccounts: number;// 최대 관리 계정 수
  apiIntegration: string; // API 연동 여부: 'Y' | 'N'
  apiCost: number;        // API 연동 비용 (원/월)
  enabled: string;        // 사용 여부: '사용' | '사용안함'
  unitPrice: string;      // 표시용 단가 문자열 (예: '10,000원/명', '협의')
}
```

### 향후 API 설계 (예정)

```
GET    /api/plans          → Plan[] 목록 조회
POST   /api/plans          → 플랜 추가
PUT    /api/plans/:id      → 플랜 수정
DELETE /api/plans/:id      → 플랜 삭제
PATCH  /api/plans/reorder  → 순서 변경 { ids: string[] }
```

### DB 테이블 설계 (예정)

```sql
CREATE TABLE plans (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          VARCHAR(100) NOT NULL,
  monthly_price INTEGER NOT NULL DEFAULT 0,
  annual_discount VARCHAR(10) NOT NULL DEFAULT '0%',
  base_students INTEGER NOT NULL DEFAULT 0,
  price_per_student INTEGER NOT NULL DEFAULT 0,
  max_students  INTEGER NOT NULL DEFAULT 0,
  max_admin_accounts INTEGER NOT NULL DEFAULT 0,
  api_integration CHAR(1) NOT NULL DEFAULT 'N',
  api_cost      INTEGER NOT NULL DEFAULT 0,
  enabled       BOOLEAN NOT NULL DEFAULT TRUE,
  unit_price    VARCHAR(50),
  sort_order    INTEGER NOT NULL DEFAULT 0,
  created_at    TIMESTAMP DEFAULT NOW(),
  updated_at    TIMESTAMP DEFAULT NOW()
);
```
