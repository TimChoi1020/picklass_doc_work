# Studio 기획서 §5 · 기능 상세 (User Story + AC)

> 참고문서(`참고문서/picklass_docs/studio/`) 기반. 화면은 §4(ST-S-###), 정책은 §6(ST-P-###), 예외는 §7(ST-E-###) 참조.
> **AC(인수조건)**: QA가 그대로 체크 가능한 Given/When/Then. `☐`는 미검증 항목.

---

## 0. 기능 목록 (Feature Index)

| 기능ID | 기능명 | 화면 | 정책 | 예외 | 상태 |
|---|---|---|---|---|---|
| ST-F-001 | 로그인 | ST-S-050 | P-024 | E-005 | ✅ |
| ST-F-002 | 회원가입 | ST-S-051 | — | E-001 | ✅ |
| ST-F-010 | 지문 목록 조회·필터 | ST-S-010 | P-023 | — | ✅ |
| ST-F-011 | AI 지문 생성 | ST-S-013 | P-012 | E-001 | ✅ |
| ST-F-012 | 지문 저장 | ST-S-013 | — | E-012 | ✅ |
| ST-F-013 | 지문 난이도 분석 조회 | ST-S-014 | — | — | ⚠️ 하드코딩 |
| ST-F-015 | 수업 준비(KPI 시퀀싱) | ST-S-011 | P-020 | E-004 | ✅ |
| ST-F-016 | 수업 진행 | ST-S-012 | — | — | ✅ |
| ST-F-020 | 과정 목록 조회·필터 | ST-S-020 | P-003, P-031 | — | ✅ |
| ST-F-021 | 과정 생성 마법사 | ST-S-022 | P-011~012, P-040~044, P-052 | E-006, E-010, E-012 | ✅ |
| ST-F-022 | 과정 액세스코드 생성(행 액션) | ST-S-020 | P-030 | E-007 | ✅ |
| ST-F-023 | 과정 편집(정보·운영·카테고리) | ST-S-023 | P-041~043, P-051 | E-010 | ✅ |
| ST-F-024 | 레슨 순서 이동 | ST-S-021 | — | — | ✅ |
| ST-F-025 | 레슨 모듈 수정 | ST-S-021 | P-031 | E-009 | ✅ |
| ST-F-026 | 레슨 추가 | ST-S-021 | — | E-011 | ✅ |
| ST-F-027 | 레슨 과정에서 제외 | ST-S-021 | P-031 | E-009 | ✅ |
| ST-F-030 | 학생 목록 조회·필터 | ST-S-030 | P-001 | — | ⚠️ mock |
| ST-F-031 | 학생 단건 등록 | ST-S-030 | P-013 | E-002 | ⚠️ mock |
| ST-F-032 | 학생 정보 수정 | ST-S-030 | P-032 | E-001 | ⚠️ mock |
| ST-F-033 | 학생 일괄 등록(+코드) | ST-S-030 | P-010, P-013 | E-002, E-003 | ⚠️ mock |
| ST-F-034 | 액세스코드 일괄 생성 | ST-S-031 | P-010, P-014, P-030 | E-003, E-007 | ✅ |
| ST-F-035 | 액세스코드 상태 전환 | ST-S-031 | P-030 | E-008 | ✅ |
| ST-F-040 | 리포트 목록 조회·필터 | ST-S-040 | P-003, P-015 | — | ✅ |
| ST-F-041 | 리포트 요약 카드 | ST-S-040 | — | — | ⚠️ 일부 DB대기 |
| ST-F-042 | 리포트 출력(다운로드) | ST-S-040 | — | — | ✅ |
| ST-F-043 | 학습자 상세 리포트 | ST-S-041 | — | — | ⚠️ Phase 3-B |

---

## 1. My Library — 지문

### ST-F-011 · AI 지문 생성
- **연관 화면**: ST-S-013 (지문 생성 모달)
- **User Story**: 강사로서, 수업 재료를 빠르게 확보하기 위해, 레벨·장르·길이·주제를 지정해 AI로 지문을 생성하고 싶다.
- **동작 상세**: 생성 모달에서 `cefrLevel · genre · wordCount · topic` 입력 → `POST /api/generate-text`(Gemini 2.5 Flash) → `{title, content, wordCount, actualWordCount}` 미리보기 → 저장(ST-F-012).
- **인수조건(AC)**:
  - ☐ Given 생성 모달, When 레벨·장르·단어수를 지정하고 생성, Then 지정 조건에 맞는 `title/content`가 미리보기에 표시된다
  - ☐ Then 응답 `actualWordCount`가 표시되어 목표 단어수와 대조할 수 있다
  - ☐ Given 생성 실패(API 오류), Then E-001 방식으로 오류 문구를 표시하고 모달을 유지한다
- **연관 정책**: P-012(콘텐츠 검증) · **연관 예외**: E-001

### ST-F-012 · 지문 저장
- **연관 화면**: ST-S-013
- **User Story**: 강사로서, 만든 지문을 재사용하기 위해, 라이브러리에 저장하고 싶다.
- **동작 상세**: `POST /passages` body `{title, content, level, category, topic, word_count, text_type}` — `text_type` = `A`(AI)/`T`(teamwork)/`U`(upload). 응답 `{id, ...}` → 목록 반영.
- **AC**:
  - ☐ Given 미리보기 상태, When 저장, Then 지문이 `text_type='A'`로 저장되고 지문 목록(ST-S-010) 최상단에 나타난다
  - ☐ Given 저장 경로 불일치 버그(§7 E-012), Then 저장 후 목록에 즉시 표시되어야 한다(현재 알려진 결함)

### ST-F-010 · 지문 목록 조회·필터
- **연관 화면**: ST-S-010
- **User Story**: 강사로서, 원하는 지문을 빨리 찾기 위해, 레벨·단어수·장르·제목으로 필터링하고 싶다.
- **동작 상세**: `GET /passages` query `search, level, genre, min_word_count, max_word_count, page, limit(20)` → `{data, total, totalPages}`. 필터: CEFR 18단계, 단어수 20구간(0~2000, 100단위), 장르, 최신순, 실시간 제목검색(대소문자 무관).
- **AC**:
  - ☐ Given 목록, When 필터/검색 변경, Then 결과가 갱신되고 **페이지가 1로 리셋**된다
  - ☐ Given 검색 결과 없음, Then 빈 상태를 표시한다(§7 참조)
- **연관 정책**: P-023(페이지네이션)

### ST-F-013 · 지문 난이도 분석 조회
- **연관 화면**: ST-S-014 (지문 상세 모달)
- **User Story**: 강사로서, 지문이 학생 레벨에 맞는지 판단하기 위해, 8개 난이도 지표를 보고 싶다.
- **동작 상세**: 상세 모달에 8지표(어휘난이도·어휘다양성·지문길이·문장길이·문장구조·문법다양성·정보밀도·배경지식의존도) 각 CEFR 배지. 데이터 소스 `texts.analysis`(JSONB).
- **AC**:
  - ☐ Given 지문 상세, Then 8지표가 각각 CEFR 값으로 표시된다
  - ⚠️ 현재 분석값 하드코딩 — 분석 API 연동 시 실데이터로 대체(구현 갭)

### ST-F-015 · 수업 준비 (KPI 기반 시퀀싱)
- **연관 화면**: ST-S-011 (`/class/lesson-setup/[passageId]`)
- **User Story**: 강사로서, 모듈을 일일이 배열하지 않고, 학습 목표(KPI)와 수업 시간만 정하면 시스템이 최적 모듈 순서를 짜주길 원한다.
- **동작 상세**:
  1. 지문 정보 카드 + KPI 멀티셀렉트(`GET /common-codes/items?groupCode=KPI_CATEGORY`, goal 텍스트만 노출) + 수업 시간 토글 `[15,20,25,30]분`
  2. **KPI ≥ 1개 AND duration ∈ {15,20,25,30}** 충족 순간 자동 `POST /lesson-plan {passageId, kpis[], duration}`
  3. 응답 `{sequence[], total_duration, filtered_from, warnings[]}` → 전체 모듈 표시하되 **선택 시 analyzer 순서대로 번호 자동 부여**(클릭 순서 무관)
  4. "수업 들어가기" → `/class/lesson/[passageId]?modules=…&kpis=…&duration=30`
- **AC**:
  - ☐ Given KPI 미선택 또는 시간 미선택, Then lesson-plan을 호출하지 않는다
  - ☐ Given KPI 1개+시간 선택, When 조건 충족, Then 자동으로 시퀀싱이 호출되고 로딩이 표시된다
  - ☐ Given 옵션 변경, When 이전 호출 진행 중, Then 이전 요청을 abort하고 재호출한다(race 방지)
  - ☐ Given 모듈 선택, Then 번호는 클릭 순서가 아닌 **시퀀스 인덱스+1**로 부여된다
  - ☐ Given 시퀀싱 실패, Then 모듈 선택·"수업 들어가기"가 차단된다(§7 E-004)
  - ☐ Given 모듈 ≥1 선택 + 시퀀싱 성공, Then "수업 들어가기"가 활성화된다
- **연관 정책**: P-020(시간 프리셋) · **연관 예외**: E-004

---

## 2. Course Hub — 과정

### ST-F-021 · 과정 생성 마법사
- **연관 화면**: ST-S-022
- **User Story**: 강사로서, 하나의 완성된 과정을 만들기 위해, 유형·생성방법·기본정보·회차구성을 단계적으로 입력하고 싶다.
- **동작 상세**:
  - **Step 0-A 유형**: 튜터링(`tutoring`, V/R/W) / 스피킹(`speaking`, S) / 통합(`integrated`, 전체) → `courses.course_type`
  - **Step 0-B 방법**: 주제 입력(`topic`) / 교재 PDF(`pdf`, **준비중·미구현**)
  - **Step 1/2 기본정보**: 과정명* · 과정목표(KPI) · 과정주제 · 회차수(1~100) · 레벨* · 단어수* · 장르* + **[과정 운영 설정]**(전체 수업 최대시간 `[10,20,30,40,60]`, 이수조건, 나만의 수업) → "다음" 시 `POST /ai/generate-topics` → `{topics[], courseFocus}`
  - **Step 2/2 회차 구성**: 회차별 topicSource(ai/lesson)·주제·단어수·장르·모듈 체크(유형 필터) + **[FRT 설정]**(`hasFrtModule` 시: FRT 시간, 마이크 모드)
  - **생성**: 회차마다 `POST /api/generate-text` → `POST /passages` → `POST /courses`(course_type, course_options, speaking_options, lessons[]). 진행 UI "지문 생성 중… (3/10)". 생성 후 상태 기본 `not-used`.
- **AC**:
  - ☐ Given Step 0-A, Then 유형에 따라 이후 모듈 선택 범위가 결정된다(P-052)
  - ☐ Given KPI 미선택, When "다음", Then `course_goal='일반 영어 학습'` 폴백으로 400을 방지한다(E-006)
  - ☐ Given 운영 설정이 기관 한도를 초과, Then 해당 프리셋이 disabled이며 서버가 400으로 재검증한다(P-041, P-043, E-010)
  - ☐ Given 회차 일부 지문 생성 실패, Then 해당 회차만 `text_id=null`로 저장되고 나머지는 정상 저장된다(E-012)
  - ☐ Then 통합(integrated) 과정은 회차 내 speaking 모듈이 항상 후순위로 배치된다(P-052)
- **연관 정책**: P-011~012, P-040~044, P-052 · **연관 예외**: E-006, E-010, E-012

### ST-F-023 · 과정 편집 (정보·운영·카테고리)
- **연관 화면**: ST-S-023
- **User Story**: 강사로서, 만든 과정을 다듬기 위해, 기본정보·운영설정·카테고리를 수정하고 싶다.
- **동작 상세**: `CourseEditModal` 탭 — 기본정보 / 운영설정(`OperationSettingsTab`, 기관설정 제약) / 카테고리(L1·L2 Select). 저장 `PATCH /courses/:id` → `course_options`·`speaking_options`(JSONB), `{title, course_focus, l1_category_id, l2_category_id}`.
- **AC**:
  - ☐ Given L1 미선택, Then L2 Select는 비활성이다(P-051)
  - ☐ Given L1 선택+L2 미선택, When 저장, Then `l2_category_id=null`(미분류 하위)로 저장된다
  - ⚠️ Given 운영설정 한도 초과, Then UI는 막지만 **편집 PATCH는 서버 재검증이 미이식**됨(P-043, 알려진 갭)
- **연관 정책**: P-041~043, P-051 · **연관 예외**: E-010

### ST-F-024~027 · 레슨 구성 (상세 화면 ST-S-021)
- **User Story**: 강사로서, 과정의 커리큘럼을 조정하기 위해, 레슨 순서·모듈·구성을 편집하고 싶다.
- **ST-F-024 순서 이동**: ↑/↓ 오버레이(항상 활성) → round 재계산. `PATCH .../lessons/reorder {ids[]}`
- **ST-F-025 모듈 수정**: `MODULE_CATEGORIES` 체크박스(0개 허용). `PATCH .../lessons/:id/modules`. **in-use 레슨은 disabled**(P-031, E-009)
- **ST-F-026 레슨 추가**: Step1 텍스트 선택(필터·라디오) → Step2 모듈 선택(≥1) → source 레슨 **앞에** splice 삽입 후 round 재계산. 추가 레슨 status 항상 `not-used`
- **ST-F-027 과정에서 제외**: `DELETE .../lessons/:id`. **in-use 레슨은 disabled**(P-031, E-009)
- **AC**:
  - ☐ Given 레슨 추가 Step1, When 텍스트 미선택, Then "다음" 불가(toast)
  - ☐ Given Step2, When 모듈 미선택, Then "추가" disabled(E-011)
  - ☐ Given in-use 레슨, Then 모듈 수정·제외 버튼이 disabled이고 ↑/↓ 이동만 가능하다

---

## 3. Enrollment — 학생·액세스코드

### ST-F-031 · 학생 단건 등록
- **연관 화면**: ST-S-030
- **User Story**: 강사로서, 개별 학생을 추가하기 위해, 아이디를 만들고 계정을 등록하고 싶다.
- **동작 상세**: 아이디 직접입력 또는 랜덤생성(`{STUDIO_CODE}{serial:3}@{STUDIO_CODE}.pick`) → **중복확인 필수** → 임시비밀번호+이름 → `POST /users {roleCode, institutionId, userId, tempPassword, name}`.
- **AC**:
  - ☐ Given 아이디 입력, When 중복확인 미통과/미실행, Then 등록 버튼이 차단된다(E-002)
  - ☐ Given 형식 `{4}{3}@{4}.pick` 위반, Then 등록 불가(P-013)
  - ☐ Then 신규 학생 상태는 `active`로 생성된다(P-032)
- ⚠️ 현재 mock/state — API 미연동

### ST-F-033 · 학생 일괄 등록 (+액세스코드)
- **연관 화면**: ST-S-030
- **User Story**: 강사로서, 반 전체를 한 번에 온보딩하기 위해, 다수 학생 계정과 액세스코드를 일괄 생성하고 싶다.
- **동작 상세**: 학생 유형 고정, 개수(1~1000, 일련번호 이어서), 이름 `신규학생{serial}` 자동, "액세스코드 함께 생성" 시 등록만료일+사용기간 필수. `POST /users/bulk {roleCode:'student', institutionId, count, withAccessCode, registrationExpiry?, usagePeriodDays?}` → `{users[], accessCodes?[]}`.
- **AC**:
  - ☐ Given 개수 <1 또는 >1000, Then E-003 오류(P-010)
  - ☐ Given "코드 함께 생성" 체크, When 등록만료일/사용기간 미입력, Then 제출 차단
  - ☐ Then 생성된 학생 아이디는 기존 일련번호에 이어서 부여된다

### ST-F-034 · 액세스코드 일괄 생성
- **연관 화면**: ST-S-031
- **User Story**: 강사로서, 과정을 배포하기 위해, 액세스코드를 원하는 개수만큼 발급하고 싶다.
- **동작 상세**: 모달 — 과정 선택(선택) · 개수 **1~1000** · 등록 만료일 · 사용 기간(`1/3/6/12/24개월`) · 사용 시작·종료일 직접설정(체크 시 종료일=시작+usagePeriodDays 자동, 수동 불일치 시 경고 배너). `POST /access-codes {count, registrationExpiry, usagePeriodDays, courseId?, usageStartDate?, usageEndDate?}` → `inactive` 생성.
- **AC**:
  - ☐ Given 개수 1~1000, When 생성, Then 코드 N건이 `inactive`로 생성되고 목록이 갱신된다
  - ☐ Given 개수 범위 위반, Then E-003 표시·생성 안 함
  - ☐ Given 코드 6자리 영대문자+숫자, Then 전역 유니크로 발급된다(P-014)
- **연관 정책**: P-010, P-014, P-030 · **연관 예외**: E-003, E-007

### ST-F-035 · 액세스코드 상태 전환
- **연관 화면**: ST-S-031
- **User Story**: 강사로서, 코드 사용을 통제하기 위해, 활성/비활성을 전환하고 싶다.
- **동작 상세**: 행 오버레이 → AlertDialog 확인 → `PATCH /access-codes/:id/status`. `inactive→active`(activated_at=현재) ↔ `active→inactive`(activated_at=null). `completed`/`expired`는 버튼 없음(최종).
- **AC**:
  - ☐ Given inactive 코드, When 활성화, Then status=active·activated_at 기록
  - ☐ Given active 코드, When 비활성화, Then status=inactive·activated_at=null
  - ☐ Given completed/expired 코드, Then 상태 전환 버튼이 없다(E-008)
- **연관 정책**: P-030 · **연관 예외**: E-008

> ST-F-022(과정 행 "액세스코드 생성")는 `POST /courses/:id/access-code` → 해당 과정 `inactive` 1건 + **deployment_count +1**(모달 일괄생성은 count 미증가).

---

## 4. Class Report — 리포트

### ST-F-040 · 리포트 목록 조회·필터
- **연관 화면**: ST-S-040
- **User Story**: 강사로서, 반 학습 현황을 파악하기 위해, 과정·기간·학생으로 필터링해 성과 테이블을 보고 싶다.
- **동작 상세**: 필터(과정[진행중 과정만] · 학습시작일 · 학습종료일 · 아이디/이름 · 정렬 · 초기화, 모두 AND) → `GET /reports {institution_id?, course_id?, student_search?, start_date_from?, end_date_to?, sort_by?, page?, limit?}`. 컬럼: 이름·과정명·수강기간(만료 7일 이내 빨간 강조)·진도(완료/전체)·레슨현황(O/△/-)·누적시간·Voca·Reading·Speaking·AI진단.
- **AC**:
  - ☐ Given 과정 필터, Then `usage_end_date >= TODAY` 과정만 선택지에 노출된다
  - ☐ Given 날짜 필터, Then `YYYY-MM-DD` 형식만 허용된다(P-015)
  - ☐ Given 결과 0건, Then "검색 결과가 없습니다" 빈 상태를 표시한다(§7)
  - ☐ Given 만료 7일 이내 수강, Then 수강기간이 빨간색으로 강조된다
- **연관 정책**: P-003, P-015

### ST-F-043 · 학습자 상세 리포트
- **연관 화면**: ST-S-041
- **User Story**: 강사로서, 개별 학생을 지도하기 위해, 레슨별 진도와 모듈별 KPI를 보고 싶다.
- **동작 상세**: 탭1 레슨별 진도 / 탭2 모듈별 KPI + FRT 대화 로그. (Phase3) `GET /reports/:studentId/lessons`, `GET /reports/:studentId/lessons/:lessonId/modules`.
- **AC**:
  - ☐ Given 학습자 상세, Then 탭1/탭2로 레슨·모듈 지표를 볼 수 있다
  - ⚠️ Phase 3-B(레슨별 상세 API + 차트 실데이터) 미구현

---

## 5. 인증 (참고)

### ST-F-001 · 로그인 / ST-F-002 · 회원가입
- **User Story**: 강사로서, 저작 도구를 쓰기 위해, 계정으로 로그인하고 싶다.
- **동작**: `/login`·`/signup`. 세션 `JWT_EXPIRES_IN=7d`, IDLE 7200s, keepalive 4분 주기 `/auth/me`(P-024).
- **AC**:
  - ☐ Given 세션 만료(401), Then 즉시 로그아웃이 아니라 **SessionExpiredDialog**로 안내한다(E-005)
