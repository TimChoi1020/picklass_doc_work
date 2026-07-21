# 플랜 관리 (Plan Management)

## 📋 개요

Picklass 관리자 페이지의 플랜 관리 시스템입니다. 기관에게 제공되는 세 가지 가격제 (Starter, Pro, Enterprise)를 정의하고 관리합니다.

**파일 경로:**
- `src/app/(admin)/admin/plans/page.tsx`

**데이터 출처:**
- `src/lib/constants.ts` → `DEFAULT_PLANS` 상수

**상태:** 신규 페이지 (2026-03-14)

---

## 🎯 주요 기능

### 1. 플랜 목록 조회
- ✅ 3가지 플랜 조회 (Starter, Pro, Enterprise)
- ✅ 플랜별 상세 정보 표시
- ✅ 가격 및 제한사항 한눈에 확인

### 2. 플랜 기본 정보
- ✅ 플랜명
- ✅ 월 가격 / 연납 할인율
- ✅ 기본 제공 학생 수 / 추가 학생 단가
- ✅ 최대 학생 수 제한
- ✅ 최대 관리 계정 수
- ✅ API 연동 포함 여부

### 3. 정책 및 제한사항
- ✅ API 연동 비용 (기본 포함 여부)
- ✅ 활성화 여부 (enable 플래그)
- ✅ 향후 커스텀 플랜 추가 가능성

---

## 🏗️ IA 구조

```
플랜 관리 (/admin/plans)
├── 페이지 제목: "요금제 관리"
│
├── Starter 플랜 카드
│   ├── 플랜명: "Starter"
│   ├── 월 가격: 100,000원
│   ├── 연납 할인: 5%
│   ├── 기본 학생 수: 10명
│   ├── 추가 학생 단가: 5,000원
│   ├── 최대 학생 수: 50명
│   ├── 최대 관리 계정: 2개
│   ├── API 연동: 미연동
│   ├── API 비용: 무료/포함
│   └── 상태: 활성
│
├── Pro 플랜 카드
│   ├── 플랜명: "Pro"
│   ├── 월 가격: 1,000,000원
│   ├── 연납 할인: 10%
│   ├── 기본 학생 수: 50명
│   ├── 추가 학생 단가: 10,000원
│   ├── 최대 학생 수: 500명
│   ├── 최대 관리 계정: 5개
│   ├── API 연동: 연동
│   ├── API 비용: 30,000원/월
│   └── 상태: 활성
│
└── Enterprise 플랜 카드
    ├── 플랜명: "Enterprise"
    ├── 월 가격: 협의
    ├── 연납 할인: 0%
    ├── 기본 학생 수: 100명
    ├── 추가 학생 단가: 협의
    ├── 최대 학생 수: 무제한
    ├── 최대 관리 계정: 10개
    ├── API 연동: 연동
    ├── API 비용: 협의
    └── 상태: 활성
```

---

## 📊 데이터 구조

### 플랜 데이터 모델
```typescript
interface Plan {
  name: string;              // 플랜명 (Starter, Pro, Enterprise)
  monthlyPrice: number;      // 월 가격 (원)
  annualDiscount: number | string; // 연납 할인 (5, 10, 0)
  baseStudents: number;      // 기본 제공 학생 수 (명)
  pricePerStudent: number;   // 추가 학생 단가 (원)
  maxStudents: number;       // 최대 학생 수 (명) 또는 0 (무제한)
  maxAdminAccounts: number;  // 최대 관리 계정 수
  apiIntegration: string;    // API 연동 상태 ('Y', 'N')
  apiCost?: number;          // API 연동 비용 (원/월)
  enabled: boolean;          // 활성화 여부
  unitPrice?: string;        // UI 표시용 (예: "1.2만원/명")
}
```

### DEFAULT_PLANS 상수
```typescript
export const DEFAULT_PLANS: Plan[] = [
  {
    name: 'Starter',
    monthlyPrice: 100000,
    annualDiscount: '5%',
    baseStudents: 10,
    pricePerStudent: 5000,
    maxStudents: 50,
    maxAdminAccounts: 2,
    apiIntegration: 'N',
    apiCost: 0,
    enabled: true,
    unitPrice: '5,000원/명',
  },
  {
    name: 'Pro',
    monthlyPrice: 1000000,
    annualDiscount: '10%',
    baseStudents: 50,
    pricePerStudent: 10000,
    maxStudents: 500,
    maxAdminAccounts: 5,
    apiIntegration: 'Y',
    apiCost: 30000,
    enabled: true,
    unitPrice: '10,000원/명',
  },
  {
    name: 'Enterprise',
    monthlyPrice: 0,        // 협의
    annualDiscount: '0%',
    baseStudents: 100,
    pricePerStudent: 0,     // 협의
    maxStudents: 999999,    // 무제한으로 간주
    maxAdminAccounts: 10,
    apiIntegration: 'Y',
    apiCost: 0,             // 협의
    enabled: true,
    unitPrice: '협의',
  },
];
```

---

## ⚙️ 정책 변경사항

### 1. 플랜별 가격 정책
| 항목 | Starter | Pro | Enterprise |
|-----|---------|-----|------------|
| **월 가격** | 100,000원 | 1,000,000원 | 협의 |
| **연납 할인** | 5% | 10% | 협의 |
| **기본학생** | 10명 | 50명 | 100명 |
| **추가학생** | 5,000원/명 | 10,000원/명 | 협의 |
| **학생제한** | 50명 | 500명 | 무제한 |
| **관리계정** | 2개 | 5개 | 10개 |
| **API연동** | 미포함 | 포함 (월 3만원) | 포함 (협의) |

### 2. 플랜 선택 시 동작
기관 등록/수정 화면에서 플랜을 선택하면:
```
1. Plan 드롭다운에서 플랜 선택
2. DEFAULT_PLANS에서 해당 플랜 객체 검색
3. 다음 필드 자동 초기화:
   - monthlyPrice
   - annualDiscount
   - baseStudents
   - pricePerStudent
   - maxAdminAccounts
   - apiIntegration
4. 사용자가 필요시 각 필드 수정 가능
```

### 3. 협의 가격 처리
```typescript
// Enterprise 플랜 가격 표시
monthlyPrice === 0 ? "협의" : `${monthlyPrice.toLocaleString()}원`
pricePerStudent === 0 ? "협의" : `${pricePerStudent.toLocaleString()}원`
apiCost === 0 ? "협의" : `${apiCost.toLocaleString()}원`
```

### 4. 최대학생 수 정책
- **Starter:** 정확히 50명 제한
- **Pro:** 정확히 500명 제한
- **Enterprise:** 무제한 (999999로 내부 표현)

---

## 🔧 개발자 체크리스트

### 플랜 관리 페이지 (/admin/plans)
- [ ] `src/lib/constants.ts`에서 `DEFAULT_PLANS` import 확인
- [ ] 플랜별 카드 렌더링: `DEFAULT_PLANS.map(plan =>)`
- [ ] 2행 정보 레이아웃 구현
  ```
  Row 1: 플랜명 | 월가격
  Row 2: 기본학생/추가학생단가 | 최대학생/관리계정
  (추가 행: API연동 상태, 연납할인)
  ```

### 플랜 선택 로직 (institute/edit, institute/register)
- [ ] 플랜 변경 handler에서 DEFAULT_PLANS 검색
  ```typescript
  const planOption = DEFAULT_PLANS.find((p) => p.name === selectedPlan);
  ```
- [ ] 6개 필드 자동 초기화:
  ```typescript
  monthlyPrice: planOption?.monthlyPrice ?? 0
  annualDiscount: planOption?.annualDiscount ?? ''
  baseStudents: planOption?.baseStudents ?? 0
  pricePerStudent: planOption?.pricePerStudent ?? 0
  maxAdminAccounts: planOption?.maxAdminAccounts ?? 0
  apiIntegration: planOption?.apiIntegration ?? ''
  ```

### API 검증 (Backend)
- [ ] `GET /api/plans` - 플랜 목록 조회 (현재 구현 불필요, 상수사용)
- [ ] 기관 API 응답에 plan 필드 포함 검증
- [ ] monthlyPrice, pricePerStudent 원 단위 저장 검증

### 데이터 일관성 검증
- [ ] 기관 기본 정보 조회 시 플랜명으로 DEFAULT_PLANS 재조회 가능
  ```typescript
  const plan = DEFAULT_PLANS.find(p => p.name === institution.plan);
  ```
- [ ] 플랜 정보 변경 시 기존 기관 데이터 영향 계획 수립

### 향후 확장성
- [ ] 플랜을 상수에서 DB로 이전 시 마이그레이션 계획
- [ ] API 시작점: `GET /api/plans` 엔드포인트 설계
- [ ] 플랜 로직 분리를 위한 모듈 구조 설계

---

## 📝 사용 예시

### 플랜 관리 페이지 접근
```
1. /admin/plans 접근
2. Starter, Pro, Enterprise 플랜 정보 표시
3. 각 플랜의 주요 정보 확인
4. (향후) 플랜 수정/추가 기능
```

### Starter → Pro 플랜 변경 시나리오
```
기존 기관: Starter 플랜 사용 중
처리 단계:
1. 기관 수정 페이지에서 플랜을 Pro로 변경
2. 자동으로 다음 값 업데이트:
   - 월가격: 100,000 → 1,000,000원
   - 연납할인: 5% → 10%
   - 기본학생: 10 → 50명
   - 추가학생단가: 5,000 → 10,000원
   - 관리계정: 2 → 5개
   - API: 미연동 → 연동
3. 사용자가 필요시 일부 값 수정 가능
4. 저장 시 새로운 플랜 정보로 업데이트
```

---

## 🔄 마이그레이션 가이드

### 상수 → DB 마이그레이션 (향후)
```typescript
// 현재 (상수 기반)
const DEFAULT_PLANS = [...]

// 마이그레이션 후 (DB 기반)
// Schema: plans 테이블
// Fields: id, name, monthlyPrice, annualDiscount, ...
// API: GET /api/plans → DEFAULT_PLANS 대체
```

### 플랜 버전 관리
```typescript
// 향후: 플랜 버전 추적으로 과거 기관 요금 계산 가능
interface PlanVersion {
  id: uuid;
  name: string;
  version: number;
  effectiveDate: Date;
  ...planData
}
```

---

## 🚀 향후 개선사항

- [ ] 플랜 동적 관리 UI (CRUD)
- [ ] 플랜 버전 관리 (변경 이력 추적)
- [ ] 기관별 커스텀 플랜 지원
- [ ] 플랜별 기능 제한사항 세부 정의
- [ ] 플랜 변경 이력 감시 로그
- [ ] A/B 가격 테스트 플랜 지원
- [ ] 플랜 추천 알고리즘 추가

---

**파일 작성:** 2026-03-14  
**최종 수정:** 2026-03-14
