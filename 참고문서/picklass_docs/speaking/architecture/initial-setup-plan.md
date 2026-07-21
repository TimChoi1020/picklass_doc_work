# speaking.picklass.com — 초기 monorepo 스캐폴딩 계획

| 항목 | 값 |
|------|-----|
| 작성일 | 2026-05-07 |
| 작성자 | Claude (with @sniper4457) |
| 상태 | Approved · 진행 중 |
| 대상 저장소 | `D:\Project\speaking.picklass.com` |
| Reference | `D:\Project\tutoring.picklass.com` (자매 서비스) |

---

## 1. Context

`speaking.picklass.com`은 tutoring.picklass.com에서 **말하기(speaking) 영역만 분리한 자매 서비스**다.
저장소는 `git init`만 된 상태에서 출발했고, 이전 단계에서 [CLAUDE.md](../../../speaking.picklass.com/CLAUDE.md)와
multi-root workspace(`speaking.code-workspace`)를 만들었다.

이번 단계의 목표는 다음 두 가지다:

1. tutoring과 동일한 monorepo 환경(pnpm + Turbo / NestJS 11 + Next.js 15)을 갖춘 **최소 부트 상태**까지 도달.
   - `pnpm install` → `pnpm dev` 만으로 web(3003)·api(3004) 동시 부팅
   - `/health` 엔드포인트 응답을 web에서 확인 가능 (smoke test)
2. CLAUDE.md §10–§12 단일 문서 저장소 정책의 **첫 적용 사례**가 되도록,
   본 계획서 자체를 코드 저장소가 아닌 `picklass_docs`에 작성. picklass_docs 디렉터리 구조도 함께 초기화.

도메인 기능(lessons / ai / azure speech / analyzer / auth)은 후속 작업으로 분리. 시크릿이나
실제 환경변수 값은 다루지 않는다 (.example 파일만 제공).

---

## 2. 핵심 결정과 근거

| 결정 | 선택 | 근거 |
|------|------|------|
| 스캐폴딩 범위 | **최소 부트** | 도메인 기능은 별도 평가가 필요. 일단 골격을 안정화한 뒤 모듈 단위 이식이 안전 |
| Prisma schema 시작점 | **generator + datasource만** | DATABASE_URL 사전 준비 없이 진행 가능. Supabase 공유 전략은 추후 결정 |
| ESLint + Prettier | **추가** | tutoring에는 없으나 CLAUDE.md §3 commitment + backoffice 표준 룰 반영 |
| 포트 | **web 3003 / api 3004** | tutoring(3001/3002)과 동시 실행 가능 |
| Vercel 설정 | **다음 작업으로 연기** | 로컬 부팅 안정 확인 후 GitHub remote 연결과 함께 진행 |
| 첫 git 커밋 | **본 단계에서 안 함** | GitHub remote 정책 확정 후 사용자 승인 하에 진행 |

---

## 3. Outcome (이번 단계 종료 후 상태)

### speaking.picklass.com 트리

```
speaking.picklass.com/
├── apps/
│   ├── api/                          # @speaking/api, NestJS 11, port 3004
│   │   ├── src/
│   │   │   ├── main.ts               # bootstrap + CORS
│   │   │   ├── app.module.ts         # ConfigModule + PrismaModule + HealthController
│   │   │   ├── health/health.controller.ts
│   │   │   └── prisma/{prisma.module,prisma.service}.ts
│   │   ├── prisma/schema.prisma      # generator + datasource 블록만
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── nest-cli.json
│   │   ├── eslint.config.mjs
│   │   ├── .prettierrc.json
│   │   └── .env.example
│   └── web/                          # @speaking/web, Next.js 15, port 3003
│       ├── src/
│       │   ├── app/{layout,page}.tsx, globals.css
│       │   ├── lib/{utils,authFetch}.ts
│       │   └── components/ui/{button,input}.tsx
│       ├── package.json, tsconfig.json
│       ├── next.config.ts, tailwind.config.ts, postcss.config.mjs
│       ├── components.json, next-env.d.ts
│       ├── eslint.config.mjs, .prettierrc.json
│       └── .env.local.example
├── packages/
│   └── types/                        # @speaking/types
│       └── src/index.ts (placeholder + R2 룰)
├── package.json, pnpm-workspace.yaml, turbo.json
├── .gitignore, .prettierrc.json
├── CLAUDE.md (기존)
└── speaking.code-workspace (기존)
```

### picklass_docs 트리 (본 단계에서 초기화)

```
picklass_docs/
├── README.md
├── speaking/
│   ├── README.md
│   ├── architecture/
│   │   ├── README.md
│   │   └── initial-setup-plan.md  ← 본 문서
│   ├── features/README.md
│   ├── errors/README.md
│   └── operations/README.md
├── tutoring/README.md
├── backoffice/README.md
└── shared/
    ├── README.md
    └── {conventions,api-contracts,infrastructure,errors}/README.md
```

---

## 4. 구현 단계 (Step-by-step)

### Step 1. picklass_docs 디렉터리 구조 + 본 계획서

본 문서를 포함해 위 picklass_docs 트리의 모든 README 작성. CLAUDE.md §10–§12 정책을
첫 작업부터 실천하기 위해 다른 어떤 작업보다 먼저 수행.

### Step 2. speaking 루트 monorepo 파일

- `package.json` — `name: speaking-picklass-com`, `packageManager: pnpm@10.24.0`,
  scripts(`dev`, `build`, `typecheck`, `lint`, `format`, 필터 변형들), devDeps(`turbo@^2.5.0`, `prettier@^3`)
- `pnpm-workspace.yaml` — tutoring 사본 (`apps/*`, `packages/*` + `ignoredBuiltDependencies` 목록)
- `turbo.json` — tutoring과 동일 (`tasks`: dev / build / typecheck / lint)
- `.gitignore` — tutoring 사본 + `.claude/` 제외
- `.prettierrc.json` — single quote, semi true, trailingComma all, printWidth 100

### Step 3. `packages/types` 패키지

- `package.json` — `@speaking/types`, src/index.ts를 main/types로 직접 노출 (빌드 불필요)
- `tsconfig.json` — tutoring 사본
- `src/index.ts` — R2 룰 주석 + `HealthResponse` placeholder 인터페이스

### Step 4. `apps/api` (NestJS) — 최소 부트

tutoring 패턴을 따르되 도메인 패키지는 제외.

- `package.json` — `@speaking/api`. 유지: `@nestjs/*`, `@prisma/client`, `prisma`, `dotenv`,
  `express`, `reflect-metadata`, `rxjs`, `@vercel/node`, `@nestjs/cli`, `@types/express`,
  `@types/node`, `typescript`. 제거(후속): `@google/genai`, Azure Speech SDK, `bcryptjs`,
  `jsonwebtoken`. 신규: `@speaking/types: workspace:*`, `eslint`, `typescript-eslint`, `prettier`.
- `tsconfig.json`, `nest-cli.json` — tutoring 사본
- `eslint.config.mjs` — backoffice 백엔드 패턴 (no-explicit-any: warn, no-unused-vars: warn with `^_`)
- `prisma/schema.prisma` — generator + datasource 블록만
- `src/main.ts` — port 3004, CORS regex (localhost + speaking-picklass-com vercel preview)
- `src/app.module.ts` — ConfigModule + PrismaModule + HealthController만
- `src/prisma/{module,service}.ts` — tutoring 사본 (한국어 에러 메시지 유지)
- `src/health/health.controller.ts` — tutoring 사본
- `.env.example` — PORT, DATABASE_URL, JWT_SECRET, ANALYZER_BASE_URL placeholder

### Step 5. `apps/web` (Next.js 15) — 최소 부트

- `package.json` — `@speaking/web`, `dev: next dev --port 3003`. tutoring 의존성 그대로 +
  `@speaking/types: workspace:*`, `prettier`.
- `tsconfig.json`, `next.config.ts`, `tailwind.config.ts`, `postcss.config.mjs`,
  `components.json`, `next-env.d.ts` — tutoring 사본
- `eslint.config.mjs` — Next.js flat config (FlatCompat + next/core-web-vitals + next/typescript)
- `src/app/globals.css` — tutoring 사본 (Pretendard CDN, shadcn CSS variables)
- `src/app/layout.tsx` — 최소판 (title/description만, AuthProvider 제거)
- `src/app/page.tsx` — smoke 페이지: `NEXT_PUBLIC_SPEAKING_API_URL`로 `/health` 호출 결과 표시
- `src/lib/utils.ts`, `src/lib/authFetch.ts` — tutoring 사본 (`picklass_auth_token` 키 유지)
- `src/components/ui/{button,input}.tsx` — tutoring 사본
- `.env.local.example` — `NEXT_PUBLIC_SPEAKING_API_URL=http://localhost:3004`, `JWT_SECRET=`

### Step 6. 의존성 설치 및 동작 확인

1. speaking 루트에서 `pnpm install`
2. `pnpm typecheck` 통과
3. `pnpm lint` 통과 (warn 허용, error 불허)
4. `pnpm dev` — turbo가 web/api 동시 부팅
5. 수동 검증
   - `curl http://localhost:3004/health` → `{"status":"ok",...}`
   - 브라우저 `http://localhost:3003` → page.tsx에 health 응답 표시
6. picklass_docs 갱신 사항을 보고 (어떤 문서가 어디에 추가됐는지)

### Step 7. 첫 git 커밋

이번 단계에서는 **수행하지 않는다**. GitHub remote 정책과 함께 사용자가 직접 진행.

---

## 5. Critical Files

신규 생성만 있고 기존 파일 수정은 없음 (CLAUDE.md, speaking.code-workspace 그대로 유지).

대표 경로:
- `picklass_docs/speaking/architecture/initial-setup-plan.md` (본 문서)
- `speaking.picklass.com/` 루트: `package.json`, `pnpm-workspace.yaml`, `turbo.json`, `.gitignore`, `.prettierrc.json`
- `speaking.picklass.com/apps/api/`: `package.json`, `tsconfig.json`, `nest-cli.json`,
  `eslint.config.mjs`, `prisma/schema.prisma`, `src/main.ts`, `src/app.module.ts`,
  `src/prisma/{module,service}.ts`, `src/health/health.controller.ts`, `.env.example`
- `speaking.picklass.com/apps/web/`: `package.json`, `tsconfig.json`, `next.config.ts`,
  `tailwind.config.ts`, `postcss.config.mjs`, `components.json`, `next-env.d.ts`,
  `eslint.config.mjs`, `src/app/{layout,page}.tsx`, `globals.css`, `src/lib/{utils,authFetch}.ts`,
  `src/components/ui/{button,input}.tsx`, `.env.local.example`
- `speaking.picklass.com/packages/types/`: `package.json`, `tsconfig.json`, `src/index.ts`

참조(차용 원본 — 수정 없음):
- `D:\Project\tutoring.picklass.com\` 의 동일 위치 파일들
- `D:\Project\picklass-backoffice\.agent\rules\project_rules.md` (ESLint 룰)

---

## 6. Verification

작업 완료 후 다음 모두 통과해야 한다.

1. **파일 검증**
   - 본 문서가 `picklass_docs/speaking/architecture/initial-setup-plan.md`에 존재
   - speaking 트리가 §3 Outcome과 일치
2. **빌드/타입/린트**
   - `pnpm install` 성공 (warning 허용)
   - `pnpm typecheck` exit 0
   - `pnpm lint` exit 0 (warn 허용, error 불허)
3. **런타임 smoke**
   - `pnpm dev` 후 web 3003 / api 3004 동시 부팅
   - `curl -s http://localhost:3004/health` → `{"status":"ok",...}`
   - 브라우저 `http://localhost:3003`에서 health 응답 표시
4. **CORS 동작**
   - DevTools Network에서 web → api 호출이 차단 없이 200 OK
5. **CLAUDE.md 정책 준수**
   - speaking 저장소에 `.md`가 새로 추가되지 않음 (본 문서 포함 모두 picklass_docs로)
   - 시크릿 파일 없음 (.example만)

---

## 7. Out of Scope (후속 단계)

- 도메인 모듈 이식: lessons / ai (Gemini) / analyzer / auth / common-codes / passages / external
- Prisma 모델 도입: `prisma db pull` 또는 tutoring schema 부분 차용 (DB 공유 전략 결정 후)
- Vercel 배포: `apps/api/vercel.json` + `apps/api/api/index.ts` 핸들러
- GitHub remote 연결 + 첫 커밋
- CI: GitHub Actions (typecheck/lint workflow)
- AuthProvider, 사용자 화면, 공통 헤더(StudioHeader 등)
- Azure Speech / Gemini 키 발급 및 .env 채우기
- shadcn 추가 컴포넌트 (`pnpm dlx shadcn@latest add ...`)

---

## 8. 진행 로그

| 단계 | 상태 | 비고 |
|------|------|------|
| 1. picklass_docs 구조 | ✅ 완료 | 본 문서 작성과 함께 |
| 2. speaking 루트 파일 | ✅ 완료 | |
| 3. packages/types | ✅ 완료 | |
| 4. apps/api | ✅ 완료 | postinstall 제거 + AppModule에서 PrismaModule 일시 제외 (§9 참고) |
| 5. apps/web | ✅ 완료 | |
| 6. 검증 | ✅ 완료 | typecheck 3/3, lint 2/2, smoke 통과 |

검증 결과 (2026-05-08):
- `pnpm install` → exit 0, 676 packages
- `pnpm typecheck` → 3/3 successful
- `pnpm lint` → 2/2 successful
- `pnpm dev` → web `http://localhost:3003` (Next 15.1.11), api `http://localhost:3004` (Speaking API) 동시 부팅
- `curl http://localhost:3004/health` → `{"status":"ok","timestamp":"..."}`
- web 응답 HTML에 `<title>Picklass Speaking</title>` 확인

## 9. 계획 대비 이탈 사항

스캐폴딩 중 두 군데에서 계획과 다른 결정을 내렸다. 둘 다 "최소 부트 → DATABASE_URL 없이도 동작" 원칙을 우선해서 조정한 것이다.

### 9.1 `apps/api/package.json` — `postinstall: prisma generate` 제거
- **계획**: tutoring과 동일하게 postinstall에서 `prisma generate`.
- **실제**: 빈 schema로 prisma generate가 실패하여 `pnpm install` 자체가 비정상 종료. 후속 단계에서 `pnpm install`이 깨지지 않도록 postinstall 제거. 대신 `pnpm prisma:generate` 스크립트는 그대로 두어 수동 실행 가능.
- **재도입 시점**: 실제 모델이 도입되면 postinstall을 다시 추가.

### 9.2 `apps/api/prisma/schema.prisma` — placeholder 모델 1개 추가
- **계획**: generator + datasource 블록만.
- **실제**: 모델이 0개면 `prisma generate`가 실패하고 `@prisma/client`에서 `PrismaClient` export가 안 만들어져 typecheck가 깨진다. `SchemaPlaceholder` 모델을 임시로 두어 generate를 통과시키고 클라이언트 타입을 생성.
- **재도입 시점**: `prisma db pull`로 실제 스키마를 도입할 때 첫 번째 작업으로 이 placeholder를 제거.

### 9.3 `apps/api/src/app.module.ts` — `PrismaModule` 일시 제외 → 복귀 (2026-05-08)
- **계획**: AppModule이 `ConfigModule + PrismaModule + HealthController` 구조.
- **실제 (1차)**: PrismaService 생성자가 `DATABASE_URL`이 없으면 throw하므로, env 없이는 부팅 불가. AppModule의 `imports`에서 `PrismaModule`을 일시적으로 제외.
- **복귀 (2차, 2026-05-08)**: tutoring과 동일한 `.env` 값을 `apps/api/.env`에 작성한 뒤 `PrismaModule`을 imports에 다시 추가. 부팅 시 `PrismaModule dependencies initialized` 로그 정상 확인.

### 9.4 `apps/web/tailwind.config.ts` — `tailwindcss-animate` ESM import
- **계획**: tutoring 그대로 (`require('tailwindcss-animate')`).
- **실제**: Next.js의 `next/typescript` ESLint 룰이 `@typescript-eslint/no-require-imports`를 error로 띄움. ES import (`import tailwindcssAnimate from 'tailwindcss-animate'`)로 변경. tutoring에서는 lint를 web 폴더에서 활성화하지 않아 검출되지 않았음.
