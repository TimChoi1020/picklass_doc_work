# AI 모듈 관리 - 관리자 페이지 명세서 (모듈등록 페이지 기준)

> 최초 작성: 2026-03-24
> 최종 수정: 2026-04-02
> 문서 대상: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

## 0. 기존 문서 대비 변경 요약

- 탭 구조 변경
  - 기존: 기본 정보 / 수업 설정 / 문항설계 / 성과KPI설정 / 레벨별 AI설정
  - 현재: 기본 정보 / 커리큘럼 배치 설정 / 문항설계(QuestionData) / 성과KPI설정 / AI 동작 설정
- 기본 정보 탭 단순화
  - 현재는 `skill`, `moduleCode`, `moduleName` 중심 구성
- 수업 설정 정책 변경
  - 레벨 다중 선택(`selectedLevels`) UI 제거
  - `roles`, `passageExposure`, `cognitiveLevel`, `suitableLevelMin/Max`, `estimatedMinutesMin/Max`, `prerequisites`, `incompatibleWith` 중심
- 문항설계 탭 변경
  - 문항 배열 직접 편집 중심에서 학습 설계 선택형 필드 중심으로 전환
- 성과 KPI 탭 컬럼 변경
  - 컬럼 순서: 코드 / 목표 / 측정 항목 / 측정 유닛 / 측정 방법 / 관리
- AI 동작 설정 탭 신설
  - 현재 노출 필드: `pedagogyInstruction`, `purpose`
  - `stage1Greeting`, `stage7CompleteMsg` 입력 항목은 UI에서 제거됨
- 목업 KPI 데이터 변경
  - 기본 선택/목업 항목을 `Fluency`, `Prediction` 중심으로 조정

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 진입

1. 관리자 사이드바의 `수업모듈` 메뉴에서 목록 페이지(`/admin/ai-modules`)로 이동
2. 목록 페이지에서
   - `+ 모듈 등록` 클릭 시 `/admin/ai-modules/register` 이동
   - `편집` 클릭 시 `/admin/ai-modules/register?id={id}` 이동

### 1-2. 등록/수정 페이지 단계

1. 기본 정보 입력
   - `skill`, `code(moduleCode)`, `name(moduleName)` 입력
2. 커리큘럼 배치 설정 입력
   - 역할/노출/인지수준/권장레벨/예상시간/선행모듈/충돌모듈 입력
3. 문항설계(QuestionData) 설정
   - `answerType`, `scoringMode`, `passageExposureMode`, `questionCount`, `feedbackStyle`, `feedbackType`
4. 성과KPI설정
   - 선택된 KPI 확인
   - 모달에서 KPI 추가 또는 삭제
5. AI 동작 설정
   - `pedagogyInstruction`(멀티라인), `purpose` 입력
6. 제출
   - 현재는 저장 API 호출 없이 `alert` 후 목록으로 라우팅

### 1-3. 수정 모드(`?id=`) 동작

- 현재는 실제 서버 조회가 아니라 페이지 내부 `modules` 목업 배열에서 일부 필드만 주입
- 주입 필드: `skill`, `moduleName`, `moduleCode`, `lesson`, `textOpen`, `priority`

---

## 2. IA 구조 정리 및 기능 정의 (IA)

### 2-1. 탭 IA

| 탭 | 목적 | 주요 입력 | 출력/상태 반영 | 조건 |
|---|---|---|---|---|
| 기본 정보 | 모듈 식별 정보 입력 | `skill`, `moduleCode`, `moduleName` | `formData` 최상위 필드 업데이트 | `moduleCode`는 입력 즉시 대문자 변환 |
| 커리큘럼 배치 설정 | 수업 배치/난이도/의존성 설정 | `roles[]`, `passageExposure`, `cognitiveLevel`, `suitableLevelMin/Max`, `estimatedMinutesMin/Max`, `prerequisites[]`, `incompatibleWith[]` | `formData`에 반영 | `prerequisites`, `incompatibleWith`는 콤마 입력을 배열로 변환/대문자화 |
| 문항설계(QuestionData) | 문항/평가 메타 정의 | `answerType`, `scoringMode`, `passageExposureMode`, `questionCount`, `feedbackStyle`, `feedbackType` | `formData` 반영 | 선택형 라디오 카드 UI |
| 성과KPI설정 | KPI 코드 선택/제거 | KPI 선택(모달), 삭제 버튼 | `contentConfig.selectedKpiCodes` 반영 | API 실패 시 mockup 데이터 fallback |
| AI 동작 설정 | AI 응답 가이드/목적 입력 | `pedagogyInstruction`, `purpose` | `formData` 반영 | `pedagogyInstruction`은 multiline placeholder 제공 |

### 2-2. KPI 모달 IA

| 기능 | 목적 | 입력 | 출력 |
|---|---|---|---|
| 선택된 KPI 리스트 | 현재 선택 코드 확인 | `contentConfig.selectedKpiCodes` | 테이블 렌더링 |
| 신규 버튼 | 선택 가능한 KPI 표시 | `availableKpis` - 선택된 코드 제외 | FormModal 오픈 |
| 선택 | KPI 추가 | `item.code` | `selectedKpiCodes` 배열 append |
| 삭제 | KPI 제거 | `item.code` | `selectedKpiCodes` 배열 filter |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. 현재 적용 정책

- KPI 조회 정책
  - `getCodesByGroup('KPI_ITEMS')`로 조회
  - 실패/빈 결과 시 `MOCK_KPI_ITEMS`로 대체
- 중복 KPI 방지 정책
  - 이미 선택된 코드는 추가 불가
- 문자열 정규화 정책
  - `moduleCode`는 대문자 강제
  - `prerequisites`, `incompatibleWith`는 콤마 분리 후 trim + 대문자
- 삭제 정책
  - KPI 삭제는 UI 배열에서 코드 제거(클라이언트 상태 기준)

### 3-2. 정책 변경사항 (기존 대비)

- 레벨 다중선택 정책 제거
  - `selectedLevels` 기반 UX/분기 로직 삭제
- AI 동작 설정 정책 단순화
  - 기존 stage greeting/complete 메시지 입력 제거
  - 교수법 지시문 + 목적 요약 중심으로 축소
- KPI 컬럼 정책 변경
  - `자동화` 표기 대신 `측정 유닛` 사용

---

## 4. 변경된 내용 기준 추가 개발 필요 작업

1. 등록/수정 저장 API 연동
   - 현재 `handleSubmit`은 `alert` + 라우팅만 수행
   - 실제 POST/PUT 연동 필요
2. 수정 모드 데이터 로딩 API 연동
   - 현재 `modules` 목업 사용
   - `id` 기반 조회 API 필요
3. 폼 검증 추가
   - 필수값/범위/상호조건(예: min <= max) 검증 미구현
4. KPI 스키마 정리
   - 백엔드 seed는 `automation` 기반, 프론트는 `measureUnit` 사용 중
   - KPI `extraData` 표준 스키마 통일 필요
5. 불필요 상태/핸들러 정리
   - 현재 UI에서 사용하지 않는 ContentConfig 대규모 필드 및 stage 핸들러 정리 필요
6. 공통 컴포넌트 분리
   - 탭별 폼 블록이 단일 파일에 과도하게 집중됨
   - 유지보수 위해 섹션 컴포넌트 분리 권장

---

## 5. 코드 규칙 (Coding Rules)

### 5-1. 공통 유틸/컴포넌트 사용 규칙

- API 호출
  - 직접 `fetch` 대신 `fetchApi` 래퍼 사용
  - 코드 그룹 조회는 `src/lib/api/code.ts`의 `getCodesByGroup` 사용
- 모달
  - 선택/추가 UX는 `FormModal` 공통 컴포넌트 사용
- Seed/Slot 타입 입력 UI
  - Seed 계열 입력이 필요한 화면은 개별 하드코딩 대신 공통 컴포넌트(예: `SeedTypeField`)를 우선 사용
- 상수
  - 스킬/레벨 등 공통 상수는 `src/lib/constants.ts`에서 가져오기

### 5-2. 금지 패턴

- `any` 타입 사용 금지
- `@ts-ignore` 사용 금지
- 저장/조회 로직에 하드코딩 데이터 의존 금지
  - 예외: 명시적 fallback mock은 임시 정책으로만 허용
- `alert()` 직접 사용 금지
  - 토스트/공통 알림 컴포넌트로 전환 필요

### 5-3. 파일 위치 규칙

- 페이지 전용 타입
  - 페이지 파일 내부가 아니라 `types` 모듈로 분리 권장
- 공용 API DTO/응답 타입
  - `packages/types/src/index.ts`에서 단일 관리
- 공용 상수
  - `apps/admin/frontend/src/lib/constants.ts`
- API 함수
  - `apps/admin/frontend/src/lib/api/*.ts`

### 5-4. 네이밍 규칙

- 인터페이스
  - `XxxDto`, `XxxResponse`, `XxxFormData` 형식 사용
- 핸들러 함수
  - `handle + 대상 + 동작` 패턴 (예: `handleAddKpiCode`)
- Boolean 상태
  - `is/has/can` 접두사 사용 (`isKpiModalOpen`, `kpiLoading`)

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

### 6-1. 하드코딩 데이터

- 등록 페이지 내부 `modules` 배열(수정모드 목업)
- `MOCK_KPI_ITEMS` 배열
- AI 동작 설정 placeholder 프롬프트 하드코딩

### 6-2. 상태 불일치/미사용 코드

- `ContentConfig`의 stage 관련 다수 필드가 현재 UI에서 편집 불가
- stage 관련 핸들러(`handleStage1ItemChange` 등)가 현재 화면에서 실사용되지 않음
- `GreetingEditor` 및 관련 변수 상수는 현재 탭 구성에서 미사용

### 6-3. 미구현 검증/임시 처리

- 저장 시 서버 검증/중복검사 없음
- `alert()` 기반 임시 완료 처리
- `kpiLoading` 초기값이 `false`라 로딩 UX가 약함
- `questionData` 초기값은 존재하나 현재 문항설계 탭에서 직접 편집하지 않음

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 7-1. 이 페이지가 사용하는 공통 요소

- Hooks
  - `useRouter`, `useSearchParams` (Next navigation)
  - `useState`, `useEffect`, `Suspense`
- API
  - `getCodesByGroup` (`src/lib/api/code.ts`)
  - 내부적으로 `fetchApi` 사용 (`src/lib/api.ts`)
- UI 컴포넌트
  - `FormModal`
- 상수
  - `SKILLS`, `LEVEL_SYSTEM`
- 타입
  - `CodeItemResponse` (`@repo/types`)

### 7-2. 진입 경로 (Entry Points)

- 사이드바 메뉴: `/admin/ai-modules`
- 목록 페이지 버튼
  - `+ 모듈 등록` -> `/admin/ai-modules/register`
  - `편집` -> `/admin/ai-modules/register?id={id}`

### 7-3. 영향 받는 기능

- 코드관리(`KPI_ITEMS`) 데이터 변경 시 등록 페이지 KPI 선택 UX 직접 영향
- 레벨 상수(`LEVEL_SYSTEM`) 변경 시 커리큘럼 배치 설정 옵션 영향

---

## 8. DB/API 구조 (Data Contract)

### 8-1. 현재 사용 API 엔드포인트

- `GET /codes`
  - 코드 그룹 + 아이템 전체 조회
- `GET /codes/:groupCode`
  - 그룹별 코드 아이템 조회
  - 등록 페이지는 `GET /codes/KPI_ITEMS` 사용
- `PUT /codes/:groupCode/items`
  - 그룹 아이템 일괄 저장
- `DELETE /codes/:groupCode/items/:itemId`
  - 그룹 아이템 soft delete

### 8-2. 등록 페이지 관점 목표 API (미구현)

- `POST /ai-modules`
  - 신규 모듈 등록
- `GET /ai-modules/:id`
  - 수정 폼 초기 데이터 조회
- `PUT /ai-modules/:id`
  - 모듈 수정 저장

### 8-3. 관련 DB 테이블/컬럼 (Prisma)

- `CodeGroup` (`code_groups`)
  - `id`, `code`, `name`, `description`, `isActive`, timestamps
- `CodeItem` (`code_items`)
  - `id`, `groupId`, `code`, `name`, `description`, `sortOrder`, `extraData(JsonB)`, `isActive`, timestamps
- KPI는 `CodeItem`에서 `groupCode = KPI_ITEMS`로 관리

### 8-4. 인터페이스/타입 정의 전문

#### 공유 타입 (`packages/types/src/index.ts`)

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

#### 등록 페이지 폼 타입 (`register/page.tsx`)

```ts
interface ContentConfig {
  groupId: number;
  groupName: string;
  selectedKpiCodes: string[];
  stagePrepEnabled: boolean;
  contentSelection: string;
  learningType: string;
  learningQty: number;
  preHintLogic: string;
  delayTime: number;
  hintButtonShow: boolean;
  stage1Enabled: boolean;
  stage1Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage1Greeting: string;
  stage2Enabled: boolean;
  stage2Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage3Enabled: boolean;
  stage3PreIntervention: boolean;
  stage3DelayTime: number;
  stage3HintButtonShow: boolean;
  stage3Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage3ProactiveLint: string;
  stage4Enabled: boolean;
  stage4Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage5Enabled: boolean;
  stage5aEnabled: boolean;
  stage5bEnabled: boolean;
  stage5bItems: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage5CorrectMsg: string;
  stage5WrongMsg: string;
  stage5FinalWrongMsg: string;
  stage5AllItemsFeedback: string;
  stage5Feedback: string;
  stage5aCorrectItems: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage5aWrongItems: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage5aAllFeedbackItems: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  stage6Enabled: boolean;
  stage6RetryMsg: string;
  stage6Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
  retryOption: string;
  retryThreshold: number;
  stage7Enabled: boolean;
  stage7CompleteMsg: string;
  stage7Items: { uid: string; uiType: string; feedbackType: string; description: string; placeholder?: string }[];
}

interface ModuleFormData {
  skill: string;
  moduleName: string;
  moduleCode: string;
  purpose: string;
  pedagogyInstruction: string;
  answerType: string;
  questionCount: 'single' | 'multi';
  flowMode: 'orchestrator' | 'content-config';
  scoringMode: string;
  passageExposureMode: string;
  feedbackStyle: string;
  feedbackType: string;
  lesson: string[];
  textOpen: string[];
  priority: string;
  roles: string[];
  passageExposure: '' | 'before' | 'during' | 'after' | 'any';
  cognitiveLevel: '' | '1' | '2' | '3' | '4' | '5' | '6';
  suitableLevelMin: string;
  suitableLevelMax: string;
  estimatedMinutesMin: string;
  estimatedMinutesMax: string;
  prerequisites: string[];
  incompatibleWith: string[];
  questionData: Array<{
    uid: string;
    id: string;
    number: number;
    type: 'essay' | 'sentence-write' | 'audio-record' | 'option' | 'short-text';
    text: string;
    hint: string;
    answer: string;
  }>;
  contentConfig: ContentConfig;
}
```

---

## 9. 버그픽스 플랜 (2026-04-02)

### 9-1. cognitiveLevel 하드코딩 제거 → 동적 검증

**배경**: 인지 수준 데이터는 코드관리에서 자유롭게 추가/삭제될 수 있음. 현재는 1~6으로 하드코딩되어 있어 데이터 변경 시마다 코드 수정이 필요한 구조.

**현재 문제점**:
- 프론트엔드: 타입이 `'' | '1' | '2' | '3' | '4' | '5' | '6'`으로 고정
- 프론트엔드: 검증이 `['1','2','3','4','5','6'].includes()`로 고정
- 백엔드: 검증이 `level < 1 || level > 6`으로 고정

**수정 방향**: cognitiveLevel은 양의 정수(`>= 1`)인지만 검증하고, 유효 값 목록은 하드코딩하지 않음

**수정 대상 파일 및 위치**:

| 파일 | 위치 | 현재 | 수정 |
|------|------|------|------|
| `apps/admin/frontend/.../register/page.tsx` | L74 타입 정의 | `'' \| '1' \| ... \| '6'` | `string` |
| `apps/admin/frontend/.../register/page.tsx` | L1014-1015 검증 | 고정 배열 포함 검사 + 고정 에러 메시지 | `parseInt >= 1` 검증, 메시지: `'인지 수준은 1 이상의 정수여야 합니다'` |
| `packages/core/src/ai-module/ai-module.service.ts` | L152-157 서버 검증 (update) | `level < 1 \|\| level > 6` | `level < 1 \|\| !Number.isInteger(level)` |
| `packages/core/src/ai-module/ai-module.service.ts` | create 메서드 내 동일 검증 | 동일 패턴 | 동일 수정 |

### 9-2. prerequisites / incompatibleWith 콤마 입력 불가 수정

**배경**: 현재 `onChange` 핸들러가 입력 즉시 `split(',') → filter(Boolean)`을 실행하여 trailing comma가 즉시 제거됨. 사용자가 "PRD,"를 입력하면 → `["PRD", ""]` → `filter(Boolean)` → `["PRD"]` → `join(', ')` → "PRD"로 되돌아가 콤마가 사라짐

**수정 대상 파일 및 위치**:

| 파일 | 위치 | 수정 내용 |
|------|------|-----------|
| `apps/admin/frontend/.../register/page.tsx` | L1443-1449 (prerequisites onChange) | 입력 중에는 raw string을 유지하고, blur/submit 시에만 배열 변환 |
| `apps/admin/frontend/.../register/page.tsx` | L1464-1470 (incompatibleWith onChange) | 동일 수정 |

**수정 방향**: `formData`에 `prerequisitesRaw: string`과 `incompatibleWithRaw: string` 필드를 추가하거나, `onChange` 시 raw 문자열을 별도 state로 관리하여 입력 중에는 `split/filter`를 실행하지 않음. `onBlur` 또는 `handleSubmit` 시점에 배열로 변환.

---

## 부록: 정책 정합성 체크 포인트

- 프론트 KPI 컬럼(`measureUnit`)과 시드 KPI 컬럼(`automation`) 불일치 해소 필요
- 등록/수정 저장 API 설계 시 `ContentConfig`를 실제로 어디까지 저장할지 범위 확정 필요
- AI 동작 설정 탭 축소에 따라 `ContentConfig` stage 필드의 유지/삭제 여부 아키텍처 결정 필요
