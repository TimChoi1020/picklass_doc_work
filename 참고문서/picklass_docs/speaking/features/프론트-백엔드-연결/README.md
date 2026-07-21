# 프론트-백엔드 연결 (Phase 7)

| 항목 | 값 |
|------|-----|
| 작성일 | 2026-05-08 |
| 상태 | 완료 |
| 대상 저장소 | `speaking.picklass.com/apps/web` |
| 관련 단계 | Phase 7 |

---

## 1. 목표

- 서비스 레이어 (`lib/services/`) 도입으로 컴포넌트 → authFetch → API 흐름 확립
- 로그인 페이지 구현 (JWT 발급 → localStorage 저장 → 인증 보호 라우팅)
- 홈 / 학습 / 마이 페이지에 실 API 데이터 연결
- 데모 계정: `tim.choi@oizi.net`

---

## 2. 아키텍처

### 2.1 레이어 구조

```
컴포넌트 (page.tsx / *.tsx)
    ↓ 호출
lib/services/*.ts          ← 이번 단계 신설
    ↓ 호출
lib/authFetch.ts           ← 기존 (JWT 헤더 + 에러 처리)
    ↓ HTTP
apps/api (NestJS, :3004)
```

**규칙**: 컴포넌트에서 직접 `fetch` / `authFetch` 호출 금지. 반드시 service 함수를 통해 호출.

### 2.2 서비스 파일

#### `lib/services/usersService.ts`

```typescript
export async function getMyProfile(): Promise<AuthUser>
// GET /users/me
```

#### `lib/services/coursesService.ts`

```typescript
export interface InProgressCourse {
  id: string; title: string; levelCode: string; genreCode: string;
  totalLessons: number; completedLessons: number;
  lastScore: number; lastCompletedAt: string | null;
  nextLessonOrder: number; nextLessonTopic: string | null;
  progress: number;
}

export async function getInProgress(): Promise<InProgressCourse[]>
// GET /courses/in-progress
```

---

## 3. 인증 흐름

### 3.1 로그인 (`/onboarding/login`)

```
사용자 입력 (userId + password)
    ↓
useAuth().login(userId, password)
    ↓ POST /auth/login
NestJS AuthService.login()
  → users WHERE user_id = $1 (이메일과 동일한 경우가 많음)
  → bcrypt.compare(password, hash)
  → JWT 발급 (payload: { userId, email, roleCode })
    ↓ response: { token, user }
localStorage.setItem('picklass_auth_token', token)
setUser(userData)
router.replace('/home')
```

> 데모 계정 `tim.choi@oizi.net`의 `user_id` 컬럼 값으로 로그인 (Supabase tutoring DB 공유).
> user_id가 이메일과 동일하면 이메일로 로그인 가능.

이미 로그인 상태로 `/onboarding/login` 접근 시 → `useEffect`로 `/home` 자동 리다이렉트.

### 3.2 세션 유지 (`useAuthProvider`)

앱 시작 시 `localStorage`에 토큰이 있으면 `GET /auth/me` 호출 → `user` 상태 복원.
토큰 만료 또는 비활성화 계정 → 토큰 삭제, 비인증 상태.

`IDLE_TIMEOUT_SECONDS` (기본 3600초) 초과 시 세션 만료.
API 응답 헤더 `X-Refresh-Token` 감지 시 localStorage 교체.

### 3.3 탭 레이아웃 인증 가드 (`(tabs)/layout.tsx`)

```typescript
// isLoading: true → 스피너 표시
// isAuthenticated: false → /onboarding/login 리다이렉트
// isAuthenticated: true → 탭 화면 렌더
```

`useEffect`로 `isLoading` 해소 후 인증 여부 판단. 미인증 시 `router.replace('/onboarding/login')`.

---

## 4. 페이지별 연결 상태

| 페이지 | 실 API 연결 항목 | 데모 고정값 |
|--------|-----------------|------------|
| 홈 | `user.name` (`useAuth()`) | 레벨, 스트릭, 미션, 추천 |
| 학습 | `/courses/in-progress` → 진행 중 과정 HERO | For You 추천, 카테고리, 시험 대비 |
| 피드백 | — (Phase 8 예정) | 전체 하드코딩 |
| 챌린지 | — (Phase 8 예정) | 전체 하드코딩 |
| 마이 | `user.name`, `user.email` (`useAuth()`) | 레벨, 역할, 구독, 시작일, 뱃지 |

---

## 5. 학습 탭 — 로딩 / 빈 상태 처리

```typescript
// 로딩 중: InProgressSkeleton (animate-pulse)
// 데이터 있음: InProgressCourseCard (실 데이터)
// 데이터 없음: 빈 상태 안내 문구 ("아직 진행 중인 과정이 없어요")
```

장르 코드 → 한국어 레이블 매핑 (learn/page.tsx 내 `GENRE_LABEL` 상수):

```typescript
const GENRE_LABEL: Record<string, string> = {
  business: '비즈니스', travel: '여행', daily: '일상',
  exam: '시험', academic: '학업',
};
```

---

## 6. 환경 변수

| 파일 | 변수 | 값 |
|------|------|-----|
| `apps/web/.env.local` | `NEXT_PUBLIC_SPEAKING_API_URL` | `http://localhost:3004` |
| `apps/web/.env.local` | `JWT_SECRET` | api와 동일한 값 (핸드오프 검증용) |

서비스 파일에서 API URL 참조:
```typescript
const api = () => process.env.NEXT_PUBLIC_SPEAKING_API_URL ?? '';
```

---

## 7. 변경 파일 목록

| 파일 | 변경 유형 | 내용 |
|------|-----------|------|
| `apps/web/src/lib/services/usersService.ts` | **신규** | GET /users/me wrapper |
| `apps/web/src/lib/services/coursesService.ts` | **신규** | GET /courses/in-progress wrapper + InProgressCourse 타입 |
| `apps/web/src/app/onboarding/login/page.tsx` | **구현** | 로그인 폼 (userId/password), 인증 시 리다이렉트 |
| `apps/web/src/app/(tabs)/layout.tsx` | **수정** | 클라이언트 컴포넌트 전환 + auth guard + 로딩 스피너 |
| `apps/web/src/app/(tabs)/home/page.tsx` | **수정** | useAuth() → displayName 실 데이터 |
| `apps/web/src/app/(tabs)/learn/page.tsx` | **수정** | getInProgress() 호출 + 스켈레톤/빈 상태 |

---

## 8. 검증 결과

- `pnpm typecheck` — 3/3 통과 ✅
- 로그인 페이지: POST /auth/login → JWT 발급 → /home 이동 확인
- 학습 탭: /courses/in-progress 응답으로 진행 중 과정 카드 표시
- 마이 탭: user.name, user.email 실 DB 값 표시

---

## 9. 다음 단계 (Phase 8 예정)

- 피드백 탭: `/feedback/summary`, `/feedback/lesson-history`, `/feedback/kpi-trends` 연결
- 챌린지 탭: 스트릭·뱃지 실데이터 (API 미구현 — 별도 모듈 필요)
- 배포: Vercel (web) + 서버 (api), 환경변수 설정, CORS 도메인 추가
