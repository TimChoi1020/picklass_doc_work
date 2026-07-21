# 오류: course.service.ts — resolveUserScope에서 userId 컬럼 vs UUID 혼동

**발생일**: 2026-06-05
**서비스**: picklass-backoffice (`packages/core/src/course/course.service.ts`)
**심각도**: 높음 — 파트너/그룹/학원 관리자 로그인 시 과정 목록이 완전히 비어 보임

---

## 증상 (Symptom)

`partner_admin`(또는 `group_admin`, `academy_admin`)으로 로그인했을 때 **과정 목록 화면에 아무 데이터도 표시되지 않는다.**

- system_admin으로 로그인하면 정상 조회됨
- 엑세스코드·통계 등 다른 화면은 정상
- 네트워크 탭에서 `/courses` 응답 `data: []`, `total: 0`

---

## 원인 (Root Cause)

`packages/core/src/course/course.service.ts` 의 `resolveUserScope` 메서드에서 Prisma 쿼리 조건 컬럼명이 잘못 사용됐다.

```ts
// ❌ 버그 코드 (수정 전)
const user = await this.prisma.user.findFirst({
  where: { userId, deletedAt: null },  // userId = 로그인 문자열 컬럼
  select: { institutionId: true },
});
```

`users` 테이블에는 두 개의 다른 필드가 있다:

| 필드 | 타입 | 의미 |
|------|------|------|
| `id` | UUID | Supabase 내부 기본키 |
| `userId` | string | 로그인 아이디 (예: `"admin01"`) |

JWT 페이로드를 서명할 때 `payload.userId = user.id`(UUID)를 사용한다:

```ts
// auth.service.ts — signToken
const payload: JwtPayload = {
  userId: user.id,  // ← UUID를 userId 키에 담음
  ...
};
```

그러므로 컨트롤러에서 `req.adminUserId`로 꺼낸 값은 **UUID**인데, `resolveUserScope`는 이 값을 `where: { userId }` (로그인 문자열 컬럼) 조건으로 검색했다. 당연히 매칭 레코드가 없어 `user = null` → `institutionId = null` → 스코프 `[]` 반환.

**스코프가 `[]`이면 발생하는 연쇄 효과:**

```
resolveUserScope() → []
intersectScopes([], selectionScope) → []
where.institutionId = { in: [] }
→ Prisma: WHERE institution_id IN () → 결과 0건
```

`system_admin`은 `resolveUserScope`가 즉시 `null`을 반환(`null` = 전체 허용)하므로 이 경로를 거치지 않아 정상 동작했다.

### 왜 다른 화면(엑세스코드 등)은 정상이었는가?

엑세스코드 컨트롤러는 `courseService.resolveUserScope`가 아니라 `institutionService.resolveUserScope`를 사용하고, 이 메서드는 처음부터 `where: { id: userId }` (UUID 컬럼)로 올바르게 구현되어 있었다.

```ts
// institution.service.ts — resolveUserScope (정상)
const user = await this.prisma.user.findFirst({
  where: { id: userId, deletedAt: null },  // ← UUID 컬럼 id 사용 ✓
  select: { institutionId: true },
});
```

`course.service.ts`에만 별도 구현된 `resolveUserScope`가 이 버그를 가지고 있었다.

---

## 해결 방법 (Resolution)

`packages/core/src/course/course.service.ts` 44번째 줄 수정:

```ts
// ✅ 수정 후
const user = await this.prisma.user.findFirst({
  where: { id: userId, deletedAt: null },  // userId → id
  select: { institutionId: true },
});
```

커밋 대상 파일: `packages/core/src/course/course.service.ts`

---

## 재발 방지 대책 (Prevention)

### 1. `userId` vs `id` 혼동 방지 규칙

`users` 테이블의 두 필드는 이름이 비슷해 혼동하기 쉽다. 코드에서 기본키 UUID를 다룰 때는 반드시 `id` 컬럼을 사용한다.

| 상황 | 사용할 컬럼 |
|------|------------|
| JWT 페이로드에서 꺼낸 사용자 식별자 | `id` (UUID) |
| 로그인 화면에서 입력받은 아이디 문자열 | `userId` |

### 2. 서비스마다 `resolveUserScope`를 중복 구현하지 않는다

`institutionService.resolveUserScope`가 이미 올바르게 구현되어 있다. 각 서비스(CourseService, AccessCodeService 등)는 자체 구현 대신 `InstitutionService`를 주입해 재사용해야 한다.

> **리팩터링 후보**: `course.service.ts`의 자체 `resolveUserScope`를 제거하고 `institutionService.resolveUserScope` 호출로 교체하면 동일 버그의 재발을 원천 차단할 수 있다.

### 3. 비-system_admin 계정으로 과정/코드 목록 수동 확인 필수

조직 스코프 로직 변경 시 `system_admin` 외에 **`partner_admin`, `group_admin`, `academy_admin`** 으로도 직접 로그인해 목록이 정상 조회되는지 확인한다. 스코프 버그는 system_admin 테스트만으로는 통과되는 특성이 있다.
