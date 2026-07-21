# Course Page - 개발 업데이트 문서

**작성일**: 2026-03-11  
**최종업데이트**: 2026-03-14  
**버전**: 1.1  
**담당자**: UI/UX & Backend Team  
**관련 페이지**: `/course` (Course Hub)

---

## 📋 목차

1. [변경 사항 개요](#변경-사항-개요)
2. [개발자 추가 작업](#개발자-추가-작업)
3. [정책 변경사항](#정책-변경사항)
4. [데이터 모델](#데이터-모델)
5. [컴포넌트 구조](#컴포넌트-구조)
6. [상태 관리](#상태-관리)

---

## 변경 사항 개요

### 1. Course 페이지 신규 생성

**파일**: `apps/web/src/app/course/page.tsx`  
**경로**: `/course`  
**헤더**: StudioHeader (통합 네비게이션)

### 2. 설계 기준

- Class 페이지(`/class`)와 **동일한 디자인** 및 **동일한 구조**
- 과정(Course) 관리에 특화된 기능 구현
- Mock 데이터로 초기 구현 완료

### 3. 핵심 기능

| 기능 | 설명 | 상태 |
|------|------|------|
| 과정 목록 조회 | 전체 과정 표시 (테이블) | ✅ UI 완료 |
| 과정 생성 | 새로운 과정 추가 모달 | ✅ UI 완료 |
| 과정 수정 | 기존 과정 정보 수정 | ✅ UI 완료 |
| 과정 삭제 | 삭제 확인 다이얼로그 | ✅ UI 완료 |
| 과정 상세 보기 | 과정별 상세 정보 모달 | ✅ UI 완료 |
| 필터링 | 레벨, 장르별 필터링 | ✅ UI 완료 |
| 검색 | 과정명 검색 | ✅ UI 완료 |
| 정렬 | 최신순 정렬 | ✅ UI 완료 |
| 페이지네이션 | 10개 항목씩 표시 | ✅ UI 완료 |
| 카드 그룹 | 최근 생성한 / 배포 중인 / 인기 있는 | ✅ UI 완료 |

### 4. 2026-03-14 업데이트 내역

#### 과정 생성 마법사 개선

**파일**: `apps/web/src/app/course/page.tsx`

- **wordCount 데이터 타입 정규화**: 범위(string) → 실제 단어수(number) 변경
  - mockLessons: L001-L030 모두 wordCount를 숫자로 통일
  - 범위(예: '200~300')를 중간값으로 자동 변환
  
- **수업모듈 선택 영역 개선**
  - 레이아웃 변경: `flex flex-wrap` → `grid grid-cols-4`
  - 각 스킬별 모듈을 한 행에 4개씩 배치
  - 더 깔끔한 모듈 선택 UI

#### 과정 상세 페이지 개선

**파일**: `apps/web/src/app/course/[courseId]/page.tsx`

- **headerSection 높이 감소**
  - margin-bottom: mb-12 → mb-4
  - 뒤로가기 버튼만 표시
  
- **테이블 섹션 레이아웃 개선**
  - 과정 제목을 좌측에 고정 배치 (whitespace-nowrap)
  - 과정 목표와 뱃지를 우측에 정렬
  - 수평 레이아웃: `flex gap-8 items-start`
  - 중앙에 flex-1 빈 공간으로 자동 정렬
  
- **wordCount 데이터 타입 정규화**
  - mockLessons: 문자열 범위 → 숫자로 변경
  - 레슨 추가 시 범위를 중간값으로 자동 변환

#### CourseSidebar 네비게이션 개선

**파일**: `apps/web/src/components/CourseSidebar.tsx`

- **"모든 과정" 버튼화**
  - 기존 div → button 요소로 변경
  - onClick 핸들러로 `/course` 페이지 이동
  - hover 시 배경색 변화 (bg-[#d4f0e9])
  - useRouter hook으로 네비게이션 구현

#### 과정 주제 생성 AI 프롬프트 추가

**신규 파일**: `prompts/course_topic_generation_prompt.md`

- **목적**: 과정 생성 마법사 (1/2) → (2/2) 단계 전환 시 AI를 통한 자동 주제 생성
- **기능**:
  - 입력: 과정명(courseName), 과정목표(courseGoal), CEFR 레벨, 장르
  - 출력: 회차별 2~3줄의 최신 정보 기반 주제 배열
  - AI 엔진: Google Gemini 2.5 Flash
  
- **지원 장르 (12가지)**:
  - 가이드 (Guide)
  - 논설문 (Editorial/Opinion Essay)
  - 뉴스 기사 (News Article)
  - 단편소설 (Short Story)
  - 대화문 (Dialogue)
  - 리뷰 (Review)
  - 보고서 (Report)
  - 설명문 (Explanatory Text)
  - 스토리 (Story)
  - 에세이 (Essay)
  - 일기 (Diary)
  - 편집 (Editing)
  
- **구성 요소**:
  - 📌 개요: 프롬프트 목적 및 적용 지점
  - 📝 프롬프트 원문: 메인 프롬프트 + 장르별 상세 가이드라인 (12가지 모두)
  - 📋 입력 파라미터: courseName, courseGoal, wizardLevel, selectedGenre, lessonRound
  - 📤 출력 형식: JSON 형식 (topics 배열 + courseFocus 요약)
  - 🎯 가이드라인: 최신 정보 기반, 레벨별 난이도 조정, 장르 일관성 등
  - 💾 구현 예시: TypeScript functional code + genreGuidanceMap (12개 장르 매핑) + 페이지 통합 예시

- **적용 페이지**:
  - `apps/web/src/app/course/page.tsx`
  - `generateAITopics()` 함수 from Step 1/2 → Step 2/2 전환 시 호출

---

## 개발자 추가 작업

### 🔴 필수 작업 (Priority: HIGH)

#### 1. Backend API 구현

**1.1 과정 목록 조회**
```
GET /api/courses
Query Parameters:
  - search?: string          # 과정명 검색
  - level?: string           # CEFR 레벨 필터 (A1, A2, B1, ...)
  - genre?: string           # 장르 필터
  - page?: number            # 페이지 번호 (기본값: 1)
  - limit?: number           # 페이지당 항목 수 (기본값: 10)

Response:
{
  data: [
    {
      id: string (UUID),
      title: string,
      level: string,          # CEFR 레벨 (18개 확장)
      genre: string,
      description?: string,
      status: 'in-use' | 'not-used',
      studentCount: number,
      createdAt: ISO8601,
      updatedAt: ISO8601,
      createdBy: string       # 강사명
    }
  ],
  total: number,
  page: number,
  totalPages: number
}
```

**1.2 과정 생성**
```
POST /api/courses
Body:
{
  title: string,             # 필수
  level: string,             # 필수 (18개 CEFR 레벨)
  genre: string,             # 필수
  description?: string,
  learningDuration?: number  # 학습 기간 (시간) 
}

Response:
{
  id: string,
  title: string,
  level: string,
  genre: string,
  status: 'in-use',
  createdAt: ISO8601
}
```

**1.3 과정 수정**
```
PUT /api/courses/:id
Body:
{
  title?: string,
  level?: string,
  genre?: string,
  description?: string,
  status?: 'in-use' | 'not-used'
}

Response:
{
  id: string,
  title: string,
  ...
  updatedAt: ISO8601
}
```

**1.4 과정 삭제**
```
DELETE /api/courses/:id

Response:
{
  success: boolean,
  message: string
}
```

**1.5 과정 상세 조회**
```
GET /api/courses/:id

Response:
{
  id: string,
  title: string,
  level: string,
  genre: string,
  description: string,
  status: 'in-use' | 'not-used',
  studentCount: number,
  classCount: number,
  totalLessons: number,
  estimatedHours: number,
  createdBy: string,
  createdAt: ISO8601,
  updatedAt: ISO8601
}
```

---

#### 2. Mock 데이터 교체

**현재 상태**:
```javascript
// 파일: apps/web/src/app/course/page.tsx (Line 61-96)
const mockCourses = [
  {
    id: '1',
    title: 'General English Course',
    level: 'B1',
    genre: '일반영어',
    status: 'in-use',
    studentCount: 25,
  },
  // ... 더 많은 데이터
];
```

**작업 내용**:
1. `mockCourses` 및 `mockRecentCourses` 제거
2. `useEffect` 또는 `useSWR` 훅으로 API 데이터 호출
3. Loading, Error 상태 처리
4. 필터링/검색 시 API 호출 로직 추가

**권장 구현**:
```typescript
// useEffect + fetch
const fetchCourses = async () => {
  try {
    const response = await fetch(
      `/api/courses?search=${searchTerm}&level=${levelFilter}&genre=${genreFilter}&page=${currentPage}`
    );
    const data = await response.json();
    setCourses(data.data);
    setTotalPages(data.totalPages);
  } catch (error) {
    console.error('Failed to fetch courses:', error);
  }
};
```

---

#### 3. 생성/수정/삭제 모달 API 연동

**생성 모달**:
```typescript
const handleCreateCourse = async (formData: CourseFormData) => {
  try {
    const response = await fetch('/api/courses', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(formData),
    });
    const newCourse = await response.json();
    // 목록 새로고침
    fetchCourses();
    setIsCreateModalOpen(false);
  } catch (error) {
    console.error('Failed to create course:', error);
  }
};
```

**수정 모달**:
```typescript
const handleUpdateCourse = async (courseId: string, formData: CourseFormData) => {
  try {
    const response = await fetch(`/api/courses/${courseId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(formData),
    });
    // 목록 새로고침
    fetchCourses();
    setIsEditModalOpen(false);
  } catch (error) {
    console.error('Failed to update course:', error);
  }
};
```

**삭제**:
```typescript
const handleDeleteCourse = async (courseId: string) => {
  try {
    await fetch(`/api/courses/${courseId}`, { method: 'DELETE' });
    // 목록 새로고침
    fetchCourses();
    setDeleteConfirmOpen(false);
  } catch (error) {
    console.error('Failed to delete course:', error);
  }
};
```

---

### 🟡 권장 작업 (Priority: MEDIUM)

#### 1. 필터 옵션 동적 생성

현재는 하드코딩된 옵션:
```jsx
<select>
  <option value="A1">A1</option>
  <option value="A2">A2</option>
  {/* ... 18개 CEFR 레벨 */}
</select>
```

**개선 방안**:
```typescript
// Util 함수
const CEFR_LEVELS = [
  'A1-', 'A1', 'A1+',
  'A2-', 'A2', 'A2+',
  'B1-', 'B1', 'B1+',
  'B2-', 'B2', 'B2+',
  'C1-', 'C1', 'C1+',
  'C2-', 'C2', 'C2+'
];

const GENRES = ['일반영어', '비즈니스', '발음', '회화', '문법', '어휘'];

// 컴포넌트에서 사용
{CEFR_LEVELS.map(level => (
  <option key={level} value={level}>{level}</option>
))}
```

#### 2. 상태 배지 스타일링

현재:
```jsx
{status === 'in-use' ? '배포 중' : '미배포'}
```

**개선**:
```jsx
<Badge className={status === 'in-use' ? 'bg-green-100 text-green-700' : 'bg-gray-100 text-gray-500'}>
  {status === 'in-use' ? '배포 중' : '미배포'}
</Badge>
```

#### 3. 과정 상세 모달 데이터 표시

현재: 기본 정보만 표시  
추가할 정보:
- 과정 설명 (description)
- 수강생 수
- 클래스 수
- 총 레슨 수
- 예상 학습 시간
- 학습 시작/종료 날짜 (선택사항)
- 강사명

---

#### 4. 벌크 작업 기능

**선택 삭제**:
```typescript
const [selectedCourses, setSelectedCourses] = useState<string[]>([]);

const handleBulkDelete = async () => {
  await Promise.all(
    selectedCourses.map(id => fetch(`/api/courses/${id}`, { method: 'DELETE' }))
  );
  fetchCourses();
};
```

**선택 상태 변경**:
```typescript
const handleBulkStatusChange = async (status: 'in-use' | 'not-used') => {
  await Promise.all(
    selectedCourses.map(id => 
      fetch(`/api/courses/${id}`, {
        method: 'PUT',
        body: JSON.stringify({ status })
      })
    )
  );
  fetchCourses();
};
```

---

### 🟢 향후 작업 (Priority: LOW)

#### 1. 과정 별 클래스 관리
- 과정 → 클래스 → 레슨 구조
- 클래스 추가/수정/삭제
- 클래스별 학생 현황

#### 2. 과정 템플릿 라이브러리
- 미리 만들어진 과정 템플릿
- 원클릭 복제 기능
- 커뮤니티 템플릿 공유

#### 3. 과정 분석
- 과정별 학생 진도율
- 과정별 완료율 분석
- 과정 효과성 평가

#### 4. 과정 일정 관리
- 캘린더 뷰
- 시즌별 과정 관리
- 자동 스케줄 제안

---

## 정책 변경사항

### 1. 과정 상태 정책

**상태 종류**:
| 상태 | 설명 | 변경 가능한 상태 |
|------|------|-----------------|
| `in-use` | 배포 중 (학생 포함) | → not-used |
| `not-used` | 미배포 (학생 없음) | → in-use |

**정책 규칙**:
- 생성 시 기본값: `in-use`
- 학생이 포함된 과정은 `not-used`로 변경 불가 (→ 경고 메시지)
- 삭제 시 확인 다이얼로그 필수

---

### 2. 과정 정보 입력 규칙

**필수 입력**:
- 과정명 (최대 100자)
- CEFR 레벨
- 장르

**선택 입력**:
- 과정 설명 (최대 500자)
- 학습 기간 (시간)
- 이미지/썸네일

**유효성 검사**:
- 과정명 중복 확인 여부 (기존 과정과 동일한 명 불허)
- 특수문자 제한 (또는 이스케이프 처리)

---

### 3. CEFR 레벨 정책

**18개 레벨 지원**:
```
A1- < A1 < A1+ < A2- < A2 < A2+ < B1- < B1 < B1+ < 
B2- < B2 < B2+ < C1- < C1 < C1+ < C2- < C2 < C2+
```

**선택 규칙**:
- 하나의 과정은 하나의 CEFR 레벨만 가능
- 향후 레벨 범위 지정 옵션 추가 가능 (예: A1~B1 과정)

---

### 4. 과정 분류(Genre) 정책

**기본 장르**:
- 일반영어 (General English)
- 비즈니스 (Business)
- 발음 (Pronunciation)
- 회화 (Conversation)
- 문법 (Grammar)
- 어휘 (Vocabulary)

**추가 가능성**:
- 시험 준비 (Exam Prep: TOEIC, IELTS, TOEFL)
- 어린이 영어 (Kids)
- 어른 영어 (Adult)
- 산업별 (IT, Healthcare, Hospitality)

---

### 5. 과정 검색 및 필터링

**검색 범위**:
- 과정명
- 과정 설명 (선택사항)

**필터 조합 가능**:
- AND 조건: level AND genre
- OR 조건: 미지원 (향후 추가 검토 가능)

**정렬 옵션**:
- 최신순 (생성일)
- 인기순 (수강생 수)
- 이름순 (가나다순)

---

## 데이터 모델

### Course 스키마

```typescript
interface Course {
  id: string;                    // UUID
  title: string;                 // 과정명 (필수)
  level: CEFRLevel;             // CEFR 레벨 (필수)
  genre: string;                // 장르 (필수)
  description?: string;         // 과정 설명
  status: CourseStatus;         // 'in-use' | 'not-used'
  studentCount: number;         // 수강생 수
  classCount?: number;          // 클래스 수
  totalLessons?: number;        // 총 레슨 수
  estimatedHours?: number;      // 예상 학습 시간
  thumbnailUrl?: string;        // 썸네일 이미지
  createdBy: string;            // 작성자 (강사명)
  createdAt: ISO8601;           // 생성일
  updatedAt: ISO8601;           // 수정일
  deletedAt?: ISO8601;          // 삭제일 (소프트 삭제)
}

type CEFRLevel = 
  | 'A1-' | 'A1' | 'A1+'
  | 'A2-' | 'A2' | 'A2+'
  | 'B1-' | 'B1' | 'B1+'
  | 'B2-' | 'B2' | 'B2+'
  | 'C1-' | 'C1' | 'C1+'
  | 'C2-' | 'C2' | 'C2+';

type CourseStatus = 'in-use' | 'not-used';
```

### 데이터베이스 스키마

```sql
CREATE TABLE courses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(100) NOT NULL,
  level VARCHAR(10) NOT NULL,
  genre VARCHAR(50) NOT NULL,
  description TEXT,
  status VARCHAR(20) DEFAULT 'in-use',
  student_count INTEGER DEFAULT 0,
  class_count INTEGER DEFAULT 0,
  total_lessons INTEGER DEFAULT 0,
  estimated_hours INTEGER,
  thumbnail_url VARCHAR(255),
  created_by UUID NOT NULL,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now(),
  deleted_at TIMESTAMP,
  
  FOREIGN KEY (created_by) REFERENCES users(id),
  INDEX idx_level (level),
  INDEX idx_genre (genre),
  INDEX idx_status (status),
  INDEX idx_created_at (created_at DESC)
);
```

---

## 컴포넌트 구조

### 파일 구조

```
apps/web/src/app/course/
├── page.tsx                    # 메인 페이지 컴포넌트
└── layout.tsx                  # (기존) 레이아웃

apps/web/src/components/
├── CreateCourseModal.tsx       # (기존) 과정 생성 모달
├── EditCourseModal.tsx         # (기존) 과정 수정 모달  
├── CourseDetailModal.tsx       # (기존) 과정 상세 모달
└── oizi/
    └── StudioHeader.tsx        # (기존) 통합 헤더
```

### 컴포넌트 Props

```typescript
// CreateCourseModal
interface CreateCourseModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onSubmit: (data: CourseFormData) => Promise<void>;
  isLoading?: boolean;
}

// EditCourseModal  
interface EditCourseModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  course: Course;
  onSubmit: (data: CourseFormData) => Promise<void>;
  isLoading?: boolean;
}

// CourseDetailModal
interface CourseDetailModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  course: Course;
  onEdit?: () => void;
  onDelete?: () => void;
}
```

---

## 상태 관리

### 페이지 상태 관리

```typescript
export default function CoursePage() {
  // 데이터
  const [courses, setCourses] = useState<Course[]>([]);
  const [totalPages, setTotalPages] = useState(1);
  
  // 필터 & 검색
  const [searchTerm, setSearchTerm] = useState('');
  const [levelFilter, setLevelFilter] = useState('');
  const [genreFilter, setGenreFilter] = useState('');
  const [currentPage, setCurrentPage] = useState(1);
  
  // 모달 상태
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false);
  const [isEditModalOpen, setIsEditModalOpen] = useState(false);
  const [isDetailModalOpen, setIsDetailModalOpen] = useState(false);
  const [deleteConfirmOpen, setDeleteConfirmOpen] = useState(false);
  
  // 선택된 과정
  const [selectedCourse, setSelectedCourse] = useState<Course | null>(null);
  const [selectedCourses, setSelectedCourses] = useState<string[]>([]);
  
  // 로딩 상태
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
}
```

### 상태 변경 흐름

```
사용자 액션
  ↓
API 호출 (fetchCourses)
  ↓
데이터 업데이트 (setCourses)
  ↓
페이지 리렌더링
  ↓
UI 업데이트
```

---

## 테스트 체크리스트

### 기능 테스트

- [ ] 과정 목록 조회
  - [ ] 초기 로딩 시 모든 과정 표시
  - [ ] 10개 항목씩 페이지네이션
  - [ ] 페이지 변경 시 데이터 새로고침

- [ ] 과정 생성
  - [ ] 모달 열고 닫기
  - [ ] 필수 필드 입력 검증
  - [ ] 성공 시 목록에 추가
  - [ ] 실패 시 에러 메시지 표시

- [ ] 과정 수정
  - [ ] 클릭 시 수정 모달 열기
  - [ ] 기존 데이터 프리필 확인
  - [ ] 저장 후 목록 업데이트

- [ ] 과정 삭제
  - [ ] 삭제 확인 다이얼로그 표시
  - [ ] 취소 시 닫기
  - [ ] 확인 시 삭제 및 목록 업데이트

- [ ] 검색 & 필터
  - [ ] 과정명 검색 작동
  - [ ] 레벨 필터링
  - [ ] 장르 필터링
  - [ ] 필터 조합 작동

- [ ] 정렬
  - [ ] 최신순 정렬
  - [ ] 다른 정렬 옵션 (향후)

### UI 테스트

- [ ] 반응형 레이아웃
  - [ ] 데스크톱 (1920px+)
  - [ ] 태블릿 (768px-1920px)
  - [ ] 모바일 (320px-767px)

- [ ] 모달 UI
  - [ ] 모달 배경 클릭 시 닫기
  - [ ] ESC 키 입력 시 닫기
  - [ ] 버튼 상태 (활성/비활성)

- [ ] 로딩 상태
  - [ ] 데이터 로딩 중 스피너 표시
  - [ ] 에러 발생 시 에러 메시지

### 성능 테스트

- [ ] API 응답 시간 (< 1초)
- [ ] 페이지 로드 시간 (< 2초)
- [ ] 메모리 누수 테스트
- [ ] 필터링 성능 (백엔드 최적화)

---

## 배포 전 체크리스트

- [ ] Mock 데이터 모두 API로 교체 완료
- [ ] API 엔드포인트 구현 완료
  - [ ] GET /api/courses
  - [ ] POST /api/courses
  - [ ] PUT /api/courses/:id
  - [ ] DELETE /api/courses/:id
- [ ] 에러 처리 추가
  - [ ] 네트워크 오류
  - [ ] 서버 오류 (5xx)
  - [ ] 클라이언트 오류 (4xx)
- [ ] 로딩 상태 UI 추가
- [ ] Toast 알림 메시지 추가
- [ ] 사용자 입력 검증
- [ ] 환경 변수 설정 (.env.local)
- [ ] API 인증 토큰 관리

---

## 🔍 레슨 추가 모달 (2단계) - 단어수 필터 통합

### 변경 파일
- `apps/web/src/app/course/[courseId]/page.tsx`
- `packages/shared/src/constants/index.ts` (WORD_COUNT_RANGES 추가)

### 상황
course/[courseId]/page.tsx의 "레슨 추가" 모달(1/2 단계)에서 단어수 필터를 사용하고 있었으나, class/page.tsx의 필터와 일관성이 없었습니다.

### 해결책
공유 상수 `WORD_COUNT_RANGES`를 생성하여 두 페이지에서 동일한 단어수 범위를 사용하도록 통합했습니다.

### 구현 상세

#### WORD_COUNT_RANGES 구조
```typescript
export const WORD_COUNT_RANGES = [
  { label: '0~100', value: '0~100' },
  { label: '100~200', value: '100~200' },
  { label: '200~300', value: '200~300' },
  // ... (20개 범위, 100씩 증가)
  { label: '1900~2000', value: '1900~2000' },
] as const;

export type WordCountRange = typeof WORD_COUNT_RANGES[number]['value'];
```

#### course/[courseId]/page.tsx 적용
```typescript
import { MODULE_CATEGORIES, GENRES, CEFR_LEVELS, WORD_COUNT_RANGES } from '@classsnap/shared';

// Step 1 필터 - 단어수 필터
<select
  value={filterLessonWordCount}
  onChange={(e) => setFilterLessonWordCount(e.target.value)}
  className="h-9 rounded border border-gray-300 px-3 text-sm bg-white text-gray-900"
>
  <option value="">단어수 전체</option>
  {WORD_COUNT_RANGES.map((range) => (
    <option key={range.value} value={range.value}>{range.label}</option>
  ))}
</select>
```

**변경 전**: 하드코딩된 8개 범위  
**변경 후**: 공유 상수의 20개 범위

### 이점
✅ 일관된 단어수 범위 데이터 사용  
✅ DRY 원칙 준수  
✅ 향후 단어수 범위 수정 시 한 곳만 변경  
✅ 타입 안전성 제공
- [ ] 빈 상태 메시지 추가 ("과정이 없습니다")
- [ ] 권한 검증 추가 (작성자만 수정/삭제)
- [ ] Accessibility 검토
  - [ ] 키보드 네비게이션
  - [ ] 스크린 리더 호환성
  - [ ] 색상 대비 확인
- [ ] 브라우저 호환성 테스트
  - [ ] Chrome 최신
  - [ ] Safari 최신
  - [ ] Firefox 최신
  - [ ] Edge 최신

---

## 참고 자료

- **Class 페이지**: `apps/web/src/app/class/page.tsx` (동일한 구조 참고)
- **CEFR 레벨**: [정의](https://www.coe.int/en/web/common-european-framework-reference-languages)
- **UI 컴포넌트**: shadcn/ui + Tailwind CSS
- **API 문서**: (백엔드 문서 링크)

---

## 이슈 추적

| 이슈 | 상태 | 담당자 | 기한 |
|------|------|--------|------|
| API 구현 | 🔴 미작업 | - | - |
| 링크 추가 필요: Class ↔ Course | 🟡 진행 중 | - | - |
| 권한 관리 | 🔴 미작업 | - | - |

---

**마지막 업데이트**: 2026-03-11
