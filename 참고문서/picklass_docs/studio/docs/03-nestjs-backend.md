# Phase 3: NestJS 백엔드 추가

> 기존 Next.js API routes는 그대로 유지한다. NestJS는 **추후 새로운 기능 개발**을 위한 백엔드로 구성한다.
> 기존 API를 NestJS로 마이그레이션하지 않는다.

---

## 1. 기존 API 유지 (Next.js API Routes)

현재 `/app/class/`가 사용하는 20개 API route는 Next.js API Routes로 그대로 유지한다.

```
src/app/api/
  ├── texts/route.ts                     # GET, POST
  ├── texts/[id]/route.ts                # GET, PATCH, DELETE
  ├── texts/twins/[id]/route.ts          # GET
  ├── generate-text/route.ts             # POST
  ├── extract-text/route.ts              # POST
  ├── text-analysis/route.ts             # POST
  ├── strategic-reading/route.ts         # POST/GET
  ├── clarification-feedback/route.ts    # POST
  ├── prediction-feedback/route.ts       # POST
  ├── qar-feedback/route.ts              # POST
  ├── summary-feedback/route.ts          # POST
  ├── skimming-feedback/route.ts         # POST
  ├── meaning-guessing-feedback/route.ts # POST
  ├── azure-speech/route.ts              # GET, POST
  ├── tts-cache/cleanup/route.ts         # GET
  ├── tts-cache/stats/route.ts           # GET
  ├── word-definitions/route.ts          # POST
  ├── word-suggestion/route.ts           # POST
  ├── word-relationship/route.ts         # POST
  ├── async-tasks/route.ts               # POST, GET, PATCH
  └── user-profile/route.ts              # GET, PUT, DELETE
```

> 삭제 대상: `src/app/api/auth/`, `src/app/api/texts/bulk-delete/` (class/에서 미사용)

---

## 2. NestJS 백엔드 목적

NestJS는 다음 목적으로 구성한다:

- **신규 API 전담**: Course, Report, AI(주제생성/지문분석), Auth, CommonCodes
- 기존 `/class` 관련 API는 Next.js API Routes 유지 (포트 3000)
- NestJS는 포트 3001에서 동작, 프론트엔드는 `NEXT_PUBLIC_API_URL`로 호출

---

## 3. NestJS 프로젝트 구조 (2026-03 구현 기준)

```
apps/api/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── common/                    # Guards, Decorators, Interceptors, Filters
│   │   ├── guards/auth.guard.ts   # Supabase JWT 검증 (ALLOW_MOCK_AUTH 시 Mock)
│   │   ├── decorators/current-user.decorator.ts
│   │   ├── interceptors/logging.interceptor.ts
│   │   ├── filters/http-exception.filter.ts
│   │   └── pipes/validation.pipe.ts
│   ├── config/
│   ├── supabase/
│   ├── health/
│   ├── auth/                      # GET /auth/me
│   ├── common-codes/              # GET /common-codes, /common-codes/:code
│   ├── ai/                        # POST /ai/generate-topics, /ai/analyze-passage
│   ├── courses/                   # CRUD /courses, /courses/:id/lessons
│   ├── passages/                  # GET/POST /passages/:id/analysis
│   ├── classes/                   # CRUD /classes, /classes/:id/students
│   └── reports/                   # GET /reports, POST /reports/export
├── test/app.e2e-spec.ts
└── package.json
```

> 기존 texts, strategic-reading 등은 Next.js API Routes로 유지.

---

## 4. Supabase 연결 (재사용 준비)

NestJS에서도 동일한 Supabase에 연결하여 새 기능 개발 시 바로 사용할 수 있도록 준비한다.

```typescript
// supabase/supabase.service.ts
@Injectable()
export class SupabaseService {
  private client: SupabaseClient<Database>;

  constructor(private configService: ConfigService) {
    this.client = createClient<Database>(
      this.configService.get('SUPABASE_URL'),
      this.configService.get('SUPABASE_SERVICE_ROLE_KEY'),
      {
        auth: {
          persistSession: false,
          autoRefreshToken: false,
        },
      },
    );
  }

  getClient(): SupabaseClient<Database> {
    return this.client;
  }
}
```

> **참조**: 현재 `src/lib/supabase/serverAdminClient.ts`가 동일한 패턴을 사용 중

---

## 5. Auth Guard (Supabase JWT + Mock)

- **Bearer 토큰 있음**: `supabase.auth.getUser(token)`으로 검증 후 `request.user` 설정
- **토큰 없음/무효 + ALLOW_MOCK_AUTH=true**: Mock 사용자 반환 (개발용)
- **토큰 없음/무효 + ALLOW_MOCK_AUTH 미설정**: 401 Unauthorized

```typescript
// common/guards/auth.guard.ts
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly supabase: SupabaseService) {}
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const token = request.headers.authorization?.replace('Bearer ', '');
    if (token) {
      const { data: { user } } = await this.supabase.getClient().auth.getUser(token);
      if (user) { request.user = mapSupabaseUserToAuthUser(user); return true; }
    }
    if (process.env.ALLOW_MOCK_AUTH === 'true') {
      request.user = MOCK_USER;
      return true;
    }
    throw new UnauthorizedException('인증이 필요합니다.');
  }
}
```

---

## 6. NestJS 부트스트랩

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableCors({
    origin: ['http://localhost:3000', 'https://studio.picklass.com'],
    credentials: true,
  });

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    transform: true,
  }));

  const port = process.env.PORT || 3001;
  await app.listen(port);
}
bootstrap();
```

---

## 7. NestJS Dependencies (최소 구성)

```json
{
  "dependencies": {
    "@nestjs/common": "^11",
    "@nestjs/core": "^11",
    "@nestjs/config": "^4",
    "@nestjs/platform-express": "^11",
    "@supabase/supabase-js": "^2.87.1",
    "class-validator": "^0.14",
    "class-transformer": "^0.5",
    "reflect-metadata": "^0.2",
    "rxjs": "^7",
    "@classsnap/shared": "workspace:*"
  },
  "devDependencies": {
    "@nestjs/cli": "^11",
    "@nestjs/schematics": "^11",
    "@nestjs/testing": "^11",
    "@types/node": "^20",
    "typescript": "^5",
    "jest": "^29",
    "@types/jest": "^29",
    "ts-jest": "^29",
    "supertest": "^6",
    "@types/supertest": "^6"
  }
}
```

> AI(Gemini), TTS(Azure) 등의 패키지는 해당 기능을 NestJS에서 개발할 때 추가한다.

---

## 8. 검증 체크리스트

- [ ] NestJS 프로젝트 생성 성공 (`apps/api/`)
- [ ] `SupabaseService`로 DB 연결 테스트
- [ ] 헬스체크 엔드포인트 (`GET /health`) 응답
- [ ] CORS 설정 확인 (프론트엔드에서 호출 가능)
- [ ] `pnpm dev:api`로 개발 서버 정상 시작 (포트 3001)
- [ ] 기존 Next.js API routes 정상 동작 유지 확인
