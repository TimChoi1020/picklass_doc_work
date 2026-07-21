# Phase 2: 필요 모듈 추출 (Extract Modules)

> `/app/class/`가 유일한 학습 모듈이다. 이 모듈의 전체 의존성 체인을 정리하고 독립 동작을 보장한다.

---

## 1. 유지할 모듈 개요

| 모듈 | 위치 | 역할 |
|------|------|------|
| Classroom Lesson | `/app/class/` | 핵심 학습 (텍스트 라이브러리 + 단계별 레슨) |
| Landing | `/app/page.tsx` | 랜딩 페이지 |
| Legal | `/app/legal/` | 법적 문서 |

> `/app/app/` 전체가 삭제되므로, class/page.tsx가 텍스트 라이브러리(지문 관리) 역할을 겸한다.

---

## 2. Classroom Lesson 모듈 — 완전한 의존성 체인

### 2.1 페이지 & 레이아웃

| 파일 | 역할 | 핵심 import |
|------|------|-------------|
| `class/page.tsx` | 텍스트 라이브러리 | useTexts, CreatePassageModal, PassageDetailModal, PreviewPassageModal, TwinPassagesModal, PassageInfoBadges, PassageTypeBadge |
| `class/layout.tsx` | 교실 레이아웃 | ClassLayout, GlobalProvider |
| `class/lesson-setup/[passageId]/page.tsx` | 레슨 설정 | useText |
| `class/lesson/[passageId]/page.tsx` | 라이브 레슨 | useText, **useStrategicReadingData**, **useGenerateStrategicReading**, FontSizeSelector, 모든 lesson step 컴포넌트 |
| `class/lesson/[passageId]/layout.tsx` | 레슨 가드 | **SubscriptionGuard** |

### 2.2 레이아웃 컴포넌트

| 컴포넌트 | 사용 위치 | 핵심 의존성 |
|----------|-----------|-------------|
| `ClassLayout.tsx` | class/layout.tsx | useGlobal, useUserProfile, useTheme, createSPASassClient, ThemeToggle, PRODUCT_NAME/COMPANY_NAME/BRAND_NAME |
| `SubscriptionGuard.tsx` | class/lesson/layout.tsx | useUserProfile, useRouter |

### 2.3 레슨 컴포넌트 (src/components/lesson/ — 18개)

#### 읽기 전략 단계
| 컴포넌트 | API 호출 | 추가 의존성 |
|----------|----------|-------------|
| `PredictionStep.tsx` | `/api/prediction-feedback` | FontSizeSelector, FeedbackPanel |
| `SkimmingStep.tsx` | `/api/skimming-feedback` | FontSizeSelector, WPMSettingsModal, FeedbackPanel |
| `ScanningStep.tsx` | (strategic data 사용) | FontSizeSelector, ScanningSettingsModal, FeedbackPanel |
| `ClarificationStep.tsx` | `/api/clarification-feedback` | FontSizeSelector, FeedbackPanel |
| `SummarizingStep.tsx` | `/api/summary-feedback` | FontSizeSelector, FeedbackPanel |
| `QARStep.tsx` | `/api/qar-feedback` | FontSizeSelector, FeedbackPanel |

#### 유창성 읽기 단계
| 컴포넌트 | API 호출 | 추가 의존성 |
|----------|----------|-------------|
| `FluencyStep.tsx` | (via **useBatchTTS**) → `/api/azure-speech` | FontSizeSelector, Slider, FeedbackPanel |
| `MeaningGuessingStep.tsx` | `/api/meaning-guessing-feedback` | FontSizeSelector, **useTextAnalysis**, WordPickerModal, WordPickerButton, FeedbackPanel |

#### 어휘 학습 단계
| 컴포넌트 | API 호출 | 추가 의존성 |
|----------|----------|-------------|
| `WordDeckStep.tsx` | (via useTextAnalysis) | FontSizeSelector, **useTextAnalysis**, WordPickerModal, WordPickerButton, WordDeckLearning, wink-lemmatizer |
| `WordDeckLearning.tsx` | `/api/word-definitions`, (via **useAzureSpeech**) → `/api/azure-speech` | FeedbackPanel |
| `WordWebStep.tsx` | — | FontSizeSelector, WordWebLearning, wink-lemmatizer, react-force-graph-2d |
| `WordWebLearning.tsx` | `/api/word-definitions`, `/api/word-relationship`, `/api/word-suggestion` | FeedbackPanel |
| `WordWebDesign.tsx` | — | (WordWebLearning 내부) |

#### 공용 유틸리티
| 컴포넌트 | 역할 |
|----------|------|
| `FeedbackPanel.tsx` | AI 피드백 표시 (모든 단계에서 사용), `FeedbackContent` 타입 export |
| `WordPickerModal.tsx` | 단어 선택 모달 (MeaningGuessing, WordDeck) |
| `WordPickerButton.tsx` | 단어 선택 버튼 |
| `ScanningSettingsModal.tsx` | 스캐닝 설정 |
| `WPMSettingsModal.tsx` | WPM 설정 |

### 2.4 텍스트 관리 컴포넌트 (class/page.tsx에서 사용)

| 컴포넌트 | class/page.tsx에서 사용 확인 |
|----------|---------------------------|
| `CreatePassageModal.tsx` | 직접 import (**내부에 ManualInputModal, AIGenerateModal 포함**) |
| `ManualInputModal.tsx` | CreatePassageModal 내부 (DOCX 업로드 → `/api/extract-text`) |
| `AIGenerateModal.tsx` | CreatePassageModal 내부 (AI 텍스트 생성 → `/api/generate-text`) |
| `PassageDetailModal.tsx` | 직접 import |
| `PreviewPassageModal.tsx` | 직접 import |
| `TwinPassagesModal.tsx` | 직접 import |
| `PassageInfoBadges.tsx` | 직접 import |
| `PassageTypeBadge.tsx` | 직접 import |

### 2.5 훅 의존성 맵

| 훅 | 사용 위치 | 호출 API |
|----|-----------|----------|
| `useTexts.ts` (useTexts, useText, useCreateText, useTwinTexts) | class/page.tsx, lesson-setup, lesson | `/api/texts`, `/api/texts/[id]`, `/api/texts/twins/[id]` |
| `useStrategicReading.ts` (useStrategicReadingData, useGenerateStrategicReading) | class/lesson/page.tsx | `/api/strategic-reading` |
| `useBatchTTS.ts` | FluencyStep | `/api/azure-speech` (POST) |
| `useAzureSpeech.ts` | WordDeckLearning | `/api/azure-speech` (POST) |
| `useTextAnalysis.ts` | WordDeckStep, MeaningGuessingStep | `/api/text-analysis` |
| `useUserProfile.ts` | ClassLayout, SubscriptionGuard | `/api/user-profile` |
| `useAsyncTask.ts` | (useStrategicReading 내부 의존 가능) | `/api/async-tasks` |

### 2.6 공용 컴포넌트
```
src/components/FontSizeSelector.tsx             # 거의 모든 lesson step에서 사용
src/components/ui/button.tsx                    # Button
src/components/ui/input.tsx                     # Input
src/components/ui/textarea.tsx                  # Textarea
src/components/ui/badge.tsx                     # Badge
src/components/ui/slider.tsx                    # Slider (FluencyStep)
src/components/ui/alert-dialog.tsx              # AlertDialog (class/page.tsx)
src/components/ui/dialog.tsx                    # Dialog (class/page.tsx)
src/components/ui/simple-pagination.tsx         # SimplePagination (class/page.tsx)
src/components/ui/theme-toggle.tsx              # ThemeToggle (ClassLayout)
```

---

## 3. 완전한 API 의존성 맵 (class/에서 호출하는 20개)

```
class/page.tsx
  ├─ GET  /api/texts                    (useTexts)
  ├─ POST /api/texts                    (useCreateText)
  ├─ GET  /api/texts/twins/{id}         (useTwinTexts)
  ├─ POST /api/generate-text            (직접 fetch + CreatePassageModal)
  └─ POST /api/extract-text             (ManualInputModal)

class/lesson-setup/[passageId]/page.tsx
  └─ GET  /api/texts/{id}              (useText)

class/lesson/[passageId]/page.tsx
  ├─ GET  /api/texts/{id}              (useText)
  ├─ POST /api/strategic-reading        (useGenerateStrategicReading)
  └─ GET  /api/strategic-reading        (useStrategicReadingData)

lesson components (각 Step)
  ├─ POST /api/prediction-feedback      (PredictionStep)
  ├─ POST /api/skimming-feedback        (SkimmingStep)
  ├─ POST /api/clarification-feedback   (ClarificationStep)
  ├─ POST /api/summary-feedback         (SummarizingStep)
  ├─ POST /api/qar-feedback             (QARStep)
  ├─ POST /api/meaning-guessing-feedback (MeaningGuessingStep)
  ├─ POST /api/azure-speech             (FluencyStep via useBatchTTS)
  ├─ POST /api/azure-speech             (WordDeckLearning via useAzureSpeech)
  ├─ POST /api/text-analysis            (WordDeckStep, MeaningGuessingStep via useTextAnalysis)
  ├─ POST /api/word-definitions         (WordDeckLearning, WordWebLearning)
  ├─ POST /api/word-relationship        (WordWebLearning)
  └─ POST /api/word-suggestion          (WordWebLearning)

공통 (레이아웃/가드)
  ├─ GET  /api/user-profile             (ClassLayout, SubscriptionGuard via useUserProfile)
  └─ *    /api/async-tasks              (useAsyncTask - strategic reading 내부)
```

---

## 4. 외부 서비스 의존성

| 서비스 | 패키지 | 사용 위치 |
|--------|--------|-----------|
| Google Gemini | `@google/genai` | strategic-reading, 모든 feedback API, generate-text, word-definitions |
| Azure Speech | `microsoft-cognitiveservices-speech-sdk` | azure-speech API (FluencyStep, WordDeckLearning) |
| Mammoth | `mammoth` | extract-text API (ManualInputModal → CreatePassageModal) |
| Supabase | `@supabase/supabase-js`, `@supabase/ssr` | 모든 API route, Realtime |
| wink-lemmatizer | `wink-lemmatizer` | WordDeckStep, WordWebStep (클라이언트 동적 import) |
| Force Graph 2D | `react-force-graph-2d` | WordWebStep (클라이언트 동적 import) |

---

## 5. Supabase 테이블 의존성

| 테이블 | 사용 API |
|--------|----------|
| `texts` | /api/texts/*, /api/generate-text |
| `strategic_reading_data` | /api/strategic-reading |
| `async_tasks` | /api/async-tasks |
| `tts_cache` | /api/tts-cache/*, /api/azure-speech |
| `user_profiles` | /api/user-profile |
| `notifications` | Realtime 구독 |

---

## 6. 독립 동작 검증 방법

### 6.1 텍스트 라이브러리 (class/page.tsx)
1. `/class` 접근 → 텍스트 목록 표시
2. 지문 생성 (CreatePassageModal 열기)
   - 수동 입력 (ManualInputModal) → DOCX 업로드 → `/api/extract-text`
   - AI 생성 (AIGenerateModal) → `/api/generate-text`
3. 지문 상세 보기 (PassageDetailModal)
4. 지문 미리보기 (PreviewPassageModal)
5. 쌍둥이 지문 (TwinPassagesModal)
6. 페이지네이션 동작

### 6.2 레슨 설정 (class/lesson-setup/[passageId])
1. 지문 정보 로드 (useText → GET /api/texts/[id])
2. 레슨 설정 UI 렌더링

### 6.3 라이브 레슨 (class/lesson/[passageId])
1. SubscriptionGuard 통과 (useUserProfile)
2. 지문 로드 + 전략적 읽기 분석 생성 (useGenerateStrategicReading)
3. **각 학습 단계 순회:**
   - Prediction → `/api/prediction-feedback` 호출 → AI 피드백 표시
   - Skimming → WPM 설정 → `/api/skimming-feedback`
   - Scanning → ScanningSettings → 전략적 읽기 데이터 활용
   - Clarification → `/api/clarification-feedback`
   - Summarizing → `/api/summary-feedback`
   - QAR → `/api/qar-feedback`
   - Fluency → useBatchTTS → TTS 재생 → 반복 읽기
   - MeaningGuessing → useTextAnalysis → 단어 선택 → `/api/meaning-guessing-feedback`
   - WordDeck → useTextAnalysis → 단어 카드 → WordDeckLearning → `/api/word-definitions` + TTS
   - WordWeb → wink-lemmatizer → Force Graph → WordWebLearning → `/api/word-*`
4. FontSizeSelector로 글꼴 크기 조절
5. 단계 간 전환 정상

### 6.4 전체 통합 검증
- [ ] `pnpm build` TypeScript 에러 없음
- [ ] 브라우저 콘솔에 import 에러 없음
- [ ] 모든 API route가 Mock user ID로 정상 동작
- [ ] Supabase Realtime 연결 정상
- [ ] `/app/*` 접근 시 404
