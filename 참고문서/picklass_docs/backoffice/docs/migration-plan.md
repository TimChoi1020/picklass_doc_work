# Picklass Background React → Backoffice 마이그레이션 플랜

> 작성일: 2026-03-10 (검토 완료: 2026-03-10)
> Source: `d:/Project/picklass-background-react` (Next.js 16 단독 프로젝트)
> Target: `d:/Project/picklass-backoffice/apps/admin/frontend` (Monorepo 내 Next.js 16)

---

## 1. 프로젝트 분석 요약

### 1.1 Source 프로젝트 (picklass-background-react)

| 항목 | 내용 |
|------|------|
| 프레임워크 | Next.js 16 (App Router), React 19, TypeScript |
| 스타일링 | Tailwind CSS 4 + globals.css (커스텀 CSS 변수) + **인라인 스타일 대량 사용** |
| 상태관리 | useState (로컬 상태만, 전역 상태 없음) |
| API 연동 | **없음** (모든 데이터가 Mock/하드코딩) |
| UI 라이브러리 | 없음 (순수 HTML + CSS 클래스) |
| 인증 | UI만 존재 (실제 구현 없음) |
| UX 패턴 | CRUD를 **별도 페이지**로 구현 (등록/수정 각각 독립 라우트) |

**Source 전체 페이지 목록 (14개):**

| # | 라우트 | 페이지 제목 | 유형 |
|---|--------|-------------|------|
| 1 | `/` | Home (랜딩) | 퍼블릭 |
| 2 | `/admin` | 관리자 대시보드 | 관리자 |
| 3 | `/admin/institute` | 기관관리 (목록) | 관리자 |
| 4 | `/admin/institute/register` | 기관 등록 (별도 페이지) | 관리자 |
| 5 | `/admin/institute/[id]/edit` | 기관 수정 (별도 페이지) | 관리자 |
| 6 | `/admin/users` | 사용자관리 (목록) | 관리자 |
| 7 | `/admin/users/register` | 강사/학생 신규등록 (별도 페이지) | 관리자 |
| 8 | `/admin/users/[id]/edit` | 강사/학생 정보수정 (별도 페이지) | 관리자 |
| 9 | `/admin/users/access-code` | 아이디 & 액세스코드 생성 (별도 페이지) | 관리자 |
| 10 | `/admin/accesscode` | 액세스코드 관리 (목록) | 관리자 |
| 11 | `/admin/accesscode/generate` | 액세스코드 생성 (별도 페이지) | 관리자 |
| 12 | `/admin/ai-modules` | 수업 모듈관리 (목록) | 관리자 |
| 13 | `/admin/ai-modules/register` | 수업 모듈 등록 (별도 페이지) | 관리자 |
| 14 | `/admin/system` | 시스템 관리 | 관리자 |

### 1.2 Target 프로젝트 (picklass-backoffice) 기 구현 현황

| 항목 | 내용 |
|------|------|
| 프레임워크 | Next.js 16 (App Router), React 19, TypeScript |
| 스타일링 | Tailwind CSS 4 + shadcn/ui + OKLch CSS 변수 |
| 상태관리 | Zustand (useAuthStore, useSidebarStore, useUIStore) |
| API 연동 | **완비** (fetchApi + 엔티티별 API 서비스) |
| UI 라이브러리 | shadcn/ui (17개 컴포넌트) + 공통 컴포넌트 10개 |
| 인증 | Zustand 기반 Auth Store (persist) |
| 백엔드 | NestJS + Prisma + PostgreSQL (완전 구현) |
| UX 패턴 | CRUD를 **모달**로 구현 (목록 페이지 내 FormModal) |

**Target 전체 페이지 목록:**

| # | 라우트 | 페이지 제목 | Source 대응 | 비고 |
|---|--------|-------------|------------|------|
| 1 | `/` | 랜딩 페이지 | Source #1 | ✅ 구현됨 |
| 2 | `/admin/dashboard` | 대시보드 | Source #2 | ✅ 구현됨 |
| 3 | `/admin/institute` | 기관관리 | Source #3,4,5 | ✅ 모달로 등록/수정 통합 |
| 4 | `/admin/users` | 사용자관리 | Source #6,7,8,9 | ✅ 모달로 등록/수정/액세스코드 통합 |
| 5 | `/admin/ai-modules` | AI 모듈관리 | Source #12 | ✅ 구현됨 |
| 6 | `/admin/ai-modules/register` | AI 모듈 등록 | Source #13 | ✅ 구현됨 |
| 7 | `/admin/billing` | Billing | — | ✅ Target에만 있음 (Source 없음) |
| 8 | `/admin/system` | 시스템관리 | Source #14 | ✅ 구현됨 |
| 9 | `/admin/learning-management` | 학습관리 | — | ✅ Target에만 있음 (Source 없음) |
| — | `/admin/accesscode` | 액세스코드 관리 | Source #10 | ❌ Target에 없음 |
| — | `/admin/accesscode/generate` | 액세스코드 생성 | Source #11 | ❌ Target에 없음 |

---

## 2. 근본적 UX 패턴 차이 ⚠️

> **핵심 결정 필요: Source는 "별도 페이지" 방식, Target은 "모달" 방식**

| 기능 | Source (별도 페이지) | Target (모달) |
|------|---------------------|--------------|
| 기관 등록 | `/admin/institute/register` (전체 화면 폼) | `institute/page.tsx` 내 FormModal |
| 기관 수정 | `/admin/institute/[id]/edit` (전체 화면 폼) | `institute/page.tsx` 내 FormModal |
| 사용자 등록 | `/admin/users/register` (전체 화면 폼) | `users/page.tsx` 내 관리자등록 FormModal |
| 사용자 수정 | `/admin/users/[id]/edit` (전체 화면 폼) | `users/page.tsx` 내 관리자등록 FormModal (수정모드) |
| ID&코드 생성 | `/admin/users/access-code` (전체 화면 폼) | `users/page.tsx` 내 액세스코드 생성 FormModal |
| 액세스코드 생성 | `/admin/accesscode/generate` (전체 화면 폼) | — (없음) |
| AI모듈 등록 | `/admin/ai-modules/register` (전체 화면 폼) | `/admin/ai-modules/register` (전체 화면 폼) |

**결정:** Target의 모달 방식 유지 (Source의 별도 페이지 방식으로 전환하지 않음)
- **이유:** Target의 모달 방식이 UX적으로 더 현대적이고, 이미 API 연동까지 완료됨
- **예외:** AI 모듈 등록은 양쪽 모두 별도 페이지이므로 그대로 유지

---

## 3. 페이지별 상세 대조 및 마이그레이션 작업

### 3.1 랜딩 페이지 (`/`)

| 상태 | ✅ 이미 구현됨 | 동기화 필요 |
|------|---------------|------------|

**섹션별 상세 비교:**

| 섹션 | Source | Target | 차이점 |
|------|--------|--------|--------|
| Nav | 로고 "🎓 Picklass" + 4개 링크 + 로그인/회원가입 | 동일 구조 | ✅ 동일 |
| Hero | 그라디언트 `#667eea→#764ba2`, "영어교육기관의 1:1 영어 수업 에이전트" | 동일 | ✅ 동일 |
| Features | 4개 카드 (🎬콘텐츠관리, 📚AI콘텐츠변환, 📊학생성과분석, 🔗LMS연동) | 동일 | ✅ 동일 |
| **Pricing** | **4개 플랜** (Lite 1.2만원, Pro 0.8~1.2만원, Enterprise 협의, Partnership 협의) | **2개 모델** (라이선스 모델 "맞춤형", 수익배분 모델 "30%") | ⚠️ **구조 다름** |
| Partners | 3개 카드 (출판사, 교육기관, 기업) | 동일 | ✅ 동일 |
| Demo | 2개 카드 (Studio 대시보드, 기관용 분석도구) | 동일 | ✅ 동일 |
| CTA | 동일 그라디언트, "귀 기관의 영어 교육을 혁신하세요" | 동일 | ✅ 동일 |
| Footer | 4열 (Picklass, 서비스, 정보, 연락처) | 동일 | ✅ 동일 |

**모달 비교:**

| 모달 | Source | Target | 차이점 |
|------|--------|--------|--------|
| LoginModal | 학습자/기관장 선택, 이메일&PW/소셜 선택, Google/Kakao/Naver | 동일 구조 | ✅ 동일 |
| SignupModal | 학습자/기관장 선택, 유형별 조건부 필드, 이용약관 | 동일 구조 | ✅ 동일 |
| ContactModal | 회사명, 담당자명, 이메일, 전화, 문의유형, 메시지 | 동일 구조 | ✅ 동일 |
| **AccessCodeModal** | **액세스코드 입력 모달 (DOM 직접제어)** | **없음** | ❌ **Target에 없음** |

**필요 작업:**
- [ ] **Pricing 섹션 동기화**: Source의 4개 플랜 카드(Lite/Pro/Enterprise/Partnership)로 변경 필요 → 현재 Target은 2개 모델(라이선스/수익배분)
- [ ] **AccessCodeModal 추가**: Source에 있는 액세스코드 입력 모달을 Target에 추가
- [ ] 나머지 섹션은 동일하므로 추가 작업 불필요

---

### 3.2 대시보드 (`/admin/dashboard`)

| 상태 | ✅ 이미 구현됨 | 미세 동기화 |
|------|---------------|------------|

**StatCard 비교:**

| # | Source | Target | 차이 |
|---|--------|--------|------|
| 1 | 등록된 기관: **15** | 등록된 기관: **15** (🏢) | ✅ 동일 |
| 2 | 등록된 학생: **1,250** | 등록된 학생: **1,250** (👥) | ✅ 동일 |
| 3 | 콘텐츠: **485** | 콘텐츠: **485** (📚) | ✅ 동일 |
| 4 | AI 모듈: **12.5K** | AI 모듈: **12.5K** (🤖) | ✅ 동일 |

**기관 목록 테이블 비교:**

| Source 컬럼 | Target 컬럼 | 차이 |
|------------|------------|------|
| (컴포넌트 내부) | 기관명, 학생수, 코스수, 상태, 플랜 | Target이 더 상세 |

**사용자 통계 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| UserStatistics (별도 컴포넌트, 차트 형태) | ProgressBar 리스트 (관리자5, 기관담당자45, 강사120, 학생1,080) | ⚠️ 표현 방식 다름 |

**AI 모듈 사용률 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| AIModuleUsage (별도 컴포넌트, 차트 형태) | ProgressBar 리스트 (QAR 95%, Grammar 98%, Speech 85%) | ⚠️ 표현 방식 다름 |

**필요 작업:**
- [ ] Source의 UserStatistics/AIModuleUsage 컴포넌트 디자인 확인 후 Target ProgressBar 스타일 동기화
- [ ] 대시보드 레이아웃 그리드 구조 확인 (Source: 4열→auto-fit vs Target: 4열→2열)

---

### 3.3 기관 관리 (`/admin/institute`)

| 상태 | ✅ 이미 구현됨 (API 연동 포함) |
|------|-------------------------------|
| 작업 | 폼 필드 세부 차이 동기화 |

**테이블 컬럼 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| 기관명 | 기관명 | ✅ |
| 관리자이메일 | — | ❌ **Target에 없음** |
| 요금제 | 요금제 | ✅ |
| 단가 | 단가(인) | ✅ |
| 현재학생수/최대학생수 | 사용자/최대허용수 | ✅ |
| 계약기간 | 시작일~종료일 | ✅ |
| 상태 (활성/휴회) | 상태 (StatusBadge) | ✅ |
| — | 작업 (수정/삭제) | Target에 추가됨 ✅ |

**등록 폼 필드 비교 (4단계):**

| Section | Source 필드 | Target 필드 | 차이 |
|---------|------------|------------|------|
| 가입정보 | **아이디(이메일) + 중복확인** | — | ❌ **Target에 없음** |
| 가입정보 | **초기 임시 비밀번호** | — | ❌ **Target에 없음** |
| 가입정보 | 기관명 | 기관명 | ✅ |
| 가입정보 | 기관 유형 | 기관 유형 | ✅ |
| 가입정보 | 담당자 성명 | 담당자 성명 | ✅ |
| 가입정보 | 담당자 연락처 | 담당자 연락처 | ✅ |
| 가입정보 | — | 담당자 이메일 | Target에 추가됨 |
| 부가정보 | 지점 수 | 지점 수 | ✅ |
| 부가정보 | 운영 형태 | 운영 형태 | ✅ |
| 부가정보 | 현재 수강생 규모 | 현재 수강생 규모 | ✅ |
| 라이선스 | 플랜 | 플랜 | ✅ |
| 라이선스 | 학생당 단가 (자동계산) | 학생당 단가 (자동계산) | ✅ |
| 라이선스 | 최대 허용 학생수 | 최대 허용 학생수 | ✅ |
| 라이선스 | 허용 관리계정수 | 허용 관리계정수 | ✅ |
| 라이선스 | 청구 주기 | 청구 주기 | ✅ |
| 라이선스 | 연납 할인율 | 연납 할인율 | ✅ |
| 라이선스 | API 연동비 (Pro 전용) | API 연동비 (Pro 전용) | ✅ |
| 라이선스 | 기술지원비 | 기술지원비 | ✅ |
| 라이선스 | 콘텐츠 IP 전환비 | 콘텐츠 IP 전환비 | ✅ |
| 라이선스 | 기타 유의사항 | 기타 유의사항 | ✅ |
| 계약 | 계약 상태 | 계약 상태 | ✅ |
| 계약 | 계약 시작일 | 계약 시작일 | ✅ |
| 계약 | 계약 종료일 | 계약 종료일 | ✅ |
| 계약 | 자동갱신 여부 | 자동갱신 여부 | ✅ |
| 계약 | 갱신 조건 | 갱신 조건 | ✅ |

**수정 폼 추가 필드 (Source에만 있음):**
| Source | Target | 비고 |
|--------|--------|------|
| 아이디(이메일) — disabled | — | ❌ Target에 없음 |
| **회원 상태** (활성/휴회/정지/탈퇴) | — | ❌ **Target에 없음 (별도 필드)** |

**필요 작업:**
- [ ] Source의 "아이디(이메일) + 중복확인" 필드 추가 여부 결정
  - Source에서는 기관 등록 시 이메일 아이디를 함께 생성
  - Target에서는 기관과 사용자를 분리하여 관리 → 추가 불필요할 수 있음
- [ ] Source의 "초기 임시 비밀번호" 필드 추가 여부 결정 (위와 동일한 이유)
- [ ] Source의 수정 시 "회원 상태" 필드 추가 여부 결정
- [ ] 테이블에 "관리자이메일" 컬럼 추가 여부 결정

---

### 3.4 사용자 관리 (`/admin/users`)

| 상태 | ✅ 이미 구현됨 (API 연동 포함) |
|------|-------------------------------|
| 작업 | 폼 필드/버튼 세부 차이 동기화 |

**버튼 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| "+ 사용자 등록" → `/admin/users/register` 페이지 이동 | "+ 관리자등록" → 모달 | ⚠️ 역할 범위 다름 |
| "+ 아이디 & 액세스코드 생성" → `/admin/users/access-code` 페이지 이동 | "+ 강사/학생 액세스코드 생성" → 모달 | ⚠️ 버튼명/동작 다름 |

**테이블 컬럼 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| 역할 | 역할 (RoleBadge) | ✅ |
| 기관 | 소속 | ✅ |
| 이름 | 사용자명 | ✅ (라벨만 다름) |
| 아이디 | 아이디 | ✅ |
| — | 액세스코드 | Target에 추가됨 |
| 상태 | 상태 (StatusBadge) | ✅ |
| 등록일 | 활성날짜 | ⚠️ 라벨/값 다름 |
| — | 작업 (수정/삭제) | Target에 추가됨 |

**사용자 등록 폼 비교:**

| Source (`/admin/users/register`) | Target (관리자등록 모달) | 차이 |
|----------------------------------|-------------------------|------|
| 사용자 유형 라디오 (강사/학생) | 역할구분 Select (system_admin/academy_admin) | ⚠️ **대상 역할 다름** |
| 소속 기관 * | 소속기관 | ✅ |
| 이메일 * + 중복확인 | 아이디 * + 중복확인 | ✅ (라벨 다름) |
| 초기 임시 비밀번호 * | 비밀번호 * | ✅ |
| 이름 * | 이름 * | ✅ |
| — | 상태 * | Target에 추가됨 |

> **핵심 차이:** Source는 강사/학생을 등록 폼으로 직접 등록, Target은 관리자(system_admin/academy_admin)만 직접 등록하고 강사/학생은 액세스코드로 생성

**사용자 수정 폼 비교:**

| Source (`/admin/users/[id]/edit`) | Target (관리자등록 모달 수정모드) | 차이 |
|-----------------------------------|----------------------------------|------|
| 사용자 유형 (표시만) | 역할구분 (disabled) | ✅ |
| 소속 기관 | 소속기관 | ✅ |
| 이메일 (disabled) | 아이디 (disabled) | ✅ |
| 이름 | 이름 | ✅ |
| 상태 (활성/정지/탈퇴) | 상태 | ✅ |
| **비밀번호 초기화 체크박스** → 체크 시 비밀번호 필드 표시 | 비밀번호 변경 (항상 표시) | ⚠️ UX 다름 |

**아이디 & 액세스코드 생성 비교:**

| Source (`/admin/users/access-code`) | Target (액세스코드 생성 모달) | 차이 |
|--------------------------------------|-------------------------------|------|
| 사용자 유형 라디오 (강사/학생) | 역할구분 Select (teacher/student) | ✅ |
| 소속 기관 * | 소속 기관 * | ✅ |
| 생성할 아이디 개수 * (1-1000) | 액세스코드 생성 수 * (1-100) | ⚠️ 최대값 다름, **라벨 다름** |
| **고유코드 * (4자리)** | — | ❌ **Target에 없음** |
| **이메일 형식 미리보기** (`{code}001@{code}.pick`) | — | ❌ **Target에 없음** |
| 액세스코드도 함께 생성 (체크박스) | — (항상 생성) | ⚠️ 로직 다름 |
| — | **아이디 동시 생성 (체크박스)** | Source와 반대 구조 |
| — | **아이디 접두사 + 도메인** | 다른 형식 (`stu001@academy`) |
| — | **미리보기** | ✅ (다른 형식) |
| 등록 만료일 * | 등록 유효기간 * | ✅ |
| 사용 기간 (1개월/3개월/6개월/1년/무제한) | 사용가능기간 (일) (숫자 입력 1-365) | ⚠️ **입력 방식 다름** |
| 제공할 과정 (학생만) | — | ❌ **Target에 없음** |
| — | 코드상태 | Target에 추가됨 |

**필요 작업:**
- [ ] Source의 강사/학생 직접 등록 기능 → Target의 액세스코드 생성 모달로 처리 가능 (기존 유지)
- [ ] Source의 비밀번호 초기화 체크박스 UX → Target에 적용 여부 결정
- [ ] Source의 고유코드(4자리) + 이메일 형식 미리보기 기능 → Target 아이디 접두사+도메인 방식과 통합 여부 결정
- [ ] Source의 사용기간 Select(1개월/3개월/6개월/1년/무제한) → Target의 숫자 입력 방식과 통합 여부 결정
- [ ] Source의 "제공할 과정" 필드 Target에 추가 여부 결정
- [ ] 테이블 "등록일" vs "활성날짜" 라벨/값 동기화

---

### 3.5 액세스코드 관리 (`/admin/accesscode`) ⭐ 신규

| 상태 | ❌ Target에 없음 | 신규 생성 필요 |
|------|-----------------|---------------|

**Source #10: 액세스코드 목록 (`/admin/accesscode`)**

구성 요소:
- 페이지 헤더: "액세스코드 관리" + 설명 "관리자 / 기관담당 / 강사 / 학생 역할 및 권한 관리 (RBAC)"
- AccessCodeStatsCards (통계 카드)
- 카드: "액세스코드 사용 현황"
  - 버튼: "+ 액세스코드 생성" (주황색) → `/admin/accesscode/generate`
  - AccessCodeSearchFilter
  - AccessCodeTable
  - Pagination

테이블 컬럼:
| 컬럼 | 타입 |
|------|------|
| 코드 | string (6자리, 예: M4G6Q2) |
| 상태 | "사용" / "미사용" |
| 역할 | "강사" / "학생" |
| 기관 | string |
| 사용자명 | string |
| 만료일 | date |
| 사용기간 | string (예: "3개월") |
| 활성화일 | date |

Mock 데이터: 7개 레코드

**Source #11: 액세스코드 생성 (`/admin/accesscode/generate`)**

폼 필드:
| 필드 | 타입 | 옵션 |
|------|------|------|
| 사용자 유형 * | radio | 강사, 학생 |
| 소속 기관 * | select | 기술학원, 개발자학습소, 언어교육전문학원 |
| 생성할 액세스코드 개수 * | number | 1-1000 |
| 등록 만료일 * | date | — |
| 사용 기간 * | select | 1개월, 3개월, 6개월, 1년, 무제한 |
| 제공할 과정 (학생만) | select | 영어기초, 영어심화, 비즈니스영어, 전체과정 |

**필요 작업:**
- [ ] `(admin)/admin/accesscode/page.tsx` 신규 생성
  - StatCard 4개 (기존 컴포넌트 재사용)
  - SearchFilter (기존 컴포넌트 재사용)
  - DataTable (기존 컴포넌트 재사용)
  - Pagination (기존 컴포넌트 재사용)
  - API 연동: `getAccessCodes()` (이미 구현됨)
- [ ] 액세스코드 생성 UI 결정:
  - 옵션 A: FormModal로 구현 (Target 패턴 일관성 유지) ← 권장
  - 옵션 B: `/admin/accesscode/generate` 별도 페이지 (Source와 동일)
- [ ] `constants.ts` ADMIN_MENU에 "액세스코드관리" 메뉴 추가 (🔑 아이콘)

---

### 3.6 AI 모듈 관리 (`/admin/ai-modules`)

| 상태 | ✅ 이미 구현됨 | 상세 차이 존재 |
|------|---------------|---------------|

**통계 카드 비교:**

| Source (4개) | Target (3개) | 차이 |
|-------------|-------------|------|
| 총 모듈수 | — | ❌ |
| 활성 모듈수 | — | ❌ |
| 완료율 | — | ❌ |
| 평균 학습시간 | — | ❌ |
| — | Vocabulary: 1 (📝) | Source에 없음 |
| — | Reading: 2 (📖) | Source에 없음 |
| — | Speaking: 1 (🎤) | Source에 없음 |

> **차이:** Source는 전체 통계 (총수/활성/완료율/시간), Target은 스킬별 모듈 수

**모듈 테이블 비교:**

| Source 컬럼 | Target 컬럼 | 차이 |
|------------|------------|------|
| 스킬 (📚/📖/🎤) | 스킬 | ✅ |
| 모듈명 + 코드 | 모듈명 (code in muted) | ✅ |
| 수업 전 (checkbox) | 수업 전 (checkbox, disabled) | ✅ |
| 수업 중 (checkbox) | 수업 중 (checkbox, disabled) | ✅ |
| 수업 후 (checkbox) | 수업 후 (checkbox, disabled) | ✅ |
| 오픈 전 (checkbox) | 오픈 전 (checkbox, disabled) | ✅ |
| 오픈 후 (checkbox) | 오픈 후 (checkbox, disabled) | ✅ |
| 우선순위 | 우선순위 | ✅ |
| 상태 (활성/비활성) | 상태 (✓ 활성/⚠ 비활성) | ✅ |
| — | 관리 (편집 버튼) | Target에 추가됨 |

**모듈 수:** Source 16개 vs Target 13개 (Mock 데이터 차이)

**모니터링 테이블 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| 6개 항목 (entries, completions, dropoffs) | 2개 항목 (진입수, 완료수, 이탈수) | ⚠️ 데이터 양 다름 |
| — | "상세 →" 버튼 | Target에 추가됨 |

**필요 작업:**
- [ ] 통계 카드를 Source 스타일(전체통계)로 변경할지, Target 스타일(스킬별)을 유지할지 결정
- [ ] Mock 데이터 16개 → 13개 차이 확인 (Source 기준으로 동기화 필요 시 데이터 추가)
- [ ] 모니터링 테이블 Mock 데이터 6개 → 2개 동기화

---

### 3.7 AI 모듈 등록 (`/admin/ai-modules/register`)

| 상태 | ✅ 이미 구현됨 | 구조적 차이 존재 ⚠️ |
|------|---------------|---------------------|

**기본정보 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| 스킬 선택 * (6개) | 스킬 선택 * (6개) | ✅ 동일 |
| 모듈명 * | 모듈명 * | ✅ |
| 수업목적 * (textarea) | 수업목적 * (input) | ⚠️ 입력 타입 다름 |

**수업 설정 비교:**

| Source | Target | 차이 |
|--------|--------|------|
| 수업 구성 (Intro/Body/Closure) | 수업 구성 (Intro/Body/Closure) | ✅ |
| **그룹 선택 (6개 그룹)**: Starter(1-3), Beginner(4-6), Intermediate(7-9), Upper-Int(10-12), Advanced(13-15), Proficient(16-18) | **레벨 선택 (18개 개별)**: 레벨 1~18 | ⚠️ **구조 다름** |
| 지문 오픈 (전/후) | 지문 오픈 (전/후) | ✅ |
| 우선순위 (1~5) | 우선순위 (1~5) | ✅ |

> **핵심 차이:** Source는 6개 그룹(각 3레벨), Target은 18개 개별 레벨

**그룹/레벨 설정 비교:**

| Source (그룹별 설정, 6행) | Target (레벨별 설정, 18행) | 차이 |
|--------------------------|---------------------------|------|
| 학습 유형 (어휘/문장/지문) | 학습 유형 (어휘/문장/지문) | ✅ |
| 학습량 (수량) | 학습량(수량) | ✅ |
| — | **학습량(분)** | Target에 추가됨 |
| 지연시간 (1-30초) | 지연시간(초) | ✅ |
| 힌트 버튼 (checkbox) | 힌트 버튼 (checkbox) | ✅ |
| 재도전 옵션 + 기준(%) | 재도전 기준 + threshold(%) | ✅ |
| — | **완료 KPI (정답률/WPM/ASR)** | Target에 추가됨 |
| — | **동기화 체크박스 (전체 행 일괄 설정)** | Target에 추가됨 |

**콘텐츠 설정 비교:**

| Source (6행, 그룹별) | Target (18행, 레벨별) | 차이 |
|---------------------|----------------------|------|
| 콘텐츠 선정 | 콘텐츠 선정 | ✅ |
| 사전 힌트 로직 | 사전 힌트 로직 | ✅ |
| 1단계 안내 | 1단계멘트 | ✅ |
| 3단계 힌트 | 3단계멘트 | ✅ |
| 5단계 정답 | 5단계 정답 | ✅ |
| 5단계 오답 | 5단계 오답 | ✅ |
| 5단계 최종 | 5단계 최종오답 | ✅ |
| 5단계 피드백 | 5단계 피드백 | ✅ |
| 6단계 재도전 | 6단계 재도전 | ✅ |
| 7단계 완료 | 7단계 완료 | ✅ |
| 각 셀: Select (Seed/Slot/Seed+LLM/LLM) | 각 셀: Source Select + **조건부 파일업로드/프롬프트** | ⚠️ Target이 더 상세 |

**상태 선택:**

| Source | Target | 차이 |
|--------|--------|------|
| — | 상태 라디오 (활성/비활성) | Target에 추가됨 |

**필요 작업:**
- [ ] 그룹(6개) vs 레벨(18개) 구조 차이 결정:
  - 옵션 A: Target의 18개 레벨 방식 유지 (더 세밀한 제어) ← 권장
  - 옵션 B: Source의 6개 그룹 방식으로 변경
- [ ] Source의 수업목적 textarea → Target의 input 통일 여부
- [ ] Target의 추가 필드(학습량(분), 완료 KPI, 동기화 체크박스, 파일업로드/프롬프트) 유지

---

### 3.8 시스템 관리 (`/admin/system`)

| 상태 | ✅ 이미 구현됨 | 미세 차이 |
|------|---------------|----------|

**섹션 비교:**

| # | Source 섹션 | Target 섹션 | 차이 |
|---|-----------|------------|------|
| 1 | — | 시스템 정보 (Version, Status, Backup) | Target에 추가됨 |
| 2 | — | 주요 관리 기능 버튼 3개 | Target에 추가됨 |
| 3 | PlanManagementTable | 플랜관리 (EditableTable) | ✅ |
| 4 | UserStatusTable | 사용자상태 (EditableTable) | ✅ |
| 5 | CodeStatusTable | 액세스코드상태 (EditableTable) | ✅ |
| 6 | **AccessCodeDurationTable** | — | ❌ **Target에 없음** |
| 7 | SystemInfo | 시스템 정보 (위 #1에 통합) | ✅ |
| 8 | SystemLog | 시스템 로그 | ✅ |

**플랜관리 테이블 비교:**

| Source (PlanManagementTable 내부) | Target EditableTable 컬럼 | 차이 |
|----------------------------------|--------------------------|------|
| (별도 컴포넌트 내부 데이터) | 플랜명, 월비용, 연납할인(%), 기본제공수, 추가단가, 최대학생수, 최대관리계정, API연동, API비용, 사용여부 | Target이 더 상세 |

**필요 작업:**
- [ ] Source의 AccessCodeDurationTable 추가 여부 결정 (액세스코드 기간 설정 테이블)
- [ ] Source의 "코드관리" 소제목 추가
- [ ] 나머지는 Target이 이미 더 완성도 높으므로 유지

---

### 3.9 Target에만 있는 페이지 (Source에 없음)

| 페이지 | 라우트 | 결정 |
|--------|--------|------|
| **Billing** | `/admin/billing` | ✅ 유지 (Source에 없지만 Target 고유 기능) |
| **학습관리** | `/admin/learning-management` | ✅ 유지 (Source에 없지만 Target 고유 기능) |
| **Studio** (6개) | `/studio/*` | ✅ 유지 |
| **Tutoring** (7개) | `/tutoring/*` | ✅ 유지 |

> Source에 없는 페이지들은 Target의 확장 기능이므로 삭제하지 않고 유지

---

## 4. 스타일 동기화 상세 계획

### 4.1 컬러 시스템 동기화

| 용도 | Source (RGB) | Target (OKLch) | 동기화 방향 |
|------|-------------|-----------------|------------|
| Primary | `#4CAF50` / `#007AFF` | black | Source 값 반영 검토 |
| 상태-활성 | `#4CAF50` | `#4CAF50` | ✅ 동일 |
| 상태-비활성 | `#f44336` (빨강) | `#FFC107` (노랑) | ⚠️ Source 값으로 변경 |
| 상태-정지 | `#FF9800` (주황) | `#f44336` (빨강) | ⚠️ Source 값으로 변경 |
| 상태-탈퇴 | `#9E9E9E` | `#9E9E9E` | ✅ 동일 |
| 카드 그림자 | `0 2px 10px rgba(0,0,0,0.1)` | `0 1px 3px rgba(0,0,0,0.1)` | Source 값으로 변경 |
| border-radius | 8px | 10px | Source 값으로 변경 |
| 사이드바 활성 | `#667eea` (파란 배경) | `#f5f5f5` (연한 배경) + `#4CAF50` (초록 텍스트) | Source 값으로 변경 |

### 4.2 레이아웃 동기화

| 항목 | Source | Target | 동기화 |
|------|--------|--------|--------|
| 사이드바 너비 | 250px | 250px (w-64) | ✅ 동일 |
| 메인 max-width | 없음 | 1200px | Target 유지 |
| 폰트 | Segoe UI | Geist Sans | 결정 필요 |
| 모바일 대응 | 없음 | Sheet 드로어 | Target 유지 (개선사항) |

---

## 5. 네비게이션 동기화

### 5.1 사이드바 메뉴 비교

| # | Source | Target | 차이 |
|---|--------|--------|------|
| 1 | Dashboard (📊) | Dashboard (📊) | ✅ |
| 2 | 기관관리 (🏢) | 기관관리 (🚀) | ⚠️ 아이콘 |
| 3 | 사용자관리 (👥) | 사용자관리 (👥) | ✅ |
| 4 | **액세스코드관리 (🔑)** | — | ❌ 추가 필요 |
| 5 | 수업모듈 (🤖) | 수업 모듈 (🤖) | ✅ |
| 6 | Billing (💳) | Billing (💳) | ✅ |
| 7 | 시스템관리 (⚙️) | 시스템관리 (⚙️) | ✅ |

### 5.2 네비바 비교

| 항목 | Source | Target | 차이 |
|------|--------|--------|------|
| 타이틀 | "Picklass - 관리자 대시보드" | "Picklass Admin" | ⚠️ |
| 우측 링크 | Dashboard, 기관관리, 사용자관리, 수업모듈, Billing, 시스템관리, 로그아웃 | Studio, Tutoring, 로그아웃 | ⚠️ 다름 |
| 모바일 | 없음 | 햄버거 메뉴 + Sheet | Target 유지 |

**필요 작업:**
- [ ] ADMIN_MENU에 "액세스코드관리" 항목 추가 (사용자관리 아래)
- [ ] 기관관리 아이콘 🚀 → 🏢 변경
- [ ] 네비바 타이틀 동기화 여부 결정
- [ ] 네비바 우측 링크: Source 스타일(전 메뉴) vs Target 스타일(Studio/Tutoring) 결정

---

## 6. 실행 계획 (Phase별)

### Phase 1: 스타일 기반 동기화

| # | 작업 | 파일 | 난이도 |
|---|------|------|--------|
| 1-1 | globals.css 색상 변수 동기화 (상태 색상, 그림자, border-radius) | `globals.css` | 소 |
| 1-2 | StatusBadge 색상 동기화 (비활성: 빨강, 정지: 주황으로 교차 변경) | `status-badge.tsx` | 소 |
| 1-3 | 사이드바 활성 스타일 변경 (#667eea 파란 배경) | `app-sidebar.tsx` | 소 |

### Phase 2: 네비게이션 동기화

| # | 작업 | 파일 | 난이도 |
|---|------|------|--------|
| 2-1 | ADMIN_MENU에 "액세스코드관리" 추가, 아이콘 동기화 | `constants.ts` | 소 |
| 2-2 | 네비바 타이틀/링크 동기화 | `app-navbar.tsx` | 소 |

### Phase 3: 랜딩 페이지 동기화

| # | 작업 | 파일 | 난이도 |
|---|------|------|--------|
| 3-1 | Pricing 섹션: 2개 모델 → 4개 플랜 카드로 변경 | `pricing-section.tsx` | 중 |
| 3-2 | AccessCodeModal 추가 | 신규 컴포넌트 | 소 |

### Phase 4: 기존 관리자 페이지 세부 동기화

| # | 작업 | 파일 | 난이도 |
|---|------|------|--------|
| 4-1 | Dashboard 레이아웃/데이터 동기화 | `dashboard/page.tsx` | 소 |
| 4-2 | Institute 폼 필드 차이 결정 및 적용 | `institute/page.tsx` | 소 |
| 4-3 | Users 폼/버튼/테이블 동기화 | `users/page.tsx` | 중 |
| 4-4 | AI Modules 통계카드/테이블 동기화 | `ai-modules/page.tsx` | 소 |
| 4-5 | AI Modules Register 구조 결정 (그룹 vs 레벨) | `ai-modules/register/page.tsx` | 중~대 |
| 4-6 | System AccessCodeDurationTable 추가 여부 | `system/page.tsx` | 소 |

### Phase 5: 액세스코드 관리 페이지 신규 생성 ⭐

| # | 작업 | 파일 | 난이도 |
|---|------|------|--------|
| 5-1 | 액세스코드 목록 페이지 생성 (StatCard + SearchFilter + DataTable + Pagination) | `accesscode/page.tsx` (신규) | 중 |
| 5-2 | 액세스코드 생성 FormModal 구현 | 위 파일 내 또는 별도 | 중 |
| 5-3 | API 연동 (기존 access-code.ts 활용) | — | 소 |

---

## 7. 결정 필요 사항 체크리스트

실행 전 아래 항목에 대한 결정이 필요합니다:

| # | 결정 사항 | 옵션 A | 옵션 B | 권장 |
|---|----------|--------|--------|------|
| 1 | UX 패턴 | Source처럼 별도 페이지로 변경 | Target의 모달 방식 유지 | **B** |
| 2 | Pricing 섹션 | Source의 4개 플랜 카드 | Target의 2개 모델 유지 | **A** (Source 기준) |
| 3 | 기관 등록 시 이메일/비밀번호 | Source처럼 추가 | 현재처럼 기관만 등록 | **B** |
| 4 | 사용자 등록 방식 | Source처럼 강사/학생 직접 등록 | Target처럼 관리자만 직접, 강사/학생은 액세스코드 | **B** |
| 5 | 아이디 생성 형식 | Source: 고유코드4자리 + @pick | Target: 접두사 + @도메인 | **B** |
| 6 | 사용기간 입력 | Source: Select(1개월/3개월...) | Target: 숫자 입력(일) | 결정 필요 |
| 7 | AI모듈 레벨 구조 | Source: 6개 그룹 | Target: 18개 개별 레벨 | **B** |
| 8 | AI모듈 통계 카드 | Source: 전체통계(4개) | Target: 스킬별(3개) | 결정 필요 |
| 9 | 액세스코드 관리 | 독립 페이지 생성 | Users에 통합 유지 | **A** (신규 생성) |
| 10 | 폰트 | Source: Segoe UI | Target: Geist Sans | 결정 필요 |
| 11 | 상태 색상 교차 | Source 기준 (비활성=빨강, 정지=주황) | Target 유지 (비활성=노랑, 정지=빨강) | 결정 필요 |

---

## 8. 전체 페이지 커버리지 매트릭스

| Source 페이지 | Target 대응 | 커버 방식 | 신규작업 |
|-------------|------------|----------|---------|
| `/` 랜딩 | `/` | 기존 + Pricing 동기화 + AccessCodeModal 추가 | 소 |
| `/admin` 대시보드 | `/admin/dashboard` | 기존 + 스타일 미세 조정 | 소 |
| `/admin/institute` 목록 | `/admin/institute` | 기존 + 테이블 컬럼 동기화 | 소 |
| `/admin/institute/register` 등록 | `/admin/institute` (모달) | 기존 모달로 커버 + 필드 차이 결정 | 소 |
| `/admin/institute/[id]/edit` 수정 | `/admin/institute` (모달) | 기존 모달로 커버 + 필드 차이 결정 | 소 |
| `/admin/users` 목록 | `/admin/users` | 기존 + 테이블/버튼 동기화 | 소 |
| `/admin/users/register` 등록 | `/admin/users` (모달) | 기존 모달로 커버 (역할 범위 다름) | 소 |
| `/admin/users/[id]/edit` 수정 | `/admin/users` (모달) | 기존 모달로 커버 + PW초기화 UX | 소 |
| `/admin/users/access-code` 생성 | `/admin/users` (모달) | 기존 모달로 커버 + 필드 차이 결정 | 소~중 |
| `/admin/accesscode` 목록 | **신규 생성 필요** | StatCard+Filter+Table+Pagination | **중** |
| `/admin/accesscode/generate` 생성 | **신규 생성 필요** (또는 모달) | FormModal 또는 별도 페이지 | **중** |
| `/admin/ai-modules` 목록 | `/admin/ai-modules` | 기존 + 통계/테이블 동기화 | 소 |
| `/admin/ai-modules/register` 등록 | `/admin/ai-modules/register` | 기존 + 구조 차이 결정 | 소~중 |
| `/admin/system` 시스템 | `/admin/system` | 기존 + DurationTable 추가 여부 | 소 |

**커버리지: 14/14 (100%)** — 모든 Source 페이지가 플랜에 포함됨

---

## 9. 재사용 매트릭스

### 컴포넌트 재사용

| Target 컴포넌트 | Source 42개 중 대체 가능 | 비고 |
|----------------|------------------------|------|
| `DataTable<T>` | InstitutionTable, UsersTable, AccessCodeTable, AIModuleTable 등 | 제네릭 |
| `SearchFilter` | InstitutionSearchFilter, UserSearchFilter, AccessCodeSearchFilter | fields prop |
| `Pagination` | Pagination | ✅ |
| `StatCard` | StatCard, StatsCards 류 (5종) | ✅ |
| `StatusBadge` | 인라인 상태 표시 전부 | 4가지 상태 |
| `RoleBadge` | 인라인 역할 표시 전부 | 4가지 역할 |
| `FormModal` | InstitutionModal, UserRegistrationModal 등 | Dialog 기반 |
| `FormSection` | 폼 섹션 구분 전부 | 제목+그리드 |
| `EditableTable` | PlanManagementTable, UserStatusTable, CodeStatusTable | 인라인 편집 |
| `ProgressBar` | UserStatistics, AIModuleUsage | 커스텀 색상 |

### API 서비스 재사용

| API 서비스 | 함수 | 비고 |
|-----------|------|------|
| `user.ts` | getUsers, createUser, updateUser, deleteUser, checkDuplicateUserId, getUserDashboard | ✅ 완비 |
| `institution.ts` | getInstitutions, createInstitution, updateInstitution, deleteInstitution, getInstitutionDashboard | ✅ 완비 |
| `access-code.ts` | getAccessCodes, createAccessCodes, updateAccessCodeStatus, activateAccessCode, deleteAccessCode | ✅ 완비 |
| `code.ts` | getAllCodes, getCodesByGroup, getMultipleCodeGroups | ✅ 완비 |

---

## 10. 백엔드 추가 작업

| API | 현재 상태 | 필요 여부 |
|-----|----------|----------|
| Users CRUD + Dashboard + 중복확인 | ✅ 구현됨 | — |
| Institutions CRUD + Dashboard | ✅ 구현됨 | — |
| Access Codes CRUD + 활성화 | ✅ 구현됨 | — |
| Code Groups (공통 코드) | ✅ 구현됨 | — |
| **AI Module CRUD** | ❌ 미구현 | Prisma 스키마 + Core 서비스 필요 |
| **System Logs API** | ❌ 미구현 | 별도 트랙 |
| **Dashboard 통합 통계** | ❌ 미구현 | 별도 트랙 |

---

## 11. 리스크 및 주의사항

| # | 리스크 | 영향도 | 대응 |
|---|--------|--------|------|
| 1 | 상태 색상 교차 (Source↔Target) 변경 시 기존 UI 혼란 | 중 | 일괄 변경 + 테스트 |
| 2 | AI Module 백엔드 미구현으로 프론트엔드만 선행 | 고 | Mock 데이터로 우선 구현, 백엔드 병렬 개발 |
| 3 | Source 인라인 스타일 → Tailwind 변환 정확도 | 중 | 브라우저 픽셀 비교 |
| 4 | 결정 사항 11개 미결정 시 작업 지연 | 고 | 사전 결정 필수 |
| 5 | 아이디 생성 형식 차이 (고유코드 vs 접두사+도메인) | 중 | 하나로 통일 결정 |

---

## 12. 최종 요약

### 작업 규모

| 구분 | 신규 생성 | 수정 | 재사용 |
|------|-----------|------|--------|
| 페이지 | 1개 (`accesscode/page.tsx`) | 7개 (스타일+필드 동기화) | 전체 레이아웃 + 공통 컴포넌트 |
| 컴포넌트 | 1개 (AccessCodeModal) | 3~5개 (미세 조정) | 10+ 공통 컴포넌트 |
| API 서비스 | 0개 | 0개 | 4개 서비스 전부 |
| Store | 0개 | 0개 | 3개 전부 |
| CSS | 0개 | 1개 (globals.css) | Tailwind 설정 |

### 핵심 결론

1. **Target이 Source보다 완성도 높음** — API 연동, 상태관리, 타입 시스템, 반응형 디자인 모두 우수
2. **신규 페이지는 1개** (액세스코드 관리) + **신규 컴포넌트 1개** (AccessCodeModal)
3. **나머지 13개 Source 페이지는 Target의 기존 구현으로 커버** (모달 방식으로 통합)
4. **11개 결정 사항** 확정 후 실행 착수 권장
5. **Source의 별도 페이지(register/edit) 방식은 Target의 모달 방식으로 유지** (역방향 마이그레이션 불필요)
6. **Target에만 있는 페이지** (Billing, Learning Management, Studio, Tutoring)는 그대로 유지
