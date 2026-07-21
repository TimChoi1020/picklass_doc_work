# PrismaService가 DATABASE_URL 없이 부팅 차단

| 발생일 | 영향 | 해결 |
|--------|------|------|
| 2026-05-08 | 초기 부트 단계에서 NestJS 부팅 자체가 실패 | AppModule에서 PrismaModule 일시 제외 |
| 2026-05-08 | (해결) tutoring `.env` 값을 복사 후 PrismaModule 복귀 | 정상 부팅 확인 |

## 증상 (Symptom)

`pnpm dev` 실행 시 web은 부팅되지만 api가 다음 에러로 즉시 종료:

```
ERROR [ExceptionHandler] Error: DATABASE_URL 환경변수가 설정되지 않았습니다.
  apps/api/.env 파일을 확인하세요.
    at new PrismaService (.../src/prisma/prisma.service.ts:11:13)
    at Injector.instantiateClass (...)
```

## 원인 (Root Cause)

`PrismaService` 생성자에서 `process.env.DATABASE_URL`이 비어 있으면 즉시 throw. NestJS는 모듈 초기화 시 모든 provider를 생성하므로 PrismaModule이 imports에 있으면 부팅 자체가 막힌다.

이는 운영 환경에서는 정확한 안전장치(`.env` 누락 → fail-fast)지만, **초기 스캐폴딩 단계의 "최소 부트" 목표와 충돌**한다. 새 사용자가 저장소를 clone → install → dev 했을 때 환경변수 없이도 health 엔드포인트가 떠야 한다.

## 해결 (Resolution)

`apps/api/src/app.module.ts`에서 PrismaModule을 일시적으로 imports에서 제외:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { HealthController } from './health/health.controller';

// PrismaModule은 첫 DB 사용이 추가될 때 import 한다.
// 지금은 DATABASE_URL 없이도 부팅되도록 의도적으로 제외한다.
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
  controllers: [HealthController],
})
export class AppModule {}
```

`src/prisma/prisma.module.ts`와 `prisma.service.ts` 코드는 그대로 둔다 — 첫 DB 쿼리가 도입되는 PR에서 한 줄(`PrismaModule`)만 추가하면 즉시 활성화된다.

## 재발 방지 (Prevention)

- **PrismaModule 재도입 체크리스트** (첫 prisma 쿼리 PR 작업 시):
  - [ ] `.env`에 실제 `DATABASE_URL` 채움 (Supabase 연결 문자열)
  - [ ] `apps/api/src/app.module.ts`의 imports에 `PrismaModule` 추가
  - [ ] `prisma db pull`로 placeholder 제거 + 실제 모델 도입
  - [ ] 로컬 부팅 확인: `curl /health` 200 OK
  - [ ] 배포 환경(Vercel)에 `DATABASE_URL` env var 등록 확인

- **유사 패턴**: 생성자에서 throw하는 NestJS provider는 모두 동일 함정.
  - 부트 시점 fail-fast가 필요한 운영용 동작은 그대로 유지하되,
  - 모듈 자체를 imports에서 빼는 식으로 "옵션 in/out"을 토글한다.
  - throw 자체를 약화시키지 않는다 (운영 안전성 우선).

## 참고 파일

- `speaking.picklass.com/apps/api/src/app.module.ts` — PrismaModule 주석 처리
- `speaking.picklass.com/apps/api/src/prisma/prisma.service.ts` — throw 동작 그대로 유지
- 자매 사례: tutoring은 처음부터 운영용 `.env`가 있어 이 함정에 부딪히지 않음
