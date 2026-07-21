# AI 모듈 관리 - 개발 구현 플랜

> 작성일: 2026-03-31
> 범위: 테이블 설계 + CRUD 전체 구현 (목록/등록/수정/삭제)
> 참조 문서:
> - `docs/ai-modules/ai-modules.md` (초기 기획)
> - `docs/ai-modules/admin-ai-modules-20260316.md` (목록 페이지 명세)
> - `docs/ai-modules/admin-ai-modules-20260324.md` (등록/수정 페이지 명세)
> - `docs/ai-modules/admin-ai-modules-code-management-20260330.md` (코드관리 명세)
> - `tutoring.picklass.com/docs/modules-lessonId-20260325.md` (학습 실행 화면 참조)

---

## 0. 현황 분석 요약

### 현재 상태
- **DB**: `ai_modules` 테이블 없음
- **Backend**: ai-modules 모듈/컨트롤러/서비스 없음
- **Frontend**: 3개 페이지가 **하드코딩 데이터**로 동작 중
  - `apps/admin/frontend/src/app/(admin)/admin/ai-modules/page.tsx` (목록)
  - `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx` (등록/수정)
  - `apps/admin/frontend/src/app/(admin)/admin/ai-modules/code-management/page.tsx` (코드관리)
- **타입**: `AIModule`, `ModuleFormData`, `ContentConfig` 등이 각 페이지에 중복 선언

### 기존 패턴 (다른 도메인 참조)
- Backend Controller → Core Service → Prisma 3계층 구조
- 공유 타입은 `packages/types/src/index.ts`에 집중
- soft delete 패턴 (`deletedAt` nullable)
- UUID PK (`gen_random_uuid()`)
- snake_case DB 매핑 + camelCase 코드

### tutoring.picklass.com 참조 사항
- `ContentConfig`가 학습 실행 화면(Orchestrator)의 핵심 설정 데이터
- 모듈별 어댑터 패턴: `moduleCode`로 어댑터를 선택하여 `fetchModuleData` + `buildContentConfig` 수행
- `questionData`, `contentConfig`의 stage 필드들이 실제 학습 흐름 제어에 사용됨
- 따라서 DB 저장 시 `contentConfig`는 **JSONB 통째 저장**이 적합 (어댑터가 런타임에 해석)

---

## 1. 테이블 설계 (Prisma Schema)

### 1-1. `ai_modules` 테이블 (메인)

```prisma
model AiModule {
  id                    String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  
  // ── 기본 정보 (탭1) ──
  skill                 String    @db.VarChar(50)          // 'vocabulary' | 'reading' | 'speaking'
  code                  String    @unique @db.VarChar(10)  // 모듈 코드 (WRD, PRD 등)
  name                  String    @db.VarChar(200)         // 모듈명 (Word Reading Deck 등)
  
  // ── 커리큘럼 배치 설정 (탭2) ──
  classBefore           Boolean   @default(false) @map("class_before")
  classMiddle           Boolean   @default(false) @map("class_middle")
  classAfter            Boolean   @default(false) @map("class_after")
  openBefore            Boolean   @default(false) @map("open_before")
  openAfter             Boolean   @default(false) @map("open_after")
  priority              Int       @default(1)              // 우선순위 1~5
  roles                 String[]  @default([])             // 역할 배열
  passageExposure       String    @default("") @map("passage_exposure") @db.VarChar(20)
  cognitiveLevel        String    @default("") @map("cognitive_level") @db.VarChar(5)
  suitableLevelMin      String    @default("") @map("suitable_level_min") @db.VarChar(20)
  suitableLevelMax      String    @default("") @map("suitable_level_max") @db.VarChar(20)
  estimatedMinutesMin   String    @default("") @map("estimated_minutes_min") @db.VarChar(10)
  estimatedMinutesMax   String    @default("") @map("estimated_minutes_max") @db.VarChar(10)
  prerequisites         String[]  @default([])             // 선행 모듈 코드 배열
  incompatibleWith      String[]  @default([]) @map("incompatible_with")  // 충돌 모듈 코드 배열

  // ── 문항설계 (탭3) ──
  answerType            String    @default("") @map("answer_type") @db.VarChar(50)
  scoringMode           String    @default("") @map("scoring_mode") @db.VarChar(50)
  passageExposureMode   String    @default("") @map("passage_exposure_mode") @db.VarChar(50)
  questionCount         String    @default("single") @map("question_count") @db.VarChar(20)
  feedbackStyle         String    @default("") @map("feedback_style") @db.VarChar(50)
  feedbackType          String    @default("") @map("feedback_type") @db.VarChar(50)

  // ── 성과 KPI (탭4) ──
  selectedKpiCodes      String[]  @default([]) @map("selected_kpi_codes")

  // ── AI 동작 설정 (탭5) ──
  purpose               String    @default("") @db.Text
  pedagogyInstruction   String    @default("") @map("pedagogy_instruction") @db.Text

  // ── 문항 데이터 + 콘텐츠 설정 (JSONB) ──
  questionData          Json      @default("[]") @map("question_data") @db.JsonB
  contentConfig         Json      @default("{}") @map("content_config") @db.JsonB

  // ── 상태/시간 ──
  status                String    @default("active") @db.VarChar(20)  // 'active' | 'inactive'
  createdAt             DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt             DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt             DateTime? @map("deleted_at") @db.Timestamptz

  @@index([skill])
  @@index([status])
  @@index([code])
  @@map("ai_modules")
}
```

### 1-2. 설계 근거

| 결정 사항 | 근거 |
|-----------|------|
| `code` UNIQUE | 모듈 코드는 시스템 전체에서 고유 식별자로 사용 (어댑터 매칭) |
| `questionData` JSONB | tutoring에서 어댑터가 런타임에 해석하는 유연 구조 필요 |
| `contentConfig` JSONB | stage 필드가 많고 모듈별로 사용 필드가 다름. 정규화 비효율 |
| `selectedKpiCodes` String[] | KPI는 코드관리(`code_items`)에서 관리. 모듈은 선택된 코드만 보관 |
| `roles`, `prerequisites`, `incompatibleWith` String[] | PostgreSQL 네이티브 배열. 조회 빈도 낮고 단순 목록 |
| `deletedAt` nullable | 기존 패턴(Institution, User) 준수. soft delete |
| 별도 `ai_module_level_configs` 미생성 | 20260324 문서에서 레벨 다중선택 UI 제거됨. 현시점 불필요 |

---

## 2. 공유 타입 정의 (`packages/types`)

### 2-1. DTO / Response 타입

```typescript
// ── 응답 ──
export interface AiModuleResponse {
  id: string;
  skill: string;
  code: string;
  name: string;
  classBefore: boolean;
  classMiddle: boolean;
  classAfter: boolean;
  openBefore: boolean;
  openAfter: boolean;
  priority: number;
  roles: string[];
  passageExposure: string;
  cognitiveLevel: string;
  suitableLevelMin: string;
  suitableLevelMax: string;
  estimatedMinutesMin: string;
  estimatedMinutesMax: string;
  prerequisites: string[];
  incompatibleWith: string[];
  answerType: string;
  scoringMode: string;
  passageExposureMode: string;
  questionCount: string;
  feedbackStyle: string;
  feedbackType: string;
  selectedKpiCodes: string[];
  purpose: string;
  pedagogyInstruction: string;
  questionData: Record<string, unknown>[];
  contentConfig: Record<string, unknown>;
  status: string;
  createdAt: string;
  updatedAt: string;
}

export interface AiModuleListResponse {
  data: AiModuleResponse[];
  pagination: PaginationResponse;
}

// ── 생성/수정 ──
export interface CreateAiModuleDto {
  skill: string;
  code: string;
  name: string;
  classBefore?: boolean;
  classMiddle?: boolean;
  classAfter?: boolean;
  openBefore?: boolean;
  openAfter?: boolean;
  priority?: number;
  roles?: string[];
  passageExposure?: string;
  cognitiveLevel?: string;
  suitableLevelMin?: string;
  suitableLevelMax?: string;
  estimatedMinutesMin?: string;
  estimatedMinutesMax?: string;
  prerequisites?: string[];
  incompatibleWith?: string[];
  answerType?: string;
  scoringMode?: string;
  passageExposureMode?: string;
  questionCount?: string;
  feedbackStyle?: string;
  feedbackType?: string;
  selectedKpiCodes?: string[];
  purpose?: string;
  pedagogyInstruction?: string;
  questionData?: Record<string, unknown>[];
  contentConfig?: Record<string, unknown>;
  status?: string;
}

export interface UpdateAiModuleDto extends Partial<CreateAiModuleDto> {}
```

---

## 3. Backend 구현 계획

### 3-1. 파일 구조

```
packages/core/src/ai-module/
  ai-module.module.ts
  ai-module.service.ts
  index.ts

apps/admin/backend/src/ai-module/
  ai-module.controller.ts
  ai-module.module.ts
```

### 3-2. API 엔드포인트

| Method | Path | 설명 | 비고 |
|--------|------|------|------|
| `GET` | `/ai-modules` | 목록 조회 (페이지네이션) | `?page=1&limit=20&skill=reading&status=active` |
| `GET` | `/ai-modules/:id` | 단건 조회 | 등록/수정 폼 초기 데이터 |
| `POST` | `/ai-modules` | 신규 등록 | `CreateAiModuleDto` |
| `PUT` | `/ai-modules/:id` | 수정 | `UpdateAiModuleDto` |
| `PATCH` | `/ai-modules/:id/status` | 상태 토글 | `{ status: 'active' | 'inactive' }` |
| `DELETE` | `/ai-modules/:id` | 삭제 (soft delete) | `deletedAt` 설정 |

### 3-3. Core Service 핵심 로직

```typescript
// packages/core/src/ai-module/ai-module.service.ts

@Injectable()
export class AiModuleService {
  constructor(private prisma: PrismaService) {}

  // 목록 조회 - soft delete 제외, 스킬/상태 필터, 페이지네이션
  async findAll(query: { page: number; limit: number; skill?: string; status?: string })

  // 단건 조회 - deletedAt IS NULL 조건
  async findOne(id: string)

  // 등록 - code 중복 검증 포함
  async create(dto: CreateAiModuleDto)

  // 수정 - code 변경 시 중복 검증
  async update(id: string, dto: UpdateAiModuleDto)

  // 상태 토글
  async updateStatus(id: string, status: string)

  // soft delete
  async remove(id: string)
}
```

### 3-4. 검증 규칙

| 필드 | 검증 | 에러 메시지 |
|------|------|-------------|
| `skill` | 필수, `vocabulary`/`reading`/`speaking` 중 하나 | "스킬을 선택해주세요" |
| `code` | 필수, 1~10자 대문자, 시스템 내 고유 | "모듈 코드가 중복됩니다" |
| `name` | 필수, 1~200자 | "모듈명을 입력해주세요" |
| `priority` | 1~5 정수 | "우선순위는 1~5 사이여야 합니다" |
| `cognitiveLevel` | 빈값 또는 `1`~`6` | "인지 수준은 1~6 사이여야 합니다" |

---

## 4. Frontend 구현 계획

### 4-1. API 함수 (`src/lib/api/ai-module.ts` 신규)

```typescript
// fetchApi 래퍼 사용
export async function getAiModules(params?: { page?: number; limit?: number; skill?: string; status?: string }): Promise<AiModuleListResponse>
export async function getAiModule(id: string): Promise<AiModuleResponse>
export async function createAiModule(data: CreateAiModuleDto): Promise<AiModuleResponse>
export async function updateAiModule(id: string, data: UpdateAiModuleDto): Promise<AiModuleResponse>
export async function updateAiModuleStatus(id: string, status: string): Promise<AiModuleResponse>
export async function deleteAiModule(id: string): Promise<void>
```

### 4-2. 목록 페이지 변경 (`page.tsx`)

**현재**: 16개 모듈 하드코딩 배열
**변경 후**:

| 항목 | 현재 | 변경 |
|------|------|------|
| 데이터 소스 | 하드코딩 `modules[]` | `getAiModules()` API 호출 |
| 로딩 | 없음 | 스켈레톤 또는 "데이터를 불러오는 중..." |
| 에러 | 없음 | 에러 메시지 + 재시도 버튼 |
| 상태 토글 | 편집 페이지 진입 필요 | 목록에서 직접 `PATCH /status` 호출 |
| 삭제 | 없음 | 행별 삭제 버튼 + 확인 모달 → `DELETE` 호출 |
| 필터 | 없음 | 스킬별 필터 (선택사항) |
| 타입 | 페이지 내 `AIModule` 인터페이스 | `packages/types`의 `AiModuleResponse` import |

**테이블 컬럼** (20260316 문서 기준 유지):

| 컬럼 | 데이터 | 비고 |
|------|--------|------|
| 스킬 | `skill` → 이모지+카테고리명 | 📚/📖/🎤 매핑 |
| 모듈명 | `(code) name` | |
| 수업 전/중/후 | `classBefore`/`classMiddle`/`classAfter` | 읽기전용 체크박스 |
| 지문 오픈전/후 | `openBefore`/`openAfter` | 읽기전용 체크박스 |
| 우선순위 | `priority` | 숫자 |
| 상태 | `status` | 활성(초록)/비활성(주황) 배지, 클릭 토글 |
| 관리 | 편집/삭제 버튼 | |

### 4-3. 등록/수정 페이지 변경 (`register/page.tsx`)

**현재**: 저장 시 `alert()` + 라우팅만. 수정 모드는 내부 목업 배열에서 일부 필드 주입.
**변경 후**:

| 항목 | 현재 | 변경 |
|------|------|------|
| 등록 | `alert("저장되었습니다")` | `createAiModule(formData)` → 성공 토스트 → 목록 이동 |
| 수정 초기 로드 | 내부 `modules[]` 목업 | `getAiModule(id)` API 호출 → 폼에 바인딩 |
| 수정 저장 | 없음 | `updateAiModule(id, formData)` → 성공 토스트 |
| 폼 검증 | 없음 | 필수값 + 범위 + min≤max 검증 |
| 알림 | `alert()` | 토스트 컴포넌트 |
| 타입 | 페이지 내 중복 선언 | `packages/types` import + 페이지 전용 `FormData`는 로컬 유지 |

**5개 탭 구조** (20260324 문서 기준 유지):

1. **기본 정보**: `skill`, `code`, `name`
2. **커리큘럼 배치 설정**: `roles`, `passageExposure`, `cognitiveLevel`, 레벨/시간 범위, 선행/충돌 모듈
3. **문항설계**: `answerType`, `scoringMode`, `passageExposureMode`, `questionCount`, `feedbackStyle`, `feedbackType`
4. **성과 KPI 설정**: `selectedKpiCodes` (KPI 모달로 선택/제거)
5. **AI 동작 설정**: `pedagogyInstruction`, `purpose`

### 4-4. 삭제 기능 (신규)

- 목록 페이지 행별 삭제 버튼
- `FormModal` 공통 컴포넌트로 삭제 확인
- `deleteAiModule(id)` 호출 후 목록 재조회
- 성공/실패 토스트 표시

---

## 5. 구현 순서 (작업 단계)

### Phase 1: DB + 타입 (기반)
| # | 작업 | 파일 | 비고 |
|---|------|------|------|
| 1-1 | Prisma 스키마에 `AiModule` 모델 추가 | `prisma/schema.prisma` | |
| 1-2 | 마이그레이션 실행 | CLI | `npx prisma migrate dev --name add-ai-modules` |
| 1-3 | 공유 타입 추가 | `packages/types/src/index.ts` | DTO, Response 인터페이스 |

### Phase 2: Backend (API)
| # | 작업 | 파일 | 비고 |
|---|------|------|------|
| 2-1 | Core 서비스 구현 | `packages/core/src/ai-module/ai-module.service.ts` | CRUD + 검증 |
| 2-2 | Core 모듈 등록 | `packages/core/src/ai-module/ai-module.module.ts` | PrismaModule 의존 |
| 2-3 | Core index 내보내기 | `packages/core/src/ai-module/index.ts` + `packages/core/src/index.ts` | barrel export |
| 2-4 | Backend 컨트롤러 구현 | `apps/admin/backend/src/ai-module/ai-module.controller.ts` | 6개 엔드포인트 |
| 2-5 | Backend 모듈 등록 | `apps/admin/backend/src/ai-module/ai-module.module.ts` + `app.module.ts` | |
| 2-6 | Core 빌드 + 백엔드 재시작 | CLI | `pnpm run build` (core) |

### Phase 3: Frontend (화면)
| # | 작업 | 파일 | 비고 |
|---|------|------|------|
| 3-1 | API 함수 작성 | `src/lib/api/ai-module.ts` (신규) | fetchApi 래퍼 사용 |
| 3-2 | 목록 페이지 리팩터링 | `src/app/(admin)/admin/ai-modules/page.tsx` | 하드코딩 → API 연동 |
| 3-3 | 목록에 상태 토글 추가 | 위와 동일 | PATCH 호출 |
| 3-4 | 목록에 삭제 기능 추가 | 위와 동일 | DELETE + 확인 모달 |
| 3-5 | 등록 페이지 - 저장 연동 | `register/page.tsx` | POST 호출 + 토스트 |
| 3-6 | 수정 페이지 - 데이터 로딩 | `register/page.tsx` | GET 조회 → 폼 바인딩 |
| 3-7 | 수정 페이지 - 저장 연동 | `register/page.tsx` | PUT 호출 + 토스트 |
| 3-8 | 폼 검증 추가 | `register/page.tsx` | 필수값, 범위, min≤max |
| 3-9 | 타입 정리 | 각 페이지 | 중복 인터페이스 제거 → types import |

### Phase 4: 정리
| # | 작업 | 파일 | 비고 |
|---|------|------|------|
| 4-1 | alert() → 토스트 전환 | 등록/수정 페이지 | |
| 4-2 | 하드코딩 목업 데이터 제거 | 목록/등록 페이지 | |
| 4-3 | 빌드 검증 | 전체 | `pnpm run build` |

---

## 6. 시드 데이터 (초기 16개 모듈)

20260316 문서 기준, 마이그레이션 후 아래 16개 모듈을 seed로 삽입:

| skill | code | name | priority | status |
|-------|------|------|----------|--------|
| vocabulary | WRD | Word Reading Deck | 1 | active |
| vocabulary | WSD | Word Speaking Deck | 2 | active |
| vocabulary | IMG | Meaning Guessing | 3 | active |
| vocabulary | WW | Word Web | 4 | active |
| reading | PRD | Prediction | 1 | active |
| reading | SCN | Scanning | 2 | active |
| reading | SKM | Skimming | 3 | active |
| reading | QAR | Reading QAR | 4 | active |
| reading | CLR | Clarification | 5 | active |
| reading | SUM | Summarizing | 6 | active |
| reading | ORL | Oral Reading | 7 | inactive |
| speaking | WSP | Word Speaking | 1 | active |
| speaking | LR | Listen & Repeat | 2 | active |
| speaking | SXP | Sentence Expansion | 3 | active |
| speaking | SHD | Shadowing | 4 | active |
| speaking | SNR | Scenario Talking | 5 | active |

> 커리큘럼 배치(classBefore, openBefore 등), 문항설계, KPI, AI 동작 설정 필드는 등록 후 관리자가 편집 페이지에서 입력.

---

## 7. 기존 코드관리 페이지 영향 범위

`code-management/page.tsx`는 이번 CRUD 구현의 직접 대상이 **아님**.
단, 아래 연관 사항 인지 필요:

| 항목 | 내용 |
|------|------|
| KPI 그룹코드 통일 | 코드관리는 `KPI_CATEGORY`, 등록페이지는 `KPI_ITEMS` 사용 중 → 통일 필요 (별도 작업) |
| 코드관리 데이터 의존 | 등록 페이지의 `answerType`, `scoringMode` 등 선택지가 코드관리에서 관리됨 |
| 변경 없음 | code-management 페이지 자체는 이미 API 연동 완료 상태 |

---

## 8. tutoring.picklass.com 연동 고려사항

백오피스에서 저장한 모듈 데이터가 tutoring 앱에서 소비되는 흐름:

```
[백오피스] 모듈 등록/수정
    → ai_modules 테이블 저장
    → contentConfig (JSONB), questionData (JSONB) 포함

[tutoring 앱] 학습 실행
    → fetchLessonPlan(lessonId) → moduleSequence 조회
    → getAdapter(moduleCode) → adapter.fetchModuleData()
    → adapter.buildContentConfig(data) → Orchestrator 실행
```

| 필드 | 백오피스 역할 | tutoring 소비 방식 |
|------|-------------|-------------------|
| `code` | 모듈 식별 코드 등록 | `getAdapter(moduleCode)`로 어댑터 선택 |
| `contentConfig` | JSON 편집기로 stage 설정 | `buildContentConfig()`에서 해석 → Orchestrator Rule 적용 |
| `questionData` | 문항 배열 편집 | `fetchModuleData()`에서 문항 목록으로 반환 |
| `pedagogyInstruction` | AI 교수법 지시문 입력 | ChatAgent/FeedbackAgent 시스템 프롬프트로 전달 |
| `selectedKpiCodes` | KPI 코드 선택 | 학습 결과 저장 시 측정 항목 매핑 |

> `contentConfig`의 stage 필드는 모듈별로 사용하는 필드가 다르므로, 백오피스에서는 JSONB 통째 저장하고 tutoring 어댑터가 필요한 필드만 해석하는 구조가 적합.

---

## 9. 리스크 및 주의사항

| 리스크 | 대응 |
|--------|------|
| 프론트엔드 타입 중복 제거 시 기존 하드코딩 로직 깨짐 | Phase 3에서 API 연동과 동시에 타입 교체 |
| `contentConfig` JSONB 구조가 tutoring과 불일치 | tutoring의 `ai-module.ts` 타입 참조하여 초기값 구조 확정 |
| KPI 그룹코드 불일치 (`KPI_CATEGORY` vs `KPI_ITEMS`) | 이번 구현에서는 기존 등록페이지의 `KPI_ITEMS` 기준 유지. 통일은 별도 작업 |
| `alert()` → 토스트 전환 시 UX 변경 | 기존 공통 토스트 컴포넌트 존재 여부 확인 후 적용 |
| Core 빌드 누락 시 백엔드 에러 | Phase 2-6에서 반드시 `pnpm run build` 실행 |

---

## 10. 코드관리 연동 플랜 — 등록 페이지 선택지를 code_groups/code_items로 이관

> 작성일: 2026-04-01
> 목적: 등록/수정 페이지에서 **하드코딩된 선택지**를 코드관리 페이지(`code-management`)의 `code_groups`/`code_items` 테이블로 이관하여, 관리자가 코드관리에서 옵션을 추가/수정/삭제하면 등록 페이지에 자동 반영되도록 함.

### 10-1. 이관 대상 필드 ↔ code_groups 매핑

| # | 등록 페이지 필드 | 현재 상태 | 신규 groupCode | code_items 구조 | 비고 |
|---|---|---|---|---|---|
| 1 | **skill** | `constants.ts` `SKILLS` 배열 하드코딩 | `SKILL` | `code`: vocabulary, reading, speaking 등 / `name`: 표시명 / `extraData.emoji`: 이모지 | 기존 `SKILLS` 상수 대체 |
| 2 | **roles** | 페이지 내 `LESSON_ROLE_OPTIONS` 하드코딩 | `MODULE_ROLE` | `code`: warming, passage-use, practice, output / `name`: 설명 | 체크박스 복수 선택 |
| 3 | **passageExposure** | 페이지 내 `LESSON_PASSAGE_EXPOSURE_OPTIONS` 하드코딩 | `PASSAGE_EXPOSURE` | `code`: before, during, after, any / `name`: 설명 | 단일 select |
| 4 | **cognitiveLevel** | 페이지 내 `<option>` 6개 하드코딩 | `COGNITIVE_LEVEL` | `code`: 1~6 / `name`: 기억, 이해, 적용, 분석, 종합, 평가 / `extraData.taxonomy`: Bloom's 단계명 | 단일 select |
| 5 | **suitableLevels (min/max)** | `constants.ts` `LEVEL_SYSTEM` 18개 레벨 | `SUITABLE_LEVEL` | `code`: 1~18 / `name`: A1-, A1, A1+ ... C2+ / `extraData.category`: Starter 등 | 두 개 select (min, max) |
| 6 | **answerType** | 페이지 내 5개 라디오 하드코딩 | `ANSWER_TYPE` | **이미 존재** / `code`: essay, sentence-write 등 / `name`: 설명 / `extraData.scoringHint`: 채점 가이드 | 기존 탭 활용 + `extraData` 확장 |
| 7 | **scoringMode** | 페이지 내 3개 라디오 하드코딩 | `SCORING_MODE` | **이미 존재** / `code`: exact, holistic, pronunciation / `name`: 설명 | 기존 탭 활용 |
| 8 | **passageExposureMode** | 페이지 내 3개 라디오 하드코딩 | `PASSAGE_EXPOSURE_MODE` | **이미 존재** / `code`: hidden, preview, full / `name`: 설명 | 기존 탭 활용 |
| 9 | **questionCount** | 페이지 내 2개 라디오 하드코딩 | `QUESTION_COUNT` | `code`: single, multi / `name`: 문항 1개, 문항 2개 이상 | 신규 그룹 |
| 10 | **feedbackStyle** | 페이지 내 2개 라디오 하드코딩 | `FEEDBACK_STYLE` | **이미 존재** / `code`: correct-wrong, strengths-weaknesses / `name`: 설명 | 기존 탭 활용 |
| 11 | **feedbackType** | 페이지 내 4개 라디오 하드코딩 | `FEEDBACK_TYPE` | **이미 존재** / `code`: correctness, holistic, pronunciation, writing / `name`: 설명 | 기존 탭 활용 |
| 12 | **성과KPI** | `KPI_ITEMS` 또는 `KPI_CATEGORY` 혼용 | `KPI_CATEGORY` | **이미 존재** / code, goal, measureUnit, measureMethod | **통일: `KPI_CATEGORY` 단일 사용** |

### 10-2. 기존 탭 vs 신규 추가 정리

**이미 code-management 탭에 존재하는 그룹 (6개):**
- `PASSAGE_EXPOSURE_MODE`, `SCORING_MODE`, `ANSWER_TYPE`, `FEEDBACK_STYLE`, `FEEDBACK_TYPE`, `KPI_CATEGORY`

**신규 code_groups 추가 필요 (6개):**

| groupCode | name | description |
|---|---|---|
| `SKILL` | 스킬 | 모듈 스킬 분류 (vocabulary, reading, speaking 등) |
| `MODULE_ROLE` | 모듈 역할 | 레슨 내 모듈 역할 (warming, passage-use, practice, output) |
| `PASSAGE_EXPOSURE` | 지문 공개 시점 | 모듈별 지문 노출 타이밍 제약 (before, during, after, any) |
| `COGNITIVE_LEVEL` | 인지 수준 | Bloom's Taxonomy 6단계 |
| `SUITABLE_LEVEL` | 적합 레벨 | CEFR 기반 18단계 레벨 시스템 |
| `QUESTION_COUNT` | 문항 수 | 단문항/다문항 구분 |

**code-management 탭에 추가할 탭:**

| 탭 key | 탭 label | 포함 그룹 |
|---|---|---|
| `skill` | Skill | `SKILL` |
| `module-role` | 모듈 역할 | `MODULE_ROLE` |
| `passage-exposure` | Passage Exposure | `PASSAGE_EXPOSURE` |
| `cognitive-level` | 인지 수준 | `COGNITIVE_LEVEL` |
| `suitable-level` | 적합 레벨 | `SUITABLE_LEVEL` |
| `question-count` | 문항 수 | `QUESTION_COUNT` |

### 10-3. code_items 시드 데이터

#### SKILL
| code | name | sortOrder | extraData |
|---|---|---|---|
| vocabulary | Vocabulary | 1 | `{"emoji": "📚", "korName": "어휘"}` |
| reading | Reading | 2 | `{"emoji": "📖", "korName": "읽기"}` |
| speaking | Speaking | 3 | `{"emoji": "🎤", "korName": "말하기"}` |

#### MODULE_ROLE
| code | name | sortOrder |
|---|---|---|
| warming | Warming | 1 |
| passage-use | Passage Use | 2 |
| practice | Practice | 3 |
| output | Output | 4 |

#### PASSAGE_EXPOSURE
| code | name | sortOrder |
|---|---|---|
| before | Before | 1 |
| during | During | 2 |
| after | After | 3 |
| any | Any | 4 |

#### COGNITIVE_LEVEL
| code | name | sortOrder | extraData |
|---|---|---|---|
| 1 | 기억 | 1 | `{"taxonomy": "Remember"}` |
| 2 | 이해 | 2 | `{"taxonomy": "Understand"}` |
| 3 | 적용 | 3 | `{"taxonomy": "Apply"}` |
| 4 | 분석 | 4 | `{"taxonomy": "Analyze"}` |
| 5 | 종합 | 5 | `{"taxonomy": "Evaluate"}` |
| 6 | 평가 | 6 | `{"taxonomy": "Create"}` |

#### SUITABLE_LEVEL
| code | name | sortOrder | extraData |
|---|---|---|---|
| 1 | A1- | 1 | `{"category": "Starter"}` |
| 2 | A1 | 2 | `{"category": "Starter"}` |
| ... | ... | ... | ... |
| 18 | C2+ | 18 | `{"category": "Proficient"}` |

> 기존 `LEVEL_SYSTEM` 상수의 18개 레벨 전체를 code_items로 이관

#### QUESTION_COUNT
| code | name | sortOrder | extraData |
|---|---|---|---|
| single | 문항 1개 | 1 | `{"description": "단문항"}` |
| multi | 문항 2개 이상 | 2 | `{"description": "다문항"}` |

#### 기존 그룹 시드 보강 (extraData 추가)

**ANSWER_TYPE** — 기존 items에 `extraData.scoringHint` 추가:
| code | extraData 추가 |
|---|---|
| essay | `{"description": "주관식 서술형", "scoringHint": "holistic"}` |
| sentence-write | `{"description": "영문 문장 작성", "scoringHint": "holistic"}` |
| audio-record | `{"description": "음성 녹음 (쉐도잉)", "scoringHint": "pronunciation"}` |
| option | `{"description": "객관식", "scoringHint": "accuracy"}` |
| short-text | `{"description": "단답형", "scoringHint": "accuracy"}` |

### 10-4. 등록 페이지 변경 계획

#### 핵심 원칙
- 등록 페이지 마운트 시 `getAllCodes()` (또는 필요 그룹만 개별 조회)로 코드 데이터를 로드
- 하드코딩 상수(`SKILLS`, `LESSON_ROLE_OPTIONS`, `LESSON_PASSAGE_EXPOSURE_OPTIONS`, 인라인 옵션 배열)를 **모두 제거**하고, 로드된 코드 데이터에서 동적으로 선택지 생성
- `constants.ts`의 `SKILLS`, `LEVEL_SYSTEM` 상수는 다른 페이지에서도 참조하므로 **즉시 삭제하지 않고 deprecated 처리** 후, 다른 소비처 이관 완료 시 제거

#### 변경 파일 목록

| 파일 | 변경 내용 |
|---|---|
| `code-management/page.tsx` | `buildTabConfig()`에 6개 신규 탭 추가 |
| `register/page.tsx` | 하드코딩 선택지 → 코드 API 기반 동적 렌더링 |
| `register/page.tsx` | KPI 그룹코드를 `KPI_CATEGORY`로 통일 (기존 `KPI_ITEMS` 조회 → `KPI_CATEGORY` 조회) |
| `src/lib/api/code.ts` | (변경 없음 — 기존 `getAllCodes`, `getCodesByGroup` 그대로 사용) |

#### 등록 페이지 데이터 로딩 전략

```
페이지 마운트
  ├─ getAiModule(id)          // 수정 모드일 때
  └─ getAllCodes()             // 코드 데이터 로드 (1회)
       ├─ codeMap['SKILL']           → skill select 옵션
       ├─ codeMap['MODULE_ROLE']     → roles 체크박스 옵션
       ├─ codeMap['PASSAGE_EXPOSURE']→ passageExposure select 옵션
       ├─ codeMap['COGNITIVE_LEVEL'] → cognitiveLevel select 옵션
       ├─ codeMap['SUITABLE_LEVEL']  → suitableLevelMin/Max select 옵션
       ├─ codeMap['ANSWER_TYPE']     → answerType 라디오 옵션
       ├─ codeMap['SCORING_MODE']    → scoringMode 라디오 옵션
       ├─ codeMap['PASSAGE_EXPOSURE_MODE'] → passageExposureMode 라디오 옵션
       ├─ codeMap['QUESTION_COUNT']  → questionCount 라디오 옵션
       ├─ codeMap['FEEDBACK_STYLE']  → feedbackStyle 라디오 옵션
       ├─ codeMap['FEEDBACK_TYPE']   → feedbackType 라디오 옵션
       └─ codeMap['KPI_CATEGORY']    → KPI 선택 모달 옵션
```

### 10-5. 필드별 UI 바인딩 상세 (등록 · 수정 · 목록)

> `codeMap`은 `Record<string, CodeItemResponse[]>` 타입.
> 각 `CodeItemResponse`는 `{ id, code, name, sortOrder, extraData }` 구조.

#### 10-5-1. 등록 페이지 (`register/page.tsx`) — 코드 로딩 및 상태

```typescript
// ── 코드 데이터 상태 ──
const [codeMap, setCodeMap] = useState<Record<string, CodeItemResponse[]>>({});
const [codeLoading, setCodeLoading] = useState(true);

// ── 마운트 시 1회 로드 ──
useEffect(() => {
  let ignore = false;
  const load = async () => {
    try {
      const groups = await getAllCodes();
      const map: Record<string, CodeItemResponse[]> = {};
      for (const g of groups) map[g.code] = g.items;
      if (!ignore) setCodeMap(map);
    } catch { /* 에러 시 빈 맵 유지 → 선택지 없음 표시 */ }
    finally { if (!ignore) setCodeLoading(false); }
  };
  load();
  return () => { ignore = true; };
}, []);
```

#### 10-5-2. 필드별 렌더링 + 등록/수정 바인딩 + 목록 표시

| # | 필드 | groupCode | UI 컨트롤 | 등록 (신규) | 수정 (로드) | 목록 표시 | 저장 DTO 값 |
|---|---|---|---|---|---|---|---|
| 1 | **skill** | `SKILL` | `<select>` 단일 선택 | `codeMap['SKILL']`로 `<option>` 생성. `formData.skill`에 `code` 값 바인딩 | API 응답 `data.skill`(= code 값) → `formData.skill` 세팅. select가 해당 값 selected | `codeMap['SKILL']`에서 code 매칭 → `extraData.emoji + ' ' + name` 표시 (예: "📚 Vocabulary") | `dto.skill = formData.skill` (code 값 그대로) |
| 2 | **roles** | `MODULE_ROLE` | `<input type="checkbox">` 복수 선택 | `codeMap['MODULE_ROLE']`로 체크박스 목록 생성. `formData.roles[]`에 checked된 `code` 값들 | API 응답 `data.roles[]` → `formData.roles` 세팅. 해당 code들 checked 상태 | 목록 페이지에 표시 안함 (상세에서만) | `dto.roles = formData.roles` (code 배열) |
| 3 | **passageExposure** | `PASSAGE_EXPOSURE` | `<select>` 단일 선택 | `codeMap['PASSAGE_EXPOSURE']`로 `<option>` 생성 | API 응답 `data.passageExposure` → select selected | 목록 페이지에 표시 안함 | `dto.passageExposure = formData.passageExposure` |
| 4 | **cognitiveLevel** | `COGNITIVE_LEVEL` | `<select>` 단일 선택 | `codeMap['COGNITIVE_LEVEL']`로 `<option value={code}>{code} - {name}</option>` 생성 (예: "1 - 기억") | API 응답 `data.cognitiveLevel` → select selected | 목록 페이지에 표시 안함 | `dto.cognitiveLevel = formData.cognitiveLevel` |
| 5 | **suitableLevelMin** | `SUITABLE_LEVEL` | `<select>` 단일 선택 | `codeMap['SUITABLE_LEVEL']`로 `<option value={code}>{name} (extraData.category)</option>` 생성 (예: "A1- (Starter)") | API 응답 `data.suitableLevelMin` → select selected | 목록 페이지에 표시 안함 | `dto.suitableLevelMin = formData.suitableLevelMin` |
| 5' | **suitableLevelMax** | `SUITABLE_LEVEL` | `<select>` 단일 선택 | 동일 (min과 같은 option 목록) | API 응답 `data.suitableLevelMax` → select selected | 목록 페이지에 표시 안함 | `dto.suitableLevelMax = formData.suitableLevelMax` |
| 6 | **answerType** | `ANSWER_TYPE` | 라디오 카드 (styled radio) | `codeMap['ANSWER_TYPE']`로 라디오 카드 생성. `code`가 value, `name`이 라벨, `extraData.scoringHint`가 보조 설명 | API 응답 `data.answerType` → 해당 라디오 checked | 목록 페이지에 표시 안함 | `dto.answerType = formData.answerType` |
| 7 | **scoringMode** | `SCORING_MODE` | 라디오 카드 | `codeMap['SCORING_MODE']`로 생성 | API 응답 `data.scoringMode` → checked | 목록 페이지에 표시 안함 | `dto.scoringMode = formData.scoringMode` |
| 8 | **passageExposureMode** | `PASSAGE_EXPOSURE_MODE` | 라디오 카드 | `codeMap['PASSAGE_EXPOSURE_MODE']`로 생성 | API 응답 `data.passageExposureMode` → checked | 목록 페이지에 표시 안함 | `dto.passageExposureMode = formData.passageExposureMode` |
| 9 | **questionCount** | `QUESTION_COUNT` | 라디오 카드 | `codeMap['QUESTION_COUNT']`로 생성. `extraData.description`이 보조 설명 | API 응답 `data.questionCount` → checked | 목록 페이지에 표시 안함 | `dto.questionCount = formData.questionCount` |
| 10 | **feedbackStyle** | `FEEDBACK_STYLE` | 라디오 카드 | `codeMap['FEEDBACK_STYLE']`로 생성 | API 응답 `data.feedbackStyle` → checked | 목록 페이지에 표시 안함 | `dto.feedbackStyle = formData.feedbackStyle` |
| 11 | **feedbackType** | `FEEDBACK_TYPE` | 라디오 카드 | `codeMap['FEEDBACK_TYPE']`로 생성 | API 응답 `data.feedbackType` → checked | 목록 페이지에 표시 안함 | `dto.feedbackType = formData.feedbackType` |
| 12 | **selectedKpiCodes** | `KPI_CATEGORY` | KPI 모달 (테이블 선택) | `codeMap['KPI_CATEGORY']`로 선택 가능 KPI 목록 표시. 선택 시 `formData.contentConfig.selectedKpiCodes[]`에 code 추가 | API 응답 `data.selectedKpiCodes[]` → contentConfig에 세팅. 모달에서 이미 선택된 항목 표시 | 목록 페이지에 표시 안함 | `dto.selectedKpiCodes = formData.contentConfig.selectedKpiCodes` |

#### 10-5-3. 등록 페이지 — 공통 렌더링 헬퍼 (의사코드)

```typescript
// select 타입 필드 렌더링
function renderCodeSelect(
  groupCode: string,            // 'SKILL' | 'COGNITIVE_LEVEL' | ...
  fieldName: string,            // formData의 키
  value: string,                // formData[fieldName]
  formatLabel?: (item: CodeItemResponse) => string  // 표시 형식 커스텀
) {
  const items = codeMap[groupCode] ?? [];
  return (
    <select name={fieldName} value={value} onChange={handleInputChange} style={FIELD_STYLE}>
      <option value="">선택하세요</option>
      {items.map((item) => (
        <option key={item.code} value={item.code}>
          {formatLabel ? formatLabel(item) : item.code}
        </option>
      ))}
    </select>
  );
}

// 라디오 카드 타입 필드 렌더링
function renderCodeRadioCards(
  groupCode: string,
  fieldName: string,
  value: string,
  getDescription?: (item: CodeItemResponse) => string
) {
  const items = codeMap[groupCode] ?? [];
  return (
    <div style={{ display: 'flex', flexWrap: 'wrap', gap: '10px' }}>
      {items.map((item) => (
        <label key={item.code} style={{ /* 기존 라디오 카드 스타일 */ }}>
          <input type="radio" name={fieldName} value={item.code}
            checked={value === item.code} onChange={handleInputChange}
            style={{ display: 'none' }} />
          {item.code}
          {getDescription && (
            <span style={{ fontSize: '12px', color: '#888' }}>
              — {getDescription(item)}
            </span>
          )}
        </label>
      ))}
    </div>
  );
}

// 체크박스 복수선택 필드 렌더링
function renderCodeCheckboxes(
  groupCode: string,
  fieldName: string,
  values: string[]
) {
  const items = codeMap[groupCode] ?? [];
  return (
    <div style={{ display: 'flex', gap: '15px', flexWrap: 'wrap' }}>
      {items.map((item) => (
        <label key={item.code} style={{ display: 'flex', alignItems: 'center' }}>
          <input type="checkbox" value={item.code}
            checked={values.includes(item.code)}
            onChange={(e) => handleCheckboxChange(e, fieldName)} />
          {item.name}
        </label>
      ))}
    </div>
  );
}
```

#### 10-5-4. 목록 페이지 (`page.tsx`) — 코드 기반 라벨 매핑

```typescript
// 목록 페이지에서도 코드 데이터를 로드하여 skill 라벨을 동적 표시
const [codeMap, setCodeMap] = useState<Record<string, CodeItemResponse[]>>({});

useEffect(() => {
  getAllCodes().then((groups) => {
    const map: Record<string, CodeItemResponse[]> = {};
    for (const g of groups) map[g.code] = g.items;
    setCodeMap(map);
  }).catch(() => { /* 실패 시 code 값 그대로 표시 */ });
}, []);

// 스킬 라벨 변환 함수
function getSkillLabel(skillCode: string): string {
  const items = codeMap['SKILL'] ?? [];
  const item = items.find((i) => i.code === skillCode);
  if (!item) return skillCode;  // fallback: code 그대로
  const emoji = typeof item.extraData?.emoji === 'string' ? item.extraData.emoji : '';
  return `${emoji} ${item.name}`.trim();
}

// 테이블에서 사용
// 현재: <td>{SKILL_LABELS[module.skill] ?? module.skill}</td>
// 변경: <td>{getSkillLabel(module.skill)}</td>
```

> 기존 `SKILL_LABELS` 하드코딩 객체를 제거하고, `codeMap` 기반 동적 매핑으로 대체

#### 10-5-5. 수정 모드 데이터 흐름 (전체)

```
1. 페이지 마운트
   ├─ getAllCodes() → codeMap 세팅 (선택지 준비)
   └─ getAiModule(id) → API 응답 (AiModuleResponse)

2. API 응답 → formData 바인딩
   ├─ data.skill ("vocabulary")        → formData.skill = "vocabulary"
   ├─ data.roles (["warming","output"])→ formData.roles = ["warming","output"]
   ├─ data.passageExposure ("before")  → formData.passageExposure = "before"
   ├─ data.cognitiveLevel ("3")        → formData.cognitiveLevel = "3"
   ├─ data.suitableLevelMin ("4")      → formData.suitableLevelMin = "4"
   ├─ data.suitableLevelMax ("12")     → formData.suitableLevelMax = "12"
   ├─ data.answerType ("essay")        → formData.answerType = "essay"
   ├─ data.scoringMode ("holistic")    → formData.scoringMode = "holistic"
   ├─ data.passageExposureMode ("full")→ formData.passageExposureMode = "full"
   ├─ data.questionCount ("single")    → formData.questionCount = "single"
   ├─ data.feedbackStyle ("correct-wrong") → formData.feedbackStyle = "correct-wrong"
   ├─ data.feedbackType ("holistic")   → formData.feedbackType = "holistic"
   └─ data.selectedKpiCodes (["Fluency"]) → formData.contentConfig.selectedKpiCodes = ["Fluency"]

3. UI 렌더링
   ├─ select 컨트롤: codeMap[groupCode]로 option 생성 → formData 값과 매칭하여 selected
   ├─ 라디오 카드: codeMap[groupCode]로 카드 생성 → formData 값과 매칭하여 checked
   ├─ 체크박스: codeMap[groupCode]로 목록 생성 → formData 배열과 매칭하여 checked
   └─ KPI 모달: codeMap['KPI_CATEGORY']에서 selectedKpiCodes에 없는 항목만 선택 가능으로 표시

4. 저장 (handleSubmit)
   ├─ formData → CreateAiModuleDto/UpdateAiModuleDto 변환
   │   (code 값 그대로 전달, UI 라벨은 저장하지 않음)
   └─ POST/PUT /api/ai-modules → DB 저장
```

#### 10-5-6. 코드관리 페이지 (`code-management/page.tsx`) — 입력/수정/삭제

> 기존 `CodeGroupTable` 컴포넌트의 CRUD 동작을 그대로 활용. 신규 탭만 추가.

| 동작 | 흐름 | 비고 |
|---|---|---|
| **조회** | 페이지 마운트 → `getAllCodes()` → `codeData[groupCode]`로 각 탭 테이블 표시 | 기존 로직 그대로 |
| **추가** | `+` 버튼 → 로컬 newItem 생성 (`_isNew: true`) → 행 추가 | 기존 로직 그대로 |
| **수정** | 인라인 편집 → 로컬 상태 반영 | 기존 로직 그대로 |
| **순서 이동** | `↑/↓` → 로컬 순서 변경 | 기존 로직 그대로 |
| **삭제** | `-` → 기존 항목은 `DELETE /codes/:groupCode/items/:itemId`, 신규 항목은 로컬 제거 | 기존 로직 그대로 |
| **전체 저장** | `PUT /codes/:groupCode/items` → `sortOrder` 재부여 후 서버 저장 → `loadData()` 재조회 | 기존 로직 그대로 |

신규 탭 컬럼 정의:

```typescript
// SKILL 탭
{
  groupCode: 'SKILL',
  title: 'Skill',
  columns: [
    { key: 'code', label: '코드', type: 'text', width: '25%', readOnlyOnExisting: true },
    { key: 'name', label: '표시명', type: 'text', width: '25%' },
    { key: 'extraData.emoji', label: '이모지', type: 'text', width: '15%' },
    { key: 'extraData.korName', label: '한글명', type: 'text', width: '25%' },
  ],
  newItemDefaults: { extraData: { emoji: '', korName: '' } },
}

// MODULE_ROLE, PASSAGE_EXPOSURE, QUESTION_COUNT 탭
// → nameCodeColumns (코드/설명) 재사용

// COGNITIVE_LEVEL 탭
{
  groupCode: 'COGNITIVE_LEVEL',
  title: '인지 수준',
  columns: [
    { key: 'code', label: '레벨', type: 'text', width: '15%', readOnlyOnExisting: true },
    { key: 'name', label: '한글명', type: 'text', width: '25%' },
    { key: 'extraData.taxonomy', label: 'Bloom 단계', type: 'text', width: '25%' },
  ],
  newItemDefaults: { extraData: { taxonomy: '' } },
}

// SUITABLE_LEVEL 탭
{
  groupCode: 'SUITABLE_LEVEL',
  title: '적합 레벨',
  columns: [
    { key: 'code', label: '레벨', type: 'text', width: '10%', readOnlyOnExisting: true },
    { key: 'name', label: 'CEFR', type: 'text', width: '20%' },
    { key: 'extraData.category', label: '카테고리', type: 'text', width: '25%' },
  ],
  newItemDefaults: { extraData: { category: '' } },
}
```

### 10-6. 구현 순서 (Phase 5)

| # | 작업 | 파일 | 비고 |
|---|------|------|------|
| 5-1 | 신규 6개 `code_groups` + `code_items` 시드 SQL 실행 | DB (raw SQL) | SKILL, MODULE_ROLE, PASSAGE_EXPOSURE, COGNITIVE_LEVEL, SUITABLE_LEVEL, QUESTION_COUNT |
| 5-2 | 기존 그룹 시드 보강 (ANSWER_TYPE extraData 등) | DB (raw SQL) | 기존 items가 없으면 INSERT, 있으면 UPDATE extraData |
| 5-3 | code-management `buildTabConfig()`에 6개 신규 탭 추가 | `code-management/page.tsx` | 기존 6개 탭 뒤에 추가 |
| 5-4 | 등록 페이지에 코드 데이터 로딩 추가 (`getAllCodes`) | `register/page.tsx` | useEffect에서 1회 로드 → `codeMap` state 관리 |
| 5-5 | 등록 페이지 하드코딩 선택지 제거 → codeMap 기반 동적 렌더링 | `register/page.tsx` | skill, roles, passageExposure, cognitiveLevel, suitableLevels, answerType, scoringMode, passageExposureMode, questionCount, feedbackStyle, feedbackType |
| 5-6 | KPI 그룹코드 통일 (`KPI_ITEMS` → `KPI_CATEGORY`) | `register/page.tsx` | `getCodesByGroup('KPI_ITEMS')` → `codeMap['KPI_CATEGORY']` 사용 |
| 5-7 | 빌드 검증 | 전체 | TypeScript 빌드 + 동작 확인 |

### 10-7. 사이드이펙트 분석

| 영향 범위 | 분석 | 대응 |
|---|---|---|
| `constants.ts` `SKILLS` | 등록 페이지 외에 목록 페이지 등에서도 참조 가능 | 목록 페이지도 같은 `codeMap`으로 스킬 라벨 매핑하도록 변경. 상수는 deprecated 주석 처리 |
| `constants.ts` `LEVEL_SYSTEM` | 등록 페이지에서 `GROUPS` 생성에 사용 | `SUITABLE_LEVEL` 코드로 대체. 상수는 deprecated 주석 처리 |
| 목록 페이지 `SKILL_LABELS` | 현재 로컬 하드코딩 매핑 | `SKILL` 그룹 코드에서 `extraData.emoji` + `name`으로 동적 생성 |
| code-management 기존 탭 | 탭 12개로 증가 시 UI 가독성 | 탭 스크롤 or 카테고리 그룹핑 고려 (현재는 단순 추가로 충분) |
| 기존 DB에 저장된 모듈 데이터 | `ai_modules.skill` 등 기존 값이 code_items.code와 일치해야 함 | 시드 데이터가 기존 하드코딩 값과 동일한 code를 사용하므로 호환 OK |
| tutoring 앱 | `moduleCode`로 어댑터 선택하는 로직에 영향 없음 | code_groups는 백오피스 UI 전용. tutoring은 `ai_modules` 테이블만 소비 |

### 10-8. 리스크 및 주의사항

| 리스크 | 대응 |
|---|---|
| `getAllCodes()` 호출 시 그룹이 12개로 증가 → 응답 크기 | 현재 각 그룹 items 수가 적어 (최대 18개) 문제 없음. 향후 lazy load 전환 가능 |
| 코드관리에서 항목 삭제 시 기존 모듈의 참조 깨짐 | code_items 삭제 시 해당 코드를 사용 중인 모듈이 있는지 경고 표시 (향후 작업) |
| ANSWER_TYPE extraData 보강 시 기존 items 덮어쓰기 | `UPDATE SET extraData = extraData || '...'` (JSONB merge)로 기존 값 보존 |
| 등록 페이지 코드 로딩 실패 시 폼 사용 불가 | 로딩 에러 시 에러 메시지 + 재시도 버튼. 폼 자체를 비활성화하지 않고 빈 선택지로 표시 |

---

## 11. Phase 5 구현 중 발견된 이슈 및 수정 플랜

> 작성일: 2026-04-01
> 발견 시점: Phase 5-1~5-7 구현 완료 후 동작 테스트

### 11-1. 문제 분석

**증상**: code-management 페이지에서 문항설계 탭 항목 저장 시 `NotFoundException: 코드 그룹 'ANSWER_TYPE'을(를) 찾을 수 없습니다.` 발생

**원인**: code-management 페이지가 참조하는 기존 6개 그룹 중 5개가 **DB에 `code_groups` 레코드가 존재하지 않음**

- `code.service.ts` `findByGroupCode()`는 그룹이 없으면 빈 배열 반환 → **조회는 에러 없이 동작** (빈 테이블 표시)
- `code.service.ts` `saveGroupItems()`는 그룹이 없으면 `NotFoundException` → **저장 시 404 에러**

### 11-2. 누락 code_groups 목록

현재 DB에 존재하는 그룹(`GET /api/codes` 응답 기준):

```
ACCESS_CODE_DURATION, ACCESS_CODE_STATUS, BILLING_CYCLE,
COGNITIVE_LEVEL, CONTRACT_STATUS, INSTITUTION_STATUS,
INSTITUTION_TYPE, KPI_CATEGORY, KPI_GRAMMAR, KPI_LISTENING,
KPI_READING, KPI_SPEAKING, KPI_VOCABULARY, KPI_WRITING,
LEVEL_CATEGORY, LEVEL_SYSTEM, MODULE_ROLE, OPERATION_TYPE,
PASSAGE_EXPOSURE, PLAN_TYPE, QUESTION_COUNT, SKILL,
SUITABLE_LEVEL, USER_ROLE, USER_STATUS
```

**DB에 존재하지 않는 그룹 (code-management에서 필요)**:

| groupCode | 상태 | 등록 페이지에서의 역할 |
|---|---|---|
| `ANSWER_TYPE` | **누락** | answerType 라디오 선택지 |
| `SCORING_MODE` | **누락** | scoringMode 라디오 선택지 |
| `PASSAGE_EXPOSURE_MODE` | **누락** | passageExposureMode 라디오 선택지 |
| `FEEDBACK_STYLE` | **누락** | feedbackStyle 라디오 선택지 |
| `FEEDBACK_TYPE` | **누락** | feedbackType 라디오 선택지 |

### 11-3. 연쇄 영향 분석

Phase 5-2에서 실행한 시드 SQL도 이 누락의 영향을 받음:

| SQL | 의도 | 실제 결과 |
|---|---|---|
| `UPDATE code_items ... WHERE group_id = (SELECT id FROM code_groups WHERE code = 'ANSWER_TYPE')` | ANSWER_TYPE items에 extraData 보강 | **0행 업데이트** (group 없음 → subquery NULL) |
| `INSERT INTO code_items ... CROSS JOIN code_groups g WHERE g.code = 'ANSWER_TYPE'` | multiple-choice 항목 추가 | **0행 삽입** (CROSS JOIN 결과 0행) |
| `INSERT INTO code_items ... WHERE g.code = 'SCORING_MODE'` | exact 항목 추가 | **0행 삽입** |
| `INSERT INTO code_items ... WHERE g.code = 'FEEDBACK_STYLE'` | 2개 항목 추가 | **0행 삽입** |
| `INSERT INTO code_items ... WHERE g.code = 'FEEDBACK_TYPE'` | 4개 항목 추가 | **0행 삽입** |

### 11-4. 수정 플랜 (Phase 5-fix)

| # | 작업 | SQL/파일 | 비고 |
|---|------|----------|------|
| fix-1 | 누락된 5개 `code_groups` 생성 | DB (raw SQL) | ANSWER_TYPE, SCORING_MODE, PASSAGE_EXPOSURE_MODE, FEEDBACK_STYLE, FEEDBACK_TYPE |
| fix-2 | 각 그룹에 `code_items` 시드 삽입 | DB (raw SQL) | 등록 페이지에서 하드코딩되어 있던 선택지 값들 |
| fix-3 | ANSWER_TYPE items에 `extraData` 보강 | DB (raw SQL) | scoringHint, description 추가 (5-2 재실행) |
| fix-4 | 동작 검증 | 브라우저 | code-management 저장 + 등록 페이지 선택지 표시 확인 |

#### fix-1: 누락 code_groups 생성

```sql
INSERT INTO "code_groups" ("code", "name", "description", "is_active")
VALUES
  ('ANSWER_TYPE',           'Answer Type',           '답변 유형 (essay, option 등)', true),
  ('SCORING_MODE',          'Scoring Mode',          '채점 방식 (exact, holistic, pronunciation)', true),
  ('PASSAGE_EXPOSURE_MODE', 'Passage Exposure Mode', '지문 표시 방식 (hidden, preview, full)', true),
  ('FEEDBACK_STYLE',        'Feedback Style',        'AI 피드백 방식 (correct-wrong, strengths-weaknesses)', true),
  ('FEEDBACK_TYPE',         'Feedback Type',         '피드백 유형 (correctness, holistic, pronunciation, writing)', true)
ON CONFLICT ("code") DO NOTHING;
```

#### fix-2: code_items 시드 삽입

**ANSWER_TYPE**:
```sql
INSERT INTO "code_items" ("group_id", "code", "name", "sort_order", "extra_data", "is_active")
SELECT g.id, v.code, v.name, v.sort_order, v.extra_data::jsonb, true
FROM (VALUES
  ('multiple-choice', '객관식 (다지선다)',      1, '{"description":"객관식 다지선다","scoringHint":"accuracy"}'),
  ('essay',           '주관식 서술형',          2, '{"description":"주관식 서술형","scoringHint":"holistic"}'),
  ('short-text',      '단답형',              3, '{"description":"단답형","scoringHint":"accuracy"}'),
  ('audio-record',    '음성 녹음 (쉐도잉)',     4, '{"description":"음성 녹음 (쉐도잉)","scoringHint":"pronunciation"}'),
  ('sentence-write',  '영문 문장 작성',        5, '{"description":"영문 문장 작성","scoringHint":"holistic"}'),
  ('option',          '객관식',              6, '{"description":"객관식","scoringHint":"accuracy"}')
) AS v(code, name, sort_order, extra_data)
CROSS JOIN code_groups g WHERE g.code = 'ANSWER_TYPE'
ON CONFLICT ("group_id", "code") DO NOTHING;
```

**SCORING_MODE**:
```sql
INSERT INTO "code_items" ("group_id", "code", "name", "sort_order", "is_active")
SELECT g.id, v.code, v.name, v.sort_order, true
FROM (VALUES
  ('exact',         '정오답 채점',     1),
  ('holistic',      'AI 총체 평가',    2),
  ('pronunciation', '발음 점수 평균',  3)
) AS v(code, name, sort_order)
CROSS JOIN code_groups g WHERE g.code = 'SCORING_MODE'
ON CONFLICT ("group_id", "code") DO NOTHING;
```

**PASSAGE_EXPOSURE_MODE**:
```sql
INSERT INTO "code_items" ("group_id", "code", "name", "sort_order", "is_active")
SELECT g.id, v.code, v.name, v.sort_order, true
FROM (VALUES
  ('hidden',  '숨김',       1),
  ('preview', '미리보기',    2),
  ('full',    '전체 표시',   3)
) AS v(code, name, sort_order)
CROSS JOIN code_groups g WHERE g.code = 'PASSAGE_EXPOSURE_MODE'
ON CONFLICT ("group_id", "code") DO NOTHING;
```

**FEEDBACK_STYLE**:
```sql
INSERT INTO "code_items" ("group_id", "code", "name", "sort_order", "is_active")
SELECT g.id, v.code, v.name, v.sort_order, true
FROM (VALUES
  ('correct-wrong',        '정답/오답 피드백', 1),
  ('strengths-weaknesses', '강점/약점 분석',  2)
) AS v(code, name, sort_order)
CROSS JOIN code_groups g WHERE g.code = 'FEEDBACK_STYLE'
ON CONFLICT ("group_id", "code") DO NOTHING;
```

**FEEDBACK_TYPE**:
```sql
INSERT INTO "code_items" ("group_id", "code", "name", "sort_order", "is_active")
SELECT g.id, v.code, v.name, v.sort_order, true
FROM (VALUES
  ('correctness',   '정오답 피드백',  1),
  ('holistic',      '총체적 피드백',  2),
  ('pronunciation', '발음 피드백',   3),
  ('writing',       '작문 피드백',   4)
) AS v(code, name, sort_order)
CROSS JOIN code_groups g WHERE g.code = 'FEEDBACK_TYPE'
ON CONFLICT ("group_id", "code") DO NOTHING;
```

### 11-5. 근본 원인 및 재발 방지

| 원인 | 설명 |
|---|---|
| 기존 code-management 가정 오류 | 10-2에서 "이미 code-management 탭에 존재하는 그룹 6개"로 기술했으나, **프론트 탭 설정만 존재**하고 DB에는 code_groups 레코드가 없었음 |
| `findByGroupCode` 설계 | 그룹 없으면 빈 배열 반환 → 에러가 조용히 삼켜져서 누락 발견이 지연됨 |
| Phase 5-2 SQL 무효 | CROSS JOIN이 0행 → INSERT 0행이지만 에러 없이 "성공" 표시 |

**재발 방지**: 시드 SQL 실행 후 반드시 `SELECT count(*) FROM code_items WHERE group_id = (SELECT id FROM code_groups WHERE code = 'XXX')` 로 삽입 결과 검증

---

## 12. 성과KPI 탭에 "측정 도구" 컬럼 추가

> 작성일: 2026-04-01
> 대상: code-management 페이지 → 성과KPI 탭

### 12-1. 현재 상태

**성과KPI 탭 컬럼 (현재 5개)**:

| 컬럼 key | label | 데이터 위치 | 비고 |
|---|---|---|---|
| `code` | 코드 | `code_items.code` | 기존 항목 readOnly |
| `extraData.goal` | 목표 | `code_items.extra_data.goal` | |
| `name` | 측정 항목 | `code_items.name` | |
| `extraData.measureUnit` | 측정 유닛 | `code_items.extra_data.measureUnit` | |
| `extraData.measureMethod` | 측정 방법 | `code_items.extra_data.measureMethod` | |

**현재 DB 데이터** (KPI_CATEGORY, 2개 항목):
```json
// Fluency
{ "goal": "유창성", "measureUnit": "WPM", "measureMethod": "STT", "automation": "STT" }

// Prediction
{ "goal": "예측 타당성 – 이해 전략", "measureUnit": "%", "measureMethod": "LLM", "automation": "LLM" }
```

> 기존에 `automation` 필드가 이미 존재하지만 UI에서 표시/편집 불가 상태

### 12-2. 변경 계획

**성과KPI 탭 컬럼 (변경 후 6개)**:

| 컬럼 key | label | type | width | 변경 |
|---|---|---|---|---|
| `code` | 코드 | text | 12% | 기존 유지 (readOnly) |
| `extraData.goal` | 목표 | text | 18% | 기존 유지 (width 축소) |
| `name` | 측정 항목 | text | 16% | 기존 유지 (width 축소) |
| `extraData.measureUnit` | 측정 유닛 | text | 12% | 기존 유지 (width 축소) |
| `extraData.measureMethod` | 측정 방법 | text | 18% | 기존 유지 (width 축소) |
| **`extraData.measureTool`** | **측정 도구** | **text** | **14%** | **신규 추가** |

### 12-3. 변경 파일 및 범위

| 파일 | 변경 | 사이드이펙트 |
|---|---|---|
| `code-management/page.tsx` | KPI_CATEGORY columns 배열에 `extraData.measureTool` 컬럼 추가 + `newItemDefaults`에 `measureTool: ''` 추가 | 없음 — `CodeGroupTable`이 `ColumnDef[]`을 그대로 렌더링하므로 컬럼 추가만으로 동작 |

### 12-4. 구현 상세

**변경 전**:
```typescript
groupCode: 'KPI_CATEGORY',
title: '성과KPI 분류',
columns: [
  { key: 'code',                    label: '코드',     type: 'text', width: '15%', readOnlyOnExisting: true },
  { key: 'extraData.goal',          label: '목표',     type: 'text', width: '20%', placeholder: '...' },
  { key: 'name',                    label: '측정 항목', type: 'text', width: '18%', placeholder: '...' },
  { key: 'extraData.measureUnit',   label: '측정 유닛', type: 'text', width: '15%', placeholder: '...' },
  { key: 'extraData.measureMethod', label: '측정 방법', type: 'text', width: '20%', placeholder: '...' },
],
newItemDefaults: { extraData: { goal: '', measureUnit: '', measureMethod: '' } },
```

**변경 후**:
```typescript
groupCode: 'KPI_CATEGORY',
title: '성과KPI 분류',
columns: [
  { key: 'code',                    label: '코드',     type: 'text', width: '12%', readOnlyOnExisting: true },
  { key: 'extraData.goal',          label: '목표',     type: 'text', width: '18%', placeholder: '...' },
  { key: 'name',                    label: '측정 항목', type: 'text', width: '16%', placeholder: '...' },
  { key: 'extraData.measureUnit',   label: '측정 유닛', type: 'text', width: '12%', placeholder: '...' },
  { key: 'extraData.measureMethod', label: '측정 방법', type: 'text', width: '18%', placeholder: '...' },
  { key: 'extraData.measureTool',   label: '측정 도구', type: 'text', width: '14%', placeholder: 'STT Engine, LLM 등' },
],
newItemDefaults: { extraData: { goal: '', measureUnit: '', measureMethod: '', measureTool: '' } },
```

### 12-5. 사이드이펙트 분석

| 영향 범위 | 분석 | 결론 |
|---|---|---|
| `CodeGroupTable` 컴포넌트 | columns 배열만 참조하여 동적 렌더링. 변경 불필요 | 영향 없음 |
| 기존 DB 데이터 | 기존 항목의 `extraData`에 `measureTool` 키가 없으면 빈 문자열로 표시 (`getValue` → `undefined` → `''`) | 영향 없음 |
| 저장 API | `saveCodeGroupItems`는 `extraData`를 JSONB 통째 전달. 새 키 추가 시 자연스럽게 저장 | 영향 없음 |
| 등록 페이지 KPI 모달 | `selectedKpiCodes`로 code만 참조. `extraData` 구조 변경에 영향 없음 | 영향 없음 |
| tutoring 앱 | KPI extraData를 직접 소비하는 경우 `measureTool` 키가 새로 추가되지만, 기존 키(`goal`, `measureUnit`, `measureMethod`)는 변경 없음 | 영향 없음 (추가만) |
| DB 마이그레이션 | JSONB 필드이므로 스키마 변경 불필요. 신규 항목 생성 시 자동 포함 | 불필요 |

---

*작성일: 2026-03-31*
*업데이트: 2026-04-01 (섹션 10 추가 — 코드관리 연동 플랜)*
*업데이트: 2026-04-01 (섹션 11 추가 — 누락 code_groups 이슈 및 수정 플랜)*
*업데이트: 2026-04-01 (섹션 12 추가 — 성과KPI 측정 도구 컬럼 추가)*
*상태: Phase 5-fix 완료, 섹션 12 구현 대기*
