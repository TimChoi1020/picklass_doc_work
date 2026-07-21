# 로그인 / 회원가입 기능 구현 계획

**작성일**: 2026-04-11  
**버전**: 3.0  
**참고 프로젝트**: tutoring.picklass.com (동일 패턴/시크릿 공유)

---

## 1. 핵심 요구사항 (확정)

1. **tutoring과 100% 동일한 인증 프로세스** — 같은 `users` 테이블, 같은 SQL, 같은 JWT 페이로드/시크릿
2. **토큰 공유**: tutoring과 studio가 동일한 JWT 시크릿 사용 → 토큰 상호 검증 가능
3. **studio 로그인 제한**: `role_code === 'teacher'`만 허용
4. **회원가입 정책**: studio에서 가입 시 **즉시 `role_code='teacher'`로 활성화** (tutoring은 'student')
5. **AuthUser 응답 형태**: tutoring과 동일 (camelCase: `roleCode`, `institutionId`)
6. **Mock 로직 완전 제거**: 백엔드/프론트엔드 모든 mock 인증 코드 삭제. 실제 로그인만 가능
7. **user_profiles 연동 유지**: 기존 학습 데이터(texts, strategic-reading 등)가 `user_profiles` 테이블 기반이므로, 로그인된 실제 사용자 ID로 연동되어야 함

---

## 2. tutoring 참고 구현 (그대로 차용)

### JWT 설정

| 항목 | 값 |
|------|-----|
| 시크릿 (env) | `JWT_SECRET` |
| 시크릿 fallback | `picklass-tutoring-secret-key-change-in-production` |
| 만료 (env) | `JWT_EXPIRES_IN`, 기본 `7d` |
| 알고리즘 | HS256 |
| 페이로드 | `{ userId, email, roleCode }` |

### users 테이블 SQL (tutoring 차용)

**signup**:
```sql
INSERT INTO users (user_id, email, name, password_hash, role_code, is_temp_password, status_code, activated_at)
VALUES ($1, $2, $3, $4, 'teacher', false, 'active', NOW())
RETURNING id, email, name, role_code, institution_id, password_hash, status_code
```

> tutoring과 차이: `role_code = 'teacher'` (tutoring은 'student')

**login**:
```sql
SELECT id, email, name, role_code, institution_id, password_hash, status_code
FROM users WHERE user_id = $1 AND deleted_at IS NULL LIMIT 1
```

**last_login_at 업데이트**:
```sql
UPDATE users SET last_login_at = NOW() WHERE id = $1::uuid
```

**getMe**:
```sql
SELECT id, email, name, role_code, institution_id, password_hash, status_code
FROM users WHERE id = $1::uuid LIMIT 1
```

### 비밀번호

- bcryptjs, salt rounds = 10

### 엔드포인트

```
POST /auth/signup
  Body: { email, password, nickname }
  Response: { token, user: AuthUser }

POST /auth/login
  Body: { userId, password }
  Response: { token, user: AuthUser }

GET /auth/me
  Header: Authorization: Bearer <token>
  Response: AuthUser
```

### AuthUser 형태 (tutoring과 동일)

```typescript
interface AuthUser {
  id: string;
  email: string | null;
  name: string;
  roleCode: string;
  institutionId: string | null;
}
```

---

## 3. user_profiles 연동 처리

### 현재 상태

| 파일 | 현재 동작 |
|------|----------|
| `apps/web/src/app/api/user-profile/route.ts` | `MOCK_USER_ID = '80bd54a1-...'` 하드코딩 |
| `apps/web/src/app/api/texts/route.ts` | 동일 |
| `apps/web/src/app/api/texts/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/texts/twins/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/strategic-reading/route.ts` | 동일 |
| `apps/web/src/app/api/async-tasks/route.ts` | 동일 |

`user_profiles` 테이블은 `id UUID REFERENCES auth.users(id)`로 정의되어 있고, 기존에는 Supabase auth 사용자 ID와 매칭되어 있었음.

### 신규 정책

로그인 후 발급된 JWT에서 `userId`(= `users.id`, UUID)를 추출하여 모든 mock ID를 실제 사용자 ID로 교체.

> **주의**: `users.id`와 `user_profiles.id`는 서로 다른 시스템(`users`는 자체 테이블, `user_profiles`는 Supabase auth 기반).  
> **확인 필요**: `user_profiles`에 row가 존재하지 않을 경우 자동 생성할지 여부, 그리고 어떤 ID로 매칭할지 (자체 `users.id`로 통일하는 것이 가장 단순).

### 처리 방안

1. Next.js API routes(`apps/web/src/app/api/**`)에서:
   - `Authorization: Bearer <token>` 헤더에서 토큰 추출
   - JWT 검증(같은 JWT_SECRET 사용)으로 `userId` 획득
   - mock 하드코딩 제거 후 `userId`를 실제 사용자 ID로 사용
2. JWT 검증 유틸을 web 측에 신규로 추가 (`apps/web/src/lib/auth/verify-jwt.ts`):
   - jsonwebtoken으로 `JWT_SECRET` 환경변수 사용해 토큰 검증
   - 미인증/미허용 시 401 응답
3. NestJS API의 모든 엔드포인트는 기존 `AuthGuard`(JWT 기반)로 처리

---

## 4. Mock 제거 대상 (전면)

### 백엔드

| 파일 | 제거 내용 |
|------|----------|
| `apps/api/src/common/guards/auth.guard.ts` | `MOCK_USER`, `ALLOW_MOCK_AUTH` 분기 전체 삭제 |
| `apps/api/src/auth/auth.service.ts` | (mock 없음, 그대로) |
| `apps/api/.env`, `.env.example` | `ALLOW_MOCK_AUTH` 변수 제거 |
| `apps/api/test/app.e2e-spec.ts` | mock 의존 테스트 정리 (해당 시) |
| `apps/api/prisma/seed.ts` | mock user seed가 있다면 정리 |

### 프론트엔드

| 파일 | 제거 내용 |
|------|----------|
| `apps/web/src/app/api/user-profile/route.ts` | `MOCK_USER_ID` 제거, JWT에서 userId 추출 |
| `apps/web/src/app/api/texts/route.ts` | 동일 |
| `apps/web/src/app/api/texts/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/texts/twins/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/strategic-reading/route.ts` | 동일 |
| `apps/web/src/app/api/async-tasks/route.ts` | 동일 |
| `apps/web/src/lib/context/GlobalContext.tsx` | mock user 의존 코드 정리 |
| `apps/web/src/components/oizi/StudioHeader.tsx` | mock 표시 제거, 실제 user 표시 |
| `apps/web/src/components/oizi/NewHeader.tsx` | 동일 |

---

## 5. 백엔드 구현 (NestJS)

### STEP 1: 패키지 설치

```
pnpm --filter @classsnap/api add bcryptjs jsonwebtoken
pnpm --filter @classsnap/api add -D @types/bcryptjs @types/jsonwebtoken
```

### STEP 2: AuthService 재작성

**파일**: `apps/api/src/auth/auth.service.ts`

tutoring의 `auth.service.ts`를 차용 + 차이점 반영:

- `signup(email, password, nickname)`: tutoring과 동일하지만 **`role_code = 'teacher'`로 INSERT**
- `login(userId, password)`: tutoring과 동일 + **role_code === 'teacher' 검증 추가**
  - 비밀번호 검증 통과 후 role 체크. 위반 시 `UnauthorizedException('Studio는 강사 계정만 접근 가능합니다.')`
- `getMe(token)`: tutoring과 동일
- `verifyToken(token)`: tutoring과 동일
- `generateToken()`: tutoring과 동일
- `toAuthUser()`: tutoring과 동일 (camelCase 반환)
- 기존 `getUserProfile()`은 AuthGuard에서 사용하므로 유지하되, 반환 형태를 tutoring AuthUser(camelCase)로 통일

### STEP 3: AuthController 재작성

**파일**: `apps/api/src/auth/auth.controller.ts`

tutoring의 controller 형태와 동일:

```typescript
@Controller('auth')
export class AuthController {
  @Post('signup') signup(@Body() body: { email; password; nickname }) {...}
  @Post('login')  login(@Body() body: { userId; password }) {...}
  @Get('me')      me(@Headers('authorization') authHeader?: string) {...}
}
```

> `@UseGuards`를 클래스에 걸지 않고, `me`에서만 헤더 직접 검증.

### STEP 4: AuthGuard 재작성

**파일**: `apps/api/src/common/guards/auth.guard.ts`

- 기존 Supabase 검증 + mock fallback **모두 삭제**
- `AuthService.verifyToken()` 호출 → payload에서 `userId` 추출
- `AuthService.getUserProfile(payload.userId)` 호출하여 `request.user` 주입
- 토큰 없거나 검증 실패 시 즉시 `UnauthorizedException`
- `SupabaseService` 의존성 제거

### STEP 5: 환경 변수

**파일**: `apps/api/.env`, `.env.example`

```
JWT_SECRET=picklass-tutoring-secret-key-change-in-production
JWT_EXPIRES_IN=7d
```

`ALLOW_MOCK_AUTH` 제거.

### STEP 6: shared AuthUser 타입 통일

**파일**: `packages/shared/src/types/auth.ts`

기존:
```typescript
interface AuthUser {
  id: string;
  user_id: string;
  name: string;
  email: string | null;
  role_code: string;
  institution_id: string | null;
  institution_name: string | null;
}
```

→ tutoring 형태로 변경:
```typescript
interface AuthUser {
  id: string;
  email: string | null;
  name: string;
  roleCode: string;
  institutionId: string | null;
}
```

> **주의**: 이 타입은 백엔드/프론트 양쪽에서 import되므로 모든 참조 코드 동시 수정 필요. `role_code` → `roleCode`, `institution_id` → `institutionId` 등.

---

## 6. 프론트엔드 구현 (Next.js)

### STEP 7: 토큰 저장소 유틸

**파일**: `apps/web/src/lib/auth-token.ts` (신규)

```typescript
const TOKEN_KEY = 'picklass_auth_token';  // tutoring과 동일

export const getStoredToken = () => {
  if (typeof window === 'undefined') return null;
  return localStorage.getItem(TOKEN_KEY);
};

export const storeToken = (token: string) => localStorage.setItem(TOKEN_KEY, token);
export const removeToken = () => localStorage.removeItem(TOKEN_KEY);
```

### STEP 8: API 클라이언트 확장

**파일**: `apps/web/src/lib/api.ts`

- `request()` 함수: localStorage 토큰을 자동으로 `Authorization: Bearer` 헤더에 첨부
- 401 응답 시 토큰 삭제 + `/login` redirect (옵션)
- `authApi.login`, `authApi.signup` 추가:

```typescript
export const authApi = {
  getMe: () => request<AuthUser>('/auth/me'),
  login: (data: { userId: string; password: string }) =>
    request<{ token: string; user: AuthUser }>('/auth/login', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
  signup: (data: { email: string; password: string; nickname: string }) =>
    request<{ token: string; user: AuthUser }>('/auth/signup', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
};
```

### STEP 9: useAuth 훅

**파일**: `apps/web/src/hooks/use-auth.ts` (신규)

tutoring `useAuth.ts` 패턴 차용:
- `user`, `loading`, `login()`, `signup()`, `logout()`
- 마운트 시 토큰 있으면 `getMe()` 호출

### STEP 10: AuthProvider Context

**파일**: `apps/web/src/components/AuthProvider.tsx` (신규)

`useAuth` Context 제공. `app/layout.tsx`에서 wrapping.

### STEP 11: 로그인 페이지

**파일**: `apps/web/src/app/login/page.tsx` (신규)

- 입력: userId, password
- 제출 → `useAuth.login()`
- 성공 → `/` redirect
- 실패 → 에러 메시지 (강사 전용 안내 포함)
- 회원가입 페이지 링크

### STEP 12: 회원가입 페이지

**파일**: `apps/web/src/app/signup/page.tsx` (신규)

- 입력: email, password, nickname
- 제출 → `useAuth.signup()` (백엔드에서 자동 'teacher'로 활성화)
- 성공 → `/` redirect

### STEP 13: 보호 경로 처리 (AuthGuard)

**파일**: `apps/web/src/components/AuthGuard.tsx` (신규)

- `useAuth`로 user 상태 확인
- `loading === false && user === null` → `/login` redirect
- `roleCode !== 'teacher'` → `/login` redirect + 토큰 삭제
- 공개 경로(`/login`, `/signup`, `/legal/*`)는 통과
- `app/layout.tsx`에서 적용

### STEP 14: Next.js API routes JWT 검증 + user_profiles 연동

**파일**: `apps/web/src/lib/auth/verify-jwt.ts` (신규)

```typescript
import jwt from 'jsonwebtoken';
import { NextRequest } from 'next/server';

const JWT_SECRET = process.env.JWT_SECRET ?? 'picklass-tutoring-secret-key-change-in-production';

export interface JwtPayload {
  userId: string;
  email: string | null;
  roleCode: string;
}

export function verifyAuthFromRequest(request: NextRequest): JwtPayload | null {
  const auth = request.headers.get('authorization');
  if (!auth?.startsWith('Bearer ')) return null;
  try {
    return jwt.verify(auth.slice(7), JWT_SECRET) as JwtPayload;
  } catch {
    return null;
  }
}
```

**적용 파일** (mock 제거 + JWT 사용):
- `apps/web/src/app/api/user-profile/route.ts`
- `apps/web/src/app/api/texts/route.ts`
- `apps/web/src/app/api/texts/[id]/route.ts`
- `apps/web/src/app/api/texts/twins/[id]/route.ts`
- `apps/web/src/app/api/strategic-reading/route.ts`
- `apps/web/src/app/api/async-tasks/route.ts`

각 라우트 패턴:
```typescript
const payload = verifyAuthFromRequest(request);
if (!payload) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
const userId = payload.userId;  // 실제 users.id
// ... 기존 로직에서 MOCK_USER_ID → userId 교체
```

### STEP 15: user_profiles 자동 생성 정책

`user-profile` API에서 프로필이 없으면 기본값으로 생성하던 로직은 유지하되, 사용 ID를 JWT의 `userId`로 변경.

> **주의**: `user_profiles.id`는 `auth.users(id)` FK였음. `users.id`(자체 테이블)와 매칭이 되지 않을 경우 FK 오류 발생 가능.  
> **대응**: FK 제약을 제거하거나, `user_profiles.id`를 자체 `users.id`와 동일하게 사용 가능한지 DB 수준에서 확인 후 결정. 필요 시 `texts.user_id_fkey`처럼 별도 SQL로 제약 해제.

### STEP 16: 헤더 / 글로벌 컨텍스트 정리

**파일**:
- `apps/web/src/components/oizi/StudioHeader.tsx`
- `apps/web/src/components/oizi/NewHeader.tsx`
- `apps/web/src/lib/context/GlobalContext.tsx`

mock user 표시 제거. `useAuth().user` 사용. 로그아웃 버튼 추가.

---

## 7. 수정 파일 요약

### 백엔드

| 파일 | 변경 |
|------|------|
| `apps/api/package.json` | bcryptjs, jsonwebtoken, types 추가 |
| `apps/api/src/auth/auth.service.ts` | tutoring 차용, signup role_code='teacher', login role 검증 |
| `apps/api/src/auth/auth.controller.ts` | signup, login, me 엔드포인트 |
| `apps/api/src/common/guards/auth.guard.ts` | mock + Supabase 제거, JWT 검증 |
| `apps/api/.env`, `.env.example` | JWT_SECRET, JWT_EXPIRES_IN, ALLOW_MOCK_AUTH 제거 |
| `apps/api/prisma/seed.ts` | mock seed 정리 |
| `apps/api/test/app.e2e-spec.ts` | mock 의존 정리 |
| `packages/shared/src/types/auth.ts` | AuthUser 타입을 camelCase로 통일 |

### 프론트엔드 (신규)

| 파일 | 내용 |
|------|------|
| `apps/web/src/lib/auth-token.ts` | localStorage 토큰 유틸 |
| `apps/web/src/lib/auth/verify-jwt.ts` | Next.js API용 JWT 검증 |
| `apps/web/src/hooks/use-auth.ts` | useAuth 훅 |
| `apps/web/src/components/AuthProvider.tsx` | Context |
| `apps/web/src/components/AuthGuard.tsx` | 보호 경로 |
| `apps/web/src/app/login/page.tsx` | 로그인 페이지 |
| `apps/web/src/app/signup/page.tsx` | 회원가입 페이지 |

### 프론트엔드 (수정)

| 파일 | 변경 |
|------|------|
| `apps/web/src/lib/api.ts` | request() 토큰 자동 첨부, authApi.login/signup |
| `apps/web/src/app/layout.tsx` | AuthProvider, AuthGuard 적용 |
| `apps/web/src/app/api/user-profile/route.ts` | mock 제거, JWT userId 사용 |
| `apps/web/src/app/api/texts/route.ts` | 동일 |
| `apps/web/src/app/api/texts/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/texts/twins/[id]/route.ts` | 동일 |
| `apps/web/src/app/api/strategic-reading/route.ts` | 동일 |
| `apps/web/src/app/api/async-tasks/route.ts` | 동일 |
| `apps/web/src/components/oizi/StudioHeader.tsx` | mock 제거, 실제 user 표시, 로그아웃 |
| `apps/web/src/components/oizi/NewHeader.tsx` | 동일 |
| `apps/web/src/lib/context/GlobalContext.tsx` | mock user 정리 |

### 환경 변수

`JWT_SECRET`은 web 측 서버 컴포넌트(API routes)에서도 필요 → `apps/web/.env.local`에도 동일한 값 추가.

---

## 8. 영향 범위 / 사이드이펙트

- **users 테이블**: 변경 없음 (기존 컬럼 활용)
- **user_profiles 테이블**: 변경 없음. 단, `auth.users(id)` FK 제약이 자체 `users.id` 사용을 막을 수 있음 → 별도 SQL 검토 필요
- **Supabase Auth**: 사용 중단. 기존 SupabaseService는 user_profiles, texts 등 DB 접근 용도로는 유지 (auth만 자체 JWT)
- **shared AuthUser 타입 변경**: snake_case → camelCase로 변경 시 모든 참조 코드 컴파일 에러 발생 — 일괄 수정 필요
- **mock 제거**: 개발 환경에서도 실제 계정으로 로그인해야 함 — 개발용 teacher 계정 사전 준비 필요
- **tutoring과 토큰 호환**: 동일 시크릿/페이로드 → 양 서비스 토큰 상호 사용 가능

---

## 9. 보안

1. **JWT_SECRET**: 운영환경에서는 강력한 시크릿으로 양 서비스 동시 교체
2. **studio role 강제**: 백엔드 login 시 `role_code === 'teacher'` 검증
3. **회원가입 즉시 teacher 활성화**: 누구나 강사 가입 가능 → 운영 단계에서는 초대코드/이메일 인증 등 추가 필요 (이번 범위는 단순 활성화)
4. **Next.js API JWT 검증**: 모든 protected route에 verify-jwt 적용
5. **user_profiles 자동 생성**: 토큰의 userId로만 생성되도록 보장

---

## 10. 작업 순서

1. **shared 타입 변경** (camelCase 통일) — 영향 범위 큼, 가장 먼저
2. 백엔드 STEP 1~5 (패키지, AuthService, Controller, Guard, env)
3. API 빌드 검증
4. 프론트엔드 STEP 7~16 (토큰, API, 훅, 페이지, 가드, mock 제거)
5. Web 빌드 검증
6. 로컬 테스트:
   - 신규 회원가입 → 자동 teacher 활성화 확인
   - 로그인 → 토큰 발급 확인
   - 보호 경로 접근 확인
   - texts/strategic-reading 등 학습 데이터 연동 확인
   - 로그아웃 확인
7. 배포

---

## 11. 결정 사항

| 항목 | 결정 |
|------|------|
| `user_profiles.id` FK 제약 | **제거** — `ALTER TABLE user_profiles DROP CONSTRAINT user_profiles_id_fkey` (texts 때와 동일 패턴) |
| AuthUser 타입 변경 | **변경 진행** — shared 타입을 camelCase로 일괄 통일, 모든 참조 코드 동시 수정 |
| 회원가입 정책 | **즉시 teacher 활성화** |
| AuthUser 응답 형태 | **tutoring과 동일 camelCase** |
| JWT_SECRET | **tutoring과 동일** (`picklass-tutoring-secret-key-change-in-production`) |
| Mock 로직 | **전면 제거** |

## 12. 사전 준비

1. **`user_profiles_id_fkey` 제거** — 완료 (`ALTER TABLE user_profiles DROP CONSTRAINT user_profiles_id_fkey` 실행됨)
2. **개발용 teacher 계정** — `sniper4457@naver.com`
   - 현재 users 테이블에 미존재 → 구현 후 신규 회원가입 페이지에서 직접 가입 (자동 teacher 활성화)
3. **JWT_SECRET 운영 교체** — 이번 작업 범위 제외
