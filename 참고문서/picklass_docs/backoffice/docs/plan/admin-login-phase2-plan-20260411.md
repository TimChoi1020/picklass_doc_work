# 인증 Phase 2: 유휴 만료 + 사용자 식별 쿠키

- 작성일: 2026-04-11
- 작성자: AI Agent
- 상태: **구현 완료 (2026-04-11)** — 빌드/타입체크 전원 통과. 런타임 E2E 검증은 별도.
- 관련 시스템: picklass-backoffice, studio.picklass.com, tutoring.picklass.com
- 선행 문서: [admin-login-plan-20260411.md](admin-login-plan-20260411.md) (Phase 1 — 구현 완료)

---

## 1. 목표

1. **세 시스템 모두 공통으로, 사용자의 마지막 활동 시점을 기준으로 1시간 동안 움직임이 없으면 토큰을 만료**시킨다.
   - 기존 Phase 1에서는 백오피스 **프론트엔드 단에서만** `picklass_admin_last_activity` 타임스탬프를 체크했음. 이번에는 **백엔드에서 강제**한다.
   - 스튜디오/튜터링의 기존 7일 절대 만료 대신(또는 더불어) 1시간 유휴 만료가 실제 접근 차단의 주요 수단이 된다.
2. **세 시스템 모두 로그인 성공 시 `users.id` 값을 담은 쿠키를 발급**한다.
   - 쿠키 이름: `picklass_user_id`
   - 쿠키 값: `users.id` (UUID 문자열)
   - 사이트별 독립 도메인에 설정(크로스 도메인 공유는 없음 — 세 사이트가 서로 다른 `vercel.app` 서브도메인과 자체 도메인에 존재하므로 공유 불가).

---

## 2. 현재 상태 요약 (Phase 1 기준)

### 2.1 토큰 / 만료 정책
| 시스템 | JWT secret | 절대 만료 | 유휴 만료 | 강제 위치 |
|---|---|---|---|---|
| 백오피스 (admin) | 공유 | `8h` (`JWT_EXPIRES_IN_ADMIN`) | `1h` | **프론트엔드만** (localStorage 타임스탬프) |
| 백오피스 (handoff) | 공유 | `60s` | - | JWT 자체 `exp` |
| 스튜디오 | 공유 | `7d` (`JWT_EXPIRES_IN`) | 없음 | - |
| 튜터링 | 공유 | `7d` (`JWT_EXPIRES_IN`) | 없음 | - |

### 2.2 세션 식별
- 세 시스템 모두 **JWT Bearer만 사용** 중. 쿠키 인증 없음.
- 백오피스: Phase 1에서 `express-session`/`cookie-parser` 제거.

### 2.3 관련 스키마 (users 테이블)
- `lastLoginAt` 필드만 존재 (로그인 시점에만 갱신됨).
- **마지막 활동 시점(`lastActiveAt`) 필드 없음.** → Phase 2에서 신규 추가 필요.

---

## 3. 설계 결정

### 3.1 유휴 만료 저장소 선택

**옵션 A. DB `users.last_active_at` (권장)**
- 모든 보호된 요청에서 가드가 `SELECT last_active_at` → 1시간 경과 체크 → 실패 시 401, 성공 시 `UPDATE last_active_at = NOW()`.
- 장점: 세 시스템이 **동일한 users 테이블**을 공유하므로 어디서 활동했든 하나의 진실 소스. 위변조 불가능.
- 단점: 요청마다 DB 왕복. → **쓰기 throttling**(현재 값이 60초 이상 오래됐을 때만 UPDATE)으로 완화.

**옵션 B. 쿠키/localStorage 타임스탬프 + 서버 검증**
- 클라이언트가 `X-Last-Activity` 헤더 또는 쿠키에 타임스탬프를 담아 전송. 서버가 검증.
- 장점: 백엔드 상태 없음.
- 단점: 클라이언트가 값을 위조할 수 있음 → 보안상 무의미.

**옵션 C. Redis 세션 스토어**
- 장점: 빠른 유휴 검사 + TTL 자동 만료.
- 단점: 신규 인프라 필요.

**결정: 옵션 A 채택.** 공유 DB라는 전제 덕분에 자연스럽고, throttling으로 비용도 작다. Redis는 Phase 3 후보로 보류.

### 3.2 절대 만료 축소 + JWT 재발급(refresh) 정책
- **세 시스템의 JWT 절대 만료를 모두 `1h`로 통일**(기존 어드민 8h / 스튜디오·튜터링 7d → 모두 1h).
  - `JWT_EXPIRES_IN_ADMIN=1h`
  - 스튜디오·튜터링 `JWT_EXPIRES_IN=1h`
- 활동이 지속되는 사용자는 JWT가 1시간 안에 만료되므로, **가드가 유휴 검사를 통과할 때마다 JWT를 새로 발급**해 응답에 포함시킨다.
  - 응답 헤더: `X-Refresh-Token: <newJwt>` 또는 응답 바디에 `{ refreshToken }` 포함.
  - 프론트엔드: fetch 래퍼가 해당 헤더를 감지하면 localStorage 토큰을 교체(`storeAdminToken` / `storeToken`).
- 쓰기 throttling과 맞춰 **60초 이내에는 재발급도 스킵**해 네트워크/연산 비용 절감.
- 결과: **유휴 1h = JWT 1h = 쿠키 Max-Age 1h** — 세 값이 일치하며 활동 시 함께 연장된다.

### 3.3 `lastActiveAt` 필드 신설 vs `lastLoginAt` 재활용
- **신설 권장.** `lastLoginAt`은 "가장 최근 로그인 시각" 의미로 이미 여러 UI에서 사용 중일 수 있고, 의미 혼선을 피하기 위해 분리.
- 마이그레이션: `users.last_active_at TIMESTAMPTZ NULL` 추가 + 인덱스는 불필요(권한 체크는 PK로 조회하므로).

### 3.4 쿠키 설계

| 항목 | 값 | 사유 |
|---|---|---|
| 이름 | `picklass_user_id` | 세 시스템 공통 규약 |
| 값 | `users.id` (UUID) | 요구사항 |
| Path | `/` | 전체 경로 |
| Max-Age | `3600` (1시간) | 유휴 만료와 일치. 활동 시 가드가 매번 연장 |
| HttpOnly | **true (권장, 확인 필요)** | 서버 식별용. XSS에 값 노출 방지. 클라이언트 JS에서 직접 읽을 필요가 없다면 true가 안전 |
| Secure | prod=`true`, dev=`false` | HTTPS 강제 |
| SameSite | `Lax` | 어드민 핸드오프는 다른 오리진에서 오는 top-level 이동이므로 Lax면 충분. Strict는 핸드오프 직후 쿠키가 따라오지 않을 수 있음 |
| Domain | 기본 (현재 호스트) | 세 사이트 도메인이 달라 공유 불가 — 각 사이트가 자기 도메인에만 설정 |

**미결정(사용자 확인 필요):**
- HttpOnly를 `true`로 둘지, 또는 프론트엔드에서 쿠키 값을 읽어야 할 유스케이스가 있는지?
- 서브도메인 통합(`*.picklass.com`)으로 운영될 예정이면 `Domain=.picklass.com`으로 설정해 크로스 사이트 공유 가능. 현재 vercel 임시 도메인 기준으로는 공유 불가.

### 3.5 쿠키 갱신 정책
- 로그인 성공 시: 쿠키 발급.
- 보호된 요청이 가드를 통과할 때마다: 응답에 `Set-Cookie`로 **Max-Age를 다시 3600으로 리셋**. 사용자가 계속 활동하면 쿠키 만료 시점도 계속 밀린다.
- 1시간 무활동 시: 쿠키가 자연 만료, 동시에 DB `last_active_at` 체크가 401을 내면서 프론트엔드가 로그아웃.
- 로그아웃 API: `Max-Age=0`으로 쿠키 삭제.

### 3.6 핸드오프 토큰은 어떻게 처리?
- 핸드오프 토큰(60초 단명) 수신 시점에도 `last_active_at`은 **갱신**되어야 한다. 그렇지 않으면 어드민이 학생으로 진입한 직후 바로 유휴 만료가 걸릴 수 있음.
- 즉, 핸드오프 수신 측(studio/tutoring `/auth/me`)의 첫 검증 시 `last_active_at = NOW()`.
- 핸드오프 토큰 자체의 유휴 규칙은 없음 — 60초 절대 만료가 유일한 제약.

---

## 4. 작업 범위

### 4.1 Prisma 스키마 + 마이그레이션 (picklass-backoffice)
- [prisma/schema.prisma](prisma/schema.prisma) — `User` 모델에 `lastActiveAt DateTime? @map("last_active_at") @db.Timestamptz` 추가.
- **`pnpm prisma db push`** 로 Supabase 운영 DB에 직접 반영(마이그레이션 히스토리 파일 생성 없음).
- studio/tutoring의 `apps/api/prisma/schema.prisma`에도 동일 컬럼 반영(읽기 전용이지만 타입 일치를 위해) + 각자 `db push`.

### 4.2 @repo/core 공통 로직 (picklass-backoffice)
- [packages/core/src/auth/auth.service.ts](packages/core/src/auth/auth.service.ts)
  - `login()` — 성공 시 `lastActiveAt = NOW()` 갱신(기존 `lastLoginAt`과 함께). 만료 `1h` JWT 발급.
  - `getMe()`, `verifyAdminToken()` 로직을 **비동기로 변경**(현재 `verifyAdminToken`은 동기). 가드에서 async 호출.
  - 신규 메서드: `async touchActivity(userId: string): Promise<{ ok: boolean; refreshToken?: string; user?: AuthUser }>`
    - `lastActiveAt` NULL이거나 `< 1h` 경과면 통과.
    - 60초 throttle 넘어가면 UPDATE + **새 JWT 발급**해서 `refreshToken` 반환.
- [apps/admin/backend/src/auth/admin-auth.guard.ts](apps/admin/backend/src/auth/admin-auth.guard.ts)
  - `canActivate` 내에서 `await authService.touchActivity(payload.id)` 호출.
  - 응답 객체에 **쿠키 설정**: `Set-Cookie: picklass_user_id=<id>; Path=/; Max-Age=3600; HttpOnly; SameSite=Lax; Secure(prod)`.
  - `refreshToken`이 있으면 응답 헤더 `X-Refresh-Token`에 실어 반환.

### 4.3 백오피스 백엔드 (admin-api)
- [apps/admin/backend/src/auth/auth.controller.ts](apps/admin/backend/src/auth/auth.controller.ts)
  - `POST /auth/login` — 응답에 쿠키 설정(admin 결과 + handoff 결과 모두 동일: `users.id`).
  - `POST /auth/logout` **신규 추가** — 쿠키 삭제(`Max-Age=0`) + DB `lastActiveAt = null` 처리.
  - `GET /auth/me` — 유휴 검사 통과 시 쿠키 Max-Age 갱신 + 필요 시 `X-Refresh-Token` 반환.

### 4.4 백오피스 프론트엔드
- 쿠키는 백엔드가 관리하므로 프론트엔드 추가 코드 최소.
- [apps/admin/frontend/src/lib/api.ts](apps/admin/frontend/src/lib/api.ts) `fetchApi`
  - 요청 시 `credentials: 'include'` 추가(쿠키 송수신).
  - 응답 헤더 `X-Refresh-Token` 감지 → `storeAdminToken(newToken)`으로 localStorage 토큰 교체.
- 기존 [admin-auth-gate.tsx](apps/admin/frontend/src/components/auth/admin-auth-gate.tsx)의 클라이언트 유휴 타이머는 **UX 목적으로 유지**(빠른 로그아웃 피드백). 실제 차단은 백엔드 가드가 한다.
- 로그아웃 버튼 → 신규 `POST /auth/logout` 호출 후 localStorage 정리 + `/login` 이동.

### 4.5 studio.picklass.com
- `.env`의 `JWT_EXPIRES_IN`: `7d` → `1h`.
- [apps/api/src/auth/auth.service.ts](../../../studio.picklass.com/apps/api/src/auth/auth.service.ts)
  - `login()` 성공 시 `lastActiveAt = NOW()`, JWT 1h 발급.
  - `touchActivity()` 신규 — 유휴 검사 + throttled UPDATE + 새 JWT 발급.
  - `getMe()` / 가드 진입 경로 — `touchActivity()` 호출.
- [apps/api/src/common/guards/auth.guard.ts](../../../studio.picklass.com/apps/api/src/common/guards/auth.guard.ts) — 보호된 모든 요청에서 유휴 검사 + 쿠키 설정 + `X-Refresh-Token` 반환.
- [apps/api/src/auth/auth.controller.ts](../../../studio.picklass.com/apps/api/src/auth/auth.controller.ts) — `/auth/login`, `/auth/me`에 쿠키 설정. `POST /auth/logout` 신설.
- CORS `credentials: true` 이미 설정됨 확인. 프론트 [lib/api.ts:81](../../../studio.picklass.com/apps/web/src/lib/api.ts#L81)에 `credentials: 'include'` 이미 있음. `X-Refresh-Token` 핸들링 추가 필요.

### 4.6 tutoring.picklass.com
- `.env`의 `JWT_EXPIRES_IN`: 현재 값 → `1h`.
- 스튜디오와 동일 작업 — `auth.service.ts`의 `login()`/`touchActivity()`/`getMe()`, 가드, `POST /auth/logout` 신설, 컨트롤러에 쿠키 설정.
- 백엔드 CORS `credentials: true` 확인(없으면 추가).
- 프론트엔드: [useAuth.ts](../../../tutoring.picklass.com/apps/web/src/lib/hooks/useAuth.ts) 및 기타 fetch 호출에 **`credentials: 'include'` 추가**(현재 없음).
- `X-Refresh-Token` 헤더 핸들링 추가.

### 4.7 환경 변수 (변경 + 신규)
**변경**:
| 변수 | 기존 | 신규 |
|---|---|---|
| `JWT_EXPIRES_IN_ADMIN` (백오피스) | `8h` | `1h` |
| `JWT_EXPIRES_IN` (스튜디오) | `7d` | `1h` |
| `JWT_EXPIRES_IN` (튜터링) | `7d` | `1h` |

**신규**:
| 변수 | 기본값 | 사용처 |
|---|---|---|
| `IDLE_TIMEOUT_SECONDS` | `3600` | 세 시스템 공통 — 유휴 만료 기준 |
| `ACTIVITY_WRITE_THROTTLE_SECONDS` | `60` | DB 쓰기 + JWT 재발급 throttle |
| `USER_ID_COOKIE_NAME` | `picklass_user_id` | 쿠키 이름 |
| `USER_ID_COOKIE_DOMAIN` | (비움 = 현재 호스트) | 현재는 사이트별 독립 — 추후 `.picklass.com` 통합 시 사용 |
| `USER_ID_COOKIE_SECURE` | `true` (prod) / `false` (dev) | HTTPS 강제 |

---

## 5. 유휴 검사 + throttled UPDATE + JWT 재발급 의사코드

```ts
async touchActivity(userId: string): Promise<{
  ok: boolean;
  refreshToken?: string;
  user?: AuthUser;
}> {
  const user = await prisma.user.findFirst({
    where: { id: userId, deletedAt: null },
    select: {
      id: true, userId: true, email: true, name: true,
      roleCode: true, institutionId: true,
      lastActiveAt: true, statusCode: true,
    },
  });
  if (!user || user.statusCode !== 'active') return { ok: false };

  const now = Date.now();
  const last = user.lastActiveAt?.getTime() ?? 0;
  const elapsed = now - last;

  // 유휴 만료 — last가 NULL이면 체크 스킵(마이그레이션 직후 호환)
  if (last > 0 && elapsed > IDLE_TIMEOUT_MS) {
    return { ok: false }; // 가드가 401 처리
  }

  // Throttle: 60초 이내는 DB UPDATE + JWT 재발급 모두 스킵
  if (elapsed > ACTIVITY_WRITE_THROTTLE_MS) {
    await prisma.user.update({
      where: { id: userId },
      data: { lastActiveAt: new Date() },
    });
    const refreshToken = this.signToken(toAuthUser(user), '1h');
    return { ok: true, refreshToken, user: toAuthUser(user) };
  }

  return { ok: true, user: toAuthUser(user) };
}
```

가드 쪽 처리:
```ts
async canActivate(context): Promise<boolean> {
  const req = context.switchToHttp().getRequest();
  const res = context.switchToHttp().getResponse();

  const token = extractBearer(req);
  const payload = await this.authService.verifyAdminToken(token); // async

  const result = await this.authService.touchActivity(payload.id);
  if (!result.ok) throw new UnauthorizedException('세션이 만료되었습니다.');

  // 쿠키 갱신
  res.cookie('picklass_user_id', payload.id, {
    path: '/',
    maxAge: 3600 * 1000,
    httpOnly: true,
    sameSite: 'lax',
    secure: process.env.NODE_ENV === 'production',
  });

  // 재발급 JWT 반환
  if (result.refreshToken) {
    res.setHeader('X-Refresh-Token', result.refreshToken);
  }

  req.adminUserId = payload.id;
  return true;
}
```

---

## 6. 영향 분석 / 호환성

1. **기존 로그인 세션 영향**: 기존 JWT는 그대로 유효하지만 `lastActiveAt`이 NULL이어서 첫 요청에서 NULL-case 처리가 필요. → 코드에서 "`last == 0`이면 유휴 체크 스킵"으로 처리(위 의사코드).
2. **DB 쓰기 부하**: throttle 60초 + GET /auth/me, 일반 API 요청 기준으로 한 사용자가 초당 ≤1회 UPDATE. 인덱스 불필요.
3. **쿠키 있음 vs 없음 병행**: 기존 localStorage JWT 인증은 그대로 유지. 쿠키는 **식별 보조 수단**이며 인증 여부는 여전히 JWT가 결정. 즉, Phase 2는 추가일 뿐 JWT 인증을 대체하지 않음.
4. **CORS / credentials**: 쿠키를 다른 오리진에 보내려면 `credentials: 'include'` + 백엔드 `Access-Control-Allow-Credentials: true` 필요. 현재 세 백엔드 모두 `credentials: true`로 CORS 설정돼 있음을 확인.
5. **SameSite=Lax의 제약**: 어드민이 "스튜디오 진입" 버튼으로 새 탭을 열 때 top-level GET이므로 Lax에서도 쿠키가 전달됨 → 핸드오프 수신 성공 후 쿠키 설정 정상 작동.

---

## 7. 검증 시나리오

1. **정상 로그인 → 쿠키 발급**: 세 사이트 로그인 후 브라우저 쿠키에 `picklass_user_id=<UUID>` 생성 확인.
2. **활동 시 쿠키 갱신**: 30분 사용 후에도 쿠키 만료 시각이 현재 시점 + 1h로 유지.
3. **1시간 무활동 → 401**: 유휴 1시간 경과 후 다음 API 호출에서 401 + 자동 로그아웃.
4. **Throttle 검증**: 연속 API 호출 시 DB `last_active_at` 업데이트가 60초 이내에는 스킵되는지 로그 확인.
5. **핸드오프 + 쿠키**: 어드민이 teacher 자동 로그인 → studio 도메인에 `picklass_user_id`=teacher의 UUID 쿠키 생성 확인.
6. **로그아웃**: `POST /auth/logout` 후 쿠키 삭제 및 localStorage 정리.
7. **절대 만료(백오피스 8h)**: 8시간 후 어드민 토큰 JWT 자체 만료로 로그아웃되는지 확인.
8. **NULL lastActiveAt 기존 사용자**: 마이그레이션 직후 첫 요청에서 401이 나지 않고 정상 작동.
9. **핸드오프 직후 lastActiveAt 초기화**: 튜터링 사이트에서 학생으로 첫 요청 시 `last_active_at`이 NOW로 세팅.
10. **악성 쿠키 변조**: 쿠키 `picklass_user_id`를 다른 UUID로 변조해도 JWT 검증이 여전히 진실의 원천이므로 인증이 뚫리지 않음을 확인(쿠키는 식별 보조일 뿐).

---

## 8. 결정 사항 (2026-04-11 확정)

1. **쿠키 HttpOnly**: **`true`**. 서버 식별 전용, 프론트엔드 JS에서 읽지 않음. XSS로 쿠키 값 유출 차단.
2. **쿠키 Domain**: **현 시점에는 각 사이트가 자기 도메인에만 설정**(공유 없음). `*.picklass.com` 서브도메인 통합은 **추후 작업**으로 이관 → §9 "추후 과제"에 명시.
3. **기존 JWT 절대 만료 축소**: 스튜디오/튜터링의 `JWT_EXPIRES_IN`을 **`7d` → `1h`** 로 축소. 백오피스 어드민 토큰 `JWT_EXPIRES_IN_ADMIN`도 **`8h` → `1h`** 로 축소. 유휴 1h 정책과 일치시켜 JWT 도용 위험 최소화.
    - 활동이 지속되는 사용자는 쿠키 Max-Age 갱신만으로는 JWT 절대 만료를 피할 수 없으므로, **가드에서 활동 갱신 시 JWT도 재발급**(refresh)하여 응답 헤더/바디로 돌려주고 프론트엔드 localStorage를 갱신하도록 한다. (§5 의사코드 + §4에 반영.)
4. **`/auth/logout` 엔드포인트**: **신설**. 세 시스템 각각 `POST /auth/logout` 추가. 쿠키 삭제(`Max-Age=0`) + 필요 시 DB `lastActiveAt = null` 처리.
5. **마이그레이션 방식**: **`prisma db push`** 사용. Supabase 운영 DB에 직접 반영.

---

## 8-1. 구현 결과 (2026-04-11)

### 실제 추가/수정된 파일

**Prisma 스키마**
- [prisma/schema.prisma](prisma/schema.prisma) — `User` 모델에 `lastActiveAt` 컬럼 추가.
- `prisma db push`는 Supabase의 `auth.users` 크로스 스키마 참조 때문에 introspection에서 실패 → `prisma db execute`로 **raw SQL** (`ALTER TABLE public.users ADD COLUMN IF NOT EXISTS last_active_at TIMESTAMPTZ NULL`) 직접 실행. 스크립트: [prisma/add-last-active-at.sql](prisma/add-last-active-at.sql).
- studio/tutoring의 `apps/api/prisma/schema.prisma`에도 동일 필드 반영(읽기 전용 mirror).

**@repo/core**
- [packages/core/src/auth/auth.service.ts](packages/core/src/auth/auth.service.ts)
  - `login()` — 성공 시 `lastLoginAt` + `lastActiveAt` 둘 다 NOW. 어드민 JWT 기본 만료 **8h → 1h** 로 축소.
  - `verifyAdminToken()` — async 전환.
  - `touchActivity()` 신규 — `lastActiveAt` 조회 → 1시간 유휴 체크 → 60초 throttle 초과 시 UPDATE + 새 JWT 발급.
  - `getMe()` — `touchActivity()` 기반으로 재작성.
  - `logout()` 신규 — `lastActiveAt = null`.
  - `IDLE_TIMEOUT_SECONDS`, `ACTIVITY_WRITE_THROTTLE_SECONDS` 환경 변수 로드.

**picklass-backoffice admin backend**
- [apps/admin/backend/src/auth/cookie.util.ts](apps/admin/backend/src/auth/cookie.util.ts) — 신규. `applyUserIdCookie(res, userId)` / `clearUserIdCookie(res)`. `HttpOnly=true`, `SameSite=lax`, Max-Age=`IDLE_TIMEOUT_SECONDS`.
- [apps/admin/backend/src/auth/admin-auth.guard.ts](apps/admin/backend/src/auth/admin-auth.guard.ts) — `canActivate` async화. `verifyAdminToken` → `touchActivity` → 실패 시 401. 성공 시 응답에 `picklass_user_id` 쿠키 설정 + `X-Refresh-Token` 헤더 + `Access-Control-Expose-Headers` 추가.
- [apps/admin/backend/src/auth/auth.controller.ts](apps/admin/backend/src/auth/auth.controller.ts)
  - `/auth/login` — 로그인 성공 시 쿠키 설정.
  - `/auth/me` — 쿠키 갱신.
  - `/auth/logout` **신규** — 쿠키 삭제 + `lastActiveAt=null`. 만료된 토큰으로 호출해도 성공.

**picklass-backoffice admin frontend**
- [apps/admin/frontend/src/lib/api.ts](apps/admin/frontend/src/lib/api.ts) — `fetchApi`에 `credentials: 'include'` 추가 + `X-Refresh-Token` 헤더 감지해 `storeAdminToken()`으로 교체.
- [apps/admin/frontend/src/lib/api/auth.ts](apps/admin/frontend/src/lib/api/auth.ts) — `apiLogout()` 추가.
- [apps/admin/frontend/src/components/layout/app-navbar.tsx](apps/admin/frontend/src/components/layout/app-navbar.tsx) — 로그아웃 링크를 버튼으로 교체, `apiLogout()` → `useAuthStore.logout()` → `/login` 이동.

**studio.picklass.com api**
- [apps/api/src/auth/auth.service.ts](../../../studio.picklass.com/apps/api/src/auth/auth.service.ts) — 기본 만료 `7d → 1h`. `touchActivity()` / `logout()` 신규. `login()`이 `last_active_at` 갱신.
- [apps/api/src/common/utils/cookie.util.ts](../../../studio.picklass.com/apps/api/src/common/utils/cookie.util.ts) — 신규.
- [apps/api/src/common/guards/auth.guard.ts](../../../studio.picklass.com/apps/api/src/common/guards/auth.guard.ts) — `touchActivity` 기반 유휴 검사 + 쿠키/헤더.
- [apps/api/src/auth/auth.controller.ts](../../../studio.picklass.com/apps/api/src/auth/auth.controller.ts) — `signup`/`login`/`me`에 쿠키. `/auth/logout` 신규.
- [apps/api/src/main.ts](../../../studio.picklass.com/apps/api/src/main.ts) — CORS `exposedHeaders: ['X-Refresh-Token']`.

**studio.picklass.com web**
- [apps/web/src/lib/api.ts](../../../studio.picklass.com/apps/web/src/lib/api.ts) — `request()`가 `X-Refresh-Token` 감지해 `storeToken()`. `authApi.logout()` 추가.
- [apps/web/src/components/AuthProvider.tsx](../../../studio.picklass.com/apps/web/src/components/AuthProvider.tsx) — `logout`이 서버 `/auth/logout` 호출 후 로컬 정리.
- [apps/web/src/hooks/use-auth.ts](../../../studio.picklass.com/apps/web/src/hooks/use-auth.ts) — `logout` 타입을 `Promise<void>`로.

**tutoring.picklass.com api**
- [apps/api/src/auth/auth.service.ts](../../../tutoring.picklass.com/apps/api/src/auth/auth.service.ts) — studio와 동일한 `touchActivity`/`logout` + 1h.
- [apps/api/src/common/utils/cookie.util.ts](../../../tutoring.picklass.com/apps/api/src/common/utils/cookie.util.ts) — 신규.
- [apps/api/src/auth/auth.guard.ts](../../../tutoring.picklass.com/apps/api/src/auth/auth.guard.ts) — `touchActivity` 기반 async guard.
- [apps/api/src/auth/auth.controller.ts](../../../tutoring.picklass.com/apps/api/src/auth/auth.controller.ts) — 쿠키 설정 + `/auth/logout` 신설.
- [apps/api/src/main.ts](../../../tutoring.picklass.com/apps/api/src/main.ts) — CORS `exposedHeaders` 추가.
- `package.json`에 `@types/express` devDependency 추가(`res.cookie` 타입 지원).

**tutoring.picklass.com web**
- [apps/web/src/lib/authFetch.ts](../../../tutoring.picklass.com/apps/web/src/lib/authFetch.ts) — **신규 공용 fetch 래퍼**. `credentials: 'include'` + Authorization 자동 첨부 + `X-Refresh-Token` 감지 → localStorage 교체.
- [apps/web/src/lib/hooks/useAuth.ts](../../../tutoring.picklass.com/apps/web/src/lib/hooks/useAuth.ts) — `login`/`signup`/`checkAuth`/`logout` 전부 `authFetch` 사용. `logout`이 서버 `/auth/logout` 호출 후 로컬 정리. 타입 `Promise<void>`로 변경.
- [apps/web/src/app/page.tsx](../../../tutoring.picklass.com/apps/web/src/app/page.tsx), [apps/web/src/lib/services/lessonService.ts](../../../tutoring.picklass.com/apps/web/src/lib/services/lessonService.ts), [apps/web/src/app/class/lesson-setup/custom/page.tsx](../../../tutoring.picklass.com/apps/web/src/app/class/lesson-setup/custom/page.tsx), [apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx](../../../tutoring.picklass.com/apps/web/src/app/modules/[lessonId]/_components/LessonSession.tsx), [apps/web/src/components/oizi/AccessCodeModal.tsx](../../../tutoring.picklass.com/apps/web/src/components/oizi/AccessCodeModal.tsx) — 기존 `fetch` + 수동 Bearer 헤더를 `authFetch`로 전환.

**환경 변수 (세 프로젝트 `.env`)**
| 변수 | 값 |
|---|---|
| `JWT_EXPIRES_IN_ADMIN` / `JWT_EXPIRES_IN` | `1h` (백오피스 어드민 전용 / 나머지 두 api 공통) |
| `JWT_EXPIRES_IN_HANDOFF` | `60s` (백오피스만) |
| `IDLE_TIMEOUT_SECONDS` | `3600` |
| `ACTIVITY_WRITE_THROTTLE_SECONDS` | `60` |
| `USER_ID_COOKIE_NAME` | `picklass_user_id` |
| `USER_ID_COOKIE_SECURE` | `false` (dev). 운영 Vercel 배포 시 `true`로 설정. |

수정된 `.env` 파일:
- [apps/admin/backend/.env](apps/admin/backend/.env), [apps/admin/backend/.env.local](apps/admin/backend/.env.local)
- `d:/Project/studio.picklass.com/apps/api/.env`
- `d:/Project/tutoring.picklass.com/apps/api/.env`

### 빌드/타입체크 결과
- `@repo/core` `tsc` ✅
- admin backend `nest build` ✅
- admin frontend `next build` ✅ (34 라우트)
- studio api `nest build` ✅
- studio web `tsc --noEmit` ✅
- tutoring api `tsc --noEmit` ✅ (`@types/express` 추가 후)
- tutoring web `tsc --noEmit` ✅

### 구현 중 변경된 설계 사항
1. **`prisma db push` → `prisma db execute + raw SQL`**. Supabase 공급자가 `public.async_tasks → auth.users` 크로스 스키마 FK를 가지고 있어 `db push` introspection이 실패. 단일 컬럼 추가였으므로 raw `ALTER TABLE`로 안전하게 처리.
2. **튜터링 프론트엔드에 `authFetch` 래퍼 신설**. 기존 코드 곳곳에서 `fetch` + 수동 Bearer 헤더를 반복하고 있어, `X-Refresh-Token` 감지와 `credentials: 'include'`를 매번 구현하기보다 공용 래퍼로 일괄 전환. 결과적으로 튜터링의 모든 보호 API 호출이 한 경로를 통과하도록 정리됨.
3. **쿠키 SameSite=Lax 고정**. 운영에선 `Secure=true` + `Lax`로 동작. 어드민 핸드오프는 top-level navigation이므로 Lax에서도 쿠키가 동반 전송됨.

### 런타임 E2E 검증 (미수행 — 운영자 확인 필요)
§7의 10가지 시나리오는 코드 레벨에서만 검증됨. 로컬 DB에 시스템 어드민/강사/학생 테스트 계정으로 수동 확인 필요.

---

## 8-2. 구현 후 발견된 오류 및 처리 내역 (2026-04-11)

Phase 2 구현 직후 로컬 dev 환경에서 발견된 문제들과 그 대응을 시간 순서로 기록한다.

### 오류 #1 — 로그인 실패 시 프론트엔드에 "Internal server error"가 표시됨

**증상**
- 잘못된 아이디로 로그인 시도 → 프론트 로그인 모달/페이지에 "Internal server error" 텍스트가 노출.
- 백엔드 로그는 정상:
  ```
  [Nest] ERROR [ExceptionsHandler] UnauthorizedException: 계정을 찾을 수 없습니다.
      at AuthService.login (…\dist\main.js:92223:19)
    status: 401
  ```
  → 백엔드는 401 + "계정을 찾을 수 없습니다." 응답을 내고 있었음에도 화면에는 다른 문자열이 표시됨.

**원인 분석 (가설 검토)**
1. **프론트 에러 파싱 문제 가능성**: `fetchApi`가 에러 응답 본문에서 `message` 필드를 제대로 꺼내지 못하거나, `instanceof ApiError` 분기에서 떨어져 일반 폴백 문구로 가는 경우. 당시 코드의 `response.clone().json().catch(() => null)` 로직이 환경에 따라 본문 더블 리드로 꼬이는 가능성 있음.
2. **NestJS 내부 예외 필터의 일관성 문제 가능성**: `UnauthorizedException`은 401로 가지만, 서비스-컨트롤러 사이 어딘가에서 예상 외 `Error`가 던져지면 기본 필터가 500 + "Internal server error" 본문을 내보냄(이 문자열이 NestJS 디폴트 500 body와 정확히 일치).
3. **CORS + credentials 불일치 가능성**: Phase 2에서 `credentials: 'include'`를 추가했는데, 프론트 origin이 CORS 화이트리스트에서 정확히 매칭되지 않으면 브라우저가 응답 본문을 읽지 못해 프론트 폴백 텍스트가 노출.
4. **`X-Refresh-Token` 미노출**: 백엔드 CORS `exposedHeaders`에 `X-Refresh-Token`이 없으면 브라우저가 그 헤더를 읽지 못해 `fetchApi`의 재발급 로직이 꼬일 가능성(로그인 응답에는 해당 없음, 그러나 전체 파이프라인 일관성을 위해 수정 필요).

**적용한 대응**

| # | 수정 대상 | 수정 내용 |
|---|---|---|
| 1 | [apps/admin/frontend/src/lib/api.ts](apps/admin/frontend/src/lib/api.ts) | 에러 응답 본문을 **텍스트로 한 번만 읽고** `JSON.parse` 시도. `message`(문자열/배열) → `error` → 원문 텍스트 → `API Error: N` 순 폴백. 에러 시 `console.error('[fetchApi] error response', { url, status, rawText, message })`로 디버그 로그 출력. |
| 2 | [apps/admin/frontend/src/components/landing/login-modal.tsx](apps/admin/frontend/src/components/landing/login-modal.tsx), [apps/admin/frontend/src/app/login/page.tsx](apps/admin/frontend/src/app/login/page.tsx) | `ApiError`가 아니어도 `Error` 인스턴스면 `err.message`를 그대로 노출. 최종 폴백만 "로그인에 실패했습니다."로 유지. |
| 3 | [apps/admin/backend/src/common/filters/all-exceptions.filter.ts](apps/admin/backend/src/common/filters/all-exceptions.filter.ts) (신규) + [main.ts](apps/admin/backend/src/main.ts) | **Global `AllExceptionsFilter`** 신설. `HttpException`의 `message`/`error`/`status`를 일관된 JSON `{ statusCode, error, message }`로 반환. 예상 외 `Error`도 500 + 해당 에러의 `message`로 직렬화해 "Internal server error" 디폴트 문자열이 절대 단독 노출되지 않도록 차단. 요청 메서드/URL/상태/메시지를 한 줄 로그. |
| 4 | [apps/admin/backend/src/main.ts](apps/admin/backend/src/main.ts) | CORS `exposedHeaders: ['X-Refresh-Token']` 누락분 추가. Phase 2의 리프레시 헤더를 브라우저가 실제로 읽을 수 있도록 함. |

**학습 / 재발 방지**
- 프론트 fetch 래퍼의 에러 본문 파싱은 **텍스트 단일 리드 + JSON.parse try/catch** 패턴을 기본으로 한다. `response.clone()`은 브라우저별 구현 차이가 있을 수 있어 단순화.
- NestJS 프로젝트는 가능한 경우 **Global ExceptionFilter**를 초기부터 두어, HttpException / 비-HttpException 모두 일관된 응답 스키마를 보장한다.
- CORS `credentials: true`를 쓰는 프로젝트는 `exposedHeaders`를 누락하지 않는다 — 응답 헤더를 JS에서 읽어야 하는 모든 경우에 필수.

**검증 필요 사항 (사용자 재현 요청)**
위 수정 이후에도 "Internal server error"가 노출되면 다음 정보를 수집한다:
1. 브라우저 DevTools Console — 새로 추가된 `[fetchApi] error response`의 `status`, `rawText`, `message` 값.
2. DevTools Network 탭 — `/api/auth/login` 요청의 Response status code, Response body (Preview), Response headers (`Access-Control-Allow-Credentials`, `Access-Control-Allow-Origin`).

이 정보로 원인이 백엔드 500 응답 vs CORS 차단 vs 프론트엔드 폴백인지를 즉시 식별 가능.

### 오류 #2 — `prisma db push` 실패 (크로스 스키마 참조)

**증상**
```
Error: P4002
The schema of the introspected database was inconsistent: Cross schema references
are only allowed when the target schema is listed in the schemas property of your
datasource. `public.async_tasks` points to `auth.users` in constraint
`async_tasks_user_id_fkey`. Please add `auth` to your `schemas` property.
```

**원인**
- Supabase 운영 DB의 `public.async_tasks` 테이블이 `auth.users`(Supabase 내장 인증 스키마)를 FK로 참조.
- `prisma db push`는 대상 DB를 introspect한 뒤 사용자 스키마와 비교하는데, 이 과정에서 크로스 스키마 참조를 감지하고 거절.
- `prisma/schema.prisma`의 `datasource db`에 `schemas = ["public", "auth"]`를 추가할 수도 있으나, 그러면 `auth` 스키마 전체가 Prisma 관리 영역에 들어와 의도치 않은 마이그레이션 리스크가 생김.

**적용한 대응**
- `prisma db push` 포기, **`prisma db execute` + raw SQL** 경로로 전환.
- 신규 스크립트 [prisma/add-last-active-at.sql](prisma/add-last-active-at.sql):
  ```sql
  ALTER TABLE public.users
    ADD COLUMN IF NOT EXISTS last_active_at TIMESTAMPTZ NULL;
  ```
- 실행 명령:
  ```
  DIRECT_URL=postgresql://... pnpm prisma db execute \
    --file prisma/add-last-active-at.sql --schema prisma/schema.prisma
  ```
- 이후 `pnpm prisma generate`로 client만 별도 재생성.

**학습 / 재발 방지**
- Supabase 기반 프로젝트에서 단일 컬럼 추가/삭제 같은 좁은 변경은 **raw SQL + `prisma db execute`** 를 우선 고려. `db push`/`migrate dev`는 Supabase의 `auth`/`storage` 내장 스키마와 충돌 가능성이 있다.
- 본 저장소는 앞으로도 이 제약을 만날 것이므로, 스키마 변경 시마다 `prisma/` 하위에 타임스탬프 SQL 스크립트를 남기는 것을 권장.

### 오류 #3 — 튜터링 API `tsc --noEmit` 실패 (`Cannot find module 'express'`)

**증상**
```
src/auth/auth.controller.ts(2,31): error TS2307: Cannot find module 'express' or its corresponding type declarations.
src/auth/auth.guard.ts(2,31): error TS2307: Cannot find module 'express'.
src/common/utils/cookie.util.ts(1,31): error TS2307: Cannot find module 'express'.
```

**원인**
- 튜터링 API 프로젝트의 `package.json` devDependencies에 `@types/express`가 없었음.
- Phase 2에서 Express의 `Response` 타입을 import하는 코드(`res.cookie()`, `res.setHeader()`)를 신규로 추가했기 때문에 타입 의존성이 새로 필요.
- 스튜디오 API는 `@types/express` 보유 중이었음(기존 `Request` 타입 사용).

**적용한 대응**
- [d:/Project/tutoring.picklass.com/apps/api/package.json](../../../tutoring.picklass.com/apps/api/package.json) devDependencies에 `"@types/express": "^5.0.6"` 추가.
- `pnpm install` 재실행 후 `tsc --noEmit` 통과 확인.

**학습 / 재발 방지**
- NestJS 프로젝트에 `@nestjs/platform-express`가 있더라도 `@types/express`는 **독립적으로 선언**해야 한다(transitive로 노출되지 않음).
- 신규로 NestJS 프로젝트의 auth/guard 층을 추가할 때는 체크리스트에 "Express 타입 import 필요성" 항목을 둔다.

### 오류 #4 — `pnpm install` postinstall 단계의 `prisma generate` 파일 잠금 실패

**증상**
```
. postinstall: Error:
. postinstall: EPERM: operation not permitted, rename
    '…\.prisma\client\query_engine-windows.dll.node.tmp22748'
    -> '…\.prisma\client\query_engine-windows.dll.node'
ELIFECYCLE  Command failed with exit code 1.
```

**원인**
- 백엔드 dev 서버(`nest start --watch`)가 이전 prisma client의 `query_engine-windows.dll.node`를 잠그고 있는 상태에서 `pnpm install`이 postinstall로 `prisma generate`를 돌려 같은 파일을 rename하려다 Windows 파일 잠금(`EPERM`)에 걸림.

**영향**
- 치명적이지 않음: 기존 generated client가 그대로 유지되므로 backend/frontend 빌드에는 영향 없음.
- 단, `pnpm install` 자체는 non-zero exit로 종료.

**적용한 대응**
- 이번 세션에서는 건너뛰고 각 패키지별 빌드(`pnpm --filter @repo/core run build` 등)로 진행.
- 정석 대응은 백엔드 dev 서버를 일시 종료한 뒤 `pnpm install` 또는 `pnpm prisma generate`를 재실행.

**학습 / 재발 방지**
- Windows + Prisma 환경에서는 `pnpm install`이나 `prisma generate` 전에 backend dev 서버를 종료하는 것을 워크플로우에 포함.
- CI 환경(Linux)에서는 이 문제가 발생하지 않음 — 로컬 Windows 한정 문제.

### 오류 #5 — 첫 Phase 2 구현 시 `LessonSession.tsx` 동적 import 중괄호 불일치 (경미)

**증상**
- 튜터링 웹의 `LessonSession.tsx`에서 `saveLessonResult` 콜백을 `import('@/lib/authFetch').then(({ authFetch }) => authFetch(...))` 형태로 리팩토링하는 과정에서 `.catch()` 중괄호가 중첩돼 한 단계 누락 가능성 있었음.

**원인**
- 기존 `fetch(...).catch(console.error)` 구조를 `import().then(authFetch(...)).catch()`로 감싸 중첩되었는데, 내부/외부 체인의 닫는 중괄호를 수동으로 두 단계 맞춰야 했음.

**적용한 대응**
- 문법 수정 후 `pnpm run typecheck` 통과 확인 → TypeScript 레벨에서 검증 완료.

**학습 / 재발 방지**
- 이런 중첩 체이닝보다는 `async` IIFE 또는 별도 함수로 추출하는 것이 안전. 차후 `authFetch`를 정식 export로 두고 static import를 사용하도록 정리 예정(Phase 3 폴리싱 항목).

---

## 9. 추후 과제 (Phase 3 후보)

1. **서브도메인 통합 후 쿠키 공유**: 세 사이트를 `*.picklass.com` 하위로 정리한 뒤, `USER_ID_COOKIE_DOMAIN=.picklass.com`으로 설정해 한 번 로그인하면 세 사이트 모두 쿠키 상으로 식별 가능하게 한다. (JWT 인증 자체는 여전히 Bearer + localStorage 기반.)
2. **리프레시 토큰(Refresh Token) 도입**: 현재 Phase 2의 "활동 시 JWT 재발급" 방식은 단일 access token만 운영한다. 추후에는 표준 access/refresh 토큰 분리로 전환:
   - **Access token**: 짧은 수명(5~15분), 매 API 호출에 사용.
   - **Refresh token**: 긴 수명(7~30일), HttpOnly + Secure 쿠키로만 보관, `POST /auth/refresh` 전용.
   - 토큰 회전(rotation) + 재사용 탐지(detect reuse → 전체 세션 무효화) + DB/Redis에 `refresh_token_hash` 저장.
   - `lastActiveAt` 유휴 검사는 refresh 엔드포인트에서 수행.
   - 이 전환 시 Phase 2의 `X-Refresh-Token` 헤더 방식은 제거.
3. Redis 기반 세션 스토어로 `lastActiveAt` 이관 — DB 쓰기 throttling 없이 매 요청 처리.
4. 어드민 자동 로그인 감사 로그 테이블.
5. 2FA / IP 화이트리스트.
6. `isTempPassword=true` 강제 비밀번호 변경 플로우.

---

## 10. 단계별 실행 순서 (구현 시)

| 단계 | 작업 | 의존성 |
|---|---|---|
| 1 | 사용자 확인사항(§8) 해결 | - |
| 2 | Prisma 스키마에 `lastActiveAt` 추가 + 마이그레이션 | 1 |
| 3 | `@repo/core` AuthService에 `touchActivity()` 추가 + 기존 메서드 async 전환 | 2 |
| 4 | 백오피스 AdminAuthGuard에 유휴 검사 + 쿠키 Set 통합 | 3 |
| 5 | 백오피스 `POST /auth/logout` 신설 | 4 |
| 6 | studio auth.service / guard / controller에 동일 로직 이식 | 2 |
| 7 | tutoring auth.service / guard / controller에 동일 로직 이식 | 2 |
| 8 | studio/tutoring 프론트 fetch에 `credentials: 'include'` 보강(필요 시) | 6,7 |
| 9 | E2E 검증 (§7) | 5,6,7,8 |
| 10 | 선행 플랜 문서 업데이트 + 본 플랜에 구현 결과 추가 | 9 |
