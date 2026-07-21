# 로그인 / 회원가입 / 액세스코드 — 기능 문서

> 생성일: 2026-04-06
> 대상 파일:
> - `apps/web/src/components/oizi/LoginModal.tsx`
> - `apps/web/src/components/oizi/SignupModal.tsx`
> - `apps/web/src/components/oizi/AccessCodeModal.tsx`
> - `apps/web/src/components/oizi/StudioHeader.tsx`

---

## 1. 사용자 흐름 (User Flow)

### 1-1. 로그인 흐름

```
[StudioHeader] → "로그인" 버튼 클릭
  ↓
LoginModal 열림
  ↓
인증 방법 선택: 이메일/소셜
  ├─ [이메일] 이메일 + 비밀번호 입력 → "로그인" 버튼
  │    ↓
  │   handleSubmit() → ⚠️ alert("로그인 기능은 준비 중입니다.") [미구현]
  │
  └─ [소셜] Google / Kakao / Naver 버튼 클릭
       ↓
      handleSocialLogin(provider) → ⚠️ alert("[provider] 로그인은 준비 중입니다.") [미구현]

LoginModal 하단: "계정이 없으신가요?" → SignupModal로 전환
```

### 1-2. 회원가입 흐름

```
[LoginModal] → 회원가입 링크 클릭 (또는 직접 진입 불가)
  ↓
SignupModal 열림
  ↓
인증 방법 선택: 이메일/소셜
  ├─ [이메일] 이메일 + 비밀번호 + 비밀번호확인 + 닉네임 입력 + 이용약관 동의
  │    ↓
  │   handleSubmit() → 유효성 검사 (약관 동의, 비밀번호 일치)
  │    ↓
  │   ⚠️ alert("회원가입이 완료되었습니다!") [미구현 — 실제 API 없음]
  │
  └─ [소셜] Google / Kakao / Naver 버튼 클릭
       ↓
      ⚠️ alert("[provider] 회원가입은 준비 중입니다.") [미구현]

SignupModal 하단: "이미 계정이 있으신가요?" → LoginModal로 전환
```

### 1-3. 액세스코드 등록 흐름 (✅ 유일하게 실제 구현된 기능)

```
[StudioHeader] → "액세스코드" 네비게이션 버튼 클릭
  ↓
AccessCodeModal 열림
  ↓
액세스코드 입력 (자동 대문자 변환, autoFocus)
  ↓
"등록" 버튼 클릭 → handleSubmit()
  ├─ 빈 값 검증 → error 표시
  ├─ NEXT_PUBLIC_TUTORING_API_URL 환경변수 확인
  ↓
POST {apiUrl}/lessons/register-accesscode
  Body: { accessCode: "ABC123XYZ" } (trim + toUpperCase 처리됨)
  ├─ 성공: success 메시지 표시 → 1.5초 후 모달 닫힘 + onSuccess() 콜백
  │         onSuccess → window.location.reload() (페이지 전체 새로고침)
  └─ 실패: errorData.message 또는 기본 에러 메시지 표시

로딩 중: 닫기 버튼 + 취소 버튼 disabled 처리
```

---

## 2. IA 구조 정리 및 기능 정의 (IA)

| 컴포넌트 | 목적 | 입력 | 출력 | 조건 |
|----------|------|------|------|------|
| `LoginModal` | 기존 사용자 인증 | 이메일, 비밀번호 / 소셜 provider | 인증 토큰 (미구현) | `open=true`일 때 렌더 |
| `SignupModal` | 신규 사용자 가입 | 이메일, 비밀번호×2, 닉네임, 이용약관 | 계정 생성 (미구현) | `open=true`일 때 렌더 |
| `AccessCodeModal` | 선생님이 발급한 코드로 수강 등록 | accessCode (문자열) | 수강 과정 등록 → 목록 갱신 | `open=true`일 때 렌더 |
| `StudioHeader` | 전역 헤더 — 모달 상태 관리자 | pathname (현재 라우트) | 네비 활성화 표시 + 모달 마운트 | 모든 페이지 공통 |

### 네비게이션 링크 매핑

| 헤더 항목 | 타입 | 대상 | 활성화 조건 |
|-----------|------|------|-------------|
| 튜터링 | `<Link>` | `/` | `pathname === '/' or '/tutoring'` |
| 리포트 | `<Link>` | `/report` | `pathname === '/report'` |
| 액세스코드 | `<button>` | AccessCodeModal 열기 | `pathname === '/accesscode'` (현재 라우트 없음) |
| 로그인 | `<button>` | LoginModal 열기 | 별도 활성 상태 없음 |
| Settings | `<button>` | 드롭다운 (계정 정보 표시) | 항상 표시 |

---

## 3. 정책 (Policy / Business Rules)

### 3-1. 현재 적용 중인 정책

| 항목 | 내용 |
|------|------|
| 인증 없이 전체 페이지 접근 가능 | 인증 미들웨어/가드 없음 — Next.js middleware 없음 |
| 사용자 세션 | `MOCK_USER` 하드코딩 (id, email 고정) |
| 액세스코드 입력 | 대소문자 구분 안 함 (입력 즉시 toUpperCase 변환) |
| 액세스코드 빈 값 제출 | 클라이언트에서 차단 — "액세스코드를 입력해주세요" 표시 |
| 액세스코드 등록 로딩 중 | 닫기/취소 버튼 비활성화 — 중복 제출 방지 |
| 등록 성공 후 처리 | 1500ms 딜레이 후 모달 닫기 + `window.location.reload()` |
| 모달 간 전환 | 로그인 ↔ 회원가입 상호 전환 (기존 모달 닫고 새 모달 열기) |
| 로그인/회원가입 | 준비 중 상태 — 실제 인증 없음 |

### 3-2. 정책 변경 필요 사항 (신규 작업 기준)

| 기존 | 변경 필요 내용 | 우선순위 |
|------|---------------|---------|
| MOCK_USER 하드코딩 | 실제 JWT/세션 기반 인증 토큰 도입 | 높음 |
| 로그인 alert("준비 중") | 실제 API 연동 (이메일, 소셜 OAuth) | 높음 |
| 회원가입 alert("완료") | 실제 API 연동 | 높음 |
| window.location.reload() | router.refresh() 또는 상태 업데이트로 교체 | 중간 |
| 액세스코드 활성 상태 | 액세스코드는 라우트가 없으므로 활성화 조건 재검토 | 낮음 |
| 이용약관 alert 검증 | toast/인라인 에러로 교체 | 낮음 |

---

## 4. 개발자 추가 작업 항목

### 4-1. 백엔드 — 미구현 API 엔드포인트

> 현재 `apps/api/src/lessons/lessons.controller.ts`에 `register-accesscode` 엔드포인트가 **존재하지 않음**

```ts
// 추가 필요: lessons.controller.ts
@Post('register-accesscode')
async registerAccessCode(@Body() body: { accessCode: string }) {
  return this.lessonsService.registerAccessCode(body.accessCode);
}
```

관련 서비스 로직 (`lessons.service.ts`):
- `access_codes` 테이블에서 accessCode 유효성 확인
- 유효하면 해당 사용자(studentId)에게 수강 등록 처리
- 만료/이미 사용된 코드 처리 포함 필요

### 4-2. 인증 시스템 구축

- `StudioHeader`의 `MOCK_USER` 제거 → 실제 auth 훅/컨텍스트로 교체
- JWT 기반 인증 또는 Supabase Auth 연동
- 로그인/회원가입 API 엔드포인트 신규 구현 (`/auth/login`, `/auth/signup`, `/auth/social`)
- 소셜 로그인: Google OAuth, Kakao OAuth, Naver OAuth 설정 필요

### 4-3. 프론트엔드 개선

- `window.location.reload()` → `router.refresh()` 또는 React Query 무효화로 교체
- `alert()` 호출 모두 toast 컴포넌트로 교체
- 로그인 상태에 따른 헤더 UI 분기 (현재: 항상 "로그인" 버튼 표시)

---

## 5. 코드 규칙 (Coding Rules)

### 5-1. 사용해야 하는 공통 유틸/컴포넌트

| 항목 | 위치 | 용도 |
|------|------|------|
| `Button` | `@/components/ui/button` | 모든 버튼 — HTML `<button>` 직접 사용 금지 |
| `Input` | `@/components/ui/input` | 텍스트 입력 필드 |
| `fetchApi` | 없음 (백오피스 패턴) | 현재 `fetch()` 직접 사용 중 — 추후 공통 유틸 도입 시 일괄 전환 |
| `useRouter` | `next/navigation` | 페이지 이동 및 refresh |

### 5-2. 금지 패턴

| 금지 패턴 | 이유 | 대안 |
|-----------|------|------|
| `alert()` 직접 사용 | UX 불일치, 차단 모달 | toast 컴포넌트 사용 |
| `any` 타입 | 타입 안전성 저하 | 명시적 인터페이스 정의 |
| `window.location.reload()` | 전체 페이지 새로고침 — SPA 경험 깨짐 | `router.refresh()` |
| 하드코딩된 userId/email | 다중 사용자 환경 오동작 | auth 컨텍스트에서 읽기 |
| 소셜 로그인 provider 문자열 하드코딩 | 확장성 부족 | 상수 배열로 관리 |

### 5-3. 파일 위치 규칙

| 항목 | 위치 |
|------|------|
| 모달 컴포넌트 | `apps/web/src/components/oizi/` |
| 공통 UI 컴포넌트 | `apps/web/src/components/ui/` |
| 타입/인터페이스 | 각 컴포넌트 상단 `interface` 선언 (컴포넌트 전용) 또는 `apps/web/src/types/` (공유 시) |
| 인증 관련 상수 | `apps/web/src/constants/auth.ts` (추후 생성 필요) |

### 5-4. 네이밍 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| Props 인터페이스 | `[ComponentName]Props` | `LoginModalProps` |
| 이벤트 핸들러 | `handle[Action]` | `handleSubmit`, `handleClose` |
| 모달 상태 | `[modalName]Open` | `loginModalOpen` |
| 환경변수 | `NEXT_PUBLIC_TUTORING_*` | `NEXT_PUBLIC_TUTORING_API_URL` |

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| # | 항목 | 위치 | 상태 |
|---|------|------|------|
| 1 | `MOCK_USER` 하드코딩 (id, email 고정) | `StudioHeader.tsx:11-14` | 🔴 미해결 |
| 2 | 로그인 submit → `alert("준비 중")` | `LoginModal.tsx:25` | 🔴 미구현 |
| 3 | 소셜 로그인 → `alert("준비 중")` | `LoginModal.tsx:29-31` | 🔴 미구현 |
| 4 | 회원가입 submit → `alert("완료")` (실제 API 없음) | `SignupModal.tsx:43` | 🔴 미구현 |
| 5 | 이용약관 미동의 → `alert()` 사용 | `SignupModal.tsx:28-30` | 🟡 임시처리 |
| 6 | 비밀번호 불일치 → `alert()` 사용 | `SignupModal.tsx:32-34` | 🟡 임시처리 |
| 7 | 소셜 회원가입 → `alert("준비 중")` | `SignupModal.tsx:48-50` | 🔴 미구현 |
| 8 | `window.location.reload()` — 전체 새로고침 | `StudioHeader.tsx:194-197` | 🟡 임시처리 |
| 9 | 액세스코드 성공 후 1500ms 딜레이 하드코딩 | `AccessCodeModal.tsx:63-66` | 🟡 임시처리 |
| 10 | 백엔드 `POST /lessons/register-accesscode` 미구현 | `lessons.controller.ts` | 🔴 미구현 |
| 11 | `console.log` 디버그 출력 | `LoginModal.tsx:24`, `SignupModal.tsx:42` | 🟡 제거 필요 |
| 12 | 액세스코드 활성 라우트 없음 (`/accesscode` 페이지 없음) | `StudioHeader.tsx:25` | 🟡 설계 불일치 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 7-1. 이 페이지가 사용하는 의존성

```
StudioHeader
  ├─ LoginModal
  │   ├─ @/components/ui/button (Button)
  │   └─ @/components/ui/input (Input)
  ├─ SignupModal
  │   ├─ @/components/ui/button (Button)
  │   └─ @/components/ui/input (Input)
  └─ AccessCodeModal
      ├─ @/components/ui/button (Button)
      ├─ @/components/ui/input (Input)
      └─ fetch() → NEXT_PUBLIC_TUTORING_API_URL
```

### 7-2. 진입점 (이 컴포넌트를 사용하는 곳)

| 사용처 | 파일 | 방식 |
|--------|------|------|
| 모든 페이지 | `apps/web/src/app/page.tsx` | `<StudioHeader />` 렌더 |
| 레슨 페이지 | `apps/web/src/app/modules/[lessonId]/page.tsx` | StudioHeader 미포함 (별도 레이아웃) |

> `layout.tsx`에 StudioHeader가 없음 — 현재는 각 페이지가 직접 StudioHeader를 포함

### 7-3. 이 페이지가 영향을 주는 기능

| 영향 대상 | 내용 |
|-----------|------|
| 홈 페이지 과정 목록 | 액세스코드 등록 성공 → `window.location.reload()` → `/lessons/enrolled-courses` 재조회 |
| 향후 인증 시스템 | StudioHeader의 `user` 상태가 모든 페이지의 로그인 여부 판단 기준 |

---

## 8. DB/API 구조 (Data Contract)

### 8-1. 현재 구현된 API (AccessCodeModal)

#### `POST /lessons/register-accesscode`

> **⚠️ 주의: 백엔드 `lessons.controller.ts`에 이 엔드포인트가 현재 구현되어 있지 않음**
> 프론트엔드에서 호출하나 404 응답 예상

**Request**
```json
{
  "accessCode": "ABC123XYZ"
}
```

**Response (성공 예상)**
```json
{
  "message": "액세스코드가 성공적으로 등록되었습니다.",
  "courseId": "uuid"
}
```

**Response (실패)**
```json
{
  "message": "유효하지 않은 액세스코드입니다."
}
```

### 8-2. 미구현 API (로그인/회원가입)

| 엔드포인트 | 메서드 | 설명 | 상태 |
|-----------|--------|------|------|
| `/auth/login` | POST | 이메일 로그인 | ❌ 미구현 |
| `/auth/signup` | POST | 이메일 회원가입 | ❌ 미구현 |
| `/auth/social/google` | POST | 구글 OAuth | ❌ 미구현 |
| `/auth/social/kakao` | POST | 카카오 OAuth | ❌ 미구현 |
| `/auth/social/naver` | POST | 네이버 OAuth | ❌ 미구현 |

### 8-3. 관련 DB 테이블 (Prisma 기반)

#### `access_codes` (backoffice prisma에서 관리)

> tutoring API에서 읽기 참조 필요

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| code | VARCHAR | 액세스코드 값 (6자리 또는 가변) |
| institution_id | UUID | 발급 기관 |
| role | VARCHAR | 부여할 역할 (student 등) |
| used_at | TIMESTAMPTZ | 사용 시각 |
| expires_at | TIMESTAMPTZ | 만료 시각 |

#### `users` (studio.picklass.com Supabase에서 관리)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK (MOCK_USER.id와 일치해야 함) |
| email | VARCHAR | 이메일 |
| institution_id | UUID | 소속 기관 |
| role_code | VARCHAR | 역할 코드 |

### 8-4. 인터페이스/타입 정의

```ts
// LoginModal Props
interface LoginModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onSwitchToSignup: () => void;
}

// SignupModal Props
interface SignupModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onSwitchToLogin: () => void;
}

// AccessCodeModal Props
interface AccessCodeModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onSuccess?: () => void;
}

// 액세스코드 등록 API Request
interface RegisterAccessCodeRequest {
  accessCode: string; // 항상 대문자로 전송
}

// 향후 정의 필요: 사용자 타입
interface AuthUser {
  id: string;
  email: string;
  nickname?: string;
  institutionId?: string;
  roleCode?: string;
}
```

---

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-06 | 최초 작성 — 코드 분석 기반 (로그인/회원가입 미구현 확인, 액세스코드 부분 구현 확인) |

> **이전 문서 참조**: `dev-plan-20260405.md` §0-3 인증 상태 — LoginModal/SignupModal이 "UI 존재하나 비기능"으로 기록됨. 본 문서는 AccessCodeModal 추가 및 상세 구현 내용 반영.
