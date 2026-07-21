# CLAUDE.md — picklass_docs

이 저장소는 picklass 산하 모든 서비스의 **단일 문서 허브**다. 코드는 없고 문서만 있다.

전체 구조와 사용법은 [`README.md`](README.md) 참조. 본 파일은 Claude/AI 에이전트 작업 시 추가 컨텍스트.

---

## 1. 본 저장소의 역할

- picklass 산하 서비스(`speaking`, `tutoring`, `studio`, `backoffice` 등)의 **공통 문서 단일 저장소**.
- 각 서비스 코드 저장소에 마크다운 문서를 두지 않고 본 저장소에 모은다 (speaking은 §10 단일 문서 저장소 정책 강제, 다른 서비스는 점진적 마이그레이션).
- 본 저장소 자체에는 **코드, package.json, 빌드 산출물이 없다**. `.md` 파일과 첨부 자료만.

## 2. 디렉터리 구조 (요약)

```
picklass_docs/
├── README.md                 # 전체 가이드 (필독)
├── CLAUDE.md                 # 본 파일
├── speaking/                 # speaking.picklass.com 전용
│   ├── features/             # 기능 단위 문서
│   ├── errors/               # 오류 기록
│   ├── architecture/         # 아키텍처 결정
│   └── operations/           # 배포·운영 (db-management.md 등)
├── tutoring/                 # tutoring.picklass.com 전용
├── studio/                   # studio.picklass.com 전용 (필요 시 생성)
├── backoffice/               # picklass-backoffice 전용
└── shared/                   # 여러 서비스 공통
    ├── conventions/          # 조직 표준 코딩 룰
    ├── api-contracts/        # 핸드오프 JWT, 공통코드 등
    ├── infrastructure/       # Supabase, analyzer, DB 정책 등
    └── errors/               # 인프라 공통 이슈
```

상세 분류 기준은 [`README.md`](README.md)의 "어디에 무엇을 둘 것인가" 표 참조.

## 3. 작성 규칙

### 3.1 파일명
- 한국어 가능 (예: `20260512_힌트시스템_개선_개발완료.md`)
- 시간 순서가 중요한 문서는 `YYYY-MM-DD_제목.md` 또는 `YYYYMMDD_제목.md` 형식 권장
- 영문 슬러그도 OK (예: `db-migration-policy.md`, `env-setup.md`)

### 3.2 cross-link
- 같은 폴더 내: `[제목](./파일.md)`
- 다른 폴더: 상대 경로 (예: `[조직 정책](../../shared/infrastructure/db-migration-policy.md)`)
- 코드 저장소 참조: 본 저장소 기준 상대 경로 (예: `../speaking.picklass.com/CLAUDE.md`)

### 3.3 의도/배경 기록
- DDL 같은 실행 가능한 산출물은 각 서비스 코드 저장소(`apps/api/prisma/manual-sql/` 등)에.
- **변경 의도, 배경, 논의 결과**는 본 저장소의 적절한 폴더에 별도 문서로.

## 4. 자주 참조되는 문서

| 주제 | 위치 |
|---|---|
| 조직 표준 DB 마이그레이션 정책 (2026-05-22 폐기 결정) | [`shared/infrastructure/db-migration-policy.md`](shared/infrastructure/db-migration-policy.md) |
| speaking DB 운영 가이드 | [`speaking/operations/db-management.md`](speaking/operations/db-management.md) |
| speaking 환경 변수 셋업 | [`speaking/operations/env-setup.md`](speaking/operations/env-setup.md) |
| speaking 초기 설계/스크린 | [`speaking/architecture/`](speaking/architecture/) |

## 5. 본 저장소를 수정할 때

- **새 폴더 생성 전**: 기존 카테고리(`features/errors/architecture/operations/`)에 들어갈 수 있는지 먼저 확인.
- **공통 vs 서비스 전용 분류**:
  - 여러 서비스 동시 영향 → `shared/`
  - 한 서비스 전용 → `[서비스]/[카테고리]/`
- **README.md는 보수적으로 수정**: 디렉터리 구조나 분류 기준이 바뀌면 반드시 함께 업데이트.
- **본 CLAUDE.md는 사실 기반 최소 버전**. 규칙을 추가할 땐 사용자 확인 후 명시적으로 작성.

## 6. 관련 워크스페이스 (코드 저장소)

| 워크스페이스 (본 저장소 기준 상대 경로) | 본 저장소에서 매핑되는 폴더 |
|---|---|
| `../speaking.picklass.com` | `speaking/` (단일 문서 저장소 정책 강제) |
| `../tutoring.picklass.com` | `tutoring/` (자체 `docs/`도 유지) |
| `../studio.picklass.com3` | `studio/` (자체 `docs/` 유지) |
| `../picklass-backoffice` | `backoffice/` (필요 시), 자체 `docs/` 유지 |
| `../api.picklass.com` | `api/` (외부업체 제공 API 전용 서비스) |
