# tutoring.picklass.com — DB 저장 경로 전체 맵

> 작성일: 2026-06-23  
> 대상 코드: `tutoring.picklass.com` (apps/api + apps/web)

---

## 0. 목적과 배경

각 모듈에서 발생하는 결과 데이터를 체계적으로 수집·저장하고, 그 데이터를 기반으로 다음 네 가지 기능을 구현하기 위한 설계 기준 문서다.

| 목적 | 활용 데이터 | 현황 |
|------|-----------|------|
| **레벨 추정** | 모듈 점수, KPI, 학습 이력 | `GET /lessons/me/level-estimate` 구현됨 (결과 미저장) |
| **부족 역량 탐색** | KPI 분류(읽기/듣기/어휘/발음 등), 오답 패턴 | 수집은 됨 — 분석 로직 미구현 |
| **수업 추천** | 레벨 추정 + 부족 KPI 조합 | 미구현 |
| **성장성 확인** | 시계열 점수 추이, KPI 변화량 | 미구현 |

> **현재 상태:** 원시 데이터(`module_histories`, `lesson_results`)는 이미 수집되고 있다.  
> 위 기능들은 이 데이터를 읽어 분석·추론하는 레이어를 추가 구현해야 완성된다.  
> 본 문서는 그 분석 레이어 설계 시 "어떤 데이터가 어디에 있는가"를 빠르게 파악할 수 있도록 저장 경로 전체를 매핑한다.
>
> **→ 데이터 체계화 계획 상세:** [`result-data-plan.md`](./result-data-plan.md)

---

## 1. 저장 테이블 일람 (수집 데이터 분류)

Prisma 스키마 기준 (`apps/api/prisma/schema.prisma`):

| 테이블 | 역할 | 분석 활용 |
|--------|------|-----------|
| `texts` | 학습 지문 (AI 생성 포함) | — |
| `text_analyses` | 지문 분석 결과 캐시 (CEFR, Gemini) | 지문 난이도 기준값 |
| `courses` | 과정 컨테이너 | — |
| `course_lessons` | 레슨 (모듈 시퀀스 포함) | — |
| `module_questions` | 모듈별 문항 캐시 | — |
| `module_histories` | 모듈 완료 시 대화 기록 + 점수 + KPI | **핵심 — 레벨 추정·역량 분석 원천** |
| `lesson_results` | 레슨 최종 결과 (모듈 집계) | **핵심 — 성장성 추이 원천** |
| `external_access_tokens` | 외부 스피킹 서비스 원타임 토큰 | — |
| `access_codes` | 수강 등록 코드 (raw SQL 업데이트) | — |

---

## 2. 저장 경로 상세

### 2-1. 모듈 이력 (`module_histories`)

**트리거:** 각 모듈 완료 시 (사용자가 "다음" 클릭, 자동 완료 신호, CLR 수동 완료)

```
ModuleRunnerInner (handleModuleNext | replanSignal | embedded.handleManualComplete)
  → LessonSession.handleModuleComplete()
      [web] apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx:112
  → saveModuleHistory()
      [web] apps/web/src/lib/services/lessonService.ts:33
  → POST /lessons/:lessonId/modules/:moduleCode/history
      [api] apps/api/src/lessons/lessons.controller.ts:71
  → LessonsService.saveModuleHistory()
      [api] apps/api/src/lessons/lessons.service.ts:463
  → prisma.module_histories.create()
      [api] apps/api/src/lessons/lessons.service.ts:483
```

**저장 필드:**

| 필드 | 타입 | 내용 |
|------|------|------|
| `course_lesson_id` | UUID | 레슨 ID |
| `student_id` | UUID | 학습자 ID |
| `module_code` | VARCHAR | 모듈 코드 (e.g. PRD, SCN) |
| `answers` | JSON | 학습자 제출 답안 |
| `chat_messages` | JSON | AI ↔ 학습자 전체 대화 |
| `score` | INT | 모듈 점수 (0–100) |
| `kpis` | JSON | KPI 평가 결과 배열 |
| `started_at` | TIMESTAMP | 모듈 시작 시각 (옵션) |
| `completed_at` | TIMESTAMP | 완료 시각 |

---

### 2-2. 레슨 결과 (`lesson_results`)

**트리거:** 모듈 완료마다 누적 저장 + 중도 이탈(onExit) 시도 저장

```
LessonSession.handleModuleComplete() — 모듈 완료마다
LessonSession.handleExit()           — 중도 이탈
  [web] apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx:88
  → saveLessonResult() (LessonSession 내부 함수)
  → POST /lessons/:lessonId/complete
      [api] apps/api/src/lessons/lessons.controller.ts:88
  → LessonsService.saveLessonResult()
      [api] apps/api/src/lessons/lessons.service.ts:554
  → 기존 레코드 있으면 update(), 없으면 create()
      [api] apps/api/src/lessons/lessons.service.ts:577 (update), 591 (create)
```

**저장 필드:**

| 필드 | 타입 | 내용 |
|------|------|------|
| `course_lesson_id` | UUID | 레슨 ID |
| `student_id` | UUID | 학습자 ID |
| `average_score` | DECIMAL(5,2) | 완료된 모듈 평균 점수 |
| `module_results` | JSON | `[{moduleCode, score, kpis}]` 배열 |
| `total_duration` | INT | 총 소요 시간(초) |
| `completed_at` | TIMESTAMP | 마지막 저장 시각 |

> **주의:** 모든 모듈이 완료되지 않아도 중도 이탈 시 저장됨. `isComplete` 플래그로 완전 완료 여부 구분.

---

### 2-3. 문항 캐시 (`module_questions`)

**트리거:** 모듈 데이터 첫 조회 시 캐시 미스 → AI 생성 후 저장

```
ModuleRunner (init)
  → fetchAiModuleByCode() + adapter.fetchModuleData()
  → GET /lessons/:lessonId/module-data?moduleCode=XXX
      [api] apps/api/src/lessons/lessons.controller.ts:61
  → LessonsService.getModuleData()
      [api] apps/api/src/lessons/lessons.service.ts:355
  → QuestionGeneratorService.getOrGenerate()
      [api] apps/api/src/ai/question-generator.service.ts
  → [캐시 미스 시] Gemini로 문항 생성
  → prisma.module_questions.createMany()
      [api] apps/api/src/ai/question-generator.service.ts:109
```

**저장 필드:**

| 필드 | 타입 | 내용 |
|------|------|------|
| `module_id` | UUID | AI 모듈 ID |
| `text_id` | INT | 지문 ID |
| `question_number` | INT | 문항 번호 |
| `type` | VARCHAR | 문항 유형 (multiple-choice, essay 등) |
| `instruction` | TEXT | 문항 지시문 |
| `source` | TEXT\|JSON | 출제 근거 (지문 내 문장 등) |
| `text` | TEXT\|JSON | 문항 텍스트 |
| `options` | JSON | 선택지 배열 |
| `answer` | TEXT | 정답 |
| `hints` | JSON | `{button?, direct?}` |
| `sort_order` | INT | 표시 순서 |
| `status` | VARCHAR | `active` \| `archived` |

---

### 2-4. 힌트 저장 (`module_questions.hints`)

**트리거:** AI 피드백/채점 요청 시 `generateHint=true` 파라미터 포함, 또는 온디맨드 힌트 생성

```
(a) 피드백 생성 시 힌트 포함
    POST /ai/feedback        [api] apps/api/src/ai/ai.controller.ts:17
    POST /ai/writing-eval    [api] apps/api/src/ai/ai.controller.ts:40

(b) 힌트 온디맨드
    POST /ai/hint/generate   [api] apps/api/src/ai/ai.controller.ts:83

  → QuestionGeneratorService.saveHintDirect()
      [api] apps/api/src/ai/question-generator.service.ts:159
  → prisma.module_questions.update()  — hints.direct 필드 업데이트
      [api] apps/api/src/ai/question-generator.service.ts:159

  → 힌트 archived 처리 (문항 재생성 시)
      prisma.module_questions.updateMany()  — status='archived'
      [api] apps/api/src/ai/question-generator.service.ts:135, 172
```

---

### 2-5. 나만의 수업 생성 — 트랜잭션 + 비동기

**트리거:** 사용자가 주제 입력 후 "수업 만들기" 버튼 클릭

```
POST /lessons/create-custom
  [api] apps/api/src/lessons/lessons.controller.ts:102
  → LessonsService.createCustomLesson()
      [api] apps/api/src/lessons/lessons.service.ts:611
```

#### (A) 트랜잭션 — 동기 (원자적)

```
prisma.$transaction(async (tx) => {
  tx.texts.create()           lessons.service.ts:695
  tx.courses.findFirst()      — 기존 "나만의 수업" 과정 재사용
  tx.courses.create()         lessons.service.ts:715  (최초 생성 시만)
  tx.course_lessons.create()  lessons.service.ts:731
})
```

**`texts` 저장 필드:**

| 필드 | 내용 |
|------|------|
| `user_id` | 학습자 ID |
| `title` | AI 생성 제목 |
| `content` | AI 생성 지문 본문 |
| `word_count` | 단어 수 |
| `text_type` | `'A'` (일반 텍스트) |
| `topic` | 사용자 입력 주제 |

**`courses` 저장 필드 (최초 1회):**

| 필드 | 내용 |
|------|------|
| `institution_id` | `00000000-...` (더미) |
| `title` | `'나만의 수업'` |
| `level_code` | `L{1~18}` |
| `genre_code` | `'custom'` |
| `created_by` | 학습자 ID |

**`course_lessons` 저장 필드:**

| 필드 | 내용 |
|------|------|
| `course_id` | 과정 ID |
| `text_id` | 방금 생성된 texts.id |
| `lesson_order` | 과정 내 순번 |
| `topic` | 사용자 입력 주제 |
| `topic_source` | `'ai'` |
| `skill_modules` | `PlannedModule[]` JSON (모듈 시퀀스) |
| `status_code` | `'active'` |

#### (B) 비동기 — fire-and-forget (트랜잭션 커밋 후)

```
[B-1] 지문 분석
  textAnalyses.runDefaults()
    → prisma.text_analyses.upsert()  (analyzer_type별 2회)
        [api] apps/api/src/analyzer/text-analyses.service.ts:27, 55
  저장 필드: { text_id, analyzer_type, analyzer_version, result(JSON), status, analyzed_at }

[B-2] 스피킹 지문 변환
  speakingPassageService.generate()
    → prisma.texts.update()
        [api] apps/api/src/lessons/lessons.service.ts:770
  저장 필드: { speaking_content, speaking_generated_at, speaking_prompt_code }

[B-3] 핵심표현 생성
  speakingCoreDataService.generateCoreExpressions()
    → prisma.texts.update()
        [api] apps/api/src/ai/speaking-core-data.service.ts:106
  저장 필드: { core_expressions (JSON) }

[B-4] 대화문 생성 (핵심표현 의존)
  speakingCoreDataService.generateCoreDialog()
    → prisma.texts.update()
        [api] apps/api/src/ai/speaking-core-data.service.ts:141
  저장 필드: { core_dialog (JSON) }
```

---

### 2-6. 외부 스피킹 토큰 (`external_access_tokens`)

**트리거:** 레슨 플랜 조회 시 스피킹 모듈(SNR / FRT) 감지

```
GET /lessons/:lessonId/plan
  [api] apps/api/src/lessons/lessons.controller.ts:25
  → LessonsService.createSpeakingModuleTokens()
      [api] apps/api/src/lessons/lessons.service.ts:280
  → ExternalService.getOrCreateToken()
      [api] apps/api/src/external/external.service.ts
  → 유효 토큰 없으면: prisma.external_access_tokens.create()
      [api] apps/api/src/external/external.service.ts:48
```

**저장/업데이트 시점:**

| 시점 | Prisma 메서드 | 파일:줄 | 변경 필드 |
|------|--------------|---------|-----------|
| 토큰 생성 | `create()` | external.service.ts:48 | 전체 레코드 |
| 토큰 만료 처리 | `update()` | external.service.ts:89 | `isExpired=true` |
| 토큰 사용 기록 | `update()` | external.service.ts:102 | `usedAt=NOW()` |
| 배치 만료 | `update()` | external.service.ts:206 | `isExpired=true` |

**주요 필드:**

| 필드 | 내용 |
|------|------|
| `token` | UUID 원타임 토큰 |
| `userId` | 학습자 ID |
| `courseLessonId` | 레슨 ID |
| `moduleCode` | `'SNR'` 또는 `'FRT'` |
| `isExpired` | 만료 여부 |
| `expiresAt` | 기본 60분 |
| `usedAt` | 사용 시각 (최초 사용 시 기록) |

---

### 2-7. 수강 등록 코드 (`access_codes`)

**트리거:** 사용자가 액세스코드 입력 후 등록

```
POST /lessons/register-accesscode
  [api] apps/api/src/lessons/lessons.controller.ts:131
  → LessonsService.registerAccessCode()
      [api] apps/api/src/lessons/lessons.service.ts:810
  → prisma.$executeRawUnsafe() — raw SQL UPDATE
      [api] apps/api/src/lessons/lessons.service.ts:864
```

**업데이트 필드:**

| 필드 | 변경 내용 |
|------|----------|
| `status_code` | `→ 'active'` |
| `user_id` | 학습자 ID 바인딩 |
| `activated_at` | `NOW()` |
| `usage_start_date` | `COALESCE(기존값, NOW())` |
| `usage_end_date` | `COALESCE(기존값, NOW() + usage_period_days)` |
| `updated_at` | `NOW()` |

> raw SQL을 사용하는 이유: `access_codes`는 Prisma 마이그레이션 대상이 아닌 공유 테이블이므로 ORM 모델 없이 직접 쿼리.

---

## 3. 저장 없는 API (읽기 전용 / 즉시 응답)

| 엔드포인트 | 설명 |
|-----------|------|
| `GET /lessons/me/level-estimate` | 학습 이력 기반 레벨 추정 — 결과 미저장 |
| `GET /common-codes/KPI_CATEGORY/items` | KPI 카탈로그 조회 |
| `POST /ai/generate-passage` | 지문 생성만 (저장은 create-custom에서) |
| `POST /ai/feedback` | 피드백 텍스트 응답 (힌트 저장은 별도 분기) |
| `POST /ai/writing-eval` | 영작 채점 응답 (힌트 저장은 별도 분기) |

---

## 4. 전체 흐름도

```
레슨 진입
  └─ GET /lessons/:id/plan
       ├─ 스피킹 모듈 감지
       │    └─ external_access_tokens.create()
       └─ 플랜 + externalTokens 반환

모듈 진행
  └─ GET /lessons/:id/module-data?moduleCode=XXX
       └─ [캐시 미스] module_questions.createMany()

각 모듈 완료
  ├─ POST /lessons/:id/modules/:code/history
  │    └─ module_histories.create()
  └─ POST /lessons/:id/complete
       └─ lesson_results.update() or create()  ← 누적

레슨 종료 (이탈 포함)
  └─ POST /lessons/:id/complete  (동일 엔드포인트)
       └─ lesson_results.update()

나만의 수업 생성 (별도 플로우)
  └─ POST /lessons/create-custom
       ├─ [동기 트랜잭션]
       │    ├─ texts.create()
       │    ├─ courses.create()  (최초)
       │    └─ course_lessons.create()
       └─ [비동기 fire-and-forget]
            ├─ text_analyses.upsert()  ×2
            ├─ texts.update (speaking_content)
            ├─ texts.update (core_expressions)
            └─ texts.update (core_dialog)

수강 등록
  └─ POST /lessons/register-accesscode
       └─ access_codes UPDATE (raw SQL)
```

---

## 5. 관련 파일 색인

| 파일 | 역할 |
|------|------|
| `apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx` | 프론트 저장 트리거 |
| `apps/web/src/lib/services/lessonService.ts` | 프론트 API 호출 |
| `apps/api/src/lessons/lessons.controller.ts` | API 라우팅 |
| `apps/api/src/lessons/lessons.service.ts` | 핵심 비즈니스 로직 + Prisma 호출 |
| `apps/api/src/ai/question-generator.service.ts` | 문항 캐시 생성/저장 |
| `apps/api/src/ai/speaking-core-data.service.ts` | 핵심표현·대화문 저장 |
| `apps/api/src/analyzer/text-analyses.service.ts` | 지문 분석 결과 저장 |
| `apps/api/src/external/external.service.ts` | 외부 토큰 관리 |
| `apps/api/prisma/schema.prisma` | 전체 테이블 스키마 |
