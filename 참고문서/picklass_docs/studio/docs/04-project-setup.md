# Phase 4: 프로젝트 셋팅 (Turborepo Monorepo)

> 정리된 프로젝트를 Turborepo 모노레포로 재구성하고, 개발/빌드/배포 환경을 설정한다.

---

## 1. 최종 프로젝트 구조

```
studio.picklass.com/
├── apps/
│   ├── web/                          # Next.js 15 프론트엔드
│   │   ├── src/
│   │   │   ├── app/                  # App Router
│   │   │   │   ├── class/           # 교실/학습 모듈 (핵심)
│   │   │   │   ├── course/          # Course Hub (과정 관리)
│   │   │   │   ├── report/          # Class Report (학습 현황)
│   │   │   │   ├── legal/           # 법적 문서
│   │   │   │   ├── api/             # Next.js API routes (기존 기능 유지)
│   │   │   │   └── page.tsx         # 랜딩 페이지
│   │   │   ├── components/           # React 컴포넌트
│   │   │   │   ├── lesson/          # 학습 단계 컴포넌트 (18개)
│   │   │   │   ├── ui/              # shadcn/ui
│   │   │   │   ├── landing/         # 랜딩 페이지
│   │   │   │   └── oizi/            # 대안 랜딩
│   │   │   ├── hooks/                # Custom hooks (use-courses, use-reports 등)
│   │   │   ├── lib/                  # 유틸리티, api.ts (NestJS 클라이언트)
│   │   │   ├── types/                # 타입 선언
│   │   │   └── middleware.ts         # 미들웨어 (스텁)
│   │   ├── public/                   # 정적 파일
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── postcss.config.mjs
│   │   ├── components.json           # shadcn/ui 설정
│   │   ├── tsconfig.json
│   │   └── package.json              # @classsnap/web
│   │
│   └── api/                          # NestJS 백엔드 (새 기능 전용)
│       ├── src/
│       │   ├── main.ts
│       │   ├── app.module.ts
│       │   ├── common/               # Guards, Decorators, Pipes
│       │   ├── config/               # 환경변수 설정
│       │   ├── supabase/             # DB 연결 (재사용 준비)
│       │   └── health/               # 헬스체크
│       ├── test/
│       ├── nest-cli.json
│       ├── tsconfig.json
│       └── package.json              # @classsnap/api
│
├── packages/
│   └── shared/                       # 공유 패키지
│       ├── src/
│       │   ├── index.ts              # 모든 export 진입점
│       │   ├── types/
│       │   │   ├── database.types.ts # Supabase 자동생성 타입
│       │   │   ├── models.ts         # Text, AsyncTask, UserProfile 등
│       │   │   ├── strategic-reading.ts
│       │   │   └── async-task.ts
│       │   ├── constants/
│       │   │   └── index.ts          # BRAND_NAME, PRODUCT_NAME 등
│       │   └── utils/
│       │       └── text-analysis.ts  # extractWords, 빈도 분석
│       ├── tsconfig.json
│       └── package.json              # @classsnap/shared
│
├── supabase/                         # DB 마이그레이션 (루트에 유지)
│   ├── config.toml
│   └── migrations/
│       ├── 20250710035319_initial_schema.sql
│       ├── 20250115000000_add_tts_cache.sql
│       ├── 20251203000000_add_word_count.sql
│       └── 20251204000000_add_text_type_origin.sql
│
├── docs/                             # 프로젝트 문서
│
├── turbo.json                        # Turborepo 설정
├── pnpm-workspace.yaml               # pnpm 워크스페이스
├── package.json                      # 루트 package.json
├── .gitignore
└── tsconfig.json                     # 루트 TypeScript 설정 (선택)
```

---

## 2. 루트 설정 파일

### 2.1 `package.json` (루트)

```json
{
  "name": "classsnap",
  "private": true,
  "packageManager": "pnpm@10.12.4",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "dev:web": "turbo dev --filter=@classsnap/web",
    "dev:api": "turbo dev --filter=@classsnap/api",
    "build:web": "turbo build --filter=@classsnap/web",
    "build:api": "turbo build --filter=@classsnap/api",
    "clean": "turbo clean"
  },
  "devDependencies": {
    "turbo": "^2"
  }
}
```

### 2.2 `pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### 2.3 `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

---

## 3. 각 패키지 설정

### 3.1 `apps/web/package.json`

```json
{
  "name": "@classsnap/web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev --port 3000",
    "build": "next build",
    "start": "next start",
    "lint": "eslint ."
  },
  "dependencies": {
    "@classsnap/shared": "workspace:*",
    "next": "15.1.11",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@supabase/ssr": "^0.5.2",
    "@supabase/supabase-js": "^2.87.1",
    "@tanstack/react-query": "^5.80.5",
    "@tanstack/react-query-devtools": "^5.80.5",
    "... (기존 프론트엔드 dependencies 유지, paddle 제외)"
  }
}
```

#### `apps/web/next.config.ts` 수정 사항

```typescript
const nextConfig: NextConfig = {
  // 기존 설정 유지
  serverExternalPackages: ['@google/genai'],
  compress: true,
  poweredByHeader: false,

  // 추가: shared 패키지 트랜스파일
  transpilePackages: ['@classsnap/shared'],

  // 기존 webpack 설정 유지
  webpack: (config, { dev, isServer }) => {
    // ...
  },
};
```

### 3.2 `apps/api/package.json`

> 최소 구성. AI(Gemini), TTS(Azure) 등의 패키지는 해당 기능을 NestJS에서 개발할 때 추가한다.

```json
{
  "name": "@classsnap/api",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "nest start --watch --port 3001",
    "build": "nest build",
    "start": "node dist/main",
    "start:prod": "node dist/main",
    "lint": "eslint .",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  },
  "dependencies": {
    "@classsnap/shared": "workspace:*",
    "@nestjs/common": "^11",
    "@nestjs/core": "^11",
    "@nestjs/config": "^4",
    "@nestjs/platform-express": "^11",
    "@supabase/supabase-js": "^2.87.1",
    "class-validator": "^0.14",
    "class-transformer": "^0.5",
    "reflect-metadata": "^0.2",
    "rxjs": "^7"
  }
}
```

### 3.3 `packages/shared/package.json`

```json
{
  "name": "@classsnap/shared",
  "version": "0.1.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "build": "tsc --noEmit",
    "lint": "eslint ."
  },
  "devDependencies": {
    "typescript": "^5"
  }
}
```

> **Note**: `main`과 `types`를 소스에 직접 연결하여 별도 빌드 없이 사용. Turborepo가 의존성 그래프를 관리.

---

## 4. Shared 패키지 내용

### 4.1 이동할 파일 매핑

| 원본 | 대상 |
|------|------|
| `src/lib/types/database.types.ts` | `packages/shared/src/types/database.types.ts` |
| `src/lib/types.ts` (타입 부분) | `packages/shared/src/types/models.ts` |
| `src/lib/types/async-task.ts` | `packages/shared/src/types/async-task.ts` |
| `src/lib/constants.ts` | `packages/shared/src/constants/index.ts` |
| `src/lib/utils/textAnalysis.ts` (순수 함수만) | `packages/shared/src/utils/text-analysis.ts` |

### 4.2 Shared에 포함되는 것

- **타입**: Database, Text, AsyncTask, UserProfile, StrategicReadingData, GenerationStatus 등
- **상수**: BRAND_NAME, PRODUCT_NAME, COMPANY_NAME
- **순수 유틸 함수**: extractWords, calculatePassageFrequencyLevel (Supabase 의존 없는 함수만)

### 4.3 Shared에 포함되지 않는 것

- Supabase 클라이언트 생성 로직
- React hooks, 컨텍스트
- NestJS 서비스, 컨트롤러
- 런타임 외부 서비스 의존 함수 (TTS 캐시 저장/조회 등)

### 4.4 `packages/shared/src/index.ts`

```typescript
// Types
export * from './types/database.types';
export * from './types/models';
export * from './types/async-task';

// Constants
export * from './constants';

// Utils
export * from './utils/text-analysis';
```

---

## 5. 환경변수 구성

### 5.1 `apps/web/.env.local`

```env
# Supabase (클라이언트용 - anon key)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# NestJS 백엔드 URL
NEXT_PUBLIC_API_URL=http://localhost:3001

# 브랜드 설정
NEXT_PUBLIC_PRODUCTNAME=ClassSnap
NEXT_PUBLIC_THEME=theme-englishai

# Analytics (선택)
NEXT_PUBLIC_GOOGLE_TAG=G-XXXXXXX
```

### 5.2 `apps/api/.env`

```env
# Supabase (서버용 - service_role key)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# AI (Gemini 2.5 Flash)
GEMINI_API_KEY=...

# 개발 시 토큰 없이 Mock 사용자 허용
ALLOW_MOCK_AUTH=true

# Server
PORT=3001
NODE_ENV=development
```

### 5.3 `.gitignore` 추가

```gitignore
# 환경변수
apps/web/.env.local
apps/api/.env

# Turborepo
.turbo

# Build outputs
apps/web/.next
apps/api/dist
```

---

## 6. API 아키텍처

### 6.1 기존 API 유지 (Next.js API Routes)

기존 Next.js API routes는 그대로 유지한다. class/ 모듈이 사용하는 20개 API는 변경 없이 동작한다.

```
apps/web/src/app/api/    # 그대로 유지 (NestJS로 마이그레이션하지 않음)
```

### 6.2 NestJS는 새 기능 전용

NestJS 백엔드(포트 3001)는 추후 새로운 기능 개발 시에만 사용한다. 프론트엔드에서 새 기능의 API를 호출할 때는 `NEXT_PUBLIC_API_URL` 환경변수를 통해 NestJS에 접근한다.

```
브라우저 ──/api/*──────> Next.js API Routes (기존 기능, 그대로 유지)
브라우저 ──NestJS URL──> NestJS 백엔드 (추후 새 기능)
```

---

## 7. Supabase Realtime 유지

Supabase Realtime은 **프론트엔드(web)에서 직접** Supabase에 연결한다.

```
브라우저 ──WebSocket──> Supabase (직접 연결, Realtime)
브라우저 ──HTTP──────> Next.js API Routes (기존 기능)
```

다음 파일들은 `apps/web/`에 그대로 유지:
- `src/lib/realtime/global-subscription.ts`
- `src/components/providers/RealtimeProvider.tsx`
- `src/lib/supabase/client.ts` (클라이언트용 Supabase 인스턴스)

---

## 8. 실행 순서

### Step 1: 모노레포 초기화
1. 루트 `package.json`, `pnpm-workspace.yaml`, `turbo.json` 생성
2. `apps/web/` 디렉토리 생성 및 현재 소스 이동
3. `packages/shared/` 생성 및 타입 추출
4. `pnpm install` → `turbo dev --filter=@classsnap/web` 동작 확인

### Step 2: 프론트엔드 정리 (Phase 1-2 실행)
1. /app/app/ 전체 삭제 (대시보드, text-management, user-settings, strategic-reading, fluency-reading)
2. /app/auth/ 전체 삭제, 결제(Paddle) 관련 삭제
3. components/strategic-reading/ 삭제, AppLayout 삭제
4. /app/class/ 학습 모듈 및 모든 의존성 유지 (lesson 컴포넌트, 훅, API routes)
5. useStrategicReading.ts 유지 (class/lesson에서 사용!), SubscriptionGuard 유지
6. Mock user 적용 (GlobalContext, ClassLayout, API routes)
7. `pnpm build:web` 성공 확인

### Step 3: NestJS 스캐폴딩 (Phase 3 실행)
1. `apps/api/` 프로젝트 생성
2. 최소 인프라 모듈 구현 (Config, Supabase, Health)
3. Auth Guard 스텁 구현
4. `pnpm dev:api` → 헬스체크 응답 확인

> 비즈니스 모듈은 추후 새 기능 개발 시 추가한다.

### Step 4: 통합 테스트
1. `turbo dev` — web(3000) + api(3001) 동시 시작
2. 기존 class/ 모듈 전체 기능 검증
3. NestJS 헬스체크 응답 확인

---

## 9. 배포 구성

### Vercel (프론트엔드)

```json
// apps/web/vercel.json
{
  "framework": "nextjs",
  "installCommand": "pnpm install",
  "buildCommand": "cd ../.. && turbo build --filter=@classsnap/web"
}
```

### 백엔드 배포 (추후)

NestJS 백엔드에 새 기능이 추가되면 배포한다. 옵션:
- **Railway** / **Render**: Node.js 서버 호스팅
- **AWS ECS / GCP Cloud Run**: 컨테이너 기반
- **Fly.io**: 글로벌 Edge 배포

Dockerfile 예시:
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY apps/api/dist ./dist
COPY apps/api/package.json .
RUN npm install --production
EXPOSE 3001
CMD ["node", "dist/main"]
```

---

## 10. 최종 검증 체크리스트

### 모노레포 구조
- [ ] `pnpm install` 루트에서 성공
- [ ] `turbo dev` — web + api 동시 시작
- [ ] `turbo build` — 양쪽 모두 빌드 성공
- [ ] `@classsnap/shared` 타입이 web, api에서 모두 import 가능

### 프론트엔드 (apps/web)
- [ ] 랜딩 페이지 (`/`) 정상 렌더링
- [ ] 교실 홈 (`/class`) — 텍스트 라이브러리, 지문 생성/조회
- [ ] 레슨 설정 (`/class/lesson-setup/[id]`) — 설정 UI
- [ ] 라이브 레슨 (`/class/lesson/[id]`) — 전체 학습 단계 순회
  - [ ] 전략적 읽기 분석 생성 (useGenerateStrategicReading)
  - [ ] 각 단계 피드백 (Prediction, Skimming, Scanning, Clarification, Summarizing, QAR)
  - [ ] TTS 음성 합성 (FluencyStep → useBatchTTS)
  - [ ] 단어 학습 (WordDeck, WordWeb)
  - [ ] 의미 추측 (MeaningGuessingStep)
- [ ] Supabase Realtime 연결 정상
- [ ] 기존 Next.js API routes 20개 정상 동작
- [ ] 삭제된 라우트 확인: `/app/*`, `/auth/*` → 404

### 백엔드 (apps/api)
- [ ] `GET /health` 헬스체크 응답
- [ ] SupabaseService DB 연결 확인
- [ ] CORS 설정 정상 (프론트엔드에서 호출 가능)
- [ ] `pnpm dev:api`로 포트 3001 정상 시작

### 개발 경험
- [ ] 코드 변경 시 핫리로드 동작 (web + api)
- [ ] TypeScript 에러 없음
- [ ] ESLint 경고/에러 없음
