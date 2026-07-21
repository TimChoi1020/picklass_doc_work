# Course 미구현 항목 구현 계획

> 작성일: 2026-03-27
> 분석 기반: 전체 course 관련 문서 9건 + 현재 코드베이스 분석

---

## 현재 구현 상태

### 완료된 항목

| 기능 | 상태 | 비고 |
|------|------|------|
| 과정 목록 CRUD | API 연동 완료 | 필터링, 페이지네이션, 삭제 disable 포함 |
| 과정 상세 (레슨 관리) | API 연동 완료 | 순서변경, 삭제, 모듈수정, 레슨추가 모두 API |
| 카드 그룹 필터링 | API 연동 완료 | 최근/배포중/인기 과정 별도 쿼리 |
| AI 토픽 생성 | API 연동 완료 | `/ai/generate-topics` |
| 레슨 추가 시 지문 선택 | API 연동 완료 | `useTextsList()` |

### 미구현 항목

| 기능 | 현재 상태 | 문제점 |
|------|-----------|--------|
| 액세스코드 관리 페이지 | **Mock 데이터 (8건 하드코딩)** | API 엔드포인트 없음, 백엔드 모듈 없음 |
| 학생 아이디 관리 페이지 | **Mock 데이터 (12건 하드코딩)** | API 엔드포인트 없음, 백엔드 모듈 없음 |
| 과정 액세스코드 생성 | **부분 구현** | `courses.service.generateAccessCode()`가 `access_codes` 테이블에 실제 저장하지 않고 deployment_count만 증가 |

---

## 기존 인프라 현황

### DB 테이블 (Supabase에 이미 존재)

**`access_codes` 테이블** (backoffice Prisma에 정의됨):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| code | CHAR(6) | 고유 코드 (UNIQUE) |
| institution_id | UUID | 소속 기관 FK |
| role_code | VARCHAR(50) | teacher / student |
| user_id | UUID? | 사용자 FK (nullable) |
| course_id | UUID? | 과정 FK (nullable) |
| status_code | VARCHAR(50) | active / inactive |
| registration_expiry | DATE | 등록 만료일 |
| usage_period_days | INT | 사용 기간 (일) |
| usage_start_date | DATE? | 사용 시작일 |
| usage_end_date | DATE? | 사용 종료일 |
| generated_user_id | VARCHAR(255)? | 자동생성 학생 아이디 |
| generated_password | VARCHAR(255)? | 자동생성 비밀번호 |
| activated_at | TIMESTAMPTZ? | 활성화 일시 |
| created_at | TIMESTAMPTZ | 생성일 |
| updated_at | TIMESTAMPTZ | 수정일 |

**`users` 테이블** (studio Prisma에 정의됨):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| institution_id | UUID? | 소속 기관 FK |
| role_code | VARCHAR(50) | 역할 코드 |
| user_id | VARCHAR(255) | 로그인 아이디 (UNIQUE) |
| password_hash | VARCHAR(255) | 비밀번호 해시 |
| name | VARCHAR(100) | 이름 |
| status_code | VARCHAR(50) | active / suspended / withdrawn |
| is_temp_password | BOOLEAN | 임시 비밀번호 여부 |
| ... | | |

### Studio API 백엔드 (`apps/api/src`)

기존 모듈: `courses`, `classes`, `passages`, `reports`, `ai`, `common-codes`, `auth`, `health`

**없는 모듈: `access-codes`, `students`**

### Studio Prisma 스키마 (`apps/api/prisma/schema.prisma`)

- `access_codes` 모델 **미정의** → Prisma를 통해 접근하려면 모델 추가 필요
- `users` 모델 **정의됨** (snake_case 필드)

---

## 구현 계획

### STEP 1: Prisma 스키마에 `access_codes` 모델 추가

**파일:** `apps/api/prisma/schema.prisma`

- 기존 Supabase `access_codes` 테이블에 맞는 모델 추가
- backoffice 스키마 참고하되, studio 프로젝트 컨벤션(snake_case) 적용
- 마이그레이션 없이 `db pull` 또는 수동 모델 추가 (테이블이 이미 존재하므로)

---

### STEP 2: 액세스코드 백엔드 모듈 (`apps/api/src/access-codes/`)

**신규 파일:**
```
apps/api/src/access-codes/
  access-codes.module.ts
  access-codes.controller.ts
  access-codes.service.ts
  dto/
    create-access-codes.dto.ts
    query-access-codes.dto.ts
    update-access-code-status.dto.ts
```

**엔드포인트:**

| Method | Path | 설명 |
|--------|------|------|
| GET | `/access-codes` | 목록 조회 (필터: status, role, code 검색, 페이지네이션) |
| POST | `/access-codes` | 대량 생성 (roleCode, count, registrationExpiry, usagePeriodDays) |
| PATCH | `/access-codes/:id/status` | 상태 토글 (active ↔ inactive) |
| DELETE | `/access-codes/:id` | 삭제 |

**코드 생성 로직:**
- 6자리 랜덤 대문자영숫자 (DB unique 제약으로 중복 방지)
- 프론트에서 표시 시 prefix 추가 (TC-/ST-), DB에는 6자리만 저장
- institution_id: 현재 사용자의 기관
- 최대 1,000개 일괄 생성

**`app.module.ts`에 등록**

---

### STEP 3: 액세스코드 프론트엔드 API 연동

**3-1. Shared 타입 추가** (`packages/shared/src/types/`)
- `AccessCode` 인터페이스 (DB 스키마 매핑)
- `AccessCodeListResponse` — `{ data: AccessCode[]; total: number; page: number; totalPages: number }`
- `AccessCodeQueryParams` — `{ status?: string; role?: string; search?: string; page?: number; limit?: number }`
- `CreateAccessCodesPayload` — `{ roleCode: string; count: number; registrationExpiry: string; usagePeriodDays: number }`

**3-2. API 클라이언트** (`apps/web/src/lib/api.ts`)
- `accessCodesApi.list(params)` → GET `/access-codes`
- `accessCodesApi.create(data)` → POST `/access-codes`
- `accessCodesApi.updateStatus(id, statusCode)` → PATCH `/access-codes/:id/status`
- `accessCodesApi.delete(id)` → DELETE `/access-codes/:id`

**3-3. React Query 훅** (`apps/web/src/hooks/use-access-codes.ts` 신규)
- `useAccessCodesList(params)` — 목록 쿼리
- `useCreateAccessCodes()` — 대량 생성 mutation
- `useUpdateAccessCodeStatus()` — 상태 변경 mutation
- `useDeleteAccessCode()` — 삭제 mutation

**3-4. queryKeys 추가** (`apps/web/src/lib/react-query.ts`)

**3-5. 페이지 수정** (`apps/web/src/app/course/accesscode/page.tsx`)
- `mockCodes` (line 79-88) 제거
- `useState(mockCodes)` → React Query 훅으로 교체
- `handleGenerateSubmit` → `useCreateAccessCodes()` 호출
- `handleStatusConfirm` → `useUpdateAccessCodeStatus()` 호출
- `handleDeleteConfirm` → `useDeleteAccessCode()` 호출
- 필터: 서버 사이드 쿼리 파라미터로 전환
- 통계 바: API 응답의 total 값 활용

---

### STEP 4: 학생 관리 백엔드 모듈 (`apps/api/src/students/`)

**신규 파일:**
```
apps/api/src/students/
  students.module.ts
  students.controller.ts
  students.service.ts
  dto/
    create-student.dto.ts
    bulk-create-students.dto.ts
    update-student.dto.ts
    query-students.dto.ts
```

**엔드포인트:**

| Method | Path | 설명 |
|--------|------|------|
| GET | `/students` | 목록 조회 (필터: role, name, userId, status, 페이지네이션) |
| POST | `/students` | 단일 등록 (userId, name, tempPassword) |
| POST | `/students/bulk` | 대량 등록 (count, withAccessCode 옵션) |
| PATCH | `/students/:id` | 수정 (name, status, password) |
| GET | `/students/check-duplicate` | 아이디 중복 확인 (?userId=...) |
| GET | `/students/next-serial` | 다음 시리얼 번호 조회 |

**핵심 로직:**
- `users` 테이블 활용 (role_code = 'student')
- 아이디 형식: `{STUDIO_CODE}{3자리 시리얼}@{STUDIO_CODE}.pick`
- `next-serial`: DB에서 현재 최대 시리얼 조회 후 +1 반환
- `check-duplicate`: `users.user_id` unique 확인
- `bulk`: 트랜잭션으로 N명 학생 생성 + (옵션) N개 액세스코드 동시 생성
- 비밀번호: 해시 처리 후 저장, `is_temp_password = true`

**`app.module.ts`에 등록**

---

### STEP 5: 학생 관리 프론트엔드 API 연동

**5-1. Shared 타입 추가** (`packages/shared/src/types/`)
- `Student` 인터페이스 (users 테이블 매핑)
- `StudentListResponse` — `{ data: Student[]; total: number; page: number; totalPages: number }`
- `StudentQueryParams` — `{ role?: string; name?: string; userId?: string; status?: string; page?: number; limit?: number }`
- `CreateStudentPayload`, `BulkCreateStudentsPayload`, `UpdateStudentPayload`

**5-2. API 클라이언트** (`apps/web/src/lib/api.ts`)
- `studentsApi.list(params)` → GET `/students`
- `studentsApi.create(data)` → POST `/students`
- `studentsApi.bulkCreate(data)` → POST `/students/bulk`
- `studentsApi.update(id, data)` → PATCH `/students/:id`
- `studentsApi.checkDuplicate(userId)` → GET `/students/check-duplicate`
- `studentsApi.nextSerial()` → GET `/students/next-serial`

**5-3. React Query 훅** (`apps/web/src/hooks/use-students.ts` 신규)
- `useStudentsList(params)` — 목록 쿼리
- `useCreateStudent()` — 단일 등록 mutation
- `useBulkCreateStudents()` — 대량 등록 mutation
- `useUpdateStudent()` — 수정 mutation
- `useCheckDuplicate(userId)` — 중복 확인 쿼리
- `useNextSerial()` — 시리얼 조회 쿼리

**5-4. queryKeys 추가** (`apps/web/src/lib/react-query.ts`)

**5-5. 페이지 수정** (`apps/web/src/app/course/students/page.tsx`)
- `mockStudents` (line 21-34) 제거
- `useState(mockStudents)` → React Query 훅으로 교체
- `handleSubmit` → `useCreateStudent()` 호출
- `handleBulkSubmit` → `useBulkCreateStudents()` 호출
- `handleEditSubmit` → `useUpdateStudent()` 호출
- `handleCheckDuplicate` → `useCheckDuplicate()` 호출
- `generateStudentId` → `useNextSerial()` 호출
- 필터: 서버 사이드 쿼리 파라미터로 전환

---

### STEP 6: 과정 액세스코드 생성 보완

**파일:** `apps/api/src/courses/courses.service.ts`

현재 `generateAccessCode()`가 deployment_count만 증가시키고 실제 `access_codes` 테이블에 저장하지 않음.

**수정 내용:**
- `access_codes` 테이블에 실제 레코드 생성
- `course_id` 필드에 과정 ID 연결
- deployment_count 증가 유지

---

## 구현 순서 및 의존성

```
STEP 1  Prisma 스키마 access_codes 모델 추가
  │
  ├── STEP 2  액세스코드 백엔드 API
  │     │
  │     └── STEP 3  액세스코드 프론트 API 연동
  │
  ├── STEP 4  학생 관리 백엔드 API (대량 등록 시 access_codes 사용)
  │     │
  │     └── STEP 5  학생 관리 프론트 API 연동
  │
  └── STEP 6  과정 액세스코드 생성 보완
```

- STEP 2, 4, 6은 STEP 1 완료 후 병렬 진행 가능
- STEP 3은 STEP 2 완료 후
- STEP 5는 STEP 4 완료 후

---

## 수정 대상 파일 목록

### 신규 생성

| 파일 | STEP |
|------|------|
| `apps/api/src/access-codes/access-codes.module.ts` | 2 |
| `apps/api/src/access-codes/access-codes.controller.ts` | 2 |
| `apps/api/src/access-codes/access-codes.service.ts` | 2 |
| `apps/api/src/access-codes/dto/create-access-codes.dto.ts` | 2 |
| `apps/api/src/access-codes/dto/query-access-codes.dto.ts` | 2 |
| `apps/api/src/access-codes/dto/update-access-code-status.dto.ts` | 2 |
| `apps/web/src/hooks/use-access-codes.ts` | 3 |
| `apps/api/src/students/students.module.ts` | 4 |
| `apps/api/src/students/students.controller.ts` | 4 |
| `apps/api/src/students/students.service.ts` | 4 |
| `apps/api/src/students/dto/create-student.dto.ts` | 4 |
| `apps/api/src/students/dto/bulk-create-students.dto.ts` | 4 |
| `apps/api/src/students/dto/update-student.dto.ts` | 4 |
| `apps/api/src/students/dto/query-students.dto.ts` | 4 |
| `apps/web/src/hooks/use-students.ts` | 5 |

### 수정

| 파일 | STEP | 내용 |
|------|------|------|
| `apps/api/prisma/schema.prisma` | 1 | access_codes 모델 추가 |
| `apps/api/src/app.module.ts` | 2, 4 | AccessCodesModule, StudentsModule 등록 |
| `packages/shared/src/types/` | 3, 5 | AccessCode, Student 타입 추가 및 export |
| `apps/web/src/lib/api.ts` | 3, 5 | accessCodesApi, studentsApi 추가 |
| `apps/web/src/lib/react-query.ts` | 3, 5 | queryKeys 추가 |
| `apps/web/src/app/course/accesscode/page.tsx` | 3 | Mock 제거, API 연동 |
| `apps/web/src/app/course/students/page.tsx` | 5 | Mock 제거, API 연동 |
| `apps/api/src/courses/courses.service.ts` | 6 | generateAccessCode에서 access_codes 테이블 실제 저장 |
