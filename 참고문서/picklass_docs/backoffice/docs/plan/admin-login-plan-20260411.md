# 어드민 로그인 및 SSO 자동 로그인 구현 플랜

- 작성일: 2026-04-11
- 작성자: AI Agent
- 상태: **구현 완료 (2026-04-11)** — 빌드/타입체크 모두 통과. 런타임 E2E 검증은 별도.
- 관련 시스템: picklass-backoffice, studio.picklass.com, tutoring.picklass.com

---

## 1. 목표

1. **picklass-backoffice 로그인 화면을 통합 진입점**으로 만든다. 사용자는 자기 계정 1개로 로그인하고, 백엔드가 역할(`role_code`)에 따라 행선지를 결정한다.
   - `system_admin` / `academy_admin` → 백오피스 대시보드 진입.
   - `teacher` → [studio.picklass.com](https://studio-picklass-com-web.vercel.app/)으로 핸드오프 토큰과 함께 자동 로그인 후 리다이렉트.
   - `student` → [tutoring.picklass.com](https://tutoring-picklass-com.vercel.app/)으로 핸드오프 토큰과 함께 자동 로그인 후 리다이렉트.
   - 그 외 역할 / 비활성 계정은 거부.
2. **이게 기본 동작.** 어드민이 별도 화면(사용자 목록/상세)에서 특정 사용자를 선택해 같은 자동 로그인 흐름을 수동으로 트리거하는 기능도 추가로 제공한다(목록 행 액션 + 상세 페이지 버튼 양쪽 모두).

---

## 2. 현재 상태 분석

### 2.1 picklass-backoffice (현재 미구현)
- [apps/admin/backend/src/main.ts](apps/admin/backend/src/main.ts): `express-session` + `cookie-parser` 인프라는 준비되어 있으나 auth 모듈 없음.
- [apps/admin/frontend/src/components/landing/login-modal.tsx](apps/admin/frontend/src/components/landing/login-modal.tsx): UI만 존재. `handleSubmit`이 `alert('로그인 기능은 준비 중입니다.')` 호출.
- [apps/admin/frontend/src/stores/use-auth-store.ts](apps/admin/frontend/src/stores/use-auth-store.ts): Zustand store 골격만 있고 API 미연결.
- [apps/admin/frontend/src/lib/api.ts](apps/admin/frontend/src/lib/api.ts): `fetchApi` 유틸 존재.
- [packages/core/src/user/user.service.ts](packages/core/src/user/user.service.ts): bcrypt 해싱을 포함한 user CRUD 구현 완료. 인증/가드 없음.
- [prisma/schema.prisma](prisma/schema.prisma) `User` 모델: `roleCode` (`system_admin` | `academy_admin` | `teacher` | `student`), `statusCode`, `passwordHash`, `lastLoginAt` 등 인증에 필요한 필드 모두 보유.

### 2.2 studio.picklass.com (구현 완료)
- 백엔드: `apps/api/src/auth/`
  - `POST /auth/login` → `userId` + `password` → `role_code='teacher'`만 허용 → JWT 발급
  - `POST /auth/signup`, `GET /auth/me`
  - JWT payload: `{ userId, email, roleCode }`
  - JWT secret: `process.env.JWT_SECRET ?? 'picklass-tutoring-secret-key-change-in-production'`
  - JWT 만료: `process.env.JWT_EXPIRES_IN ?? '7d'`
- 프론트엔드: `apps/web/src/components/AuthProvider.tsx`
  - localStorage 키: `picklass_auth_token`
  - 모든 요청에 `Authorization: Bearer <token>` 자동 첨부
  - 로드 시 `/auth/me`로 토큰 검증
- **DB는 백오피스와 동일 PostgreSQL `users` 테이블 공유.**

### 2.3 tutoring.picklass.com (구현 완료)
- 백엔드: studio와 거의 동일 구조. 차이점은 로그인 시 **role 제한 없음** (학생 포함 모두 허용).
- JWT secret/만료 정책 동일.
- 프론트엔드: `apps/web/src/lib/hooks/useAuth.ts` — localStorage 키 동일 (`picklass_auth_token`).
- DB 공유.

### 2.4 핵심 발견
- 세 시스템 모두 **동일 PostgreSQL `users` 테이블 사용** → 사용자 동기화 불필요.
- studio/tutoring 양쪽 모두 **JWT 시크릿이 같으면 토큰 호환 가능** (현재 기본값이 동일).
- 단, 두 사이트 모두 **URL 쿼리 파라미터 기반 토큰 수신(자동 로그인) 메커니즘은 미구현** → 양쪽 사이트 로그인 페이지에 핸드오프 수신 로직 추가 필요.

---

## 3. 아키텍처 결정

### 3.1 토큰 핸드오프 방식 (선택)

**옵션 A. 공유 JWT secret + URL 쿼리 토큰 전달 (권장)**
- 백오피스 백엔드가 studio/tutoring과 **동일한 `JWT_SECRET`을 사용**해 대상 사용자용 JWT를 직접 발급한다.
- 백오피스 프런트엔드는 `https://studio.../login?token=...` 또는 `https://tutoring.../login?token=...`로 새 탭을 연다.
- studio/tutoring 로그인 페이지는 쿼리스트링의 `token`을 감지하면 `/auth/me`로 검증 후 localStorage에 저장하고 대시보드로 리다이렉트한다.
- 장점: studio/tutoring 백엔드 변경 최소 (로그인 페이지에 작은 hook만 추가).
- 단점: 토큰이 URL에 노출 → **단명(short-lived) 토큰**(예: 60초)으로 발급, 사용 즉시 localStorage의 정식 토큰으로 교체.

**옵션 B. 백오피스 백엔드 → studio/tutoring 백엔드 신규 "impersonation" 엔드포인트 호출**
- 별도 secret 기반 서버 간 통신 + nonce 발급.
- 장점: URL 토큰 노출 없음.
- 단점: 두 사이트 백엔드 모두 신규 엔드포인트 추가, 어드민-only API 키 관리 필요.

**결정: 옵션 A 채택 (단명 토큰 + 핸드오프).** 구현 부담이 작고 DB 공유라는 전제 덕분에 안전성 확보 가능. 차후 옵션 B로 보강 가능.

### 3.2 어드민 인증 방식
- **JWT (Bearer)** 방식으로 통일. 기존 `express-session` 미들웨어는 제거 또는 보존(보존 결정).
- 어드민 토큰은 별도 키(`picklass_admin_token`)로 localStorage 저장 (스튜디오/튜터링과 충돌 방지).
- 어드민 토큰 payload: `{ userId, email, roleCode }` — 단 `roleCode in ['system_admin','academy_admin']`만 발급.
- 어드민 토큰 만료: 활성 만료 `8h` + **유휴 만료 `1h`** (마지막 활동 후 1시간 무동작 시 강제 만료). 프런트엔드에서 활동 타임스탬프를 갱신하고, 매 요청 시 백엔드가 `iat`/last-activity 기준으로 검증.

### 3.3 환경 변수 정리

| 변수 | backoffice | studio | tutoring |
|---|---|---|---|
| `JWT_SECRET` | 신규, **단일 공유 값** | 동일 값으로 통일 | 동일 값으로 통일 |
| `JWT_EXPIRES_IN_ADMIN` | `8h` | - | - |
| `ADMIN_IDLE_TIMEOUT` | `3600` (초) | - | - |
| `JWT_EXPIRES_IN_HANDOFF` | `60s` | - | - |
| `JWT_EXPIRES_IN` (사용자 일반) | - | `7d` | `7d` |
| `STUDIO_WEB_URL` | `https://studio-picklass-com-web.vercel.app` | - | - |
| `TUTORING_WEB_URL` | `https://tutoring-picklass-com.vercel.app` | - | - |

`.env.example` 갱신 필수.

---

## 4. 작업 범위

### 4.1 picklass-backoffice — Backend (NestJS)

1. **`packages/core`에 `auth` 모듈 신설** (CLAUDE.md 규칙: 비즈니스 로직은 core)
   - `packages/core/src/auth/auth.service.ts`
     - `login(userId, password)` — **통합 진입점**.
       1. bcrypt로 비밀번호 검증, `statusCode='active'` 확인, `lastLoginAt` 갱신.
       2. `roleCode`에 따라 분기:
          - `system_admin` / `academy_admin` → 어드민 JWT 발급 후 `{ kind: 'admin', token, user }` 반환.
          - `teacher` → studio용 핸드오프 토큰(60초) 발급 후 `{ kind: 'handoff', target: 'studio', token, redirectUrl, user }` 반환.
          - `student` → tutoring용 핸드오프 토큰(60초) 발급 후 `{ kind: 'handoff', target: 'tutoring', token, redirectUrl, user }` 반환.
          - 그 외 역할 → 거부.
     - `verifyAdminToken(token)` — Bearer 검증, 유휴 만료 검사, 페이로드 반환.
     - `issueHandoffToken(adminUserId, targetUserId, target: 'studio'|'tutoring')` — **수동 트리거용**.
       - 어드민 권한 재확인.
       - 대상 user 조회 → 역할 일치 검증 (`teacher` ↔ studio, `student` ↔ tutoring).
       - studio/tutoring과 호환되는 JWT 페이로드(`{ userId, email, roleCode }`)로 60초 만료 토큰 발급.
       - 결과: `{ token, redirectUrl }`.
   - `packages/core/src/auth/auth.module.ts`
   - `packages/core/src/auth/dto/` — `LoginDto`, `HandoffRequestDto`
2. **`apps/admin/backend`에 게이트웨이 추가**
   - `src/auth/auth.controller.ts`
     - `POST /auth/login` → core `login` 결과를 그대로 반환. 응답 스키마는 `kind`로 분기되므로 프런트엔드가 그에 맞춰 라우팅 처리.
     - `GET /auth/me` → 어드민 토큰 검증.
     - `POST /auth/handoff` (body: `{ targetUserId, target }`) → 어드민 가드 통과 시 수동 핸드오프 토큰 발급.
   - `src/auth/auth.module.ts` import 등록
   - `src/common/guards/admin-auth.guard.ts` — Bearer 토큰 검증 + 역할 화이트리스트 검사
3. **메인 부트스트랩 정리**
   - `apps/admin/backend/src/main.ts`의 CORS에 `STUDIO_WEB_URL`, `TUTORING_WEB_URL` 도메인 포함 (필요 시).
4. **빌드 절차** (CLAUDE.md 규칙)
   - `packages/core` 수정 후 `pnpm run build` → 백엔드 재시작.

### 4.2 picklass-backoffice — Frontend (Next.js)

1. **`src/lib/api.ts`** — `authApi.login`, `authApi.me`, `authApi.handoff` 추가. 어드민 토큰을 모든 요청에 자동 첨부.
2. **`src/stores/use-auth-store.ts`** — `login`, `logout`, `hydrate(token)` 액션 구현. localStorage 키 `picklass_admin_token`.
3. **`src/components/landing/login-modal.tsx`** — `handleSubmit`을 실제 API 호출로 교체.
   - 응답 `kind`에 따라:
     - `admin`: 어드민 토큰을 `picklass_admin_token`에 저장 후 `/dashboard`로 이동.
     - `handoff`: 응답의 `redirectUrl`로 **`window.location.href`** (현재 탭) 이동. 모달은 닫지 않고 "이동 중" 상태 표시.
   - 실패 케이스(허용되지 않은 역할, 비밀번호 오류, 비활성 계정) 한국어 메시지 처리.
4. **라우트 가드** — `app/(admin)/layout.tsx` 또는 미들웨어로 비로그인 시 랜딩으로 리다이렉트. 로그인 후 토큰 검증(`/auth/me`) 실패 시 강제 로그아웃.
5. **수동 자동 로그인 액션 추가 (목록 + 상세 양쪽)**
   - 대상: `apps/admin/frontend/src/app/(admin)/users/` (목록 행 액션) + 사용자 상세 페이지(헤더 버튼).
   - 역할에 따라 버튼 노출:
     - `teacher` → "스튜디오로 자동 로그인"
     - `student` → "튜터링으로 자동 로그인"
   - 클릭 시 `POST /auth/handoff` 호출 → 응답의 `redirectUrl`로 `window.open(..., '_blank')` (어드민 세션을 잃지 않도록 새 탭).

### 4.3 studio.picklass.com — 자동 로그인 수신 로직

1. **로그인 페이지에 토큰 핸드오프 처리 추가**
   - 파일: `apps/web/src/app/login/page.tsx` (또는 동등 위치)
   - `useSearchParams()`로 `token` 쿼리 추출
   - 존재 시:
     1. `localStorage.setItem('picklass_auth_token', token)`
     2. `GET /auth/me` 호출로 검증
     3. 성공 → URL에서 `token` 제거 후 대시보드로 `router.replace`
     4. 실패 → 토큰 제거 후 일반 로그인 폼 표시 + 토스트
2. **JWT_SECRET 환경 변수 통일** — 백오피스와 동일 값으로 배포 시 설정.
3. (선택) 핸드오프 토큰 1회성 보장을 위해 백엔드에 `jti` 블랙리스트 추가 — Phase 2로 미룸.

### 4.4 tutoring.picklass.com — 자동 로그인 수신 로직
- studio와 동일 패턴. 파일 위치는 `apps/web/src/app/(login route)/page.tsx`. localStorage 키, `/auth/me` 검증, URL 정리, 리다이렉트.

---

## 5. 보안 고려사항

1. **핸드오프 토큰 만료 60초** — URL 노출 시간 최소화.
2. **role 검증을 양쪽에서** — 백오피스가 발급 시점에서, 그리고 studio는 `role_code='teacher'`만 로그인 허용 로직이 이미 존재하므로 이중 차단.
3. **HTTPS 강제** — 운영 환경에서만 핸드오프 허용. 로컬 개발은 예외 처리.
4. **감사 로그** — 어드민의 자동 로그인 사용 이력을 `audit_logs` 또는 신규 테이블에 기록 (Phase 2 권장, 본 플랜에서는 TODO).
5. **어드민 토큰 분리** — `picklass_admin_token` vs `picklass_auth_token` 키 분리로 동일 브라우저에서 어드민이 사용자 사이트를 열어도 어드민 세션 영향 없음.
6. **CORS 화이트리스트** — 백오피스 백엔드는 admin 프런트엔드 도메인만 허용.
7. **비밀번호 정책** — 본 플랜에서는 변경 없음. `isTempPassword` 처리는 별도 작업으로 분리.

---

## 6. 단계별 실행 순서

| 단계 | 작업 | 의존성 |
|---|---|---|
| 1 | 환경 변수 정리 + `JWT_SECRET` 공유 값 결정 | - |
| 2 | `packages/core/auth` 모듈 작성 + 빌드 | 1 |
| 3 | `apps/admin/backend` auth 컨트롤러/가드 연결 | 2 |
| 4 | `apps/admin/frontend` 로그인 모달/스토어/가드 구현 | 3 |
| 5 | 어드민 계정으로 로그인 E2E 검증 | 4 |
| 6 | 백오피스 핸드오프 엔드포인트(`/auth/handoff`) 구현 | 3 |
| 7 | studio 로그인 페이지에 토큰 수신 hook 추가 | 1 |
| 8 | tutoring 로그인 페이지에 토큰 수신 hook 추가 | 1 |
| 9 | 사용자 목록에 자동 로그인 버튼 추가 + 통합 검증 | 6,7,8 |
| 10 | 문서화 (`docs/users/`, `docs/system/auth.md`) | 9 |

---

## 7. 검증 시나리오

1. **어드민 로그인 정상**: `system_admin` 계정 로그인 → 대시보드 접근 가능.
2. **권한 없는 계정 차단**: `teacher` 계정으로 백오피스 로그인 시도 → 403 + 한국어 에러.
3. **잘못된 비밀번호**: 401 + 한국어 에러.
4. **비활성 계정**: `statusCode='inactive'` 사용자 → 403 + 한국어 에러.
5. **토큰 만료**: 8시간 경과 후 어드민 API 호출 → 401 → 자동 로그아웃 + 랜딩 이동.
6. **teacher 자동 로그인**: 사용자 목록에서 teacher 행의 버튼 클릭 → 새 탭 studio.picklass.com 대시보드 진입.
7. **student 자동 로그인**: 동일 패턴으로 tutoring.picklass.com 진입.
8. **역할 불일치 차단**: teacher 사용자에게 tutoring 핸드오프 시도 → 백오피스 백엔드 400.
9. **핸드오프 토큰 재사용 차단**: 60초 경과 후 동일 URL 접속 → studio `/auth/me` 401 → 일반 로그인 폼 노출.
10. **CORS / 환경 변수**: 운영 도메인 기준 동작 확인.

---

## 8. 변경 영향 파일 (예상)

### picklass-backoffice
- `packages/core/src/auth/**` (신규)
- `packages/core/src/index.ts` (export)
- `apps/admin/backend/src/auth/**` (신규)
- `apps/admin/backend/src/app.module.ts`
- `apps/admin/backend/src/main.ts` (CORS)
- `apps/admin/backend/.env`, `.env.example`
- `apps/admin/frontend/src/lib/api.ts`
- `apps/admin/frontend/src/stores/use-auth-store.ts`
- `apps/admin/frontend/src/components/landing/login-modal.tsx`
- `apps/admin/frontend/src/app/(admin)/layout.tsx` (가드)
- `apps/admin/frontend/src/app/(admin)/users/...` (자동 로그인 버튼)
- `apps/admin/frontend/.env.local`, `.env.example`

### studio.picklass.com
- `apps/web/src/app/login/page.tsx` (또는 동등 라우트)
- `apps/web/.env.example` — `JWT_SECRET` 공유 값 명시
- `apps/api/.env.example`

### tutoring.picklass.com
- `apps/web/src/app/<login route>/page.tsx`
- `apps/web/.env.example`
- `apps/api/.env.example`

---

## 9-1. 구현 결과 (2026-04-11)

### 실제 추가/수정된 파일

**@repo/types** — 공유 타입
- [packages/types/src/index.ts](packages/types/src/index.ts) — `AuthUser`, `LoginDto`, `LoginResult`(=`AdminLoginResult | HandoffLoginResult`), `HandoffTarget`, `HandoffRequestDto`, `HandoffResponse` 추가.

**@repo/core** — 비즈니스 로직 (NestJS 모듈)
- [packages/core/src/auth/auth.service.ts](packages/core/src/auth/auth.service.ts) — `login()`(역할 기반 분기), `getMe()`, `verifyAdminToken()`, `issueHandoffToken()`. `jsonwebtoken`으로 서명, bcrypt 비교, `lastLoginAt` 갱신, 대상 사이트 URL 빌드.
- [packages/core/src/auth/auth.module.ts](packages/core/src/auth/auth.module.ts) — `ConfigModule` + `PrismaModule` 의존.
- [packages/core/src/auth/index.ts](packages/core/src/auth/index.ts)
- [packages/core/src/index.ts](packages/core/src/index.ts) — `./auth` re-export 추가.
- [packages/core/package.json](packages/core/package.json) — `jsonwebtoken`, `@types/jsonwebtoken`, `@nestjs/config` 의존성 추가.

**admin backend** (NestJS 게이트웨이)
- [apps/admin/backend/src/auth/auth.controller.ts](apps/admin/backend/src/auth/auth.controller.ts) — `POST /auth/login`, `GET /auth/me`, `POST /auth/handoff`. `/auth/handoff`는 `AdminAuthGuard` 적용.
- [apps/admin/backend/src/auth/admin-auth.guard.ts](apps/admin/backend/src/auth/admin-auth.guard.ts) — Bearer 토큰 추출 → `authService.verifyAdminToken()` → `req.adminUserId`/`adminRole` 주입.
- [apps/admin/backend/src/auth/auth.module.ts](apps/admin/backend/src/auth/auth.module.ts)
- [apps/admin/backend/src/app.module.ts](apps/admin/backend/src/app.module.ts) — `AuthModule` 등록.
- [apps/admin/backend/src/main.ts](apps/admin/backend/src/main.ts) — **`express-session` + `cookie-parser` 제거 완료** (데드 코드). CORS 유지.
- [apps/admin/backend/package.json](apps/admin/backend/package.json) — `cookie-parser`, `express-session`, 관련 `@types/*` 의존성 제거.

**admin frontend** (Next.js)
- [apps/admin/frontend/src/lib/auth-token.ts](apps/admin/frontend/src/lib/auth-token.ts) — `picklass_admin_token` 관리 + 유휴 1시간 체크(`isAdminSessionIdle`, `touchAdminActivity`, `IDLE_TIMEOUT_MS=3600000`).
- [apps/admin/frontend/src/lib/api.ts](apps/admin/frontend/src/lib/api.ts) — `fetchApi`가 매 요청에 `Authorization: Bearer` 자동 첨부, 유휴 체크, 401 → 자동 로그아웃 + `/login` 리다이렉트. `ApiError` 클래스 export. `skipAuth` 옵션으로 로그인 요청 분리.
- [apps/admin/frontend/src/lib/api/auth.ts](apps/admin/frontend/src/lib/api/auth.ts) — `apiLogin`, `apiMe`, `apiHandoff`.
- [apps/admin/frontend/src/stores/use-auth-store.ts](apps/admin/frontend/src/stores/use-auth-store.ts) — Zustand 스토어 재작성. `AuthUser` 보관, `setToken`, `logout`, `hasToken`.
- [apps/admin/frontend/src/app/login/page.tsx](apps/admin/frontend/src/app/login/page.tsx) — **신규 통합 로그인 페이지**. 응답 `kind`에 따라 `admin`이면 `/admin/dashboard` 이동, `handoff`면 `result.redirectUrl`로 `window.location.href` 이동.
- [apps/admin/frontend/src/components/auth/admin-auth-gate.tsx](apps/admin/frontend/src/components/auth/admin-auth-gate.tsx) — 클라이언트 가드. 마운트 시 토큰/유휴 체크 → `/auth/me` 검증 → 활동 이벤트(mousemove/keydown/click/scroll) 구독 → 60초마다 유휴 검사.
- [apps/admin/frontend/src/app/(admin)/layout.tsx](apps/admin/frontend/src/app/(admin)/layout.tsx) — `<AdminAuthGate>`로 감쌈.
- [apps/admin/frontend/src/components/landing/login-modal.tsx](apps/admin/frontend/src/components/landing/login-modal.tsx) — 기존 `alert('준비 중')` 제거, 실제 `apiLogin` 호출 + 역할 분기 이동.
- [apps/admin/frontend/src/app/(admin)/admin/users/page.tsx](apps/admin/frontend/src/app/(admin)/admin/users/page.tsx) — 행별 "스튜디오 진입" / "튜터링 진입" 버튼 추가 (`handleAutoLogin`).
- [apps/admin/frontend/src/app/(admin)/admin/users/[id]/edit/page.tsx](apps/admin/frontend/src/app/(admin)/admin/users/[id]/edit/page.tsx) — 헤더에 "스튜디오로 자동 로그인" / "튜터링으로 자동 로그인" 버튼 추가.

**studio.picklass.com** — 자동 로그인 수신
- [apps/web/src/app/login/page.tsx](../../../studio.picklass.com/apps/web/src/app/login/page.tsx) — `useSearchParams()`로 `token` 감지 → `storeToken()` → `authApi.getMe()` 검증 → 성공 시 `/`로 이동, 실패 시 토큰 제거 + 에러. 처리 중에는 "자동 로그인 중..." 화면.

**tutoring.picklass.com** — 자동 로그인 수신
- [apps/web/src/lib/hooks/useAuth.ts](../../../tutoring.picklass.com/apps/web/src/lib/hooks/useAuth.ts) — `useAuthProvider` 초기 `checkAuth`에 핸드오프 토큰 처리 추가. 쿼리의 `token`을 localStorage에 저장한 뒤 URL에서 제거(`history.replaceState`) → 기존 `getStoredToken()` + `/auth/me` 검증 플로우 재사용.

### 검증 수행 사항
- `pnpm install` → 신규 패키지 설치 완료 (prisma generate는 파일 잠금으로 실패했으나 기존 client가 그대로 사용되어 이후 빌드에는 영향 없음).
- `pnpm --filter @repo/types run build` ✅
- `pnpm --filter @repo/core run build` ✅
- `pnpm --filter @app/admin-api run build` (nest build) ✅
- `pnpm --filter @app/admin-frontend run build` (next build) ✅ — `/login` 라우트 포함 34개 라우트 생성 확인.
- `studio.picklass.com` → `pnpm run typecheck` ✅
- `tutoring.picklass.com` → `pnpm run typecheck` ✅

### 환경 변수 (운영 배포 시 확인 필요)
- `JWT_SECRET` — **세 시스템 동일 값**으로 배포해야 핸드오프 토큰 검증 가능. 기본값(`picklass-tutoring-secret-key-change-in-production`)은 studio/tutoring 기존 기본값과 일치하도록 맞춤.
- `JWT_EXPIRES_IN_ADMIN` — 기본 `8h`.
- `JWT_EXPIRES_IN_HANDOFF` — 기본 `60s`.
- `STUDIO_WEB_URL` — 기본 `https://studio-picklass-com-web.vercel.app`.
- `TUTORING_WEB_URL` — 기본 `https://tutoring-picklass-com.vercel.app`.

### 런타임 E2E 검증 (미수행 — 운영자 확인 필요)
§7의 10가지 시나리오는 아직 수동 검증되지 않았다. 로컬 환경에서 DB의 테스트 계정(`system_admin`, `teacher`, `student`)으로 각 시나리오를 확인할 것.

### 구현 중 변경된 설계 사항
1. **유휴 만료는 프론트엔드 단에서 처리**. 당초 백엔드에서 `iat` 기반 재검증을 고려했으나 JWT는 재발급이 필요하므로, MVP로서 프론트엔드의 활동 타임스탬프(`picklass_admin_last_activity`) + 60초 주기 폴링으로 구현. 백엔드는 8시간 절대 만료만 강제. (보안 강화가 필요하면 Phase 2에서 Redis + 세션 상태 추가.)
2. **스튜디오/튜터링 API 쪽은 수정하지 않음**. 이미 Bearer 토큰 + `/auth/me` 플로우가 완비되어 있고, 백오피스가 같은 `JWT_SECRET`으로 서명하므로 기존 검증이 그대로 통한다. 수신 hook만 web 쪽에 추가.
3. **튜터링 로그인 경로**. 튜터링은 별도 `/login` 라우트 없이 `page.tsx`에서 모달 기반으로 동작하므로, 핸드오프 처리를 `useAuthProvider` 초기 로딩에 포함시켜 어느 페이지로 진입하든 토큰이 처리된다. 백오피스 `AuthService.buildRedirectUrl`은 튜터링에 대해서는 `/?token=...`, 스튜디오에 대해서는 `/login?token=...`을 사용한다.

---

## 9. 추후 과제 (Phase 2 후보)

1. 핸드오프 토큰의 `jti` 1회성 검증(Redis 또는 DB).
2. 어드민 자동 로그인 감사 로그 테이블.
3. 어드민 IP 화이트리스트 / 2FA.
4. `isTempPassword=true` 사용자에 대한 임시 비밀번호 변경 강제 플로우.
5. 옵션 B (서버 간 impersonation API)로의 보강.

---

## 10. 결정 사항 (2026-04-11 확정)

1. **JWT_SECRET 공유 정책**: 세 시스템 모두 **단일 공유 값** 사용.
2. **어드민 토큰 만료**: 활성 만료 `8h` + **유휴 만료 `1h`** (마지막 활동 후 1시간 무동작 시 강제 만료).
3. **자동 로그인 동작**:
   - 기본은 **백오피스 통합 로그인 화면**에서 역할 분기 자동 처리.
   - 추가로 사용자 목록 행 액션 + 사용자 상세 페이지 버튼 **양쪽 모두** 제공.
4. **기존 `express-session` / `cookie-parser` 미들웨어**: **제거 결정.**
   - 정체: [apps/admin/backend/src/main.ts:30-37](apps/admin/backend/src/main.ts#L30-L37)에서 `MemoryStore` 기본 세션을 등록하고 있으나, 코드베이스 어디에서도 `req.session`을 참조하지 않는 **데드 코드**.
   - 시크릿 기본값(`'picklass-admin-secret'`)이 하드코딩되어 있고, Vercel 서버리스 환경(같은 파일 line 60의 `handler` export)에서는 인스턴스 간 메모리 공유가 안 되어 동작 자체도 불완전.
   - 본 플랜은 **JWT(Bearer) 단독** 방식이므로 세션 미들웨어와 `cookieParser`를 함께 제거한다. 단, CORS의 `credentials: true`는 향후 쿠키 기반 보강 가능성을 위해 보존.
