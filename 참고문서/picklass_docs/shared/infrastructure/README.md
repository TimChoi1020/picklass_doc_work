# shared / infrastructure

여러 서비스가 공유하는 인프라 리소스 정보.

## 다룰 항목

- **Supabase PostgreSQL** — tutoring과 speaking이 동일 인스턴스 공유
  - 공유 테이블: `users`, `code_groups`, `code_items`
  - 변경 절차 / 마이그레이션 정책
- **analyzer.picklass.com** — 외부 텍스트 분석기
  - 엔드포인트, 인증, 응답 스키마
- **Vercel 배포 환경** — 프로젝트 매핑, 환경변수 관리
- **Azure Cognitive Services** — Speech SDK 키 관리
- **Google Gemini** — API 키, 모델 선택 정책
