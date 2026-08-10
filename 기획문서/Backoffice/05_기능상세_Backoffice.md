# Backoffice 기획서 §5 · 기능 상세 (User Story + AC)

> 참고문서(`참고문서/picklass_docs/backoffice/`) 기반. 화면 §4(BO-S-###), 정책 §6(BO-P-###), 예외 §7(BO-E-###) 참조.
> **AC**: Given/When/Then. `☐` 미검증.

---

## 0. 기능 목록 (Feature Index)

| 기능ID | 기능명 | 화면 | 정책 | 예외 | 상태 |
|---|---|---|---|---|---|
| BO-F-010 | 조직 목록 조회·필터 | BO-S-010 | BO-P-004, BO-P-005 | BO-E-004, BO-E-005 | ✅ |
| BO-F-011 | 조직 등록(플랜 자동채움) | BO-S-011 | BO-P-010, BO-P-013 | BO-E-001, BO-E-002 | ✅ |
| BO-F-012 | 조직 수정 | BO-S-013 | BO-P-010 | BO-E-001 | ✅ |
| BO-F-013 | 기관 수업설정 편집 | BO-S-013 | BO-P-023 | BO-E-001 | ✅ |
| BO-F-020 | 사용자 목록 조회·필터 | BO-S-020 | BO-P-004, BO-P-011 | BO-E-005 | ⚠️ Scope UI만 |
| BO-F-021 | 사용자 등록 | BO-S-021 | BO-P-010 | BO-E-002 | ✅ |
| BO-F-022 | 사용자 수정 | BO-S-022 | — | BO-E-001 | ✅ |
| BO-F-023 | 자동 로그인(handoff) | BO-S-020 | BO-P-012 | BO-E-007 | ✅ |
| BO-F-024 | 아이디 일괄생성 | BO-S-023 | BO-P-010 | BO-E-003 | ✅ |
| BO-F-030 | 과정 목록 조회·필터 | BO-S-031 | BO-P-004, BO-P-032, BO-P-033 | — | ✅ |
| BO-F-031 | 과정 상세 조회 | BO-S-032 | BO-P-006 | BO-E-006 | ✅ |
| BO-F-032 | 과정 목록 내보내기 | BO-S-031 | — | — | ✅ |
| BO-F-033 | 액세스코드 목록·필터 | BO-S-033 | BO-P-004, BO-P-020 | — | ✅ |
| BO-F-034 | 액세스코드 생성 | BO-S-034 | BO-P-010, BO-P-020 | BO-E-001, BO-E-003 | ✅ |
| BO-F-035 | 액세스코드 상태전환 | BO-S-033 | BO-P-020 | BO-E-008 | ✅ |
| BO-F-036 | 과정 카테고리 관리(L1/L2) | BO-S-035 | BO-P-030 | BO-E-011 | ⚠️ 일부 미연동 |
| BO-F-037 | 과정 카테고리 배정 | BO-S-035 | BO-P-031 | BO-E-011 | ⚠️ bulk 미구현 |
| BO-F-040 | 기관 학습 현황(드릴다운) | BO-S-041 | BO-P-004, BO-P-005 | BO-E-004 | ✅ |
| BO-F-041 | 출석률 | BO-S-043 | BO-P-004 | — | ✅ |
| BO-F-042 | B2B 학습데이터 | BO-S-044 | BO-P-004 | — | ✅ |
| BO-F-043 | 플랫폼 현황 | BO-S-040 | BO-P-002 | — | ✅ |
| BO-F-050 | 코드관리 | BO-S-050 | — | BO-E-010 | ✅ |
| BO-F-051 | 수업모듈 관리 | BO-S-051 | — | — | ✅ |
| BO-F-052 | 시스템관리자 계정 | BO-S-054 | BO-P-002 | — | ✅ |
| BO-F-053 | API키관리 | BO-S-055 | — | — | ⚠️ 미연동 |
| BO-F-060 | API 연동 가이드 | BO-S-060 | BO-P-007 | — | ⚠️ Step1 |
| BO-F-061 | 내 연동 설정 | BO-S-061 | BO-P-002, BO-P-007 | — | 🔲 Step2 |
| BO-F-062 | 연동 테스트 | BO-S-062 | BO-P-002 | — | 🔲 Step3 |

---

## 1. 조직관리

### BO-F-011 · 조직 등록 (플랜 자동채움)
- **화면**: BO-S-011
- **User Story**: 관리자로서, 신규 파트너/그룹/기관을 온보딩하기 위해, 조직 정보와 요금제·계약을 등록하고 싶다.
- **동작**: 4섹션 — ①가입정보(아이디이메일[중복확인 필수]·임시비번·기관명·기관유형[개인학원/프랜차이즈/어학원/공교육/기업교육]·담당자·회원상태) ②부가정보(지점수·운영형태[직영/가맹]·수강생규모) ③라이선스·요금제 ④계약정보(계약상태[협의중/계약완료/계약중/만료/해지]·시작/종료일·자동갱신). 요금제 선택 시 **DEFAULT_PLANS**(Starter/Pro/Enterprise) 값 자동채움 후 편집 가능(BO-P-013). `POST /api/institutions`.
- **AC**:
  - ☐ Given 요금제 선택, Then monthlyPrice·baseStudents·pricePerStudent·maxAdminAccounts 등이 자동 채워지고 이후 수정 가능하다
  - ☐ Given 이메일 중복확인 미완료, When 등록, Then 차단·"이메일 중복 확인을 완료해주세요"(BO-E-002)
  - ☐ Given 계약 시작일 ≥ 종료일, Then 검증 실패(BO-E-001)
- **정책**: BO-P-010, BO-P-013 · **예외**: BO-E-001, BO-E-002

### BO-F-013 · 기관 수업설정 편집
- **화면**: BO-S-013 (institution type 상세 `[수업설정]` 탭)
- **User Story**: 기관 관리자로서, 소속 학생의 학습 정책을 통제하기 위해, 수업·FRT 정책을 설정하고 싶다.
- **동작**: `GET/PUT /institutions/:id/settings`. [과정운영] maxTotalClassMinutes(무제한/5~50분)·defaultCompletionSentences·defaultCompletionLessonRate(0~100%)·defaultCompletionMinutes·freeLearningEnabled. [FRT모듈] maxFrtModuleMinutes(무제한/5~30분)·allowedMicModes(PTT/Hybrid/Always on, **최소 1개 강제**)·micModeUserChange. → **Studio 과정생성 speaking_options 초기값으로 전달**(저장 후 독립, BO-P-023).
- **AC**:
  - ☐ Given 마이크 모드 전체 해제, Then 저장 불가(최소 1개 강제)
  - ☐ Given 수업설정 저장, Then 이후 Studio 과정 생성 시 해당 값이 초기값·한도로 반영된다
  - ☐ Then 이 탭은 institution 유형에서만 노출된다(partner/group 미노출)
- **정책**: BO-P-023

---

## 2. 사용자 관리

### BO-F-021 · 사용자 등록
- **화면**: BO-S-021
- **User Story**: 관리자로서, 강사/학생/관리자 계정을 만들기 위해, 역할·소속·계정정보를 등록하고 싶다.
- **동작**: roleCode(기본 teacher)·institutionId·email(중복확인 `POST /users/check-duplicate`, 정규식 `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`)·임시비번(bcrypt rounds=10)·이름. statusCode 'active' 고정.
- **AC**:
  - ☐ Given 이메일 형식 위반, Then 검증 실패(BO-E-001)
  - ☐ Given 중복 이메일, Then "이미 사용 중인 이메일입니다"(BO-E-002)
  - ☐ Then 신규 계정은 즉시 active로 생성된다

### BO-F-023 · 자동 로그인 (handoff)
- **화면**: BO-S-020 (목록 행 액션)
- **User Story**: 관리자로서, 강사/학생 화면을 직접 확인하기 위해, 해당 계정으로 대신 로그인하고 싶다.
- **동작**: `POST /handoff` → redirectUrl 새 탭. teacher→Studio, student→Tutoring. **핸드오프 JWT 만료 60초 단명**, JWT_SECRET 3시스템 공유(BO-P-012).
- **AC**:
  - ☐ Given 강사 행, When 자동 로그인, Then Studio가 새 탭에서 로그인된 상태로 열린다
  - ☐ Given 핸드오프 토큰 60초 경과, Then 재사용 불가
  - ☐ Given 실패, Then "자동 로그인에 실패했습니다"(BO-E-007)
- **정책**: BO-P-012 · **예외**: BO-E-007

### BO-F-024 · 아이디 일괄생성
- **화면**: BO-S-023
- **User Story**: 관리자로서, 다수 계정을 빠르게 만들기 위해, 아이디를 일괄 생성하고 싶다.
- **동작**: userType·institution·idCount(**1~1000**)·customCode(4자리 대문자). 이메일 `{code소문자}{3자리일련}@{code소문자}.pick`(예 `abcd001@abcd.pick`), 일련번호=동일 도메인 기존수+1. **초기비번=아이디**(isTempPassword). registrationExpiry=+365일·usagePeriodDays=365 고정. `POST /access-codes`.
- **AC**:
  - ☐ Given idCount>1000, Then "최대 1,000개까지 생성 가능합니다"(BO-E-003)
  - ⚠️ **[버그]** 서비스 내부 `Math.min(count,100)` 클램핑 — UI 1000 입력해도 실제 100개만 생성(BO-E-003)
  - ☐ Then "N개의 {역할} 아이디가 생성되었습니다. 초기 비밀번호는 아이디와 동일합니다" 표시

---

## 3. 콘텐츠 플랫폼

### BO-F-030 · 과정 목록 조회·필터 (읽기전용)
- **화면**: BO-S-031
- **User Story**: 관리자로서, 플랫폼 과정 현황을 파악하기 위해, 조직·상태·레벨·장르로 필터링해 조회하고 싶다.
- **동작**: 컬럼 과정명·레벨·장르·총레슨수·수강인원·배포횟수·상태(in_use/archived)·소속기관(+타입 뱃지). 필터 상태·레벨·장르·검색 + 조직 cascading. **수강인원/배포횟수는 응답시점 access_codes 실시간 집계**(비정규화 컬럼 폐기, `resolveAccessCodeCounts` 배치, BO-P-033). `GET /courses`. 생성·편집은 **Studio 전담**(BO-P-032).
- **AC**:
  - ☐ Then 배포횟수=courseId 연결 코드 전체수, 수강인원=그중 statusCode='active'
  - ☐ Given 관리자 역할, Then 생성/편집/삭제 액션은 노출되지 않는다(읽기전용)
- **정책**: BO-P-004, BO-P-032, BO-P-033

### BO-F-034 · 액세스코드 생성
- **화면**: BO-S-034
- **User Story**: 관리자로서, 과정을 배포하기 위해, 액세스코드를 발급하고 싶다.
- **동작**: userType(강사/학생)·codeCount(**1~1000**)·expiryDate(오늘 이후)·usagePeriod(30/90/180/365)·selectedCourse(학생 필수). 코드형식 6자리 A–Z(I,O 제외)+2–9(0,1 제외)(32종). 사용시작/종료일 직접설정(미체크=자동계산, endDate=startDate+usagePeriodDays, 불일치 시 경고배너). `POST /access-codes`.
- **AC**:
  - ☐ Given 학생 유형, When 과정 미선택, Then "학생 선택 시 제공할 과정을 선택해주세요"(BO-E-001)
  - ☐ Given expiryDate 과거, Then "등록 만료일은 오늘 이후로 설정해주세요"
  - ☐ Given 생성, Then 코드 N건이 inactive로 생성된다
- **정책**: BO-P-010, BO-P-020 · **예외**: BO-E-001, BO-E-003

### BO-F-035 · 액세스코드 상태전환
- **화면**: BO-S-033
- **동작**: inactive→활성화, active→비활성화, completed/expired→버튼 없음(최종). `activate()`는 사전설정 usageStart/End 우선, 없으면 등록시점 자동계산(BO-P-020).
- **AC**:
  - ☐ Given completed/expired 코드, Then 상태 전환 버튼이 없다(BO-E-008)
  - ☐ Given active+usageEndDate 경과, Then 조회 시 `effectiveStatus()`가 expired로 반환(DB status_code는 불변)

### BO-F-036 · 과정 카테고리 관리 / BO-F-037 · 배정
- **화면**: BO-S-035 (PARTNER_ABOVE)
- **User Story**: 파트너 관리자로서, 과정을 체계적으로 분류하기 위해, L1/L2 카테고리를 관리하고 과정에 배정하고 싶다.
- **동작**: L1(대)/L2(중) 2-depth, `course_categories`(partner_id·depth CHECK(1,2)·parent_id·series 등). `GET/POST/PATCH/DELETE /course-categories`·`/reorder`. 배정: 과정은 **L2에만 배정**(L1 단독 금지, BO-P-031), **동일 파트너 카테고리만**. 삭제: L2 CASCADE, courses SET NULL.
- **AC**:
  - ☐ Given L1에만 배정 시도, Then BadRequest(BO-E-011)
  - ☐ Given 교차 파트너 카테고리 배정, Then 가드로 차단(BO-E-011)
  - ⚠️ `POST /courses/bulk-assign-category` 미구현
- **정책**: BO-P-030, BO-P-031 · **예외**: BO-E-011

---

## 4. 통계 / 리포트

### BO-F-040 · 기관 학습 현황 (3단 드릴다운)
- **화면**: BO-S-041 (ORG_ABOVE)
- **동작**: 평면 리스트(학습자×과정), 필터 상위파트너/상위그룹(HARD)·기관명·과정명·학습자아이디. 페이지 20행(limit 최대100). 드릴다운 `GET /stats/search-learners` → `/stats/learners/:studentId/lessons` → `/stats/lessons/:lessonId/modules`(FRT).
- **AC**:
  - ☐ Given 리스트 행, Then 레슨 상세 → 모듈 상세로 3단 드릴다운된다
  - ☐ Given academy_admin, Then 이 메뉴에 접근할 수 없다(ORG_ABOVE, BO-E-004)

### BO-F-041 · 출석률
- **화면**: BO-S-043 (ORG_ABOVE)
- **동작**: user_login_logs + access_codes 기준. **출석 = 해당일(KST) 로그인 기록**, 출석률 = 로그인일수 ÷ 조회기간 평일수(월~금) × 100. 앱필터(전체/tutoring/speaking). 색상 ≥80% 초록 / 60~79% 주황 / <60% 빨강 / null 회색. `GET /stats/attendance/*` 3종.
- **AC**:
  - ☐ Then 주말은 분모(평일수)에서 제외된다
  - ⚠️ user_login_logs 기록 이전 로그인은 소급 불가(안내 문구)

### BO-F-042 · B2B 학습데이터
- **화면**: BO-S-044 (SYS)
- **동작**: lesson_results 기준. 필터 기간(완료일)·과정명·레슨명(**300ms 디바운스, 조회 버튼 없음**). `GET /stats/b2b/results`(인메모리 페이지네이션). 외부 파트너 Pull `GET /b2b/results`는 Phase B-2 미시작.

---

## 5. 시스템 · 파트너 API (요약)

- **BO-F-050 코드관리**(BO-S-050/052): 공통코드·상수 관리(USER_STATUSES·ACCESS_CODE_STATUSES·ACCESS_CODE_DURATIONS·LEVEL_SYSTEM 18단계). 저장 시 트랜잭션 타임아웃 주의(BO-E-010).
- **BO-F-052 시스템관리자 계정**(BO-S-054): system_admin 별도 관리.
- **BO-F-053 API키관리**(BO-S-055): ⚠️ UI완료·백엔드 미연동(Phase G). Webhook 필드 Phase C.
- **BO-F-060 API 연동 가이드**(BO-S-060): 6섹션 정적 스펙(로그인인증/수강목록/이수완료Push/이전학습내역/학습결과조회/체크리스트), JWT exp 5분. Step2 현황카드 대기.
- **BO-F-061 내 연동 설정**(BO-S-061): `GET/POST/PUT /b2b/api-configs`·`PATCH /:id/status`. partner_admin은 조회전용(등록·수정 SYS 전용, BO-P-002). 🔲 Phase F-3.
- **BO-F-062 연동 테스트**(BO-S-062, SYS): 인증·수강목록 실호출 테스트. 🔲 Phase F-2 이후.
