# 학생 아이디 관리 페이지 — course-students-20260316

**경로:** `apps/web/src/app/course/students/page.tsx`
**라우트:** `/course/students`
**작성일:** 2026-03-16
**이전 문서:** 없음 (신규 페이지)

---

## 1. 사용자 흐름 (User Flow)

```
/course/students 진입
  └── 학생 아이디 관리 테이블

필터링:
  ├── 이름 입력 → 실시간 필터 (includes)
  ├── 아이디 입력 → 실시간 필터 (toLowerCase 포함)
  ├── 상태 선택 (전체/활성/정지/탈퇴)
  └── 초기화 버튼 → 모든 필터 초기화

학생 등록:
  "학생 등록" 버튼 → 등록 모달 오픈
  ├── 소속 기관 (읽기 전용 — STUDIO_INSTITUTION 자동 표시)
  ├── 학생 아이디 입력 + "랜덤생성" 버튼 + "중복확인" 버튼
  │     └── 랜덤생성: 현재 최대 일련번호+1 → {code}{serial}@{code}.pick
  │     └── 중복확인: students state에서 userId 중복 체크
  ├── 초기 임시 비밀번호 입력
  ├── 이름 입력
  └── 등록 버튼 (중복확인 완료 필수) → state에 추가 + toast.success

학생 정보 수정:
  행 클릭 → 오버레이 표시 → "수정" 클릭 → 수정 모달 오픈
  ├── 사용자 유형 (읽기 전용 — "학생")
  ├── 소속 기관 (읽기 전용 — STUDIO_INSTITUTION)
  ├── 학생 아이디 (읽기 전용 — 변경 불가)
  ├── 이름 (수정 가능)
  ├── 상태 선택 (활성/정지/탈퇴)
  └── 비밀번호 초기화 체크박스 + 조건부 새 비밀번호 입력
        → 저장 → name/status state 업데이트 + toast.success
```

---

## 2. IA 구조 (Information Architecture)

```
/course/students
├── StudioHeader
└── 콘텐츠 영역 (CourseSidebar + 메인)
    ├── CourseSidebar (검색 비활성, itemCount = 필터된 학생 수)
    │   ├── "모든 과정" → /course
    │   ├── "학생 아이디 관리" → /course/students (현재 페이지, 활성)
    │   └── "액세스코드 관리" (미구현)
    │
    └── 메인 영역
        ├── 상단: "학생 아이디 관리" 제목 + "학생 등록" 버튼
        ├── 필터바: 이름 | 아이디 | 상태 | 초기화
        ├── 테이블
        │   ├── 헤더: 번호 | 사용자명 | 아이디 | 상태 | 등록일
        │   └── 행: name, userId, status(뱃지), createdAt
        │       └── 오버레이: "수정" 버튼 (삭제 없음)
        ├── 페이지네이션 (10개/페이지)
        └── 상태 범례: 활성(초록) | 정지(빨강) | 탈퇴(회색)
```

**Student 데이터 필드:**

| 필드 | 타입 | 설명 |
|------|------|------|
| id | string | 내부 ID (U001 형식) |
| name | string | 학생 이름 |
| userId | string | 학생 아이디 ({code}{serial}@{code}.pick) |
| institution | string | 소속 기관명 (STUDIO_INSTITUTION.name 고정) |
| status | 'active' \| 'suspended' \| 'withdrawn' | 계정 상태 |
| createdAt | string | 등록일 ('YYYY-MM-DD' 형식) |

---

## 3. 정책 (Policy / Business Rules)

### 소속 기관 정책 (핵심 변경)
- 스튜디오는 **단일 학원 플랫폼** — 소속 기관은 `STUDIO_INSTITUTION` 하나만 존재
- 등록/수정 모달 모두 기관 선택 불가, 읽기 전용으로만 표시
- `STUDIO_INSTITUTION = { id: 'I001', name: '픽클래스 아카데미' }` (현재 mock)
- `STUDIO_CODE = 'pkls'` (현재 mock, 실제로는 기관 설정값에서 가져와야 함)

### 학생 아이디 생성 정책 (핵심 변경)
- 형식: `{4자리학원코드}{3자리일련번호}@{4자리학원코드}.pick`
- 예시: `pkls001@pkls.pick`, `pkls013@pkls.pick`
- **랜덤생성** 버튼: 현재 등록된 students에서 최대 일련번호를 찾아 +1로 생성
- **중복확인** 버튼: students state에서 userId 중복 체크 후 `duplicateCheckDone = true`
- 등록 완료 전까지 중복확인 필수 (미완료 시 toast.error)
- 아이디는 등록 후 변경 불가 (수정 모달에서 읽기 전용)
- 이메일 형식 검증 제거 — 커스텀 도메인 형식(.pick)이므로 별도 포맷 검증 불필요

### 계정 상태 정책
- `active` → "활성": 정상 이용 중 (초록 뱃지)
- `suspended` → "정지": 이용 정지 상태 (빨간 뱃지)
- `withdrawn` → "탈퇴": 탈퇴 처리됨 (회색 뱃지)
- 신규 등록 시 기본값: `'active'`
- 상태 변경은 수정 모달의 select를 통해서만 가능

### 삭제 정책 (핵심 변경)
- **학생 삭제 버튼 없음** — 행 오버레이에 "수정" 버튼만 존재
- 삭제 기능은 현재 미구현 (향후 요구사항 확정 후 추가 필요)

### 비밀번호 초기화 정책
- 수정 모달에서 "비밀번호 초기화" 체크박스 활성화 시에만 새 비밀번호 입력 가능
- 체크박스 비활성화 시 newPassword 필드 초기화
- 체크박스 활성화 + 비밀번호 미입력 시 toast.error

---

## 4. 개발자 추가 작업

### 🔴 필수
- [ ] **학생 목록 API 연동**: `GET /api/students` — mockStudents 대체
- [ ] **학생 등록 API**: `POST /api/students` — userId 중복 확인도 서버에서 검증
- [ ] **학생 수정 API**: `PATCH /api/students/:id` — name, status, password
- [ ] **STUDIO_INSTITUTION 실제 연동**: 현재 하드코딩된 기관 정보를 인증 컨텍스트(로그인한 스튜디오의 기관 정보)에서 가져오도록 변경
- [ ] **STUDIO_CODE 실제 연동**: 4자리 학원코드를 기관 설정에서 읽어오도록 변경

### 🟡 권장
- [ ] **아이디 중복확인 서버 검증**: 현재 클라이언트 state로만 중복 확인 → `GET /api/students/check-duplicate?userId=...`
- [ ] **일련번호 서버 계산**: 랜덤생성 시 서버에서 다음 일련번호 반환 (동시 등록 충돌 방지)
- [ ] **학생 삭제 정책 결정**: 완전 삭제 vs. 탈퇴 상태 변경 중 방향 결정 후 구현

---

## 5. 코딩 규칙 (Coding Rules)

### 공통 컴포넌트
- `CourseSidebar` from `@/components/CourseSidebar`
- `StudioHeader` from `@/components/oizi/StudioHeader`
- `SimplePagination` from `@/components/ui/simple-pagination`
- `Dialog`, `DialogContent`, `DialogTitle` from `@/components/ui/dialog`
- `toast` from `sonner`

### 학원코드/기관 상수 패턴
```typescript
// 현재 (mock):
const STUDIO_INSTITUTION = { id: 'I001', name: '픽클래스 아카데미' };
const STUDIO_CODE = 'pkls';

// 향후 (실제):
// const { institution, studioCode } = useStudioContext();
```

### 학생 아이디 생성 패턴
```typescript
const generateStudentId = () => {
  const pattern = new RegExp(`^${STUDIO_CODE}(\\d+)@${STUDIO_CODE}\\.pick$`);
  const maxSerial = students.reduce((max, s) => {
    const m = s.userId.match(pattern);
    return m ? Math.max(max, parseInt(m[1], 10)) : max;
  }, 0);
  const next = String(maxSerial + 1).padStart(3, '0');
  return `${STUDIO_CODE}${next}@${STUDIO_CODE}.pick`;
};
```

### 상태 표시 패턴
```typescript
const STATUS_LABELS: Record<string, string> = {
  active: '활성', suspended: '정지', withdrawn: '탈퇴',
};
const STATUS_CLASS: Record<string, string> = {
  active: 'bg-emerald-100 text-emerald-700',
  suspended: 'bg-red-100 text-red-700',
  withdrawn: 'bg-gray-100 text-gray-600',
};
```

### EMPTY_FORM 상수 패턴
```typescript
const EMPTY_FORM = { userId: '', tempPassword: '', name: '' };
const EMPTY_EDIT_FORM = { userId: '', name: '', status: 'active', newPassword: '' };
```

### 금지 패턴
- 기관 선택 `<select>` 렌더링 금지 (단일 기관 플랫폼)
- `type="email"` input 사용 금지 (`.pick` 도메인은 이메일 유효성 검사에 실패할 수 있음)
- `alert()`, `confirm()` 직접 사용 금지
- `key={index}` 사용 금지 → `key={student.id}`

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| # | 이슈 | 심각도 | 상태 |
|---|------|--------|------|
| 1 | mockStudents 12개 하드코딩 — API 미연동 | High | 미해결 |
| 2 | STUDIO_INSTITUTION, STUDIO_CODE 하드코딩 — 실제 기관 설정 미연동 | High | 미해결 |
| 3 | 중복확인이 클라이언트 state 기준 — 서버 검증 없음 | High | 미해결 |
| 4 | 랜덤생성 일련번호 계산이 클라이언트 기준 — 동시 등록 시 충돌 가능 | Medium | 미해결 |
| 5 | 학생 삭제 기능 미구현 (버튼 자체 없음) | Medium | 정책 결정 대기 |
| 6 | 등록 후 비밀번호 실제 저장 없음 (state에 포함 안 됨) | High | 미해결 |
| 7 | 수정 시 비밀번호 초기화 state에 반영 안 됨 (newPassword 미사용) | High | 미해결 |
| 8 | institution 필드 수정 시 state 업데이트 미반영 (수정 모달에서 readonly이므로 현재는 무관) | Low | 확인됨 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

| 항목 | 경로 | 용도 |
|------|------|------|
| `CourseSidebar` | `@/components/CourseSidebar` | 좌측 사이드바 |
| `StudioHeader` | `@/components/oizi/StudioHeader` | 상단 헤더 |
| `SimplePagination` | `@/components/ui/simple-pagination` | 페이지네이션 |
| `Dialog`, `DialogContent`, `DialogTitle` | `@/components/ui/dialog` | 등록/수정 모달 |
| `Button` | `@/components/ui/button` | 버튼 |
| `Input` | `@/components/ui/input` | 텍스트 입력 |
| `toast` | `sonner` | 알림 |

**진입점:**
- `CourseSidebar`의 "학생 아이디 관리" 버튼 (`router.push('/course/students')`)

**이 페이지가 영향을 주는 기능:**
- `CourseSidebar` itemCount → 필터된 학생 수 전달 (현재 itemCount = filteredStudents.length)

**AlertDialog 미사용 주의:**
- 이 페이지는 삭제 기능이 없으므로 AlertDialog import 불필요

---

## 8. DB/API 구조 (Data Contract)

### 현재 상태
- 데이터: `mockStudents` 상수 (12개)
- API 미연동

### Student 인터페이스

```typescript
interface Student {
  id: string;              // 내부 ID (UUID 예정)
  name: string;            // 학생 이름
  userId: string;          // 학생 아이디 ({code}{serial}@{code}.pick)
  institution: string;     // 소속 기관명 (STUDIO_INSTITUTION.name 고정)
  status: 'active' | 'suspended' | 'withdrawn';
  createdAt: string;       // 'YYYY-MM-DD'
}

// 등록 Form
interface RegisterForm {
  userId: string;          // 생성된 학생 아이디
  tempPassword: string;    // 초기 임시 비밀번호
  name: string;
}

// 수정 Form
interface EditForm {
  userId: string;          // 읽기 전용
  name: string;
  status: 'active' | 'suspended' | 'withdrawn';
  newPassword: string;     // passwordReset === true 일 때만 사용
}
```

### 향후 API 설계 (예정)

```
GET    /api/students                     → Student[] (이름/아이디/상태 필터, 페이지)
POST   /api/students                     → 학생 등록 { userId, name, tempPassword }
PATCH  /api/students/:id                 → 학생 수정 { name?, status?, newPassword? }
GET    /api/students/check-duplicate     → 아이디 중복 확인 { userId: string }
GET    /api/institutions/current/code    → STUDIO_CODE 반환
```

### DB 테이블 설계 (예정)

```sql
CREATE TABLE students (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id UUID NOT NULL REFERENCES institutions(id),
  user_id        VARCHAR(100) NOT NULL UNIQUE,  -- e.g. pkls001@pkls.pick
  name           VARCHAR(100) NOT NULL,
  password_hash  VARCHAR(255) NOT NULL,
  status         VARCHAR(20) NOT NULL DEFAULT 'active',
  created_at     TIMESTAMP DEFAULT NOW(),
  updated_at     TIMESTAMP DEFAULT NOW(),

  CONSTRAINT chk_status CHECK (status IN ('active', 'suspended', 'withdrawn'))
);

CREATE TABLE institutions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(100) NOT NULL,
  code        CHAR(4) NOT NULL UNIQUE,  -- 4자리 학원코드 (pkls)
  created_at  TIMESTAMP DEFAULT NOW()
);
```
