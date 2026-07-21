---
title: AI 튜터링 엔진 (Agent 아키텍처)
version: v1.1
updated: 2026-07-18
owner_service: tutoring
master_origin: v0.21 §12 (§12.5 제외)
depends_on:
  - ../00_공통/05_모듈시퀀싱엔진_analyzer.md
  - ../Speaking/기획서_Speaking대화엔진.md
---

> 📄 본 문서는 통합 기획서 v0.21(→ `_archive/`)에서 분리된 문서이다. 본문 내 **§번호는 구 통합 기획서 기준**이며, §번호 → 신규 문서 매핑은 [README](../README.md)의 매핑표를 참조한다.

## 12. AI 튜터링 엔진 (Agent 아키텍처)

### 12.1 2단계 Agent 구조 개관

```
CurriculumPlannerAgent (Macro, 1회 호출)
    ↓ LessonPlan 생성
ModuleOrchestratorAgent (Micro, 매 상호작용)
    ↓ 4요소 통제 + Tool Call
[UI/콘텐츠/피드백/다음액션]
```

### 12.2 CurriculumPlannerAgent

- **입력**: PassageData + LearningGoal + TargetLevel + 시간제약
- **도구**: analyzePassage, filterModules, sequenceModules
- **출력**: LessonPlan (moduleSequence)

### 12.3 ModuleOrchestratorAgent

#### 12.3.1 4요소 통제
UI 노출 / 콘텐츠 선택 / 피드백 타이밍 / 다음 액션.

#### 12.3.2 Rule-Based 우선순위 (9단계)
1. IdleCheck (120초 무응답) → checkEngagement
2. Disengaged (참여도 낮음 + 180초) → signalReplan
3. RepeatWrongAnswer (2회+ 오답) → 재시도 유도 + "힌트 보기" 버튼 노출 강화 (⚠️ v1.1 — **자동 힌트 레벨 상승 폐지**, B1/B4 학습자 요청형 3레벨 정합)
4. HolisticFeedback (essay 제출) → provideFeedback(AI)
5. PronunciationFeedback (audio 제출) → provideFeedback(발음)
6. WrongAnswerFeedback (객관식 오답) → provideFeedback
7. CorrectAnswer (정답) → presentQuestion or celebrate
8. InitialEntry (최초 진입) → showPassage or presentQuestion
9. PresentNextQuestion (기본) → presentQuestion

#### 12.3.3 Tool Use 스키마
showPassage / presentQuestion / provideFeedback / checkEngagement / signalReplan / celebrate

### 12.4 FeedbackGenerationAgent (텍스트 SSE)

- Essay/Audio 피드백 스트리밍 (Server-Sent Events)
- 첫 토큰 지연(TTFT) < 1.5초 목표
- 실시간 타이핑 효과로 체감 지연 감소

### 12.5 SpeakingConversationAgent *(→ 이동)*

> 📎 본 절은 [speaking/기획서_Speaking대화엔진.md](../Speaking/기획서_Speaking대화엔진.md)로 이동되었다. Orchestrator ↔ SpeakingConversationAgent 핸드오프 인터페이스는 §14.10.2 참조.

### 12.6 프롬프트 엔지니어링

**시스템 프롬프트 계층** *(⚠️ v1.1 재정의 — feedbackStyle 제거)*:
1. 역할 정의 (You are Pickle…)
2. 교수법 지시문 (pedagogyInstruction, 모듈별 동적 삽입) — **피드백 스타일 통합**
3. 컨텍스트 스냅샷 (구조화, 자연어 최소화)
4. 도구 목록 (Tool Use 형식)

> ✅ **B5 확정 (2026-07-18)**: 별도 `feedbackStyle` 계층을 제거하고 **pedagogyInstruction 단일 소스로 통합**했다(구현 정합).

**프롬프트 스냅샷**: 세션 시작 시 고정, 진행 중 변경 미적용 → 일관성 보장.

### 12.7 컨텍스트 윈도우 관리

| 전략 | 적용 시점 | 방법 |
|---|---|---|
| Rolling Window | 항상 | 최근 N=20 메시지 유지 |
| 점진적 요약 | 메시지 > 20 | 오래된 블록 Claude 요약 |
| 구조화 컨텍스트 | 매 요청 | LearnerState 구조체 전달 |

**토큰 예산**: System ~500 / PassageAnalysis ~200 / History ~1,500 / 답안 ~300 / Tools ~400 → 합계 ~2,900

### 12.8 에러 처리 · Fallback · Guardrail

| 시나리오 | Fallback |
|---|---|
| API 타임아웃 (>5초) | Rule-Based 자동 전환 |
| Rate Limit | 지수 백오프 (1s→2s→4s, 최대 3회) |
| 잘못된 Tool Call | 스키마 검증 실패 → 기본 액션 |
| 빈 응답/파싱 실패 | 이전 결과 재사용 or 재시도 안내 |
| 컨텍스트 초과 | 강제 Rolling Window 축소 |

### 12.9 프롬프트 인젝션 방어

- 학생 입력을 user role로 분리 (system에 직접 삽입 금지)
- 입력 길이 제한: essay 2,000자 / 채팅 500자
- 특수 패턴("ignore previous instructions", XML 태그) 이스케이프

---

