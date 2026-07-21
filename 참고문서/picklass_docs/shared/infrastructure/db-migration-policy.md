# DB 마이그레이션 정책 (조직 표준)

> **정책 결정일**: 2026-05-22
> **적용 대상**: picklass-backoffice, speaking.picklass.com, studio.picklass.com, tutoring.picklass.com
> **위치 분류**: 조직 차원 인프라 운영 정책 (`shared/infrastructure/`)

---

## 1. 배경

picklass-backoffice에서 Prisma 마이그레이션(`prisma migrate`, `prisma db push`) 실행 중 **데이터 롤백/삭제 이슈**가 발생하여 폐기되었다. Supabase를 공유하거나 동일 ORM 스택을 쓰는 모든 picklass 서비스에 동일 정책을 적용한다.

**핵심 원칙**: DB 스키마 변경은 **DBMS에서 직접 DDL을 실행**하고, Prisma는 **타입/쿼리 빌더 용도로만** 사용한다.

---

## 2. 금지 명령어 (모든 서비스 공통)

| 명령 | 위험 |
|---|---|
| `prisma migrate dev` | shadow DB 비교 후 drop & recreate 가능성 |
| `prisma migrate deploy` | 미적용 마이그레이션 자동 실행 |
| `prisma db push` | schema.prisma 기준으로 DB를 **덮어씀** (가장 위험) |
| `npx prisma migrate ...` / `pnpm exec prisma migrate ...` | 위 명령의 우회 호출 |

각 서비스 `package.json`에 위 명령은 등록하지 않는다. 수동 호출도 금지.

### 2.1 2026-05-22 일괄 정리 내역

| 서비스 | 제거된 스크립트 |
|---|---|
| picklass-backoffice | `prisma:push`, `prisma:migrate` |
| studio.picklass.com (`@classsnap/api`) | `db:migrate`, `db:migrate:deploy` |
| speaking.picklass.com | 해당 없음 (원래 없었음) |
| tutoring.picklass.com | 해당 없음 (원래 없었음) |

---

## 3. 허용 명령어 (서비스별)

| 서비스 | DB 동기화 | Client 생성 |
|---|---|---|
| picklass-backoffice | `pnpm prisma:pull` | `pnpm prisma:generate` |
| speaking.picklass.com | `pnpm --filter @speaking/api prisma:pull` | `pnpm --filter @speaking/api prisma:generate` |
| studio.picklass.com | `pnpm --filter @classsnap/api db:pull` | `pnpm --filter @classsnap/api build` (자동 포함) |
| tutoring.picklass.com | `pnpm --filter @tutoring/api prisma:pull` | `pnpm --filter @tutoring/api prisma:generate` |

`prisma db pull`은 **introspection 전용** — DB를 변경하지 않고 실제 스키마를 읽어 `schema.prisma`를 갱신한다.

---

## 4. 표준 워크플로우

### 4.1 DDL 작성 및 보관

각 서비스 저장소 내 정해진 위치에 SQL 파일로 보관한다. 파일명: `YYYY-MM-DD_설명.sql`.

| 서비스 | DDL 보관 위치 |
|---|---|
| picklass-backoffice | `prisma/manual-sql/` |
| speaking.picklass.com | `apps/api/prisma/manual-sql/` |
| studio.picklass.com | `apps/api/prisma/manual-sql/` |
| tutoring.picklass.com | `apps/api/prisma/manual-sql/` |

**작성 규칙**:
- 반드시 `BEGIN; ... COMMIT;` 트랜잭션으로 감쌀 것
- 운영 적용 전 반드시 백업
- Supabase 공유 환경에서는 모든 영향 서비스 사전 정렬

### 4.2 공유 DB 주의 (speaking ↔ tutoring)

speaking과 tutoring은 **동일 Supabase 인스턴스**를 공유한다. 공통 테이블(`users`, `code_*` 등) 변경 시:
1. 양쪽 팀 사전 정렬 필수
2. 변경 영향 검토 결과를 본 문서 또는 `speaking/operations/`, `tutoring/operations/`에 기록
3. 두 서비스 모두에서 `prisma:pull` → `build` → 재시작

### 4.3 6단계 절차 (모든 서비스 공통)

1. **DDL 작성**: 위 위치에 `YYYY-MM-DD_*.sql` 보관 (BEGIN/COMMIT 필수)
2. **백업**:
   ```bash
   pg_dump -h HOST -U USER -d DB_NAME -F c -f backup_YYYYMMDD.dump
   ```
3. **DBMS 직접 실행**: Supabase Studio / pgAdmin / DBeaver / psql
4. **schema.prisma 동기화**: 해당 서비스의 `prisma:pull` (또는 `db:pull`)
   - 한글 주석 사라질 수 있음 → `git diff` 필수
   - 모델/필드 이름이 DB 컬럼 기준으로 재생성될 수 있음
5. **Prisma Client 재생성**: 해당 서비스의 `prisma:generate` (또는 `build`)
6. **API 빌드 및 재시작**

---

## 5. 기존 migrations 폴더 처리

- 각 서비스의 `prisma/migrations/` (또는 `apps/api/prisma/migrations/`)는 **이력 보존 목적으로 유지**.
- DB의 `_prisma_migrations` 테이블도 그대로 둠.
- **새로운 변경은 모두 `manual-sql/`에만 작성**.

---

## 6. 문제 발생 시

- DDL 실행 중 오류 → `ROLLBACK;` 후 SQL 재검토
- 이미 COMMIT 후 데이터 손상 → §4.3-2의 백업 파일로 복구:
  ```bash
  pg_restore -h HOST -U USER -d DB_NAME -c backup_YYYYMMDD.dump
  ```

---

## 7. 변경 의도/배경 기록

DDL 자체는 각 서비스의 `manual-sql/`에 코드와 함께 보관하되, **변경 의도/배경/논의 기록**은 본 docs 저장소에 둔다:
- speaking 단독 변경 → `speaking/operations/`
- tutoring 단독 변경 → `tutoring/operations/`
- 양쪽 영향 → `shared/infrastructure/`
- backoffice 단독 → `backoffice/operations/`

---

## 8. 관련 문서

- speaking 운영 가이드: [`speaking/operations/db-management.md`](../../speaking/operations/db-management.md)
- speaking CLAUDE.md §15 (룰 마스터): `c:/Projects/speaking.picklass.com/CLAUDE.md`
- 각 서비스 자체 가이드 (picklass_docs 외):
  - picklass-backoffice: `docs/database/dev.md`
  - studio.picklass.com: `docs/database/dev.md`
  - tutoring.picklass.com: `docs/database/dev.md`
