# 로그인 후속 수정 계획

**작성일**: 2026-04-11  
**버전**: 1.0  
**관련 작업**: [login-impl-plan-20260411.md](./login-impl-plan-20260411.md) 후속

---

## 1. 작업 항목

| # | 항목 | 우선순위 |
|---|------|---------|
| 1 | 회원가입 시 기관명(institution) 검색/바인딩 | HIGH |
| 2 | 메인 페이지 헤더가 로그인 전/후 동일한 문제 수정 | HIGH |

---

## 2. 항목 1: 회원가입 시 기관명 입력 → submit 시 검증 바인딩

### 현황

- `users.institution_id`가 null이면 `courses`, `classes`, `students` 등 생성 시 FK/NOT NULL 에러 발생
- 현재 회원가입 폼은 email, password, nickname만 입력
- 신규 가입 시 institution이 자동 할당되지 않아 기능이 작동 안 함

### DB 구조

`institutions` 테이블:
| 컬럼 | 타입 | 비고 |
|------|------|------|
| id | UUID | PK |
| name | VARCHAR(200) | 기관명 |
| member_status_code | VARCHAR(50) | 'active' / 'inactive' |
| ... | | |

`users.institution_id`는 `institutions(id)` FK (nullable).

### 정책

- 회원가입 폼에 **기관명 텍스트 input만 추가**
- **검색/드롭다운/자동완성 없음** — 사용자가 정확한 기관명을 직접 입력
- **submit 시 백엔드에서 검증** — 입력한 기관명이 DB에 존재하면 해당 id를 user에 바인딩, 없으면 에러 반환

### 구현 계획

#### STEP 1: 백엔드 — Signup에서 기관명 → ID 조회 후 바인딩

**파일**: `apps/api/src/auth/auth.service.ts`

`signup()` 시그니처/로직 변경:
```typescript
async signup(email, password, nickname, institutionName): Promise<AuthResult>
```

처리 순서:
1. 이메일 중복 검사 (기존)
2. **기관명 조회**:
   ```sql
   SELECT id FROM institutions
   WHERE name = $1
     AND member_status_code = 'active'
     AND deleted_at IS NULL
   LIMIT 1
   ```
3. 결과 없으면 `BadRequestException('존재하지 않는 기관명입니다.')` 반환
4. 결과 있으면 user INSERT 시 `institution_id` 포함:
   ```sql
   INSERT INTO users (user_id, email, name, password_hash, role_code, is_temp_password, status_code, activated_at, institution_id)
   VALUES ($1, $2, $3, $4, 'teacher', false, 'active', NOW(), $5::uuid)
   RETURNING id, email, name, role_code, institution_id, password_hash, status_code
   ```

`AuthController.signup` body type도 `institutionName` 추가.

> 별도 institution 검색 API/모듈 불필요. 기관 정보는 외부에 노출되지 않음.

#### STEP 2: 프론트엔드 — API 클라이언트

**파일**: `apps/web/src/lib/api.ts`

`authApi.signup` 시그니처 변경:
```typescript
signup: (data: { email; password; nickname; institutionName }) =>
  request<{ token; user }>('/auth/signup', { method: 'POST', body: ... })
```

#### STEP 3: 프론트엔드 — useAuth 훅

**파일**: `apps/web/src/hooks/use-auth.ts`, `apps/web/src/components/AuthProvider.tsx`

`signup()` 시그니처에 `institutionName` 추가.

#### STEP 4: 프론트엔드 — 회원가입 페이지 UI

**파일**: `apps/web/src/app/signup/page.tsx`

추가 입력 필드: **기관명** (단순 텍스트 input)
- placeholder: "정확한 기관명을 입력하세요"
- 필수 입력
- 검색/자동완성/드롭다운 없음
- 입력 값을 그대로 `signup()`에 전달
- 백엔드가 400 응답하면 폼 에러 메시지 표시 ("존재하지 않는 기관명입니다.")

상태:
```typescript
const [institutionName, setInstitutionName] = useState('');
```

---

## 3. 항목 2: 메인 페이지 헤더 로그인 전/후 동일 문제

### 현황

- 메인 페이지 [page.tsx](apps/web/src/app/page.tsx)는 [NewHeader.tsx](apps/web/src/components/oizi/NewHeader.tsx) 사용
- NewHeader는 user 유무에 따라 SVG 이미지만 교체:
  ```jsx
  <img src={user ? "/oizi/images/newheader.svg" : "/oizi/images/header.svg"} />
  ```
- 사용자 메뉴/로그인 버튼 등 실제 UI 요소 없음 (SVG 이미지에 그려진 영역에 absolute Link만 배치)
- **문제**: `useAuth().user`가 정상 동작해도 SVG만 바뀌므로, 시각적 차이가 거의 없거나 SVG 자체가 동일하게 보일 수 있음. 또한 메인 페이지(`/`)는 `AuthGuard`가 막지 않으므로 비로그인 상태도 접근 가능.

### 진단 방향

다음 중 하나 이상이 원인:

1. **SVG 이미지 자체가 시각적으로 거의 동일** (header.svg vs newheader.svg)
2. **로그인/로그아웃 버튼 부재** — 헤더에 로그인 페이지로 이동하거나 로그아웃하는 UI가 없음
3. **`useAuth()` 호출 시점 불일치** — main 페이지가 server component인 상태에서 NewHeader만 client component로 동작하면서 hydration 후에야 user state가 업데이트되어 첫 렌더 시 user=null로 보일 수 있음 (이 경우는 시간이 지나면 자동 갱신됨)

### 구현 계획

**파일**: `apps/web/src/components/oizi/NewHeader.tsx`

#### 변경 1: 명시적 로그인/로그아웃 버튼 추가

SVG 위에 absolute 영역으로 다음을 표시:
- **비로그인 상태**: "로그인" 버튼 (`/login`으로 이동)
- **로그인 상태**: 사용자 이름 + Settings 드롭다운(이미 존재) + "로그아웃" 항목 추가

기존 settings dropdown에 로그아웃 추가:
```tsx
<button onClick={handleLogout}>로그아웃</button>
```

#### 변경 2: 로그아웃 핸들러 추가

```tsx
const { user, logout } = useAuth();
const router = useRouter();

const handleLogout = () => {
  logout();
  setIsSettingsOpen(false);
  router.replace('/login');
};
```

#### 변경 3: 비로그인 상태 로그인 버튼 영역 표시

settings 영역(`91.528%` 위치)에 user가 없으면 로그인 버튼, 있으면 settings 버튼 분기.

```tsx
{!user ? (
  <Link href="/login">로그인</Link>
) : (
  <button onClick={() => setIsSettingsOpen(!isSettingsOpen)}>...</button>
)}
```

> SVG는 그대로 두고, 위에 텍스트 버튼을 absolute로 추가하는 방식. 디자인 가이드가 별도로 있으면 그에 맞춰 조정.

---

## 4. 수정 파일 요약

### 백엔드

| 파일 | 변경 |
|------|------|
| `apps/api/src/auth/auth.service.ts` | signup에 institutionName 추가 + DB 조회 후 institution_id 바인딩 |
| `apps/api/src/auth/auth.controller.ts` | signup body에 institutionName 추가 |

### 프론트엔드

| 파일 | 변경 |
|------|------|
| `apps/web/src/lib/api.ts` | authApi.signup 시그니처에 institutionName 추가 |
| `apps/web/src/hooks/use-auth.ts` | signup 시그니처에 institutionName 추가 |
| `apps/web/src/components/AuthProvider.tsx` | 동일 |
| `apps/web/src/app/signup/page.tsx` | 기관명 input 필드 추가, 입력값 그대로 전달 |
| `apps/web/src/components/oizi/NewHeader.tsx` | 로그인/로그아웃 버튼 추가, 로그아웃 핸들러 |

---

## 5. 영향 범위 / 사이드 이펙트

- **기존 가입자**: 영향 없음 (institution_id가 이미 있음)
- **테스트 계정 sniper4457**: 이미 institution 할당됨 — 영향 없음
- **기존 로그인 흐름**: 변경 없음 (login은 기존대로 user_id/password로 동작)
- **메인 페이지 비인증 접근**: 그대로 허용 (랜딩 페이지는 public)
- **AuthGuard**: 변경 없음

---

## 6. 확인 필요 사항

1. **헤더 디자인**: 로그인/로그아웃 버튼을 SVG 위에 추가할 때 위치/스타일 가이드
2. **메인 페이지 SVG 차이**: 두 SVG 파일을 직접 열어서 시각적 차이가 있는지 사용자가 확인 필요
