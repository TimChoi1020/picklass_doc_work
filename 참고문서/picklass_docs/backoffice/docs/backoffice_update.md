# Picklass Backoffice 업데이트 가이드

## 개요
Admin 백오피스 시스템의 데이터 관리 및 UI/UX 개선을 위한 종합 업데이트입니다.

---

## 1. 상수화 (Constants Centralization)

### 목표
모든 공유 데이터를 `src/lib/constants.ts`에 중앙집중식으로 관리하여 코드 중복 제거 및 유지보수성 향상

### 적용된 상수

#### 1.1 사용자 상태 (USER_STATUSES)
```typescript
export interface StatusRow {
  name: string;      // 화면 표시명
  code: string;      // DB 저장 코드
  description: string;
}

export const USER_STATUSES: StatusRow[] = [
  { name: '활성', code: 'active', description: '활성 사용자 상태' },
  { name: '비활성', code: 'inactive', description: '비활성 사용자 상태' },
  { name: '정지', code: 'suspended', description: '일시적 정지 상태' },
  { name: '탈퇴', code: 'withdrawn', description: '계정 탈퇴 상태' },
];
```
**적용 페이지:**
- `admin/users/[id]/edit/page.tsx`: 사용자 상태 select 동적 생성
- `admin/system/page.tsx`: 액세스코드 상태 관리 섹션

#### 1.2 기관 상태 (INSTITUTION_STATUSES)
```typescript
export const INSTITUTION_STATUSES: StatusRow[] = [
  { name: '활성', code: 'active', description: '활성 기관 상태' },
  { name: '휴회', code: 'inactive', description: '휴회 상태' },  // 사용자와 다름
  { name: '정지', code: 'suspended', description: '일시적 정지 상태' },
  { name: '탈퇴', code: 'withdrawn', description: '기관 탈퇴 상태' },
];
```
**적용 페이지:**
- `admin/institute/[id]/edit/page.tsx`: 기관 상태 select
- `admin/institute/register/page.tsx`: 기관 등록 상태 select

#### 1.3 액세스코드 사용기간 (ACCESS_CODE_DURATIONS)
```typescript
export interface DurationRow {
  name: string;      // "1개월"
  code: string;      // "1month"
  days: number;      // 30
  description: string;
}

export const ACCESS_CODE_DURATIONS: DurationRow[] = [
  { name: '1개월', code: '1month', days: 30, description: '1개월 사용 기간' },
  { name: '3개월', code: '3month', days: 90, description: '3개월 사용 기간' },
  { name: '6개월', code: '6month', days: 180, description: '6개월 사용 기간' },
  { name: '1년', code: '1year', days: 365, description: '1년 사용 기간' },
];
```
**적용 페이지:**
- `admin/system/page.tsx`: 액세스코드 생성 시 사용기간 select

#### 1.4 레벨 시스템 (LEVEL_SYSTEM)
```typescript
export interface LevelRow {
  level: number;     // 1-18
  cefrLevel: string; // "A1-", "C2+"
  category: string;  // "Starter", "Proficient"
}

export const LEVEL_SYSTEM: LevelRow[] = [
  { level: 1, cefrLevel: 'A1-', category: 'Starter' },
  // ... 18개 레벨
  { level: 18, cefrLevel: 'C2+', category: 'Proficient' },
];
```
**적용 페이지:**
- `admin/system/page.tsx`: 레벨 시스템 관리 테이블

#### 1.5 요금제 (DEFAULT_PLANS)
```typescript
export interface Plan {
  name: string;           // "Starter", "Pro", "Enterprise"
  monthlyPrice: number;   // 100000 (원)
  annualDiscount: string; // "5%"
  baseStudents: number;   // 기본 제공 학생 수 (10명)
  pricePerStudent: number; // 추가 학생당 단가 (5000원)
  maxStudents: number;    // 50, 500, 0(무제한)
  maxAdminAccounts: number; // 2, 5, 10
  apiIntegration: string; // "미연동", "연동"
  apiCost: number;        // API 비용
  enabled: string;        // "사용"
  unitPrice: string;      // "1.2만원/명" (표시용)
}

export const DEFAULT_PLANS: Plan[] = [
  {
    name: 'Starter',
    monthlyPrice: 100000,
    annualDiscount: '5%',
    baseStudents: 10,
    pricePerStudent: 5000,
    maxStudents: 50,
    maxAdminAccounts: 2,
    apiIntegration: '미연동',
    apiCost: 0,
    enabled: '사용',
    unitPrice: '1.2만원/명',
  },
  // ... Pro, Enterprise
];
```
**적용 페이지:**
- `admin/plans/page.tsx`: 요금제 관리
- `admin/institute/[id]/edit/page.tsx`: 기관 요금제 선택
- `admin/institute/register/page.tsx`: 기관 등록 시 요금제 선택

---

## 2. 페이지별 변경사항

### 2.1 플랜 관리 페이지 (NEW)
**경로:** `admin/plans/page.tsx`

**개요:**
- 시스템 관리 페이지에서 분리된 독립적인 플랜 관리 페이지
- DEFAULT_PLANS를 활용한 CRUD 관리

**기능:**
- 3개 플랜(Starter, Pro, Enterprise) 조회/수정
- 각 플랜별 상세 정보 표시 (2행 레이아웃)
- 플랜별 저장/취소 버튼

**업데이트 필요:**
- 사이드바 메뉴 추가: "플랜관리" → `/admin/plans` ✅ 완료

---

### 2.2 기관 관리 - 등록/수정 (institute)

#### 섹션 3: 라이선스 & 요금제 개편

**변경 전:**
- 플랜 선택
- 학생당 단가 (read-only)
- 최대 학생수, 관리계정수
- 청구주기, 연납할인율 (별도)
- API 비용 필드 분산

**변경 후 (7행 구조):**

| 행 | 필드명 | 타입 | 입력방식 | 설명 |
|---|---|---|---|---|
| 1 | 플랜 | Select | 선택 | DEFAULT_PLANS에서 선택 |
| 2 | 월비용 / 연납할인 | Input / Input | 원 단위 / 텍스트 | 플랜 선택 시 자동 채움 (편집 가능) |
| 3 | 학생기본제공 / 추가학생당단가 | Input / Input | 수량 / 원 단위 | 플랜 선택 시 자동 채움 (편집 가능) |
| 4 | 최대학생수 / 최대관리계정수 | Input / Input | 수량 / 수량 | 사용자 입력 |
| 5 | API연동 / API연동비용 | Select / Input | 선택 / 원 단위 | 선택적 입력 |
| 6 | 기술지원비 / 콘텐츠IP전환비 | Input / Input | 원 단위 / 원 단위 | 사용자 입력 |
| 7 | 기타 | Textarea | 텍스트 | 유의사항 기입 |

**플랜 선택 자동 채움 로직:**
```typescript
if (name === 'plan') {
  const planOption = DEFAULT_PLANS.find((p) => p.name === value);
  setForm((p) => ({
    ...p,
    [name]: value,
    monthlyPrice: planOption?.monthlyPrice ?? 0,      // 월비용
    pricePerStudent: planOption?.pricePerStudent ?? 0, // 추가학생당단가
    unitPrice: planOption?.unitPrice ?? '',            // 단가 표시용
    baseStudents: String(planOption?.baseStudents ?? ''),    // 학생기본제공
    annualDiscount: String(planOption?.annualDiscount ?? ''), // 연납할인
    maxAdminAccounts: String(planOption?.maxAdminAccounts ?? ''), // 관리계정수
    apiIntegration: planOption?.apiIntegration ?? '',  // API연동 상태
  }));
}
```

**적용 파일:**
- ✅ `admin/institute/[id]/edit/page.tsx`
- ✅ `admin/institute/register/page.tsx`

**필드명 정리:**
| 용도 | 필드명 | 타입 |
|------|--------|------|
| 기본 제공 학생 수 | baseStudents | string |
| 최대 허용 학생 수 | maxStudents | string |
| 추가 학생 단가 | pricePerStudent | number |
| 월 비용 | monthlyPrice | number |

---

### 2.3 사용자 관리 (users)

**페이지:**
- `admin/users/[id]/edit/page.tsx`

**변경사항:**
- ✅ 회원 상태(status) select: USER_STATUSES.map() 동적 생성
- 상태 코드 변환: name(UI) → code(DB) 처리 필요

**업데이트 필요:**
- handleChange에서 상태 저장 시 `status.code` 사용 (현재 `status.name` 사용 가능)

---

### 2.4 시스템 관리 (system)

**페이지:** `admin/system/page.tsx`

**섹션별 상수 적용:**
1. **사용자 상태 관리**: USER_STATUSES.map()
2. **액세스코드 상태 + 사용기간**: ACCESS_CODE_STATUSES, ACCESS_CODE_DURATIONS
3. **레벨 시스템**: LEVEL_SYSTEM (18개 레벨 하드코딩 제거)

**업데이트 필요:**
- 모든 select 필드를 constants에서 map으로 동적 생성하는지 확인
- 상태 코드/이름 변환 로직 검증

---

### 2.5 액세스코드 관리 (accesscode)

**페이지:** `admin/accesscode/generate/page.tsx`

**변경사항:**
- ✅ 액세스코드 생성 시 사용기간 선택: ACCESS_CODE_DURATIONS.map()
- 상태 선택: ACCESS_CODE_STATUSES.map()

**업데이트 필요:**
- 생성 API 호출 시 days 값 포함 (ACCESS_CODE_DURATIONS에서 조회)

---

### 2.6 AI 모듈 관리 (ai-modules)

**현재 상태:** 별도의 상수화 작업 미실시

**향후 고려사항:**
- MODULE_CATEGORIES를 활용한 모듈 관리 페이지
- 스킬별 모듈 조회, 활성화/비활성화 관리

---

## 3. 정책변경사항

### 3.1 상태(Status) 정책

#### 사용자 vs 기관 상태 차이
| 상태명 | 사용자 코드 | 기관 코드 | 설명 |
|--------|-----------|---------|------|
| 첫번째 | active | active | 활성 상태 |
| 두번째 | **inactive** | **inactive** | 사용자: 비활성, 기관: 휴회 |
| 세번째 | suspended | suspended | 일시적 정지 |
| 네번째 | withdrawn | withdrawn | 탈퇴/중단 |

**주의:** UI 표시는 다르지만(비활성 vs 휴회), 코드는 동일(`inactive`)하게 유지

### 3.2 금액 입력 정책

#### 원 단위 입력 표준화
**모든 금액 필드는 원(₩) 단위로 입력:**
- 월비용: 100,000원
- 추가학생당단가: 5,000원
- API 연동비용: 30,000원
- 기술지원비: N원
- 콘텐츠 IP 전환비: N원

**표시 단위 (UI):**
- 만원 단위로 표시 가능 (내부적으로는 원 단위 저장)
- 예: 100,000 → "10만원" 또는 "100,000원"

### 3.3 기관 요금제 정책

#### 플랜별 자동 설정값
| 항목 | Starter | Pro | Enterprise |
|-----|---------|-----|------------|
| 월비용 | 100,000원 | 1,000,000원 | 협의 |
| 연납할인 | 5% | 10% | 0% |
| 기본학생수 | 10명 | 50명 | 100명 |
| 학생당단가 | 5,000원 | 10,000원 | 협의 |
| 최대학생수 | 50명 | 500명 | 무제한(0) |
| 관리계정수 | 2개 | 5개 | 10개 |
| API연동 | 미연동 | 연동 | 연동 |
| API비용 | 무료 | 30,000원 | 협의 |

---

## 4. 사용자 흐름 (User Flow)

### 4.1 기관 관리 (Institute Management)

#### UF-001: 새로운 기관 등록
```
1. 기관관리 페이지 접근 (/admin/institute)
2. "+ 기관 등록" 버튼 클릭
3. 등록 페이지로 이동 (/admin/institute/register)
4. Section 1 입력: 아이디, 기관명, 담당자 정보
   └─ 아이디 중복확인 API 호출
5. Section 2 입력 (선택): 지점수, 운영형태, 수강생 규모
6. Section 3 입력: 플랜 선택
   └─ DEFAULT_PLANS에서 값 자동 채움
   └─ monthlyPrice, pricePerStudent 등 6개 필드 초기화
7. Section 3 편집: 필요시 자동 채움값 수정
   └─ 월비용, 연납할인, 기본학생, 추가학생단가, 최대학생수 등 편집
8. Section 3 추가: API연동, 기술지원비, IP전환비, 기타 입력
9. Section 4 입력 (선택): 계약정보
10. "등록" 버튼 클릭
    └─ 필수 필드 검증 (아이디 중복확인, 기관명, 플랜 등)
    └─ POST /api/institutions 호출
11. 성공 메시지 표시
12. 기관 목록으로 이동 (/admin/institute)
```

#### UF-002: 기관 정보 수정
```
1. 기관 목록에서 기관 선택 (수정 버튼 클릭)
2. 수정 페이지로 이동 (/admin/institute/[id]/edit)
3. 기존 데이터 로드
   └─ GET /api/institutions/[id] 호출
   └─ 폼 필드에 데이터 채움
4. Section별 정보 수정
   └─ Section 3: 플랜 변경 시 관련 필드 자동 재계산
   └─ 필드 수정: monthlyPrice, maxStudents 등 개별 편집
5. "저장" 버튼 클릭
   └─ PATCH /api/institutions/[id] 호출
6. 성공 메시지 표시
7. 기관 목록으로 이동
```

### 4.2 플랜 관리 (Plan Management)

#### UF-003: 플랜 정보 확인
```
1. 플랜관리 페이지 접근 (/admin/plans)
2. 3개 플랜 카드(Starter, Pro, Enterprise) 표시
3. 각 카드에서 상세 정보 확인
   ├─ 월가격, 연납할인
   ├─ 기본학생수, 추가학생단가
   ├─ 최대학생수, 관리계정수
   └─ API연동 포함 여부
```

### 4.3 사용자 관리 (User Management)

#### UF-004: 사용자 상태 변경
```
1. 사용자관리 페이지 접근 (/admin/users)
2. 사용자 목록 조회 및 검색
3. 특정 사용자의 "수정" 버튼 클릭
4. 사용자 수정 페이지로 이동 (/admin/users/[id]/edit)
5. 상태 필드에서 드롭다운 열기
   └─ USER_STATUSES: 활성, 휴회, 정지, 탈퇴
6. 새 상태 선택
7. "저장" 버튼 클릭
   └─ PATCH /api/users/[id] 호출
   └─ status 코드('active', 'inactive', 'suspended', 'withdrawn') 저장
```

### 4.4 액세스코드 관리 (Access Code Management)

#### UF-005: 액세스코드 생성
```
1. 액세스코드 관리 페이지 접근 (/admin/accesscode)
2. "+ 액세스코드 생성" 버튼 클릭
3. 생성 페이지로 이동 (/admin/accesscode/generate)
4. 생성 정보 입력
   ├─ 사용자 유형 선택 (강사/학생)
   ├─ 생성 개수 입력 (1-1000)
   ├─ 기간 선택 → ACCESS_CODE_DURATIONS
   │   └─ 1개월(30일), 3개월(90일), 6개월(180일), 12개월(365일)
   └─ 등록 만료일 선택
5. "생성" 버튼 클릭
   └─ 유효성 검증
   └─ POST /api/accesscodes/generate 호출
   └─ 기간에서 days값 자동 계산 (발급일 + days = 만료일)
6. 생성 완료 메시지 표시
7. 액세스코드 목록으로 이동
   └─ 새 코드들이 상태 "활성"으로 표시됨
```

### 4.5 시스템 관리 (System Management)

#### UF-006: 시스템 정보 조회
```
1. 시스템관리 페이지 접근 (/admin/system)
2. Section 1: 사용자 상태 정의 확인
   └─ USER_STATUSES 테이블 표시
   └─ 활성, 휴회, 정지, 탈퇴 상태코드 및 설명 확인
3. Section 2: 액세스코드 상태 정의 확인
   └─ ACCESS_CODE_STATUSES 테이블 표시
   └─ 활성, 예약됨, 사용됨, 만료됨 상태 확인
4. Section 3: 액세스코드 기간 정의 확인
   └─ ACCESS_CODE_DURATIONS 테이블 표시
   └─ [기간명, 코드, 일수] 매핑 확인
5. Section 4: 레벨 시스템 확인
   └─ LEVEL_SYSTEM 테이블 표시 (18개 CEFR 레벨)
   └─ Basic, Intermediate, Advanced, Mastery 카테고리별 표시
```

### 4.6 AI 모듈 관리 (AI Modules Management) - 미구현

#### UF-007: AI 모듈 관리 (향후)
```
1. AI 모듈 관리 페이지 접근 (/admin/ai-modules) [향후]
2. 5개 AI 모듈 조회
   ├─ AI 생성 (활성, Pro/Enterprise)
   ├─ TTS (활성, Pro/Enterprise)
   ├─ 전략적 읽기 (활성, Pro/Enterprise)
   ├─ STT (미활성, 개발중)
   └─ 단어분석 (활성, 모든 플랜)
3. 모듈별 설정 확인
   ├─ 포함된 플랜
   ├─ 월간 사용 한도
   ├─ 초과 비용 정책
   └─ API 제공사 및 엔드포인트
```

---

## 5. IA 구조 및 기능 정의 (Information Architecture)

### 5.1 기관 관리 (Institute Management)

#### 기능: 기관 등록
**목적:** 새로운 기관 정보를 시스템에 등록  
**입력:**
- Section 1: 이메일(unique), 기관명, 기관유형, 담당자 정보, 회원상태
- Section 2: 지점수, 운영형태, 수강생 규모 (선택)
- Section 3: 플랜 선택 → 자동 채움
  - monthlyPrice, annualDiscount, baseStudents, pricePerStudent, maxAdminAccounts, apiIntegration
- Section 3 추가: maxStudents, apiCostAmount, techSupportCost, contentIPCost, notes
- Section 4: 계약정보 (선택)

**출력:** Institution 객체
```json
{
  "id": "uuid",
  "name": "K학원",
  "adminEmail": "admin@example.com",
  "plan": "Starter",
  "monthlyPrice": 100000,
  "pricePerStudent": 5000,
  "baseStudents": 10,
  "maxStudents": 50,
  "apiIntegration": "N",
  "status": "active"
}
```

**조건:**
- 이메일 중복 불가
- 기관명 2자 이상
- 플랜 필수 선택
- 모든 금액 필드는 원 단위 정수

**실패 처리:**
- 중복 이메일: "이미 등록된 이메일입니다"
- 필수 필드 공란: "필수 항목을 입력하세요"

---

#### 기능: 기관 정보 수정
**목적:** 기관 정보 변경 및 플랜 업데이트  
**입력:** 기관 ID + 수정할 필드들

**출력:** 업데이트된 Institution 객체

**조건:**
- 기관명은 읽기 전용 (수정 불가)
- maxStudents ≥ currentStudents 검증 (선택)
- 플랜 변경 시 관련 필드 자동 재계산

---

### 5.2 플랜 관리 (Plan Management)

#### 기능: 플랜 조회
**목적:** 3가지 가격제 정보 확인  
**입력:** 없음

**출력:** 3개 Plan 객체 배열
```json
[
  {
    "name": "Starter",
    "monthlyPrice": 100000,
    "annualDiscount": "5%",
    "baseStudents": 10,
    "pricePerStudent": 5000,
    "maxStudents": 50,
    "maxAdminAccounts": 2,
    "apiIntegration": "N"
  },
  ...
]
```

**조건:** DEFAULT_PLANS 상수에서 직접 참조

---

### 5.3 사용자 관리 (User Management)

#### 기능: 사용자 상태 변경
**목적:** 사용자 활성/비활성 상태 관리  
**입력:** 사용자 ID + 상태 코드 ('active', 'inactive', 'suspended', 'withdrawn')

**출력:** 업데이트된 User 객체

**조건:**
- 상태 코드는 USER_STATUSES에서만 선택
- 탈퇴('withdrawn') 상태는 복구 불가 (선택사항)

---

### 5.4 액세스코드 관리 (Access Code Management)

#### 기능: 액세스코드 생성
**목적:** 대량의 액세스코드를 일괄 생성  
**입력:**
- userType: 'teacher' | 'student'
- codeCount: 1-1000
- duration: '1M' | '3M' | '6M' | '12M'
- expiryDate: YYYY-MM-DD (미래 날짜)

**출력:** AccessCode 객체 배열
```json
{
  "code": "ABC123",
  "status": "active",
  "duration": "1M",
  "durationDays": 30,
  "expiryDate": "2026-04-14",
  "issuedDate": "2026-03-14"
}
```

**조건:**
- codeCount: 1 이상 1000 이하
- expiryDate: 현재 날짜보다 이후
- 기간별 일수 자동 계산 (ACCESS_CODE_DURATIONS 참조)

**실패 처리:**
- 잘못된 범위: "1~1000 사이의 개수만 가능합니다"
- 과거 날짜: "미래 날짜를 선택하세요"

---

### 5.5 시스템 관리 (System Management)

#### 기능: 상태/기간/레벨 정의 조회
**목적:** 시스템 전역 설정값 확인  
**입력:** 없음

**출력:** 4개 섹션의 상수 데이터
```json
{
  "userStatuses": [...],           // USER_STATUSES
  "accessCodeStatuses": [...],     // ACCESS_CODE_STATUSES
  "accessCodeDurations": [...],    // ACCESS_CODE_DURATIONS
  "levelSystem": [...]             // LEVEL_SYSTEM
}
```

**조건:** 읽기 전용

---

### 5.6 AI 모듈 관리 (AI Modules Management)

#### 기능: AI 모듈 조회 (향후)
**목적:** 활성화된 AI 기능 모듈 확인  
**입력:** 없음

**출력:** AIModule 객체 배열
```json
[
  {
    "id": "ai-generate",
    "name": "AI 생성",
    "status": "active",
    "includedPlans": ["Pro", "Enterprise"],
    "baseCost": 0,
    "monthlyLimit": null
  },
  ...
]
```

**조건:** 활성화된 모듈만 반환

---

## 6. 정책 (Policy/Business Rules)

### 6.1 상태(Status) 정책

#### 사용자 상태 (USER_STATUSES)
| 상태명 | 코드 | 특징 | 복구 | 권한 |
|--------|-------|------|------|------|
| 활성 | active | 정상 사용 중 | N/A | 모든 권한 |
| 휴회 | inactive | 일시 중단 | 가능 | 제한 (로그인 불가) |
| 정지 | suspended | 관리자 조치 | 가능 | 제한 (접근 불가) |
| 탈퇴 | withdrawn | 영구 탈퇴 | 불가 | 없음 (읽기 전용) |

#### 기관 상태 (INSTITUTION_STATUSES)
| 상태명 | 코드 | 특징 | 복구 | 서비스 |
|--------|-------|------|------|--------|
| 활성 | active | 정상 운영 | N/A | 제공 |
| 휴회 | inactive | 일시적 중단 | 가능 | 제한 (제한적 접근) |
| 정지 | suspended | 관리자 조치 | 가능 | 중단 |
| 탈퇴 | withdrawn | 완전 탈퇴 | 불가 | 없음 |

**주의:** UI 표시("휴회")와 코드('inactive')는 다를 수 있습니다.

### 6.2 액세스코드 정책

#### 상태(ACCESS_CODE_STATUSES)
| 상태명 | 코드 | 설명 | 다음 상태 |
|--------|-------|------|---------|
| 활성 | active | 사용 중인 유효 코드 | → used / expired |
| 예약됨 | reserved | 생성됨, 미배포 | → active |
| 사용됨 | used | 할당량 모두 사용 | × (최종) |
| 만료됨 | expired | 유효기간 종료 | × (최종) |

#### 유효기간(ACCESS_CODE_DURATIONS)
| 기간명 | 코드 | 일수 | 용도 | 만료일 계산 |
|--------|------|------|------|------------|
| 1개월 | 1M | 30 | 단기 트라이얼 | 발급일 + 30일 |
| 3개월 | 3M | 90 | 분기 구독 | 발급일 + 90일 |
| 6개월 | 6M | 180 | 반년 구독 | 발급일 + 180일 |
| 12개월 | 12M | 365 | 연간 구독 | 발급일 + 365일 |

**정책:**
- 유효기간 선택 시 만료일은 자동 계산 (수정 불가)
- 만료 후 자동 갱신 정책은 별도로 정의

### 6.3 기관 요금제 정책

#### 플랜별 기본 설정
| 항목 | Starter | Pro | Enterprise |
|-----|---------|-----|------------|
| **월 가격** | 100,000원 | 1,000,000원 | 협의 |
| **연납 할인** | 5% | 10% | 0% |
| **기본 학생** | 10명 | 50명 | 100명 |
| **추가 학생 단가** | 5,000원 | 10,000원 | 협의 |
| **최대 학생** | 50명 | 500명 | 무제한 |
| **관리 계정** | 2개 | 5개 | 10개 |
| **API 연동** | 미포함 | 포함 | 포함 |
| **API 비용** | 무료 | 월 30,000원 | 협의 |

#### 플랜 선택 시 자동 채움 필드 (6개)
```
monthlyPrice        ← Plan.monthlyPrice
annualDiscount      ← Plan.annualDiscount
baseStudents        ← Plan.baseStudents
pricePerStudent     ← Plan.pricePerStudent
maxAdminAccounts    ← Plan.maxAdminAccounts
apiIntegration      ← Plan.apiIntegration
```

**정책:**
- 자동 채움 이후 모든 필드 편집 가능
- 플랜 변경 시 위 6개 필드는 재계산 (사용자 입력값 덮어쓰기)
- 다른 필드(maxStudents, techSupportCost 등)는 유지됨

### 6.4 금액 입력 정책

#### 원 단위 저장 표준화
```
모든 금액 필드는 원(₩) 단위 정수로 저장:
- monthlyPrice: 100000 (= 10만원)
- pricePerStudent: 5000 (= 5천원)
- apiCostAmount: 30000 (= 3만원)
- techSupportCost: 50000 (= 5만원)
- contentIPCost: 100000 (= 10만원)
```

#### 표시 정책
- UI 입력: 숫자만 입력 (콤마 자동)
- UI 표시: 3자리 구분 + "원" 단위
  예: `100,000원`, `5,000원`

#### 검증
- 음수 불가: 0 이상만 허용
- 최대값: 플랜별 maxStudents 제한 적용 (선택)

### 6.5 필드 분리 정책

#### baseStudents vs maxStudents
```
baseStudents (학생기본제공)
├─ 의미: 플랜에 포함된 기본 제공 학생 수
├─ 초기화: 플랜 선택 시 자동 채움
├─ 수정: 사용자 편집 가능
└─ 플랜 변경 시: 재계산됨

maxStudents (최대학생수)
├─ 의미: 기관이 운영할 수 있는 최대 학생 수
├─ 초기화: 플랜 선택 시 자동 채움
├─ 수정: 사용자 직접 입력
└─ 플랜 변경 시: 재계산됨

관계: 독립적 (교차 검증 없음, 선택사항)
예: Starter 선택 → baseStudents=10, maxStudents=50
    사용자가 수정 → baseStudents=20, maxStudents=40 가능
```

### 6.6 레벨 시스템 정책

#### CEFR 18단계
```
카테고리       레벨 범위    CEFR       특징
────────────────────────────────────────
Basic          1-2         A1-A2     기초 단계
Intermediate   3-4         B1-B2     중급 단계
Advanced       5-6         C1-C2     고급 단계
Mastery        7-18        (확장)    숙달 단계
```

**정책:**
- 각 레벨별 학습 요구사항 정의 (향후)
- 레벨 업그레이드 조건 설정 (향후)
- 레벨별 콘텐츠 추천 (향후)

### 6.7 AI 모듈 정책 (향후)

#### 포함 플랜별 정책
| 모듈 | Starter | Pro | Enterprise | 상태 |
|-----|---------|-----|------------|------|
| AI 생성 | × | ○ | ○ | 활성 |
| TTS | × | ○ | ○ | 활성 |
| 전략적 읽기 | × | ○ | ○ | 활성 |
| 단어분석 | ○ | ○ | ○ | 활성 |
| STT | × | × | × | 미활성 |

#### 월간 사용 한도
```
모듈             Starter    Pro         Enterprise
─────────────────────────────────────
AI 생성          -          무제한      무제한
TTS              -          10,000자    무제한
전략적 읽기      -          무제한      무제한
단어분석         무제한      무제한      무제한
STT              -          향후        향후
```

#### 초과 비용
```
모듈             단위      초과 비용
────────────────────────────────
AI 생성          요청/회   1,000원
TTS              1,000자   50원
전략적 읽기      -         없음 (무제한)
단어분석         -         없음 (무제한)
```

---

### 4.1 Admin 메뉴 구조
```
Dashboard (📊)
기관관리 (🏢)
  ├─ 기관 목록 조회
  ├─ 기관 등록
  └─ 기관 수정/삭제
사용자관리 (👥)
  ├─ 사용자 목록 조회
  └─ 사용자 수정
액세스코드관리 (🔑)
  ├─ 코드 생성
  └─ 코드 목록 조회
플랜관리 (📋) [NEW]
  ├─ 플랜 목록 (Starter, Pro, Enterprise)
  └─ 플랜 수정
수업 모듈 (🤖)
Billing (💳)
시스템관리 (⚙️)
  ├─ 사용자 상태 관리
  ├─ 액세스코드 상태/기간 관리
  └─ 레벨 시스템 관리
```

### 4.2 데이터 흐름 (기관 등록 예시)

```
기관 등록 페이지
  ├─ Section 1: 가입정보
  │   ├─ 이메일: 중복확인 필수
  │   ├─ 임시비밀번호
  │   ├─ 기관명, 유형
  │   ├─ 담당자 정보
  │   └─ 회원상태 (INSTITUTION_STATUSES)
  │
  ├─ Section 2: 부가정보
  │   ├─ 지점수
  │   ├─ 운영형태
  │   └─ 현재 수강생 규모
  │
  ├─ Section 3: 라이선스 & 요금제
  │   ├─ 플랜 선택 (DEFAULT_PLANS)
  │   │   └─ 자동 채움: monthlyPrice, annualDiscount, baseStudents, 
  │   │              pricePerStudent, maxAdminAccounts, apiIntegration
  │   ├─ 월비용 (편집 가능)
  │   ├─ 연납할인 (편집 가능)
  │   ├─ 학생기본제공 (편집 가능)
  │   ├─ 추가학생당단가 (편집 가능)
  │   ├─ 최대학생수 (편집 가능)
  │   ├─ 최대관리계정수 (편집 가능)
  │   ├─ API연동 선택
  │   ├─ API연동비용 (편집 가능)
  │   ├─ 기술지원비 (편집 가능)
  │   ├─ 콘텐츠IP전환비 (편집 가능)
  │   └─ 기타 유의사항
  │
  └─ Section 4: 계약 정보
      ├─ 계약상태
      ├─ 계약 시작/종료일
      ├─ 자동갱신 여부
      └─ 갱신 조건

↓ 제출 (POST /api/institutions)

생성된 기관 데이터:
{
  name: string;
  adminEmail: string;
  plan: 'Starter' | 'Pro' | 'Enterprise';
  monthlyPrice: number;
  pricePerStudent: number;
  baseStudents: number;
  maxStudents: number;       // 실제 저장값
  maxAdminAccounts: number;
  apiIntegration: 'Y' | 'N';
  apiCostAmount: number;
  techSupportCost: number;
  contentIPCost: number;
  // ... 계약정보
}
```

---

## 8. 개발자 체크리스트 (Developer Tasks)

### 8.1 기관 관리 (Institute Management)

#### Frontend - institute/register/page.tsx & institute/[id]/edit/page.tsx
- [ ] DEFAULT_PLANS 상수 import 확인
- [ ] Form state 필드 추가
  - [ ] baseStudents: string (학생기본제공)
  - [ ] maxStudents: string (최대학생수)
  - [ ] monthlyPrice: number (월비용)
  - [ ] pricePerStudent: number (추가학생단가)
  - [ ] apiCostAmount: string (API비용)
  - [ ] techSupportCost: string (기술지원비)
  - [ ] contentIPCost: string (IP전환비)
  - [ ] notes: string (기타)

- [ ] handleChange() 로직 구현
  ```typescript
  if (name === 'plan') {
    const planOption = DEFAULT_PLANS.find((p) => p.name === value);
    // 6개 필드 자동 채움
  }
  ```
  
- [ ] 숫자 필드 처리
  - [ ] monthlyPrice, pricePerStudent, apiCostAmount 등 parseInt 변환
  - [ ] 음수 입력 검증

- [ ] Section 3 레이아웃 (7행 구조)
  - [ ] Row 1: 플랜 (Select)
  - [ ] Row 2: 월비용 / 연납할인 (Input/Input)
  - [ ] Row 3: 학생기본제공 / 추가학생단가 (Input/Input)
  - [ ] Row 4: 최대학생수 / 최대관리계정수 (Input/Input)
  - [ ] Row 5: API연동 / API비용 (Select/Input)
  - [ ] Row 6: 기술지원비 / IP전환비 (Input/Input)
  - [ ] Row 7: 기타 (Textarea)

- [ ] 자동 채움 후 수정 가능 확인 (readonly 제거)

#### Backend - Institution API
- [ ] POST /api/institutions 엔드포인트
  - [ ] 요청 DTO에 필드 추가
    - [ ] monthlyPrice: number
    - [ ] pricePerStudent: number (apiCostAmount 아님)
    - [ ] baseStudents: number
    - [ ] maxStudents: number
    - [ ] apiIntegration: string
    - [ ] apiCostAmount: number
    - [ ] techSupportCost: number
    - [ ] contentIPCost: number
    - [ ] notes: string (선택)
  
  - [ ] 유효성 검증
    - [ ] monthlyPrice ≥ 0
    - [ ] pricePerStudent ≥ 0
    - [ ] maxStudents ≥ 1
    - [ ] apiCostAmount ≥ 0
  
  - [ ] DB 저장
    - [ ] 모든 금액 필드는 원 단위 숫자로 저장
    - [ ] plan 코드 변환 (필요시: Starter → lite)

- [ ] PATCH /api/institutions/:id 엔드포인트
  - [ ] 동일한 필드 업데이트 지원
  - [ ] 플랜 변경 시 관련 필드 처리

- [ ] GET /api/institutions/:id 엔드포인트
  - [ ] 응답에 모든 필드 포함
    - [ ] monthlyPrice
    - [ ] pricePerStudent
    - [ ] baseStudents
    - [ ] maxStudents
    - [ ] 기타 Section 3 필드

- [ ] DB 마이그레이션
  - [ ] baseStudents 컬럼 추가
  - [ ] maxStudents 컬럼 추가 (또는 기존 확인)
  - [ ] monthlyPrice 컬럼 추가
  - [ ] pricePerStudent 컬럼 추가
  - [ ] apiCostAmount 컬럼 추가
  - [ ] techSupportCost 컬럼 추가
  - [ ] contentIPCost 컬럼 추가

#### 통합 테스트
- [ ] 기관 등록: 플랜 선택 → 6개 필드 자동 채움 확인
- [ ] 기관 등록: 자동 채움값 수정 후 저장 확인
- [ ] 기관 수정: 플랜 변경 → 필드 재계산 확인
- [ ] 조회: 저장된 금액이 원 단위로 정상 반환 확인

---

### 8.2 플랜 관리 (Plan Management)

#### Frontend - admin/plans/page.tsx
- [ ] DEFAULT_PLANS import 확인
- [ ] 플랜 목록 렌더링
  ```typescript
  DEFAULT_PLANS.map(plan => (
    <Card>
      {plan.name}
      {plan.monthlyPrice}
      ...
    </Card>
  ))
  ```

- [ ] 각 플랜별 2행 레이아웃
  - [ ] Row 1: 플랜명 | 월가격
  - [ ] Row 2: 기본학생/추가학생단가 | 최대학생/관리계정

#### Backend
- [ ] GET /api/plans 엔드포인트 (선택, 현재 상수만 사용)

---

### 8.3 사용자 관리 (User Management)

#### Frontend - admin/users/[id]/edit/page.tsx
- [ ] USER_STATUSES import 확인
- [ ] 상태 select 동적 생성
  ```typescript
  <select value={status}>
    {USER_STATUSES.map(s => (
      <option value={s.code}>{s.name}</option>
    ))}
  </select>
  ```

- [ ] form.status 필드 처리 (code 값 저장)

#### Backend - User API
- [ ] PATCH /api/users/:id
  - [ ] status 필드: 'active' | 'inactive' | 'suspended' | 'withdrawn'

#### 통합 테스트
- [ ] 상태 변경 저장 → 조회 시 정상 반환 확인

---

### 8.4 액세스코드 관리 (Access Code Management)

#### Frontend - admin/accesscode/generate/page.tsx
- [ ] ACCESS_CODE_DURATIONS import 확인
- [ ] 기간 select 동적 생성
  ```typescript
  <select value={duration}>
    {ACCESS_CODE_DURATIONS.map(d => (
      <option value={d.code}>{d.name}</option>
    ))}
  </select>
  ```

- [ ] duration에서 days 값 조회 및 전달

#### Backend - AccessCode API
- [ ] POST /api/accesscodes/generate
  - [ ] duration 필드로 days 계산
  - [ ] expiryDate = issuedDate + days 자동 계산
  - [ ] status 초기값: 'active'

#### 통합 테스트
- [ ] 기간 선택 (예: 3M) → 90일 계산 확인
- [ ] 만료일 = 발급일 + 90일 확인

---

### 8.5 시스템 관리 (System Management)

#### Frontend - admin/system/page.tsx
- [ ] 5개 상수 import 확인
  - [ ] USER_STATUSES
  - [ ] INSTITUTION_STATUSES
  - [ ] ACCESS_CODE_STATUSES
  - [ ] ACCESS_CODE_DURATIONS
  - [ ] LEVEL_SYSTEM

- [ ] Section별 테이블 렌더링
  - [ ] Section 1: USER_STATUSES.map()
  - [ ] Section 2: ACCESS_CODE_STATUSES.map()
  - [ ] Section 3: ACCESS_CODE_DURATIONS.map() (days 컬럼 필수)
  - [ ] Section 4: LEVEL_SYSTEM.map()

#### 통합 테스트
- [ ] 각 Section 동적 생성 확인
- [ ] 상수 변경 → 페이지 렌더링 확인

---

### 8.6 AI 모듈 관리 (AI Modules Management)

#### Frontend - admin/ai-modules/page.tsx (향후)
- [ ] AI_MODULES 상수 정의 (미정의)
- [ ] 5개 모듈 카드 렌더링
- [ ] 모듈별 상세 정보 표시

#### Backend
- [ ] GET /api/ai-modules 엔드포인트 (설계)

---

### 8.7 공통 (Common)

#### 상수 검증 (src/lib/constants.ts)
- [ ] 5개 상수 정의 완료
  - [ ] USER_STATUSES (4개)
  - [ ] INSTITUTION_STATUSES (4개)
  - [ ] ACCESS_CODE_STATUSES (4개)
  - [ ] ACCESS_CODE_DURATIONS (4개)
  - [ ] LEVEL_SYSTEM (18개)
  - [ ] DEFAULT_PLANS (3개)

- [ ] 각 상수 인터페이스 정의
  - [ ] StatusRow
  - [ ] DurationRow
  - [ ] LevelRow
  - [ ] Plan

#### API Client (src/lib/api.ts)
- [ ] Institution API 함수
  ```typescript
  POST /api/institutions (createInstitution)
  PATCH /api/institutions/:id (updateInstitution)
  GET /api/institutions/:id (fetchInstitution)
  ```

- [ ] User API 함수
  ```typescript
  PATCH /api/users/:id (updateUser)
  ```

- [ ] AccessCode API 함수
  ```typescript
  POST /api/accesscodes/generate (generateAccessCodes)
  ```

#### 데이터 타입 일관성
- [ ] 상태 코드: 항상 code 값 사용 (name 아님)
  - [ ] 저장: code ('active', 'inactive' 등)
  - [ ] 표시: name ('활성', '휴회' 등)

- [ ] 금액 필드: 원 단위 정수
  - [ ] 저장: 숫자만 (100000)
  - [ ] 표시: 3자리 구분 + 원 (100,000원)

- [ ] 기간 필드: days 활용
  - [ ] 저장: days 정수 (30, 90, 180, 365)
  - [ ] 표시: DurationRow.name ('1개월', '3개월' 등)

---

## 9. 마이그레이션 가이드

### 9.1 기존 데이터 호환성

**선택 필드 연결:**
```
기존: unitPrice: "1.2만원/명" (표시용만 사용)
신규: pricePerStudent: 5000 (실제 금액, 원 단위)
      unitPrice: "1.2만원/명" (표시용, 선택사항)
```

**필드 매핑:**
| 기존 필드 | 신규 필드 | 타입 | 설명 |
|----------|----------|------|------|
| (없음) | baseStudents | number | 플랜 기본 학생 수 |
| maxStudents | maxStudents | number | 최대 허용 학생 수 |
| (없음) | monthlyPrice | number | 월 비용 |
| (표시용) | pricePerStudent | number | 추가 학생 단가 |
| (없음) | apiCostAmount | number | API 연동 비용 |

### 9.2 데이터 백필 (Data Backfill)

**기존 기관 데이터에 신규 필드 추가 (선택):**

```sql
-- 1단계: baseStudents 추가 (플랜별 기본값)
UPDATE institutions
SET base_students = CASE plan
  WHEN 'lite' THEN 10
  WHEN 'pro' THEN 50
  WHEN 'enterprise' THEN 100
END
WHERE base_students IS NULL;

-- 2단계: monthlyPrice 추가 (기존 가격 데이터에서)
UPDATE institutions
SET monthly_price = CASE plan
  WHEN 'lite' THEN 100000
  WHEN 'pro' THEN 1000000
  ELSE 0
END
WHERE monthly_price IS NULL;

-- 3단계: pricePerStudent 추가
UPDATE institutions
SET price_per_student = CASE plan
  WHEN 'lite' THEN 5000
  WHEN 'pro' THEN 10000
  ELSE 0
END
WHERE price_per_student IS NULL;
```

### 9.3 배포 단계

#### Phase 1: 백엔드 준비 (1-2일)
1. Prisma schema 수정
   - baseStudents, monthlyPrice, pricePerStudent 등 컬럼 추가
   ```bash
   pnpm prisma migrate dev --name add_section3_fields
   ```

2. API 서비스 업데이트
   - POST/PATCH/GET 엔드포인트 DTO 확장
   - 유효성 검증 로직 추가

3. 데이터 백필 (선택)
   - 기존 기관 데이터 마이그레이션

4. 백엔드 배포
   - 테스트 서버 배포 및 검증
   - 프로덕션 배포

#### Phase 2: 프론트엔드 배포 (1-2일)
1. 상수 적용
   - src/lib/constants.ts 5개 상수 정의
   - pnpm run build 실행

2. 페이지 업데이트
   - institute/*.tsx: Section 3 7행 레이아웃 적용
   - users/*.tsx: USER_STATUSES 적용
   - system/page.tsx: 4개 상수 적용
   - accesscode/*.tsx: ACCESS_CODE_DURATIONS 적용
   - plans/page.tsx: DEFAULT_PLANS 적용

3. API 클라이언트 업데이트
   - src/lib/api.ts 함수 확장

4. 프론트엔드 배포
   - 스테이징 배포 및 E2E 테스트
   - 프로덕션 배포

#### Phase 3: 검증 및 모니터링 (1-2일)
1. 데이터 정합성 검증
   - 기존 기관 정상 조회 확인
   - 신규 기관 생성/수정 테스트

2. UI/UX 검증
   - 플랜 선택 자동 채움 동작 확인
   - 금액 표시 단위 확인
   - 모든 페이지 상태 선택 정상 동작 확인

3. 성능 모니터링
   - API 응답 시간 확인
   - 데이터베이스 쿼리 퍼포먼스 확인

---

## 10. 참고 파일 및 코드

### 문서
- [institute.md](docs/institute.md) - 기관 관리 모듈 가이드
- [plans.md](docs/plans.md) - 플랜 관리 모듈 가이드
- [system.md](docs/system.md) - 시스템 관리 모듈 가이드
- [ai-modules.md](docs/ai-modules.md) - AI 모듈 관리 (계획)
- [users.md](docs/users.md) - 사용자 관리 가이드
- [accesscode.md](docs/accesscode.md) - 액세스코드 관리 가이드

### 코드 파일
- `src/lib/constants.ts` - 모든 공유 상수 정의
- `src/lib/api.ts` - API 호출 함수
- `src/app/(admin)/admin/institute/[id]/edit/page.tsx` - 기관 수정
- `src/app/(admin)/admin/institute/register/page.tsx` - 기관 등록
- `src/app/(admin)/admin/users/[id]/edit/page.tsx` - 사용자 수정
- `src/app/(admin)/admin/system/page.tsx` - 시스템 관리
- `src/app/(admin)/admin/plans/page.tsx` - 플랜 관리 (신규)
- `src/app/(admin)/admin/accesscode/generate/page.tsx` - 액세스코드 생성

---

## 11. FAQ

### Q1: 왜 baseStudents와 maxStudents를 분리했는가?
**A:** 
- `baseStudents`: 플랜에 포함된 기본 제공 학생 수 (플랜 변경 시 재계산)
- `maxStudents`: 기관이 실제로 운영할 수 있는 최대 학생 수 (사용자 직접 입력)

두 값은 비즈니스 의미가 다르기 때문에 분리하여 관리합니다.

### Q2: 플랜 선택 후 필드를 다시 수정할 수 있는가?
**A:** 네. 플랜 선택 시 6개 필드가 자동으로 초기화되지만, 초기화된 값들도 모두 편집 가능합니다. 
다른 필드(maxStudents, techSupportCost 등)는 초기화되지 않고 유지됩니다.

### Q3: 플랜을 변경하면 모든 필드가 재계산되는가?
**A:** 
- **재계산됨:** monthlyPrice, annualDiscount, baseStudents, pricePerStudent, maxAdminAccounts, apiIntegration (6개)
- **유지됨:** maxStudents, apiCostAmount, techSupportCost, contentIPCost, notes

### Q4: 금액 필드의 단위는 어떻게 되는가?
**A:** 모든 금액은 **원 단위** 정수로 저장합니다.
- 입력: 100000 (숫자만)
- 저장: 100000 (원 단위)
- 표시: 100,000원 (UI에서 포매팅)

### Q5: 기존 기관의 요금제 필드가 빈 경우는?
**A:** 
- API 응답: 기본값 제공 (monthlyPrice: 0, pricePerStudent: 0)
- 수정 페이지: 필드가 빈 경우 플랜 재선택으로 자동 채움
- 또는 데이터 백필 마이그레이션으로 기존 데이터 보정

### Q6: unitPrice와 pricePerStudent의 차이는?
**A:**
- `pricePerStudent`: 실제 가격 (숫자, 원 단위) - DB 저장용
  ```
  5000  // 5천원/명
  ```

- `unitPrice`: 표시 형식 (문자열) - UI 표시용
  ```
  "1.2만원/명"
  ```

### Q7: 상태 코드와 이름이 어떻게 다른가?
**A:**
- **코드:** 저장값 (active, inactive, suspended, withdrawn)
  ```
  status: 'active'  // DB 저장
  ```

- **이름:** 표시값 (활성, 휴회, 정지, 탈퇴)
  ```
  <option value='active'>활성</option>  // UI 표시
  ```

### Q8: AI 모듈 관리는 언제 구현되는가?
**A:** 현재(2026-03-14)는 계획 단계입니다. 
- 1단계: AI_MODULES 상수 정의 (04월 예정)
- 2단계: /admin/ai-modules 페이지 구현 (05월 예정)
- 3단계: 모듈별 사용 한도 관리 (06월 예정)

### Q9: 기관 이름은 수정할 수 있는가?
**A:** 아니오. 기관 이름은 **등록 후 변경 불가** (읽기 전용)입니다. 
변경이 필요한 경우 관리자에게 문의하도록 안내하세요.

### Q10: 데이터 타입 검증은 어디서 하는가?
**A:** 
- **Frontend:** handleChange()에서 즉시 검증 (음수 불가 등)
- **Backend:** API 수신 시 DTO validation (Zod 또는 class-validator)
- **Database:** 컬럼 타입 제약 (NOT NULL, decimal 등)

---

**문서 작성일:** 2026.03.14  
**최종 수정:** 2026.03.14  
**버전:** 2.0 (모든 문서 통합)  
**담당자:** AI Assistant
