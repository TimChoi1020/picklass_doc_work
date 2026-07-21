# Course Detail 페이지 — Mock → API 연동 개발 플랜

**작성일**: 2026-03-24
**대상 파일**: `apps/web/src/app/course/[courseId]/page.tsx`
**URL 패턴**: `/course/{courseId}` (예: `/course/473b045f-3857-4264-ad5c-1db46ded05b6`)

---

## 1. 현재 상태 분석

### 1.1 프론트엔드 (Mock 상태)

| 항목 | 현재 상태 | 비고 |
|------|----------|------|
| 과정 정보 | `mockCourses[0]` 폴백 | courseId로 찾되, 없으면 첫 번째 항목 사용 |
| 레슨 목록 | `mockLessons` (L001~L008, 8개) | `useState(mockLessons)`로 로컬 상태 관리 |
| 레슨 추가용 지문 | `mockTexts` (T001~T008, 8개) | 모달 Step 1에서 사용 |
| 레슨 삭제 | 로컬 state에서 filter | API 호출 없음 |
| 레슨 순서 변경 | 로컬 state에서 swap | API 호출 없음 |
| 모듈 수정 | 로컬 state에서 map | API 호출 없음 |
| 레슨 추가 | 로컬 state에 push | API 호출 없음 |

### 1.2 백엔드 (이미 구현됨)

| 레이어 | 파일 | 상태 |
|--------|------|------|
| Controller | `apps/api/src/courses/courses.controller.ts` | ✅ 완료 |
| Controller | `apps/api/src/courses/lessons.controller.ts` | ✅ 완료 |
| Service | `apps/api/src/courses/courses.service.ts` | ✅ 완료 |
| Service | `apps/api/src/courses/lessons.service.ts` | ✅ 완료 |
| DTO | `apps/api/src/courses/dto/courses.dto.ts` | ✅ 완료 |
| DTO | `apps/api/src/courses/dto/lessons.dto.ts` | ✅ 완료 |
| Prisma | `apps/api/prisma/schema.prisma` (courses, course_lessons) | ✅ 완료 |

### 1.3 프론트엔드 인프라 (이미 구현됨)

| 레이어 | 파일 | 상태 |
|--------|------|------|
| React Query 훅 | `apps/web/src/hooks/use-courses.ts` | ✅ 완료 |
| API 클라이언트 | `apps/web/src/lib/api.ts` (coursesApi, lessonsApi) | ✅ 완료 |
| Query Keys | `apps/web/src/lib/react-query.ts` | ✅ 완료 |
| 공유 타입 | `packages/shared/src/types/course.ts` | ✅ 완료 |

---

## 2. GAP 분석 (해야 할 일)

프론트엔드 페이지에서 이미 준비된 훅/API를 호출하기만 하면 됩니다.

### 2.1 제거할 Mock 데이터

```
Line 29~119:  mockLessons (8개 레슨, id: L001~L008)
Line 122~131: mockTexts (8개 지문, id: T001~T008)
Line 134~171: mockCourses (3개 과정, id: C001~C003)
```

### 2.2 교체할 데이터 소스

| Mock 데이터 | 교체 대상 훅 | API 엔드포인트 |
|------------|------------|--------------|
| `mockCourses.find(c => c.id === courseId)` | `useCourse(courseId)` | `GET /courses/:id` |
| `useState(mockLessons)` | `useCourseLessons(courseId)` | `GET /courses/:courseId/lessons` |
| `mockTexts` (레슨 추가 모달) | `useTextsList(params)` | `GET /passages` |

### 2.3 교체할 로컬 조작 → API Mutation

| UI 액션 | 현재 (로컬 state) | 교체 (API mutation) |
|---------|------------------|-------------------|
| 레슨 삭제 | `setLessons(lessons.filter(...))` | `useDeleteLesson(courseId).mutate(lessonId)` |
| 레슨 순서변경 | `setLessons(swapped)` | `lessonsApi.reorder(courseId, orderedIds)` |
| 모듈 수정 | `setLessons(lessons.map(...))` | `useUpdateLesson(courseId).mutate({ lessonId, data: { skill_modules } })` |
| 레슨 추가 | 모달에서 로컬 추가 | `useCreateLesson(courseId).mutate(payload)` |

---

## 3. 데이터 매핑

### 3.1 Course 객체 필드 매핑

| Mock 필드 | API 필드 (`Course` 타입) | 표시 위치 |
|-----------|------------------------|----------|
| `title` | `title` | 헤더 좌측 |
| `goal` | `description` 또는 `course_focus` | 헤더 우측 |
| `lessonCount` | `total_lessons` | 뱃지 `{n}회차` |
| `level` | `level_code` | 뱃지 |
| `wordCount` | (course_lessons에서 집계 또는 별도 필드) | 뱃지 |
| `genre` | `genre_code` | 뱃지 |

> **주의**: Mock의 `goal` 필드는 API의 `course_focus`에 매핑.
> Mock의 `wordCount`(예: '100~200')는 API Course 타입에 직접 없음 → `course_focus` 또는 별도 처리 필요.

### 3.2 Lesson 객체 필드 매핑

| Mock 필드 | API 필드 (`CourseLesson` 타입) | 비고 |
|-----------|------------------------------|------|
| `id` (L001) | `id` (UUID) | 타입 변경: string → UUID |
| `round` | `lesson_order` | 이름만 다름 |
| `title` | `topic` | mock에서는 title, API에서는 topic |
| `level` | 없음 (text 조인 필요) | ⚠️ text_id가 있으면 texts.level 참조 |
| `wordCount` | 없음 (text 조인 필요) | ⚠️ text_id가 있으면 texts.word_count 참조 |
| `genre` | 없음 (text 조인 필요) | ⚠️ text_id가 있으면 texts.category 참조 |
| `modules` | `skill_modules` (JSON) | 배열 형태 동일 |
| `createdAt` | `created_at` (ISO 8601) | 포맷팅 필요 |
| `status` ('in-use'/'not-used') | `status_code` ('draft'/'ready'/'in_progress'/'completed') | ⚠️ 값 체계 다름 |

### 3.3 상태 코드 매핑

| UI 표시 | Mock 값 | API 값 (`LessonStatusCode`) | 매핑 로직 |
|---------|---------|---------------------------|----------|
| 수업진행중 (빨간색) | `'in-use'` | `'in_progress'` 또는 `'completed'` | `status_code !== 'draft' && status_code !== 'ready'` |
| 수업진입전 (회색) | `'not-used'` | `'draft'` 또는 `'ready'` | `status_code === 'draft' \|\| status_code === 'ready'` |

---

## 4. 백엔드 보완 필요 사항

### 4.1 Lessons API에 Text 정보 포함 (JOIN)

**현재 문제**: `GET /courses/:courseId/lessons` 응답에 `text_id`만 있고 texts의 `title`, `level`, `category`, `word_count` 정보가 없음.

**해결 방안 A (권장)**: LessonsService.findAllByCourse()에서 texts 테이블 JOIN

```
// lessons.service.ts 수정 필요
findAllByCourse(courseId) {
  prisma.course_lessons.findMany({
    where: { course_id: courseId },
    orderBy: { lesson_order: 'asc' },
    include: { text: { select: { id, title, level, category, word_count } } }  // ← 추가
  });
}
```

**해결 방안 B**: 프론트엔드에서 레슨 목록 + 텍스트 목록 별도 조회 후 클라이언트 조인

**결정**: 방안 A 권장 (네트워크 1회, 데이터 일관성)

### 4.2 공유 타입 확장

`CourseLesson` 타입에 text 정보를 포함하는 확장 타입 필요:

```typescript
// packages/shared/src/types/course.ts에 추가
export interface CourseLessonWithText extends CourseLesson {
  text?: {
    id: number;
    title: string;
    level: string | null;
    category: string | null;
    word_count: number | null;
  } | null;
}
```

---

## 5. 구현 작업 목록

### Phase 1: 백엔드 보완 (선행 작업)

| # | 작업 | 파일 | 상세 |
|---|------|------|------|
| 1-1 | Lessons API에 text JOIN 추가 | `apps/api/src/courses/lessons.service.ts` | `findAllByCourse()`에 `include: { text: true }` 추가 |
| 1-2 | 공유 타입 확장 | `packages/shared/src/types/course.ts` | `CourseLessonWithText` 인터페이스 추가 |
| 1-3 | React Query 훅 반환 타입 업데이트 | `apps/web/src/hooks/use-courses.ts` | `useCourseLessons` 반환 타입 수정 |

### Phase 2: 과정 정보 연동

| # | 작업 | 상세 |
|---|------|------|
| 2-1 | `mockCourses` 제거 | 전체 삭제 (Line 134~171) |
| 2-2 | `useCourse(courseId)` 호출 | 페이지 상단에 훅 추가 |
| 2-3 | 로딩/에러 상태 처리 | 과정 로딩 중 스켈레톤, 없으면 404 처리 |
| 2-4 | 헤더 영역 필드 매핑 | `course.title`, `course.course_focus`, `course.level_code`, `course.genre_code`, `course.total_lessons` |

### Phase 3: 레슨 목록 연동

| # | 작업 | 상세 |
|---|------|------|
| 3-1 | `mockLessons` 제거 | 전체 삭제 (Line 29~119) |
| 3-2 | `useCourseLessons(courseId)` 호출 | `useState(mockLessons)` 제거, 서버 데이터 사용 |
| 3-3 | 필드 매핑 적용 | `lesson_order` → 회차, `topic` → 제목, `text.level` → 레벨, `skill_modules` → 모듈, `status_code` → 상태 |
| 3-4 | 날짜 포맷팅 | `created_at` (ISO) → `YYYY. M. D.` 포맷 |
| 3-5 | 상태 배지 매핑 | `draft`/`ready` → 수업진입전, `in_progress`/`completed` → 수업진행중 |
| 3-6 | 로딩/빈 상태 UI | 레슨 로딩 중 스켈레톤, 0개일 때 빈 상태 메시지 |

### Phase 4: CRUD 액션 연동

| # | 작업 | Mock 코드 | API 코드 |
|---|------|----------|---------|
| 4-1 | 레슨 삭제 | `setLessons(lessons.filter(...))` | `useDeleteLesson(courseId).mutateAsync(lessonId)` |
| 4-2 | 레슨 순서 변경 (위/아래) | `setLessons(swapped.map(...))` | swap 후 `lessonsApi.reorder(courseId, newOrderIds)` |
| 4-3 | 모듈 수정 | `setLessons(lessons.map(...))` | `useUpdateLesson(courseId).mutateAsync({ lessonId, data: { skill_modules } })` |
| 4-4 | 레슨 추가 | 로컬 push | `useCreateLesson(courseId).mutateAsync(payload)` |

**각 mutation 성공 시**: React Query `invalidateQueries`로 레슨 목록 자동 리프레시 (이미 훅에 구현됨)

### Phase 5: 레슨 추가 모달 연동

| # | 작업 | 상세 |
|---|------|------|
| 5-1 | `mockTexts` 제거 | 전체 삭제 (Line 122~131) |
| 5-2 | `useTextsList(params)` 호출 | 모달 Step 1에서 필터 파라미터와 함께 사용 |
| 5-3 | 필터 연동 | `searchLessonTitle` → `search`, `filterLessonLevel` → `level`, `filterLessonGenre` → `category` |
| 5-4 | 단어수 필터 매핑 | `filterLessonWordCount` (예: '200~300') → `min_word_count: 200, max_word_count: 300` |
| 5-5 | Step 2 → API 호출 | 모듈 선택 후 `useCreateLesson` mutation 호출 |

### Phase 6: 에러 처리 + UX 개선

| # | 작업 | 상세 |
|---|------|------|
| 6-1 | API 에러 → toast | `onError` 콜백에서 `toast.error(error.message)` |
| 6-2 | Mutation 로딩 상태 | 버튼에 `isPending` 상태 반영 (disabled + 로딩 스피너) |
| 6-3 | Optimistic Update (선택) | 순서 변경 시 즉각 UI 반영 후 서버 동기화 |
| 6-4 | 삭제 시 in-use 체크 | `status_code`가 'in_progress'이면 삭제 버튼 비활성화 (이미 UI에 있음) |

---

## 6. 파일별 변경 요약

| 파일 | 변경 유형 | 변경량 |
|------|----------|-------|
| `apps/api/src/courses/lessons.service.ts` | 수정 | `findAllByCourse` 1곳 (include 추가) |
| `packages/shared/src/types/course.ts` | 수정 | `CourseLessonWithText` 타입 추가 (~10줄) |
| `apps/web/src/hooks/use-courses.ts` | 수정 | `useCourseLessons` 반환 타입 변경 |
| `apps/web/src/app/course/[courseId]/page.tsx` | **대폭 수정** | Mock 삭제 (~143줄), 훅 연동, 필드 매핑, 에러/로딩 처리 |

---

## 7. 의존성 및 선행 조건

```
[필수] NestJS 백엔드 서버 실행 (localhost:3001)
  └── Prisma 마이그레이션 완료 (courses, course_lessons 테이블 존재)
      └── .env에 DATABASE_URL 설정

[필수] apps/web/.env.local에 NEXT_PUBLIC_API_URL=http://localhost:3001 설정

[필수] 테스트용 데이터
  └── courses 테이블에 1개 이상의 과정 (UUID)
  └── course_lessons 테이블에 해당 과정의 레슨들
  └── texts 테이블에 연결 가능한 지문들
```

---

## 8. 실행 순서

```
1. 환경 확인
   ├── NestJS 서버 실행 가능 여부
   ├── DB 마이그레이션 상태
   └── 테스트 데이터 존재 여부

2. Phase 1: 백엔드 보완 (lessons에 text JOIN)
   ↓
3. Phase 2: 과정 정보 연동 (useCourse)
   ↓
4. Phase 3: 레슨 목록 연동 (useCourseLessons)
   ↓
5. Phase 4: CRUD 액션 연동 (delete, reorder, update, create)
   ↓
6. Phase 5: 레슨 추가 모달 연동 (useTextsList)
   ↓
7. Phase 6: 에러 처리 + UX 개선
   ↓
8. 통합 테스트
   ├── 과정 상세 조회 (UUID로 접근)
   ├── 레슨 목록 표시 (순서, 필드 매핑)
   ├── 레슨 삭제 → 목록 갱신
   ├── 레슨 순서 변경 → 서버 반영
   ├── 모듈 수정 → 서버 반영
   ├── 레슨 추가 (기존 지문 선택) → 서버 반영
   ├── 에러 케이스 (네트워크 오류, 404)
   └── 빈 상태 (레슨 0개 과정)
```

---

## 9. 리스크

| 리스크 | 영향 | 대응 |
|--------|------|------|
| NestJS 서버 미실행 | 모든 API 호출 실패 | `pnpm dev:api` 실행 필요, 환경변수 확인 |
| DB 마이그레이션 미완료 | courses/course_lessons 테이블 없음 | `npx prisma db push` 또는 `npx prisma migrate dev` |
| 테스트 데이터 없음 | 빈 화면만 표시 | seed 스크립트 또는 course/page.tsx에서 먼저 과정 생성 |
| text JOIN 시 N+1 쿼리 | 대량 레슨에서 성능 저하 | Prisma `include`는 JOIN이므로 1회 쿼리 (문제 없음) |
| `wordCount` 필드 누락 | 뱃지 표시 불가 | course_focus에 포함하거나 별도 computed 필드 |
