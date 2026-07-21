---
기간: 2026-04-06 ~ 2026-06-09
원본 파일: agent-architecture-context-20260406.md, 20260417_CLR_content-driven_문항생성개선.md, 20260512_힌트시스템_개선_개발완료.md, 20260412_SKM_CLR_개발내역.md, 20260410_튜터링_모듈학습_UI개선및버그수정.md, 20260512_튜터링_모듈학습_오케스트레이터_피드백_개선.md
작성자: TimChoi1020
문서 목적: 문항생성 & 피드백 LLM 전략 레퍼런스 (기능별 재정리, 2026-06-09 코드 기준으로 갱신)
---

## 개발 상태

### ✅ 완료 항목

| 항목 | 완료일 |
|---|---|
| PedagogyProfile 설계 및 ContentConfig 대체 (7단계) | 2026-04-07 |
| LLM 문항 생성 전략 타입/스키마 (Step 1) | 2026-04-07 |
| 백오피스 문항 생성 전략 UI (Step 2) | 2026-04-07 |
| content-driven questionCount 도입 (CLR 적용) | 2026-04-17 |
| module_questions hints 필드 + Eager 생성 정책 | 2026-05-12 |
| feedbackStyle **백엔드** 제거 (`feedback.ts`, `ai.controller.ts`) | 2026-05-12 |
| `/ai/hint` API 제거 (사전 생성 힌트로 대체) | 2026-05-12 |
| Step 3: 백엔드 문항 생성 API (`question-generator.service.ts` `generateAndSave()`) | 2026-05-31 확인 |
| Step 4: Cache-Aside 패턴 정식화 (`lessons.service.ts` `getOrGenerate()`) | 2026-05-31 확인 |
| Step 5: Legacy 어댑터 제거 (`adapters/index.ts` — GenericAdapter 단일화) | 2026-04-07 |
| feedbackStyle **프론트엔드** 완전 제거 (5개 파일) | 2026-06-09 |
| scoringMode `'writing-eval'` 추가 — SWR 모듈 적용 | 2026-06-09 |
| pronunciation 피드백 LLM 제거 — `buildPronunciationFeedbackMessage()` 로컬 전환 | 2026-06-09 |
| `module_questions.explanation` JSONB 컬럼 추가 — 부가 메타데이터 분리 저장 (Phase 2 SQL) | 2026-07-02 |
| `multiple-choice` `answer` 포맷 정규화 — JSON 혼용 `{"correctOption":"③",...}` → 단순 문자열 `"③"` (Phase 3 SQL) | 2026-07-02 |
| `generateAndSave()` `explanation` 저장 누락 수정 — `QuestionGenerationResult` 타입 추가, 프롬프트 출력 스키마 추가, DB 저장 매핑 추가 (3파일) | 2026-07-02 |

### ❌ 미완료 항목

| 항목 | 비고 |
|---|---|
| `elapsedSeconds` 피드백 API 연동 | 프론트 타이머 UI(`floatingTimer`)만 구현. `feedback.ts` 파라미터 없음, API 전달 미구현 (§6.3 참조) |

---

## 1. 아키텍처 개요

### 1.1 두 레이어 역할 분리

모든 LLM 동작은 두 레이어로 역할을 명확히 나눈다:

```
[Orchestrator — 규칙 기반 흐름 제어]  ← ai_modules DB 필드로 제어
  scoringMode         → 어떻게 평가할지 (exact / holistic / pronunciation)
  questionMaxAttempts → 문항당 최대 시도 횟수
  retryScope          → 재도전 기준 지표
  answerType          → 어떤 입력 UI를 쓸지
  passageMode         → 지문 노출 방식
  questionCount       → 단일/다중 문항 여부
  questionGenerationStrategy → extract / instruct

[LLM — 자율 생성]  ← pedagogyInstruction DB 필드로 제어
  문항 생성   → strategy + contentGenerationInstruction 기반으로 지문에서 추출/생성
  피드백 생성 → pedagogyInstruction을 시스템 프롬프트로, 형식/기준/KPI 모두 LLM 자율 결정
  인사/힌트   → purpose + pedagogyInstruction 기반 동적 생성
```

### 1.2 전체 데이터 흐름

```
ai_modules (DB)
  ├─ question_generation_strategy: 'extract' | 'instruct'
  ├─ question_count: 'single' | 'multi' | 'content-driven'
  ├─ question_min_count, question_max_count
  ├─ content_generation_instruction   (문항 생성용 LLM 지시)
  └─ pedagogy_instruction             (피드백용 LLM 지시)
       ↓
GET /lessons/:id/module-data
       ↓
module_questions 조회 (Cache-Aside)
  ├─ 캐시 히트 → QuestionData[] 즉시 반환
  └─ 캐시 미스 → POST /ai/generate-questions
                    → Gemini 문항 + hints 동시 생성
                    → module_questions INSERT (status='active')
                    → QuestionData[] 반환
       ↓
useModuleOrchestrator (학습 진행)
  ├─ 오답 → ruleWrongAnswerFeedback / ruleRepeatWrongAnswer
  │    └─ giveHint: module_questions.hints.direct (API 호출 없음)
  └─ 답변 제출 → giveFeedback
                    → POST /ai/feedback
                    → pedagogyInstruction 시스템 프롬프트
                    → LLM 자율 형식 피드백 반환
```

---

## 2. PedagogyProfile (제어 플레인)

### 2.1 ModulePedagogyProfile 인터페이스

**파일**: `apps/web/src/lib/agents/agent-types.ts`

```typescript
export interface ModulePedagogyProfile {
  // Orchestrator 제어 필드
  scoringMode:                  'exact' | 'holistic' | 'pronunciation';
  answerType:                   'multiple-choice' | 'essay' | 'short-text' | 'mixed'
                                | 'audio-record' | 'sentence-write' | 'sentence-select'
                                | 'sentence-explain' | 'keyword-chips';
  passageMode:                  'hidden' | 'preview' | 'full' | 'timed-blur' | 'timed-select';
  questionCount:                'single' | 'multi' | 'content-driven';
  questionMaxAttempts?:         number;
  hintTypes:                    HintType[];
  retryScope:                   RetryScope;
  inputLanguage:                InputLanguage;
  passageRole:                  PassageRole;
  retryThreshold?:              number;    // % 기반 재도전 판정 (undefined = CLR 등 입력 없는 모듈)

  // LLM 자율 생성 필드
  purpose:                      string;    // 모듈 목적 요약 (greeting 생성)
  pedagogyInstruction:          string;    // 피드백/힌트 시스템 프롬프트

  // 문항 생성 전략 필드
  questionGenerationStrategy:   'extract' | 'instruct';
  questionMinCount:             number;
  questionMaxCount:             number;

  // ⚠️ 잔존 필드 (완전 제거 미완료)
  // feedbackStyle이 agent-types.ts에서는 아직 삭제되지 않음
  feedbackStyle:                'correct-wrong' | 'strengths-weaknesses' | 'holistic' | 'sentence-explain';
  // pronunciation 피드백 API 호출에서도 여전히 전달됨 (useModuleOrchestrator.ts)
  // TODO: agent-types.ts + useModuleOrchestrator.ts pronunciation 분기에서 완전 제거 필요
}
```

### 2.2 ContentConfig → PedagogyProfile 전환 결정

**전환 이유**: `ContentConfig`(7단계 JSONB) 기반 처방적 방식의 문제:
- Orchestrator가 `stage1~7Enabled` 등 설정값을 따라가는 수동적 역할
- 신규 모듈 추가 시마다 `ContentConfig` 하드코딩 필요 (R4 위반)
- `stage1Greeting` 고정 문자열이 LLM 자율 피드백 생성을 방해

```
기존 (처방적):
  ContentConfig.stage1Greeting → Orchestrator가 문자열 그대로 출력
  ContentConfig.stage1~7Enabled → UI 흐름 단계 하드코딩

새 방식 (자율적):
  purpose → LLM이 인사말 동적 생성
  pedagogyInstruction → LLM이 맥락에 맞게 피드백 자율 생성
  scoringMode + retryScope → Orchestrator가 상황 판단
```

**7단계 작업 내역 (모두 완료)**:

| 단계 | 작업 내용 | 핵심 변경 파일 |
|---|---|---|
| Step 1 | `ChatMessage` 타입 `useStageStateMachine.ts` → `agent-types.ts` 이동. 순환 참조 해소 | `agent-types.ts` |
| Step 2 | `OrchestratorContext.moduleConfig: ContentConfig` → `pedagogyProfile` 교체 | `agent-types.ts` |
| Step 3 | `ModuleOrchestratorAgent`에서 `stage1Greeting` → `purpose` 기반 동적 생성 | `ModuleOrchestratorAgent.ts` |
| Step 4 | `useModuleOrchestrator` 입력 타입 `moduleConfig` → `pedagogyProfile` | `useModuleOrchestrator.ts` |
| Step 5 | `LessonSession.tsx`에서 `config` 상태 변수 및 `ContentConfig` import 완전 제거 | `LessonSession.tsx` |
| Step 6 | 모든 어댑터에서 `buildContentConfig()` 제거 | `ModuleAdapter.ts`, `GenericAdapter.ts`, Legacy 어댑터 5개 |
| Step 7 | `useStageStateMachine.ts`, `usePanelRouter.ts` 삭제. `ai-module.ts` helper 제거 | 파일 2개 삭제 |

### 2.3 ContentConfig 처리 방침 (최종)

| 위치 | 처리 | 이유 |
|---|---|---|
| `OrchestratorContext.moduleConfig` | ✅ 제거 → `pedagogyProfile` 교체 | UI 흐름 제어 불필요 |
| `ModuleAdapter.buildContentConfig()` | ✅ 제거 | 어댑터가 반환할 필요 없음 |
| `useStageStateMachine.ts`, `usePanelRouter.ts` | ✅ 파일 삭제 | `ContentConfig` 의존, 외부 참조 없음 |
| `ai-module.ts` — `ContentConfig`, `StageItem` 타입 | **유지** | `AiModuleData.contentConfig` 필드 타입으로 사용 중 |
| `ai-module-data.ts` — `contentConfig: ContentConfig` | **유지** | DB/API 스키마 목적 (백오피스 JSONB 수신용) |

> `contentConfig`는 DB에 계속 존재하지만 **튜터링 UI 흐름에서는 일절 참조하지 않는다**.

### 2.4 Orchestrator 자율화 — 6개 필드 활성화 현황

| 필드 | 역할 | 상태 |
|---|---|---|
| `passageMode` | `ruleInitialEntry` switch 분기 | ✅ 활성 |
| `questionMaxAttempts` | `ruleRepeatWrongAnswer` hintTrigger 동적 계산 | ✅ 활성 |
| `hintTypes` | `giveHint` 유형 선택 | ✅ 활성 |
| `retryThreshold` | `ruleCompleteAfterCelebrate` 재도전 판정 | ⚠️ **미구현** — 필드는 존재하나 실제 판정 로직 없음 (dead code) |
| `purpose` | `ruleGreet` greeting 메시지 생성 | ✅ 활성 |
| `pedagogyInstruction` | `giveFeedback` LLM 시스템 프롬프트 (feedbackStyle 흡수) | ✅ 활성 |

---

## 3. 문항 생성 파이프라인

### 3.1 설계 원칙

- **모든 문항 콘텐츠는 LLM이 생성한다.** 사전 등록 문항 없음, 지문 무관 공통 문항 없음.
- **생성된 문항은 DB에 캐시한다.** 동일 `text_id × module_id` 조합은 LLM 반복 호출 없음.
- **힌트도 문항 생성 시 동시 생성(Eager)한다.** 힌트 요청 시점에 별도 API 호출 없음.
- 첫 번째 학습자가 생성을 트리거, 이후 학습자는 캐시(`module_questions`) 재사용.

### 3.2 전략 2종: extract / instruct

| 전략 | 값 | 설명 | 해당 모듈 |
|---|---|---|---|
| 추출형 | `extract` | 지문에서 단어·문장·선택지를 LLM이 직접 추출 | WRD, WSD, GMN, WWB, SKM, QAR, RRD, SWR |
| 지시형 | `instruct` | 지문 맥락 기반 지시문 LLM 생성. 학생 자유 응답 → holistic 평가 | PRD, CLR, SUM, PWR |

**모듈별 설정값:**

| 모듈 | questionCount | strategy | min | max |
|---|---|---|---|---|
| WRD, WSD | multi | extract | 5 | 10 |
| GMN, WWB | multi | extract | 2 | 5 |
| SKM | multi | extract | 2 | 4 |
| QAR | multi | extract | 3 | 5 |
| RRD | multi | extract | 3 | 6 |
| SWR | multi | extract | 2 | 5 |
| PRD, SUM | single | instruct | 1 | 1 |
| SCN | single | instruct | 1 | 1 |
| PWR | multi | instruct | 4 | 4 |
| CLR | content-driven | extract | — | — |

### 3.3 questionCount 3종: single / multi / content-driven

**multi — LLM이 지문 분석 후 min~max 범위 내 최적 수 결정:**

| 지문 특성 | LLM 판단 |
|---|---|
| 길이 길수록 | max에 가깝게 |
| 고급 어휘 밀도 높을수록 | 수 줄이기 |
| 핵심 아이디어 많을수록 | 수 늘리기 |
| 문장 구조 복잡할수록 | 수 줄이기 |

**content-driven — CLR 전용:**

| 항목 | 내용 |
|---|---|
| 목적 | LLM이 지문 전체를 기준으로 문항 수 결정 (지문 문장 수 = 문항 수) |
| min/max 처리 | DB 값 무시, 코드에서 `min=1, max=999` override |
| recommendedCount | `0` (LLM에 전달 안 함) |
| 프롬프트 변화 | `[문항 수 결정]` 블록 완전 생략 |

```typescript
// ✅ 올바른 패턴
const questionCount = (aiModule.questionCount ?? 'single') as 'single' | 'multi' | 'content-driven';
const min = questionCount === 'content-driven' ? 1 : (aiModule.questionMinCount ?? 1);
const max = questionCount === 'content-driven' ? 999 : (aiModule.questionMaxCount ?? 1);
const recommendedCount = questionCount === 'content-driven' ? 0 : calcRecommendedCount(...);

// ❌ 금지 패턴
const isMulti = questionCount === 'multi';  // content-driven 누락
```

### 3.4 Cache-Aside 패턴 (module_questions)

```
요청 (module_id × text_id)
    ↓
module_questions 조회 (status='active', orderBy sort_order)
    ↓ 캐시 히트              ↓ 캐시 미스
즉시 반환            QuestionGeneratorService.getOrGenerate()
                       → 내부에서 캐시 재확인 후 캐시 미스 시 generateAndSave() 호출
                          ↓
                     Gemini 호출 (문항 + hints 동시 생성)
                          ↓
                     module_questions.createMany() (status='active')
                          ↓
                     QuestionData[] 반환
```

**소프트 삭제 방식**: 캐시 무효화는 `status='archived'` 처리. 이후 요청 시 자동 재생성.

### 3.5 캐시 무효화 API

| 메서드 | 경로 | 설명 |
|---|---|---|
| `DELETE` | `/ai/generate-questions/:moduleId` | 모듈 전체 캐시 무효화 |
| `DELETE` | `/ai/generate-questions/:moduleId/:textId` | 특정 지문 캐시 무효화 |

```
DELETE /ai/generate-questions/:moduleId
  → module_questions WHERE module_id=:moduleId AND status='active'
  → UPDATE status='archived'
  → { invalidated: N }
```

**무효화가 필요한 시점**: `contentGenerationInstruction`, `questionCount`, `questionMinCount`, `questionMaxCount` 변경 시 (현재 수동 API 호출 필요 → 자동화 미구현, 기술 부채).

---

## 4. LLM 프롬프트 설계

### 4.1 공통 프롬프트 구조

```typescript
// buildQuestionGenerationPrompt() — question-generation.ts

// extract 전략
`[시스템]
당신은 영어 교육 전문 문항 생성 AI입니다.
아래 지문을 분석하여 {strategy} 방식으로 {min}~{max}개 범위에서
최적 문항 수를 결정하고 문항을 생성하세요.

[지문 난이도 판단 기준]
beginner / intermediate / advanced

[extract 생성 규칙]
- answerType='audio-record' / 'short-text': 지문에서 핵심 문장/단어 직접 추출
- answerType='multiple-choice': 지문 기반 선택지 4개 + 정답 1개
- answerType='sentence-write': 핵심 문장을 한글로 패러프레이징 → text, answer는 영어 원문

[출력 형식] JSON만 출력 (설명 없음)`

// instruct 전략
`[instruct 생성 규칙]
- 지문 제목·핵심 내용을 반영한 자연스러운 지시문 생성
- text: 학생에게 보여줄 지시문 전체
- answer: "" (빈 문자열, holistic 채점)`
```

### 4.2 content-driven: [문항 수 결정] 블록 생략 정책

```typescript
// ✅ content-driven이면 countBlock 완전 생략
const countBlock = params.questionCount === 'content-driven'
  ? ''
  : `\n[문항 수 결정]\n${params.minCount}~${params.maxCount}개 범위에서 LLM이 최적 수 결정`;
```

- 블록 생략 시 LLM은 `contentGenerationInstruction`에 명시된 방식으로 문항 수 자체 결정
- CLR: `"지문 전체 문장 수만큼 반복"` → `decidedCount: 지문 전체 문장 수`

### 4.3 CLR sentence-explain 전용 출력 템플릿

일반 템플릿의 `"options": ["A. ...", "B. ..."]`이 CLR JSON 객체 형식과 충돌 → `answerType === 'sentence-explain'`일 때 전용 분기:

```typescript
// question-generation.ts
${params.answerType === 'sentence-explain' ? `{
  "questions": [{
    "text": "지문 원문 문장 (절대 수정 금지)",
    "options": { "해석": "...", "문법설명": "...", "주요표현": ["표현1: 설명", "표현2: 설명"] },
    "answer": "",
    "hint": null
  }]
}` : `{ /* 일반 문항 형식 */ }`}
```

**options 시행착오 이력 (설계 결정 기록)**:
1. 최초: `hint`에 JSON 문자열 → LLM이 JSON 이스케이프 불안정
2. 2차: `answer`에 해석, `hint`에 문법 → LLM 템플릿 `"answer": "정답"` 충돌로 해석 미생성
3. **최종**: `options`에 JSON 객체 → DB 스키마 변경 불필요, 파싱 단순

### 4.4 `explanation` 출력 스키마 이슈 (2026-07-02 수정)

**문제**: `contentGenerationInstruction`에서 `explanation` 필드를 정의해도 Gemini가 생성하지 않음.

**원인**: `question-generation.ts`의 `[출력 형식]` 블록이 Gemini에게 "반드시 아래 JSON만 출력"을 강제하는데, 이 스키마에 `explanation`이 없었음 → `contentGenerationInstruction`의 `explanation` 지침이 출력 스키마에 묻혀 무시됨.

**수정 (3파일)**:

```
apps/api/src/ai/prompts/question-generation.ts   — [출력 형식] 스키마에 explanation 필드 추가
apps/api/src/ai/ai.service.ts                    — QuestionGenerationResult 인터페이스에 explanation 추가
apps/api/src/ai/question-generator.service.ts    — generateAndSave() DB 저장 매핑에 explanation 추가
```

`question-generation.ts` 출력 스키마 (`explanation` 추가 후):
```json
{
  "answer": "정답",
  "explanation": { "키": "값" } 또는 null (contentGenerationInstruction에 정의된 부가 메타데이터. 없으면 null),
  "hints": { "button": "...", "direct": "..." } 또는 null
}
```

`ai.service.ts` `QuestionGenerationResult` (`explanation` 추가 후):
```typescript
export interface QuestionGenerationResult {
  passageDifficultyLevel: 'beginner' | 'intermediate' | 'advanced';
  decidedCount: number;
  questions: {
    question_number: number;
    type: string;
    instruction: string | null;
    source: string | null;
    text: string | null;
    options: string[] | null;
    answer: string | Record<string, unknown>;
    explanation: Record<string, unknown> | null;  // ← 추가
    hints: { button?: string; direct?: string } | null;
  }[];
}
```

`question-generator.service.ts` 저장 매핑 (`explanation` 추가 후):
```typescript
answer: typeof q.answer === 'string' ? q.answer : JSON.stringify(q.answer),
explanation: q.explanation ? (q.explanation as Prisma.InputJsonObject) : undefined,  // ← 추가
hints: q.hints ? (q.hints as Record<string, string>) : undefined,
```

> ⚠️ **기존 캐시 주의**: Phase 3 SQL 실행 전에 저장된 문항은 `explanation`이 NULL. `invalidateCache` API 호출 후 재생성 필요.

### 4.5 인터페이스 정의

**GenerateQuestionsDto** (`question-generator.service.ts`):

```typescript
export interface GenerateQuestionsDto {
  moduleId:                     string;         // ai_modules.id
  textId:                       number;         // texts.id
  passage:                      { title: string; content: string; };
  strategy:                     'extract' | 'instruct';
  questionCount:                'single' | 'multi' | 'content-driven';
  minCount:                     number;
  maxCount:                     number;
  recommendedCount:             number;         // content-driven이면 0
  passageLevel:                 number;
  answerType:                   string;
  pedagogyInstruction:          string;         // Gemini system 프롬프트
  contentGenerationInstruction?: string;
}
```

**QuestionGenerationPromptParams** (`question-generation.ts`):

```typescript
export interface QuestionGenerationPromptParams {
  passageTitle:                 string;
  passageContent:               string;
  strategy:                     'extract' | 'instruct';
  questionCount?:               'single' | 'multi' | 'content-driven';
  minCount:                     number;
  maxCount:                     number;
  recommendedCount:             number;
  passageLevel:                 number;
  answerType:                   string;
  pedagogyInstruction:          string;
  contentGenerationInstruction?: string;
}
```

---

## 5. module_questions 테이블

### 5.1 테이블 컬럼 구조

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | UUID PK | |
| `module_id` | UUID FK → ai_modules | 어떤 모듈의 문항인지 |
| `text_id` | INT FK → texts | 어떤 지문에 대한 문항인지 |
| `question_number` | INT | |
| `type` | VARCHAR(30) | answerType (code_items ANSWER_TYPE 참조) |
| `text` | TEXT NOT NULL | 학습자에게 보여줄 문항 텍스트 또는 지시문 |
| `instruction` | TEXT | QuestionsPanel 헤더 표시 문자열 |
| `options` | JSONB | 객관식 선택지 또는 CLR 해설 JSON |
| `answer` | TEXT | 채점 기준값만. exact 채점용 단순 문자열 (예: `"③"`, `"urban"`). holistic이면 빈 문자열. |
| `explanation` | JSONB | 부가 메타데이터. multiple-choice: `{goodAnswer, evidence, evidenceReason, questionType, 해석, 문법설명, 주요표현, 설명}`. sentence-explain(CLR 예정): `{해석, 문법설명, 주요표현}` |
| `hints` | JSONB | `{ button: string, direct: string }` |
| `sort_order` | INT DEFAULT 0 | |
| `status` | VARCHAR(20) DEFAULT 'active' | active / archived |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

### 5.2 hints 3레벨 구조 + Eager 생성 정책

**hints 3레벨:**

| 필드 | 레벨 | 트리거 | 내용 |
|---|---|---|---|
| `hints.button` | Level 0 | 학습자 버튼 클릭 | 학습자 주도 힌트 텍스트 |
| `hints.direct` | Level 1 | ruleRepeatWrongAnswer 자동 | 오답 누적 시 직접 힌트 |
| `answer` | Level 2 | revealAnswer | 최종 정답 제시 |

**Eager 생성 정책** (Lazy 미채택 이유):
- 힌트 사용 빈도 높은 모듈(WRD, GMN 등)은 Eager가 더 비용 효율적
- Lazy는 첫 번째 요청자에게 LLM 응답 지연 발생
- 두 방식 모두 1회 호출 후 캐시 — 차이는 타이밍뿐

```
문항 생성 시: Gemini가 문항 + hints 동시 생성 → module_questions 저장
힌트 요청 시: module_questions.hints.direct 즉시 반환 (API 호출 0회)
```

```typescript
// giveHint 케이스
staticHint 있음 → "💡 힌트: {hints.direct 값}"
staticHint 없음 → buildHintMessage() (범용 폴백)
```

**API 구현** (`question-generator.service.ts`):
```typescript
saveHintDirect(questionId: string, direct: string):
  → module_questions.hints.direct 저장
```

**API 변경** (`ai.controller.ts`):
```typescript
POST /ai/feedback:   generateHint?, hintTypes?, questionId? 파라미터 추가
POST /ai/writing-eval: 동일 추가
// generateHint=true 시: { "feedback": "...", "hint": "..." } 동시 생성
```

### 5.3 CLR options 필드 구조 (현행 — `explanation` 이전 예정)

현재 CLR(`sentence-explain`)의 해설 데이터는 `options` JSONB에 저장됨 (설계 이력 §4.3 참조).
향후 `explanation` 컬럼으로 이전 예정이며, `GenericAdapter`는 이미 양쪽 폴백을 지원한다.

```json
{
  "해석": "국문 해석 텍스트",
  "문법설명": "문법 구조 설명",
  "주요표현": ["표현1: 설명", "표현2: 설명"]
}
```

**파싱 코드** (`GenericAdapter.ts` — `explanation` 컬럼 우선, `options` 폴백):
```typescript
// sentence-explain: explanation 컬럼 우선, 없으면 options JSONB 폴백
const raw = (q.explanation ?? q.options) as
  | { 해석?: string; 문법설명?: string; 주요표현?: string[] }
  | null | undefined;
if (raw) {
  explanation = {
    해석: raw.해석 ?? '',
    문법: raw.문법설명 ?? '',
    주요표현: Array.isArray(raw.주요표현) ? raw.주요표현 : [],
  };
}
```

### 5.4 multiple-choice explanation 필드 구조 (2026-07-02 확정)

QAR 등 `multiple-choice` 타입은 `explanation` JSONB에 8개 서브필드를 저장한다:

```json
{
  "goodAnswer": "tough and able to recover",
  "evidence": "The resilient tree withstood the storm despite its age.",
  "evidenceReason": "'withstood the storm despite its age'에서 강인함을 직접 보여주므로 정답의 근거가 됩니다.",
  "questionType": "유형 3",
  "해석": "그 강인한 나무는 오래되었음에도 폭풍을 견뎌냈다.",
  "문법설명": "'despite + 명사(구)' 구조로 '~에도 불구하고'라는 양보 의미. 절이 오면 'although' 사용.",
  "주요표현": ["withstand the storm: 폭풍을 견뎌내다", "resilient: 회복력 있는, 강인한"],
  "설명": "정답은 B. 'withstood the storm despite its age'에서 강인함과 회복력이 드러납니다. A fragile은 반대 의미, C ancient는 나이만 강조..."
}
```

이 데이터는 Tutor Agent 피드백 생성 및 학생 해설 열람 시 활용한다 (`answer`는 채점에만 사용).

---

## 6. 피드백 LLM 전략

### 6.1 pedagogyInstruction 단일 소스 원칙

**기존 문제 (`feedbackStyle` 코드 강제)**:
- `'holistic'` → "잘한 점 / 개선할 점" 두 섹션 강제
- `'correct-wrong'` → 정답/오답 형식 강제
- `'wpm-pronunciation'` → WPM + 발음 형식 강제
→ `pedagogyInstruction`에 상세 지침을 작성해도 코드 형식이 우선 → DB 설정 무시

**결정** (2026-04-10 Phase 1 → 2026-05-12 백엔드 완성 → **2026-06-09 프론트엔드 완전 제거**):

> `pedagogyInstruction`을 피드백 형식·평가 기준·KPI의 **단일 소스**로 지정. 백엔드·프론트엔드 모두에서 `feedbackStyle` 완전 제거 완료 (§6.4 참조).

**scoringMode 기반 LLM 호출 분기 (2026-06-09 확정)**:

| scoringMode | 피드백 경로 | LLM 호출 |
|---|---|---|
| `exact` | `buildFeedbackMessage()` 로컬 | ❌ 없음 |
| `holistic` | `/ai/feedback` | ✅ |
| `pronunciation` | `buildPronunciationFeedbackMessage()` 로컬 | ❌ 없음 (힌트 생성만 별도 호출) |
| `writing-eval` | `/ai/writing-eval` | ✅ |

> **pronunciation LLM 제거 근거**: Azure Speech SDK가 이미 수치 점수(0–100)를 제공하고, `rulePronunciationFeedback`이 `fluencyPoints`·`pronunciationPoints`를 로컬에서 계산 완료. `buildPronunciationFeedbackMessage()` 폴백이 동일 역할 수행 가능.

> **`writing-eval` scoringMode 신설 근거**: `sentence-write` answerType은 `holistic`과 채점 방식이 다름. LLM 기반 정확한 영작문 평가(`/ai/writing-eval`)가 필요하며, PWR 구현 시 동일 파이프라인 재사용 예정. `answerType` 역추론 없이 `scoringMode`로 명시적 분기.

```
[코드] scoringMode 기반 피드백 분기
   ↓
holistic / writing-eval → [DB] pedagogyInstruction → [LLM] KPI 직접 계산 → 피드백 생성
exact / pronunciation  → 로컬 규칙 기반 피드백 (LLM 0회)
      ※ elapsedSeconds는 미구현 (§6.3 참조)
```

### 6.2 피드백 프롬프트 파이프라인

**`apps/api/src/ai/prompts/feedback.ts` 실제 현재 코드:**

```typescript
// feedbackStyle 제거 완료 (2026-05-12)
// generateHint, hintTypes 파라미터 추가됨

export interface FeedbackPromptParams {
  passageTitle: string;
  passageContent: string;
  questionText: string;
  studentAnswer: string;
  pedagogyInstruction: string;
  generateHint?: boolean;   // true 시 피드백+힌트 동시 생성
  hintTypes?: string[];
  // ⚠️ elapsedSeconds 미구현 — 타이머 UI에만 존재, 피드백 프롬프트 미전달
}

export function buildFeedbackPrompt(params: FeedbackPromptParams): string {
  const hintInstruction = params.generateHint
    ? `\n[힌트 생성 — 2차 시도 안내]\n...`
    : '';

  return `...
[교수법 지시문 — 형식 및 내용 모두 준수]
${params.pedagogyInstruction}
${hintInstruction}
[지문]
...
[학생 답안]
...`;
}
```

**`POST /ai/feedback` 요청 바디 변경 이력:**

| 파라미터 | 변경 전 | 변경 후 |
|---|---|---|
| `feedbackStyle` | string (required) | **삭제** (2026-05-12) ✅ |
| `pedagogyInstruction` | string | string (형식 지침 포함) |
| `generateHint`, `hintTypes`, `questionId` | 없음 | 추가됨 (2026-05-12) ✅ |
| `elapsedSeconds` | 없음 | **미구현** — UI 타이머만 존재, API 연동 없음 ❌ |

### 6.3 elapsedSeconds 활용 (WPM 등 KPI) — ⚠️ 미구현

> **현재 상태**: `LessonSession.tsx`의 `floatingTimer.elapsedSeconds`는 UI 타이머 표시 전용으로만 사용됨.
> `finalElapsedSeconds`, `elapsedSecondsRef` 패턴은 아직 구현되지 않았으며,
> 피드백 API(`POST /ai/feedback`)로 `elapsedSeconds`가 전달되지 않음.
> `feedback.ts`의 `FeedbackPromptParams`에도 `elapsedSeconds` 필드 없음.

timed 모듈(SCN, SKM 등)에서 읽기 소요 시간을 피드백 API에 전달하여 LLM이 WPM 등 KPI를 직접 계산하는 **목표 설계** (구현 필요):

```typescript
// [목표] LessonSession.tsx — 읽기완료 시 elapsed 캡처
const handleTimedDone = useCallback(() => {
  setFinalElapsedSeconds(Math.floor((Date.now() - (readingStartTime ?? Date.now())) / 1000));
  setReadingPhase('done');
}, [readingStartTime]);

// [목표] ref로 공유 (circular dep 해결)
const elapsedSecondsRef = useRef<number | undefined>(undefined);
useEffect(() => {
  if (finalElapsedSeconds !== undefined) elapsedSecondsRef.current = finalElapsedSeconds;
}, [finalElapsedSeconds]);

// [목표] useModuleOrchestrator.ts — 피드백 API 호출 시
body: JSON.stringify({
  ...
  elapsedSeconds: input.elapsedSecondsRef?.current,
})
```

**구현 필요 파일**: `LessonSession.tsx`, `useModuleOrchestrator.ts`, `ai.controller.ts`, `feedback.ts`

**KPI 활용 방향 (구현 전 결정 필요):**

| 옵션 | 방법 | 권장 여부 |
|---|---|---|
| A | `selectedKpiCodes`만 전달 | ❌ LLM이 코드만으로 해석해야 함 |
| B | KPI 메타데이터(이름/단위/측정방법)까지 전달 | ✅ 권장 |
| C | 피드백 + KPI 평가 통합 1회 호출 | 검토 필요 |

### 6.4 feedbackStyle 제거 관련 파일 변경 현황 (2026-06-09 완전 제거 완료)

| 파일 | 상태 | 내용 |
|---|---|---|
| `apps/api/src/ai/prompts/feedback.ts` | ✅ 완료 | `feedbackStyle` 파라미터 삭제, `formatInstruction` 분기 블록 삭제 (2026-05-12) |
| `apps/api/src/ai/ai.controller.ts` | ✅ 완료 | `feedbackStyle: string` 파라미터 제거 (2026-05-12) |
| `apps/web/src/lib/hooks/useModuleOrchestrator.ts` | ✅ 완료 | pronunciation 분기 API 호출 전체 제거 → `feedbackStyle` 전달 라인 소멸 (2026-06-09) |
| `apps/web/src/lib/agents/agent-types.ts` | ✅ 완료 | `ModulePedagogyProfile.feedbackStyle` 필드 삭제 (2026-06-09) |
| `apps/web/src/lib/types/ai-module-data.ts` | ✅ 완료 | `AiModuleData.feedbackStyle` 필드 삭제 (2026-06-09) |
| `apps/web/src/lib/adapters/GenericAdapter.ts` | ✅ 완료 | `buildPedagogyProfile()`에서 `feedbackStyle` 복사 라인 삭제 (2026-06-09) |
| `apps/web/src/lib/pedagogy/picklass_pedagogy_profiles.ts` | ✅ 완료 | 13개 모듈 프로파일 전체 `feedbackStyle` 값 삭제 (2026-06-09) |
| `apps/web/src/lib/services/aiModuleService.ts` | ✅ 완료 | `parseFeedbackStyle()` 호출 라인 삭제 (2026-06-09) |

---

## 7. 구현 현황 & 미완료 작업

### 7.1 완료 항목

**Step 1 — 타입/스키마 변경 (2026-04-07, 11개 파일):**

| 파일 | 변경 내용 |
|---|---|
| `apps/web/src/lib/agents/agent-types.ts` | `ModulePedagogyProfile`에 `questionGenerationStrategy`, `questionMinCount`, `questionMaxCount` 추가 |
| `apps/web/src/lib/types/ai-module-data.ts` | `AiModuleData`에 동일 3개 필드 추가 |
| `apps/web/src/lib/services/aiModuleService.ts` | `parseQuestionGenerationStrategy()` 파서 추가 |
| `apps/web/src/lib/adapters/GenericAdapter.ts` | `buildPedagogyProfile()`에 3개 필드 추가 |
| Legacy 어댑터 5개 | `buildPedagogyProfile()`에 strategy, min, max 추가 |
| `apps/web/src/lib/pedagogy/picklass_pedagogy_profiles.ts` | 13개 모듈 프로파일에 3개 필드 추가 |
| `packages/types/src/index.ts` (backoffice) | `AiModuleResponse`, `CreateAiModuleDto`, `UpdateAiModuleDto` 반영 |

**Step 2 — 백오피스 UI (2026-04-07):**
- `apps/admin/frontend/.../ai-modules/register/page.tsx`
- 문항설계2 탭에 `questionGenerationStrategy` 카드형 라디오, `questionMinCount`/`questionMaxCount` 숫자 입력 추가

### 7.2 완료된 Step 3·4·5 구현 이력 (2026-05-31 코드 확인)

**Step 3 — 백엔드 문항 생성 서비스 (✅ 구현 완료)**

파일: `apps/api/src/ai/question-generator.service.ts`

- `generateAndSave()`: Gemini 호출 → `module_questions.createMany()` 일괄 insert
- `getOrGenerate()`: 캐시 조회 → 미스 시 `generateAndSave()` 호출 (lessons.service.ts에서 진입)
- `saveHintDirect()`: `module_questions.hints.direct` 업데이트

**Step 4 — Cache-Aside 패턴 (✅ 구현 완료)**

파일: `apps/api/src/lessons/lessons.service.ts`

```typescript
// 실제 구현 패턴
const generated = await this.questionGeneratorService.getOrGenerate({
  moduleId: aiModule.id,
  textId: lesson.text_id,
  passage: { title: lesson.text.title, content: lesson.text.content },
  ...
});
```

**Step 5 — Legacy 어댑터 제거 (✅ 완료 — 2026-04-07)**

파일: `apps/web/src/lib/adapters/index.ts`

- `PredictionAdapter`, `ScanningAdapter`, `ShadowReadingAdapter`, `SentenceWritingAdapter`, `ProcessWritingAdapter` 파일 삭제
- `createAdapterForModule()` 함수만 노출 — `AiModuleData` 있으면 항상 `GenericAdapter` 사용
- `getAdapter()` deprecated stub으로만 유지 (호출 시 예외 던짐)

> ⚠️ **데드 코드 잔존**: `apps/web/src/lib/pedagogy/picklass_pedagogy_profiles.ts` — 레거시 어댑터 삭제 후 어디서도 import되지 않는 고아 파일. 삭제 권장 (기술 부채). 단, 모듈별 PedagogyProfile 설계 기준값이 담겨 있으므로 삭제 전 내용 검토 필요.

---

## 8. DB 스키마 변경 이력

```sql
-- 2026-04-07: LLM 문항 생성 전략 필드 추가 (backoffice + tutoring 양쪽)
ALTER TABLE ai_modules ADD COLUMN question_generation_strategy VARCHAR(20) NOT NULL DEFAULT 'extract';
ALTER TABLE ai_modules ADD COLUMN question_min_count           INT         NOT NULL DEFAULT 1;
ALTER TABLE ai_modules ADD COLUMN question_max_count           INT         NOT NULL DEFAULT 1;

-- 2026-04-17: content-driven 도입 후 CLR 직접 업데이트
-- (Supabase REST API PATCH)
-- question_count: 'content-driven'  (CLR 모듈 id: 0f79ce16-c903-4049-aeb0-43df7309c9eb)

-- 2026-07-02: module_questions answer 정규화 + explanation 컬럼 추가
-- (apps/api/prisma/manual-sql/2026-07-02_module_questions_explanation_column.sql)

-- Phase 2: explanation JSONB 컬럼 추가
ALTER TABLE module_questions ADD COLUMN IF NOT EXISTS explanation JSONB;

-- Phase 3: JSON 형태 answer 레코드 정규화
-- (answer에 {"correctOption":"③",...} JSON이 혼용되던 레코드를 분리)
UPDATE module_questions
SET
  explanation = (answer::jsonb - 'correctOption'),
  answer      = answer::jsonb->>'correctOption'
WHERE
  answer LIKE '{%' AND answer::jsonb ? 'correctOption';
```

---

## 9. 확장 가이드

### questionCount 타입 추가 시 체크리스트 (10개 파일 동시 수정)

```
tutoring/apps/web/src/lib/types/ai-module-data.ts
tutoring/apps/web/src/lib/agents/agent-types.ts       (2곳)
tutoring/apps/web/src/lib/adapters/GenericAdapter.ts
tutoring/apps/web/src/app/modules/[lessonId]/_components/panels.tsx
tutoring/apps/web/src/app/modules/[lessonId]/_components/MobileSplitLayout.tsx
tutoring/apps/api/src/ai/ai.controller.ts
tutoring/apps/api/src/ai/question-generator.service.ts
tutoring/apps/api/src/ai/prompts/question-generation.ts
tutoring/apps/api/src/lessons/lessons.service.ts
backoffice/apps/admin/frontend/.../ai-modules/register/page.tsx
```

### 신규 answerType 추가 시 영향 범위

| 영향 파일 | 내용 |
|---|---|
| `module.ts` | `QuestionType` 유니온에 추가 |
| `ai-module-data.ts` | `answerType` 유니온에 추가 |
| `agent-types.ts` | `ModulePedagogyProfile.answerType` 유니온에 추가 |
| `aiModuleService.ts` | `VALID_ANSWER_TYPES` 배열에 추가 |
| `ModuleOrchestratorAgent.ts` | holistic/exact 분기 체크 (`HOLISTIC_TYPES` 또는 `hasRightWrongConcept`) |
| `question-generation.ts` | answerType별 프롬프트 분기 추가 여부 검토 |
| `panels.tsx` | `QuestionInputArea` 분기 추가 여부 검토 |

### 튜터링↔백오피스 DB 동기화 주의

```
백오피스 seed.ts → 백오피스 PostgreSQL만 업데이트
튜터링 API → 별도 Supabase DB ai_modules 테이블 조회 (lessons.service.ts:307)

→ 모듈 설정 변경 시 Supabase에도 직접 PATCH 필요 (현재 수동)
→ 자동 동기화 미구현 (기술 부채)
```
