# Learn Explore — 과정 탐색 카테고리 구현 개발계획

> **작성일**: 2026-06-20
> **서비스**: speaking.picklass.com
> **연관 문서**:
> - [과정 카테고리 2-Depth 백오피스 개발계획](../../../backoffice/features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md)
> - [Learn 페이지 실 API 연동](../learn페이지-실API연동/20260619_learn페이지_실API연동_개발계획.md)

---

## 1. 배경 및 목적

픽클래스 기본 B2B 플랫폼은 **액세스 코드(`access_codes`)**로 수강을 관리한다 — 학습자는 코드에
연결된 **단일 과정**을 수강하는 것이 기본이다.

파고다와 같은 파트너는 이와 다르게 **모든 과정을 수강할 수 있는 방식**으로 운영한다. 이때 수강 권한은
access_codes가 아니라 **외부 API로 받는 별도 수강 내역 데이터**로 부여된다(§4.1 참조). 이 경우 학습자에게
노출되는 과정이 많아지므로, 자신에게 맞는 유용한 과정을 쉽게 탐색하도록 **2-Depth 카테고리**를 제공한다.
파고다의 경우 1-Depth(분류) / 2-Depth(교재 시리즈) 구성이다.

이를 위해 `/learn/explore` 페이지의 하드코딩 Mock 데이터(`COURSES`, `CATEGORIES`, `RECOMMENDED`)를
제거하고, backoffice에서 구축한 **2-Depth 과정 카테고리** (`course_categories` 테이블)를 실 데이터로
연동하여 L1/L2 이중 칩 UI로 과정 탐색 필터링을 구현한다.

### 1.1 개발 갈래 (2가지)

| 구분 | 조건 | 동작 |
|------|------|------|
| **① 파트너 카테고리 있음** | 파트너 레벨 계정에 카테고리 설정 존재 | "과정 탐색하기" 진입 + 이중 칩 카테고리 탐색 구현. 외부 API 기반 수강 내역으로 **모든 과정을 수강할 수 있는** 사용자가 진입 가능 |
| **② 파트너 카테고리 없음** | 파트너 레벨 계정이 없는 픽클래스 기본 B2B (카테고리 설정 없음) | "과정 탐색하기" 버튼 자체를 **노출하지 않음** (단일 과정 access_code 수강자) |

> **진입 가능 조건**: 파트너가 부여한 **외부 API 기반 수강 내역**으로 전체 과정을 수강할 수 있는 사용자.
> 단일 과정만 `access_code`로 보유한 픽클래스 기본 B2B 사용자에게는 "과정 탐색하기"가 노출되지 않는다.

---

## 2. 현황 분석

### 2.1 현재 문제점

| 레이어 | 파일 | 문제 |
|--------|------|------|
| 프론트엔드 | `apps/web/src/app/(tabs)/learn/explore/page.tsx` | COURSES·CATEGORIES·RECOMMENDED 하드코딩, 실 API 없음 |
| 프론트엔드 서비스 | `apps/web/src/lib/services/coursesService.ts` | explore용 `getExploreCourses()` 없음, 카테고리 서비스 없음 |
| 백엔드 | `apps/api/src/courses/courses.service.ts` | `getCourses()`에 `l1_category_id`/`l2_category_id` 필터 없음 |
| 백엔드 | `apps/api/src/` | `course-categories` 모듈 자체가 없음 (speaking API에 미이식) |

### 2.2 현재 explore 페이지 구조

```
헤더 (뒤로가기 + "과정 탐색")
검색창
L1 칩 1행: [전체][비즈니스][일상][여행][시험]   ← 하드코딩
─────────────────────────────────────────────
추천 과정 섹션 (전체·미검색 시만)
전체 과정 / 검색결과 섹션
```

---

## 3. 목표 UI — Option A (이중 칩)

```
헤더 (뒤로가기 + "과정 탐색")
검색창
L1 칩 1행: [전체][비즈니스][일상][여행][시험]   ← DB course_categories (depth=1)
L2 칩 2행: [전체][패턴영어 1권][카페영어][…]    ← L1 선택 시 표시 (depth=2, L1 하위)
─────────────────────────────────────────────
추천 과정 섹션 (전체 선택·미검색 시만)
전체 과정 / 카테고리 결과 / 검색결과 섹션
```

### 3.1 칩 동작 규칙

| 상태 | L1 칩 | L2 칩 행 | 과정 목록 |
|------|-------|----------|----------|
| 초기 (전체) | "전체" 활성 | **숨김** | 추천 과정 + 전체 과정 |
| L1 선택 | 해당 L1 활성 | **표시** ("전체" 기본 선택) | 해당 L1 과정 전체 |
| L2 선택 | L1 활성 유지 | 해당 L2 활성 | 해당 L2 과정 |
| 검색 입력 | 변경 없음 | 변경 없음 | 검색어 + 활성 카테고리 교집합 |

### 3.2 예외 처리

- **파트너 없는 기관 (픽클래스 기본 B2B)**: 카테고리 설정 없음 → `/learn`에서 "과정 탐색하기" 버튼
  **자체를 노출하지 않음** (explore 페이지 진입 차단). genre_code 기반 fallback 칩은 사용하지 않는다.
- **비로그인**: 토큰 없음 → 카테고리 API 비인증 → 빈 배열 → "과정 탐색하기" 미노출
- **카테고리 데이터 없음**: 위와 동일하게 explore 진입 차단 (칩만 숨기는 것이 아니라 버튼 자체 미표시)
- **과정 없음**: explore 진입 후 해당 카테고리에 과정이 없을 때 "해당 카테고리에 과정이 없습니다." 빈 상태 표시

### 3.3 추천 과정 / 전체 과정 로직 (2026-06-23)

#### 추천 과정 (Option: 레벨 매칭 + 인기순)

- 표시 조건: **검색어 없음 + L1 미선택**일 때만 (`showRecommended`).
- 로직: **사용자와 동일 레벨(`levelCode`)의 과정 중 수강자수(`student_count`) 상위 3개**.
  - 목록은 백엔드가 `student_count DESC` 로 정렬해 내려주므로, 레벨 필터 후 앞 3개를 취한다.
  - **전체 과정(`rest`)에서는 추천 3개를 제외**해 중복 노출하지 않는다.
- ⚠️ **사용자 레벨 미구현**: 현재 `useAuth().user` 에 레벨 필드가 없어 **레벨 필터는 보류**(seam +
  `TODO`). `userLevel = null` 인 동안은 전체에서 인기순 상위 3개로 **폴백**한다. 레벨 연동 시
  `userLevel` 만 채우면 자동 적용된다.

#### 전체 과정 — 해당 파트너 카테고리 과정만 (파트너 스코프)

- "전체 과정" 목록은 **로그인 유저 파트너(`req.user.partnerId`)가 소유한 카테고리에 분류된 과정만**
  노출한다. 미분류 과정과 **타 파트너 카테고리 과정**은 나오지 않는다.
- 구현(`getCourses(filters, partnerId)`):
  1. `partnerId` 없으면 `[]` (explore 는 파트너 전용).
  2. 파트너 소유 **L1 카테고리 id 집합**을 raw SQL 로 조회
     (`course_categories WHERE partner_id=$1 AND is_active AND depth=1`).
  3. `where.l1_category_id: { in: 파트너L1ids }` 로 스코프. L1/L2 선택 시 스프레드가 특정 카테고리로 더 좁힘.
- **수정 이력**: 최초 구현은 `l1_category_id: { not: null }`(분류 여부만) 이었으나, **타 파트너(파고다)
  과정이 누출**되는 버그가 발견되어(2026-06-23) 파트너 소유 카테고리 id 기준으로 교정.
  - 실측: tim.choi 파트너 explore 전체과정 5개(파고다 3 + 본인 2) → 교정 후 **본인 2개만**(파고다 누출 0).
- `partnerId` 는 로그인 시 JWT 에 캐싱된 값을 그대로 사용(요청당 파트너 재리졸브 없음).

---

## 4. DB 구조 (기존, 변경 없음)

`course_categories` 테이블 (backoffice Phase 1에서 구축 완료):

```
depth=1 (L1)  partner_id=UUID  code='business'  label='비즈니스'  parent_id=NULL
depth=2 (L2)  partner_id=UUID  code='pattern1'  label='패턴영어 1권'  parent_id={L1.id}
```

`courses` 테이블에 `l1_category_id INT`, `l2_category_id INT` 컬럼 존재.

> **구현 노트(2026-06-22)**: `course_categories`·`institutions` 는 backoffice 소유 테이블로 **speaking
> `schema.prisma`에 모델이 없다**. 따라서 계획 §7의 Prisma client(`prisma.course_categories.findMany`)
> 대신, config.service 의 `access_codes`·`institution_settings` 패턴과 동일하게 **`$queryRawUnsafe`**로
> 읽는다. 반면 `courses` 는 speaking 모델이므로 DB에 이미 존재하는 `l1_category_id`/`l2_category_id`
> 스칼라 컬럼 2개만 모델에 추가했다(DDL 없음, introspection 동등). 관계 모델은 추가하지 않았다.

### 4.1 외부 수강 내역 API (차후 추가개발)

> ⚠️ **본 항목은 차후 추가개발 대상이다.** 현재 단계에서는 명세가 확정되지 않았으며,
> 파트너 수강 내역 연동은 별도 후속 작업으로 진행한다. 본 문서의 Phase 1~4(§7)는
> **카테고리 조회 + 과정 필터 + UI** 까지를 범위로 하고, 외부 수강 내역 연동은 §4.1의
> 인터페이스 가정 위에서 후속으로 결합한다.

**배경**: 파트너(파고다 등)는 픽클래스 기본 B2B의 `access_codes`와 달리, 학습자의 수강 권한을
**외부 API로 받는 별도 수강 내역 데이터**로 관리한다. 이 데이터가 "모든 과정 수강 가능" 여부와
explore 진입 가능 여부를 결정한다.

**현재 미확정 (TODO)**:

- [ ] 외부 수강 내역 API 엔드포인트 / 인증 방식 확정
- [ ] 응답 스키마 (수강 가능 과정 범위, 유효 기간 등) 확정
- [ ] speaking API 내 어댑터/프록시 모듈 설계 (직접 호출 vs 백엔드 경유)
- [ ] 캐싱/만료 정책

**임시 인터페이스 가정** (확정 시 교체):

```ts
// 후속 개발 시 실제 외부 API 스펙으로 대체
interface PartnerEnrollment {
  hasFullCourseAccess: boolean;   // 전체 과정 수강 가능 여부 → explore 진입 게이트
  // 향후 과정 범위·기간 등 필드 추가 예정
}
```

> 본 문서 Phase 1~4 구현 단계에서는 진입 게이트 판정을 **카테고리 존재 여부**(`l1List.length > 0`)로
> 대체한다(§7.5). 외부 수강 내역 API 확정 후, 이 게이트를 `hasFullCourseAccess` 기준으로 정식 교체한다.

---

## 5. API 설계

### 5.1 GET /course-categories (신설)

로그인 유저의 파트너 카테고리 트리를 반환한다.

> **구현 변경(2026-06-22)**: 파라미터를 받지 않는다. 서버는 **로그인 시 JWT 에 캐싱된 `req.user.partnerId`**
> 로 바로 카테고리를 조회한다(서비스 메서드 `getCategoriesForPartner(partnerId)`). 기관→파트너 재귀 CTE 는
> 로그인 1회로 이관돼 **요청당 재리졸브가 없다**(타 기관 조회 차단도 동일하게 보장). 프론트
> `getCourseCategories()`도 인자를 받지 않는다.
> 상세: [로그인 JWT partnerId 캐싱 설계](../../architecture/2026-06-22_로그인-JWT-partnerId-institutionId-캐싱.md).

```
GET /course-categories
Authorization: Bearer {token}        # partnerId 는 토큰에서 추출
```

**응답**:

```ts
// depth=1 목록, 각 L1에 depth=2 children 포함
[
  {
    id: 3,
    code: 'business',
    label: '비즈니스',
    sortOrder: 0,
    children: [
      { id: 11, code: 'pattern1', label: '패턴영어 1권', sortOrder: 0 },
      { id: 12, code: 'pattern2', label: '패턴영어 2권', sortOrder: 1 },
    ],
  },
  ...
]
```

- 파트너 없는 기관: `[]` 반환 (에러 아님)
- `is_active = true`인 카테고리만 반환
- `sort_order ASC` 정렬

### 5.2 GET /courses 파라미터 확장

기존 `level`, `genre` 외에 카테고리 ID 필터 추가.

```
GET /courses?l1CategoryId=3&l2CategoryId=11&search=패턴&limit=30
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `l1CategoryId` | number (선택) | L1 카테고리 ID 필터 |
| `l2CategoryId` | number (선택) | L2 카테고리 ID 필터 |
| `search` | string (선택) | 과정명 부분 검색 (`title ILIKE %search%`) |
| `limit` | number (선택) | 기본값 30 |

> **course_type 고정 필터(2026-06-23)**: explore 목록은 **`course_type IN ('speaking','integrated')`**
> 과정만 노출한다(파라미터 아님, 항상 적용). `courses.course_type` 기본값이 `'tutoring'`이라 필터가
> 없으면 tutoring 전용 과정이 섞여 노출되기 때문이다(실측: in_use 39개 중 tutoring 31 → 필터 후 8개).
> 수강내역(`getEnrolledCourses`)과 동일 기준이되, **NULL 은 포함하지 않는다**(speaking/integrated 만).

**응답 타입 추가 필드**:

> **구현 노트**: `/courses` 는 Prisma `findMany` 결과를 그대로 직렬화하므로 응답은 **snake_case**
> (`level_code`, `l1_category_id` …)다. camelCase 변환은 web 서비스 `getExploreCourses()`에서
> `ExploreCourseRow → ExploreCourse` 매핑으로 처리한다.

```ts
// web ExploreCourse (camelCase) — API 응답(snake_case)을 매핑한 결과
{
  id: string;
  title: string;
  description: string | null;
  levelCode: string;
  genreCode: string;
  totalLessons: number;
  thumbnailUrl: string | null;
  studentCount: number;
  l1CategoryId: number | null;   // 추가
  l2CategoryId: number | null;   // 추가
}
```

---

## 6. 변경 파일 목록

| 파일 | 변경 종류 | 내용 |
|------|-----------|------|
| `apps/api/prisma/schema.prisma` | **수정** | `courses` 모델에 `l1_category_id`/`l2_category_id` + `course_type` 스칼라 + 인덱스 추가 (DB 기존 컬럼 동기화) |
| `apps/api/src/course-categories/course-categories.service.ts` | **신설** | `getCategoriesForPartner(partnerId)` 카테고리 트리 조회 (**raw SQL** — backoffice 소유 테이블). 파트너 리졸브 CTE 는 로그인으로 이관됨 |
| `apps/api/src/course-categories/course-categories.controller.ts` | **신설** | `GET /course-categories` (partnerId는 `req.user.partnerId`=JWT 캐싱값) |
| `apps/api/src/course-categories/course-categories.module.ts` | **신설** | 모듈 등록 |
| `apps/api/src/app.module.ts` | **수정** | `CourseCategoriesModule` import 추가 |
| `apps/api/src/courses/courses.service.ts` | **수정** | `l1CategoryId`, `l2CategoryId`, `search`, `limit` 파라미터 + `partnerId` 인자, **`course_type IN ('speaking','integrated')` + `l1_category_id IN (파트너 소유 카테고리)` 스코프 필터**, 정렬 `student_count desc` |
| `apps/api/src/courses/courses.controller.ts` | **수정** | 쿼리 파라미터 3개 + `parsePositiveInt` 헬퍼, **`req.user.partnerId` 를 서비스에 전달** |
| `apps/web/src/lib/services/categoryService.ts` | **신설** | `getCourseCategories()` (인자 없음) |
| `apps/web/src/lib/services/coursesService.ts` | **수정** | `getExploreCourses()`, `ExploreCourse` 인터페이스, snake→camel 매핑 추가 |
| `apps/web/src/app/(tabs)/learn/explore/page.tsx` | **전면 재작성** | Mock 제거, 실 API 연동, 이중 칩 UI, 추천(레벨매칭+인기순, 레벨 seam)·전체 과정 분리 |
| `apps/web/src/app/(tabs)/learn/page.tsx` | **수정** | Phase 5 진입 게이트 — `canExplore` 시만 "과정 탐색하기" 노출 |

---

## 7. 작업 순서

### Phase 1 — 백엔드: 카테고리 조회 API

**신설 파일**: `apps/api/src/course-categories/`

**course-categories.service.ts 핵심 로직** (실제 구현 = raw SQL):

> **중요(2026-06-22 정정)**: 기관 계층이 `institution → group → partner` 처럼 **다단계**일 수 있다.
> `institution.parent_id`로 **한 단계만** 올라가면 중간 `group`(카테고리 없음)에서 멈춘다.
> 따라서 **재귀 CTE로 `type='partner'` 조상**을 찾아 partner_id 를 리졸브한다. (상세: §10.1 오류 기록)
>
> **갱신(2026-06-22 후속)**: 파트너 판정 CTE 는 **로그인 시 1회 수행해 JWT 의 `partnerId` 로 캐싱**하도록
> 이관됐다. 따라서 서비스는 `getCategoriesForInstitution()` → **`getCategoriesForPartner(req.user.partnerId)`**
> 로 바뀌었고, 아래 코드의 1단계(CTE)는 서비스에서 제거됐다(카테고리 조회만 남음).
> 상세: [로그인 JWT partnerId 캐싱 설계](../../architecture/2026-06-22_로그인-JWT-partnerId-institutionId-캐싱.md).

```ts
async getCategoriesForInstitution(institutionId: string | null): Promise<L1Category[]> {
  if (!institutionId) return [];

  // 1. parent_id 체인을 위로 거슬러 올라가 type='partner' 조상 리졸브 (다단계 계층 대응)
  const partnerRows = await this.prisma.$queryRawUnsafe<{ id: string }[]>(
    `WITH RECURSIVE ancestry AS (
       SELECT id, parent_id, type, 0 AS lvl FROM institutions WHERE id = $1::uuid
       UNION ALL
       SELECT i.id, i.parent_id, i.type, a.lvl + 1
       FROM institutions i JOIN ancestry a ON i.id = a.parent_id)
     SELECT id FROM ancestry WHERE type = 'partner' ORDER BY lvl LIMIT 1`,
    institutionId,
  );
  const partnerId = partnerRows[0]?.id;
  if (!partnerId) return [];  // 파트너 조상 없음 → 픽클래스 기본 B2B

  // 2. 해당 파트너의 활성 카테고리(L1+L2) 일괄 조회 후 트리 구성 (raw SQL)
  const rows = await this.prisma.$queryRawUnsafe<CategoryRow[]>(
    `SELECT id, parent_id, depth, code, label, sort_order
     FROM course_categories WHERE partner_id = $1::uuid AND is_active = true
     ORDER BY depth ASC, sort_order ASC`,
    partnerId,
  );
  // depth=2 를 parent_id 로 묶어 L1.children 에 매핑 (전체 코드는 소스 참조)
}
```

**course-categories.controller.ts** (실제 구현):

```ts
// GET /course-categories — partnerId 는 로그인 시 JWT 에 캐싱된 값 (요청당 재리졸브 없음)
@Get()
getCategories(@Req() req: AuthenticatedRequest) {
  return this.service.getCategoriesForPartner(req.user.partnerId);
}
```

> `JwtAuthGuard` 적용. `partnerId`는 쿼리 파라미터가 아니라 `req.user.partnerId`(JWT 캐싱값)에서 추출.

---

### Phase 2 — 백엔드: GET /courses 확장

**courses.service.ts `getCourses()` 수정**:

```ts
async getCourses(filters: {
  level?: string;
  genre?: string;
  l1CategoryId?: number;
  l2CategoryId?: number;
  search?: string;
  limit?: number;
} = {}) {
  return this.prisma.courses.findMany({
    where: {
      status_code: 'in_use',
      deleted_at: null,
      ...(filters.level && { level_code: filters.level }),
      ...(filters.genre && { genre_code: filters.genre }),
      ...(filters.l1CategoryId && { l1_category_id: filters.l1CategoryId }),
      ...(filters.l2CategoryId && { l2_category_id: filters.l2CategoryId }),
      ...(filters.search && { title: { contains: filters.search, mode: 'insensitive' } }),
    },
    select: {
      id: true, title: true, description: true,
      level_code: true, genre_code: true,
      total_lessons: true, thumbnail_url: true, student_count: true,
      l1_category_id: true,
      l2_category_id: true,
    },
    orderBy: { student_count: 'desc' },
    take: filters.limit ?? 30,
  });
}
```

**courses.controller.ts `getCourses()` 수정**:

```ts
@Get()
getCourses(
  @Query('level') level?: string,
  @Query('genre') genre?: string,
  @Query('l1CategoryId') l1CategoryId?: string,
  @Query('l2CategoryId') l2CategoryId?: string,
  @Query('search') search?: string,
) {
  return this.coursesService.getCourses({
    level,
    genre,
    l1CategoryId: l1CategoryId ? parseInt(l1CategoryId, 10) : undefined,
    l2CategoryId: l2CategoryId ? parseInt(l2CategoryId, 10) : undefined,
    search,
  });
}
```

---

### Phase 3 — Web 서비스 레이어

**신설**: `apps/web/src/lib/services/categoryService.ts`

```ts
import { authFetch } from '../authFetch';

const api = () => process.env.NEXT_PUBLIC_SPEAKING_API_URL ?? '';

export interface L2Category {
  id: number;
  code: string;
  label: string;
  sortOrder: number;
}

export interface L1Category {
  id: number;
  code: string;
  label: string;
  sortOrder: number;
  children: L2Category[];
}

// 실제 구현: 인자 없음 — partnerId 는 서버가 JWT 에서 추출 (§5.1 구현 변경)
export async function getCourseCategories(): Promise<L1Category[]> {
  const res = await authFetch(`${api()}/course-categories`);
  if (!res.ok) return [];
  return res.json() as Promise<L1Category[]>;
}
```

**수정**: `apps/web/src/lib/services/coursesService.ts` — 함수 추가

```ts
export interface ExploreCourse {
  id: string;
  title: string;
  description: string | null;
  levelCode: string;
  genreCode: string;
  totalLessons: number;
  thumbnailUrl: string | null;
  studentCount: number;
  l1CategoryId: number | null;
  l2CategoryId: number | null;
}

export async function getExploreCourses(params: {
  l1CategoryId?: number;
  l2CategoryId?: number;
  search?: string;
} = {}): Promise<ExploreCourse[]> {
  const query = new URLSearchParams();
  if (params.l1CategoryId) query.set('l1CategoryId', String(params.l1CategoryId));
  if (params.l2CategoryId) query.set('l2CategoryId', String(params.l2CategoryId));
  if (params.search)       query.set('search', params.search);

  const res = await authFetch(`${api()}/courses?${query.toString()}`);
  if (!res.ok) throw new Error('과정 목록 조회 실패');
  return res.json() as Promise<ExploreCourse[]>;
}
```

---

### Phase 4 — 프론트엔드: explore/page.tsx 재작성

**상태 구조**:

```ts
const { user, isAuthenticated } = useAuth();
const [l1List, setL1List]               = useState<L1Category[]>([]);
const [selectedL1, setSelectedL1]       = useState<L1Category | null>(null);
const [selectedL2, setSelectedL2]       = useState<L2Category | null>(null);
const [searchQuery, setSearchQuery]     = useState('');
const [courses, setCourses]             = useState<ExploreCourse[]>([]);
const [categoriesLoading, setCategoriesLoading] = useState(true);
const [coursesLoading, setCoursesLoading]       = useState(true);
```

**데이터 로드 흐름**:

```
마운트
  → getCourseCategories() → setL1List   // 인자 없음, partnerId 는 서버가 JWT 에서 추출
  → getExploreCourses() → setCourses (초기: 전체, studentCount 내림차순)

L1 선택
  → setSelectedL1 / setSelectedL2(null)
  → getExploreCourses({ l1CategoryId }) → setCourses

L2 선택
  → setSelectedL2
  → getExploreCourses({ l1CategoryId, l2CategoryId }) → setCourses

검색 입력 (debounce 300ms)
  → getExploreCourses({ l1CategoryId?, l2CategoryId?, search }) → setCourses
```

**추천 과정 섹션 표시 조건** (실제 구현 — §3.3 참조):

```ts
// 추천: 사용자 동일 레벨 + 수강자수 상위 3개. 레벨 미구현 → 전체에서 상위 3개로 폴백.
const userLevel: string | null = null;   // TODO: useAuth().user 레벨 연동 시 교체
const showRecommended = !searchQuery && !selectedL1;
const recommendedPool = userLevel ? courses.filter((c) => c.levelCode === userLevel) : courses;
const recommended = showRecommended ? recommendedPool.slice(0, 3) : [];
const recommendedIds = new Set(recommended.map((c) => c.id));
// 전체 과정에서 추천 3개는 제외 (중복 노출 방지)
const rest = showRecommended ? courses.filter((c) => !recommendedIds.has(c.id)) : courses;
```

**UI 렌더링 구조**:

```tsx
<div className="flex-1 overflow-y-auto bg-gray-50 pb-24">
  {/* 헤더 */}
  <Header onBack={router.back} title="과정 탐색" />

  <div className="px-4 pt-4 space-y-3">
    {/* 검색 */}
    <SearchInput value={searchQuery} onChange={setSearchQuery} />

    {/* L1 칩 */}
    <L1ChipRow
      items={l1List}
      selected={selectedL1}
      onSelect={handleL1Select}
    />

    {/* L2 칩 — L1 선택 시만 표시 */}
    {selectedL1 && selectedL1.children.length > 0 && (
      <L2ChipRow
        items={selectedL1.children}
        selected={selectedL2}
        onSelect={handleL2Select}
      />
    )}

    {/* 추천 과정 */}
    {showRecommended && recommended.length > 0 && (
      <CourseSection title="추천 과정" courses={recommended} />
    )}

    {/* 전체 / 필터 결과 */}
    <CourseSection
      title={sectionTitle}   // "전체 과정" | "비즈니스" | '"패턴" 검색결과' 등
      courses={rest}
      loading={coursesLoading}
      empty="해당 카테고리에 과정이 없습니다."
    />
  </div>
</div>
```

**L1/L2 칩 컴포넌트 공통 설계**:

```tsx
// ChipRow — L1, L2 공용
function ChipRow<T extends { id: number | string; label: string }>({
  items,
  selected,
  onSelect,
}: {
  items: T[];
  selected: T | null;
  onSelect: (item: T | null) => void;
}) {
  return (
    <div className="flex gap-2 overflow-x-auto pb-1 scrollbar-hide">
      <Chip label="전체" active={!selected} onClick={() => onSelect(null)} />
      {items.map((item) => (
        <Chip
          key={item.id}
          label={item.label}
          active={selected?.id === item.id}
          onClick={() => onSelect(item)}
        />
      ))}
    </div>
  );
}
```

---

### Phase 5 — 진입 게이트: "과정 탐색하기" 노출 제어

explore 페이지로 들어가는 `/learn` 진입점에서 노출 여부를 판정한다.

```ts
// learn/page.tsx (또는 진입점 컴포넌트)
const { isAuthenticated } = useAuth();
const [canExplore, setCanExplore] = useState(false);

useEffect(() => {
  if (!isAuthenticated) return;
  // 현 단계 게이트: 카테고리 존재 여부로 파트너 여부 판정 (partnerId 는 서버가 JWT 에서 추출)
  getCourseCategories().then((list) => {
    setCanExplore(list.length > 0);
  });
}, [isAuthenticated]);

// "과정 탐색하기" 버튼 — 파트너 카테고리가 있을 때만 노출
{canExplore && (
  <button onClick={() => router.push('/learn/explore')}>과정 탐색하기</button>
)}
```

> **차후 추가개발 (§4.1)**: 외부 수강 내역 API 확정 시, 게이트 판정을 카테고리 존재 여부에서
> `PartnerEnrollment.hasFullCourseAccess` 기준으로 교체한다. 카테고리 유무와 수강 권한을 분리해
> "카테고리는 있으나 전체 수강 권한이 없는" 케이스까지 정확히 차단한다.

---

## 8. 작업 체크리스트

> **구현 완료(2026-06-22)**: 코드 작성 + `typecheck`/`lint`(소스) + `nest build` 통과. `[ ]`로 남은 항목은
> 실행 환경에서의 수동 검증(curl/브라우저) 대기 항목이다.

### Phase 1 — 백엔드: 카테고리 API

- [x] `apps/api/src/course-categories/course-categories.service.ts` 신설 (raw SQL)
- [x] `apps/api/src/course-categories/course-categories.controller.ts` 신설
- [x] `apps/api/src/course-categories/course-categories.module.ts` 신설
- [x] `apps/api/src/app.module.ts` — `CourseCategoriesModule` import 추가
- [x] `pnpm --filter @speaking/api typecheck` 통과
- [ ] 로컬 테스트: `curl /course-categories` (JWT) 응답 확인

### Phase 2 — 백엔드: 과정 필터 확장

- [x] `courses.service.ts` — `l1CategoryId`, `l2CategoryId`, `search`, `limit` 파라미터 추가
- [x] `courses.service.ts` — `course_type IN ('speaking','integrated')` 고정 필터 추가 (2026-06-23)
- [x] `courses.controller.ts` — 쿼리 파라미터 파싱 추가 (문자열 → 양수 검증 `parsePositiveInt`)
- [x] `schema.prisma` — `courses`에 `l1_category_id`/`l2_category_id`/`course_type` 스칼라 + 인덱스 추가 → `prisma:generate`
- [x] `pnpm --filter @speaking/api typecheck` 통과
- [x] 로컬 검증: course_type 필터 — in_use 39개 중 speaking 6 + integrated 2 = 8개만 반환(tutoring 31 제외)
- [ ] 로컬 테스트: `curl /courses?l1CategoryId=3` 카테고리 필터 동작 확인

### Phase 3 — Web 서비스

- [x] `categoryService.ts` 신설 (`L1Category`, `L2Category`, `getCourseCategories()`)
- [x] `coursesService.ts` — `ExploreCourse` 인터페이스, `getExploreCourses()`, snake→camel 매핑 추가
- [x] `pnpm --filter @speaking/web typecheck` 통과

### Phase 4 — 프론트엔드

- [x] `explore/page.tsx` — Mock 데이터·상수 전체 제거
- [x] `explore/page.tsx` — `getCourseCategories()` 연동 (L1 칩 생성)
- [x] `explore/page.tsx` — L1 선택 시 L2 칩 행 표시
- [x] `explore/page.tsx` — `getExploreCourses()` 연동 (초기·L1·L2·검색 각 트리거)
- [x] `explore/page.tsx` — 추천 과정 섹션 (전체·미검색 시만; 레벨매칭+인기순, 레벨 seam/TODO, 전체에서 추천 제외) (2026-06-23)
- [x] `courses.service.ts` — 전체과정 파트너 스코프: `l1_category_id IN (파트너 소유 카테고리)` (2026-06-23). 최초 `IS NOT NULL` → 타 파트너 누출 버그 교정. 실측: tim.choi 5개(파고다 3 포함)→본인 2개만 |
- [x] `explore/page.tsx` — 빈 상태 처리
- [x] `explore/page.tsx` — 로딩 스피너
- [x] `explore/page.tsx` — 비로그인 처리 (카테고리 [] → 칩 없이 전체 과정 표시)
- [x] `pnpm --filter @speaking/web typecheck` 통과
- [ ] 로컬 테스트: L1 칩 → L2 칩 표시 확인
- [ ] 로컬 테스트: L2 선택 → 과정 목록 필터 확인
- [ ] 로컬 테스트: 검색 + L2 교집합 필터 확인

### Phase 5 — 진입 게이트

- [x] `learn/page.tsx` — `getCourseCategories()`로 `canExplore` 판정
- [x] `learn/page.tsx` — 카테고리 없으면 "과정 탐색하기" 버튼 미노출
- [ ] 로컬 테스트: 파트너 카테고리 있는 계정 → 버튼 노출 + explore 진입
- [ ] 로컬 테스트: 파트너 없는 픽클래스 기본 B2B 계정 → 버튼 미노출

### 차후 추가개발 (§4.1 — 외부 수강 내역 API)

- [ ] 외부 수강 내역 API 명세 확정 (엔드포인트·인증·응답 스키마)
- [ ] 게이트 판정을 `hasFullCourseAccess` 기준으로 교체

---

## 9. 의도적 제외

| 항목 | 이유 |
|------|------|
| 검색 debounce 최적화 | 초기 구현에서는 onChange 즉시 반영. 성능 이슈 발생 시 후속 작업 |
| 무한 스크롤 / 페이징 | `limit=30` 고정. 과정 수가 많아질 때 후속 작업 |
| 추천 과정 레벨 필터 | 사용자 레벨 미구현(`useAuth().user` 에 레벨 없음) → seam(`userLevel=null`)만 두고 인기순 폴백. 레벨 연동 시 활성화 (§3.3) |
| 카테고리 없는 기관 `genre_code` fallback 칩 | 요구사항 변경 — 카테고리 없으면 "과정 탐색하기" 버튼 자체를 미노출(§3.2, §7 Phase 5). fallback 칩 미사용 |
| 외부 수강 내역 API 연동 | 명세 미확정. 차후 추가개발(§4.1). 현 단계 진입 게이트는 카테고리 존재 여부로 대체 |
| 과정 상세 페이지 이동 | 현재 `router.back()`으로 임시 처리. 상세 페이지 기획 후 교체 |
| 수강 중 과정 배지 (`수강 중`) | `getExploreCourses`와 `getEnrolledCourses` 교집합 필요. 후속 작업 |

---

## 10. 오류 기록 (Troubleshooting)

### 10.1 과정 탐색하기 버튼 미노출 — 파트너 계층 다단계 리졸브 누락 (2026-06-22)

**증상 (Symptom)**

파트너(`팀초이파트너스`)에 속하고 카테고리 설정(L1 4 / L2 6, 총 10개)이 존재하는 사용자
(`tim.choi@oizi.net`)인데도 `/learn` 페이지에 **"과정 탐색하기" 버튼이 노출되지 않음**.
진입 게이트는 `getCourseCategories()` 결과가 비어 있지 않을 때만 버튼을 노출하므로
(`canExplore = list.length > 0`), 카테고리 API가 `[]`를 반환한 것이 원인.

**원인 (Root Cause)**

기관 계층이 **3단계**였다:

```
Test Academy (type=institution, parent_id → 팀초이그룹)
  └ 팀초이그룹   (type=group,       parent_id → 팀초이파트너스)
      └ 팀초이파트너스 (type=partner,  parent_id = null)   ← course_categories 10개 보유
```

`course-categories.service.ts` 의 파트너 리졸브가 `institution.parent_id`로 **한 단계만**
올라가 중간의 `group`(카테고리 0개)을 partner_id 로 사용 → `course_categories` 조회 결과 0건 → `[]`.

`course_categories.partner_id` 는 **항상 `type='partner'` 기관**을 가리키는데, 파트너는
사용자 기관의 **부모일 수도, 조부모(또는 그 이상)일 수도** 있다. 단일 `parent_id` 가정이 틀렸다.

**해결 방법 (Resolution)**

파트너 리졸브를 **재귀 CTE**로 변경 — parent_id 체인을 위로 거슬러 올라가
`type='partner'` 인 가장 가까운 조상을 찾는다 (§7 Phase 1 코드 참조).

- 수정 파일: `apps/api/src/course-categories/course-categories.service.ts`
- 검증: `tim.choi@oizi.net`(institution `00000000-…`) → partner `38a3bebb-…`(lvl=2) →
  카테고리 L1 4개 + L2 children 정상 반환. typecheck/lint 통과.
- **API 재시작 필요** (서비스 코드 변경).

**재발 방지 대책 (Prevention)**

- 기관 계층은 **institution / group / partner 다단계**임을 전제로 코드를 작성한다.
  "부모 = 파트너" 가정 금지.
- 파트너 소유 데이터(`course_categories`, 향후 외부 수강내역 등)를 조회할 때는
  **`type='partner'` 조상 리졸브**를 공통 규칙으로 사용한다.
- 카테고리/권한 관련 신규 기능은 **3단계 계층 계정**(예: `tim.choi@oizi.net`)으로 반드시 실데이터 검증.

체크리스트:

- [ ] 파트너 데이터 조회 시 단일 `parent_id`가 아니라 `type='partner'` 조상까지 거슬러 올라갔는가?
- [ ] 다단계 계층 계정으로 실제 응답을 확인했는가?
