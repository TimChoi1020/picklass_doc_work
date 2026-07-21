# QuestionPanelFooter 분리 및 VoiceQuestionPanel 버그 수정

## 변경 사항 요약

### 1. QuestionPanelFooter 컴포넌트 분리

`QuestionsPanel`과 `VoiceQuestionPanel`이 각각 독립적으로 "다음문항 / 다시시도" 버튼 로직을 구현해야 하는 구조적 문제를 해결하기 위해 공유 컴포넌트를 추출했다.

**파일**: `apps/web/src/app/modules/[lessonId]/_components/panels.tsx:822`

```
QuestionPanelFooter   ← 버튼 로직 단일 소스
      ↑                      ↑
QuestionsPanel       VoiceQuestionPanel    (FuturePanel)
(답변 입력 UI)        (녹음·재생 UI)         (미래 패널)
```

**Props**:
```ts
{
  isAnswered: boolean;
  isLastQuestion: boolean;
  isLoading: boolean;
  questionMaxAttempts?: number | null;
  usedAttempts: number;
  questionId: string;
  onNextQuestion?: () => void;
  onRetryAnswer?: (questionId: string) => void;
}
```

**렌더 조건**: `isAnswered && !isLastQuestion` 일 때만 렌더. 마지막 문항 구간의 "이해했어요"는 `FeedbackPanel`이 담당.

**적용 위치**:
- `QuestionsPanel` deck 모드 하단 (`panels.tsx:1010`)
- `QuestionsPanel` sequential 모드 하단 (`panels.tsx:1267`) ※ 아래 주의사항 참고
- `VoiceQuestionPanel` deck 모드 하단 (`panels.tsx:1428`)
- `VoiceQuestionPanel` sequential 모드 하단 (`panels.tsx:1576`)

> **⚠️ `QuestionsPanel` sequential 모드 주의**
> `QuestionPanelFooter`는 코드 상 존재하지만(`panels.tsx:1267`), 실제로는 렌더되지 않는다.
>
> **원인**: sequential 모드에서는 사용자가 답변을 제출하는 즉시 오케스트레이터가
> `activeQuestionId`를 다음 문항으로 전진시킨다. 이 때문에
> `isAnswered(= !!activeQuestionId && answers[activeQuestionId] !== undefined)`가
> `true`인 구간이 사실상 존재하지 않아 `QuestionPanelFooter`는 항상 `null`을 반환한다.
>
> **의도된 동작**: sequential 모드에서 `QuestionsPanel`의 흐름은 오케스트레이터(AI)가
> 자동으로 제어하므로 학습자가 수동으로 "다음문항" 버튼을 누를 필요가 없다.
>
> **`VoiceQuestionPanel` sequential과의 차이**: 음성 패널에서는 `activeRecordingId`가
> 학습자가 직접 "다음문항"을 클릭할 때까지 유지되므로 버튼이 렌더되는 구간이 생긴다.

---

### 2. LessonSession VoiceQuestionPanel props 누락 수정

**파일**: `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx:808`

`VoiceQuestionPanel` 호출부에 `QuestionPanelFooter` 동작에 필요한 props 5개가 누락되어 있었다.

```tsx
// 추가된 props
isLastQuestion={isLastQuestion}
questionMaxAttempts={pedagogyProfile.questionMaxAttempts}
attemptCounts={attemptCounts}
onNextQuestion={confirmFeedback}
onRetryAnswer={retryVoiceAnswer}
```

---

### 3. VoiceQuestionPanel "준비 중이에요..." 오버레이 버그 수정

**파일**: `apps/web/src/app/modules/[lessonId]/_components/panels.tsx`

**증상**: 한 문장을 완료하면 이미 답변된 문항이 있어도 "Pickle AI가 문항을 준비 중이에요..." 화면으로 덮였다.

**원인**: 문항 완료 후 오케스트레이터가 `activeRecordingId`를 `null`로 잠시 리셋하는 시점에, "아직 시작 안 된 초기 상태"와 "완료 후 다음 문항 대기 상태"를 구분하지 않고 동일하게 처리했다.

**수정 (deck 모드, `panels.tsx:1339`)**:
```tsx
// 수정 전
const activeQ = activeRecordingId
  ? (questions.find((q) => q.id === activeRecordingId) ?? questions[questions.length - 1])
  : null;

// 수정 후 — 이미 답변이 있으면 마지막 카드 유지
const activeQ = activeRecordingId
  ? (questions.find((q) => q.id === activeRecordingId) ?? questions[questions.length - 1])
  : doneCount > 0 ? questions[questions.length - 1] : null;
```

**수정 (sequential 모드, `panels.tsx:1443`)**:
```tsx
// 수정 전
if (!activeRecordingId) return (준비중...)

// 수정 후 — 아직 시작 전일 때만 준비 중 표시
if (!activeRecordingId && doneCount === 0) return (준비중...)
```

---

## 테스트 항목

- [ ] `VoiceQuestionPanel` (deck 모드): 문장 완료 후 다음 문항 대기 중 마지막 카드가 유지되는가
- [ ] `VoiceQuestionPanel` (sequential 모드): 문장 완료 후 목록이 유지되는가
- [ ] `VoiceQuestionPanel` 초기 진입 시 "준비 중이에요..." 가 정상 표시되는가
- [ ] `QuestionPanelFooter` "다음문항" 버튼이 양쪽 패널 모두에서 동작하는가
- [ ] `QuestionPanelFooter` "다시시도" 버튼이 `questionMaxAttempts` 조건에 따라 노출/숨김되는가
- [ ] 마지막 문항(`isLastQuestion=true`)에서 `QuestionPanelFooter`가 렌더되지 않는가
