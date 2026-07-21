# Picklass AI 튜터링 기획문서

**작성일**: 2026년 4월 1일  
**버전**: 1.0  
**대상**: 기획자, 개발자, 디자이너, 투자자

---

## 📑 목차

1. [개요](#1-개요)
2. [제품 기능](#2-제품-기능)
3. [사용자 여정 & 플로우](#3-사용자-여정--플로우)
4. [기술 아키텍처](#4-기술-아키텍처)
5. [모듈 시스템](#5-모듈-시스템)
6. [로드맵](#6-로드맵)
7. [성공 지표](#7-성공-지표)
8. [리스크 & 완화 전략](#8-리스크--완화-전략)
9. [부록](#9-부록)

---

## 1. 개요

### 1.1 프로젝트 소개

**Picklass AI 튜터링(tutoring.picklass.com)**은 학생이 AI 튜터(Pickle)의 개인맞춤형 안내를 받으며 영어 학습을 진행하는 웹 기반 학습 플랫폼입니다.

**핵심 특징:**
- 🤖 **AI 기반 개인 맞춤형 학습**: CurriculumPlannerAgent가 학생의 목표와 수준에 맞춰 지문과 모듈을 조합하여 맞춤형 레슨 플랜 자동 생성
- 🎯 **실시간 적응형 학습 경로**: ModuleOrchestratorAgent가 매 상호작용마다 학생 상태를 분석하고 다음 행동 결정 (4가지 요소 통제: UI, 콘텐츠, 피드백, 다음 액션)
- 📚 **모듈식 학습 구조**: 지문 1개 + 모듈 다수 조합으로 유연한 레슨 설계
- 📊 **성과 추적 및 피드백**: 모듈별 KPI 기반 상세 성과 분석

### 1.2 핵심 기능

1. **혼합형 문제 유형 지원**
   - 객관식(Multiple Choice)
   - 객관식 듣기(Audio Listen)
   - 단답형(Short Text)
   - 서술형(Essay)
   - 음성 녹음(Audio Record - 쉐도잉)
   - 영작문(Sentence Write)
   - 창작형(Process Writing - 개요, 단락 작성)

2. **AI 피드백 시스템**
   - 객관식: 즉시 정오 판정
   - 서술형(Essay): AI 홀리스틱 평가 + 피드백
   - 음성(Audio-Record): 발음 점수 기반 평가
   - 조건부 힌트: 반복 오답 시 단계적 힌트 제공

3. **지문 관리**
   - 학습 단계별 지문 노출 제어 (전 → 중 → 후)
   - 어휘 강조(Highlights) 기능
   - 읽기 시간 예측

### 1.3 타겟 사용자

| 역할 | 사용 시나리오 | 주요 목표 |
|------|-------------|---------|
| **학생** | /modules/[lessonId] 접속 | 개인 맞춤형 레슨 진행, 실시간 피드백 수집 |
| **교사/백오피스** | 레슨 플랜 구성 또는 AI 자동 생성 | 교육과정 설계, 학생 성과 모니터링 |
| **관리자** | 모듈 생성 & 버전 관리 | 모듈 라이브러리 확대, 콘텐츠 품질 관리 |

---

## 2. 제품 기능

### 2.1 학생 측면

#### 2.1.1 AI 튜터 상호작용 (Pickle)
- **실시간 대화**: 학생의 질문이나 피드백 요청에 AI 튜터가 응답
- **단계적 스캐폴딩**: 힌트 레벨 상승 (LV1 → LV4)
- **격려 및 동기부여**: 성공/어려움 상황에 맞춘 감정 지원

**AI 통제 항목:**
- UI 렌더링: 어떤 화면(지문, 문항, 피드백 패널)을 언제 노출할지
- 콘텐츠 선택: 제공할 피드백의 깊이와 유형
- 피드백 타이밍: 즉시/지연 피드백 결정
- 다음 액션: 다음 문항 제시 vs 모듈 완료 vs 재교수 판단

#### 2.1.2 모듈식 레슨 구조
```
지문(Passage) 1개
  ↓
모듈 1 (예: 예측하기)
  ↓
모듈 2 (쉐도잉)
  ↓
모듈 3 (창작형)
  → 모듈 완료, 성과카드 표시
```

**특징:**
- 모듈 간 독립적 답변 저장
- 모듈별 KPI 기반 성과 추적
- 모듈 실패 시 대체 경로 제공 (리플래닝)

#### 2.1.3 진행도 추적
- 레슨 내 모듈별 완료 진행도 바
- 모듈별 정답률(`score`) 표시
- 최종 완료: 평균 정답률 + 모듈 목록

### 2.2 교사/백오피스 측면

#### 2.2.1 과정 생성 (지문 중심)
**흐름: 지문 → 레슨 → 과정**

1. **지문 선정**: 학습 목표에 맞는 지문 선택 또는 작성
2. **지문 분석** (CurriculumPlannerAgent):
   - CEFR 레벨 추정 (1–18)
   - 유형 분류 (서사, 설명, 논증, 묘사)
   - 핵심 어휘 추출
   - 예상 읽기 시간
3. **지문당 레슨 생성** (CurriculumPlannerAgent):
   - 학습 목표 + 예상 소요시간 기반 모듈 선정
   - 모듈 순서 최적화 (warming → passage-use → practice → output)
   - 난이도 조절 (학생 수준에 따른 선택 모듈)
4. **레슨 다중화 → 과정**: 여러 지문의 레슨을 모아 완성된 과정 구성

#### 2.2.2 지문 관리 (Passage Management)
- 지문 생성/수정
- 난이도 레벨 설정
- 주제 및 스킬 태그 지정

#### 2.2.3 적응형 모듈 결합
- **역할(Role) 기반 모듈 선택**:
  - `warming`: 도입 (어휘, 배경지식)
  - `passage-use`: 본격 읽기 (예측, 스캔, 스키밍)
  - `practice`: 연습 (이해도 확인)
  - `output`: 산출 (창작, 요약)
- **선행조건(Prerequisites)**: 모듈 A 완료 후 모듈 B 진행
- **양립불가(Incompatible)**: 중복/충돌 모듈 방지

---

## 3. 사용자 여정 & 플로우

### 3.1 학생 여정 (온보딩 → 레슨 진행 → 완료)

```
1단계: 레슨 진입
┌─ /modules/[lessonId] 접속
├─ LessonPlan 로드 (레슨 구조, 모듈 시퀀스, 지문 분석)
└─ 로딩 스피너 표시 중

2단계: 첫 모듈 시작
┌─ ModuleRunner가 모듈 데이터 fetch
├─ ContentConfig 생성 (어댑터가 UI 배치 정책 결정)
├─ 지문 노출 여부 결정 (hidden/preview/full)
└─ 첫 화면 렌더링 (ModuleRunnerInner)

3단계: 실시간 학습 (ModuleOrchestratorAgent)
┌─ Rule 기반 의사결정 루프:
│  1. 무응답 체크 (120초 이상 → 참여도 확인)
│  2. 반복 오답 (2회 이상 → 힌트 레벨 상승)
│  3. 피드백 생성 (essay/audio → AI 분석)
│  4. 다음 문항 or 완료 결정
├─ 매 상호작용마다 4가지 요소 통제:
│  • UI 노출: 지문/문항/피드백
│  • 콘텐츠: 피드백 깊이/유형
│  • 피드백: 즉시/지연 타이밍
│  • 다음액션: 선택지 및 권유사항
└─ 모듈 완료까지 반복

4단계: 모듈 완료 & 성과카드
┌─ onModuleComplete 호출
├─ 모듈 결과 저장 (정답률, KPI, 채팅 이력)
├─ 성과카드 표시:
│  • 모듈명
│  • 정답률 (%)
│  • 모듈별 KPI 차트 (예: 발음 점수 평균, 예측 타당성)
└─ "다음 모듈" 버튼 제시

5단계: 다음 모듈 → 반복 (3–4 반복)
┌─ LessonSession.setCurrentModuleIdx(n+1)
├─ ModuleRunner 리렌더링
└─ 2단계~4단계 반복

6단계: 레슨 완료
┌─ moduleSequence 모두 완료
├─ 최종 화면:
│  • "레슨 완료!" 메시지
│  • 평균 정답률 (모듈별 점수 평균)
│  • 모듈별 성과 목록
│  • 모듈별 상세 KPI (선택 시 추가 상세화면)
└─ "홈으로 돌아가기" 버튼
```

### 3.2 교사 여정 (과정 생성)

```
1단계: 지문 선정 또는 작성
┌─ 백오피스 "지문 관리" → "지문 생성"
├─ 지문 텍스트 입력
├─ 학습 목표 설정 (예: "쇼핑 관련 표현 숙달")
└─ 목표 난이도 레벨 선택 (CEFR: A1~C2)

2단계: AI 자동 지문 분석
┌─ CurriculumPlannerAgent 호출
├─ PassageAnalysis 생성:
│  • 추정 난이도 레벨
│  • 지문 유형 (서사/설명/논증/묘사)
│  • 핵심 어휘 목록
│  • 예상 읽기 시간
└─ 결과 검토 및 수정 가능

3단계: 지문별 레슨 생성 (AI 기반)
┌─ "레슨 생성" 버튼
├─ CurriculumPlannerAgent 호출:
│  • 지문분석 + 목표 + 가용시간(예: 15분)
│  • 이용 가능한 모듈 라이브러리 조회
│  • 자동 모듈 필터링 & 시퀀싱
├─ LessonPlan 생성 (moduleSequence 확정)
├─ 결과 검토:
│  • 선정 이유(Rationale) 표시
│  • 각 모듈의 역할 확인
│  • 소요 시간 검증
└─ 승인 → DB 저장 또는 수정

4단계: 모듈 수동 조정 (선택)
┌─ 자동 생성 결과가 마음에 들지 않으면
├─ 모듈 추가/제거/순서 변경
├─ 각 모듈별 선행조건/확장조건 설정
│  예: "이전 모듈 점수 > 90이면 다음 스킵"
└─ 저장

5단계: 학생 배정 및 배포
┌─ 구성된 레슨을 클래스/학생에 배정
├─ 학생은 링크를 통해 /modules/[lessonId] 접속
└─ 학습 시작

6단계: 학생 성과 모니터링
┌─ 어댑터(대시보드) 접속
├─ 학생별 진행도:
│  • 각 모듈 정답률
│  • 모듈별 KPI 상세 데이터
│  • 평균 소요 시간
└─ 리포트 생성 (월간/주간)
```

### 3.3 관리자 여정 (모듈 관리)

```
1단계: 모듈 생성 & 설정
┌─ 백오피스 "모듈 라이브러리" → "새 모듈 생성"
├─ 기본 정보:
│  • 모듈 코드 (예: "PRD", "SHR")
│  • 모듈 이름 및 설명
│  • 스킬 분류 (reading/speaking/listening/writing)
├─ 교수법 프로파일 설정:
│  • 문항 유형 (essay/audio-record/multiple-choice 등)
│  • 채점 방식 (exact/holistic/pronunciation)
│  • 지문 노출 모드 (hidden/preview/full)
│  • AI 피드백 스타일 (correct-wrong/strengths-weaknesses)
│  • 모듈 목적 및 지시문
├─ 모듈 메타데이터:
│  • 역할 (warming/passage-use/practice/output)
│  • 선행조건 (prerequisites)
│  • 양립불가 (incompatibleWith)
│  • 적정 레벨 범위 (min–max CEFR)
│  • 예상 소요시간
│  • Bloom's 인지 수준 (1–6)
└─ 초안 저장

2단계: 모듈 콘텐츠 구성
┌─ 모듈별 어댑터 구현 (ModuleAdapter)
├─ 지문 및 문항 데이터 DB에 저장
├─ ContentConfig 정의:
│  • UI 배치 (데스크탑/모바일 레이아웃)
│  • 컴포넌트 노출 순서
│  • 탭 구조 (지문/문항/피드백)
└─ 테스트 모드 실행

3단계: 모듈 버전 관리
┌─ 기존 모듈 수정 필요 시
├─ 새 버전 생성 (v1.0 → v1.1)
├─ 변경사항:
│  • 문항 수정
│  • 채점 기준 변경
│  • 피드백 템플릿 업데이트
├─ 호환성 검증:
│  • 기존 레슨 플랜 영향도 검토
│  • 필요 시 마이그레이션 계획 수립
└─ 배포 (기존 레슨에 적용할지, 신규만 적용할지 선택)

4단계: 모듈별 성과 분석
┌─ 모듈 대시보드 접속
├─ 집계 지표:
│  • 모듈별 평균 정답률
│  • 모듈별 KPI 분포
│  • 난이도별 이용률
│  • 학생 피드백 (별점, 의견)
├─ 개선 아이템 식별:
│  • 이해도 낮은 부분 (정답률 < 50%)
│  • 시간 소비 이상 (예상 시간 대비 2배 이상)
│  • 높은 난이도 모듈 접근성 개선
└─ 버전 업그레이드 또는 재설계 검토
```

### 3.4 주요 UX 시나리오

#### 시나리오 1: 학생이 반복해서 같은 문항을 틀린 경우
```
1회 오답 → "한 번 더 시도해봐!"
2회 오답 → hint level 1: "문장 구조를 다시 읽어보세요"
3회 오답 → hint level 2: "선택지 (A)를 고려해보세요"
4회 이상  → hint level 3: "정답은 (A)입니다. 이유는…"
```

#### 시나리오 2: 120초 이상 무응답
```
AI 메시지: "잠깐 쉬고 있나요? 계속할 준비가 되면 알려주세요!"
→ 만약 다시 180초 이상 무응답이면 "disengaged" 신호
→ ModuleOrchestratorAgent가 signalReplan 호출
→ 다음 모듈 난이도 하향 또는 스킵 제안
```

#### 시나리오 3: 영작문(Sentence Write) 답안 제출
```
1. 학생 답안 제출
2. ModuleOrchestratorAgent 판단
   → ruleHolisticFeedback 트리거
3. AI (Claude API)가 답안 평가
   - 문법 정확성
   - 의미 적절성
   - 표현력 평가
4. 피드백 패널에 세부 평가 + 개선 제안 표시
5. "이해했어요" 버튼 클릭 시 다음 문항 진행
```

#### 시나리오 4: 음성 녹음(SHR) 문항 진행
```
1. 모델 음성 재생 (문장/단락)
2. 학생이 "쉐도잉 시작" 버튼 클릭
3. 브라우저 마이크 녹음 시작
4. 녹음 완료 후 제출
5. AI 발음 평가 엔진 (예: Google Speech-to-Text + Phonetic Analysis)
   → 각 단어별 발음 점수 계산
   → 전체 점수 = 모든 단어 평균
6. 점수 + 피드백 패널 표시
7. "다시 시도" 또는 "다음" 선택
```

---

## 4. 기술 아키텍처

### 4.1 시스템 구조도

```
┌─ Frontend (Next.js 15) ────────────────────────────────────┐
│                                                              │
│  /modules/[lessonId]                                         │
│    ├─ ModulesPage (페이지 진입, LessonPlan fetch)          │
│    └─ LessonSession (moduleSequence 순서 관리)            │
│         └─ ModuleRunner (어댑터 기반 모듈 데이터 fetch)    │
│              └─ ModuleRunnerInner (UI 렌더링)             │
│                   └─ useModuleOrchestrator (AI 로직)      │
│                        └─ ModuleOrchestratorAgent        │
│                            (Rule 기반 실시간 의사결정)    │
│                                                              │
│  API Layer (Axios + fetchApi 유틸)                          │
│    ├─ GET /api/lesson/{id}/plan                           │
│    ├─ POST /api/module/{code}/data                        │
│    ├─ POST /api/module/{code}/evaluate                    │
│    └─ POST /api/agent/orchestrator/decide                 │
│                                                              │
│  Local State (Zustand 예정)                                 │
│    ├─ moduleState (현재 모듈, 답변, 피드백)               │
│    ├─ sessionState (사용자, 레슨 진행도)                  │
│    └─ aiState (AI 메시지, 로딩)                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌─ Backend (NestJS) ────────────────────────────────────────┐
│                                                              │
│  APIs:                                                       │
│  ├─ LessonController                                        │
│  │  ├─ GET /api/lesson/{id}/plan                          │
│  │  └─ GET /api/lesson/{id}/history                       │
│  ├─ ModuleController                                        │
│  │  ├─ POST /api/module/{code}/data                       │
│  │  ├─ POST /api/module/{code}/evaluate                   │
│  │  └─ POST /api/module/{code}/metadata                   │
│  └─ ChatController (피드백 생성)                           │
│     ├─ POST /api/chat/feedback                            │
│     └─ POST /api/chat/message                             │
│                                                              │
│  Service Layer (packages/core):                             │
│  ├─ LessonService (LessonPlan CRUD)                       │
│  ├─ ModuleService (모듈 데이터 관리)                       │
│  ├─ CurriculumPlannerAgent (레슨 자동 생성)              │
│  ├─ ModuleOrchestratorAgent (실시간 의사결정)            │
│  └─ FeedbackService (AI 피드백 생성)                      │
│                                                              │
│  Database:                                                   │
│  ├─ Lesson (레슨 플랜, moduleSequence, learningGoal)      │
│  ├─ Module (모듈 마스터, 메타데이터, 버전)               │
│  ├─ PassageData (지문 콘텐츠)                             │
│  ├─ QuestionData (문항 데이터)                            │
│  ├─ ModuleHistory (학생별 모듈 답변 기록)                │
│  └─ LessonResult (레슨 최종 결과, KPI)                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌─ AI Layer ────────────────────────────────────────────────┐
│                                                              │
│  CurriculumPlannerAgent:                                    │
│  ├─ Input: PassageData + LearningGoal + TargetLevel       │
│  ├─ Tools: analyzePassage, filterModules, sequenceModules │
│  └─ Output: LessonPlan (with ModuleSequence)              │
│                                                              │
│  ModuleOrchestratorAgent:                                   │
│  ├─ Input: OrchestratorContext (LessonPlan + ModuleData    │
│  │         + LearnerState + ChatHistory)                   │
│  ├─ Tools: showPassage, presentQuestion, provideFeedback,  │
│  │         signalReplan, checkEngagement, celebrate        │
│  └─ Output: OrchestratorToolCall (단일 Tool 호출)        │
│                                                              │
│  FeedbackGenerationAgent:                                   │
│  ├─ Input: 학생 답안 + 정정 루브릭                       │
│  ├─ Handle: essay, audio-record 피드백                   │
│  └─ Output: 상세 피드백 텍스트                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 백엔드 (NestJS, PostgreSQL)

**역할**: API Gateway + 비즈니스 로직 오케스트레이션

**주요 엔드포인트:**
- `GET /api/lesson/{lessonId}/plan` — LessonPlan 조회
- `POST /api/module/{code}/data` — 모듈 콘텐츠 데이터 fetch
- `POST /api/module/{code}/evaluate` — 답안 채점 (즉시 판정)
- `POST /api/chat/feedback` — AI 피드백 생성 (essay/audio)
- `POST /api/agent/orchestrator/decide` — 다음 액션 결정 (향후 Claude API 호출로 교체)

**데이터베이스:**
```sql
-- 레슨 계획
CREATE TABLE lessons (
  id UUID PRIMARY KEY,
  passage_id UUID NOT NULL,
  learning_goal TEXT,
  target_level INT,
  module_sequence JSONB, -- PlannedModule[]
  created_at TIMESTAMP
);

-- 모듈 마스터
CREATE TABLE modules (
  id UUID PRIMARY KEY,
  code VARCHAR(10) UNIQUE, -- 'PRD', 'SHR' 등
  name VARCHAR(255),
  skill VARCHAR(50),
  pedagogy_profile JSONB, -- ModulePedagogyProfile
  content_config JSONB,   -- ContentConfig
  version VARCHAR(20),
  created_at TIMESTAMP
);

-- 모듈 학생별 이력
CREATE TABLE module_histories (
  id UUID PRIMARY KEY,
  lesson_id UUID,
  student_id UUID,
  module_code VARCHAR(10),
  answers JSONB, -- { questionId: answer }
  chat_messages JSONB, -- ChatMessage[]
  score INT,
  kpis JSONB, -- KpiResult[]
  completed_at TIMESTAMP
);
```

### 4.3 프론트엔드 (Next.js 15)

**아키텍처:**
- **App Router**: `/modules/[lessonId]/page.tsx`
- **컴포넌트 계층**:
  - `ModulesPage`: 진입점, LessonPlan 로드
  - `LessonSession`: 모듈 시퀀스 관리, 완료 누적
  - `ModuleRunner`: 어댑터 기반 모듈 데이터 fetch
  - `ModuleRunnerInner`: UI 렌더링 (지문, 문항, 피드백)

**상태 관리:**
- **Hooks**:
  - `useModuleOrchestrator`: 실시간 의사결정 로직 (OrchestratorContext → OrchestratorToolCall)
  - `useStageStateMachine`: 문항별 상태 머신 (unanswered → answered → feedback-shown → completed)

**API 호출:**
```typescript
// src/lib/api.ts
export const fetchApi = async (endpoint: string, options?: {}) => {
  // API_BASE_URL + endpoint로 요청
};
```

**UI 라이브러리:** Tailwind CSS + shadcn/ui

### 4.4 AI 튜터링 엔진 (Agent 오케스트레이션)

**핵심 개념: 2단계 Agent 구조**

#### 4.4.1 Agent 아키텍처 개요

```
        CurriculumPlannerAgent (Macro Layer)
                ↓
    [레슨 생성 시 1회 호출]
         LessonPlan 생성
   (moduleSequence 확정)
                ↓
   학생이 레슨에 접속
                ↓
   ModuleOrchestratorAgent (Micro Layer)
   [매 상호작용마다 호출]
     Rule 기반 의사결정
      (Tool 실행 결정)
                ↓
    UI 업데이트 + 피드백
```

#### 4.4.2 동적 모듈 시퀀싱 메커니즘

**입력:**
- 지문(Passage): 학습 자료
- 학습 목표: "영어 표현 능력 향상"
- 목표 레벨: CEFR B2
- 가용 시간: 15분
- 이용 가능 모듈 목록: [PRD, SHR, QAR, SWR, PWR] (메타데이터 포함)

**처리 (CurriculumPlannerAgent):**
1. **PassageAnalysis**: 지문 난이도, 유형, 주제 분석
2. **ModuleFiltering**: 지문 분석 + 목표 레벨 기반 적합 모듈 필터링
   - 예: "B2 레벨 지문 → B1~C1 모듈만 선택"
3. **Sequencing**: 역할 순서에 따른 모듈 배열
   - warming → passage-use → practice → output
   - 선행조건, 양립불가 제약 적용
4. **TimeEstimation**: 총 소요시간 계산
5. **Adjustment**: 시간 초과 시 모듈 축소/교체

**출력: LessonPlan**
```json
{
  "lessonId": "lesson-123",
  "passageId": "passage-456",
  "passageAnalysis": {
    "estimatedLevel": 13,
    "type": "expository",
    "keyVocabulary": ["commerce", "transaction", "digital"],
    "estimatedReadingMinutes": 5
  },
  "moduleSequence": [
    {
      "order": 1,
      "moduleCode": "PRD",
      "role": "warming",
      "passageExposed": false,
      "estimatedMinutes": 3
    },
    {
      "order": 2,
      "moduleCode": "SHR",
      "role": "passage-use",
      "passageExposed": true,
      "estimatedMinutes": 5
    },
    {
      "order": 3,
      "moduleCode": "SWR",
      "role": "practice",
      "passageExposed": true,
      "estimatedMinutes": 4
    }
  ],
  "totalEstimatedMinutes": 12
}
```

#### 4.4.3 4가지 요소 통제 구조

**ModuleOrchestratorAgent가 실시간으로 제어하는 요소:**

| 요소 | 설명 | 예시 |
|------|------|------|
| **UI 노출** | 어떤 화면 요소를 언제 보여줄지 | "지문 숨김" → 문항 3 오답 후 → "지문 전체 공개" |
| **콘텐츠 선택** | 피드백의 깊이/유형 결정 | essay 답안: "간단한 평가" vs "상세 피드백" (학생 참여도에 따라) |
| **피드백 타이밍** | 즉시/지연 피드백 판단 | 첫 오답: 즉시, 4회 오답: 힌트 레벨 상승 후 지연 |
| **다음 액션** | 다음 문항 vs 모듈 완료 vs 리플래닝 | 모든 문항 정답 → 다음 모듈, 120초 무응답 + 참여도 낮음 → 난이도 하향 제안 |

#### 4.4.4 실시간 의사결정 메커니즘

**Rule-Based Decision Engine (현재 구현)**

```
학습자 상호작용 (답변 제출 / 메시지 발송 / 타이머)
  ↓
orchestrate(silent=false)
  ↓
OrchestratorContext 구성:
  • LessonPlan
  • ModuleData
  • LearnerState (답변, 채팅 이력, 참여도, 유휴시간)
  • ContentConfig
  ↓
ModuleOrchestratorAgent.decideNextAction(context)
  ↓
우선순위 규칙 순서대로 평가:
  1. ruleIdleCheck (120초 무응답)
  2. ruleDisengaged (참여도 낮음 + 180초 무응답)
  3. ruleRepeatWrongAnswer (2회 이상 오답)
  4. ruleHolisticFeedback (essay 제출 직후)
  5. rulePronunciationFeedback (audio 제출 직후)
  6. ruleWrongAnswerFeedback (객관식 오답)
  7. ruleCorrectAnswer (정답 처리)
  8. ruleInitialEntry (최초 진입)
  9. rulePresentNextQuestion (기본 흐름)
  ↓
OrchestratorToolCall 반환:
  {
    tool: "provideFeedback" | "presentQuestion" | "signalReplan" | ...
    params: { /* 도구 실행 매개변수 */ },
    reason: "설명"
  }
  ↓
executeToolCall(toolCall)
  → React 상태 업데이트
  → AI 피드백 생성 (필요 시 Claude API 호출)
  → UI 리렌더링
  ↓
(필요 시) 자동 체이닝:
  setTimeout(() => orchestrate(silent=true), 1000)
```

**규칙 우선순위 상세:**

| # | 규칙명 | 조건 | 출력 Tool | 설명 |
|---|--------|------|----------|------|
| 1 | IdleCheck | idleSeconds >= 120 | checkEngagement | 120초 이상 무응답 → 참여도 확인 |
| 2 | Disengaged | engagementLevel='low' && idleSeconds >= 180 && 미완료 | signalReplan | 3분 무응답 + 저참여 → 리플래닝 |
| 3 | RepeatWrongAnswer | 동일 문항 2회 이상 오답 (다지선, 빈칸 등) | provideFeedback | 힌트 레벨 상승: LV1→2→3→정답공개 |
| 4 | HolisticFeedback | essay 타입 & 방금 제출 | provideFeedback | AI 홀리스틱 평가 (Claude) |
| 5 | PronunciationFeedback | audio-record 타입 & 방금 제출 | provideFeedback | 발음 점수 + 개선 제안 |
| 6 | WrongAnswerFeedback | 객관식 오답 | provideFeedback | "틀렸어요. 다시 시도해보세요" |
| 7 | CorrectAnswer | 정답 | presentQuestion 또는 celebrate | 다음 문항 또는 모듈 완료 |
| 8 | InitialEntry | 답변 없음 && 채팅 없음 | showPassage 또는 presentQuestion | 지문 노출 → 첫 문항 제시 |
| 9 | PresentNextQuestion | 기본 흐름 | presentQuestion | 다음 미답변 문항 제시 |

#### 4.4.5 모듈 메타데이터 인터페이스 (Agent 데이터)

**CurriculumPlannerAgent가 사용하는 프로퍼티:**
```typescript
interface ModulePlanningMeta {
  code: string;              // 모듈 코드 (시퀀싱 키)
  name: string;              // 모듈 이름
  skill: string;             // 스킬 카테고리
  roles: string[];           // ["warming", "passage-use", ...]
  passageExposure: string;   // "before" | "during" | "after" | "any"
  cognitiveLevel: number;    // Bloom's 1–6
  suitableLevels: { min, max };  // CEFR 레벨 범위
  estimatedMinutes: { min, max };
  prerequisites: string[];   // 선행 모듈
  incompatibleWith: string[]; // 양립불가 모듈
}
```

**ModuleOrchestratorAgent가 사용하는 프로퍼티:**
```typescript
interface ModuleData {
  code: string;              // 어댑터 판별
  questions: QuestionData[]; // 문항 목록 (개수, 유형, 정답)
  passage: PassageData;      // 지문 콘텐츠
}

interface ModulePedagogyProfile {
  scoringMode: "exact" | "holistic" | "pronunciation";
  answerType: string;
  passageExposureMode: "hidden" | "preview" | "full";
  feedbackStyle: string;
  pedagogyInstruction: string; // AI 시스템 프롬프트에 포함
}
```

**데이터 흐름 예시**
```
백오피스 레슨 편집
  ↓
[CurriculumPlannerAgent 영역]
  ModulePlanningMeta 읽기
  → 모듈 필터링, 시퀀싱
  → LessonPlan 생성 (moduleSequence 확정)
  ↓
학생 레슨 진입 (/modules/[lessonId])
  ↓
[ModuleOrchestratorAgent 영역]
  ModuleData 로드: questions 개수, 유형 파악
  ModulePedagogyProfile 로드: 채점 방식, 피드백 스타일 적용
  → 실시간 의사결정
  → UI 통제, 피드백 생성
```

#### 4.4.6 Agent Tool 스키마 정의

Agent가 호출할 수 있는 도구의 입출력 타입을 명확히 정의해야 Anthropic Tool Use API와 연동 가능하다.

**CurriculumPlannerAgent 도구**

| 도구 | 입력 | 출력 | 역할 |
|------|------|------|------|
| `analyzePassage` | `{ text: string, targetLevel: number }` | `PassageAnalysis` | 지문 난이도·유형·핵심 어휘 분석 |
| `filterModules` | `{ passageAnalysis, availableModules, learningGoal }` | `ModulePlanningMeta[]` | 적합 모듈 필터링 |
| `sequenceModules` | `{ modules, timeConstraint: number, learningGoal: string }` | `PlannedModule[]` | 최적 모듈 순서 결정 |

**ModuleOrchestratorAgent 도구**

| 도구 | 입력 파라미터 | 역할 |
|------|------------|------|
| `showPassage` | `{ mode: "hidden" \| "preview" \| "full" }` | 지문 노출 모드 전환 |
| `presentQuestion` | `{ questionId: string, hintLevel?: 0 \| 1 \| 2 \| 3 \| 4 }` | 문항 표시 (선택적 힌트 포함) |
| `provideFeedback` | `{ type: "correct" \| "incorrect" \| "holistic" \| "pronunciation", content: string, immediate: boolean }` | 피드백 패널 렌더링 |
| `checkEngagement` | `{ message: string, timeoutSeconds: number }` | 참여도 확인 메시지 전송 |
| `signalReplan` | `{ reason: string, suggestedDifficultyDelta: -1 \| 0 \| 1 }` | 난이도 재조정 요청 |
| `celebrate` | `{ completionType: "module" \| "lesson", score: number }` | 완료 축하 메시지 |

> **설계 원칙**: 각 Tool Call은 단일 의도(single intent)를 표현한다. 복수 액션이 필요한 경우 Tool 체이닝(자동 setTimeout orchestrate)으로 순차 처리한다.

#### 4.4.7 프롬프트 엔지니어링 구조

**시스템 프롬프트 계층 (ModuleOrchestratorAgent)**

```
[SYSTEM PROMPT 구조 — 매 모듈 시작 시 조립]

1. 역할 정의
   "You are Pickle, an AI English tutor. Your goal is to guide the student
   through the lesson using the available tools only — never respond in plain text."

2. 교수법 지시문 (모듈별 동적 삽입)
   pedagogyInstruction: "학생의 예측이 타당한지, 어떤 추론 전략을 사용했는지 분석합니다."

3. 피드백 스타일 (feedbackStyle 값 기반)
   "Provide feedback using the strengths-weaknesses format."

4. 컨텍스트 스냅샷 (구조화 전달, 자연어 최소화)
   - passageTitle, estimatedLevel
   - currentQuestionId, attemptCount, hintLevel
   - scoringMode (holistic / exact / pronunciation)

5. 도구 목록 (Anthropic Tool Use 형식)
```

**프롬프트 버전 관리 정책:**
- `pedagogyInstruction`은 모듈 버전(v1.0 / v1.1)과 함께 DB에 저장
- 세션 시작 시 프롬프트 내용을 스냅샷으로 고정 → 진행 중인 세션에 프롬프트 변경 미적용
- A/B 테스트: 동일 모듈에 두 가지 프롬프트 변형(`variant_a`, `variant_b`)을 동시 운영하여 피드백 품질 비교

#### 4.4.8 컨텍스트 윈도우 관리

**문제**: 긴 세션(20+ 상호작용)에서 `ChatHistory`가 컨텍스트 윈도우를 초과하거나 불필요한 토큰 비용 발생.

**전략:**

| 전략 | 적용 시점 | 방법 |
|------|---------|------|
| **Rolling Window** | 항상 | 최근 N개 메시지만 유지 (기본 N=20) |
| **점진적 요약** | 메시지 수 > 20 | 오래된 메시지 블록을 Claude로 요약 → 단일 요약 메시지로 교체 |
| **구조화 컨텍스트** | 매 요청 | 전체 히스토리 대신 `LearnerState` 구조체로 상태 전달 (토큰 절약) |

```typescript
// 컨텍스트 요약 트리거 예시
if (chatHistory.length > 20) {
  const summary = await summarizeHistory(chatHistory.slice(0, -10));
  chatHistory = [
    { role: 'system', content: `[이전 학습 요약] ${summary}` },
    ...chatHistory.slice(-10)
  ];
}
```

**토큰 예산 가이드라인:**

| 컴포넌트 | 권장 최대 토큰 |
|---------|-------------|
| System Prompt (역할 + 지시문) | ~500 |
| PassageAnalysis 스냅샷 | ~200 |
| ChatHistory (rolling) | ~1,500 |
| 학생 답안 (현재) | ~300 |
| Tool Definitions | ~400 |
| **합계** | **~2,900** (claude-haiku 컨텍스트 여유분 충분) |

#### 4.4.9 Agent 에러 처리 & Fallback

**Claude API 장애 시나리오별 대응:**

| 시나리오 | Fallback 전략 |
|---------|-------------|
| **API 타임아웃** (>5초) | Rule-Based 판단으로 자동 전환 (현재 구현이 기본값) |
| **Rate Limit 초과** | 지수 백오프 재시도 (최대 3회, 1s → 2s → 4s) |
| **잘못된 Tool Call 반환** | 스키마 검증 실패 시 `presentNextQuestion` 기본 액션 실행 |
| **빈 응답 / 파싱 실패** | 이전 정상 결과 재사용 또는 "잠시 후 다시 시도" 안내 |
| **컨텍스트 길이 초과** | 강제 Rolling Window 축소 후 재시도 |

**잘못된 Agent 출력 방어 (`guardrail` 레이어):**
```typescript
function validateToolCall(toolCall: OrchestratorToolCall): boolean {
  const validTools = ['showPassage', 'presentQuestion', 'provideFeedback',
                      'checkEngagement', 'signalReplan', 'celebrate'];
  return validTools.includes(toolCall.tool) && toolCall.params !== undefined;
}

// 검증 실패 시 기본 액션
if (!validateToolCall(result)) {
  return { tool: 'presentNextQuestion', params: { questionId: nextUnansweredId }, reason: 'fallback' };
}
```

**프롬프트 인젝션 방어:**
- 학생 입력(답안, 채팅)을 시스템 프롬프트에 직접 삽입하지 않고 별도 `user` role로 분리
- 입력 길이 제한: essay 최대 2,000자, 채팅 최대 500자
- 학생 입력 내 시스템 지시어 패턴(`ignore previous instructions`, XML 태그 등) 감지 및 이스케이프 처리

#### 4.4.10 스트리밍 피드백 전략

**문제**: essay/audio 피드백 생성에 3–8초 소요 → 빈 화면 대기로 UX 저하.

**해결: SSE(Server-Sent Events) 기반 스트리밍**

```
Frontend                    Backend                    Claude API
    │                           │                           │
    │── POST /api/chat/stream ──►│                           │
    │                           │── stream: true 요청 ──────►│
    │◄── SSE 스트림 시작 ────────│◄── 토큰 청크 전달 ─────────│
    │◄── "강점: " ──────────────│                           │
    │◄── "표현이 자연스러워요" ──│                           │
    │◄── [done 이벤트] ──────────│                           │
```

**추가 API 엔드포인트:**
- `POST /api/chat/feedback/stream` — essay/audio 피드백 SSE 스트리밍

**Frontend 처리:**
```typescript
const response = await fetch('/api/chat/feedback/stream', { method: 'POST', body: JSON.stringify(payload) });
const reader = response.body!.getReader();
for await (const chunk of readStream(reader)) {
  appendFeedbackText(chunk); // 실시간 타이핑 효과
}
```

**FeedbackGenerationAgent 상세:**
```
FeedbackGenerationAgent:
├─ Input: { studentAnswer, question, pedagogyProfile, passageContext }
├─ Streaming: true (SSE)
├─ Handle:
│  ├─ essay → holistic 평가 (문법 정확성, 의미 적절성, 표현력)
│  └─ audio-record → 발음 점수 기반 강점/약점 분석
└─ Output: 스트리밍 텍스트 → 완료 후 KpiResult 생성
```

### 4.5 데이터 흐름

```
시간 순서별 데이터 흐름:

[T0] 교사가 지문과 목표로 레슨 요청
  ↓ CurriculumPlannerAgent 호출
  ↓ PassageAnalysis + ModuleSequence 생성
  ↓ LessonPlan DB 저장

[T1] 학생이 /modules/[lessonId] 접속
  ↓ GET /api/lesson/{id}/plan
  ↓ LessonPlan 로드 (moduleSequence 포함)
  ↓ 첫 모듈(order=1) ModuleRunner 초기화

[T2] ModuleRunner가 모듈 데이터 로드
  ↓ getAdapter(moduleCode) 선택
  ↓ adapter.fetchModuleData(lessonId)
  ↓ ModuleData (passage + questions) 반환
  ↓ adapter.buildContentConfig(moduleData) 생성

[T3] ModuleRunnerInner 렌더링
  ↓ useModuleOrchestrator 초기화
  ↓ OrchestratorContext 구성
  ↓ ModuleOrchestratorAgent.decideNextAction() 호출 (silent=false)
  ↓ OrchestratorToolCall 반환 (예: "showPassage")
  ↓ executeToolCall() → UI 업데이트

[T4] 학생 상호작용 (답변 제출)
  ↓ submitAnswer(questionId, answer)
  ↓ LearnerState 업데이트 (answers, attemptHistory)
  ↓ orchestrate() 호출 (silent=false)
  ↓ ModuleOrchestratorAgent.decideNextAction()
  ↓ 예: ruleCorrectAnswer 또는 ruleWrongAnswerFeedback
  ↓ 예: provideFeedback Tool 실행
  ↓ 필요 시 FeedbackService.generateFeedback() (Claude API)
  ↓ 피드백 텍스트 반환
  ↓ UI 업데이트: 피드백 패널 표시

[T5] 모든 문항 완료
  ↓ orchestrate(silent=true) 자동 체이닝
  ↓ ruleCorrectAnswer 판정
  ↓ celebrate Tool 실행 (격려 메시지)
  ↓ 또는 replanSignal 발생
  ↓ onModuleComplete() 호출

[T6] 모듈 결과 저장
  ↓ ModuleResult 구성:
    { moduleCode, score, kpis, completedAt }
  ↓ saveModuleHistory(lessonId, moduleCode, ...)
  ↓ POST /api/module/{code}/history
  ↓ DB 저장

[T7] 다음 모듈 진입
  ↓ LessonSession.setCurrentModuleIdx(n+1)
  ↓ current PlannedModule = moduleSequence[n+1]
  ↓ ModuleRunner 리렌더링 (T2 반복)

[T8] 모두 완료
  ↓ completedResults.length >= moduleSequence.length
  ↓ LessonSession 최종 화면 렌더링
  ↓ 모듈별 점수 + 평균 정답률 표시
```

---

## 5. 모듈 시스템

### 5.1 모듈 프로퍼티 스키마

#### 5.1.1 모듈 구성 요소

**시퀀싱용 프로퍼티** (CurriculumPlannerAgent가 사용)
```typescript
{
  code: "PRD",                    // 모듈 식별자
  roles: ["warming"],              // 레슨 구조 내 역할
  passageExposure: "before",      // 지문 공개 시점
  cognitiveLevel: 2,              // Bloom's 수준
  suitableLevels: { min: 10, max: 14 },  // 적정 난이도
  estimatedMinutes: { min: 3, max: 5 },  // 표준 소요시간
  prerequisites: [],              // 선행 모듈
  incompatibleWith: ["SHR"],      // 충돌 모듈
}
```

**Agent 전달 데이터** (ModuleOrchestratorAgent가 사용)
```typescript
{
  questions: [                    // 문항 목록 (개수가 Agent 결정 영향)
    {
      type: "essay",              // 답변 유형 (정오판 방식)
      hint: "문장 구조를 생각해봐" // 동적 힌트
    }
  ],
  pedagogyProfile: {
    scoringMode: "holistic",      // 채점 방식
    feedbackStyle: "strengths-weaknesses",  // 피드백 톤
    passageExposureMode: "full"   // 지문 노출 모드
  }
}
```

**메타데이터** (UI 표시, 학생 정보용)
```typescript
{
  name: "Prediction (예측하기)",
  description: "지문 읽기 전 예측을 통한 독해 전략",
  skill: "reading",
  purpose: "배경지식과 예측 능력 강화"
}
```

#### 5.1.2 프로퍼티별 역할

| 프로퍼티 | 읽는 주체 | 영향 | 예시 |
|---------|---------|------|------|
| `code` | 시퀀서 + Agent | 모듈 식별, 어댑터 선택 | "PRD" → PredictionAdapter |
| `roles` | 시퀀서 | 레슨 구조 설계 (순서 결정) | ["warming"] → 시작 |
| `suitableLevels` | 시퀀서 | 학생 레벨과 비교 필터링 | B2 학생 → min=10 모듈만 포함 |
| `estimatedMinutes` | 시퀀서 | 총 시간 계산, 시간 제약 적용 | 15분 제한 → 3분 모듈 5개 선택 불가 |
| `prerequisites` | 시퀀서 | 선행 조건 확인 | PRD 완료 후만 SWR 진입 가능 |
| `questions의 개수` | Agent | 난이도 판단 | 국당식 10개 → 에세이 1개 (쉬움) vs 5개 (어려움) |
| `questions의 type` | Agent | 피드백 방식 결정 | type="essay" → holistic 평가, type="audio-record" → pronunciation 점수 |
| `pedagogyProfile.scoringMode` | Agent | 정답 판정 방식 | holistic → AI 평가, exact → 문자 비교 |
| `pedagogyProfile.feedbackStyle` | Agent | 피드백 톤 결정 | strengths-weaknesses → 긍정적 피드백 |
| `pedagogyProfile.passageExposureMode` | Agent | UI 노출 결정 | hidden → 지문 숨김, full → 공개 |

#### 5.1.3 구체적 모듈 예시

**모듈 1: PRD (Prediction)**
```json
{
  "code": "PRD",
  "name": "Prediction (예측하기)",
  "skill": "reading",
  "roles": ["warming"],
  "passageExposure": "before",
  "cognitiveLevel": 2,
  "suitableLevels": { "min": 10, "max": 14 },
  "estimatedMinutes": { "min": 3, "max": 5 },
  "prerequisites": [],
  "incompatibleWith": [],
  
  "pedagogyProfile": {
    "scoringMode": "holistic",
    "answerType": "essay",
    "passageExposureMode": "hidden",
    "questionCount": "single",
    "feedbackStyle": "strengths-weaknesses",
    "purpose": "배경지식 활성화 및 예측 능력 강화",
    "pedagogyInstruction": "학생의 예측이 타당한지, 어떤 추론 전략을 사용했는지 분석합니다."
  },
  
  "contentConfig": {
    "layout": "desktop-single-column",
    "tabs": ["result-card"],
    "components": {
      "passage": { "visible": false },
      "question": { "visible": true, "position": 1 },
      "feedback": { "visible": true, "position": 2 }
    }
  }
}
```

**모듈 2: SHR (Shadow Reading)**
```json
{
  "code": "SHR",
  "name": "Shadow Reading (쉐도잉)",
  "skill": "speaking",
  "roles": ["passage-use", "practice"],
  "passageExposure": "during",
  "cognitiveLevel": 3,
  "suitableLevels": { "min": 11, "max": 15 },
  "estimatedMinutes": { "min": 5, "max": 8 },
  "prerequisites": [],
  "incompatibleWith": ["PRD"],  // 같은 세션에 함께 쓰지 않음
  
  "pedagogyProfile": {
    "scoringMode": "pronunciation",
    "answerType": "audio-record",
    "passageExposureMode": "full",
    "questionCount": "multi",
    "feedbackStyle": "correct-wrong",
    "purpose": "발음과 유창성 향상",
    "pedagogyInstruction": "학생의 발음을 단어 수준에서 평가합니다. 각 단어를 0-100으로 점수 매기고, 전체 평균을 반환합니다."
  },
  
  "contentConfig": {
    "layout": "mobile-tabs",
    "tabs": ["content", "voice"],
    "components": {
      "passage": { "visible": true, "position": 1 },
      "question": { "visible": true, "position": 2 },
      "voicePanel": { "visible": true, "position": 3 }
    }
  }
}
```

### 5.2 모듈 타입 & 구성

**현재 구현된 모듈:**

| 코드 | 이름 | 답변 유형 | 스킬 | 채점 방식 | 특징 |
|------|------|---------|------|---------|------|
| **PRD** | Prediction | Essay (단일) | Reading | Holistic (AI) | 지문 없이 배경지식으로 예측 |
| **SHR** | Shadow Reading | Audio-Record (다중) | Speaking | Pronunciation | 음성 녹음 + 발음 점수 평가 |

**향후 추가 모듈 계획:**

| 코드 | 이름 | 설명 |
|------|------|------|
| **SCN** | Scanning | 특정 정보 찾기 (객관식) |
| **SKM** | Skimming | 대의 파악 (객관식) |
| **QAR** | Question-Answer Relationship | 추론형 문제 (객관식) |
| **SWR** | Sentence Writing | 한글 문장을 영작 (영작문) |
| **PWR** | Process Writing | 개요 → 단락 작성 → 완성 (다단계) |
| **VCB** | Vocabulary | 어휘 학습 (다양한 형식) |

### 5.3 모듈 버전 관리

**버전 정책:**
```
v1.0 (초기)
  ├─ v1.1 (문항 수정)
  │ └─ v1.2 (채점 기준 변경)
  └─ v2.0 (구조 변경, 주요 업그레이드)
```

**마이그레이션 전략:**
- **Backward Compatible**: v1.0 → v1.1은 자동 적용 (기존 레슨에 영향)
- **Breaking Change**: v1.0 → v2.0은 신규 레슨만 적용, 기존 레슨은 v1 유지
- **호환성 검증**: 버전 변경 시 기존 답변 데이터와의 호환성 검토

---

## 6. 로드맵

### 6.1 Phase 1 (현재, ~2026년 Q2)

**목표**: 핵심 Agent 기반 튜터링 엔진 완성

- ✅ ModulesPage + LessonSession 레이어 구현
- ✅ ModuleRunner 어댑터 시스템 (PRD, SHR 모듈 완성)
- ✅ ModuleOrchestratorAgent Rule 기반 의사결정 엔진
- ✅ useModuleOrchestrator 훅 및 상태 머신
- ✅ 기본 UI (지문, 문항, 피드백 패널)
- 🔄 **진행 중**:
  - CurriculumPlannerAgent Mock → Claude API 연결 (Tool Use)
  - FeedbackGenerationAgent Mock → Claude API 연결
  - 발음 평가 API (Google Speech-to-Text 또는 유사)
- 📋 **예정**:
  - 백오피스 레슨 편집 UI (수동 모듈 조합)
  - 학생 성과 대시보드 (모듈별 KPI 시각화)

### 6.2 Phase 2 (2026년 Q3)

**목표**: 모듈 라이브러리 확대 및 적응형 학습 고도화

- 추가 모듈 구현 (SCN, SKM, QAR, SWR, PWR)
- **ModuleOrchestratorAgent Rule → LLM 기반 전환:**
  - 현재 Rule 기반 구현을 Claude API Tool Use로 교체
  - 전환 전략: 규칙 엔진을 유지한 채 Claude API 결과와 병렬 실행 → 차이 로깅 → 신뢰도 기준 달성 후 전환
  - 평가 기준: Claude 결정이 Rule 결정과 95% 이상 일치 + 피드백 품질 평점 4.0/5.0 이상
- ModuleOrchestratorAgent 규칙 확대 (LLM 전환 병행):
  - 누적 참여도 추적 (세션 전 학습 기록 반영)
  - 개인 학습 패턴 인식 (예: 음성 약함 → audio 모듈 추가)
  - 동적 리플래닝 (실시간 난이도 조절)
- **Agent 관찰성(Observability) 구축:**
  - 모든 Agent 결정 이유(`reason` 필드) DB 로깅
  - 토큰 사용량 추적 (모듈별, 학생별)
  - 프롬프트 버전 추적 (어떤 버전으로 어떤 결정을 내렸는지 재현 가능)
- 백오피스 관리자 모듈 라이브러리 (CRUD)
- 모듈 버전 관리 시스템
- 성과 분석 대시보드 (교사/관리자용)

### 6.3 Phase 3 (2026년 Q4 이후, 장기 비전)

**목표**: 전사 학습 생태계 통합

- 학습자 그룹 기반 커리큘럼 (반별 맞춤형 과정)
- 동료 학습 기능 (학생 간 피드백)
- AI 튜터 성격 커스터마이징 (Pickle의 톤/스타일 조정)
- 멀티모달 입출력 (비디오 분석, 이모지 감정 표현)
- 학습 게임화 (배지, 랭킹, 미션)
- 부모/관리자 정기 리포트 (학습 진행도 + 추천 액션)

---

## 7. 성공 지표

### 7.1 사용자 지표

| 지표 | 목표 | 측정 방법 |
|------|------|---------|
| **월활동사용자(MAU)** | 1,000명 (3개월) | Google Analytics |
| **일완료율** | 7일 내 1회 이상 진행 | ModuleHistory 기록 |
| **모듈완료율** | 시작한 레슨 80% 이상 완료 | completedResults 개수 |
| **평균세션길이** | 15–20분 | session 시간 기록 |
| **재방문율** | 주 1회 이상 40% 이상 | 사용자 방문 빈도 |

### 7.2 학습 효과 지표

| 지표 | 목표 | 측정 방법 |
|------|------|---------|
| **평균정답률** | 모듈별 점수 >= 70% | ModuleResult.score 평균 |
| **스키밍난이도상승** | 12주 내 CEFR 1단계 상승 | moduleSequence 난이도 추이 |
| **발음점수개선** | SHR 모듈 점수 주간 +5% | KpiResult (pronunciation) 추이 |
| **에세이지표** | 7주 내 에세이 점수 +20% | PRD 모듈 점수 추이 |
| **재학습필요율** | 모듈실패(score<50%) <= 20% | score < 50 모듈 비율 |

### 7.3 기술 지표

| 지표 | 목표 | 측정 방법 |
|------|------|---------|
| **API응답시간** | < 200ms (p95) | CloudWatch, New Relic |
| **페이지로딩시간** | < 2초 (desktop), < 3초 (mobile) | Lighthouse, Core Web Vitals |
| **시스템가용성** | >= 99.5% | 서버 모니터링 |
| **오류율** | < 0.1% | Error tracking (Sentry) |
| **Agent응답생성시간** | < 5초 (Claude API 호출) | 로그 분석 |
| **토큰 사용량** | 모듈당 평균 < 3,000 토큰 | Anthropic API 사용량 대시보드 |
| **Agent Fallback 발생률** | < 2% (전체 결정 대비) | Agent 결정 로그 분석 |
| **스트리밍 첫 토큰 지연** | < 1.5초 (TTFT) | SSE 응답 시작 시간 측정 |

---

## 8. 리스크 & 완화 전략

### 8.1 기술 리스크

| 리스크 | 영향 | 확률 | 완화책 |
|--------|------|------|--------|
| **Claude API 비용 초과** | 서비스 비용 증가 | M | 요청 배치 처리, 캐싱 (통일 피드백), 토큰 사용 제한, claude-haiku 우선 사용 |
| **음성 평가 API 정확도 저하** | 사용자 불만족 | M | 대체 API 확보 (Azure Speech, Deepgram) |
| **DB 성능 저하** (대규모 사용자) | 응답 속도 저하 | M | 파티셔닝, 인덱싱, 읽기 복제본 추가 |
| **프론트엔드 상태 관리 복잡도** | 유지보수 어려움 | M | Zustand 전환, 테스트 커버리지 강화 |
| **프롬프트 인젝션 공격** | 비정상 AI 응답, 보안 침해 | L | 학생 입력을 user role로 분리, 입력 길이 제한, 특수 패턴 이스케이프 |
| **AI 환각(Hallucination)** | 잘못된 피드백 제공 → 학생 혼란 | M | Tool Use 강제로 출력 형식 구조화, guardrail 레이어 검증, 교사 피드백 검토 루프 |
| **컨텍스트 윈도우 초과** | Agent 오작동, 세션 중단 | L | Rolling Window + 점진적 요약 전략 적용 (4.4.8 참고) |

### 8.2 비즈니스/교육 리스크

| 리스크 | 영향 | 완화책 |
|--------|------|--------|
| **AI 튜터 오답** (잘못된 피드백 제공) | 학생 신뢰도 저하 | 교사 검토 프로세스, A/B 테스트, 피드백 평가 |
| **모듈 과다 설계** (학생 피로) | 중도 포기율 증가 | 모듈 시간 예측 검증, 사용자 테스트 |
| **이종 브라우저 호환성** | 일부 사용자 배제 | Cross-browser 테스트, graceful degradation |

### 8.3 운영 리스크

| 리스크 | 대응 |
|--------|------|
| **모듈 버전 충돌** (기존 레슨과 불호환) | 버전 관리 정책 엄격 실행, 마이그레이션 테스트 |
| **학생 데이터 손실** | 정기 백업 (1일 1회), replication |
| **배포 중단** | CI/CD 자동화, Blue-Green 배포 |

---

## 9. 부록

### 9.1 스크린샷 & 데모

**화면 1: 레슨 진입 & 로딩**
```
┌─────────────────────────────────────────┐
│ ← Lesson 1                      Pickle AI │
│ Digital Commerce                        │
├─────────────────────────────────────────┤
│                                          │
│      [로딩 애니메이션]                  │
│      학습을 불러오는 중...              │
│                                          │
└─────────────────────────────────────────┘
```

**화면 2: 모듈 진행 (PRD 모듈)**
```
┌─────────────────────────────────────────┐
│ ← Lesson 1                      Pickle AI │
│ What do you think the passage is about? │
├─────┬─────────────────────────────────┤
│ 지문│ 예측할 때 어떤 전략을 사용했어? │
│     │ (textarea with AI guidance)     │
│     │                                 │
│     │ [제출] [취소]                   │
├─────┴─────────────────────────────────┤
│ [Passage] [Questions] [Feedback]       │
└─────────────────────────────────────────┘
```

**화면 3: 피드백 및 성과카드**
```
┌─────────────────────────────────────────┐
│ ✓ 좋은 예측입니다!                      │
│                                          │
│ 강점: 배경지식 활용이 뛰어났어요.       │
│ 개선: 더 구체적인 근거를 제시하면 좋겠어│
│                                          │
│ [이해했어요] [더 설명해줘]               │
├─────────────────────────────────────────┤
│ 모듈 완료!                              │
│ Prediction - 정답률 85%                 │
│ 예측 타당성: 8/10                       │
│ [다음 모듈 →]                            │
└─────────────────────────────────────────┘
```

### 9.2 기술 스택 상세

**Frontend:**
- Next.js 15 (App Router)
- React 19
- TypeScript 5
- Tailwind CSS 3 + shadcn/ui
- Zustand (향후 상태 관리)
- Axios (API 호출)

**Backend:**
- NestJS 10
- TypeScript
- PostgreSQL 15
- Prisma ORM
- Anthropic Claude API (claude-haiku-4-5-20251001)
- Google Speech-to-Text (음성 평가)

**Infra:**
- Docker + Docker Compose
- GitHub Actions (CI/CD)
- AWS EC2 (또는 Vercel Next.js 배포)
- AWS RDS (PostgreSQL)
- AWS S3 (미디어 저장소)

### 9.3 용어 정의

| 용어 | 정의 |
|------|------|
| **LessonPlan** | 지문 1개 + 모듈 시퀀스로 구성된 학습 계획 |
| **PlannedModule** | 레슨 플랜 내의 개별 모듈 슬롯 (order, role, estimatedMinutes 포함) |
| **ModuleData** | 실제 학습 콘텐츠 (passage + questions) |
| **ModuleResult** | 모듈 완료 결과 (score, kpis, completedAt) |
| **OrchestratorContext** | Agent의 입력 컨텍스트 (LessonPlan + ModuleData + LearnerState + ChatHistory) |
| **OrchestratorToolCall** | Agent의 출력 (tool 호출 결정 + 이유) |
| **LearnerState** | 학습자의 현재 상태 (답변, 채팅, 참여도, 유휴시간) |
| **PassageAnalysis** | CurriculumPlannerAgent가 생성한 지문 분석 결과 |
| **ModulePedagogyProfile** | 모듈의 교수법 특성 (채점방식, 피드백스타일 등) |
| **ContentConfig** | UI 배치 정책 (어댑터가 모듈에 맞춰 정의) |
| **KpiResult** | 모듈별 성과 지표 (label, value, unit, description) |
| **Passage** | 학습 자료 (지문, 기사 등) |
| **Adaptation** | 실시간 학생 상태에 따른 UI/콘텐츠/피드백 동적 변경 |
| **Rule-Based Decision** | 우선순위 규칙에 따른 의사결정 (현재 구현) |
| **Tool Use** | Anthropic Claude API의 구조화 출력 기능 — Agent가 자연어 대신 정해진 도구 호출 형식으로만 응답하도록 강제 |
| **Streaming (SSE)** | Server-Sent Events 방식으로 Claude API 응답을 토큰 단위로 실시간 전송 — 피드백 생성 대기 UX 개선 |
| **Guardrail** | Agent 출력 유효성 검증 레이어 — 잘못된 Tool Call 반환 시 안전한 기본 액션으로 대체 |
| **Prompt Snapshot** | 세션 시작 시 프롬프트 내용을 고정하는 방식 — 세션 중 프롬프트 변경이 진행 중인 학습에 영향을 미치지 않도록 보장 |
| **Rolling Window** | 컨텍스트 윈도우 관리 전략 — 최근 N개 메시지만 유지하여 토큰 초과 방지 |
| **TTFT (Time To First Token)** | 스트리밍 응답에서 첫 번째 토큰이 도달하는 데 걸리는 시간 — 체감 응답 속도의 핵심 지표 |

---

## 개정 이력

| 버전 | 작성자 | 작성일 | 변경 사항 |
|------|--------|--------|---------|
| 1.0 | AI Agent | 2026-04-01 | 초기 문서 작성 |
| 1.1 | AI Agent | 2026-04-01 | Agent 기술 개발 관점 검토: 오류 수정(OpenAI→Anthropic), Tool 스키마·프롬프트 구조·컨텍스트 관리·에러 처리·스트리밍 전략 추가, LLM 전환 로드맵·리스크·지표 보강 |

---

**문서 관리:**
- 소유: 기술 팀
- 검토 주기: 월 1회 (또는 주요 변경 시)
- 최종 승인: 프로덕트 매니저 + 기술 리드

