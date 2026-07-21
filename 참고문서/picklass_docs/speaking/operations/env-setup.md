# 환경변수 셋업

speaking은 tutoring과 자매 서비스로 **Supabase / JWT / Azure / Gemini / Analyzer 키를 모두 공유**한다. 따라서 로컬 셋업은 tutoring의 `.env` 값을 그대로 복사하고 PORT·API URL만 바꾸면 된다.

## 파일 위치

| 파일 | 용도 | git 추적 |
|------|------|----------|
| `apps/api/.env` | api 런타임 비밀값 | ❌ (.gitignore) |
| `apps/api/.env.example` | api 키 목록 placeholder | ✅ |
| `apps/web/.env.local` | web 런타임 변수 | ❌ (.gitignore) |
| `apps/web/.env.local.example` | web 변수 목록 placeholder | ✅ |

## tutoring 대비 차이

speaking 전용으로 바뀌는 값은 **PORT와 web의 API URL뿐**이다.

| 변수 | tutoring | speaking |
|------|----------|----------|
| `PORT` | 3002 | **3004** |
| `NEXT_PUBLIC_*_API_URL` | `NEXT_PUBLIC_TUTORING_API_URL=http://localhost:3002` | `NEXT_PUBLIC_SPEAKING_API_URL=http://localhost:3004` |
| `NEXT_PUBLIC_OIZI_SPEAKING_URL` | (게이트웨이 대상) | 미사용 (speaking 자체가 그 시스템) |

이외 값(`DATABASE_URL`, `JWT_SECRET`, `GEMINI_API_KEY`, `AZURE_SPEECH_*`, `IDLE_TIMEOUT_SECONDS`, `ACTIVITY_WRITE_THROTTLE_SECONDS`, `USER_ID_COOKIE_NAME`, `USER_ID_COOKIE_SECURE`, `ANALYZER_BASE_URL`, `EXTERNAL_TOKEN_ISSUE_SECRET`)는 **tutoring과 동일**.

## 핸드오프 토큰 호환

tutoring/speaking/backoffice는 **`JWT_SECRET`이 동일**해야 핸드오프 토큰이 시스템 간 검증된다. 임의로 새 값을 발급하면 SSO가 깨지므로 변경 금지.

## 새 개발자 셋업 절차

```sh
# 1. tutoring 저장소가 있다면 .env 그대로 복사
cp ../tutoring.picklass.com/apps/api/.env apps/api/.env
cp ../tutoring.picklass.com/apps/web/.env.local apps/web/.env.local

# 2. PORT/API URL만 speaking 값으로 교정
#    apps/api/.env:        PORT=3002 → PORT=3004
#    apps/web/.env.local:  NEXT_PUBLIC_TUTORING_API_URL=http://localhost:3002
#                       →  NEXT_PUBLIC_SPEAKING_API_URL=http://localhost:3004

# 3. 부팅
pnpm install
pnpm dev
# → web http://localhost:3003, api http://localhost:3004
```

tutoring 저장소가 없는 경우엔 picklass 인프라 담당자에게 값을 받는다.

## 배포 환경 (Vercel)

배포 단계에서 동일 변수를 Vercel 프로젝트의 Environment Variables에 등록한다.
공개되어도 되는 변수는 `NEXT_PUBLIC_` prefix만 사용 (`NEXT_PUBLIC_SPEAKING_API_URL`).
나머지는 모두 server-side only.

## 주의

- `.env` / `.env.local`은 절대 커밋 금지. `.gitignore`에 `.env`, `.env*.local` 포함됨을 확인.
- 로그/콘솔/에러 메시지에 비밀값 출력 금지. PrismaService는 `DATABASE_URL` 미설정 시 키 자체는 출력하지 않고 메시지만 남긴다.
- 키 회전 시 **세 시스템 동시 갱신** 필요 (`JWT_SECRET`, `EXTERNAL_TOKEN_ISSUE_SECRET`).

## 관련 문서

- 오류 사례: [prisma-service-throws-without-database-url.md](../errors/prisma-service-throws-without-database-url.md)
- 초기 셋업: [../architecture/initial-setup-plan.md](../architecture/initial-setup-plan.md)
