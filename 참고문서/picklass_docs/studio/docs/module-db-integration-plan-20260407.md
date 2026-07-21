# 수업모듈 DB 연동 계획 (ai_modules 테이블)

**작성일**: 2026-04-07  
**버전**: 1.0  
**관련 페이지**: `/course` (과정 생성 마법사 Step 2), `/course/[courseId]` (과정 상세)

---

## 1. 현황 분석

### 현재 구조

**프론트엔드**: `MODULE_CATEGORIES` 상수 (`packages/shared/src/constants/index.ts`)에 18개 모듈이 하드코딩

```
Vocabulary (5): WRD, WSD, IMG, MG, WW
Reading (7):    PRD, SCN, SKM, QAR, CLR, SUM, ORL
Speaking (6):   WSP, LR, SXP, SHD, SNR, FRT
```

**DB 저장**: `course_lessons.skill_modules`에 JSON 배열로 코드만 저장 (예: `["WRD", "SCN"]`)

**문제점**:
- 모듈 추가/삭제/수정 시 코드 배포 필요 (상수 하드코딩)
- `ai_modules` 테이블의 풍부한 메타데이터 (레벨 적합도, 선행 조건, 비호환 모듈 등) 활용 불가
- backoffice에서 모듈을 관리해도 studio에 반영되지 않음

### ai_modules 테이블 구조 (backoffice Prisma 정의)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| skill | VARCHAR(50) | 카테고리 (Vocabulary, Reading, Speaking) |
| code | VARCHAR(10) | 모듈 코드 (WRD, SCN 등) - UNIQUE |
| name | VARCHAR(200) | 모듈 이름 |
| class_before / class_middle / class_after | Boolean | 수업 배치 설정 |
| open_before / open_after | Boolean | 공개 배치 설정 |
| priority | Int | 우선순위 |
| roles | String[] | 역할 |
| passage_exposure | VARCHAR(20) | 지문 노출 방식 |
| cognitive_level | VARCHAR(5) | 인지 수준 |
| suitable_level_min / suitable_level_max | VARCHAR(20) | 적합 CEFR 레벨 범위 |
| estimated_minutes_min / estimated_minutes_max | VARCHAR(10) | 예상 소요 시간 |
| prerequisites | String[] | 선행 모듈 코드 |
| incompatible_with | String[] | 비호환 모듈 코드 |
| status | VARCHAR(20) | active / inactive |
| created_at / updated_at / deleted_at | Timestamp | 시간 |

> 이 외에도 question_data, content_config 등 AI 운영 설정이 있지만, studio 모듈 선택 UI에서는 위 컬럼들만 필요.

---

## 2. 구현 목표

1. `ai_modules` 테이블을 studio API에서 조회할 수 있도록 Prisma 모델 추가
2. 모듈 목록 API 엔드포인트 제공
3. 프론트엔드에서 하드코딩된 `MODULE_CATEGORIES` 대신 API 데이터 사용
4. 레벨 적합도(`suitable_level_min/max`) 기반 모듈 자동 추천 고도화

---

## 3. 상세 구현 계획

### STEP 1: Prisma 스키마에 ai_modules 모델 추가

**파일**: `apps/api/prisma/schema.prisma`

기존 테이블을 읽기 전용으로 추가 (studio에서는 모듈을 생성/수정하지 않음).

```prisma
model ai_modules {
  id                   String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  skill                String   @db.VarChar(50)
  code                 String   @unique @db.VarChar(10)
  name                 String   @db.VarChar(200)
  class_before         Boolean  @default(false)
  class_middle         Boolean  @default(false)
  class_after          Boolean  @default(false)
  open_before          Boolean  @default(false)
  open_after           Boolean  @default(false)
  priority             Int      @default(1)
  roles                String[] @default([])
  passage_exposure     String   @default("") @db.VarChar(20)
  cognitive_level      String   @default("") @db.VarChar(5)
  suitable_level_min   String   @default("") @db.VarChar(20)
  suitable_level_max   String   @default("") @db.VarChar(20)
  estimated_minutes_min String  @default("") @db.VarChar(10)
  estimated_minutes_max String  @default("") @db.VarChar(10)
  prerequisites        String[] @default([])
  incompatible_with    String[] @default([])
  status               String   @default("active") @db.VarChar(20)
  created_at           DateTime @default(now()) @db.Timestamptz(6)
  updated_at           DateTime @default(now()) @db.Timestamptz(6)
  deleted_at           DateTime? @db.Timestamptz(6)

  @@index([skill])
  @@index([status])
  @@index([code])
}
```

> 테이블이 이미 Supabase에 존재하므로 마이그레이션 없이 `prisma db pull` 또는 수동 모델 추가로 처리.

### STEP 2: 모듈 API 모듈 생성 (백엔드)

**새 파일:**
- `apps/api/src/modules/modules.module.ts`
- `apps/api/src/modules/modules.controller.ts`
- `apps/api/src/modules/modules.service.ts`

**엔드포인트:**

```
GET /modules
  Query: status? (기본값: active)
  Response: {
    data: AiModuleItem[],
    categories: { [skill: string]: AiModuleItem[] }
  }
```

**AiModuleItem 타입** (shared에 추가):

```typescript
interface AiModuleItem {
  id: string;
  skill: string;               // 카테고리 (Vocabulary, Reading, Speaking)
  code: string;                // 모듈 코드
  name: string;                // 모듈 이름
  priority: number;            // 정렬 우선순위
  suitable_level_min: string;  // 적합 최소 레벨
  suitable_level_max: string;  // 적합 최대 레벨
  prerequisites: string[];     // 선행 모듈
  incompatible_with: string[]; // 비호환 모듈
}

interface ModuleListResponse {
  data: AiModuleItem[];
  categories: Record<string, AiModuleItem[]>;
}
```

### STEP 3: 프론트엔드 훅 및 API 클라이언트 추가

**파일**: `apps/web/src/lib/api.ts`

```typescript
modulesApi: {
  list: () => request<ModuleListResponse>('/modules'),
}
```

**파일**: `apps/web/src/hooks/use-modules.ts` (신규)

```typescript
export function useModulesList() {
  return useQuery({
    queryKey: ['modules'],
    queryFn: () => modulesApi.list(),
    staleTime: 5 * 60 * 1000,  // 5분 캐시 (자주 변경되지 않음)
  });
}
```

### STEP 4: 과정 생성/상세 페이지 모듈 선택 UI 수정

**수정 파일:**
- `apps/web/src/app/course/page.tsx` — 과정 생성 마법사 Step 2
- `apps/web/src/app/course/[courseId]/page.tsx` — 과정 상세 레슨 모듈 선택

**변경 내용:**
1. `MODULE_CATEGORIES` import 제거 (하드코딩 상수)
2. `useModulesList()` 훅으로 API 데이터 조회
3. API 응답의 `categories`를 기존 UI 구조에 매핑
4. `getRecommendedModules()` 함수를 `suitable_level_min/max` 기반으로 고도화

**현재 추천 로직 (하드코딩):**
```
A1-A2: WRD, IMG, WW (어휘 위주)
B1:    WRD, SCN, SKM (읽기+어휘)
B2+:   WRD, SCN, WSP, SHD (읽기+말하기+어휘)
```

**변경 후 추천 로직 (DB 기반):**
```
해당 CEFR 레벨이 suitable_level_min ~ suitable_level_max 범위에 포함되는 모듈 필터
→ priority 순 정렬
→ prerequisites 충족 여부 검증
→ incompatible_with 충돌 검사
```

### STEP 5: MODULE_CATEGORIES 상수 정리

**파일**: `packages/shared/src/constants/index.ts`

- `MODULE_CATEGORIES` 상수는 **fallback 용도**로 유지 (API 장애 시)
- 주석으로 "DB 연동 후 fallback 전용" 표기
- `ALL_MODULES`, `getModuleByCode()`, `getCategoryByModuleCode()` 유틸은 유지 (API 데이터로도 동작하도록 인터페이스 통일)

---

## 4. 수정 대상 파일 요약

| 파일 | 변경 내용 |
|------|----------|
| `apps/api/prisma/schema.prisma` | `ai_modules` 모델 추가 |
| `apps/api/src/modules/modules.module.ts` | 신규 NestJS 모듈 |
| `apps/api/src/modules/modules.controller.ts` | `GET /modules` 엔드포인트 |
| `apps/api/src/modules/modules.service.ts` | 모듈 목록 조회 서비스 |
| `apps/api/src/app.module.ts` | ModulesModule import 추가 |
| `packages/shared/src/types/module.ts` | `AiModuleItem`, `ModuleListResponse` 타입 |
| `packages/shared/src/index.ts` | module 타입 export |
| `apps/web/src/lib/api.ts` | `modulesApi.list()` 추가 |
| `apps/web/src/lib/react-query.ts` | `modules` queryKey 추가 |
| `apps/web/src/hooks/use-modules.ts` | `useModulesList()` 훅 (신규) |
| `apps/web/src/app/course/page.tsx` | 모듈 선택 UI를 API 데이터로 변경 |
| `apps/web/src/app/course/[courseId]/page.tsx` | 모듈 선택 UI를 API 데이터로 변경 |

---

## 5. 영향 범위

- **기존 course_lessons.skill_modules**: 변경 없음 (여전히 코드 문자열 배열로 저장)
- **ai_modules 테이블**: 읽기 전용 (studio에서 INSERT/UPDATE/DELETE 하지 않음)
- **backoffice**: 영향 없음 (같은 테이블을 공유하되 studio는 읽기만)
- **기존 과정 데이터**: 영향 없음 (저장된 코드 값은 동일)

---

## 6. 고려사항

1. **테이블 공유**: `ai_modules`는 backoffice에서 관리하는 테이블. studio Prisma에 모델을 추가하되 마이그레이션은 하지 않음 (이미 존재하는 테이블).
2. **캐싱**: 모듈 목록은 자주 변경되지 않으므로 프론트엔드에서 5분 staleTime 적용.
3. **Fallback**: API 실패 시 기존 `MODULE_CATEGORIES` 상수를 fallback으로 사용.
4. **레벨 비교**: CEFR 레벨 비교 로직 필요 (A1- < A1 < A1+ < A2- ... < C2+). 기존 `CEFR_LEVELS` 배열의 인덱스 기반으로 비교 가능.
