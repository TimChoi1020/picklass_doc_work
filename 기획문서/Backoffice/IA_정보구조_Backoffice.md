# Backoffice IA (정보구조)

> 참고문서(`참고문서/picklass_docs/backoffice/`)를 기반으로 정리한 백오피스 관리자 서비스의 정보구조 문서입니다.
> **SSOT**: `menu-redesign/20260527_메뉴구조_권한모델_개발계획.md` (코드 원본: `apps/admin/frontend/src/lib/constants.ts`의 `ADMIN_MENU`)

---

## 0. 서비스 개요 & IA 설계 배경

**Backoffice는 Picklass 플랫폼 전체를 운영·관리하는 통합 관리자(admin) 도구**입니다. 학습 콘텐츠를 만드는 Studio, 학습을 수행하는 Tutoring·Speaking 앱과 달리, 백오피스는 **조직·계정·콘텐츠·통계·시스템 설정을 관리하는 "운영 본부"** 역할을 합니다.

**IA를 이해하는 데 가장 중요한 두 축은 "관리 대상"과 "관리자 권한"입니다.**

**축 1 — 관리 대상별 대메뉴 구성.** 대메뉴는 관리자가 다루는 업무 객체를 기준으로 나뉩니다.
- **조직관리 / 사용자관리** → 누가(조직·사람)
- **콘텐츠 플랫폼** → 무엇을(지문·과정·코드)
- **통계/리포트** → 얼마나(학습 성과)
- **시스템 / API 로그 / 파트너 API 연동** → 어떻게(플랫폼 설정·연동)

**축 2 — 계층형 조직과 역할 기반 접근제어.** Picklass는 **플랫폼 > 파트너 > 그룹 > 기관**의 4계층 조직 구조를 가지며, 관리자도 자신이 속한 계층만큼만 볼 수 있어야 합니다. 그래서 **같은 메뉴라도 관리자 역할(system/partner/group/academy_admin)에 따라 "메뉴 자체가 안 보이거나(Tier 1)", "메뉴는 보이되 데이터 범위가 자기 조직으로 제한되는(Tier 2)"** 2단계 접근제어가 IA 전반에 걸려 있습니다. 이 권한 모델이 백오피스 IA의 가장 큰 특징이며, 과거 이메일 하드코딩(`restrictedToEmails`) 방식에서 역할 기반 모델로 전환되었습니다.

따라서 이 문서는 §1~2에서 메뉴·화면을, **§3에서 역할별 접근 구조를, §4에서 데이터 스코프(범위 제한) 구조를 별도로 다룹니다** — 이 두 절이 백오피스 IA의 핵심입니다.

---

## 1. 전체 메뉴 트리

범례
- ★ = Tier 1, `system_admin` 전용 (라우트 차단)
- 🤝 = PARTNER_ABOVE (`system_admin` + `partner_admin`)
- ⬢ = ORG_ABOVE (SYS·PRT·GRP, `academy_admin` 접근 불가)
- (무표기) = ALL_ADMIN (모든 관리자 노출, 데이터 범위만 제한)

| 대메뉴 | 소메뉴 | 경로 | 접근권한 | 상태 |
|---|---|---|---|---|
| **조직관리** | — | `/admin/institute` | ALL_ADMIN | ✅ Scope 연동 완료 |
| **사용자 관리** | — | `/admin/users` | ALL_ADMIN | ✅ (Scope UI만) |
| **콘텐츠 플랫폼** | 지문 관리 | `/admin/speaking-core-data` | ★ SYS | ✅ |
| | 과정 목록 | `/admin/courses` | ALL_ADMIN | ✅ |
| | 액세스코드 | `/admin/accesscode` | ALL_ADMIN | ✅ |
| | 과정 카테고리 | `/admin/content/categories` | 🤝 | ⚠️ UI완료·백엔드 미연동 |
| **통계 / 리포트** | 플랫폼 현황 | `/admin/stats/platform` | ★ SYS | ✅ |
| | 기관 학습 현황 | `/admin/stats/institutions` | ⬢ | ✅ |
| | 학습자 현황 | `/admin/stats/learners` | ALL_ADMIN | ✅ |
| | 출석률 | `/admin/stats/attendance` | ⬢ | ✅ |
| | B2B 학습데이터 | `/admin/b2b/results` | ★ SYS | ✅ |
| **시스템** ★ | 코드관리 | `/admin/system` | ★ SYS | ✅ |
| | 수업모듈 | `/admin/ai-modules` | ★ SYS | ✅ |
| | 수업모듈 코드관리 | `/admin/ai-modules/code-management` | ★ SYS | ✅ |
| | 스피킹 지문 프롬프트 | `/admin/speaking-passage-prompts` | ★ SYS | ✅ |
| | 시스템관리자 계정 | `/admin/system/admins` | ★ SYS | ✅ |
| | API키관리 | `/admin/b2b/api-keys` | ★ SYS | ⚠️ UI완료·백엔드 미연동(Phase G) |
| **파트너 API 연동** 🤝 | API 연동 가이드 | `/admin/partner/api-guide` | 🤝 | ⚠️ Step1 완료 |
| | 내 연동 설정 | `/admin/partner/settings` | 🤝 | 🔲 Step2 미착수 |
| | 연동 테스트 | `/admin/partner/test` | ★ SYS | 🔲 Step3 미착수 |
| **API 로그** ★ | 수강로그 | `/admin/b2b/enrollments` | ★ SYS | ✅ |
| | 로그인로그 | `/admin/b2b/members` | ★ SYS | ✅ |

> 3depth는 사이드바 메뉴가 아니라 페이지 내 **탭/드릴다운**으로 구현됩니다(§2 참조).

---

## 2. 메뉴별 기능 설명

### 조직관리 `/admin/institute`
**파트너·그룹·기관의 계층 조직을 등록·관리하고, 기관별 수업 운영 정책을 설정하는 화면**입니다. Picklass의 모든 데이터가 이 조직 계층에 소속되므로, 조직관리는 백오피스의 근간이 되는 메뉴입니다.
- **목록**: partner / group / institution 3계층을 한 화면에 혼합 표시, 계층 타입별 셀 채우기, 파트너·그룹 드롭다운 필터
- **상세** `/admin/institute/[id]`: 탭 구조 — partner·group은 `[기본정보]`만, institution은 `[기본정보]` `[수업설정]`
  - **수업설정 탭**(기관 단위 정책의 핵심): 과정 운영 설정(전체 수업 최대시간, 이수조건 기본값[발화량/레슨이수율/누적시간], 나만의수업 토글) + FRT 모듈 설정(FRT 최대시간, 허용 마이크 모드[PTT/Hybrid/Always on], 학습자 변경 허용) — `GET/PUT /institutions/:id/settings`. **여기서 정한 정책이 Studio 과정 생성 화면에 강제로 반영**됩니다.
  - 딥링크 `[이 기관의 과정 보기 →]` → type별 `/admin/courses?partnerId|groupId|institutionId=`
- (과거 [산하조직]·[관리자계정] 탭, `/settings` 독립 페이지는 제거됨)

### 사용자 관리 `/admin/users`
**모든 역할(관리자·강사·학생)의 계정을 생성·수정·조회하는 화면**입니다. 개별 등록뿐 아니라 아이디+액세스코드 일괄 생성으로 대량 온보딩을 지원하며, 강사/학생 계정으로 대신 로그인(handoff)해 문제를 진단할 수 있습니다.
- 하위 경로: `/register`(신규 등록), `/[id]/edit`(수정), `/access-code`(아이디 일괄 생성)
- 6종 역할 기반 CRUD, 강사/학생 자동 로그인(handoff) 진입, 아이디+액세스코드 일괄 생성
- 검색필터: 상위파트너 · 상위그룹 · 소속기관(HARD) · 역할 · 이름 · 아이디 · 상태
- 목록에서 `system_admin`은 백엔드에서 제외(`excludeSystemAdmin`)

### 콘텐츠 플랫폼
**플랫폼에 존재하는 학습 콘텐츠(지문·과정·코드)를 관리자 관점에서 관리·모니터링하는 메뉴군**입니다. 콘텐츠 "생성"은 Studio가 담당하므로 백오피스는 주로 조회·발급·정책성 관리를 맡습니다.
- **지문 관리** `/admin/speaking-core-data`: Speaking 앱의 핵심 데이터(지문 등) 관리(SYS 전용)
- **과정 목록** `/admin/courses`: **읽기 전용**(생성/편집은 Studio 전담이라는 역할 분리 원칙)
  - 컬럼: 과정명 / 레벨 / 장르 / 총 레슨수 / 수강인원 / 배포횟수 / 상태 / 소속기관
  - 필터: 상태 · 레벨 · 장르 · 검색 + 파트너/그룹/기관(role별) · `[다운로드 xlsx/csv]`
  - 상세 `/admin/courses/[id]` 탭: `[기본정보]` `[액세스코드(N)]`
- **액세스코드** `/admin/accesscode`: 과정 수강권한을 담은 코드를 대량 발급·관리. 생성 `/generate`(사용자유형[강사/학생], 개수 1~1000, 등록만료일, 사용기간[1M/3M/6M/12M], 과정 선택), 상태 inactive/active/completed/expired
- **과정 카테고리** `/admin/content/categories`: 파트너별 과정 분류 체계 관리(PARTNER_ABOVE, 파트너 LOCK)

### 통계 / 리포트
**학습 데이터를 조직·학습자·출석 관점으로 집계해 보여주는 분석 메뉴군**입니다. 대부분 "요약 리스트 → 드릴다운"으로 상세를 파고드는 구조이며, 외부 파트너(B2B) 연동 학습결과도 별도로 조회합니다.
- **플랫폼 현황** `/admin/stats/platform`: 플랫폼 전체 지표(SYS 전용)
- **기관 학습 현황** `/admin/stats/institutions`: 검색필터 + 평면리스트 + 페이지네이션(20행), **드릴다운 3단계** — 학습자×과정 → 레슨 상세 → 모듈 상세(FRT) — `/stats/search-learners`, `/stats/learners/:studentId/lessons`, `/stats/lessons/:lessonId/modules`
- **학습자 현황** `/admin/stats/learners`: 개별 학습자 단위 학습 지표
- **출석률** `/admin/stats/attendance`: **로그인 이력 기준** 출석(user_login_logs), 탭1 기관별(기관→학습자→월별 캘린더), 탭2 학습자별 캘린더, 앱 필터(전체/튜터링/스피킹) — `/stats/attendance/*` 3종
- **B2B 학습데이터** `/admin/b2b/results`: 외부 파트너 API로 연동된 학습결과, 자동검색(날짜 즉시·텍스트 300ms 디바운스) + 평면리스트 + 모듈 상세 — `GET /stats/b2b/results`

### 시스템 ★ (SYS 전용)
**플랫폼 전체에 영향을 주는 코드·모듈·프롬프트·관리자 계정 등을 다루는 최고 권한 메뉴군**입니다. 잘못 건드리면 전 서비스에 파급되므로 `system_admin`만 접근합니다.
- 코드관리 · 수업모듈(AI모듈 등록·관리) · 수업모듈 코드관리 · 스피킹 지문 프롬프트 · 시스템관리자 계정 · API키관리(파트너 API Key 발급/만료/Webhook, Phase G 백엔드 미연동)

### 파트너 API 연동 🤝
**B2B 파트너가 자사 시스템과 Picklass를 연동할 때 필요한 가이드·설정·테스트를 제공하는 포털**입니다. 파트너 담당자(partner_admin)가 직접 자신의 연동 상태를 확인할 수 있게 한 것이 특징.
- **API 연동 가이드** `/admin/partner/api-guide`: 내 연동 현황 카드 + API 스펙 문서(로그인 인증 / 수강목록 / 이수완료 Push / 이전 학습내역 / 학습결과 KPI 3종 / 체크리스트)
- **내 연동 설정** `/admin/partner/settings`: partner_admin=읽기전용 카드, system_admin=CRUD 테이블 + 모달 — `GET/POST/PUT /b2b/api-configs`, `PATCH /:id/status`
- **연동 테스트** `/admin/partner/test`(SYS): 인증 API + 수강목록 API 실호출 테스트

### API 로그 ★ (SYS 전용)
**외부 연동 API의 호출 이력을 감사(audit)하는 로그 메뉴**입니다. 문제 발생 시 원인 추적용.
- 수강로그 `/admin/b2b/enrollments`, 로그인로그 `/admin/b2b/members`

---

## 3. 권한/역할별 메뉴 접근 구조

> 백오피스 IA의 핵심. **"메뉴가 보이느냐(Tier 1)"와 "데이터가 얼마나 보이느냐(Tier 2)"의 2단계 제어**로 이해합니다.

### 역할 코드 체계
관리자는 자신이 소속된 조직 계층에 대응하는 역할을 가지며, 계층이 낮을수록 접근 범위가 좁아집니다.

| 코드 | UI 명칭 | 소속계층 | 백오피스 접근 |
|---|---|---|---|
| `system_admin` | 시스템관리자 | Platform | 전체 |
| `partner_admin` | 파트너 담당자 | Partner | Tier 2 |
| `group_admin` | 그룹 관리자 | Group | Tier 2 |
| `academy_admin` | 기관 관리자 | Institution | Tier 2 |
| `teacher` / `student` | 강사 / 학생 | Institution | 없음 |

### 2-Tier 권한 모델
- **Tier 1 — 메뉴 접근 차단(SYS ONLY)**: 애초에 메뉴가 안 보이고, URL을 직접 입력해도 `AdminRoleGate`가 라우트 수준에서 막아 `/admin/dashboard`로 리다이렉트. **"봐서는 안 되는 화면"에 적용.**
  - 대상: 시스템 대메뉴 전체, API 로그 대메뉴 전체, 통계 내 플랫폼 현황·B2B 학습데이터, 파트너 연동 테스트
- **Tier 2 — 데이터 범위 제한(ALL_ADMIN 노출)**: 메뉴는 모든 관리자에게 보이되, 검색 필터가 자기 조직으로 pre-fill + disable(고정)되어 남의 데이터를 볼 수 없음. **"화면은 공유하되 데이터는 격리"에 적용.**
  - 대상: 조직관리 / 사용자관리 / 콘텐츠 플랫폼 / 통계·리포트
- **PARTNER_ABOVE** `['system_admin','partner_admin']`: 과정 카테고리, 파트너 API 연동 — group_admin·academy_admin 차단
- **ORG_ABOVE** (SYS·PRT·GRP): 기관 학습 현황, 출석률 — academy_admin 접근 불가

### 구현 메커니즘
- `MenuItem.allowedRoles`가 단일 진실의 원천 (구 `restrictedToEmails` 이메일 하드코딩에서 전환)
- 사이드바: `app-sidebar.tsx`의 `filterByRole()` role별 재귀 필터링(안 보이는 메뉴 제거)
- 라우트 가드: `admin-role-gate.tsx`가 `ADMIN_MENU` 최장 경로 매칭으로 통제(URL 직접 접근 차단)
- 레이아웃: `AdminAuthGate`(로그인 여부) → `AdminRoleGate`(role 권한) 이중 래핑

---

## 4. 데이터 스코프 구조

> Tier 2의 실제 동작 방식. **로그인 시 결정된 관리자의 조직 위치(orgContext)에 따라 UI 필터가 잠기고, 백엔드가 쿼리를 자기 조직으로 강제 제한**합니다.

### 역할별 데이터 범위
| 역할 | 범위 |
|---|---|
| `system_admin` | 전체 |
| `partner_admin` | 자기 파트너 + 산하 모든 그룹·기관 |
| `group_admin` | 자기 그룹 + 산하 모든 기관 |
| `academy_admin` | 자기 기관 1개 |

### 필터 UI 렌더링 규칙 (`orgContext.fixedLevel` 기준)
자기 계층보다 상위 필터는 잠기고(LOCK), 하위는 선택 가능하게 렌더됩니다.

| 필드 | system_admin | partner_admin | group_admin | academy_admin |
|---|---|---|---|---|
| 파트너 | 드롭다운 | 🔒 LOCK | 🔒 LOCK | 🔒 LOCK |
| 그룹 | 드롭다운 | 드롭다운 | 🔒 LOCK | 🔒 LOCK |
| 기관 | 텍스트검색 | 텍스트검색 | 텍스트검색 | 🔒 LOCK |

### 아키텍처 (UI 고정 + 백엔드 강제의 이중 방어)
- 로그인 시 `GET /auth/me` → `getOrgContext()` → `UserOrgContext{partnerId/Name, groupId/Name, institutionId/Name, fixedLevel}`를 Zustand `useAuthStore`에 저장, 전 페이지 공유 (`fixedLevel: null`=무제한 | `'partner'` | `'group'` | `'institution'`)
- 백엔드 강제(핵심): `AdminAuthGuard`(JWT) → `resolveUserScope(userId, roleCode)`(재귀 CTE `resolveDescendants`로 산하 조직 전체 계산) → `null`(전체) 또는 허용 기관 ID 배열 → `WHERE ... IN (scope)`. **UI만 잠그면 우회 가능하므로 백엔드에서 재차 강제.**
- 엔티티별 scope 필드: Institution=`where.id`, User/Course/AccessCode=`where.institutionId`, 통계 Raw SQL=`AND institution_id = ANY(...)`
- PARTNER_ABOVE 페이지(과정 카테고리)는 **partner 단위** scope: `where.partnerId = req.partnerId`(단순 equality)
- 공통 쿼리 파라미터: `partnerId, groupId, institutionId, courseId, startDate, endDate`

### Scope 연동 현황
- ✅ 조직관리만 백엔드 연동 완료
- ❌ 사용자관리·과정목록·액세스코드·통계 페이지는 UI만 완료, 백엔드 scope 미연동
- 통계의 상위파트너/상위그룹 필터는 HARD 마커(UI 고정만, DB `parent_id` 미비)
- 단건 조회(`GET /:id`)는 현재 scope 확인 없음(알려진 갭)

---

## 5. 근거 문서
- `참고문서/picklass_docs/backoffice/menu-redesign/20260527_메뉴구조_권한모델_개발계획.md` — 메뉴 트리·권한모델 SSOT
- `.../menu-redesign/20260529_권한별-데이터-스코프-구현계획.md` — 데이터 스코프 구조
- `.../docs/report/20260525_학습결과_리포트_메뉴_개발계획.md` — 통계/리포트 소메뉴·화면·API
- `.../features/b2b/20260527_파트너API연동_포털_개발계획.md` — 파트너 API 연동 3화면
- `.../features/기관상세페이지/기관상세-탭구조-정리.md` — 조직/기관 상세 탭·수업설정
- `.../docs/courses/20260520_콘텐츠관리_역할분리_개발완료.md` — 과정 목록/상세, Studio vs 백오피스 역할 분리
- `.../docs/accesscode/accesscode.md` · `.../docs/users/users.md` — 액세스코드·사용자관리 화면
- `.../docs/index.md` — 랜딩/학습자 인증 흐름(관리자 메뉴 아님, 참고용)
