# Course 목록 페이지 — 구현 플랜

**작성일:** 2026-03-23
**대상 파일:** `apps/web/src/app/course/page.tsx`
**참고 기획 문서:** `docs/course_modi.md`, `docs/course-20260316.md`

---

## 현재 상태 요약

### 이미 완료된 것 (건드리지 않음)

| 항목 | 파일 | 상태 |
|------|------|------|
| Backend API (CRUD) | `apps/api/src/courses/` | ✅ 완료 |
| React Query 훅 | `apps/web/src/hooks/use-courses.ts` | ✅ 완료 |
| 과정 목록 API 연동 | `course/page.tsx` L328–344 | ✅ 완료 |
| 과정 생성 API 연동 | `useCreateCourse` hook | ✅ 완료 |
| 과정 삭제 API 연동 | `useDeleteCourse` hook | ✅ 완료 |
| AI 주제 생성 연동 | `useGenerateTopics` hook | ✅ 완료 |
| Shared 타입 정의 | `packages/shared/src/types/course.ts` | ✅ 완료 |

### 아직 남은 문제

#### 🔴 High — 데이터 불일치

1. **`deploymentCodeCount`, `studentCount` 항상 0**
   - `mapApiCourseToRow()` (L305–319)에서 두 필드를 하드코딩 `0`으로 매핑 중
   - DB `courses` 테이블에 해당 컬럼 없거나 API 응답에 포함 안 됨

2. **레슨 선택 팝오버가 `mockLessons` 사용** (L62–303)
   - 30개 하드코딩 레슨 → 실제 texts API 연동 필요

#### 🟡 Medium — UI/UX 미구현

3. **카드 그룹 필터링 미구현**
   - "최근 생성한", "배포 중인", "인기 있는" 카드가 단순히 `courses[0]`, `courses[1]`, `courses[2]`
   - 각각 정렬 기준으로 API 별도 호출 또는 클라이언트 필터링 필요

4. **`in-use` 과정 삭제 버튼 비활성화 미적용**
   - 기획 정책: 수업진행중(`in-use`) 과정은 삭제 불가
   - 현재 모든 과정에 삭제 버튼 노출

5. **액세스코드 생성 API 미연동**
   - `handleGenerateAccessCode` 함수가 `toast.success`만 표시

---

## 구현 범위 (이번 작업)

기획 문서 기준 **🔴 필수** 항목 중 아직 미완성인 것만 구현한다.
(이미 완료된 목록·생성·삭제 API 연동은 제외)

---

## 구현 플랜

### STEP 1 — `deploymentCodeCount` / `studentCount` 필드 확보

**문제:** `mapApiCourseToRow()`에서 두 필드가 `0`으로 고정.

**해결 방향 A (권장):** API 응답에 필드 추가
- `apps/api/src/courses/courses.service.ts` `findAll()`에서
  `courses` 테이블에 `student_count`, `deployment_count` 컬럼이 있으면 그대로 반환
- `packages/shared/src/types/course.ts` `Course` 인터페이스에 두 필드 추가
- `mapApiCourseToRow()`에서 실제 값 매핑

**해결 방향 B:** DB 컬럼 확인 후 마이그레이션
- `supabase/migrations/`에 `courses` 테이블 컬럼이 없으면 추가
  ```sql
  ALTER TABLE courses ADD COLUMN student_count INTEGER DEFAULT 0;
  ALTER TABLE courses ADD COLUMN deployment_count INTEGER DEFAULT 0;
  ```
- 방향 A와 함께 적용

**확인 필요:** `supabase/migrations/` 내 `courses` 테이블 스키마

---

### STEP 2 — 레슨 선택 팝오버: `mockLessons` → texts API 연동

**현재 코드:**
```typescript
// course/page.tsx L452
const filteredLessons = useMemo(() => {
  return mockLessons.filter(...)  // 하드코딩 30개
}, [lessonSelectOpen, wordCount]);
```

**변경 계획:**

1. `apps/web/src/hooks/use-courses.ts`에 `useTextsList` 훅 추가
   (또는 별도 `use-texts.ts` 파일)
   ```typescript
   export function useTextsList(params?: TextQueryParams) {
     return useQuery({
       queryKey: queryKeys.texts.lists(params),
       queryFn: () => textsApi.list(params),
       enabled: !!params,
     });
   }
   ```

2. `course/page.tsx`에서 팝오버 열릴 때 쿼리 파라미터 전달
   - `lessonSelectOpen.idx !== null` 일 때만 fetch (`enabled` 조건)
   - 필터값(`searchQuery`, `filterLevel`, `filterGenre`, `filterWordRange`) → API 쿼리 파라미터

3. `mockLessons` 상수 및 `filteredLessons` useMemo 제거, API 데이터로 교체

4. 로딩·에러 상태 처리 (팝오버 내 스피너)

**전제 조건:** `apps/api/src/passages/` 또는 `texts/` API가 목록 쿼리를 지원해야 함
→ 기존 `passages.service.ts` 확인 필요

---

### STEP 3 — 카드 그룹 실제 필터링

**현재 코드 (추정):**
```typescript
// 단순 slice — 실제 기준 없음
const recentCourse = courses[0];
const activeCourse = courses[1];   // 배포 중인
const popularCourse = courses[2];  // 인기 있는
```

**변경 계획:**

카드별로 별도 `useCoursesList` 쿼리 3개 호출:

```typescript
// 최근 생성한: 생성일 내림차순 1개
const { data: recentData } = useCoursesList({ limit: 1, sort: 'created_at', order: 'desc' });

// 배포 중인: status=in_use, 생성일 내림차순 1개
const { data: activeData } = useCoursesList({ limit: 1, status: 'in_use', sort: 'created_at', order: 'desc' });

// 인기 있는: student_count 내림차순 1개 (API sort 옵션 추가 필요)
const { data: popularData } = useCoursesList({ limit: 1, sort: 'student_count', order: 'desc' });
```

**추가 필요:** `CourseQueryDto`에 `sort: 'student_count'` 옵션 추가
(현재 `'created_at' | 'title'`만 허용)

---

### STEP 4 — `in-use` 과정 삭제 버튼 비활성화

**기획 정책:** 수업진행중(`status === 'in-use'`) 과정은 삭제 불가

**변경 위치:** `course/page.tsx` 행 오버레이 삭제 버튼

```tsx
// 변경 전
<button onClick={() => { setCourseToDelete(course.id); setDeleteAlertOpen(true); }}>
  삭제
</button>

// 변경 후
<button
  onClick={() => { setCourseToDelete(course.id); setDeleteAlertOpen(true); }}
  disabled={course.status === 'in-use'}
  title={course.status === 'in-use' ? '수업진행중인 과정은 삭제할 수 없습니다' : undefined}
>
  삭제
</button>
```

---

### STEP 5 — 액세스코드 생성 API 연동 (필수 범위 내)

**현재 코드:**
```typescript
const handleGenerateAccessCode = () => {
  toast.success('액세스코드가 생성되었습니다.');  // mock
};
```

**API 설계 (기획 문서 기준):**
```
POST /api/courses/:id/access-code
```

**변경 계획:**
1. `apps/api/src/courses/courses.controller.ts`에 라우트 추가
2. `apps/web/src/lib/api.ts`에 `coursesApi.generateAccessCode(id)` 추가
3. `use-courses.ts`에 `useGenerateAccessCode` mutation 훅 추가
4. `course/page.tsx`의 `handleGenerateAccessCode`에서 mutation 호출

---

## 작업 순서 및 우선순위

| 순서 | STEP | 우선순위 | 예상 작업 범위 |
|------|------|----------|--------------|
| 1 | STEP 4 (삭제 버튼 비활성화) | 🔴 즉시 | 프론트 1줄 수정 |
| 2 | STEP 1 (student/deployment count) | 🔴 높음 | DB 확인 → shared 타입 → API → 프론트 |
| 3 | STEP 3 (카드 그룹 필터링) | 🟡 중간 | 프론트 훅 3개 분리 + API sort 옵션 |
| 4 | STEP 2 (레슨 선택 texts API) | 🟡 중간 | API 확인 → 훅 추가 → 프론트 교체 |
| 5 | STEP 5 (액세스코드 API) | 🟡 중간 | API 신규 + 프론트 연동 |

---

## 확인 필요 사항 (구현 전 선행 확인)

1. `supabase/migrations/`에서 `courses` 테이블에 `student_count`, `deployment_count` 컬럼 존재 여부
2. `apps/api/src/passages/passages.service.ts`에서 texts 목록 쿼리(검색/필터) 지원 여부
3. `apps/web/src/lib/api.ts`에 `textsApi` 또는 `passagesApi` 존재 여부
