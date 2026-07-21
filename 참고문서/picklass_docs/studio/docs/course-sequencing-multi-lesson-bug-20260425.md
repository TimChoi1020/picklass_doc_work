# 다중 회차 시퀀싱 적용 시 마지막 회차만 반영되는 버그 — 분석 + 수정 계획

**작성일**: 2026-04-25  
**증상**: Step 2 진입 시 모든 회차에 동일한 KPI + duration이 적용되어 모든 회차가 시퀀싱되어야 하지만, **마지막 회차만 시퀀싱 결과가 반영**됨.

---

## 1. 데이터 흐름 트레이스

```
Step 1: wizardLevel + KPI + wizardDuration 선택
   ↓ "다음" 클릭
runGenerateAITopics:
   - topicsDuration = Array(N).fill(wizardDuration)
   ↓
Step 2 진입, N개 회차 카드 렌더 (각각 LessonModuleSection 인스턴스)
   ↓
각 LessonModuleSection 인스턴스:
   - useLessonPlanSequence({passageLevel: wizardLevel}, kpiCodes, duration) 호출
   - 모든 회차의 input이 동일 → 같은 queryKey → 같은 query 결과
   ↓
React Query 결과 도착 → 모든 LessonModuleSection의 isPlanReady=true
   ↓
각 LessonModuleSection의 useEffect 실행 (거의 동시):
   - lastAppliedSequenceRef 갱신
   - 부모의 onModulesChange(sequence) 호출
   ↓
부모 onModulesChange (course/page.tsx):
   const newModules = [...selectedModulesPerTopic];   // ← 문제 지점
   newModules[idx] = modules;
   setSelectedModulesPerTopic(newModules);
```

---

## 2. 버그 원인 (Race condition + stale closure)

[course/page.tsx](apps/web/src/app/course/page.tsx) 의 `onModulesChange` 핸들러:

```typescript
onModulesChange={(modules) => {
  const newModules = [...selectedModulesPerTopic];   // ← stale state 캡처
  newModules[idx] = modules;
  setSelectedModulesPerTopic(newModules);
}}
```

### 동시 호출 시나리오 (회차 3개)

같은 렌더 사이클 안에서 3개 LessonModuleSection의 `useEffect`가 동시에 실행되면:

| 순서 | 회차 0 | 회차 1 | 회차 2 |
|------|--------|--------|--------|
| 진입 시 캡처 | `[[],[],[]]` | `[[],[],[]]` | `[[],[],[]]` |
| 처리 | `[seq, [], []]` | `[[], seq, []]` | `[[], [], seq]` |
| `setSelectedModulesPerTopic` | 호출 | 호출 | 호출 |

세 콜백 모두 **렌더링 시점에 캡처된 같은 stale state** 를 기반으로 새 배열을 만든다. React가 setState batch를 처리할 때, 마지막 호출(`[[], [], seq]`)이 최종 반영됨.

→ **회차 2(마지막)만 sequence가 남고 회차 0, 1은 빈 배열로 덮어쓰임.**

같은 패턴이 `topicsDuration` 등 다른 회차별 배열 setter에도 잠재적으로 존재.

---

## 3. 해결 방법

### 핵심 원칙

부모의 setState 핸들러를 **functional updater 형태**로 변경하여 prev 값을 항상 최신으로 받도록.

### 수정 패턴

```typescript
onModulesChange={(modules) => {
  setSelectedModulesPerTopic((prev) => {
    const next = [...prev];
    next[idx] = modules;
    return next;
  });
}}

onDurationChange={(d) => {
  setTopicsDuration((prev) => {
    const next = [...prev];
    next[idx] = d;
    return next;
  });
}}
```

이렇게 하면 React가 `prev`를 항상 최신(직전 commit) 값으로 전달 → race 없음.

---

## 4. 수정 범위

### 4-1. course/page.tsx

회차 단위로 `Array<T>` 상태를 변경하는 모든 위치를 functional updater로 변경:

| 핸들러 | 대상 setter |
|--------|------------|
| LessonModuleSection.onModulesChange | `setSelectedModulesPerTopic` |
| LessonModuleSection.onDurationChange | `setTopicsDuration` |
| 회차의 source 변경 (AI/lesson 토글) | `setTopicsSource`, `setSelectedLessonPerTopic` |
| 회차 단어수/장르/주제 입력 | `setTopicsWordCount`, `setTopicsGenres`, `setTopics` |
| 회차의 lesson 선택 (popover) | `setSelectedLessonPerTopic`, `setTopics`, `setTopicsWordCount`, `setTopicsGenres`, `setSelectedModulesPerTopic` |

> **중요**: race가 실제로 발생하는 곳은 시퀀싱 자동 갱신(여러 회차 동시 적용)이지만, **모든 배열 setter에 일관성 있게 적용** 권장.

### 4-2. 검증 포인트

- 회차 3개 이상에서 시퀀싱 결과 → 모든 회차에 sequence 적용
- DevTools Network 탭에서 같은 query 1회만 호출되는 것 확인 (React Query 캐시)
- 회차별 duration 변경 시 다른 회차 영향 없음

---

## 5. 영향 범위 / 사이드 이펙트

- 함수 동작 변경 없음 (functional updater는 동일 로직, race-safe만 추가)
- 백엔드/DB 변경 없음
- ModuleChipList, LessonModuleSection 컴포넌트 자체는 변경 없음

---

## 6. 검증 시나리오 (체크리스트 적용)

| # | 시나리오 | 기대 |
|---|---------|------|
| 1 | Step 1에서 KPI + duration 선택 → Step 2 | 모든 N개 회차 시퀀싱 적용 (모듈 칩에 번호 표시) |
| 2 | 회차 1 duration만 변경 | 회차 1만 재시퀀싱, 회차 2~N 그대로 |
| 3 | 사용자가 모듈 추가 클릭 → 다른 회차에 영향 없음 | 해당 회차 selectedModules에만 추가 |
| 4 | 회차 추가/삭제 (lessonRound 변경) | topicsDuration 등 길이 동기화, 디폴트 적용 |
| 5 | 모듈 수정 모달 (course detail) 진입 | DB 저장 순서 그대로 번호 표시 (이전 수정과 호환) |

dev 서버에서 시나리오 1~3 직접 확인 + Network 탭 호출 횟수 확인.

---

## 7. 작업 순서

1. course/page.tsx의 모든 회차 배열 setter를 functional updater로 변경
2. 빌드 검증 + `.next/` 정리
3. dev 서버에서 시나리오 1~3 직접 확인
4. 보고

---

## 9. 구현 결과 (2026-04-25)

### 9-1. 수정 위치 (13곳 → functional updater)

| 위치 | setter | 호출 컨텍스트 |
|------|--------|-------------|
| AI 토글 | `setTopicsSource`, `setSelectedLessonPerTopic` | "AI 생성" 버튼 |
| Lesson 토글 | `setTopicsSource` | "기존 레슨" 버튼 |
| 단어수 입력 | `setTopicsWordCount` | onChange |
| 장르 Select | `setTopicsGenres` | onValueChange |
| 주제 textarea | `setTopics` | onChange |
| Lesson popover 선택 | `setSelectedLessonPerTopic`, `setTopics`, `setTopicsWordCount`, `setTopicsGenres`, `setSelectedModulesPerTopic` | 한 클릭에 5개 setter |
| LessonModuleSection.onDurationChange | `setTopicsDuration` | duration 버튼 |
| LessonModuleSection.onModulesChange | `setSelectedModulesPerTopic` | 시퀀싱 결과 자동 적용 + 사용자 토글 |

### 9-2. Race Condition 해결 검증

**시뮬레이션** (회차 3개가 동시에 sequence 적용):

```
state: [[], [], []]
회차 0 setSelectedModulesPerTopic((prev) => ... prev=[[],[],[]] → [seq0,[],[]])
회차 1 setSelectedModulesPerTopic((prev) => ... prev=[seq0,[],[]] → [seq0,seq1,[]])
회차 2 setSelectedModulesPerTopic((prev) => ... prev=[seq0,seq1,[]] → [seq0,seq1,seq2])
```

functional updater는 React가 직전 결과를 prev로 전달 → race 없음 → **모든 회차에 sequence 반영**.

### 9-3. 빌드/배포

- API + Web build 성공
- `.next/` 정리 완료

### 9-4. 미검증 (브라우저 직접 확인 필요)

- 회차 3개 이상에서 모두 시퀀싱 표시 (모듈 칩 1,2,3,...)
- 회차 1 duration 변경 → 회차 1만 재시퀀싱 (Network 탭 확인)
- 회차별 입력 (단어수/장르/주제) 변경 시 다른 회차 영향 없음

### 9-5. 학습 (체크리스트 갱신 후보)

- "마지막만 반영" 증상 = 비함수형 setState race 의심
- 동일 query를 여러 컴포넌트에서 동시 구독 시, 부모 setState는 **반드시 functional updater**
- 인덱스로 배열 갱신하는 모든 핸들러는 일관성 있게 functional updater 적용

---

## 8. 추가 학습 (체크리스트 갱신 후보)

이번 사례 → **다음과 같은 체크리스트 항목 추가 권장**:

| 신규 항목 |
|----------|
| 동일 query를 여러 컴포넌트 인스턴스에서 동시 구독할 때, 부모 setState가 비함수형이면 race 발생 가능. **배열/객체 인덱스 갱신은 항상 functional updater** 사용 |
| "마지막만 반영" 증상 = 연속 setState race 의심 |
