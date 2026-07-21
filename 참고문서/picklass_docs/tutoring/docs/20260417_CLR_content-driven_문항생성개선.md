# CLR content-driven 문항 생성 개선 및 타입 전파

> **작성일**: 2026-04-17  
> **이전 문서**: `20260412_SKM_CLR_개발내역.md`  
> **영향 범위**: tutoring API (question-generator, lessons, ai/prompts), tutoring Web (types, adapters, panels), backoffice seed

---

## 배경

`20260412_SKM_CLR_개발내역.md`에서 CLR 모듈 구현은 완료되었으나, 다음 문제가 남아있었다:

- **CLR 문장 생성이 첫 단락만 되는 버그**: LLM 프롬프트의 `[문항 수 결정]` 블록이 `min=5, max=20`으로 범위를 제한하여 지문 전체 문장을 추출하지 못했음.
- **근본 원인**: 튜터링 Supabase DB의 CLR `question_count`가 `'multi'`로 남아있어, 코드 변경이 반영되지 않았음.

---

## 1. 사용자 흐름 (User Flow)

### 1-1. CLR 모듈 학습 흐름 (변경 후)

1. 학생이 CLR 모듈이 포함된 레슨에 진입
2. `GET /lessons/:lessonId/module/:moduleCode` → `lessons.service.ts:getModuleData()` 호출
3. `ai_modules` 테이블에서 CLR 레코드 조회 → `questionCount = 'content-driven'`
4. `questionCount === 'content-driven'` → `min=1, max=999, recommendedCount=0`
5. `module_questions` 캐시 조회 → 캐시 히트 시 반환, 미스 시 Gemini 생성
6. Gemini 호출 시 `buildQuestionGenerationPrompt()` → `[문항 수 결정]` 블록 **생략**
7. LLM이 `contentGenerationInstruction`에 따라 지문 전체 문장을 1문장 1레코드로 생성
8. `decidedCount: 숫자 (지문 전체 문장 수)` 형태로 응답
9. `module_questions`에 저장 → 이후 캐시 히트로 즉시 반환

### 1-2. 캐시 무효화 흐름

- 모듈 설정(`contentGenerationInstruction`, `questionCount` 등) 변경 후 수동 호출:
  - `DELETE /ai/generate-questions/:moduleId` — 모듈 전체 캐시 무효화
  - `DELETE /ai/generate-questions/:moduleId/:textId` — 특정 지문 캐시 무효화

---

## 2. IA 구조 및 기능 정의 (IA)

### 2-1. questionCount: 'content-driven' 기능 정의

| 항목 | 내용 |
|------|------|
| 목적 | LLM이 지문 전체를 기준으로 문항 수를 결정 (지문 문장 수 = 문항 수) |
| 적용 모듈 | CLR (sentence-explain 타입) |
| min/max 처리 | DB 값 무시, 코드에서 `min=1, max=999`로 override |
| recommendedCount | `0` (LLM에 전달하지 않음) |
| 프롬프트 변화 | `[문항 수 결정]` 블록 완전 생략 |

### 2-2. 신규 API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `DELETE` | `/ai/generate-questions/:moduleId` | 모듈 전체 module_questions 캐시 무효화 (archived 처리) |
| `DELETE` | `/ai/generate-questions/:moduleId/:textId` | 특정 지문 캐시 무효화 (기존 유지) |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. content-driven 처리 정책 (신규)

- `questionCount === 'content-driven'`이면 `[문항 수 결정]` 블록을 LLM 프롬프트에서 생략한다.
- `[문항 수 결정]` 블록 생략 시 LLM은 `contentGenerationInstruction`에 명시된 방식에 따라 문항 수를 자체 결정한다.
- CLR의 경우 `contentGenerationInstruction`에 "지문 전체 문장 수만큼 반복"이 명시되어 있다.

### 3-2. 튜터링 Supabase DB 동기화 정책 (중요)

- 백오피스 `seed.ts`는 **백오피스 PostgreSQL**만 업데이트한다.
- 튜터링 API는 **별도의 Supabase DB** `ai_modules` 테이블을 조회한다 (`lessons.service.ts:307`).
- 모듈 설정 변경 시 Supabase DB도 직접 업데이트해야 한다.
- 현재 동기화 방법: Supabase REST API PATCH 요청.

```bash
curl -X PATCH \
  "https://{supabase-project}.supabase.co/rest/v1/ai_modules?code=eq.CLR&status=eq.active" \
  -H "Authorization: Bearer {SERVICE_ROLE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"question_count": "content-driven"}'
```

### 3-3. module_questions 캐시 무효화 정책

- `contentGenerationInstruction` 또는 `questionCount` 변경 후에는 반드시 캐시를 무효화해야 한다.
- 무효화는 `status = 'archived'` soft-delete 방식이다.
- 오늘 CLR 캐시 14건 무효화 완료 (기존 잘못된 형식 레코드 archived 처리).

### 3-4. 출력 포맷 템플릿 정책 (변경됨)

- `sentence-explain` 타입: `"decidedCount": 숫자 (지문 전체 문장 수)` — 명확한 기대값 명시
- 일반 타입 `content-driven`: `"decidedCount": 숫자 (콘텐츠 기반 결정)` — LLM 자율 결정 명시
- 일반 타입 `single/multi`: `"decidedCount": 숫자 ({min}~{max} 범위)` — 기존 유지

---

## 4. 추가 개발 필요 사항

### 4-1. 튜터링-백오피스 DB 자동 동기화 (높음)

현재 수동 동기화 구조. 모듈 설정 변경 시 누락 가능성이 있음.  
백오피스 AI 모듈 저장 API(`PUT /ai-modules/:id`)에서 튜터링 Supabase로 자동 PATCH 호출하는 로직 필요.

### 4-2. 캐시 무효화 자동 트리거 (중간)

`contentGenerationInstruction`, `questionCount`, `questionMinCount`, `questionMaxCount` 변경 시  
튜터링 API의 `DELETE /ai/generate-questions/:moduleId`를 자동으로 호출하는 웹훅 또는 미들웨어 필요.

### 4-3. CLR 피드백 경로 개선 (미구현)

CLR은 답안 제출이 없는 단방향 모듈이나, 현재 `/ai/chat` 호출 시 `answerType` 컨텍스트가 없어  
"답을 찾는 데 도움이 될 거예요" 등 부적절한 피드백이 발생할 수 있음.  

개선 방향:
- Path B (`/ai/chat`): `answerType`, 활성 질문, 학생 답안 컨텍스트 추가
- CLR `pedagogyInstruction`: 답안 제출 없음을 명시하는 금지 지시문 추가

---

## 5. 코드 규칙 (Coding Rules)

### 5-1. questionCount 타입 추가 시 체크리스트

`questionCount`에 새 값 추가 시 모든 파일 동시 수정 필요:

```
tutoring/apps/web/src/lib/types/ai-module-data.ts
tutoring/apps/web/src/lib/agents/agent-types.ts (2곳)
tutoring/apps/web/src/lib/adapters/GenericAdapter.ts
tutoring/apps/web/src/app/modules/[lessonId]/_components/panels.tsx
tutoring/apps/web/src/app/modules/[lessonId]/_components/MobileSplitLayout.tsx
tutoring/apps/api/src/ai/ai.controller.ts
tutoring/apps/api/src/ai/question-generator.service.ts
tutoring/apps/api/src/ai/prompts/question-generation.ts
tutoring/apps/api/src/lessons/lessons.service.ts
backoffice/apps/admin/frontend/.../ai-modules/register/page.tsx
```

### 5-2. content-driven 분기 패턴

```typescript
// ✅ 올바른 패턴
const questionCount = (aiModule.questionCount ?? 'single') as 'single' | 'multi' | 'content-driven';
const min = questionCount === 'content-driven' ? 1 : (aiModule.questionMinCount ?? 1);
const max = questionCount === 'content-driven' ? 999 : (aiModule.questionMaxCount ?? 1);
const recommendedCount = questionCount === 'content-driven' ? 0 : calcRecommendedCount(...);

// ❌ 금지 패턴 — 이진 분기
const isMulti = questionCount === 'multi';
```

### 5-3. 프롬프트 countBlock 패턴

```typescript
// ✅ content-driven이면 countBlock 완전 생략
const countBlock = params.questionCount === 'content-driven'
  ? ''
  : `\n[문항 수 결정]\n...`;
```

### 5-4. 금지 패턴

- `content-driven` 모듈에서 `calcRecommendedCount()` 호출 금지 (0으로 고정)
- LLM 프롬프트에 `1~999 범위` 표현 노출 금지 (LLM 혼란 유발)

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 구분 | 파일 | 내용 | 우선순위 |
|------|------|------|---------|
| **DB 이중화** | `lessons.service.ts:307` | 튜터링 Supabase AI 모듈 수동 동기화 | 높음 |
| **CLR 피드백 경로** | `prompts/chat.ts` | `/ai/chat` 호출 시 answerType 미전달, CLR 전용 컨텍스트 없음 | 높음 |
| **캐시 무효화** | `question-generator.service.ts` | 모듈 설정 변경 후 수동 API 호출 필요 | 중간 |
| **dead code** | `picklass_pedagogy_profiles.ts` | 이전 하드코딩 프로파일, DB 기반으로 전환 완료 후 삭제 필요 | 낮음 |
| **min/max 의미없는 저장** | `ai_modules.question_min/max_count` | content-driven 모듈에 `1/1` 저장, 의미 없음 | 낮음 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 7-1. content-driven 관련 코드 흐름

```
lessons.service.ts:getModuleData()
  │  aiModule.questionCount === 'content-driven'
  │  → min=1, max=999, recommendedCount=0
  ↓
questionGeneratorService.getOrGenerate(dto)
  │  dto.questionCount = 'content-driven'
  ↓
aiService.generateQuestions(params)
  ↓
buildQuestionGenerationPrompt(params)
  │  questionCount === 'content-driven' → countBlock = ''
  │  answerType === 'sentence-explain' → CLR 전용 템플릿
  ↓
Gemini API
  → JSON { decidedCount, questions: [...전체 문장...] }
  ↓
module_questions.createMany()
```

### 7-2. 타입 파일 의존성

```
ai-module-data.ts (AiModuleData.questionCount)
  └── aiModuleService.ts (parseAiModuleResponse, VALID_QUESTION_COUNTS)
       └── lessons.service.ts (getModuleData 응답)
            └── GenericAdapter.ts (buildPedagogyProfile)
                 └── agent-types.ts (ModulePedagogyProfile)
                      └── panels.tsx, MobileSplitLayout.tsx (props)
```

### 7-3. 진입점

- 튜터링 Web: `modules/[lessonId]/page.tsx` → `LessonSession.tsx`
- 튜터링 API: `GET /lessons/:lessonId/module/:moduleCode`
- 관리자 수동: `DELETE /ai/generate-questions/:moduleId` (캐시 무효화)

---

## 8. DB/API 구조 (Data Contract)

### 8-1. 튜터링 Supabase ai_modules — CLR 현재 상태

```
id:                   0f79ce16-c903-4049-aeb0-43df7309c9eb
code:                 CLR
question_count:       content-driven       ← 오늘 업데이트됨
question_min_count:   1
question_max_count:   1
question_generation_strategy: extract
answer_type:          sentence-explain
feedback_style:       sentence-explain
```

### 8-2. GenerateQuestionsDto (question-generator.service.ts)

```typescript
export interface GenerateQuestionsDto {
  moduleId: string;
  textId: number;
  passage: { title: string; content: string };
  strategy: 'extract' | 'instruct';
  questionCount: 'single' | 'multi' | 'content-driven';  // 추가됨
  minCount: number;
  maxCount: number;
  recommendedCount: number;
  passageLevel: number;
  answerType: string;
  pedagogyInstruction: string;
  contentGenerationInstruction?: string;
}
```

### 8-3. QuestionGenerationPromptParams (question-generation.ts)

```typescript
export interface QuestionGenerationPromptParams {
  passageTitle: string;
  passageContent: string;
  strategy: 'extract' | 'instruct';
  questionCount?: 'single' | 'multi' | 'content-driven';  // 추가됨
  minCount: number;
  maxCount: number;
  recommendedCount: number;
  passageLevel: number;
  answerType: string;
  pedagogyInstruction: string;
  contentGenerationInstruction?: string;
}
```

### 8-4. module_questions 캐시 무효화 API

```
DELETE /ai/generate-questions/:moduleId
  → module_questions WHERE module_id=:moduleId AND status='active'
  → UPDATE status='archived'
  → { invalidated: N }

DELETE /ai/generate-questions/:moduleId/:textId
  → 특정 지문만 무효화
```

### 8-5. agent-types.ts — 영향 받은 타입

```typescript
// ModulePedagogyProfile
questionCount: 'single' | 'multi' | 'content-driven';  // 변경됨

// CorrectnessFeedbackParams
questionCount?: 'single' | 'multi' | 'content-driven';  // 변경됨
```

---

## 변경 요약

| 변경 | 파일 | 내용 |
|------|------|------|
| **신규** | `ai.controller.ts` | `DELETE /ai/generate-questions/:moduleId` 엔드포인트 추가 |
| **수정** | `question-generator.service.ts` | `GenerateQuestionsDto.questionCount` 타입, `invalidateModuleCache()` 추가 |
| **수정** | `prompts/question-generation.ts` | `content-driven` 분기 + decidedCount 출력 포맷 개선 |
| **수정** | `lessons.service.ts` | `content-driven` → min/max override, `questionCount` 전달 |
| **수정** | `ai-module-data.ts` | `questionCount` 타입에 `'content-driven'` 추가 |
| **수정** | `agent-types.ts` | `ModulePedagogyProfile`, `CorrectnessFeedbackParams` 타입 확장 |
| **수정** | `panels.tsx` | `FeedbackPanel` props 타입 확장 |
| **수정** | `MobileSplitLayout.tsx` | props 타입 확장 |
| **DB 업데이트** | Supabase ai_modules | CLR `question_count = 'content-driven'` 직접 PATCH |
| **캐시 무효화** | Supabase module_questions | CLR 14건 archived 처리 |
