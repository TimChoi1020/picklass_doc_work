# NestJS API: Supabase → Prisma 전환 플랜

## Context

현재 NestJS API의 모든 서비스가 `SupabaseService.getClient()`를 통해 Supabase JS SDK(PostgREST)로 DB에 접근하고 있다.
신규 API는 Prisma ORM으로 PostgreSQL에 직접 연결해야 한다. **기존 API(프론트엔드 직접 호출)는 기존 Supabase 방식을 유지**한다.

---

## 영향 범위 분석

### Supabase를 사용하는 NestJS 서비스 전체 목록

| 모듈 | 서비스 파일 | Supabase 테이블 | 비고 |
|------|------------|-----------------|------|
| Auth | `src/auth/auth.service.ts` | `users`, `institutions` | 사용자 프로필 조회 |
| Auth Guard | `src/common/guards/auth.guard.ts` | - | `supabase.auth.getUser()` 토큰 검증 |
| Courses | `src/courses/courses.service.ts` | `courses` | CRUD + soft delete |
| Lessons | `src/courses/lessons.service.ts` | `course_lessons` | CRUD + reorder |
| Classes | `src/classes/classes.service.ts` | `classes` | CRUD + soft delete |
| ClassStudents | `src/classes/class-students.service.ts` | `class_students` | 등록/배치/상태관리 |
| Reports | `src/reports/reports.service.ts` | `student_learning_records` | 조회 + CSV export |
| Passages | `src/passages/passages.service.ts` | `texts` | analysis 캐시 조회 |
| AI | `src/ai/ai.service.ts` | `texts` | 분석 결과 저장 |
| CommonCodes | `src/common-codes/common-codes.service.ts` | `code_groups`, `code_items` | 공통코드 조회 |
| Health | `src/health/health.controller.ts` | `texts` | DB 헬스체크 |

---

## 수정 플랜

### Step 1: Prisma 설치 및 초기 설정

**새 파일:**
- `apps/api/prisma/schema.prisma` — Prisma 스키마 정의

**수정 파일:**
- `apps/api/package.json` — `@prisma/client`, `prisma` 의존성 추가
- `apps/api/.env` — `DATABASE_URL` 추가 (Supabase PostgreSQL 직접 연결 문자열)

**작업 내용:**
1. `pnpm add @prisma/client` + `pnpm add -D prisma` (apps/api)
2. `npx prisma init` 으로 초기 구조 생성
3. Supabase PostgreSQL 연결 문자열을 `DATABASE_URL`로 설정
4. `npx prisma db pull`로 기존 스키마 introspection → `schema.prisma` 생성

### Step 2: Prisma 스키마 정의 (`schema.prisma`)

기존 마이그레이션 기반으로 모델 정의:

```
- User, Institution (기존 테이블)
- Course, CourseLesson (과정/레슨)
- Class, ClassStudent (클래스/학생)
- StudentLearningRecord (학습 기록)
- Text (지문)
- CodeGroup, CodeItem (공통코드)
```

> **주의:** `npx prisma db pull`로 기존 스키마를 가져온 뒤 모델명/관계를 정리하는 방식 권장.
> 새 마이그레이션은 Prisma로 관리하지 않고, 기존 `supabase/migrations/` 방식 유지 가능.

### Step 3: PrismaModule 생성

**새 파일:**
- `apps/api/src/prisma/prisma.module.ts`
- `apps/api/src/prisma/prisma.service.ts`

**작업 내용:**
- NestJS 글로벌 모듈로 `PrismaService` 제공
- `onModuleInit`에서 `$connect()`, `onModuleDestroy`에서 `$disconnect()`
- `SupabaseModule`과 공존 (기존 코드는 Supabase 유지)

### Step 4: 서비스별 Prisma 전환

각 서비스에서 `this.supabase.getClient().from(...)` → `this.prisma.tableName.findMany(...)` 등으로 변경.

#### 4-1. CoursesService (`src/courses/courses.service.ts`)
- `findAll()`: `.from('courses').select(...)` → `prisma.course.findMany({ where, orderBy, skip, take })`
- `findOne()`: → `prisma.course.findUnique()`
- `create()`: → `prisma.course.create()` + `prisma.courseLesson.createMany()`
- `update()`: → `prisma.course.update()`
- `remove()`: → `prisma.course.update({ deleted_at })`

#### 4-2. LessonsService (`src/courses/lessons.service.ts`)
- `findAllByCourse()`: → `prisma.courseLesson.findMany({ where: { course_id } })`
- `findOne()`: → `prisma.courseLesson.findFirst()`
- `create()`: → `prisma.courseLesson.create()` + course total_lessons 동기화
- `update()`: → `prisma.courseLesson.update()`
- `remove()`: → `prisma.courseLesson.delete()`
- `reorder()`: → `prisma.$transaction()` 으로 배치 업데이트

#### 4-3. ClassesService (`src/classes/classes.service.ts`)
- `findAll()`: → `prisma.class.findMany()`
- `findOne()`: → `prisma.class.findUnique()`
- `create()`: → `prisma.class.create()`
- `update()`: → `prisma.class.update()`
- `remove()`: → `prisma.class.update({ deleted_at })`

#### 4-4. ClassStudentsService (`src/classes/class-students.service.ts`)
- `findAllByClass()`: → `prisma.classStudent.findMany()`
- `addStudent()`: → `prisma.classStudent.create()` (중복 체크)
- `addStudentsBatch()`: → `prisma.classStudent.createMany({ skipDuplicates: true })`
- `removeStudent()`: → `prisma.classStudent.delete()`
- `updateStudentStatus()`: → `prisma.classStudent.update()`

#### 4-5. ReportsService (`src/reports/reports.service.ts`)
- `getStudentReports()`: → `prisma.studentLearningRecord.findMany()` + JS 집계
- `getStudentDetail()`: → `prisma.studentLearningRecord.findMany()`
- `exportReport()`: 동일 로직, 데이터 소스만 Prisma로 변경

#### 4-6. PassagesService (`src/passages/passages.service.ts`)
- `getAnalysis()`: → `prisma.text.findUnique({ select: { analysis } })`
- `getBatchAnalysis()`: → `prisma.text.findMany({ where: { id: { in: ids } } })`

#### 4-7. AiService (`src/ai/ai.service.ts`)
- 텍스트 조회: → `prisma.text.findUnique()`
- 분석 결과 저장: → `prisma.text.update({ data: { analysis, analyzer_version, analyzed_at } })`

#### 4-8. AuthService (`src/auth/auth.service.ts`)
- 사용자 프로필 조회: → `prisma.user.findUnique({ include: { institution: true } })`

#### 4-9. AuthGuard (`src/common/guards/auth.guard.ts`)
- `supabase.auth.getUser(token)` → **Supabase Auth는 유지** (인증은 Supabase Auth 서비스이므로 DB 접근과 무관)

#### 4-10. CommonCodesService (`src/common-codes/common-codes.service.ts`)
- `findAllGroups()`: → `prisma.codeGroup.findMany()`
- `findGroupByCode()`: → `prisma.codeGroup.findUnique({ include: { items } })`
- `findItemsByGroupCode()`: → `prisma.codeItem.findMany()`

#### 4-11. HealthController (`src/health/health.controller.ts`)
- DB 체크: → `prisma.$queryRaw` 또는 `prisma.text.findFirst()`

### Step 5: AppModule 업데이트

**수정 파일:** `src/app.module.ts`
- `PrismaModule` import 추가
- `SupabaseModule`은 유지 (AuthGuard에서 계속 사용)

### Step 6: env.validation 업데이트

**수정 파일:** `src/config/env.validation.ts`
- `DATABASE_URL` 필수 환경변수 추가
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`는 유지 (AuthGuard용)

---

## 수정 파일 요약

| 파일 | 변경 내용 |
|------|----------|
| `apps/api/package.json` | prisma, @prisma/client 추가 |
| `apps/api/.env` | DATABASE_URL 추가 |
| `apps/api/prisma/schema.prisma` | **신규** - DB 스키마 정의 |
| `apps/api/src/prisma/prisma.module.ts` | **신규** - Prisma 글로벌 모듈 |
| `apps/api/src/prisma/prisma.service.ts` | **신규** - PrismaClient 래퍼 |
| `apps/api/src/app.module.ts` | PrismaModule import 추가 |
| `apps/api/src/config/env.validation.ts` | DATABASE_URL 검증 추가 |
| `apps/api/src/courses/courses.service.ts` | Supabase → Prisma |
| `apps/api/src/courses/lessons.service.ts` | Supabase → Prisma |
| `apps/api/src/classes/classes.service.ts` | Supabase → Prisma |
| `apps/api/src/classes/class-students.service.ts` | Supabase → Prisma |
| `apps/api/src/reports/reports.service.ts` | Supabase → Prisma |
| `apps/api/src/passages/passages.service.ts` | Supabase → Prisma |
| `apps/api/src/ai/ai.service.ts` | Supabase → Prisma |
| `apps/api/src/auth/auth.service.ts` | Supabase → Prisma |
| `apps/api/src/common-codes/common-codes.service.ts` | Supabase → Prisma |
| `apps/api/src/health/health.controller.ts` | Supabase → Prisma |

**변경하지 않는 파일:**
- `apps/api/src/common/guards/auth.guard.ts` — Supabase Auth 유지
- `apps/api/src/supabase/` — AuthGuard용으로 유지
- `apps/web/` — 프론트엔드는 기존 Supabase 방식 유지
- `supabase/migrations/` — 기존 마이그레이션 유지

---

## 검증 방법

1. `npx prisma db pull` → `schema.prisma` 생성 확인
2. `npx prisma generate` → Prisma Client 생성 확인
3. `pnpm run build` (apps/api) → 타입 에러 없이 빌드 성공
4. `pnpm dev` → API 서버 정상 기동
5. 각 엔드포인트 수동 테스트:
   - `GET /health` — DB 연결 확인
   - `GET /common-codes` — 공통코드 조회
   - `GET /courses` — 과정 목록
   - `POST /courses` — 과정 생성
   - `GET /classes` — 클래스 목록
   - `GET /reports` — 리포트 조회
6. 기존 e2e 테스트 실행: `pnpm test:e2e` (apps/api)
