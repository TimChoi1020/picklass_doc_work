# Tutoring 기획서 §5 · 기능 상세 (User Story + AC)

> 참고문서(`참고문서/picklass_docs/tutoring/`) 기반. 화면 §4(TU-S-###), 정책 §6(TU-P-###), 예외 §7(TU-E-###) 참조.
> **AC**: Given/When/Then. `☐` 미검증.

---

## 0. 기능 목록 (Feature Index)

| 기능ID | 기능명 | 화면 | 정책 | 예외 | 상태 |
|---|---|---|---|---|---|
| TU-F-001 | 로그인·회원가입·액세스코드 인증 | TU-S-060/061 | P-001 | E-001 | ✅(소셜 미구현) |
| TU-F-010 | 튜터링 홈(오늘의 레슨) | TU-S-001 | P-002 | — | ✅ |
| TU-F-020 | 나의수업 3섹션 | TU-S-010 | P-002 | E-006 | ✅ |
| TU-F-021 | 과정 드로어 탐색 | TU-S-012 | — | — | ✅ |
| TU-F-030 | 나만의 수업 생성 | TU-S-020 | P-003 | E-002 | ✅ |
| TU-F-040 | 액세스코드 등록 | TU-S-030 | P-002 | E-001 | ✅ |
| TU-F-050 | 모듈 학습(오케스트레이터) | TU-S-040 | P-004, P-005 | E-003, E-004, E-005 | ✅ |
| TU-F-051 | 레슨 완료 리포트 | TU-S-041 | P-006 | E-006 | ✅ |

---

## 1. 홈 · 나의수업

### TU-F-010 · 튜터링 홈 (오늘의 레슨)
- **화면**: TU-S-001
- **User Story**: 학습자로서, 선택 피로 없이 바로 학습하기 위해, 지금 이어서 할 레슨 하나만 보고 싶다.
- **동작**: "현재 수강중인 과정 전체" 대신 **첫 번째 미완료 활성 과정 1개 카드만**(`find(active)`). "전체 보기 →" → `/my-classes`. `GET /lessons/enrolled` 계열.
- **AC**:
  - ☐ Given 미완료 활성 과정 다수, Then 첫 번째 1개만 카드로 노출된다
  - ☐ Then "전체 보기 →"로 나의수업으로 이동한다

### TU-F-020 · 나의수업 3섹션
- **화면**: TU-S-010
- **User Story**: 학습자로서, 내 과정을 상태별로 관리하기 위해, 수강중/내가 만든/수강종료로 구분해 보고 싶다.
- **동작**: 오늘 자정 기준 분류 — `nonCustom`/`custom` 분리 후 active=`!usageEndDate || usageEndDate >= today`, expired=`< today`. 무제한 활성 있으면 기간 null, 아니면 `maxActiveEndDate` → 헤더 `~YYYY.MM.DD 까지 이용 가능`. CourseCard 배지: integrated=통합(보라)/speaking=스피킹(파랑)/tutoring·null=배지 없음.
  - `GET getEnrolledCourses`: `isCustom`(genre_code==='custom')·courseType·usageStart/EndDate·`completedLessons`(실계산 `lesson_results.completed_at IS NOT NULL` count)·레슨별 isCompleted·`modules[]`(ai_modules JOIN `{moduleCode,name,skill}`). `nextLesson` = 미완료+text 있는 것 우선 → 첫 미완료 → 전부완료 시 마지막.

| 섹션 | 표시조건 | 아이콘/색 |
|---|---|---|
| 수강중인 과정 | 항상 | Sparkles/초록 |
| 내가 만든 과정 | `customCourses.length > 0` | PenLine/앰버 |
| 수강종료 과정 | 항상 | CalendarX/회색 |

- **AC**:
  - ☐ Given `usage_end_date < 오늘`, Then 수강종료 섹션에 표시된다
  - ☐ Given custom 과정 0개, Then "내가 만든 과정" 섹션이 숨겨진다
  - ☐ Then completedLessons는 lesson_results 실계산 값이다

### TU-F-021 · 과정 드로어 탐색
- **화면**: TU-S-012
- **동작**: CourseCard 클릭 → CourseDrawer(우측 슬라이드) → 레슨 아코디언 → 모듈 클릭 → `/modules/[lessonId]`. 모듈명/스킬은 백엔드 `getEnrolledCourses` JOIN 값(프론트 하드코딩 제거).
- **AC**:
  - ☐ Given 드로어, When 모듈 클릭, Then 페이지 이동으로 학습이 시작된다

---

## 2. 나만의 수업 · 액세스코드

### TU-F-030 · 나만의 수업 생성
- **화면**: TU-S-020 (`/class/lesson-setup/custom?topic=...`)
- **User Story**: 학습자로서, 원하는 주제로 자유 학습하기 위해, 목표·시간·레벨을 골라 AI 수업을 만들고 싶다.
- **동작**: Step `options|generating|preview|error`. 진입 시 동시 fetch — `GET /common-codes/KPI_CATEGORY/items`(KPI 칩), `GET /lessons/me/level-estimate`(추정 CEFR), `GET /institutions/settings`(기관 제약).
  - 옵션 3입력: 학습목표(KPI 칩 다중, `extra_data.goal` 그룹핑+스킬 배지 V/R/W/L/S) · 학습시간(`[15][20][25][30]`, 기본 30) · CEFR 슬라이더(19단계, 기본=추정 또는 B1). 추정 배지 "최근 N회 학습 분석: B1 (신뢰도 76%)".
  - CTA "수업 만들기" → `POST /lessons/create-custom {topic, targetLevel?, selectedKpiCodes?, durationMin?, targetCefrLevel?}` (기본 `['VOCAB_RECOG']`/30/'B1'). 응답 `{lessonId, moduleSequence, sequencing{...}}`.
  - **트랜잭션화**: 외부호출(generatePassage → analyzer 시퀀싱) 완료 후 `$transaction{texts, courses('나만의 수업', genre_code='custom'), course_lessons(skill_modules=moduleSequence)}` → orphan 방지. 미리보기 "수업 준비 완료!" → "학습 시작" → `/modules/{lessonId}`.
- **AC**:
  - ☐ Given `free_learning_enabled=false` 기관, Then 생성 차단(403, error step, TU-P-003)
  - ☐ Given 학습시간 > `max_total_class_minutes`, Then 초과 프리셋 disabled + 서버 400(TU-E-002)
  - ☐ Given 학습이력 <3건, Then CEFR 추정 신뢰도 ≤0.4(cold-start)
  - ☐ Given analyzer 장애, Then Gemini 폴백 없이 명확한 에러(502/503)
- **정책**: P-003 · **예외**: E-002

### TU-F-040 · 액세스코드 등록
- **화면**: TU-S-030 (모달)
- **동작**: 코드 입력(autoFocus·자동 대문자) → "등록" → `POST /lessons/register-accesscode {accessCode}`(trim+toUpperCase). 성공 → 모달 닫힘 + `window.location.reload()`(개선 대상). 등록 처리: `registration_expiry` 만료 체크, 이미사용 판별 `status_code==='active'`, UPDATE `status='active', activated_at=NOW(), usage_start/end_date=COALESCE(기존, NOW()[+usage_period_days])`.
- **AC**:
  - ☐ Given 빈 값, Then "액세스코드를 입력해주세요"(인라인)
  - ☐ Given 등록기간 만료 코드, Then "액세스코드 등록 기간이 만료되었습니다."
  - ☐ Given 성공, Then 과정이 수강중으로 등록된다(`{message, courseId}`)
  - ⚠️ 성공 메시지 미표시(`success` state 미호출) — 알려진 갭

---

## 3. 모듈 학습 · 레슨 완료

### TU-F-050 · 모듈 학습 (오케스트레이터)
- **화면**: TU-S-040
- **User Story**: 학습자로서, AI 튜터의 실시간 지도를 받으며, 지문 기반 문항을 풀고 피드백을 받고 싶다.
- **동작**: `/modules/[lessonId]` → LessonPlan fetch → `LessonSession`(모듈 시퀀스) → `ModuleRunner`(어댑터·`GET /lessons/:id/module-data`) → `ModuleRunnerInner`. 모듈 동작은 **DB 3축**(passageMode/uiTemplate/questionFlowMode)로 결정(TU-P-004). 학습 진행은 **Rule 엔진**(`useModuleOrchestrator`)이 지휘 — greet → showPassage → askQuestion → 답변 → 피드백/힌트 → 다음 → celebrate → completeModule(TU-P-005).
- **AC**:
  - ☐ Given 문항 미준비, Then "Pickle AI가 문항을 준비 중이에요..." 표시(activeQuestionId null)
  - ☐ Given exact 채점 오답 1회, Then 힌트(direct) / 2회↑, Then 정답 공개(TU-P-005 Rule 3)
  - ☐ Given 120초 무응답, Then 참여도 확인 / 180초+저참여, Then 리플래닝
  - ☐ Given 모든 문항 완료, Then celebrate → completeModule → 다음 모듈 또는 레슨 완료
  - ☐ Given LLM 대기 중 제출, Then pending 마킹 후 완료 시 재실행
- **정책**: P-004, P-005 · **예외**: E-003, E-004, E-005

### TU-F-051 · 레슨 완료 리포트 (7섹션)
- **화면**: TU-S-041
- **User Story**: 학습자로서, 레슨 성과를 확인하고 다음을 정하기 위해, 종합 리포트를 보고 싶다.
- **동작**: `GET /lessons/:lessonId/complete-summary`(JWT 필수) → DB-first(module_histories 최신 1건/모듈) + 네트워크 실패 시 React state 폴백. AI 피드백은 **5초 후 재조회**로 갱신(Gemini fire-and-forget).

| 섹션 | 내용 |
|---|---|
| S1 HERO | SVG 도넛 게이지+count-up, 지문타이틀(passageTitle ?? lessonTitle ?? learningGoal), 시간, CEFR 뱃지 |
| S2 AI 종합 피드백 | 강점/개선점/다음학습 3컬럼(5초 재조회, 스켈레톤) |
| S3 종합 평가 | 등급 A~F + 통계 그리드 + 모듈별 막대 |
| S4 역량 분석 | 전체 KPI(pending 포함) |
| S5 학습 행동 분석 | 힌트·재시도·참여도 3패널 |
| S6 모듈별 성과 | ModuleTimeline 아코디언 KPI |
| S7 CTA | 다음 레슨 / 홈 |

- **AC**:
  - ☐ Given summaryFeedback=null, Then 스켈레톤 유지(숨기지 않음)
  - ☐ Given estimatedLevel=null, Then 레벨 뱃지 숨김
  - ☐ Given 다음 레슨 없음, Then 홈 버튼만 노출
- **정책**: P-006 · **예외**: E-006

---

## 4. 인증 (참고)

### TU-F-001 · 로그인·회원가입·인증
- **동작**: `/auth/login`(userId+pw)·`/auth/signup`(email/pw/nickname→users.name)·`/auth/me`(touchActivity + refreshToken 헤더 갱신)·`/auth/logout`. JWT 단일 토큰(기본 1h), `localStorage('picklass_auth_token')`, `Authorization: Bearer`. 유휴 `IDLE_TIMEOUT_SECONDS`(3600s) 초과 → 401. 장시간 레슨 `useSessionKeepalive`(30분마다 `/auth/me`).
- **AC**:
  - ☐ Given 소셜 로그인(google/kakao/naver), Then `alert("준비 중")`(미구현)
  - ☐ Given 유휴 3600s 초과, Then 401 → 로그아웃
