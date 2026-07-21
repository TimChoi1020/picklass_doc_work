# Studio Picklass - UI/UX 리뉴얼 종합 업데이트 문서

**작성일**: 2026-03-11  
**최종업데이트**: 2026-03-14  
**버전**: 2.1 (통합 버전)  
**담당자**: UI/UX & Backend Team

**포함 범위**: Class (My Library) / Course (Course Hub) / Report (Class Report)

---

## 📅 최근 업데이트 (2026-03-14)

### Course Hub 페이지 개선

#### 1. 과정 생성 UI 개선
- **wordCount 데이터 정규화**: 범위(string) → 숫자(number) 변환
- **모듈 선택 레이아웃**: Flexbox 래핑 → Grid 4열 배치
- **목표 및 뱃지 레이아웃**: 테이블 우측에 정렬

#### 2. 과정 상세 페이지 개선  
- **헤더 높이 최적화**: Header margin 감소 (mb-12 → mb-4)
- **정보 패널 재배치**: 과정명 좌측, 목표/뱃지 우측 정렬
- **레이아웃**: 수평 배치로 공간 효율화

#### 3. 네비게이션 강화
- **CourseSidebar**: "모든 과정" 버튼화
- **라우팅**: 클릭 시 `/course` 페이지로 이동
- **UX**: Hover 효과 추가

#### 4. AI 주제 생성 프롬프트 추가
- **신규 파일**: `prompts/course_topic_generation_prompt.md`
- **목적**: 과정 생성 시 AI 기반 자동 주제 생성
- **지원 장르**: 12가지 (가이드, 논설문, 뉴스 기사, 단편소설, 대화문, 리뷰, 보고서, 설명문, 스토리, 에세이, 일기, 편집)
- **기능**:
  - 입력: 과정명, 과정목표, CEFR 레벨, 장르 선택
  - 출력: 회차별 2~3줄의 최신 정보 기반 주제
  - AI: Google Gemini 2.5 Flash 활용
  - 장르별 상세 가이드라인: 각 장르별 주제 생성 요소 및 형식 명시
- **적용**: 과정 생성 마법사 (Step 1/2 → Step 2/2 전환 시)
- **구현**: genreGuidanceMap으로 12개 장르 모두 매핑됨

---

## 📋 목차

1. [변경 사항 종합](#변경-사항-종합)
2. [개발자 추가 작업 목록](#개발자-추가-작업-목록)
3. [통합 정책 변경사항](#통합-정책-변경사항)
4. [통합 IA 구조](#통합-ia-구조)
5. [데이터 모델](#데이터-모델)
6. [컴포넌트 구조](#컴포넌트-구조)

---

## 변경 사항 종합

### 📍 1. 공통 변경 사항

#### A. StudioHeader 통합 네비게이션

**파일**: `apps/web/src/components/oizi/StudioHeader.tsx`

- **네비게이션 탭** (3개):
  - 📚 **My Library** (`/class`) - 지문 관리
  - 📖 **Course Hub** (`/course`) - 과정 관리
  - 📊 **Class Report** (`/report`) - 학습 현황

- **기능**:
  - 로고 클릭 시 홈(`/`)으로 이동
  - 활성 탭 표시 (녹색 밑줄)
  - 설정 드롭다운 (계정 정보)

#### B. CEFR 레벨 확장

**변경**: 6개 레벨 → 18개 레벨

```
A1- < A1 < A1+ < A2- < A2 < A2+ < B1- < B1 < B1+ < 
B2- < B2 < B2+ < C1- < C1 < C1+ < C2- < C2 < C2+
```

적용 파일:
- `AIGenerateModal.tsx` (지문 생성)
- `course/page.tsx` (과정 필터)
- `report/page.tsx` (리포트 필터)

---

### 📍 2. Class 페이지 변경사항

**파일**: `apps/web/src/app/class/page.tsx`

| 항목 | 변경 내용 | 상태 |
|------|---------|------|
| 헤더 통합 | NewHeader 제거, StudioHeader 적용 | ✅ 완료 |
| CEFR 레벨 | 6개 → 18개 확장 | ✅ 완료 |
| 텍스트 가시성 | `text-gray-900` 추가 (4곳) | ✅ 완료 |
| 지문 분석 | 8개 난이도 지표 UI 추가 | ✅ UI 완료 |

**지문 난이도 분석** (PassageDetailModal.tsx):
```
1. 어휘 난이도 (Vocabulary Difficulty)
2. 어휘 다양성 (Vocabulary Variety)
3. 지문 길이 (Text Length)
4. 문장 길이 (Sentence Length)
5. 문장 구조 (Sentence Structure)
6. 문법 다양성 (Grammar Variety)
7. 정보 밀도 (Information Density)
8. 배경지식 의존도 (Background Knowledge Dependency)
```

**Card Group**:
- 최근 학습한 지문
- 최근 생성한 지문
- 이 달의 Discover Pick

---

### 📍 3. Course 페이지 변경사항 (신규)

**파일**: `apps/web/src/app/course/page.tsx` (신규)

**기능**:
| 기능 | 설명 | 상태 |
|------|------|------|
| 과정 목록 | 가나다순 정렬 테이블 | ✅ UI 완료 |
| 과정 생성 | CEFR, 장르 선택 모달 | ✅ UI 완료 |
| 과정 수정 | 기존 과정 정보 수정 | ✅ UI 완료 |
| 과정 삭제 | 확인 다이얼로그 | ✅ UI 완료 |
| 필터링 | 레벨 × 장르 조합 필터 | ✅ UI 완료 |
| 검색 | 과정명 검색 | ✅ UI 완료 |
| 페이지네이션 | 10개 항목씩 표시 | ✅ UI 완료 |

**Card Group**:
- 최근 생성한 과정
- 배포 중인 과정 (상태: in-use)
- 인기 있는 과정 (수강생 수 기준)

**상태 관리**:
- `in-use`: 배포 중 (학생 포함 가능)
- `not-used`: 미배포 (학생 미포함)

---

### 📍 4. Report 페이지 변경사항 (신규)

**파일**: `apps/web/src/app/report/page.tsx` (신규)

**핵심 기능**: 학생 개인별 학습 현황 분석

| 메트릭 | 설명 | 형식 |
|--------|------|------|
| Unit 진행 | 10개 Unit 완료 상태 | O/△/- 기호 |
| 진도율 | 전체 학습 진행도 | 0-100% |
| 어휘 | 학습한 어휘 수 | 개수 |
| 읽기유창성 | 분당 단어 읽기 속도 | WPM |
| 발음정확도 | 발음 정확도 점수 | 0-100% |
| 말하기유창성 | 분당 단어 말하기 속도 | WPM |
| 문장구조 | 문법 수준 | CEFR 레벨 |
| 평균문장길이 | 문제별 평균 문장 길이 | 단어 수 |
| 정답률 | 문제 정답 비율 | 0-100% |

**필터 옵션**:
- 업체명(본사)
- 캠퍼스
- 담당(강사)
- 과정
- 클래스(반)
- 개인(학생명 검색)
- 학습기간 (1개월 기본)

**카드 그룹**:
1. 데이터 관리 - Excel 다운로드
2. 진도율 상위 - 상위 1명 학생
3. 학습완료율 상위 - 완료된 Unit 가장 많은 학생

**학습 상태 기호**:
| 기호 | 의미 | 색상 |
|------|------|------|
| `O` | 수업 완료 | 🟢 Green |
| `△` | 수업 중 | 🟡 Yellow |
| `-` | 수업 전 | ⚪ Gray |

**성과 평가 기준**:
```
발음정각도:
  90%+ 우수 | 80-89% 양호 | 70% 이하 개선 필요

유창성(WPM):
  120+ 우수 | 90-119 양호 | 89 이하 개선 필요
```

---

## 개발자 추가 작업 목록

### 🔴 필수 작업 (Priority: HIGH)

#### 1. Backend API 구현

**1.1 Class/Report 공통 API - 지문 분석**
```
POST /api/passages/:id/analyze

요청 본문:
{
  id: string (지문 ID)
}

응답:
{
  id: string,
  analysis: {
    vocabularyDifficulty: string,     # CEFR 레벨
    vocabularyVariety: string,        # CEFR 레벨
    textLength: string,               # CEFR 레벨
    sentenceLength: string,           # CEFR 레벨
    sentenceStructure: string,        # CEFR 레벨
    grammarVariety: string,           # CEFR 레벨
    informationDensity: string,       # CEFR 레벨
    backgroundKnowledge: string       # CEFR 레벨
  }
}
```

**1.2 Course API - 과정 관리**
```
GET /api/courses
  Query: search?, level?, genre?, page?, limit?
  Response: { data[], total, page, totalPages }

POST /api/courses
  Body: { title, level, genre, description?, duration? }
  Response: { id, ...course }

PUT /api/courses/:id
  Body: { title?, level?, genre?, status?, ... }
  Response: { id, ...updated_course }

DELETE /api/courses/:id
  Response: { success, message }

GET /api/courses/:id
  Response: { id, title, level, genre, ... details }
```

**1.3 Report API - 학생 학습 데이터**
```
GET /api/reports/students
  Query: className?, level?, genre?, studentName?, period?
  Response: [{
    id, name, progressRate, unit1-10, vocaSize,
    readingWPM, pronunciationAccuracy, speakingWPM,
    sentenceStructure, avgSentenceLength, correctAnswerRate
  }]

GET /api/reports/students/:id
  Response: { id, name, ...full_metrics }

POST /api/reports/export
  Body: { filters: {...}, format: 'csv' | 'xlsx' }
  Response: { downloadUrl: string }
```

**1.4 인증 관련 API**
```
GET /api/auth/me
  Response: { id, email, displayName, role, profile }

POST /api/auth/logout
  Response: { success }
```

#### 2. Mock 데이터 교체

**Course 페이지**:
```javascript
// 제거할 Mock 데이터
- mockCourses (기존 3개)
- mockRecentCourses (기존 3개)

// 추가할 로직
- useEffect로 GET /api/courses 호출
- 필터 변경 시 자동 API 호출
- 페이지 변경 시 API 호출
```

**Report 페이지**:
```javascript
// 제거할 Mock 데이터
- mockStudentReports (10명 학생)
- mockReports (3개 클래스)

// 추가할 로직
- useEffect로 GET /api/reports/students 호출
- 필터 선택 시 자동 조회
- 학습기간 선택 시 API 호출
```

**Class 페이지**:
```javascript
// 분석 데이터 Mock → API 교체
- PassageDetailModal의 하드코딩된 분석 데이터
- POST /api/passages/:id/analyze 호출
- 캐싱 처리 (동일 지문 분석 시 재호출 방지)
```

#### 3. 에러 처리 및 로딩 상태

**구현 필요**:
```typescript
// 로딩 상태
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);

// API 호출 시
try {
  setIsLoading(true);
  const response = await fetch(url);
  if (!response.ok) throw new Error(`${response.status}`);
  const data = await response.json();
  // 상태 업데이트
} catch (err) {
  setError(err.message);
  toast.error('작업 실패'); // Sonner 토스트
} finally {
  setIsLoading(false);
}
```

**UI 표시**:
- 로딩 중: 스피너 표시
- 에러: 에러 메시지 토스트
- 빈 상태: "데이터가 없습니다" 메시지

---

### 🟡 권장 작업 (Priority: MEDIUM)

#### 1. 인증 시스템 연동

**파일**: 
- `StudioHeader.tsx` (Line 8)
- `ClassLayout.tsx` (인증 확인)

```typescript
// TODO: [AUTH] 인증 시스템 연동

// 현재
const MOCK_USER = {
  id: '80bd54a1-0734-43c5-a9ce-d8ec29cb8892',
  email: 'sniper4457@naver.com',
};

// 변경 필요
const { user, logout } = useAuth();
// 또는
const { data: user } = useSWR('/api/auth/me', fetcher);
```

**구현 사항**:
- 실제 사용자 정보 조회
- 로그아웃 기능
- 프로필 이미지 동적 로딩
- 권한 확인 (조직 관리자, 강사 등)

#### 2. 필터 옵션 동적 생성

**Course 페이지**:
```typescript
// 상수화
const CEFR_LEVELS = ['A1-', 'A1', 'A1+', ..., 'C2+'];
const GENRES = ['일반영어', '비즈니스', '발음', '회화', '문법', '어휘'];

// 선택 옵션 동적 생성
<select>
  <option value="">선택</option>
  {CEFR_LEVELS.map(level => (
    <option key={level} value={level}>{level}</option>
  ))}
</select>
```

**Report 페이지**:
```typescript
// API에서 옵션 조회
const [companies, setCompanies] = useState([]);
const [campuses, setCampuses] = useState([]);
const [instructors, setInstructors] = useState([]);
const [courses, setCourses] = useState([]);
const [classes, setClasses] = useState([]);

useEffect(() => {
  Promise.all([
    fetch('/api/companies').then(r => r.json()),
    fetch('/api/campuses').then(r => r.json()),
    fetch('/api/instructors').then(r => r.json()),
    fetch('/api/courses').then(r => r.json()),
    fetch('/api/classes').then(r => r.json()),
  ]).then(([comp, camp, inst, cour, clas]) => {
    setCompanies(comp);
    setCampuses(camp);
    setInstructors(inst);
    setCourses(cour);
    setClasses(clas);
  });
}, []);
```

#### 3. 다중 선택 필터 활성화

**Report 페이지**:
```typescript
// 현재: 각 필터는 독립적
const [levelFilter, setLevelFilter] = useState('');
const [genreFilter, setGenreFilter] = useState('');

// 변경: 필터 객체로 통합
const [filters, setFilters] = useState({
  company: '',
  campus: '',
  instructor: '',
  course: '',
  class: '',
  period: '1개월',
});

// 필터 적용 로직
useEffect(() => {
  const params = new URLSearchParams();
  Object.entries(filters).forEach(([key, value]) => {
    if (value) params.append(key, value);
  });
  fetchReports(params.toString());
}, [filters]);
```

#### 4. 리포트 출력 기능

**Report 페이지** (Line 233):
```typescript
const handlePrintReport = async (studentName: string) => {
  try {
    // 옵션 1: 브라우저 인쇄
    window.print();
    
    // 옵션 2: PDF 생성 (jsPDF 라이브러리 필요)
    // const doc = new jsPDF();
    // doc.text(`${selectedReport.name} 학습 보고서`, 10, 10);
    // doc.save(`report_${studentName}.pdf`);
    
    // 옵션 3: 서버 기반 PDF 생성
    // const response = await fetch('/api/reports/generate-pdf', {
    //   method: 'POST',
    //   body: JSON.stringify({ studentId: selectedReport.id })
    // });
    // const blob = await response.blob();
    // downloadFile(blob, `report_${studentName}.pdf`);
  } catch (error) {
    toast.error('리포트 출력 실패');
  }
};
```

#### 5. Excel 다운로드 개선

**Report 페이지** (handleDownloadExcel):
```typescript
// CSV → Excel (.xlsx) 변환
// 라이브러리: xlsx, export-to-csv, papaparse 등

import * as XLSX from 'xlsx';

const handleDownloadExcel = () => {
  const ws = XLSX.utils.json_to_sheet(filteredReports);
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'Students');
  XLSX.writeFile(wb, `학생_학습_보고서_${new Date().toISOString().split('T')[0]}.xlsx`);
};
```

---

### 🟢 향후 작업 (Priority: LOW)

#### 1. Class 페이지
- 지문 분석 자동화 (생성 시 자동 분석)
- 난이도 차트 시각화
- 유사 지문 추천

#### 2. Course 페이지
- 과정 템플릿 라이브러리
- 과정별 클래스 관리
- 벌크 작업 (일괄 삭제, 상태 변경)

#### 3. Report 페이지
- 대시보드 분석 차트
  - 학생 진도율 분포도
  - 성과 지표별 그래프
  - 반별 학습 현황 히트맵
- 실시간 학습 현황 (WebSocket)
- 맞춤 리포트 생성
- 데이터 내보내기 (PDF, PowerPoint)

#### 4. 공통 기능
- 다크 모드 지원
- 다국어 지원 (i18n)
- 오프라인 모드
- 모바일 앱 버전

---

## 통합 정책 변경사항

### 1. 네비게이션 통합 정책

**변경 전**:
- 페이지별 독립적인 헤더
- NewHeader와 ClassLayout 헤더 중복

**변경 후**:
- 모든 인증 페이지에서 `StudioHeader` 통합 사용
- 일관된 네비게이션 경험 제공

**적용 페이지**:
- `/class` - My Library (지문 관리)
- `/course` - Course Hub (과정 관리)
- `/report` - Class Report (학습 현황)
- 향후: `/dashboard`, `/settings` 등 추가 가능

---

### 2. CEFR 레벨 확장 정책

**18개 레벨 시스템 도입**:
```
기초(Beginner): A1-, A1, A1+
초급(Elementary): A2-, A2, A2+
중급(Intermediate): B1-, B1, B1+
중상급(Upper-Intermediate): B2-, B2, B2+
고급(Advanced): C1-, C1, C1+
최상급(Mastery): C2-, C2, C2+
```

**적용 범위**:
- 지문(Passage) 생성 시 레벨 선택
- 과정(Course) 레벨 지정
- 리포트(Report) 난이도 분석에 사용
- 학생 맞춤형 학습 경로 추천에 활용

**이점**:
- 더 세밀한 난이도 구분
- 국제 표준 (CEFR) 준수
- 학습자별 정확한 수준 파악

---

### 3. 데이터 분류 및 카테고리 정책

#### 과정 분류 (Genre)
```
기본: 일반영어, 비즈니스, 발음, 회화, 문법, 어휘
추가가능: 시험준비, 어린이, 산업별, 특정주제
```

#### 학습 상태 기호
```
O (동그라미) - 완료: 수업 완료, 모든 활동 이수
△ (세모) - 진행중: 진행 중, 일부 완료
- (대시) - 미시작: 수업 전, 미개시
```

#### 성과 평가 기준
```
발음정확도: 90%+(우수), 80-89%(양호), 70%이하(개선필요)
유창성(WPM): 120+(우수), 90-119(양호), 89이하(개선필요)
```

---

### 4. 과정 상태 및 권한 정책

#### 과정 상태 (Status)
| 상태 | 설명 | 학생 포함 | 변경 가능 |
|------|------|---------|---------|
| `in-use` | 배포 중 | 가능 | ↔ not-used |
| `not-used` | 미배포 | 불가 | ↔ in-use |

#### 접근 권한
```
조직 관리자: 모든 과정 CRUD, 강사 관리
강사: 자신의 과정만 CRUD
학생: 포함된 과정만 조회
```

---

### 5. 검색 및 필터링 정책

#### Class 페이지
```
검색: 지문명 기준
필터: AND 조건 (level AND genre)
정렬: 최신순 (기본)
```

#### Course 페이지
```
검색: 과정명 기준
필터: AND 조건 (level AND genre)
정렬: 최신순, 인기순 (학생 수)
```

#### Report 페이지
```
검색: 학생명 기준
필터: AND 조건 (company AND campus AND instructor AND ...)
정렬: 학생명 (가나다순)
집계: 학습기간 (기본값: 1개월)
```

---

### 6. 데이터 내보내기 정책

#### 현재 지원
- CSV 형식 (Report 페이지)

#### 향후 지원
- Excel (.xlsx)
- PDF (지표 포함)
- Google Sheets 연동
- PowerPoint (프레젠테이션)

#### 포함 데이터
- 학생 기본 정보
- Unit 완료 상태
- 모든 성과 지표
- 평가 등급

---

## 통합 IA 구조

### 1. 전체 사이트맵

```
Studio Picklass
├── / (Landing)
│
├── /auth
│   ├── /login
│   ├── /signup
│   └── /forgot-password
│
└── [Authenticated]
    ├── /class (My Library)
    │   ├── Dashboard
    │   │   ├── Greeting + Cards (3개)
    │   │   └── Sidebar (Filters)
    │   ├── Table (지문 목록)
    │   ├── Modals
    │   │   ├── CreatePassageModal
    │   │   ├── PassageDetailModal
    │   │   ├── TwinPassagesModal
    │   │   └── PreviewPassageModal
    │   ├── /lesson/:id (수업 진행)
    │   └── /lesson-setup/:id (수업 설정)
    │
    ├── /course (Course Hub)
    │   ├── Dashboard
    │   │   ├── Greeting + Cards (3개)
    │   │   └── Sidebar (Filters)
    │   ├── Table (과정 목록)
    │   └── Modals
    │       ├── CreateCourseModal
    │       ├── EditCourseModal
    │       ├── CourseDetailModal
    │       └── DeleteConfirmDialog
    │
    ├── /report (Class Report)
    │   ├── Dashboard
    │   │   ├── Greeting
    │   │   ├── Cards (3개) - Data Export, Top Progress, Top Completion
    │   │   └── Filters (7개 필터)
    │   ├── Table (학생 학습 현황)
    │   ├── Legend (기호 설명)
    │   └── Modal
    │       └── StudentDetailModal
    │
    ├── /dashboard (향후)
    │   ├── Overview Cards
    │   ├── Charts
    │   └── Analytics
    │
    ├── /settings (향후)
    │   ├── Account
    │   ├── Organization
    │   └── Preferences
    │
    └── /legal/**
        ├── /terms-of-service
        ├── /privacy-policy
        └── /refund-policy
```

### 2. StudioHeader 네비게이션 구조

```
StudioHeader
├── Logo (클릭 시 /)
│
├── Navigation Tabs (활성 표시)
│   ├── 📚 My Library (/class)
│   ├── 📖 Course Hub (/course)
│   └── 📊 Class Report (/report)
│
└── Settings
    ├── Account Info
    │   ├── Display Name
    │   ├── Email
    │   └── User ID
    │
    └── Actions (향후)
        ├── Preferences
        ├── Support
        └── Logout
```

### 3. Class 페이지 IA

```
/class (My Library)
│
├── Header Section
│   ├── Greeting: "Hello, Oizi. Pick your class."
│   └── Card Group (3개)
│       ├── 최근 학습한 지문
│       ├── 최근 생성한 지문
│       └── 이 달의 Discover Pick
│
├── Sidebar + Table Layout
│   │
│   ├── Sidebar
│   │   ├── Title: "My Library"
│   │   ├── Search Input
│   │   └── Folders
│   │       ├── View All (전체 보기)
│   │       ├── My Folder
│   │       └── Create New Folder
│   │
│   └── Main Table
│       ├── Filter Bar
│       │   ├── CEFR (6개 옵션)
│       │   ├── 단어수
│       │   ├── 장르
│       │   └── Sort (최신순)
│       │
│       └── Table
│           ├── Header: 제목, 수업횟수, 제작자, 생성일
│           ├── Rows (페이지네이션 10개)
│           └── Pagination
│
└── Modals
    ├── CreatePassageModal (AI 생성 / 수동 입력)
    ├── PassageDetailModal (분석 포함)
    ├── TwinPassagesModal
    └── PreviewPassageModal
```

### 4. Course 페이지 IA

```
/course (Course Hub)
│
├── Header Section
│   ├── Greeting: "Hello, Oizi. Manage your courses."
│   └── Card Group (3개)
│       ├── 최근 생성한 과정
│       ├── 배포 중인 과정
│       └── 인기 있는 과정
│
├── Sidebar + Table Layout
│   │
│   ├── Sidebar
│   │   ├── Title: "Courses"
│   │   ├── Search Input (과정명)
│   │   └── Categories
│   │       ├── All Courses (전체)
│   │       ├── In-use (배포 중)
│   │       └── Not-used (미배포)
│   │
│   └── Main Table
│       ├── Filter Bar
│       │   ├── CEFR (18개 옵션)
│       │   ├── Genre (6개 옵션)
│       │   └── Sort
│       │
│       ├── Action Bar
│       │   ├── Create Button (+ 새 과정)
│       │   └── Bulk Actions (향후)
│       │
│       └── Table
│           ├── Header: 과정명, 레벨, 장르, 학생수, 상태
│           ├── Rows (페이지네이션 10개)
│           └── Pagination
│
└── Modals / Dialogs
    ├── CreateCourseModal
    ├── EditCourseModal
    ├── CourseDetailModal
    └── DeleteConfirmDialog
```

### 5. Report 페이지 IA

```
/report (Class Report)
│
├── Header Section
│   ├── Greeting: "Hello, Oizi. View Class Reports."
│   └── Card Group (3개)
│       ├── 데이터 관리 (Excel 다운로드)
│       ├── 진도율 상위 (상위 1명)
│       └── 학습완료율 상위 (완료도 높은 1명)
│
├── Filter Section (7개 필터)
│   ├── 업체명(본사) - Select
│   ├── 캠퍼스 - Select
│   ├── 담당(강사) - Select
│   ├── 과정 - Select
│   ├── 클래스(반) - Select
│   ├── 개인(학생명) - Search Input
│   └── 학습기간 - Select (1개월 기본)
│
├── Student Learning Table
│   ├── Column Headers
│   │   ├── 학생 이름
│   │   ├── Unit Progress (U1-U10)
│   │   ├── 진도율, 어휘, 읽기유창성
│   │   ├── 발음정확도, 말하기유창성, 문장구조
│   │   ├── 평균문장길이, 문항정답률
│   │   └── 리포트 출력 버튼
│   │
│   ├── Rows (페이지네이션 10개)
│   │   ├── Unit 상태 배지 (색상 표시)
│   │   ├── 수치 데이터
│   │   └── 출력 버튼 (모달 열기)
│   │
│   └── Pagination
│
├── Legend Section
│   ├── Unit Status Symbols (O/△/-)
│   ├── Pronunciation Accuracy Scale
│   └── Fluency Scale
│
└── Modals
    ├── StudentDetailModal
    │   ├── Student Info
    │   ├── Unit Progress Grid
    │   ├── Performance Metrics (6개 카드)
    │   └── Actions (Close, Print)
    │
    └── (향후) Report Print / Export Dialog
```

---

## 데이터 모델

### 1. Passage 스키마 (Class)

```typescript
interface Passage {
  id: string;
  title: string;
  content: string;
  level: CEFRLevel;          // 18개 레벨
  wordCount: string;         // "200~400" 형식
  category: string;
  genre: string;
  type: 'generated' | 'manual' | 'imported';
  author: string;
  classCount: number;
  createdAt: string;
  updatedAt: string;
  
  // 분석 데이터 (선택사항)
  analysis?: {
    vocabularyDifficulty: CEFRLevel;
    vocabularyVariety: CEFRLevel;
    textLength: CEFRLevel;
    sentenceLength: CEFRLevel;
    sentenceStructure: CEFRLevel;
    grammarVariety: CEFRLevel;
    informationDensity: CEFRLevel;
    backgroundKnowledge: CEFRLevel;
  };
}

type CEFRLevel = 
  | 'A1-' | 'A1' | 'A1+'
  | 'A2-' | 'A2' | 'A2+'
  | 'B1-' | 'B1' | 'B1+'
  | 'B2-' | 'B2' | 'B2+'
  | 'C1-' | 'C1' | 'C1+'
  | 'C2-' | 'C2' | 'C2+';
```

### 2. Course 스키마

```typescript
interface Course {
  id: string;
  title: string;
  level: CEFRLevel;
  genre: string;
  description?: string;
  status: 'in-use' | 'not-used';
  studentCount: number;
  classCount?: number;
  totalLessons?: number;
  estimatedHours?: number;
  createdBy: string;
  createdAt: string;
  updatedAt: string;
}
```

### 3. StudentLearningReport 스키마

```typescript
interface StudentLearningReport {
  id: string;
  name: string;
  classId: string;
  periodStart: string;
  periodEnd: string;
  
  // Unit Progress (10개)
  unitStatuses: ('O' | '△' | '-')[];  // unit1-10
  progressRate: number;                // 0-100%
  
  // Vocabulary Metrics
  vocaSize: number;                    // 어휘 개수
  
  // Fluency Metrics
  readingWPM: number;                  // Words Per Minute
  speakingWPM: number;
  
  // Accuracy Metrics
  pronunciationAccuracy: number;       // 0-100%
  
  // Structure Analysis
  sentenceStructure: CEFRLevel;        // 문법 수준
  avgSentenceLength: number;           // 단어 수
  
  // Test Metrics
  correctAnswerRate: number;           // 0-100%
  
  // Metadata
  createdAt: string;
  updatedAt: string;
}
```

---

## 컴포넌트 구조

### 1. 공통 컴포넌트

```
components/
├── oizi/
│   ├── StudioHeader.tsx         ✅ 신규 (네비게이션)
│   ├── NewHeader.tsx            (deprecated)
│   └── ClassLayout.tsx          ✅ 수정 (NewHeader 제거)
│
├── Modals/
│   ├── CreatePassageModal.tsx   ✅ 수정 (18 CEFR)
│   ├── PassageDetailModal.tsx   ✅ 수정 (분석 추가)
│   ├── TwinPassagesModal.tsx    (기존)
│   ├── PreviewPassageModal.tsx  (기존)
│   ├── CreateCourseModal.tsx    ✅ 신규
│   ├── EditCourseModal.tsx      ✅ 신규
│   └── CourseDetailModal.tsx    ✅ 신규
│
├── ui/
│   ├── button.tsx               (기존)
│   ├── input.tsx                (기존)
│   ├── badge.tsx                (기존)
│   ├── dialog.tsx               (기존)
│   ├── alert-dialog.tsx         (기존)
│   └── simple-pagination.tsx    (기존)
│
└── PassageTypeBar... (기존)
```

### 2. 페이지별 구조

```
Class Page (/class)
├── StudioHeader           (공유)
├── Dashboard
│   ├── Greeting Section
│   └── Card Group (3개)
├── Sidebar + Main
│   ├── Sidebar (Folders)
│   ├── Filter Bar
│   └── Table + Pagination
└── Modals (필요 시)

Course Page (/course)
├── StudioHeader           (공유)
├── Dashboard
│   ├── Greeting Section
│   └── Card Group (3개)
├── Sidebar + Main
│   ├── Sidebar (Categories)
│   ├── Filter Bar
│   └── Table + Pagination
└── Modals (필요 시)

Report Page (/report)
├── StudioHeader           (공유)
├── Dashboard
│   ├── Greeting Section
│   └── Card Group (3개)
├── Filter Bar (7개)
├── Table + Legend
├── Pagination
└── Modal (상세 보기)
```

---

## 배포 체크리스트

### Phase 1: 기초 (Priority: HIGH)
- [ ] API 엔드포인트 구현
  - [ ] Course API (CRUD)
  - [ ] Report API (조회)
  - [ ] Passage Analysis API
- [ ] Mock 데이터 → API 교체
- [ ] 에러 처리 & 로딩 상태
- [ ] 기본 기능 테스트

### Phase 2: 기능 (Priority: MEDIUM)
- [ ] 인증 시스템 연동
- [ ] 필터 옵션 동적 생성
- [ ] Excel 다운로드 개선 (XLSX)
- [ ] 리포트 출력 기능
- [ ] 벌크 작업 기능

### Phase 3: 최적화 (Priority: LOW)
- [ ] 성능 최적화 (캐싱, 지연 로딩)
- [ ] UI/UX 애니메이션
- [ ] 다크 모드 지원
- [ ] 국제화 (i18n)

### 배포 전 필수 확인
- [ ] 모든 Mock 데이터 제거
- [ ] AUTH 관련 TODO 완료
- [ ] 환경 변수 설정 (.env.local)
- [ ] API 엔드포인트 확인
- [ ] 브라우저 호환성 테스트
- [ ] 모바일 반응형 테스트
- [ ] 접근성 검토 (WCAG 2.1)
- [ ] 보안 검토
- [ ] 성능 검사

---

## 참고 자료

- **CEFR 레벨**: https://www.coe.int/en/web/common-european-framework-reference-languages
- **UI 라이브러리**: shadcn/ui (https://ui.shadcn.com)
- **스타일링**: Tailwind CSS (https://tailwindcss.com)
- **아이콘**: Lucide React (https://lucide.dev)

---

## 🔍 필터링 및 검색 기능 구현 (Class 페이지)

**작성일**: 2026-03-13  
**관련 파일**: 
- `apps/web/src/app/class/page.tsx`
- `packages/shared/src/constants/index.ts` (WORD_COUNT_RANGES)

### 상황
Class 페이지의 필터 버튼(CEFR, 단어수, 장르)이 UI 레벨에서만 존재했고, 실제 필터링 로직이 구현되지 않았습니다.

### 구현 내용

#### 1. 공유 상수 추가

**WORD_COUNT_RANGES** (20개 범위):
```typescript
export const WORD_COUNT_RANGES = [
  { label: '0~100', value: '0~100' },
  { label: '100~200', value: '100~200' },
  // ... (100씩 증가)
  { label: '1900~2000', value: '1900~2000' },
] as const;
```

**목적**:
- Class, Course 페이지에서 동일한 단어수 범위 사용
- DRY 원칙 준수
- 타입 안전성 제공

#### 2. State 추가

```typescript
const [filterLevel, setFilterLevel] = useState<string>('');
const [filterGenre, setFilterGenre] = useState<string>('');
const [filterWordCount, setFilterWordCount] = useState<string>('');
const [searchText, setSearchText] = useState<string>('');
const [openLevelDropdown, setOpenLevelDropdown] = useState(false);
const [openGenreDropdown, setOpenGenreDropdown] = useState(false);
const [openWordCountDropdown, setOpenWordCountDropdown] = useState(false);
```

#### 3. 필터링 로직

```typescript
const filteredLibraryTexts = useMemo(() => {
  return libraryTexts.filter((text) => {
    if (searchText && !text.title.toLowerCase().includes(searchText.toLowerCase())) return false;
    if (filterLevel && text.level !== filterLevel) return false;
    if (filterGenre && text.category !== filterGenre) return false;
    if (filterWordCount && text.wordCountRange !== filterWordCount) return false;
    return true;
  });
}, [libraryTexts, searchText, filterLevel, filterGenre, filterWordCount]);
```

#### 4. 검색창 구현

```jsx
<Input
  placeholder="지문 찾기"
  value={searchText}
  onChange={(e) => {
    setSearchText(e.target.value);
    setCurrentPage(1);  // 검색 시 첫 페이지로 리셋
  }}
  className="h-10 rounded-full border-gray-400 bg-white pl-10 text-sm text-gray-900 placeholder:text-gray-400"
/>
```

**동작**:
- 실시간 제목 검색 (대소문자 구분 안 함)
- 검색 시 페이지를 1로 리셋
- 다른 필터와 조합 가능

#### 5. 드롭다운 필터 UI

구현된 3개 필터 (드롭다운 형태):
1. **CEFR Filter** - LEVEL_SYSTEM.map()
2. **Word Count Filter** - WORD_COUNT_RANGES.map()
3. **Genre Filter** - GENRES.map()

**UI 특징**:
- 선택값이 버튼에 표시
- 드롭다운 자동 닫기 (다른 필터 열 때)
- "전체" 옵션으로 필터 해제 가능
- 필터 선택 시 페이지 1로 자동 리셋
- Shadow와 z-50으로 레이어 관리

#### 6. WordCount 관련 함수 분리

**주요 개선**:

```typescript
// 필터링용: 범위 계산
const getWordCountRange = (content: string): string => {
  const wordCount = content.trim().split(/\s+/).length;
  for (const range of WORD_COUNT_RANGES) {
    const [min, max] = range.value.split('~').map(Number);
    if (wordCount >= min && wordCount <= max) return range.value;
  }
  return WORD_COUNT_RANGES[WORD_COUNT_RANGES.length - 1].value;
};

// 표시용: 실제 단어수
const formatWordCount = (content: string): string => {
  const wordCount = content.trim().split(/\s+/).length;
  return `${wordCount} Words`;
};
```

**중요**:
- `wordCount`: 테이블에 표시 ("234 Words")
- `wordCountRange`: 필터링에 사용 ("200~300")

#### 7. libraryTexts 데이터 구조 변경

```typescript
const libraryTexts = useMemo(() => {
  return texts.map((text) => ({
    ...text,
    wordCount: formatWordCount(text.content),          // 표시용
    wordCountRange: getWordCountRange(text.content),   // 필터용
    // ... 기타 필드
  }));
}, [texts]);
```

### 호환성

#### Course 페이지 통합
`course/[courseId]/page.tsx`의 "레슨 추가" 모달에서:
```typescript
import { WORD_COUNT_RANGES } from '@classsnap/shared';

// Step 1에서 단어수 필터
<select value={filterLessonWordCount} onChange={...}>
  {WORD_COUNT_RANGES.map((range) => (
    <option value={range.value}>{range.label}</option>
  ))}
</select>
```

하드코딩된 8개 범위 → 공유 상수의 20개 범위로 통일

### 개선 효과

✅ **사용자 경험**:
- 실시간 검색으로 빠른 지문 찾기
- 다중 필터로 정밀한 검색
- 선명한 드롭다운 UI

✅ **개발 효율**:
- 공유 상수로 코드 중복 제거
- 일관된 데이터 구조
- 유지보수 용이

✅ **데이터 정확성**:
- 실제 단어수와 필터링 범위 분리
- WORD_COUNT_RANGES에 따른 동적 범위 할당
- 자동으로 올바른 범위로 분류

### 개발자 추가 작업

- [ ] 필터 초기화 기능 ("필터 전체 초기화" 버튼)
- [ ] 필터 프리셋 저장 (자주 사용하는 조합)
- [ ] 검색 결과 카운트 표시 ("검색 결과: ○개")
- [ ] 정렬 기능 구현 (현재는 버튼만 존재)
- [ ] 모바일 드롭다운 수정 (화면 밖 나감 이슈)
- [ ] 검색 하이라이트 (검색어 강조 표시)
- [ ] 필터 히스토리 (최근 검색 필터)

---

**마지막 업데이트**: 2026-03-13  
**버전**: 2.1 (필터링 & 검색 기능 추가)
