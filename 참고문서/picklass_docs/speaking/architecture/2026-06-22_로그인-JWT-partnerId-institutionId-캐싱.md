# 로그인 인증 — partnerId · institutionId JWT 캐싱 설계

> **작성일**: 2026-06-22
> **서비스**: speaking.picklass.com
> **유형**: 아키텍처 결정 (인증)
> **연관 문서**:
> - [learn 페이지 실 API 연동 개발계획 §9](../features/learn페이지-실API연동/20260619_learn페이지_실API연동_개발계획.md) — 수강 소스 분기(파트너 판정)
> - [learn-explore 카테고리 개발계획 §10.1](../features/learn페이지-실API연동/20260620_learn-explore-카테고리_개발계획.md) — 기관 계층 다단계 오류 기록

---

## 1. 배경 및 문제

learn/explore/수강내역 등 여러 기능이 **파트너 관리 사용자 여부**(그리고 `partnerId`)를 알아야 한다.
파트너 판정은 기관 계층을 거슬러 올라가 `type='partner'` 조상을 찾는 **재귀 CTE**다
(계층이 `institution → group → partner` 다단계이기 때문 — explore §10.1 참조).

### 1.1 현재 인증 흐름

| 단계 | 동작 | 파일 |
|------|------|------|
| 토큰 발급 | JWT payload = `{ userId, email, roleCode }` (**institutionId 미포함**) | `auth.service.ts:201` `generateToken()` |
| 요청 검증 | `verifyToken()` → payload 복원 | `auth.guard.ts:20` |
| 사용자 해석 | `touchActivity(userId)` → **users SELECT** (idle/status 검증 + institutionId 획득) | `auth.service.ts:144` |
| 주입 | `request.user = { ...payload, ...activity.user }` | `auth.guard.ts:33` |
| 토큰 갱신 | 활동 throttle(기본 60s)마다 `refreshToken` 재발급 → `X-Refresh-Token` 헤더 | `auth.service.ts:165` |

> 핵심: **요청당 users SELECT는 이미 일어난다.** `touchActivity`가 idle timeout·status_code 검증을
> 위해 user 행을 반드시 읽기 때문이다. 따라서 `institutionId`는 그 SELECT에서 이미 무료로 얻는다.

### 1.2 문제

파트너 판정을 **요청당** 수행하면, 이미 있는 users SELECT 위에 **재귀 CTE가 매 요청 추가**된다.
learn/explore는 진입 빈도가 높아 불필요한 DB 부하가 누적된다.

기관→파트너 관계는 **거의 변하지 않는 조직 구조**이므로, 요청마다 재계산할 이유가 없다.

---

## 2. 결정

**로그인(토큰 발급) 시 `institutionId` 와 `partnerId` 를 1회 리졸브해 JWT 에 담는다.**

- `partnerId`: 재귀 CTE로 `type='partner'` 조상 1회 해석 (없으면 `null` = 픽클래스 기본 B2B).
- `institutionId`: 토큰에 함께 실어 토큰만으로 사용자 컨텍스트가 완결되게 한다.
- 이후 요청은 `req.user.partnerId` / `req.user.institutionId` 를 **추가 쿼리 없이** 사용한다.

> 캐싱 대상은 **판정 결과(partnerId)와 institutionId 뿐**이다. 파트너 **수강 과정 데이터**(외부 API)는
> 동적이므로 **토큰에 담지 않는다** — 별도 단기 캐시로 처리한다(learn 개발계획 §10.4).

---

## 3. 토큰 페이로드 변경

```ts
// 현재
{ userId, email, roleCode }

// 변경 후
{ userId, email, roleCode, institutionId, partnerId }
//                          string|null    string|null
```

`verifyToken()` 반환 타입도 동일하게 확장한다.

---

## 4. 상세 설계

### 4.1 파트너 리졸브 헬퍼 (재사용)

explore에서 검증된 재귀 CTE를 인증에서도 쓴다. 공통 헬퍼로 둔다.

```ts
// 기관 → type='partner' 조상 partnerId (없으면 null)
async function resolvePartnerId(prisma, institutionId: string | null): Promise<string | null> {
  if (!institutionId) return null;
  const rows = await prisma.$queryRawUnsafe<{ id: string }[]>(
    `WITH RECURSIVE ancestry AS (
       SELECT id, parent_id, type, 0 AS lvl FROM institutions WHERE id = $1::uuid
       UNION ALL
       SELECT i.id, i.parent_id, i.type, a.lvl + 1
       FROM institutions i JOIN ancestry a ON i.id = a.parent_id)
     SELECT id FROM ancestry WHERE type = 'partner' ORDER BY lvl LIMIT 1`,
    institutionId,
  );
  return rows[0]?.id ?? null;
}
```

### 4.2 `generateToken()` 시그니처 확장

```ts
private generateToken(
  userId: string, email: string | null, roleCode: string,
  institutionId: string | null, partnerId: string | null,
): string {
  return jwt.sign(
    { userId, email, roleCode, institutionId, partnerId },
    this.jwtSecret, { expiresIn: this.jwtExpiresIn },
  );
}
```

### 4.3 발급 지점별 처리 (3곳)

| 지점 | institutionId 출처 | partnerId 처리 |
|------|-------------------|----------------|
| `signup()` | 신규 user (보통 institution 없음) | `resolvePartnerId()` (대개 null) |
| `login()` | login SELECT 의 user 행 | `resolvePartnerId(institutionId)` **1회** |
| `touchActivity()` 갱신 | user SELECT 행 | **§4.4 갱신 전략 참조** |

### 4.4 토큰 갱신(refresh) 전략 — "로그인 시 1회" 충실도

`touchActivity`는 활동 throttle(60s)마다 토큰을 재발급한다. 여기서 partnerId 를 어떻게 채울지 선택:

- **(권장) carry-forward**: 갱신 시 **재계산하지 않고**, 들어온(검증된) 토큰의 `partnerId` 를
  그대로 새 토큰에 복사한다. → CTE 는 **로그인/회원가입 때 딱 1회**만 실행된다.
  - 이를 위해 guard 가 `touchActivity(payload.userId, payload.partnerId, payload.institutionId)` 처럼
    기존 payload 값을 넘겨, 갱신 토큰에 그대로 싣는다.
  - staleness: 다음 **완전 로그인** 전까지 고정. 조직 구조는 거의 안 바뀌므로 허용 가능.
- (대안) recompute-on-refresh: 갱신마다 CTE 재실행(~60s 주기). 더 최신이나 "1회" 취지엔 약함.

> 본 설계는 **carry-forward** 를 채택한다 (요청은 물론 갱신에서도 CTE 미실행).

### 4.5 guard 주입 우선순위

```ts
request.user = { ...payload, ...activity.user };
```

- `partnerId`: `activity.user`(DB)엔 없으므로 **토큰 payload 값**이 그대로 쓰인다. ✅
- `institutionId`: 토큰·DB 양쪽에 존재. 현재 머지 규칙상 **DB(activity.user) 값이 우선**(항상 최신).
  - 즉 institutionId 는 토큰에 실어도 실효값은 DB 기준 → 갱신 지연 없음.
  - partnerId 는 institutionId 기반이므로, institutionId 가 DB에서 바뀌면 partnerId(토큰)는
    carry-forward 로 잠깐 어긋날 수 있다 → §6 staleness 참조.

### 4.6 `RequestUser` 타입 확장

```ts
// common/types/request.types.ts
export interface RequestUser {
  id: string; userId: string; email: string | null; name: string;
  roleCode: string;
  institutionId: string | null;
  partnerId: string | null;   // 추가
}
```

---

## 5. 다운스트림 영향 (이 설계의 이득)

- **course-categories** ✅ **적용 완료(2026-06-22)**: `getCategoriesForInstitution(institutionId)` →
  `getCategoriesForPartner(req.user.partnerId)` 로 변경, 서비스 내 재귀 CTE 제거 → 요청당 단일
  카테고리 쿼리만 수행.
- **learn 수강 분기** (개발계획 §9.1, 차후): `req.user.partnerId` 유무로 즉시 Provider 분기 → 요청당 판정 없음.
- 파트너 외부 수강내역 Provider(차후): partnerId 가 이미 컨텍스트에 있으므로 바로 사용.

---

## 6. Staleness 분석 및 완화

| 변경 사건 | 반영 시점 | 평가 |
|-----------|-----------|------|
| 사용자의 기관이 파트너 산하로 재배치 | **다음 완전 로그인** | 조직 구조 변경은 드묾 → 허용 |
| 파트너 신규 설정/해제 | 다음 완전 로그인 | 허용 |
| 계정 비활성/탈퇴 | 즉시 (`touchActivity` status_code 검증) | 영향 없음 (별도 경로) |

완화책:
- 운영상 즉시 반영이 필요하면 **강제 재로그인**(토큰 무효화)으로 해소.
- 필요 시 §4.4 를 recompute-on-refresh 로 전환하면 최대 ~60s 로 단축 가능(트레이드오프 존재).

---

## 7. 변경 파일 목록

| 파일 | 변경 |
|------|------|
| `apps/api/src/auth/auth.service.ts` | `generateToken` 시그니처 확장, `verifyToken` 반환 타입 확장, `resolvePartnerId` 헬퍼, signup/login/touchActivity 발급부 수정 |
| `apps/api/src/auth/auth.guard.ts` | `touchActivity` 에 payload(partnerId/institutionId) 전달(carry-forward), 머지 유지 |
| `apps/api/src/common/types/request.types.ts` | `RequestUser.partnerId` 추가 |
| `course-categories.service.ts` / `.controller.ts` | ✅ `getCategoriesForPartner(partnerId)` 로 변경, 재귀 CTE 제거 (req.user.partnerId 사용) |
| (다운스트림, 차후) learn 분기 | `req.user.partnerId` 사용으로 요청당 판정 제거 |

---

## 8. 작업 체크리스트

- [x] `resolvePartnerId()` 헬퍼 추가 (재귀 CTE)
- [x] `generateToken()` 에 institutionId·partnerId 파라미터 추가
- [x] `verifyToken()` 반환 타입 확장 (institutionId·partnerId, 구 토큰 undefined→null 호환)
- [x] `signup()` / `login()` 발급부 — partnerId 1회 리졸브
- [x] `touchActivity()` 갱신부 — carry-forward (재계산 안 함, institutionId 는 DB 최신값)
- [x] `auth.guard.ts` — touchActivity 에 payload.partnerId 전달
- [x] `RequestUser.partnerId` 추가
- [x] `pnpm --filter @speaking/api typecheck` / `lint` 통과
- [x] 로컬 검증: 파트너 기관(`00000000-…`, tim.choi) → 토큰 `partnerId=38a3bebb-…` 탑재 확인
- [x] 로컬 검증: 기본 B2B(institution null) → `partnerId = null` 확인
- [ ] 로컬 테스트: 실제 로그인 API + 60s 이상 활동 → refresh 토큰 partnerId 유지(carry-forward) 확인

> **구현 완료(2026-06-22)**: §4 설계대로 반영. carry-forward 채택 → CTE 는 login/signup 시 1회만 실행,
> refresh(touchActivity)·요청당 재계산 없음.

---

## 9. 의도적 제외 / 차후

| 항목 | 사유 |
|------|------|
| 파트너 외부 수강내역 데이터의 토큰 캐싱 | 동적 데이터 — 별도 단기 캐시(개발계획 §10.4) |
| handoff 토큰(어드민 자동 로그인) 경로의 partnerId | 별도 토큰 발급 경로 — 동일 원칙 적용 필요 시 후속 |
| institutionId 세션 중 변경 실시간 반영 | 드문 케이스, 강제 재로그인으로 해소 |
