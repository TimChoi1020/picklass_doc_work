# 사용자관리 — 통합 문서

**최종 수정**: 2026-05-28  
**상태**: 운영 중  
**영향 파일**: `apps/admin/frontend/src/app/(admin)/admin/users/`, `apps/admin/backend/src/user/`, `packages/core/src/user/`

---

## 1. 개요

Picklass 백오피스의 사용자 관리 모듈이다. 강사·학생·관리자 계정의 등록·수정·조회·삭제와 아이디 일괄 생성을 제공한다.

**핵심 기능:**
- 역할(6종) 기반 사용자 CRUD
- 역할별 소속 범위 고정 검색 필터
- 강사/학생 대상 자동 로그인(handoff) 진입
- 아이디+액세스코드 일괄 생성

---

## 2. 기술 스택

| 영역 | 스택 |
|------|------|
| Frontend | Next.js 15, React 19, TypeScript |
| Backend | NestJS 11 (`apps/admin/backend`, port 3001) |
| ORM | Prisma + Supabase PostgreSQL |
| 상태 관리 | Zustand (`useAuthStore`) |
| API 호출 | `@/lib/api` (authFetch wrapper) |
| 타입 공유 | `@repo/types` |

---

## 3. 페이지 구조

```
/admin/users                     목록 + 검색 필터
/admin/users/register            신규 등록
/admin/users/[id]/edit           정보 수정
/admin/users/access-code         아이디 일괄 생성
```

---

## 4. 사용자 목록 페이지 (`/admin/users`)

### 4-1. 상단 버튼

| 버튼 | 색상 | 이동 경로 |
|------|------|----------|
| + 사용자 등록 | 파랑 (#2196F3) | `/admin/users/register` |
| 사용자 일괄 생성 | 주황 (#FF9800) | `/admin/users/access-code` |

### 4-2. 검색 필터

| 필드 | 타입 | 설명 | HARD |
|------|------|------|------|
| 상위 파트너 | text | 파트너명 검색 | ✓ (API 미연동) |
| 상위 그룹 | text | 그룹명 검색 | ✓ (API 미연동) |
| 소속기관 | text | 기관명 검색 | ✓ (API 미연동) |
| 역할 | select | `USER_ROLE` 코드 그룹 (system_admin 제외) | — |
| 이름 | text | 부분 일치 | — |
| 아이디 | text | 부분 일치 (현재 클라이언트 미연동) | — |
| 상태 | select | `USER_STATUS` 코드 그룹 | — |

**역할별 필터 고정 (소속 범위 제한):**

| role | 파트너 | 그룹 | 소속기관 |
|------|--------|------|---------|
| `system_admin` | 입력 가능 | 입력 가능 | 입력 가능 |
| `partner_admin` | 🔒 자기 파트너 고정 | 입력 가능 | 입력 가능 |
| `group_admin` | 🔒 소속 파트너 고정 | 🔒 자기 그룹 고정 | 입력 가능 |
| `academy_admin` | 입력 가능 | 입력 가능 | 🔒 자기 기관 고정 |

비활성 필드 스타일: `background: #f5f5f5`, `color: #999`, tooltip "내 소속 기준으로 고정됩니다"

### 4-3. 테이블 컬럼

| 순서 | 컬럼 | 설명 |
|------|------|------|
| 1 | No. | 현재 페이지 기준 순번 |
| 2 | 파트너명 `HARD` | institutions 계층 미연결 — 현재 `-` 표시 |
| 3 | 그룹명 `HARD` | 동일 |
| 4 | 소속기관 | `institutionName` (본사=`null` → '본사') |
| 5 | 역할 | 색상 배지 |
| 6 | 사용자명 | `name` |
| 7 | 아이디 | `userId` (이메일 형식) |
| 8 | 상태 | 색상 배지 |
| 9 | 등록일 | `createdAt` (YYYY-MM-DD) |
| 10 | 작업 | 버튼 그룹 |

### 4-4. 역할별 배지 색상

| roleCode | 색상 | HEX |
|----------|------|-----|
| `teacher` | 주황 | #FF9800 |
| `student` | 보라 | #9C27B0 |
| `academy_admin` | 파랑 | #2196F3 |
| 그 외 | 연두 | #8BC34A |

### 4-5. 상태별 배지 색상

| statusCode | 색상 | HEX |
|------------|------|-----|
| `active` | 녹색 | #4CAF50 |
| `suspended` | 빨강 | #f44336 |
| `withdrawn` | 회색 | #9E9E9E |
| `inactive` 외 | 주황 | #FF9800 |

### 4-6. 작업 버튼

| 조건 | 버튼 | 색상 | 동작 |
|------|------|------|------|
| `teacher` | 스튜디오 진입 | #FF9800 | handoff → studio (새 탭) |
| `student` | 튜터링 진입 | #9C27B0 | handoff → tutoring (새 탭) |
| 모두 | 수정 | btn-secondary | `/admin/users/[id]/edit` 이동 |
| 모두 | 삭제 | btn-danger | confirm 후 soft delete |

### 4-7. 페이지네이션

- 페이지당 10건 고정
- 버튼: 이전 / 페이지 번호 / 다음
- 현재 페이지: 녹색 (#4CAF50)

---

## 5. 사용자 등록 페이지 (`/admin/users/register`)

### 5-1. 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| 사용자 유형 | radio | ✓ | `USER_ROLE` 코드 그룹 (system_admin 제외), 기본값: `teacher` |
| 소속 기관 | select | ✓ | `GET /institutions` API 동적 로드 |
| 이메일 | email | ✓ | 아이디로 사용. 중복확인 필수 |
| 초기 임시 비밀번호 | text | ✓ | 평문 입력 → 백엔드에서 bcrypt(rounds=10) |
| 이름 | text | ✓ | |

### 5-2. 중복확인 동작

1. 이메일 형식 검증 (`/^[^\s@]+@[^\s@]+\.[^\s@]+$/`)
2. `POST /users/check-duplicate` 호출
3. `available: true` → 버튼 녹색 ✓, 등록 허용
4. `available: false` → 에러 메시지, 등록 차단
5. 이메일 변경 시 확인 상태 초기화

### 5-3. 등록 흐름

```
역할 선택 → 기관 선택 → 이메일 입력 → 중복확인 → 비밀번호/이름 입력 → 등록
  → POST /users → 성공 시 /admin/users로 이동
```

### 5-4. 등록 DTO (`CreateUserDto`)

```typescript
{
  roleCode: string;           // USER_ROLE 코드
  institutionId: string | null;
  userId: string;             // 이메일 (아이디)
  password: string;
  name: string;
  email: string;              // userId와 동일값
  phone: string;              // '' (현재 미사용)
  statusCode: string;         // 'active' 고정
}
```

---

## 6. 사용자 수정 페이지 (`/admin/users/[id]/edit`)

### 6-1. 수정 가능 필드

| 필드 | 수정 가능 | 비고 |
|------|----------|------|
| 사용자 유형 (roleCode) | ✓ | select (전체 역할) |
| 소속 기관 | ✓ | `system_admin`은 미표시 |
| 이메일 | ✗ READ-ONLY | 배경 #f5f5f5 |
| 이름 | ✓ | |
| 상태 | ✓ | `USER_STATUS` 코드 그룹 |
| 비밀번호 | 조건부 | 체크박스 "비밀번호 초기화" 선택 후 입력 |

### 6-2. 자동 로그인 버튼

- `teacher` → "스튜디오로 자동 로그인" (#FF9800)
- `student` → "튜터링으로 자동 로그인" (#9C27B0)
- `POST /handoff` API 호출 → `redirectUrl` 새 탭 오픈

### 6-3. 수정 DTO (`UpdateUserDto`)

```typescript
{
  roleCode?: string;
  institutionId: string | null;
  name: string;
  statusCode: string;
  password?: string;          // passwordReset 체크 시만
}
```

---

## 7. 아이디 일괄 생성 (`/admin/users/access-code`)

### 7-1. 필드

| 필드 | 설명 | 제약 |
|------|------|------|
| 사용자 유형 | radio: teacher / student | 필수 |
| 소속 기관 | select | 필수 |
| 생성 개수 | number | 1~1,000 |
| 고유코드 | text (4자리 대문자) | 필수 |

### 7-2. 이메일 생성 형식

```
{고유코드소문자}{일련번호}@{고유코드소문자}.pick

예) 고유코드 'JEIL' → jeil001@jeil.pick, jeil002@jeil.pick, ...
```

### 7-3. 초기 비밀번호

아이디와 동일하게 설정. 사용자는 첫 로그인 후 변경 필요.

### 7-4. 호출 API (`createAccessCodes`)

```typescript
{
  roleCode: 'teacher' | 'student';
  institutionId: string;
  count: number;
  registrationExpiry: string;     // 현재: 1년 후 (고정)
  usagePeriodDays: 365;           // 고정
  statusCode: 'active';
  createUserSimultaneously: true;
  userIdPrefix: string;           // 고유코드 소문자
  userIdDomain: string;           // '{고유코드소문자}.pick'
}
```

---

## 8. API 설계 (NestJS — `@Controller('users')`)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/users` | 목록 조회 (페이징·필터) |
| `GET` | `/users/dashboard` | 역할별 집계 |
| `GET` | `/users/:id` | 단건 조회 |
| `POST` | `/users` | 등록 |
| `PUT` | `/users/:id` | 수정 |
| `DELETE` | `/users/:id` | soft delete (`deletedAt`) |
| `POST` | `/users/check-duplicate` | 아이디 중복 확인 |

### 쿼리 파라미터 (`GET /users`)

| 파라미터 | 설명 |
|----------|------|
| `page` | 기본값 1 |
| `limit` | 기본값 10 |
| `name` | 부분 일치 (insensitive) |
| `roleCode` | 완전 일치 |
| `statusCode` | 완전 일치 |
| `institutionId` | 완전 일치 |

### 응답 타입 (`UserResponse`)

```typescript
{
  id: string;
  institutionId: string | null;
  institutionName: string | null;
  roleCode: string;
  userId: string;               // 아이디 (이메일)
  name: string;
  email: string | null;
  phone: string | null;
  statusCode: string;
  isTempPassword: boolean;
  lastLoginAt: string | null;
  activatedAt: string | null;
  createdAt: string;
  accessCode: string | null;    // active 상태 코드 1개
}
```

---

## 9. 역할 코드 체계

코드 값은 `USER_ROLE` code_items에서 관리 (DB 기준).

| roleCode | 명칭 | 백오피스 접근 | 소속 조직 레벨 |
|----------|------|-------------|--------------|
| `system_admin` | 시스템관리자 | ✓ (전체) | Platform |
| `partner_admin` | 파트너 담당자 | ✓ (범위 제한) | Partner |
| `group_admin` | 그룹 관리자 | ✓ (범위 제한) | Group |
| `academy_admin` | 기관 관리자 | ✓ (범위 제한) | Institution |
| `teacher` | 강사 | ✗ | Institution |
| `student` | 학생 | ✗ | Institution |

> `system_admin` 계정은 `/admin/system/admins`에서 별도 관리. 사용자 관리 목록·등록에서 제외.

---

## 10. 사용자 상태 코드

코드 값은 `USER_STATUS` code_items에서 관리.

| statusCode | 명칭 | 시스템 접근 | 전환 가능 |
|------------|------|-----------|---------|
| `active` | 활성 | ✓ | inactive, suspended, withdrawn |
| `inactive` | 비활성 | ✗ | active, suspended, withdrawn |
| `suspended` | 정지 | ✗ | active, inactive, withdrawn |
| `withdrawn` | 탈퇴 | ✗ | **불가역** (최종 상태) |

---

## 11. 정책

### 11-1. 아이디 정책
- 이메일 형식 필수 (`user@example.com`)
- 시스템 내 고유 (중복 불가)
- 등록 후 **변경 불가** (수정 페이지에서 READ-ONLY)

### 11-2. 비밀번호 정책
- 등록 시: 관리자가 임시 비밀번호 직접 입력 (필수)
- 수정 시: "비밀번호 초기화" 체크박스 선택 후 입력 (선택)
- 저장: bcrypt hash (rounds=10), `isTempPassword: true` 마킹

### 11-3. 검색 정책
- 모든 필드 선택사항, 미입력 시 해당 조건 제외
- AND 조건 결합
- `name`: 부분 일치 (insensitive)
- `roleCode`, `statusCode`: 완전 일치

### 11-4. 삭제 정책
- soft delete: `deletedAt` 타임스탬프 기록
- 목록 조회 시 `deletedAt: null` 조건으로 자동 제외

---

## 12. 에러·성공 메시지

| 상황 | 메시지 |
|------|--------|
| 등록 성공 | `{roleName} 계정이 등록되었습니다.` |
| 수정 성공 | `사용자 정보가 수정되었습니다.` |
| 삭제 성공 | (목록 새로고침) |
| 일괄 생성 성공 | `{N}개의 {역할} 아이디가 생성되었습니다. 초기 비밀번호는 아이디와 동일합니다.` |
| 아이디 사용 가능 | `사용 가능한 이메일입니다.` |
| 아이디 중복 | `이미 사용 중인 이메일입니다.` |
| 중복확인 미완료 | `이메일 중복 확인을 완료해주세요.` |
| 자동 로그인 실패 | `자동 로그인에 실패했습니다.` |

---

## 13. TODO (미구현·진행 중)

| 항목 | 위치 | 내용 |
|------|------|------|
| `TODO[API]` | `users/page.tsx` | `partnerName`, `groupName`, `institution` 쿼리 파라미터 Backend 미연동 |
| `TODO[API]` | `users/page.tsx` | `excludeRole=system_admin` 파라미터 미구현 (현재 클라이언트 필터링) |
| `TODO[DB]` | `users/page.tsx` | `institutions.parent_id` 연결 후 파트너명·그룹명 컬럼 표시 |
| `TODO[DB]` | `users/page.tsx` | `group_admin` 소속 파트너명 자동 세팅 (parent_id 연결 전까지 수동) |

---

## 14. 관련 문서

- 역할·메뉴 권한: [20260527_메뉴구조_권한모델_개발계획.md](./20260527_메뉴구조_권한모델_개발계획.md)
- 회원계정 시스템 전체 계획: [20260520_회원계정시스템_개발계획.md](./20260520_회원계정시스템_개발계획.md)
- 액세스코드 관련: `apps/admin/frontend/src/app/(admin)/admin/accesscode/`
