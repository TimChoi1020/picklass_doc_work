# 외부 API 계약 (External API Contract)

외부업체에 제공하는 REST API 의 인증·에러·페이로드 규약. 호스트: `api.picklass.com`(`../../api/README.md`).
기존 출처: backoffice `b2b-external`(API Key), tutoring `external`(원타임 토큰).

## 1. 인증 방식 2종

### 1-A. 장기 API Key (파트너 데이터 pull)

- 헤더: `Authorization: Bearer pk_live_...`
- 검증: `partner_api_keys` 에서 `status_code='active'` 인 키 → 파트너 기관(`institution_id`) 해석 → 요청 스코프 자동 적용(타 기관 데이터 접근 불가).
- 키 형식: `pk_live_` + `randomBytes(24).base64url`.
- **발급/재발급/폐기는 backoffice(system_admin 화면) 소유.** 재발급 시 기존 active 키 즉시 폐기.
- 실패: `401 { "code": "UNAUTHORIZED", "message": "..." }`.

### 1-B. 원타임 액세스 토큰 (학습 핸드오프)

- 헤더: `X-Access-Token: {token}` + `X-Module-Code: {code}` (예: `SNR`, `FRT`).
- 검증 순서: 토큰 존재 → `is_expired` → 시간만료(초과 시 자동 expire) → `module_code` 일치 → `used_at` 기록.
- 토큰: `external_access_tokens`, UUID, 기본 만료 ~60분, 동일 user+lesson+module 유효 토큰 재사용.
- **발급은 tutoring(또는 인증된 내부 호출) 소유.** `api.picklass.com` 은 검증/만료만.
- 실패 코드: `INVALID_TOKEN`(401) · `TOKEN_EXPIRED`(403) · `MODULE_MISMATCH`(401) · `MISSING_ACCESS_TOKEN`/`MISSING_MODULE_CODE`(401).

## 2. 표준 에러 envelope

모든 에러 응답은 다음 형태로 통일한다(backoffice `b2b-external` 포맷 기준).

```json
{ "code": "UNAUTHORIZED", "message": "API Key가 유효하지 않거나 만료되었습니다." }
```

대표 코드: `BAD_REQUEST` · `UNAUTHORIZED` · `FORBIDDEN` · `NOT_FOUND` · `CONFLICT` · `TOO_MANY_REQUESTS` · `INTERNAL_ERROR`, 그리고 1-B 의 토큰 전용 코드.

## 3. 공통 규약

- 경로: `/api/v1/...` (글로벌 prefix `api` + URI 버저닝).
- Rate limit: 기본 IP당 `THROTTLE_TTL`(ms)당 `THROTTLE_LIMIT` 회. 초과 시 `429 { code: "TOO_MANY_REQUESTS" }`.
- 페이지네이션(목록 API): `page`, `limit` 쿼리. (b2b-external `GET /b2b/results` 선례)
- 날짜 범위(조회 API): `startDate`, `endDate` 필수, `startDate <= endDate`.
- 문서: Swagger `GET /docs`.

## 4. 관련 공유 테이블 (read-only)

| 테이블 | 소유 | 용도 |
|---|---|---|
| `partner_api_keys` | backoffice | API Key 인증 |
| `external_access_tokens` | tutoring/speaking | 원타임 토큰 인증 |

> 마이그레이션은 소유 서비스에서만. 외부 API 서비스는 schema 동기화(`prisma db pull`)만 한다 → [DB 마이그레이션 정책](../infrastructure/db-migration-policy.md).

## 5. 외부 토큰 발급 규약 (`EXTERNAL_TOKEN_ISSUE_SECRET`)

tutoring 의 한시적 발급 엔드포인트는 `X-Admin-Secret`(= `EXTERNAL_TOKEN_ISSUE_SECRET`) 으로 보호된다.
이는 운영용 정식 API 가 아니며(사용자/레슨 ID 하드코딩), `api.picklass.com` 으로 이식하지 않는다. 정식 발급 플로우가 필요하면 별도 설계 후 본 문서에 추가한다.
