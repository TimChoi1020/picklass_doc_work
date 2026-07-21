# AI 모듈 관리 (AI Modules Management)

## 📋 개요

Picklass 관리자 페이지의 AI 모듈 관리 시스템입니다. AI 기반 학습 기능(AI 생성, TTS, 전략적 읽기 등) 모듈의 상태, 비용, 사용 한도를 관리합니다.

**파일 경로:**
- 페이지: `src/app/(admin)/admin/ai-modules/page.tsx` (향후 생성)
- 데이터: `src/lib/constants.ts` → (신규 상수 추가 필요)

**상태:** 계획 단계 (2026-03-14)

---

## 🎯 주요 기능 (계획)

### 1. AI 모듈 목록 관리
- ✅ 활성화된 AI 모듈 조회
- ✅ 모듈별 상태 (활성, 비활성)
- ✅ 모듈별 비용 정책

### 2. 모듈별 설정
- ✅ 사용자별 월간 사용 한도
- ✅ 기관별 AI 기능 모듈 포함 여부
- ✅ 모듈별 API 호출 비용

### 3. 사용량 추적
- ✅ 모듈별 요청 통계
- ✅ 사용자별 사용량 현황
- ✅ 월별 비용 집계

---

## 🏗️ IA 구조

```
AI 모듈 관리 (/admin/ai-modules)
│
├── Section 1: AI 모듈 목록
│   ├── 테이블 헤더: [모듈명, 상태, 포함 플랜, 월간 한도, 기본 비용]
│   ├── 행 1: AI 생성 | 활성 | Pro, Enterprise | 무제한 | 포함
│   ├── 행 2: TTS | 활성 | Pro, Enterprise | 10,000자 | 포함
│   ├── 행 3: 전략적 읽기 | 활성 | Pro, Enterprise | 무제한 | 포함
│   ├── 행 4: 음성인식 (STT) | 미활성 | - | - | 개발중
│   └── 행 5: 단어 분석 검색 | 활성 | 모두 포함 | 무제한 | 포함
│
├── Section 2: 모듈별 설정
│   ├── AI 생성 모듈
│   │   ├── 설명: 특정 주제에 대한 영문 지문 자동 생성
│   │   ├── API: OpenAI GPT-4 (또는 동등 기능)
│   │   ├── 월간 한도: 사용자 무제한 (Pro/Enterprise)
│   │   ├── 기본 비용: 포함 (월 X조원 상쇄)
│   │   └── 초과 비용: N원/요청
│   │
│   ├── TTS (Text-to-Speech) 모듈
│   │   ├── 설명: 학습 지문 음성 변환
│   │   ├── API: Azure Speech Services 또는 제3자 API
│   │   ├── 월간 한도: 10,000자/월 (Pro/Enterprise)
│   │   ├── 기본 비용: 포함
│   │   └── 초과 비용: N원/1000자
│   │
│   ├── 전략적 읽기 (Strategic Reading) 모듈
│   │   ├── 설명: 지문 분석, 어려운 단어 추출, 핵심 내용 요약
│   │   ├── API: 내부 NLP 엔진
│   │   ├── 월간 한도: 사용자 무제한
│   │   ├── 기본 비용: 포함
│   │   └── 초과 비용: 없음
│   │
│   ├── STT (Speech-to-Text) 모듈
│   │   ├── 상태: 미활성 (개발중)
│   │   ├── 설명: 사용자 음성 입력 → 텍스트 변환
│   │   ├── 예상 출시: 2026-06-30
│   │   └── 예상 비용: 협의 중
│   │
│   └── 단어 분석 검색 (Word Analysis Search) 모듈
│       ├── 설명: 단어 해석, 관련 표현 검색
│       ├── API: 내부 데이터베이스
│       ├── 월간 한도: 무제한
│       └── 기본 비용: 포함 (모든 플랜)
│
└── Section 3: 사용량 통계 (향후)
    ├── 일별 모듈별 요청 건수
    ├── 월별 모듈별 비용 집계
    ├── 사용자별 AI 기능 사용률
    └── 초과 사용료 계산
```

---

## 📊 데이터 구조

### AI 모듈 데이터 모델
```typescript
interface AIModule {
  id: string;              // 모듈 ID (ai-generate, tts, etc.)
  name: string;            // 한글명 ("AI 생성", "TTS" 등)
  description: string;     // 설명
  status: 'active' | 'inactive' | 'beta'; // 상태
  
  // 가격 정책
  includedPlans: string[]; // 포함된 플랜 ['Pro', 'Enterprise']
  baseCost: number;        // 기본 비용 (원/월, 플랜에 포함 시 0)
  overageCost?: number;    // 초과 비용 (원/단위)
  
  // 사용 한도
  monthlyLimit?: number;   // 월간 한도 (null=무제한)
  limitUnit?: string;      // 한도 단위 ("회", "자", "건")
  
  // API 정보
  apiProvider?: string;    // API 제공사 ("OpenAI", "Azure", "Internal")
  apiEndpoint?: string;    // API 엔드포인트
  
  // 활성화 시간
  launchDate: Date;        // 출시일
  deprecationDate?: Date;  // 폐기 예정일
}
```

### 상수 정의 (신규 - 추가 필요)

#### AI_MODULES (추가 필요)
```typescript
export const AI_MODULES: AIModule[] = [
  {
    id: 'ai-generate',
    name: 'AI 생성',
    description: '특정 주제에 대한 영문 지문 자동 생성',
    status: 'active',
    includedPlans: ['Pro', 'Enterprise'],
    baseCost: 0, // 월정액에 포함
    overageCost: 1000, // 초과 시 1000원/요청
    monthlyLimit: null, // 무제한
    limitUnit: '요청',
    apiProvider: 'OpenAI',
    apiEndpoint: 'https://api.openai.com/v1/chat/completions',
    launchDate: new Date('2024-01-15'),
  },
  {
    id: 'tts',
    name: 'TTS',
    description: '학습 지문 음성 변환',
    status: 'active',
    includedPlans: ['Pro', 'Enterprise'],
    baseCost: 0, // 월정액에 포함
    overageCost: 50, // 초과 시 50원/1000자
    monthlyLimit: 10000, // 월 10,000자
    limitUnit: '자',
    apiProvider: 'Azure',
    apiEndpoint: 'https://<region>.tts.speech.microsoft.com/',
    launchDate: new Date('2024-03-01'),
  },
  {
    id: 'strategic-reading',
    name: '전략적 읽기',
    description: '지문 분석, 어려운 단어 추출, 핵심 내용 요약',
    status: 'active',
    includedPlans: ['Pro', 'Enterprise'],
    baseCost: 0,
    monthlyLimit: null, // 무제한
    limitUnit: '활용',
    apiProvider: 'Internal NLP',
    launchDate: new Date('2024-06-01'),
  },
  {
    id: 'stt',
    name: 'STT',
    description: '사용자 음성 입력 → 텍스트 변환',
    status: 'inactive', // 미활성 - 개발중
    includedPlans: [],
    baseCost: 0,
    launchDate: new Date('2026-06-30'), // 예상 출시
  },
  {
    id: 'word-analysis',
    name: '단어 분석 검색',
    description: '단어 해석, 관련 표현 검색',
    status: 'active',
    includedPlans: ['Starter', 'Pro', 'Enterprise'], // 모든 플랜
    baseCost: 0,
    monthlyLimit: null,
    apiProvider: 'Internal Database',
    launchDate: new Date('2024-01-01'),
  },
];
```

---

## ⚙️ 정책 변경사항

### 1. AI 모듈 포함 정책
```
모듈명          Starter    Pro        Enterprise
───────────────────────────────────────────
AI 생성          미포함     포함        포함
TTS              미포함     포함        포함
전략적 읽기      미포함     포함        포함
단어분석         포함       포함        포함
STT              미포함     향후포함    향후포함
```

### 2. 사용 한도 정책
```
모듈명                한도           초과 처리
─────────────────────────────────────────────
AI 생성               월 무제한      1000원/요청
TTS                   월 10,000자    50원/1000자
전략적 읽기           무제한         불가
단어분석              무제한         불가
STT (향후)            협의중         협의중
```

### 3. 비용 정책
- **기본 포함:** AI 모듈 기본 비용은 플랜 월정액에 포함
- **초과 사용 요금:** 한도 초과 시 추가 요금 청구
- **사용자별 한도:** 기관 단위 한도 / 사용자 단위 한도 이원화 가능

### 4. API 비용 구조 (내부 참고)
```
OpenAI (AI 생성):
  - 초과 시 API 호출 실제 비용 + 마진율 30%

Azure TTS:
  - 월 10,000자 포함 (Pro/Enterprise)
  - 초과: 실제 API 비용 적용

내부 서비스:
  - 전략적 읽기, 단어분석: 추가 비용 없음
```

---

## 🔧 개발자 체크리스트

### 상수 추가 및 통합
- [ ] `src/lib/constants.ts`에 `AI_MODULES` 상수 추가
- [ ] AIModule 인터페이스 정의
- [ ] 5개 모듈 데이터 입력
- [ ] `src/lib/constants.ts` 내보내기

### AI 모듈 관리 페이지 생성
- [ ] `src/app/(admin)/admin/ai-modules/page.tsx` 생성
- [ ] AI_MODULES 상수 import
- [ ] Section 1: 모듈 목록 테이블 렌더링
  ```typescript
  AI_MODULES.map(module => (
    <tr key={module.id}>
      <td>{module.name}</td>
      <td>{module.status === 'active' ? '활성' : '비활성'}</td>
      <td>{module.includedPlans.join(', ')}</td>
      <td>{module.monthlyLimit ? `${module.monthlyLimit}${module.limitUnit}` : '무제한'}</td>
      <td>{module.baseCost === 0 ? '포함' : `${module.baseCost.toLocaleString()}원`}</td>
    </tr>
  ))
  ```

### API 통합
- [ ] `GET /api/ai-modules` - 모듈 목록 조회
- [ ] `GET /api/ai-modules/:id` - 모듈 상세 조회
- [ ] `POST /api/ai-modules/:id/usage` - 사용량 기록
- [ ] `GET /api/ai-modules/:id/stats` - 모듈별 통계

### 사용량 추적
- [ ] 모듈별 API 호출 시 `usage` 테이블에 기록
  ```typescript
  interface Usage {
    id: uuid;
    moduleId: string;           // ai-generate, tts 등
    userId: string;
    institutionId: string;
    quantity: number;           // 사용량 (요청 수, 글자 수 등)
    cost: number;               // 비용 (초과 시에만)
    createdAt: Date;
  }
  ```

### 사용자 UI 통합
- [ ] 기관 상세 페이지: 포함된 AI 모듈 표시
- [ ] 사용자 프로필: 월간 사용량 현황 표시
- [ ] 접근 코드 생성: AI 기능 포함 여부 선택 옵션

### 데이터 유효성 검증
- [ ] 플랜에 포함된 모듈만 사용 가능
- [ ] 월 한도 초과 실시간 체크
- [ ] 초과 사용료 자동 계산

---

## 📝 사용 예시

### AI 모듈 관리 페이지 접근
```
1. /admin/ai-modules 접근
2. 5개 모듈 현황 표시
   - AI 생성: 활성 (Pro, Enterprise)
   - TTS: 활성 (Pro, Enterprise)
   - 전략적 읽기: 활성 (Pro, Enterprise)
   - STT: 비활성 (개발중)
   - 단어분석: 활성 (모든 플랜)
3. 모듈별 설정 확인 및 수정 (향후)
```

### Pro 플랜 기관의 AI 기능 사용
```
기관: K 학원 (Pro 플랜)
포함 기능:
- AI 생성 (무제한)
- TTS (월 10,000자)
- 전략적 읽기 (무제한)
- 단어분석 (무제한)

미포함 기능:
- STT (향후 가능)
```

### 모듈 사용량 기록 (Backend)
```
예시: TTS 1000자 사용
→ Usage 테이블에 기록
{
  moduleId: 'tts',
  userId: 'user-123',
  institutionId: 'inst-456',
  quantity: 1000,      // 1000자
  cost: 0,             // 월 한도 내이므로 무료
  createdAt: 2026-03-14
}
```

---

## 🔄 마이그레이션 가이드

### 기존 AI 기능 데이터 수집
현재 코드에서 사용 중인 AI 기능:
- `src/components/AIGenerateModal.tsx` - AI 생성
- `src/hooks/useBatchTTS.ts` - TTS
- `src/hooks/useStrategicReading.ts` - 전략적 읽기

이들 기능의 API 엔드포인트, 비용 정보를 `AI_MODULES` 상수에 정의

### API 통합 마이그레이션
```typescript
// 기존: API 호출 시 모듈 정보 없음
await callAIAPI('/api/ai/generate', {...})

// 신규: 모듈ID와 사용량 함께 기록
await callAIAPI('/api/ai/generate', {...}, {
  moduleId: 'ai-generate',
  quantity: 1,
  recordUsage: true,
})
```

---

## 🚀 향후 개선사항

- [ ] **AI 모듈 동적 추가/수정 UI**
  - 관리자: 모듈 추가, 가격 조정 가능
  
- [ ] **사용량 기반 자동 청구**
  - 월말에 초과 사용료 집계 및 청구
  
- [ ] **모듈별 사용량 상세 보고서**
  - 기관별, 사용자별, 일별 분석
  
- [ ] **AI 기능 A/B 테스트**
  - 특정 기관에만 신기능 선제 제공
  
- [ ] **API 요금 최적화**
  - 배치 처리, 캐싱으로 API 호출 최소화
  
- [ ] **사용자별 AI 기능 제한**
  - 플랜 외 기능 접근 차단
  
- [ ] **STT 모듈 활성화** (2026-06-30)
  - 음성 인식 기능 추가
  
- [ ] **커스텀 모듈 지원**
  - 기관별 특화 AI 기능 제공

---

**파일 작성:** 2026-03-14  
**최종 수정:** 2026-03-14
