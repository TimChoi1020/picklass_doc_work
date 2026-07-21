# 계정별 데이터 격리 정책 재수립

**작성일**: 2026-04-11  
**버전**: 1.1 — ✅ **구현 완료**

---

## 1. 정책 정의

**부분 계정별(Account-based) 격리**:
- **courses, classes, access_codes, texts**: 계정(teacher) 단위 격리
- **students (users)**: 기관(institution) 단위 격리 유지 — 학생은 기관 공유 리소스

### 격리 기준

```
institution (조직 단위)
  ├── students (기관 공유)
  └── teacher (account = 계정 단위 소유자)
        ├── courses (강사가 생성)
        ├── classes (강사가 담당)
        ├── access_codes (강사가 발급)
        └── texts (강사가 생성) — 이미 계정별 격리
```

---

## 2. 현재 상태 vs 변경안

### 현재 (institution 기반 격리)

| 모듈 | 필터 | 문제 |
|------|------|------|
| courses | `institution_id = user.institutionId` | 같은 기관 모든 강사가 공유 |
| classes | `institution_id = user.institutionId` | 동일 |
| access_codes | `institution_id = user.institutionId` | 동일 |
| students (users) | `institution_id = user.institutionId` | 동일 |
| texts | `user_id = user.id` | OK (이미 계정별) |

### 변경안 (account 기반 격리)

| 모듈 | 변경 후 필터 | 스키마 변경 | 비고 |
|------|-------------|------------|------|
| courses | `created_by = user.id` | ❌ (기존 컬럼 사용) | `courses.created_by` 기존 존재 |
| classes | `instructor_id = user.id` | ❌ (기존 컬럼 사용) | `classes.instructor_id` 기존 존재 |
| access_codes | `created_by = user.id` | ✅ **컬럼 추가 필요** | 다중 발급 주체(강사/관리자/기관) — 생성자 컬럼 신규 |
| students (users) | `institution_id` (유지) | ❌ | **변경 없음 — 기관 격리 유지** |
| texts | `user_id = user.id` | ❌ | 변경 없음 |

---

## 3. 스키마 변경 (최소 침습)

### 3-A. `access_codes` 테이블

**맥락**: access_codes는 studio(강사), backoffice(슈퍼관리자 / 기관관리자)에서 발급. 다양한 발급 주체를 구분해야 함.

컬럼 추가:
```sql
ALTER TABLE access_codes ADD COLUMN created_by UUID;
-- FK는 선택. users 테이블 참조 가능
CREATE INDEX idx_access_codes_created_by ON access_codes(created_by);
```

#### 발급 주체별 저장 정책

| 발급 주체 | created_by | institution_id |
|----------|------------|----------------|
| studio 강사 | 강사 user.id | 강사의 institution_id |
| backoffice 슈퍼관리자 | 슈퍼관리자 user.id | 대상 기관의 id |
| backoffice 기관관리자 | 기관관리자 user.id | 본인 기관의 id |

#### 조회 필터 정책

| 사용자 | 필터 |
|--------|------|
| **studio 강사** | `created_by = user.id` (본인 발급분만) |
| **backoffice 슈퍼관리자** | 필터 없음 (전체 조회) |
| **backoffice 기관관리자** | `institution_id = user.institutionId` (본인 기관 전체 — 강사/관리자 발급 모두 포함) |

- **기존 레거시 레코드**: `created_by = NULL` — studio 강사에게 노출 안 됨, backoffice에서는 institution_id로 조회 가능

### 3-B. `users` 테이블 (학생) — 변경 없음

**students는 기관 단위 격리를 유지합니다.** 학생 데이터는 같은 기관 강사들이 공유하는 리소스로 간주.

- 스키마 변경 없음
- `students.service.ts` 필터 변경 없음 (`institution_id = user.institutionId` 그대로)

### 3-C. Prisma schema 반영

`apps/api/prisma/schema.prisma`의 `access_codes`, `users` 모델에 `created_by` 필드 추가.

---

## 4. 서비스 코드 변경

### 4-A. `courses.service.ts`

```typescript
// findAll
where.created_by = user.id;  // institution_id 필터 제거

// create
data: {
  ...,
  institution_id: user.institutionId,  // 기관 정보는 여전히 기록
  created_by: user.id,                 // 기존 유지
}

// verifyOwnership
if (course.created_by !== user.id) {  // role 검증 제거
  throw new NotFoundException(...);
}
```

> `institution_id`는 계속 저장하되(조직 정보 유지), 필터는 `created_by` 사용.

### 4-B. `classes.service.ts`

```typescript
// findAll
where.instructor_id = user.id;  // institution_id 필터 제거

// create
data: {
  ...,
  institution_id: user.institutionId,  // 저장만
  instructor_id: user.id,              // 기존 유지
}

// verifyOwnership
if (cls.instructor_id !== user.id) { ... }
```

### 4-C. `access-codes.service.ts`

```typescript
// findAll
where.created_by = user.id;  // institution_id 필터 제거

// create
data: {
  ...,
  institution_id: user.institutionId,  // 저장만
  created_by: user.id,                 // 신규
}

// updateStatus / remove
where: { id, created_by: user.id }
```

### 4-D. `students.service.ts` — 변경 없음

기관 단위 격리 유지. 현재 구현 그대로.

### 4-E. `texts` (변경 없음)

이미 `user_id` 기반 계정 격리.

---

## 5. 프론트엔드 영향

백엔드 필터만 변경되므로 프론트 로직은 거의 영향 없음. 단, 다음 주의:

- **UI에서 "기관 소속 모든 강사 데이터" 가정이 있다면** 해당 부분 재검토 필요
  - students/accesscode 페이지에서 "기관 전체 학생/코드"를 보여주는 가정이 있으면 UX 변경 가능
- 현재 UI는 특별히 institution 전체 목록을 암시하지 않으므로 영향 제한적

---

## 6. 수정 대상 파일 요약

### DB

| 대상 | 작업 |
|------|------|
| `access_codes` 테이블 | `created_by UUID` 컬럼 추가 + 인덱스 |

### Prisma

| 파일 | 변경 |
|------|------|
| `apps/api/prisma/schema.prisma` | access_codes 모델에 `created_by` 추가 |

### 백엔드 서비스

| 파일 | 변경 |
|------|------|
| `apps/api/src/courses/courses.service.ts` | filter institution_id → created_by |
| `apps/api/src/classes/classes.service.ts` | filter institution_id → instructor_id |
| `apps/api/src/access-codes/access-codes.service.ts` | filter + created_by 기록 |
| `apps/api/src/students/students.service.ts` | **변경 없음** |

### 프론트엔드

- 변경 없음 (필요 시 UI 재검토만)

---

## 7. 기존 데이터 처리

### 옵션 A: 레거시 데이터 방치 (권장)

- 변경 후 생성되는 데이터만 `created_by` 채워짐
- 기존 데이터는 `created_by = NULL`로 남아 조회 안 됨
- 영향: 기존에 생성된 과정/클래스/코드/학생은 조회 불가 (필요 시 관리자 SQL로 소유자 지정)

### 옵션 B: 레거시 데이터 일괄 마이그레이션

- `courses`, `classes`는 이미 `created_by`/`instructor_id`가 있으므로 그대로 사용
- `access_codes`, `users`(students)는 생성자 추적 불가 → 수동 매핑 필요
- 또는 특정 관리자 계정으로 일괄 귀속

### 기본 방침 제안

- **옵션 A (방치)** — 깔끔하고 안전
- 필요 시 특정 과정/학생만 선택적으로 관리자가 소유자 지정

---

## 8. 영향 범위 / 사이드 이펙트

- **Prisma 스키마 변경**: schema.prisma 수정 + `prisma generate` 필요 (마이그레이션은 수동 SQL)
- **기존 레코드 조회 불가**: 옵션 A 선택 시 레거시 데이터는 사라진 것처럼 보임
- **institution 폐기 아님**: institution_id는 여전히 생성 시 기록됨 (청구/조직 정보용)
- **권한 체크**: verifyOwnership이 role_code 예외(`system_admin`) 무시 — 정책상 문제 없으면 유지, 관리자 전체 조회가 필요하면 별도 엔드포인트

---

## 9. 작업 순서 / 구현 결과

1. ✅ **DB 스키마 변경** — `access_codes.created_by UUID` 컬럼 + `idx_access_codes_created_by` 인덱스 추가
2. ✅ **Prisma schema 동기화** — `schema.prisma` 업데이트 + `prisma generate` 완료
3. ✅ **서비스 로직 변경**:
   - `courses.service.ts`: `where.created_by = user.id`로 변경
   - `classes.service.ts`: `where.instructor_id = user.id`로 변경, `query.instructor_id` override 제거 (보안)
   - `access-codes.service.ts`: findAll/updateStatus/remove 필터 `created_by = user.id`, create 시 `created_by` 기록
   - `students.service.ts`: **변경 없음** (기관 격리 유지)
4. ✅ **빌드 검증** — API, Web 모두 성공
5. ⏳ **로컬 테스트**: 사용자 확인 필요
6. ⏳ **배포**

---

## 10. 결정 필요 사항

1. **레거시 데이터 처리**: 옵션 A (방치) vs 옵션 B (관리자 계정 일괄 귀속)
2. **sniper4457 계정 기존 2개 course**: 그대로 유지 (created_by가 sniper4457이면 자동 조회됨)
3. **관리자/system_admin 역할**: 전체 조회가 필요한 role이 있는지 (`system_admin`이 모든 데이터 조회 가능하도록 예외 처리 유지 여부)
4. **access_codes 다중 발급 주체 (확정됨)**:
   - studio 강사: `created_by = user.id` 필터 (본인 발급분만)
   - backoffice 슈퍼관리자: 필터 없음 (전체 조회)
   - backoffice 기관관리자: `institution_id = user.institutionId` 필터 (본인 기관 전체)
   - **backoffice 측 변경사항**:
     - access_codes 생성 로직에서 `created_by = 발급자 user.id` 기록 추가
     - 조회 로직을 슈퍼관리자 vs 기관관리자 role에 따라 분기
     - 이 작업은 별도 문서(backoffice 측)에서 관리
