# SKM timed-select 지문 흐름 추가 + 오케스트레이터 버그 수정

> **작성일**: 2026-04-26  
> **기반 커밋**: tutoring `ac58e86`  
> **연관 문서**: [SKM_CLR 개발내역](./20260412_SKM_CLR_개발내역.md), [AI모듈필드재분류_동기화](./20260426_AI모듈필드재분류_동기화.md)  
> **영향 모듈**: SKM(Skimming), 전체 holistic 모듈, 전체 다문항 모듈, WRD(deck mode)

---

## 1. 사용자 흐름 (User Flow)

### SKM timed-select 지문 읽기 흐름

```
모듈 진입
  → usePassageMode('timed-select')
  → before 단계: 지문 blur + "읽기 시작" 버튼 표시
  → 버튼 클릭 → reading 단계: 지문 전체 표시 + 타이머 시작 + 문장 선택 즉시 활성화
  → 학생이 문장 선택(답안 제출) → done 단계 자동 전환 (완료 버튼 없음)
  → 이후 흐름: 오케스트레이터 → holistic 피드백 → 완료
```

### timed-blur(SCN) vs timed-select(SKM) 비교

| 단계 | timed-blur (SCN) | timed-select (SKM) |
|---|---|---|
| before | blur + 시작 버튼 | blur + 시작 버튼 |
| reading | 지문 표시 + 타이머 | 지문 표시 + 타이머 + **문장 선택 즉시 활성** |
| done 전환 | 읽기완료 버튼 클릭 | **첫 답안 제출 시 자동 전환** |
| done 상태 | blur 복원 | blur 없음 |

### 다문항 모듈 문항 이동 버튼 (QuestionsPanel)

**배경**: 다문항 모듈에서 답안 제출 후 다음 문항으로의 이동 트리거가 없어 학생이 대기 상태에 빠지는 UX 문제.  
**해결**: `QuestionsPanel` 안에 "다음문항으로 →" / "다시시도" 버튼을 직접 추가. `awaitingFeedbackConfirm` 상태 의존성 제거.

```
답안 제출(answers[activeQuestionId] !== undefined)
  → "다음문항으로 →" 버튼 표시 (항상)
  → "다시시도" 버튼 표시 (questionMaxAttempts > 1 && usedAttempts < max 일 때)
  → isLoading 중 disabled
  → 클릭 → confirmFeedback() → 다음 미답 문항 제시 or orchestrate
```

**FeedbackPanel 변경**: `questionCount !== 'multi'` 조건 추가 → 다문항 모듈에서는 FeedbackPanel에 다시시도/다음문항 버튼 미표시.

**WRD(deck mode) 적용**: WRD는 `questionFlowMode === 'deck'`로 `QuestionsPanel` 내 early-return 분기를 사용. 동일한 버튼 블록을 deck mode return 내부에도 추가.

---

### idle 타이머 버그 수정 — 다문항 모듈 전반

**버그 1 (confirmFeedback)**: 피드백 후 "다음문항" 버튼 대기 중 120초 경과 → 클릭 후 "잠깐 쉬고 있나요?" 메시지 연속 발화  
**버그 2 (giveHint)**: Q1 힌트 발화 후 40초 추가 대기 시 Q2 힌트 자동 발화 (Q1에 머물러 있음에도)

두 버그 모두 **피드백 확인 / 힌트 발화 후 lastActionTimeRef 또는 idle 임계값 플래그 미리셋** 이 원인.

---

## 2. IA 구조 및 기능 정의

### 변경된 파일 목록

| 파일 | 변경 내용 |
|---|---|
| `apps/web/src/lib/types/ai-module-data.ts` | `PassageMode`에 `'timed-select'` 추가 |
| `apps/web/src/lib/services/aiModuleService.ts` | `parsePassageMode` valid 배열에 `'timed-select'` 추가 |
| `apps/web/src/lib/adapters/GenericAdapter.ts` | `passageExposureMode` 타입 캐스트에 `'timed-select'` 추가 |
| `apps/web/src/lib/agents/agent-types.ts` | `ModulePedagogyProfile.passageExposureMode`에 `'timed-select'` 추가, `AiModuleFlags` 주석 보강, `OrchestratorContext`에 `activeQuestionId` · `hasCelebrated` 추가 |
| `apps/web/src/lib/agents/orchestrator/ModuleOrchestratorAgent.ts` | `ruleInitialEntry` timed-select 케이스 추가, holistic 조건 일반화 3곳, `ruleRepeatWrongAnswer` activeQuestionId 필터 추가, `ruleCompleteAfterCelebrate` hasCelebrated ref 기반으로 교체 |
| `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` | `usePassageMode` timed-select 분기 추가, `readingPhase` 반환, `sentenceSelectMode` 조건 보강, `QuestionsPanel`에 `questionCount` · `isLoading` · `onNextQuestion` · `onRetryAnswer` prop 추가 |
| `apps/web/src/app/modules/[lessonId]/_components/MobileSplitLayout.tsx` | `floatingTimer` prop 전달 누락 수정, `attemptCounts` prop 추가 |
| `apps/web/src/app/modules/[lessonId]/_components/panels.tsx` | `NO_BINARY_FEEDBACK_TYPES` Set 도입, `QuestionsPanel` 다음문항/다시시도 버튼 추가(sequential + deck mode), `FeedbackPanel` 다문항 조건 분리 |
| `apps/web/src/lib/hooks/useModuleOrchestrator.ts` | `confirmFeedback` / `confirmPronunciationFeedback` / `confirmWritingFeedback` idle 리셋 추가, `giveHint` case idle 임계값 소모 추가, `activeQuestionIdRef` · `hasCelebratedRef` 추가 |

---

## 3. 정책 (Policy / Business Rules)

### PassageMode 값 정책

| 값 | 모듈 | 동작 |
|---|---|---|
| `full` | SUM, QAR, SWR, RRD 등 | 처음부터 전체 공개 |
| `hidden` | WRD, WSD | 지문 패널 없음 |
| `preview` | PRD | 첫 단락만 → 답변 후 전체 공개 |
| `timed-blur` | SCN | 타이머 읽기 → 읽기완료 버튼 → blur 복원 |
| `timed-select` | SKM | 타이머 읽기 → 첫 답안 제출 시 done 자동 전환 (blur 없음) |

### 다문항 모듈 문항 이동 버튼 정책

| 조건 | 버튼 표시 |
|---|---|
| `questionCount === 'multi'` && `answers[activeQuestionId] !== undefined` | "다음문항으로 →" 항상 표시 |
| 위 조건 + `questionMaxAttempts > 1` && `usedAttempts < max` | "다시시도" 추가 표시 |
| `isLoading === true` | 두 버튼 모두 `disabled` |
| `questionCount !== 'multi'` | QuestionsPanel 버튼 미표시 (FeedbackPanel에서 처리) |

**FeedbackPanel**: `questionCount !== 'multi'` 조건 추가 → single/content-driven 모듈에서만 다시시도/이해했어요 버튼 표시. multi 모듈은 QuestionsPanel 버튼으로 통일.

---

### holistic 모듈 오케스트레이터 정책 변경

**변경 전**: `scoringMode === 'holistic' && questionCount === 'single'` 조건으로 celebrate 스킵  
**변경 후**: `scoringMode === 'holistic'` 단독 조건으로 일반화

**영향**: holistic + multi(SKM 포함) 모듈에서 celebrate 중복 발화 버그 해소.  
**적용 위치**: `ruleCorrectAnswer`, `ruleCompleteAfterCelebrate`, `rulePresentNextQuestion` 3곳.

### activeQuestionId 오케스트레이터 컨텍스트 정책

`ruleRepeatWrongAnswer`가 전체 문항을 검색하면, 이전 문항(Q1)의 오답이 현재 문항(Q2) 진행 중에도 힌트를 유발하는 버그 발생.

**해결**: `OrchestratorContext`에 `activeQuestionId: string | null` 추가. `ruleRepeatWrongAnswer` 내에서 `activeQuestionId`가 있으면 해당 문항만 검사.

```typescript
// ruleRepeatWrongAnswer 필터 조건
if (context.activeQuestionId && q.id !== context.activeQuestionId) return false;
```

`activeQuestionId`는 `useModuleOrchestrator`의 `activeQuestionIdRef`를 통해 ref 동기화 패턴으로 전달 (state 지연 없음).

---

### hasCelebrated 루프 방지 정책

**버그**: `celebrate` 발화 → `isLoading=false` → QuestionsPanel "다음문항으로 →" 버튼 활성화 → 학생 클릭 → `setMessages` 비동기로 `messagesRef` 미갱신 → `hasCelebrated=false` 판정 → `ruleCorrectAnswer`가 다시 `celebrate` 발화 → 무한 루프.

**해결**: `recentMessages` 문자열 스캔 대신 `hasCelebratedRef`(boolean ref) 사용. `celebrate` case에서 **동기적으로** `hasCelebratedRef.current = true` 설정.

```typescript
// useModuleOrchestrator.ts — celebrate case
case 'celebrate': {
  postAiMessage(celebrateText);
  hasCelebratedRef.current = true;   // ← React state 커밋 전에 이미 true
  setTimeout(() => { if (!isCompletedRef.current) orchestrateRef.current(true); }, 2000);
  break;
}

// OrchestratorContext 전달
hasCelebrated: hasCelebratedRef.current,
```

---

### idle 힌트 정책 변경

| 상황 | 변경 전 | 변경 후 |
|---|---|---|
| Q1 힌트 발화 후 40초 추가 대기 | Q2 힌트 자동 발화 | 추가 idle 이벤트 없음 (학생 액션 시 재시작) |
| 피드백 버튼 대기 120초 후 클릭 | 클릭 후 "잠깐 쉬고 있나요?" 연속 발화 | 클릭 시 idle 타이머 리셋 → 정상 |

### WRD deck mode 버튼 적용 규칙

WRD는 `questionFlowMode === 'deck'`로 `QuestionsPanel` 내부에서 early-return. 버튼을 sequential mode 끝에만 추가하면 WRD에서 표시되지 않음.

**규칙**: `QuestionsPanel`에 버튼을 추가할 때는 **deck mode return 블록** 과 **sequential mode 끝** 양쪽 모두에 동일하게 추가.

```tsx
// deck mode (early return 내부, QuestionInputArea 아래)
{isAnswered && (
  <div className="flex shrink-0 justify-end gap-2 ...">
    {/* 다시시도 / 다음문항으로 → */}
  </div>
)}

// sequential mode (return 끝, 닫는 </div> 직전)
{questionCount === 'multi' && activeQuestionId && answers[activeQuestionId] !== undefined && (
  <div className="flex justify-end gap-2 ...">
    {/* 다시시도 / 다음문항으로 → */}
  </div>
)}
```

---

### NO_BINARY_FEEDBACK_TYPES 정책

정오답 UI(✅/❌)를 표시하지 않는 타입을 Set으로 명시 관리:

```typescript
const NO_BINARY_FEEDBACK_TYPES = new Set([
  'essay', 'short-text', 'sentence-write', 'keyword-chips',
  'sentence-select', 'sentence-explain',
]);
```

새 타입 추가 시 이 Set에만 추가하면 `QuestionsPanel` 전체에 일괄 반영.

---

## 4. 개발자 추가 작업

- [ ] **SKM contentGenerationInstruction 설정**: 백오피스 seed.ts + Supabase ai_modules 업데이트  
  → 현재 SKM은 `contentGenerationInstruction`이 없어 `buildInstructRules()` fallback 사용
- [ ] **SKM passageMode Supabase 동기화**: `passage_mode = 'timed-select'` 설정 확인
- [ ] **timed-select floatingTimer onDone 동작 검증**: SKM에서 읽기완료 버튼이 실제로 숨겨지는지 확인  
  (`floatingTimer.show: readingPhase === 'before'` → reading 진입 시 타이머 패널 숨김)
- [ ] **holistic multi 모듈 전수 검증**: GMN, WWB 등 multi holistic 모듈에서 완료 흐름 정상 동작 확인

---

## 5. 코드 규칙 (Coding Rules)

### PassageMode 추가 시 체크리스트

1. `apps/web/src/lib/types/ai-module-data.ts` — `PassageMode` 타입에 추가
2. `apps/web/src/lib/services/aiModuleService.ts` — `parsePassageMode` valid 배열에 추가
3. `apps/web/src/lib/adapters/GenericAdapter.ts` — `passageExposureMode` 타입 캐스트에 추가
4. `apps/web/src/lib/agents/agent-types.ts` — `ModulePedagogyProfile.passageExposureMode` 유니온에 추가
5. `LessonSession.tsx` — `usePassageMode` 분기 추가
6. `ModuleOrchestratorAgent.ts` — `ruleInitialEntry` switch 케이스 추가
7. 백오피스 `prisma/seed.ts` — `PASSAGE_MODE` 코드 아이템 추가

### OrchestratorContext 확장 시 패턴

컨텍스트에 새 필드가 필요하면 **ref 동기화 패턴** 사용:

```typescript
// useModuleOrchestrator.ts
const newFieldRef = useRef<T>(initialValue);
useEffect(() => { newFieldRef.current = reactState; }, [reactState]);

// orchestrate() 내 context 객체
const context: OrchestratorContext = {
  ...
  newField: newFieldRef.current,  // state 지연 없이 항상 최신값
};
```

`activeQuestionId`와 `hasCelebrated`가 이 패턴으로 추가된 사례.

---

### idle 타이머 관련 규칙

**피드백 확인 핸들러 패턴** (confirmFeedback / confirmPronunciationFeedback / confirmWritingFeedback):
```typescript
// 반드시 포함:
lastActionTimeRef.current = Date.now();
idleThreshold20FiredRef.current = false;
idleThreshold60FiredRef.current = false;
idleThreshold120FiredRef.current = false;
```

**giveHint case 패턴** (executeToolCall 내부):
```typescript
// 힌트 발화 후 반드시 포함:
idleThreshold20FiredRef.current = true;
idleThreshold60FiredRef.current = true;
idleThreshold120FiredRef.current = true;
// → 학생 액션(submitAnswer 등)이 발생할 때까지 추가 idle 발화 없음
```

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 항목 | 내용 |
|---|---|
| `timed-select` 완료 버튼 | `floatingTimer.onDone`이 timed-select에서도 바인딩되지만, `show: readingPhase === 'before'`로 타이머 패널이 reading 단계에서 숨김 — 완료 버튼 노출 여부 UX 검토 필요 |
| `readingPhase` prop drilling | `usePassageMode`가 반환하는 `readingPhase`를 여러 컴포넌트에 전달 — 추후 context화 고려 |
| holistic 일반화 후 `questionCount: 'multi'` holistic 모듈 celebrate 미발화 | 설계상 의도 (holistic은 celebrate 없이 completeModule 직행) — 문서화 완료 |
| `activeQuestionId` 초기값 null | 모듈 진입 직후 `activeQuestionId`가 null인 동안 `ruleRepeatWrongAnswer` 필터가 적용되지 않음 — 첫 문항 제시 전 오답 시나리오는 실제로 발생 불가하므로 무시 가능 |
| `hasCelebratedRef` 리셋 미구현 | 모듈 재시작 시 `hasCelebratedRef.current = false` 리셋 없음 — 현재 모듈 단위 마운트/언마운트로 자동 초기화되므로 문제 없음 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

```
LessonSession.tsx
  ├─ usePassageMode()              ← PassageMode 분기 (timed-blur / timed-select / hidden / full)
  ├─ useModuleOrchestrator()       ← idle 타이머, confirmFeedback, giveHint, activeQuestionIdRef, hasCelebratedRef
  └─ ModuleRunnerInner
       ├─ ContentPanel             ← floatingTimer, blurActive, sentenceSelectMode
       ├─ MobileSplitLayout        ← floatingTimer prop (누락 수정), attemptCounts
       └─ QuestionsPanel           ← NO_BINARY_FEEDBACK_TYPES, questionCount, isLoading, onNextQuestion, onRetryAnswer
            └─ FeedbackPanel       ← questionCount !== 'multi' 조건 추가

ModuleOrchestratorAgent.ts
  └─ ruleInitialEntry             ← timed-select case 추가
  └─ ruleCorrectAnswer            ← holistic 조건 일반화
  └─ ruleCompleteAfterCelebrate   ← holistic 조건 일반화, hasCelebrated ref 기반 교체
  └─ rulePresentNextQuestion      ← holistic 조건 일반화
  └─ ruleRepeatWrongAnswer        ← activeQuestionId 필터 추가

OrchestratorContext (agent-types.ts)
  └─ activeQuestionId             ← ruleRepeatWrongAnswer 활성 문항 필터
  └─ hasCelebrated                ← ruleCompleteAfterCelebrate celebrate 루프 방지
```

### 진입점

- `/modules/[lessonId]` — 레슨 세션 페이지에서 `passageMode: 'timed-select'`인 모듈 진입 시 자동 적용

---

## 8. DB/API 구조 (Data Contract)

### PassageMode 타입 (ai-module-data.ts)

```typescript
export type PassageMode =
  | 'full'          // 처음부터 전체 공개 (기본값)
  | 'hidden'        // 지문 패널 없음 (WRD, WSD)
  | 'preview'       // 첫 단락만 → 답변 후 전체 공개 (PRD)
  | 'timed-blur'    // 타이머 읽기 → blur 복원 (SCN)
  | 'timed-select'; // 타이머 읽기 → 문장 선택 활성화 (SKM)
```

### ModulePedagogyProfile.passageExposureMode (agent-types.ts)

```typescript
passageExposureMode: 'hidden' | 'preview' | 'full' | 'timed-blur' | 'timed-select';
```

### usePassageMode 반환 타입

```typescript
{
  passageVisibleOverride: boolean | undefined;
  floatingTimer: FloatingTimer | undefined;
  blurActive: boolean;
  readingPhase: 'before' | 'reading' | 'done';
  contentPanelFooter: undefined;
}
```

### 외부 API 계약 변경 없음

`GET /lessons/module-meta` 응답의 `passageMode` 필드 값으로 `'timed-select'`가 추가될 뿐, 구조 변경 없음.
