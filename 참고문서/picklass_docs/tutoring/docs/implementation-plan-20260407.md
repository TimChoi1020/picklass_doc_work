# 구현 플랜 — tutoring.picklass.com

**작성일**: 2026-04-07  
**기준 문서**:
- `docs/agent-architecture-context-20260406.md` (2026-04-07 최종 업데이트) — 정본
- `docs/dev-plan-20260405.md` — R4 설계 원칙, DB 설계, API 구현 계획
- `docs/login-accesscode-20260406.md` — 로그인/회원가입/액세스코드 기능 현황
- `docs/picklass-tutoring-planning.md` — 기획서 v1.0
- `docs/modules-lessonId-20260325.md` — 모듈 실행 페이지 상세

---

## 1. 현재 프로젝트 상태

### 1-1. 완료된 작업 (코드에 반영됨)

| 영역 | 상태 | 상세 |
|------|------|------|
| PedagogyProfile 재설계 7단계 | ✅ | ContentConfig UI 흐름 의존성 전면 제거. `OrchestratorContext.pedagogyProfile` 교체 완료 |
| Orchestrator 자율화 Phase 1-0~1-4 | ✅ | passageExposureMode, questionMaxAttempts, hintTypes, retryThreshold, purpose, pedagogyInstruction 6개 필드 활성 사용 |
| uiTemplate 5종 switch 분기 | ✅ | LessonSession.tsx switch 완성. VocabDeckFlow, InteractiveFlow 신규 구현 |
| PedagogyProfile 13개 모듈 | ✅ | `picklass_pedagogy_profiles.ts` 전체 정의 완료 (SHR→RRD 통합) |
| LLM 문항 생성 아키텍처 Step 1 | ✅ | `questionGenerationStrategy`, `questionMinCount`, `questionMaxCount` 타입/스키마 추가 (11개 파일) |
| LLM 문항 생성 아키텍처 Step 2 | ✅ | 백오피스 등록/수정 UI에 문항 생성 전략 필드 추가 |
| 백엔드 AI 서비스 | ✅ | Gemini 2.5 Flash (7개 메서드) + Azure Speech SDK (발음 평가 + TTS) |
| 프론트→백엔드 AI 연동 | ✅ | 7/9 API 호출 연동 완료 (feedback, writing-eval, hint, chat, kpi, pronunciation-eval, tts) |
| DB 테이블 | ✅ | module_questions, module_histories, lesson_results 생성 완료 |
| 백오피스 AI Module CRUD | ✅ | 전체 구현 완료. uiTemplate, 문항 생성 전략 필드 포함 |
| 파일 정리 | ✅ | useStageStateMachine.ts, usePanelRouter.ts 삭제. PanelAssignment 제거 |

### 1-2. 부분 완료 (추가 작업 필요)

| 영역 | 상태 | 미완료 사항 |
|------|------|-----------|
| ModulePedagogyProfile enum 확장 | ⚠️ | scoringMode/answerType/passageExposureMode 타입에 확장 값 일부 반영됨, `interventionDelaySec` 미추가 |
| RetryScope/HintType/PassageRole 타입 | ⚠️ | profiles.ts 값에 맞게 확장 완료 (빌드 오류 수정), 일부 값은 향후 추가 가능 |
| contentConfig | ⚠️ | UI 흐름 제거 완료. 타입 정의(`ContentConfig`, `StageItem`)와 `AiModuleData.contentConfig` 필드는 DB 스키마 목적으로 유지 |
| Legacy 어댑터 5개 | ✅ | 5개 파일 삭제 완료 (2026-04-07). GenericAdapter 전환 완료 |

### 1-3. 미구현

| 영역 | 상태 | 비고 |
|------|------|------|
| LLM 문항 생성 Step 3~5 | ✅ | 문항 생성 API + Cache-Aside + Legacy 어댑터 제거 완료 (2026-04-07) |
| Orchestrator Phase 1-5 | ✅ | pedagogyInstruction 기반 Gemini 피드백 연동 완료. greeting 동적 생성. 503 fallback (2026-04-07) |
| 인증 시스템 | ✅ | 자체 JWT (signup/login/me). AuthProvider + useAuth. MOCK_USER 제거 (2026-04-07) |
| 액세스코드 백엔드 | ✅ | `POST /lessons/register-accesscode` JWT 기반 user_id 바인딩 (2026-04-07) |
| lessonService Mock | ✅ | API 우선 패턴 정리 완료 (2026-04-07) |
| 모듈 이력 저장 | ✅ | `module_histories` 실제 DB 저장 (score/kpis/answers 포함) (2026-04-07) |
| resolveDefaultUiTemplate() | ✅ | 제거 완료. DB uiTemplate 값만 사용 (2026-04-07) |
| ProcessWritingFlow 범용화 | ❌ | 4단계 하드코딩 (추후 개발) |
| 신규 answerType 패널 | ❌ | click, mcq-with-evidence 등 QuestionsPanel 확장 (추후 개발) |
| 나만의 수업 전체 흐름 | ✅ | 지문 생성 API + 레슨 생성 + 셋업 페이지 구현 완료 (2026-04-07) |
| CurriculumPlanner 프론트 연동 | ✅ | create-custom에서 analyze-passage + generate-lesson 호출 (2026-04-07) |
| 리포트 페이지 (`/report`) | ❌ | 추후 개발 (Phase 7) |
| lesson_results 저장 | ✅ | `POST /lessons/:id/complete` 구현 완료 (2026-04-08). 레슨 완료 시 자동 호출 |
| student_learning_records 갱신 | ❌ | 모듈 완료 시 학습 기록 집계 테이블 업데이트 로직 (추후 개발) |
| Replan 메커니즘 | ✅ | signalReplan 도구 이미 구현 완료 (Orchestrator 규칙 + useModuleOrchestrator 처리) |
| passageExposureMode 렌더링 | ✅ | highlight/blurred/sentence-by-sentence/minimized ContentPanel 구현 완료 (2026-04-07) |
| Highlights (어휘 강조) | ✅ | focusVocab 기반 단어 강조 렌더링 구현 완료 (2026-04-07) |
| PRD 전체 지문 공개 규칙 | ✅ | ruleRevealFullPassageAfterFeedback 이미 구현됨 확인 |
| 레슨 완료 화면 | ✅ | 평균 정답률 + 모듈별 성과 + KPI 바 차트 + "홈으로" 버튼 (2026-04-07) |
| 모듈 성과카드 | ✅ | 레슨 완료 화면에 모듈별 KPI 바 차트 포함 (2026-04-07) |
| Engagement/Disengage 체크 | ✅ | 이미 구현됨 확인 (120초/180초 타이머 + 규칙) |
| ModuleProgressBar | ⚠️ | 2개 이상 모듈 레슨에서 진행 표시 (코드 존재, 동작 확인 필요) |
| 피드백 UI 분리 (단문항/다문항) | ✅ | 5개 파일 수정 완료 (2026-04-08). 타입 오류 없음 확인 |
| LLM 문항 생성 contentGenerationInstruction | ✅ | 9개 파일 수정 완료 (2026-04-08). 빌드 성공 확인 |

---

## 2. 결정 완료 사항

| # | 항목 | 결정 | 근거 |
|---|------|------|------|
| 1 | 모듈 코드 | ✅ `picklass_pedagogy_profiles.ts` 정본 | WRD/WSD/GMN/WWB/RRD. backoffice 3개 UPDATE (IMG→GMN, WW→WWB, ORL→RRD) |
| 2 | 인증 방식 | ✅ 자체 JWT | Supabase Auth 사용하지 않음 |
| 3 | contentConfig | ✅ UI 흐름 제거 완료 | 타입 정의만 유지 (DB 스키마 목적) |
| 4 | DB 접근 | ✅ Prisma 직접 접근 | Supabase JS Client API 절대 금지 (R1 규칙) |
| 5 | AI 엔진 | ✅ Gemini 2.5 Flash + Azure Speech SDK | 이미 구현 완료. Claude API 사용하지 않음 |
| 6 | 문항 생성 | ✅ LLM 전량 생성 + DB 캐시 | 사전 등록 문항 없음. module_questions 캐시 |

---

## 3. 구현 플랜

### Phase 1 — LLM 문항 생성 완성 (Step 3~5)

> 목표: 모든 모듈의 문항을 LLM이 지문 기반으로 자동 생성하고 DB에 캐시

#### 1-1. Step 3 — 백엔드 문항 생성 API

- **신규 파일**: `apps/api/src/ai/question-generator.service.ts`
- **엔드포인트**: `POST /api/ai/generate-questions`
- **로직**:
  - Request: `{ moduleId, textId, passage, strategy, minCount, maxCount, answerType, pedagogyInstruction }`
  - Gemini 호출 → `extract`/`instruct` 전략별 프롬프트 분기
  - LLM이 지문 난이도 판단 → min~max 범위 내 최적 문항 수 결정
  - `module_questions` 테이블에 일괄 INSERT (status='active')
  - Response: 생성된 QuestionData[] 반환
- **캐시 무효화**: `DELETE /api/ai/generate-questions/:moduleId/:textId` → soft-delete (status='archived')

#### 1-2. Step 4 — Cache-Aside 패턴 적용

- **파일**: `apps/api/src/lessons/lessons.service.ts` `getModuleData()`
- **현재**: try/catch로 module_questions 실패 → questionData JSONB fallback
- **변경**: 캐시 히트 → 즉시 반환 / 캐시 미스 → `QuestionGeneratorService.generateAndSave()` 호출 → 저장 후 반환
- text_id 없는 레슨은 JSONB fallback 유지

#### 1-3-a. Step 3-b — contentGenerationInstruction + 지문 레벨 기반 문항 수 (✅ 완료, 2026-04-08)

> 목표: WRD가 comprehension 문항을 생성하는 버그 수정 + 모듈별 LLM 문항 생성 지시문 관리 기능 추가

##### 배경 및 문제

**문제**: WRD(어휘) 모듈에서 `short-text` answerType임에도 comprehension 주관식 문항이 생성됨.

**근본 원인**: 기존 `buildExtractRules('short-text')` fallback이 단 한 줄로 너무 간략하여 LLM이 컨텍스트를 이해하지 못하고 comprehension 스타일 문항 생성.

**해결 방향**: 백오피스에서 모듈별 `contentGenerationInstruction`(콘텐츠 생성 지시문)을 직접 입력하여 LLM 문항 생성 방식을 정밀 제어.

##### 설계 결정

| 항목 | 결정 |
|------|------|
| 지시문 우선순위 | 1순위: `contentGenerationInstruction` (백오피스 입력값) → 2순위: `buildExtractRules`/`buildInstructRules` fallback |
| 문항 수 결정 방식 | `texts.level` (1~18) + 모듈의 `questionMinCount`/`questionMaxCount` → lerp로 `recommendedCount` 계산 |
| instruct 전략 명확화 | "지문에서 추출하는 것이 아니라 LLM이 직접 과제 지시문을 작성"임을 명시하여 extract와 혼동 방지 |
| DB 데이터 보존 | 필드 추가 시 `DEFAULT ''` 사용으로 기존 데이터 무손실 보장 |

##### `recommendedCount` 계산 로직

```typescript
// texts.level: String | null (DB) → 1~18 정수로 파싱
function parsePassageLevel(level: string | null | undefined): number {
  if (!level) return 9;  // 기본값 중간 레벨
  const num = parseInt(level, 10);
  if (!isNaN(num) && num >= 1 && num <= 18) return num;
  // CEFR 코드 지원 (A1=1, A2=4, B1=7, B2=10, C1=14, C2=17)
  const cefrMap: Record<string, number> = { A1: 1, A2: 4, B1: 7, B2: 10, C1: 14, C2: 17 };
  return cefrMap[level.toUpperCase().trim()] ?? 9;
}

// min~max 사이 lerp (지문 레벨이 높을수록 문항 많이)
function calcRecommendedCount(level: number, min: number, max: number): number {
  if (min === max) return min;
  return Math.round(min + ((level - 1) / 17) * (max - min));
}
```

##### LLM 프롬프트 구조 (`question-generation.ts`)

```
[지문]
제목: {passageTitle}
{passageContent}

[교수법 지시문]
{pedagogyInstruction}        ← 피드백 스타일, 모듈 목적

[콘텐츠 생성 지시문]
{contentGenerationInstruction}  ← text/answer/hint 형식 포함 (fallback: buildExtractRules/buildInstructRules)

[문항 수 결정]
- 목표 문항 수: {recommendedCount}개 (지문 레벨 {passageLevel}/18 기반)
- 허용 범위: {minCount}~{maxCount}개
- 실제 지문 콘텐츠에 따라 허용 범위 내에서 조정 가능

[출력 형식] 반드시 순수 JSON만 출력
```

##### `buildInstructRules()` 핵심 가이드

```
instruct 전략은 지문에서 콘텐츠를 추출하는 것이 아니라,
지문 맥락을 바탕으로 학생이 수행할 과제 지시문을 LLM이 직접 작성합니다.
- text: 학생에게 보여줄 과제 지시문 전체 (한국어 2~3문장)
  예) "아래 지문을 읽고 핵심 주제를 한 문단으로 요약하세요."
- answer: "" (빈 문자열, holistic 채점)
- options: null, hint: null
```

##### 구현 대상 파일

**① DB 스키마 (backoffice Prisma)** — ✅ 완료

```prisma
model AiModule {
  contentGenerationInstruction String @default("") @map("content_generation_instruction") @db.Text
}
```

DDL (데이터 보존 방식):
```sql
ALTER TABLE ai_modules ADD COLUMN IF NOT EXISTS content_generation_instruction TEXT NOT NULL DEFAULT '';
```

**② DB 스키마 (tutoring Prisma)** — ✅ 완료

```prisma
model AiModule {
  contentGenerationInstruction String @default("") @map("content_generation_instruction") @db.Text
}
```

동일한 DDL로 기존 행 보존.

**③ `apps/api/src/ai/prompts/question-generation.ts`** — ✅ 완료 (전면 재작성)

- 신규 파라미터: `recommendedCount`, `passageLevel`, `contentGenerationInstruction?`
- 콘텐츠 블록 우선순위 로직 추가
- `buildInstructRules()` 상세 가이드 추가
- `buildExtractRules(answerType)` 5개 타입별 상세 규칙 (short-text, multiple-choice, essay, sentence-write, audio-record)
- LLM 출력 JSON에서 `passageDifficultyLevel` 제거 (단순화)

**④ `apps/api/src/ai/question-generator.service.ts`** — ✅ 완료

```typescript
interface GenerateQuestionsDto {
  // 기존 필드 유지
  recommendedCount: number;                    // 신규
  passageLevel: number;                        // 신규
  contentGenerationInstruction?: string;       // 신규
}
```

**⑤ `apps/api/src/ai/ai.controller.ts`** — ✅ 완료

`POST /ai/generate-questions` body 타입에 `recommendedCount`, `passageLevel`, `contentGenerationInstruction?` 추가.

**⑥ `apps/api/src/lessons/lessons.service.ts`** — ✅ 완료

```typescript
// texts.level 파싱 + recommendedCount 계산 후 getOrGenerate 전달
const passageLevel = parsePassageLevel(text?.level);
const recommendedCount = calcRecommendedCount(passageLevel, mod.questionMinCount, mod.questionMaxCount);

await questionGeneratorService.getOrGenerate({
  ...기존 필드,
  recommendedCount,
  passageLevel,
  contentGenerationInstruction: mod.contentGenerationInstruction,
});
```

**⑦ `packages/types/src/index.ts` (backoffice)** — ✅ 완료

```typescript
interface AiModuleResponse {
  contentGenerationInstruction: string;  // 추가
}
interface CreateAiModuleDto {
  contentGenerationInstruction?: string;  // 추가
}
interface UpdateAiModuleDto {
  contentGenerationInstruction?: string;  // 추가
}
```

**⑧ `packages/core/src/ai-module/ai-module.service.ts` (backoffice)** — ✅ 완료

- `create()`: `contentGenerationInstruction: dto.contentGenerationInstruction ?? ''` 추가
- `update()`: `if (dto.contentGenerationInstruction !== undefined) updateData.contentGenerationInstruction = ...` 추가
- `toResponse()`: `contentGenerationInstruction: module.contentGenerationInstruction` 추가

**⑨ `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx` (backoffice)** — ✅ 완료

- `ModuleFormData` 인터페이스에 `contentGenerationInstruction: string` 추가
- 초기값 `''`, 로드 시 `data.contentGenerationInstruction ?? ''`
- 저장 시 DTO에 포함
- UI: `pedagogyInstruction` textarea 아래에 `contentGenerationInstruction` textarea 추가
  - placeholder: WRD short-text 예시 (text/answer/hint 형식 포함)
  - 설명: "문항 콘텐츠 생성 지시문. 비어 있으면 answerType 기반 기본 규칙 적용"

##### 빌드 검증

```bash
# backoffice
pnpm run build
# → @repo/types, @repo/core, @app/admin-api, @app/admin-web 모두 성공 (2026-04-08)
```

##### 후속 작업

| 작업 | 방법 |
|------|------|
| 튜터링 서버 재시작 | `prisma generate` DLL 잠금 해제 필요 (서버 종료 후 실행) |
| WRD 백오피스 설정 | 모듈 편집 페이지에서 `contentGenerationInstruction` 입력 |
| WRD 캐시 무효화 | `DELETE /api/ai/generate-questions/:moduleId/:textId` |

---

#### 1-3. Step 5 — GenericAdapter 정리 + Legacy 어댑터 제거

- **5-A**: `GenericAdapter.buildAgentPrompt()` — `questionGenerationStrategy` 참조하여 extract/instruct별 피드백 프롬프트 구성
- **5-B**: Legacy 어댑터 5개 제거 (PRD/SCN/RRD/SWR/PWR)
  - 전제: DB 등록 완료 + module-data API 서빙 + module_questions 캐시 동작 확인
  - `adapters/index.ts` registry에서 제거
- **5-C**: `resolveDefaultUiTemplate()` 제거 → DB `uiTemplate` 값만으로 분기

---

### Phase 2 — 백엔드 API 잔여 구현

> 목표: 프론트엔드 Mock 데이터를 실제 API로 교체

#### 2-1. Lesson API

| 엔드포인트 | 설명 | 우선순위 |
|-----------|------|---------|
| `GET /api/lesson/{lessonId}/plan` | 레슨 플랜 조회 (Mock 교체) | P0 |
| `POST /api/lesson` | 레슨 생성 | P1 |
| `PUT /api/lesson/{lessonId}` | 레슨 수정 | P1 |

#### 2-2. 액세스코드 API

| 엔드포인트 | 설명 | 우선순위 |
|-----------|------|---------|
| `POST /api/lessons/register-accesscode` | 액세스코드 등록 → 수강 등록 | P0 |

- access_codes 유효성 확인 (만료, 이미 사용 여부)
- 유효 시 user_id 바인딩 + class_students 배정

#### 2-3. 모듈 이력 저장 + 학습 기록 집계

- **파일**: `apps/api/src/lessons/lessons.service.ts` `saveModuleHistory()`
- **현재**: 빈 함수 (로깅만)
- **변경**:
  - `module_histories` 테이블에 실제 저장 (answers, chat_messages, score, kpis, started_at, completed_at)
  - 저장 후 `student_learning_records` 집계 업데이트 (progress_rate, correct_answer_rate, pronunciation_accuracy 등)
  - `lesson_results` 테이블에 레슨 최종 결과 저장 (모든 모듈 완료 시)

#### 2-4. 프론트엔드 Mock 교체

| 대상 | 현재 | 목표 |
|------|------|------|
| `lessonService.fetchLessonPlan()` | 하드코딩 (L004, L008) | `GET /api/lesson/{id}/plan` |
| `LessonSession.saveModuleHistory()` | 빈 함수 | `POST /api/lessons/{id}/modules/{code}/history` |

#### 2-5. 레슨/모듈 완료 화면 개선

- **모듈 성과카드** (모듈 완료 시): 정답률(%) + 모듈별 KPI 차트 표시
- **레슨 완료 화면** (전체 모듈 완료 시): 평균 정답률 + 모듈별 성과 목록 + "홈으로 돌아가기" 버튼
- **ModuleProgressBar**: 2개 이상 모듈 레슨에서 진행 표시 동작 확인

---

### Phase 3 — Orchestrator Phase 1-5 (LLM 동적 피드백)

> 목표: pedagogyInstruction을 시스템 프롬프트로 Gemini API 호출하여 동적 피드백 생성

#### 3-1. 대상 규칙

| 규칙 | 현재 | 변경 |
|------|------|------|
| `ruleHolisticFeedback()` | 정적 문자열 | → `POST /ai/feedback` (pedagogyInstruction 기반 Gemini 생성) |
| `ruleWritingFeedback()` | 단어 수 비율 계산 | → `POST /ai/writing-eval` (이미 연동됨, pedagogyInstruction 활용 강화) |
| `rulePronunciationFeedback()` | 점수 구간별 정적 | → `POST /ai/pronunciation-eval` (이미 연동됨) + Gemini 피드백 텍스트 생성 |

#### 3-2. greeting 동적 생성

- 현재: `purpose` 기반 문자열 조합
- 목표: Gemini API로 맥락에 맞는 인사말 동적 생성

#### 3-3. Engagement/Disengage 체크 연동

- `ruleIdleCheck` — 120초 이상 무응답 시 `checkEngagement` 도구 호출 (참여도 확인 메시지)
- `ruleDisengaged` — 180초 + low 참여도 시 `signalReplan` 도구 호출 (대체 경로 제공)
- 현재: 규칙 존재하나 실제 Replan 흐름 미연결

#### 3-4. PRD 전체 지문 공개 규칙 검증

- `ruleRevealFullPassageAfterFeedback` — essay 피드백 완료 후 전체 지문 공개 + "예측과 비교해보세요" 메시지
- 현재: 규칙 코드 존재하나 PedagogyProfile 전환 후 정상 동작 검증 필요

---

### Phase 4 — 인증 시스템 구축 (회원가입 → 로그인 → 액세스코드 → 수강)

> 목표: MOCK_USER 제거, 자체 JWT 인증 도입

#### 4-0. 전체 사용자 인증 여정

```
[1단계: 회원가입]
  SignupModal → POST /api/auth/signup
    └─ [이메일] email + password + nickname → bcrypt 해시 → users INSERT → JWT 발급

[2단계: 로그인]
  LoginModal → POST /api/auth/login
    └─ [이메일] email + password → bcrypt.compare → JWT 발급

[3단계: 액세스코드 등록 (인증 필수)]
  AccessCodeModal → POST /api/lessons/register-accesscode
    → JWT에서 userId 추출 → access_codes 검증 → user_id 바인딩

[4단계: 수강 (인증 기반 필터링)]
  GET /api/lessons/enrolled-courses → JWT userId로 필터링
```

#### 4-1. 인증 백엔드 API

| 엔드포인트 | 설명 |
|-----------|------|
| `POST /api/auth/signup` | 이메일 회원가입 → JWT 발급 |
| `POST /api/auth/login` | 이메일 로그인 → JWT 발급 |
| `GET /api/auth/me` | 토큰 검증 + 사용자 정보 반환 |

- 자체 JWT 방식 확정. Supabase Auth 사용하지 않음.
- `apps/api/prisma/schema.prisma`에 `users` 모델 추가 (backoffice 스키마 참조, Prisma 직접 접근)
- **소셜 로그인/회원가입 (Google/Kakao/Naver OAuth)은 추후 개발** — 이메일 인증 안정화 후 진행

#### 4-2. 프론트엔드 인증 인프라

- `AuthProvider` + `useAuth()` 훅 (전역 인증 상태)
- `StudioHeader.tsx` — MOCK_USER 제거 → useAuth()
- `LoginModal.tsx` — alert → 이메일 로그인 API 호출
- `SignupModal.tsx` — alert → 이메일 회원가입 API 호출
- 소셜 로그인 버튼 — "준비 중" 상태 유지 (추후 개발)
- 로그인 전/후 헤더 UI 분기
- `window.location.reload()` → `router.refresh()`
- `alert()` → toast 컴포넌트 교체

#### 4-3. 인증 미들웨어

- **NestJS**: `JwtAuthGuard` — 보호 엔드포인트에 적용
- **Next.js**: `middleware.ts` — `/modules/[lessonId]` 접근 시 토큰 확인

---

### Phase 5 — 신규 모듈 UI 확장

> 목표: 13개 모듈 전체를 실제 학습 가능 상태로 만들기

#### 5-1. DB 등록만으로 동작 (코드 변경 없음)

| 모듈 | uiTemplate | answerType | 비고 |
|------|-----------|-----------|------|
| GMN | standard | essay | DB 등록만 |
| SUM | standard | essay | DB 등록만 |
| PRD | standard | essay | 이미 Mock 동작 중 |
| SCN | timed-gate | short-text | 이미 Mock 동작 중 |
| SWR | standard | sentence-write | 이미 Mock 동작 중 |

#### 5-2. answerType 패널 추가 필요

| 모듈 | answerType | 필요 UI |
|------|-----------|---------|
| SKM | click (→ 현재 multiple-choice) | 문장 클릭 선택 패널 |
| QAR | mcq-with-evidence (→ 현재 essay) | 근거 문장 클릭 + 답변 패널 |
| CLR | essay | sentence-by-sentence passageExposureMode 렌더링 |

#### 5-6. passageExposureMode 확장 렌더링 (ContentPanel)

profiles.ts에서 사용하는 4개 신규 모드에 대한 ContentPanel 렌더링 구현:

| passageExposureMode | 렌더링 | 사용 모듈 |
|---------------------|--------|----------|
| `highlight` | 지문 내 특정 단어/문장 강조 표시 (`highlights[]` 배열 기반) | WRD, WSD, GMN, RRD |
| `blurred` | 지문 전체 블러 처리 (읽기 후 가림) | SCN |
| `sentence-by-sentence` | 문장 단위 네비게이션 (한 문장씩 노출/이동) | CLR |
| `minimized` | 지문 최소화 (핵심어 추출 후 화면 축소) | WWB |

#### 5-7. Highlights (어휘 강조) 기능

- ContentPanel에서 `highlights[]` 배열을 받아 지문 내 해당 단어/문장을 시각적으로 강조
- 현재: ContentPanel에 `highlights` prop 존재하나 실제 렌더링 미구현
- 대상: WRD/WSD (학습 대상 단어), GMN (추론 대상 어휘), RRD (낭독 대상 문장)

#### 5-3. VocabDeckFlow 실제 연결

- 대상: WRD (short-text), WSD (audio-record), RRD (audio-record)
- 컴포넌트 구현 완료. DB 카드 데이터 서빙 시 연결 테스트

#### 5-4. InteractiveFlow 세부 타입

- 대상: WWB (Word Web), GMN (interactive 사용 시)
- 세부 인터랙션: word-web, image-match, dialog

#### 5-5. ProcessWritingFlow 범용화

- 현재: 4단계 하드코딩 → `StepWorkflowFlow`로 rename, DB 기반 단계 정의

---

#### 5-8. 피드백 UI 분리 — 단문항 / 다문항 (✅ 완료, 2026-04-08)

> 목표: 단문항(PRD 등)과 다문항(WRD 등) 피드백 확인 UI를 분리하여 UX 개선. `questionMaxAttempts` 기반 재시도 버튼 통합.

##### 배경 및 설계 결정

**문제**: `awaitingFeedbackConfirm: boolean` 상태가 단문항/다문항 공유 → 어느 문항의 피드백인지 구분 불가, 재시도 구현 불가능

**분기 기준 확정**:

| 기준 | 결정 이유 |
|------|---------|
| 상태 타입: `string \| null` 통합 | 단문항도 `questionMaxAttempts > 1`이면 questionId 필요 |
| 버튼 UI: `questionCount`로 분기 | single → "이해했어요", multi → "다음문항으로 →" |
| 재시도 버튼: `canRetry`로 분기 | `questionMaxAttempts === null \|\| attemptCount < max` |
| `scoringMode`로 분기하지 않는 이유 | 다문항 holistic 모듈도 동일한 "다음문항으로 →" 필요 |

**모듈별 `questionFlowMode` 확인** (`ai-module-data.ts` 주석 기준):

| 모드 | 대상 모듈 | 비고 |
|------|----------|------|
| `sequential` | PRD 등 단문항 | 기본값 |
| `deck` | WRD, WSD | `attemptCounts` dots 이미 구현됨 |
| `locked-steps` | PWR | 순서 잠금 |

##### 최종 UI 동작 매트릭스

| `questionCount` | `canRetry` | 표시 버튼 |
|-----------------|-----------|----------|
| single | true | 다시시도 + 이해했어요 👍 + 다시 설명해주세요 |
| single | false | 이해했어요 👍 + 다시 설명해주세요 |
| multi | true | 다시시도 + 다음문항으로 → |
| multi | false | 다음문항으로 → |

```typescript
const canRetry = questionMaxAttempts === null || currentAttemptCount < questionMaxAttempts;
```

##### 구현 대상 파일 및 변경 내용

**① `useModuleOrchestrator.ts`** — ✅ 완료

| 변경 | 내용 |
|------|------|
| `awaitingFeedbackConfirm: boolean` → `string \| null` | 상태 선언, 타입 정의, return 모두 변경 |
| `giveFeedback` case (correctness/holistic) | `setAwaitingFeedbackConfirm(params.questionId)` |
| `submitUserMessage` | `setAwaitingFeedbackConfirm(null)` — 사용자 메시지 입력 시 해제 |
| `confirmFeedback` | `setAwaitingFeedbackConfirm(null)` |
| `retryAnswer(questionId)` 신규 추가 | `answers[qId]` 삭제 + `feedbackGivenIds` 해당 항목 제거 + `orchestrate()` 재호출 |
| `attemptCounts: Record<string, number>` 반환 추가 | `attemptHistory`에서 useMemo로 파생 (`history.length`) |

**② `panels.tsx` — `FeedbackPanel`** — ✅ 완료

추가된 props:
```typescript
awaitingFeedbackConfirm?: string | null;    // boolean → string|null 타입 변경
onRetryAnswer?: (questionId: string) => void;
questionCount?: 'single' | 'multi';
currentAttemptCount?: number;               // default: 0
questionMaxAttempts?: number | null;
```

`canRetry` 계산 및 버튼 분기 렌더링:
```tsx
const canRetry = questionMaxAttempts === null || questionMaxAttempts === undefined
  ? false
  : currentAttemptCount < questionMaxAttempts;

// 단문항: 다시시도(조건부) + 이해했어요 + 다시설명해주세요
// 다문항: 다시시도(조건부) + 다음문항으로 →
```

**③ `panels.tsx` — `QuestionsPanel`** — ✅ 완료

`answers[qId]` 삭제(재시도) 시 `essayInputs[qId]` 자동 초기화:
```typescript
useEffect(() => {
  setEssayInputs((prev) => {
    const next = { ...prev };
    let changed = false;
    Object.keys(next).forEach((qId) => {
      if (answers[qId] === undefined && next[qId]) {
        next[qId] = '';
        changed = true;
      }
    });
    return changed ? next : prev;
  });
}, [answers]);
```

**④ `LessonSession.tsx`** — ✅ 완료

```typescript
// 오케스트레이터에서 신규 추출
const { retryAnswer, attemptCounts, ... } = useModuleOrchestrator(orchestratorInput);

// FeedbackPanel (desktop)에 전달
onRetryAnswer={retryAnswer}
questionCount={pedagogyProfile.questionCount}
questionMaxAttempts={pedagogyProfile.questionMaxAttempts}
currentAttemptCount={awaitingFeedbackConfirm ? (attemptCounts[awaitingFeedbackConfirm] ?? 0) : 0}

// MobileSplitLayout에도 동일하게 전달
// onFeedbackQuickReply: submitUserMessage → confirmFeedback 으로 변경
//   (모바일도 AI에 텍스트 전송이 아닌 피드백 확인 동작으로 통일)
```

**⑤ `MobileSplitLayout.tsx`** — ✅ 완료

- `awaitingFeedbackConfirm: boolean` → `string | null` 타입 변경
- `onFeedbackQuickReply: (text: string) => void` → `() => void` 타입 변경
- `onRetryAnswer`, `questionCount`, `currentAttemptCount` props 추가
- FeedbackPanel에 모든 신규 props 전달 (`questionMaxAttempts`는 기존 prop 재사용)

##### 영향 파일 요약

| 파일 | 변경 종류 | 상태 |
|------|---------|------|
| `useModuleOrchestrator.ts` | 상태 타입 변경, `retryAnswer` 콜백 추가, `attemptCounts` 반환 | ✅ |
| `panels.tsx` (FeedbackPanel) | props 확장, `canRetry` 계산, 단문항/다문항 버튼 분기 | ✅ |
| `panels.tsx` (QuestionsPanel) | `answers` 감지 useEffect → `essayInputs` 자동 초기화 | ✅ |
| `LessonSession.tsx` | 신규 props 추출 및 FeedbackPanel·MobileSplitLayout 연결 | ✅ |
| `MobileSplitLayout.tsx` | 타입 변경, 신규 props 추가 및 FeedbackPanel 전달 | ✅ |

##### 타입 검증

```bash
npx tsc --noEmit -p apps/web/tsconfig.json
# → 오류 없음 (2026-04-08 확인)
```

---

### Phase 6 — 나만의 수업 (주제 → 지문 생성 → 모듈 시퀀싱 → 학습)

> 목표: 학생이 주제를 입력하면 AI가 지문을 자동 생성하고, CurriculumPlanner가 모듈 시퀀스를 구성하여 즉시 학습 시작

#### 6-0. 현재 상태

| 구성 요소 | 상태 |
|----------|------|
| 홈 UI ("나만의 수업" 섹션) | ✅ 주제 입력 + 추천 주제 UI 완료 (`page.tsx`) |
| `/class/lesson-setup/custom` 라우트 | ✅ 구현 완료 (2026-04-07) |
| 주제 → 지문 텍스트 생성 API | ✅ `POST /ai/generate-passage` 구현 완료 (2026-04-07) |
| `POST /ai/analyze-passage` | ✅ Gemini 백엔드 구현 완료, 프론트 미연동 |
| `POST /ai/generate-lesson` | ✅ Gemini 백엔드 구현 완료, 프론트 미연동 |
| `CurriculumPlannerAgent.ts` | ⚠️ Mock 구현체 존재. create-custom API에서 백엔드 Gemini 직접 호출로 대체 |

#### 6-1. 주제 → 지문 자동 생성 API (신규)

- **엔드포인트**: `POST /api/ai/generate-passage`
- **Request**: `{ topic: string, targetLevel: number, wordCount?: number }`
- **로직**: Gemini에 주제/레벨/분량 전달 → 영어 지문 생성 → `texts` 테이블 INSERT
- **Response**: `{ textId: number, title: string, content: string, wordCount: number }`
- **프롬프트**: 주제 기반 교육용 영어 지문 생성 (CEFR 레벨 맞춤, 단락 구조 포함)

#### 6-2. 지문 분석 + 레슨 플랜 생성 연동

- **흐름**: 지문 생성(6-1) → `POST /ai/analyze-passage` → `POST /ai/generate-lesson`
- 이미 구현된 백엔드 API 2개를 프론트에서 순차 호출
- `CurriculumPlannerAgent.ts` Mock → 백엔드 API 호출로 교체 (또는 직접 API 호출 방식)

#### 6-3. 레슨 생성 + DB 저장

- 생성된 레슨 플랜을 `course_lessons` 테이블에 저장
  - `text_id`: 6-1에서 생성한 지문 ID
  - `skill_modules`: 6-2에서 생성한 모듈 시퀀스 JSONB
  - `topic`: 학생 입력 주제
  - `topic_source`: 'ai'
- 임시 과정(course) 생성 또는 "나만의 수업" 전용 과정에 레슨 추가

#### 6-4. 레슨 셋업 페이지 (신규)

- **라우트**: `/class/lesson-setup/custom`
- **UI 흐름**:
  ```
  [1] 주제 확인 (query param에서 topic 수신)
  [2] "지문 생성 중..." 로딩 (POST /ai/generate-passage)
  [3] 생성된 지문 미리보기 + 레벨 표시
  [4] "모듈 구성 중..." 로딩 (analyze-passage → generate-lesson)
  [5] 레슨 플랜 미리보기 (모듈 시퀀스 목록)
  [6] "학습 시작" 버튼 → /modules/{lessonId} 이동
  ```
- 선택적: 지문 재생성, 모듈 수동 조정 기능

#### 6-5. 문항 자동 생성 연결

- Phase 1(LLM 문항 생성)의 Cache-Aside 패턴으로 자동 처리
- 첫 학습 진입 시 `module_questions` 캐시 미스 → Gemini가 문항 생성 → DB 저장

---

### Phase 7 — 리포트 / KPI 대시보드

> 목표: 학생 성과 추적, 모듈별 KPI 시각화, 학습 이력 조회

#### 7-1. 리포트 페이지 (`/report`)

- **현재**: 헤더 네비게이션에 링크만 존재, 페이지 미구현
- **구현**:
  - 라우트: `/report`
  - 학생별 수강 과정 목록 + 각 과정의 레슨 완료 현황
  - 레슨별 모듈 성과 요약 (정답률, KPI 주요 지표)
  - 기간별 학습 추이 그래프

#### 7-2. 리포트 백엔드 API

| 엔드포인트 | 설명 |
|-----------|------|
| `GET /api/report/summary` | 학생 전체 학습 요약 (총 레슨 수, 평균 점수, 학습 시간) |
| `GET /api/report/lessons/{lessonId}` | 레슨별 모듈 결과 상세 (`lesson_results` + `module_histories` JOIN) |
| `GET /api/report/kpi-trends` | KPI 추이 데이터 (기간별 pronunciation_accuracy, correct_answer_rate 등) |

- 데이터 소스: `lesson_results`, `module_histories`, `student_learning_records`

#### 7-3. KPI 집계 로직

- `lesson_results` — 레슨 완료 시 `average_score`, `module_results` JSONB, `total_duration` 저장
- `student_learning_records` — 단원별 집계 (reading_wpm, pronunciation_accuracy, speaking_wpm, correct_answer_rate 등)
- `module_histories` → `student_learning_records` 집계 업데이트 트리거 (Phase 2-3에서 구현)

#### 7-4. 성과 이상 탐지 (선택적)

- 정답률 < 50% 모듈 하이라이트
- 학습 시간 이상 (평균 대비 2배 이상 소요) 경고
- 반복 오답 패턴 분석

---

## 4. 구현 순서 및 의존성

```
Phase 1 (LLM 문항 생성 완성)
  ├─ 1-1. Step 3 — 문항 생성 API (question-generator.service.ts)
  ├─ 1-2. Step 4 — Cache-Aside 패턴 (lessons.service.ts)
  └─ 1-3. Step 5 — GenericAdapter 정리 + Legacy 어댑터 제거
       ↓
Phase 2 (백엔드 API + 학습 흐름 완성)
  ├─ 2-1. Lesson API (GET plan Mock 교체)
  ├─ 2-2. 액세스코드 API
  ├─ 2-3. 모듈 이력 저장 + student_learning_records 집계
  ├─ 2-4. 프론트 Mock 교체
  └─ 2-5. 레슨/모듈 완료 화면 (성과카드, 평균 정답률, KPI)
       ↓
Phase 3 (Orchestrator 자율화)         Phase 4 (인증 — 자체 JWT)
  ├─ 3-1. LLM 동적 피드백               ├─ 4-1. 인증 백엔드 API
  ├─ 3-2. greeting 동적 생성            ├─ 4-2. 프론트 인증 연동
  ├─ 3-3. Engagement/Disengage 연동    └─ 4-3. 인증 미들웨어
  └─ 3-4. PRD 전체 지문 공개 검증
       ↓                                     ↓
Phase 5 (신규 모듈 UI 확장)
  ├─ 5-1. DB 등록 모듈 (GMN, SUM)
  ├─ 5-2. answerType 패널 추가 (click, mcq-with-evidence)
  ├─ 5-3. VocabDeckFlow 연결 (WRD, WSD, RRD)
  ├─ 5-4. InteractiveFlow 세부 (WWB, word-web/image-match/dialog)
  ├─ 5-5. ProcessWritingFlow 범용화
  ├─ 5-6. passageExposureMode 확장 렌더링 (highlight/blurred/sentence-by-sentence/minimized)
  ├─ 5-7. Highlights (어휘 강조) 기능
  └─ 5-8. 피드백 UI 분리 — 단문항/다문항 + canRetry (🔄 진행 중)
       ↓
Phase 6 (나만의 수업) ← Phase 1 + Phase 2 완료 필요
  ├─ 6-1. 주제→지문 생성 API (POST /ai/generate-passage)
  ├─ 6-2. 지문 분석 + 레슨 플랜 연동 (analyze-passage, generate-lesson)
  ├─ 6-3. 레슨 DB 저장 (course_lessons INSERT)
  ├─ 6-4. 레슨 셋업 페이지 (/class/lesson-setup/custom)
  └─ 6-5. 문항 자동 생성 연결 (Phase 1 Cache-Aside)
       ↓
Phase 7 (리포트 / KPI 대시보드) ← Phase 2-3 집계 로직 필요
  ├─ 7-1. 리포트 페이지 (/report)
  ├─ 7-2. 리포트 백엔드 API (summary, lesson detail, kpi-trends)
  ├─ 7-3. KPI 집계 로직 (lesson_results + student_learning_records)
  └─ 7-4. 성과 이상 탐지 (선택적)
```

---

## 5. Phase별 주요 파일 영향도

| Phase | 주요 변경 파일 | 신규/삭제 |
|-------|--------------|----------|
| 1 | `lessons.service.ts`, `GenericAdapter.ts`, `adapters/index.ts` | 신규: `question-generator.service.ts`. 삭제: Legacy 어댑터 5개, `resolveDefaultUiTemplate()` |
| 2 | `lessonService.ts`, `LessonSession.tsx`, `lessons.service.ts` | 신규: register-accesscode 로직, lesson_results 저장 로직 |
| 3 | `useModuleOrchestrator.ts`, `ModuleOrchestratorAgent.ts` | — |
| 4 | `StudioHeader.tsx`, `LoginModal.tsx`, `SignupModal.tsx`, `AccessCodeModal.tsx`, `schema.prisma` | 신규: `auth.controller.ts`, `auth.service.ts`, `auth.guard.ts`, `AuthProvider.tsx`, `useAuth.ts` |
| 5 | `panels.tsx`, `ProcessWritingFlow.tsx`, ContentPanel (passageExposureMode 렌더링), `useModuleOrchestrator.ts`, `LessonSession.tsx`, `MobileSplitLayout.tsx` | — (VocabDeckFlow, InteractiveFlow 이미 구현). 5-8: 피드백 UI 분리 |
| 6 | `page.tsx`, `CurriculumPlannerAgent.ts`, `ai.controller.ts`, `ai.service.ts` | 신규: `generate-passage` 프롬프트, `/class/lesson-setup/custom/page.tsx` |
| 7 | `lessons.service.ts` (집계 조회) | 신규: `/report/page.tsx`, `report.controller.ts`, `report.service.ts` |

---

## 6. 현재 AI 서비스 연동 현황

### 백엔드 (✅ 구현 완료)

| 서비스 | 엔진 | 파일 |
|--------|------|------|
| AiService (7개 메서드) | Gemini 2.5 Flash (`@google/genai`) | `apps/api/src/ai/ai.service.ts` |
| PronunciationService | Azure Speech SDK | `apps/api/src/ai/pronunciation.service.ts` |
| TtsService | Azure Speech SDK (Neural Voice) | `apps/api/src/ai/tts.service.ts` |

### 프론트→백엔드 연동 현황

| API | 프론트 호출 | 상태 |
|-----|-----------|------|
| `POST /ai/feedback` | useModuleOrchestrator, VocabDeckFlow, InteractiveFlow | ✅ |
| `POST /ai/writing-eval` | useModuleOrchestrator | ✅ |
| `POST /ai/hint` | useModuleOrchestrator | ✅ |
| `POST /ai/chat` | useModuleOrchestrator (2곳) | ✅ |
| `POST /ai/evaluate-kpi` | useModuleOrchestrator | ✅ |
| `POST /ai/pronunciation-eval` | useModuleOrchestrator | ✅ |
| `POST /ai/tts` | useModuleOrchestrator (2곳) | ✅ |
| `POST /ai/analyze-passage` | — | ⏳ 미연동 (CurriculumPlanner용) |
| `POST /ai/generate-lesson` | — | ⏳ 미연동 (CurriculumPlanner용) |

---

## 7. 파일 구조 (현재 상태)

```
apps/web/src/
├── lib/
│   ├── adapters/
│   │   ├── index.ts                      # 어댑터 레지스트리
│   │   ├── GenericAdapter.ts             # DB 기반 범용 어댑터 (R4 핵심)
│   │   ├── ModuleAdapter.ts              # 인터페이스 (buildContentConfig 제거됨)
│   │   ├── PredictionAdapter.ts          # Legacy (삭제 대기)
│   │   ├── ScanningAdapter.ts            # Legacy (삭제 대기)
│   │   ├── ShadowReadingAdapter.ts       # Legacy (삭제 대기)
│   │   ├── SentenceWritingAdapter.ts     # Legacy (삭제 대기)
│   │   └── ProcessWritingAdapter.ts      # Legacy (삭제 대기)
│   ├── agents/
│   │   ├── agent-types.ts                # ChatMessage, ModulePedagogyProfile, OrchestratorContext
│   │   └── orchestrator/
│   │       └── ModuleOrchestratorAgent.ts  # 규칙 기반 Orchestrator (pedagogyProfile 기반)
│   ├── types/
│   │   ├── ai-module.ts                  # ContentConfig, StageItem (DB 스키마 목적 유지)
│   │   ├── ai-module-data.ts             # AiModuleData + UiTemplate
│   │   └── module.ts                     # ModuleData, QuestionData
│   ├── services/
│   │   ├── aiModuleService.ts            # DB 응답 → AiModuleData 변환
│   │   └── lessonService.ts              # Mock 레슨 플랜 (교체 대상)
│   ├── pedagogy/
│   │   └── picklass_pedagogy_profiles.ts # 13개 모듈 PedagogyProfile 중앙 정의
│   └── hooks/
│       ├── useModuleOrchestrator.ts      # React Hook (AI API 7개 연동)
│       └── messageBuilders.ts            # 메시지 헬퍼
│       ※ useStageStateMachine.ts — 삭제됨
│       ※ usePanelRouter.ts — 삭제됨
├── app/
│   ├── page.tsx                          # 홈 대시보드 (enrolled-courses API 연동)
│   └── modules/[lessonId]/
│       └── _components/
│           ├── LessonSession.tsx          # uiTemplate switch 분기
│           ├── ProcessWritingFlow.tsx     # step-workflow (범용화 대상)
│           ├── VocabDeckFlow.tsx          # vocab-deck (구현 완료)
│           ├── InteractiveFlow.tsx        # interactive (구현 완료)
│           └── panels.tsx                # ContentPanel, QuestionsPanel 등
└── components/oizi/
    ├── StudioHeader.tsx                  # MOCK_USER (교체 대상)
    ├── LoginModal.tsx                    # alert 미구현 (교체 대상)
    ├── SignupModal.tsx                   # alert 미구현 (교체 대상)
    └── AccessCodeModal.tsx               # 프론트 구현됨, 백엔드 미구현

apps/api/src/
├── ai/
│   ├── ai.controller.ts                 # 9개 엔드포인트
│   ├── ai.service.ts                    # Gemini 2.5 Flash (7개 메서드)
│   ├── pronunciation.service.ts         # Azure Speech SDK
│   └── tts.service.ts                   # Azure Speech SDK TTS
├── lessons/
│   ├── lessons.controller.ts            # enrolled-courses, plan, module-data, module-meta, history
│   └── lessons.service.ts               # DB 조회 + 문항 fallback
└── prisma/
    └── schema.prisma                    # texts, courses, course_lessons, ai_modules, module_questions, module_histories, lesson_results
```
