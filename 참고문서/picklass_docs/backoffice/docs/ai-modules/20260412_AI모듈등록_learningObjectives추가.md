---
date: 2026-04-12
title: AI 모듈 등록 — learningObjectives 필드 추가 및 코드관리 패널타입 동기화
commits: 73aab6b (모듈등록업뎃), 54cc6fc (모듈코드관리추가_패널타입동기화)
prev_doc: ai-module-management-20260407.md
---

# AI 모듈 관리 — 2026-04-12 업데이트

> **이전 문서**: `ai-module-management-20260407.md` — Prisma 스키마 신규 필드 정의 완료, DTO 오류 해결 전  
> **이번 업데이트**: `learningObjectives` 필드 전 레이어 적용 완료 + 코드 관리 페이지 패널타입 탭 추가

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 모듈 등록/수정 흐름 (기존과 동일, learningObjectives 추가)

1. `/admin/ai-modules/register` 진입
   - `?id=` 파라미터 있음 → 수정 모드: `getAiModule(id)` 호출로 기존 데이터 로드 (`learningObjectives` 포함)
   - 파라미터 없음 → 신규 등록 모드 (초기값 `[]`)
2. 페이지 진입 시 `getAllCodes()` 호출 → `codeMap['LEARNING_OBJECTIVES']` 포함 전체 코드 그룹 로드
3. **[문항설계(QuestionData) 탭]** 하단에서 학습 목적 다중 선택
   - 항목 클릭 → 보라색 테두리 + 연보라 배경 표시 (선택됨)
   - 다시 클릭 → 선택 해제
4. [저장] → `createAiModule` 또는 `updateAiModule` 호출 시 `learningObjectives: string[]` 포함

### 1-2. 코드 관리 페이지 — 패널 타입 탭 추가 흐름

1. `/admin/ai-modules/code-management` 진입
2. **[UI / 패널 제어]** 탭 그룹 클릭
3. 하위 3개 탭 확인:
   - `UI_TEMPLATE`: UI 템플릿 코드 관리
   - `PASSAGE_MODE`: 지문 패널 동작 코드 관리 ⚠️ 새 값 추가 시 개발 작업 필요
   - `QUESTION_FLOW_MODE`: 문항 진행 방식 코드 관리 ⚠️ 새 값 추가 시 개발 작업 필요

---

## 2. IA 구조 및 기능 정의 (IA)

### 2-1. 모듈 등록/수정 탭 구성 (전체)

| 탭 key | 탭 명칭 | 주요 필드 | 변경 |
|--------|---------|-----------|------|
| `basicInfo` | 기본 정보 | skill, code, name | - |
| `courseSettings` | 커리큘럼 배치 설정 | classBefore/Middle/After, openBefore/After, priority, roles, suitableLevel, estimatedMinutes, prerequisites, incompatibleWith | - |
| `questionDataDesign` | 문항설계(QuestionData) | answerType, scoringMode, passageExposureMode, questionCount, questionGenerationStrategy, questionMin/MaxCount, feedbackStyle, feedbackType, **learningObjectives** (신규), questionFlowMode, passageDisplayMode | ✅ learningObjectives 추가 |
| `questionDesignV2` | 문항설계2 | hintTypes, retryScope, inputLanguage, passageRole, questionMaxAttempts | - |
| `kpiSettings` | 성과KPI설정 | selectedKpiCodes | - |
| `contentConfig` | AI 동작 설정 | purpose, pedagogyInstruction | - |

### 2-2. 코드 관리 탭 구성 (전체)

| 탭 그룹 | 탭 key | GroupCode | 설명 | 변경 |
|---------|--------|-----------|------|------|
| 문항/피드백 | `passage-exposure-mode` | PASSAGE_EXPOSURE_MODE | 지문 노출 방식 | - |
| | `scoring-mode` | SCORING_MODE | 채점 방식 | - |
| | `answer-type` | ANSWER_TYPE | 답변 유형 | - |
| | `feedback-style` | FEEDBACK_STYLE | 피드백 스타일 | - |
| | `feedback-type` | FEEDBACK_TYPE | 피드백 타입 | - |
| KPI | `kpi` | KPI_CATEGORY | 성과 KPI 정의 | - |
| **UI / 패널 제어** | `ui-template` | UI_TEMPLATE | UI 템플릿 코드 | ✅ 신규 탭 그룹 |
| | `passage-mode` | PASSAGE_MODE | 지문 패널 동작 | ✅ 신규 |
| | `question-flow-mode` | QUESTION_FLOW_MODE | 문항 진행 방식 | ✅ 신규 |

### 2-3. learningObjectives 선택지 (`LEARNING_OBJECTIVES` 코드 그룹, DB 관리)

총 22개 항목. 코드 관리 페이지에서 추가/수정/삭제 가능.

| code | name (저장값) |
|------|-------------|
| `lo_vocab_meaning` | 어휘 인지 및 의미 |
| `lo_pronunciation` | 발음 및 운율 |
| `lo_vocab_relations` | 어휘 관계 |
| `lo_comprehension` | 이해 전략 |
| `lo_critical_reading` | 비판적 읽기 |
| `lo_decoding_fluency` | 해독 및 유창성 |
| `lo_text_types` | 다양한 텍스트 유형 |
| `lo_discourse_org` | 문단 및 담화 조직 |
| `lo_realtime_processing` | 실시간 처리 |
| `lo_fluency` | 유창성 |
| `lo_sentence_construct` | 문장 구성 |
| `lo_writing_process` | 쓰기 과정 |
| `lo_genre_writing` | 장르별 쓰기 |
| `lo_sound_recognition` | 음성 인식 |
| `lo_listening_comp` | 청취 이해 |
| `lo_discourse_types` | 다양한 담화 유형 |
| `lo_utterance_volume` | 발화량 |
| `lo_interaction` | 상호작용 기술 |
| `lo_error_correction` | 오류 인식 및 교정 |
| `lo_morphosyntax` | 형태 및 통사 |
| `lo_contextual_grammar` | 맥락적 문법 |
| `lo_grammar_accuracy` | 문법 정확성 |

> ⚠️ **저장 방식**: DB에는 `code`가 아닌 `name`(한국어 문자열)으로 저장됨.  
> 선택/비교 로직: `item.name` 기준으로 `formData.learningObjectives.includes(item.name)` 사용.

---

## 3. 정책 (Policy / Business Rules)

### 3-1. learningObjectives 정책

| 규칙 | 내용 |
|------|------|
| 선택 수 | 0개 이상 (선택 필수 아님) |
| 저장 형식 | `String[]` — `name` 한국어 문자열 배열 |
| 빈 값 | `null` 아님, 빈 배열 `[]`로 저장 |
| 선택지 관리 | `LEARNING_OBJECTIVES` 코드 그룹 — 코드 관리 페이지에서 CRUD 가능 |
| 선택 UI | 토글 버튼 (선택됨: 보라색 테두리 + 연보라 배경) |

### 3-2. 패널 타입 코드 관리 정책 (신규)

| 규칙 | 내용 |
|------|------|
| `PASSAGE_MODE` 새 값 추가 시 | 개발팀 작업 필요 — 튜터링 렌더러에 패널 처리 로직 추가 |
| `QUESTION_FLOW_MODE` 새 값 추가 시 | 개발팀 작업 필요 — 문항 진행 흐름 로직 추가 |
| `UI_TEMPLATE` | 코드값과 렌더러 등록 키 일치 필요 |

### 3-3. 기존 정책 (변경 없음)

- 모듈 `code` 필드: 시스템 어댑터 등록 키와 반드시 일치 (자동 대문자 변환)
- `skill`: `['vocabulary', 'reading', 'speaking', 'writing']` 중 하나
- `priority`: 1~5
- `cognitiveLevel`: 빈값 또는 1 이상의 정수

---

## 4. 추가 개발 필요 사항

- [x] `learningObjectives` Prisma 스키마 컬럼 추가 (`prisma/migrations/20260412000000_add_learning_objectives`)
- [x] `@repo/types` DTO에 `learningObjectives?: string[]` 추가
- [x] `ai-module.service.ts` create/update/toResponse에 매핑 추가
- [x] 등록 페이지 UI — `LEARNING_OBJECTIVES` 코드 그룹 연동 토글 버튼
- [x] 코드 관리 페이지 — UI/패널 제어 탭 그룹 추가 (UI_TEMPLATE, PASSAGE_MODE, QUESTION_FLOW_MODE)
- [ ] **코드 관리 — `LEARNING_OBJECTIVES` 탭 추가**: 현재 코드 관리 페이지에서 learningObjectives 항목을 직접 관리하는 탭 없음 (별도 시스템 코드로만 관리 중)
- [ ] **수정 모드 `code` 필드 readonly 처리**: 어댑터 키 연결 관계로 수정 불가 처리 필요
- [ ] **`alert()` 토스트 교체**: 저장 성공/실패 알림을 sonner 토스트로 교체
- [ ] **탭별 유효성 검사 추가**: 현재 저장 버튼 클릭 시에만 검증
- [ ] **Studio `ai_modules` 스키마 동기화**: Studio의 `packages/shared/src/types/module.ts`에 `learning_objectives` 추가 필요 (오늘 별도 작업 완료됨 — Studio 쪽 docs 참고)

---

## 5. 코드 규칙 (Coding Rules)

### 사용해야 하는 공통 유틸/컴포넌트

| 항목 | 위치 | 용도 |
|------|------|------|
| `fetchApi` 기반 `getAiModule`, `createAiModule`, `updateAiModule` | `apps/admin/frontend/src/lib/api.ts` | 모든 모듈 API 호출 |
| `getAllCodes` | `apps/admin/frontend/src/lib/api/code.ts` | 전체 코드 그룹 로드 → `codeMap` 변환 |
| `getCodesByGroup` | 동일 | 단일 코드 그룹 조회 |
| `AiModuleResponse`, `CreateAiModuleDto`, `UpdateAiModuleDto` | `@repo/types` | 모든 AI 모듈 타입 |
| `CodeItemResponse` | `@repo/types` | 코드 항목 타입 |

### 금지 패턴

- `any` 타입, `@ts-ignore` 사용 금지
- 학습 목적 선택지 등 코드 그룹 데이터 **프론트 하드코딩 금지** → 반드시 `codeMap[그룹코드]` 사용
- `alert()` 직접 사용 금지 (현재는 임시 허용, 교체 예정)
- Nullable 외래키: 빈 문자열 → 반드시 `null`로 변환 후 API 전달
- `learningObjectives` 저장 시 `code`가 아닌 `name`(한국어) 기준 사용

### 파일 위치 규칙

| 레이어 | 위치 |
|--------|------|
| 타입 | `packages/types/src/index.ts` |
| 비즈니스 로직 | `packages/core/src/ai-module/ai-module.service.ts` |
| API 함수 | `apps/admin/frontend/src/lib/api.ts` |
| 페이지 | `apps/admin/frontend/src/app/(admin)/admin/ai-modules/` |
| 공통 컴포넌트 | `apps/admin/frontend/src/components/admin/module-*.tsx` |

### `packages/core` 수정 시 규칙

```
@repo/types 수정 → pnpm build (packages/types) →
@repo/core 수정 → pnpm build (packages/core) →
백엔드 재시작 → 프론트엔드 확인
```

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 항목 | 현황 | 심각도 | 개선 방향 |
|------|------|--------|-----------|
| ~~`learningObjectives` 하드코딩~~ | ✅ 해결 — `LEARNING_OBJECTIVES` 코드 그룹 DB 연동 완료 | - | - |
| ~~07.md의 8개 DTO 필드 누락~~ | ✅ 해결 — `@repo/types` 및 service 모두 추가 완료 | - | - |
| `alert()` 사용 | 저장 성공/실패, 유효성 검사 실패 | 낮음 | sonner 토스트 교체 |
| `code` 필드 수정 가능 | 수정 모드에서도 변경 가능 | 중간 | `disabled` 처리 필요 |
| 탭별 유효성 검사 없음 | 저장 클릭 시에만 검증 | 낮음 | 탭 이탈 시 검사 추가 |
| `LEARNING_OBJECTIVES` 코드 그룹 관리 탭 없음 | 코드 관리 페이지에 탭 미추가 | 낮음 | 코드 관리 탭 추가 |
| `learningObjectives` 저장값이 `name`(한국어) | 코드 재구성 시 데이터 마이그레이션 필요 | 중간 | 장기적으로 `code`값 저장으로 전환 검토 |
| 하드코딩 상수 | `module-basic-info.tsx`의 SKILLS, `module-level-settings.tsx`의 LEVELS(1~18) 등 | 낮음 | 공통 상수 파일로 추출 |

---

## 7. 컴포넌트 / 훅 의존성 (Dependencies)

### register/page.tsx 의존 항목

| 항목 | 종류 | 출처 |
|------|------|------|
| `getAiModule`, `createAiModule`, `updateAiModule` | API 함수 | `@/lib/api` |
| `getAllCodes` | API 함수 | `@/lib/api/code` |
| `AiModuleResponse`, `CreateAiModuleDto`, `UpdateAiModuleDto` | 타입 | `@repo/types` |
| `CodeItemResponse` | 타입 | `@repo/types` |
| `ModuleBasicInfo` | 컴포넌트 | `@/components/admin/module-basic-info` |
| `ModuleContentSettings` | 컴포넌트 | `@/components/admin/module-content-settings` |
| `ModuleLessonSettings` | 컴포넌트 | `@/components/admin/module-lesson-settings` |
| `ModuleLevelSettings` | 컴포넌트 | `@/components/admin/module-level-settings` |

### code-management/page.tsx 의존 항목

| 항목 | 종류 | 출처 |
|------|------|------|
| `getAllCodes`, `saveCodeGroupItems`, `deleteCodeItem` | API 함수 | `@/lib/api/code` |
| `CodeGroupTable` | 컴포넌트 | `@/components/common/editable-table` |
| `buildTabConfig()` | 내부 함수 | page.tsx 내부 |

### 진입 경로

- `/admin/ai-modules` 목록 → [+ 모듈 등록] → `register` (신규)
- `/admin/ai-modules` 목록 → [편집] → `register?id={moduleId}` (수정)
- 사이드바 메뉴 → [AI 모듈 관리] → [코드 관리]

### 이 페이지가 영향을 주는 외부 시스템

| 시스템 | 영향 |
|--------|------|
| **튜터링 서비스** (`tutoring.picklass.com`) | AI 모듈 설정값이 LLM 프롬프트 생성에 직접 사용 |
| **스튜디오** (`studio.picklass.com3`) | 수업 모듈 선택 시 `skill`, `code`, `name`, `learning_objectives` 참조 |
| **Studio `/modules` API** | `GET /modules` → `ai_modules` 테이블 직접 조회 (공유 DB) |
| **코드 관리 페이지** | `PASSAGE_MODE`, `QUESTION_FLOW_MODE`, `UI_TEMPLATE` 값 동기화 |

---

## 8. DB / API 구조 (Data Contract)

### API 엔드포인트 (변경 없음)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/ai-modules` | 목록 조회 (skill/status 필터, 기본 limit=20) |
| GET | `/ai-modules/:id` | 단건 조회 |
| POST | `/ai-modules` | 신규 등록 |
| PUT | `/ai-modules/:id` | 전체 수정 |
| PATCH | `/ai-modules/:id/status` | 상태 토글 (active ↔ inactive) |
| DELETE | `/ai-modules/:id` | 소프트 삭제 |

### Prisma 스키마 — 2026-04-12 추가 필드

```prisma
model AiModule {
  // ... 기존 필드 생략 ...

  // ── 성과 KPI (탭4) ──
  selectedKpiCodes    String[]  @default([]) @map("selected_kpi_codes")

  // ── 학습 목적 (탭3: 문항설계 하단) — 2026-04-12 추가 ──
  learningObjectives  String[]  @default([]) @map("learning_objectives")

  // ── AI 동작 설정 (탭5) ──
  purpose             String    @default("") @db.Text
  pedagogyInstruction String    @default("") @map("pedagogy_instruction") @db.Text
  // ...
}
```

### 마이그레이션 이력

| 날짜 | 파일 | 내용 |
|------|------|------|
| 2026-03-08 | `20260308080228_init_institution_user_tables` | 초기 테이블 생성 |
| 2026-04-12 | `20260412000000_add_learning_objectives` | `ai_modules` 테이블에 `learning_objectives text[]` 추가 |

### 타입 정의 (`@repo/types` — 2026-04-12 기준 전체)

```typescript
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
  questionGenerationStrategy: string;   // 'extract' | 'generate'
  questionMinCount: number;
  questionMaxCount: number;
  feedbackStyle: string;
  feedbackType: string;
  selectedKpiCodes: string[];
  learningObjectives: string[];          // ← 2026-04-12 추가
  purpose: string;
  pedagogyInstruction: string;
  contentGenerationInstruction: string;
  questionData: Record<string, unknown>[];
  contentConfig: Record<string, unknown>;
  hintTypes: string;
  retryScope: string;
  inputLanguage: string;
  passageRole: string;
  questionMaxAttempts: number | null;
  uiTemplate: string;
  passageMode: string;
  questionFlowMode: string;
  status: string;
  createdAt: string;
  updatedAt: string;
}

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
  questionGenerationStrategy?: string;
  questionMinCount?: number;
  questionMaxCount?: number;
  feedbackStyle?: string;
  feedbackType?: string;
  selectedKpiCodes?: string[];
  learningObjectives?: string[];         // ← 2026-04-12 추가
  purpose?: string;
  pedagogyInstruction?: string;
  contentGenerationInstruction?: string;
  questionData?: Record<string, unknown>[];
  contentConfig?: Record<string, unknown>;
  hintTypes?: string;
  retryScope?: string;
  inputLanguage?: string;
  passageRole?: string;
  questionMaxAttempts?: number | null;
  uiTemplate?: string;
  passageMode?: string;
  questionFlowMode?: string;
  status?: string;
}

export type UpdateAiModuleDto = Partial<CreateAiModuleDto>;
```

### ai-module.service.ts 핵심 로직 (learningObjectives 관련)

```typescript
// create()
learningObjectives: dto.learningObjectives ?? []

// update()
if (dto.learningObjectives !== undefined)
  updateData.learningObjectives = dto.learningObjectives;

// toResponse()
learningObjectives: module.learningObjectives,
```

---

**작성일**: 2026-04-12  
**상태**: ✅ 완료 — learningObjectives 전 레이어 적용, 패널타입 코드 관리 탭 추가
