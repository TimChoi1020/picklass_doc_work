# studio.picklass.com 문서

studio.picklass.com 전용 개발 문서 허브.

## 디렉터리 구조

```
studio/
├── features/     # 기능 단위 문서 (기획·구현·API 스펙)
├── errors/       # 오류 기록 (증상 / 원인 / 해결 / 재발방지)
├── architecture/ # 아키텍처 결정·다이어그램
└── operations/   # 배포·운영·환경 가이드
```

## 서비스 개요

- 강사(teacher)가 과정·레슨·지문을 생성·관리하는 콘텐츠 제작 도구
- 조직 구조: 파트너 > 그룹 > 기관 > 강사
- 공유 DB: tutoring, speaking과 동일 Supabase 인스턴스
- 포트: web 3005, api 3006 (studio 전용)
