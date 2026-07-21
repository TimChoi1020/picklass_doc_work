---
기간: 2026-04-08 ~ 2026-06-01
원본 파일: 모듈렌더링데이터드리븐리팩토링_20260408.md, 20260428_questionFlowMode_재정의_개발완료.md, 20260410_튜터링_모듈학습_UI개선및버그수정.md, 20260412_SKM_CLR_개발내역.md, 20260426_SKM_timed-select_오케스트레이터_버그수정.md
작성자: TimChoi1020
문서 목적: 모듈 렌더링 구조 · 기능 · 구현 레퍼런스 (기능별 재정리, 2026-06-01 기준)
---

## 개발 상태

### ✅ 완료 항목

| 항목 | 완료일 |
|---|---|
| uiTemplate switch-case 제거 및 passageMode / questionFlowMode 도입 | 2026-04-08 |
| passageMode 4종 값 정의 및 DB 컬럼 추가 | 2026-04-08 |
| VocabDeckFlow / ProcessWritingFlow / InteractiveFlow 삭제 | 2026-04-08 |
| SKM sentence-select UI 구현 | 2026-04-12 |
| CLR sentence-explain UI (SentenceExplainPopover) | 2026-04-12 |
| timed-then-blur Blur 처리 수정 (blurActive 분리) | 2026-04-10 |
| Floating Timer 구현 + 단계별 재설계 (overlay / header) | 2026-04-10 |
| greet 툴 독립 분리 (Rule 0) | 2026-04-10 |
| SCN keyword-chips 답안 입력 UI | 2026-04-10 |
| HOLISTIC_TYPES 상수화 | 2026-04-10 |
| before 페이즈 blur 처리 추가 (SKM / SCN) | 2026-04-10 |
| SKM timed-select passageMode 추가 | 2026-04-26 |
| passageMode `hidden` DOM 조건부 렌더 수정 | 2026-04-28 |
| uiTemplate 재정의 (standard / voice / embedded / hidden) | 2026-04-28 |
| resolveUiTemplate() 헬퍼 + 하위 호환 브릿지 | 2026-04-28 |
| VoiceQuestionPanel deck 모드 지원 | 2026-04-28 |
| questionFlowMode 재정의 (단문항 / 다문항 명시) | 2026-04-28 |
| 백오피스 seed 코드그룹 정비 (레거시 삭제) | 2026-04-28 |
| usePassageMode switch 5-case 재구성 (옵션별 로직 분리) | 2026-06-01 |
| ui_template DB 마이그레이션 (WSD/RRD/SHD/WSP→voice, CLR→embedded) | 2026-06-01 |
| resolveUiTemplate 제거 + uiTemplate 단일 상수 + VALID_UI_TEMPLATES 확장 | 2026-06-01 |
| useEmbeddedMode 훅 분리 (CLR 전용 로직) | 2026-06-01 |
| uiTemplate 렌더 구조 — 옵션별 명시 4-case로 재구성 | 2026-06-01 |
| NO_BINARY_FEEDBACK_TYPES 제거 → QuestionsPanel scoringMode prop으로 교체 | 2026-06-01 |
| `isLastQuestion` 도입 — "다음문항" / "이해했어요" 버튼 분기 재설계 | 2026-06-01 |
| `sequential` 모드 — 전체 문항 동시 표시로 변경 (deck과 UX 명확히 분리) | 2026-06-02 |
| QuestionsPanel `activeQuestionId` 게이트 — 인사말 완료 후 문항 표시, 그 전에는 "준비 중이에요..." | 2026-06-02 |
| VoiceQuestionPanel sequential 모드 `activeRecordingId` 게이트 추가 — 전 모드 일관 적용 | 2026-06-02 |
| QuestionsPanel 입력 위젯 `q.type` 기준 통일 — `short-text`→`<input>`, `essay`→`<textarea>` (모드 무관) | 2026-06-02 |
| `usePassageMode` `full` 케이스 `passageVisibleOverride: undefined → true` — ContentPanel 초기 preview 플래시 수정 | 2026-06-02 |
| QuestionsPanel / VoiceQuestionPanel deck 모드 미시작 구간 플래시 제거 + 로딩 메시지 일관성 확보 | 2026-06-02 |
| `HOLISTIC_TYPES` / `isHolisticType` 제거 — `scoringMode` 직접 참조로 전환 | 2026-06-02 |
| `module_questions.explanation` JSONB 컬럼 추가 — 부가 메타데이터(`evidence`, `questionType` 등) 분리 저장 (`2026-07-02_module_questions_explanation_column.sql`) | 2026-07-02 |
| `multiple-choice` `q.answer` 포맷 정규화 — DB에서 JSON 문자열 `{"correctOption":"③",...}` 혼용 → `"③"` 단순 문자열로 일괄 마이그레이션 | 2026-07-02 |
| `GenericAdapter` 정리 — API response 타입 `Record<string, unknown>[]`, `sentence-explain` `explanation` 컬럼 우선 / `options` 폴백 파싱, `calculateScore` exact 비교 `.trim().toLowerCase()` 정규화 | 2026-07-02 |

---

## 1. 아키텍처 개요

### 1.1 설계 원칙 — R4: DB 등록만으로 렌더링 자동 구성

> 신규 모듈이 추가되더라도 백오피스에서 DB 등록만으로 튜터링 학습 화면이 자동 구성되어야 한다. 모듈별 전용 코드를 작성하지 않는 것이 기본 원칙.

**이전 방식 (switch-case 하드코딩):**
```typescript
// LessonSession.tsx — 모듈코드 직접 분기
switch (uiTemplate) {
  case 'step-workflow': return <ProcessWritingFlow />;
  case 'timed-gate':    return <ScanningGate />;
  case 'vocab-deck':    return <VocabDeckFlow />;
  case 'interactive':   return <InteractiveFlow />;
  default:              return <ModuleRunnerInner />;
}
```

**현재 방식 (DB 3축 제어):**
```
passageMode      → 지문 패널 동작 모드  (full / hidden / preview / timed-blur / timed-select)
uiTemplate       → 문항 패널 동작 모드  (standard / voice / embedded / hidden)
questionFlowMode → 문항 진행 방식       (sequential / deck / locked-steps)
```
모든 모듈이 `ModuleRunnerInner` 단일 컴포넌트를 통해 렌더링되고, 내부 동작은 DB 3축 값으로 결정된다.

### 1.2 렌더링 데이터 흐름

```
ai_modules (DB)
  ├─ passage_mode
  ├─ ui_template
  └─ question_flow_mode
       ↓
GET /lessons/:id/module-data (NestJS API)
       ↓
GenericAdapter.fetchModuleData()
  → ModuleData { passageMode, uiTemplate, questionFlowMode, ... }
       ↓
LessonSession.tsx
  ├─ usePassageMode(passageMode)  → passageVisibleOverride, blurActive, floatingTimer
  ├─ uiTemplate (DB 직접 참조)
  ├─ useEmbeddedMode(uiTemplate==='embedded') → embedded.* (CLR 전용)
  └─ ModuleRunnerInner
       ├─ ContentPanel          (passageMode 제어)
       ├─ QuestionsPanel        (uiTemplate==='standard')
       │   VoiceQuestionPanel   (uiTemplate==='voice')
       │   [없음]               (uiTemplate==='embedded' | 'hidden')
       └─ FeedbackPanel
```

---

## 2. 컴포넌트 계층 구조

```
LessonSession.tsx                          # 세션 전체 관리 (모듈 시퀀스)
  └─ ModuleRunner                          # 단일 모듈 실행, 어댑터 초기화
       └─ ModuleRunnerInner               # 단일 렌더 진입점 (모든 모듈 공통), usePassageMode 호출
            ├─ ModuleProgressBar           # 2개 이상 모듈일 때 순서 진행 표시
            ├─ [Desktop lg+] 2-column 레이아웃
            │   ├─ 좌: ContentPanel + QuestionsPanel | VoiceQuestionPanel
            │   └─ 우: FeedbackPanel + AIQuestionPanel
            └─ [Mobile <lg] MobileSplitLayout.tsx
                 ├─ 상단(55%): 탭(지문|문항|쉐도잉) 전환
                 ├─ 중간: 드래그 분할선 (25~72% 범위)
                 └─ 하단(45%): FeedbackPanel + AIQuestionPanel

패널 컴포넌트 (panels.tsx):
  ContentPanel         # 지문 표시. passageMode 제어 대상
  QuestionsPanel       # uiTemplate=standard. questionFlowMode 제어 대상
  VoiceQuestionPanel   # uiTemplate=voice. questionFlowMode=deck 지원
  QuestionPanelFooter  # (내부 공유) "다음문항 →" / "다시시도" 버튼. QuestionsPanel·VoiceQuestionPanel 양쪽에서 사용 (deck·sequential 각 1곳, 총 4곳)
  FeedbackPanel        # AI 채팅 + 피드백 확인 버튼. hint / manualComplete 버튼 포함 (→ 20260512_힌트시스템_개선_개발완료.md 참조)
  AIQuestionPanel      # Pickle AI에게 질문 입력
  ModuleCompleteCard   # 모듈 완료 시 좌측 패널(데스크톱) / 상단(모바일) 교체 렌더
  ModuleProgressBar    # 모듈 2개 이상일 때 상단 진행 상태바 (LessonSession에서 직접 렌더)
  TypewriterText       # (내부 유틸) AI 메시지 타이프라이터 애니메이션
```

### passageMode → ContentPanel 연결 경로

```typescript
// LessonSession.tsx
const { passageVisibleOverride, blurActive, floatingTimer } = usePassageMode(passageMode, firstAnswerSubmitted);

// passageMode='hidden': ContentPanel wrapper div 자체를 렌더하지 않음
{passageMode !== 'hidden' && (
  <div className="flex-1 overflow-hidden">
    <ContentPanel
      previewOnly={!passageVisible || passagePreviewOnly}
      exposureMode={blurActive ? 'blurred' : undefined}
      floatingTimer={floatingTimer}
    />
  </div>
)}
```

### uiTemplate → 문항 패널 연결 경로

```typescript
// LessonSession.tsx (2026-06-01 기준)
const uiTemplate = (moduleData.uiTemplate ?? 'standard') as 'standard' | 'voice' | 'embedded' | 'hidden';
const embedded   = useEmbeddedMode(uiTemplate === 'embedded', ...);

// 문항 패널 라우팅 — 옵션별 명시
{uiTemplate === 'standard' && <QuestionsPanel questionFlowMode={questionFlowMode} ... />}
{uiTemplate === 'voice'    && <VoiceQuestionPanel questionFlowMode={questionFlowMode} ... />}
// embedded: 문항 패널 없음 → CLR SentenceExplainPopover가 ContentPanel 내부에 렌더
// hidden:   문항 패널 없음
```

---

## 3. passageMode — 지문 패널 동작

### 3.1 5종 값 정의

| 값 | 지문 패널 DOM | 동작 | 해당 모듈 |
|---|---|---|---|
| `full` | 렌더 | 처음부터 전체 공개 | SUM, QAR, CLR, WRD, WSD, GMN 등 대부분 |
| `hidden` | **렌더 안 함** | wrapper div 조건부 제거, flex 공간도 점유 안 함 | RRD, SWR |
| `preview` | 렌더 | 도입 2문장만 → 답변 후 전체 공개 (`revealPassage` 플래그) | PRD |
| `timed-blur` | 렌더 | 타이머 읽기 → "읽기완료" 클릭 → blur 복원 | SCN |
| `timed-select` | 렌더 | 타이머 읽기 → 문장 선택 즉시 활성화, 첫 답안 제출 시 done | SKM |

> **구 값 → 현재 값 변경 이력** (2026-04-26):
> - `preview-then-reveal` → `preview`
> - `timed-then-blur` → `timed-blur`

### 3.2 usePassageMode 훅

#### 구현 원칙 — 옵션별 로직 완전 분리

**`usePassageMode`는 `switch` 5-case 구조를 유지한다. 새 passageMode를 추가할 때도 반드시 새 `case`로만 추가해야 하며, 기존 케이스에 조건을 끼워 넣거나 여러 옵션이 공유하는 분기를 만들지 않는다.**

이 원칙을 지켜야 하는 이유:
- 각 passageMode는 독립적인 학습 흐름(before → reading → done 전환 방식, blur 여부, 타이머 표시 조건)을 가진다.
- 조건을 혼재하면 "A mode에서 왜 이 조건이 true인가"를 추적하기 위해 다른 mode의 로직까지 읽어야 한다.
- switch-case는 5개 옵션이 동등한 위상으로 나열되므로 새 옵션 추가 위치가 명확하다.

**반환 타입:**
```typescript
{
  passageVisibleOverride: boolean | undefined;
  blurActive:             boolean;
  floatingTimer:          FloatingTimer | undefined;
  readingPhase:           'before' | 'reading' | 'done';
  contentPanelFooter:     undefined;
}

interface FloatingTimer {
  show:           boolean;
  phase:          'before' | 'reading' | 'done';
  elapsedSeconds: number;
  onStart:        () => void;
  onDone:         () => void;
}
```

**전체 구현 (2026-06-01 기준):**
```typescript
function usePassageMode(passageMode: string, firstAnswerSubmitted: boolean) {
  // timed 모드(timed-blur, timed-select)에서만 쓰이는 읽기 상태
  const [readingPhase, setReadingPhase] = useState<'before' | 'reading' | 'done'>('before');
  const [readingStartTime, setReadingStartTime] = useState<number | null>(null);
  const [elapsedSeconds, setElapsedSeconds] = useState(0);

  // reading 단계 경과 타이머
  // passageMode 가드 불필요 — readingPhase는 onStart(floatingTimer.onStart)를 통해서만
  // 'reading'으로 전환되며, floatingTimer는 timed-blur/timed-select 케이스에서만 반환된다.
  useEffect(() => {
    if (readingPhase !== 'reading' || readingStartTime === null) return;
    const id = setInterval(
      () => setElapsedSeconds(Math.floor((Date.now() - readingStartTime) / 1000)),
      1000,
    );
    return () => clearInterval(id);
  }, [readingPhase, readingStartTime]);

  // timed-select 전용: 첫 답안 제출 → done 자동 전환 (읽기완료 버튼 없음)
  useEffect(() => {
    if (passageMode === 'timed-select' && firstAnswerSubmitted && readingPhase === 'reading') {
      setReadingPhase('done');
    }
  }, [passageMode, firstAnswerSubmitted, readingPhase]);

  const onStart = useCallback(() => { setReadingStartTime(Date.now()); setReadingPhase('reading'); }, []);
  const onDone  = useCallback(() => setReadingPhase('done'), []);

  // timed 모드 floatingTimer 공유 베이스 (show 값만 케이스별로 다름)
  const timedBase: Omit<FloatingTimer, 'show'> = { phase: readingPhase, elapsedSeconds, onStart, onDone };

  switch (passageMode) {
    case 'timed-blur':
      // before → reading(지문 공개) → done(blur 복원) → 첫 제출 후 blur 해제
      return {
        passageVisibleOverride: true,
        blurActive: readingPhase === 'before' || (readingPhase === 'done' && !firstAnswerSubmitted),
        floatingTimer: { ...timedBase, show: readingPhase !== 'done' },
        readingPhase,
        contentPanelFooter: undefined,
      };

    case 'timed-select':
      // before → reading(지문 공개 + 문장선택 즉시 활성) → 첫 제출 시 자동 done
      return {
        passageVisibleOverride: true,
        blurActive: readingPhase === 'before',
        floatingTimer: { ...timedBase, show: readingPhase === 'before' },
        readingPhase,
        contentPanelFooter: undefined,
      };

    case 'hidden':
      // 지문 패널 DOM 자체를 제거 (flex 공간도 미점유)
      return {
        passageVisibleOverride: false,
        blurActive: false,
        floatingTimer: undefined,
        readingPhase: 'done' as const,
        contentPanelFooter: undefined,
      };

    case 'full':
      // 처음부터 전체 공개 — timed-* 와 동일하게 훅이 visibility 확정
      // undefined였을 때: passageVisible 초기값(false) → previewOnly=true → 초기 preview 플래시 발생 (2026-06-02 수정)
      return {
        passageVisibleOverride: true,
        blurActive: false,
        floatingTimer: undefined,
        readingPhase: 'done' as const,
        contentPanelFooter: undefined,
      };

    case 'preview':
      // Orchestrator가 holistic 피드백 시점에 revealPassage 플래그로 제어
      return {
        passageVisibleOverride: undefined,
        blurActive: false,
        floatingTimer: undefined,
        readingPhase: 'done' as const,
        contentPanelFooter: undefined,
      };

    default:
      return {
        passageVisibleOverride: undefined,
        blurActive: false,
        floatingTimer: undefined,
        readingPhase: 'done' as const,
        contentPanelFooter: undefined,
      };
  }
}
```

> ⚠️ **이전 버그**: `passageVisibleOverride = false` → `previewOnly=true` 경로로 연결되어 blur 대신 2문장 preview가 표시됨. `blurActive` 플래그를 분리하여 `ContentPanel`에 `exposureMode="blurred"`로 직접 전달하는 방식으로 수정 (2026-04-10).

### 3.3 timed-blur vs timed-select 비교

| 단계 | timed-blur (SCN) | timed-select (SKM) |
|---|---|---|
| before | blur + Floating Timer "지문읽기" overlay | blur + Floating Timer "지문읽기" overlay |
| reading | 지문 정상 표시 + 헤더 inline 타이머 + "읽기완료" 버튼 | 지문 정상 표시 + 헤더 inline 타이머 + **문장 선택 즉시 활성** |
| done 전환 | "읽기완료" 버튼 클릭 | **첫 답안 제출 시 자동 전환** |
| done 상태 | blur 복원 | blur 없음 (지문 계속 표시) |

### 3.4 Floating Timer UI 구조

Floating Timer는 `before`와 `reading` 단계에서 다른 방식으로 렌더된다:

**before 단계 — 전체 overlay:**
```tsx
// ContentPanel 내부. 지문 위 전체를 덮는 overlay (backdrop-blur-sm 없음)
<div className="absolute inset-0 z-10 flex flex-col items-center justify-center gap-3 rounded-2xl bg-gray-50/90">
  <p className="text-sm text-gray-400">지문을 읽을 준비가 되면 시작하세요</p>
  <button onClick={floatingTimer.onStart} className="... bg-[#18c37e] ...">
    <Play className="h-4 w-4" /> 지문읽기
  </button>
</div>
```

**reading 단계 — 헤더 우측 inline:**
```tsx
// ContentPanel 헤더 내부. 지문 공간 100% 확보
<div className="ml-auto flex items-center gap-2">
  <span className="text-xs font-medium tabular-nums text-amber-500">
    읽기 중 {mm}:{ss}
  </span>
  <button onClick={floatingTimer.onDone} className="... bg-[#18c37e] ...">
    <CheckCheck className="h-3.5 w-3.5" /> 읽기완료
  </button>
</div>
```

> 기존 `contentPanelFooter` 방식(지문 아래 고정 영역 ~100px 잠식)에서 변경. reading 단계를 헤더 inline으로 옮겨 지문 가시 영역 100% 확보.

### 3.5 preview passageMode — revealPassage 플래그

PRD 등 `passageMode='preview'` 모듈은 holistic 피드백 발화 시점에 지문 전체를 함께 공개한다.

```typescript
// ModuleOrchestratorAgent.ts — ruleHolisticFeedback
const revealPassage = context.passagePreviewOnly && isAllAnswered(questions, state.answers);
return { tool: 'giveFeedback', params: { type: 'holistic', revealPassage } };

// useModuleOrchestrator.ts — giveFeedback holistic 케이스
if (params.type === 'holistic' && params.revealPassage) {
  passagePreviewOnlyRef.current = false;
  setPassagePreviewOnly(false);
  postAiMessage('이제 전체 지문을 읽어보세요! 예측과 어떻게 다른지 비교해 보세요. 📖');
}
```

preview 도입 2문장 파싱:
```typescript
const previewText = passage.content
  .split(/(?<=[.!?])\s+/)
  .filter(Boolean)
  .slice(0, 2)
  .join(' ');
```

### 3.6 passageMode 추가 시 체크리스트

1. `apps/web/src/lib/types/ai-module-data.ts` — `PassageMode` 타입 유니온에 추가
2. `apps/web/src/lib/services/aiModuleService.ts` — `parsePassageMode` valid 배열에 추가
3. `apps/web/src/lib/adapters/GenericAdapter.ts` — 타입 캐스트에 추가
4. `apps/web/src/lib/agents/agent-types.ts` — `ModulePedagogyProfile.passageMode` 유니온에 추가
5. `LessonSession.tsx` — `usePassageMode` 내 `switch`에 **새 `case`로 추가** (§3.2 원칙 참조)
   - 기존 케이스에 조건을 끼워 넣지 않는다
   - 반환 객체의 키는 5개 모두 명시한다 (`passageVisibleOverride`, `blurActive`, `floatingTimer`, `readingPhase`, `contentPanelFooter`)
6. `ModuleOrchestratorAgent.ts` — `ruleInitialEntry` switch 케이스 추가
7. `prisma/seed.ts` — `PASSAGE_MODE` 코드 아이템 추가

### 3.7 ContentPanel `exposureMode` 확장값 (미연결 모드)

ContentPanel의 `exposureMode` prop은 `usePassageMode` 출력(`blurred`) 외에 아래 두 값도 지원한다. 현재 passageMode 5종 및 어떤 모듈과도 연결되어 있지 않은 독립 UI 모드이다.

| 값 | ContentPanel 동작 | 연결 passageMode | 사용 모듈 |
|---|---|---|---|
| `minimized` | "지문이 최소화되어 있습니다." 안내만 표시 | 미연결 | 미확정 |
| `sentence-by-sentence` | 문장 단위 이전/다음 버튼 순차 읽기 | 미연결 | 미확정 |

> ⚠️ 이 두 값은 `ContentPanel` 내부에 렌더 분기가 구현되어 있으나, 현재 어느 passageMode / 모듈과도 연결되어 있지 않다. 향후 신규 모듈 설계 시 passageMode에 새 값을 추가(§3.6 체크리스트)하거나, `usePassageMode`에서 해당 `exposureMode`를 반환하여 연결한다.

---

## 4. uiTemplate — 문항 패널 동작

### 4.1 4종 값 정의

| 값 | 문항 패널 | 렌더 컴포넌트 | 해당 모듈 |
|---|---|---|---|
| `standard` | 렌더 | `QuestionsPanel` | WRD, GMN, PRD, SCN, SKM, SUM, QAR, SWR, PWR |
| `voice` | 렌더 | `VoiceQuestionPanel` | WSD, RRD, SHD, WSP |
| `embedded` | **없음** | 지문에 문항 임베드 (CLR SentenceExplainPopover) | CLR |
| `hidden` | **없음** | 향후 확장용 | — |

### 4.2 DB 마이그레이션 완료 (2026-06-01) — resolveUiTemplate 제거

DB `ui_template` 컬럼이 실제 값으로 업데이트됨에 따라 하위 호환 브릿지 함수 `resolveUiTemplate`이 제거되었다.

**삭제된 코드:**
```typescript
// ❌ 제거됨 — LessonSession.tsx
function resolveUiTemplate(uiTemplate: string, answerType: string) { ... }
const effectiveUiTemplate = resolveUiTemplate(moduleData.uiTemplate ?? 'standard', pedagogyProfile.answerType);
const isVoiceModule       = effectiveUiTemplate === 'voice';
const sentenceExplainMode = effectiveUiTemplate === 'embedded';
```

**현재 코드:**
```typescript
// ✅ DB 값 직접 참조
const uiTemplate = (moduleData.uiTemplate ?? 'standard') as 'standard' | 'voice' | 'embedded' | 'hidden';
```

**함께 수정된 파일:**
- `LessonSession.tsx` — boolean 파생 변수 제거, `uiTemplate` 단일 상수로 통합
- `aiModuleService.ts` — `VALID_UI_TEMPLATES: ['standard']` → `['standard', 'voice', 'embedded', 'hidden']`
  (마이그레이션 전에는 'voice'/'embedded' DB 값이 'standard'로 덮어씌워지는 버그 존재)
- `ai-module-data.ts` — stale 주석 제거

### 4.3 모듈별 DB 실제값 (2026-06-01 기준)

| 모듈 | ui_template (DB) | 렌더 컴포넌트 |
|---|---|---|
| WRD, GMN, PRD, SCN, SKM, SUM, QAR, SWR, PWR | **standard** | `QuestionsPanel` |
| WSD, RRD, SHD, WSP | **voice** | `VoiceQuestionPanel` |
| CLR | **embedded** | `SentenceExplainPopover` (ContentPanel 내부) |

> 마이그레이션 SQL:
> ```sql
> UPDATE ai_modules SET ui_template = 'voice'    WHERE module_code IN ('WSD', 'RRD', 'SHD', 'WSP');
> UPDATE ai_modules SET ui_template = 'embedded' WHERE module_code = 'CLR';
> ```

### 4.4 구현 원칙 — 옵션별 로직 완전 분리

**`uiTemplate`은 4-case 구조를 유지한다. 새 옵션 추가 시 기존 케이스에 조건을 끼워 넣지 않는다.**

**렌더 구조 (2026-06-01 기준):**
```tsx
// ✅ 옵션별 명시 — 각 케이스가 무엇을 렌더하는지 한눈에 파악 가능
{uiTemplate === 'standard' && <QuestionsPanel ... />}
{uiTemplate === 'voice'    && <VoiceQuestionPanel ... />}
{/* embedded: ContentPanel 내부 SentenceExplainPopover — 별도 패널 없음 */}
{/* hidden:   문항 패널 없음 */}
```

**이전 구조 (제거됨):**
```tsx
// ❌ boolean 중첩 — embedded/standard/hidden이 else로 뭉쳐져 있어 파악 어려움
{!sentenceExplainMode && (
  <div>{isVoiceModule ? <VoiceQuestionPanel /> : <QuestionsPanel />}</div>
)}
```

### 4.5 embedded 전용 로직 — useEmbeddedMode 훅

CLR(`embedded`) 전용 상태·effect·핸들러를 `useEmbeddedMode` 훅으로 분리하여 메인 컴포넌트(`ModuleRunnerInner`)에 노출되지 않도록 했다.

**파일**: `LessonSession.tsx` 내 모듈 스코프 함수

```typescript
function useEmbeddedMode(
  active: boolean,                                    // uiTemplate === 'embedded'
  passageContent: string,
  submitUserMessage: (msg: string) => Promise<void>,
  completedRef: React.RefObject<boolean>,
  onComplete: (explainedCount: number, totalCount: number) => void,
) → {
  explainedSentences, activeSentence, manualCompleteReady, totalSentences,
  handleSentenceExplain, handleAskAboutSentence, handleManualComplete,
}
```

- `active=false`(standard/voice/hidden)일 때 모든 state는 초기값 유지, 어떤 effect도 실행되지 않는다.
- 완료 조건: 지문 문장 수 80% 탐색 시 `manualCompleteReady=true` → "완료" 버튼 활성화
- 완료 기준 문장 수: `questions.length` 아님 — `passage.content` 직접 파싱 (CLR은 문항 미생성 상태에서 시작할 수 있음)

**호출부:**
```typescript
const embedded = useEmbeddedMode(
  uiTemplate === 'embedded',
  moduleData.passage.content,
  submitUserMessage,
  completedRef,
  (explainedCount, totalCount) => { onModuleComplete({ score: ... }, messages, ...); },
);
// 이후 embedded.explainedSentences, embedded.handleManualComplete 등으로 접근
```

### 4.6 uiTemplate 추가 시 체크리스트

1. 백오피스에서 해당 모듈 `ai_modules.ui_template` 업데이트 (SQL 직접 실행)
2. `apps/web/src/lib/services/aiModuleService.ts` — `VALID_UI_TEMPLATES` 배열에 추가
3. `apps/web/src/lib/types/ai-module-data.ts` — `UiTemplate` 유니온에 추가
4. `LessonSession.tsx` 렌더 섹션 — **새 JSX 블록으로 추가** (§4.4 원칙 참조)
   - 기존 블록에 조건 끼워 넣기 금지
   - `embedded`·`hidden` 처럼 패널 없음 케이스는 주석으로 명시
5. 새 옵션이 전용 상태·로직을 가지면 → 별도 훅(`useXxxMode`)으로 분리 (§4.5 패턴)

---

## 5. questionFlowMode / questionCount — 역할 분리 원칙

### 5.0 역할 분리 원칙 (2026-06-01 확립)

> **`questionFlowMode`는 카드 렌더링 방식에만 관여한다. 문항 수·진행도·버튼 표시는 `questionCount`가 결정한다.**

| 관심사 | 결정 주체 | 이유 |
|---|---|---|
| 카드 배치 방식 (1개/전체/잠금) | `questionFlowMode` | 레이아웃 결정 |
| 잠금 오버레이 | `questionFlowMode === 'locked-steps'` | 레이아웃 결정 |
| 진행도 표시 (N/M) | `questionCount !== 'single'` | 콘텐츠 결정 |
| 다음문항/다시시도 버튼 | `questionCount === 'multi'` | 콘텐츠 결정 |

두 개념은 독립적으로 조합된다. 예: PWR = `locked-steps`(렌더) + `multi`(콘텐츠).

### 5.1 questionFlowMode 3종 — 카드 렌더링 방식

| 값 | 화면 표시 | 잠금 | 해당 모듈 |
|---|---|---|---|
| `deck` | `activeQuestionId` 기준 **1개만** | — | WRD, GMN, QAR, WSD, RRD 등 |
| `sequential` | **전체 동시** 표시 (2026-06-02 변경) | 없음 (자유 답변) | 단문항 모듈 대부분 |
| `locked-steps` | **전체 동시** 표시 | 이전 미완료 시 🔒 | PWR |

**변경 이력 (2026-06-02)**: `sequential` 이전 동작은 "답변된 문항 + 현재 활성 문항만 누적 표시"였다. 이 방식은 deck과 UX 차이가 없어 두 모드의 구분이 불명확했다. `sequential`을 전체 동시 표시로 변경해 세 모드 간 역할을 명확히 분리했다.

```
deck          → 집중형 (1개 카드)
sequential    → 조망형 (전체 보임, 자유 답변)
locked-steps  → 단계형 (전체 보임, 순서 강제)
```

### 5.1.1 문항 표시 게이트 — `activeQuestionId`

**모든 questionFlowMode**에서 문항은 `activeQuestionId !== null`이 될 때까지 표시되지 않는다.

| 모드 | 구현 | null 처리 |
|---|---|---|
| `deck` | `activeQuestionId ? find(...) ?? last : null` | `null` → "준비 중이에요..." |
| `sequential` | `activeQuestionId !== null ? questions : []` | `[]` → "준비 중이에요..." |
| `locked-steps` | 동일 | 동일 |

```typescript
// deck 모드
const activeQ = activeQuestionId
  ? (questions.find((q) => q.id === activeQuestionId) ?? questions[questions.length - 1])
  : null;  // null = 미시작 / ?? 폴백 = 모두 완료 후 마지막 카드 유지

// sequential·locked-steps 모드
const visibleQuestions = activeQuestionId !== null ? questions : [];
```

**이유**: `activeQuestionId`는 `askQuestion(Q1)` 실행 시 설정된다. 이 시점은 `ruleGreet`(인사말) + `ruleInitialEntry`(지문 표시) 완료 이후이므로, 문항 패널이 피드백 패널의 인사말보다 먼저 나오는 일이 없다.

```
모듈 진입 (activeQuestionId = null)
  FeedbackPanel  → 인사말 표시
  QuestionsPanel → "Pickle AI가 문항을 준비 중이에요..."

ruleGreet + ruleInitialEntry 완료
  → askQuestion(Q1) → activeQuestionId = Q1.id
  → QuestionsPanel 문항 표시
```

이 게이트는 `questionFlowMode` · `questionCount`에 무관하게 모든 모드에 적용된다.

> `sequential` / `locked-steps` 둘 다 `questionCount`와 독립적으로 조합 가능.
> 단문항(`single`) + `sequential`, 다문항(`multi`) + `locked-steps`(PWR) 모두 유효.

### 5.2 questionCount 3종 — 문항 수 / 진행도 / 버튼

| 값 | 의미 | 진행도 표시 | 다음문항 버튼 |
|---|---|---|---|
| `single` | 문항 1개 | 숨김 | 숨김 |
| `multi` | 문항 여러 개 | 표시 (N/M) | 표시 |
| `content-driven` | 지문 문장 수 기반 (CLR) | 별도 진행도 | 없음 |

```
questionCount: 'content-driven' → questionFlowMode 미사용 (CLR: uiTemplate=embedded)
```

### 5.3 QuestionsPanel props

```typescript
questionFlowMode?:    'sequential' | 'deck' | 'locked-steps';  // 카드 렌더 방식
questionCount?:       'single' | 'multi' | 'content-driven';   // 진행도·버튼 표시 여부
questionMaxAttempts?: number | null;
attemptCounts?:       Record<string, number>;
```

**panels.tsx 적용 현황 (2026-06-01 기준):**
```typescript
// 카드 렌더 방식 — questionFlowMode
if (questionFlowMode === 'deck') { /* 카드 1개 렌더 */ }
const visibleQuestions = questionFlowMode === 'locked-steps' ? questions : questions.filter(...);
const isLocked = questionFlowMode === 'locked-steps' && !isAnswered && !isActive;

// 진행도·버튼 — questionCount
{questionCount !== 'single' && <span>N/M 진행도</span>}
{isAnswered && questionCount === 'multi' && <button>다음문항으로 →</button>}
```

### 5.4 VoiceQuestionPanel deck 모드 (2026-04-28 추가)

기존에는 `questionFlowMode: 'deck'` 설정을 무시하고 전체 문장을 누적 표시했다. 개선:

```typescript
// panels.tsx — VoiceQuestionPanel props 추가
questionFlowMode?: 'sequential' | 'deck' | 'locked-steps';

// deck 모드: activeRecordingId 기준 문장 1개만 카드형으로 표시
// sequential 모드(기본): 기존 전체 누적 표시 동작 유지
```

### 5.5 ~~`NO_BINARY_FEEDBACK_TYPES`~~ → `scoringMode` prop (2026-06-01 제거)

#### 이전 방식 (제거됨)

`QuestionsPanel`이 `scoringMode`를 전달받지 못해 question `type`으로 채점 방식을 역추론하는 file-private Set을 유지했다. 새 holistic 타입 추가 시 `HOLISTIC_TYPES`(§9.4), `NO_BINARY_FEEDBACK_TYPES`, DB `scoring_mode` 세 곳을 동시에 수정해야 하는 동기화 부담이 있었다.

```typescript
// ❌ 제거됨 — panels.tsx
const NO_BINARY_FEEDBACK_TYPES = new Set([
  'essay', 'short-text', 'sentence-write', 'keyword-chips',
  'sentence-select', 'sentence-explain',
]);
// !NO_BINARY_FEEDBACK_TYPES.has(q.type) 로 ✓/✗ 표시 여부 판단
```

#### 현재 방식

`scoringMode`를 `QuestionsPanel` prop으로 전달해 DB 값을 단일 소스로 사용한다.

```typescript
// QuestionsPanel props 추가
scoringMode?: 'exact' | 'holistic' | 'pronunciation';

// ✓/✗ 표시 조건 — scoringMode가 undefined이면 표시 안 함 (기존 holistic 동작 유지)
{isAnswered && scoringMode === 'exact' && (
  <p>✓ 정답입니다! / ✗ 틀렸습니다.</p>
)}
```

**전달 경로 (2026-06-01 기준):**
```
pedagogyProfile.scoringMode (LessonSession)
  → QuestionsPanel (scoringMode prop)                   — 데스크톱
  → MobileSplitLayout → QuestionsPanel (scoringMode prop) — 모바일
```

**수정 파일:** `panels.tsx`, `LessonSession.tsx`, `MobileSplitLayout.tsx`

> `ModuleOrchestratorAgent.ts`의 `HOLISTIC_TYPES` / `isHolisticType`은 2026-06-02에 제거 완료. `scoringMode` 직접 참조로 전환됨 (§11 기술 부채 해소).

### 5.6 `isLastQuestion` — "다음문항" / "이해했어요" 버튼 분기 (2026-06-01)

#### 이유

기존 버튼 표시 조건이 `questionCount`에 의존해 두 개념이 혼재됐다.

| 개념 | 기존 조건 | 문제 |
|---|---|---|
| "다음문항으로 →" | `isAnswered && questionCount === 'multi'` | 마지막 문항에도 버튼이 표시됨 |
| "이해했어요 👍" | `awaitingFeedbackConfirm && questionCount !== 'multi'` | single 모듈에서만 표시 |

`questionCount`는 모듈 구조(단문항/다문항)를 나타내는 개념이고, "다음 문항이 존재하는가"는 현재 위치 정보다. 이 두 개념을 분리해야 한다.

**핵심 원칙**: `!isLastQuestion` = "다음 문항이 있다" → QuestionsPanel "다음문항으로 →"  
**핵심 원칙**: `isLastQuestion` = "마지막 문항이다" → FeedbackPanel "이해했어요 👍"

`single` 모듈은 항상 `isLastQuestion === true`이므로 기존 동작이 그대로 유지된다.

#### 구현

**`isLastQuestion` 계산 (LessonSession.tsx)**
```typescript
const isLastQuestion =
  !!activeQuestionId &&
  activeQuestionId === moduleData.questions.at(-1)?.id;
```
모든 `questionFlowMode`에 공통 적용. deck은 현재 카드, sequential/locked-steps는 현재 `activeQuestionId`가 마지막인지 판단.

**QuestionsPanel 버튼 조건**
```typescript
// Before
{isAnswered && questionCount === 'multi' && <button>다음문항으로 →</button>}

// After — 다음 문항이 있을 때만 표시
{isAnswered && !isLastQuestion && <button>다음문항으로 →</button>}
```

**FeedbackPanel "이해했어요" 조건**
```typescript
// Before
{awaitingFeedbackConfirm && !isLoading && questionCount !== 'multi' && ...}

// After — 마지막 문항이면 항상 표시 (single = 항상 마지막)
{awaitingFeedbackConfirm && !isLoading && isLastQuestion && ...}
```

> `questionCount`는 FeedbackPanel에서 제거하지 않는다. 모듈 구조 정보로서 향후 다른 용도에 활용 가능하다.

#### exact 정답 마지막 문항 — 자동완료

exact 모듈 마지막 문항을 정답으로 맞히면 `celebrate` → 2초 후 자동 완료. `awaitingFeedbackConfirm`이 설정되지 않아 "이해했어요" 버튼이 표시되지 않지만, 자동 완료되므로 문제 없다.

#### 수정 파일

`panels.tsx` (QuestionsPanel 2곳, FeedbackPanel 1곳), `LessonSession.tsx`, `MobileSplitLayout.tsx`

---

## 6. 모듈별 특수 렌더링

### 6.1 SKM — sentence-select + timed-select

#### sentence-select UI (2026-04-12)

**버그 수정**: `VALID_ANSWER_TYPES`에 `'sentence-select'` 누락 → `parseAnswerType()`이 `'essay'`로 폴백 → `sentenceSelectMode=false`.
수정: `VALID_ANSWER_TYPES`, `VALID_FEEDBACK_STYLES`에 `'sentence-select'`, `'holistic'`, `'sentence-explain'` 추가.

**ContentPanel 인라인 span 렌더링:**
- 변경 전: 각 문장을 블록 `<button>`으로 렌더 → 지문 형태 파괴
- 변경 후: 단락(`<p>`) 구조 유지, 문장을 인라인 `<span>`으로 분리 → 클릭 가능한 원문 형태

완료 현황:

| Step | 내용 | 상태 |
|---|---|---|
| 1 | `module.ts` QuestionType에 `sentence-select` 추가 | ✅ |
| 2 | DB seed + packages/types 동기화 | ✅ |
| 3 | SKM 프로파일 변경 (holistic, sentence-select) | ✅ |
| 4 | `ContentPanel` 인라인 span 문장 선택 인터랙션 | ✅ |
| 5 | `QuestionsPanel` sentence-select 칩 렌더링 | ✅ |
| 6 | `LessonSession` / `MobileSplitLayout` 상태 연결 | ✅ |
| 7 | ~~`isHolisticType()`에 `sentence-select` 추가~~ → 2026-06-02 `isHolisticType` 제거로 불필요 | ✅ |
| 8 | `messageBuilders` holistic 피드백 분기 보완 | ✅ |

#### timed-select 지문 흐름 (2026-04-26)

```
before → reading → done (자동)

before:  usePassageMode('timed-select') → blur + Floating Timer overlay
reading: "지문읽기" 클릭 → 지문 전체 표시 + 타이머 + sentence-select 즉시 활성
done:    첫 답안 제출 시 자동 전환 (완료 버튼 없음) → blur 없음
→ 오케스트레이터 → holistic 피드백 → 완료
```

### 6.2 CLR — sentence-explain + SentenceExplainPopover

#### SentenceExplainPopover 구조

```
[기본]  해석 | [상세설명↓] [질문하기] [닫기]
[확장]  해석
        ─────────────
        📐 문법: ...
        💡 주요표현: 칩1 칩2 칩3
        | [간략히↑] [질문하기] [닫기]
```

- 클릭된 문장이 속한 단락 바로 아래 **인라인** 렌더링 (플로팅 미채택)
- 플로팅 미채택 이유: 줄바꿈 span rect 계산 복잡도, overflow 클리핑, 모바일 패널 높이 충돌

#### ContentPanel sentence-explain 렌더링

- `sentenceExplainMode` 활성 시 단락을 문장 단위 인라인 `<span>`으로 분리
- 클릭 문장(`activeSentence`): 파란 ring
- 해설 확인 완료 문장(`explainedSentences`): 초록 배경 + `CheckCircle2` 아이콘
- 헤더 배지: `"📖 문장을 클릭하면 해설해드려요"` + 진행도 `(N/M)`

#### LessonSession 상태

```typescript
// 2026-06-01 이후: useEmbeddedMode 훅으로 분리 (§4.5 참조)
const uiTemplate = (moduleData.uiTemplate ?? 'standard') as 'standard' | 'voice' | 'embedded' | 'hidden';
const embedded = useEmbeddedMode(
  uiTemplate === 'embedded',
  moduleData.passage.content,
  submitUserMessage,
  completedRef,
  (explainedCount, totalCount) => { onModuleComplete({ score: Math.round(explainedCount / totalCount * 100) }, ...); },
);
// embedded.explainedSentences, embedded.activeSentence,
// embedded.handleSentenceExplain, embedded.handleManualComplete 등으로 접근
```

> ~~`effectiveUiTemplate` / `sentenceExplainMode` boolean~~은 2026-06-01 DB 마이그레이션과 함께 제거됨 (§4.2 참조).

완료 기준: `passage.content`를 직접 파싱해 문장 수 계산. 80% 이상 탐색 시 완료 버튼 표시.

> ⚠️ `moduleData.questions.length` 사용 금지. 최초 진입 시 questions가 1개뿐이어서 첫 클릭에 즉시 완료 버그 발생.

완료 현황:

| 항목 | 상태 |
|---|---|
| 타입 정의 (module.ts, ai-module-data.ts, agent-types.ts) | ✅ |
| aiModuleService VALID_ANSWER_TYPES / VALID_FEEDBACK_STYLES 추가 | ✅ |
| GenericAdapter options JSONB 파싱 | ✅ |
| GenericAdapter `explanation` 컬럼 우선 / `options` 폴백 파싱 (2026-07-02) | ✅ |
| panels.tsx SentenceExplainPopover + ContentPanel 렌더링 | ✅ |
| LessonSession 상태·핸들러·완료 조건 | ✅ |
| MobileSplitLayout props 연결 | ✅ |
| ModuleOrchestratorAgent CLR 가드·null 에러 수정 | ✅ |
| messageBuilders.ts sentence-explain 방어 분기 | ✅ |
| seed.ts CLR 모듈 설정 + contentGenerationInstruction | ✅ |
| question-generation.ts CLR 전용 JSON 출력 템플릿 | ✅ |

### 6.3 WSD / RRD / SWR — hidden 지문 + deck + voice

§7 표 참조: WRD는 `passageMode='full'` (hidden 아님). hidden 모드 모듈은 RRD, SWR.

- `passageMode='hidden'` (RRD, SWR): ContentPanel wrapper div 자체 미렌더 (flex 공간 미점유)
- `passageMode='full'` (WRD, WSD): 지문 정상 표시
- `questionFlowMode='deck'`: 카드 1개씩 순환
- WSD, RRD: `uiTemplate='voice'` (DB 직접값) → VoiceQuestionPanel

### 6.4 SCN — timed-blur + keyword-chips

**keyword-chips 답안 입력 UI (2026-04-10):**

SCN 모듈은 기억나는 키워드를 최대한 많이 입력하는 과제이므로 textarea 대신 chip 방식 채택.

- 입력창 + "추가" 버튼: 키워드를 chip으로 하나씩 추가
- 중복 방지, chip 삭제 (X 버튼)
- Enter 키 → 키워드 추가 (제출 아님)
- "제출하기": chip들을 공백 구분 문자열로 일괄 제출
- 제출 후: 읽기 전용 pill 표시

타입 정의: `'keyword-chips'` — `module.ts` QuestionType, `ai-module-data.ts` answerType, `aiModuleService.ts` VALID_ANSWER_TYPES

**short-text 문항 — textarea → input 변경 (2026-04-10):**

SCN keyword-chips와 별개로, `short-text` 타입(1~3 단어 답변)을 기존 textarea에서 단일 행 `<input>`으로 변경. Enter 키 자동 제출 추가.

---

## 7. 모듈 전체 렌더링 설정표 (2026-06-01 기준)

| 모듈 | question_count | passage_mode | ui_template (DB) | question_flow_mode | 비고 |
|---|---|---|---|---|---|
| WRD | multi | full | standard | deck | ✅ |
| WSD | multi | full | **voice** | deck | ✅ |
| GMN | multi | full | standard | deck | ✅ |
| WWB | multi | full | standard | sequential | ⚠️ 기술 부채 |
| PRD | single | preview | standard | sequential | ✅ |
| SCN | single | timed-blur | standard | sequential | ✅ |
| SKM | single | timed-select | standard | sequential | ✅ |
| CLR | content-driven | full | **embedded** | — | ✅ |
| SUM | single | full | standard | sequential | ✅ |
| QAR | multi | full | standard | deck | ✅ |
| RRD | multi | **hidden** | **voice** | deck | ✅ |
| SWR | multi | **hidden** | standard | deck | ✅ |
| SHD | multi | full | **voice** | sequential | ⚠️ 기술 부채 |
| WSP | multi | full | **voice** | sequential | ⚠️ 기술 부채 |
| PWR | multi | full | standard | locked-steps | ✅ |

> 2026-06-01 DB 마이그레이션 완료: `ui_template` 컬럼이 실제값(`voice`/`embedded`)으로 업데이트됨. `resolveUiTemplate()` 함수 제거 (§4.2~4.3 참조).

---

## 8. DB / 백오피스 연동

### 8.1 ai_modules 컬럼 추가 SQL

```sql
-- backoffice + tutoring 양쪽 ai_modules 테이블에 실행
ALTER TABLE ai_modules ADD COLUMN IF NOT EXISTS passage_mode       VARCHAR(30) NOT NULL DEFAULT 'full';
ALTER TABLE ai_modules ADD COLUMN IF NOT EXISTS question_flow_mode VARCHAR(30) NOT NULL DEFAULT 'sequential';
```

Prisma schema:
```prisma
passageMode      String @default("full")       @map("passage_mode")       @db.VarChar(30)
questionFlowMode String @default("sequential") @map("question_flow_mode") @db.VarChar(30)
```

> 기존 데이터는 모두 default값(`full` / `sequential`)으로 유지 — 동작 변경 없음.

### 8.2 코드그룹 seed (PASSAGE_MODE / QUESTION_FLOW_MODE / UI_TEMPLATE)

**PASSAGE_MODE:**
```typescript
{ code: 'full',         name: '전체 표시',        sortOrder: 1 },
{ code: 'hidden',       name: '숨김',             sortOrder: 2 },
{ code: 'preview',      name: '미리보기 후 공개', sortOrder: 3 },
{ code: 'timed-blur',   name: '타이머 후 블러',   sortOrder: 4 },
{ code: 'timed-select', name: '타이머 후 선택',   sortOrder: 5 },
```

**QUESTION_FLOW_MODE:**
```typescript
{ code: 'sequential',   name: '단문항 표시',       sortOrder: 1 },
{ code: 'deck',         name: '다문항 카드형 순환', sortOrder: 2 },
{ code: 'locked-steps', name: '단문항 단계 잠금형', sortOrder: 3 },
```

**UI_TEMPLATE (재정의):**
```typescript
{ code: 'standard', name: '기본 문항 패널',         sortOrder: 1 },
{ code: 'voice',    name: '음성 문항 패널',          sortOrder: 2 },
{ code: 'embedded', name: '지문 임베드 (패널 없음)', sortOrder: 3 },
{ code: 'hidden',   name: '문항 패널 없음',          sortOrder: 4 },
```

**레거시 항목 제거 (Supabase SQL 직접 실행):**
- `chat` (id 629), `quiz` (id 630), `card` (id 631) 삭제
- 해당 값을 사용하는 ai_modules 없음 확인 후 삭제

### 8.3 module_questions 컬럼 구조 (2026-07-02 정규화 완료)

**문제 배경:**
`module_questions.answer` TEXT 컬럼에 두 가지 포맷이 혼용됨.
- `contentGenerationInstruction` 적용 모듈(QAR 등): `{"correctOption":"③","evidence":"...","questionType":"유형 4"}` (JSON 문자열)
- 기타 모듈(short-text, essay 등): `"urban"` 등 단순 문자열

오케스트레이터 / GenericAdapter 채점 로직에서 DB값을 그대로 `q.answer`로 비교하므로,
UI가 `"③"` 제출 → DB `{"correctOption":"③",...}` 비교 → 항상 오답 처리.

**해결 (Phase 2+3):**

`apps/api/prisma/manual-sql/2026-07-02_module_questions_explanation_column.sql` 실행:

```sql
-- Phase 2: 컬럼 추가
ALTER TABLE module_questions ADD COLUMN IF NOT EXISTS explanation JSONB;

-- Phase 3: JSON answer 레코드 정규화
UPDATE module_questions
SET
  explanation = (answer::jsonb - 'correctOption'),
  answer      = answer::jsonb->>'correctOption'
WHERE
  answer LIKE '{%' AND answer::jsonb ? 'correctOption';
```

이후 구조:
| 컬럼 | 타입 | 내용 |
|---|---|---|
| `answer` | TEXT | 채점 기준값만 (항상 단순 문자열. 예: `"③"`, `"urban"`, `""`) |
| `explanation` | JSONB | 부가 메타데이터. multiple-choice: `{evidence, evidenceReason, questionType}`. sentence-explain(CLR 예정): `{해석, 문법설명, 주요표현}` |

**GenericAdapter 대응:**
- `q.answer`: `String(q.answer ?? '')` — 항상 단순 문자열이므로 `correctOption` 추출 불필요
- `sentence-explain` 파싱: `q.explanation ?? q.options` 순서로 폴백 (CLR은 아직 `options`에 저장 중)
- `options` 처리: `Array.isArray(q.options)` 조건으로 string[] 전용 파싱 — JSONB 객체 제외

**향후 할 일:**
- CLR `sentence-explain` 데이터를 `options` JSONB → `explanation` JSONB로 마이그레이션 (GenericAdapter는 이미 양쪽 폴백 지원)
- 백오피스 `contentGenerationInstruction` 템플릿에서 `multiple-choice` answer를 `"③"` 형식으로 생성하도록 수정

### 8.4 DTO 및 서비스 변경

**types/src/index.ts:**
```typescript
// AiModuleResponse
passageMode:      string;
questionFlowMode: string;

// CreateAiModuleDto / UpdateAiModuleDto
passageMode?:      string;
questionFlowMode?: string;
```

**ai-module.service.ts:**
```typescript
// Create
passageMode:      dto.passageMode      ?? 'full',
questionFlowMode: dto.questionFlowMode ?? 'sequential',

// Update
if (dto.passageMode      !== undefined) updateData.passageMode      = dto.passageMode;
if (dto.questionFlowMode !== undefined) updateData.questionFlowMode = dto.questionFlowMode;
```

**register/page.tsx 변경:**
- `uiTemplate` 레이블: `화면 레이아웃 템플릿` → `문항 패널 동작 모드`
- `questionFlowMode` 설명: `단문항 전용 / 다문항 전용` 명시
- `passageMode` 설명: `hidden=지문 패널 DOM 제거` 추가

---

## 9. 주요 설계 결정 이력

### 9.1 uiTemplate switch-case 제거 (2026-04-08)

기존 `VocabDeckFlow`, `ProcessWritingFlow`, `InteractiveFlow` 별도 컴포넌트 방식은 신규 모듈마다 코드 변경이 필요해 R4 원칙 위반. `passageMode + questionFlowMode + uiTemplate` 3축으로 `ModuleRunnerInner` 단일 컴포넌트에서 모든 동작을 처리하도록 전환. 삭제된 파일:
- `VocabDeckFlow.tsx`, `ProcessWritingFlow.tsx`, `InteractiveFlow.tsx`

### 9.2 celebrate/completeModule 자동 실행 버그 수정 (2026-04-10)

**버그**: essay/short-text 모듈에서 holistic 피드백 완료 전에 `celebrate → completeModule`이 자동 실행.

**수정 1** — `ruleCorrectAnswer`에 holistic 피드백 가드:
```typescript
// 2026-04-10 초기 수정 — q.type 기반 (이후 교체됨)
// if ((q.type === 'essay' || q.type === 'short-text') && ...) { return null; }

// 2026-06-02 현재 코드 — scoringMode 직접 참조 (§9.4 참조)
if (
  !hasRightWrongConcept(context) &&
  context.pedagogyProfile.scoringMode === 'holistic' &&
  !(context.feedbackGivenIds['holistic'] ?? []).includes(lastId)
) {
  return null;  // holistic 피드백 완료 전엔 celebrate 진행 안 함
}
```

**수정 2** — `awaitingFeedbackConfirm` 상태 중 유휴 타이머 차단:
```typescript
const awaitingFeedbackConfirmRef = useRef<string | null>(null);
// 유휴 타이머 진입 시
if (awaitingFeedbackConfirmRef.current) return;
```

### 9.3 greet 툴 독립 분리 (Rule 0) (2026-04-10)

**문제**: `passageMode='hidden'` 모듈(RRD, SWR 등)은 `showPassage`가 스킵되어 인사 메시지가 전혀 표시되지 않음. timed-blur(SCN) 등도 `before` 페이즈 동안 `showPassage`가 지연되므로 같은 문제가 발생.

**해결**: `greet` 툴을 Rule 0으로 독립. `showPassage`는 지문 표시만 담당.

```typescript
// agent-types.ts — greet 툴 추가
| ({ tool: 'greet'; params: { purpose: string }; } & BaseToolCall)

// OrchestratorContext에 greetingShown 추가
greetingShown: boolean;

// ruleGreet — 항상 체인 맨 앞
function ruleGreet(state, context): OrchestratorToolCall | null {
  if (context.greetingShown) return null;
  if (Object.keys(state.answers).length > 0) return null;
  if (state.questionsAsked.length > 0) return null;  // 사용자가 먼저 질문한 경우도 스킵
  return { tool: 'greet', params: { purpose: context.pedagogyProfile.purpose }, reason: '...' };
}

// executeToolCall greet 케이스
case 'greet': {
  const greetTime = Date.now();
  postAiMessage(toolCall.params.purpose);   // fallback 즉시 표시
  greetingShownRef.current = true;
  generateGreeting(input, toolCall.params.purpose).then((text) => {
    if (text && text !== toolCall.params.purpose) {
      // 첫 메시지 후 최소 600ms 간격 보장
      const showDelay = Math.max(0, 600 - (Date.now() - greetTime));
      setTimeout(() => {
        postAiMessage(text);
        setTimeout(() => orchestrateRef.current(true), 300);  // LLM 응답 이후 다음 규칙
      }, showDelay);
    } else {
      setTimeout(() => orchestrateRef.current(true), 300);    // LLM 실패 시 즉시 진행
    }
  });
  break;
}
```

### 9.4 HOLISTIC_TYPES 상수화 (2026-04-10) → scoringMode 직접 참조로 전환 (2026-06-02)

**도입 배경 (2026-04-10)**: `keyword-chips` 타입 추가 후 6곳의 holistic 체크가 누락 → LLM 피드백 없이 즉시 완료되는 버그. 당시 `answerType`으로 채점 방식을 역추론하는 상수로 일괄 관리.

**제거 배경 (2026-06-02)**: `scoringMode`(채점 방식)와 `answerType`(입력 형태)은 독립적인 두 축이며 둘 다 DB에서 직접 정의한다. `answerType`으로 `scoringMode`를 역추론하는 것 자체가 잘못된 설계였다. WRD·GMN이 동일한 `short-text`이면서 채점 방식이 다른 것이 이를 증명한다.

**현재 코드 (2026-06-02 기준):**

```typescript
// 삭제됨
// const HOLISTIC_TYPES = [...] as const;
// function isHolisticType(type: string): boolean { ... }

// calcScore — scoringMode 파라미터로 직접 판단
function calcScore(questions, answers, scoringMode: 'exact' | 'holistic' | 'pronunciation'): number {
  if (scoringMode === 'pronunciation') { /* 발음 점수 평균 */ }
  const correct = questions.filter((q) =>
    scoringMode !== 'exact' ? !!answers[q.id] : answers[q.id] === q.answer,
  ).length;
  ...
}

// ruleCorrectAnswer — hasRightWrongConcept이 false면 답변 존재 여부로 정답 판정
const isAnsweredCorrectly = hasRightWrongConcept(context)
  ? state.answers[lastId] === q.answer
  : !!state.answers[lastId];
```

새 holistic 모듈 추가 시 DB `scoring_mode = 'holistic'`만 설정하면 오케스트레이터 코드 수정 불필요.

---

## 10. UI 개선 이력 (2026-04-10)

| # | 항목 | 핵심 변경 | 관련 파일 |
|---|---|---|---|
| 1 | 나의수업 메뉴 + CourseDrawer | `나의수업` 메뉴 추가, 우측 슬라이드 드로어로 과정→레슨→모듈 탐색 | `StudioHeader.tsx`, `my-classes/page.tsx`, `CourseDrawer.tsx` |
| 2 | AI-Modules 리스트 필드 수정 + 6개 필터 | 컬럼 재정의 (answerType, scoringMode 등), FilterState 인터페이스 | `admin/ai-modules/page.tsx` |
| 3 | getEnrolledCourses MODULE_LABELS 하드코딩 제거 | ai_modules JOIN으로 모듈명/스킬 서버에서 전달 | `lessons.service.ts` |
| 4 | PRD 버그 — 제목 중복·도입 2문장 | greeting에서 `passage.title` 제거, 문장 단위 2개 slice | `ModuleOrchestratorAgent.ts`, `panels.tsx` |
| 5 | celebrate/completeModule 자동 실행 버그 | holistic 피드백 가드, awaitingFeedbackConfirmRef (§9.2 참조) | `ModuleOrchestratorAgent.ts`, `useModuleOrchestrator.ts` |
| 6 | revealPassage — 피드백과 동시 지문 공개 | `giveFeedback holistic`에 `revealPassage` 플래그 (§3.5 참조) | `agent-types.ts`, `ModuleOrchestratorAgent.ts`, `useModuleOrchestrator.ts` |
| 7 | "다시 설명해주세요" 버튼 제거 | `onQuickReply` `() => void`로 단순화, placeholder 업데이트 | `panels.tsx`, `LessonSession.tsx`, `MobileSplitLayout.tsx` |
| 8 | timed-blur Blur 처리 수정 | `blurActive` 플래그 분리 (§3.2 참조) | `LessonSession.tsx`, `MobileSplitLayout.tsx` |
| 9 | greet 툴 독립 분리 | Rule 0 추가 (§9.3 참조) | `agent-types.ts`, `ModuleOrchestratorAgent.ts`, `useModuleOrchestrator.ts` |
| 10 | Floating Timer 구현 | footer 제거, 우측 하단 floating (§3.4 참조) | `LessonSession.tsx`, `panels.tsx` |
| 11 | short-text → `<input>` 변경 | Enter 키 자동 제출, placeholder 구체화 | `panels.tsx` |
| 12 | before 페이즈 blur 추가 | `blurActive = readingPhase === 'before' \|\| ...` | `LessonSession.tsx` |
| 13 | Floating Timer 단계별 재설계 | before=overlay, reading=헤더 inline (§3.4 참조) | `panels.tsx` |
| 14 | SCN keyword-chips UI | chip 추가/삭제/제출, 일괄 submit (§6.4 참조) | `panels.tsx`, `module.ts`, `ai-module-data.ts` |
| 15 | HOLISTIC_TYPES 상수화 | `isHolisticType()` 헬퍼 (§9.4 참조) | `ModuleOrchestratorAgent.ts` |
| 16 | 피드백 아키텍처 Phase 1 | pedagogyInstruction 단일 소스화, elapsedSeconds 추가 | `feedback.ts`, `ai.controller.ts`, `useModuleOrchestrator.ts` |
| 17 | usePassageMode switch 재구성 (2026-06-01) | if 체인 → switch 5-case, timer effect 단순화, timedBase 공유 객체 (§3.2 참조) | `LessonSession.tsx` |
| 18 | uiTemplate DB 마이그레이션 + resolveUiTemplate 제거 (2026-06-01) | WSD/RRD/SHD/WSP→voice, CLR→embedded UPDATE 실행. `resolveUiTemplate` 함수·boolean 파생 변수 제거, `uiTemplate` 단일 상수로 통합. `VALID_UI_TEMPLATES` 전체 4종으로 확장 (§4.2~4.4 참조) | `LessonSession.tsx`, `aiModuleService.ts`, `ai-module-data.ts` |
| 19 | useEmbeddedMode 훅 분리 (2026-06-01) | CLR 전용 로직 50줄을 훅으로 추출, 메인 컴포넌트에서 `embedded.*` 네임스페이스로 접근 (§4.5 참조) | `LessonSession.tsx` |
| 20 | questionFlowMode·questionCount 역할 분리 원칙 확립 (2026-06-01) | questionFlowMode=카드 렌더 전용, questionCount=진행도·버튼 전용으로 관심사 분리. deck 모드 진행도·버튼에 questionCount 조건 추가. PWR questionFlowMode를 locked-steps로 확정 (§5.0~5.3 참조) | `panels.tsx` |
| 21 | NO_BINARY_FEEDBACK_TYPES 제거 (2026-06-01) | `scoringMode` prop을 `QuestionsPanel`까지 전달. question `type` 역추론 Set 제거. `!NO_BINARY_FEEDBACK_TYPES.has(q.type)` → `scoringMode === 'exact'` (§5.5 참조) | `panels.tsx`, `LessonSession.tsx`, `MobileSplitLayout.tsx` |
| 22 | `isLastQuestion` 도입 (2026-06-01) | "다음문항으로 →" 조건 `questionCount === 'multi'` → `!isLastQuestion`. "이해했어요" 조건 `questionCount !== 'multi'` → `isLastQuestion`. 마지막 문항 기준으로 두 버튼을 명확히 분리. (§5.6 참조) | `panels.tsx`, `LessonSession.tsx`, `MobileSplitLayout.tsx` |
| 22-1 | `QuestionPanelFooter` 컴포넌트 분리 + `LessonSession` VoiceQuestionPanel props 보완 | **분리 이유**: `QuestionsPanel`과 `VoiceQuestionPanel`이 "다음문항 →" / "다시시도" 버튼 로직을 각각 독립 구현 → 중복. 공유 컴포넌트 `QuestionPanelFooter`로 추출. **적용 위치**: QuestionsPanel deck·sequential, VoiceQuestionPanel deck·sequential (총 4곳). **렌더 조건**: `isAnswered && !isLastQuestion`일 때만 렌더 — 마지막 문항의 "이해했어요"는 FeedbackPanel 담당. **LessonSession 수정**: `VoiceQuestionPanel` 호출부에 `isLastQuestion`, `questionMaxAttempts`, `attemptCounts`, `onNextQuestion`, `onRetryAnswer` 5개 props 누락 추가. | `panels.tsx`, `LessonSession.tsx` |
| 23 | ContentPanel 초기 preview 플래시 수정 (2026-06-02) | **원인**: `useModuleOrchestrator`의 `passageVisible` 초기값 `false` → `previewOnly = !false \|\| false = true` → 앞 2문장이 플래시. `full`과 `preview` 케이스가 모두 `passageVisibleOverride: undefined`를 반환해 동일 경로를 타는 것이 근본 원인. **수정**: `full` 케이스를 `passageVisibleOverride: true`로 변경 — `timed-*` 모드와 동일하게 훅이 visibility를 즉시 확정. 이후 오케스트레이터의 `showPassage` 호출은 `setPassageHighlights` 등 부수 효과만 처리. | `LessonSession.tsx` |
| 24 | QuestionsPanel / VoiceQuestionPanel deck 미시작 플래시 제거 + 로딩 메시지 일관성 (2026-06-02) | **원인**: `activeQuestionId` 초기값 `null` → deck 모드의 `?? questions[questions.length - 1]` 폴백이 마지막 문항을 렌더 → 플래시. 폴백 의도는 "모두 완료 후 마지막 카드 유지"였으나 미시작 상태와 구분 불가. **수정(QuestionsPanel)**: `activeQuestionId`가 `null`이면 로딩 메시지 표시. `setActiveQuestionId`는 초기화 이후 `null`로 재설정되지 않으므로 완료 후 동작 무변경. **수정(VoiceQuestionPanel deck)**: `activeRecordingId`는 문항 완료 후 오케스트레이터가 일시적으로 `null`로 리셋하므로, `activeQuestionId`와 동일한 단순 null 체크를 적용할 수 없다. 대신 `doneCount > 0`(답변 이력 유무)으로 "완료 후 대기"와 "미시작"을 구분 — `doneCount > 0`이면 마지막 카드 유지, `0`이면 로딩 메시지 표시. | `panels.tsx` |
| 25 | `sequential` 모드 전체 문항 동시 표시 (2026-06-02) | **이전**: "답변된 문항 + 현재 활성 문항만 누적 표시" → deck과 UX 차이 없음. **변경**: `visibleQuestions = questions` (전체). deck=집중형(1개), sequential=조망형(전체·자유), locked-steps=단계형(전체·잠금)으로 세 모드 역할 명확히 분리. (§5.1 참조) | `panels.tsx` |
| 26 | QuestionsPanel 문항 표시 게이트 — `activeQuestionId` 기반 (2026-06-02) | **의도**: 인사말(FeedbackPanel)이 먼저 표시되고 문항이 그 다음에 나와야 함. 모든 `questionFlowMode` · `questionCount`에 일관 적용. **deck**: `activeQuestionId ? find ?? last : null` — null이면 "준비 중이에요...". **sequential·locked-steps**: `visibleQuestions = activeQuestionId !== null ? questions : []`. 두 경로 모두 `askQuestion(Q1)` 발동 시점(`ruleGreet`+`ruleInitialEntry` 완료 후)에 문항 표시. (§5.1.1 참조) | `panels.tsx` |
| 27 | VoiceQuestionPanel sequential 모드 `activeRecordingId` 게이트 추가 (2026-06-02) | sequential+voice(SHD, WSP)에서 `activeRecordingId = null` 구간에 "Pickle AI가 문항을 준비 중이에요..." 미표시 버그 수정. **수정**: `!activeRecordingId && doneCount === 0`일 때만 로딩 메시지 표시 — 미시작 구간만 게이트, 완료 후 일시 null 구간은 목록 유지. (deck 모드와 동일하게 `doneCount` 기준으로 미시작 vs 완료 후 대기를 구분 — §24 참조) passageMode · questionFlowMode · uiTemplate 조합에 무관하게 모든 패널에서 인사말 선행 보장. | `panels.tsx` |
| 28 | QuestionsPanel 입력 위젯 `q.type` 기준 통일 (2026-06-02) | **이전**: 입력 위젯이 `questionFlowMode`에 의존 — deck `short-text`=`<textarea>`, sequential `short-text`=`<input>`. **변경**: `q.type`만으로 결정. `short-text`→`<input>`(한 줄, Enter 제출), `essay`→`<textarea>`(여러 줄). deck / sequential / locked-steps 전 모드에서 동일 위젯 사용. | `panels.tsx` |

---

## 11. 기술 부채

| 항목 | 내용 | 해소 조건 |
|---|---|---|
| ~~`resolveUiTemplate` 하위 호환 로직~~ | ~~DB `ui_template`이 전체 `'standard'`인 동안 `answerType`으로 실효값 결정~~ | **✅ 해소** — DB 마이그레이션 완료 후 함수 제거 (2026-06-01) |
| `uiTemplate` 컬럼명 의미 불일치 | DB 컬럼명은 `ui_template`이나 실질적으론 "문항 패널 동작 모드" | DB 컬럼 rename은 마이그레이션 비용 큼 → 문서로 관리 |
| WWB `questionFlowMode: sequential` | multi 모듈인데 sequential 유지 중 | 별도 의사결정 후 deck 전환 |
| ~~PWR `questionFlowMode: sequential`~~ | ~~multi 모듈인데 sequential 유지 중~~ | **✅ 해소** — locked-steps로 확정 (2026-06-01) |
| SHD, WSP `questionFlowMode: sequential` | voice 패널 + multi인데 sequential. deck 전환 시 VoiceQuestionPanel deck 동작 확인 필요 | 모듈 운영 방침 확정 후 전환 |
| 삭제 모듈 DB 잔류 | WDR, WDS, SHR(삭제예정), IMG, SNR — DB `status: active` 잔류. seed.ts 미등록, 소스 참조 없음 | 백오피스 UI에서 `status: inactive` 처리 권장 |
| ~~`HOLISTIC_TYPES` / `isHolisticType` 잔류~~ | ~~`ModuleOrchestratorAgent.ts`의 `HOLISTIC_TYPES` + `isHolisticType(q.type)` 3곳이 `scoringMode`와 중복~~ | **✅ 해소** — `scoringMode` / `answerType` 직접 참조로 전환. 상수·함수 삭제 (2026-06-02) |

---

## 12. 확장 가이드 — 신규 모듈 렌더링 설정

### passageMode 선택 기준

```
지문 없는 모듈          → passageMode: 'hidden'
지문 항상 전체 표시     → passageMode: 'full'     (기본값, 생략 가능)
지문 일부 → 전체 공개  → passageMode: 'preview'
타이머 읽기 (blur 후)  → passageMode: 'timed-blur'
타이머 읽기 (선택 후)  → passageMode: 'timed-select'
```

### uiTemplate 선택 기준

```
텍스트 입력 문항  → uiTemplate: 'standard'  (기본값, 생략 가능)
음성 입력 문항   → uiTemplate: 'voice'      (반드시 DB에 명시)
지문 임베드 방식  → uiTemplate: 'embedded'  (반드시 DB에 명시)
문항 패널 없음   → uiTemplate: 'hidden'     (향후 확장용)
```

> `answerType` 기반 자동 결정(`resolveUiTemplate`)은 2026-06-01 제거됨. DB `ui_template` 컬럼에 직접 등록한다.

### questionFlowMode 선택 기준

```
questionCount: 'single'         → questionFlowMode 생략 (sequential 기본값)
questionCount: 'multi'          → questionFlowMode: 'deck' 반드시 명시
questionCount: 'content-driven' → questionFlowMode 미사용
```

### 신규 모듈 추가 체크리스트

1. 백오피스에서 `ai_modules` 등록 (passageMode / uiTemplate / questionFlowMode 설정)
2. `answerType`이 기존 5종(multiple-choice, essay, short-text, sentence-write, audio-record)이면 코드 변경 없음
3. 새 `answerType` 추가 시: `VALID_ANSWER_TYPES`, 관련 타입 파일 수정. `scoringMode`는 DB에서 직접 제어하므로 오케스트레이터 코드 수정 불필요
4. 새 `passageMode` 추가 시: §3.6 체크리스트 7개 파일 수정
