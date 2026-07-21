# 오류: 과정 생성 Wizard — 복합 버그 2건

> **최초 발견일**: 2026-06-05
> **최종 업데이트**: 2026-06-07
> **서비스**: studio.picklass.com3
> **관련 기능**: 과정 생성 wizard (1/2 → 2/2)

---

## Bug 1 — 과정주제가 AI 주제 생성에 반영되지 않음

> **발견일**: 2026-06-05
> **관련 화면**: 과정 생성 (1/2) — AI 주제 생성

### 증상 (Symptom)

과정 생성 (1/2) 화면에서 "3. 과정주제" 입력란에 주제를 입력한 후 "다음" 버튼을 누르면,
AI가 생성하는 회차별 구성 주제가 입력한 과정주제를 전혀 반영하지 않는다.

### 원인 (Root Cause)

`courseSubject` state가 AI 주제 생성 API 요청 payload에 포함되지 않는다.

#### 데이터 흐름

```
UI 입력 (courseSubject)
  ↓
runGenerateAITopics()                    ← courseSubject 누락
  ↓
generateTopics.mutateAsync({
  course_name,
  course_goal,   ← KPI 목표 텍스트 (courseSubject 아님)
  level,
  genre,
  lesson_count,
  // course_subject 없음 ← 버그
})
  ↓
POST /ai/generate-topics
  ↓
GenerateTopicsDto                        ← course_subject 필드 없음
  ↓
buildTopicGenerationPrompt()             ← course_subject 반영 없음
```

#### 누락된 연결 고리 목록

| 위치 | 파일 | 문제 |
|------|------|------|
| 프론트엔드 호출부 | `apps/web/src/app/course/page.tsx:308` | `courseSubject`를 mutateAsync payload에 전달하지 않음 |
| 공유 타입 | `packages/shared/src/types/auth.ts:15` | `GenerateTopicsRequest`에 `course_subject` 필드 없음 |
| 백엔드 DTO | `apps/api/src/ai/dto/ai.dto.ts:3` | `GenerateTopicsDto`에 `course_subject` 필드 없음 |
| 백엔드 컨트롤러 | `apps/api/src/ai/ai.controller.ts:14` | `dto.course_subject`를 service로 전달하지 않음 |
| 백엔드 서비스 | `apps/api/src/ai/ai.service.ts:26` | `courseSubject` 파라미터 없음 |
| AI 프롬프트 | `apps/api/src/ai/prompts/topic-generation.ts:11` | 프롬프트에 과정주제 항목 없음 |

### 해결 방법 (Resolution)

총 6개 파일 수정. ✅ 2026-06-07 적용 완료.

#### 1. 공유 타입 (`packages/shared/src/types/auth.ts`)

```ts
export interface GenerateTopicsRequest {
  course_name: string;
  course_goal: string;
  course_subject: string;   // 추가
  level: string;
  genre: string;
  lesson_count: number;
}
```

#### 2. 백엔드 DTO (`apps/api/src/ai/dto/ai.dto.ts`)

```ts
export class GenerateTopicsDto {
  // ...기존 필드...

  @IsString()
  @IsNotEmpty()
  course_subject!: string;   // 추가
}
```

#### 3. 백엔드 컨트롤러 (`apps/api/src/ai/ai.controller.ts`)

```ts
return this.aiService.generateTopics({
  courseName: dto.course_name,
  courseGoal: dto.course_goal,
  courseSubject: dto.course_subject,   // 추가
  level: dto.level,
  genre: dto.genre,
  lessonCount: dto.lesson_count,
});
```

#### 4. 백엔드 서비스 (`apps/api/src/ai/ai.service.ts`)

```ts
async generateTopics(params: {
  courseName: string;
  courseGoal: string;
  courseSubject: string;   // 추가
  level: string;
  genre: string;
  lessonCount: number;
}): Promise<TopicsResult> {
  const prompt = buildTopicGenerationPrompt({
    courseName: params.courseName,
    courseGoal: params.courseGoal,
    courseSubject: params.courseSubject,   // 추가
    // ...
  });
```

#### 5. AI 프롬프트 (`apps/api/src/ai/prompts/topic-generation.ts`)

```ts
interface TopicPromptParams {
  courseName: string;
  courseGoal: string;
  courseSubject: string;   // 추가
  level: string;
  genre: string;
  lessonCount: number;
}
```

프롬프트 Course Details 항목에 추가:
```
- Course Subject: ${courseSubject}
```

Requirements 2번 교체:
```
2. Topics must be directly related to the course subject and aligned with the course goal
```

#### 6. 프론트엔드 호출부 (`apps/web/src/app/course/page.tsx`)

```ts
const result = await generateTopics.mutateAsync({
  course_name: courseName,
  course_goal: goalText,
  course_subject: courseSubject,   // 추가
  level: wizardLevel,
  genre: selectedGenres[0] || '설명문',
  lesson_count: rounds,
});
```

---

## Bug 2 — 과정유형에 관계없이 모든 모듈이 표시되고 시퀀싱됨

> **발견일**: 2026-06-07
> **관련 화면**: 과정 생성 (2/2) — 회차별 구성

### 증상 (Symptom)

- speaking 과정에서도 tutoring 모듈이 표시되고 시퀀싱에 포함됨
- tutoring 과정에서도 speaking 모듈이 표시되고 시퀀싱에 포함됨
- 통합 과정만 모든 모듈이 보여야 하는데 구분 없이 전체가 표시됨

### 원인 (Root Cause)

`page.tsx`에서 `moduleCategories`를 `courseType` 필터 없이 그대로 `LessonModuleSection`에 전달.
`LessonModuleSection` 내부에서도 시퀀싱 API 응답(`sequenceCodes`)을 전달받은 `moduleCategories`로 걸러내지 않아, 서버가 반환한 전체 모듈 코드가 `selectedModules`에 그대로 적용됨.

#### 데이터 흐름

```
useModuleCategories()               ← 전체 스킬 모듈 반환
  ↓
moduleCategories (필터 없음)
  ↓
LessonModuleSection
  ├── ModuleChipList                 ← 전체 모듈 표시 (버그)
  └── useLessonPlanSequence()
        ↓
      sequenceCodes (서버 전체 결과) ← 스킬 구분 없이 selectedModules에 적용 (버그)
```

#### 관련 상수/함수

`kpiSkillFilter`(page.tsx:69-75)가 동일한 필터 로직을 이미 구현하고 있었으나 Step 1 KPI 목록에만 적용되고 Step 2 모듈 영역에는 적용되지 않았음.

### 해결 방법 (Resolution)

2개 파일 수정. ✅ 2026-06-07 적용 완료.

#### 1. `apps/web/src/app/course/page.tsx`

`courseType` state 선언 직후 `filteredModuleCategories` 파생값 추가:

```ts
const filteredModuleCategories = useMemo(() => {
  if (!courseType || courseType === 'integrated') return moduleCategories;
  return Object.fromEntries(
    Object.entries(moduleCategories).filter(([skill]) =>
      courseType === 'speaking' ? skill === SPEAKING_SKILL : skill !== SPEAKING_SKILL,
    ),
  );
}, [moduleCategories, courseType]);
```

`LessonModuleSection`에 전달 시 교체:
```ts
// Before
moduleCategories={moduleCategories}
// After
moduleCategories={filteredModuleCategories}
```

#### 2. `apps/web/src/components/LessonModuleSection.tsx`

시퀀싱 결과를 허용 모듈 코드로 제한:

```ts
const allowedModuleCodes = useMemo(
  () => new Set(Object.values(moduleCategories).flatMap((cat) => cat.modules.map((m) => m.code))),
  [moduleCategories],
);

const sequenceCodes = useMemo(
  () =>
    (lessonPlanQuery.data?.sequence.map((s) => s.module_code) ?? []).filter((code) =>
      allowedModuleCodes.has(code),
    ),
  [lessonPlanQuery.data, allowedModuleCodes],
);
```

### 재발 방지 (Prevention)

- 모듈 목록 표시와 시퀀싱 적용은 **같은 필터 기준**을 공유해야 한다.
  표시 필터와 시퀀싱 필터가 분리되면 "안 보이지만 선택된" 모듈이 생겨 저장 데이터가 오염된다.
- `LessonModuleSection`은 `moduleCategories` prop으로 허용 범위를 전달받으므로,
  시퀀싱 결과도 반드시 같은 `moduleCategories` 기준으로 교집합을 취한다.

---

## 공통 재발 방지 (Prevention)

- UI에 입력 필드가 추가되거나 필터 조건이 생기면, **UI → 서비스 함수 → API payload → DTO → Service → 프롬프트/시퀀서** 전 흐름을 한 번에 추적한다.
- `courseType` 같은 분기 조건은 표시·시퀀싱·저장 세 레이어 모두에 일관되게 적용한다.
- 공유 타입(`GenerateTopicsRequest` 등)이 DTO / Service 파라미터와 동기화되어 있는지 확인한다.
