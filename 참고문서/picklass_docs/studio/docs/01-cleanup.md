# Phase 1: 프로젝트 정리 (Cleanup)

> 복제된 ClassSnap 프로젝트에서 불필요한 모듈을 제거하고, 인증을 Mock으로 대체한다.
> `/app/class/`가 유일한 학습 모듈이며, 이 모듈의 모든 의존성은 반드시 유지한다.

---

## 핵심 원칙

- **유지**: `/app/class/` 및 이 모듈이 의존하는 모든 컴포넌트, 훅, API, 라이브러리
- **삭제**: `/app/app/` 전체 (대시보드, text-management, user-settings, strategic-reading, fluency-reading)
- **삭제**: `/app/auth/` 전체 (인증은 추후 별도 프로젝트에서 제공)
- **삭제**: 결제(Paddle) 관련 전체

---

## 1. 제거할 디렉토리

### /app/app/ 전체 삭제
```
src/app/app/                           # 앱 라우트 전체 삭제
  ├── page.tsx                         # 대시보드
  ├── layout.tsx                       # 앱 래퍼
  ├── text-management/page.tsx         # 텍스트 관리 (class/page.tsx가 대체)
  ├── user-settings/page.tsx           # 사용자 설정
  ├── strategic-reading/               # 전략적 읽기 (앱 버전)
  │   ├── page.tsx
  │   └── [id]/page.tsx
  └── fluency-reading/                 # 유창성 읽기 (앱 버전)
      ├── page.tsx
      └── [id]/page.tsx
```

### /app/auth/ 전체 삭제
```
src/app/auth/                          # 인증 페이지 전체
  ├── login/page.tsx
  ├── register/page.tsx
  ├── 2fa/page.tsx
  ├── forgot-password/page.tsx
  ├── reset-password/page.tsx
  ├── verify-email/page.tsx
  ├── test/page.tsx
  └── layout.tsx

src/app/api/auth/                      # OAuth callback
  └── callback/route.ts
```

### /app/app/ 전용 컴포넌트
```
src/components/strategic-reading/      # 앱 전략적 읽기 전용 (10개) → 삭제
  ├── AIFeedback.tsx
  ├── ClarificationStep.tsx
  ├── GenerationOverlay.tsx
  ├── PredictionStep.tsx
  ├── QARStep.tsx
  ├── ScanningStep.tsx
  ├── StepCompletionModal.tsx
  ├── StepNavigator.tsx
  ├── SummaryStep.tsx
  └── TextDisplayPanel.tsx
```

### 테스트/백업 파일
```
src/app/test-login-bg/                 # 테스트 디렉토리
middleware.ts.bak                      # 백업 미들웨어
```

---

## 2. 제거할 개별 파일

### 인증 관련
| 파일 | 이유 |
|------|------|
| `src/components/MFASetup.tsx` | 2FA 인증 |
| `src/components/MFAVerification.tsx` | 2FA 인증 |
| `src/components/SSOButtons.tsx` | SSO 로그인 |
| `src/components/AuthAwareButtons.tsx` | 인증 상태 의존 버튼 |

### 결제 관련
| 파일 | 이유 |
|------|------|
| `src/components/HomePricing.tsx` | 결제 UI |
| `src/lib/pricing.ts` | 가격 정책 (PricingService) |

### /app/app/ 전용
| 파일 | 이유 |
|------|------|
| `src/components/AppLayout.tsx` | /app/app/layout.tsx 전용 (class/는 ClassLayout 사용) |
| `src/hooks/useStrategicReadingV2.ts` | /app/app/strategic-reading 전용 (class/에서 미사용) |
| `src/hooks/useTextFilter.ts` | /app/app/text-management 전용 (class/에서 미사용) |
| `src/hooks/useTextFilterWithPagination.ts` | /app/app/text-management 전용 (class/에서 미사용) |

### 미사용 컴포넌트
| 파일 | 이유 |
|------|------|
| `src/components/Confetti.tsx` | 프로젝트 전체에서 미사용 |
| `src/components/TTSCacheManager.tsx` | 프로젝트 전체에서 미사용 |

---

## 3. 반드시 유지해야 할 파일 (class/ 의존성)

> 아래 파일들은 class/ 라우트가 직접 또는 간접적으로 의존한다. **절대 삭제하면 안 된다.**

### 페이지 & 레이아웃
```
src/app/class/page.tsx                          # 교실 홈 (텍스트 라이브러리)
src/app/class/layout.tsx                        # ClassLayout + GlobalProvider
src/app/class/lesson-setup/[passageId]/page.tsx # 레슨 설정
src/app/class/lesson/[passageId]/page.tsx       # 라이브 레슨
src/app/class/lesson/[passageId]/layout.tsx     # SubscriptionGuard
```

### 레이아웃 컴포넌트
```
src/components/ClassLayout.tsx                  # class/layout.tsx에서 사용
src/components/SubscriptionGuard.tsx            # class/lesson/layout.tsx에서 사용
```

### 레슨 컴포넌트 (src/components/lesson/ — 18개 전체)
```
ClarificationStep.tsx, FluencyStep.tsx, MeaningGuessingStep.tsx,
PredictionStep.tsx, QARStep.tsx, ScanningStep.tsx,
SkimmingStep.tsx, SummarizingStep.tsx,
WordDeckStep.tsx, WordDeckLearning.tsx,
WordWebStep.tsx, WordWebLearning.tsx, WordWebDesign.tsx,
FeedbackPanel.tsx, WordPickerModal.tsx, WordPickerButton.tsx,
ScanningSettingsModal.tsx, WPMSettingsModal.tsx
```

### 텍스트 관리 컴포넌트 (class/page.tsx에서 사용)
```
src/components/CreatePassageModal.tsx           # 지문 생성 (내부에 ManualInputModal, AIGenerateModal 포함)
src/components/ManualInputModal.tsx             # CreatePassageModal 내부에서 사용
src/components/AIGenerateModal.tsx              # CreatePassageModal 내부에서 사용
src/components/PassageDetailModal.tsx           # 지문 상세 보기
src/components/PreviewPassageModal.tsx          # 지문 미리보기
src/components/TwinPassagesModal.tsx            # 쌍둥이 지문 관리
src/components/PassageInfoBadges.tsx            # 지문 정보 뱃지
src/components/PassageTypeBadge.tsx             # 지문 유형 뱃지
```

### 공용 컴포넌트
```
src/components/FontSizeSelector.tsx             # 거의 모든 lesson 컴포넌트에서 사용
src/components/Cookies.tsx                      # 루트 레이아웃(layout.tsx)에서 사용
src/components/LegalDocument.tsx                # /legal 라우트
src/components/LegalDocuments.tsx               # /legal 라우트
```

### 훅 (class/에서 사용 확인됨)
| 훅 | 사용 위치 |
|----|-----------|
| `useTexts.ts` (useTexts, useText, useCreateText, useTwinTexts) | class/page.tsx, lesson-setup, lesson |
| `useStrategicReading.ts` (useStrategicReadingData, useGenerateStrategicReading) | class/lesson/page.tsx |
| `useBatchTTS.ts` | FluencyStep |
| `useAzureSpeech.ts` | WordDeckLearning |
| `useTextAnalysis.ts` | WordDeckStep, MeaningGuessingStep |
| `useUserProfile.ts` | ClassLayout, SubscriptionGuard |
| `useAsyncTask.ts` | useStrategicReading 내부에서 사용 가능 (안전상 유지) |

### API Routes (class/에서 호출 확인된 20개)
```
# 텍스트 CRUD
src/app/api/texts/route.ts                     # GET, POST
src/app/api/texts/[id]/route.ts                # GET, PATCH, DELETE
src/app/api/texts/twins/[id]/route.ts          # GET

# 텍스트 생성/분석
src/app/api/generate-text/route.ts             # POST (class/page.tsx, CreatePassageModal)
src/app/api/extract-text/route.ts              # POST (ManualInputModal)
src/app/api/text-analysis/route.ts             # POST (useTextAnalysis)

# 전략적 읽기
src/app/api/strategic-reading/route.ts         # POST/GET (useStrategicReading)

# 피드백 (각 lesson step)
src/app/api/clarification-feedback/route.ts    # ClarificationStep
src/app/api/prediction-feedback/route.ts       # PredictionStep
src/app/api/qar-feedback/route.ts              # QARStep
src/app/api/summary-feedback/route.ts          # SummarizingStep
src/app/api/skimming-feedback/route.ts         # SkimmingStep
src/app/api/meaning-guessing-feedback/route.ts # MeaningGuessingStep

# TTS
src/app/api/azure-speech/route.ts              # FluencyStep(useBatchTTS), WordDeckLearning(useAzureSpeech)
src/app/api/tts-cache/cleanup/route.ts
src/app/api/tts-cache/stats/route.ts

# 어휘
src/app/api/word-definitions/route.ts          # WordDeckLearning, WordWebLearning
src/app/api/word-suggestion/route.ts           # WordWebLearning
src/app/api/word-relationship/route.ts         # WordWebLearning

# 공통
src/app/api/async-tasks/route.ts               # 비동기 작업 관리
src/app/api/user-profile/route.ts              # ClassLayout, SubscriptionGuard
```

### 삭제 가능한 API Route
```
src/app/api/texts/bulk-delete/route.ts         # class/에서 미사용 (text-management 전용)
```

### 라이브러리 (유지)
```
src/lib/supabase/client.ts                     # 클라이언트 Supabase
src/lib/supabase/server.ts                     # 서버 Supabase
src/lib/supabase/serverAdminClient.ts          # Admin Supabase
src/lib/supabase/middleware.ts                 # 미들웨어 (스텁 처리)
src/lib/supabase/unified.ts                    # 통합 클라이언트 (auth 메서드 제거)
src/lib/context/GlobalContext.tsx               # 전역 상태 (Mock user)
src/lib/context/ThemeContext.tsx                # 테마
src/lib/async-task/task-manager.ts             # 작업 관리
src/lib/realtime/global-subscription.ts        # Realtime
src/lib/utils/textAnalysis.ts                  # 단어 분석
src/lib/tts-cache.ts                           # TTS 캐시
src/lib/react-query.ts                         # React Query
src/lib/constants.ts                           # 브랜드 상수
src/lib/utils.ts                               # 유틸리티
src/lib/types.ts                               # 타입 정의
src/lib/types/database.types.ts                # DB 타입
src/lib/types/async-task.ts                    # 작업 타입
```

---

## 4. 제거할 Dependencies (package.json)

```json
"@paddle/paddle-js": "^1.3.3",              // 결제 (클라이언트)
"@paddle/paddle-node-sdk": "^2.3.2",        // 결제 (서버)
"@supabase/auth-helpers-nextjs": "^0.10.0",  // 레거시 auth helper
"@supabase/auth-js": "^2.87.1",             // 직접 auth-js (중복)
```

### 유지할 Dependencies (class/ 의존)
- `@supabase/ssr`, `@supabase/supabase-js` — 데이터 접근
- `@google/genai` — Gemini AI (전략적 읽기, 텍스트 생성, 피드백)
- `microsoft-cognitiveservices-speech-sdk` — Azure TTS (FluencyStep, WordDeckLearning)
- `mammoth` — DOCX 파싱 (ManualInputModal → CreatePassageModal)
- `@tanstack/react-query` — 데이터 페칭
- `wink-lemmatizer` — 단어 레마타이징 (WordDeckStep, WordWebStep)
- `react-force-graph-2d` — 단어 관계망 시각화 (WordWebStep)
- 모든 Radix/shadcn/ui, framer-motion, recharts 등

---

## 5. 수정할 파일 (Auth → Mock User)

### 5.1 `src/middleware.ts`
```typescript
import { type NextRequest, NextResponse } from 'next/server';

export async function middleware(request: NextRequest) {
  // TODO: [AUTH] 인증 시스템 연동 시 Supabase middleware 복원
  return NextResponse.next({ request });
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api|$).*)'],
};
```

### 5.2 `src/lib/supabase/middleware.ts`
`updateSession` 함수를 패스스루로 교체

### 5.3 `src/lib/context/GlobalContext.tsx`
```typescript
// TODO: [AUTH] 인증 시스템 연동 시 실제 auth 로직 복원
const MOCK_USER = {
  email: 'dev@studio.picklass.com',
  id: 'mock-user-id-for-development',
  registered_at: new Date(),
};
// auth 이벤트 리스너 제거, loading: false, user: MOCK_USER로 즉시 설정
```

### 5.4 `src/components/ClassLayout.tsx`
auth 관련 핸들러(로그아웃, 비밀번호 변경) 제거, 사용자 표시를 정적으로 변경
```typescript
// TODO: [AUTH] 로그아웃, 비밀번호 변경 핸들러 복원
```

### 5.5 모든 API route 파일 (약 20개)
현재 패턴:
```typescript
const supabase = await createServerClient();
const { data: { user } } = await supabase.auth.getUser();
if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

변경 패턴:
```typescript
// TODO: [AUTH] 인증 시스템 연동 시 실제 auth 검증 복원
const MOCK_USER_ID = 'mock-user-id-for-development';
const userId = MOCK_USER_ID;
```

### 5.6 `src/lib/supabase/unified.ts`
auth 관련 메서드 제거, 데이터 접근 메서드만 유지
```typescript
// TODO: [AUTH] 인증 메서드 복원
// 제거: loginEmail, registerEmail, resendVerificationEmail, logout, exchangeCodeForSession
// 유지: getStrategicReadingData, deleteStrategicReadingData, getSupabaseClient, 파일 업로드 메서드
```

---

## 6. 삭제할 Lock 파일

```
yarn.lock          # pnpm 사용
package-lock.json  # pnpm 사용
```

---

## 7. 변경 요약 다이어그램

```
삭제                              유지 (class/ 의존)
─────────────────────────────────────────────────────────
/app/app/* (전체)                /app/class/*          (학습 모듈)
/app/auth/*                      /app/api/*            (20개 API route)
/app/api/auth/*                  /app/page.tsx         (랜딩)
/app/api/texts/bulk-delete       /app/legal/*
                                 /app/layout.tsx       (루트 레이아웃)

components/strategic-reading/*   components/lesson/*   (18개)
components/AppLayout             components/ClassLayout
components/MFA*, SSO*, Auth*     components/SubscriptionGuard
components/HomePricing           components/FontSizeSelector
components/Confetti (미사용)     components/Create/Detail/Preview/TwinPassageModal*
components/TTSCacheManager       components/PassageInfoBadges, PassageTypeBadge
                                 components/ManualInputModal, AIGenerateModal
                                 components/ui/*
                                 components/landing/*, oizi/*
                                 components/providers/*
                                 components/Cookies, LegalDocument*

hooks/useStrategicReadingV2      hooks/useStrategicReading  ← class/lesson에서 사용!
hooks/useTextFilter*             hooks/useTexts, useTextAnalysis
                                 hooks/useBatchTTS, useAzureSpeech
                                 hooks/useUserProfile, useAsyncTask

lib/pricing.ts                   lib/supabase/*, lib/context/*
                                 lib/async-task/*, lib/realtime/*
                                 lib/tts-cache.ts, lib/utils/*
                                 lib/types*, lib/constants, lib/react-query
```

---

## 8. 검증 체크리스트

### 빌드 검증
- [ ] `pnpm install` 성공
- [ ] `pnpm build` TypeScript 에러 없이 성공
- [ ] 콘솔에 import 에러 없음

### 라우트 검증
- [ ] 랜딩 페이지 (`/`) 정상 렌더링
- [ ] 교실 홈 (`/class`) — 텍스트 라이브러리 표시, 지문 생성 가능
- [ ] 레슨 설정 (`/class/lesson-setup/[id]`) — 설정 UI 렌더링
- [ ] 라이브 레슨 (`/class/lesson/[id]`) — 단계별 학습 진행
- [ ] 삭제된 라우트 → 404: `/app/*`, `/auth/*`

### class/ 기능 검증
- [ ] 텍스트 목록 조회 (useTexts → GET /api/texts)
- [ ] 지문 생성 (CreatePassageModal → POST /api/generate-text)
- [ ] 전략적 읽기 분석 생성 (useGenerateStrategicReading → POST /api/strategic-reading)
- [ ] 각 학습 단계 피드백 (Prediction, Skimming, Scanning, Clarification, Summarizing, QAR)
- [ ] TTS 음성 합성 (FluencyStep → useBatchTTS → POST /api/azure-speech)
- [ ] 단어 학습 (WordDeck, WordWeb → /api/word-definitions, word-suggestion, word-relationship)
- [ ] 의미 추측 피드백 (MeaningGuessingStep → POST /api/meaning-guessing-feedback)
- [ ] Supabase Realtime 연결 정상
- [ ] Mock user로 모든 API 정상 동작
