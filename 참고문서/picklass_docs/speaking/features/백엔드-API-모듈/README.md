# 백엔드 API 모듈 (Phase 6)

| 항목 | 값 |
|------|-----|
| 작성일 | 2026-05-08 |
| 상태 | 완료 |
| 대상 저장소 | `speaking.picklass.com/apps/api` |
| 관련 단계 | Phase 3 (Prisma 스키마) + Phase 6 (3개 API 모듈) |

---

## 1. Prisma 스키마 (Phase 3)

tutoring과 동일한 Supabase 인스턴스를 공유하므로, `prisma db pull`(pgbouncer 트랜잭션 모드와 비호환) 대신 tutoring의 `schema.prisma`를 복사해 사용.

**경로**: `apps/api/prisma/schema.prisma`

복사된 모델 11개:

| 모델 | 설명 |
|------|------|
| `code_groups` | 공통 코드 그룹 |
| `code_items` | 공통 코드 항목 |
| `users` | 사용자 (roleCode, institutionId 포함) |
| `texts` | 학습 텍스트 |
| `text_analyses` | 텍스트 분석 결과 |
| `courses` | 과정 (level_code, genre_code, total_lessons) |
| `course_lessons` | 레슨 (lesson_order, topic, skill_modules) |
| `ai_modules` | AI 모듈 정의 |
| `module_questions` | 모듈 문항 |
| `module_histories` | 모듈 수행 이력 (kpis JSON) |
| `lesson_results` | 레슨 결과 (average_score, total_duration) |
| `external_access_tokens` | 핸드오프 토큰 |

> **주의**: Supabase를 tutoring과 공유하므로, 공통 테이블(`users`, `code_*`) 스키마 변경은 tutoring 팀과 사전 조율 필수.

---

## 2. 공통 타입

**경로**: `apps/api/src/common/types/request.types.ts`

```typescript
export interface RequestUser {
  id: string;         // UUID (users.id)
  userId: string;     // users.user_id
  email: string | null;
  name: string;
  roleCode: string;
  institutionId: string | null;
}

export interface AuthenticatedRequest extends Request {
  user: RequestUser;
}
```

모든 인증 필요 컨트롤러에서 `@Req() req: AuthenticatedRequest`로 타입 안전하게 사용.

---

## 3. 모듈 구조

3개 모듈 모두 동일한 패턴을 따른다:

```
src/{모듈명}/
├── {모듈명}.module.ts      # AuthModule import
├── {모듈명}.controller.ts  # @Controller + @UseGuards(JwtAuthGuard)
└── {모듈명}.service.ts     # 비즈니스 로직 + Prisma 쿼리
```

`apps/api/src/app.module.ts`에 UsersModule, CoursesModule, FeedbackModule 등록.

---

## 4. API 명세

모든 엔드포인트는 `Authorization: Bearer {JWT}` 헤더 필수.

### 4.1 Users

#### `GET /users/me`

로그인 유저의 프로필 반환.

**응답 예시**:
```json
{
  "id": "uuid",
  "name": "Tim Choi",
  "email": "tim.choi@oizi.net",
  "roleCode": "student",
  "statusCode": "active",
  "institutionId": null,
  "lastLoginAt": "2026-05-08T...",
  "lastActiveAt": "2026-05-08T...",
  "createdAt": "2026-01-01T..."
}
```

**서비스 파일**: `apps/api/src/users/users.service.ts:17`

---

### 4.2 Courses

#### `GET /courses?level=B1&genre=business`

전체 과정 목록 (status_code='in_use', deleted_at IS NULL). 최대 30개.

쿼리 파라미터:
- `level` (optional): CEFR 레벨 코드 (A1, A2, B1, B2, C1)
- `genre` (optional): 장르 코드 (business, travel, daily, exam, academic)

**응답 예시**:
```json
[
  {
    "id": "uuid",
    "title": "비즈니스 영어 핵심 표현 50",
    "description": "...",
    "level_code": "B1",
    "genre_code": "business",
    "total_lessons": 12,
    "thumbnail_url": null,
    "student_count": 234
  }
]
```

#### `GET /courses/in-progress`

로그인 유저가 학습 중인 과정 목록. lesson_results로부터 집계.

> ⚠️ 라우트 순서 주의: `GET /courses/:id` 앞에 선언해야 "in-progress"가 id 파라미터로 해석되지 않음.

**응답 예시**:
```json
[
  {
    "id": "uuid",
    "title": "비즈니스 영어 핵심 표현 50",
    "levelCode": "B1",
    "genreCode": "business",
    "totalLessons": 12,
    "completedLessons": 5,
    "lastScore": 78,
    "lastCompletedAt": "2026-05-07T...",
    "nextLessonOrder": 6,
    "nextLessonTopic": "미팅에서 의견 제안하기",
    "progress": 42
  }
]
```

집계 로직:
- lesson_results를 학생별로 최대 100개 조회
- 과정 ID별로 Map 집계
- `completedLessons`: completed_at이 있는 레코드 수
- `nextLessonOrder`: 가장 최근 완료 레슨 + 1
- `progress`: `Math.round(completedLessons / totalLessons * 100)`

**서비스 파일**: `apps/api/src/courses/courses.service.ts:31`

#### `GET /courses/:id`

특정 과정 + 레슨 목록 (lesson_order 오름차순).

---

### 4.3 Feedback

#### `GET /feedback/summary`

학생의 전체 학습 요약. `Promise.all`로 병렬 조회.

**응답 예시**:
```json
{
  "avgScore": 76,
  "totalLessons": 23
}
```

#### `GET /feedback/lesson-history?limit=10`

레슨 이력 목록. 최근 순.

쿼리 파라미터:
- `limit` (optional): 1–50, 기본값 10

**응답 예시**:
```json
[
  {
    "id": "uuid",
    "score": 82,
    "totalDuration": 1180,
    "completedAt": "2026-05-07T...",
    "lessonOrder": 5,
    "topic": "미팅에서 의견 제안하기",
    "skillModules": ["EDR", "RPL"],
    "course": {
      "id": "uuid",
      "title": "비즈니스 영어 핵심 표현 50",
      "genre_code": "business",
      "total_lessons": 12
    }
  }
]
```

#### `GET /feedback/kpi-trends?period=7d`

날짜별 5축 KPI 평균 추이. module_histories.kpis JSON 집계.

쿼리 파라미터:
- `period`: `7d`(기본) | `30d`

**응답 예시**:
```json
[
  { "date": "05-01", "pron": 72, "flu": 68, "gram": 81, "prag": 65, "vol": 77 },
  { "date": "05-02", "pron": 75, "flu": 70, "gram": 80, "prag": 67, "vol": 79 }
]
```

집계 로직:
- `module_histories.completed_at >= since` 조회
- 날짜 키: `toISOString().slice(5, 10)` → `"MM-DD"` 형식
- kpis가 null인 레코드는 `if (!h.kpis) continue`로 건너뜀
  (Prisma JSON null filter `kpis: { not: null }` 구문이 타입 에러를 발생시키므로 앱 레이어에서 처리)
- 각 KPI 합산 후 count로 나눠 `Math.round`

**서비스 파일**: `apps/api/src/feedback/feedback.service.ts:70`

---

## 5. 타입 안전성 규칙 적용 사례

```typescript
// ❌ 나쁜 예 — TS7053
const studentId = req['user'].id as string;

// ✅ 좋은 예 — AuthenticatedRequest 사용
async getProfile(@Req() req: AuthenticatedRequest) {
  return this.usersService.getProfile(req.user.id);
}
```

```typescript
// ❌ Prisma JSON null filter 에러
where: { kpis: { not: null } }   // TS2322

// ✅ 앱 레이어에서 처리
for (const h of histories) {
  if (!h.completed_at || !h.kpis) continue;
  // ...
}
```

---

## 6. 검증 결과

- `pnpm typecheck` — 3/3 통과
- `curl http://localhost:3004/users/me` (with JWT) → 200 OK
- `curl http://localhost:3004/courses/in-progress` (with JWT) → 200 OK

---

## 7. 관련 오류 기록

- `apps/api/src/common/types/request.types.ts` — req.user 타입 에러 해결
- `picklass_docs/speaking/errors/prisma-empty-schema-blocks-typecheck.md` — Prisma 빈 스키마 이슈
