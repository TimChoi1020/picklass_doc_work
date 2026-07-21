# 스피킹 지문 생성 프롬프트 관리

## 개요
studio.picklass.com이 영어 학습 원문을 **스피킹 연습용 지문**으로 변환할 때 사용할 LLM 시스템 프롬프트를 관리하는 백오피스 메뉴입니다.

- **상태:** 신규 (2026-05-10)
- **메뉴 경로:** `/admin/speaking-passage-prompts`
- **메뉴 라벨:** `스피킹 지문 생성 프롬프트`
- **studio 연동:** 본 작업 범위 외 (별도 진행 예정). 본 메뉴는 데이터(프롬프트) 등록·수정만 담당.

## 결정 사항
- AI 모듈(`AiModule`)과 분리된 별도 테이블·메뉴
- 모듈과 직접 연결(FK) 없음. studio가 추후 `code` 기반으로 매칭
- 카테고리 컬럼 없음 ("당분간 지문 생성만"). 다른 종류 프롬프트가 필요해지면 별도 테이블 신설
- 단순 CRUD만. 버전/이력/A·B 테스트/미리보기 모두 추후 과제

## 데이터 모델

테이블: `speaking_passage_prompts`

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK, `gen_random_uuid()` 기본값 |
| `code` | VARCHAR(30) | 식별자 (unique). studio가 참조 |
| `name` | VARCHAR(200) | 관리자용 표시 이름 |
| `description` | TEXT | 설명, 메모 |
| `prompt` | TEXT | LLM 시스템 프롬프트 본문 |
| `status` | VARCHAR(20) | `active` / `inactive` |
| `created_at`, `updated_at`, `deleted_at` | Timestamptz | 표준 |

### `code` 명명 규칙
- 영문 소문자/숫자/하이픈, 1~30자
- 의미 단위로 짧게 (`default`, `business`, `ielts-prep` 등)
- `unique`. 중복 시 409 Conflict.

## 필드 의미

- **prompt**: studio가 `<<<RAW_LESSON_TEXT>>>` 같은 placeholder를 본문에서 직접 치환하는 시스템 프롬프트. 백오피스는 텍스트만 저장하며 placeholder 검증은 하지 않음 (studio 책임).

## API

| 메서드 | 경로 | 설명 |
|---|---|---|
| `GET` | `/speaking-passage-prompts` | 목록 (status, search, page, limit) |
| `GET` | `/speaking-passage-prompts/:id` | 단건 |
| `POST` | `/speaking-passage-prompts` | 생성 |
| `PUT` | `/speaking-passage-prompts/:id` | 수정 |
| `PATCH` | `/speaking-passage-prompts/:id/status` | 상태 변경 |
| `DELETE` | `/speaking-passage-prompts/:id` | 삭제 (soft delete) |

기존 백오피스 인증(JWT Bearer)을 그대로 사용. 외부(studio)용 API는 본 작업 범위 외.

## 작업 순서

1. 스키마 + 마이그레이션 SQL → `pnpm prisma generate`
2. `packages/types` 인터페이스 추가
3. `packages/core` 신규 모듈/서비스 → `pnpm run build`
4. `apps/admin/backend` 컨트롤러/모듈 + AppModule 등록
5. `apps/admin/frontend` API 클라이언트
6. `apps/admin/frontend` 목록·등록 페이지
7. 사이드바 메뉴 추가
8. 시드 1건 추가 (`code: 'default'`)
9. 빌드/타입체크
10. 동작 확인

## 의도적 제외

- 외부 조회 API (studio용), API Key 인증
- 모듈 ↔ 프롬프트 연결
- 카테고리/종류 컬럼
- 버전 관리, A·B 테스트, 사용 이력
- LLM 호출 미리보기
