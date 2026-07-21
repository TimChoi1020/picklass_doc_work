# DB 관리 운영 가이드 (speaking.picklass.com)

> 본 문서는 speaking 전용 운영 가이드다. 조직 표준 정책은 [`shared/infrastructure/db-migration-policy.md`](../../shared/infrastructure/db-migration-policy.md)를 따른다.

## 핵심 원칙 (요약)

- Prisma 마이그레이션 명령(`migrate dev/deploy`, `db push`)은 **절대 금지** (2026-05-22 폐기).
- DB 스키마 변경은 **Supabase Studio / pgAdmin / DBeaver / psql에서 직접 DDL 실행**.
- 변경 후 `prisma:pull`로 schema.prisma 동기화, `prisma:generate`로 Client 재생성, API 재시작.

전체 배경/금지 명령/6단계 절차는 [조직 표준 문서](../../shared/infrastructure/db-migration-policy.md) 참조.

---

## speaking 특화 사항

### 공유 DB 환경
- **tutoring.picklass.com과 동일한 Supabase 인스턴스 공유**.
- 공통 테이블(`users`, `code_*`)은 양쪽 영향 → tutoring 팀과 사전 정렬 필수.
- speaking만 사용하는 테이블(speech/passage 관련 등)은 speaking 독립 진행 가능.

### DDL 보관 위치
```
apps/api/prisma/manual-sql/YYYY-MM-DD_설명.sql
```

### 명령어
```bash
# DB → schema.prisma 동기화 (introspection, DB 변경 안 함)
pnpm --filter @speaking/api prisma:pull

# Prisma Client 재생성
pnpm --filter @speaking/api prisma:generate

# 빌드 (prisma generate 포함)
pnpm --filter @speaking/api build
```

### 변경 영향 분류

| 변경 범위 | 사전 정렬 대상 | 의도 기록 위치 |
|---|---|---|
| speaking 단독 테이블 | speaking 팀 내부 | `speaking/operations/` (본 폴더) |
| 공통 테이블 (users, code_*) | speaking + tutoring | `shared/infrastructure/` |
| 공유 인프라(인증, JWT 등) | speaking + tutoring + backoffice | `shared/infrastructure/` 또는 `shared/api-contracts/` |

---

## 예시: phone_number 컬럼 추가 (speaking 단독)

**1. SQL 작성** `apps/api/prisma/manual-sql/2026-05-22_add_phone_number.sql`
```sql
BEGIN;

ALTER TABLE speech_users ADD COLUMN phone_number VARCHAR(20);

COMMIT;
```

**2. tutoring 영향 검토**
- `speech_users`는 speaking 전용 → 영향 없음 ✅

**3. DBMS에서 실행** (Supabase Studio Query Editor)

**4. 동기화 및 빌드**
```bash
pnpm --filter @speaking/api prisma:pull
git diff apps/api/prisma/schema.prisma   # 변경 확인
pnpm --filter @speaking/api build
```

**5. API 재시작 후 동작 확인**

**6. 본 문서 옆에 변경 의도 파일 작성** (선택)
- `speaking/operations/2026-05-22_add_phone_number.md` — 변경 이유, 영향 범위, 검증 절차

---

## 공유 테이블 변경 예시 (speaking + tutoring)

`users` 테이블에 `last_active_at` 컬럼 추가 시:

1. **사전 정렬**: tutoring 팀과 컬럼명/타입/nullable/기본값 확정
2. **공통 DDL**: `shared/infrastructure/` 폴더에 의도 문서 + SQL 작성
3. **양쪽 manual-sql 동기화**: speaking과 tutoring 양쪽의 `apps/api/prisma/manual-sql/`에 동일 SQL 복사
4. **DBMS 실행**: 공유 DB는 한 번만 실행 (양쪽 동시 적용)
5. **양쪽 prisma:pull → build → 재시작**

## 문제 발생 시
- DDL 실행 중 오류 → `ROLLBACK;` 후 SQL 검토
- COMMIT 후 데이터 손상 → 백업 파일로 복구 (pg_restore)
- 자세한 절차: [조직 표준 문서 §6](../../shared/infrastructure/db-migration-policy.md)

## 관련 룰 마스터
speaking 저장소의 [`CLAUDE.md`](../../../../speaking.picklass.com/CLAUDE.md) §13.4 / §15에 동일 정책이 요약돼 있다 (소스 코드 저장소에서 빠른 참조용).
