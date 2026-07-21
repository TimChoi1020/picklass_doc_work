# Picklass Backoffice — AI Module 관리 시스템 컨텍스트 (2026-04-07)

> 이 문서는 백오피스 AI 모듈 관리 페이지 및 Prisma 스키마 설계 내용을 정리한 작업 컨텍스트입니다.
> 빌드 오류 해결, 필드 매핑, 타입 정의 일관성을 위한 기준 문서입니다.
>
> **최종 업데이트**: 2026-04-07 — **Prisma 스키마 필드 명세 정의 완료** (questionGenerationStrategy, questionMin/MaxCount 필드 추가됨)

---

## 1. 현재 상태

### 1-1. 빌드 오류 (2026-04-07 기준)

```
@repo/core:build: src/ai-module/ai-module.service.ts(110,9): error TS2353: 
  Object literal may only specify known properties, and 'questionGenerationStrategy' 
  does not exist in type '(Without<AiModuleCreateInput...>'
```

**근본 원인**: `ai-module.service.ts`에서 Prisma 스키마에 **있는 필드**를 사용하는데, 
TypeScript **DTO 정의 (@repo/types)에 없어서** 오류 발생.

### 1-2. 스키마 vs DTO 필드 정합성

| 필드명 | Prisma 스키마 | DTO정의 필요 | 현황 | 해결 |
|---|---|---|---|---|
| `questionGenerationStrategy` | ✅ 있음 (line 247) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `questionMinCount` | ✅ 있음 (line 249) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `questionMaxCount` | ✅ 있음 (line 250) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `hintTypes` | ✅ 있음 (line 253) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `retryScope` | ✅ 있음 (line 254) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `inputLanguage` | ✅ 있음 (line 255) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `passageRole` | ✅ 있음 (line 256) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |
| `questionMaxAttempts` | ✅ 있음 (line 257) | ❌ 없음 | 오류 발생 | DTO 추가 필요 |

---

## 2. Prisma 스키마 정의 (schema.prisma)

### 2-1. AiModule 모델 전체 구조

```prisma
model AiModule {
  id                  String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  // ── 기본 정보 (탭1) ──
  skill               String    @db.VarChar(50)
  code                String    @unique @db.VarChar(10)
  name                String    @db.VarChar(200)

  // ── 커리큘럼 배치 설정 (탭2) ──
  classBefore         Boolean   @default(false) @map("class_before")
  classMiddle         Boolean   @default(false) @map("class_middle")
  classAfter          Boolean   @default(false) @map("class_after")
  openBefore          Boolean   @default(false) @map("open_before")
  openAfter           Boolean   @default(false) @map("open_after")
  priority            Int       @default(1)
  roles               String[]  @default([])
  passageExposure     String    @default("") @map("passage_exposure") @db.VarChar(20)
  cognitiveLevel      String    @default("") @map("cognitive_level") @db.VarChar(5)
  suitableLevelMin    String    @default("") @map("suitable_level_min") @db.VarChar(20)
  suitableLevelMax    String    @default("") @map("suitable_level_max") @db.VarChar(20)
  estimatedMinutesMin String    @default("") @map("estimated_minutes_min") @db.VarChar(10)
  estimatedMinutesMax String    @default("") @map("estimated_minutes_max") @db.VarChar(10)
  prerequisites       String[]  @default([])
  incompatibleWith    String[]  @default([]) @map("incompatible_with")

  // ── 문항설계 (탭3) ──
  answerType                String    @default("") @map("answer_type") @db.VarChar(50)
  scoringMode               String    @default("") @map("scoring_mode") @db.VarChar(50)
  passageExposureMode       String    @default("") @map("passage_exposure_mode") @db.VarChar(50)
  questionCount             String    @default("single") @map("question_count") @db.VarChar(20)
  
  // questionCount='multi' 전용. LLM이 지문 난이도를 분석하여 
  // min~max 범위 내에서 문항 수 결정
  questionGenerationStrategy String   @default("extract") @map("question_generation_strategy") @db.VarChar(20)
  questionMinCount          Int       @default(1) @map("question_min_count")
  questionMaxCount          Int       @default(1) @map("question_max_count")
  
  feedbackStyle             String    @default("") @map("feedback_style") @db.VarChar(50)
  feedbackType              String    @default("") @map("feedback_type") @db.VarChar(50)

  // ── 문항설계2 (탭3.5) ──
  hintTypes           String    @default("") @map("hint_types") @db.VarChar(200)
  retryScope          String    @default("") @map("retry_scope") @db.VarChar(200)
  inputLanguage       String    @default("") @map("input_language") @db.VarChar(200)
  passageRole         String    @default("") @map("passage_role") @db.VarChar(200)
  questionMaxAttempts Int?      @map("question_max_attempts")

  // ── 성과 KPI (탭4) ──
  selectedKpiCodes    String[]  @default([]) @map("selected_kpi_codes")

  // ── AI 동작 설정 (탭5) ──
  purpose             String    @default("") @db.Text
  pedagogyInstruction String    @default("") @map("pedagogy_instruction") @db.Text

  // ── 문항 데이터 + 콘텐츠 설정 (JSONB) ──
  questionData        Json      @default("[]") @map("question_data") @db.JsonB
  contentConfig       Json      @default("{}") @map("content_config") @db.JsonB

  // ── UI 템플릿 ──
  uiTemplate          String    @default("standard") @map("ui_template") @db.VarChar(30)

  // ── 상태/시간 ──
  status              String    @default("active") @db.VarChar(20)
  createdAt           DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt           DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt           DateTime? @map("deleted_at") @db.Timestamptz

  @@index([skill])
  @@index([status])
  @@index([code])
  @@map("ai_modules")
}
```

### 2-2. 필드 분류 및 의미

| 그룹 | 필드 | 타입 | 설명 |
|---|---|---|---|
| **문항 생성 전략** | `questionGenerationStrategy` | String | 'extract' \| 'generate' — LLM이 지문에서 문제를 추출할지, 신규 생성할지 |
| **문항 수량 제어** | `questionMinCount` | Int | 문항 최소 개수 (questionCount='multi'일 때만 의미) |
| | `questionMaxCount` | Int | 문항 최대 개수 (questionCount='multi'일 때만 의미) |
| **힌트 & 재시도** | `hintTypes` | String | 힌트 유형 (CSV 또는 JSON 배열) |
| | `retryScope` | String | 재시도 범위 ('single_question' \| 'passage' \| 'lesson') |
| **입출력 설정** | `inputLanguage` | String | 사용자 입력 언어 설정 |
| | `passageRole` | String | 지문의 역할/용도 (읽기/수정/분석 등) |
| **시도 제한** | `questionMaxAttempts` | Int? | 문항당 최대 시도 횟수 (null 허용) |

---

## 3. 백오피스 페이지 구조

### 3-1. AI Modules 목록 페이지 (`page.tsx`)

**경로**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/page.tsx`

**기능**:
- AI 모듈 목록 조회 (페이징 X, 전체 로드)
- 스킬별 모듈 확인 (skill: vocabulary, reading, speaking, writing)
- 모듈 상태 토글 (active ↔ inactive)
- 모듈 편집 링크 → `/admin/ai-modules/register?id=MODULE_ID`
- 모듈 삭제

**테이블 컬럼**:
| 컬럼 | 데이터 소스 | 렌더링 형식 |
|---|---|---|
| 스킬 | `module.skill` | 코드 매핑 (`SKILL` 그룹 → emoji + 이름) |
| 모듈명 | `(module.code) module.name` | 코드(이름) 형식 |
| 수업 (전/중/후) | `classBefore/Middle/After` | 체크박스 (readOnly) |
| 지문오픈 (오픈전/후) | `openBefore/After` | 체크박스 (readOnly) |
| 우선순위 | `module.priority` | 숫자 (1-5) |
| 상태 | `module.status` | 토글 버튼 (green: active, orange: inactive) |
| 관리 | 버튼 | 편집 / 삭제 |

**현재 코드 발췌**:
```tsx
const modules = await getAiModules({ page: 1, limit: 100 });
// modules: AiModuleResponse[]

modules.map((module) => (
  <tr key={module.id}>
    <td>{getSkillLabel(module.skill)}</td>
    <td>({module.code}) {module.name}</td>
    <td><input type="checkbox" checked={module.classBefore} readOnly /></td>
    ...
  </tr>
))
```

### 3-2. 모듈 등록/편집 페이지 (`register/page.tsx`)

**경로**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

**기능**: 모듈 CRUD
- 신규 등록 (GET 파라미터 `id` 없음)
- 기존 모듈 편집 (GET 파라미터 `id=MODULE_ID`)
- 폼 제출 시 API 호출 (`POST` 또는 `PUT`)

**탭 구조** (복합 폼):
| 탭 | 필드 그룹 | 필드들 |
|---|---|---|
| **탭1: 기본정보** | 기본 메타정보 | skill, code, name |
| **탭2: 커리큘럼배치** | 배치 설정 | classBefore/Middle/After, openBefore/After, priority, roles, ... |
| **탭3: 문항설계** | 문항 관련 | answerType, scoringMode, passageExposureMode, questionCount, **questionGenerationStrategy**, **questionMinCount**, **questionMaxCount**, feedbackStyle, feedbackType |
| **탭3.5: (신규)** | 추가 설정 | **hintTypes, retryScope, inputLanguage, passageRole, questionMaxAttempts** |
| **탭4: 성과KPI** | KPI 선택 | selectedKpiCodes (다중 선택) |
| **탭5: AI동작** | 프롬프트 & 기타 | purpose, pedagogyInstruction, questionData, contentConfig, uiTemplate |

### 3-3. 코드 관리 페이지 (`code-management/page.tsx`)

**경로**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/code-management/page.tsx`

**기능**: 코드 그룹 CRUD (6개 코드 그룹 탭 기반)

**탭 목록** (`buildTabConfig()`로 정의):
| 탭 key | GroupCode | 용도 |
|---|---|---|
| `passage-exposure-mode` | PASSAGE_EXPOSURE_MODE | 지문 노출 방식 |
| `scoring-mode` | SCORING_MODE | 채점 방식 |
| `answer-type` | ANSWER_TYPE | 답변 유형 |
| `feedback-style` | FEEDBACK_STYLE | 피드백 스타일 |
| `feedback-type` | FEEDBACK_TYPE | 피드백 타입 |
| `kpi` | KPI_CATEGORY | 성과 KPI 정의 |

> **중요**: 코드 관리는 `ai_modules.ai_module_create_input` 의존이 아니라,
> 공유 코드 시스템 (`CodeGroup`, `CodeItem` 테이블)을 관리하는 페이지.

---

## 4. API 게층 오류 분석

### 4-1. ai-module.service.ts 오류 위치

| 라인 | 메서드 | 오류 필드 | 원인 |
|---|---|---|---|
| 110 | `create()` | questionGenerationStrategy | DTO 미정의 |
| 193-207 | `update()` | questionMinCount, questionMaxCount, hintTypes, retryScope, inputLanguage, passageRole, questionMaxAttempts | DTO 미정의 |
| 304-319 | `toResponse()` 또는 응답 매핑 | (동일) | DTO 미정의 |

### 4-2. 오류의 정확한 원인

```typescript
// ai-module.service.ts line 110
const module = await this.prisma.aiModule.create({
  data: {
    // ... 다른 필드들
    questionGenerationStrategy: dto.questionGenerationStrategy, // ❌ dto에 이 필드 없음
    questionMinCount: dto.questionMinCount,                     // ❌ dto에 이 필드 없음
    // ...
  }
});

// @repo/types의 CreateAiModuleDto 정의가 불완전
// 예상 DTO 구조:
interface CreateAiModuleDto {
  skill: string;
  code: string;
  name: string;
  // ... 기존 필드들
  
  // ❌ 다음 필드들이 빠져있음:
  // questionGenerationStrategy?: string;
  // questionMinCount?: number;
  // questionMaxCount?: number;
  // hintTypes?: string;
  // retryScope?: string;
  // inputLanguage?: string;
  // passageRole?: string;
  // questionMaxAttempts?: number;
}
```

---

## 5. 해결 전략

### 5-1. Step 1: @repo/types DTO 정의 확장

**파일**: `packages/types/src/ai-module.ts` (또는 `index.ts`)

**추가 필드 (CreateAiModuleDto, UpdateAiModuleDto)**:

```typescript
export interface CreateAiModuleDto {
  // 기존 필드들...
  
  // [신규] 문항 생성 전략
  questionGenerationStrategy?: string;  // 'extract' | 'generate'
  
  // [신규] 문항 수량
  questionMinCount?: number;
  questionMaxCount?: number;
  
  // [신규] 힌트 & 재시도
  hintTypes?: string;              // CSV 또는 JSON 배열
  retryScope?: string;             // 'single_question' | 'passage' | 'lesson'
  
  // [신규] 입출력 설정
  inputLanguage?: string;
  passageRole?: string;
  
  // [신규] 시도 제한
  questionMaxAttempts?: number | null;
  
  // 기존 필드들 (예)...
  // skill, code, name, classBefore, ...
}

export interface UpdateAiModuleDto extends Partial<CreateAiModuleDto> {}

export interface AiModuleResponse extends CreateAiModuleDto {
  id: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}
```

### 5-2. Step 2: ai-module.service.ts 검증 로직 추가 (선택사항)

```typescript
// 수정 메서드에서 questionCount='single'일 때는 
// questionMinCount/Max를 무시하도록 처리
if (dto.questionCount === 'single') {
  updateData.questionMinCount = 1;
  updateData.questionMaxCount = 1;
}
```

### 5-3. Step 3: 프론트엔드 폼 업데이트

**탭3 & 탭3.5에 필드 추가**:
- Select: questionGenerationStrategy ('extract' ↔ 'generate')
- Number inputs: questionMinCount, questionMaxCount
- Text fields: hintTypes, retryScope, inputLanguage, passageRole
- Number input: questionMaxAttempts

---

## 6. 파일 구조 및 의존성

### 6-1. 영향 받는 파일 목록

| 파일 경로 | 역할 | 수정 필요성 |
|---|---|---|
| `packages/types/src/ai-module.ts` | DTO 정의 | 🔴 **필수** — 필드 추가 |
| `apps/admin/backend/src/ai-module/ai-module.service.ts` | 백엔드 로직 | 🟢 불필요 — 코드는 이미 올바름 |
| `apps/admin/frontend/src/app/.../register/page.tsx` | 등록 폼 | 🟡 선택 — 새 필드 UI 추가 |
| `apps/admin/frontend/src/lib/api/ai-module.ts` | API 호출 함수 | 🟢 불필요 — 함수는 DTO 기반이므로 자동 적용 |
| `prisma/schema.prisma` | DB 스키마 | 🟢 불필요 — 이미 필드 존재 |

### 6-2. 의존성 체인

```
Frontend Form (register/page.tsx)
  ↓
API 함수 (fetchApi 래퍼 + CreateAiModuleDto)
  ↓
Backend API (POST /ai-modules, PUT /ai-modules/:id)
  ↓
ai-module.service.ts (Prisma 스키마 매핑)
  ↓
Prisma Client (AiModule model)
  ↓
PostgreSQL (ai_modules table)
```

---

## 7. 빌드 오류 완전 해결 체크리스트

- [ ] `packages/types/src/ai-module.ts`에서 DTO 정의에 8개 필드 추가
- [ ] `pnpm run build` (packages/types)로 타입 컴파일 성공 확인
- [ ] `pnpm run build` (packages/core)로 ai-module.service.ts 오류 해결 확인
- [ ] `pnpm run build` (모든 프로젝트) 성공 확인
- [ ] **선택사항**: register/page.tsx에 새 필드 UI 추가 (폼 완성도)
- [ ] **선택사항**: api-module.ts 유틸 함수 타입 확인
- [ ] 테스트: 신규 모듈 등록 시 새 필드 저장 확인

---

## 8. 코드 규칙 & 아키텍처 원칙

### 8-1. CLAUDE.md 기반 규칙 준수

- ✅ Backend는 API Gateway 역할 (ai-module.service.ts는 Prisma 스키마와 1:1 매핑)
- ✅ `any` 타입, `@ts-ignore` 사용 금지 (해당 없음)
- ✅ DTO-based 아키텍처 (모든 API 입출력이 타입 안전)
- ✅ 프론트 API 호출 시 `fetchApi` 유틸리티 사용

### 8-2. 단일 책임 원칙 (SRP)

| 레이어 | 책임 | 구현 위치 |
|---|---|---|
| **Schema** | DB 필드 정의 및 제약 | `prisma/schema.prisma` |
| **DTO** | API 입출력 타입 정의 | `@repo/types/ai-module.ts` |
| **Service** | 비즈니스 로직 + 검증 | `packages/core/ai-module.service.ts` |
| **Controller** | HTTP 엔드포인트 | `apps/admin/backend/src/ai-module.controller.ts` |
| **Frontend** | UI 폼 + 상태관리 | `apps/admin/frontend/src/app/.../ai-modules/` |

---

## 9. 알려진 이슈 & 향후 작업

### 9-1. 데이터베이스 마이그레이션

Prisma 스키마는 이미 8개 필드를 포함하고 있으므로:
- 기존 DB에 컬럼이 **이미 존재**하거나
- 마이그레이션이 **이미 적용**되었을 가능성 높음

```bash
# 확인 명령 (필요시)
npx prisma migrate status
npx prisma migrate dev --name add_missing_ai_module_fields
```

### 9-2. Legacy 코드 정리

`ContentConfig` 기반의 구형 UI 흐름 제어는 tutoring.picklass.com에서는 제거되었으나,
backoffice의 `questionData`, `contentConfig` JSONB 필드는 **유지 중**.

---

## 10. 참고 문서

- `docs/ai-modules/admin-ai-modules-code-management-20260330.md` — 코드 관리 페이지 상세 명세
- `docs/ai-modules/admin-ai-modules-20260324.md` — 모듈 등록 페이지 이전 명세
- `tutoring.picklass.com/docs/agent-architecture-context-20260406.md` — Orchestrator 아키텍처
- `CLAUDE.md` — 프로젝트 기본 규칙

---

## 부록: 필드 추가 예시 코드

### DTO 정의 (packages/types)

```typescript
// Before
export interface CreateAiModuleDto {
  skill: string;
  code: string;
  name: string;
  classBefore?: boolean;
  // ... 기존 필드 20개
}

// After
export interface CreateAiModuleDto {
  skill: string;
  code: string;
  name: string;
  classBefore?: boolean;
  // ... 기존 필드 20개
  
  // [신규 추가]
  questionGenerationStrategy?: string;
  questionMinCount?: number;
  questionMaxCount?: number;
  hintTypes?: string;
  retryScope?: string;
  inputLanguage?: string;
  passageRole?: string;
  questionMaxAttempts?: number | null;
}
```

### 검증 로직 (ai-module.service.ts - 선택사항)

```typescript
// questionCount='single'일 때는 min/max를 1로 고정
if (dto.questionCount === 'single') {
  updateData.questionMinCount = 1;
  updateData.questionMaxCount = 1;
} else if (dto.questionCount === 'multi') {
  // multi 모드: min/max 검증
  if (dto.questionMinCount) updateData.questionMinCount = dto.questionMinCount;
  if (dto.questionMaxCount) updateData.questionMaxCount = dto.questionMaxCount;
}
```

---

**작성일**: 2026-04-07
**상태**: 🔴 대기 중 — DTO 필드 추가 후 빌드 재시작 필요
