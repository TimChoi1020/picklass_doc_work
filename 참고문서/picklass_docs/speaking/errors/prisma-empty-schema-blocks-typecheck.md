# Prisma 빈 스키마가 typecheck를 차단

| 발생일 | 영향 | 해결 |
|--------|------|------|
| 2026-05-08 | 초기 스캐폴딩 단계의 typecheck/install 실패 | placeholder 모델 1개 추가 + postinstall 제거 |

## 증상 (Symptom)

신규 monorepo 스캐폴딩 단계에서 다음 두 명령이 연쇄적으로 실패:

```
$ pnpm install
...
apps/api postinstall: Error:
You don't have any models defined in your schema.prisma, so nothing will be generated.
ELIFECYCLE  Command failed with exit code 1.

$ pnpm typecheck
@speaking/api:typecheck: src/prisma/prisma.service.ts(2,10): error TS2305:
  Module '"@prisma/client"' has no exported member 'PrismaClient'.
@speaking/api:typecheck: src/prisma/prisma.service.ts(19,16): error TS2339:
  Property '$connect' does not exist on type 'PrismaService'.
```

## 원인 (Root Cause)

`@prisma/client`는 **빌드 시 `prisma generate`가 실행되어야 비로소 타입과 클라이언트를 코드 생성**하는 패키지다. `schema.prisma`에 모델이 0개면 `prisma generate`가 다음과 같이 동작한다:

1. CLI가 "no models" 메시지를 출력하고 **exit code 1**로 종료.
2. `node_modules/.pnpm/@prisma+client@.../...` 안에 `PrismaClient` export가 만들어지지 않음.
3. 이 상태에서 `prisma.service.ts`가 `PrismaClient`를 import → `TS2305: no exported member`.

postinstall에 `prisma generate`를 두면 `pnpm install` 자체도 실패한다.

## 해결 (Resolution)

세 가지를 동시에 적용:

1. **`apps/api/package.json`에서 postinstall 제거**
   ```diff
    "scripts": {
   -  "postinstall": "prisma generate --schema=prisma/schema.prisma",
      "dev": "nest start --watch",
   ```

2. **`apps/api/prisma/schema.prisma`에 placeholder 모델 1개 추가**
   ```prisma
   // TODO: prisma db pull로 실제 스키마 가져온 뒤 제거
   model SchemaPlaceholder {
     id Int @id @default(autoincrement())
     @@map("_schema_placeholder")
   }
   ```
   - 모델명 첫 글자가 underscore면 Prisma validation 실패 → PascalCase로 두고 `@@map`으로 underscore 테이블명 매핑.

3. **수동 generate**
   ```sh
   pnpm --filter @speaking/api prisma:generate
   ```
   이후 typecheck 통과.

## 재발 방지 (Prevention)

- **새 NestJS+Prisma 프로젝트 스캐폴딩 시 체크리스트**:
  - [ ] `schema.prisma`에 모델이 비어 있으면 placeholder 1개 추가하거나 prisma 통합 자체를 첫 사용 시점까지 미룬다.
  - [ ] `postinstall: prisma generate`는 **모델이 들어간 이후**에 추가한다.
  - [ ] `prisma generate` 실패는 install 전체를 깨뜨리므로 CI에서도 동일 함정.

- **실제 스키마 도입 시점에 placeholder 제거 절차**:
  1. `prisma db pull` 또는 모델 직접 작성
  2. `model SchemaPlaceholder { ... }` 블록 + `_schema_placeholder` 테이블(있다면) 제거
  3. `pnpm prisma:generate`
  4. `pnpm typecheck` 통과 확인
  5. postinstall 재도입 검토 (`prisma generate --schema=prisma/schema.prisma`)

- **유사 패턴**: 코드 생성형 패키지(prisma, GraphQL codegen, OpenAPI codegen)는 모두 같은 함정. 빈 입력 → 코드 미생성 → 타입 부재 → typecheck 실패.

## 참고 파일

- `speaking.picklass.com/apps/api/package.json` — postinstall 제거됨
- `speaking.picklass.com/apps/api/prisma/schema.prisma` — placeholder 모델 포함
- `speaking.picklass.com/apps/api/src/prisma/prisma.service.ts` — PrismaClient import (수정 없음)
- 자매 사례: `tutoring.picklass.com`은 모델이 풍부해서 동일 문제 미발생
