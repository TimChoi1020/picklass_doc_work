# Studio IA (정보구조)

> 참고문서(`참고문서/picklass_docs/studio/`)를 기반으로 정리한 Studio(강사용 콘텐츠 저작 도구) 서비스의 정보구조 문서입니다.
> **SSOT**: `architecture/menu-structure.md` (2026-06-05, 코드 원본: `apps/web/src/components/oizi/StudioHeader.tsx`)

---

## 0. 서비스 개요 & IA 설계 배경

**Studio는 강사(teacher)가 학습 콘텐츠를 직접 만들고 배포하는 저작 도구**입니다. 도메인은 `studio.picklass.com`(web 3005 / api 3006)이며, 조직 구조는 **파트너 > 그룹 > 기관 > 강사** 계층으로 강사는 자신이 소속된 기관 범위 안에서 작업합니다.

**IA의 조직 원리는 "강사의 업무 흐름을 그대로 상단 탭 순서로 매핑"한 것**입니다. 강사가 콘텐츠를 만들어 학생에게 전달하기까지의 자연스러운 순서 —

> **① 지문 만들기 → ② 과정 구성 → ③ 수강자 배정 → ④ 결과 확인**

— 이 네 단계가 곧 상단 네비게이션 `My Library → Course Hub → Enrollment → Class Report`의 순서가 됩니다. 처음 쓰는 강사도 왼쪽에서 오른쪽으로 메뉴를 따라가면 한 사이클이 완성되도록 설계되어 있습니다.

**Backoffice(백오피스)와의 역할 분리**가 IA의 또 다른 핵심 전제입니다. **"콘텐츠를 만드는 일"은 Studio, "운영·모니터링·계정관리"는 백오피스**가 담당합니다. 그래서 같은 '과정'이라도 Studio에서는 생성·편집(쓰기)이 가능하지만 백오피스에서는 읽기 전용으로만 보입니다. 현재는 수강자 관리(학생 아이디·액세스코드)가 Course Hub 하위에 섞여 있으나, 역할을 더 명확히 하기 위해 **Enrollment(수강자 배정) 메뉴를 별도 1depth로 분리하는 개편이 진행 중**입니다.

메뉴명은 영문(My Library, Course Hub 등)을 사용하되 화면 내부 라벨은 한글을 씁니다.

---

## 1. 전체 메뉴 트리 (헤더 네비 + 사이드바)

각 1depth 메뉴가 업무 흐름의 한 단계에 대응합니다. `★` = 신설 예정(미구현).

```
Studio (StudioHeader)
├── 로고 → 홈(/)
│
├── [1] My Library     /class                     ① 지문(텍스트) 만들기·보관
│   ├─ (사이드바) 전체 보기 / My folder(UI만) / 새 폴더 만들기(미구현)
│   ├─ 지문 목록          /class
│   ├─ 수업 준비          /class/lesson-setup/[passageId]
│   └─ 수업 진행          /class/lesson/[passageId]
│
├── [2] Course Hub     /course                     ② 지문을 엮어 과정 구성
│   ├─ (사이드바 현재) 모든 과정 / 학생 아이디 관리 / 액세스코드 관리
│   ├─ (사이드바 목표) 전체 보기 / 카테고리 트리(L1·L2) / 미분류
│   ├─ 과정 목록          /course
│   └─ 과정 상세·편집      /course/[courseId]
│
├── [3] Enrollment ★   /enrollment                 ③ 학생 등록·접근코드 (신설 예정)
│   ├─ Students          /enrollment/students        ← /course/students 이동
│   └─ Access Codes      /enrollment/access-codes    ← /course/accesscode 이동
│
└── [4] Class Report   /report                     ④ 학습 결과 확인
    ├─ 리포트 목록        /report
    └─ 학습자 상세 리포트  /report/[studentId]
```

- **현재 구현 헤더**: `My Library | Course Hub | Class Report` (3개)
- **목표 헤더**: `My Library | Course Hub | Enrollment | Class Report` (4개)

### 전체 라우트 맵
```
/ · /login · /signup · /legal/[document]
/class · /class/lesson-setup/[passageId] · /class/lesson/[passageId]
/course · /course/[courseId] · /course/students(→redirect) · /course/accesscode(→redirect)
/enrollment · /enrollment/students · /enrollment/access-codes   (신설)
/report · /report/[studentId]
```

---

## 2. 메뉴·화면별 기능 설명

### ① My Library `/class` — 지문 관리
**"수업의 원재료인 지문(영어 텍스트)을 만들고 모아두는 서재"** 역할입니다. 강사는 AI로 지문을 생성하거나 직접 입력해 보관하고, 그 지문 하나를 골라 KPI 목표에 맞는 수업을 즉석에서 구성해 진행할 수 있습니다. 과정(course)에 편입되기 전의 낱개 콘텐츠 단위를 다루는 공간입니다.

- **대시보드**: Greeting + Card Group 3종(최근 학습한 지문 / 최근 생성한 지문 / 이 달의 Discover Pick) — 강사가 자주 쓰는 지문에 빠르게 재접근하도록 돕는 진입점
- **지문 목록 테이블**: 제목·수업횟수·제작자·생성일, 페이지네이션 10개
- **필터 바**: CEFR(18레벨) · 단어수(20범위) · 장르 · 정렬(최신순) + 제목 검색 — 레벨·길이별로 원하는 지문을 빠르게 좁혀 찾기 위함
- **모달**: CreatePassageModal(AI 생성/수동), PassageDetailModal(난이도 8지표 분석), TwinPassagesModal(난이도 쌍둥이 지문), PreviewPassageModal
- **지문 난이도 8지표**: 어휘난이도·어휘다양성·지문길이·문장길이·문장구조·문법다양성·정보밀도·배경지식의존도 — 학생 레벨에 맞는 지문인지 객관적으로 판단하는 근거
- **수업 준비(`lesson-setup`)**: 지문 하나에 대해 **KPI 목표 + 수업 시간을 고르면 시스템이 자동으로 학습 모듈 순서를 짜주는(analyzer 시퀀싱) 화면**. 강사가 모듈 하나하나를 직접 배열하지 않아도 되도록 자동화한 것이 핵심.
- **수업 진행(`lesson`)**: 준비된 구성으로 실제 수업을 진행하는 화면

### ② Course Hub `/course` — 과정 관리
**"낱개 지문을 회차(레슨) 단위로 엮어 하나의 정규 과정으로 만들고 배포하는 공간"**입니다. My Library가 재료 창고라면 Course Hub는 완성된 커리큘럼을 다루는 곳으로, 과정의 생성·편집·삭제와 배포용 액세스코드 발급이 모두 여기서 일어납니다.

- **대시보드**: Card Group 3종(최근 생성한 과정 / 배포 중인 과정 / 인기 있는 과정)
- **사이드바(CourseSidebar)**: **L1/L2 카테고리 아코디언 트리** + "미분류" 폴더(파트너 소속 기관만) + 과정 검색 — 과정이 많아질 때 카테고리로 탐색하기 위한 구조
- **과정 목록 테이블**: 번호·과정명·유형뱃지·레슨·배포코드·수강자·상태, 페이지네이션 10개
  - **유형 뱃지**: 튜터링(파랑)/스피킹(주황)/통합(초록)/미설정(회색) — 과정이 어떤 학습 앱에서 열리는지 한눈에 구분
  - **상태**: `in-use`(배포중, 학생 포함 가능) / `not-used`(미배포) — 배포된 과정은 학생 데이터 보호를 위해 일부 편집이 잠김
- **과정 CRUD**: 생성 마법사, 편집(CourseEditModal: 기본정보/카테고리/운영설정 탭), 삭제
- **행 액션 "액세스코드 생성"**: 과정을 학생에게 나눠줄 코드를 즉석 발급(`POST /courses/:id/access-code`, deployment_count 증가)
- **과정 상세·편집 `/course/[courseId]`**: **한 과정의 회차별 레슨을 편집하는 커리큘럼 편집기**. 레슨 목록(회차·제목·레벨·모듈·생성일·상태), 순서 이동(↑/↓, round 재계산), 모듈 수정, 레슨 추가(2스텝: 텍스트→모듈), 과정에서 제외. **`in-use`(배포된) 레슨은 모듈수정·제외가 비활성**되어 이미 학습 중인 학생의 데이터를 보호.

### ③ Enrollment `/enrollment` — 수강자 등록·코드 (신설 예정)
**"만든 과정을 실제 학생에게 연결하는 배정 단계"**입니다. 현재는 Course Hub 사이드바 하위에 있으나, '콘텐츠 저작'과 '수강자 관리'는 성격이 다르므로 별도 1depth로 분리 예정입니다. 하위 호환을 위해 기존 경로는 redirect로 유지합니다.

- **Students(현 `/course/students`)**: 학생 아이디를 만들고 관리. 단건 등록(아이디 자동생성 `{STUDIO_CODE}{serial}@{STUDIO_CODE}.pick` + 중복확인), 정보 수정(이름·상태·비밀번호초기화), 학생+액세스코드 일괄 등록. 상태 active/suspended/withdrawn.
- **Access Codes(현 `/course/accesscode`)**: 과정 수강권한을 담은 코드를 발급·관리. 코드 목록(통계 전체/수강중/미사용), 일괄 등록(개수·등록만료일·사용기간·시작종료일), 활성화↔비활성화 토글. 상태 4종 inactive/active/completed/expired. **역할은 student 고정, 소유권은 institution_id 기준. 삭제는 백오피스 전용**(역할 분리 원칙).

### ④ Class Report `/report` — 학습 리포트
**"배포한 과정을 학생들이 얼마나·어떻게 학습했는지 확인하는 결과 대시보드"**입니다. 업무 흐름의 마지막 단계로, 강사가 학습 성과를 점검하고 다음 지도에 반영하는 피드백 루프를 담당합니다.

- **대시보드**: Card Group(데이터 관리 Excel 다운로드 / 진도율 상위 / 학습완료율 상위 + 평균 학습시간·미학습자 수) — 반 전체 현황을 요약
- **필터**: 과정 · 학습시작일 · 학습종료일 · 아이디(이름) 검색 · 정렬 · 초기화
- **학생 학습 테이블**: 이름·과정명·수강기간·진도·레슨현황(동적 O/△/-)·누적시간·Voca·Reading WPM·발음정확도·Speaking WPM·문장구조·정답률, 리포트 출력 버튼 — 정량 KPI를 학생별로 나열
- **데이터 계층**: access_codes → lesson_results → module_histories(kpis) — 코드 발급→레슨 수행→모듈별 KPI로 데이터가 쌓이는 흐름
- **학습자 상세 `/report/[studentId]`**: 탭1 레슨별 진도, 탭2 모듈별 KPI + FRT 대화 로그 (Phase 3 일부 미구현)

---

## 3. 주요 저작 플로우

### A. 과정 생성 마법사 (Course Hub)
과정을 만들 때 **어떤 학습 앱에서 열리는 과정인지(유형)부터 정하고, 기본정보·운영정책·회차 구성을 단계적으로 채우는** 위저드입니다. 운영 설정(수업시간·이수조건 등)은 기관 정책과 연동되어 강사가 임의로 넘어설 수 없습니다.

```
[새 과정 만들기]
 → Step 0-A 과정 유형 선택 (①튜터링 V/R/W  ②스피킹 S  ③통합 V/R/W/S)
 → Step 0-B 생성 방법 선택 (①주제 입력[현행]  ②교재 PDF 업로드[미구현 "준비중"])
 → Step 1/2 기본정보(과정명/레벨/회차수) + [과정 운영 설정](기관설정 연동)
            · 전체 수업 최대시간(기관 max 이하)
            · 이수조건(레슨이수율 / 발화량[Speaking 포함시만] / 누적수업시간)
            · 나만의 수업(자유학습, 기관 허용시만 ON)
 → Step 2/2 회차별 레슨(모듈) 구성 + [FRT 설정](FRT 모듈 존재시만)
            · FRT 학습시간(5~30분 프리셋) · 마이크 모드(PTT/Hybrid/Always on)
 → 완료
```
- 유형별 모듈 필터: 튜터링=speaking 제외 전체 / 스피킹=speaking만 / 통합=전체 (통합은 스피킹 모듈을 회차 내 항상 후순위)
- AI 주제 생성: Gemini 2.5 Flash, 12개 장르. 카테고리 배정은 생성이 아닌 **편집 페이지**에서 수행.

### B. 수업 준비 → 진행 (My Library, KPI 기반 시퀀싱)
지문 하나로 즉석 수업을 구성하는 흐름. **강사가 목표와 시간만 지정하면 모듈 순서는 시스템이 자동으로 계산**합니다.
```
/class 지문 목록 → 지문 선택 → /class/lesson-setup/[passageId]
 → KPI 선택 + 수업시간 선택(둘 다 충족 트리거) → 자동 POST /lesson-plan(시퀀싱) → 로딩
 → 모듈 선택 시 분석 순서대로 번호 자동 부여·재정렬
 → [수업 들어가기] → /class/lesson/[passageId]
```

### C. 과정 편집 (레슨 구성)
```
/course → 과정 클릭 → /course/[courseId]
 → 레슨 행 클릭 → 오버레이(↑/↓ 이동 | 모듈 수정 | +레슨추가 | 과정에서 제외)
 → 레슨 추가: Step1 텍스트 선택 → Step2 모듈 선택 → 삽입(round 재계산)
```

### D. 수강자 배정 (Enrollment)
```
학생 아이디 등록(단건/일괄) → 액세스코드 생성(일괄 or 과정별 1건)
 → 코드 활성화 → 학생이 tutoring 앱에서 코드 등록 → 수강 시작
```

---

## 4. 화면 전환 관계
- **StudioHeader**(전 인증 페이지 공유): 로고→`/`, 탭 3~4개 상호 전환(활성 탭 녹색 밑줄), 설정 드롭다운(계정 정보)
- **CourseSidebar**(course/* 공통): "모든 과정"→`/course`, L1/L2 클릭 시 우측 테이블 필터링
- `/course` 카드 → `/course/[courseId]`; 상세 뒤로가기 → `router.back()`
- `/class` 지문 → `/class/lesson-setup/[passageId]` → `/class/lesson/[passageId]`
- `/report` 행 → 오버레이 "상세보기" → `/report/[studentId]`
- 과정 행 "액세스코드 생성" → `/course/accesscode` 목록 반영
- Enrollment 신설 시 `/course/students`·`/course/accesscode`는 redirect로 하위 호환

---

## 5. 구현 상태
- ✅ My Library, Course Hub, 과정 상세, 학생/액세스코드(`/course/*`), Class Report, 과정 유형 선택, 운영·FRT 설정, 카테고리 L1/L2 트리
- 🔲 **Enrollment 메뉴 신설**(최우선), 교재 PDF 업로드 생성, My Library 실제 폴더, `max_class_minutes`·`hasFrtModule` 실계산(현재 하드코딩), 리포트 상세 Phase 3-B

---

## 6. 근거 문서
- `참고문서/picklass_docs/studio/architecture/menu-structure.md` — 메뉴 구조 마스터(SSOT)
- `.../studio/README.md` — 서비스 개요·포트·조직구조
- `.../studio/docs/studio_update.md` — 전체 사이트맵/IA(초기), 화면·데이터 모델
- `.../studio/docs/course-detail-20260316.md` — 과정 상세·편집 화면·플로우
- `.../studio/docs/20260526_리포트_페이지_구현계획.md` — Class Report 목록·상세·필터
- `.../studio/docs/students-2026-03-24.md` · `.../docs/20260424_액세스코드관리.md` — 학생/액세스코드 관리
- `.../studio/docs/20260424_lesson-setup_시퀀싱_표시_설계.md` — 수업 준비 KPI 시퀀싱
- `.../studio/features/과정생성/20260519_과정생성_유형선택_PDF업로드_업데이트계획.md` — 과정 생성 마법사
- `.../studio/features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md` — CourseSidebar L1/L2 트리
