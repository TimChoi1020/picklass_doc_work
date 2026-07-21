# 기관 관리 (Institute Management)

## 📋 개요

Picklass 관리자 페이지의 기관(학원) 관리 시스템입니다. 기관 등록, 수정, 조회 및 요금제 관리를 제공합니다.

**파일 경로:**
- 목록: `src/app/(admin)/admin/institute/page.tsx`
- 등록: `src/app/(admin)/admin/institute/register/page.tsx`
- 수정: `src/app/(admin)/admin/institute/[id]/edit/page.tsx`

**상태:** 업데이트 완료 (2026-03-14)

---

## 🎯 주요 기능

### 1. 기관 관리
- ✅ 기관 정보 조회 (목록)
- ✅ 기관 신규 등록
- ✅ 기관 정보 수정
- ✅ 기관 상태 관리

### 2. 요금제 관리 (NEW)
- ✅ 플랜별 가격 정보 관리
- ✅ 플랜 선택 시 자동 설정값 채움
- ✅ 기본 학생 수, 추가 학생 단가 설정
- ✅ API 연동 상태 및 비용 관리

### 3. 계약 정보 관리
- ✅ 계약 상태 추적 (협의중, 계약완료, 계약중, 만료, 해지)
- ✅ 계약 기간 정보 (시작일, 종료일)
- ✅ 자동갱신 여부 설정
- ✅ 갱신 조건 기록

---

## 🏗️ IA 구조

```
기관관리 (/admin/institute)
├── 기관 목록 조회
│   ├── 검색 필터
│   ├── 기관 테이블
│   │   ├── 기관명
│   │   ├── 담당자
│   │   ├── 상태
│   │   └── 액션 (수정, 삭제)
│   └── "+ 기관 등록" 버튼
│
├── 기관 등록 (/admin/institute/register)
│   ├── Section 1: 가입정보
│   │   ├── 아이디 (이메일) * [중복확인 필수]
│   │   ├── 초기 임시 비밀번호 *
│   │   ├── 기관명 *
│   │   ├── 기관 유형 * [개인학원, 프랜차이즈, 어학원, 공교육, 기업교육]
│   │   ├── 담당자 성명 *
│   │   ├── 담당자 연락처 *
│   │   └── 회원 상태 * [활성, 휴회, 정지, 탈퇴]
│   │
│   ├── Section 2: 부가정보
│   │   ├── 지점 수
│   │   ├── 운영 형태 [직영, 가맹]
│   │   └── 현재 수강생 규모
│   │
│   ├── Section 3: 라이선스 & 요금제
│   │   ├── 플랜 (선택) *
│   │   ├── 월비용 (원) [플랜 선택 시 자동, 편집 가능]
│   │   ├── 연납할인 [플랜 선택 시 자동, 편집 가능]
│   │   ├── 학생기본제공 [플랜 선택 시 자동, 편집 가능]
│   │   ├── 추가학생당단가 (원) [플랜 선택 시 자동, 편집 가능]
│   │   ├── 최대학생수 [편집 가능]
│   │   ├── 최대관리계정수 [편집 가능]
│   │   ├── API연동 (선택) [미연동, 연동]
│   │   ├── API연동비용 (원)
│   │   ├── 기술지원비 (원)
│   │   ├── 콘텐츠IP전환비 (원)
│   │   └── 기타 [유의사항]
│   │
│   └── Section 4: 계약 기본 정보
│       ├── 계약 상태 [협의중, 계약완료, 계약중, 만료, 해지]
│       ├── 계약 시작일
│       ├── 계약 종료일
│       ├── 자동갱신 여부 [예, 아니오]
│       └── 갱신 조건
│
└── 기관 수정 (/admin/institute/[id]/edit)
    └── [등록과 동일한 구조]
```

---

## 📊 데이터 구조

### 기관 데이터 모델
```typescript
interface Institution {
  // Section 1: 가입정보
  adminEmail: string;        // 아이디 (이메일)
  name: string;              // 기관명
  institutionType: string;   // 기관 유형
  partnerManager: string;    // 담당자 성명
  partnerManagerPhone: string; // 담당자 연락처
  status: 'active' | 'inactive' | 'suspended' | 'withdrawn'; // 회원 상태
  
  // Section 2: 부가정보
  branchCount?: number;      // 지점 수
  operationType?: string;    // 운영 형태
  currentStudents?: number;  // 현재 수강생 규모
  
  // Section 3: 라이선스
  plan: 'Starter' | 'Pro' | 'Enterprise'; // 플랜
  monthlyPrice: number;      // 월비용 (원)
  annualDiscount: string;    // 연납할인 (예: "5%")
  baseStudents: number;      // 학생기본제공 (명)
  pricePerStudent: number;   // 추가학생당단가 (원)
  maxStudents: number;       // 최대학생수 (명)
  maxAdminAccounts: number;  // 최대관리계정수
  apiIntegration: 'Y' | 'N'; // API연동 여부
  apiCostAmount?: number;    // API연동비용 (원)
  techSupportCost?: number;  // 기술지원비 (원)
  contentIPCost?: number;    // 콘텐츠IP전환비 (원)
  notes?: string;            // 기타 유의사항
  
  // Section 4: 계약정보
  contractStatus?: string;   // 계약 상태
  contractStartDate?: string; // 계약 시작일
  contractPeriod?: string;   // 계약 종료일
  autoRenewal?: string;      // 자동갱신 여부
  renewalCondition?: string; // 갱신 조건
}
```

### 플랜별 설정값 (DEFAULT_PLANS)
| 항목 | Starter | Pro | Enterprise |
|-----|---------|-----|------------|
| 월비용 | 100,000원 | 1,000,000원 | 협의 |
| 연납할인 | 5% | 10% | 0% |
| 기본학생수 | 10명 | 50명 | 100명 |
| 학생당단가 | 5,000원 | 10,000원 | 협의 |
| 최대학생수 | 50명 | 500명 | 무제한 |
| 관리계정수 | 2개 | 5개 | 10개 |
| API연동 | 미연동 | 연동 | 연동 |
| API비용 | 무료 | 30,000원 | 협의 |

---

## ⚙️ 정책 변경사항

### 1. 기관 상태 정의
**상태 정의 (INSTITUTION_STATUSES):**
```
- 활성 (code: active): 정상 운영 중인 기관
- 휴회 (code: inactive): 일시적으로 운영 중단 (사용자 비활성과 구분)
- 정지 (code: suspended): 관리자 조치로 인한 임시 정지
- 탈퇴 (code: withdrawn): 기관 완전 탈퇴
```

**주의:** UI 표시("휴회" vs "비활성")는 다르지만, DB 저장 코드(`inactive`)는 동일합니다.

### 2. 요금제 자동 채움 정책
플랜 선택 시 다음 필드가 자동으로 DEFAULT_PLANS 값으로 초기화됩니다:
- `monthlyPrice` (월비용)
- `annualDiscount` (연납할인)
- `baseStudents` (학생기본제공)
- `pricePerStudent` (추가학생당단가)
- `maxAdminAccounts` (최대관리계정수)
- `apiIntegration` (API연동 상태)

**초기화 이후 모든 필드는 편집 가능합니다.**

### 3. 금액 입력 정책
**원 단위 입력 표준화:**
```
monthlyPrice: 100,000  // 10만원 입력 시
pricePerStudent: 5,000 // 5천원 입력 시
apiCostAmount: 30,000  // 3만원 입력 시
techSupportCost: N원
contentIPCost: N원
```

### 4. 필수 필드 정책
**등록 시 필수:**
- 이메일 (중복확인 필수)
- 기관명 (2자 이상)
- 기관 유형
- 담당자 정보 (성명, 연락처)
- 회원 상태
- 플랜 (선택)

---

## 🔧 개발자 체크리스트

### Form State 관리
- [ ] `monthlyPrice`, `pricePerStudent`는 number 타입 (원 단위)
- [ ] `baseStudents`, `maxStudents` 필드 분리
  - `baseStudents`: 학생기본제공 (플랜에서 초기화)
  - `maxStudents`: 최대학생수 (사용자 직접 입력)
- [ ] `annualDiscount`: string 타입 ("5%", "10%" 형식)

### handleChange 로직
```typescript
if (name === 'plan') {
  const planOption = DEFAULT_PLANS.find((p) => p.name === value);
  setForm((p) => ({
    ...p,
    [name]: value,
    monthlyPrice: planOption?.monthlyPrice ?? 0,
    pricePerStudent: planOption?.pricePerStudent ?? 0,
    baseStudents: String(planOption?.baseStudents ?? ''),
    annualDiscount: String(planOption?.annualDiscount ?? ''),
    maxAdminAccounts: String(planOption?.maxAdminAccounts ?? ''),
    apiIntegration: planOption?.apiIntegration ?? '',
  }));
}

// 숫자 필드 처리
if (['monthlyPrice', 'pricePerStudent', 'apiCostAmount', 'techSupportCost', 'contentIPCost'].includes(name)) {
  setForm((p) => ({ ...p, [name]: value === '' ? 0 : parseInt(value) }));
}
```

### API 검증 (Backend)
- [ ] `POST /api/institutions` - 기관 등록
  - monthlyPrice 원 단위 저장
  - pricePerStudent 원 단위 저장
  - maxStudents 값 저장 (baseStudents와 구분)
  
- [ ] `PATCH /api/institutions/:id` - 기관 수정
  - 플랜 변경 시 관련 필드 업데이트
  - 기존 데이터와의 호환성 유지

- [ ] `GET /api/institutions/:id` - 기관 조회
  - 모든 필드 반환 (monthlyPrice, pricePerStudent 포함)

### 폼 검증 (Frontend)
- [ ] 이메일 중복확인 필수 체크
- [ ] 기관명 2자 이상 입력 검증
- [ ] 플랜 필수 선택 검증
- [ ] 숫자 필드 음수 불가 검증
- [ ] 계약 관련 날짜: 시작일 < 종료일 검증

---

## 📝 사용 예시

### 기관 등록 플로우
```
1. /admin/institute/register 접근
2. Section 1 정보 입력 (이메일, 기관명 등)
3. Section 2 부가정보 입력 (선택)
4. Section 3에서 플랜 선택
   → 월비용, 연납할인, 학생기본제공, 추가학생당단가, 관리계정수 자동 채움
5. 필요시 자동 채움된 값 수정
6. API 비용, 기술지원비, 콘텐츠IP비 입력
7. Section 4 계약정보 입력 (선택)
8. 등록 버튼 클릭
```

### 플랜 변경 시나리오
```
기존: Starter 플랜 (월 10만원)
→ Pro 플랜으로 변경
  - 월비용: 100,000 → 1,000,000
  - 연납할인: 5% → 10%
  - 기본학생수: 10명 → 50명
  - 학생당단가: 5,000 → 10,000
  - 관리계정수: 2 → 5
  - API 상태: 미연동 → 연동
```

---

## 🔄 데이터 마이그레이션

### 기존 데이터 호환성
```typescript
// 기존 필드 (unitPrice: 표시용)
unitPrice: "1.2만원/명"  // UI에서만 사용

// 신규 필드 (실제 숫자 데이터)
pricePerStudent: 5000     // 원 단위 저장
baseStudents: 10          // 기본 학생 수
monthlyPrice: 100000      // 월 비용
```

### 마이그레이션 단계
1. **DB 스키마 업데이트**
   - `baseStudents` 컬럼 추가
   - `maxStudents` 컬럼 추가 (기존 max_students 기존 사용 여부 확인)
   - `monthlyPrice` 컬럼 추가 (기존 값 활용)

2. **기존 데이터 백필**
   - 구기관 데이터에 planceDefault_PLANS 값으로 초기화

3. **API 응답 검증**
   - GET 응답에 새 필드 포함 확인
   - 기존 필드와의 중복 제거

---

## 🚀 향후 개선사항

- [ ] 플랜 동적 추가 (현재 3개 하드코딩)
- [ ] 요금제 버전 관리 (과거 요금제 데이터 추적)
- [ ] 기관별 커스텀 요금제 지원
- [ ] 할인율 자동 계산 (월납 vs 연납)
- [ ] 요금 변경 이력 감사 로그

---

**파일 작성:** 2026-03-14  
**최종 수정:** 2026-03-14
