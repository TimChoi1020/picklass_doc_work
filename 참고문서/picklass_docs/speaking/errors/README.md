# speaking / errors

개발 중 발생한 오류 기록. 같은 실수를 반복하지 않기 위한 조직 자산.

## 작성 의무 (CLAUDE.md §12.2)

speaking 작업 중 디버깅으로 시간을 쓴 모든 오류는 이 폴더에 남긴다.

## 파일명

`[증상-키워드]-[원인].md` (예: `prisma-fk-violation-nullable-처리.md`).
- 미래의 자신이 grep할 키워드를 우선.

## 필수 항목

```markdown
# [짧은 제목]

## 증상 (Symptom)
무엇이 어떻게 잘못 보였는가. 에러 메시지/스크린샷.

## 원인 (Root Cause)
실제 원인. 추측이 아니라 확인된 원인.

## 해결 (Resolution)
어떻게 고쳤는가. 코드 diff 또는 명령어. 관련 파일/커밋 링크.

## 재발 방지 (Prevention)
- 체크리스트
- 코드 가드 (린트룰, 타입, validation)
- 다른 기능 개발 시 적용할 항목
```

## 인프라 공통 이슈

여러 서비스가 공유하는 인프라(Supabase, analyzer, Vercel 등)에서 발생한 오류는
[shared/errors/](../../shared/errors/)로 이동하거나 cross-link.

## 기록 목록

- [prisma-empty-schema-blocks-typecheck.md](./prisma-empty-schema-blocks-typecheck.md)
  — 모델 0개인 schema.prisma가 prisma generate 및 typecheck를 차단 (2026-05-08)
- [prisma-service-throws-without-database-url.md](./prisma-service-throws-without-database-url.md)
  — PrismaService 생성자 throw가 DATABASE_URL 없는 부팅을 차단 (2026-05-08)
