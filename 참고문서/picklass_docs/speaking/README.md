# speaking — 문서 인덱스

`D:\Project\speaking.picklass.com` (자매 서비스 — tutoring에서 말하기 영역만 분리) 전용 문서.

## 카테고리

- [architecture/](./architecture/) — 아키텍처 결정, 시스템 구성도, 초기 셋업 계획
- [features/](./features/) — 기능 단위 문서 (`[기능명]/` 하위 폴더로 분리)
- [errors/](./errors/) — 디버깅 후 남기는 오류 기록
- [operations/](./operations/) — 배포·환경변수·운영 가이드

## 진행 단계 요약

| 단계 | 내용 | 상태 | 문서 |
|------|------|------|------|
| Phase 1 | monorepo 스캐폴딩 (pnpm + Turbo + NestJS + Next.js) | ✅ | [initial-setup-plan](./architecture/initial-setup-plan.md) |
| Phase 2 | AuthModule (JWT 로그인 / 세션 / 핸드오프 토큰) | ✅ | — |
| Phase 3 | Prisma 스키마 (tutoring 복사, 11개 모델) | ✅ | [백엔드-API-모듈](./features/백엔드-API-모듈/README.md) |
| Phase 4 | 라우팅 스캐폴딩 (5탭 + 학습 흐름 + 온보딩) | ✅ | [프로토타입-화면-구성](./features/프로토타입-화면-구성/README.md) |
| Phase 5 | 5탭 UI 화면 (홈/학습/피드백/챌린지/마이) | ✅ | [프로토타입-화면-구성](./features/프로토타입-화면-구성/README.md) |
| Phase 6 | 백엔드 API 모듈 (Users / Courses / Feedback) | ✅ | [백엔드-API-모듈](./features/백엔드-API-모듈/README.md) |
| Phase 7 | 프론트-백엔드 연결 (서비스 레이어 + 로그인 + 실데이터) | ✅ | [프론트-백엔드-연결](./features/프론트-백엔드-연결/README.md) |
| Phase 8 | 배포 (Vercel + 서버, 환경변수, CORS) | 🔜 예정 | — |

> 학습 흐름 5단계 (Pick-Speak Method) + 6개 모듈은 **별도 레포에서 개발 예정**.

## 주요 문서

- [architecture/initial-setup-plan.md](./architecture/initial-setup-plan.md) — 초기 monorepo 스캐폴딩 계획서
- [features/README.md](./features/README.md) — 기능 문서 인덱스
