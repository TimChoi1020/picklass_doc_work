# AI 모듈 등록 — `questionCount: content-driven` 추가

> **작성일**: 2026-04-17  
> **이전 문서**: `20260412_AI모듈등록.md`  
> **변경 커밋**: `7a388a6` (모듈옵션추가)  
> **영향 범위**: 백오피스 등록 UI, seed.ts, 튜터링 API / Web

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 모듈 등록/수정 — questionCount 선택 흐름 (변경됨)

1. **문항설계(QuestionData)** 탭 → `questionCount` 드롭다운에 `content-driven` 항목 추가
2. `content-driven` 선택 시:
   - `questionMinCount` / `questionMaxCount` 입력 필드 **비활성화** (`disabled`)
   - 경고 메시지: `※ content-driven 모듈은 지문 전체 추출 — min/max 무효`
3. `single` 선택 시:
   - 기존과 동일: `※ questionCount = 'multi' 전용 (single 모듈은 항상 1)` 표시
4. `multi` 선택 시:
   - `questionMinCount` / `questionMaxCount` 활성화

### 1-2. 데이터 로드 (수정 모드) — 변경됨

```typescript
// 이전
questionCount: (data.questionCount === 'multi' ? 'multi' : 'single')

// 변경 후
questionCount: (['single', 'multi', 'content-driven'].includes(data.questionCount)
  ? data.questionCount
  : 'single')
```

---

## 2. IA 구조 및 기능 정의 (IA)

### 2-1. questionCount 필드 변경

| 항목 | 이전 | 변경 후 |
|------|------|---------|
| 타입 | `'single' \| 'multi'` | `'single' \| 'multi' \| 'content-driven'` |
| DB 코드 그룹 | 없음 (하드코딩) | `QUESTION_COUNT` 코드 그룹 신규 추가 |
| `content-driven` 의미 | — | LLM이 지문 전체 문장을 콘텐츠 기준으로 모두 추출 (min/max 무시) |

### 2-2. `QUESTION_COUNT` 코드 그룹 (신규)

| code | name | sortOrder |
|------|------|-----------|
| `single` | 단일 문항 | 1 |
| `multi` | 복수 문항 (레벨 기반) | 2 |
| `content-driven` | 지문 전체 추출 | 3 |

> **비고**: `content-driven` 모듈에서 `questionMinCount` / `questionMaxCount`는 튜터링 API에서 무시되며, 문항 수는 지문 전체 문장 수에 따라 결정됩니다.

### 2-3. 탭 — 문항설계(QuestionData) UI 규칙 (업데이트)

| questionCount 값 | min/max 필드 | 경고 메시지 |
|-----------------|-------------|------------|
| `single` | disabled | `※ questionCount = 'multi' 전용 (single 모듈은 항상 1)` |
| `multi` | **enabled** | 없음 |
| `content-driven` | disabled | `※ content-driven 모듈은 지문 전체 추출 — min/max 무효` |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. content-driven 모듈 정책 (신규)

- `questionCount = 'content-driven'`인 모듈은 LLM이 지문의 모든 문장을 1문장 1레코드로 추출한다.
- `questionMinCount` / `questionMaxCount`는 저장되지만 **튜터링 API에서 override**: `min=1, max=999, recommendedCount=0`.
- `[문항 수 결정]` 프롬프트 블록은 LLM에게 전달되지 않는다.
- 대표 모듈: **CLR** (Clarification — sentence-explain)

### 3-2. seed.ts 정책 변경 (오늘 적용)

| 항목 | 이전 | 변경 후 |
|------|------|---------|
| 모듈 코드 | `WDR`, `WDS` (오타) | `WRD`, `WSD` (수정) |
| `questionGenerationStrategy` | 일부 모듈 누락 | 전체 14개 모듈 명시 |
| `questionMinCount` / `questionMaxCount` | 일부 모듈 누락 | 전체 14개 모듈 명시 |
| `pedagogyInstruction` | 미작성 모듈 多 (`'## 모듈 목적'` placeholder) | 전체 작성 완료 |
| `passageMode` | 조건부 spread (`'passageMode' in mod ? ...`) | 항상 포함 (default `'full'`) |
| `contentGenerationInstruction` | 조건부 spread | 항상 포함 (default `''`) |
| WSD `scoringMode` | `'exact'` | `'pronunciation'` |
| QAR `scoringMode` | `'exact'` | `'holistic'` |
| QAR `feedbackStyle` | `'correct-wrong'` | `'strengths-weaknesses'` |
| QAR `inputLanguage` | `''` | `'korean-or-english'` |
| CLR `questionCount` | `'multi'` | `'content-driven'` |
| CLR `questionMinCount/Max` | `5` / `20` | `1` / `1` |

### 3-3. 코드 그룹 등록 정책

- `questionCount` 값은 `QUESTION_COUNT` 코드 그룹으로 관리한다.
- 새로운 `questionCount` 유형 추가 시 반드시 `seed.ts`의 `QUESTION_COUNT` 코드 그룹에 먼저 등록 후 프론트엔드 타입을 확장한다.

---

## 4. 추가 개발 필요 사항

### 4-1. 튜터링 Supabase DB 동기화 필요 (중요)

백오피스 `seed.ts`는 **백오피스 PostgreSQL DB**만 업데이트한다.  
튜터링 API는 **별도의 Supabase DB**의 `ai_modules` 테이블을 참조하므로,  
모듈 설정 변경 시 튜터링 Supabase DB를 **별도로 동기화**해야 한다.

현재 동기화 방법:
- Supabase REST API를 통한 수동 PATCH 요청
- 예: `PATCH /rest/v1/ai_modules?code=eq.CLR` with `{"question_count": "content-driven"}`

**개선 필요**: 백오피스 모듈 저장 시 튜터링 Supabase로 자동 동기화하는 API 또는 훅 구현.

### 4-2. questionCount 드롭다운 — DB 코드 그룹 연동 필요

현재 `questionCount` 선택지는 TypeScript 타입으로 하드코딩되어 있다.  
`QUESTION_COUNT` 코드 그룹이 DB에 추가되었으므로, 다른 코드 그룹처럼 `codeMap['QUESTION_COUNT']`에서 로드하도록 변경 권장.

### 4-3. module_questions 캐시 무효화 절차 표준화

`contentGenerationInstruction` 또는 `questionCount` 변경 시 기존 캐시된 `module_questions` 레코드를 archived 처리해야 한다.  
현재는 수동으로 `DELETE /ai/generate-questions/:moduleId` API를 호출해야 한다.  
백오피스 모듈 수정 저장 시 자동으로 캐시 무효화 API를 호출하는 로직 추가 권장.

---

## 5. 코드 규칙 (Coding Rules)

### 5-1. questionCount 타입 확장 시 체크리스트

`questionCount`에 새 값을 추가할 때 반드시 아래 파일 모두 수정:

| 파일 | 수정 위치 |
|------|----------|
| `backoffice/apps/admin/frontend/.../register/page.tsx` | `ModuleFormData.questionCount` 타입, 로드 분기, UI 경고 메시지 |
| `tutoring/apps/web/src/lib/types/ai-module-data.ts` | `AiModuleData.questionCount` 타입 |
| `tutoring/apps/web/src/lib/agents/agent-types.ts` | `ModulePedagogyProfile.questionCount`, `CorrectnessFeedbackParams.questionCount` |
| `tutoring/apps/web/src/lib/adapters/GenericAdapter.ts` | `buildPedagogyProfile()` 타입 |
| `tutoring/apps/web/src/app/modules/[lessonId]/_components/panels.tsx` | `FeedbackPanel` props |
| `tutoring/apps/web/src/app/modules/[lessonId]/_components/MobileSplitLayout.tsx` | props 타입 |
| `tutoring/apps/api/src/lessons/lessons.service.ts` | `questionCount` 분기 로직 |
| `tutoring/apps/api/src/ai/prompts/question-generation.ts` | `QuestionGenerationPromptParams` 타입, `countBlock` 분기 |

### 5-2. 금지 패턴

- `questionCount === 'multi'`만 체크하는 이진 분기 금지 → `'content-driven'`을 별도 케이스로 처리
- seed.ts에서 `'passageMode' in mod ? ... : {}` 같은 조건부 spread 사용 금지 → 항상 기본값 포함

### 5-3. 네이밍

- 코드 그룹 key: `QUESTION_COUNT` (대문자 언더스코어)
- 코드 item: `content-driven` (소문자 하이픈, DB VarChar(20))

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 구분 | 내용 | 우선순위 |
|------|------|---------|
| **DB 이중화** | 백오피스 PostgreSQL과 튜터링 Supabase의 `ai_modules`가 분리되어 수동 동기화 필요 | 높음 |
| **하드코딩** | `questionCount` 선택지가 TypeScript 타입으로 고정, `QUESTION_COUNT` 코드 그룹에서 동적 로드 미구현 | 중간 |
| **캐시 무효화** | `contentGenerationInstruction` / `questionCount` 변경 후 수동으로 캐시 무효화 API 호출 필요 | 중간 |
| **min/max 저장** | `content-driven` 모듈의 `questionMinCount=1`, `questionMaxCount=1` 저장 (의미 없는 값) — 무시되지만 혼란 여지 | 낮음 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 7-1. 백오피스 등록 페이지 의존성

- `fetchApi` (`src/lib/api.ts`) — API 호출 공통 유틸
- `codeMap['QUESTION_COUNT']` — (현재 미연동, TypeScript 타입으로 하드코딩)
- `codeMap['QUESTION_GENERATION_STRATEGY']` — `extract` / `instruct`
- `disabled={formData.questionCount !== 'multi'}` — min/max 필드 비활성화 조건

### 7-2. 영향 받는 시스템

```
백오피스 register page
  └─ POST/PUT /api/ai-modules (백오피스 API)
       └─ backoffice PostgreSQL (ai_modules 테이블)
  
  [수동 동기화 필요]
  └─ Supabase REST API PATCH
       └─ tutoring Supabase (ai_modules 테이블)
            └─ tutoring API lessons.service.ts getModuleData()
                 └─ question-generator.service.ts getOrGenerate()
                      └─ ai/prompts/question-generation.ts buildQuestionGenerationPrompt()
```

---

## 8. DB/API 구조 (Data Contract)

### 8-1. ai_modules 테이블 — questionCount 관련 필드

```sql
question_count              VARCHAR(20)  DEFAULT 'single'   -- 'single' | 'multi' | 'content-driven'
question_generation_strategy VARCHAR(20) DEFAULT 'extract'  -- 'extract' | 'instruct'
question_min_count          INT          DEFAULT 1
question_max_count          INT          DEFAULT 1
```

> `content-driven`이면 `question_min_count`, `question_max_count`는 튜터링 API에서 무시됨.

### 8-2. 튜터링 API — lessons.service.ts content-driven 분기

```typescript
const questionCount = (aiModule.questionCount ?? 'single') as 'single' | 'multi' | 'content-driven';
const min = questionCount === 'content-driven' ? 1 : (aiModule.questionMinCount ?? 1);
const max = questionCount === 'content-driven' ? 999 : (aiModule.questionMaxCount ?? 1);
const recommendedCount = questionCount === 'content-driven' ? 0 : calcRecommendedCount(passageLevel, min, max);
```

### 8-3. question-generation.ts countBlock 분기

```typescript
const countBlock = params.questionCount === 'content-driven'
  ? ''  // [문항 수 결정] 블록 완전 생략
  : `\n[문항 수 결정]\n- 목표 문항 수: ${params.recommendedCount}개 ...`;
```

### 8-4. 출력 형식 템플릿 — decidedCount

```typescript
// sentence-explain (CLR)
"decidedCount": 숫자 (지문 전체 문장 수),

// 일반 타입
"decidedCount": 숫자 (${params.questionCount === 'content-driven' 
  ? '콘텐츠 기반 결정' 
  : `${params.minCount}~${params.maxCount} 범위`}),
```

### 8-5. ModuleFormData 타입 (register/page.tsx)

```typescript
interface ModuleFormData {
  // ...
  questionCount: 'single' | 'multi' | 'content-driven';  // 변경됨
  questionGenerationStrategy: 'extract' | 'instruct';
  questionMinCount: number;
  questionMaxCount: number;
  // ...
}
```

### 8-6. 오늘 기준 14개 모듈 questionCount 설정

| 모듈 코드 | 이름 | questionCount | min | max | strategy |
|----------|------|--------------|-----|-----|---------|
| WRD | Word Decker Reading | multi | 5 | 10 | instruct |
| WSD | Word Decker Speaking | multi | 5 | 10 | instruct |
| GMN | Guessing Meaning | multi | 3 | 8 | instruct |
| WWB | Word Web | multi | 3 | 7 | instruct |
| PRD | Prediction | single | 1 | 1 | instruct |
| SCN | Scanning | single | 1 | 1 | instruct |
| SKM | Skimming | multi | 3 | 7 | instruct |
| **CLR** | **Clarification** | **content-driven** | **1** | **1** | extract |
| SUM | Summarizing | single | 1 | 1 | instruct |
| QAR | Q-A Relationship | multi | 4 | 8 | instruct |
| RRD | Repeated Reading | multi | 3 | 5 | instruct |
| SHR | Story Q Type Analysis | multi | 3 | 8 | instruct |
| SWR | Sentence Writing | multi | 3 | 7 | instruct |
| PWR | Paragraph Writing | multi | 1 | 3 | instruct |
