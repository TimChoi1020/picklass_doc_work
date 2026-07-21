# Picklass Playground → Backoffice Next.js 퍼블리싱 실행 계획

## Context
picklass-playground는 순수 HTML/CSS/JS로 구현된 교육 플랫폼 프로토타입이다.
이 프로토타입의 UI를 picklass-backoffice의 Next.js 16 프론트엔드(`apps/admin/frontend`)로 그대로 퍼블리싱한다.

**소스**: `D:\Project\picklass-playground` (HTML/CSS/JS 프로토타입)
**타겟**: `D:\Project\picklass-backoffice\apps\admin\frontend` (Next.js 16)

---

## Part 1: picklass-playground MD 문서 분석

### 1.1 admin/users.md (740줄) — 사용자관리 기능 명세

**핵심 개념**: 4가지 사용자 역할(시스템관리자, 학원관리자, 강사, 학생) RBAC 관리

| 기능 | 설명 |
|------|------|
| 사용자 검색 | 이름, 아이디, 역할, 소속, 상태 5개 필드 다중 필터링 |
| 액세스코드 | 6자리 영숫자 코드 생성 (I,O,0,1 제외 → 32자 풀) |
| 관리자 등록/수정 | 모달 폼 (아이디, 이름, 비밀번호, 역할, 상태) |
| 사용자 등록 | 별도 모달 (소속 기관 선택, 액세스코드 자동 생성) |
| 상태 관리 | 활성(#4CAF50), 비활성(#FFC107), 정지(#f44336), 탈퇴(#9E9E9E) |
| 역할 색상 | 시스템관리자(#8BC34A), 학원관리자(#2196F3), 강사(#FF9800), 학생(#9C27B0) |

**테이블 구조**: 역할 | 소속 | 사용자명 | 아이디 | 액세스코드 | 상태 | 활성날짜 | 작업

### 1.2 admin/billing.md (494줄) — 기관관리 빌링 명세

**핵심 개념**: 기관(학원)의 구독 현황, 요금제, 청구 정보 관리

| 기능 | 설명 |
|------|------|
| 요금제 | Lite(1.2만/명), Pro(0.8~1.2만/명, 구간할인), Enterprise(협의), 제휴 |
| 통계 대시보드 | 요금제별 가입 현황 + 월간 매출액 추이 |
| 기관 등록/수정 | 3섹션 모달 폼: ① 파트너 정보 ② 라이선스/요금제 ③ 계약 정보 |
| 기관 검색 | 기관명, 요금제, 상태별 필터링 |

**테이블 구조**: 기관명 | 요금제 | 단가(인) | 사용자 | 최대허용수 | 기간 | 상태 | 작업

### 1.3 Studio/lesson-management.md (940줄) — 레슨 관리 명세

**핵심 개념**: 지문(Text) → 모듈 선택 → 레슨(Lesson) 생성

| 구분 | 정의 |
|------|------|
| 지문(Text) | My Library에서 생성된 원본 콘텐츠 |
| 레슨(Lesson) | 지문 + 선택된 모듈의 조합 |
| 모듈(Module) | 어휘(5개) + 독해(7개) + 스피킹(6개) = 총 18개 |

**교수 모듈 18개**:
- 어휘(5): WRD, WSD, IMG, MG, WW
- 독해(7): PRD, SCN, SKM, QAR, CLR, SUM, ORL
- 스피킹(6): WSP, LR, SXP, SHD, SNR, FRT

**사용자 흐름**: 모듈 카테고리 선택 → 지문 검색(제목/레벨/어휘수/장르) → 모듈 선택(프로퍼티 기반 사전 선택) → 생성 방식 선택(레슨만 vs 레슨+과정)

**레슨 상태**: 사용 전(초록, 수정 가능) / 사용 중(주황, 수정 불가)

### 1.4 Tutoring/Reading/Player/README.md (337줄) — 레슨 플레이어 명세

**핵심 개념**: AI 튜터 "Pickle"과 학생의 상호작용 학습 환경

| 영역 | 기능 |
|------|------|
| 좌측 패널 | 지문 표시, 문제 섹션, 텍스트/음성 입력 |
| 우측 패널 | 통합 채팅 (수업 가이드, 피드백 3단계, 핵심 피드백 카드, 빠른 액션) |
| 텍스트 선택 | 팝업 메뉴 (음성 재생, 단어장 추가, 사전 보기) |
| TTS/STT | Web Speech API, 한/영 자동 감지 |

**대화 흐름**: 피클 가이드 → 학생 힌트 요청 → 피클 답변 → 학생 답변 제출 → 피클 피드백 → 핵심 피드백

**디자인 시스템**: 주색상 #7C5CFF(보라), 강조 #FF8A7A(코랄), #FFD666(노란), 성공 #6FCF97(초록)

---

## Part 2: HTML → Next.js 퍼블리싱 구현 계획

### 2.1 라우팅 매핑

```
src/app/
├── page.tsx                              ← index.html (역할 선택)
├── layout.tsx                            ← 루트 레이아웃
│
├── (admin)/                              ← Route Group (URL에 미포함)
│   ├── layout.tsx                        ← Admin Navbar + Sidebar
│   ├── dashboard/page.tsx                ← admin/index.html
│   ├── users/page.tsx                    ← admin/users.html
│   ├── billing/page.tsx                  ← admin/billing.html
│   ├── ai-modules/page.tsx               ← admin/ai-modules.html
│   ├── ai-modules/register/page.tsx      ← admin/ai-modules-register.html
│   ├── learning-management/page.tsx      ← admin/learning-management.html
│   └── system/page.tsx                   ← admin/system.html
│
├── (studio)/                             ← Route Group
│   ├── layout.tsx                        ← Studio Navbar + Sidebar
│   ├── studio-dashboard/page.tsx         ← Studio/index.html
│   ├── course-wizard/page.tsx            ← Studio/course-wizard.html
│   ├── my-library/page.tsx               ← Studio/my-library.html
│   ├── lesson-management/page.tsx        ← Studio/lesson-management.html
│   ├── course-management/page.tsx        ← Studio/course-management.html
│   ├── course-deployment/page.tsx        ← Studio/course-deployment.html
│   └── class-monitoring/page.tsx         ← Studio/class-monitoring.html
│
└── (tutoring)/                           ← Route Group
    ├── layout.tsx                        ← Tutoring Navbar + Sidebar
    ├── tutoring-dashboard/page.tsx       ← Tutoring/index.html
    ├── courses/page.tsx                  ← Tutoring/courses.html
    ├── mylearning/page.tsx               ← Tutoring/mylearning.html
    ├── profile/page.tsx                  ← Tutoring/profile.html
    ├── reading/page.tsx                  ← Tutoring/Reading/index.html
    ├── reading/reports/page.tsx          ← Tutoring/Reading/reports.html
    ├── reading/player/page.tsx           ← Tutoring/Reading/Player/lesson-player.html
    └── speaking/page.tsx                 ← Tutoring/Speaking/index.html
```

### 2.2 공통 컴포넌트 설계

#### shadcn/ui 설치 목록 (Phase 0)
```bash
pnpm dlx shadcn@latest add button input select label table card badge \
  dialog form checkbox textarea tabs pagination separator \
  dropdown-menu sheet tooltip progress avatar
```

#### 레이아웃 컴포넌트
| 파일 | 설명 |
|------|------|
| `components/layout/app-navbar.tsx` | 상단 네비게이션 (영역별 메뉴 props) |
| `components/layout/app-sidebar.tsx` | 좌측 사이드바 (usePathname 기반 active 자동) |

#### 비즈니스 공통 컴포넌트
| 파일 | 설명 | 사용처 |
|------|------|--------|
| `components/common/stat-card.tsx` | 그라디언트 통계 카드 | 모든 대시보드 |
| `components/common/data-table.tsx` | 범용 데이터 테이블 (컬럼 정의 기반) | users, billing, ai-modules 등 |
| `components/common/search-filter.tsx` | 검색 필터 바 (필드 정의 기반 동적 생성) | users, billing, my-library 등 |
| `components/common/pagination.tsx` | 페이지네이션 | 테이블 있는 모든 페이지 |
| `components/common/status-badge.tsx` | 상태 뱃지 (활성/비활성/정지/탈퇴) | users, billing |
| `components/common/role-badge.tsx` | 역할 뱃지 (관리자/강사/학생) | users |
| `components/common/form-modal.tsx` | 등록/수정 모달 래퍼 (Dialog 기반) | users, billing |
| `components/common/form-section.tsx` | 폼 섹션 구분 헤더 | 모달 내 폼 |
| `components/common/progress-bar.tsx` | 진행률 바 | dashboard, tutoring |
| `components/common/editable-table.tsx` | 인라인 편집 테이블 | system |

### 2.3 스타일 변환 전략

`globals.css`에 Picklass 커스텀 CSS 변수 추가:

```css
:root {
  /* 역할별 색상 */
  --role-system-admin: oklch(0.685 0.149 128.52);   /* #8BC34A */
  --role-academy-admin: oklch(0.567 0.174 248.09);  /* #2196F3 */
  --role-teacher: oklch(0.705 0.163 64.84);         /* #FF9800 */
  --role-student: oklch(0.468 0.219 302.97);        /* #9C27B0 */

  /* 상태별 색상 */
  --status-active: oklch(0.637 0.167 149.48);       /* #4CAF50 */
  --status-inactive: oklch(0.792 0.167 83.94);      /* #FFC107 */
  --status-suspended: oklch(0.536 0.245 27.33);     /* #f44336 */
  --status-withdrawn: oklch(0.631 0 0);             /* #9E9E9E */
}
```

변환 규칙:
- `common.css` 클래스 → Tailwind 유틸리티 클래스
- 인라인 `style=""` → Tailwind 클래스 또는 CSS 변수
- 그라디언트 → CSS 변수 기반 `style` prop

### 2.4 상태 관리

```
stores/
├── use-auth-store.ts       # 인증/사용자 정보 (localStorage 대체)
├── use-sidebar-store.ts    # 사이드바 열림/닫힘
└── use-ui-store.ts         # 공통 UI (모달, 토스트)
```

- **정적 페이지 (대시보드 등)**: 서버 컴포넌트
- **인터랙티브 페이지 (검색/모달/폼)**: 클라이언트 컴포넌트 (`"use client"`)
- **Mock 데이터**: `src/data/mock/` 에 TS 상수로 배치

### 2.5 구현 순서

#### Phase 0: 인프라 셋업 (~15파일)
1. shadcn/ui 컴포넌트 설치 (~15개)
2. `globals.css` 커스텀 색상 변수 추가
3. Zustand 스토어 3개 생성
4. Mock 데이터 파일 생성 (`data/mock/`)
5. `lib/constants.ts` 생성 (색상, 역할, 상태 매핑)

#### Phase 1: 공통 레이아웃 및 컴포넌트 (~20파일)
1. `AppNavbar` + `AppSidebar` 구현
2. 3개 Route Group 레이아웃 (`(admin)`, `(studio)`, `(tutoring)`)
3. 비즈니스 공통 컴포넌트 10개
4. 루트 `page.tsx` (역할 선택 페이지)

#### Phase 2: Admin 영역 (~25파일) — 단순→복잡 순서
| 순서 | 페이지 | 원본 크기 | 복잡도 |
|------|--------|----------|--------|
| 1 | dashboard | 8.5KB | 낮음 — 통계카드 + 테이블 |
| 2 | learning-management | 8.5KB | 낮음 |
| 3 | ai-modules | 18.2KB | 중간 — 체크박스 매트릭스 |
| 4 | system | 29KB | 높음 — 인라인 편집 테이블 3개 |
| 5 | users | 42.7KB | 높음 — 검색+테이블+모달 2개 |
| 6 | billing | 37.7KB | 높음 — 3섹션 폼 모달 |
| 7 | ai-modules/register | 158.7KB | 매우 높음 — 복합 등록 폼 (서브컴포넌트 분리) |

#### Phase 3: Studio 영역 (~30파일) — 단순→복잡 순서
| 순서 | 페이지 | 원본 크기 | 복잡도 |
|------|--------|----------|--------|
| 1 | studio-dashboard | ~10KB | 낮음 |
| 2 | class-monitoring | 8.8KB | 낮음 |
| 3 | course-deployment | 11.8KB | 중간 |
| 4 | course-wizard | 12.5KB | 중간 — 스텝 위자드 |
| 5 | my-library | 24.8KB | 중간 — CRUD |
| 6 | lesson-management | 93.6KB | 매우 높음 — 서브컴포넌트 분리 필수 |
| 7 | course-management | 100.9KB | 매우 높음 — 서브컴포넌트 분리 필수 |

#### Phase 4: Tutoring 영역 (~20파일) — 단순→복잡 순서
| 순서 | 페이지 | 원본 크기 | 복잡도 |
|------|--------|----------|--------|
| 1 | speaking | 1.8KB | 낮음 |
| 2 | mylearning | 3.2KB | 낮음 |
| 3 | profile | 3.3KB | 낮음 |
| 4 | courses | 3.4KB | 낮음 |
| 5 | tutoring-dashboard | 9.7KB | 낮음 |
| 6 | reading | 14.4KB | 중간 |
| 7 | reading/reports | 17.2KB | 중간 |
| 8 | reading/player | 47.6KB | 높음 — TTS/STT, 채팅 UI |

### 2.6 대형 파일 분해 전략

**158KB ai-modules-register.html**:
→ `components/admin/ai-module-register-form.tsx` + 3~5개 섹션 컴포넌트

**93KB lesson-management.html**:
→ `components/studio/lesson-list.tsx` + `lesson-editor.tsx` + `module-selector.tsx` + `passage-search.tsx`

**100KB course-management.html**:
→ `components/studio/course-list.tsx` + `course-editor.tsx` + `lesson-organizer.tsx`

### 2.7 최종 디렉토리 구조

```
apps/admin/frontend/src/
├── app/
│   ├── globals.css
│   ├── layout.tsx
│   ├── page.tsx                     # 역할 선택
│   ├── (admin)/layout.tsx + 7개 페이지
│   ├── (studio)/layout.tsx + 7개 페이지
│   └── (tutoring)/layout.tsx + 8개 페이지
├── components/
│   ├── ui/          (~15개, shadcn/ui)
│   ├── layout/      (2개: navbar, sidebar)
│   ├── common/      (10개: 공통 비즈니스)
│   ├── admin/       (~5개: Admin 전용 서브컴포넌트)
│   ├── studio/      (~5개: Studio 전용 서브컴포넌트)
│   └── tutoring/    (~3개: Tutoring 전용 서브컴포넌트)
├── data/mock/       (6개: JSON 데이터 → TS 상수)
├── hooks/           (2개: use-pagination, use-search-filter)
├── stores/          (3개: auth, sidebar, ui)
├── lib/
│   ├── api.ts       (기존)
│   ├── utils.ts     (기존)
│   └── constants.ts (신규: 색상/역할/상태 매핑)
└── types/index.ts   (프론트엔드 전용 타입)
```

### 2.8 예상 파일 수 총괄

| 범주 | 파일 수 |
|------|---------|
| shadcn/ui 컴포넌트 | ~15 |
| 레이아웃 컴포넌트 | 2 |
| 공통 비즈니스 컴포넌트 | 10 |
| Route Group 레이아웃 | 3 |
| Admin 페이지 + 서브컴포넌트 | ~25 |
| Studio 페이지 + 서브컴포넌트 | ~30 |
| Tutoring 페이지 + 서브컴포넌트 | ~20 |
| Zustand 스토어 | 3 |
| Mock 데이터 | 6 |
| Hooks, Constants, Types | 5 |
| **합계** | **약 120~130개** |

### 2.9 모바일 반응형 대응 전략

모든 페이지는 **모바일 퍼스트**로 퍼블리싱하며, Tailwind 브레이크포인트(`sm:640px`, `md:768px`, `lg:1024px`, `xl:1280px`) 기반으로 반응형 처리한다.

#### 브레이크포인트별 레이아웃 전환

| 구성 요소 | 모바일 (<768px) | 태블릿 (768~1023px) | 데스크톱 (≥1024px) |
|-----------|----------------|--------------------|--------------------|
| **사이드바** | 숨김 → Sheet(슬라이드) 토글 | 숨김 → Sheet 토글 | 고정 표시 (250px) |
| **Navbar** | 햄버거 메뉴 버튼 추가 | 햄버거 메뉴 버튼 추가 | 전체 메뉴 표시 |
| **통계 카드** | 1열 (`grid-cols-1`) | 2열 (`md:grid-cols-2`) | 4열 (`lg:grid-cols-4`) |
| **데이터 테이블** | 가로 스크롤 (`overflow-x-auto`) | 가로 스크롤 | 전체 표시 |
| **검색 필터** | 세로 1열 스택 | 2열 그리드 | 가로 1행 나열 |
| **모달** | 전체 화면 (`w-full h-full`) | 중앙 모달 (90% 너비) | 중앙 모달 (max-w-3xl) |
| **폼 필드** | 1열 스택 | 2열 그리드 | 2열 그리드 |
| **레슨 플레이어** | 좌/우 패널 세로 스택 | 좌/우 패널 세로 스택 | 좌(45%)/우(55%) 2열 |

#### 핵심 구현 방식

1. **사이드바**: shadcn/ui `Sheet` 컴포넌트 사용 — 모바일에서 햄버거 버튼 클릭 시 좌측 슬라이드 메뉴
   ```
   <div className="hidden lg:block w-[250px]">  ← 데스크톱 고정
   <Sheet>  ← 모바일 슬라이드
   ```

2. **테이블**: `<div className="overflow-x-auto">` 래핑 — 모바일에서 가로 스크롤 허용

3. **그리드 카드**: `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4` — 점진적 확장

4. **모달**: `DialogContent` className에 `w-full h-full md:w-auto md:h-auto md:max-w-3xl` — 모바일 풀스크린

5. **검색 필터**: `flex flex-col md:flex-row gap-2` — 모바일 세로, 데스크톱 가로

6. **폰트 크기**: 모바일에서 `text-sm`, 데스크톱에서 `md:text-base` 적용

7. **터치 대응**: 버튼/링크에 최소 `min-h-[44px] min-w-[44px]` 터치 영역 보장

#### Zustand 사이드바 스토어 활용

```
use-sidebar-store.ts:
- isOpen: boolean        ← 모바일 Sheet 열림/닫힘
- toggle(): void         ← 햄버거 버튼 클릭
- close(): void          ← 메뉴 선택 시 자동 닫힘 (모바일)
```

### 2.10 검증 방법

1. `pnpm --filter=@app/admin-web dev`로 프론트엔드 실행
2. `http://localhost:4100` 접속 → 역할 선택 페이지 확인
3. 각 영역(Admin/Studio/Tutoring) 진입 → 사이드바 메뉴 동작 확인
4. 각 페이지 UI가 playground HTML과 동일한지 시각적 비교
5. 모달 열기/닫기, 검색 필터, 페이지네이션 등 인터랙션 동작 확인
6. **모바일 반응형 검증**: 브라우저 DevTools에서 375px(iPhone SE), 768px(iPad), 1024px(Desktop) 3가지 뷰포트로 확인
7. 모바일에서 햄버거 메뉴 → 사이드바 Sheet 열림/닫힘 확인
8. 모바일에서 테이블 가로 스크롤, 모달 풀스크린 동작 확인

---

## Part 3: 실행 이력

### 실행 일자: 2026-03-04

### Phase 0: 인프라 셋업 ✅
- shadcn/ui 컴포넌트 17개 설치 (button, input, label, table, card, badge, dialog, checkbox, textarea, tabs, separator, dropdown-menu, sheet, tooltip, progress, avatar, select)
- `globals.css`에 Picklass 커스텀 CSS 변수 추가 (역할별 4색, 상태별 4색, 브랜드 6색)
- Zustand 스토어 3개 생성 (`use-auth-store.ts`, `use-sidebar-store.ts`, `use-ui-store.ts`)
- Mock 데이터 5개 파일 생성 (`users.ts`, `institutions.ts`, `courses.ts`, `ai-modules.ts`, `index.ts`)
- `lib/constants.ts` 생성 (역할/상태/요금제/모듈 정의, 3개 영역 사이드바 메뉴)
- `types/index.ts` 생성 (User, Institution, AIModule, Course, Lesson, Passage, SystemPlan, StatCardData)
- Hooks 2개 생성 (`use-pagination.ts`, `use-search-filter.ts`)

### Phase 1: 공통 레이아웃 및 컴포넌트 ✅
- `AppNavbar` 생성 — 반응형 네비게이션 (모바일 햄버거 + 데스크톱 전체 메뉴)
- `AppSidebar` 생성 — 데스크톱 고정(250px) + 모바일 Sheet 슬라이드
- Route Group 레이아웃 3개 생성 (`(admin)`, `(studio)`, `(tutoring)`)
- 루트 `page.tsx` 생성 — 역할 선택 페이지 (3개 그라디언트 카드)
- 공통 비즈니스 컴포넌트 10개:
  1. `stat-card.tsx` — 그라디언트 통계 카드
  2. `data-table.tsx` — 제네릭 데이터 테이블
  3. `search-filter.tsx` — 동적 필터 (text/select)
  4. `status-badge.tsx` — 상태 뱃지 (color-mix 사용)
  5. `role-badge.tsx` — 역할 뱃지 (color-mix 사용)
  6. `form-modal.tsx` — Dialog 기반 모달 래퍼
  7. `form-section.tsx` — 폼 섹션 구분 헤더
  8. `progress-bar.tsx` — 진행률 바
  9. `editable-table.tsx` — 인라인 편집 테이블
  10. `pagination.tsx` — 페이지네이션

### Phase 2: Admin 영역 ✅ (7개 페이지 + 4개 서브컴포넌트)
1. `/dashboard` — 통계 카드 4개, 기관 목록, 사용자 통계, AI 모듈 사용률
2. `/learning-management` — 코스별 학생 현황, 배포ID 집계, 진행률 분포, 주의 학생
3. `/ai-modules` — 모듈 체크박스 테이블 (13개 모듈), 사용 현황 모니터링
4. `/system` — 플랜 관리, 사용자상태, 액세스코드 상태 (인라인 편집), 시스템 로그
5. `/users` — 5개 필터 검색, 사용자 테이블+페이지네이션, 관리자/강사학생 등록 모달 2개
6. `/billing` — 요금제 현황, 매출 추이, 기관 테이블, 3섹션 등록 모달
7. `/ai-modules/register` — 5개 섹션 폼 (기본정보, 수업설정, 레벨별 18행 설정, 콘텐츠 10열x18행 설정, 상태)
   - 서브컴포넌트: `module-basic-info.tsx`, `module-lesson-settings.tsx`, `module-level-settings.tsx`, `module-content-settings.tsx`

### Phase 3: Studio 영역 ✅ (7개 페이지)
1. `/studio-dashboard` — 통계 카드, 교육 프로세스 4단계, 최근 코스, 수업 중 코스
2. `/class-monitoring` — 클래스 현황, 학생 모니터링, 주의 학생, 모듈별 성적
3. `/course-deployment` — 통계 카드, QR 코드 생성, 배포 코스 목록, 수강자 현황
4. `/course-wizard` — 레슨 회차, 레벨, 단어수, 장르(체크박스), 주제(동적 textarea)
5. `/my-library` — 지문 목록 테이블(태그 뱃지), 3개 모달 (방식 선택/직접 입력/AI 생성)
6. `/lesson-management` — 레슨 테이블(모듈 뱃지), 생성 모달(모듈 카테고리 선택, 지문 검색)
7. `/course-management` — 과정 테이블, 생성 모달(레슨 검색+체크박스 선택)

### Phase 4: Tutoring 영역 ✅ (8개 페이지)
1. `/speaking` — 스피킹 모듈 안내 (단순 정보 페이지)
2. `/mylearning` — 통계 카드 4개, 학습 기록 테이블
3. `/profile` — 사용자 정보, 관심분야 태그, 프로필/비밀번호/삭제 버튼
4. `/courses` — 과정 목록 테이블 (진행률 바)
5. `/tutoring-dashboard` — 통계 카드, 진행 중 레슨, 코스 카드 그리드
6. `/reading` — 코스 카드 (난이도 뱃지), 현재 학습 지문, 이해도 확인 문제
7. `/reading/reports` — 날짜 필터, 통계 카드(좌측 색상 보더), 차트 placeholder, 학습 기록, 학습 능력 점수
8. `/reading/player` — 2열 레이아웃 (좌: 지문+문제 / 우: Pickle AI 채팅), 피드백 3단계, 핵심 피드백 카드, 빠른 액션, 음성 입력

### 빌드 검증 ✅
- `pnpm next build` 성공 (25개 라우트 모두 정적 생성)
- TypeScript 오류 수정:
  - `DataTable` 제네릭 타입 제약 `<T extends Record<string, unknown>>` → `<T>` 변경 (인터페이스 호환성)
  - `EditableTable` 동일 수정
  - Server Component에서 함수 전달 오류 → 7개 페이지에 `'use client'` 추가

### 최종 생성 파일 수

| 범주 | 파일 수 |
|------|---------|
| shadcn/ui 컴포넌트 | 17 |
| 레이아웃 컴포넌트 | 2 |
| 공통 비즈니스 컴포넌트 | 10 |
| Route Group 레이아웃 | 3 |
| Admin 페이지 + 서브컴포넌트 | 11 |
| Studio 페이지 | 7 |
| Tutoring 페이지 | 8 |
| Zustand 스토어 | 3 |
| Mock 데이터 | 5 |
| Hooks, Constants, Types | 5 |
| **합계** | **약 71개 신규 파일** |

### 라우트 목록 (25개)
```
○ /                      ← 역할 선택
○ /dashboard             ← Admin 대시보드
○ /users                 ← 사용자관리
○ /billing               ← 기관관리(빌링)
○ /ai-modules            ← AI 모듈관리
○ /ai-modules/register   ← AI 모듈 등록
○ /learning-management   ← 학습 관리
○ /system                ← 시스템 관리
○ /studio-dashboard      ← Studio 대시보드
○ /course-wizard         ← 코스 마법사
○ /my-library            ← My Library
○ /lesson-management     ← 레슨 관리
○ /course-management     ← 과정 관리
○ /course-deployment     ← 과정 배포
○ /class-monitoring      ← 수업 모니터링
○ /tutoring-dashboard    ← Tutoring 대시보드
○ /courses               ← 과정 목록
○ /mylearning            ← 내 학습
○ /profile               ← 프로필
○ /reading               ← Reading 레슨
○ /reading/reports       ← 진행 현황
○ /reading/player        ← 레슨 플레이어
○ /speaking              ← Speaking
```

### 레이아웃 전폭(100%) 수정 ✅
- 3개 Route Group 레이아웃에서 `max-w-[1600px] mx-auto` 제거
- picklass-playground과 동일하게 화면 100% 너비로 콘텐츠 표시
- 수정 파일: `(admin)/layout.tsx`, `(studio)/layout.tsx`, `(tutoring)/layout.tsx`
- 변경: `<div className="flex gap-6 p-4 lg:p-6 max-w-[1600px] mx-auto">` → `<div className="flex gap-6 p-4 lg:p-6">`

---

## 핵심 참조 파일

| 파일 | 용도 |
|------|------|
| `picklass-playground/css/common.css` | 원본 스타일 → Tailwind 변환 기준 |
| `picklass-playground/js/common.js` | 공통 JS → React hooks/stores 변환 기준 |
| `backoffice/apps/admin/frontend/src/app/globals.css` | 색상 변수 확장 대상 |
| `backoffice/apps/admin/frontend/components.json` | shadcn/ui 설치 설정 |
| `backoffice/apps/admin/frontend/src/lib/api.ts` | 기존 fetchApi 유틸 (재사용) |
| `backoffice/apps/admin/frontend/src/lib/utils.ts` | 기존 cn 유틸 (재사용) |

---

## Part 4: 2026-03-08 업데이트 계획

### 4.1 picklass-playground 변경사항 분석

#### 신규 파일
| 파일 | 크기 | 설명 |
|------|------|------|
| `index.html` | 707줄 | 랜딩 페이지 (로그인/회원가입 모달 포함) |
| `index.md` | 407줄 | 랜딩 페이지 명세 (사용자 흐름, 기능 정의, 정책) |
| `admin/institute.html` | 686줄 | 기관관리 전용 페이지 (billing에서 분리) |
| `admin/institute.md` | 512줄 | 기관관리 명세 (4섹션 폼) |
| `Picklass_policy_260306.md` | 659줄 | B2B 구조·플로우 설계 기획 문서 |

#### 주요 업데이트 파일
| 파일 | 변경사항 |
|------|---------|
| `admin/users.html` | 액세스코드 생성 모달 확장 (아이디 동시 생성, 등록유효기간, 사용가능기간) |
| `admin/users.md` | 1047줄로 확장 (F-002 강사/학생 모달 상세 정의) |
| `admin/billing.html` | 4섹션 → 3섹션 간소화 (파트너정보, 라이선스, 계약정보) |
| `admin/billing.md` | 500줄로 간소화 |

### 4.2 핵심 변경사항 요약

#### 1) 랜딩 페이지 신규 (index.html)
- **목적**: B2B 유입 → 데모 체험 → 제휴/도입 문의 → 가입
- **구성**:
  - 헤더: 네비게이션 + 로그인/회원가입 버튼
  - 히어로 섹션: 서비스 소개
  - 주요 기능 / 요금제 / 파트너 제휴 섹션
  - 데모 신청 섹션
  - 푸터: 연락처 정보
- **인증 모달**:
  - 로그인 모달: 사용자타입 선택 (학습자/기관장) → 인증방식 선택 (이메일/소셜) → 인증정보 입력
  - 회원가입 모달: 사용자타입 → 인증방식 → 공통정보 → 타입별 추가정보 → 이용약관
  - 추가정보 입력 모달: 로그인 후 누락된 프로필 정보 수집

#### 2) 기관관리 페이지 분리
- **기존**: `/billing` (기관 + 청구 통합)
- **변경**:
  - `/institute` — 기관 CRUD (4섹션: 가입정보/부가정보/라이선스/계약정보)
  - `/billing` — 청구/요금 관리 (3섹션: 파트너정보/라이선스/계약정보)

#### 3) 사용자관리 액세스코드 확장 (users.html)
- **아이디 동시 생성 옵션** 추가
  - 체크박스: "아이디 동시 생성"
  - 형식: `{문자열}{일련번호}@{문자열}.pickle`
  - 예: `user001@gangnam.pickle`, `user002@gangnam.pickle`
- **필드 변경**:
  - "유효기간" → "등록 유효기간" (date picker, 기본값: 현재+1개월)
  - "사용가능기간" 필드 추가 (일 단위, 1~365)
  - "코드상태" 기본값: "활성" → "비활성" 변경

#### 4) B2B 플로우 체계화 (Picklass_policy_260306.md)
- **도메인 구조**:
  - `www.picklass.com` — Landing (유입/가입)
  - `studio.picklass.com` — Studio (교사용 수업 제작)
  - `tutoring.picklass.com` — Tutoring (학생용 AI 과외)
  - `admin.picklass.com` — Admin (운영/기관 관리)
- **역할 권한 매트릭스**:
  | 기능 | 학생 | 강사 | 학원관리자 | 시스템관리자 |
  |------|:----:|:----:|:----------:|:------------:|
  | 지문 등록 | ❌ | ✅ | ❌ | ✅ |
  | 과정 생성 | ❌ | ✅ | ❌ | ✅ |
  | 코드 발급 | ❌ | ✅ | ✅ | ✅ |
  | 레슨 수행 | ✅ | ✅ | ❌ | ❌ |
  | 수업 모니터링 | ❌ | ✅(자기) | ✅(기관) | ✅ |
- **비즈니스 모델별 가입 정책**:
  - B2B 제휴: 액세스코드 기반, 무료체험 없음
  - SaaS 구독(중소학원): 강사 자유가입 + 7일 무료체험, 학생은 코드 기반
  - B2C 개인(차후 개발): 학생 자유가입 + 7일 무료체험

### 4.3 라우팅 매핑 업데이트

```
src/app/
├── page.tsx                              ← index.html (랜딩 페이지, 인증 모달)
├── (admin)/
│   ├── institute/page.tsx                ← admin/institute.html (신규: 기관관리)
│   ├── billing/page.tsx                  ← admin/billing.html (수정: 3섹션)
│   └── users/page.tsx                    ← admin/users.html (수정: 액세스코드 확장)
```

### 4.4 구현 계획

#### Phase A: 랜딩 페이지 및 인증 시스템 (~15파일)
1. **루트 페이지 재작업** (`page.tsx`)
   - 헤더 (네비게이션, 로그인/회원가입 버튼)
   - 히어로 섹션
   - 주요 기능 섹션
   - 요금제 섹션
   - 파트너 제휴 섹션
   - 데모 신청 섹션
   - 푸터
2. **인증 모달 컴포넌트 신규**
   - `components/auth/login-modal.tsx` — 로그인 모달 (사용자타입+인증방식+입력폼)
   - `components/auth/signup-modal.tsx` — 회원가입 모달 (5단계 폼)
   - `components/auth/profile-modal.tsx` — 추가정보 입력 모달
   - `components/auth/social-buttons.tsx` — 소셜 로그인 버튼 (Google/Kakao/Naver)
3. **Zustand 스토어 확장**
   - `use-auth-store.ts` 확장: userType (learner/institution), authMethod (email/social)

#### Phase B: Admin 페이지 업데이트 (~10파일)
1. **기관관리 페이지 신규** (`(admin)/institute/page.tsx`)
   - 통계 대시보드 (요금제별 현황, 매출 추이)
   - 기관 검색 (기관명/요금제/상태)
   - 기관 테이블 (페이지네이션 20개/페이지)
   - 기관 등록/수정 모달 (4섹션: 가입정보/부가정보/라이선스/계약정보)
2. **기관관리(빌링) 수정** (`(admin)/billing/page.tsx`)
   - 모달 3섹션으로 간소화 (파트너정보/라이선스/계약정보)
3. **사용자관리 수정** (`(admin)/users/page.tsx`)
   - 사용자 등록 모달 확장:
     - 아이디 동시 생성 체크박스 + 형식 입력 UI
     - 등록 유효기간 (date picker)
     - 사용가능기간 (number input, 일 단위)
     - 코드 상태 기본값 "비활성"

### 4.5 신규 컴포넌트 목록

| 파일 | 설명 |
|------|------|
| `components/auth/login-modal.tsx` | 로그인 모달 (사용자타입/인증방식/입력폼) |
| `components/auth/signup-modal.tsx` | 회원가입 모달 (5단계 폼) |
| `components/auth/profile-modal.tsx` | 추가정보 입력 모달 (학습자: 닉네임, 기관장: 4필드) |
| `components/auth/social-buttons.tsx` | 소셜 로그인 버튼 그룹 |
| `components/landing/hero-section.tsx` | 히어로 섹션 |
| `components/landing/features-section.tsx` | 주요 기능 섹션 |
| `components/landing/pricing-section.tsx` | 요금제 섹션 |
| `components/landing/partner-section.tsx` | 파트너 제휴 섹션 |
| `components/landing/demo-section.tsx` | 데모 신청 섹션 |
| `components/landing/footer.tsx` | 푸터 |

### 4.6 데이터 타입 확장

```typescript
// types/index.ts 확장

// 사용자 타입
type UserType = 'learner' | 'institution';

// 인증 방식
type AuthMethod = 'email' | 'social';
type SocialProvider = 'google' | 'kakao' | 'naver';

// 기관 정보 (4섹션)
interface Institution {
  // 가입정보
  id: string;
  name: string;
  type: '개인학원' | '프랜차이즈' | '어학원' | '공교육' | '기업교육';
  contactName: string;
  contactPhone: string;

  // 부가정보
  branchCount?: number;
  operationType: '직영' | '가맹';
  currentStudents: number;

  // 라이선스
  plan: 'Lite' | 'Pro' | 'Enterprise' | '제휴';
  pricePerStudent: string;
  maxStudents: number;
  adminAccounts: number;
  billingCycle: '월납' | '연납';
  annualDiscount?: number;
  apiIntegrationFee?: number;
  techSupportFee?: number;
  ipTransferFee?: number;
  notes?: string;

  // 계약정보
  contractStatus: '협의중' | '계약완료' | '활성' | '만료' | '해지';
  startDate: string;
  endDate: string;
  autoRenewal: boolean;
  renewalConditions?: string;
}

// 액세스코드 확장
interface AccessCode {
  code: string;
  role: 'teacher' | 'student';
  institutionId: string;
  status: '활성' | '비활성' | '탈퇴';
  registrationExpiry: string;  // 등록 유효기간
  usagePeriodDays: number;     // 사용가능기간 (일)
  activatedAt?: string;
  userId?: string;
  userName?: string;

  // 아이디 동시 생성
  generatedUserId?: string;
  generatedPassword?: string;
}
```

### 4.7 예상 파일 수

| 범주 | 파일 수 |
|------|---------|
| 랜딩 페이지 + 섹션 컴포넌트 | 10 |
| 인증 컴포넌트 | 4 |
| 기관관리 페이지 | 1 |
| 기존 페이지 수정 | 2 (billing, users) |
| 타입/스토어 확장 | 2 |
| **합계** | **약 19~25개** |

### 4.8 검증 방법

1. **랜딩 페이지**
   - `http://localhost:4100` 접속 → 랜딩 페이지 UI 확인
   - 로그인/회원가입 모달 동작 확인
   - 사용자타입 선택 → 인증방식 선택 → 폼 전환 확인
   - 소셜 로그인 버튼 표시 확인

2. **기관관리**
   - `/institute` 접속 → 기관 목록/등록/수정 확인
   - 4섹션 모달 폼 동작 확인
   - `/billing` 접속 → 3섹션 간소화 확인

3. **사용자관리**
   - `/users` 접속 → 사용자 등록 모달 확인
   - 아이디 동시 생성 옵션 체크 → UI 변경 확인
   - 등록유효기간/사용가능기간 필드 확인

4. **반응형**
   - 375px/768px/1024px 뷰포트 테스트
