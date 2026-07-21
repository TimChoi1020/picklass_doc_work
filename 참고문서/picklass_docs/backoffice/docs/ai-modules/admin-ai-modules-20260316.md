# admin-ai-modules (수업 모듈 관리 목록)

**페이지 경로:** `/admin/ai-modules`
**파일:** `apps/admin/frontend/src/app/(admin)/admin/ai-modules/page.tsx`
**작성일:** 2026-03-16

---

## 1. 사용자 흐름 (User Flow)

```
[수업 모듈 관리 페이지 진입]
    │
    ▼
[등록된 수업 모듈 목록 테이블 조회]
    │
    ├── [+ 모듈 등록 버튼 클릭]
    │       └──→ /admin/ai-modules/register (신규 등록)
    │
    └── [모듈 행에서 [편집] 버튼 클릭]
            └──→ /admin/ai-modules/register?id={module.id} (수정)
```

### 단계별 설명
1. 관리자가 `/admin/ai-modules`에 접근
2. 스킬 카테고리별(Vocabulary 4개 / Reading 7개 / Speaking 5개) 16개 모듈 전체 목록이 테이블로 표시
3. 각 모듈의 수업 위치(전/중/후), 지문 오픈(전/후), 우선순위, 상태를 한눈에 확인
4. 우측 상단 `+ 모듈 등록` 버튼으로 신규 등록 페이지 이동
5. 각 행 `편집` 버튼으로 해당 모듈 수정 페이지 이동 (`?id=` query param 전달)

---

## 2. IA 구조 정리 및 기능 정의

### 페이지 레이아웃 구조
```
수업 모듈관리 (h2)
└── [card] 등록된 수업 모듈
    ├── 헤더: 타이틀 + [+ 모듈 등록] 버튼 (우측)
    └── 모듈 목록 테이블
        ├── 헤더 행 1: 스킬 / 모듈명 / 수업(3칸) / 지문오픈(2칸) / 우선순위 / 상태 / 관리
        ├── 헤더 행 2: - / - / 전 / 중 / 후 / 오픈전 / 오픈후 / - / - / -
        └── 데이터 행 × 16
```

### 테이블 컬럼 정의
| 컬럼 | 목적 | 출력 형식 | 조건 |
|------|------|-----------|------|
| 스킬 | 언어 기술 카테고리 | 이모지+카테고리명 | Vocabulary / Reading / Speaking |
| 모듈명 | 모듈 식별 | `(코드)모듈명` | 3자 코드 + 전체명 |
| 수업 전/중/후 | 수업 내 활용 위치 | 체크박스(readOnly) | classBefore / classMiddle / classAfter |
| 지문 오픈전/후 | 지문 공개 타이밍 | 체크박스(readOnly) | openBefore / openAfter |
| 우선순위 | 동일 스킬 내 제공 순서 | 숫자 1~5 | 1이 최우선 |
| 상태 | 활성화 여부 | ✓ 활성(초록) / ⚠ 비활성(주황) | - |
| 관리 | 수정 진입 | [편집] 버튼 | 클릭 시 `?id=` 전달 |

### 등록 모듈 현황 (현재 하드코딩)
| 스킬 | 코드 | 모듈명 | 상태 |
|------|------|--------|------|
| 📚 Vocabulary | WRD | Word Reading Deck | 활성 |
| 📚 Vocabulary | WSD | Word Speaking Deck | 활성 |
| 📚 Vocabulary | IMG | Meaning Guessing | 활성 |
| 📚 Vocabulary | WW | Word Web | 활성 |
| 📖 Reading | PRD | Prediction | 활성 |
| 📖 Reading | SCN | Scanning | 활성 |
| 📖 Reading | SKM | Skimming | 활성 |
| 📖 Reading | QAR | Reading QAR | 활성 |
| 📖 Reading | CLR | Clarification | 활성 |
| 📖 Reading | SUM | Summarizing | 활성 |
| 📖 Reading | ORL | Oral Reading | **비활성** |
| 🎤 Speaking | WSP | Word Speaking | 활성 |
| 🎤 Speaking | LR | Listen & Repeat | 활성 |
| 🎤 Speaking | SXP | Sentence Expansion | 활성 |
| 🎤 Speaking | SHD | Shadowing | 활성 |
| 🎤 Speaking | SNR | Scenario Talking | 활성 |

---

## 3. 정책 (Policy / Business Rules)

### 모듈 분류 정책
- 모듈은 반드시 하나의 스킬에 속해야 함
- 모듈 코드는 3자 대문자 알파벳 고정 (WRD, PRD 등)
- 동일 스킬 내 우선순위(1~5)로 노출 순서 결정. 1이 최우선

### 수업 위치 정책
- **전(intro)**: 본 수업 시작 전 워밍업 단계
- **중(body)**: 본 수업 진행 중 활용
- **후(closure)**: 수업 마무리/복습 단계
- 하나의 모듈이 여러 위치에 중복 배치 가능

### 지문 오픈 정책
- **오픈전**: 지문 공개 전 활용 가능
- **오픈후**: 지문 공개 후 활용 가능
- 오픈전/후 중복 해당 가능

### 상태 정책
- **활성**: 수업에서 실제 사용 가능
- **비활성**: 등록은 되어 있으나 수업에서 비활성 (현재 ORL 비활성)

### 정책 변경사항 (이전 대비)
- `constants.ts`의 `SKILLS`에 Writing / Listening / Grammar가 포함되어 있으나 현재 목록 페이지에는 Vocabulary / Reading / Speaking 3개 카테고리만 존재 → 추후 스킬 확장 시 반영 필요
- `constants.ts`의 `MODULE_CATEGORIES`와 page.tsx 하드코딩 데이터 간 불일치 확인됨 (아래 기술 부채 참고)

---

## 4. 개발자 추가 작업 사항

### 하드코딩 → API 연동
```typescript
// 현재
const modules: AIModule[] = [ { id: '1', skill: '📚 Vocabulary', ... } ]

// 목표
const { data: modules } = useQuery(['ai-modules'], fetchModules)
```

### 필요 API
- `GET /api/admin/ai-modules` — 전체 모듈 목록
- `PATCH /api/admin/ai-modules/:id/status` — 상태 토글

### UI 개선
- 스킬 카테고리별 행 그룹 구분 (시각적 분리)
- 상태 직접 토글 (편집 페이지 진입 없이)
- 모듈 삭제 기능 (현재 없음)
- 우선순위 drag & drop 정렬
- 스킬별 필터/검색

---

## 5. 코드 규칙 (Coding Rules)

### 사용해야 하는 공통 요소
| 구분 | 경로 | 용도 |
|------|------|------|
| 상수 | `src/lib/constants.ts` → `SKILLS`, `LEVEL_SYSTEM`, `MODULE_CATEGORIES` | 스킬 목록, 레벨 정보, 모듈 분류 |
| 공통 컴포넌트 | `src/components/common/status-badge.tsx` | 활성/비활성 상태 표시 |
| 공통 컴포넌트 | `src/components/common/data-table.tsx` | 테이블 렌더링 |
| UI 컴포넌트 | `@/components/ui/*` (shadcn/ui) | Input, Button, Select, Checkbox 등 |
| 라우팅 | `useRouter` (Next.js) | 페이지 이동 |

### 금지 패턴
- `any` 타입 사용 금지 (`@ts-ignore` 포함)
- 모듈 데이터 페이지 내 하드코딩 금지 → 반드시 API 또는 constants 참조
- `alert()` 직접 사용 금지 → toast 컴포넌트 사용
- 인라인 스타일(`style={{ }}`) 사용 금지 → Tailwind 클래스 사용
- SKILLS, LEVEL_SYSTEM 재정의 금지 → constants.ts에서 import

### 파일 위치 규칙
```
타입 정의     → packages/types/ 또는 src/lib/types.ts
상수          → src/lib/constants.ts
공통 컴포넌트 → src/components/common/
모듈 전용 컴포넌트 → src/components/admin/module-*.tsx
페이지        → src/app/(admin)/admin/ai-modules/page.tsx
```

### 네이밍 규칙
- 인터페이스: `AIModule`, `ModuleFormData` (PascalCase, 의미 명확히)
- 컴포넌트 파일: `module-basic-info.tsx` (kebab-case)
- 컴포넌트 함수: `ModuleBasicInfo` (PascalCase)
- API 함수: `fetchModules`, `createModule`, `updateModule` (camelCase 동사+명사)

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 구분 | 문제 | 위치 | 심각도 |
|------|------|------|--------|
| 하드코딩 | 모듈 16개 데이터가 page.tsx 내부에 고정 | page.tsx:27~44 | 높음 |
| 데이터 불일치 | `MODULE_CATEGORIES`의 IMG="Image Match", page.tsx의 IMG="Meaning Guessing" | constants.ts vs page.tsx | 중간 |
| 누락 데이터 | constants에 FRT(Free Talking) 있으나 목록에 없음 | constants.ts:175 | 낮음 |
| 타입 중복 | `AIModule` 인터페이스가 page.tsx와 register/page.tsx에 각각 선언 | 두 파일 모두 | 중간 |
| 미구현 | 모듈 삭제 기능 없음 | - | 낮음 |
| 미구현 | 상태 직접 토글 없음 (편집 페이지 거쳐야 함) | - | 낮음 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 이 페이지가 사용하는 것
```
page.tsx
├── react: useState
├── next/navigation: useRouter
└── (하드코딩 데이터 - 향후 API로 교체)
```

### 이 페이지로의 진입점
- 사이드바 메뉴 → `ADMIN_MENU` (constants.ts) → `/admin/ai-modules`

### 이 페이지가 영향을 주는 곳
- `/admin/ai-modules/register` — 이 페이지에서 등록/편집 진입
- 수업 모듈이 활성화되면 → 학습 관리(`/studio/lesson-management`)에서 해당 모듈 사용 가능

### 관련 공통 컴포넌트 (현재 미사용, 리팩터링 시 활용)
- `src/components/common/data-table.tsx`
- `src/components/common/status-badge.tsx`
- `src/components/common/pagination.tsx`

---

## 8. DB/API 구조 (Data Contract)

### 현재 상태
DB에 ai_modules 테이블 없음. Backend에 ai-modules 모듈/컨트롤러 없음.
모든 데이터는 프론트엔드 하드코딩으로 처리 중.

### 목표 API 엔드포인트
```
GET    /api/admin/ai-modules           모듈 전체 목록
GET    /api/admin/ai-modules/:id       모듈 단건 조회
POST   /api/admin/ai-modules           모듈 등록
PUT    /api/admin/ai-modules/:id       모듈 수정
PATCH  /api/admin/ai-modules/:id/status  상태 토글
DELETE /api/admin/ai-modules/:id       모듈 삭제
```

### 타입 정의 전문
```typescript
// packages/types/ 또는 src/lib/types.ts 에 정의 예정

export interface AIModule {
  id: string;
  skill: string;           // 'vocabulary' | 'reading' | 'speaking' 등
  name: string;            // 전체 모듈명 (예: "Word Reading Deck")
  code: string;            // 3자 대문자 코드 (예: "WRD")
  classBefore: boolean;    // 수업 前 활용 여부
  classMiddle: boolean;    // 수업 中 활용 여부
  classAfter: boolean;     // 수업 後 활용 여부
  openBefore: boolean;     // 지문 오픈 前 활용 여부
  openAfter: boolean;      // 지문 오픈 後 활용 여부
  priority: number;        // 우선순위 1~5
  status: 'active' | 'inactive';
  createdAt?: string;
  updatedAt?: string;
}
```

### 목표 DB 테이블 (Prisma 설계 예시)
```prisma
model AiModule {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  skill       String   @db.VarChar(50)
  name        String   @db.VarChar(200)
  code        String   @unique @db.Char(3)
  classBefore Boolean  @default(false) @map("class_before")
  classMiddle Boolean  @default(false) @map("class_middle")
  classAfter  Boolean  @default(false) @map("class_after")
  openBefore  Boolean  @default(false) @map("open_before")
  openAfter   Boolean  @default(false) @map("open_after")
  priority    Int      @default(1)
  status      String   @default("active") @db.VarChar(20)
  createdAt   DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt   DateTime @updatedAt @map("updated_at") @db.Timestamptz

  levelConfigs AiModuleLevelConfig[]

  @@index([skill])
  @@index([status])
  @@map("ai_modules")
}
```

---

*작성일: 2026-03-16*
*페이지: admin/ai-modules*
