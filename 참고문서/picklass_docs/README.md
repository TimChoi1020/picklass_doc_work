# picklass_docs

picklass 산하 모든 서비스의 **단일 문서 허브**. 코드 저장소에는 마크다운 문서를 두지 않고
모두 이 저장소로 모은다.

## 디렉터리 구조

```
picklass_docs/
├── speaking/      # speaking.picklass.com 전용 문서
├── tutoring/      # tutoring.picklass.com 전용 문서
├── studio/        # studio.picklass.com 전용 문서
├── backoffice/    # picklass-backoffice 전용 문서
├── api/           # api.picklass.com (외부업체 제공 API) 전용 문서
└── shared/        # 여러 서비스에 걸친 공통 문서
```

각 서비스 폴더는 다음 하위 카테고리를 갖는다:
- `features/` — 기능 단위 문서 (기획·구현·API 스펙)
- `errors/` — 오류 기록 (증상 / 원인 / 해결 / 재발방지)
- `architecture/` — 아키텍처 결정·다이어그램
- `operations/` — 배포·운영·환경 가이드

`shared/`는 다음 카테고리를 갖는다:
- `conventions/` — 조직 차원 코딩 컨벤션
- `api-contracts/` — 서비스 간 계약 (handoff 토큰, 공통코드 등)
- `infrastructure/` — Supabase, analyzer 등 공유 인프라
- `errors/` — 인프라 공통 이슈

## 어디에 무엇을 둘 것인가

| 작업 유형 | 위치 |
|----------|------|
| speaking 신규 기능 명세 | `speaking/features/[기능명]/` |
| speaking 디버깅 후 오류 기록 | `speaking/errors/` |
| speaking 아키텍처 결정 | `speaking/architecture/` |
| studio 신규 기능 명세 | `studio/features/[기능명]/` |
| api(외부 제공) 환경·배포 가이드 | `api/operations/` |
| 배포·환경변수 가이드 | `[서비스]/operations/` |
| 여러 서비스가 공유하는 DB 스키마 결정 | `shared/infrastructure/` |
| 핸드오프 토큰 포맷, 공통코드 규약, 외부 API 계약 | `shared/api-contracts/` |
| 조직 표준 코딩 룰 | `shared/conventions/` |

## 사용 규칙

1. **코드 변경과 문서 변경은 동시에**: 코드 PR과 문서 PR을 짝지어 cross-link.
2. **파일명 컨벤션**: `[기능명]-[작업내용].md` (예: `발음평가-점수계산-개선.md`).
3. **오류 기록 필수 항목**: 증상(Symptom) / 원인(Root Cause) / 해결(Resolution) / 재발 방지(Prevention).
4. **빈 폴더 방지**: 새 카테고리를 만들 때 README.md 또는 `.gitkeep`을 함께 둔다.

각 서비스 / 카테고리별 사용 방법은 해당 폴더의 README.md 참조.
