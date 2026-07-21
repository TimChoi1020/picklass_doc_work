# `/modules/[lessonId]` 페이지 문서

> 최초 작성: 2026-03-25 / 최종 수정: 2026-03-30
>
> **변경 이력**
> - 2026-03-27: PWR 어댑터 등록, L008 mock 추가, ModuleRunner PWR 분기
> - 2026-03-30: `giveFeedback` 유니온 통합 (givePronunciationFeedback·giveWritingFeedback 제거),
>   `OrchestratorContext` 4개 전용 필드 → `eventFiredIds`/`feedbackGivenIds` 범용 맵으로 교체,
>   `showPassage.params.greeting` 추가 (stage1Greeting 제어권 Orchestrator로 이전),
>   섹션 5–8 신규 추가 (코딩 규칙·기술 부채·의존성·Data Contract)
>
> **파일 구조**
> ```
> src/app/modules/[lessonId]/
>   page.tsx                          ← 라우트 진입점만
>   _components/
>     panels.tsx                      ← UI 패널 컴포넌트 일체
>     MobileSplitLayout.tsx           ← 모바일 드래그 레이아웃
>     LessonSession.tsx               ← 세션/모듈 실행 레이어
>     ProcessWritingFlow.tsx          ← PWR 전용 4단계 쓰기 UI
>
> src/lib/hooks/
>   useModuleOrchestrator.ts          ← 학습 흐름 훅 (Orchestrator 기반 모듈용)
>   useStageStateMachine.ts           ← Stage 기반 상태 머신 (현재 미사용)
>   usePanelRouter.ts                 ← 패널 라우팅 (현재 미사용)
>   messageBuilders.ts                ← 정적 메시지 생성 헬퍼
>
> src/lib/agents/
>   orchestrator/ModuleOrchestratorAgent.ts  ← Rule-based 결정 엔진
>   ChatAgent.ts                      ← 채팅 AI 에이전트 인터페이스 + Mock
>   FeedbackAgent.ts                  ← 피드백 AI 에이전트 인터페이스 + Mock
>   InterventionAgent.ts              ← 개입 AI 에이전트 인터페이스 + Mock
>
> src/lib/adapters/
>   ModuleAdapter.ts                  ← 어댑터 인터페이스
>   PredictionAdapter.ts              ← PRD 어댑터
>   ShadowReadingAdapter.ts           ← SHR 어댑터
>   SentenceWritingAdapter.ts         ← SWR 어댑터
>   ProcessWritingAdapter.ts          ← PWR 어댑터
>   index.ts                          ← 어댑터 레지스트리
>
> src/lib/types/
>   module.ts                         ← 학습 콘텐츠 도메인 타입 (PassageData, QuestionData 등)
>   ai-module.ts                      ← ContentConfig, PanelAssignment 등 UI 배치 타입
> ```
> 관련 파일: `agent-types.ts`, `lessonService.ts`

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 전체 레슨 흐름

```
① URL 진입: /modules/[lessonId]
   └─ page.tsx → fetchLessonPlan(lessonId) → LessonSession 렌더

② LessonSession
   └─ moduleSequence[0] → ModuleRunner (key=0)

③ ModuleRunner
   ├─ getAdapter(moduleCode) — 어댑터 선택
   ├─ adapter.fetchModuleData(lessonId) — 콘텐츠 로드
   ├─ adapter.buildContentConfig(data) — UI 정책 생성
   └─ moduleCode === 'PWR'?
       ├─ YES → ProcessWritingFlow (4단계 수동 흐름)
       └─ NO  → ModuleRunnerInner + useModuleOrchestrator

④ 모듈 완료 (moduleResult 설정)
   └─ LessonSession.handleModuleComplete()
       ├─ saveModuleHistory() (비동기, 실패 무시)
       ├─ currentModuleIdx + 1
       └─ 다음 모듈? YES → ③ 반복 / NO → 최종 완료 화면
```

### 1-2. Orchestrator 기반 모듈 (PRD·SHR·SWR) 단계별 흐름

```
마운트
  └─ useModuleOrchestrator → orchestrate() 첫 호출

[Rule 7] ruleInitialEntry
  └─ showPassage(previewOnly? | autoAdvance?, greeting?)
      ├─ PRD: previewOnly=true, greeting=stage1Greeting → 400ms 후 silent
      └─ SHR/SWR/기타: autoAdvance=true, greeting=stage1Greeting → 800ms 후 silent

[Rule 8] rulePresentNextQuestion
  └─ askQuestion(questionId, guidance?)
      └─ activeQuestionId 설정 → QuestionsPanel / VoiceQuestionPanel 활성화

사용자 상호작용 (3종)
  ├─ submitAnswer(qId, value) → orchestrate()
  ├─ submitVoiceAnswer(qId)   → orchestrate()  [SHR]
  └─ submitUserMessage(text)  → orchestrate()  [AI 질문]

피드백 처리 (모듈별 분기)
  ├─ PRD essay      → giveFeedback(type:'holistic') → awaitingFeedbackConfirm=true
  ├─ SWR sentence-write → giveFeedback(type:'writing') → awaitingWritingConfirm=questionId
  └─ SHR audio-record  → giveFeedback(type:'pronunciation') → awaitingPronunciationConfirm=questionId

피드백 확인 버튼
  ├─ holistic: "다음" → setAwaitingFeedbackConfirm(false) → (자동 진행)
  ├─ writing:  "다음 문항으로 →" → confirmWritingFeedback() → orchestrate()
  └─ pronunciation:
      ├─ "다시 시도 🎤"      → retryVoiceAnswer(qId) → 이력 초기화 → orchestrate()
      └─ "다음 문항으로 →"  → confirmPronunciationFeedback() → orchestrate()

모든 문항 완료
  └─ celebrate(score) → 2000ms → completeModule(score, summary)
      └─ moduleResult 설정 → LessonSession.onModuleComplete()
```

### 1-3. PRD 세부 흐름

```
1. ruleInitialEntry
   → showPassage({ previewOnly: true, greeting })
   → FeedbackPanel: greeting 메시지 게시
   → 400ms 후 silent orchestrate

2. rulePresentNextQuestion
   → askQuestion('q1') — 단일 essay 문항

3. 학습자 서술 제출
   → submitAnswer('q1', text) → orchestrate()

4. ruleHolisticFeedback
   → giveFeedback({ type: 'holistic', elaborate: true })
   → awaitingFeedbackConfirm = true

5. ruleRevealFullPassageAfterFeedback
   → showPassage({}) — 전체 지문 공개
   → "이제 전체 지문을 읽어보세요! 예측과 어떻게 다른지 비교해 보세요. 📖"
   → 800ms 후 silent orchestrate

6. ruleCorrectAnswer → celebrate(score)
   → 2000ms 후 silent orchestrate

7. ruleCompleteAfterCelebrate → completeModule
```

### 1-4. SHR 세부 흐름

```
1. ruleInitialEntry
   → showPassage({ autoAdvance: true, greeting })
   → 800ms 후 silent orchestrate

2. rulePlayModelAudio (SHR-A)
   → playModelAudio({ questionId, text })
   → "Q1번 문장을 들어보세요! 🎧\n[문장]"
   → 2500ms(Mock) 후 silent orchestrate

3. ruleStartRecordingAfterAudio (SHR-B)
   → startRecording({ questionId, prompt })
   → 말하기 버튼 활성화

4. submitVoiceAnswer(qId) → Mock 점수(70-100) → orchestrate()

5. rulePronunciationFeedback (SHR-C)
   → giveFeedback({ type: 'pronunciation', score, fluencyPoints, pronunciationPoints, elaborate })
   → awaitingPronunciationConfirm = questionId

6. "다시 시도" → retryVoiceAnswer() / "다음 문항으로 →" → confirmPronunciationFeedback()

7. (2–6 반복)

8. 모든 문항 완료 → celebrate → completeModule
```

### 1-5. SWR 세부 흐름

```
1. ruleInitialEntry
   → showPassage({ autoAdvance: true, greeting })
   → 800ms 후 silent orchestrate

2. rulePresentNextQuestion
   → askQuestion(qId, guidance?) — sentence-write 문항
   → QuestionsPanel: 한글 패러프레이즈 + textarea

3. submitAnswer(qId, 영작 텍스트) → orchestrate()

4. ruleWritingFeedback (SWR-A)
   → giveFeedback({ type: 'writing', accuracyScore, complexityScore, grammarErrorRate, ... })
   → awaitingWritingConfirm = questionId

5. "다음 문장으로 →" → confirmWritingFeedback() → orchestrate()

6. (2–5 반복)

7. 모든 문항 완료 → celebrate → completeModule
```

### 1-6. PWR 세부 흐름 (Orchestrator 미사용)

```
Step 1: Outlining
  → Topic Sentence / Supporting Details × 3 / Concluding Sentence 입력
  → "다음" 클릭 → Step 2

Step 2: Self-Check
  → INITIAL_DRAFT 표시 + STEP2_FEEDBACK 게시
  → textarea 수정 → "다음" → Step 3

Step 3: 1st Draft
  → 이전 입력 유지 + STEP3_FEEDBACK 게시
  → textarea 수정 → "다음" → Step 4

Step 4: Final Draft
  → MODEL_ANSWER + STEP4_FEEDBACK(총평)
  → "완료" → moduleResult 설정
```

---

## 2. IA 구조 및 기능 정의

### 2-1. 페이지·컴포넌트 계층

```
ModulesPage (/modules/[lessonId])
  └─ LessonSession
      └─ ModuleRunner key={currentModuleIdx}
          ├─ ProcessWritingFlow          [PWR 전용]
          │   ├─ StepIndicator (1–4)
          │   ├─ ContentPanel
          │   ├─ WritingPanel (Outline폼 / textarea / 최종본)
          │   └─ FeedbackPanel
          └─ ModuleRunnerInner           [PRD·SHR·SWR·기타]
              ├─ ModuleProgressBar       (다중 모듈 레슨 시)
              ├─ [Desktop lg+]
              │   ├─ 좌측: ContentPanel + QuestionsPanel / VoiceQuestionPanel
              │   └─ 우측: FeedbackPanel + AIQuestionPanel
              └─ [Mobile < lg]
                  ├─ 탭: Content / Questions / Voice (조건부 노출)
                  ├─ MobileSplitLayout (드래그 분할선)
                  └─ FeedbackPanel + AIQuestionPanel (항상 표시)
```

### 2-2. 컴포넌트별 목적·입력·출력

| 컴포넌트 | 목적 | 주요 입력 | 주요 출력/이벤트 |
|---|---|---|---|
| `LessonSession` | 모듈 시퀀스 실행·결과 누적 | `lessonPlan` | `onModuleComplete`, `isLessonComplete` |
| `ModuleRunner` | 어댑터 선택·데이터 로드 | `plannedModule` | `ModuleRunnerInner` or `ProcessWritingFlow` |
| `ModuleRunnerInner` | Orchestrator 결과 → UI 상태 | `useModuleOrchestrator` 반환값 | 지문/문항/채팅 렌더링 |
| `ContentPanel` | 지문 표시·강조 | `passage`, `highlights`, `previewOnly` | (없음) |
| `QuestionsPanel` | 문항 제시·답변 수집 | `questions`, `activeQuestionId`, `answers` | `onSubmitAnswer` |
| `VoiceQuestionPanel` | 음성 문항 UI | `isPlayingAudio`, `isRecording`, `pronunciationScores` | `onSubmitVoice`, `onReplayAudio`, `onPlayback` |
| `FeedbackPanel` | AI 채팅·피드백 확인 | `messages`, `awaitingXxxConfirm`, `moduleResult` | `onConfirm*`, `onRetry`, `submitUserMessage` |
| `AIQuestionPanel` | 자유 질문 입력 | (없음) | `onSubmitUserMessage` |
| `ModuleProgressBar` | 모듈 순서 시각화 | `moduleSequence`, `currentIdx` | (없음) |
| `ModuleCompleteCard` | 성과카드 표시 | `moduleResult` | (없음) |
| `TypewriterText` | 마크다운·타자기 렌더링 | `text`, `animate` | (없음) |
| `MobileSplitLayout` | 드래그 분할 레이아웃 | `topContent`, `bottomContent` | (없음) |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. Orchestrator 결정 규칙 우선순위

```
Rule 1:     ruleIdleCheck              — 120초+ 무응답 → checkEngagement
Rule 2:     ruleDisengaged             — 180초 + engagementLevel='low' → signalReplan
Rule 3:     ruleRepeatWrongAnswer      — 오답 2회+ (essay·audio-record·sentence-write 제외) → giveHint
Rule 4a:    ruleHolisticFeedback       — essay 답변 직후 + 피드백 미발화 → giveFeedback(type:'holistic')
Rule 4b:    ruleRevealFullPassage      — 미리보기 + 전체 답변 + 피드백 완료 → showPassage({})
Rule SHR-C: rulePronunciationFeedback  — audio-record 답변 직후 + 미발화 → giveFeedback(type:'pronunciation')
Rule SWR-A: ruleWritingFeedback        — sentence-write 답변 직후 + 미발화 → giveFeedback(type:'writing') ※ ruleCorrectAnswer 앞
Rule 4d:    ruleWrongAnswerFeedback    — 객관식 첫 오답 → giveFeedback(type:'correctness', correct:false)
Rule 5:     ruleCompleteAfterCelebrate — celebrate 후 → completeModule or signalReplan ※ ruleCorrectAnswer 앞
Rule SHR-A: rulePlayModelAudio         — 미재생 audio-record 문항 존재 → playModelAudio ※ ruleCorrectAnswer 앞
Rule SHR-B: ruleStartRecordingAfterAudio — 오디오 재생 완료 후 → startRecording ※ ruleCorrectAnswer 앞
Rule 6:     ruleCorrectAnswer          — 정답 → 다음 askQuestion or celebrate
Rule 7:     ruleInitialEntry           — 첫 진입 → showPassage (문항 유형으로 분기)
Rule 8:     rulePresentNextQuestion    — 기본 흐름 → 미답변 문항 제시
```

> **순서 주의 1**: `ruleCompleteAfterCelebrate`는 반드시 `ruleCorrectAnswer` **앞** — 그렇지 않으면 celebrate 무한 반복.
> **순서 주의 2**: `rulePlayModelAudio` / `ruleStartRecordingAfterAudio`는 반드시 `ruleCorrectAnswer` **앞** — 피드백 확인 후 다음 오디오 자동 재생 보장.
> **순서 주의 3**: `ruleWritingFeedback`은 반드시 `ruleCorrectAnswer` **앞** — 피드백 없이 다음 문항 진행 방지.

### 3-2. ruleInitialEntry 분기 로직

```
essay / short-text  →  showPassage({ previewOnly: true, greeting })
                        hook: greeting 게시 → 400ms 후 silent orchestrate

sentence-write      →  showPassage({ autoAdvance: true, greeting })
  (SWR)               hook: greeting 게시 → 800ms 후 silent orchestrate

audio-record        →  showPassage({ autoAdvance: true, greeting })
  (SHR)               hook: greeting 게시 → 800ms 후 silent orchestrate

그 외 (객관식 등)    →  showPassage({ autoAdvance: true, greeting })
                        hook: greeting 게시 → 800ms 후 silent orchestrate
```

**greeting 제어 정책** (2026-03-30 변경):
- `greeting`은 `showPassage.params.greeting`으로 전달된다.
- `ruleInitialEntry`가 `context.moduleConfig.stage1Greeting`을 읽어 `greeting`으로 패키징 후 Tool에 포함.
- hook은 더 이상 `input.moduleConfig.stage1Greeting`을 직접 읽지 않는다.
- 향후 LLM Orchestrator 전환 시 `ruleInitialEntry`의 TODO 라인만 교체하면 동적 생성 가능.
- `stage7CompleteMsg`는 `completeModule` Tool이 직접 요약 메시지를 포함하므로 ContentConfig에서 읽지 않음 (데드코드).

### 3-3. giveFeedback 타입 분기 정책 (2026-03-30 변경)

`givePronunciationFeedback`·`giveWritingFeedback` Tool은 `giveFeedback`으로 통합되었다.
`params.type`으로 피드백 종류를 구분한다.

| params.type | 발화 조건 | 처리 로직 |
|---|---|---|
| `correctness` | 객관식 오답 | `buildFeedbackMessage` → `awaitingFeedbackConfirm=true` |
| `holistic` | essay 답변 직후 | `buildFeedbackMessage` → `awaitingFeedbackConfirm=true` |
| `pronunciation` | audio-record 답변 직후 | `buildPronunciationFeedbackMessage` + `setPronunciationScores` → `awaitingPronunciationConfirm=qId` |
| `writing` | sentence-write 답변 직후 | `buildWritingFeedbackMessage` → `awaitingWritingConfirm=qId` |

새 피드백 유형 추가 시 `GiveFeedbackParams` 유니온에 variant만 추가. `OrchestratorToolCall` 유니온은 변경 불필요.

### 3-4. ruleCorrectAnswer 조건 (모듈별 정답 판정)

| 문항 타입 | 정답 조건 |
|---|---|
| `multiple-choice` / `short-text` | `answers[qId] === q.answer` |
| `essay` | `answers[qId]` 존재 (내용 무관) |
| `audio-record` | `answers[qId]` 존재 + `feedbackGivenIds['pronunciation']`에 qId 포함 |
| `sentence-write` | `answers[qId]` 존재 + `feedbackGivenIds['writing']`에 qId 포함 |

### 3-5. ruleCompleteAfterCelebrate 분기

```
오답률 ≥ 50%  → signalReplan (레슨 플랜 재조정 요청)
오답률 < 50%  → completeModule (정상 완료)
```

### 3-6. FeedbackPanel 버튼 표시 조건

| 버튼 | 조건 |
|---|---|
| "다음" (holistic/correctness 피드백 확인) | `awaitingFeedbackConfirm && !isLoading` |
| "다시 시도 🎤" / "다음 문항으로 →" (발음) | `awaitingPronunciationConfirm !== null && !isLoading` |
| "다음 문장으로 →" (영작) | `awaitingWritingConfirm !== null && !isLoading` |
| "다음 모듈로 →" / "홈으로" | `moduleResult !== null` |

### 3-7. VoiceQuestionPanel 버튼 표시 조건

| 버튼 | 조건 | 동작 |
|---|---|---|
| 듣기 | `isActive || isAnswered` | `replayModelAudio()` (orchestrate 우회) |
| 말하기 / 말하기 완료 | `isRecording || isAnswered` | `submitVoiceAnswer()` |
| 녹음듣기 | `isAnswered` | `playbackRecording()` (2초 mock) |

### 3-8. 모바일 탭 표시 조건

```
'content'   → passageVisible
'questions' → !isVoiceModule && (activeQuestionId || 답변 1개 이상)
'voice'     → isVoiceModule && (activeQuestionId || 답변 1개 이상)
```

---

## 4. 변경된 내용 및 추가 개발 필요 항목

### 4-1. 2026-03-30 변경 사항 (개발자 대응 필요)

#### ① OrchestratorContext — 4개 필드 → 2개 범용 맵 교체

**구 타입 (삭제됨)**:
```typescript
// agent-types.ts — 더 이상 존재하지 않음
playedQuestionIds: string[];
recordingPromptedIds: string[];
pronunciationFeedbackGivenIds: string[];
writingFeedbackGivenIds: string[];
```

**신 타입**:
```typescript
eventFiredIds: Record<string, string[]>;
// 현재 key: 'modelAudioPlayed', 'recordingPrompted'

feedbackGivenIds: Record<string, string[]>;
// 현재 key: 'pronunciation', 'writing'
```

**대응**: 새 모듈 추가 시 `eventFiredIds`/`feedbackGivenIds`에 새 key를 자유롭게 추가.
`OrchestratorContext` 타입 자체는 변경 불필요.

#### ② OrchestratorToolCall — giveFeedback 통합

**구 Tool (삭제됨)**:
- `givePronunciationFeedback` — `ShadowReadingAdapter` 등에서 직접 참조하는 경우 수정 필요
- `giveWritingFeedback` — 동상

**신 Tool**: `giveFeedback` with `params.type: 'correctness' | 'holistic' | 'pronunciation' | 'writing'`

**대응**:
- `ModuleOrchestratorAgent`의 `rulePronunciationFeedback` / `ruleWritingFeedback`은 이미 수정 완료.
- messageBuilders의 `buildPronunciationFeedbackMessage` 파라미터 타입이 `PronunciationFeedbackParams`로 변경됨.
  - 구: `accuracyPoints?: string[]` → 신: `pronunciationPoints?: string[]`
- messageBuilders의 `buildWritingFeedbackMessage` 파라미터 타입이 `WritingFeedbackParams`로 변경됨.

#### ③ showPassage.params.greeting 추가

**대응**:
- `ruleInitialEntry`에서 `context.moduleConfig.stage1Greeting`을 greeting으로 넘기는 로직 구현 완료.
- hook(`useModuleOrchestrator`)에서 `input.moduleConfig.stage1Greeting` 직접 참조 코드 제거 완료.
- **LLM Orchestrator 연동 시**: `ruleInitialEntry`의 TODO 라인(`context.moduleConfig.stage1Greeting`)을 `PedagogyProfile.purpose + passage.title` 기반 LLM 생성으로 교체하면 됨. 나머지 코드 수정 불필요.

#### ④ PRD buildContentConfig 정리

- stage2–stage6 관련 Items가 정리되어 Orchestrator 기반 모듈의 ContentConfig 최소 구성이 명확해짐.
- `stage7CompleteMsg`는 ContentConfig에 필드가 있지만 hook·Orchestrator 어디서도 읽지 않음. 데드코드.
  - **개발 작업 필요**: ContentConfig 타입에서 `stage7CompleteMsg` 제거 고려 (단, 백오피스 연동 후 결정).

---

## 5. 코딩 규칙 (Coding Rules)

### 5-1. 반드시 사용해야 하는 패턴/유틸

| 상황 | 사용 패턴 | 근거 |
|---|---|---|
| 새 모듈 추가 | `ModuleAdapter` 인터페이스 구현 + `index.ts` 레지스트리 등록 | 어댑터 패턴 — 모듈별 캡슐화 |
| 피드백 메시지 생성 | `messageBuilders.ts` 함수 사용 | LLM 교체 시 함수 단위로 교체하기 위함 |
| 새 피드백 타입 추가 | `GiveFeedbackParams` 유니온에 variant 추가 | `OrchestratorToolCall` 유니온 불변 유지 |
| 이벤트 추적 | `eventFiredIds[key]` / `feedbackGivenIds[key]` | 새 key를 임의 추가하면 됨, 타입 변경 불필요 |
| 마크다운 메시지 렌더링 | `TypewriterText` + `renderMarkdown()` | `**bold**`, `\n` 처리 포함 |
| 채팅 메시지 발행 | `postAiMessage(text)` 내부 함수 | id 생성·state 일관성 보장 |
| stale closure 방지 | 비동기 콜백 내에서는 항상 `xxxRef.current` 사용 | orchestrate·answers 등 최신값 보장 |

### 5-2. 금지 패턴

| 금지 | 이유 | 대안 |
|---|---|---|
| `any` 타입 사용 | TypeScript 타입 안정성 파괴 | `GiveFeedbackParams` 같은 discriminated union 사용 |
| hook 내부에서 `input.moduleConfig.stage1Greeting` 직접 읽기 | 제어권이 Orchestrator에 있음 | `toolCall.params.greeting` 사용 |
| `givePronunciationFeedback` / `giveWritingFeedback` Tool 이름 사용 | 삭제된 Tool — 타입 오류 발생 | `giveFeedback` + `type` 필드 사용 |
| `context.playedQuestionIds` 등 구 필드 참조 | 삭제됨 | `context.eventFiredIds['modelAudioPlayed']` 사용 |
| `alert()` 직접 사용 | UX 파괴, 테스트 불가 | `postAiMessage()` 또는 토스트 컴포넌트 |
| 새 모듈에서 `useModuleOrchestrator` 우회하는 전용 상태 관리 | 두 번째 step-workflow 모듈 추가 전까지 YAGNI | PWR처럼 불가피한 경우에만 전용 Flow 컴포넌트 허용 |
| OrchestratorContext에 모듈 전용 필드 직접 추가 | 유니온 확장 필요 | `eventFiredIds` / `feedbackGivenIds`의 새 key로 해결 |

### 5-3. 파일 위치 규칙

| 종류 | 위치 |
|---|---|
| 도메인 타입 (PassageData, QuestionData, ModuleData) | `src/lib/types/module.ts` |
| UI 배치 타입 (ContentConfig, PanelAssignment) | `src/lib/types/ai-module.ts` |
| Agent 공유 타입 (OrchestratorContext, LessonPlan, Tool Calls) | `src/lib/agents/agent-types.ts` |
| 모듈별 어댑터 | `src/lib/adapters/[모듈코드]Adapter.ts` |
| 정적 메시지 생성 함수 | `src/lib/hooks/messageBuilders.ts` |
| 페이지 UI 컴포넌트 | `src/app/modules/[lessonId]/_components/panels.tsx` |
| 전용 흐름 UI (PWR처럼 불가피한 경우) | `src/app/modules/[lessonId]/_components/[모듈코드]Flow.tsx` |

### 5-4. 네이밍 규칙

| 종류 | 규칙 | 예시 |
|---|---|---|
| 어댑터 클래스 | `[한글스킬명]Adapter` (PascalCase) | `PredictionAdapter`, `ShadowReadingAdapter` |
| 어댑터 코드 (moduleCode) | 3자리 대문자 | `PRD`, `SHR`, `SWR`, `PWR` |
| Tool 이름 | camelCase 동사+명사 | `giveFeedback`, `showPassage`, `completeModule` |
| Orchestrator Rule 함수 | `rule[기능명]` (camelCase) | `ruleInitialEntry`, `ruleHolisticFeedback` |
| messageBuilder 함수 | `build[대응Tool]Message` | `buildFeedbackMessage`, `buildCelebrateMessage` |
| 피드백 타입 인터페이스 | `[타입명]FeedbackParams` | `PronunciationFeedbackParams`, `WritingFeedbackParams` |
| eventFiredIds key | camelCase 이벤트명 | `'modelAudioPlayed'`, `'recordingPrompted'` |
| feedbackGivenIds key | 피드백 type 값과 동일 | `'pronunciation'`, `'writing'` |

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

### 6-1. Mock 데이터 목록 (API 교체 필요)

| 파일 | 내용 | TODO 엔드포인트 |
|---|---|---|
| `lessonService.ts` | `fetchLessonPlan()` 전체 Mock (L008 / 기타 lessonId) | `GET /api/lesson/{lessonId}/plan` |
| `PredictionAdapter.ts` | `MOCK_PREDICTION_DATA` — Technology and Society 지문 + essay 1문항 | `GET /api/lessons/:id/module-data` |
| `ShadowReadingAdapter.ts` | `MOCK_SHR_DATA` — 5개 문장 audio-record | 동상 |
| `SentenceWritingAdapter.ts` | `MOCK_SWR_DATA` — 4개 sentence-write 문항 | 동상 |
| `ProcessWritingAdapter.ts` | `MOCK_PWR_DATA` — 에세이 지문 + 아웃라인 예제 | 동상 |
| `LessonSession.tsx` | `saveModuleHistory()` 비동기 Mock (실패 무시) | `POST /api/lessons/:lessonId/modules/:moduleCode/history` |

### 6-2. Mock 계산 (LLM 교체 필요)

| 파일 | 내용 | 목표 |
|---|---|---|
| `ModuleOrchestratorAgent.ts` | `decideNextAction()` — Rule-based Mock | `POST /api/agent/orchestrator/decide` (Claude Haiku 4.5) |
| `ModuleOrchestratorAgent.ts` | 영작 점수: 단어 수 비율 계산 | `POST /api/agent/writing-eval` |
| `ModuleOrchestratorAgent.ts` | 발음 점수 피드백: 점수 구간별 정적 문자열 | `POST /api/agent/pronunciation-eval` |
| `messageBuilders.ts` | `buildHintMessage()` — 문항 텍스트 앞 4단어 기반 정적 메시지 | `POST /api/agent/chat` |
| `messageBuilders.ts` | `buildFeedbackMessage()` — 답안 길이 기반 정적 메시지 | `POST /api/agent/feedback` |
| `messageBuilders.ts` | `buildExplainMessage()` — 고정 안내 메시지 | `POST /api/agent/chat` |
| `PredictionAdapter.ts` | KPI: 답안 길이 → 60–90% proxy | `POST /api/agent/kpi/prediction` |
| `SentenceWritingAdapter.ts` | KPI: 단어 수·패턴 감지 Mock | `POST /api/agent/kpi/swr` |
| `ProcessWritingAdapter.ts` | KPI: 문장 수·어휘 변화율 Mock | `POST /api/agent/kpi/pwr` |
| `ProcessWritingFlow.tsx` | 단계별 피드백: 정적 텍스트 상수 | `POST /api/agent/feedback/pwr` |

### 6-3. Mock 동작 (실제 기능으로 교체 필요)

| 파일 | 내용 |
|---|---|
| `useModuleOrchestrator.ts` | `submitVoiceAnswer()` — 발음 점수 70–100 랜덤 생성 (실제: STT + 발음 평가 API) |
| `useModuleOrchestrator.ts` | `playModelAudio()` — 2500ms 딜레이 시뮬레이션 (실제: TTS 엔진) |
| `useModuleOrchestrator.ts` | `playbackRecording()` — 2000ms 애니메이션 (실제: 녹음 파일 재생) |
| `useModuleOrchestrator.ts` | 이전 모듈 결과 `score: 0`, `kpis: []` 하드코딩 |

### 6-4. 데드코드 / 미사용 파일

| 파일 | 상태 |
|---|---|
| `src/lib/hooks/useStageStateMachine.ts` | 현재 사용되지 않음. Orchestrator로 완전 대체. |
| `src/lib/hooks/usePanelRouter.ts` | 현재 사용되지 않음. `MOCK_CONTENT_CONFIG` 포함. |
| `ContentConfig.stage7CompleteMsg` | hook·Orchestrator 어디서도 읽히지 않음. `completeModule` Tool이 직접 메시지 처리. |

### 6-5. 미구현 검증 / 알려진 제약

| 항목 | 내용 |
|---|---|
| `ruleRevealFullPassageAfterFeedback` | 피드백 발화 확인을 `recentMessages`의 텍스트 스캔으로 판단 (Q번호 포함 여부) — 메시지 형식 변경 시 오동작 가능 |
| PRD `stage1Greeting` | 현재 `context.moduleConfig.stage1Greeting`에서 읽음. 백오피스 연동 전까지는 어댑터의 `buildContentConfig`에 하드코딩 |
| 유휴 타이머 | `orchestrate()` 호출 후에도 타이머 계속 진행. 60/120초 임계값 각 1회만 발화 (ref로 방지) |
| PWR `flowType` | 현재 `moduleData.code === 'PWR'`으로 하드코딩 분기. 두 번째 step-workflow 모듈 추가 시 `ModuleAdapter.flowType` 필드로 일반화 필요 |

---

## 7. 컴포넌트·훅 의존성 (Dependencies)

### 7-1. 이 페이지가 사용하는 공통 컴포넌트·훅·상수

```
useModuleOrchestrator
  ├─ ModuleOrchestratorAgent (createModuleOrchestratorAgent)
  ├─ messageBuilders (buildHintMessage, buildFeedbackMessage, ...)
  └─ agent-types (OrchestratorContext, OrchestratorToolCall, GiveFeedbackParams, ...)

ModuleRunner
  ├─ adapters/index.ts (getAdapter)
  └─ 각 ModuleAdapter (fetchModuleData, buildContentConfig, buildKpis, calculateScore)

panels.tsx
  └─ TypewriterText (renderMarkdown 내장)

LessonSession
  └─ lessonService.ts (fetchLessonPlan, saveModuleHistory)
```

### 7-2. 진입점 (이 페이지로 오는 경로)

| 경로 | 설명 |
|---|---|
| `/modules/[lessonId]` — 직접 URL | 선생님이 구성한 레슨 진입 |
| `/class/lesson-setup/custom` → redirect | `CurriculumPlannerAgent`가 생성한 AI 레슨 완료 후 redirect |
| 개발/테스트 | `lessonService.ts`의 Mock lessonId (L008 등) 직접 입력 |

### 7-3. 이 페이지가 영향을 주는 다른 기능

| 기능 | 영향 |
|---|---|
| 학습 이력 (미구현) | `saveModuleHistory()` 호출 → DB 저장 대상 |
| 성과 대시보드 (미구현) | `ModuleResult.kpis` 데이터 공급원 |
| 레슨 플랜 재조정 | `signalReplan` 발화 → `CurriculumPlannerAgent` 재호출 대상 |
| 백오피스 모듈 설정 | `ContentConfig`의 `stage1Greeting`, `hintButtonShow` 등을 백오피스에서 관리 (미연동) |

### 7-4. 모듈 추가 시 건드려야 하는 파일 목록

새 모듈(예: `QAR`) 추가 시 필수 작업:

| 파일 | 작업 |
|---|---|
| `src/lib/adapters/[QAR]Adapter.ts` | **신규 생성** — `ModuleAdapter` 구현 |
| `src/lib/adapters/index.ts` | 레지스트리에 `QAR: new QARAdapter()` 추가 |
| `src/lib/agents/orchestrator/ModuleOrchestratorAgent.ts` | 새 문항 타입 또는 피드백 규칙 필요 시 Rule 추가 |
| `src/lib/agents/agent-types.ts` | 새 피드백 타입 필요 시 `GiveFeedbackParams`에 variant 추가 |
| `src/lib/hooks/messageBuilders.ts` | 새 피드백 메시지 빌더 추가 |

---

## 8. DB/API 구조 (Data Contract)

### 8-1. 현재 또는 목표 API 엔드포인트

| 메서드 | 엔드포인트 | 상태 | 설명 |
|---|---|---|---|
| GET | `/api/lesson/{lessonId}/plan` | Mock | 레슨 플랜 조회 (`LessonPlan` 반환) |
| GET | `/api/lessons/{lessonId}/module-data?moduleCode=PRD` | Mock | 모듈 콘텐츠 조회 (`ModuleData` 반환) |
| POST | `/api/lessons/{lessonId}/modules/{moduleCode}/history` | Mock | 채팅 히스토리 저장 |
| POST | `/api/agent/orchestrator/decide` | Mock (Rule-based) | Orchestrator 결정 (Claude Haiku 4.5) |
| POST | `/api/agent/chat` | Mock | 힌트·개념 설명 응답 (Claude Sonnet 4.6) |
| POST | `/api/agent/feedback` | Mock | 홀리스틱/정오답 피드백 생성 |
| POST | `/api/agent/writing-eval` | Mock | 영작 점수 계산 (accuracyScore, complexityScore, grammarErrorRate) |
| POST | `/api/agent/pronunciation-eval` | Mock | 발음 점수 + 피드백 포인트 생성 |
| POST | `/api/agent/kpi/prediction` | Mock | PRD KPI 평가 (예측 타당성) |
| POST | `/api/agent/kpi/swr` | Mock | SWR KPI 평가 (정확성·복잡도·문법) |
| POST | `/api/agent/kpi/pwr` | Mock | PWR KPI 평가 (에세이 구조·자기 교정률) |
| POST | `/api/agent/feedback/pwr` | Mock | PWR 단계별 피드백 생성 |

### 8-2. 핵심 타입 정의

```typescript
// ── 도메인 타입 (module.ts) ──────────────────────────────────────────────

interface PassageData {
  title: string;
  content: string;
}

interface QuestionData {
  id: string;
  number: number;
  type: 'multiple-choice' | 'essay' | 'short-text' | 'audio-record' | 'sentence-write';
  text: string;
  options?: string[];
  answer: string;          // essay·audio-record: 빈 문자열
  hint?: string;           // sentence-write: 힌트 텍스트
}

interface ModuleData {
  lessonId: string;
  code: string;            // 'PRD' | 'SHR' | 'SWR' | 'PWR'
  skill: string;           // 'reading' | 'speaking' | 'writing'
  round: number;
  title: string;
  passage: PassageData;
  questions: QuestionData[];
}

// ── UI 배치 타입 (ai-module.ts) ──────────────────────────────────────────

interface ContentConfig {
  stagePrepEnabled: boolean;
  contentSelection: string;
  learningType: string;
  delayTime: number;
  hintButtonShow: boolean;
  stage1Enabled: boolean;
  stage1Greeting: string;  // Orchestrator가 showPassage.params.greeting으로 전달. hook에서 직접 읽지 않음.
  stage1Items: StageItem[];
  // ... stage2–stage6 (Orchestrator 기반 모듈에서는 모두 false)
  stage7Enabled: boolean;
  stage7CompleteMsg: string; // 현재 미사용 — completeModule Tool이 직접 처리
  stage7Items: StageItem[];
}

// ── Agent 공유 타입 (agent-types.ts) ────────────────────────────────────

interface OrchestratorContext {
  lessonPlan: LessonPlan;
  currentPlannedModule: PlannedModule;
  moduleConfig: ContentConfig;
  passage: PassageData;
  questions: QuestionData[];
  learnerState: LearnerState;
  recentMessages: ChatMessage[];    // 최대 10개
  passageShown: boolean;
  passagePreviewOnly: boolean;
  eventFiredIds: Record<string, string[]>;
  // 현재 key: 'modelAudioPlayed' | 'recordingPrompted'
  feedbackGivenIds: Record<string, string[]>;
  // 현재 key: 'pronunciation' | 'writing'
}

type FeedbackBase = { questionId: string; elaborate: boolean };

type CorrectnessFeedbackParams = FeedbackBase & { type: 'correctness'; correct: boolean };
type HolisticFeedbackParams    = FeedbackBase & { type: 'holistic' };
type PronunciationFeedbackParams = FeedbackBase & {
  type: 'pronunciation';
  score: number;
  fluencyPoints?: string[];
  pronunciationPoints?: string[];  // 구 accuracyPoints → pronunciationPoints (2026-03-30 변경)
};
type WritingFeedbackParams = FeedbackBase & {
  type: 'writing';
  accuracyScore: number;
  complexityScore: number;
  grammarErrorRate: number;
  accuracyPoints?: string[];
  complexityPoints?: string[];
  grammarPoints?: string[];
};
type GiveFeedbackParams =
  | CorrectnessFeedbackParams
  | HolisticFeedbackParams
  | PronunciationFeedbackParams
  | WritingFeedbackParams;

// showPassage Tool — greeting 추가 (2026-03-30)
type ShowPassageParams = {
  highlightParagraphs?: number[];
  focusVocab?: string[];
  previewOnly?: boolean;
  autoAdvance?: boolean;
  greeting?: string;       // Orchestrator가 생성해서 전달. 미제공 시 메시지 생략.
};

interface ModuleResult {
  moduleCode: string;
  score: number;           // 0–100
  kpis: KpiResult[];
  engagementLevel: 'high' | 'medium' | 'low';
  completedAt: string;
  timeSpentMinutes: number;
}

interface KpiResult {
  label: string;
  value: number;
  unit: string;            // '%' | 'WPM' | '점'
  description: string;
}
```

### 8-3. OrchestratorToolCall 전체 목록

| Tool | 파라미터 | 비고 |
|---|---|---|
| `showPassage` | `previewOnly?`, `autoAdvance?`, `highlightParagraphs?`, `focusVocab?`, `greeting?` | greeting 추가 (2026-03-30) |
| `askQuestion` | `questionId`, `guidance?`, `autoAdvance?` | |
| `giveHint` | `questionId`, `level: 'subtle'|'direct'|'explicit'` | |
| `explainConcept` | `term`, `inContext` | |
| `giveFeedback` | `GiveFeedbackParams` (type으로 분기) | givePronunciationFeedback·giveWritingFeedback 통합 (2026-03-30) |
| `suggestReread` | `paragraphIndex`, `guidance` | |
| `celebrate` | `score`, `message?` | |
| `checkEngagement` | `message` | |
| `completeModule` | `score`, `summary` | |
| `playModelAudio` | `questionId`, `text` | SHR |
| `startRecording` | `questionId`, `prompt?` | SHR |
| `signalReplan` | `reason`, `suggestion?` | |

### 8-4. ModuleAdapter 인터페이스

```typescript
interface ModuleAdapter {
  readonly code: string;
  fetchModuleData(lessonId: string): Promise<ModuleData>;
  buildContentConfig(data: ModuleData): ContentConfig;
  buildPedagogyProfile(): ModulePedagogyProfile;
  buildAgentPrompt(params: AgentPromptParams): AgentPrompt;
  calculateScore(questions: QuestionData[], answers: Record<string, string>): ScoreResult;
  buildKpis(
    questions: QuestionData[],
    answers: Record<string, string>,
    pronunciationScores?: Record<string, number>,
  ): KpiResult[];
}
```

### 8-5. 모듈별 교수법 특성

| 코드 | scoringMode | answerType | passageExposureMode | questionCount | feedbackStyle |
|---|---|---|---|---|---|
| PRD | holistic | essay | preview | single | strengths-weaknesses |
| SHR | pronunciation | audio-record | full | multi | strengths-weaknesses |
| SWR | holistic | sentence-write | full | multi | strengths-weaknesses |
| PWR | holistic | essay | full | multi | strengths-weaknesses |

### 8-6. Mock 레슨 ID (lessonService.ts)

| lessonId | 모듈 시퀀스 | 지문 |
|---|---|---|
| `L008` | PWR × 1 | tesla-process-writing |
| 기타 | SWR → PRD → SHR | technology-and-society |

---

## 9. 주요 버그 수정 이력

| 버그 | 원인 | 수정 |
|---|---|---|
| celebrate 무한 반복 | `ruleCompleteAfterCelebrate`가 `ruleCorrectAnswer` 뒤 | 규칙 순서 변경 |
| 오답 피드백 후 진행 불가 | `giveFeedback(correct:false)` 후 orchestrate 미호출 | 1500ms 후 자동 재호출 |
| 전체 피드백 후 전체 지문 미공개 | `giveFeedback(elaborate:true)` 후 orchestrate 미호출 | 0ms 후 즉시 재호출 |
| AI 피드백이 열렸다 닫혔다 | TypewriterText: `animate→false` 시 partial text 유지 | `setDisplayed(text)` 즉시 호출 |
| celebrate stale closure | `executeToolCall`에서 `orchestrate` 직접 참조 | `orchestrateRef.current()` 사용 |
| 모듈 완료 시 채팅 히스토리 사라짐 | `isCompleted` early return으로 FeedbackPanel 제거 | early return 제거, 왼쪽 패널만 교체 |
| FeedbackPanel 리프레시 (체이닝 중) | 자동 체이닝 시에도 `setIsLoading(true)` 호출 | `orchestrate(silent=true)` 패턴 도입 |
| SHR 진입 시 AI 메시지 없음 | `passageExposed:true` 분기 → `showPassage({})` 반환 → 체이닝 안 됨 | `passageExposed` 분기 제거, 문항 유형으로만 결정 |
| PRD 진입 시 AI 메시지 없음 | `showPassage(previewOnly)` 체이닝 시 greeting 미게시 | `showPassage.params.greeting` 추가 (2026-03-30) |
| page.tsx 1,246줄 비대화 | 모든 컴포넌트가 단일 파일 | 2026-03-26: `_components/` 분리 |
| SHR 발음 피드백 미발화 | `alreadyFeedback` 체크가 `startRecording` 안내 메시지와 충돌 | `feedbackGivenIds['pronunciation']` ref로 대체 |
| 재시도 후 피드백 재발화 안됨 | `retryVoiceAnswer` 시 이전 피드백이 messages에 남아 중복 판단 | ref 방식 + 재시도 시 ref 초기화 |
| 다음 문항 오디오 버튼 미표시 | `rulePlayModelAudio`가 `ruleCorrectAnswer` 뒤 | SHR-A/B를 `ruleCorrectAnswer` 앞으로 이동 |
| 마크다운 미렌더링 | `TypewriterText`가 `<span>{text}</span>` only | `renderMarkdown()` 추가 |
| `ModuleResult.kpis` 누락 | `replanSignal` 경로에서 누락 | `kpis: buildKpis(answers, {})` 추가 |
