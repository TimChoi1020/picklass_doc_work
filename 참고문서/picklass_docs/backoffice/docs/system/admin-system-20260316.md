# admin-system (시스템 관리)

**페이지 경로:** `/admin/system`
**파일:** `apps/admin/frontend/src/app/(admin)/admin/system/page.tsx`
**작성일:** 2026-03-16

---

## 1. 사용자 흐름 (User Flow)

```
[시스템 관리 페이지 진입]
    │
    ├── [섹션 1: 사용자상태]
    │   ├── 행 인라인 편집 (상태명 / 상태코드 / 설명)
    │   ├── [✓] 저장 버튼 → alert (미구현)
    │   ├── [-] 삭제 버튼 → FormModal 확인 → 행 제거
    │   ├── [↑][↓] 순서 변경
    │   └── [+] 새 행 추가
    │
    ├── [섹션 2: 레벨시스템]
    │   ├── 행 인라인 편집 (레벨번호 / CEFR레벨 / 카테고리 select)
    │   ├── [✓] 저장 버튼 → alert (미구현)
    │   ├── [-] 삭제 버튼 → FormModal 확인 → 행 제거
    │   ├── [↑][↓] 순서 변경
    │   └── [+] 새 행 추가 (레벨번호 자동 max+1)
    │
    ├── [섹션 3: 액세스코드 상태]
    │   ├── (사용자상태와 동일한 구조)
    │   └── ...
    │
    ├── [섹션 4: 액세스코드 사용기간]
    │   ├── 행 인라인 편집 (기간명 / 기간코드 / 일수 / 설명)
    │   ├── [✓] 저장 버튼 → alert (미구현)
    │   ├── [-] 삭제 버튼 → FormModal 확인 → 행 제거
    │   ├── [↑][↓] 순서 변경
    │   └── [+] 새 행 추가 (days 기본값 30)
    │
    └── [섹션 5: 시스템 로그]
        └── 하드코딩 3건 조회 (읽기 전용)
```

### 삭제 흐름 상세
```
[-] 버튼 클릭
    → deleteConfirm state 설정 { type, uid, label }
    → FormModal 표시 ("○○을(를) 삭제하시겠습니까?")
    → [삭제] 클릭 → handleDeleteConfirm() → uid로 필터링 → state 갱신
    → [취소] 클릭 → deleteConfirm = null → 모달 닫힘
```

---

## 2. IA 구조 정리 및 기능 정의

### 페이지 레이아웃
```
시스템 관리 (h2)
└── 코드관리 (h3)
    ├── [card] 사용자상태
    ├── [card] 레벨시스템
    ├── [card] 액세스코드 상태
    ├── [card] 액세스코드 사용기간
    └── [card] 등록된 시스템 로그
```

### 섹션별 기능 정의

#### 공통 인라인 편집 테이블 패턴
| 기능 | 목적 | 입력 | 출력 | 조건 |
|------|------|------|------|------|
| 행 편집 | 코드 데이터 수정 | 인라인 input/textarea/select | state 갱신 | 실시간 반영, 저장은 별도 |
| ✓ 저장 | 변경사항 서버 저장 | 현재 행 데이터 | alert (미구현) | API 연동 필요 |
| - 삭제 | 행 제거 | uid | FormModal → state 필터링 | 1개 남으면 버튼 숨김 |
| ↑↓ 순서 | 표시 순서 변경 | index | state 배열 재정렬 | 첫/마지막 행 disabled |
| + 추가 | 새 빈 행 삽입 | - | state 끝에 빈 행 추가 | `_uid: crypto.randomUUID()` |

#### 사용자 상태 (StatusRow)
| 컬럼 | 타입 | 목적 |
|------|------|------|
| 상태명 | text input | UI 표시용 한글명 |
| 상태 코드 | text input | DB 저장값 (active/inactive 등) |
| 설명 | textarea | 상태 의미 설명 |

#### 레벨시스템 (LevelRow)
| 컬럼 | 타입 | 목적 |
|------|------|------|
| 레벨 | number input (min=1) | 정렬 번호 (1~18) |
| CEFR 레벨 | text input | CEFR 표기 (A1-, A1, A1+ 등) |
| 카테고리 | select (LEVEL_CATEGORIES 파생) | 6개 카테고리 중 선택 |

> 카테고리 옵션은 `LEVEL_SYSTEM`에서 `[...new Set(...map)]`으로 파생 — constants 변경 시 자동 반영

#### 액세스코드 사용기간 (DurationRow)
| 컬럼 | 타입 | 목적 |
|------|------|------|
| 기간명 | text input | 표시명 (1개월, 3개월 등) |
| 기간 코드 | text input | 저장 코드 (1month, 3month 등) |
| 일수 | number input (min=1, text-right) | 실제 계산에 사용되는 일수 |
| 설명 | textarea | 기간 설명 |

#### 시스템 로그 (LogEntry)
| 컬럼 | 목적 |
|------|------|
| 일시 | 작업 발생 시각 |
| 작업내용 | 수행된 액션 |
| 관리자 | 작업 수행자 |
| 상태 | 완료/실패 여부 |

---

## 3. 정책 (Policy / Business Rules)

### 코드값 정책
- UI 표시명(상태명)과 DB 저장 코드(상태코드)는 분리 관리
- 상태 코드는 영문 소문자 (`active`, `inactive`, `suspended`, `withdrawn`)
- 기간 코드는 영문 소문자 (`1month`, `3month`, `6month`, `1year`)
- 코드값 변경 시 해당 코드를 참조하는 모든 DB 레코드에 영향 → **신중하게 변경**

### 사용자 상태 기준값
| 상태명 | 코드 | 의미 |
|--------|------|------|
| 활성 | active | 정상 사용 중 |
| 비활성 | inactive | 비활성 상태 |
| 정지 | suspended | 관리자 조치 정지 |
| 탈퇴 | withdrawn | 계정 탈퇴 |

### 레벨시스템 정책
| 카테고리 | 레벨 | CEFR |
|----------|------|------|
| Starter | 1~3 | A1-, A1, A1+ |
| Beginner | 4~6 | A2-, A2, A2+ |
| Intermediate | 7~9 | B1-, B1, B1+ |
| Upper-Intermediate | 10~12 | B2-, B2, B2+ |
| Advanced | 13~15 | C1-, C1, C1+ |
| Proficient | 16~18 | C2-, C2, C2+ |

- 총 18단계 고정 (현재 기준)
- 레벨 번호는 1부터 시작, 중복 불가 (현재 중복 체크 미구현)

### 액세스코드 사용기간 기준값
| 기간명 | 코드 | 일수 |
|--------|------|------|
| 1개월 | 1month | 30 |
| 3개월 | 3month | 90 |
| 6개월 | 6month | 180 |
| 1년 | 1year | 365 |

- 만료일 = 발급일 + days (백엔드에서 계산)

### 삭제 정책
- 각 섹션에서 **마지막 1개는 삭제 불가** (버튼 자체를 숨김)
- 삭제 전 FormModal로 확인 필수

### 순서 변경 정책
- ↑↓ 버튼으로 UI에서 즉시 반영
- 현재 순서 변경은 서버에 저장되지 않음 (미구현)

### 정책 변경사항 (이전 system.md 대비)
- 기존 문서의 레벨 카테고리 `Basic/Mastery` → 실제 코드 기준 `Starter~Proficient` 6단계로 수정
- 기존 문서의 ACCESS_CODE_STATUSES 코드 `reserved/used/expired` → 실제 `inactive/suspended/withdrawn`으로 수정
- 기존 문서의 ACCESS_CODE_DURATIONS 코드 `1M/3M/12M` → 실제 `1month/3month/1year`으로 수정

---

## 4. 개발자 추가 작업 사항

### 최우선: 저장 버튼 API 연동
현재 4개 섹션의 ✓ 저장 버튼이 모두 `alert()`만 실행. 실제 저장 없음.

```typescript
// 현재
<button onClick={() => alert('사용자 상태가 저장되었습니다.')}>✓</button>

// 목표 (각 섹션별)
<button onClick={() => handleSave('user-statuses', status)}>✓</button>

// 또는 전체 일괄 저장 방식
<button onClick={() => handleSaveAll()}>전체 저장</button>
```

필요 API:
```
PUT /api/admin/system/user-statuses        사용자 상태 전체 저장
PUT /api/admin/system/level-system         레벨시스템 전체 저장
PUT /api/admin/system/access-code-statuses 액세스코드 상태 전체 저장
PUT /api/admin/system/access-code-durations 사용기간 전체 저장
```

또는 기존 `GET /codes`, `GET /codes/:groupCode` 백엔드에 맞춰:
```
PUT /api/codes/:groupCode  — code_items 일괄 업데이트
POST /api/codes/:groupCode/items — 항목 추가
DELETE /api/codes/:groupCode/items/:code — 항목 삭제
```

### 순서 저장
현재 ↑↓ 순서 변경이 state에만 반영되고 서버에 저장되지 않음.
`sortOrder` 필드를 DB에 추가하고 저장 시 반영 필요.

### 레벨 번호 중복 체크
새 레벨 추가 시 `Math.max(...) + 1`로 자동 부여하지만,
레벨 번호를 직접 편집할 경우 중복 허용됨 → 저장 전 중복 검사 필요.

### 시스템 로그 API 연동
```typescript
// 현재 하드코딩
const logs: LogEntry[] = [
  { timestamp: '2026-02-13 14:30', action: '시스템 로그아웃', ... },
  ...
];

// 목표
const { data: logs } = useQuery(['system-logs'], fetchSystemLogs);
```
필요 API: `GET /api/admin/system/logs?page=1&limit=20`

### 페이지 새로고침 시 편집 내용 초기화 문제
현재 state 초기화가 constants 하드코딩 데이터 기반. 새로고침 시 DB 저장값이 아닌 constants 값으로 초기화됨.
→ 페이지 로드 시 API에서 최신 데이터 fetch 필요.

---

## 5. 코드 규칙 (Coding Rules)

### 사용해야 하는 공통 요소
| 구분 | 경로 | 용도 |
|------|------|------|
| 유틸 | `src/lib/utils.ts` → `moveUp`, `moveDown` | 배열 순서 변경. 페이지 내 재정의 금지 |
| 공통 컴포넌트 | `src/components/common/form-modal.tsx` | 삭제 확인 다이얼로그. `confirm()` 금지 |
| 상수 | `src/lib/constants.ts` → `USER_STATUSES`, `ACCESS_CODE_STATUSES`, `ACCESS_CODE_DURATIONS`, `LEVEL_SYSTEM` | 초기 state 값 |
| UI | Tailwind CSS 클래스 | 모든 스타일링. 인라인 스타일 금지 |

### 공통 Tailwind 클래스 상수 (페이지 최상단에 정의)
```typescript
const cellInput   = 'w-full p-1 border border-gray-300 rounded-sm box-border text-sm';
const cellTextarea = 'w-full p-1 border border-gray-300 rounded-sm font-[inherit] min-h-[40px] box-border text-sm';
const moveBtn     = 'size-[30px] rounded-full flex items-center justify-center bg-gray-400 text-white text-sm border-none cursor-pointer disabled:cursor-not-allowed disabled:opacity-50';
```

### WithId 패턴 (이 페이지의 핵심 패턴)
```typescript
// 타입 확장 — 파일 최상단에 선언
type WithId<T> = T & { _uid: string };

// 초기화 — crypto.randomUUID() 사용
const toRows = <T,>(items: readonly T[]): WithId<T>[] =>
  items.map((item) => ({ ...item, _uid: crypto.randomUUID() }));

// state 선언
const [items, setItems] = useState<WithId<SomeRow>[]>(() => toRows(SOME_CONSTANT));

// key — 반드시 _uid 사용
{items.map((item, index) => (
  <tr key={item._uid}>...</tr>
))}

// 새 행 추가 — _uid 포함 필수
setItems((p) => [...p, { name: '', code: '', ..., _uid: crypto.randomUUID() }]);

// 삭제 — uid로 필터링
setItems((p) => p.filter((r) => r._uid !== uid));
```

### 금지 패턴
- `any` 타입, `@ts-ignore` 금지
- `confirm()` 직접 사용 금지 → `FormModal` 사용
- `alert()` 저장 완료 메시지 금지 → toast 또는 API 응답 처리
- 인라인 스타일 `style={{}}` 금지 → Tailwind 클래스
- `moveUp` / `moveDown` 페이지 내 재정의 금지 → `@/lib/utils` import
- `key={index}` 금지 → `key={item._uid}`
- `<>` 단축 fragment에 key 금지 → `<Fragment key={...}>` 사용

### 파일 위치 규칙
```
타입 정의 (StatusRow, DurationRow, LevelRow) → src/lib/constants.ts
공통 유틸 (moveUp, moveDown)               → src/lib/utils.ts
공통 모달 컴포넌트                          → src/components/common/form-modal.tsx
페이지                                     → src/app/(admin)/admin/system/page.tsx
```

### 네이밍 규칙
- 상태 setter: `setUserStatuses`, `setCodeStatuses`, `setDurations`, `setLevels`
- 삭제 확인 state: `deleteConfirm` (`{ type, uid, label }` 구조)
- 삭제 핸들러: `handleDeleteConfirm`
- 섹션 타입 구분자: `'user' | 'code' | 'duration' | 'level'`

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 구분 | 문제 | 위치 | 심각도 |
|------|------|------|--------|
| 미구현 | ✓ 저장 버튼 4개 전부 `alert()`만 — 실제 DB 저장 없음 | page.tsx:61,105,142,181 | **높음** |
| 미구현 | 페이지 새로고침 시 constants 초기값으로 리셋 (DB 데이터 미로드) | useState 초기값 | **높음** |
| 미구현 | ↑↓ 순서 변경 서버 미반영 | moveUp/moveDown 호출부 | 중간 |
| 미구현 | 시스템 로그 하드코딩 3건 — API 연동 없음 | page.tsx:43~47 | 중간 |
| 미구현 | 레벨 번호 중복 체크 없음 | 레벨 number input | 중간 |
| 미구현 | 상태 코드 중복 체크 없음 | 상태코드 text input | 중간 |
| 미구현 | 시스템 로그 페이지네이션 없음 | 로그 섹션 | 낮음 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

### 이 페이지가 사용하는 것
```
page.tsx
├── react: useState
├── @/lib/constants: USER_STATUSES, ACCESS_CODE_STATUSES, ACCESS_CODE_DURATIONS, LEVEL_SYSTEM
│                   StatusRow, DurationRow, LevelRow (타입)
├── @/components/common/form-modal: FormModal
└── @/lib/utils: moveUp, moveDown
```

### 진입점
- 사이드바 메뉴 → `ADMIN_MENU` (constants.ts) → `/admin/system`

### 이 페이지가 영향을 주는 곳
이 페이지에서 설정하는 코드값들은 시스템 전역에서 참조됨:

| 데이터 | 참조하는 곳 |
|--------|------------|
| USER_STATUSES | 사용자 관리(`/admin/users`), 사용자 편집(`/admin/users/[id]/edit`) |
| ACCESS_CODE_STATUSES | 액세스코드 관리(`/admin/accesscode`) |
| ACCESS_CODE_DURATIONS | 액세스코드 생성(`/admin/accesscode/generate`) — 기간 드롭다운 |
| LEVEL_SYSTEM | AI 모듈 등록(`/admin/ai-modules/register`) — 레벨 선택, module-*.tsx 컴포넌트 |

> **주의:** 상태 코드 변경 시 해당 코드를 `status_code`, `member_status_code` 등으로 저장한 DB 레코드 전체에 영향. 코드값은 신중하게 변경할 것.

---

## 8. DB/API 구조 (Data Contract)

### 현재 백엔드 상태
```
GET  /codes              — 전체 코드 그룹 목록 (CodeController 존재)
GET  /codes/:groupCode   — 특정 그룹의 코드 항목 목록
```
프론트에서 이 API를 **사용하지 않음** — constants.ts 하드코딩으로 처리 중.

### 목표 API 엔드포인트
```
GET    /api/admin/system/user-statuses          사용자 상태 목록 조회
PUT    /api/admin/system/user-statuses          사용자 상태 전체 저장 (순서 포함)

GET    /api/admin/system/level-system           레벨시스템 목록 조회
PUT    /api/admin/system/level-system           레벨시스템 전체 저장

GET    /api/admin/system/access-code-statuses   액세스코드 상태 목록 조회
PUT    /api/admin/system/access-code-statuses   액세스코드 상태 전체 저장

GET    /api/admin/system/access-code-durations  사용기간 목록 조회
PUT    /api/admin/system/access-code-durations  사용기간 전체 저장

GET    /api/admin/system/logs?page=1&limit=20   시스템 로그 목록 조회
```

### DB 테이블 (기존 Prisma 스키마)
```prisma
// 이미 존재하는 테이블 — 시스템 코드 관리에 활용 가능
model CodeGroup {
  id        BigInt     @id @default(autoincrement())
  code      String     @unique @db.VarChar(50)   // 'user-statuses', 'level-system' 등
  name      String     @db.VarChar(100)
  isActive  Boolean    @default(true) @map("is_active")
  items     CodeItem[]
  @@map("code_groups")
}

model CodeItem {
  id        BigInt   @id @default(autoincrement())
  groupId   BigInt   @map("group_id")
  code      String   @db.VarChar(50)             // 'active', 'inactive', '1month' 등
  name      String   @db.VarChar(100)             // '활성', '비활성', '1개월' 등
  sortOrder Int      @default(0) @map("sort_order") // 순서 저장
  extraData Json?    @map("extra_data") @db.JsonB  // days, cefrLevel 등 추가 데이터
  isActive  Boolean  @default(true) @map("is_active")
  group     CodeGroup @relation(fields: [groupId], references: [id])
  @@unique([groupId, code])
  @@map("code_items")
}
```

> `extraData` JSON 컬럼에 레벨의 `cefrLevel`, 기간의 `days` 등 추가 필드 저장 가능.

### 타입 정의 전문
```typescript
// src/lib/constants.ts — 현재 정의된 타입

export interface StatusRow {
  name: string;         // UI 표시명 ("활성")
  code: string;         // DB 저장 코드 ("active")
  description: string;  // 설명
}

export interface DurationRow {
  name: string;         // 기간명 ("1개월")
  code: string;         // 기간 코드 ("1month")
  days: number;         // 일수 (30)
  description: string;  // 설명
}

export interface LevelRow {
  level: number;        // 레벨 번호 (1~18)
  cefrLevel: string;    // CEFR 표기 ("A1-", "A1", "A1+" 등)
  category: string;     // 카테고리 ("Starter" ~ "Proficient")
}

// page.tsx 내 로컬 타입
interface LogEntry {
  timestamp: string;
  action: string;
  admin: string;
  status: string;
}

// page.tsx 내 WithId 패턴 (API 연동 후에도 유지)
type WithId<T> = T & { _uid: string };
```

---

*작성일: 2026-03-16*
*페이지: admin/system*
