# api / api.picklass.com

외부업체 제공 **API 전용 서비스**(`../api.picklass.com`) 문서.

단일 패키지 NestJS 11 + Prisma 5.22(공유 Supabase Postgres, **read-only**). 웹 프론트 없음.

## 핵심 성격

- **외부업체 전용**: 사용자 로그인(JWT)이 아니라 **장기 API Key** 또는 **원타임 액세스 토큰**으로 인증.
- **검증/조회 전용**: 키·토큰 **발급은 backoffice / tutoring 이 소유**. 본 서비스는 마이그레이션을 두지 않는다 → [DB 마이그레이션 정책](../shared/infrastructure/db-migration-policy.md).
- **공유 DB 직접 접근**: 다른 picklass 서비스와 동일 Supabase 인스턴스에 Prisma 로 직접 read.

## 하위 카테고리

- `features/` — 연동 기능 규격 (우유랩스 스피킹앱 연동 등)
- `architecture/` — 아키텍처 결정 (인증 두 패턴 등)
- `operations/` — 초기 환경 구성, 배포(도커), 환경변수

## 진행 중 연동

- [파고다 연동 (Doc A · 10 API)](features/파고다-연동/README.md) — **계획 변경(2026-07-08)**: Doc A 10개 API 전부 대응(픽클래스=오이지 역할). #1~7 인터페이스 테이블 설계 완료
- [우유랩스 스피킹앱 연동](features/우유랩스-스피킹앱-연동/README.md) — 8개 모듈 콘텐츠 + verify/results/tokens API (구현 범위: Doc B 3개)

## 관련 문서

- 외부 API 계약(헤더·에러 포맷·키/토큰 규약): [shared/api-contracts/external-api.md](../shared/api-contracts/external-api.md)
- 기존 외부 API 출처: backoffice `b2b-external`(API Key), tutoring `external`(원타임 토큰)
