---
title: Writing 영역 — 서술형·첨삭 (P1-1)
version: v1.0
updated: 2026-07-19
owner_service: tutoring
gap_code: G4
priority: P1 (파일럿 직후 — 단, 어학원 1순위 수요라 P0 후반 착수 검토)
source_doc: 이퓨쳐/P1-1_Writing영역_상세기획.md
depends_on:
  - ./기획서_AI튜터링엔진.md
  - ../00_공통/04_통합모듈시스템.md
  - ../Studio/기획서_강사검수UX.md
  - ../Studio/기획서_성장리포트MVP.md
---

> 📋 **기획 단계 (P1 우선).** `이퓨쳐/P1-1_Writing영역_상세기획.md` 명세를 기획문서 구조로 이식.

## 개요

Writing/서술형을 **부가기능이 아니라 코어 영역**으로 완성한다. 문항 학습 + 첨삭(교정) + 진단 신호의 **3중 역할**.

**시장 근거**: 어학원 세그먼트 AI 작문·첨삭 1순위 50%. 중등 서술형(수행평가) 급증. "이상하게 쓰는 애 = 가짜로 이해한 애"(현장 인터뷰) → 서술형이 이해도 진단 신호.

**톤 원칙**: "AI 티" 없이 — 첨삭은 교정 위주, 완벽한 재작성 강요 금지.

---

## Writing 모듈 현황 (§13.2)

| 모듈 코드 | 이름 | 상태 |
|---|---|---|
| SWR | Sentence Writing | ✅ 완료 |
| UNS | Unscramble | 📋 P1 |
| CPW | Copy Writing | 📋 P1 |
| SCP | Sentence Completion | 📋 P1 |
| PPR | Paraphrasing | 📋 P1 |
| PWR | Process Writing | 📋 P1 |
| SMW | Summary Writing *(구 PWT 분리·재명명, 신규)* | 📋 P1 |

> ⚠️ WFT·WDN은 Writing 아님 → Vocabulary 계열

---

## 기능 요구사항 (FR)

| # | 요구사항 | 판정 |
|---|---|---|
| FR-1 | 모듈 등록 — Writing 7종(`AiModule` 필드 세팅, §13 시퀀싱 편입) | 확장 |
| FR-2 | 쓰기 실행 — 학생이 문장/문단/에세이 작성 (answerType = sentence-write / essay) | 확장 |
| FR-3 | 첨삭 생성 — 홀리스틱 피드백(강점/개선, §12.4 ✅) + 문장 단위 오류 교정 (SSE 스트리밍) | 재사용+확장 |
| FR-4 | 단계적 쓰기 (PWR) — 개요→초안→수정 단계 지원 | 신규 |
| FR-5 | 진단 신호화 — 서술형 수행 지표(정확성·완성도·이해 반영도)를 P0-2 리포트·진단으로 전달 | 신규 |
| FR-6 | 검수·톤 — 첨삭 결과를 P0-4 검수 큐로, 톤 프리셋(교정 위주/격려형) | 재사용 |
| FR-7 | 채점 루브릭 — 6항목(내용·구조·담화·문법·어휘·맞춤법) 정량 스코어 → 리포트·P1-3 입력 | 신규 |

---

## 첨삭 엔진 구조

```
학생 작성 텍스트
  → 홀리스틱 피드백 (§12.4 FeedbackGenerationAgent 재사용)
      강점 / 개선 사항 (에세이 전체)
  → 문장 단위 교정
      오류 지적(span) + 수정 제안 + 근거
  → 루브릭 스코어 (내용·구조·담화·문법·어휘·맞춤법)
  → P0-4 검수 큐 → 승인 → 학생에게 전달
```

---

## 데이터 모델 (요약)

| 엔티티 | 주요 필드 |
|---|---|
| `WritingSubmission` | studentId, moduleCode, text, level, rubricScore, createdAt |
| `WritingFeedback` | submissionId, holistic{strengths, improvements}, corrections[{span, issueType, suggestion}], reviewStatus |
| 진단 지표 | comprehensionSignal(이해 반영도), completeness, accuracy → 리포트/진단 |

---

## 화면·플로우

```
[Tutoring] Writing 모듈 진입
  → 문항 제시 (레벨별) → 학생 작성 (sentence-write / essay)
  → [제출] → 첨삭 생성 (홀리스틱 + 문장 교정, SSE 스트리밍)
  → 학생: 교정 확인 → (PWR) 재작성
  → 강사: 검수 큐(P0-4) → 승인
  → 서술형 지표 → 리포트(P0-2) / Achievement Test(P1-3)
```

---

## 기존 자산 연계

| 자산 | 상태 | 연계 방식 |
|---|---|---|
| SWR 모듈 | ✅ | P1 확장 기반 |
| FeedbackGenerationAgent essay (§12.4) | ✅ | 홀리스틱 피드백 재사용 |
| SSE 스트리밍 | ✅ | 첨삭 결과 스트리밍 |
| `AiModule` 시퀀싱 | ✅ | Writing 모듈 등록 |
| P0-4 검수 워크플로우 | P0 신규 | 첨삭 검수 연동 |

---

## 개발팀 확인 필요 (S5)

> S5: Writing 모듈을 Phase 2에서 P1으로 앞당길 때 의존성(모델·채점 루브릭)은? ([09_로드맵 S5](../00_공통/09_로드맵_리스크_보안.md) 참조)
