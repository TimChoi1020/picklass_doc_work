# AI 모듈 × KPI 매핑 현황

> **데이터 기준**: DB 실측값 기준 (2026-06-08)
> **최종 업데이트**: 2026-06-08
> **관련 문서**: [KPI 코드 카탈로그](./kpi-catalog.md)

`ai_modules` 테이블은 backoffice에서 관리하며 speaking·tutoring·backoffice가 공유하는 테이블이다.  
각 모듈의 `selected_kpi_codes` 배열에 `KPI_CATEGORY.code` 값이 저장된다.

> ⚠️ **카탈로그 미등록 KPI 코드 4종** — 아래 코드는 DB에서 사용 중이나 `kpi-catalog.md`에 아직 등록되지 않음. 별도 추가 필요.
> `READING_COMPLETE` (CLR) · `LISTEN_COMPLETE` (LRN) · `EXPRESSION_COM` (RPF·SFB·SMK) · `VOCAB_GUESS` (GMN)

---

## 요약

| 스킬 | 전체 | KPI 설정 | KPI 미설정 | 클린업 필요 |
|---|---|---|---|---|
| reading | 8 | 8 | 0 | 0 |
| speaking | 9 | 9 | 0 | 0 |
| vocabulary | 7 | 4 | 2 (WDR·WDS) | 1 (IMG중복삭제) |
| writing | 2 | 1 | 1 (PWR) | 0 |
| **합계** | **26** | **22** | **3** | **1** |

---

## 1. Reading (8개 모듈)

| code | name | selectedKpiCodes | KPI 한국어명 | KPI 영역 | 비고 |
|---|---|---|---|---|---|
| CLR | Clarification | `READING_COMPLETE` | 읽기 완료율 | 독해 | ⚠️ 카탈로그 미등록 코드 |
| ORL | Oral Reading | `READ_ACCURACY` | 읽기 정확성 (WCPM) | 읽기 유창성 | |
| PRD | Prediction | `PREDICT_COMP` | 예측 타당성 점수 — 이해전략 | 읽기 이해 전략 | |
| QAR | Question-Answer Relationship | `EVIDENCE_EXTRACT` | 근거 추출 능력 | 비판적 읽기 | |
| RRD | Repeated Reading | `READ_ACCURACY` | 읽기 정확성 (WCPM) | 읽기 유창성 | |
| | | `READING_SPEED` | 읽기 속도 (WPM) | 읽기 유창성 | |
| SCN | Scanning | `READING_SPEED` | 읽기 속도 (WPM) | 읽기 유창성 | |
| | | `KEYWORD_HIT` | 키워드 적중률 | 읽기 이해 전략 | |
| SKM | Skimming | `TOPIC_SELECTION` | 주제문 선택 정확도 | 비판적 읽기 | |
| | | `READING_SPEED` | 읽기 속도 (WPM) | 읽기 유창성 | |
| SUM | Summarizing | `KEYWORD_RATE` | 핵심 키워드 충족률 | 문단 쓰기 | |

---

## 2. Speaking (9개 모듈 — 전체 활성)

> Phase 1(2026-06-07)에서 LR·SHR·SHR(삭제예정)·SNR중복삭제·SXP·WSP 삭제.  
> LRN·VLM·SHD·SFB·SMK·RPB·RPF·OMP 신규 등록 및 KPI 확정(2026-06-08).

| code | name | selectedKpiCodes | KPI 한국어명 | KPI 영역 | 비고 |
|---|---|---|---|---|---|
| FRT | Scenario Based Free Talking | `MLU_LENGTH` | MLU (평균 발화 길이) | 발화량 | |
| | | `TOTAL_UTTERANCE` | 총 발화 Sentence Count | 발화량 | |
| | | `INTERACTION_ACT` | 화행 적절성 | 대화하기 | |
| | | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `GRAMMAR_ERROR` | 문법 오류율 | 말하기 문법 정확성 | |
| | | `EXPRESSION_APPROP` | 표현 적절성 | 말하기 문법 정확성 | |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| LRN | Learn & Study | `LISTEN_COMPLETE` | 듣기 완료도 | 듣기 이해 | ⚠️ 카탈로그 미등록 코드 |
| OMP | One Minute Presentation | `MLU_LENGTH` | MLU (평균 발화 길이) | 발화량 | |
| | | `TOTAL_UTTERANCE` | 총 발화 Sentence Count | 발화량 | |
| | | `PRESENT_FLUENCY` | 발표 유창성 | 발표하기 | |
| | | `PRESENT_CONTENT` | 내용 전달 명확성 | 발표하기 | |
| | | `PRESENT_STRUCT` | 발표 구성 완성도 | 발표하기 | |
| | | `GRAMMAR_ERROR` | 문법 오류율 | 말하기 문법 정확성 | |
| | | `EXPRESSION_APPROP` | 표현 적절성 | 말하기 문법 정확성 | |
| RPB | Role-Play Basic | `MLU_LENGTH` | MLU (평균 발화 길이) | 발화량 | |
| | | `RESPONSE_LATENCY` | 즉각 반응 속도 | 말하기 유창성 | |
| | | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| | | `PROSODY_PATTERN` | 억양 ASR 인식률 | 발음 정확도 | |
| RPF | Role-Play Free | `MLU_LENGTH` | MLU (평균 발화 길이) | 발화량 | |
| | | `TOTAL_UTTERANCE` | 총 발화 Sentence Count | 발화량 | |
| | | `INTERACTION_ACT` | 화행 적절성 | 대화하기 | |
| | | `RESPONSE_LATENCY` | 즉각 반응 속도 | 말하기 유창성 | |
| | | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `GRAMMAR_ERROR` | 문법 오류율 | 말하기 문법 정확성 | |
| | | `EXPRESSION_COM` | 표현 이해 정답률 | 표현 이해 | ⚠️ 카탈로그 미등록 코드 |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| SFB | Sentence Fill-in Blank | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `EXPRESSION_COM` | 표현 이해 정답률 | 표현 이해 | ⚠️ 카탈로그 미등록 코드 |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| | | `PROSODY_PATTERN` | 억양 ASR 인식률 | 발음 정확도 | |
| SHD | Shadowing Drill | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| | | `PROSODY_PATTERN` | 억양 ASR 인식률 | 발음 정확도 | |
| SMK | Sentence Making Drill | `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | |
| | | `EXPRESSION_COM` | 표현 이해 정답률 | 표현 이해 | ⚠️ 카탈로그 미등록 코드 |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| | | `PROSODY_PATTERN` | 억양 ASR 인식률 | 발음 정확도 | |
| VLM | Vocabulary Listening & Meaning | `VOCAB_RECOG` | 어휘 인지 정답률 | 어휘 확장 | |

---

## 3. Vocabulary (7개 모듈)

| code | name | selectedKpiCodes | KPI 한국어명 | KPI 영역 | 비고 |
|---|---|---|---|---|---|
| GMN | Guessing Meaning | `VOCAB_GUESS` | 어휘 유추 정확도 | 어휘 유추 | ⚠️ 카탈로그 미등록 코드 |
| IMG중복삭제 | Meaning Guessing | _(없음)_ | — | — | 🗑️ 클린업 필요 (GMN과 중복) |
| WDR | Word Decker Reading | _(미설정)_ | — | — | ⚠️ KPI 미설정 |
| WDS | Word Decker Speaking | _(미설정)_ | — | — | ⚠️ KPI 미설정 |
| WRD | Word Reading Decker | `RECOG_SPEED` | 단어 인식 속도 | 어휘 확장 | |
| | | `VOCAB_RECOG` | 어휘 인지 정답률 | 어휘 확장 | |
| WSD | Word Speaking Decker | `VOCAB_ACCURACY` | 어휘 발화 정확도 | 어휘 발화 | |
| | | `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | |
| WWB | Word Web | `RELATION_APPROP` | 관계어 연관 적절성 | 어휘 관계 | |

---

## 4. Writing (2개 모듈)

| code | name | selectedKpiCodes | KPI 한국어명 | KPI 영역 | 비고 |
|---|---|---|---|---|---|
| PWR | Process Writing | _(미설정)_ | — | — | ⚠️ KPI 미설정 |
| SWR | Sentence Writing | `STRUCT_COMPLEX` | 문장 정확성 | 문장 쓰기 | |
| | | `WRITING_ERROR` | 문법 오류율 | 글 쓰기 과정 | |

---

## 5. 클린업 필요 항목

| code | 문제 유형 | 조치 |
|---|---|---|
| `IMG중복삭제` | GMN과 중복 | `status = 'inactive'` 또는 레코드 삭제 |

---

## 6. KPI 미설정 모듈

| module_code | name | 사유 |
|---|---|---|
| `PWR` | Process Writing | KPI 코드 미지정 상태 |
| `WDR` | Word Decker Reading | KPI 코드 미지정 상태 |
| `WDS` | Word Decker Speaking | KPI 코드 미지정 상태 |

---

## 7. KPI 코드 → 모듈 역방향 조회

| KPI 코드 | KPI 한국어명 | 사용 모듈 |
|---|---|---|
| `READING_COMPLETE` | 읽기 완료율 | CLR |
| `READ_ACCURACY` | 읽기 정확성 (WCPM) | ORL, RRD |
| `READING_SPEED` | 읽기 속도 (WPM) | RRD, SCN, SKM |
| `PREDICT_COMP` | 예측 타당성 점수 — 이해전략 | PRD |
| `KEYWORD_HIT` | 키워드 적중률 | SCN |
| `EVIDENCE_EXTRACT` | 근거 추출 능력 | QAR |
| `TOPIC_SELECTION` | 주제문 선택 정확도 | SKM |
| `KEYWORD_RATE` | 핵심 키워드 충족률 | SUM |
| `LISTEN_COMPLETE` | 듣기 완료도 | LRN |
| `MLU_LENGTH` | MLU (평균 발화 길이) | FRT, OMP, RPB, RPF |
| `TOTAL_UTTERANCE` | 총 발화 Sentence Count | FRT, OMP, RPF |
| `INTERACTION_ACT` | 화행 적절성 | FRT, RPF |
| `SPEAKING_RATE` | 말하기 속도 (WPM) | FRT, RPB, RPF, SFB, SHD, SMK |
| `GRAMMAR_ERROR` | 문법 오류율 | FRT, OMP, RPF |
| `EXPRESSION_APPROP` | 표현 적절성 | FRT, OMP |
| `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | FRT, RPB, RPF, SFB, SHD, SMK, WSD |
| `PROSODY_PATTERN` | 억양 ASR 인식률 | RPB, SFB, SHD, SMK |
| `RESPONSE_LATENCY` | 즉각 반응 속도 | RPB, RPF |
| `EXPRESSION_COM` | 표현 이해 정답률 | RPF, SFB, SMK |
| `PRESENT_FLUENCY` | 발표 유창성 | OMP |
| `PRESENT_CONTENT` | 내용 전달 명확성 | OMP |
| `PRESENT_STRUCT` | 발표 구성 완성도 | OMP |
| `VOCAB_RECOG` | 어휘 인지 정답률 | VLM, WRD |
| `VOCAB_GUESS` | 어휘 유추 정확도 | GMN |
| `RECOG_SPEED` | 단어 인식 속도 | WRD |
| `VOCAB_ACCURACY` | 어휘 발화 정확도 | WSD |
| `RELATION_APPROP` | 관계어 연관 적절성 | WWB |
| `STRUCT_COMPLEX` | 문장 정확성 | SWR |
| `WRITING_ERROR` | 문법 오류율 | SWR |
