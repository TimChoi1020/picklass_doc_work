# 시스템 관리 (System Management)

## 📋 개요

Picklass 관리자 페이지의 시스템 관리 페이지입니다. 사용자 상태, 접근 코드 정책, 레벨 시스템 등 전역 설정과 정책을 관리합니다.

**파일 경로:**
- `src/app/(admin)/admin/system/page.tsx`

**데이터 출처:**
- `src/lib/constants.ts` → 다중 상수

**상태:** 업데이트 완료 (2026-03-14)

---

## 🎯 주요 기능

### 1. 사용자 상태 관리
- ✅ 사용자 상태코드 정의 및 관리
- ✅ 상태별 설명 및 운영 정책
- ✅ 상태 코드 → UI 표시명 매핑

### 2. 접근 코드 정책
- ✅ 접근 코드 상태 정의
- ✅ 코드 유효기간 설정 (1, 3, 6, 12개월)
- ✅ 기간별 일수 매핑

### 3. 레벨 시스템 관리
- ✅ CEFR(Common European Framework) 기반 18단계 레벨
- ✅ 레벨별 분류 (Basic, Intermediate, Advanced, Mastery)
- ✅ 각 단계별 설명

---

## 🏗️ IA 구조

```
시스템 관리 (/admin/system)
│
├── Section 1: 사용자 상태 관리
│   ├── 테이블 헤더: [상태명, 상태코드, 설명]
│   ├── 행 1: 활성 | active | 정상 활동 중인 사용자
│   ├── 행 2: 휴회 | inactive | 일시적으로 비활성 (재활성 가능)
│   ├── 행 3: 정지 | suspended | 관리자 조치로 인한 계정 정지
│   └── 행 4: 탈퇴 | withdrawn | 영구 탈퇴 (복원 불가)
│
├── Section 2: 접근 코드 상태 관리
│   ├── 테이블 헤더: [상태명, 상태코드, 설명]
│   ├── 행 1: 활성 | active | 사용 가능한 상태
│   ├── 행 2: 비활성 | inactive | 비활성 상태
│   ├── 행 3: 정지 | suspended | 일시적 정지 상태
│   └── 행 4: 탈퇴 | withdrawn | 계정 탈퇴 상태
│
├── Section 3: 접근 코드 유효기간 설정
│   ├── 테이블 헤더: [기간명, 코드, 일수, 설명]
│   ├── 행 1: 1개월 | 1month | 30 | 1개월 사용 기간
│   ├── 행 2: 3개월 | 3month | 90 | 3개월 사용 기간
│   ├── 행 3: 6개월 | 6month | 180 | 6개월 사용 기간
│   └── 행 4: 1년 | 1year | 365 | 1년 사용 기간
│
└── Section 4: 레벨 시스템 (18단계 CEFR)
    ├── 테이블 헤더: [레벨, CEFR, 카테고리]
    ├── Starter (A1-, A1, A1+): 레벨 1~3
    ├── Beginner (A2-, A2, A2+): 레벨 4~6
    ├── Intermediate (B1-, B1, B1+): 레벨 7~9
    ├── Upper-Intermediate (B2-, B2, B2+): 레벨 10~12
    ├── Advanced (C1-, C1, C1+): 레벨 13~15
    └── Proficient (C2-, C2, C2+): 레벨 16~18
```

---

## 📊 데이터 구조

### 상태 데이터 모델
```typescript
interface StatusRow {
  name: string;        // 표시명 ("활성", "휴회" 등)
  code: string;        // 저장 코드 ("active", "inactive" 등)
  description: string; // 설명
}
```

### 기간 데이터 모델
```typescript
interface DurationRow {
  name: string;        // 기간 이름 ("1개월", "3개월" 등)
  code: string;        // 기간 코드 ("1M", "3M" 등)
  days: number;        // 일수 (30, 90, 180, 365)
  description: string; // 설명
}
```

### 레벨 데이터 모델
```typescript
interface LevelRow {
  level: number;       // 레벨 번호 (1-18)
  cefrLevel: string;   // CEFR 레벨 ("A1", "A2", "B1", ...)
  category: string;    // 카테고리 ("Basic", "Intermediate", ...)
  description?: string; // 설명
}
```

### 상수 정의

#### USER_STATUSES
```typescript
export const USER_STATUSES: StatusRow[] = [
  {
    name: '활성',
    code: 'active',
    description: '정상 활동 중인 사용자',
  },
  {
    name: '휴회',
    code: 'inactive',
    description: '일시적으로 비활성 (재활성 가능)',
  },
  {
    name: '정지',
    code: 'suspended',
    description: '관리자 조치로 인한 계정 정지',
  },
  {
    name: '탈퇴',
    code: 'withdrawn',
    description: '영구 탈퇴 (복원 불가)',
  },
];
```

#### INSTITUTION_STATUSES
```typescript
export const INSTITUTION_STATUSES: StatusRow[] = [
  {
    name: '활성',
    code: 'active',
    description: '정상 운영 중인 기관',
  },
  {
    name: '휴회',
    code: 'inactive',
    description: '일시적 운영 중단 (재활성 가능)',
  },
  {
    name: '정지',
    code: 'suspended',
    description: '관리자 조치로 인한 정지',
  },
  {
    name: '탈퇴',
    code: 'withdrawn',
    description: '기관 완전 탈퇴',
  },
];
```

#### ACCESS_CODE_STATUSES
```typescript
export const ACCESS_CODE_STATUSES: StatusRow[] = [
  { name: '활성', code: 'active', description: '사용 가능한 상태' },
  { name: '비활성', code: 'inactive', description: '비활성 상태' },
  { name: '정지', code: 'suspended', description: '일시적 정지 상태' },
  { name: '탈퇴', code: 'withdrawn', description: '계정 탈퇴 상태' },
];
```

#### ACCESS_CODE_DURATIONS
```typescript
export const ACCESS_CODE_DURATIONS: DurationRow[] = [
  { name: '1개월', code: '1month', days: 30, description: '1개월 사용 기간' },
  { name: '3개월', code: '3month', days: 90, description: '3개월 사용 기간' },
  { name: '6개월', code: '6month', days: 180, description: '6개월 사용 기간' },
  { name: '1년', code: '1year', days: 365, description: '1년 사용 기간' },
];
```

#### LEVEL_SYSTEM
```typescript
export const LEVEL_SYSTEM: LevelRow[] = [
  { level: 1,  cefrLevel: 'A1-', category: 'Starter' },
  { level: 2,  cefrLevel: 'A1',  category: 'Starter' },
  { level: 3,  cefrLevel: 'A1+', category: 'Starter' },
  { level: 4,  cefrLevel: 'A2-', category: 'Beginner' },
  { level: 5,  cefrLevel: 'A2',  category: 'Beginner' },
  { level: 6,  cefrLevel: 'A2+', category: 'Beginner' },
  { level: 7,  cefrLevel: 'B1-', category: 'Intermediate' },
  { level: 8,  cefrLevel: 'B1',  category: 'Intermediate' },
  { level: 9,  cefrLevel: 'B1+', category: 'Intermediate' },
  { level: 10, cefrLevel: 'B2-', category: 'Upper-Intermediate' },
  { level: 11, cefrLevel: 'B2',  category: 'Upper-Intermediate' },
  { level: 12, cefrLevel: 'B2+', category: 'Upper-Intermediate' },
  { level: 13, cefrLevel: 'C1-', category: 'Advanced' },
  { level: 14, cefrLevel: 'C1',  category: 'Advanced' },
  { level: 15, cefrLevel: 'C1+', category: 'Advanced' },
  { level: 16, cefrLevel: 'C2-', category: 'Proficient' },
  { level: 17, cefrLevel: 'C2',  category: 'Proficient' },
  { level: 18, cefrLevel: 'C2+', category: 'Proficient' },
];
```

---

## ⚙️ 정책 변경사항

### 1. 사용자 상태 코드 표준화
```
상태명         상태코드    설명
─────────────────────────────────
활성          active      정상 활동
휴회          inactive    일시적 비활성 (재활성 가능)
정지          suspended   관리자 조치 정지
탈퇴          withdrawn   영구 탈퇴
```

**중요:** UI 표시("활성", "휴회")와 저장 코드("active", "inactive")는 다릅니다.

### 2. 기관 상태 정책
기관(Institution) 상태는 사용자(User) 상태와 다릅니다:
- **사용자:** 활성, 휴회, 정지, 탈퇴
- **기관:** 활성, 휴회, 정지, 탈퇴 (코드는 동일하나 의미는 맥락별 다름)

### 3. 접근 코드 유효기간 정책
```
기간명    코드      일수   용도
──────────────────────────────
1개월    1month   30    단기 평가용
3개월    3month   90    분기별 학습
6개월    6month   180   반년 구독
1년      1year    365   연간 계약
```

- 기간 선택 시 자동으로 만료일 계산 (발급일 + days)
- 특정 날짜 기간 설정도 가능 (향후)

### 4. 레벨 시스템 정책
```
카테고리              레벨    CEFR
──────────────────────────────────
Starter               1~3    A1-, A1, A1+
Beginner              4~6    A2-, A2, A2+
Intermediate          7~9    B1-, B1, B1+
Upper-Intermediate    10~12  B2-, B2, B2+
Advanced              13~15  C1-, C1, C1+
Proficient            16~18  C2-, C2, C2+
```

- 각 레벨별 요구 학습량 설정 가능 (향후)
- 진급 테스트 제도 (향후)

---

## 🔧 개발자 체크리스트

### 상수 통합 검증
- [ ] `src/lib/constants.ts`에서 5개 상수 import 확인
  - USER_STATUSES
  - INSTITUTION_STATUSES
  - ACCESS_CODE_STATUSES
  - ACCESS_CODE_DURATIONS
  - LEVEL_SYSTEM

### 페이지 렌더링 (system/page.tsx)
- [ ] Section 1: `USER_STATUSES.map()` 테이블 렌더링
- [ ] Section 2: `ACCESS_CODE_STATUSES.map()` 테이블 렌더링
- [ ] Section 3: `ACCESS_CODE_DURATIONS.map()` 테이블 렌더링 (days 컬럼 표시)
- [ ] Section 4: `LEVEL_SYSTEM.map()` 테이블 렌더링
  - CEFR 레벨별 색상 구분 (A1/A2 파란색, B1/B2 초록색, C1/C2 빨강색)

### 동적 Select 구현 (다른 페이지)
#### users/[id]/edit에서 사용자 상태 선택
```typescript
<select value={form.status} onChange={(e) => setForm({...form, status: e.target.value})}>
  {USER_STATUSES.map((status) => (
    <option key={status.code} value={status.code}>
      {status.name}
    </option>
  ))}
</select>
```

#### 접근 코드 생성 시 기간 선택
```typescript
<select value={form.duration} onChange={(e) => setForm({...form, duration: e.target.value})}>
  {ACCESS_CODE_DURATIONS.map((duration) => (
    <option key={duration.code} value={duration.code}>
      {duration.name}
    </option>
  ))}
</select>
```

### 날짜 계산 로직 (Backend)
```typescript
// 접근 코드 만료일 자동 계산
const durationDays = ACCESS_CODE_DURATIONS.find(d => d.code === durationCode)?.days;
const expiryDate = new Date(issuedDate);
expiryDate.setDate(expiryDate.getDate() + durationDays);
```

### API 응답 검증
- [ ] 사용자 조회: status 필드가 code 형식으로 반환 확인 (표시명 아님)
- [ ] 접근코드 조회: duration 필드가 "1M", "3M" 등 code 형식으로 반환 확인
- [ ] 기관 조회: status 필드가 code 형식으로 반환 확인

### 데이터 타입 검증
- [ ] 상태 코드는 문자열 (string)만 사용
- [ ] 기간 일수는 숫자 (number)만 사용
- [ ] 레벨은 1-18 범위 정수 (number)만 사용

---

## 📝 사용 예시

### 시스템 관리 페이지 접근
```
1. /admin/system 접근
2. 4개 섹션 정보 표시
   - 사용자 상태 정의
   - 접근코드 상태 정의
   - 기간별 일수 매핑
   - 레벨 시스템
3. 각 항목의 코드, 설명 확인 가능
```

### 사용자 상태 변경 플로우 (users/[id]/edit)
```
1. 사용자 편집 페이지 접근
2. 상태 드롭다운 클릭
3. USER_STATUSES 기반 옵션 표시
   - 활성
   - 휴회
   - 정지
   - 탈퇴
4. 상태 선택 (예: "정지")
5. 저장 시 code ("suspended")로 저장
```

### 접근코드 생성 플로우 (accesscode/generate)
```
1. 접근 코드 생성 페이지 접근
2. 기간 선택 (예: "3개월")
   → ACCESS_CODE_DURATIONS에서 "3M" 선택
   → 90일이 자동으로 계산
3. 만료일 = 발급일 + 90일
4. 접근코드 생성
```

---

## 🔄 마이그레이션 가이드

### 기존 상태 코드 호환성
```typescript
// 기존 코드 (문자열 직접 사용)
status: 'active'

// 신규 코드 (상수 사용으로 일관성 확보)
const statusCode = USER_STATUSES[0].code; // 'active'
```

### 기간 처리 마이그레이션
```typescript
// 기존: 기간을 문자열로 저장 ("1개월", "3개월" 등)
duration: '3개월'

// 신규: 코드와 일수로 저장
{
  duration: '3M',        // ACCESS_CODE_DURATIONS[1].code
  durationDays: 90,      // ACCESS_CODE_DURATIONS[1].days
  expiryDate: '2026-06-14' // 자동 계산
}
```

---

## 🚀 향후 개선사항

- [ ] 상태별 권한 정의 및 검증
- [ ] 상태 전환 규칙 추가 (예: 탈퇴 상태는 복구 불가)
- [ ] 상태 변경 감시 로그 기록
- [ ] 레벨별 콘텐츠 추천 시스템
- [ ] 접근코드 자동 갱신 정책
- [ ] 기간별 사용자 초대 메일 템플릿
- [ ] 대량 상태 변경 배치 작업

---

**파일 작성:** 2026-03-14  
**최종 수정:** 2026-03-14
