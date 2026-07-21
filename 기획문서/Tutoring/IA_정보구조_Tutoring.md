# Tutoring IA (정보구조)

> 참고문서(`참고문서/picklass_docs/tutoring/`)를 기반으로 정리한 Tutoring(AI 튜터링) 서비스의 정보구조 문서입니다.
> **핵심 근거**: `docs/picklass-tutoring-planning.md`, `docs/modules-lessonId-20260325.md`, `docs/20260410_튜터링_모듈학습_UI개선및버그수정.md`
> 도메인: `tutoring.picklass.com` / 스택: Next.js 15 App Router(`apps/web`) + NestJS(`apps/api`), 공유 Supabase

---

## 0. 서비스 개요 & IA 설계 배경

**Tutoring은 학습자가 AI 튜터와 함께 지문 기반 학습(읽기·쓰기·말하기)을 진행하는 웹앱**입니다. Speaking이 말하기 특화 앱이라면, Tutoring은 **여러 스킬 모듈을 한 레슨 안에서 순서대로 학습하는 종합 학습 플랫폼**이자 Picklass 학습자의 기본 진입점입니다.

**IA 설계의 3가지 배경 원리:**

**① "관리 뷰"와 "몰입 학습 뷰"의 분리.** 상단 헤더가 있는 일반 페이지(홈·나의수업·리포트)는 과정을 고르고 성과를 보는 관리 영역이고, 실제 학습은 **헤더를 걷어낸 전용 레이아웃(`/modules/[lessonId]`)** 에서 진행됩니다. 학습 중에는 네비게이션이 사라져 몰입을 돕습니다.

**② 홈은 "액션 허브", 나의수업은 "관리 뷰"로 역할 분리.** 2026-04-10 개편의 핵심으로, 예전에는 홈에 전체 과정이 뒤섞여 있었으나, **홈(`/`)은 "오늘 이어서 할 레슨 1개"만 보여주는 즉시 실행 화면**으로, **나의수업(`/my-classes`)은 수강중·내가 만든·수강종료 과정을 모두 관리하는 뷰**로 나눴습니다. "지금 뭘 해야 하나?"와 "내 과정 전체를 보자"를 분리한 것입니다.

**③ 데이터드리븐 모듈 렌더링.** IA에서 가장 중요한 설계 결정. 학습 화면은 모듈 종류마다 코드를 분기하지 않고, **DB의 두 필드(`passageMode`·`questionFlowMode`)로 화면 동작을 결정**합니다(§4.2). 덕분에 새 모듈을 코드 수정 없이 DB 등록만으로 추가할 수 있어, 학습 화면 IA가 콘텐츠 확장에 유연하게 대응합니다.

**학습 흐름의 큰 그림**: 액세스코드로 과정을 등록 → 나의수업/홈에서 레슨 진입 → 모듈을 순서대로 학습(AI 오케스트레이터가 실시간 지도) → 레슨 완료 리포트 확인.

---

## 1. 전역 네비게이션 & 라우트

### 상단 네비게이션 (`StudioHeader.tsx`, 2026-04-10 개편)
| 순번 | 메뉴명 | 경로 | 역할 |
|---|---|---|---|
| 1 | 튜터링 | `/` | 홈/액션 허브 — "오늘 이어서 할 레슨" 1카드 + "전체 보기 →" |
| 2 | 나의수업 | `/my-classes` | 수강 과정 관리 (전체 목록) |
| 3 | 리포트 | `/report` | 학습 리포트 (보호 경로) |
| 4 | 액세스코드 | (모달) | `AccessCodeModal` — 과정 등록, 별도 라우트 없음 |

- 개편 전: `튜터링 | 리포트 | 액세스코드` (홈에 전체 과정 혼재)
- 인증·등록 모달(헤더 관리): `LoginModal`, `SignupModal`, `AccessCodeModal`
- 보호 경로(계획): `/modules/*`, `/report`, `/my-classes` — `middleware.ts` 미구현(localStorage 토큰이라 SSR 미들웨어에서 못 읽음)

### 라우트 맵
| 경로 | 화면 | 성격 |
|---|---|---|
| `/` | 튜터링 홈 (오늘의 레슨 1카드) | 관리(헤더 O) |
| `/my-classes` | 나의수업 (수강중/내가 만든/수강종료 3섹션) | 관리(헤더 O) |
| `/class/lesson-setup/custom` | 나만의 수업 생성 | 관리(헤더 O) |
| `/modules/[lessonId]` | 학습(모듈 실행) | **몰입(헤더 X, 전용 레이아웃)** |
| `/report` | 리포트 (보호 경로) | 관리(헤더 O) |

---

## 2. 화면별 기능 설명

### 튜터링 홈 `/`
**"지금 당장 이어서 할 레슨 하나만 제시하는 즉시 실행 허브"**. 선택 피로를 줄이고 학습을 바로 시작시키기 위해, 여러 과정 중 첫 번째 미완료 과정 1개 카드만 노출합니다.
- 첫 번째 미완료 과정 1개 카드만 표시(`find(active)`), 액션 허브 역할
- "전체 보기 →" → `/my-classes`

### 나의수업 `/my-classes` — 3섹션
**"내가 가진 모든 과정을 상태별로 관리하는 뷰"**. 진행 상태에 따라 3섹션으로 나눠, 수강중인 것과 끝난 것을 명확히 구분합니다.

| 섹션 | 표시 조건 | 아이콘/색 |
|---|---|---|
| 수강중인 과정 | 항상 | Sparkles / 초록 |
| 내가 만든 과정 | custom 과정 존재 시 | PenLine / 앰버 |
| 수강종료 과정 | 항상 | CalendarX / 회색 |

- **수강 상태**(계약 기간 기준): `usage_end_date IS NULL 또는 ≥ 오늘` = 수강중 / `< 오늘` = 수강종료
- **구성요소**: `CourseCard`(완료·진도 통계, 프로그레스 바, 유형 배지[통합=보라/스피킹=파랑], 수강기간 배지) + `CourseDrawer`(우측 슬라이드 → 레슨 아코디언 → 모듈 클릭 → `/modules/[lessonId]`로 학습 진입). **드로어로 별도 페이지 이동 없이 과정 속을 탐색**하도록 함.
- "내가 만든 과정"은 수강중 일반 과정 1개+ 있을 때만 이용(`canUseCustom`), 없으면 잠금 배너
- API: `GET /lessons/enrolled-courses`(access_codes UNION custom)

### 나만의 수업 생성 `/class/lesson-setup/custom`
**"학생이 주제·목표·시간을 골라 AI가 즉석에서 만들어주는 맞춤 수업 생성 화면"**. 정규 과정 외에 자유 학습을 원하는 학생을 위한 기능으로, Studio의 과정 생성과 옵션 UI를 맞췄습니다.
- Step 상태: `options | generating | preview | error`
- 옵션 폼: 주제(read-only) / 학습목표(KPI) 칩 다중(스킬 배지 V·R·W·L·S) / 학습 시간(15·20·25·30분, 기본 30) / CEFR 슬라이더(19단계, 학습이력 추정 또는 B1) / 추정 안내 배지 / CTA "수업 만들기" → `POST /lessons/create-custom`
- 생성 후 "수업 준비 완료!" → "학습 시작" → `/modules/{lessonId}`
- **기관 소속 학생은 기관설정(`max_total_class_minutes`, `free_learning_enabled`) 강제** — 백오피스에서 정한 정책이 여기 반영

### 레슨 완료 화면 (`/modules/[lessonId]` 완료 분기 → `LessonCompleteView`) — 7섹션
**"한 레슨을 마친 뒤 성과를 종합 리포트로 보여주는 결산 화면"**. AI 종합 피드백부터 모듈별 KPI까지 단계적으로 펼쳐 학습 동기를 부여합니다.

| 섹션 | 컴포넌트 | 내용 |
|---|---|---|
| S1 HERO | HeroSection | SVG 도넛 게이지, 타이틀, 시간, CEFR 뱃지 |
| S2 AI 종합 피드백 | AiFeedbackSection | 강점/개선점/다음 학습 3컬럼(5초 후 갱신) |
| S3 종합 평가 | OverallSection | 등급(A~F) + 통계 그리드 + 모듈별 막대 |
| S4 역량 분석 | KpiSection | 전체 KPI + 약점 강조 |
| S5 학습 행동 분석 | BehaviorSection | 힌트·재시도·참여도 3패널 |
| S6 모듈별 성과 | ModuleTimeline | 아코디언 KPI 상세 |
| S7 CTA | — | 다음 레슨 / 홈으로 |

- 데이터: `GET /lessons/:id/complete-summary`(moduleResults + weak_kpi_codes + estimated_level), DB-first + state 폴백. AI 피드백은 Gemini fire-and-forget → 5초 재조회로 갱신(먼저 화면을 띄우고 피드백은 나중에 채우는 방식)

---

## 3. 주요 학습 플로우 (수업 진입 ~ 레슨 완료)

**한 레슨은 여러 모듈의 시퀀스이며, 각 모듈은 AI 오케스트레이터가 실시간으로 진행을 지휘**합니다. 모듈이 끝날 때마다 다음 모듈로 넘어가고, 마지막에 완료 리포트로 이어집니다.

```
① /modules/[lessonId] → fetchLessonPlan → LessonSession 렌더
② LessonSession → moduleSequence[0] → ModuleRunner (key=currentModuleIdx)
③ ModuleRunner
   ├ getAdapter(moduleCode) — 모듈 종류별 어댑터 선택
   ├ adapter.fetchModuleData / buildContentConfig
   └ 분기: PWR → ProcessWritingFlow(4단계 수동) / 그 외 → ModuleRunnerInner + useModuleOrchestrator
④ 실시간 학습 (ModuleOrchestratorAgent, Rule 기반 결정 루프)
   greet → showPassage → askQuestion → 답변 → orchestrate() → Tool(피드백/힌트/다음)
⑤ 모든 문항 완료 → celebrate → completeModule → handleModuleComplete()
   ├ saveModuleHistory (비동기) / currentModuleIdx+1
   └ 다음 모듈? YES→③ 반복 / NO→ LessonCompleteView(완료 리포트)
```

### 모듈별 세부 흐름
- **PRD(예측)**: greet→미리보기 지문→essay→holistic 피드백(+전체 지문 공개)→celebrate
- **SHR(쉐도잉)**: 모델 오디오(SHR-A)→녹음(SHR-B)→제출→발음 피드백(SHR-C)
- **SWR(영작)**: 지문→sentence-write(한글 패러프레이즈+textarea)→writing 피드백
- **PWR(창작)**: Orchestrator 미사용, 4단계 수동(Outlining → Self-Check → 1st Draft → Final Draft)

### AI 오케스트레이터 규칙 우선순위 (학습 중 AI가 무엇을 할지 결정하는 규칙 순서)
Rule 0 ruleGreet(첫 인사) → 1 ruleIdleCheck(120s 무응답) → 2 ruleDisengaged(180s+low) → 3 ruleRepeatWrongAnswer(오답 2회+ → 힌트) → 4a ruleHolisticFeedback(essay) → SHR-C 발음 / SWR-A 작문 피드백 → 4d 객관식 오답 → 5 ruleCompleteAfterCelebrate(오답률 ≥50% → 재계획) → SHR-A/B 오디오·녹음 → 6 정답 → 7 초기진입(showPassage) → 8 다음 문항

---

## 4. 모듈 렌더링 / 학습 화면 구조

### 컴포넌트 계층
학습 화면은 레슨(시퀀스) → 모듈(러너) → 패널(콘텐츠/문항/피드백)의 3단 구조로 조립됩니다.
```
ModulesPage (/modules/[lessonId])
 └ LessonSession (시퀀스 실행·결과 누적, usePassageMode)
    └ ModuleRunner key={currentModuleIdx}
       ├ ProcessWritingFlow            [PWR 전용] StepIndicator/ContentPanel/WritingPanel/FeedbackPanel
       └ ModuleRunnerInner            [PRD·SHR·SWR 등]
          ├ ModuleProgressBar (다중 모듈 시)
          ├ [Desktop lg+] 좌: ContentPanel+QuestionsPanel/VoiceQuestionPanel / 우: FeedbackPanel+AIQuestionPanel
          └ [Mobile <lg] 탭 Content/Questions/Voice + MobileSplitLayout + FeedbackPanel+AIQuestionPanel
```
- 주요 패널: ContentPanel(지문) · QuestionsPanel(문항) · VoiceQuestionPanel(음성) · FeedbackPanel(AI 채팅·피드백) · AIQuestionPanel(자유 질문) · ModuleProgressBar · ModuleCompleteCard · TypewriterText

### 데이터드리븐 렌더링 (IA의 핵심 설계 — uiTemplate switch 제거 → DB 필드 2개)
**모듈 종류마다 화면 컴포넌트를 따로 만들지 않고, DB 필드 2개의 조합으로 화면 동작을 결정**합니다. 신규 모듈은 코드 변경 없이 DB 등록만으로 동작합니다.
- **`passageMode`**(지문 패널 동작): `full` / `hidden` / `preview-then-reveal` / `timed-then-blur`
- **`questionFlowMode`**(문항 패널 동작): `sequential` / `deck`(카드 1장+시도 dots) / `locked-steps`(전체 표시+🔒)
- 두 필드는 독립 조합 가능(DB `ai_modules` 컬럼)

### timed-then-blur 학습 화면 (SCN/SKM — 속독·스캐닝)
지문을 일정 시간만 보여주고 가리는 훈련 화면. `blurActive`: `before`=blur → `reading`=공개 → `done+미제출`=blur 복원 → `done+제출`=해제. Floating Timer: `before`=전체 overlay(중앙 "지문읽기") / `reading`=헤더 우측 inline("읽기완료")

### 문항 입력 타입
`multiple-choice`, `short-text`(Enter 제출), `essay`/`sentence-write`(3줄 textarea), `audio-record`(음성), `keyword-chips`(칩 일괄 제출, SCN)

### 모듈 목록
| 코드 | 이름 | 답변유형 | 스킬 | 채점 | passageMode |
|---|---|---|---|---|---|
| PRD | Prediction(예측) | essay | reading | holistic | preview |
| SHR | Shadow Reading(쉐도잉) | audio-record | speaking | pronunciation | full |
| SWR | Sentence Writing(영작) | sentence-write | writing | holistic | full |
| PWR | Process Writing(창작) | essay(4단계) | writing | holistic | full |
| SCN | Scanning | keyword-chips/short-text | reading | holistic | timed-then-blur |
| SKM | Skimming | 객관식 | reading | — | timed-then-blur |

- 어댑터 패턴: `src/lib/adapters/[코드]Adapter.ts` + `index.ts` 레지스트리 + `GenericAdapter.ts`(데이터드리븐). 계획 모듈: QAR, VCB, WSD, WRD, CLR 등

---

## 5. 백엔드 API / 데이터 (IA 뒷받침)
- `GET /lessons/enrolled-courses` — 수강 과정(access_codes UNION custom)
- `POST /lessons/create-custom` — 나만의 수업(지문 생성→analyzer 시퀀싱→트랜잭션 저장)
- `GET /lessons/me/level-estimate` — 학습이력 기반 CEFR 추정(JWT)
- `GET /lessons/:id/complete-summary` — 레슨 완료 요약
- `GET /common-codes/KPI_CATEGORY/items` — KPI 카탈로그
- `POST /lessons/register-accesscode`, `/auth/login|signup|me|logout`
- 핵심 테이블: `courses`, `course_lessons`(skill_modules JSON), `texts`, `ai_modules`(passageMode/questionFlowMode/pedagogyProfile), `access_codes`(공유), `module_histories`, `lesson_results`, `institution_settings`, `user_login_logs`(→백오피스 출석률, app_code='tutoring')

---

## 6. 미확인/제한 사항
- `/report` 화면은 전용 문서 없이 보호 경로로만 언급 — 세부 IA 미확인
- 실제 소스코드는 `picklass_docs` 저장소가 아닌 `tutoring.picklass.com` 워크스페이스. 위 구조는 문서 기술 기준
- `README.md`는 "빈 인덱스"라 명시하나 실제 다수 문서 존재(인덱스 미최신화)

---

## 7. 근거 문서
- `참고문서/picklass_docs/tutoring/docs/picklass-tutoring-planning.md` — 핵심 기획서(여정/아키텍처/모듈)
- `.../docs/modules-lessonId-20260325.md` — 학습화면 IA·컴포넌트·Orchestrator 규칙
- `.../docs/모듈렌더링데이터드리븐리팩토링_20260408.md` · `.../docs/20260410_튜터링_모듈학습_UI개선및버그수정.md`
- `.../docs/나만의수업_시퀀싱통합_20260425.md` · `.../docs/20260406_인증_로그인_액세스코드_개발계획.md`
- `.../features/레슨완료/레슨완료-페이지-기획.md`
- `.../features/수강현황/20260619_수강중_수강종료_구분_개발계획.md` · `.../features/수강현황/20260526_수강조회_통합_개발계획.md`
- `.../features/나만의수업/20260623_*.md`
- `.../architecture/20260408-0428_모듈렌더링_passageMode_uiTemplate_기능개발_개발계획.md` · `.../architecture/20260409-0512_오케스트레이터_피드백_힌트시스템_개발계획.md`
