# Course 목록 — 상세 구현 문서

**작성일:** 2026-03-23
**대상:** `course/page.tsx` API 연동 완성 및 미구현 기능 처리
**참고:** `docs/course-list-plan-20260323.md`

---

## 현황 점검 결과

코드 전체를 분석한 결과를 정리한다.

### ✅ 이미 완료 (변경 불필요)
- `GET /courses` 목록 조회 → `useCoursesList` 훅으로 연동 완료
- `POST /courses` 생성 → `useCreateCourse` 훅으로 연동 완료
- `DELETE /courses/:id` 삭제 → `useDeleteCourse` 훅으로 연동 완료
- AI 주제 생성 → `useGenerateTopics` 훅으로 연동 완료
- 검색·레벨·장르 필터, 페이지네이션 → API 파라미터 전달 완료
- 로딩·에러·빈 상태 UI → 완료

### ❌ 미구현 항목 (이번 작업 범위)

| # | 항목 | 원인 |
|---|------|------|
| 1 | 삭제 버튼 — `in-use` 비활성화 | 조건 분기 없음 |
| 2 | 카드 그룹 — "배포 중인", "인기 있는" 실제 데이터 | `courses.slice(0,3)` 고정 슬라이스 |
| 3 | 테이블 — `deploymentCodeCount` 항상 0 | DB에 `deployment_count` 컬럼 없음 |
| 4 | 테이블 — `studentCount` 항상 0 | DB에 `student_count` 컬럼 없음 |
| 5 | 레슨 선택 팝오버 — `mockLessons` 30개 고정 | texts 목록 API 없음 |
| 6 | 액세스코드 생성 — toast만 표시 | 백엔드 API 없음 |

---

## STEP 1 — 삭제 버튼 `in-use` 비활성화

**범위:** `apps/web/src/app/course/page.tsx` 1개 파일
**코드 위치:** L821–830 (삭제 Button)

### 변경 내용

```tsx
// 변경 전 (L821)
<Button
  onClick={(e) => {
    e.stopPropagation();
    handleDeleteCourse(course.id);
  }}
  className="h-8 rounded-full bg-red-500 px-3 text-xs font-bold text-white hover:bg-red-600"
>
  <Trash2 className="h-3 w-3" />
  삭제
</Button>

// 변경 후
<Button
  onClick={(e) => {
    e.stopPropagation();
    handleDeleteCourse(course.id);
  }}
  disabled={course.status === 'in-use'}
  title={course.status === 'in-use' ? '수업진행중인 과정은 삭제할 수 없습니다.' : undefined}
  className="h-8 rounded-full bg-red-500 px-3 text-xs font-bold text-white hover:bg-red-600 disabled:opacity-40 disabled:cursor-not-allowed"
>
  <Trash2 className="h-3 w-3" />
  삭제
</Button>
```

---

## STEP 2 — 카드 그룹 실제 필터링

**범위:** `apps/web/src/app/course/page.tsx`

### 현재 문제

```typescript
// L562 — 단순 슬라이스, 기획 의도와 무관
const recentCourses = courses.slice(0, 3);
// → recentCourses[0]: 최근 생성한  (우연히 맞을 수 있음, sort: created_at desc)
// → recentCourses[1]: 배포 중인    (실제로는 그냥 두 번째 항목)
// → recentCourses[2]: 인기 있는    (실제로는 그냥 세 번째 항목)
```

### DB 제약 사항 확인

`courses` 테이블 (`20260316000002_create_courses.sql`) 에는
`student_count`, `deployment_count` 컬럼이 **없다**.

→ "인기 있는 과정"(student_count 기준 정렬)은 STEP 3 DB 마이그레이션 이후에만 구현 가능.

### 변경 계획

**"최근 생성한"** — 기존 `coursesData.data[0]` 사용 가능 (이미 `sort: created_at`, `order: desc`)
**"배포 중인"** — `status: 'in_use'` 조건으로 별도 쿼리 1개 추가
**"인기 있는"** — STEP 3 완료 전까지 카드 숨김 처리

### 추가할 코드 (page.tsx 상단 state 선언부 이후)

```typescript
// "배포 중인" 카드 전용 쿼리 — 기존 useCoursesList 아래에 추가
const { data: activeCoursesData } = useCoursesList({
  limit: 1,
  status: 'in_use',
  sort: 'created_at',
  order: 'desc',
});
const activeCourse = activeCoursesData?.data[0]
  ? mapApiCourseToRow(activeCoursesData.data[0], 0)
  : null;
```

### 제거할 코드

```typescript
// L562 제거
const recentCourses = courses.slice(0, 3);
```

### 카드 렌더링 변경 (L605–681)

```tsx
// 최근 생성한: courses[0] 그대로 사용
{courses[0] && (
  <div className="flex h-44 flex-col justify-between">
    <p className="text-left text-sm font-semibold text-gray-600">최근 생성한</p>
    <div ...onClick={() => handleCardClick(courses[0].id)}>
      ...courses[0].title, courses[0].level, courses[0].genre, courses[0].lessonCount
    </div>
  </div>
)}

// 배포 중인: activeCourse 사용
{activeCourse && (
  <div className="flex h-44 flex-col justify-between">
    <p className="text-left text-sm font-semibold text-gray-600">배포 중인 과정</p>
    <div ...onClick={() => handleCardClick(activeCourse.id)}>
      ...activeCourse.title, activeCourse.level, activeCourse.genre, activeCourse.deploymentCodeCount
    </div>
  </div>
)}

// 인기 있는: STEP 3 완료 전까지 렌더링 제거 (빈 슬롯 유지)
```

---

## STEP 3 — DB 마이그레이션: `student_count` / `deployment_count`

**목적:** STEP 2 "인기 있는 과정" 카드 및 테이블 수강자·배포코드 수 실제 표시

### 3-1. DB 마이그레이션 파일 신규 생성

**파일:** `supabase/migrations/20260323000001_add_course_counts.sql`

```sql
ALTER TABLE courses
  ADD COLUMN IF NOT EXISTS student_count    INTEGER NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS deployment_count INTEGER NOT NULL DEFAULT 0;

CREATE INDEX IF NOT EXISTS idx_courses_student_count    ON courses(student_count DESC);
CREATE INDEX IF NOT EXISTS idx_courses_deployment_count ON courses(deployment_count DESC);
```

### 3-2. Prisma 스키마 반영

**파일:** `apps/api/prisma/schema.prisma` — `courses` 모델에 두 필드 추가

```prisma
model courses {
  // ... 기존 필드 ...
  student_count    Int  @default(0)
  deployment_count Int  @default(0)
}
```

이후 `prisma generate` 실행 (build 스크립트에 이미 포함됨).

### 3-3. Shared 타입 업데이트

**파일:** `packages/shared/src/types/course.ts`

```typescript
export interface Course {
  // ... 기존 필드 ...
  student_count: number;     // 추가
  deployment_count: number;  // 추가
}
```

### 3-4. API 서비스 — sort 옵션 추가

**파일:** `apps/api/src/courses/dto/courses.dto.ts`

```typescript
// CourseQueryDto — sort 필드 옵션 추가
@IsIn(['created_at', 'title', 'student_count'])
sort?: 'created_at' | 'title' | 'student_count';
```

**파일:** `apps/api/src/courses/courses.service.ts`

```typescript
// findAll() — sortField 매핑 (기존 코드와 동일하게 동작, 옵션만 확장됨)
const sortField = query.sort ?? 'created_at';
// student_count로 정렬 시 Prisma가 처리 — 별도 로직 불필요
```

### 3-5. 프론트엔드 `mapApiCourseToRow` 업데이트

**파일:** `apps/web/src/app/course/page.tsx` L305–319

```typescript
function mapApiCourseToRow(c: Course, index: number): CourseRow {
  return {
    id: c.id,
    round: index + 1,
    title: c.title,
    lessonCount: c.total_lessons,
    deploymentCodeCount: c.deployment_count,  // 0 → 실제 값
    studentCount: c.student_count,            // 0 → 실제 값
    status: c.status_code === 'in_use' ? 'in-use' : 'not-used',
    level: c.level_code,
    genre: c.genre_code,
    goal: c.description ?? c.course_focus ?? '',
    wordCount: '',
  };
}
```

타입 파라미터도 `Course` (shared 타입)로 명시.

### 3-6. STEP 2 "인기 있는" 카드 활성화

STEP 3 완료 후 추가:

```typescript
// page.tsx — activeCourse 쿼리 아래에 추가
const { data: popularCoursesData } = useCoursesList({
  limit: 1,
  sort: 'student_count',
  order: 'desc',
});
const popularCourse = popularCoursesData?.data[0]
  ? mapApiCourseToRow(popularCoursesData.data[0], 0)
  : null;
```

카드 렌더링:

```tsx
{popularCourse && (
  <div className="flex h-44 flex-col justify-between">
    <p className="text-left text-sm font-semibold text-gray-600">인기 있는 과정</p>
    <div ...onClick={() => handleCardClick(popularCourse.id)}>
      ...popularCourse.title, popularCourse.level, popularCourse.genre
      <Badge>{popularCourse.studentCount}명</Badge>
    </div>
  </div>
)}
```

---

## STEP 4 — 레슨 선택 팝오버: `mockLessons` → texts API 연동

### 4-1. 현재 상태

`passages.controller.ts` — `GET /passages` (목록) 엔드포인트 **없음**
`api.ts` — `textsApi` 혹은 `passagesApi.list()` **없음**
`use-courses.ts` — texts 목록 훅 **없음**
`react-query.ts` — `queryKeys.texts.lists()` 정의는 있으나 params 미지원

### 4-2. 백엔드 — texts 목록 API 추가

**파일:** `apps/api/src/passages/passages.service.ts` — `findAll()` 메서드 추가

```typescript
async findAll(params: {
  search?: string;
  level?: string;
  genre?: string;
  minWordCount?: number;
  maxWordCount?: number;
  page?: number;
  limit?: number;
}): Promise<{ data: TextRow[]; total: number; totalPages: number }> {
  const page = params.page ?? 1;
  const limit = params.limit ?? 20;
  const offset = (page - 1) * limit;

  const where: Record<string, unknown> = {};
  if (params.search) {
    where.title = { contains: params.search, mode: 'insensitive' };
  }
  if (params.level) {
    where.level_code = params.level;
  }
  if (params.genre) {
    where.genre = params.genre;
  }
  if (params.minWordCount !== undefined || params.maxWordCount !== undefined) {
    where.word_count = {
      ...(params.minWordCount !== undefined ? { gte: params.minWordCount } : {}),
      ...(params.maxWordCount !== undefined ? { lte: params.maxWordCount } : {}),
    };
  }

  const [data, total] = await this.prisma.$transaction([
    this.prisma.texts.findMany({
      where,
      select: { id: true, title: true, level_code: true, genre: true, word_count: true },
      orderBy: { id: 'desc' },
      skip: offset,
      take: limit,
    }),
    this.prisma.texts.count({ where }),
  ]);

  return { data, total, totalPages: Math.ceil(total / limit) };
}
```

> **주의:** `texts` 테이블 컬럼명(`level_code`, `genre`, `word_count`)은 Prisma 스키마 확인 후 실제 컬럼명으로 맞출 것.

**파일:** `apps/api/src/passages/passages.controller.ts` — `GET /passages` 추가

```typescript
@Get()
async findAll(@Query() query: PassageQueryDto) {
  return this.passagesService.findAll({
    search: query.search,
    level: query.level,
    genre: query.genre,
    minWordCount: query.min_word_count,
    maxWordCount: query.max_word_count,
    page: query.page,
    limit: query.limit,
  });
}
```

**파일:** `apps/api/src/passages/dto/passages.dto.ts` — 신규 생성

```typescript
export class PassageQueryDto {
  @IsString() @IsOptional() search?: string;
  @IsString() @IsOptional() level?: string;
  @IsString() @IsOptional() genre?: string;
  @Transform(({ value }) => parseInt(value)) @IsNumber() @IsOptional() min_word_count?: number;
  @Transform(({ value }) => parseInt(value)) @IsNumber() @IsOptional() max_word_count?: number;
  @Transform(({ value }) => parseInt(value)) @IsNumber() @IsOptional() @Min(1) page?: number;
  @Transform(({ value }) => parseInt(value)) @IsNumber() @IsOptional() @Min(1) limit?: number;
}
```

### 4-3. Shared 타입 추가

**파일:** `packages/shared/src/types/models.ts` (또는 적절한 위치) — `TextItem` 타입 추가

```typescript
export interface TextItem {
  id: number;
  title: string;
  level_code: string;
  genre: string;
  word_count: number | null;
}

export interface TextListResponse {
  data: TextItem[];
  total: number;
  totalPages: number;
}

export interface TextQueryParams {
  search?: string;
  level?: string;
  genre?: string;
  min_word_count?: number;
  max_word_count?: number;
  page?: number;
  limit?: number;
}
```

### 4-4. API 클라이언트 추가

**파일:** `apps/web/src/lib/api.ts` — `passagesApi.list()` 추가

```typescript
// ─── Passages ─────────────────────────────────────────────
export const passagesApi = {
  list: (params?: TextQueryParams) =>
    request<TextListResponse>(`/passages${toQueryString({ ...params })}`),

  getAnalysis: (textId: number) =>
    request<PassageAnalysisResponse>(`/passages/${textId}/analysis`),
  // ... 기존 메서드 유지
};
```

`aiApi.analyzePassage`, `aiApi.refreshAnalysis`는 현재 `/passages/:id/analysis` 호출 중이므로
`passagesApi`로 이동하거나 그대로 유지 (동작 변경 없음).

### 4-5. React Query 키 업데이트

**파일:** `apps/web/src/lib/react-query.ts`

```typescript
texts: {
  all: ['texts'] as const,
  lists: (params?: TextQueryParams) => [...queryKeys.texts.all, 'list', params] as const,
  // ...
},
```

### 4-6. 훅 추가

**파일:** `apps/web/src/hooks/use-courses.ts` (또는 별도 `use-texts.ts`)

```typescript
export function useTextsList(params?: TextQueryParams, enabled = true) {
  return useQuery({
    queryKey: queryKeys.texts.lists(params),
    queryFn: () => passagesApi.list(params),
    enabled,
  });
}
```

### 4-7. page.tsx 변경

**제거:** `mockLessons` 상수 (L62–303), `filteredLessons` useMemo (L451–488)

**추가 (state 선언부):**

```typescript
// 레슨 선택 팝오버 열려 있을 때만 쿼리 실행
const isLessonPopoverOpen = lessonSelectOpen.idx !== null;
const { data: textsData, isLoading: textsLoading } = useTextsList(
  {
    search: lessonSelectOpen.searchQuery || undefined,
    level: lessonSelectOpen.filterLevel || undefined,
    genre: lessonSelectOpen.filterGenre || undefined,
    min_word_count: lessonSelectOpen.filterWordRange
      ? parseInt(lessonSelectOpen.filterWordRange.split('~')[0])
      : undefined,
    max_word_count: lessonSelectOpen.filterWordRange
      ? parseInt(lessonSelectOpen.filterWordRange.split('~')[1])
      : undefined,
    limit: 50,
  },
  isLessonPopoverOpen,
);
const availableLessons = textsData?.data ?? [];
```

팝오버 내 `filteredLessons` 참조를 `availableLessons`로 교체.
로딩 상태는 `textsLoading`으로 처리.

---

## STEP 5 — 액세스코드 생성 API 연동

### 5-1. 백엔드 — 라우트 추가

**파일:** `apps/api/src/courses/courses.controller.ts`

```typescript
@Post(':id/access-code')
async generateAccessCode(
  @Param('id', ParseUUIDPipe) id: string,
  @CurrentUser() user: AuthUser,
): Promise<{ code: string }> {
  return this.coursesService.generateAccessCode(id, user);
}
```

**파일:** `apps/api/src/courses/courses.service.ts` — `generateAccessCode()` 추가

```typescript
async generateAccessCode(courseId: string, user: AuthUser): Promise<{ code: string }> {
  await this.verifyOwnership(courseId, user);

  // 6자리 영숫자 랜덤 코드 생성
  const code = Math.random().toString(36).substring(2, 8).toUpperCase();

  // deployment_count 증가 (STEP 3 마이그레이션 완료 후 활성화)
  await this.prisma.courses.update({
    where: { id: courseId },
    data: { deployment_count: { increment: 1 } },
  });

  // TODO: access_codes 테이블이 생기면 코드 저장 로직 추가
  return { code };
}
```

> **현재 범위:** access_codes 전용 테이블 없이 코드만 반환. 추후 별도 테이블 설계 필요.

### 5-2. API 클라이언트 추가

**파일:** `apps/web/src/lib/api.ts` — `coursesApi`에 추가

```typescript
generateAccessCode: (id: string) =>
  request<{ code: string }>(`/courses/${id}/access-code`, { method: 'POST' }),
```

### 5-3. 훅 추가

**파일:** `apps/web/src/hooks/use-courses.ts`

```typescript
export function useGenerateAccessCode() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => coursesApi.generateAccessCode(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.courses.all });
    },
  });
}
```

### 5-4. page.tsx 변경

```typescript
// 훅 추가 (기존 훅 선언부)
const generateAccessCode = useGenerateAccessCode();

// handleGenerateAccessCode 교체
const handleGenerateAccessCode = async (course: CourseRow) => {
  try {
    const { code } = await generateAccessCode.mutateAsync(course.id);
    toast.success(`액세스코드: ${code}`, {
      description: `${course.title}의 액세스코드가 생성되었습니다.`,
      duration: 8000,
    });
  } catch (err) {
    toast.error(err instanceof ApiError ? err.message : '액세스코드 생성에 실패했습니다.');
  }
  setSelectedItem(null);
};
```

---

## 구현 순서 및 의존관계

```
STEP 1 (삭제 버튼 비활성화)
  → 독립 / 즉시 구현 가능

STEP 2 (카드 그룹 — 배포 중인)
  → 독립 / 즉시 구현 가능

STEP 3 (DB 마이그레이션 + student_count/deployment_count)
  → STEP 3 완료 후 → STEP 2 "인기 있는" 카드 활성화
  → STEP 3 완료 후 → STEP 5 generateAccessCode의 deployment_count 증가 로직 활성화

STEP 4 (texts API 연동)
  → DB 컬럼명(texts 테이블) 사전 확인 필요
  → STEP 4 완료 후 → mockLessons 완전 제거 가능

STEP 5 (액세스코드 생성)
  → 독립 / 즉시 구현 가능 (STEP 3과 무관하게 코드만 반환하는 버전으로 먼저 구현)
```

### 권장 구현 순서

| 순서 | STEP | 변경 파일 수 | 선행 조건 |
|------|------|------------|----------|
| 1 | STEP 1 — 삭제 버튼 비활성화 | 1 | 없음 |
| 2 | STEP 2 — 카드 배포 중인 | 1 | 없음 |
| 3 | STEP 5 — 액세스코드 API | 3 (api, hook, page) + 백엔드 2 | 없음 |
| 4 | STEP 3 — DB 마이그레이션 + 카운트 | 4 (migration, schema, shared, page) | DB 반영 필요 |
| 5 | STEP 4 — texts API 연동 | 백엔드 3 + 프론트 4 | texts 테이블 컬럼명 확인 |

---

## STEP 4 전 확인 필요 사항

STEP 4 구현 전에 아래 사항을 먼저 확인해야 한다.

1. **`texts` 테이블 실제 컬럼명**
   - `supabase/migrations/20250710035319_initial_schema.sql` 에서 `texts` 테이블 확인
   - `level_code`인지 `level`인지, `genre`인지 `genre_code`인지, `word_count`인지 확인

2. **Prisma 스키마에서 `texts` 모델 필드 확인**
   - `apps/api/prisma/schema.prisma` 의 `texts` 모델
