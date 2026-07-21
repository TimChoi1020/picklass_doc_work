# 과정 생성 시 회차별 지문 자동 생성 - 개발 계획

**작성일**: 2026-04-07  
**버전**: 1.0  
**관련 페이지**: `/course` (과정 생성 마법사 Step 2)

---

## 1. 현황 분석

### 현재 과정 생성 흐름

```
Step 1 (마법사 1/2)
  → 과정명, 목표, CEFR 레벨, 장르, 회차수, 단어수 입력
  → "다음" 클릭 시 AI 주제 생성 (POST /ai/generate-topics)

Step 2 (마법사 2/2)
  → 회차별 구성: "AI 생성" 또는 "기존 레슨" 선택
  → 각 회차별 주제(topic), 단어수, 장르, 수업모듈 설정
  → "완료" 클릭 시 과정 생성 (POST /courses)
```

### 문제점

**Step 2에서 "AI 생성" 모드를 선택한 회차는 주제(topic)만 존재하고, 실제 지문(text/passage)이 생성되지 않음.**

- `course_lessons.text_id`가 `null`로 저장됨
- "기존 레슨" 모드에서만 `texts` 테이블의 기존 지문을 `text_id`로 연결 가능
- AI가 주제만 제안하고 지문 본문은 생성하지 않는 구조

### 기존 지문 생성 인프라

이미 `apps/web/src/app/api/generate-text/route.ts`에 지문 생성 API가 구현되어 있음:

| 항목 | 내용 |
|------|------|
| 엔드포인트 | `POST /api/generate-text` (Next.js API Route) |
| AI 모델 | Gemini 2.5 Flash |
| 입력 파라미터 | `cefrLevel`, `genre`, `wordCount`, `topic` |
| 출력 | `title`, `content`, `wordCount`, `actualWordCount` |
| 사용처 | Class 페이지의 `CreatePassageModal.tsx` |

이 API는 Class 페이지에서 개별 지문 생성 시에만 사용되고 있으며, Course 생성 흐름에서는 호출되지 않음.

---

## 2. 구현 목표

과정 생성 "완료" 버튼 클릭 시, "AI 생성" 모드인 각 회차에 대해:

1. 해당 회차의 주제(topic) + 레벨 + 장르 + 단어수를 기반으로 지문(passage) 자동 생성
2. 생성된 지문을 `texts` 테이블에 저장
3. `course_lessons.text_id`에 생성된 지문 ID 연결

---

## 3. 구현 방안

### 방안 A: 프론트엔드에서 순차 생성 (권장)

기존 `POST /api/generate-text` API를 그대로 활용하여, 프론트엔드에서 "완료" 클릭 시 AI 회차들의 지문을 순차 생성 후 과정 생성 API 호출.

**장점:**
- 기존 지문 생성 API 재활용 (코드 변경 최소)
- 생성 진행률 UI 표시 용이
- 기존 로직에 영향 없음

**단점:**
- 회차 수가 많으면 시간이 오래 걸림 (회차당 2~5초)
- 네트워크 실패 시 일부만 생성될 수 있음

### 방안 B: 백엔드에서 일괄 생성

NestJS AI 모듈에 지문 생성 기능을 추가하고, 과정 생성 시 트랜잭션으로 일괄 처리.

**장점:**
- 트랜잭션으로 원자성 보장
- 서버 측 처리로 안정적

**단점:**
- 지문 생성 프롬프트가 현재 Next.js Route에만 있어 NestJS로 이관 필요
- 응답 시간이 매우 길어질 수 있음 (10회차 = 20~50초)
- 타임아웃 위험

### 선택: 방안 A (프론트엔드 순차 생성)

---

## 4. 상세 구현 계획

### STEP 1: 지문 저장 API 추가 (백엔드)

생성된 지문을 `texts` 테이블에 저장하는 엔드포인트가 필요함. 현재 `passages.controller.ts`에 POST 엔드포인트가 없음.

**파일**: `apps/api/src/passages/passages.controller.ts`

```
POST /passages
Body:
{
  title: string,
  content: string,
  level: string,        // CEFR 레벨
  category: string,     // 장르
  topic: string,        // 주제
  word_count: number,
  text_type: string     // 'A' (AI 생성)
}

Response:
{
  id: number,
  title: string,
  content: string,
  ...
}
```

**수정 대상 파일:**
- `apps/api/src/passages/passages.service.ts` — `create()` 메서드 추가
- `apps/api/src/passages/passages.controller.ts` — `POST /passages` 엔드포인트 추가
- `apps/api/src/passages/dto/passages.dto.ts` — `CreatePassageDto` 추가

### STEP 2: 프론트엔드 API 클라이언트 업데이트

**파일**: `apps/web/src/lib/api.ts`

`passagesApi`에 `create()` 메서드 추가:
```typescript
passagesApi: {
  // 기존 list()
  create: (data: CreatePassagePayload) => apiRequest<TextItem>('/passages', { method: 'POST', body: data }),
}
```

**파일**: `apps/web/src/hooks/use-courses.ts`

지문 생성 + 저장을 결합하는 `useGeneratePassage()` 훅 추가:
```typescript
// 1. POST /api/generate-text로 AI 지문 생성
// 2. POST /passages로 DB 저장
// 3. 저장된 text_id 반환
```

### STEP 3: 과정 생성 플로우 수정 (프론트엔드)

**파일**: `apps/web/src/app/course/page.tsx`

"완료" 버튼 클릭 핸들러 (`line ~1300`) 수정:

```
현재:
  완료 클릭 → POST /courses (text_id=null인 채로)

변경 후:
  완료 클릭
    → AI 회차들 식별 (topicsSource[idx] === 'ai' && !selectedLessonPerTopic[idx])
    → 각 AI 회차에 대해 순차적으로:
      1. POST /api/generate-text (지문 AI 생성)
      2. POST /passages (DB 저장, text_id 획득)
    → POST /courses (text_id 포함)
```

### STEP 4: 생성 진행 UI

**파일**: `apps/web/src/app/course/page.tsx`

과정 생성 중 진행 상태 표시:
- "지문 생성 중... (3/10)" 형태의 프로그레스 표시
- 버튼 비활성화 + 로딩 상태
- 개별 회차 생성 실패 시 해당 회차만 text_id=null로 처리 (부분 실패 허용)

---

## 5. 데이터 흐름 (변경 후)

```
[Step 2 완료 클릭]
     │
     ▼
[AI 생성 회차 필터링]
     │
     ▼ (각 AI 회차마다)
┌─────────────────────────────────┐
│ POST /api/generate-text         │
│  params: {                      │
│    cefrLevel: wizardLevel,      │
│    genre: topicsGenres[i],      │
│    wordCount: topicsWordCount[i]│
│    topic: topics[i]             │
│  }                              │
│  → { title, content }           │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ POST /passages                  │
│  params: {                      │
│    title, content,              │
│    level, category, topic,      │
│    word_count, text_type: 'A'   │
│  }                              │
│  → { id: text_id }              │
└──────────────┬──────────────────┘
               ▼
[lessons[i].text_id = 생성된 ID]
     │
     ▼ (모든 회차 완료 후)
┌─────────────────────────────────┐
│ POST /courses                   │
│  lessons: [                     │
│    { text_id: 123, ... },       │
│    { text_id: 124, ... },       │
│    { text_id: null, ... },  ← 기존 레슨 미선택 or 생성 실패 시
│  ]                              │
└─────────────────────────────────┘
```

---

## 6. 수정 대상 파일 요약

| 파일 | 변경 내용 |
|------|----------|
| `apps/api/src/passages/passages.service.ts` | `create()` 메서드 추가 |
| `apps/api/src/passages/passages.controller.ts` | `POST /passages` 엔드포인트 추가 |
| `apps/api/src/passages/dto/passages.dto.ts` | `CreatePassageDto` 클래스 추가 |
| `apps/web/src/lib/api.ts` | `passagesApi.create()` 추가 |
| `apps/web/src/hooks/use-courses.ts` | `useGeneratePassage()` 훅 추가 |
| `apps/web/src/app/course/page.tsx` | 완료 핸들러에 지문 생성 로직 + 진행 UI 추가 |

---

## 7. 영향 범위

- **기존 "기존 레슨" 모드**: 변경 없음 (기존처럼 texts 테이블에서 선택)
- **기존 Class 페이지 지문 생성**: 변경 없음 (독립 동작)
- **과정 상세 페이지 (`/course/[courseId]`)**: 변경 없음 (text_id가 있으면 지문 표시, 없으면 기존대로)
- **기존 AI 모듈 (NestJS)**: 변경 없음

---

## 8. 고려사항

1. **생성 시간**: 10회차 기준 약 20~50초 예상. 진행 UI 필수.
2. **부분 실패**: 일부 회차 지문 생성 실패 시 해당 회차만 `text_id=null`로 처리하고 과정은 정상 생성. 이후 과정 상세에서 개별 재생성 가능.
3. **단어수 기본값**: 회차별 단어수 미입력 시 Step 1의 기본 단어수 사용.
4. **장르 기본값**: 회차별 장르 미선택 시 Step 1의 기본 장르 사용.
5. **중복 생성 방지**: 완료 버튼 더블클릭 방지 (isPending 상태 활용).
