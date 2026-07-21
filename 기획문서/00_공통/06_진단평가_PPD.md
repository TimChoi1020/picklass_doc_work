---
title: 학습 진단·평가 — 레벨 테스트 체계 및 PPD
version: v1.5
updated: 2026-07-19
owner_service: common
master_origin: v0.21 §15
depends_on:
  - ./04_통합모듈시스템.md
  - ../Studio/기획서_진단온보딩.md
  - ../Tutoring/기획서_AchievementTest.md
---

> 📄 본 문서는 통합 기획서 v0.21(→ `_archive/`)에서 분리된 문서이다. 본문 내 **§번호는 구 통합 기획서 기준**이며, §번호 → 신규 문서 매핑은 [README](../README.md)의 매핑표를 참조한다.

## 15. 학습 진단·평가 — 레벨 테스트 체계 *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 회의록 결정에 따라 픽클래스의 **레벨 진단·성취도 평가** 체계를 일원화한다. 본 장은 (a) 학습 시작 시점의 **Level Test**, (b) 학습 종료 시점의 **Achievement Test**, (c) 진단 결과의 활용, (d) 전사(B2B/B2C/전화/인강/출강) 일원화 로드맵을 정의한다.

> 🔁 **상위 프레이밍 연결**: 본 장의 진단(Level Test/PPD)은 제품 히어로인 **"진단 → 처방 → 증명" 폐쇄 루프**의 첫 단계(①)이다. 루프 전체 서사의 SSoT는 [01_서비스개요 §2.1.1](01_서비스개요_전체구조.md)이며, 본 장은 그중 **진단(①)** 과 강사 진단카드의 **처방 권장(②)**, Achievement Delta의 **증명(③)** 요소를 소유한다.

### 15.1 핵심 원칙

회의록(260413) 결정 사항을 5개 원칙으로 정리한다.

| 원칙 | 내용 | 출처 |
|---|---|---|
| **P1. 별도 레벨 테스트 미강제** | **레벨 확인 수업 1회**를 통해 별도 레벨 테스트 없이 학습자 수준 자동 분석 | 회의록 "레벨 설계" |
| **P2. 점수 표기 이중화** | **CEFR 기준 점수** + **파고다 레벨**을 병행 표기 | 회의록 "레벨 설계" |
| **P3. Level/Achievement Test 동일 방식** | 학습 전 Level Test와 학습 종료 후 Achievement Test는 **동일 방식**이어도 무관 | 회의록 "추후 고도화 1)" |
| **P4. 정밀 진단 (고도화)** | 체계적인 루브릭에 기반한 **정밀 레벨테스트 결과**, MBTI 형태의 개인 성향·학습 환경 반영 | 회의록 "추후 고도화 1)" |
| **P5. 전사 일원화** | B2B, B2C, 전화/인강/출강 등 **전사 레벨테스트 체계 일원화** | 회의록 "추후 고도화 6)" |

### 15.2 Level Test (학습 시작 시 진단)

#### 15.2.1 진입 경로

| 진입 채널 | 흐름 |
|---|---|
| 신규 학습자 (B2C/B2B 공통) | 회원가입 → "레벨 확인 수업" 자동 배정 (P1) |
| 기존 학습자 신규 과정 등록 | 첫 레슨이 자동으로 Level Test 모드로 진행 |
| 강사·관리자 수동 트리거 | Studio/Admin에서 "레벨 재진단" 발급 (재시험 시) |

#### 15.2.2 진행 방식

**기본 — 레벨 확인 수업 1회 (P1)**

별도의 객관식 시험 없이, 학습자가 1회 수업을 진행하면서 발생한 **발화·청해·읽기·쓰기 데이터**를 자동 수집·분석하여 레벨 추정. 회의록 P1 원칙에 따라 학습자에게 "시험"이라는 부담을 주지 않는다.

**측정 데이터 (Level Test 모드) — 스피킹 트랙**

| 영역 | 측정 데이터 | 산출 KPI |
|---|---|---|
| Speaking | 발화량·MLU·WPM·문법 오류율·어휘 다양도 | SPEAKING_RATE, MLU_LENGTH, GRAMMAR_ERROR, VOCAB_DIVERSITY |
| Reading | 읽기 속도·이해도 | SILENT_READING, READ_COMPLETION |
| Writing | 작문 길이·정확도·표현력 | TYPE_ACCURACY, VOCAB_PROPRIETY |

> ⚠️ **Listening 영역은 폐기 확정** (Ӱ13.1.5) — 스피킹 트랙 레벨 테스트는 Vocab / Speaking / (Reading 참고) 세 축으로 운영한다. **P-ALT R/W 트랙**(Vocab / Reading / Writing 적응형 CAT)는 별도 실시 시스템으로 운영되며 상세 스펙은 [기획서_진단온보딩 (P0-1)](../Studio/기획서_진단온보딩.md) 소유.

#### 15.2.3 레벨 산출 알고리즘 (개요)

```
[레벨 확인 수업 종료]
    ↓
[KPI 누적 — 영역별 점수]
    ↓
[CEFR 분류 모델 (LLM + 룰 기반 하이브리드)]
   - PreA1 / A1 / A2 / B1 / B2 / C1 / C2
    ↓
[CEFR ↔ 파고다 레벨 매핑 (P2)]
   - Pagoda L1 ~ L18 (Picklass §14.2 정의)
    ↓
[학습자 페이지 노출]
   - "당신의 CEFR 레벨: A2"
   - "파고다 레벨: L5 (CEFR A2 중반)"
```

#### 15.2.4 CEFR ↔ 파고다 레벨 매핑 (참조)

§14.2 정의를 본 장에서 일원화 표로 명시.

| CEFR | 파고다 그룹 | 파고다 레벨 |
|---|---|---|
| PreA1 | Beginner | L1 |
| A1 | Beginner | L2~L3 |
| A2 | Elementary | L4~L6 |
| B1 | Intermediate | L7~L10 |
| B2 | Upper-Intermediate | L11~L14 |
| C1 | Advanced | L15~L17 |
| C2 | Master | L18 |

### 15.3 Achievement Test (학습 종료 시 성취도 평가)

#### 15.3.1 트리거

| 트리거 ID | 조건 |
|---|---|
| `ACH_COURSE_END` | 과정(course) 모든 레슨 완료 시 |
| `ACH_MONTHLY` | 월말 정기 평가 (옵션) |
| `ACH_MANUAL` | 강사/관리자 수동 발급 |

#### 15.3.2 동일 방식 원칙 (P3)

회의록 명시 — Level Test와 Achievement Test는 **동일한 진행 방식**으로 운영해도 무방. 즉:

| 측령 | Level Test | Achievement Test |
|---|---|---|
| 진행 형식 | 1회 학습 세션 | 1회 학습 세션 (동일) |
| 측정 영역 | Vocab / Speaking / Reading (Listening 폐기) | Vocab / Speaking / Reading (Listening 폐기) |
| 출력 | CEFR + 파고다 레벨 | CEFR + 파고다 레벨 + **변화량(Delta)** |
| 주요 차이 | 베이스라인 측정 | **베이스라인 대비 향상도 측정** |

→ Level Test 결과를 베이스라인으로 저장하고, Achievement Test에서 동일 측정 후 `Delta = Achievement - Level` 산출.

#### 15.3.3 결과 노출

| 항목 | 표시 형태 |
|---|---|
| 시작 레벨 | "학습 시작 시: A1 / Pagoda L3" |
| 현재 레벨 | "현재: A2 / Pagoda L5" |
| 변화량 | "📈 1단계 향상 (CEFR A1 → A2, +2 Pagoda 레벨)" |
| 영역별 변화 | Speaking +12 WPM, Reading +30 WPM, Writing +5pt *(Listening 폐기 제거)* |

#### 15.3.4 P-ALT — 적응형 레벨 테스트 엔진 (스피킹 트랙) *(v1.1 신설 — P-ALT_Speaking v8.1 반영)*

> 트랙 안내: 본 절은 **Speaking 트랙** P-ALT를 정의한다. 어휘(Vocab)·낙돁(P2)·대화(P3)·발표(P4) 4파트. **P-ALT R/W 트랙** (어휘·독해·작문 적응형 CAT, P0-1 파일럿 필수)는 [기획서_진단온보딩](../Studio/기획서_진단온보딩.md)이 소유한다.

**핵심 철학**: "아는 단어만큼 말할 수 있다" — 자체 코퍼스 어휘량 기반 적응형(CAT, Computerized Adaptive Test) 진단.

**(1) PT + AT 2종 체인 구조** — 기존 PT/AT-1/AT-2 3분류를 폐지하고 2종 체인으로 재정의.

| 종류 | 역할 | 비교 기준 |
|---|---|---|
| **PT (Placement Test)** | 배치 — 학습 시작 시 레벨 확정 (= §15.2 Level Test) | 절대 진단 |
| **AT (Achievement Test)** | 성취 — 정기 재진단 (= §15.3 Achievement Test) | **직전 진단(previous_session) 대비** |

> 하나의 AT가 동시에 "직전 대비 Post"이자 "다음 대비 Pre"로 작동한다(체인). 성장 추이가 연속적으로 연결된다.

**(2) 파트 구조 (4파트, CAT)**

| 파트 | 내용 | 형식 |
|---|---|---|
| **P1 Vocab Size** | 어휘량 측정 | 4지선다 CAT |
| **P2 문장 낭독** | 발음·유창성 | 3~7문장 CAT |
| **P3 일상 대화** | 상호작용 | 챗봇 6~10턴 (IELTS Part 1형) |
| **P4 주제 발표** | 논리·표현 | 1분 스피치 (IELTS Part 2형, 조건부) |

- P3·P4는 **LLM 단일 호출로 4개 축 동시 채점**. AT의 P3 주제는 학습 이력 기반 **런타임 동적 생성**(정적 풀은 PT/AT 공통 50주제).

**(3) 6대 루브릭 + 가중치**

최종 레벨 = 가중합 / 7.5. **어휘·유창성·문법 ×1.5**, 발음·표현·화용성 ×1.0.

**(4) CAT 규칙 + AT 하한 보호**

| 규칙 | 내용 |
|---|---|
| 하향 판정 | **PT**: '하' 1회 즉시 −1 / **AT**: '하' 연속 2회만 −1 (일시 컨디션 보정) |
| **AT 하한 보호** | AT 결과는 **직전 진단 이하로 하락 불가** |
| **결과 표현 정책** | 학습자에게 **"하락" 표현 금지**, 성장 루브릭·향상 영역 강조 (동기 보호) |

**(5) 공인시험 환산** — L1=PreA1 / L2–L3=A1 / L4–L6=A2 / L7–L10=B1 / L11–L14=B2(OPIc IM–IH / TOEIC-S 130–140 / IELTS 5.5–6.5) / L15–L17=C1 / L18=C2 (정본 밴딩 §15.2.4·§15.4.8과 일치). 파일럿 100건 이상 축적 후 실측 보정.

> ✅ **WPM 앵커 확정 (2026-07-17)**: P-ALT v8.1의 개정을 채택하여 WPM 앵커를 **L1=50 → L18=148 (18레벨 선형보간, 약 `50 + (N−1) × 5.76`)으로 단일 통일**했다. 이 앵커는 진단(P-ALT)·TTS 출력 속도([Speaking §14.2.2](../Speaking/기획서_Speaking.md))·학습자 발화 벤치마크(§14.6.5 SPEAKING_RATE A1=40–60)에 **모두 동일 적용**된다. 기존 `80+(N−1)×4`(L1=80)는 폐기. (레벨별 반올림 정수값의 정확한 규칙은 코드 소유)
>
> ⚠️ **버전 주의**: 리포트 목업 `p_alt_report_v8.html`은 **폐지된 AT-1/AT-2 고정짝 용어**를 아직 사용한다(v8.1 이전). 리포트 화면 구현 시 본 절(체인 구조)을 기준으로 하고 목업은 UI 레퍼런스로만 참조.

**(6) 콘텐츠 설계 원칙** *(v1.3 신설 — `picklass_docs/speaking/architecture/P-ALT_콘텐츠_개발.md`·`어휘테이블-18단계설계` 정합)*

| 원칙 | 내용 |
|---|---|
| **Vocab Size 측정 철학** | "아는 단어만큼 말한다". 재인(recognition) 수준 측정. 문맥 단서 누출 차단 위해 **Nation VST식 비정의 캐리어 문장**(의미 단서 0) 사용 — 정의·유의어로 정답 유추 불가하게 설계 |
| **probe(측정 의도) 기반 P3/P4** | 고정 질문 원문 대신 **"측정 의도(probe)"만 사전 작성** → 런타임 LLM이 학습자 답변에 반응해 질문 생성. 고정 원문은 화용성 측정을 훼손하므로 금지. AT 학습연계 주제도 사전 제작 안 함(런타임 동적 생성) |
| **어휘 레벨링 단일 권위 = CEFR** | 레벨의 유일 권위는 **CEFR(picklass_rank)**. 원어민 코퍼스 빈도(COCA)는 **참고값**일 뿐 — 한국 EFL 학습자의 체감 난이도는 교재·교실 노출 순서를 따르므로, 정렬만 EFL 노출 순서를 우선한다 |
| **모바일 최적화** | Writing 파트 제거, 4파트(P1~P4)로 모바일 진단 완결 |

> 📎 4번째 공인시험 **TOEFL** 예상 구간도 환산 대상에 포함(기존 OPIc·TOEIC-S·IELTS + TOEFL). 실측 보정 전 예상치.
> ⚠️ **축 정합 주의**: 진단은 **6대 루브릭**(어휘·유창성·발음·문법·표현·화용성), 학습 측정은 **5축**(발음·유창·문법·화용·발화량)으로 축이 다르다 — "어휘/표현" 추가·"발화량" 제외 관계가 의도된 분리인지 명시 필요(§15.4 PPD 매핑과 정합 확인).

### 15.4 KPI 통합 진단 엔진 — 개인 절대 수준 측정 (Picklass Proficiency Diagnostics, PPD) *(v0.13 신설)*

> 📋 **기획 단계 — 본 절은 Level Test(§15.2) / Achievement Test(§15.3) / 수료 기준(§11.10) / 게이미피케이션(§11.11) / 푸시·알림(§11.12) / 개인·그룹 리포트가 공통으로 사용하는 측정 엔진**이다. 각 모듈이 측정하는 다수의 KPI(§14.6, 70 코드)를 개인별로 정리해 **현재 수준에 대한 절대값 진단**을 산출한다.

#### 15.4.1 문제 정의

각 모듈은 자체 KPI를 측정한다(§13.1, §14.6.6). 학습이 누적되면 학습자 1명당 **수십~수백 개의 KPI 측정값**이 쌓인다. 이를 그대로 두면 다음 문제가 발생한다.

| 문제 | 결과 |
|---|---|
| KPI 단위·범위가 다름 | WPM, %, ms, count, score 등을 **단일 비교 척도로 환산 필요** |
| KPI 수가 너무 많음 | 학습자·강사·관리자가 한눈에 이해하기 어려움 — **압축 필요** |
| 모듈별·기간별 편향 | 자주 한 모듈만 한 학습자가 과대평가될 수 있음 — **균형 필요** |
| 절대 수준 비교 불가 | "이 학습자가 어느 수준인가?"에 대한 **단일 답이 없음** |

**해결 목표 — 절대값 진단 (Absolute Level Diagnosis)**

학습자 1명에 대해 다음 3개 출력을 **레벨/모듈/기간 무관한 절대값**으로 산출한다.

1. **Picklass Proficiency Score (PPS)** — 0~100 단일 종합 점수
2. **8대 역량별 점수** — Speaking/Reading/Listening/Writing/Vocabulary/Grammar/Fluency/Pragmatics 각 0~100
3. **CEFR + 파트너 레벨** — PPS로부터 자동 매핑

#### 15.4.2 진단 파이프라인 5단계

```
[Step 1] Raw KPI Collection           ← 모듈 세션 종료 시 KpiResult 적재
   ↓
[Step 2] Normalization                 ← 단위 통일 (raw → 0~100 normalized)
   ↓
[Step 3] Weighting                     ← KPI별 중요도 × 측정 신뢰도 × 시간 감쇠
   ↓
[Step 4] Capability Aggregation        ← 8대 역량별 가중 평균
   ↓
[Step 5] Composite Score (PPS)         ← 8대 역량 가중 종합 + 절대 레벨 매핑
```

#### 15.4.3 Step 1 — Raw KPI 적재

모듈 세션 종료 시점에 `kpi_results` 테이블에 적재. (§14.6.0 운영 모델 참조)

```sql
table kpi_results (
  id pk,
  user_id fk,
  session_id fk,           -- ModuleHistory 또는 SpeakingSession
  module_code varchar(10), -- 측정 모듈 (WRD/SUM/FRT 등)
  kpi_code varchar(40),    -- 70 KPI 코드 중 1
  raw_value numeric,       -- 단위별 원값
  unit varchar(20),        -- %, WPM, ms, count, score, ratio
  level_band enum,         -- 측정 시점 학습자 레벨
  measured_at timestamp,
  index (user_id, kpi_code, measured_at)
);
```

#### 15.4.4 Step 2 — KPI 정규화 (Normalization, raw → 0~100)

각 KPI는 단위가 다르므로 **0~100 정규화 점수**로 환산. **§14.6.5 레벨 벤치마크**의 floor(PreA1)·ceiling(C2) 기준값을 활용.

| 단위 | 정규화 공식 | 예시 |
|---|---|---|
| **% (정확도/완료율)** | `score = raw_value` (직접 매핑) | 정답률 80% → 80점 |
| **WPM (속도)** | `score = clamp((raw - floor) / (ceiling - floor) × 100, 0, 100)` | PreA1 floor=50·C2 ceiling=148(§14.6.5 앵커 L1·L18), raw=80 → (80-50)/(148-50)×100 ≈ **31점** |
| **ms (반응 속도, 짧을수록 우수)** | `score = clamp(100 - (raw - target) / target × 100, 0, 100)` | target=1000ms, raw=1500 → 100-(500/1000×100) = **50점** |
| **count (빈도)** | KPI별 룩업 테이블 (예: hesitation 0회=100, 5회=60, 10회=20) | hesitation 5회 → **60점** |
| **score 1-5 (루브릭)** | `score = raw × 20` | 4점 → **80점** |
| **ratio (TTR 등 0~1)** | KPI별 정규화 함수 (S자 곡선 등) | TTR 0.5 → **65점** |

> 📎 정규화 floor/ceiling은 §14.6.5 레벨 벤치마크 표의 PreA1~C2 6개 구간에서 추출하며, KPI 카탈로그(`kpi_definitions` 테이블)에 저장된다.

**정규화 결과 의미**
- **0~9**: 측정 자체 부진 (PreA1 수준 미만)
- **10~24**: A1 수준
- **25~39**: A2 수준
- **40~59**: B1 수준
- **60~79**: B2 수준
- **80~94**: C1 수준
- **95~100**: C2 수준 도달

#### 15.4.5 Step 3 — 가중치 적용 (Weighting)

각 KPI 측정값에는 3종 가중치가 곱해진다.

**(1) 중요도 가중치 (Importance, 1~5점)**

KPI 카탈로그에 사전 정의. 핵심 KPI는 5점, 보조 KPI는 1점.

| KPI 예시 | 중요도 | 사유 |
|---|---|---|
| FLUENCY_WPM | 5 | Speaking 핵심 측정값 |
| GRAMMAR_ERROR | 4 | 문법 정확성 핵심 |
| HESITATION_FREQ | 2 | 보조 신호 |
| RE_UTTERANCE | 1 | 부수 정보 |

**(2) 측정 신뢰도 (Confidence, 0~1)**

측정 횟수 N이 많을수록 신뢰 ↑.

```
confidence = min(1.0, N / N_min)
```
- N_min: KPI별 최소 측정 횟수 (보통 3~5회)
- N=0 → 0 (측정 없음 — 진단 불가)
- N=N_min → 1.0 (충분)

**(3) 시간 감쇠 (Recency Weight)**

오래된 측정값은 영향력 감소.

| 측정 시점 | 가중치 |
|---|---|
| 0~30일 이내 | **1.0** |
| 31~90일 | **0.7** |
| 91~180일 | **0.4** |
| 181일 이상 | **0.2** |

**최종 가중치**
```
weight = importance × confidence × recency
```

#### 15.4.6 Step 4 — 8대 역량별 점수 산출

각 KPI는 8대 역량(§14.6.2) 중 하나에 매핑된다. 역량별로 해당 KPI들의 **가중 평균**을 산출.

```
역량 점수 = Σ(정규화 점수 × 최종 가중치) / Σ(최종 가중치)
```

**예시 — "Fluency(발화 유창성)" 역량 산출**

| KPI | 정규화 | 중요도 | 신뢰도 | 시간 가중 | 최종 가중치 | 기여 |
|---|---|---|---|---|---|---|
| FLUENCY_WPM | 72 | 5 | 0.9 | 1.0 | 4.5 | 324 |
| SILENCE_RATIO | 85 | 4 | 0.8 | 1.0 | 3.2 | 272 |
| HESITATION_FREQ | 60 | 2 | 1.0 | 0.7 | 1.4 | 84 |
| **합계** |  |  |  |  | **9.1** | **680** |

→ Fluency 역량 점수 = 680 / 9.1 ≈ **74.7점**

#### 15.4.7 Step 5 — Picklass Proficiency Score (PPS)

8대 역량 점수의 **가중 종합**. 역량 가중치는 학습자의 학습 목적별로 차등.

**학습 목적별 역량 가중치 매트릭스**

| 학습 목적 | Speaking | Reading | Listening | Writing | Vocabulary | Grammar | Fluency | Pragmatics |
|---|---|---|---|---|---|---|---|---|
| **일반 (균형)** | 1.5 | 1.5 | 1.5 | 1.0 | 1.0 | 1.0 | 1.5 | 1.0 |
| **회화 중심** | 2.0 | 0.5 | 1.5 | 0.3 | 1.0 | 1.0 | 2.0 | 1.5 |
| **시험 대비** | 1.0 | 2.0 | 2.0 | 1.5 | 1.5 | 2.0 | 0.5 | 0.5 |
| **비즈니스** | 1.5 | 1.5 | 2.0 | 2.0 | 1.5 | 1.5 | 1.0 | 2.0 |
| **여행 영어** | 2.0 | 0.5 | 1.5 | 0.3 | 1.5 | 0.5 | 1.5 | 2.0 |

**PPS 공식**

```
PPS = round(Σ(역량 점수 × 역량 가중치) / Σ(역량 가중치))
```

> 학습 목적은 §15.5 정밀 진단 결과 또는 학습자 자가 설정에서 결정된다. 미설정 시 "일반(균형)" 적용.

#### 15.4.8 절대 레벨 매핑 (PPS → CEFR / 파트너 레벨)

PPS 0~100을 **CEFR 7단계** 및 **파트너 레벨(L1~L18)** 두 체계에 매핑.

| PPS 구간 | CEFR | 파트너 레벨 (예시: L1~L18) | 의미 |
|---|---|---|---|
| 0–9 | **PreA1** | L1 | 학습 시작 단계 |
| 10–24 | **A1** | L2~L3 | 기본 표현 가능 |
| 25–39 | **A2** | L4~L6 | 일상 의사소통 |
| 40–59 | **B1** | L7~L10 | 익숙한 주제 자유 표현 |
| 60–79 | **B2** | L11~L14 | 복잡한 주제 능숙 표현 |
| 80–94 | **C1** | L15~L17 | 학술·전문 영어 활용 |
| 95–100 | **C2** | L18 | 원어민 수준 |

> 매핑 임계값은 학습자 데이터 누적 후 분포 기반 calibration (Phase 2 이후). 초기에는 위 고정 임계값을 사용한다.

#### 15.4.9 측정 충분성 (Sufficiency) 및 진단 신뢰도

PPD 출력은 **측정 데이터 충분성**에 따라 3단계 신뢰도 등급을 가진다.

| 등급 | 조건 | UI 표기 |
|---|---|---|
| **✅ 확정 (Confirmed)** | 8대 역량 모두 N ≥ N_min, 최근 30일 데이터 ≥ 1회 | "PPS: 73 (확정)" |
| **🟡 추정 (Estimated)** | 8대 역량 중 6개 이상 N ≥ N_min | "PPS: 73 ± 5 (추정 — 데이터 일부 부족)" |
| **⚠️ 불충분 (Insufficient)** | 6개 미만 충족 | "측정 데이터 부족 — 추가 학습 후 진단 가능" |

신규 학습자는 §15.2 Level Test 1회 세션 직후 보통 "추정" 등급. 4~6주 학습 누적 후 "확정" 등급 진입.

#### 15.4.10 진단 카드 (Diagnostic Card)

PPD 산출 결과를 청자별 표준 카드 형식으로 노출.

**학습자용 진단 카드 (학생 페이지)**

```
📊 당신의 영어 종합 능력 (Picklass Proficiency Score)

PPS: ████████░░  73 / 100   ✅ 확정
레벨: B2 (Upper-Intermediate) / Pagoda L12

8대 역량 (레이더 차트)
  Speaking      ━━━━━━━━●━━ 78
  Reading       ━━━━━━━●━━━ 70
  Listening     ━━●━━━━━━━━ 62
  Writing       ━━━━━━●━━━━ 68
  Vocabulary    ━━━━━●━━━━━ 65
  Grammar       ━━━●━━━━━━━ 55  ← 약점
  Fluency       ━━━━━━━━●━━ 75  ← 강점
  Pragmatics    ━━━━━━━●━━━ 72

📈 강점 TOP 3: Speaking · Fluency · Pragmatics
⚠️ 약점 TOP 3: Grammar · Listening · Vocabulary

🎯 다음 추천 학습:
  · Sentence Making Drill (SMK, 문장 만들어말하기) — 문법 정확도 향상
  · Process Writing (PWR) — 문법·구조 동시 강화
  · Vocabulary Listening & Meaning (VLM) — 듣기 보강
🎯 4주 후 목표: Grammar 65점 (+10) · Listening 70점 (+8)
```

**강사용 진단 카드 (Studio 학생 상세 페이지)**

```
📋 [학생 #45 / 김OO / 4월 진단 스냅샷]

PPS: 73 (확정) | 베이스라인 65 → +8 (4주)
약점 1순위: Grammar 55점 — 자세히
  · 관련 KPI: GRAMMAR_ERROR (74점), TENSE_ACC (40점), PREP_USAGE (51점)
  · 약점 모듈 데이터: SCP 평균 60%, SMK 평균 55%
  · 처방 권장:
    - 다음 레슨 SMK 모듈 1개 추가 (사후 분석 70%↑ 효과 예상)
    - 한국어 grammar hint 활성화 (현재 영문만 사용 중)
    - PWR 4주 후 도전 권장 (현재 prerequisite 미달)
```

**관리자용 진단 카드 (Admin 학생 목록 행)**

```
[김OO]  PPS 73  B2  📈+8  ✅확정  강점Speaking  약점Grammar  [상세]
```

#### 15.4.11 데이터 모델

```sql
-- KPI 정의 카탈로그 (정규화·가중치 사전 정의)
table kpi_definitions (
  kpi_code pk,                  -- 'FLUENCY_WPM' 등 70 코드
  capability enum,              -- 8대 역량 (Speaking/Reading/.../Pragmatics)
  unit varchar(20),             -- %, WPM, ms, count, score, ratio
  importance int,               -- 1~5
  min_measurement_count int,    -- N_min
  benchmark_floor numeric,      -- §14.6.5 PreA1 기준
  benchmark_ceiling numeric,    -- §14.6.5 C2 기준
  normalization_fn varchar(20)  -- 'percent' | 'wpm-linear' | 'ms-inverse' | 'count-lookup' | 'score-x20' | 'ratio-sigmoid'
);

-- 학습자별 진단 스냅샷 (배치 + 실시간 갱신)
table proficiency_snapshots (
  id pk,
  user_id fk,
  computed_at timestamp,
  source_period varchar(20),         -- 'last_30d' | 'last_90d' | 'lifetime'
  goal_profile varchar(20),          -- '일반' | '회화' | '시험' | '비즈니스' | '여행'
  capability_scores jsonb,           -- {speaking:78, reading:70, ...} 0~100
  pps int,                           -- 0~100
  cefr enum,                         -- PreA1 ~ C2
  partner_level int,                 -- L1 ~ L18
  confidence_status enum('confirmed','estimated','insufficient'),
  measurement_count_summary jsonb,   -- 역량별 측정 N
  computed_runtime_ms int            -- 산출 시간 (성능 모니터링)
);

-- 산출 감사 로그 (재현·디버그용)
table proficiency_calc_log (
  id pk,
  snapshot_id fk,
  kpi_code,
  raw_value numeric,
  normalized_score numeric,
  importance numeric,
  confidence numeric,
  recency_weight numeric,
  final_weight numeric,
  contribution_to_capability numeric,
  capability enum
);
```

**산출 주기**
- **실시간**: 모듈 세션 종료 직후 해당 학습자의 PPD 재산출 (~ 200ms 목표)
- **일별 배치**: 매일 00:30 KST 전체 활성 학습자 PPD 재산출 (시간 감쇠 갱신)
- **월말 결산**: 매월 1일 03:00 KST 월간 리포트용 스냅샷 생성

#### 15.4.12 사용처 매트릭스

PPD 엔진은 다음 8개 영역에서 재사용된다. **단일 진단 엔진 — 다중 사용처** 패턴.

| 사용처 | PPD 활용 방식 | 출력 |
|---|---|---|
| §15.2 Level Test | 첫 1회 학습 후 PPD 산출 | CEFR + 파트너 레벨 + 진단 카드 (확정/추정 표기) |
| §15.3 Achievement Test | 학습 N개월 후 PPD 재산출, baseline과 Delta 비교 | ΔPPS + ΔCEFR + 영역별 변화량 |
| §11.10 수료 기준 | PPS·역량 점수 임계값 충족 시 자동 수료 | 수료 진척도 게이지 |
| §11.11 게이미피케이션 | PPS 향상량 → 캐릭터 성장·랭킹 | 캐릭터 표정·기업 랭킹 순위 |
| §11.12 푸시·알림 | PPS 정체 N주, 약점 KPI 임계 등 트리거 | 자동 발송 알림 |
| **개인 리포트** (예정) | 진단 카드 메인 위젯 | 주간/월간 PDF |
| **그룹 리포트** (예정) | 학생별 PPS 매트릭스 + 분포 | 반·기관·기업 종합 |
| §10 시퀀싱 엔진 | 약점 역량 → 다음 모듈 추천 가중치 | LessonPlan 우선순위 |

#### 15.4.13 운영 KPI (PPD 자체 성능 지표)

| KPI | 목표 |
|---|---|
| 진단 산출 시간 (실시간) | ≤ 200 ms |
| Level Test ↔ 강사 평가 일치율 | ≥ 80% |
| Achievement Test 향상 검증 신뢰도 | 학습 3개월 후 평균 ΔPPS ≥ +5 (학습 효과 검증) |
| "확정" 등급 진입 평균 기간 | ≤ 6주 학습 후 |
| 측정 부족 케이스 비율 | ≤ 10% (런칭 6개월 후) |

#### 15.4.14 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 정규화 공식의 부정확성 | 레벨 오판정 | Phase 2 이후 실제 학습자 분포 기반 calibration, 분기 1회 검증 |
| KPI 가중치 공정성 | 특정 모듈 학습자 편향 | 분기 1회 가중치 리뷰 위원회, 8대 역량 균형 점검 |
| 측정 데이터 부족 | 신규 학습자 진단 불가 | "추정 + 신뢰구간" 표기로 점진 정확도 향상, Level Test 안내 강화 |
| 시간 감쇠 과도 | 우수 학습자가 휴면 후 등급 하향 | Achievement Test 주기 안내, 휴면 30일 후 자동 알림 |
| 점수 게이밍 (PPS만 올리려는 행위) | 학습 본질 왜곡 | 다축 측정·일정 시간 학습 요건 부과, 8대 역량 균형 보너스 |

> 📎 본 PPD 엔진의 전체 KPI 코드 정의·정규화 함수·중요도는 컴패니언 문서 **`Picklass_KPI_체계_상세.md`** §5(신설 예정)에 표 형태로 수록한다.

### 15.5 정밀 진단 (Advanced Diagnostics) *(고도화)*

> 📋 **기획 단계 (고도화)** — P4 원칙에 따라 단순 CEFR/파고다 레벨을 넘어 **MBTI형 성향·학습 환경 반영**.

#### 15.5.1 추가 진단 차원

| 차원 | 측정 항목 |
|---|---|
| **학습 성향 (MBTI 형식)** | 청각형 vs 시각형, 분석형 vs 직관형, 도전형 vs 신중형 등 |
| **학습 환경** | 학습 가능 시간대, 학습 디바이스(모바일/PC), 1회 평균 학습시간 |
| **약점 영역** | 4기능 중 가장 취약한 영역, 세부 KPI 약점 (예: 발음·억양·문법) |
| **목표** | 시험 대비, 비즈니스 영어, 일상 회화, 여행 영어 등 |

#### 15.5.2 결과 활용

- **개인화 학습 로드맵 자동 생성**: AI가 진단 결과를 기반으로 추천 모듈 시퀀스 + 추천 콘텐츠 생성 모드(§10.15) + 추천 수강 형태(§11.9) 제안
- **튜터 매칭** (1:1 회화 채널): 학습자 성향에 맞는 AI 튜터 페르소나 추천
- **콘텐츠 톤 조정**: 분석형은 문법 설명 더 많이, 직관형은 빠른 대화 더 많이

#### 15.5.3 루브릭 (Rubric) — 향후 정의

| 영역 | 루브릭 차원 | 점수 척도 |
|---|---|---|
| Speaking | 유창성 · 정확성 · 발음 · 어휘력 · 상호작용 | 1~5점 |
| Listening | 핵심 파악 · 세부 이해 · 추론 | 1~5점 |
| Reading | 읽기 속도 · 어휘력 · 추론 · 비판적 읽기 | 1~5점 |
| Writing | 문장 구조 · 문법 · 어휘 · 응집성 | 1~5점 |

→ Phase 2 이후 상세 루브릭 정의 (§20 로드맵 연동).

### 15.6 전사 일원화 로드맵 *(고도화)*

회의록 "추후 고도화 6)" — B2B/B2C/전화/인강/출강 등 전사의 레벨테스트 체계를 일원화한다.

#### 15.6.1 현재 상태 (Baseline)

| 채널 | 현재 레벨 진단 방식 | 결과 호환성 |
|---|---|---|
| Picklass B2B | 레벨 확인 수업 1회 (P1) | CEFR + 파고다 레벨 |
| Picklass B2C | 동일 | CEFR + 파고다 레벨 |
| 1:1 회화 (전화/화상) | 자체 강사 평가 | 파고다 자체 |
| 인강 (이러닝) | 사전 객관식 테스트 | 자체 |
| 출강 | 강사 인터뷰 | 자체 |

#### 15.6.2 통합 비전

```
[모든 채널의 레벨 진단 데이터]
    ↓
[픽클래스 통합 레벨 진단 엔진 (Picklass Unified Level System, PULS)]
    ↓
[CEFR + 파고다 레벨 + 학습 성향 + 약점 영역]
    ↓
[모든 채널에서 동일 레벨 적용]
   - 학습자가 채널을 옮겨도 동일 레벨로 시작
   - 한 채널의 학습 성과가 다른 채널에 반영
```

#### 15.6.3 단계별 추진 (예정)

| Phase | 시기 | 목표 |
|---|---|---|
| **Phase A** | 2026 Q3 | Picklass B2B/B2C 채널 일원화 (CEFR + 파고다 레벨 표준화) |
| **Phase B** | 2026 Q4 | 1:1 회화(전화/화상) 채널과 양방향 데이터 연동 (§20 블렌디드 학습) |
| **Phase C** | 2027 Q1 | 인강·출강 채널까지 통합 |

### 15.7 데이터 모델 (요약)

```sql
-- 레벨 진단 결과 (Level Test + Achievement Test 공통)
table level_assessments (
  id pk,
  user_id fk,
  type enum('LEVEL_TEST','ACHIEVEMENT_TEST','MANUAL'),
  source_session_id fk,        -- 측정에 사용된 SpeakingSession 또는 LessonResult
  cefr enum('PreA1','A1','A2','B1','B2','C1','C2'),
  pagoda_level int,             -- L1 ~ L18
  scores jsonb,                 -- 영역별 점수 {speaking: 65, listening: 72, ...}
  rubric jsonb null,            -- 정밀 루브릭 (Phase 2+)
  diagnostic jsonb null,        -- MBTI형 성향, 약점, 목표 등 (Phase 2+)
  baseline_id fk null,          -- ACHIEVEMENT_TEST일 때 비교 베이스라인 LEVEL_TEST id
  delta jsonb null,             -- baseline 대비 변화량
  created_at timestamp
);

-- 채널별 통합 매핑 (Phase A+)
table cross_channel_levels (
  user_id pk,
  channel enum('picklass_b2b','picklass_b2c','phone','elearning','outclass'),
  cefr enum('PreA1','A1','A2','B1','B2','C1','C2'),
  pagoda_level int,
  last_assessed_at timestamp,
  source_assessment_id fk
);
```

### 15.8 운영 KPI

| KPI | 목표 |
|---|---|
| Level Test 1회 세션 완료율 | ≥ 95% (학습자가 진단을 완료하는 비율) |
| Level Test ↔ 강사 평가 일치율 | ≥ 80% (강사가 학습자를 직접 평가했을 때의 일치도) |
| Achievement Test 향상 검증 | 학습 3개월 후 평균 ΔCEFR ≥ 0.5단계 |
| 통합 레벨 매핑 정확도 (Phase B+) | 채널 간 매핑 오류율 ≤ 5% |

### 15.9 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 1회 세션의 측정 신뢰도 부족 | Level Test 정확도 ↓ | KPI 다축 측정 + LLM 하이브리드 보정 (§15.2.3) |
| 채널 간 레벨 정의 충돌 | 일원화 실패 | CEFR을 표준으로 고정, 채널별 매핑은 변환 함수로 정의 |
| 정밀 루브릭 과부하 | 운영 복잡도 ↑ | Phase 2 이후 점진 도입, Phase 1에는 CEFR만 사용 |
| 학습자 거부감 | 진단 회피 | "시험"이 아닌 "확인 수업" UX (P1) |

> 📎 본 장과 연관된 KPI 코드 정의는 §14.6, 학습자 페이지 노출 형식은 §11.10.4, 운영 모드 분류(정규/자유)는 §11.9 참조.

---


