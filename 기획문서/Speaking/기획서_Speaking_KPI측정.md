---
title: Speaking KPI 측정 체계
version: v1.1
updated: 2026-07-18
owner_service: speaking
master_origin: v0.21 §14.6
depends_on:
  - ./기획서_Speaking.md
  - ../00_공통/04_통합모듈시스템.md
  - ../00_공통/06_진단평가_PPD.md
---

> 📄 본 문서는 [기획서_Speaking.md](./기획서_Speaking.md) §14.6에서 분리된 KPI 측정 체계 전용 문서이다(2026-07-18 분할). 5축·70 KPI 레지스트리·운영 모델·벤치마크·모듈×KPI 매핑을 다룬다. Pick-Speak 9모듈·5축 색상 토큰 등 학습 흐름 정의는 본편 [§14.4](./기획서_Speaking.md) 참조.

### 14.6 KPI 측정 체계 *(⚠️ v0.10 운영 정합화)*

> **확장 요약**: v0.8까지는 Speaking FRT(Free Talking, 구 CMP) 전용 5지표였으나, v0.9에서는 **40개 모듈을 아우르는 70 KPI 코드 레지스트리**로 확장. v0.12에서 모듈은 29개로 정리되어 일부 KPI는 고아 상태(§13.1.6 폐기 모듈 KPI 재할당 후속 과제). 8대 역량 분류, 2-depth 계층(역량→하위목표→측정항목), 측정 도구 구분, 레벨별 목표 벤치마크, 모듈×KPI 매핑을 통합.

> ⚠️ **v1.3 재정의 — 절대단위 원칙** (구현 정합): 학습자·B2B·엔진이 보는 숫자를 **절대단위(%·WPM·오류/문장·문장수)로 통일**하고 **100점 환산은 폐기**한다. KPI는 **외부 7 + 내부 8 계층**으로 분리(외부 7 = 5축 + 듣기 누적 분 + 섀도잉 카운트). 피드백 탭 시각화는 **Q/Q/G 3축**(Quality 정확성 / Quantity 양 / Growth 5축 절대변화량 — [기획서_Speaking_앱UX §14.9.0 피드백 탭](./기획서_Speaking_앱UX.md)). 서브웨이 신규 KPI 2개(듣기 누적 분·섀도잉 카운트) 정식화. → 기존 70 KPI 레지스트리와의 계층 정합은 후속 과제.
>
> ⚠️ **v0.10 정합화**: 70 KPI 레지스트리(📋 기획)는 유지하되, **실제 운영 단위는 `KPI_CATEGORY` 코드그룹의 `extraData.goal` 멀티셀렉트(✅ 구현 완료)**임을 명시. 두 체계의 매핑·운용 방식을 §14.6.0에 정의.
>
> 📎 **상세 컴패니언 문서**: 70 KPI 전체 속성 및 모듈×KPI 매핑 매트릭스는 `Picklass_KPI_체계_상세.md` 참조.
>
> 📊 **엑셀 카탈로그 연동**: `Picklass_모듈_카탈로그.xlsx` (v0.4)의 시트 — `KPI 코드` / `모듈×KPI 매핑` / `레벨 벤치마크` 3종 참조.

#### 14.6.0 운영 모델 — `KPI_CATEGORY` 코드그룹 *(✅ 구현 완료)*

실제 시스템에서는 KPI를 **백오피스 코드그룹 `KPI_CATEGORY`**로 관리하며, 사용자(강사·학생)에게 노출되는 단위는 **`extraData.goal` 텍스트**이다.

```
[코드그룹] KPI_CATEGORY
  ├─ code: 'FLUENCY_RATE'
  │   extraData: { goal: '말하기 유창성 향상' }
  ├─ code: 'PRAGMATICS'
  │   extraData: { goal: '화용적 의미 이해' }
  ├─ code: 'PRONUNC'
  │   extraData: { goal: '발음 정확도' }
  └─ ...
```

**운영 흐름**

```
[강사·학생 UI]
  goal 텍스트 멀티셀렉트 (예: "말하기 유창성", "화용성")
        ↓
[Studio·Tutoring 프론트]
  내부적으로 goal → code 변환 (FLUENCY_RATE, PRAGMATICS)
        ↓
[NestJS API → analyzer 서버]
  selected_kpi_codes: ['FLUENCY_RATE', 'PRAGMATICS']
        ↓
[저장]
  courses.kpi_codes TEXT[] = ARRAY['FLUENCY_RATE', 'PRAGMATICS']
  ai_modules.selectedKpiCodes = [...]
```

**운영 데이터 위치 (✅ 구현 완료)**

| 항목 | 위치 |
|---|---|
| KPI 마스터 정의 | 백오피스 시스템관리 → 코드관리 → `KPI_CATEGORY` 코드그룹 |
| 모듈 ↔ KPI 매핑 | `ai_modules.selectedKpiCodes: TEXT[]` |
| 과정 ↔ KPI 매핑 | `courses.kpi_codes: TEXT[]` (2026-04-23 신설, NOT NULL DEFAULT '{}') |
| analyzer 시퀀싱 입력 | `selected_kpi_codes: string[]` |
| 학생 화면 표시 | `goal` 텍스트 (코드 미노출) |

> **❌ 폐기**: 이전의 `ai_modules.learning_objectives TEXT[]` 컬럼은 2026-04-22 마이그레이션으로 DROP. 모듈 학습 목표는 KPI_CATEGORY 코드 매핑으로 대체.

#### 14.6.0+ KPI_CATEGORY ↔ 70 KPI 레지스트리 정합화

| 항목 | KPI_CATEGORY 운영 모델 | 70 KPI 레지스트리 (기획) |
|---|---|---|
| 단위 | 코드 + goal 텍스트 (1:1) | 코드 + 8대 역량 + 2-depth 계층 |
| 코드 수 | (📋 운영 코드 수 별도 집계) | 70 (`Picklass_KPI_체계_상세.md` §2) |
| 사용자 노출 | goal 텍스트 멀티셀렉트 | (현 시점 미노출) |
| 시퀀싱 호환성 | `selected_kpi_codes` 직접 전달 | 동일 코드 체계 사용 시 호환 |
| 정합화 후속 | KPI_CATEGORY 운영 코드를 70 레지스트리와 1:1 매핑하는 백필 작업 필요 | `picklass-backoffice/docs/ai-modules/20260423_KPI기반_시퀀싱_작업목록.md` §10 (P1 미착수) |

#### 14.6.1 Speaking 5지표 (FRT Free Talking 전용 · v0.12 코드 갱신)

Free Talking(FRT) 세션의 사용자 피드백·리포트에 우선 적용되는 5지표 (기존 v0.8 정의 유지, v0.12에 코드만 갱신).

| 지표 | 정의 (개략) | 측정 방법 |
|---|---|---|
| 유창성 (Fluency) | 발화 속도, 끊김 횟수 | WPM + 침묵 구간 분석 |
| 정확성 (Accuracy) | 문법·어휘 오류율 | AI 분석 |
| 복잡성 (Complexity) | 문장 구조 다양성 | 구문 분석 |
| 상호작용성 (Interactivity) | 질문·호응 빈도 | 턴 분석 |
| 발음 (Pronunciation) | 단어별 발음 점수 | STT + Phonetic 분석 |

> 이는 Free Talking 세션의 **요약 대시보드용 대표 지표**이며, 하위의 세분화된 70 KPI 코드 중 주요 항목을 묶은 상위 뷰이다.

#### 14.6.2 통합 KPI 레지스트리 — 8대 역량 분류 *(v0.9 신설)*

전체 KPI는 **8대 역량**으로 대분류되며, 각 역량 아래 **하위 목표(Sub-goal)**가 있고, 하위 목표 아래 **측정 항목(Measurement Item)**이 있는 2-depth 구조.

| 대분류 (역량) | 하위 목표 (Sub-goal) | 측정 항목 수 | 대표 KPI 코드 |
|---|---|---|---|
| **읽기 (Reading)** | 해독 및 유창성, 이해 전략, 비판적 읽기, 다양한 텍스트 유형 | 11 | SILENT_READING, KEYWORD_HIT, READING_SPEED, TOPIC_SELECTION, TYPE_ACCURACY, EVIDENCE_EXTRACT, READ_COMPLETION, STAY_TIME, READ_ACCURACY, CHUNKING_APPROP, FLUENCY_GROWTH, PREDICT_COMP, PREDICT_CRITICAL, INFER_VALIDITY |
| **듣기 (Listening)** | 음성 인식, 청취 이해, 실시간 처리, 다양한 담화 유형 | 8 | LINKED_SPEECH, DICTATION_ACC, MAIN_IDEA, TYPE_SCORE, DETAIL_LISTENING, INFER_LISTENING, PRAGMATIC_RECOG, PHONEME_DISCRIM, LIAISON_RECOG |
| **말하기 (Speaking)** | 발화량, 발음 및 운율, 유창성, 상호작용 기술 | 14 | ASR_ACCURACY, ASR_RECOG, PROSODY_PATTERN, SPEAKING_RATE, RESPONSE_LATENCY, HESITATION_FREQ, MLU_LENGTH, SILENCE_RATIO, TOTAL_UTTERANCE, INTERACTION_ACT, FLUENCY_WPM |
| **어휘 (Vocabulary)** | 어휘 인지 및 의미, 어휘 관계, 어휘 사용, 어휘 학습 전략 | 13 | VOCAB_RECOG, RECOG_SPEED, VOCAB_ACCURACY, PRONUN_ACCURACY, IMAGE_MATCHING, COLLOC_RECOG, VOCAB_PROPRIETY, CONTEXT_INFER, SEMANTIC_CONN, RELATION_APPROP, DERIV_RECOG, ROOT_ANALYSIS, ADV_VOCAB, VOCAB_DIVERSITY |
| **문법 (Grammar)** | 문법 정확성, 맥락적 문법, 오류 인식 및 교정, 형태 및 통사 | 11 | BLANK_ACCURACY, GRAMMAR_ITEM, EXPRESSION_APPROP, SENTENCE_COMPLEX, GRAMMAR_ERROR, ACCURACY_SCORE, RE_UTTERANCE, WORD_ORDER, SENTENCE_ACCURACY, ERROR_DETECTION, GRAMMAR_ASSIST |
| **쓰기 (Writing)** | 문장 구성, 쓰기 과정, 문단 및 담화 조직, 장르별 쓰기 | 10 | HANDWRITING_MATCH, KEYWORD_BLANK, STRUCT_COMPLEX, WRITING_ERROR, SEMANTIC_IDENTITY, KEYWORD_RATE, LOGIC_STRUCT, OUTLINE_COMPLETE, LOGIC_COHESION, ESSAY_STRUCT, SELF_CORRECTION |
| **실시간 처리 (Real-time)** | 응답 지연, 화용 이해 | 3 | ASR_ACCURACY, RESPONSE_LATENCY, PRAGMATIC_RECOG |
| **총합** | | **70개** (일부 KPI는 복수 역량 교차 매핑) | |

> 📎 **KPI 코드 전수 목록(70개) · 측정 유닛 · 측정 방법 · 측정 도구**는 `Picklass_KPI_체계_상세.md` §2 참조.

#### 14.6.3 측정 도구 분류

| 도구 | 설명 | 비용 | KPI 수 |
|---|---|---|---|
| **측정 (measured)** | 룰 기반 자동 채점 (단어 매칭, 클릭 정답, 타이머, WPM 계산 등) | $0 (로컬) | ~40개 (57%) |
| **LLM** | AI 분석 (문맥 유추, 의미 관계, 구조 완성도 등) | API 비용 발생 | ~30개 (43%) |
| **mixed** | 복합 (일부 측정 + 일부 LLM) | 일부 비용 | 일부 모듈 전용 |

이 분류는 `ModulePlanningMeta.automationLevel` 속성과 연동되어 **Planner가 시간/비용 예산에 따라 자동 최적화** 가능.

#### 14.6.4 측정 유닛 표준

| 유닛 | 용도 | 예시 KPI |
|---|---|---|
| `%` | 정답률·정확도 | VOCAB_RECOG, KEYWORD_HIT, TYPE_ACCURACY |
| `ms` | 반응 시간 | RECOG_SPEED, RESPONSE_LATENCY |
| `WPM` | 단어/분 (읽기·말하기 속도) | SILENT_READING, SPEAKING_RATE, READING_SPEED |
| `WPM Delta` | 회차별 WPM 변화 | FLUENCY_GROWTH |
| `Score (1-5)` | 루브릭 5점 척도 (LLM 평가) | VOCAB_PROPRIETY, INFER_VALIDITY, LOGIC_STRUCT |
| `Score (0-1)` | 확률/유사도 | SEMANTIC_IDENTITY |
| `Count` | 빈도·카운트 | READ_COMPLETION, HESITATION_FREQ, MLU_LENGTH |
| `Sec/Min` | 체류·발화 시간 | STAY_TIME, TOTAL_UTTERANCE |
| `Ratio (TTR)` | 비율 지표 | VOCAB_DIVERSITY |
| `Length/Level` | 길이·레벨 증감 | SENTENCE_COMPLEX |

#### 14.6.5 레벨별 목표 벤치마크 *(v0.9 신설, 재검토 후 확정 예정)*

엑셀 원자료 기반 초안 수치. **실제 서비스 런칭 전 재검토·확정 필요** (v0.9 태그 = `DRAFT`).

| KPI | PreA1 | A1 | A2 | B1 | B2 | C1+ | 단위 |
|---|---|---|---|---|---|---|---|
| **읽기 속도** (SILENT_READING, READING_SPEED) | 40–60 | 60–90 | 90–120 | 120–150 | 150+ | 150+ | WPM |
| **말하기 속도** (SPEAKING_RATE, FLUENCY_WPM) | — | 40–60 | 60–90 | 90–120 | 120–150 | 150+ | WPM |
| **MLU 평균 발화 길이** (MLU_LENGTH) | — | 3–5 | 5–8 | 8–12 | 12+ | 12+ | 단어/발화 |
| **문장 복잡도** (SENTENCE_COMPLEX) | — | 5–8 | 8–12 | 12–15 | 15+ | 15+ | 단어/문장 |
| **문법 오류율** (GRAMMAR_ERROR, ACCURACY_SCORE) | — | — | 10%↑ 개선 | 5–10% 보통 | 5%↓ 우수 | 5%↓ 우수 | % |
| **침묵 비율** (SILENCE_RATIO) | — | — | — | 20%↓ 유창 | 20%↓ 유창 | 20%↓ 유창 | % |
| **총 발화 시간** (TOTAL_UTTERANCE) | — | 30–60 | 60–90 | 90+ | 90+ | 90+ | 초 |
| **읽기 정확성 ASR** (READ_ACCURACY) | 80%↓ 부적절 | 80–90% 지도 필요 | 90%↑ 독립 | 90%↑ 독립 | 90%↑ 독립 | 90%↑ 독립 | % |
| **ASR 인식률** (ASR_ACCURACY, ASR_RECOG) | — | — | 80%↑ 이해 가능 | 80%↑ | 80%↑ | 80%↑ | % |
| **핵심 키워드 충족률** (KEYWORD_RATE) | — | — | — | 80%↑ 합격 | 80%↑ 합격 | 80%↑ 합격 | % |

> 📎 **전체 벤치마크 표**(70 KPI × 6 레벨 × 경고/합격/우수 3단계 기준)는 `Picklass_KPI_체계_상세.md` §4 참조.

#### 14.6.6 모듈 × KPI 매핑 요약

각 모듈은 **1~3개의 핵심 KPI**를 주 측정 지표로 가진다. §13.1 레지스트리의 `핵심 KPI` 컬럼과 1:1 대응.

| 영역 | 모듈 예시 | 대표 KPI |
|---|---|---|
| Vocabulary | WRD | VOCAB_RECOG + RECOG_SPEED |
| Vocabulary | IMG | IMAGE_MATCHING |
| Reading | SCN | SILENT_READING + KEYWORD_HIT |
| Reading | QAR | TYPE_ACCURACY + EVIDENCE_EXTRACT |
| Reading | RRD | FLUENCY_GROWTH + ASR_ACCURACY *(영역 이동 v0.12)* |
| Speaking | SHD *(따라말하기)* | PROSODY_PATTERN + SPEAKING_RATE |
| Speaking | FRT | TOTAL_UTTERANCE + SILENCE_RATIO + INTERACTION_ACT |
| Writing | SWR | STRUCT_COMPLEX + WRITING_ERROR |
| Writing | PWR | ESSAY_STRUCT + SELF_CORRECTION |

> 📎 **v1.3 Pick-Speak 9모듈 정렬에 따라 모듈 × 70 KPI 매트릭스 재구성 진행 중**. 폐기 모듈의 KPI는 고아 상태이며, 후속 작업에서 (a) 다른 모듈로 재할당 또는 (b) KPI 자체 폐기 결정. 정밀 드릴(SHD·SFB·SMK)·롤플레이(RPB·RPF)는 구 EDR/RPL의 INTERACTION_ACT·SPEAKING_RATE 등 KPI를 승계. 상세 매핑은 `Picklass_KPI_체계_상세.md` §3 또는 엑셀 `모듈×KPI 매핑` 시트 참조.

#### 14.6.7 대시보드·리포트 활용

| 화면 | 표시 범위 | 주체 |
|---|---|---|
| Tutoring 학생 대시보드 | 본인 KPI 추이 (스킬별 요약) | 학생 |
| Speaking FRT 세션 종료 리포트 | 5지표(§14.6.1) + 상세 KPI | 학생 |
| Studio 강사 모니터링 | 배정 학생의 모듈별·KPI별 집계 | 강사 |
| Admin Billing/성과 | 기관 전체 평균 KPI, 벤치마크 대비 | 학원관리자·본부 |
