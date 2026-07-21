# 튜터링 모듈 콘텐츠 생성 구현 — Speaking 참조 문서

> **작성일**: 2026-06-12  
> **대상**: speaking.picklass.com 개발 참조용  
> **출처**: tutoring.picklass.com 실제 구현 기준  
> **중점 범위**: 2단계(어댑터 패턴) + 3단계(백엔드 Cache-Aside 콘텐츠 생성)
>
> 튜터링에서 검증된 패턴을 speaking에서 동일하게 차용한다.
> 튜터링과 구조가 동일한 부분은 이 문서를 단일 참조점으로 삼는다.

---

## 배경 — 왜 최초 진입 시 콘텐츠를 생성하는가

### 지문 생성 시점에 콘텐츠를 미리 만들 수 없는 이유

하나의 지문은 여러 모듈에 걸쳐 다양하게 시퀀싱될 수 있다. 예를 들어 같은 지문에 대해
어떤 과정은 `PRD → SWR → SHR` 순으로, 다른 과정은 `CLR → PRD → SUM` 순으로 구성될 수 있다.
또한 새 모듈이 언제든지 백오피스에 등록될 수 있어 모듈 조합은 사전에 확정되지 않는다.

이 구조에서 지문을 등록하는 시점에 "이 지문을 어떤 모듈에서 어떻게 쓸 것인가"를 알 수 없으므로,
**지문 생성 단계에서 학습 문항을 미리 만들어 둘 수 없다**.

```
지문 등록 시점
  ├── 어떤 과정에 쓰일지 모름
  ├── 어떤 모듈과 조합될지 모름
  └── 새 모듈이 추가될 수도 있음
  → 콘텐츠 사전 생성 불가

사용자 모듈 진입 시점
  ├── 지문이 확정됨 (text_id)
  ├── 모듈이 확정됨 (moduleCode)
  └── content_generation_instruction이 있음
  → 이 조합으로 LLM이 콘텐츠 생성 가능
```

### 최초 진입 사용자 — 콘텐츠 생성 트리거

해당 지문 + 모듈 조합의 `module_questions`가 없는 경우(캐시 미스),
**그 학습 세션의 최초 진입 사용자**가 콘텐츠 생성을 트리거한다.

생성은 `ai_modules.content_generation_instruction` 필드에 저장된 프롬프트를
LLM API(Gemini)에 전달하는 방식으로 이루어진다.
이 필드는 "이 모듈에서 이 지문으로 어떤 문항을 어떻게 만들 것인가"를
백오피스에서 관리자가 설정하는 모듈별 고유 지시문이다.

```
최초 진입 사용자
  ↓ module_questions 없음 (캐시 미스)
  ↓ ai_modules.content_generation_instruction 읽기
  ↓ 지문(texts.content) + 프롬프트 → Gemini API 호출
  ↓ 생성된 문항 → module_questions 테이블에 저장 (status='active')
  ↓ 이후 사용자는 저장된 데이터 바로 사용
```

### 이후 사용자 — 저장된 콘텐츠 재사용

동일한 지문 + 모듈 조합이 이미 `module_questions`에 있으면(캐시 히트),
LLM 호출 없이 DB에서 즉시 반환한다.

이 설계로 얻는 효과:
- **LLM 비용 절감**: 동일 조합은 LLM을 한 번만 호출
- **응답 속도**: DB 조회만으로 즉시 반환, 체감 속도 대폭 향상
- **일관성**: 같은 과정을 수강하는 모든 학생이 동일한 문항을 받음

```
캐시 미스 (최초 진입): 지문 분석 + LLM 생성 → 수 초 소요
캐시 히트 (이후 진입): DB 조회만 → 수십 ms
```

---

## 전체 흐름 요약

| 단계 | 사용자 행동 | API 호출 | DB 처리 |
|---|---|---|---|
| **① 학습 시작** | `/modules/[lessonId]` URL 진입, 로딩 화면 표시 | `GET /lessons/:lessonId/plan` | `course_lessons` 조회 (text_id, skill_modules JSONB) → `texts` 조회 (지문 제목·내용) |
| **② 어댑터 생성** | 로딩 대기 | — (클라이언트 내부 처리) | — |
| **③ 콘텐츠 요청** | 로딩 대기 | `GET /lessons/:lessonId/module-data?moduleCode=XXX` | `ai_modules` 조회 (모듈 설정 전체: 생성 전략, 프롬프트, 문항 수 범위 등) |
| **④-A 캐시 히트** | 학습 화면 즉시 노출 | — | `module_questions` 조회 (module_id + text_id, status='active') → 즉시 반환 |
| **④-B 캐시 미스** | 수 초 로딩 대기 (최초 진입 시) | Gemini API 호출 (외부) | `module_questions` 조회 → 없음 확인 → Gemini 생성 → `module_questions` INSERT (status='active') |
| **⑤ 학습 진행·완료** | 지문 읽기 → 문항 풀기 → 완료 버튼 | `POST /lessons/:lessonId/modules/:moduleCode/history` | `module_histories` INSERT (score, answers, chat_messages) |

> **④-A / ④-B 분기**: 동일 `module_id + text_id` 조합이 `module_questions`에 존재하면 히트, 없으면 미스.
> 최초 진입 이후 모든 사용자는 저장된 캐시(④-A)를 사용한다.

---

## 2단계 — 모듈 콘텐츠 로드 (어댑터 패턴)

### 2-1. 어댑터 생성 흐름

프론트엔드(`LessonSession.tsx`)는 모듈을 실행하기 전에 어댑터를 생성한다.

```typescript
// apps/web/src/lib/adapters/index.ts
export function createAdapterForModule(
  code: string,
  aiModuleData?: AiModuleData,
): ModuleAdapter {
  if (aiModuleData) return new GenericAdapter(aiModuleData);
  throw new Error(`AiModuleData is required for module "${code}".`);
}
```

- `AiModuleData`가 있으면 항상 `GenericAdapter`를 사용한다 (R4 원칙: DB 기반 단일 어댑터).
- 과거에는 모듈 코드별 전용 어댑터 파일이 있었으나 전량 제거됨 (2026-04-07).

### 2-2. AiModuleData — ai_modules 테이블 매핑

`AiModuleData`는 `ai_modules` 테이블을 TypeScript 타입으로 매핑한 것이다.
어댑터가 알아야 하는 **모든 모듈 설정이 이 한 테이블에서 온다**.

| 필드명 (TypeScript) | DB 컬럼 | 설명 |
|---|---|---|
| `code` | `code` | 모듈 식별자 (예: `PRD`, `SWR`, `CLR`) |
| `name` | `name` | 모듈 표시명 |
| `skill` | `skill` | 영역 (`reading`, `writing`, `speaking` 등) |
| `uiTemplate` | `ui_template` | UI 구조 결정 (`standard`, `step-workflow`, `vocab-deck`, `interactive`, `timed-gate`) |
| `passageMode` | `passage_mode` | 지문 노출 방식 |
| `questionFlowMode` | `question_flow_mode` | 문항 흐름 모드 |
| `answerType` | `answer_type` | 문항 입력 타입 (`multiple-choice`, `essay`, `audio-record`, `sentence-write` 등) |
| `scoringMode` | `scoring_mode` | 채점 방식 (`exact`, `holistic`, `pronunciation`) |
| `questionCount` | `question_count` | 문항 수 유형 (`single`, `multi`, `content-driven`) |
| `questionGenerationStrategy` | `question_generation_strategy` | LLM 생성 전략 (`extract`, `instruct`) |
| `questionMinCount` | `question_min_count` | 최소 문항 수 |
| `questionMaxCount` | `question_max_count` | 최대 문항 수 |
| `feedbackStyle` | `feedback_style` | 피드백 스타일 (`correct-wrong`, `strengths-weaknesses`) |
| `purpose` | `purpose` | 모듈 목적 (Orchestrator greeting 생성에 사용) |
| `pedagogyInstruction` | `pedagogy_instruction` | LLM 피드백 시스템 프롬프트 |
| `contentGenerationInstruction` | `content_generation_instruction` | 문항 생성 전용 지시문 (없으면 기본값 사용) |
| `hintTypes` | `hint_types` | 허용 힌트 종류 배열 |
| `retryScope` | `retry_scope` | 재도전 기준 |
| `inputLanguage` | `input_language` | 학습자 입력 언어 |
| `passageRole` | `passage_role` | 지문의 학습 역할 |
| `questionMaxAttempts` | `question_max_attempts` | 문항당 최대 시도 횟수 |
| `selectedKpiCodes` | `selected_kpi_codes` | KPI 코드 배열 |
| `questionData` | `question_data` | JSONB fallback 문항 데이터 (캐시 미스 시 비상용) |

### 2-3. GenericAdapter.fetchModuleData()

어댑터는 `GET /lessons/:lessonId/module-data?moduleCode=XXX`를 호출하여
지문과 문항을 받아온다.

```typescript
// apps/web/src/lib/adapters/GenericAdapter.ts
async fetchModuleData(lessonId: string): Promise<ModuleData> {
  const res = await fetch(
    `${apiUrl}/lessons/${lessonId}/module-data?moduleCode=${this.code}`
  );
  const data = await res.json();
  // module_questions → QuestionData 매핑 (question_number → number 등)
  return {
    lessonId, code, skill, passage, questions, uiTemplate, passageMode, ...
  };
}
```

반환되는 `ModuleData` 구조:

```typescript
interface ModuleData {
  lessonId:        string;
  code:            string;
  skill:           string;
  round:           number;
  title:           string;
  passage:         { title: string; content: string };
  questions:       QuestionData[];
  uiTemplate:      'standard' | 'step-workflow' | 'vocab-deck' | 'interactive' | 'timed-gate';
  passageMode?:    PassageMode;
  questionFlowMode?: QuestionFlowMode;
}
```

---

## 3단계 — 콘텐츠 생성 (백엔드 Cache-Aside)

### 3-1. 관련 테이블

| 테이블 | 역할 |
|---|---|
| `course_lessons` | 레슨 단위 데이터. `text_id`(지문 FK), `skill_modules`(JSONB 모듈 시퀀스) 보유 |
| `texts` | 지문 원본. `title`, `content`, `level`(CEFR 또는 1~18 숫자) |
| `ai_modules` | 모듈 마스터. 교수법/생성 전략/프롬프트 등 모든 모듈 설정 |
| `module_questions` | 생성된 문항 캐시. `module_id + text_id` 조합이 캐시 키 |

#### module_questions 테이블 주요 컬럼

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `module_id` | UUID | `ai_modules.id` FK |
| `text_id` | INT | `texts.id` FK |
| `question_number` | INT | 문항 순서 |
| `type` | VARCHAR | 문항 타입 (`multiple-choice`, `essay`, `audio-record` 등) |
| `instruction` | TEXT | 지시문 (예: "다음 밑줄 친 단어의 의미를 고르시오") |
| `source` | TEXT | extract 전략: 추출 원문 문장 |
| `text` | TEXT | 핵심 질문 본문 |
| `options` | JSONB | 객관식 선택지 배열 또는 CLR의 `{해석, 문법설명, 주요표현}` 객체 |
| `answer` | TEXT | 정답 (holistic 채점 모듈은 빈 문자열) |
| `hints` | JSONB | `{ button: "단서", direct: "방향 안내" }` |
| `sort_order` | INT | 정렬 순서 |
| `status` | VARCHAR | `active` (사용 중) / `archived` (캐시 무효화됨) |

### 3-2. Cache-Aside 패턴 흐름

```
GET /lessons/:lessonId/module-data?moduleCode=XXX
  ↓
lessons.service.ts: getModuleData()
  │
  ├── 1) course_lessons 조회 (text_id, lesson 정보)
  ├── 2) ai_modules 조회 (questionCount, strategy, min/max, pedagogyInstruction 등)
  │
  └── 3) 문항 조회: Cache-Aside
        │
        ├── module_questions.findMany({
        │     module_id: aiModule.id,
        │     text_id:   lesson.text_id,
        │     status:    'active'
        │   })
        │
        ├── 캐시 HIT → 즉시 반환 (Gemini 호출 없음)
        │
        └── 캐시 MISS
              ↓
              questionGeneratorService.getOrGenerate(dto)
              ↓
              aiService.generateQuestions(params)
              ↓
              [Gemini 호출]
              ↓
              module_questions.createMany({ data: generatedQuestions })
              ↓
              저장된 레코드 재조회 후 반환
```

**캐시 키**: `module_id + text_id` 조합. 동일 지문에서 동일 모듈 문항은 첫 번째 학습자가
생성을 트리거하고, 이후 모든 학습자는 저장된 캐시를 재사용한다.

### 3-3. 문항 수 결정 로직

```typescript
// lessons.service.ts: getModuleData()
const questionCount = aiModule.questionCount as 'single' | 'multi' | 'content-driven';
const passageLevel  = parsePassageLevel(lesson.text.level); // 1~18

// content-driven: LLM이 지문 콘텐츠 기준으로 문항 수 자체 결정
const min             = questionCount === 'content-driven' ? 1   : aiModule.questionMinCount;
const max             = questionCount === 'content-driven' ? 999 : aiModule.questionMaxCount;
const recommendedCount = questionCount === 'content-driven' ? 0
                       : calcRecommendedCount(passageLevel, min, max);
// calcRecommendedCount: level이 낮으면 min에 가깝게, 높으면 max에 가깝게
```

| questionCount 값 | min/max | recommendedCount | 사용 모듈 예 |
|---|---|---|---|
| `single` | 1 / 1 | 1 | PRD, SUM (서술형 1문항) |
| `multi` | 모듈별 설정 | 지문 레벨 기반 계산 | SWR(2~5), QAR(3~5), WRD(5~10) |
| `content-driven` | 1 / 999 (override) | 0 (LLM 자율) | CLR (문장 수 = 문항 수) |

### 3-4. LLM 프롬프트 구조 — 데이터가 콘텐츠를 만드는 방식

문항 생성 프롬프트는 **DB에 등록된 데이터**를 조합해서 자동 구성된다.
백오피스에서 모듈 설정만 바꿔도 생성되는 문항이 달라진다.

```
[지문]
제목: {texts.title}
{texts.content}

[교수법 지시문]
{ai_modules.pedagogy_instruction}
← 모듈의 학습 목적, 피드백 원칙, 학습자 상호작용 방식

[콘텐츠 생성 지시문]
{ai_modules.content_generation_instruction}  ← 있으면 이걸 사용
← 없으면 answerType 기반 fallback 규칙 사용
  (short-text: "핵심 단어 추출", audio-record: "낭독 문장 추출" 등)

[문항 수 결정]  ← questionCount = 'content-driven'이면 블록 자체를 생략
- 목표 문항 수: {recommendedCount}개 (지문 레벨 {level}/18 기반)
- 허용 범위: {min}~{max}개

[출력 형식] JSON만 출력 (answerType에 따라 스키마 달라짐)
```

#### 생성 전략 2종

| 전략 (`question_generation_strategy`) | 동작 | 적용 모듈 예 |
|---|---|---|
| `extract` | 지문에서 단어·문장·선택지를 LLM이 직접 추출. `source` 필드에 원문 발췌 포함 | WRD, SWR, CLR, QAR |
| `instruct` | 지문 맥락 기반 과제 지시문을 LLM이 생성. `answer`는 빈 문자열(holistic 채점) | PRD, SUM, PWR |

#### pedagogy_instruction vs content_generation_instruction 역할 분리

| 필드 | 역할 | 실제 사용처 |
|---|---|---|
| `pedagogy_instruction` | **피드백 원칙** — "어떻게 피드백할 것인가". LLM 피드백 API에서 system 프롬프트로 사용 | 피드백 생성, Orchestrator greeting 생성 |
| `content_generation_instruction` | **문항 생성 규칙** — "어떤 문항을 만들 것인가". 없으면 answerType 기반 기본값 적용 | 문항 생성 프롬프트 `[콘텐츠 생성 지시문]` 블록 |

#### 교재 데이터 주입 (textbookContext, 2026-05-21 추가)

studio에서 교재 PDF를 파싱한 경우, `textbooks` 테이블에 학습 어휘·표현·대화문이 저장된다.
해당 데이터가 있으면 프롬프트에 블록을 추가해 LLM이 교재의 학습 의도를 반영하도록 한다.

```
[지문]
{passageText}

[교재에서 선정된 학습 어휘]
[{ "word": "indigenous", "definition": "...", "example": "..." }, ...]

[교재에서 선정된 핵심 표현]
[{ "expression": "in the wake of", "meaning": "~의 결과로", ... }]

[문항 생성 지시문]
콘텐츠 생성 대상을 추출하기 전에 제공되는 교재 데이터에 추출하려는 콘텐츠가 있다면
해당 콘텐츠를 활용하여 아래 지시에 따라 필요한 나머지 콘텐츠를 생성하세요.

{content_generation_instruction}
```

교재 데이터가 없는 일반 과정은 이 블록이 없는 기존 경로를 그대로 사용한다.

### 3-5. 생성 결과 JSON 구조 (Gemini 응답)

```json
{
  "decidedCount": 4,
  "questions": [
    {
      "question_number": 1,
      "type": "sentence-write",
      "instruction": "다음 한국어 문장을 영어로 영작하세요.",
      "source": null,
      "text": "기후 변화는 전 세계 생태계에 영향을 미친다.",
      "options": null,
      "answer": "Climate change affects ecosystems around the world.",
      "hints": {
        "button": "Climate change...",
        "direct": "주어는 'Climate change', 동사는 'affects'를 사용하세요."
      }
    }
  ]
}
```

응답 파싱 후 `module_questions.createMany()`로 DB 저장.

### 3-6. Gemini 모델 + 장애 대응

```
Primary:  gemini-2.5-flash
Fallback: gemini-2.0-flash-001

503 (과부하): 2초 대기 후 primary 재시도
429 (쿼터):   즉시 fallback 전환
둘 다 실패:  InternalServerErrorException → JSONB fallback 문항 사용 (학습 진행 우선)
```

### 3-7. 캐시 무효화

관리자가 `content_generation_instruction` 등 모듈 설정을 변경하면
기존 캐시가 이전 설정으로 생성된 것이므로 무효화해야 한다.

```
DELETE /ai/generate-questions/:moduleId
  → module_questions WHERE module_id=:moduleId AND status='active'
  → UPDATE status='archived'  (soft-delete, 데이터 보존)

이후 학습 요청 시 캐시 미스 발생 → 새 설정으로 자동 재생성
```

---

## 관련 파일 위치 (tutoring 코드베이스)

| 역할 | 파일 |
|---|---|
| 어댑터 인터페이스 | `apps/web/src/lib/adapters/ModuleAdapter.ts` |
| 어댑터 레지스트리 | `apps/web/src/lib/adapters/index.ts` |
| GenericAdapter 구현 | `apps/web/src/lib/adapters/GenericAdapter.ts` |
| AiModuleData 타입 | `apps/web/src/lib/types/ai-module-data.ts` |
| ModuleData, QuestionData 타입 | `apps/web/src/lib/types/module.ts` |
| 모듈 콘텐츠 API 엔드포인트 | `apps/api/src/lessons/lessons.controller.ts` |
| Cache-Aside 구현 | `apps/api/src/lessons/lessons.service.ts: getModuleData()` |
| 문항 생성 서비스 | `apps/api/src/ai/question-generator.service.ts` |
| Gemini 호출 래퍼 | `apps/api/src/ai/ai.service.ts: generateQuestions()` |
| 문항 생성 프롬프트 빌더 | `apps/api/src/ai/prompts/question-generation.ts` |

---

## Speaking 구현 시 유의점

1. **테이블은 동일 Supabase 인스턴스 공유**: `ai_modules`, `module_questions`, `texts`는 tutoring과 같은 DB를 사용한다. speaking 전용 모듈은 `ai_modules.skill = 'speaking'`으로 구분.

2. **speaking 전용 모듈 코드**: speaking의 `ai_modules`에 현재 등록된 활성 모듈은 다음과 같다. 어댑터는 이 코드 기준으로 동작한다.

   | 단계 | 코드 | 모듈명 |
   |---|---|---|
   | LEARN | `LRN` | Learn & Study |
   | DRILL | `VLM` | Vocabulary Listening & Meaning |
   | DRILL | `SHD` | Shadowing Drill |
   | DRILL | `SFB` | Sentence Fill-in Blank |
   | DRILL | `SMK` | Sentence Making Drill |
   | APPLY | `RPB` | Role-Play Basic |
   | APPLY | `RPF` | Role-Play Free |
   | APPLY | `OMP` | One Minute Presentation |
   | APPLY | `FRT` | Free Talking |

   등록 이력 및 삭제된 구버전 모듈 상세는 [`operations/2026-06-05_스피킹-모듈-시퀀싱-데이터-등록-완료.md`](../../operations/2026-06-05_스피킹-모듈-시퀀싱-데이터-등록-완료.md) 참조.
