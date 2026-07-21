# Speaking 기획서 §5 · 기능 상세 (User Story + AC)

> 참고문서(`참고문서/picklass_docs/speaking/`) 기반. 화면 §4(PKS-**), 정책 §6(SP-P-###), 예외 §7(SP-E-###) 참조.
> **AC**: Given/When/Then. `☐` 미검증. 상당수 기능이 **부분 실연동 / Mock / 외주범위**이므로 상태를 병기.

---

## 0. 기능 목록 (Feature Index)

| 기능ID | 기능명 | 화면 | 정책 | 예외 | 상태 |
|---|---|---|---|---|---|
| SP-F-010 | 홈 오늘의 학습 | PKS-HM-001 | P-006 | E-006 | ✅ 부분 |
| SP-F-011 | 홈 학습현황(이수율·발화량) | PKS-HM-001 | P-004 | E-006 | ✅ 부분(Mock 잔존) |
| SP-F-012 | 홈 스피킹 밸런스(레이더) | PKS-HM-001 | P-002 | E-006 | ✅ 사용자폴리곤/평균 Mock |
| SP-F-020 | 수업 과정·레슨 조회 | PKS-CL-001 | P-006 | E-001 | ✅ |
| SP-F-021 | 과정 선택/변경 | PKS-CL-002 | P-006 | — | ✅ |
| SP-F-022 | FRT 시작 | PKS-CL-001 | P-006, P-007 | E-005 | ✅ 생성/⚠️플레이어 |
| SP-F-023 | 과정 탐색 | PKS-CL-003 | P-006 | E-006 | ✅ 조건부 |
| SP-F-030 | 챌린지 발화량 통계 | PKS-CH-001 | P-004 | — | ✅ 헤드라인/⚠️시계열 Mock |
| SP-F-031 | 복습 퀴즈(문장 마스터리) | PKS-FL-004 | P-009 | E-003, E-004 | ⚠️ 스켈레톤 |
| SP-F-032 | 표현 노트 | PKS-FL-005 | P-009 | E-006 | ⚠️ 스켈레톤 |
| SP-F-033 | 배지·랭킹 | PKS-CH-002~003 | — | — | ⚠️ Mock |
| SP-F-040 | 리포트 종합 | PKS-RP-001 | P-004 | E-006 | ⚠️ P0 데이터 대기 |
| SP-F-041 | 약점 분석 | PKS-RP-001 | P-002 | — | ✅ |
| SP-F-042 | 레슨별 리포트 | PKS-RP-002/004 | — | — | ✅ |
| SP-F-043 | 1MP 갤러리 | PKS-RP-003/005 | — | E-006 | ⚠️ 준비중 |
| SP-F-050 | MY 프로필·레벨 | PKS-MY-001 | P-002 | — | ✅ |
| SP-F-051 | 수강 내역 | PKS-MY-003 | P-006 | E-006 | ✅ |
| SP-F-052 | 설정류(닉네임/볼륨/시간/알림/페르소나) | PKS-MY-002~009 | P-007, P-008 | — | ✅ |
| SP-F-060 | 정규 학습 5단계 | PKS-FL-002 | P-002 | E-003 | ⚠️ 외주범위 |
| SP-F-061 | FRT 대화 | PKS-FL-003 | P-005 | E-003 | ⚠️ 스켈레톤 |
| SP-F-070 | 레벨테스트 P-ALT | PKS-LV-002 | P-003 | E-005 | ⚠️ 빈 스텁 |
| SP-F-071 | 레벨테스트 게이팅 | PKS-HM-001 | P-001 | E-005 | ✅ 상태연동 |

---

## 1. 홈 탭 (PKS-HM-001)

### SP-F-010 · 오늘의 학습
- **User Story**: 학습자로서, 앱을 켜자마자 오늘 할 학습을 시작하기 위해, 이어서 할 레슨과 FRT 카드를 보고 싶다.
- **동작**: 카드 2종 세로 스택 + 빨강 play.
  - ① 정규 카드 = `GET /courses/in-progress`(`nextLessonId` + `lastCompletedAt desc`) → `inProgress[0]`. href = `nextLessonId` 있으면 `/learn-flow/pick?lessonId=`, 없으면 `/learn`. in-progress 0건/비로그인 시 숨김.
  - ② Talk with Pickle(FRT) = `canUseCustom`(active 과정 존재) 게이팅 → `POST /courses/my-curriculum/lessons` → `/player/frt?lessonId=`. 잠금 시 Lock+"수강 중인 과정 없음".
- **AC**:
  - ☐ Given in-progress 과정, Then 정규 카드에 제목·`Lesson {order}·{topic}`·진행바가 표시된다
  - ☐ Given in-progress 0건, Then 정규 카드가 숨겨진다
  - ☐ Given active 과정 없음, Then FRT 카드가 잠기고 "수강 중인 과정 없음"을 표시한다
  - ⚠️ `분(minutes)`은 Mock(API 미제공)
- **정책**: P-006 · **예외**: E-006

### SP-F-011 · 학습현황 / SP-F-012 · 스피킹 밸런스
- **동작**:
  - 이수율 바 = `GET /config/program`(`course_completion_pct.pct`, 실연동) + `N/전체강`(Mock)
  - 발화량 = `GET /challenge/utterance`(`monthly.current/target`, 실연동)
  - 스피킹 밸런스 5축(발음·유창성·문법·화용성·발화량) = 사용자 폴리곤 `GET /feedback/kpi-trends` 최신 스냅샷(실연동) + 기관 평균(`AVG_SNAPSHOT` Mock)
- **AC**:
  - ☐ Given 학습 데이터 없음, Then "학습 데이터가 쌓이면 스피킹 밸런스가 표시됩니다"
  - ⚠️ 이수율 개수·기관평균 폴리곤·기간칩은 Mock(백엔드 소스 부재)

---

## 2. 수업 탭 (PKS-CL-001)

### SP-F-020 · 과정·레슨 조회 / SP-F-021 · 과정 변경
- **User Story**: 학습자로서, 수강 중인 과정을 이어서 학습하기 위해, 현재 과정과 레슨 목록을 보고 선택하고 싶다.
- **동작**: `GET /lessons/enrolled-courses` → active 필터(미커스텀 + `usageEndDate` 미만료). 3섹션 split(active/expired/custom). 초기선택 = `nextLessonId` 있는 active 우선, `?courseId=` 우선. 과정명 탭 → PKS-CL-002(완료이력 `hasProgress` 과정만). Chapter 탭 → ChapterSheet(모듈 미리보기+이어하기). 레슨 상태 `done/active/todo`.
- **AC**:
  - ☐ Given 만료 과정, Then expiredCourses 섹션에 분리 표시된다
  - ☐ Given `?courseId=` 진입, Then 해당 과정이 선택된 상태로 열린다
  - ⚠️ 모듈 진입 라우트/플레이어(`/learn-flow/pick`)는 **외주개발 범위**

### SP-F-022 · FRT 시작
- **동작**: "시작하기" → 주제 입력 바텀시트 → `POST /lessons/frt`(`skill_modules:[FRT]`, topic 미전달 시 `'자유 주제'`) → `/player/frt?lessonId=`. 잠금(`canUseCustom=false`): amber 배너 "수강 중인 과정이 없어 이용이 제한됩니다."
- **AC**:
  - ☐ Given active 과정 없음, Then FRT 섹션이 잠기고 배너를 표시한다
  - ☐ Given 주제 미입력, Then `'자유 주제'`로 생성된다

### SP-F-023 · 과정 탐색 (PKS-CL-003)
- **진입 조건**: 파트너 카테고리 존재(`getCourseCategories().length>0`)해야 버튼 노출(픽클래스 기본 B2B 단일 access_code는 미노출).
- **동작**: `GET /course-categories`(L1+children 트리) + `GET /courses?l1CategoryId=&l2CategoryId=&search=&limit=30`(항상 `course_type IN ('speaking','integrated')` + 파트너 스코프, `student_count desc`). 이중 칩(L1→L2), 추천 상위 3개(검색없음+L1미선택, 현재 인기순 폴백). 검색 debounce 300ms.
- **AC**:
  - ☐ Given 카테고리 없음, Then 과정 탐색 버튼이 노출되지 않는다
  - ☐ Given 카테고리에 과정 없음, Then "해당 카테고리에 과정이 없습니다."

---

## 3. 챌린지 탭 (PKS-CH-001) · 문장 마스터리

### SP-F-030 · 발화량 통계
- **동작**: 주간·월간 세그먼트 토글(일간 제거), 막대 3-상태(filled 살몬 / zero 점 / empty dashed) + 평균선(empty 제외) + 헤드라인 `getUtterance()[period].current`(실API). series는 Mock. 랭킹 pill → PKS-CH-002.
- **AC**:
  - ☐ Given API 로딩/실패, Then 헤드라인은 `–`를 표시한다
  - ⚠️ series(WEEKLY/MONTHLY)는 `_mock/challengeMock.ts`

### SP-F-031 · 복습 퀴즈 (문장 마스터리, PKS-FL-004)
- **User Story**: 학습자로서, 배운 문장을 정착시키기 위해, 망각곡선에 따라 복습 퀴즈를 풀고 싶다.
- **동작**: 세션당 **6문장**(5~8), 문장별 1~2발화. 4단계(듣기→발화→평가→리워드). `GET /expressions/quiz?limit=6`(due) → `POST /expressions/:id/utterance {score?,tapMode?}` → 마지막 문장 → `POST /expressions/:id/complete-session`(망각곡선 `[1,2,4,7,16,35]`일 전진). 발음평가 Azure(tutoring `pronunciation.service` 차용). 종료 요약(발화수·레벨업·마스터·"발화량 챌린지 +N문장 반영").
- **AC**:
  - ☐ Given due=0, Then S1 "복습 완료" + 표현노트 유도, 세션 진입 차단
  - ☐ Given 문장 8회 발화 누적, Then Lv4(마스터) 도달(`mastery_target=8`, P-009)
  - ☐ Given 세션 이탈, Then 진행분은 즉시 POST 저장(complete-session만 누락 시 다음 진입 보정)
  - ☐ Given 탭 모드, Then 발화량 ×0.6 가중
- **정책**: P-009 · **예외**: E-003, E-004

### SP-F-032 · 표현 노트 (PKS-FL-005)
- **동작**: `GET /expressions?filter=all|unlearned|master`, `POST /expressions`(source='note'). 단일 문장 섀도잉은 세션 엔진 재사용. 저장 = 단일 `user_expressions`(source `learning/note/preset`).
- **AC**:
  - ☐ Given 표현 0개, Then 빈 상태를 표시한다

---

## 4. 리포트 탭 (PKS-RP-001)

### SP-F-040 · 종합 리포트 / SP-F-041 · 약점 분석
- **User Story**: 학습자로서, 내 실력 변화를 파악하고 다음 학습으로 연결하기 위해, Quality·Quantity·Growth를 보고 약점을 추천받고 싶다.
- **동작**: 조회기간(주/월/90일→30d 매핑). `GET /feedback/summary`·`/feedback/kpi-trends?period=`·`/feedback/quantity?period=`.
  - Quality: 발음·유창성·문법·화용 (모두 0~100 점수 통일)
  - Quantity: 발화(문장)·듣기(VLM 카운트)·섀도잉(SHD 카운트) — 동일 기간 기준(2026-06-24)
  - Growth: 첫↔마지막 델타(클라 계산), 약점분석 최저축 → `/learn` 추천
- **AC**:
  - ☐ Given 최저 KPI 축, Then 약점분석이 해당 학습(`/learn` 또는 `/challenge`)을 추천한다
  - ⚠️ "기관 평균 비교"는 API 부재로 리포트에서 제거됨
  - ⚠️ 1MP 갤러리(SP-F-043)는 **P0(별도 학습앱 산출물) 의존 → "준비중" 빈상태**

---

## 5. MY 탭 (PKS-MY-001)

### SP-F-050 · 프로필·레벨 / SP-F-051 · 수강 내역 / SP-F-052 · 설정류
- **동작**:
  - 헤더 히어로: 이름(`englishNickname ?? user.name`)+연필(MY-002), 레벨배지 `L{n}·18라벨(·파트너명)`(미진단 시 "레벨 미진단")+[레벨 테스트 리포트]("준비 중"), 수강권 드롭다운(→`/learn?courseId=`), 로그아웃
  - 설정 바텀시트: MY-002 닉네임(영문검증)·MY-005 볼륨(0–100 localStorage)·MY-007 학습시간·MY-008 학습알림(시각+방해금지)·MY-009 페르소나(Sarah/James/Mia)
  - API: `GET/PATCH /users/learning-profile`, `GET /config/level-mappings`
  - **수강내역** `/my/history`(PKS-MY-003): 단일 목록(카드=access_code=`enrollmentId`, 재수강 각 카드). 상태=`usage_end_date` 조회시점 계산(NULL/≥오늘=수강중 초록, <오늘=종료 회색). 클릭: 수강중→`/learn?courseId=`, 종료→`/feedback?enrollmentId=`. 이수율=`round(completed/total*100)`.
- **AC**:
  - ☐ Given 닉네임 비영문 입력, Then 검증 실패
  - ☐ Given 수강 내역 없음, Then "수강 내역이 없습니다"
  - ☐ Given 종료 카드 클릭, Then `/feedback?enrollmentId=`로 이동
  - ⚠️ 아바타·카메라·레벨테스트 리포트·공지사항 실데이터는 준비중

---

## 6. 학습 플레이어 · 레벨테스트

### SP-F-060 · 정규 학습 5단계 (PKS-FL-002)
- **동작**: Pick-Speak — PICK(1~2분)→LEARN(3~5)→DRILL(5~8)→APPLY(5~10, role-play/free-talking/presentation)→MEASURE(2~3). 총 18~22분. 9모듈 자동 시퀀싱(1~3개, 사용자 노출 없음). 재도전 임계 70%.
- **AC**:
  - ☐ Given DRILL 점수 <70%, Then 재도전이 제공된다
  - ⚠️ `learn-flow/*` 전부 스켈레톤 — **모듈 플레이어는 외주개발 범위**

### SP-F-070 · 레벨테스트 P-ALT / SP-F-071 · 게이팅
- **동작(게이팅)**: `GET /levels/status` → `{requiresTest, testType, reason, enforcement, hasNickname, hasLevel}`. 팝오버 [시작하기]→`/onboarding/level-test`, [나중에]→닫기(hard_block이면 카드 잠금 유지).
- **동작(P-ALT)**: P1 Vocab(4지선다 CAT) → P2 문장낭독(CAT) → P3 일상대화(6~10턴) → P4 주제발표(조건부). PT 완료 시 닉네임+레벨 동시 확보.
- **AC**:
  - ☐ Given 신규 사용자(진단 0건+레벨없음), Then hard_block → 수업 카드 잠금 + 수업 시작 403 `LEVEL_TEST_REQUIRED`(E-005)
  - ☐ Given soft_prompt, When [나중에], Then 24h 스누즈(localStorage)
  - ☐ Given P3 6축 평균 조건, Then P4 진입/생략이 결정된다(P-003)
  - ⚠️ `onboarding/level-test/page.tsx`는 현재 빈 스텁
- **정책**: P-001, P-003 · **예외**: E-005
