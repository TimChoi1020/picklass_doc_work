# DB 스키마 관리 가이드 (studio.picklass.com)

## 배경
자매 프로젝트 `picklass-backoffice`에서 Prisma 마이그레이션(`prisma migrate`, `prisma db push`) 실행 중 **데이터 롤백/삭제 이슈**가 발생하여 2026-05-22에 폐기되었다. studio도 동일 정책을 적용한다.

앞으로 DB 스키마 변경은 **DBMS에서 직접 DDL을 실행**하고, Prisma는 타입/쿼리 빌더 용도로만 사용한다.

## 금지 명령어 (절대 실행 금지)

| 명령 | 위험 |
|---|---|
| `prisma migrate dev` | shadow DB 비교 후 drop & recreate 가능성 |
| `prisma migrate deploy` | 미적용 마이그레이션 자동 실행 |
| `prisma db push` | schema.prisma 기준으로 DB를 **덮어씀** (가장 위험) |
| `npx prisma migrate ...` / `pnpm exec prisma migrate ...` | 위 명령의 우회 호출 |

**`apps/api/package.json`에서 다음 스크립트는 2026-05-22에 제거됨**:
- `db:migrate` (`prisma migrate dev`)
- `db:migrate:deploy` (`prisma migrate deploy`)

수동으로 위 명령을 호출하지 말 것.

## 허용 명령어

| 명령 | 용도 |
|---|---|
| `pnpm --filter @classsnap/api db:pull` | DB → schema.prisma 동기화 (introspection, **DB 변경 안 함**) |
| `pnpm --filter @classsnap/api db:seed` | 시드 데이터 삽입 |
| `pnpm --filter @classsnap/api build` | Prisma generate 포함 빌드 |

## 표준 워크플로우

### 1단계: DDL 작성 및 보관
변경 사항을 SQL 파일로 작성한다. 파일명 규칙:
```
apps/api/prisma/manual-sql/YYYY-MM-DD_설명.sql
```

예시:
```
apps/api/prisma/manual-sql/2026-05-22_add_phone_number.sql
```

**SQL 작성 시 반드시 트랜잭션으로 감쌀 것:**
```sql
BEGIN;

ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);

COMMIT;
-- 문제 발생 시: ROLLBACK;
```

### 2단계: 백업
운영/스테이징 DB는 **반드시 백업 후 실행**한다.
```bash
pg_dump -h HOST -U USER -d DB_NAME -F c -f backup_YYYYMMDD.dump
```

### 3단계: DBMS에서 직접 실행
- Supabase Studio / pgAdmin / DBeaver / psql 중 하나로 접속
- 작성한 SQL을 실행
- 결과 확인 (테이블 구조, 데이터 영향 범위)

### 4단계: schema.prisma 동기화
```bash
pnpm --filter @classsnap/api db:pull
```
- 실제 DB 스키마를 읽어 `apps/api/prisma/schema.prisma`를 자동 업데이트
- **DB는 건드리지 않음** (안전)
- 실행 후 반드시 `git diff apps/api/prisma/schema.prisma`로 변경 확인

### 5단계: Prisma Client 재생성 + 빌드/재시작
```bash
pnpm --filter @classsnap/api build
```
- `build` 스크립트가 `prisma generate`를 포함하므로 별도 호출 불필요
- API 프로세스 재시작

## 주의 사항

### `prisma db pull` 사용 시 주의
- DB의 실제 컬럼명/관계명 기준으로 schema.prisma가 재생성되므로 **모델/필드 이름이 바뀔 수 있음**
- `@map`, `@@map`, `@db.*` 어노테이션은 보존됨
- 한글 주석은 **사라짐** → 풀 후 수동 복구 필요
- 반드시 `git diff`로 의도하지 않은 변경 확인

### 다중 환경 동기화
- `manual-sql/` 폴더의 SQL을 dev → staging → prod 순으로 동일하게 적용
- 적용한 환경/일시를 별도 로그로 관리하는 것을 권장

### 기존 migrations 폴더
- `apps/api/prisma/migrations/` 폴더는 **이력 보존 목적으로 유지** (삭제하지 말 것)
- DB의 `_prisma_migrations` 테이블도 그대로 둠
- 새로운 변경은 모두 `apps/api/prisma/manual-sql/`에만 작성

## 예시: phone_number 컬럼 추가

**1. SQL 작성** `apps/api/prisma/manual-sql/2026-05-22_add_phone_number.sql`
```sql
BEGIN;

ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);

COMMIT;
```

**2. DBMS에서 실행** (Supabase Studio Query Editor 등)

**3. 동기화 및 빌드**
```bash
pnpm --filter @classsnap/api db:pull
git diff apps/api/prisma/schema.prisma   # 변경 확인
pnpm --filter @classsnap/api build
```

**4. API 재시작 후 동작 확인**

## 문제 발생 시
- DDL 실행 중 오류 → `ROLLBACK;` 후 SQL 검토
- 이미 COMMIT 후 데이터 손상 → 1단계의 백업 파일로 복구
  ```bash
  pg_restore -h HOST -U USER -d DB_NAME -c backup_YYYYMMDD.dump
  ```
