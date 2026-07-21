# shared — 여러 서비스 공통 문서

여러 picklass 서비스에 동시에 영향을 주는 문서를 모은다.

## 카테고리

- [conventions/](./conventions/) — 조직 차원 코딩 컨벤션, 룰
- [api-contracts/](./api-contracts/) — 서비스 간 계약 (handoff JWT, 공통코드, REST contract 등)
- [infrastructure/](./infrastructure/) — Supabase, analyzer.picklass.com, Vercel 등 공유 인프라
- [errors/](./errors/) — 인프라/공유 영역에서 발생한 오류

## 무엇을 여기에 둘 것인가

다음 중 하나라도 해당하면 `shared/`:
- 변경 시 여러 서비스에 동시 영향 (예: 공통 DB 테이블, JWT 포맷, 공통코드 카탈로그)
- 외부 시스템 통합 정보가 여러 서비스에 재사용됨
- 조직 표준 룰
