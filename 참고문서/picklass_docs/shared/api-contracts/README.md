# shared / api-contracts

서비스 간 계약. 한 쪽이 바꾸면 다른 쪽도 바뀌어야 하는 인터페이스.

## 다룰 항목

- **[외부 API 계약](./external-api.md)** — `api.picklass.com` 외부 제공 API 인증(API Key·원타임 토큰)·에러·페이로드 규약
- **Handoff JWT 포맷** — tutoring/speaking/backoffice 간 사용자 토큰 전달 규약
- **공통코드 (`code_groups` / `code_items`)** — Supabase 공유 카탈로그
- **외부 토큰 발급 규약** — `EXTERNAL_TOKEN_ISSUE_SECRET` 사용 절차 (→ [external-api.md §5](./external-api.md))
- **REST 응답 포맷** — 공통 에러 응답, 페이지네이션 등 (→ [external-api.md §2·§3](./external-api.md))
