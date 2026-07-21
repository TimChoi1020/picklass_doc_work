# 과정 상세 페이지 — course-detail-20260316

**경로:** `apps/web/src/app/course/[courseId]/page.tsx`
**라우트:** `/course/[courseId]`
**작성일:** 2026-03-16
**이전 문서:** `docs/course_modi.md` (2026-03-14) — courseId 섹션

---

## 1. 사용자 흐름 (User Flow)

```
/course/[courseId] 진입
  ├── 뒤로가기 버튼 (←) → router.back()
  └── 과정 정보 헤더: title (좌측 고정), goal + 뱃지(레슨수/레벨/단어수/장르) (우측)

레슨 목록 (테이블):
  ├── 제목 검색 (CourseSidebar searchTitle → 레슨 필터링)
  ├── 헤더: 회차 | 제목 | 레벨+단어수+장르 | 모듈 | 생성일 | 상태
  └── 행 클릭 → 오버레이 표시
        ├── ↑ 버튼 → 레슨 위로 이동 (round 재정렬)
        ├── ↓ 버튼 → 레슨 아래로 이동 (round 재정렬)
        ├── "모듈 수정" (in-use이면 disabled) → 모듈 수정 모달
        ├── "+레슨추가" → 레슨 추가 모달 오픈 (선택된 레슨 앞에 삽입)
        └── "과정에서제외" (in-use이면 disabled) → AlertDialog → 삭제 후 round 재정렬

모듈 수정 모달:
  입력: MODULE_CATEGORIES checkbox
  → "수정 완료" → 해당 레슨 modules 업데이트 + toast.success

레슨 추가 모달 (1/2):
  ├── 검색 (제목/레벨/장르)
  ├── 필터: CEFR 레벨, 단어수(WORD_COUNT_RANGES), 장르(GENRES)
  └── 라디오 선택 → "다음" → Step 2

레슨 추가 모달 (2/2):
  ├── 선택된 텍스트 정보 표시
  ├── MODULE_CATEGORIES checkbox
  └── "추가" (모듈 1개 이상 선택 필수)
        → addLessonSourceLessonId 위치 앞에 삽입
        → round 재정렬
        → toast.success
```

---

## 2. IA 구조 (Information Architecture)

```
/course/[courseId]
├── StudioHeader
├── 뒤로가기 버튼 (mb-4)
└── 라이브러리 섹션 (CourseSidebar + 테이블)
    ├── CourseSidebar
    │   ├── 검색 (searchTitle → 레슨 제목 필터)
    │   ├── "모든 과정" → /course
    │   ├── "학생 아이디 관리" → /course/students
    │   └── "액세스코드 관리" (미구현)
    │
    └── 테이블 영역
        ├── 과정 정보 헤더 (border-b)
        │   ├── 과정 제목 (좌측, whitespace-nowrap)
        │   ├── flex-1 빈 공간
        │   └── 우측: goal + 뱃지(lessonCount회차, level, wordCount, genre)
        ├── 테이블 헤더: 회차 | 제목 | 레벨(+단어수+장르) | 모듈 | 생성일 | 상태
        ├── 행: round, title, level, wordCount, genre, modules[], createdAt, status
        │   └── 오버레이: ↑ | ↓ | 모듈 수정 | +레슨추가 | 과정에서제외
        ├── 페이지네이션
        └── 상태 범례: 수업진입전(수정 가능) | 수업진행중(수정 불가)
```

**Lesson 데이터 필드:**

| 필드 | 타입 | 설명 |
|------|------|------|
| id | string | 레슨 ID |
| round | number | 회차 (순서 변경 시 자동 재계산) |
| title | string | 레슨 제목 |
| level | string | CEFR 레벨 |
| wordCount | number | 단어 수 (숫자) |
| genre | string | 장르 |
| modules | string[] | 모듈 코드 배열 (예: ['WRD', 'MG', 'QAR']) |
| createdAt | string | 생성일 ('2026. 2. 6.' 형식) |
| status | 'in-use' \| 'not-used' | 수업 진행 상태 |

---

## 3. 정책 (Policy / Business Rules)

### 레슨 수정 제한 정책
- `status === 'in-use'`인 레슨은 **모듈 수정** 버튼과 **과정에서제외** 버튼 비활성화 (`disabled`)
- ↑/↓ 이동 버튼은 in-use 여부와 무관하게 항상 활성화

### 레슨 순서 정책
- 레슨 이동(↑/↓) 후 전체 `round` 번호 재계산 (`map((l, idx) => ({ ...l, round: idx + 1 }))`)
- 레슨 삭제 후 동일하게 `round` 재계산
- 레슨 추가 시 `addLessonSourceLessonId` 위치 **앞에** splice 삽입 후 `round` 재계산

### 레슨 추가 정책
- 텍스트 선택 (Step 1) 없이 "다음" 불가 → toast.error
- 모듈 선택 (Step 2) 없이 "추가" 불가 → 버튼 disabled + title tooltip
- 삽입 위치: 행 오버레이에서 "+레슨추가" 클릭한 레슨의 앞에 삽입
- 추가된 레슨 `wordCount`: 텍스트의 `wordCount` 범위 문자열 → `(min+max)/2` 중간값으로 변환
- 추가된 레슨 `status`: 항상 `'not-used'`

### 모듈 수정 정책
- 모달 내에서 체크박스로 모듈 추가/제거
- 최소 모듈 수 제한 없음 (0개도 허용)
- 취소 시 변경사항 폐기

### 정책 변경 사항 (이전 문서 대비)
- **wordCount 타입 변경**: 문자열 범위('200~300') → 숫자(200) 저장 / 화면 표시는 level 셀에 함께 표시
- **헤더 레이아웃 변경**: 과정 제목 좌측 고정, 목표+뱃지 우측 정렬 (`flex gap-8 items-start`)
- **레슨 표시 레이아웃**: 레벨 셀에 `wordCount + genre` 소문자로 함께 표시

---

## 4. 개발자 추가 작업

### 🔴 필수
- [ ] **레슨 목록 API 연동**: `GET /api/courses/:courseId/lessons` — mockLessons 대체
- [ ] **레슨 추가 API**: `POST /api/courses/:courseId/lessons` — 삽입 위치(round) 포함
- [ ] **레슨 삭제 API**: `DELETE /api/courses/:courseId/lessons/:lessonId`
- [ ] **모듈 수정 API**: `PATCH /api/courses/:courseId/lessons/:lessonId/modules`
- [ ] **레슨 순서 변경 API**: `PATCH /api/courses/:courseId/lessons/reorder` — `{ ids: string[] }`
- [ ] **과정 데이터 조회**: `mockCourses.find(c => c.id === courseId)` → API 연동

### 🟡 권장
- [ ] **텍스트 목록 API 연동**: 레슨 추가 모달 Step 1의 mockTexts → 실제 텍스트 DB 조회
- [ ] **수업진행중 과정 전체 잠금**: in-use 레슨이 있는 과정은 순서 변경도 제한 검토
- [ ] **레슨 추가 후 scroll**: 새로 추가된 레슨 행으로 자동 스크롤

---

## 5. 코딩 규칙 (Coding Rules)

### 공통 컴포넌트
- `CourseSidebar` — searchTitle / onSearchChange / itemCount
- `StudioHeader`, `SimplePagination`
- `AlertDialog` — 레슨 삭제 확인
- `Dialog` — 모듈 수정 모달, 레슨 추가 모달
- `MODULE_CATEGORIES`, `GENRES`, `CEFR_LEVELS`, `WORD_COUNT_RANGES` from `@classsnap/shared`

### 금지 패턴
- `array[index] = value` 직접 mutation → `.map()` 으로 불변 업데이트
- `lesson.round = idx + 1` 직접 mutation → `{ ...lesson, round: idx + 1 }`

### 순서 변경 패턴 (인라인, utils 함수 미사용)
```typescript
const handleMoveLesson = (id: string, direction: 'up' | 'down') => {
  const index = lessons.findIndex(l => l.id === id);
  if ((direction === 'up' && index > 0) || (direction === 'down' && index < lessons.length - 1)) {
    const swapped = [...lessons];
    const swapIndex = direction === 'up' ? index - 1 : index + 1;
    [swapped[index], swapped[swapIndex]] = [swapped[swapIndex], swapped[index]];
    setLessons(swapped.map((l, idx) => ({ ...l, round: idx + 1 })));
  }
};
```

### 레슨 추가 위치 삽입 패턴
```typescript
const targetIndex = lessons.findIndex(l => l.id === addLessonSourceLessonId);
const newLessons = [...lessons];
newLessons.splice(targetIndex, 0, newLesson); // 해당 위치 앞에 삽입
newLessons.forEach((lesson, idx) => { lesson.round = idx + 1; });
setLessons(newLessons);
```

### 모달 리셋 패턴 (레슨 추가)
```typescript
onOpenChange={(open) => {
  setAddLessonModalOpen(open);
  if (!open) {
    setAddLessonModalStep(1);
    setSelectedTextId(null);
    setLessonModulesForAdd([]);
    // 모든 필터 초기화
  }
}}
```

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| # | 이슈 | 심각도 | 상태 |
|---|------|--------|------|
| 1 | mockLessons 8개 하드코딩 (레슨 목록) | High | 미해결 |
| 2 | mockTexts 8개 하드코딩 (레슨 추가 팝업) | High | 미해결 |
| 3 | mockCourses 3개로 과정 정보 조회 (실제 API 없음) | High | 미해결 |
| 4 | 레슨 순서 변경이 state만 변경 — API 미연동 | High | 미해결 |
| 5 | 모듈 수정 모달 배경색이 `bg-gray-900` (다크) — 다른 모달과 스타일 불일치 | Low | 미해결 |
| 6 | `addLessonSourceLessonId`가 null이면 레슨 목록 맨 끝에 추가 (엣지케이스 처리됨) | Low | 확인됨 |
| 7 | wordCount가 number이나 화면에 단위 없이 표시 ('150 뉴스 기사' 형태) | Low | 미해결 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

| 항목 | 경로 | 용도 |
|------|------|------|
| `CourseSidebar` | `@/components/CourseSidebar` | 검색 + 네비게이션 |
| `StudioHeader` | `@/components/oizi/StudioHeader` | 상단 헤더 |
| `SimplePagination` | `@/components/ui/simple-pagination` | 레슨 페이지네이션 |
| `AlertDialog` 계열 | `@/components/ui/alert-dialog` | 레슨 삭제 확인 |
| `Dialog` 계열 | `@/components/ui/dialog` | 모듈 수정 / 레슨 추가 모달 |
| `Badge` | `@/components/ui/badge` | 과정 정보 뱃지 |
| `MODULE_CATEGORIES` 등 | `@classsnap/shared` | 공통 상수 |
| `useRouter`, `useParams` | `next/navigation` | 라우팅, courseId 추출 |
| `toast` | `sonner` | 알림 |

**진입점:**
- `/course` 페이지 "자세히 보기" 버튼
- `/course` 페이지 과정 카드 클릭

**이 페이지가 영향을 주는 기능:**
- CourseSidebar itemCount → 필터된 레슨 수

---

## 8. DB/API 구조 (Data Contract)

### 현재 상태
- 레슨 데이터: `mockLessons` 상수 (8개)
- 텍스트 데이터: `mockTexts` 상수 (8개, 레슨 추가용)
- API 미연동

### Lesson 인터페이스

```typescript
interface Lesson {
  id: string;
  round: number;           // 1부터 시작, 순서 변경 시 재계산
  title: string;
  level: string;           // CEFR 레벨
  wordCount: number;       // 단어 수 (정수)
  genre: string;
  modules: string[];       // 모듈 코드 배열
  createdAt: string;       // 'YYYY. M. D.' 형식
  status: 'in-use' | 'not-used';
}

interface TextItem {
  id: string;
  title: string;
  level: string;
  wordCount: string;       // 범위 문자열 (예: '300~450')
  genre: string;
}
```

### 향후 API 설계 (예정)

```
GET    /api/courses/:courseId                        → 과정 상세 정보
GET    /api/courses/:courseId/lessons                → 레슨 목록 (순서 포함)
POST   /api/courses/:courseId/lessons                → 레슨 추가 { textId, modules[], insertBeforeLessonId? }
PATCH  /api/courses/:courseId/lessons/:lessonId/modules → 모듈 수정 { modules: string[] }
PATCH  /api/courses/:courseId/lessons/reorder        → 순서 변경 { ids: string[] }
DELETE /api/courses/:courseId/lessons/:lessonId      → 레슨 삭제

GET    /api/texts                                    → 텍스트 목록 (레슨 추가 팝업용)
  Query: search, level, genre, wordCount
```

### DB 테이블 설계 (예정)

```sql
CREATE TABLE course_lessons (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id   UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  text_id     UUID REFERENCES texts(id),
  title       VARCHAR(200) NOT NULL,
  level       VARCHAR(10) NOT NULL,
  word_count  INTEGER NOT NULL,
  genre       VARCHAR(50) NOT NULL,
  modules     TEXT[] NOT NULL DEFAULT '{}',
  sort_order  INTEGER NOT NULL DEFAULT 0,
  status      VARCHAR(20) DEFAULT 'not-used',
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```
