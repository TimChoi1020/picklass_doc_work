# Picklass Tutoring — Agent 아키텍처 컨텍스트 (2026-04-07 업데이트)

> 이 문서는 AI Agent 설계 관련 논의 내용을 정리한 작업 컨텍스트입니다.
> 새 채팅 세션에서 바로 작업 재개가 가능하도록 결정사항, 현재 코드 상태, 미완료 작업을 포함합니다.
>
> **최종 업데이트**: 2026-04-07 — **문항 생성 아키텍처 Step 1~2 완료** (LLM 문항 생성 전략 타입/스키마/UI 추가)

---

## 1. 핵심 설계 원칙 (R4)

### R4 요약 (dev-plan-20260405.md 기준)
- **Single Source of Truth**: `ai_modules` 테이블이 모듈의 유일한 정의
- **모듈별 하드코딩 금지**: 프론트엔드에 `moduleCode === 'PWR'` 같은 분기 금지
- **GenericAdapter**: 모든 모듈은 DB 데이터 기반으로 동작, 전용 어댑터 파일 불필요
- **uiTemplate**: 페이지 수준 UI 구조 결정 (5가지 값)
- **answerType**: 문항 패널 내부 분기 결정 (`QuestionsPanel` 자동 렌더)

### uiTemplate 5가지 값 (`ai-module-data.ts` 기준)

| uiTemplate | UI 구조 | 구현 컴포넌트 | 해당 모듈 | 상태 |
|---|---|---|---|---|
| `standard` | 지문 + 문항 패널 + 피드백 (Orchestrator 기반) | `ModuleRunnerInner` | PRD, SKM, QAR, CLR, SUM, SWR 등 | ✅ 구현 완료 |
| `timed-gate` | 읽기 타이머 → 지문 숨김 → 문항 | `ScanningGate` → `ModuleRunnerInner` | SCN | ✅ 구현 완료 |
| `step-workflow` | 단계별 수동 흐름 (Orchestrator 미사용) | `ProcessWritingFlow` | PWR | ✅ 구현 완료 (범용화 미완료) |
| `vocab-deck` | 플래시카드 덱 UI | `VocabDeckFlow` | WRD, WSD 등 | ✅ 신규 구현 완료 |
| `interactive` | 드래그앤드롭, 매칭, 대화 UI | `InteractiveFlow` | IMG, WW, SNR 등 | ✅ 신규 구현 완료 |

> **LessonSession.tsx 분기 코드** (현재):
> ```typescript
> const uiTemplate = moduleData.uiTemplate ?? resolveDefaultUiTemplate(moduleData.code);
> switch (uiTemplate) {
>   case 'step-workflow': return <ProcessWritingFlow ... />;
>   case 'timed-gate':    return <ScanningGate ... />;
>   case 'vocab-deck':    return <VocabDeckFlow ... />;
>   case 'interactive':   return <InteractiveFlow ... />;
>   default:              return <ModuleRunnerInner ... />;  // 'standard'
> }
> ```

---

## 2. PedagogyProfile 기반 재설계 ✅ 완료 (2026-04-07)

### 2-1. 재설계 목적

기존 `ContentConfig`(7단계 JSONB) 기반 UI 흐름 제어 방식은 다음 문제가 있었다:
- Orchestrator가 `stage1~7Enabled`, `stageItems` 등 처방적 설정값을 따라가는 **수동적 역할**에 그침
- 신규 모듈 추가 시마다 `ContentConfig`를 하드코딩해야 함 (R4 위반)
- `stage1Greeting` 등 고정 문자열이 LLM의 자율적 피드백 생성을 방해

**목표**: `ContentConfig` 의존성을 UI 흐름에서 완전히 제거하고, Orchestrator가 `ModulePedagogyProfile` 필드 + `pedagogyInstruction`(LLM 시스템 프롬프트)만으로 자율적으로 수업을 이끌어가도록 전환.

```
기존 (처방적 방식):
  ContentConfig.stage1Greeting → Orchestrator가 문자열 그대로 출력
  ContentConfig.stage1~7Enabled → UI 흐름 단계 결정
  ContentConfig.retryThreshold → 재도전 임계값

새 방식 (자율적):
  pedagogyInstruction → LLM이 맥락에 맞게 인사말/피드백 동적 생성
  scoringMode + retryScope → Orchestrator가 상황 판단하여 자율 제어
  questionMaxAttempts → 시도 횟수 자율 관리
```

### 2-2. 7단계 작업 내역 (모두 완료)

| 단계 | 작업 내용 | 핵심 변경 파일 |
|---|---|---|
| **Step 1** | `ChatMessage` 타입을 `useStageStateMachine.ts` → `agent-types.ts`로 이동. 순환 참조 해소 | `agent-types.ts`, `useStageStateMachine.ts` |
| **Step 2** | `OrchestratorContext.moduleConfig: ContentConfig` → `pedagogyProfile: ModulePedagogyProfile` 교체 | `agent-types.ts` |
| **Step 3** | `ModuleOrchestratorAgent`에서 `context.moduleConfig.stage1Greeting` → `context.pedagogyProfile.purpose` 기반 동적 생성으로 교체 | `ModuleOrchestratorAgent.ts:463` |
| **Step 4** | `useModuleOrchestrator` 입력 타입 `ModuleOrchestratorInput.moduleConfig: ContentConfig` → `pedagogyProfile: ModulePedagogyProfile` 변경 | `useModuleOrchestrator.ts` |
| **Step 5** | `LessonSession.tsx`에서 `config` 상태 변수 및 `ContentConfig` import 완전 제거. `ScanningGate`, `ModuleRunnerInner` 호출부 모두 `pedagogyProfile` 전달로 변경 | `LessonSession.tsx` |
| **Step 6** | 모든 어댑터에서 `buildContentConfig()` 제거. `ModuleAdapter` 인터페이스 시그니처 삭제, Legacy 어댑터 5개 + `GenericAdapter` 메서드 제거 | `ModuleAdapter.ts`, `GenericAdapter.ts`, `PredictionAdapter.ts`, `ScanningAdapter.ts`, `ProcessWritingAdapter.ts`, `SentenceWritingAdapter.ts`, `ShadowReadingAdapter.ts` |
| **Step 7** | `useStageStateMachine.ts`, `usePanelRouter.ts` 삭제. `ai-module.ts`에서 `PanelAssignment` 인터페이스 및 `getEnabledStages()`, `getStageItems()`, `assignToPanels()` helper 함수 제거 | 파일 2개 삭제, `ai-module.ts` 정리 |

### 2-3. ContentConfig 처리 방침 (최종)

| 위치 | 처리 결과 | 이유 |
|---|---|---|
| `OrchestratorContext.moduleConfig` | ✅ 제거 → `pedagogyProfile`로 교체 | UI 흐름 제어 불필요 |
| `ModuleAdapter.buildContentConfig()` | ✅ 제거 | 어댑터가 반환할 필요 없음 |
| `useStageStateMachine.ts`, `usePanelRouter.ts` | ✅ 파일 삭제 | `ContentConfig` 의존 파일, 외부 참조 없음 |
| `ai-module.ts` — `PanelAssignment`, helper 함수 | ✅ 제거 | `usePanelRouter.ts` 삭제로 참조처 소멸 |
| `ai-module.ts` — `ContentConfig`, `StageItem` 타입 | **유지** | `AiModuleData.contentConfig` 필드 타입으로 사용 중 |
| `ai-module-data.ts` — `contentConfig: ContentConfig` | **유지** | DB/API 스키마 목적 (백오피스 JSONB 필드 수신용) |

> `contentConfig`는 DB 레코드에 계속 존재하지만, **튜터링 UI 흐름에서는 일절 참조하지 않는다**. 백오피스가 JSONB 데이터를 관리·전달하는 역할은 유지.

---

## 3. PedagogyProfile 아키텍처

### 두 레이어 역할 분리

```
[Orchestrator — 규칙 기반 흐름 제어]
  scoringMode         → 어떻게 평가할지 (exact / holistic / pronunciation)
  questionMaxAttempts → 문항당 최대 시도 횟수
  retryScope          → 재도전 기준 지표
  answerType          → 어떤 입력 UI를 쓸지
  passageExposureMode → 지문을 어떻게 노출할지
  questionCount       → 단일/다중 문항 여부

[LLM — 자율 피드백 생성]
  pedagogyInstruction → 시스템 프롬프트 (피드백 구조, 예외 처리, 톤 지시)
  purpose             → 모듈 목적 요약 (greeting 생성에 활용)
  feedbackStyle       → 피드백 스타일 (correct-wrong / strengths-weaknesses)
```

### 현재 `ModulePedagogyProfile` 인터페이스 (`agent-types.ts:160`)

```typescript
export interface ModulePedagogyProfile {
  scoringMode:          'exact' | 'holistic' | 'pronunciation';
  answerType:           'multiple-choice' | 'essay' | 'short-text' | 'mixed' | 'audio-record' | 'sentence-write';
  passageExposureMode:  'hidden' | 'preview' | 'full';
  questionCount:        'single' | 'multi';
  feedbackStyle:        'correct-wrong' | 'strengths-weaknesses';
  purpose:              string;
  questionMaxAttempts?: number;
  hintTypes:            HintType[];
  retryScope:           RetryScope;
  inputLanguage:        InputLanguage;
  passageRole:          PassageRole;
  pedagogyInstruction:  string;
}
```

### `OrchestratorContext` 현재 구조 (`agent-types.ts:440`)

```typescript
export interface OrchestratorContext {
  lessonPlan:            LessonPlan;
  currentPlannedModule:  PlannedModule;
  pedagogyProfile:       ModulePedagogyProfile;  // ← contentConfig 완전 대체
  passage:               PassageData;
  questions:             QuestionData[];
  learnerState:          LearnerState;
  recentMessages:        ChatMessage[];
  passageShown:          boolean;
  passagePreviewOnly:    boolean;
  // ... 기타 상태 필드
}
```

### `ModulePedagogyProfile` 향후 확장 필요 필드

현재 enum 값이 일부 모듈의 실제 동작을 커버하지 못한다:

| 필드 | 현재 값 | 향후 필요 추가 값 |
|---|---|---|
| `scoringMode` | `exact \| holistic \| pronunciation` | `binary`, `asr`, `keyword-match`, `semantic`, `rubric`, `click-match` |
| `answerType` | `multiple-choice \| essay \| short-text \| mixed \| audio-record \| sentence-write` | `voice`, `keyword-list`, `click`, `mcq-with-evidence` |
| `passageExposureMode` | `hidden \| preview \| full` | `highlight`, `blurred`, `sentence-by-sentence`, `minimized` |
| `retryThreshold` | 없음 (retryScope 열거형만 있음) | `number` (%) 필드 추가 필요 |
| `interventionDelaySec` | 없음 | `number` (초) 필드 추가 필요 |

---

## 4. 모듈별 PedagogyProfile 현황

`apps/web/src/lib/pedagogy/picklass_pedagogy_profiles.ts` 기준 전체 모듈 목록 (총 13개, 확정):

| 모듈코드 | 모듈명 | Skill | scoringMode | answerType | passageExposureMode | maxAttempts |
|---|---|---|---|---|---|---|
| **WRD** | Word Reading Decker | Voca | binary | short-text | highlight | 3 |
| **WSD** | Word Speaking Decker | Voca | asr | voice | highlight | 3 |
| **GMN** | Guessing Meaning | Reading | holistic | essay | highlight | 1 |
| **WWB** | Word Web | Reading | semantic | short-text | minimized | 1 |
| **PRD** | Prediction | Reading | holistic | essay | preview | 1 |
| **SCN** | Scanning | Reading | keyword-match | keyword-list | blurred | 1 |
| **SKM** | Skimming | Reading | click-match | click | full | 1 |
| **CLR** | Clarification | Reading | holistic | click | sentence-by-sentence | undefined |
| **SUM** | Summarizing | Reading | rubric | essay | full | 1 |
| **QAR** | QAR | Reading | rubric | mcq-with-evidence | full | 1 |
| **RRD** | Repeated Reading | Reading | asr | voice | highlight | 3 |
| **SWR** | Sentence Writing | Writing | holistic | sentence-write | full | 1 |
| **PWR** | Process Writing | Writing | holistic | essay | full | 1 |

---

## 5. 모듈 코드 확정 현황

`picklass_pedagogy_profiles.ts` 기준 코드가 확정 기준. dev-plan-20260405.md의 일부 코드와 차이가 있으며, **profiles.ts 코드를 정본으로 사용**.

| 모듈명 | 확정 코드 (profiles.ts) | dev-plan 기재 코드 | 비고 |
|---|---|---|---|
| Word Reading Decker | **WRD** | WRD | 일치 ✅ |
| Word Speaking Decker | **WSD** | WSD | 일치 ✅ |
| Guessing Meaning | **GMN** | IMG | ⚠️ 불일치 — GMN 사용 |
| Word Web | **WWB** | WW | ⚠️ 불일치 — WWB 사용 |
| Repeated Reading | **RRD** | ORL | ⚠️ 불일치 — RRD 사용 |

> **처리 방향**: 백오피스 `ai_modules` 테이블에 모듈 등록 시 **profiles.ts 코드(WRD, WSD, GMN, WWB, RRD)** 로 등록한다. dev-plan의 IMG/WW/ORL 코드는 더 이상 사용하지 않는다.

---

## 6. 주요 파일 위치 정리 (현재 상태)

```
apps/web/src/
├── lib/
│   ├── adapters/
│   │   ├── index.ts                    # 어댑터 레지스트리 + createAdapterForModule()
│   │   ├── GenericAdapter.ts           # DB 기반 범용 어댑터 (R4 핵심)
│   │   ├── ModuleAdapter.ts            # 어댑터 인터페이스 (buildContentConfig 제거됨)
│   │   ├── PredictionAdapter.ts        # Legacy (PRD mock, buildContentConfig 제거됨)
│   │   ├── ScanningAdapter.ts          # Legacy (SCN mock, buildContentConfig 제거됨)
│   │   ├── ShadowReadingAdapter.ts     # Legacy (RRD mock, buildContentConfig 제거됨)
│   │   ├── SentenceWritingAdapter.ts   # Legacy (SWR mock, buildContentConfig 제거됨)
│   │   └── ProcessWritingAdapter.ts    # Legacy (PWR mock, buildContentConfig 제거됨)
│   ├── agents/
│   │   ├── agent-types.ts              # ChatMessage, ModulePedagogyProfile, OrchestratorContext 등 핵심 타입
│   │   └── orchestrator/
│   │       └── ModuleOrchestratorAgent.ts  # 규칙 기반 Orchestrator (pedagogyProfile 기반)
│   ├── types/
│   │   ├── ai-module.ts                # ContentConfig, StageItem 타입 (PanelAssignment 제거됨)
│   │   ├── ai-module-data.ts           # AiModuleData + UiTemplate (DB → TypeScript)
│   │   └── module.ts                   # ModuleData, QuestionData
│   ├── services/
│   │   └── aiModuleService.ts          # DB 응답 → AiModuleData 변환
│   ├── pedagogy/
│   │   └── picklass_pedagogy_profiles.ts  # 전체 모듈 PedagogyProfile 중앙 정의 (13개)
│   └── hooks/
│       └── useModuleOrchestrator.ts    # React Hook (pedagogyProfile 기반으로 전환 완료)
│       ※ useStageStateMachine.ts — 삭제됨 (2026-04-07)
│       ※ usePanelRouter.ts — 삭제됨 (2026-04-07)
└── app/
    └── modules/[lessonId]/
        └── _components/
            ├── LessonSession.tsx        # uiTemplate switch 분기 (config 상태 완전 제거)
            ├── ProcessWritingFlow.tsx   # step-workflow Flow (범용화 대상)
            ├── VocabDeckFlow.tsx        # vocab-deck Flow (신규, PedagogyProfile 기반)
            ├── InteractiveFlow.tsx      # interactive Flow (신규, PedagogyProfile 기반)
            └── panels.tsx              # ContentPanel, QuestionsPanel, FeedbackPanel 등

docs/
├── dev-plan-20260405.md                # R4 전체 설계 문서 (가장 중요)
├── picklass-tutoring-planning.md       # 기획/개발자 대상 구조 설명 (v1.1)
└── agent-architecture-context-20260406.md  # 이 파일
```

---

## 7. Orchestrator 자율화 설계 방향

### Phase 1 진행 상황 (2026-04-07 완료)

```
기존 (contentConfig 처방적 방식):
  stage1 → 콘텐츠UI 표시
  stage2 → 문항UI 표시
  stage3 → 피드백UI 표시
  stage5 → 재도전 여부 결정
  → Orchestrator가 config를 따라가는 수동적 역할

새 방식 (PedagogyProfile 자율적) ← 지금 구현 중:
  passageExposureMode: 'full'   → 스마트 지문 노출 (Q: ruleInitialEntry)
  questionMaxAttempts: 3        → 시도 횟수 자율 관리 (Q: ruleRepeatWrongAnswer)
  retryThreshold: 70            → 재도전 기준 자율 판정 (Q: ruleCompleteAfterCelebrate)
  hintTypes: ['subtle', ...]    → hint 선택 자동화 (Q: ruleRepeatWrongAnswer)
  pedagogyInstruction           → LLM이 피드백을 자율 생성 (Q: Phase 1-5 예정)
  purpose                       → LLM이 인사말을 동적 생성 (Q: ruleInitialEntry)
  → Orchestrator가 상황을 읽고 스스로 결정하는 능동적 역할
```

### Phase 1 규칙별 개선 사항

| 규칙 | 변경 내용 | 파일 줄 | 상태 |
|---|---|---|---|
| `ruleInitialEntry()` | `q.type` 체크 제거 → `pedagogyProfile.passageExposureMode` 기반 switch | 452-504 | ✅ 1-4 |
| `ruleRepeatWrongAnswer()` | hardcoded 2→3 시도 제거 → `questionMaxAttempts` 동적 계산, `hintTypes[]` 배열 선택 | 133-206 | ✅ 1-1,1-3 |
| `ruleCompleteAfterCelebrate()` | hardcoded 50% 제거 → `retryThreshold` 기반 비교 | 408-440 | ✅ 1-2 |
| `giveFeedback()` | (아직 구현 안 됨) → `pedagogyInstruction` + Gemini API 호출 | TBD | ⏳ 1-5 |

### `ModuleOrchestratorAgent.ts` 현재 greeting 생성 방식

```typescript
// 변경 전 (step 3 이전):
const greeting = context.moduleConfig.stage1Greeting || undefined;

// 변경 후 Phase 1-4 (현재):
const greeting = context.pedagogyProfile.purpose
  ? `${context.passage.title} — ${context.pedagogyProfile.purpose}`
  : undefined;

// Phase 1-5 예정 (LLM):
const greeting = await claudeAPI.generateGreeting({
  systemPrompt: context.pedagogyProfile.pedagogyInstruction,
  passage: context.passage,
  purpose: context.pedagogyProfile.purpose,
});
```

---

## 8. Orchestrator 규칙 이름 vs 실제 로직

`ModuleOrchestratorAgent.ts`에서 규칙 이름이 모듈명으로 되어 있지만
실제 조건은 `q.type` (answerType) 기반이므로 기능상 범용적이다.

```typescript
// 이름은 RRD/SWR이지만 실제로는 answerType 체크 → 범용
'SHR-A': q.type === 'audio-record'    // 음성 녹음 모듈 전체에 적용 (규칙명은 레거시)
'SHR-B': q.type === 'audio-record'
'SHR-C': q.type === 'audio-record'
'SWR-A': q.type === 'sentence-write'  // 영작 모듈 전체에 적용
```

향후 `retryThreshold`, `interventionDelaySec` 필드가 DB에서 직접 공급되면
이 규칙들도 모듈 코드 불문 동일한 로직으로 처리 가능.

---

## 9. GenericAdapter 현재 동작 방식

```typescript
// createAdapterForModule() 호출 흐름 (adapters/index.ts)
if (aiModuleData) → new GenericAdapter(aiModuleData)   // DB 데이터 있으면 무조건 GenericAdapter
else if (registry[code]) → 기존 어댑터 (Legacy fallback)
else → PRD fallback

// GenericAdapter 내부 (buildContentConfig 제거 후)
buildPedagogyProfile()  → ai_modules 필드 직접 구성 (DB 기반) ← 핵심
buildAgentPrompt()      → pedagogyInstruction을 system 프롬프트에 주입
calculateScore()        → scoringMode에 따라 분기 (exact/holistic/pronunciation)
buildKpis()             → selectedKpiCodes 기반 KPI 계산
```

---

## 10. 완료된 작업 이력

| 날짜 | 작업 내용 |
|---|---|
| 2026-04-06 | `picklass-tutoring-planning.md` v1.1 업데이트 (AI 서비스 정보, Tool schemas, Streaming 등) |
| 2026-04-07 | `VocabDeckFlow`, `InteractiveFlow` 신규 컴포넌트 구현 (PedagogyProfile 기반) |
| 2026-04-07 | PedagogyProfile 기반 재설계 7단계 완료 (ContentConfig UI 흐름 의존성 전면 제거) |
| 2026-04-07 | `uiTemplate` 타입 `ai-module-data.ts`에 정의, `LessonSession.tsx` switch 분기 완성 |
| **2026-04-07** | **Orchestrator 자율화 Phase 1-0~1-4 전체 완료** (총 5개 파일 수정): `agent-types.ts`, `ai-module-data.ts`, `picklass_pedagogy_profiles.ts`, `ModuleOrchestratorAgent.ts` × 4회 |
| **2026-04-07** | **LLM 문항 생성 아키텍처 Step 1 완료** — 타입/스키마 변경 (총 11개 파일 수정) |
| **2026-04-07** | **LLM 문항 생성 아키텍처 Step 2 완료** — 백오피스 등록/수정 UI에 문항 생성 전략 필드 추가 |

---

---

## 11. LLM 문항 생성 아키텍처

### 11-1. 설계 원칙

모든 문항 콘텐츠는 LLM이 생성한다. 사전 등록 문항 없음. 지문 무관 공통 문항 없음. 단어·문장·객관식 모두 지문에서 LLM이 추출하거나 지시문으로 생성한다.

**생성된 문항은 DB에 캐시한다.** 동일 지문(`text_id`) + 동일 모듈(`module_id`) 조합은 LLM을 반복 호출하지 않는다. 첫 번째 학습자가 생성을 트리거하고, 이후 학습자는 `module_questions` 테이블 캐시를 재사용한다.

```
요청 (module_id × text_id)
    ↓
module_questions 테이블 조회 (status='active')
    ↓ 캐시 히트                    ↓ 캐시 미스
즉시 QuestionData[] 반환    POST /api/ai/generate-questions
                                 ↓
                            Gemini가 지문 분석
                            + 전략(extract/instruct)
                            + min~max 범위 내 문항 수 결정
                                 ↓
                            module_questions 저장 (status='active')
                                 ↓
                            QuestionData[] 반환
```

---

### 11-2. 문항 생성 전략 (2가지)

| 전략 | 값 | 설명 | 해당 모듈 |
|---|---|---|---|
| 추출형 | `extract` | 지문에서 콘텐츠(단어·문장·선택지)를 LLM이 직접 추출 | WRD, WSD, GMN, WWB, SKM, QAR, RRD, SWR |
| 지시형 | `instruct` | 지문 맥락 기반 지시문 LLM 생성. 학생 자유 응답 → holistic 평가 | PRD, CLR, SUM, PWR |

---

### 11-3. 문항 수 범위 관리

`ai_modules` 테이블에 `question_min_count`, `question_max_count` 두 컬럼으로 관리한다.

- `questionCount === 'single'` 모듈: 항상 1 (min/max 참조 불필요)
- `questionCount === 'multi'` 모듈: LLM이 지문 난이도·길이를 분석하여 범위 내에서 최적 수 결정

**LLM 판단 기준:**
- 지문 길이 — 길수록 max에 가깝게
- 어휘 난이도 — 고급 어휘 밀도 높으면 수 줄이기
- 핵심 아이디어 수 — 많을수록 수 늘리기
- 문장 구조 복잡도 — 복잡하면 수 줄이기

**모듈별 기준값 (picklass_pedagogy_profiles.ts 정의값):**

| 모듈 | questionCount | strategy | min | max |
|---|---|---|---|---|
| WRD, WSD | multi | extract | 5 | 10 |
| GMN, WWB | multi | extract | 2 | 5 |
| PRD, SUM | single | instruct | 1 | 1 |
| SCN | single | instruct | 1 | 1 |
| SKM | multi | extract | 2 | 4 |
| QAR | multi | extract | 3 | 5 |
| CLR | multi | instruct | 3 | 6 |
| RRD | multi | extract | 3 | 6 |
| SWR | multi | extract | 2 | 5 |
| PWR | multi | instruct | 4 | 4 |

---

### 11-4. Step 1 — 타입/스키마 변경 ✅ 완료 (2026-04-07)

**변경된 파일 (총 11개):**

| 파일 | 변경 내용 |
|---|---|
| `apps/web/src/lib/agents/agent-types.ts` | `ModulePedagogyProfile`에 `questionGenerationStrategy`, `questionMinCount`, `questionMaxCount` 추가. `questionCount` 주석 명확화 (single/multi 역할 분리) |
| `apps/web/src/lib/types/ai-module-data.ts` | `AiModuleData`에 동일 3개 필드 추가 |
| `apps/web/src/lib/services/aiModuleService.ts` | `RawAiModuleResponse` + `parseQuestionGenerationStrategy()` 파서 + `parseAiModuleResponse()` 반영 |
| `apps/web/src/lib/adapters/GenericAdapter.ts` | `buildPedagogyProfile()`에 3개 필드 추가 |
| `apps/web/src/lib/adapters/PredictionAdapter.ts` | `buildPedagogyProfile()`에 `instruct`, min=1, max=1 추가 |
| `apps/web/src/lib/adapters/ScanningAdapter.ts` | `buildPedagogyProfile()`에 `instruct`, min=1, max=1 추가 |
| `apps/web/src/lib/adapters/ShadowReadingAdapter.ts` | `buildPedagogyProfile()`에 `extract`, min=3, max=6 추가 |
| `apps/web/src/lib/adapters/SentenceWritingAdapter.ts` | `buildPedagogyProfile()`에 `extract`, min=2, max=5 추가 |
| `apps/web/src/lib/adapters/ProcessWritingAdapter.ts` | `buildPedagogyProfile()`에 `instruct`, min=4, max=4 추가 |
| `apps/web/src/lib/pedagogy/picklass_pedagogy_profiles.ts` | 13개 모듈 전체 프로파일에 3개 필드 추가 |
| `packages/types/src/index.ts` (backoffice) | `AiModuleResponse`, `CreateAiModuleDto`, `UpdateAiModuleDto` 모두 반영 |
| `packages/core/src/ai-module/ai-module.service.ts` (backoffice) | create/update/toResponse 반영 |
| `apps/api/src/lessons/lessons.service.ts` (tutoring) | `getModuleMeta()`, `getModuleData()` 반영 |

**DB 스키마 변경 (데이터 유지, 컬럼만 추가):**

```sql
-- backoffice + tutoring 양쪽 ai_modules 테이블에 추가
ALTER TABLE ai_modules ADD COLUMN question_generation_strategy VARCHAR(20) NOT NULL DEFAULT 'extract';
ALTER TABLE ai_modules ADD COLUMN question_min_count INT NOT NULL DEFAULT 1;
ALTER TABLE ai_modules ADD COLUMN question_max_count INT NOT NULL DEFAULT 1;
```

---

### 11-5. Step 2 — 백오피스 UI 추가 ✅ 완료 (2026-04-07)

**변경 파일:** `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

**추가된 UI (문항설계2 탭 내):**

1. **문항 생성 전략** (`questionGenerationStrategy`) — 카드형 라디오 버튼
   - `extract`: 지문에서 콘텐츠 추출 (카드에 설명 + 해당 모듈 예시 표시)
   - `instruct`: 지시문 LLM 생성 (카드에 설명 + 해당 모듈 예시 표시)

2. **문항 수 범위** (`questionMinCount` / `questionMaxCount`) — 숫자 입력 2개
   - `questionCount === 'single'`이면 입력 비활성화 + 경고 문구 표시
   - `questionMaxCount` 입력 시 `questionMinCount` 이상 자동 보정

---

## 12. 다음 작업 목록 (우선순위 순)

### ✅ 완료됨 (2026-04-07) — Orchestrator 자율화 Phase 1-0~1-4

✅ **Phase 1 전체 완료** (`ModuleOrchestratorAgent.ts`에서 pedagogyProfile 기반 제어 활성화)
- Phase 1-0: `retryThreshold` 필드 추가 (agent-types.ts, ai-module-data.ts, picklass_pedagogy_profiles.ts에 모든 13개 모듈에 값 입력)
- Phase 1-1: `ruleRepeatWrongAnswer()` → `questionMaxAttempts` 기반으로 하intTrigger 동적 계산 (hardcoded 2회→3회 제거)
- Phase 1-2: `ruleCompleteAfterCelebrate()` → `retryThreshold` 기반 재도전 판정 (hardcoded 50%→module별 동적값)
- Phase 1-3: `giveHint` params에 `hintType` 추가, `hintTypes[]` 배열에서 실제 hint 유형 선택 (subtle/direct/explicit → array index mapping)
- Phase 1-4: `ruleInitialEntry()` → `passageExposureMode` 기반 switch 분기 (q.type 체크 제거, DB설정 기반 지문노출)

**현재 상태**: Orchestrator가 pedagogyProfile의 **6개 필드를 실제로 사용 중**
- ✅ passageExposureMode (hidden/preview/full)
- ✅ questionMaxAttempts (1/3/undefined)
- ✅ hintTypes (배열)
- ✅ retryThreshold (%)
- ✅ purpose (greeting 생성)
- ✅ pedagogyInstruction (피드백 프롬프트 템플릿)

---

### �🔴 즉시 필요 — 타입 확장 (신규 모듈 지원)

**1. `ModulePedagogyProfile` 인터페이스 확장** (`agent-types.ts`) — **부분 완료**
- ✅ ~~`retryThreshold: number` 필드~~ (Phase 1-0 완료)
- ⏳ `interventionDelaySec: number` 필드 추가 (힌트 개입 딜레이)
- ⏳ `scoringMode` 값 추가: `binary`, `asr`, `keyword-match`, `semantic`, `rubric`, `click-match`
- ⏳ `answerType` 값 추가: `voice`, `keyword-list`, `click`, `mcq-with-evidence`
- ⏳ `passageExposureMode` 값 추가: `highlight`, `blurred`, `sentence-by-sentence`, `minimized`

**2. `AiModuleData` 타입 동기화** (`types/ai-module-data.ts`) — **부분 완료**
- ✅ ~~`retryThreshold` 필드~~ (Phase 1-0 완료)
- ⏳ `interventionDelaySec` 등 신규 필드들을 DB 타입에 반영
- ✅ ~~`contentConfig` 필드~~ (현재 유지 중, "DB 스키마 목적, UI 흐름 미사용" 주석 명시됨)

**3. backoffice DB 마이그레이션** — **부분 진행 중**
- ✅ ~~`retryThreshold` 컬럼 추가~~ (picklass_pedagogy_profiles.ts에서 모든 13개 모듈에 값 정의됨)
- ⏳ `interventionDelaySec` 컬럼 추가
- ⏳ `scoringMode`, `answerType`, `passageExposureMode` ENUM 값 확장 (code_groups)
- ⏳ `uiTemplate` 필드 백오피스 등록 UI에 반영 (탭1 select 추가)

### 🟡 Phase 1-5 (예정) — Gemini API 연동으로 Orchestrator 자율화 마무리

**4. 동적 피드백 생성** (pedagogyInstruction LLM 프롬프트 활용)
- **상태**: Orchestrator가 pedagogyProfile.pedagogyInstruction을 읽기만 함 (아직 LLM 호출 없음)
- **목표**: 다음 규칙들이 `pedagogyInstruction`을 시스템 프롬프트로 Gemini API 호출
  - `rulePronunciationFeedback()` → `giveFeedback` 도구에서 Gemini API 호출해 동적 피드백 생성
  - `ruleWritingFeedback()` → `giveFeedback` 도구에서 Gemini API 호출해 동적 피드백 생성
  - `ruleHolisticFeedback()` → `giveFeedback` 도구에서 Gemini API 호출해 동적 피드백 생성
- **사전조건**: NestJS 백엔드에서 `POST /api/agent/feedback` 엔드포인트 구현 (백엔드 GeminiService 구현 완료)

### 🟡 R4 마무리 — Legacy 어댑터 전환

**5. `resolveDefaultUiTemplate()` 제거** (`LessonSession.tsx:129-135`)
- 현재: `PWR → step-workflow`, `SCN → timed-gate` 하드코딩 fallback 함수 존재
- 조건: Legacy 어댑터들이 `uiTemplate`을 반환하도록 수정하거나, GenericAdapter 전환이 완료된 후 제거
- **지금 바로 제거하면 Legacy 어댑터 사용 시 동작이 깨짐 — 전환 완료 후 삭제**

**6. Legacy 어댑터 5개 GenericAdapter로 전환 후 삭제**
- `PredictionAdapter.ts`, `ScanningAdapter.ts`, `ShadowReadingAdapter.ts`
- `SentenceWritingAdapter.ts`, `ProcessWritingAdapter.ts`
- 전환 조건: DB에서 해당 모듈 데이터가 실제 서빙될 때

**7. `ProcessWritingFlow` 범용화** (`step-workflow` 일반화)
- 현재: 4단계(Outlining → Self-Check → 1st Draft → Final Draft) 하드코딩
- 목표: `contentConfig.steps[]` 또는 `PedagogyProfile` 필드에서 단계 정의 읽기
- dev-plan 설계: `StepWorkflowFlow`로 rename, DB 단계 데이터 기반으로 범용화

### 🟢 신규 모듈 DB 등록

**Phase 1 — DB 등록만으로 동작 가능 (코드 변경 없음)**
- `GMN` (Guessing Meaning): `uiTemplate=standard`, `answerType=essay`
- `SUM` (Summarizing): `uiTemplate=standard`, `answerType=essay`

**Phase 2 — 소규모 UI 코드 추가 필요**
- `SKM` (Skimming): `answerType=click` → 문장 클릭 UI 추가
- `QAR` (QAR): `answerType=mcq-with-evidence` → 근거 문장 클릭 UI 추가
- `RRD` (Repeated Reading): `uiTemplate=vocab-deck` → VocabDeckFlow 음성 모드 확인

**Phase 3 — 새 Flow 컴포넌트 구현 필요**
- `CLR` (Clarification): 문장 단위 네비게이션 UI (`sentence-by-sentence`)
- `WRD/WSD` (Word Reading/Speaking Decker): VocabDeckFlow 실제 연결 확인
- `WWB/GMN` (Word Web, Guessing Meaning): InteractiveFlow 세부 타입 구현

### 🔵 인프라 / API 연동

**7. 튜터링 백엔드 NestJS API 구현**
- `GET /api/lessons/:lessonId/module-data?moduleCode=XXX` — 지문 + 문항 서빙
- `GET /api/lessons/:lessonId/plan` — LessonPlan 서빙 (현재 Mock)
- `POST /api/agent/feedback` — Gemini API 연동 AI 피드백 (Phase 1-5에서 필요)
- `POST /api/agent/kpi/:moduleCode` — KPI 계산
- `POST /api/module-history` — 학습 이력 저장

**9. AI 실제 연동** (Phase 1-5 이후)
- **Gemini 2.5 Flash** 연동 — `pedagogyInstruction` 기반 AI 피드백 (백엔드 GeminiService 구현 완료, 프론트 Mock→API 연동 필요)
- **Azure Speech SDK** — 발음 평가(PronunciationService) + TTS(TtsService) 백엔드 구현 완료, 프론트 Mock→API 연동 필요
  - `submitVoiceAnswer()`: 현재 70~100 랜덤 점수 → `POST /api/ai/pronunciation` 호출로 교체
  - `playModelAudio()`: 현재 2500ms 딜레이 시뮬레이션 → `POST /api/ai/tts` 호출로 교체

---

### 🔴 LLM 문항 생성 아키텍처 Step 3~5 (미완료)

---

#### Step 3 — 백엔드 문항 생성 API 구현 (`POST /api/ai/generate-questions`)

**파일:** `apps/api/src/ai/ai.controller.ts`, 신규 `apps/api/src/ai/question-generator.service.ts`

**목적:** `module_questions` 캐시 미스 시 Gemini가 지문을 분석하여 문항을 생성하고 DB에 저장한다.

**Request Body:**
```typescript
interface GenerateQuestionsDto {
  moduleId: string;           // ai_modules.id
  textId: number;             // texts.id (지문 식별자)
  passage: {
    title: string;
    content: string;
  };
  strategy: 'extract' | 'instruct';
  minCount: number;
  maxCount: number;
  answerType: string;         // 문항 UI 타입 참고용
  pedagogyInstruction: string; // Gemini system 프롬프트
}
```

**Response:**
```typescript
interface GenerateQuestionsResponse {
  passageDifficultyLevel: 'beginner' | 'intermediate' | 'advanced';
  decidedCount: number;        // min~max 범위 내 LLM 결정값
  questions: {
    question_number: number;
    type: string;              // QuestionType
    text: string;
    options: string[] | null;  // multiple-choice 전용
    answer: string;
    hint: string | null;
  }[];
}
```

**Gemini 프롬프트 구조:**

`extract` 전략:
```
[시스템]
당신은 영어 교육 전문 문항 생성 AI입니다.
아래 지문을 분석하여 {strategy} 방식으로 {minCount}~{maxCount}개 범위에서 최적 문항 수를 결정하고 문항을 생성하세요.

[지문 난이도 판단 기준]
- beginner: 짧고 단순한 문장, 기본 어휘
- intermediate: 중간 길이, 일반 어휘, 복합문
- advanced: 긴 지문, 전문 어휘, 복잡한 구조

[extract 생성 규칙]
- answerType이 'audio-record' / 'short-text': 지문에서 핵심 문장/단어 직접 추출
- answerType이 'multiple-choice': 지문 기반 선택지 4개 + 정답 1개 구성
- answerType이 'sentence-write': 핵심 문장을 한글로 패러프레이징하여 text 필드에 작성, answer는 영어 원문

[출력 형식] JSON만 출력 (설명 없음)
```

`instruct` 전략:
```
[instruct 생성 규칙]
- 지문 제목·핵심 내용을 반영한 자연스러운 지시문 1~N개 생성
- 매번 조금씩 다른 표현으로 단조로움 방지 (같은 지문이어도 변형 가능)
- text 필드: 학생에게 보여줄 지시문 전체
- answer 필드: 빈 문자열 (holistic 채점)
```

**DB 저장 로직:**
```typescript
// question-generator.service.ts
async generateAndSave(dto: GenerateQuestionsDto): Promise<module_questions[]> {
  // 1. Gemini 호출
  const result = await this.geminiService.generateQuestions(prompt);

  // 2. module_questions 일괄 insert (기존 데이터 유지)
  const saved = await this.prisma.module_questions.createMany({
    data: result.questions.map((q, i) => ({
      module_id: dto.moduleId,
      text_id: dto.textId,
      question_number: i + 1,
      type: q.type,
      text: q.text,
      options: q.options ?? undefined,
      answer: q.answer,
      hint: q.hint ?? undefined,
      sort_order: i,
      status: 'active',
    })),
  });

  return saved;
}
```

**캐시 무효화 API (관리자용):**
- `DELETE /api/ai/generate-questions/:moduleId/:textId`
- 해당 조합의 `module_questions`를 `status = 'archived'`로 soft-delete
- 이후 학습 요청 시 자동 재생성

---

#### Step 4 — `lessons.service.ts` Cache-Aside 패턴 정식화

**파일:** `apps/api/src/lessons/lessons.service.ts`

**현재 코드 문제점:**
```typescript
// 현재: try/catch로 module_questions 조회 실패를 무시하고 questionData JSONB로 폴백
try {
  const moduleQuestions = await this.prisma.module_questions.findMany({ ... });
  if (moduleQuestions) questions = moduleQuestions;
} catch {
  // module_questions 테이블이 아직 없으면 questionData fallback
  questions = (aiModule.questionData as Record<string, unknown>[]) ?? [];
}
```

**변경 목표 — Cache-Aside 패턴:**
```typescript
async getModuleData(lessonId: string, moduleCode: string) {
  // ... 레슨/모듈 조회 (기존과 동일)

  // ── 문항 조회: Cache-Aside ──────────────────────────────
  let questions: Record<string, unknown>[];

  if (lesson.text_id) {
    // 1. 캐시 조회
    const cached = await this.prisma.module_questions.findMany({
      where: {
        module_id: aiModule.id,
        text_id: lesson.text_id,
        status: 'active',
      },
      orderBy: { sort_order: 'asc' },
    });

    if (cached.length > 0) {
      // 2a. 캐시 히트 → 즉시 반환
      questions = cached as Record<string, unknown>[];
    } else {
      // 2b. 캐시 미스 → QuestionGeneratorService로 생성 + 저장
      questions = await this.questionGeneratorService.generateAndSave({
        moduleId: aiModule.id,
        textId: lesson.text_id,
        passage: { title: lesson.text!.title, content: lesson.text!.content },
        strategy: aiModule.questionGenerationStrategy as 'extract' | 'instruct',
        minCount: aiModule.questionMinCount,
        maxCount: aiModule.questionMaxCount,
        answerType: aiModule.answerType,
        pedagogyInstruction: aiModule.pedagogyInstruction,
      }) as Record<string, unknown>[];
    }
  } else {
    // text_id 없는 레슨: JSONB fallback (임시, 실제 서비스에서는 발생 안 함)
    questions = (aiModule.questionData as Record<string, unknown>[]) ?? [];
  }

  // ... 반환 (기존과 동일)
}
```

**구현 시 주의사항:**
- `QuestionGeneratorService`를 `LessonsModule`에 주입 (순환 의존성 주의)
- 생성 실패 시 예외 던지지 말고 JSONB fallback + 에러 로깅 (학습 진행 우선)
- 생성 시간이 길 수 있으므로 응답 timeout 고려 (Gemini 호출 최대 10초)
- 레슨 시작 전 pre-warm 옵션 검토 (백오피스에서 지문 등록 시 트리거)

---

#### Step 5 — `GenericAdapter` 정리 및 Legacy 어댑터 제거

**5-A. GenericAdapter `buildAgentPrompt()` 개선**

**파일:** `apps/web/src/lib/adapters/GenericAdapter.ts`

현재 `buildAgentPrompt()`는 전략 구분 없이 단일 프롬프트를 구성한다. `questionGenerationStrategy`를 참조하여 피드백 프롬프트를 전략에 맞게 구성:

```typescript
buildAgentPrompt(params: AgentPromptParams): AgentPrompt {
  const profile = this.buildPedagogyProfile();

  // extract: 문항별 학생 답안 + 정답 비교 요약
  // instruct: 학생의 자유 응답 전체를 holistic 평가 요청
  const answerContext = profile.questionGenerationStrategy === 'extract'
    ? this.buildExtractAnswerContext(params)
    : this.buildInstructAnswerContext(params);

  const system = [
    '당신은 영어 학습 AI 튜터 Pickle입니다.',
    '',
    '[교수법 지시문]',
    profile.pedagogyInstruction,
    '',
    '[지문]',
    `제목: ${params.passage.title}`,
    params.passage.content,
    '',
    answerContext,
  ].filter(Boolean).join('\n');

  return { system, user: params.userMessage };
}

private buildExtractAnswerContext(params: AgentPromptParams): string {
  // 문항별 학생 답안 vs 정답 비교 (정오답 피드백용)
  const lines = params.questions
    .filter((q) => params.answers[q.id] !== undefined)
    .map((q) => `Q${q.number}: 학생="${params.answers[q.id]}" / 정답="${q.answer}"`);
  return lines.length > 0 ? `[문항별 답안]\n${lines.join('\n')}` : '';
}

private buildInstructAnswerContext(params: AgentPromptParams): string {
  // 자유 응답 전체를 holistic 평가 (지시형)
  const answer = Object.values(params.answers)[0] ?? '';
  return answer ? `[학생 응답]\n${answer}` : '';
}
```

**5-B. Legacy 어댑터 제거 조건 및 절차**

**제거 대상 5개 파일:**
- `PredictionAdapter.ts` (PRD)
- `ScanningAdapter.ts` (SCN)
- `ShadowReadingAdapter.ts` (RRD)
- `SentenceWritingAdapter.ts` (SWR)
- `ProcessWritingAdapter.ts` (PWR)

**제거 전제조건 (모두 충족 시):**
1. ✅ 해당 모듈이 백오피스 `ai_modules`에 DB 등록 완료
2. ✅ `GET /api/lessons/:id/module-data`가 실제 서빙 중 (Mock 아님)
3. ✅ `module_questions` 캐시 생성 확인 (Step 3~4 완료 후)
4. ✅ `adapters/index.ts`의 registry에서 해당 코드 제거

**제거 절차:**
```typescript
// adapters/index.ts — 해당 항목 삭제
const registry: Record<string, () => ModuleAdapter> = {
  // PRD: () => new PredictionAdapter(),  ← 제거
  // SCN: () => new ScanningAdapter(),    ← 제거
  // RRD: () => new ShadowReadingAdapter(), ← 제거
  // SWR: () => new SentenceWritingAdapter(), ← 제거
  // PWR: () => new ProcessWritingAdapter(), ← 제거
};
```

**`resolveDefaultUiTemplate()` 제거 (`LessonSession.tsx`):**
- Legacy 어댑터 5개 모두 제거 완료 후 삭제
- 현재: `PWR → step-workflow`, `SCN → timed-gate` 하드코딩 fallback
- 제거 후: DB `uiTemplate` 필드값만으로 분기 (R4 원칙 완성)
