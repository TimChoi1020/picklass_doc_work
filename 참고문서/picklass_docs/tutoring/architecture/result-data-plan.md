# 학습 결과 데이터 체계화 계획

> 작성일: 2026-06-23  
> 대상: `tutoring.picklass.com` (module_histories, lesson_results) + `speaking.picklass.com` (SessionReport, TurnEvaluation, AudioFile)  
> 목적: 이용자 레벨 추정 · 역량 탐색 · 수업 추천 · 성장성 확인에 필요한 결과 데이터를 체계적으로 저장하기 위한 설계 계획

---

## 0. 이 문서가 다루는 범위

현재 학습이 끝날 때마다 `module_histories`와 `lesson_results` 두 테이블에 결과가 저장된다.  
그러나 저장되는 데이터의 구조와 범위에 한계가 있어 아래 네 가지 분석 기능을 구현하기 어렵다.

| 목표 기능 | 필요 데이터 | 현재 상태 |
|-----------|------------|-----------|
| 레벨 추정 | KPI 코드 기반 역량 수치, 시도 난이도 지표 | KPI label이 자유 텍스트 — 집계 불가 |
| 부족 역량 탐색 | 구조화된 KPI 카테고리, 오답 패턴, 힌트 의존도 | 힌트 사용·시도 횟수 저장 안 됨 |
| 수업 추천 | 약점 KPI 코드 목록, 추정 레벨 | 미저장 |
| 성장성 확인 | 시점별 KPI 추이, 레슨 완료 여부 | `is_complete` 플래그 없음 |

본 문서는 **현황 진단 → 단기·중기·장기 개선 방안** 순서로 계획을 정리한다.

---

## 1. 현황 진단

### 1-1. `module_histories` — 현재 저장 구조

모듈 하나가 완료될 때마다 1개 레코드가 쌓인다.

```
[api] apps/api/src/lessons/lessons.service.ts:483
      prisma.module_histories.create()
```

| 컬럼 | 타입 | 현재 저장값 | 한계 |
|------|------|------------|------|
| `score` | INT | 모듈 점수 0–100 | 점수 산출 근거 없음 |
| `kpis` | JSONB | `[{label, value, unit, description}]` | label이 자유 텍스트 → 집계/비교 불가 |
| `answers` | JSONB | `{questionId: answerText}` | 시도 횟수·오답 패턴 없음 |
| `chat_messages` | JSONB | AI ↔ 학습자 전체 대화 | 분석용이 아닌 로그 성격 |
| `started_at` | TIMESTAMPTZ | 옵션 — 실제로 비어있는 경우 있음 | 실소요시간 계산 불가 |
| `completed_at` | TIMESTAMPTZ | 완료 시각 | ✓ |

**프론트에 이미 있으나 API로 전송되지 않는 데이터:**

```typescript
// apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx
attemptCounts:     Record<questionId, number>   // 문항별 시도 횟수
hintButtonUsedIds: Set<string>                  // 힌트 버튼을 누른 문항 ID 목록
engagementLevel:   'high' | 'medium' | 'low'   // ModuleResult에 포함되나 API 전송 안 함
timeSpentMinutes:  number                       // estimatedMinutes 기준 (실측값 아님)
```

---

### 1-2. `lesson_results` — 현재 저장 구조

모듈 완료마다 누적 update, 중도 이탈 시도 저장. 레슨당 1개 레코드.

```
[api] apps/api/src/lessons/lessons.service.ts:577 (update), 591 (create)
```

| 컬럼 | 타입 | 현재 저장값 | 한계 |
|------|------|------------|------|
| `average_score` | DECIMAL(5,2) | 완료 모듈 평균 점수 | 분포·편차 없음 |
| `module_results` | JSONB | `[{moduleCode, score, kpis}]` | kpis 구조 동일한 한계 |
| `total_duration` | INT | 총 소요시간(초) | ✓ |
| `completed_at` | TIMESTAMPTZ | 마지막 저장 시각 | ✓ |
| _(없음)_ | — | — | **완료/중도이탈 구분 불가** |
| _(없음)_ | — | — | **추정 레벨 스냅샷 없음** |

---

## 2. 개선 방안

세 단계로 나눈다. 단기는 DDL 없이 적용 가능하고, 중기·장기는 스키마 변경이 필요하다.

---

### 2-1. 단기 — 기존 구조 안에서 즉시 적용

#### (A) KPI 구조에 `kpi_code` 추가

현재 KPI `label`은 자유 텍스트(`"예측 타당성 – 이해 전략"`)라 집계가 불가능하다.  
`kpi_code`를 추가하면 별도 DDL 없이 `code_items` 카탈로그와 연결할 수 있다.

**현재 저장 구조:**
```json
[
  { "label": "어휘 추론 정확도", "value": 80, "unit": "%", "description": "..." }
]
```

**개선 후 구조:**
```json
[
  {
    "kpi_code": "VOCAB_INFERENCE",
    "label": "어휘 추론 정확도",
    "value": 80,
    "unit": "%",
    "description": "..."
  }
]
```

**적용 위치:**
- 각 어댑터의 `buildKpis()` 반환값에 `kpi_code` 추가
  - `[web] apps/web/src/lib/adapters/` 각 어댑터 파일
- `KpiResult` 인터페이스에 `kpi_code` 필드 추가
  - `[web] apps/web/src/lib/agents/agent-types.ts:475`

**기대 효과:** `module_histories.kpis[].kpi_code`로 GROUP BY 집계 가능 → 역량별 시계열 분석 기반 확보

---

#### (B) `module_histories` API에 추가 데이터 전송

프론트에 이미 있는 데이터를 POST body에 포함시킨다. 컬럼 추가 없이 기존 컬럼 활용.

**전송 데이터 추가:**

| 데이터 | 출처 | 저장 방식 |
|--------|------|---------|
| `attemptCounts` | `useModuleOrchestrator` 반환값 | `answers` JSONB 옆에 별도 키 또는 신규 컬럼 |
| `hintUsedIds` | `hintButtonUsedIds` Set | 배열로 직렬화 |
| `engagementLevel` | `ModuleResult.engagementLevel` | 문자열 |
| `timeSpentSeconds` | `completed_at - started_at` 실측 | 정수(초) |

**`saveModuleHistory()` body 확장 (프론트):**
```typescript
// apps/web/src/lib/services/lessonService.ts
{
  moduleOrder,
  messages,
  answers,
  score,
  kpis,               // kpi_code 포함된 버전
  startedAt,
  // 추가
  attemptCounts,      // Record<questionId, number>
  hintUsedIds,        // string[]
  engagementLevel,    // 'high' | 'medium' | 'low'
  timeSpentSeconds,   // number
}
```

---

#### (C) `lesson_results` API에 `isComplete` 전송

현재 `POST /lessons/:id/complete`의 body에 `isComplete` 필드가 포함되어 있으나 DB에 저장되지 않는다.  
이를 저장하면 중도 이탈과 완전 완료를 구분할 수 있다.

```typescript
// LessonSession.tsx:97-104 — 이미 계산하여 전송 중
body: JSON.stringify({
  moduleResults: ...,
  totalDuration: ...,
  isComplete: results.length >= sequence.length,  // ← 현재 전송되나 DB 미저장
})
```

---

### 2-2. 중기 — 컬럼 추가 (DDL 필요)

단기 개선으로 수집 시작한 데이터를 전용 컬럼에 구조화하여 쿼리 성능을 확보한다.

#### `module_histories` 컬럼 추가

```sql
-- apps/api/prisma/manual-sql/YYYY-MM-DD_module_histories_result_columns.sql
BEGIN;

ALTER TABLE module_histories
  ADD COLUMN attempt_counts    JSONB,          -- {"q_id_1": 2, "q_id_2": 1}
  ADD COLUMN hint_used_ids     JSONB,          -- ["q_id_1", "q_id_3"]
  ADD COLUMN engagement_level  VARCHAR(10),    -- 'high' | 'medium' | 'low'
  ADD COLUMN time_spent_sec    INTEGER;        -- 실측 소요시간 (초)

COMMENT ON COLUMN module_histories.attempt_counts   IS '문항별 시도 횟수. 힌트 의존도·난이도 체감 지표';
COMMENT ON COLUMN module_histories.hint_used_ids    IS '힌트 버튼을 사용한 문항 ID 목록';
COMMENT ON COLUMN module_histories.engagement_level IS '참여도 (high/medium/low). 재학습 필요 시그널';
COMMENT ON COLUMN module_histories.time_spent_sec   IS '모듈 실측 소요시간. started_at 누락 시 fallback';

COMMIT;
```

#### `lesson_results` 컬럼 추가

speaking의 `SessionReport`를 대체하는 레슨 집계 레이어로 확장한다.
tutoring 레슨과 speaking 세션이 동일한 테이블로 집계된다.

```sql
-- apps/api/prisma/manual-sql/YYYY-MM-DD_lesson_results_columns.sql
BEGIN;

ALTER TABLE lesson_results
  ADD COLUMN is_complete       BOOLEAN  DEFAULT FALSE,  -- 전체 모듈 완료 여부
  ADD COLUMN estimated_level   SMALLINT,                -- 완료 시점 추정 레벨 (1–18)
  ADD COLUMN weak_kpi_codes    JSONB,                   -- ["VOCAB_INFERENCE", "PRONUNCIATION"]
  ADD COLUMN summary_feedback  TEXT,                    -- AI 종합 피드백
  ADD COLUMN strengths         JSONB,                   -- ["잘한 점 1", "잘한 점 2"]
  ADD COLUMN improvements      JSONB,                   -- ["개선 항목 1", "개선 항목 2"]
  ADD COLUMN next_steps        JSONB;                   -- ["다음 학습 제안 1", ...]

COMMENT ON COLUMN lesson_results.is_complete      IS '중도이탈(false) vs 완전완료(true) 구분';
COMMENT ON COLUMN lesson_results.estimated_level  IS '이 레슨 완료 시점의 레벨 추정값 스냅샷';
COMMENT ON COLUMN lesson_results.weak_kpi_codes   IS '이 레슨에서 하위 역량으로 분류된 KPI 코드 목록';
COMMENT ON COLUMN lesson_results.summary_feedback IS 'AI 종합 피드백. speaking SessionReport.summaryFeedback 대응';
COMMENT ON COLUMN lesson_results.strengths        IS '잘한 점 목록. speaking SessionReport.strengths 대응';
COMMENT ON COLUMN lesson_results.improvements     IS '개선 필요 항목 목록. speaking SessionReport.improvements 대응';
COMMENT ON COLUMN lesson_results.next_steps       IS '다음 학습 제안 목록. speaking SessionReport.nextSteps 대응';

COMMIT;
```

**speaking → `lesson_results` 필드 대응:**

| `SessionReport` | `lesson_results` (추가 후) |
|----------------|--------------------------|
| `overallScore` | `average_score` |
| `durationMinutes` | `total_duration` (초 단위) |
| `summaryFeedback` | `summary_feedback` |
| `strengths` | `strengths` |
| `improvements` | `improvements` |
| `nextSteps` | `next_steps` |
| `pronScore`, `gramScore`… | → `module_kpi_results` 행으로 저장 |

---

### 2-3. 장기 — 새 테이블 추가

레벨 추정과 수업 추천을 위한 독립 테이블. `lesson_results`의 누적이 충분해진 이후 도입.

#### `student_level_snapshots` — 레벨 추정 이력

```sql
CREATE TABLE student_level_snapshots (
  id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id       UUID        NOT NULL,
  estimated_level  SMALLINT    NOT NULL,               -- 1–18
  cefr_level       VARCHAR(2)  NOT NULL,               -- A1 A2 B1 B2 C1 C2
  confidence       SMALLINT    CHECK (confidence BETWEEN 0 AND 100),
  weak_kpi_codes   JSONB,                              -- 근거 KPI 코드 목록
  basis_lesson_ids JSONB,                              -- 근거 레슨 ID 목록 (최근 N개)
  snapshot_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT fk_student FOREIGN KEY (student_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_student_level_snapshots_student ON student_level_snapshots (student_id, snapshot_at DESC);

COMMENT ON TABLE student_level_snapshots IS
  '레벨 추정 결과 이력. 레슨 완료·일정 기간 경과·수동 요청 시 생성';
```

**생성 트리거 (구현 예정):**
- 레슨 완료 시 (`is_complete = true`) 자동 추정 → 저장
- 일정 기간 비학습 후 복귀 시
- 관리자/학습자 수동 재평가 요청 시

---

## 3. 스피킹 모듈 결과 데이터 구조

스피킹 모듈(`speaking.picklass.com`)은 tutoring과 **동일한 Supabase 인스턴스**를 사용하며,
발화 세션 결과를 아래 세 테이블에 독립적으로 저장한다.
tutoring의 텍스트 기반 KPI보다 훨씬 세밀한 발화·발음·문법 지표를 갖추고 있으며,
tutoring 결과 데이터 개선의 **참조 모델**로 활용할 수 있다.

### 3-1. `SessionReport` — 세션 전체 요약

발화 세션 1회의 종합 결과. `module_histories` / `lesson_results`에 해당하는 집계 레이어.

| 컬럼 | 타입 | 의미 | 분석 활용 |
|------|------|------|-----------|
| `id` | text | PK | — |
| `sessionId` | text | 세션 ID (FK) | — |
| `durationMinutes` | float | 세션 소요 시간(분) | 학습 지속성 지표 |
| `overallScore` | int | 종합 점수 (0–100) | 레벨 추정·성장성 추이 |
| `uvScore` | int | Utterance Variety — 표현 다양성 점수 | 어휘·표현 역량 |
| `pronScore` | int | 발음 종합 점수 | 발음 역량 |
| `wcpmScore` | int | Words/Chars Per Minute — 발화 속도 | 유창성 역량 |
| `pragScore` | int | Pragmatics — 맥락 적절성 점수 | 의사소통 역량 |
| `gramScore` | int | Grammar — 문법 정확도 점수 | 문법 역량 |
| `summaryFeedback` | text | AI 종합 피드백 | — |
| `strengths` | text[] | 잘한 점 목록 | — |
| `improvements` | text[] | 개선 필요 항목 목록 | 약점 역량 시그널 |
| `nextSteps` | text[] | 다음 학습 제안 목록 | 수업 추천 시드 |
| `detailMetrics` | jsonb | 추가 세부 지표 (확장용) | — |
| `createdAt` | timestamp | 생성 시각 | 시계열 |

**역량 코드 매핑 (tutoring KPI 연계 시):**

| 스피킹 점수 | 대응 역량 코드 (제안) |
|------------|-------------------|
| `uvScore` | `VOCAB_VARIETY` |
| `pronScore` | `PRONUNCIATION` |
| `wcpmScore` | `FLUENCY_SPEED` |
| `pragScore` | `PRAGMATICS` |
| `gramScore` | `GRAMMAR` |

---

### 3-2. `TurnEvaluation` — 발화 턴 단위 평가

AI와 학습자 간 **한 번의 발화 교환(turn)** 마다 1개 레코드.
tutoring의 문항 단위 `answers`보다 훨씬 세밀한 발화 분석 데이터를 담는다.

**발화 기본 정보:**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `id` | text | PK |
| `turnId` | text | 턴 ID (FK) |
| `wordCount` | int | 발화 단어 수 |
| `charCount` | int | 발화 글자 수 |
| `durationMs` | int | 발화 길이(ms) |

**발음 평가 (Azure Pronunciation Assessment):**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `pronAccuracy` | float | 발음 정확도 (0–100) |
| `pronFluency` | float | 유창성 (0–100) |
| `pronCompleteness` | float | 완전성 — 모든 단어 발화 비율 |
| `pronProsody` | float | 운율·억양 (0–100) |
| `pronScore` | float | 발음 종합 점수 |
| `pronErrorWords` | int | 발음 오류 단어 수 |

**어법·화용 평가:**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `pragRelevance` | float | 맥락 관련성 (0–1) |
| `pragContext` | float | 맥락 이해도 (0–1) |
| `grammarCorrect` | bool | 문법 정확 여부 |
| `grammarErrorWords` | int | 문법 오류 단어 수 |
| `grammarErrors` | jsonb | 오류 상세 `[{word, type, suggestion}]` |

**교정 정보 (AI가 교정 제공 시):**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `correctionAction` | text | 교정 행동 유형 |
| `correctionArea` | text | 교정 영역 (발음/문법/어휘 등) |
| `correctionCategory` | text | 교정 카테고리 세부 분류 |
| `correctionOriginal` | text | 학습자 원문 |
| `correctionCorrected` | text | 교정된 문장 |
| `correctionKoExplain` | text | 한국어 교정 설명 |
| `correctionPriority` | int | 교정 우선순위 (낮을수록 중요) |

**반복 연습 정보:**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `repeatAttempts` | int | 반복 시도 횟수 |
| `repeatMatched` | bool | 목표 패턴 달성 여부 |
| `repeatTranscript` | text | 반복 발화 텍스트 |

**기타:**

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `hints` | jsonb | 제공된 힌트 목록 |
| `studentAskedQuestion` | bool | 학습자가 질문했는지 여부 |
| `utteranceLevel` | text | 발화 수준 평가 (beginner / intermediate 등) |
| `attemptedPriorPattern` | bool | 이전 학습 패턴 적용 시도 여부 |

---

### 3-3. `AudioFile` — 오디오 파일 메타데이터

발화 턴당 AI 모델 음성과 학습자 녹음 파일의 S3 참조 정보.

| 컬럼 | 타입 | 의미 |
|------|------|------|
| `id` | text | PK |
| `turnId` | text | TurnEvaluation FK |
| `kind` | text | `'model'` (AI 음성) \| `'student'` (학습자 녹음) |
| `s3Bucket` | text | S3 버킷명 |
| `s3Key` | text | S3 오브젝트 키 |
| `format` | text | 오디오 포맷 (wav, mp3 등) |
| `sizeBytes` | int | 파일 크기 |
| `durationMs` | int | 오디오 길이(ms) |
| `createdAt` | timestamp | 생성 시각 |

---

### 3-4. tutoring vs speaking 결과 데이터 비교 및 통합 방향

| 관점 | tutoring (현재) | speaking (현재) | 통합 후 |
|------|----------------|----------------|--------|
| **레슨/세션 집계** | `lesson_results` | `SessionReport` | `lesson_results` (통합) |
| **KPI 집계** | `module_histories.kpis` JSON | `SessionReport` 고정 컬럼 | `module_kpi_results` (통합) |
| **원시 평가 데이터** | `module_histories` | `TurnEvaluation` | 각자 유지 (성격 상이) |
| **AI 피드백 텍스트** | 없음 | `SessionReport.summaryFeedback` | `lesson_results.summary_feedback` (통합) |
| **강점·개선점** | 없음 | `SessionReport.strengths/improvements` | `lesson_results.strengths/improvements` (통합) |
| **집계 가능성** | JSON → 불가 | 고정 컬럼 → 즉시 가능 | `kpi_code` 행 → 즉시 가능 (통합) |
| **오디오·교정 원시 데이터** | 없음 | `TurnEvaluation`, `AudioFile` | speaking 전용 유지 |

**핵심 판단:**
- `TurnEvaluation`: speaking 고유 원시 발화 데이터 — 대체 불가, 유지
- `SessionReport`: 집계 역할 → `lesson_results` + `module_kpi_results`로 흡수 가능
- `module_kpi_results`: tutoring·speaking 양측 KPI를 `kpi_code`로 통합하는 단일 집계소

> 자세한 통합 설계는 §4 참고.

---

## 4. `module_kpi_results` — 두 서비스 KPI 통합 집계소

### 4-1. 설계 원칙

speaking의 `TurnEvaluation`이 **1턴 = 1행**으로 평가 단위를 정규화한 것처럼,
tutoring은 **1 KPI 결과 = 1행**으로 정규화한다.
그리고 speaking의 `SessionReport` 점수 컬럼들도 이 테이블에 미러링하여,
**tutoring과 speaking의 KPI가 동일한 테이블에서 통합 집계**된다.

| | Speaking | Tutoring | 통합 후 |
|--|----------|---------|--------|
| 역량 식별 | 컬럼명 (`pronScore`…) | `kpi_code` 컬럼값 | `kpi_code` 컬럼값 |
| 역량 정의 | 시스템 고정 5종 | 백오피스 KPI 카탈로그 | 백오피스 KPI 카탈로그 |
| 행 생성 시점 | (미러링 파이프라인) | 모듈 완료 시 KPI 수만큼 | 동일 |
| 행 수 (예시) | 세션당 5행 (5종 점수) | 모듈당 1~5행 | 동일 테이블에 혼재 |

```
[tutoring 흐름]
  adapter.buildKpis()  →  module_kpi_results.createMany()
  module_history_id | kpi_code          | kpi_value | source
   mh001            | VOCAB_INFERENCE   |    80     | tutoring
   mh001            | PREDICTION_VALID  |    70     | tutoring

[speaking 흐름 — 미러링 파이프라인]
  SessionReport 저장 완료 후  →  module_kpi_results INSERT
  session_id (as ref)        | kpi_code          | kpi_value | source
   (session_report_id)       | PRONUNCIATION     |    85     | speaking
   (session_report_id)       | GRAMMAR           |    72     | speaking
   (session_report_id)       | PRAGMATICS        |    90     | speaking

→ 두 서비스 구분 없이 동일 쿼리로 KPI 집계 가능
   SELECT kpi_code, AVG(kpi_value)
   FROM module_kpi_results
   WHERE student_id = ?
   GROUP BY kpi_code;
```

---

### 4-2. `module_kpi_results` 테이블 DDL

```sql
-- apps/api/prisma/manual-sql/YYYY-MM-DD_module_kpi_results.sql
BEGIN;

CREATE TABLE module_kpi_results (
  id                UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  module_history_id UUID,                        -- tutoring: module_histories FK / speaking: NULL
  session_report_id TEXT,                        -- speaking: SessionReport FK / tutoring: NULL
  student_id        UUID          NOT NULL,
  lesson_id         UUID,                        -- tutoring: course_lessons FK / speaking: NULL
  module_code       VARCHAR(10),                 -- tutoring 전용 (speaking은 NULL)
  kpi_code          VARCHAR(50)   NOT NULL,      -- 백오피스 KPI 카탈로그 코드
  kpi_value         DECIMAL(6,2)  NOT NULL,      -- 0–100 또는 절대값 (WPM 등)
  kpi_unit          VARCHAR(20),                 -- '%', 'WPM', '점'
  source            VARCHAR(20)   NOT NULL,      -- 'tutoring' | 'speaking'
  attempt_count     SMALLINT,                    -- tutoring: 모듈 내 총 시도 횟수
  hint_used         BOOLEAN       DEFAULT FALSE, -- tutoring: 힌트 사용 여부
  engagement_level  VARCHAR(10),                 -- tutoring: 'high' | 'medium' | 'low'
  completed_at      TIMESTAMPTZ   NOT NULL
);

COMMENT ON TABLE module_kpi_results IS
  'tutoring·speaking 두 서비스의 KPI 통합 집계소. '
  '1 KPI 결과 = 1행. source 컬럼으로 서비스 구분. 백오피스 KPI 카탈로그의 kpi_code로 연결.';

COMMENT ON COLUMN module_kpi_results.source           IS 'tutoring | speaking — 행 출처 서비스';
COMMENT ON COLUMN module_kpi_results.module_history_id IS 'tutoring 전용 FK. speaking 행은 NULL';
COMMENT ON COLUMN module_kpi_results.session_report_id IS 'speaking 전용 FK. tutoring 행은 NULL';
COMMENT ON COLUMN module_kpi_results.kpi_code          IS '백오피스 KPI 카탈로그 코드. 나만의 수업 목표와 동일 식별자';
COMMENT ON COLUMN module_kpi_results.attempt_count     IS 'tutoring 전용. 문항 시도 횟수. 난이도 체감 지표';
COMMENT ON COLUMN module_kpi_results.hint_used         IS 'tutoring 전용. 힌트 버튼 사용 여부';
COMMENT ON COLUMN module_kpi_results.engagement_level  IS 'tutoring 전용. 모듈 참여도. 재학습 필요 시그널';

-- 집계 쿼리 전용 인덱스
CREATE INDEX idx_mkr_student_kpi ON module_kpi_results (student_id, kpi_code, completed_at DESC);
CREATE INDEX idx_mkr_source_kpi  ON module_kpi_results (source,     kpi_code, kpi_value);
CREATE INDEX idx_mkr_lesson_kpi  ON module_kpi_results (lesson_id,  kpi_code);

COMMIT;
```

---

### 4-3. 저장 흐름

#### (A) tutoring — 모듈 완료 시

```
모듈 완료
  → adapter.buildKpis()  ← KPI 배열 (1~5개, kpi_code 포함)
      ↓
  module_histories.create()         ← 대화 기록·원시 데이터 (기존 유지)
  module_kpi_results.createMany()   ← KPI 수만큼, source='tutoring'
```

```typescript
// apps/api/src/lessons/lessons.service.ts — saveModuleHistory 확장
const record = await this.prisma.module_histories.create({ data: { ... } });

await this.prisma.module_kpi_results.createMany({
  data: body.kpis.map((kpi) => ({
    module_history_id: record.id,
    student_id:        studentId,
    lesson_id:         lessonId,
    module_code:       moduleCode,
    kpi_code:          kpi.kpi_code,
    kpi_value:         kpi.value,
    kpi_unit:          kpi.unit,
    source:            'tutoring',
    attempt_count:     body.attemptCount ?? null,
    hint_used:         (body.hintUsedIds?.length ?? 0) > 0,
    engagement_level:  body.engagementLevel ?? null,
    completed_at:      new Date(),
  })),
});
```

#### (B) speaking — 세션 완료 후 미러링 파이프라인

`SessionReport` 저장 완료 직후 점수 컬럼들을 `module_kpi_results`에 INSERT한다.

```typescript
// apps/api/src/ [speaking] session.service.ts — SessionReport 저장 후 실행
const SPEAKING_KPI_MAP = [
  { kpi_code: 'PRONUNCIATION',  getValue: (r) => r.pronScore  },
  { kpi_code: 'GRAMMAR',        getValue: (r) => r.gramScore  },
  { kpi_code: 'PRAGMATICS',     getValue: (r) => r.pragScore  },
  { kpi_code: 'VOCAB_VARIETY',  getValue: (r) => r.uvScore    },
  { kpi_code: 'FLUENCY_SPEED',  getValue: (r) => r.wcpmScore  },
];

await prisma.module_kpi_results.createMany({
  data: SPEAKING_KPI_MAP.map(({ kpi_code, getValue }) => ({
    session_report_id: sessionReport.id,
    student_id:        studentId,
    kpi_code,
    kpi_value:         getValue(sessionReport),
    source:            'speaking',
    completed_at:      sessionReport.createdAt,
  })),
});
```

---

### 4-4. 가능해지는 통계 쿼리

```sql
-- 개인별 KPI 월간 추이 (성장성 확인)
SELECT kpi_code,
       DATE_TRUNC('month', completed_at) AS month,
       AVG(kpi_value)                    AS avg_value,
       COUNT(*)                          AS sample_count
FROM module_kpi_results
WHERE student_id = :student_id
GROUP BY kpi_code, month
ORDER BY kpi_code, month;

-- 그룹별(반·기관) KPI 분포 통계
SELECT kpi_code,
       AVG(kpi_value)    AS avg,
       STDDEV(kpi_value) AS stddev,
       MIN(kpi_value)    AS min,
       MAX(kpi_value)    AS max,
       COUNT(*)          AS count
FROM module_kpi_results
WHERE lesson_id IN (
  SELECT id FROM course_lessons WHERE course_id = :course_id
)
GROUP BY kpi_code;

-- 개인 약점 역량 추출 (최근 3개월, 60점 미만)
SELECT kpi_code, AVG(kpi_value) AS avg_score
FROM module_kpi_results
WHERE student_id = :student_id
  AND completed_at > NOW() - INTERVAL '3 months'
GROUP BY kpi_code
HAVING AVG(kpi_value) < 60
ORDER BY avg_score;

-- 힌트 의존도 높은 역량 (취약 신호)
SELECT kpi_code,
       ROUND(COUNT(*) FILTER (WHERE hint_used)::numeric / COUNT(*) * 100) AS hint_rate_pct,
       AVG(kpi_value) AS avg_score
FROM module_kpi_results
WHERE student_id = :student_id
GROUP BY kpi_code
ORDER BY hint_rate_pct DESC;
```

---

### 4-5. `lesson_results` — `SessionReport` 역할 강화

`SessionReport`가 세션 집계를 담당하듯 `lesson_results`는 레슨 집계를 담당한다.
§2-2 컬럼 추가에 더해, `module_kpi_results`에서 집계한 값을 스냅샷으로 저장한다.

```
lesson_results.weak_kpi_codes  ←  module_kpi_results GROUP BY kpi_code HAVING AVG < 60
lesson_results.estimated_level ←  level-estimate.service.ts (kpi 집계 기반 추정)
lesson_results.is_complete     ←  전체 모듈 완료 여부
```

---

## 5. 목표별 데이터 활용 경로 (개선 후)

### 레벨 추정

```
[tutoring]
module_kpi_results (kpi_code, kpi_value, attempt_count, hint_used)
  + lesson_results.is_complete (완료/이탈 가중치)

[speaking]
SessionReport.overallScore + pronScore + gramScore + uvScore + pragScore
  + TurnEvaluation.pronAccuracy, grammarCorrect, utteranceLevel
    ↓
level-estimate.service.ts (기존 로직 강화 — 양측 데이터 통합)
    ↓
student_level_snapshots.create()   ← 장기
```

### 부족 역량 탐색

```
[tutoring]
module_kpi_results (student_id, kpi_code, kpi_value, hint_used)  ← 최근 N개 레슨
  → GROUP BY kpi_code → 역량별 평균 직접 집계
  → HAVING AVG < 60 → 약점 역량 추출
  → hint_used 비율 높은 역량 → 의존도 분석

[speaking]
SessionReport.pronScore / gramScore / pragScore 등  (최근 N개 세션)
  → 컬럼 기반 집계 → 동일 방식
  → TurnEvaluation.grammarErrors, correctionArea → 반복 오류 패턴 추출
    ↓
lesson_results.weak_kpi_codes / student_level_snapshots.weak_kpi_codes 저장  ← 중기·장기
```

### 수업 추천

```
student_level_snapshots.estimated_level (현재 레벨)
  + student_level_snapshots.weak_kpi_codes (약점 역량 — tutoring + speaking 통합)
  + code_items (KPI 카탈로그 → 모듈 코드 매핑)
    ↓
레벨에 맞는 지문 + 약점 KPI 강화 모듈 우선 배치
(예: PRONUNCIATION 낮음 → SNR/SHD 모듈, GRAMMAR 낮음 → GMR 모듈)
```

### 성장성 확인

```
[tutoring]
lesson_results (student_id, completed_at, average_score, is_complete)
  → 레슨 단위 점수 시계열
module_kpi_results (kpi_code, kpi_value, completed_at)
  → KPI별 역량 변화 추이 (인덱스 기반 — JSON 파싱 불필요)

[speaking]
SessionReport (createdAt, overallScore, pronScore, gramScore …)
  → 발음·문법·유창성 각 영역 시계열
TurnEvaluation (repeatAttempts, repeatMatched, pronScore 집계)
  → 세션 내 반복 달성률 추이

[통합]
student_level_snapshots (student_id, snapshot_at, estimated_level)
  → 레벨 변화 곡선 (tutoring + speaking 복합 추정)
```

---

## 6. 구현 우선순위

| 순위 | 작업 | 분류 | 난이도 | 효과 | 상태 |
|------|------|------|--------|------|------|
| ★★★ | KPI에 `kpi_code` 추가 (§2-1-A) | 단기 | 낮음 | 높음 — `module_kpi_results` 전제조건 | ✅ 2026-06-29 |
| ★★★ | `lesson_results.is_complete` 저장 (§2-1-C) | 단기 | 낮음 | 중간 | ✅ 2026-06-29 |
| ★★★ | `module_kpi_results` DDL + tutoring 저장 로직 (§4-2, 4-3A) | 중기 | 중간 | 높음 — 통계 인프라 핵심 | ✅ 2026-06-29 / LLM KPI 실제값+description 저장 2026-07-02 |
| ★★★ | `lesson_results` 피드백 컬럼 추가 DDL (§2-2) | 중기 | 낮음 | 높음 — SessionReport 대체 | ✅ 2026-06-29 (DDL만, 저장 로직 미구현) |
| ★★☆ | `attemptCounts`, `hintUsedIds` API 전송 (§2-1-B) | 단기 | 낮음 | 높음 | ✅ 2026-06-29 |
| ★★☆ | speaking → `module_kpi_results` 미러링 파이프라인 (§4-3B) | 중기 | 중간 | 높음 — 통합 집계 완성 | ⬜ 미구현 |
| ★★☆ | `module_histories` 컬럼 추가 DDL (§2-2) | 중기 | 중간 | 중간 | ✅ 2026-06-29 |
| ★☆☆ | `student_level_snapshots` 테이블 (§2-3) | 장기 | 높음 | 높음 — 레슨 누적 후 도입 | ⬜ 미구현 |

---

## 7. 구현 시 주의사항

1. **`kpi_code` 선결**: `module_kpi_results` 투입 전 백오피스 KPI 카탈로그의 코드값이 확정되어야 한다. 코드 없이 행을 쌓으면 집계 의미가 없다.
2. **기존 레코드 호환**: `kpi_code`가 없는 기존 `module_histories.kpis` 레코드는 `null` 처리. 집계 쿼리에서 `WHERE kpi_code IS NOT NULL` 필터 사용.
3. **`module_histories` 역할 분리**: 대화 로그·원시 데이터 보관소로 유지. 통계는 `module_kpi_results`만 조회 — 두 테이블 동시 집계 금지.
4. **Prisma 마이그레이션 금지**: DDL은 모두 `apps/api/prisma/manual-sql/` 파일로 작성, Supabase에서 직접 실행. (`CLAUDE.md §15` 참고)
5. **tutoring/speaking 공유 테이블 없음**: `module_kpi_results`, `module_histories`, `lesson_results`는 tutoring 전용. `users`, `code_*` 변경 시만 speaking 팀 정렬 필요.
6. **`started_at` 신뢰도**: 현재 프론트에서 옵션 전송, 누락이 많다. `module_kpi_results.completed_at` 기준 소요시간으로 대체 계산.

---

## 8. 관련 파일

| 파일 | 수정 내용 | 단계 | 상태 |
|------|----------|------|------|
| `apps/web/src/lib/agents/agent-types.ts` | `KpiResult`에 `kpi_code?` 추가; `ModuleResult`에 `attemptCounts?`, `hintUsedIds?`, `timeSpentSeconds?` 추가 | 단기 | ✅ |
| `apps/web/src/lib/adapters/GenericAdapter.ts` | `buildKpis()` 반환 객체에 `kpi_code: kpiCode` 추가 | 단기 | ✅ |
| `apps/web/src/lib/services/lessonService.ts` | `saveModuleHistory()` extra 파라미터 확장 (engagementLevel, attemptCounts, hintUsedIds, timeSpentSeconds) | 단기 | ✅ |
| `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` | `moduleStartedAt` ref 추가; 3곳 `onModuleComplete` 호출에 extra 필드 전달; `handleModuleComplete → saveModuleHistory` 확장; `lesson_results` 저장 시점을 `handleModuleSave` 내 LLM 응답 후로 이동; 이후 `saveLessonResult`·`lastSavedResultsRef` 제거 → 백엔드 트랜잭션으로 이관 | 단기 | ✅ |
| `apps/api/src/lessons/lessons.controller.ts` | `saveModuleHistory` / `completeLessonResult` DTO 확장 (`kpi_code?`, `engagementLevel?`, `attemptCounts?`, `hintUsedIds?`, `timeSpentSeconds?`, `isComplete?`) | 단기 | ✅ |
| `apps/api/src/lessons/lessons.service.ts` | `saveModuleHistory` 전체 재구현: `$transaction` 래핑 (module_histories + module_kpi_results + lesson_results 원자적 저장); LLM 값 평가(pending KPI); **kpi_code 있는 모든 KPI(측정+LLM)** description 일괄 생성(`generateLlmKpiDescriptions`); `module_histories.kpis` 최종 UPDATE; LLM 값 변경 시 `lesson_results` 재집계 | 단기·중기 | ✅ |
| `apps/api/prisma/schema.prisma` | `module_histories` / `lesson_results` / `module_kpi_results` 필드 반영; `course_lessons ↔ lesson_results`, `course_lessons ↔ module_kpi_results`, `module_histories ↔ module_kpi_results` relation 추가; `directUrl` 추가 | 중기 | ✅ |
| `apps/api/prisma/manual-sql/2026-06-29_lesson_results_is_complete.sql` | `lesson_results.is_complete BOOLEAN DEFAULT FALSE` 추가 | 단기 | ✅ |
| `apps/api/prisma/manual-sql/2026-06-29_module_histories_result_columns.sql` | `attempt_counts`, `hint_used_ids`, `engagement_level`, `time_spent_sec` 추가 | 중기 | ✅ |
| `apps/api/prisma/manual-sql/2026-06-29_lesson_results_feedback_columns.sql` | `estimated_level`, `weak_kpi_codes`, `summary_feedback`, `strengths`, `improvements`, `next_steps` 추가 | 중기 | ✅ (DDL만) |
| `apps/api/prisma/manual-sql/2026-06-29_module_kpi_results.sql` | `module_kpi_results` 테이블 + 인덱스 3개 생성 | 중기 | ✅ |

---

## 9. 구현 이력

### 2026-06-29 — 단기·중기 구현 완료

**단기 A: `kpi_code` 추가**
- `KpiResult` 인터페이스에 `kpi_code?: string` 추가
- `GenericAdapter.buildKpis()`에서 `kpi_code: kpiCode` 반환
- controller / service / lessonService DTO 타입 동기화
- JSONB `kpis` 컬럼에 자동 저장 (DDL 없음)

**단기 B: 추가 메타데이터 전송 파이프라인**
- `ModuleResult`에 `attemptCounts?`, `hintUsedIds?`, `timeSpentSeconds?` 추가
- `ModuleRunnerInner`에 `moduleStartedAt` ref 추가 → 실측 소요시간 계산
- `onModuleComplete` 3곳에 extra 필드 전달 (replanSignal / handleModuleNext / embedded)
- `handleModuleComplete → saveModuleHistory` 경로로 백엔드까지 전달

**단기 C: `is_complete` 저장**
- DDL: `lesson_results.is_complete BOOLEAN NOT NULL DEFAULT FALSE`
- `saveLessonResult` — update/create 양쪽에 `is_complete` 저장
- `completeLessonResult` DTO에 `isComplete?: boolean` 수용

**중기: DB 컬럼 추가 + `module_kpi_results` 구축**
- DDL 3개 Supabase 적용 → `prisma:pull` → Prisma schema 갱신
- `module_histories`: `attempt_counts`, `hint_used_ids`, `engagement_level`, `time_spent_sec` 저장
- `module_kpi_results.createMany()`: kpi_code 있는 KPI마다 tutoring 집계 행 생성
- `lesson_results`: 피드백 컬럼 DDL 적용 완료 (저장 로직은 speaking 연동 시 추가)
- schema.prisma relation 설정: `lesson_results.course_lesson`, `module_kpi_results.module_history`, `module_kpi_results.lesson`
- `schema.prisma`에 `directUrl` 추가 (session pooler port 5432 — introspection용)

**미구현 (다음 단계)**
- `lesson_results` 피드백 컬럼 저장 로직 (speaking SessionReport 연동 시 구현)
- speaking → `module_kpi_results` 미러링 파이프라인 (`source='speaking'`)
- `student_level_snapshots` 테이블 (레슨 누적 후 장기 도입)

---

### 2026-07-02 — LLM KPI 실제값 저장 + AI 패널 표시 수정

**문제**: LLM KPI가 `module_histories.kpis`에 0으로 영구 저장되고, AI 패널에도 0으로 표시됨.
초기 구현의 fire-and-forget 방식이 `module_histories`를 업데이트하지 않고,
프론트도 API 응답을 활용하지 않아서 발생.

**해결 (방안 A — 동기 처리)**:
- `saveModuleHistory` 내에서 LLM KPI Gemini 평가를 동기 완료
- `module_kpi_results` UPDATE + `module_histories.kpis` UPDATE → DB 양쪽 실제값 저장
- 응답에 `kpis: finalKpis` 포함 → 프론트 `setSavedKpis(finalKpis)` → FeedbackPanel 업데이트
- LLM 평가 중 "평가 중…" 표시, 완료 후 실제 점수 표시

**변경 파일**:
- `apps/api/src/lessons/lessons.service.ts` — `evaluateLlmKpisAsync` 제거, 동기 평가 + `generateLlmKpiDescriptions()` 호출 + description 포함 `finalKpis` 빌드 + `module_histories` UPDATE + 응답에 `kpis` 포함
- `apps/api/src/ai/ai.service.ts` — `generateLlmKpiDescriptions()` 추가 (평가된 LLM KPI 일괄 description 생성, 실패 시 `{}` 반환)
- `apps/api/src/ai/prompts/kpi.ts` — `buildKpiEvalPrompt` 피드백 톤 존댓말로 통일
- `apps/web/src/lib/services/lessonService.ts` — 반환 타입 `void → KpiResult[] | null`
- `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` — `savedKpis` state, `displayModuleResult`, `handleModuleNext` 업데이트
- `apps/web/src/app/modules/[lessonId]/_components/panels.tsx` — `kpi.pending` 시 "평가 중…" 표시

**검증**: `pnpm typecheck` — @tutoring/types · @tutoring/api · @tutoring/web 3개 모두 통과 ✅

---

### 2026-07-02 — `lesson_results` 저장 시점 수정 (LLM KPI 값 누락 방지)

**문제**: `lesson_results`가 LLM 평가 완료 전에 저장되어 LLM KPI 값이 누락됨.

- **케이스 1/3 (replanSignal / embedded)**: `onModuleSave`와 `onModuleComplete`가 동시 호출 → `handleModuleComplete`에서 `saveLessonResult`가 즉시 실행되어 LLM 평가 전 값이 저장됨.
- **케이스 2 (정상)**: "다음" 클릭 시점이 LLM 응답보다 빠르면 마찬가지로 누락.
- **handleExit 역방향 덮어쓰기**: `handleModuleSave`가 현재 모듈을 포함해 저장 완료 후 `handleExit`가 그보다 적은 모듈만 포함한 데이터로 덮어쓰는 버그.

**해결 (1차 — 프론트 타이밍 조정)**:

1. `lastSavedResultsRef = React.useRef<ModuleResult[]>([])` 추가 — 마지막으로 저장한 results 추적.
2. `saveLessonResult`: 호출 시 `lastSavedResultsRef.current = results` 선행 갱신.
3. `handleModuleSave`: `await saveModuleHistory()` 전에 `prevResults = completedResultsRef.current` 캡처 → `finalKpis` 응답 후 `saveLessonResult([...prevResults, resultWithFinalKpis])` 호출. LLM 값이 반드시 포함된 시점에 저장.
4. `handleModuleComplete`: `saveLessonResult()` 호출 제거. 내비게이션 전용으로 단순화.
5. `handleExit`: `completedResults.length > lastSavedResultsRef.current.length` 조건 — `handleModuleSave`가 이미 더 많은 모듈을 저장한 경우 덮어쓰지 않음.

**변경 파일**:
- `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` — `lastSavedResultsRef` 추가, `handleModuleSave` 확장, `handleModuleComplete` 단순화, `handleExit` 조건 변경

**검증**: `pnpm typecheck` — @tutoring/web 통과 ✅

---

### 2026-07-02 — 세 테이블 원자적 저장 (백엔드 `$transaction`)

**배경**: 위 1차 수정 이후에도 `module_histories.create` ↔ `module_kpi_results.createMany` ↔ `lesson_results.upsert` 사이에 트랜잭션이 없어 중간 단계에서 실패하면 불일치가 발생할 수 있었다. 또한 프론트의 `saveLessonResult` 계층이 불필요하게 복잡했다.

**해결 — 백엔드 트랜잭션으로 완전 이관**:

```
saveModuleHistory()
  └─ $transaction(async tx => {
       1. tx.module_histories.create()           ← 항상
       2. tx.module_kpi_results.createMany()     ← kpi_code 있는 KPI만 (pending 값=0 포함)
       3. tx.module_histories.findMany()         ← 이 레슨의 전체 이력 집계
          tx.course_lessons.findUnique()         ← skill_modules 길이 → totalModules
       4. tx.lesson_results.update / create()    ← 집계값으로 upsert
     })
     ↓ (트랜잭션 외부 — kpi_code 있는 KPI가 하나라도 있으면)
     지문·답변 내용 조회 (passageContent, answersText)
     pending KPI (LLM형) → 값 평가 → module_kpi_results UPDATE
     kpi_code 있는 전체 KPI (측정+LLM) → description 일괄 생성 (generateLlmKpiDescriptions)
     module_histories UPDATE (값 + description 통합 finalKpis)
     LLM 값 변경이 있었으면 → lesson_results 재집계 UPDATE
```

**`lesson_results` 집계 방식 (백엔드 DB 집계)**:

| 컬럼 | 계산 방법 |
|------|---------|
| `module_results` | `module_histories.findMany()` → `{ moduleCode, score, kpis }` 배열 |
| `average_score` | 전체 이력 `score` 평균 |
| `total_duration` | 전체 이력 `time_spent_sec` 합산 |
| `is_complete` | `moduleResults.length >= lesson.skill_modules.length` |

**Prisma interactive transaction의 read-your-own-writes**: 같은 트랜잭션 내 `module_histories.create()` 직후 `module_histories.findMany()`를 해도 방금 생성한 행이 포함됨.

**프론트 단순화**:
- `saveLessonResult()` 함수 완전 제거
- `lastSavedResultsRef` 완전 제거
- `handleModuleSave`: `saveModuleHistory()` 결과만 반환 (백엔드가 세 테이블 처리)
- `handleExit`: `onExit()` 단순 호출 (별도 저장 불필요)

**변경 파일**:
- `apps/api/src/lessons/lessons.service.ts` — `saveModuleHistory` 전체 재구현: `$transaction` 래핑 + LLM은 트랜잭션 외부 실행 + 필드명 `module_sequence` → `skill_modules` 수정
- `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` — `saveLessonResult`, `lastSavedResultsRef` 제거; `handleModuleSave`, `handleExit` 단순화

**검증**: `pnpm typecheck` — @tutoring/types · @tutoring/api · @tutoring/web 3개 모두 통과 ✅

---

### 2026-07-02 — CLR 값 0 + 측정형 KPI description 없음 수정

**문제 1 — CLR `READING_COMPLETE` 값 0**

- embedded 콜백(케이스 3)의 KpiCtx에서 `answeredCount: Object.keys(answers).length = 0`
- CLR(embedded uiTemplate)은 문장 설명 완료를 `answers` 상태가 아닌 `useEmbeddedMode` 내부의 `explainedCount`로 추적하므로 `Object.keys(answers)` 는 항상 0
- `READING_COMPLETE` 수식 `Math.round(ctx.answeredCount / Math.max(ctx.totalCount, 1) * 100)` → 항상 0

**해결**: 임베디드 콜백 KpiCtx에서 `answeredCount: explainedCount`, `totalCount: totalCount` 으로 수정 (콜백 파라미터 직접 사용)

**문제 2 — 측정형(`측정` measureTool) KPI description 없음**

- `generateLlmKpiDescriptions()` 호출이 `pending=true`(LLM형) KPI에만 걸려 있었음
- `측정` 형 KPI의 description은 `meta.goal`(DB `code_items.extra_data.goal`) 값 그대로 반환 → 비어있으면 `''`
- "KPI 측정에서도 LLM을 호출하기로 했었다"는 의도가 구현에 반영 안 된 상태

**해결**: `saveModuleHistory` 트랜잭션 외부 처리에서 진입 조건을 `llmKpiCodes.length > 0` → `hasAnyKpiCode`(kpi_code 있는 KPI 하나라도 존재)로 변경. `generateLlmKpiDescriptions()` 를 LLM 값 평가 후 **전체 kpi_code 있는 KPI(측정+LLM 통합)**에 대해 호출. `module_histories` 는 값+description 최종 상태로 1회만 UPDATE.

**변경 파일**:
- `apps/api/src/lessons/lessons.service.ts` — 트랜잭션 외부 진입 조건 변경; description 생성 대상 전체 KPI로 확장
- `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` — 임베디드 콜백 KpiCtx `answeredCount: explainedCount`, `totalCount: totalCount` 수정

**검증**: `pnpm typecheck` — @tutoring/types · @tutoring/api · @tutoring/web 3개 모두 통과 ✅

---

### 2026-07-02 — 다른 사용자 계정으로 결과 저장되는 버그 수정

**문제**: 모듈 완료 결과가 다른 사용자 계정으로 저장됨.

**원인**: `saveModuleHistory` / `saveLessonResult` 서비스 메서드에 개발 편의용 폴백 UUID가 남아있었음.
- `saveModuleHistory`: `studentId ?? '631ea343-4639-4a9a-b6d2-28795c0dc040'`
- `saveLessonResult`: `authenticatedUserId ?? '00000000-0000-0000-0000-000000000000'`

JWT 만료·로그아웃으로 `extractUserId()`가 `undefined`를 반환하면 이 UUID 계정으로 결과가 저장됐다.

**해결**: `undefined`이면 `UnauthorizedException`을 던져 저장 자체를 차단.

```typescript
// lessons.service.ts
if (!studentId) throw new UnauthorizedException('인증이 필요합니다.');
const resolvedStudentId = studentId;
```

**변경 파일**:
- `apps/api/src/lessons/lessons.service.ts` — `UnauthorizedException` import 추가; 두 폴백 UUID 제거 → `throw` 로 대체

**검증**: `pnpm typecheck` — @tutoring/api 통과 ✅

---

### 2026-07-02 — 장시간 레슨 중 JWT 만료로 로그아웃되는 현상 수정

**문제**: 1시간 이상 수업 진행 시 로그아웃 상태가 되어 다음 API 호출 실패.

**원인**: 레슨 엔드포인트들이 `JwtAuthGuard`를 사용하지 않아 `touchActivity()`가 호출되지 않음 → `last_active_at` 미갱신 + JWT 재발급 없음. `JWT_EXPIRES_IN=1h` 만료 후 세션 복구 불가.
추가로 `/auth/me` 가 `touchActivity()`를 통해 새 토큰을 생성하면서도 응답에 반환하지 않아 `X-Refresh-Token` 헤더를 활용할 수 없었음.

**해결**:
1. `auth.service.ts` — `getMe()`가 `{ user, refreshToken? }` 반환하도록 수정
2. `auth.controller.ts` — `GET /auth/me` 응답에 `X-Refresh-Token` + `Access-Control-Expose-Headers` 헤더 추가
3. `useSessionKeepalive` 훅 신설 — 레슨 중 30분마다 `/auth/me` 호출 → JWT 갱신
4. `LessonSession.tsx` — `useSessionKeepalive()` 적용

**흐름**: 레슨 30분 경과 → `useSessionKeepalive`가 `/auth/me` 호출 → `touchActivity()` → 새 JWT 발급 → `X-Refresh-Token` 헤더 → `authFetch`가 `localStorage` 토큰 자동 교체.

**변경 파일**:
- `apps/api/src/auth/auth.service.ts` — `getMe()` 반환 타입 변경
- `apps/api/src/auth/auth.controller.ts` — `/auth/me` 응답 헤더 추가
- `apps/web/src/lib/hooks/useSessionKeepalive.ts` — 신규 훅 (30분 인터벌)
- `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` — `useSessionKeepalive()` 호출 추가

**검증**: `pnpm typecheck` — @tutoring/api · @tutoring/web 모두 통과 ✅

---

### 2026-07-02 — 재학습 시 결과 중복 저장 방지 (방안 C)

**배경**: 학습 완료 후 재학습 시 `module_histories`, `module_kpi_results`, `lesson_results`에 데이터가 중복 저장되는 문제.

**설계 결정 — 방안 C (모듈+레슨 조합 게이트)**:

두 조건을 병렬 조회 후 OR 로 판단:

| 조건 | 의미 | 차단 범위 |
|------|------|---------|
| `lesson_results.is_complete = true` | 레슨 전체 완료 | 해당 레슨의 모든 모듈 저장 |
| `module_histories` 동일 레코드 존재 | 이 모듈 이미 완료 | 이 모듈만 |

두 조건 중 하나라도 해당하면 `$transaction` 진입 전에 종료. 기존 `module_histories.kpis`를 그대로 반환하여 프론트 UI 유지.

```typescript
// lessons.service.ts — $transaction 진입 전
const [existingModule, completedLesson] = await Promise.all([
  this.prisma.module_histories.findFirst({
    where: { course_lesson_id: lessonId, student_id: resolvedStudentId, module_code: moduleCode },
    select: { kpis: true },
  }),
  this.prisma.lesson_results.findFirst({
    where: { course_lesson_id: lessonId, student_id: resolvedStudentId, is_complete: true },
    select: { id: true },
  }),
]);
if (completedLesson || existingModule) {
  return { success: true, skipped: true, id: null, lessonId, moduleCode,
           kpis: (existingModule?.kpis ?? []) as typeof body.kpis };
}
```

**응답**: `skipped: true` 플래그 포함 → 프론트에서 UI 분기 가능.

**변경 파일**:
- `apps/api/src/lessons/lessons.service.ts` — `$transaction` 앞에 이중 게이트 체크 추가 (쿼리 2개 병렬)

**검증**: `pnpm typecheck` — @tutoring/api 통과 ✅
