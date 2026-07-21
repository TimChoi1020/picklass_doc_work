# 오류: code.service.ts — saveGroupItems 트랜잭션 타임아웃

**발생일**: 2026-06-08  
**서비스**: picklass-backoffice (`packages/core/src/code/code.service.ts`)  
**심각도**: 높음 — 코드관리 페이지에서 저장 시 간헐적으로 500 에러 발생

---

## 증상 (Symptom)

백오피스 수업모듈 코드관리 페이지에서 항목을 저장할 때 아래 에러가 팝업으로 표시됨.

```
Invalid `prisma.codeItem.update()` invocation:

Transaction API error: Transaction not found. Transaction ID is invalid,
refers to an old closed transaction Prisma doesn't have information
about anymore, or was obtained before disconnecting.
```

- 로컬 개발 환경에서는 거의 재현되지 않음
- Vercel 배포 환경에서 간헐적으로 발생 (콜드 스타트 직후 더 빈번)
- 항목 수가 많은 그룹(KPI_CATEGORY 등)에서 더 자주 발생

---

## 원인 (Root Cause)

`saveGroupItems`가 트랜잭션 안에서 항목 수(N)만큼 `update` / `create`를 **순차 직렬** 실행했다.

```ts
// ❌ 버그 코드 (수정 전)
return this.prisma.$transaction(async (tx) => {
  await tx.codeItem.updateMany(...);  // soft-delete

  for (const item of items) {        // N번 순차 실행
    if (item.id) {
      await tx.codeItem.update(...);
    } else {
      await tx.codeItem.create(...);
    }
  }
});
```

Prisma 인터랙티브 트랜잭션의 기본 타임아웃은 **5초**다. 루프 안에서 쿼리가 많을수록 트랜잭션 점유 시간이 선형으로 증가한다.

Vercel 서버리스 환경에서는 함수 실행 전 DB 재연결이 발생할 수 있어, 트랜잭션 ID를 획득한 시점과 이후 `update()` 실행 시점 사이에 연결이 끊어지면 "Transaction not found" 에러가 난다.

**트랜잭션 소요 시간 (예: 항목 10개)**
```
updateMany(1) + update(1) + update(2) + ... + update(10) = 합산 시간
```

---

## 해결 방법 (Resolution)

`packages/core/src/code/code.service.ts` `saveGroupItems` 메서드를 배치 쿼리로 전환.

```ts
// ✅ 수정 후
const toUpdate = items.filter((item) => !!item.id);
const toCreate = items.filter((item) => !item.id);

return this.prisma.$transaction(async (tx) => {
  // 1. soft-delete (기존과 동일)
  await tx.codeItem.updateMany({ ... });

  // 2. 기존 항목 update — 병렬 실행
  const updatedItems = await Promise.all(
    toUpdate.map((item) => tx.codeItem.update({ ... })),
  );

  // 3. 신규 항목 — createMany(1회) + findMany(1회)
  let createdItems = [];
  if (toCreate.length > 0) {
    await tx.codeItem.createMany({ data: [...] });
    createdItems = await tx.codeItem.findMany({
      where: { groupId, isActive: true, code: { in: [...] } },
    });
  }

  return [...updatedItems, ...createdItems].sort(...).map(...);
});
```

**트랜잭션 소요 시간 (동일 항목 10개)**
```
updateMany(1) → Promise.all(update×10 병렬) → createMany(1) → findMany(1)
= max(단일 쿼리) × 4단계  ← 항목 수에 무관하게 고정
```

---

## 재발 방지 대책 (Prevention)

### 1. 트랜잭션 안 루프 쿼리 금지

트랜잭션 내에서 `for...of` + `await` 패턴은 항목 수만큼 트랜잭션을 점유하므로 사용하지 않는다.

| 패턴 | 결과 |
|------|------|
| `for (const x of items) { await tx.update(x) }` | ❌ 직렬, 타임아웃 위험 |
| `await Promise.all(items.map(x => tx.update(x)))` | ⚠️ 트랜잭션 안에서는 단일 커넥션으로 여전히 직렬 실행됨 |
| `await tx.createMany({ data: items })` | ✅ 배치, 가장 빠름 |

### 2. 배열 쓰기는 `createMany` / `updateMany` 우선 고려

- 같은 값으로 여러 행을 쓸 때 → `updateMany`
- 여러 행을 새로 삽입할 때 → `createMany`
- 행마다 다른 값으로 update가 필요할 때만 → `Promise.all` + 개별 `update`

### 3. Vercel 서버리스에서의 트랜잭션 주의

로컬에서 재현이 안 된다고 무시하지 않는다. 서버리스 환경은 DB 연결이 끊기는 빈도가 높아, 트랜잭션 내 소요 시간이 길면 반드시 문제가 생긴다.

**추가 발견 (2026-06-08)**: `Promise.all`을 트랜잭션 안에서 사용해도 Prisma 인터랙티브 트랜잭션은 단일 커넥션을 점유하므로 쿼리가 실제로 병렬 실행되지 않는다. 직렬과 동일한 소요 시간이 발생한다.

현재 적용된 임시 대책: `timeout: 30000`으로 상향.

```ts
this.prisma.$transaction(async (tx) => { ... }, { timeout: 30000 });
```

근본 해결은 트랜잭션 자체를 제거하고 각 쿼리를 독립 실행하는 방향으로 후속 리팩터링 예정 (어드민 코드 카탈로그 특성상 원자성 필수 아님).
