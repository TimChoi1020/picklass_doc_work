# admin-ai-modules-code-management (수업모듈 코드관리)

> 작성일: 2026-03-30
> 대상 페이지: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/code-management/page.tsx`
> 비교 기준 문서:
> - `docs/ai-modules/admin-ai-modules-20260324.md` (모듈 등록 페이지)
> - `docs/ai-modules/admin-ai-modules-20260316.md` (모듈 목록 페이지)

## 0. 기존 문서 대비 변경 요약

- 기존 문서는 목록/등록 페이지 중심이며, `code-management` 전용 명세가 없었음.
- `code-management`는 코드그룹 CRUD 운영 페이지로 역할이 분리됨.
- 탭 기반 코드 관리(총 6개 탭):
  - Passage Exposure Mode
  - Scoring Mode
  - Answer Type
  - feedbackStyle
  - feedbackType
  - 성과KPI
- KPI 컬럼이 `코드/목표/측정 항목/측정 유닛/측정 방법` 순서로 정리됨.
- 공통 편집 컴포넌트 `CodeGroupTable` 재사용 구조가 핵심.

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 진입

1. 관리자 사이드바에서 `수업모듈 코드관리` 클릭
2. `/admin/ai-modules/code-management` 페이지 진입

### 1-2. 페이지 로딩

1. 페이지 마운트 시 `getAllCodes()` 호출
2. 로딩 중: "데이터를 불러오는 중..." 표시
3. 성공 시: 탭 + 그룹별 테이블 렌더링
4. 실패 시: 에러 메시지 + "다시 시도" 버튼 노출

### 1-3. 탭별 코드 관리

1. 탭 선택
2. 그룹 테이블에서 항목 조회/편집
3. `+` 버튼으로 신규 행 추가
4. `↑/↓`로 순서 이동
5. `-`로 삭제(확인 모달)
6. `전체 저장` 클릭 시 서버 저장 후 목록 재조회

---

## 2. IA 구조 정리 및 기능 정의 (IA)

### 2-1. 화면 IA

| 영역 | 목적 | 입력 | 출력 | 조건 |
|---|---|---|---|---|
| 헤더 (`수업모듈 관리`/`수업모듈 코드관리`) | 페이지 식별 | 없음 | 제목 텍스트 | 항상 노출 |
| TabsList | 코드그룹 카테고리 전환 | 탭 클릭 | 해당 탭 콘텐츠 노출 | `buildTabConfig()` 기반 |
| TabsContent | 그룹 단위 편집 | 행 단위 입력 | 로컬 상태 반영 | 탭별 1개 그룹 렌더링 |
| CodeGroupTable | 코드 항목 CRUD | text/number/textarea/select | 저장 DTO 생성 및 API 호출 | `groupCode`, `columns`로 동작 정의 |

### 2-2. 탭/그룹 정의

| 탭 key | 탭 label | groupCode | 비고 |
|---|---|---|---|
| passage-exposure-mode | Passage Exposure Mode | PASSAGE_EXPOSURE_MODE | 코드/설명 |
| scoring-mode | Scoring Mode | SCORING_MODE | 코드/설명 |
| answer-type | Answer Type | ANSWER_TYPE | 코드/설명 |
| feedback-style | feedbackStyle | FEEDBACK_STYLE | 코드/설명 |
| feedback-type | feedbackType | FEEDBACK_TYPE | 코드/설명 |
| kpi | 성과KPI | KPI_CATEGORY | KPI 전용 컬럼 |

### 2-3. KPI 컬럼 정의

| key | label | type | 의미 |
|---|---|---|---|
| code | 코드 | text | KPI 코드(기존 항목 readOnly) |
| extraData.goal | 목표 | text | KPI 목적 |
| name | 측정 항목 | text | KPI 항목명 |
| extraData.measureUnit | 측정 유닛 | text | 단위(예: %, WPM) |
| extraData.measureMethod | 측정 방법 | text | 측정 방식 |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. 현재 적용 정책

- 전체 코드 조회 정책
  - 페이지 진입 시 `GET /codes` 1회 호출
- 저장 정책
  - 그룹 단위 `PUT /codes/:groupCode/items`
  - 정렬 순서대로 `sortOrder` 재부여
- 삭제 정책
  - 서버 항목은 `DELETE /codes/:groupCode/items/:itemId`
  - 신규 로컬 항목은 로컬에서만 제거
- 코드 필드 보호 정책
  - 기존 항목의 `code`는 수정 불가(`readOnlyOnExisting`)

### 3-2. 정책 변경사항 (비교 기준 대비)

- 등록 페이지에서 분산 관리하던 일부 코드 성격 데이터를 전용 코드관리 페이지로 이관
- KPI 필드 네이밍이 `자동화` 중심에서 `측정 유닛` 중심으로 변경됨
- 탭 기반 그룹 편집으로 코드 그룹 운영 표준화

---

## 4. 변경된 내용 기준 추가 개발 필요 작업

1. 그룹 코드 정합성 정리
   - `code-management` KPI는 `KPI_CATEGORY`, 등록 페이지 KPI는 `KPI_ITEMS` 사용 중
   - 단일 정책으로 통일 필요
2. 조회 최적화
   - 현재 `getAllCodes()`로 전 그룹 로드
   - 탭 진입 시 그룹별 lazy load 고려
3. 저장 UX 개선
   - 저장/삭제 성공 토스트, 실패 메시지 표준화
4. 권한 정책 연결
   - 코드 관리 수정/삭제 권한(role) 체크 미구현
5. 검증 강화
   - 코드 중복/빈값/형식 검증 프론트 선검증 추가

---

## 5. 코드 규칙 (Coding Rules)

### 5-1. 공통 유틸/컴포넌트 사용 규칙

- API 호출
  - `fetchApi` 래퍼를 사용하는 `src/lib/api/code.ts` 함수 사용 (`getAllCodes`, `saveCodeGroupItems`, `deleteCodeItem`)
- 공통 편집 컴포넌트
  - 그룹 테이블 편집은 `CodeGroupTable` 재사용
- 모달
  - 삭제 확인은 `FormModal` 사용
- 탭
  - `@/components/ui/tabs` 사용
- Seed 계열 입력
  - Seed/Slot 선택 UI는 향후 `SeedTypeField` 공통 컴포넌트 우선 사용

### 5-2. 금지 패턴

- `any`, `@ts-ignore` 사용 금지
- 페이지 내 임의 하드코딩 API 경로 사용 금지
- `alert()` 직접 사용 금지 (토스트/공통 알림으로 전환 권장)
- 동일 도메인 타입의 페이지 중복 선언 지양

### 5-3. 파일 위치 규칙

- API 함수: `apps/admin/frontend/src/lib/api/*.ts`
- 공통 상수: `apps/admin/frontend/src/lib/constants.ts`
- 공통 타입: `packages/types/src/index.ts`
- 공통 테이블 컴포넌트: `apps/admin/frontend/src/app/(admin)/admin/system/_components/code-group-table.tsx`
- 페이지 문서: `docs/ai-modules/`

### 5-4. 네이밍 규칙

- 페이지 컴포넌트: `XxxPage` (`ModuleCodeManagementPage`)
- 설정 타입: `XxxConfig` (`TabConfig`, `GroupConfig`)
- 이벤트 함수: `handle + 대상 + 동작` (`handleItemsChange`)

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

### 6-1. 하드코딩 데이터

- KPI 컬럼 placeholder 문구 하드코딩
- 탭/그룹 매핑(`buildTabConfig`)이 페이지 내부 하드코딩

### 6-2. 상태/구조 이슈

- 전체 코드 데이터(`getAllCodes`)를 한 번에 들고 있어 탭 스케일 확장 시 비효율 가능
- `newItemDefaults` 타입 단언(`as Partial<...>`) 사용으로 정적 안전성 낮음

### 6-3. 임시 처리/미구현

- `CodeGroupTable` 내부 에러 처리에 `alert()` 사용
- 저장 전 중복 코드 검증, 필수 필드 검증이 약함
- KPI `measureUnit`와 시드 데이터(`automation`) 스키마 불일치 가능성

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 7-1. 이 페이지가 사용하는 공통 요소

- Hooks
  - `useState`, `useEffect`, `useCallback`, `useMemo`
- API
  - `getAllCodes`
- 공통 컴포넌트
  - `Tabs`, `TabsList`, `TabsTrigger`, `TabsContent`
  - `CodeGroupTable`
- 공통 타입
  - `CodeGroupResponse`, `ColumnDef`, `LocalItem`

### 7-2. 진입점 (Entry Points)

- 사이드바 메뉴: `/admin/ai-modules/code-management`
- 메뉴 정의 위치: `src/lib/constants.ts`의 `ADMIN_MENU`

### 7-3. 영향 받는 기능

- 등록 페이지 KPI 선택 UX (코드그룹 데이터 변경 영향)
- 시스템 코드관리 공통 컴포넌트(`CodeGroupTable`)의 변경이 본 페이지와 system 페이지에 동시 영향

---

## 8. DB/API 구조 (Data Contract)

### 8-1. 현재 사용 API 엔드포인트

- `GET /codes`
  - 응답: `CodeGroupResponse[]`
- `PUT /codes/:groupCode/items`
  - 요청: `SaveCodeGroupItemsDto`
  - 응답: `CodeItemResponse[]`
- `DELETE /codes/:groupCode/items/:itemId`
  - 응답: `{ success: true }`

### 8-2. 관련 DB 테이블/컬럼 (Prisma)

- `code_groups`
  - `id`, `code`, `name`, `description`, `is_active`, `created_at`, `updated_at`
- `code_items`
  - `id`, `group_id`, `code`, `name`, `description`, `sort_order`, `extra_data(jsonb)`, `is_active`, `created_at`, `updated_at`

### 8-3. 인터페이스/타입 정의 전문

#### 페이지/테이블 핵심 타입

```ts
interface GroupConfig {
  groupCode: string;
  title: string;
  columns: ColumnDef[];
  newItemDefaults?: Record<string, unknown>;
}

interface TabConfig {
  key: string;
  label: string;
  groups: GroupConfig[];
}

export interface ColumnDef {
  key: string;
  label: string;
  type: 'text' | 'number' | 'textarea' | 'select';
  width?: string;
  options?: { value: string; label: string }[];
  readOnlyOnExisting?: boolean;
  placeholder?: string;
}

export type LocalItem = CodeItemResponse & { _uid: string; _isNew?: boolean };
```

#### 공유 API 타입

```ts
export interface CodeItemResponse {
  id: number;
  code: string;
  name: string;
  description: string | null;
  sortOrder: number;
  extraData: Record<string, unknown> | null;
}

export interface CodeGroupResponse {
  code: string;
  name: string;
  description: string | null;
  items: CodeItemResponse[];
}

export interface UpsertCodeItemDto {
  id?: number;
  code: string;
  name: string;
  description?: string;
  sortOrder: number;
  extraData?: Record<string, unknown>;
}

export interface SaveCodeGroupItemsDto {
  items: UpsertCodeItemDto[];
}
```

---

## 부록: 검토 포인트

- KPI를 `KPI_CATEGORY`로 관리할지 `KPI_ITEMS`로 관리할지 정책 확정 필요
- `CodeGroupTable`의 alert 기반 에러 처리 공통 토스트로 치환 필요
- 대규모 그룹 추가 전 탭별 지연 로딩 전략 확정 필요
