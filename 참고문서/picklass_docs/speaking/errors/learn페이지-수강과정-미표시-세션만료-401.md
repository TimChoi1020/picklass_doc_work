# learn 페이지 수강 과정 미표시 — 세션 만료 401 미처리 (2026-06-23)

> **서비스**: speaking.picklass.com
> **관련 기능**: [Learn 페이지 실 API 연동](../features/learn페이지-실API연동/20260619_learn페이지_실API연동_개발계획.md)
> **변경 파일**: `apps/web/src/lib/authFetch.ts`, `apps/web/src/lib/hooks/useAuth.ts`
>
> 두 가지 증상을 다룬다: (1) 수강 과정 미표시(세션 만료 401 미처리), (2) learn 진입 시 로그아웃 반복(일시적 API 재시작 블립).

---

## 증상 (Symptom)

learn 페이지에서 수강 과정이 보이지 않고 빈 상태 메시지가 표시됨:

```
{사용자}님의 수업 현황
등록된 과정이 없습니다.
선생님에게 액세스코드를 받아 과정을 등록해보세요.
```

- 화면 상단에는 **사용자 이름이 정상 표시**(로그인된 것처럼 보임)되는데 과정만 비어 있음.
- `tim.choi@oizi.net`(파트너 소속), `student@picklass.com`(기관 소속, 파트너 없음) **양쪽 모두** 재현.

## 원인 (Root Cause)

**세션 토큰(JWT) 만료.** `JWT_EXPIRES_IN=1h`이고 토큰 갱신(`X-Refresh-Token`)은 요청이 있을 때만
일어나므로, 탭을 **1시간 이상 idle**로 두면 토큰이 만료된다.

진단 과정에서 백엔드는 전부 정상임을 확인:

- DB `access_codes` 쿼리 → tim 2개(speaking) / student 1개(speaking) 정상 반환.
- 라이브 API `GET /lessons/enrolled-courses` (신규 발급 토큰) → 두 계정 모두 과정 정상 반환.
- `GET /auth/me`(신규 토큰) 200, CORS preflight 204, env(`NEXT_PUBLIC_SPEAKING_API_URL`) 정상.
- **DevTools Network: `me → 401`** 확인 → localStorage 토큰이 만료돼 `/auth/me`가 거부됨.

**왜 "로그인된 듯 보였는가"**: `useAuth`의 `user` state(이름)가 이전 성공한 `/auth/me` 결과로
메모리에 남아 있었고, learn 페이지의 `getEnrolledCourses().catch(() => setCourses([]))`가
**401을 조용히 삼켜** 빈 배열로 처리 → "등록된 과정이 없습니다"라는 **오해 메시지**로 표출.

> 핵심: 데이터/백엔드 문제가 아니라 **만료 세션 + 401 미처리**였다. 파트너 여부와도 무관
> (learn 수강 과정은 `access_codes` 기반만 사용, `partnerId` 미사용).

## 해결 방법 (Resolution)

CLAUDE.md §6.4("wrapper가 401 → 로그아웃 일괄 처리")에 맞춰 `authFetch`에서 401을 중앙 처리.

### `apps/web/src/lib/authFetch.ts`
- 요청에 **Authorization 토큰을 실어 보냈는지**(`sentAuth`) 추적.
- 응답이 `401 && sentAuth`이면 → `localStorage` 토큰 제거 + `auth:unauthorized`(`AUTH_UNAUTHORIZED_EVENT`) 윈도우 이벤트 발생.
- 로그인/회원가입의 **자격증명 실패 401**은 토큰을 첨부하지 않으므로(`sentAuth=false`) 자동 제외 → 오작동 없음.

### `apps/web/src/lib/hooks/useAuth.ts`
- `useAuthProvider`에 `AUTH_UNAUTHORIZED_EVENT` 리스너 추가 → 수신 시 `setUser(null)`.
- `user=null` → `(tabs)/layout.tsx`의 가드가 `/onboarding/login`으로 리다이렉트.

결과: idle 만료 시 **빈 과정 목록(오해 메시지) 대신 로그인 화면으로 자연 전환**. 학습 흐름 중간
만료도 동일하게 처리됨.

**검증**: `pnpm --filter @speaking/web typecheck` 통과, 두 파일 eslint 통과.
사용자 측 즉시 해결책은 **재로그인**(신규 토큰 발급 시 과정 정상 표시 — 라이브 API로 확인 완료).

---

## 후속 증상: learn 진입 시 "로그아웃 반복" (2026-06-23)

### 증상
401 처리 적용 후, learn 페이지로 갈 때 로그아웃이 반복됨.

### 원인
**dev API(`nest start --watch`)의 재시작 블립.** `nest-cli.json`의 `deleteOutDir: true` 때문에
재컴파일마다 `dist/`를 지웠다 재생성 → `dist/main` 프로세스가 수 초간 다운. 그 윈도우 동안:

- `useAuth`의 `checkAuth()`가 `/auth/me`를 호출 → API 다운이라 **fetch 가 throw**(401 아님, 네트워크 오류)
- 기존 코드 `catch { removeToken() }` 가 **일시 오류와 토큰 만료를 구분하지 않고** 멀쩡한 토큰까지 삭제 → 로그아웃
- API가 여러 번 재시작되면(개발 중 api 코드 저장 시마다) 로그아웃 반복

> 진단: `curl /health` 가 `000`(연결 실패) → 수 초 후 `200` 으로 복구되는 flapping 확인. `dist/main.js`
> mtime 이 재시작 시각과 일치. 일시 다운은 **네트워크 오류**라 401 핸들러(authFetch)와는 무관.

### 해결
`checkAuth()` 를 **401일 때만 로그아웃**, 네트워크/5xx 일시 오류는 **토큰 유지 + 최대 3회 재시도**
(800ms 간격)로 변경. API 재시작(~1–2초)에는 세션이 보존된다.

```ts
for (let attempt = 0; attempt < 3; attempt++) {
  try {
    const res = await authFetch(`${apiUrl}/auth/me`);
    if (res.ok) { setUser(await res.json()); break; }
    if (res.status === 401) { removeToken(); break; }  // 만료/무효만 로그아웃
    // 5xx 등 → 재시도
  } catch { /* 네트워크 오류: 토큰 유지하고 재시도 */ }
  if (attempt < 2) await new Promise((r) => setTimeout(r, 800));
}
```

### 부수 정리 항목 (선택)
- `apps/api` prisma client 폴더에 실패한 `prisma generate`의 `query_engine-windows.dll.node.tmp*`
  잔여 파일 다수. 실행 중인 API가 `.dll`을 잠가 rename 이 EPERM 으로 실패한 흔적. 런타임엔 영향
  없으나(실제 `query_engine-windows.dll.node` 로드), 정리 권장. **API 실행 중 `prisma generate` 금지**.

---

## 진짜 근본 원인: JWT_SECRET 폴백으로 인한 서명 불일치 (2026-06-23, 확정)

### 증상
재로그인 직후에도 learn 진입 시 로그인으로 리다이렉트가 **지속**. 같은 토큰으로 `/config/brand`,
`/challenge/utterance` 등은 **200**인데 `/lessons/enrolled-courses`, `/course-categories`만 **401**.

### 진단
- 가드에 임시 로그 추가 → `401 [verifyToken 실패=JWT 만료/서명불일치]` 확인.
- idle 아님(student idle 549초 ≪ 3600초), 만료 아님(로그인 9분 전, JWT 1h).
- 현재 API에 **REAL secret 토큰 → 200, FALLBACK secret 토큰 → 401** 확인 → 사용자의 실패 토큰은
  **폴백 secret 으로 서명된 것**.

### 원인 (Root Cause)
`app.module.ts` 의 `ConfigModule.forRoot({ isGlobal: true })` 에 **`envFilePath` 미지정** →
**cwd 기준**으로 `.env` 를 로드한다. API 가 재시작(`nest start --watch` + `deleteOutDir`)될 때
cwd 가 `apps/api` 가 아니면 `.env` 를 못 찾아 `JWT_SECRET` 미로드 → `auth.service` 의
**폴백 secret**(`'picklass-speaking-secret-key-change-in-production'`)으로 토큰을 발급/검증.

→ 어떤 부팅은 폴백, 어떤 부팅은 실제 secret 을 쓰게 되어, **폴백으로 서명된 로그인 토큰이 실제
secret 으로 검증하는 인스턴스에서 401**. config/challenge 는 폴백 인스턴스에서 처리돼 200, 그 뒤
재시작된 실제-secret 인스턴스에서 learn 요청이 401 → 로그아웃.

### 해결 (Resolution)
1. `apps/api/src/app.module.ts` — `envFilePath: join(__dirname, '..', '.env')` 로 **cwd 무관 절대경로 고정**.
2. `apps/api/src/main.ts` — `dotenv.config({ path: resolve(__dirname, '..', '.env') })` 동일 고정.
3. `apps/api/src/auth/auth.service.ts` — **폴백 secret 제거**. `JWT_SECRET` 미로드 시 부팅에서
   `throw` (조용한 secret 불일치 대신 명시적 실패).

검증: typecheck 통과. 재기동 후 REAL→200 / FALLBACK→401 일관. **사용자는 1회 재로그인 필요**
(기존 토큰은 폴백 서명이라 무효).

> **임시**: `auth.guard.ts` 에 추가한 401 사유 로그(`WARN [JwtAuthGuard] 401 [...]`)는 확인용. 검증 후 제거.

---

## 재발 방지 대책 (Prevention)

- **인증 API 호출은 반드시 `authFetch`(wrapper) 경유** — 401 일괄 처리가 한 곳에서 보장됨(§6.4).
  컴포넌트/서비스에서 직접 `fetch` 금지.
- **401을 `.catch(() => setEmpty([]))`로 삼키지 말 것** — "데이터 없음"과 "세션 만료"를 혼동시키는
  오해 메시지의 원인. 만료는 로그인 유도로 분기.
- 빈 상태 메시지를 띄우기 전, 그 빈 데이터가 **정상 빈 결과인지 / 인증 실패 결과인지** 구분한다.
- 토큰 검증에서 **일시적 네트워크 오류(API 재시작/다운)와 401(만료/무효)를 구분**한다. 네트워크 오류로
  토큰을 삭제하지 말 것 — dev API 가 코드 저장마다 재시작되므로 세션이 계속 끊긴다.

체크리스트:

- [ ] 새 인증 API 호출이 `authFetch`를 거치는가? (직접 fetch 금지)
- [ ] 401이 사용자에게 "데이터 없음"이 아니라 "세션 만료 → 로그인"으로 보이는가?
- [ ] 토큰 검증이 **네트워크 오류에 토큰을 지우지 않고 재시도**하는가? (401일 때만 로그아웃)
- [ ] API 실행 중 `prisma generate`/`build`를 돌리지 않았는가? (dll 잠금 EPERM + 재시작 유발)
- [ ] **인증 secret/환경변수에 코드 폴백을 두지 않았는가?** 폴백은 미로드를 숨겨 재시작 후 토큰을
      무효화한다. `.env` 경로는 `__dirname` 기준 절대경로로 고정하고, 미로드 시 부팅에서 실패시킨다.
