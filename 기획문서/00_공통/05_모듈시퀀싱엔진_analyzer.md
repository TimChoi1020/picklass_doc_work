---
title: 모듈 시퀀싱 엔진 (analyzer)
version: v1.1
updated: 2026-07-18
owner_service: common
master_origin: v0.21 §10.1~§10.14
depends_on:
  - ./04_통합모듈시스템.md
  - ../Studio/기획서_콘텐츠생성엔진.md
---

> 📄 본 문서는 통합 기획서 v0.21(→ `_archive/`)에서 분리된 문서이다. 본문 내 **§번호는 구 통합 기획서 기준**이며, §번호 → 신규 문서 매핑은 [README](../README.md)의 매핑표를 참조한다.

## 10. 모듈 시퀀싱 엔진 (Module Sequencing Engine)

> CurriculumPlannerAgent의 핵심. §9에서 생성된 지문 + 학습 목표 + 시간 예산을 입력으로 받아 **LessonPlan(모듈 시퀀스)**을 자동 생성한다. Studio(§8) 레슨 편집 UX에서 호출된다.
>
> ⚠️ **재정의 (v0.10)**: 본 장의 "CurriculumPlannerAgent"는 **추상 명칭**으로 유지하되, **실제 구현체는 별도 마이크로서비스 `analyzer` 서버**이다 (✅ 구현 완료). LLM Tool Use 기반 Agent로의 향후 전환 가능성은 §10.9 참조.

### 10.1 개요 및 책임 범위

- **엔진 미션**: "주어진 지문과 15분으로 최적의 교수 설계 산출"
- **3-모드**: 자동 생성 / 수동 편집 / 하이브리드
- **경계**: LessonPlan 생성까지. 실시간 학습 중 의사결정은 §12 ModuleOrchestratorAgent가 담당.

### 10.1.1 실제 구현체 — `analyzer` 서버 *(✅ 구현 완료)*

| 항목 | 내용 |
|---|---|
| 서비스 명 | `analyzer` (별도 마이크로서비스) |
| API 엔드포인트 | `POST /lesson-plan` (NestJS API Gateway → analyzer 서버 호출) |
| 호출 주체 | **Studio + Tutoring 양쪽** (단일 진실 원천 → 결과 동등성 parity 보장) |
| 호출 위치 | studio: `/class/lesson-setup/[passageId]`, `/course-hub` / tutoring: `/class/lesson-setup/custom` (나만의 수업) |
| 핵심 알고리즘 | **KPI 집합 커버(Set Cover) + Stage Diversity + 인지 부하 곡선(cognitiveLevel) + 시간 제약** |
| 폴백 정책 | Gemini 폴백 **폐지(2026-04-23)** — analyzer 장애 시 명확한 에러 응답 |
| 데이터 소스 | 운영 Supabase DB (폴리필·목업 모두 제거 완료) |

**요청 스키마 (실제)**

```typescript
// POST /lesson-plan
interface LessonPlanAnalyzerRequest {
  passage_level: string;            // CEFR (예: 'B1', 'C1')
  selected_kpi_codes: string[];     // KPI_CATEGORY 코드 배열
  duration_min: 15 | 20 | 25 | 30;  // 4개 옵션
  passage_text?: string;            // optional, 향후 정밀도 향상용
  passage_id?: number;              // optional, 백엔드가 DB에서 level 조회 시 사용
}
```

**응답 스키마 (실제)**

```typescript
interface LessonPlanResponse {
  module_sequence: PlannedModule[];
  total_estimated_minutes: number;
  sequencing: {
    coverage_score: number;          // KPI 커버리지 점수
    stage_distribution: { before: number, middle: number, after: number };
    fallback_modules: string[];      // 보충된 모듈 (KPI 부족 시)
  };
}
```

### 10.1.2 알고리즘 4단계 *(✅ 구현 완료)*

```
[입력]
  passage_level + selected_kpi_codes + duration_min
        ↓
[1단계] 모듈 풀 필터링 (Module Filtering)
  - suitableLevelMin ≤ passage_level ≤ suitableLevelMax
  - 도메인 매칭 (목적에 부합하는 영역)
        ↓
[2단계] KPI 커버리지 + 시간 제약 시퀀싱 (Set Cover Knapsack)
  - 신규 KPI를 추가로 커버하는 모듈을 우선 선택
  - 이미 커버된 KPI만 담당하는 모듈은 우선순위 하향
  - sum(estimatedMinutes) ≤ duration_min 제약
  - 시간 초과 시 신규 KPI 기여도 낮은 모듈부터 제거
        ↓
[3단계] 단계 결정 + 정렬 (3순위 체계)
  - 1순위: classBefore/Middle/After 플래그 → Stage 결정
  - 2순위: cognitiveLevel ASC → Stage 내 인지 부하 순서
  - 3순위: priority DESC → 동일 cognitiveLevel 내 운영 우선순위
        ↓
[4단계] prerequisites / incompatibleWith 제약 검증·보정
        ↓
[출력] LessonPlan (moduleSequence)
```

> 📎 알고리즘 상세 의사코드는 `studio.picklass.com3/docs/20260421_지능형 수업 설계 자동화 로직.md` 참조.

### 10.2 입력 스키마

```typescript
interface PlanningRequest {
  passageAnalysis: PassageAnalysis;     // §9.4 출력
  learningGoal: string | GoalTag[];     // 자연어 또는 구조화된 목표
  targetLevel: number;                  // CEFR 1~18
  timeBudget: number;                   // 분
  availableModules: ModulePlanningMeta[]; // 라이브러리 카탈로그
  learnerProfile?: {                    // 선택 (누적 학습 이력)
    recentModules: string[];            // 중복 회피
    weakSkills: string[];               // 약점 보강
    avgCompletionRate: number;
  };
}
```

### 10.3 필터링 알고리즘 (Module Filtering)

#### 10.3.1 레벨 적합성
- `suitableLevels.min ≤ targetLevel ≤ suitableLevels.max` 교집합 통과한 모듈만 선택

#### 10.3.2 교수 목표 매칭
- learningGoal → module roles 매핑 테이블
- 예: "표현력 향상" → output 역할 가산

#### 10.3.3 양립불가 제거
- `incompatibleWith` 제약 충족 모듈만 잔존

#### 10.3.4 선행조건 검증
- `prerequisites` 배열의 모듈이 이미 포함될 수 있는 경우만 허용

#### 10.3.5 학습자 이력 반영
- 최근 사용 모듈은 가중치 감점 (중복 회피)
- 약점 스킬 매칭 모듈은 가중치 가산

### 10.4 시퀀싱 알고리즘 (Module Sequencing)

#### 10.4.1 역할 순서
- 기본 순서: warming → passage-use → practice → output
- 예외: Speaking 단독 세션에서는 warming 생략 가능

#### 10.4.2 인지 부하 곡선 최적화
- Bloom's level을 완만히 상승시키도록 배치
- 급격한 난이도 점프 방지

#### 10.4.3 지문 노출 일관성
- 한 레슨 내에서 passage exposure가 before → during → after 순으로 진행되도록 검증
- 역전(after 후 before 노출 금지)

#### 10.4.4 문항 유형 다양성
- 같은 `answerType` 모듈 3개 이상 연속 배치 방지
- 강제: essay 연속 2회 후 다른 유형 삽입

### 10.5 시간 예산 조정

```
total_estimated = Σ moduleSequence[i].estimatedMinutes
```

- total > timeBudget + 10%: 우선순위 낮은 모듈 제거 or 대체
- total < timeBudget − 10%: 심화/반복 모듈 추가
- 반복(최대 3회) 후 수렴 실패 시 강사 알림

### 10.6 강사 오버라이드 UI (§8.5 연동)

#### 10.6.1 Planner 결과 시각화
- 타임라인(바 차트) + 각 모듈 Rationale 팝오버

#### 10.6.2 드래그 기반 모듈 추가/삭제/순서 변경

#### 10.6.3 조건부 규칙 편집
- 예: "이전 모듈 점수 > 90 → 다음 모듈 스킵"
- 예: "정답률 < 50 → 재시도 모듈 자동 삽입"

#### 10.6.4 실시간 제약 위반 경고
- incompatible/missing prereq/시간 초과 인라인 표시

### 10.7 출력 스키마: LessonPlan

```json
{
  "lessonId": "lesson-123",
  "passageId": "passage-456",
  "passageAnalysis": { ... },
  "moduleSequence": [
    {
      "order": 1,
      "moduleCode": "PRD",
      "role": "warming",
      "passageExposed": false,
      "estimatedMinutes": 3,
      "rationale": "B2 레벨 학습자의 배경지식 활성화를 위해 예측하기 모듈 우선 배치"
    },
    { "order": 2, "moduleCode": "RRD", ... },
    { "order": 3, "moduleCode": "SWR", ... }
  ],
  "totalEstimatedMinutes": 12,
  "generatedAt": "2026-04-17T00:00:00Z",
  "plannerVersion": "v1.0",
  "editedByTeacher": false
}
```

### 10.8 검증 및 품질 관리

- **자동 검증**: 시간 합산, 역할 순서, 제약 준수 일괄 체크
- **A/B 생성**: 같은 입력에 대해 2안 제시 (10.12 API 참조)
- **학습 결과 피드백 루프**: 완료율·정답률 기반 모듈 가중치 주기적 조정

### 10.9 AI 호출 최적화

- 필터링(10.3)은 로컬 룰 기반 (비용 0)
- 시퀀싱(10.4)은 LLM Tool Use로 구조화 출력
- 프롬프트 스냅샷 저장 → 재현성 보장
- 캐싱: 동일 입력 해시 1회 이상 → 결과 재사용 (TTL 24h)

### 10.10 버전 관리

- LessonPlan 수정 이력 (Planner 자동 / 강사 편집 구분)
- 모듈 버전 변경 시 기존 LessonPlan 영향 평가 (§13.11 연동)

### 10.11 데이터 모델 (요약)

```sql
lessons (id, passage_id, learning_goal, target_level, module_sequence JSONB,
         total_minutes, created_at)
lesson_plans (id, lesson_id, version, generated_by, edited_by_teacher, payload JSONB)
planned_modules (id, lesson_plan_id, order, module_code, role, passage_exposed,
                 estimated_minutes, rationale)
planner_runs (id, request JSONB, response JSONB, tokens_used, cost,
              duration_ms, status, created_at)
```

### 10.12 API 엔드포인트

```
POST /api/lesson/plan              (자동 생성)
POST /api/lesson/plan/stream       (SSE)
POST /api/lesson/{id}/plan/edit    (강사 편집 저장)
POST /api/lesson/{id}/plan/ab      (A/B 제안)
GET  /api/lesson/{id}/plan/history (버전 이력)
```

### 10.13 KPI 및 관찰성

| 지표 | 목표 |
|---|---|
| 자동 수용률 (수정 없이 승인) | ≥ 50% |
| 평균 제안 시간 | < 5초 |
| 토큰 사용량 (1회 제안당) | < 3,000 |
| 학생 최종 완료율 (Planner 품질의 역지표) | ≥ 70% |
| 시간 예산 수렴률 (±10% 내) | ≥ 95% |

### 10.14 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 모듈 라이브러리 빈약 | 시퀀싱 다양성 저하 | v1.3 기준 활성 30개 모듈 운영, Phase 2에서 미작성 15개 + 추가 5개 모듈 단계적 합류 (§13.1 레지스트리 참조) |
| LLM 비용 폭증 | 수익성 | 룰 기반 1차 필터 + LLM 최종 선별, 캐싱 |
| 강사 불신 | 자동 수용률 저하 | Rationale 상세 제공, 수정 자유도 보장, A/B 제안 |
| 레벨·제약 충돌 | 유효 모듈 0개 | 완화 규칙 (제약 가중치화), 강사 알림 후 수동 모드 전환 |
| 버전 호환성 | 기존 LessonPlan 파손 | 모듈 버전 고정 참조(§9.7과 같은 패턴) |



### 10.15 Speaking 콘텐츠 생성 모드 (3종) *(→ 이동)*

> 📎 본 절은 [speaking/기획서_Speaking대화엔진.md](../Speaking/기획서_Speaking대화엔진.md)로 이동되었다.

---
