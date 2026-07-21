# KPI 카탈로그 정비 — 2026년 6월 마이그레이션

> **작업일**: 2026-06-06 ~ 2026-06-07
> **관련 카탈로그**: [KPI 코드 카탈로그](./kpi-catalog.md)
> **관련 매핑**: [모듈 KPI 매핑](./module-kpi-mapping.md)

---

## 1. 정비 배경

기존 DB의 KPI 코드(`KPI_CATEGORY`)는 연구 단계에서 설계된 80+개 항목을 포함하고 있었으나,
픽클래스 시스템에서 자동 측정 가능하고 모듈에 실제 할당되는 항목은 소수였다.

**정비 목표:**
- 자동 측정 불가 코드 (TSV 연구 문서 ❌ 수동 분류) 비활용 처리
- 미사용·레거시·중복 코드 정리
- 측정 도구 체계화: `측정` / `ASR` / `STT` / `STT+LLM` / `LLM`
- 측정 단위 `%` 통일 (Score 1~5 → %)
- KPI_ITEMS 영역명 사용자 친화적으로 변경

**판단 기준 문서:**  
`picklass_docs/shared/conventions/성과평가 KPI(연구중) - 픽클래스_적용.tsv`

---

## 2. 비활용 코드 목록

`code_items.is_active = false` 처리 대상 (총 44개 KPI_CATEGORY + 5개 KPI_ITEMS + 2개 KPI_GRAMMAR).

### 2.1 KPI_CATEGORY (44개)

| code | 구분 | 비활용 사유 | 모듈 영향 |
|---|---|---|---|
| `Fluency` | 레거시 | `SPEAKING_RATE`로 대체됨 | — |
| `Flu` | 레거시 | `FLUENCY_WPM`으로 대체됨 | — |
| `` (빈 코드) | 오류 | 데이터 입력 오류 (sort_order=5) | — |
| `SILENT_READING` | 섹션 제외 | `r_decoding_fluency` 정비 | — |
| `CHUNKING_APPROP` | 섹션 제외 | TSV ❌ 수동, `r_decoding_fluency` 정비 | **ORL** 수정 |
| `FLUENCY_GROWTH` | 섹션 제외 | `r_decoding_fluency` 정비 | — |
| `TYPE_ACCURACY` | 섹션 제외 | `r_comprehension_strategies` 정비 | **QAR** 수정 |
| `READ_COMPLETION` | 섹션 제외 | `r_comprehension_strategies` 정비 | **CLR** 수정 |
| `LOGIC_STRUCT` | 섹션 제외 | TSV ❌ 수동, `r_comprehension_strategies` 정비 | **SUM** 수정 |
| `PREDICT_CRITICAL` | 섹션 제외 | `r_critical_reading` 정비 | **PRD** 수정 |
| `STAY_TIME` | 섹션 전체 제거 | `r_text_types` 섹션 삭제 | **CLR** 수정 |
| `LINKED_SPEECH` | 섹션 전체 제거 | `l_sound_recognition` 섹션 삭제 | **LR** 수정 |
| `DICTATION_ACC` | 섹션 전체 제거 | `l_sound_recognition` 섹션 삭제 | — |
| `ERROR_DETECTION` | 섹션 전체 제거 | `l_sound_recognition` 섹션 삭제 | — |
| `PHONEME_DISCRIM` | 섹션 전체 제거 | `l_sound_recognition` 섹션 삭제 | — |
| `LIAISON_RECOG` | 섹션 전체 제거 | `l_sound_recognition` 섹션 삭제 | — |
| `TYPE_SCORE` | 섹션 제외 | `l_comprehension` 정비 | — |
| `ASR_ACCURACY` | 섹션 전체 제거 | `l_realtime_processing` 섹션 삭제 | **RRD, FRT** 수정 |
| `PRAGMATIC_RECOG` | 섹션 전체 제거 | `l_realtime_processing` 섹션 삭제 | **FRT** 수정 |
| `DETAIL_LISTENING` | 섹션 전체 제거 | `l_discourse_types` 섹션 삭제 | — |
| `ASR_RECOG` | 섹션 제외 | `s_pronunciation_prosody` 정비 | **LR** 수정 (KPI 전체 공백) |
| `FLUENCY_WPM` | 섹션 제외 | `s_fluency` 정비 | — |
| `BLANK_ACCURACY` | 섹션 제외 | `s_accuracy` 정비 | — |
| `ACCURACY_SCORE` | 섹션 제외 | `s_accuracy` 정비 | — |
| `HESITATION_FREQ` | 섹션 제외 | `s_interaction_skills` 정비 | — |
| `SENTENCE_COMPLEX` | 섹션 전체 제거 | 오류 인식 및 교정 섹션 삭제 | **SXP** 수정 |
| `GRAMMAR_ITEM` | 섹션 전체 제거 | 맥락적 문법 섹션 삭제 | — |
| `RE_UTTERANCE` | 섹션 전체 제거 | 맥락적 문법 섹션 삭제 | — |
| `GRAMMAR_ASSIST` | 섹션 전체 제거 | 맥락적 문법 섹션 삭제 | — |
| `SENTENCE_ACCURACY` | 섹션 제외 | `w_sentence_construction` 정비 | — |
| `HANDWRITING_MATCH` | 섹션 제외 | `w_sentence_construction` 정비 | — |
| `KEYWORD_BLANK` | 섹션 제외 | `w_sentence_construction` 정비 | — |
| `SEMANTIC_IDENTITY` | 섹션 제외 | `w_discourse_organization` 정비 | — |
| `ESSAY_STRUCT` | 섹션 전체 제거 | `w_genre_writing` 섹션 삭제 | **PWR** 수정 |
| `SELF_CORRECTION` | 섹션 제외 | `w_writing_process` 정비 | **PWR** 수정 |
| `IMAGE_MATCHING` | 섹션 제외 | `v_recognition_meaning` 정비 | — |
| `CONTEXT_INFER` | 섹션 제외 | `v_recognition_meaning` 정비 | **GMN** 수정 |
| `COLLOC_RECOG` | 섹션 제외 | `v_word_relationships` 정비 | — |
| `VOCAB_PROPRIETY` | 섹션 제외 | `v_word_relationships` 정비 | — |
| `DERIV_RECOG` | 섹션 제외 | `v_word_relationships` 정비 | — |
| `SEMANTIC_CONN` | 섹션 제외 | `v_word_relationships` 정비 | **WWB** 수정 |
| `ROOT_ANALYSIS` | 섹션 제외 | `v_word_relationships` 정비 | — |
| `ADV_VOCAB` | 섹션 제외 | `v_word_relationships` 정비 | — |

### 2.2 KPI_ITEMS (5개)

| code | name | 사유 |
|---|---|---|
| `r_text_types` | 다양한 텍스트 유형 | 해당 카테고리 전체 사용 안 함 |
| `l_sound_recognition` | 음성 인식 | 해당 카테고리 전체 사용 안 함 |
| `l_realtime_processing` | 실시간 처리 | 해당 카테고리 전체 사용 안 함 |
| `l_discourse_types` | 다양한 담화 유형 | 해당 카테고리 전체 사용 안 함 |
| `w_genre_writing` | 장르별 쓰기 | 해당 카테고리 전체 사용 안 함 |

### 2.3 KPI_GRAMMAR (2개)

| code | name | 사유 |
|---|---|---|
| `grammar_in_context` | 맥락적 문법 | 연결된 KPI_CATEGORY 코드 모두 비활용 |
| `error_analysis` | 오류 인식 및 교정 | 연결된 KPI_CATEGORY 코드 모두 비활용 |

---

## 3. DB 반영 계획

### 3.1 변경 요약

| 테이블 | 대상 | 작업 |
|---|---|---|
| `code_items` (KPI_CATEGORY) | 44개 코드 | `is_active = false` |
| `code_items` (KPI_CATEGORY) | 신규 3개 (PRESENT_*) | INSERT |
| `code_items` (KPI_CATEGORY) | 19개 코드 | `name` / `extra_data` 변경 |
| `code_items` (KPI_ITEMS) | 10개 | `name` 변경 |
| `code_items` (KPI_ITEMS) | 5개 | `is_active = false` |
| `code_items` (KPI_GRAMMAR) | 2개 | `is_active = false` |
| `ai_modules` | ORL, QAR, SUM, PRD, CLR, LR, RRD, SCN, FRT, SXP, PWR, GMN, WWB | `selected_kpi_codes` 수정 |

### 3.2 ai_modules selectedKpiCodes 수정

| module_code | 현재 | 변경 후 | 비고 |
|---|---|---|---|
| `ORL` | `["READ_ACCURACY","CHUNKING_APPROP"]` | `["READ_ACCURACY"]` | |
| `QAR` | `["TYPE_ACCURACY","EVIDENCE_EXTRACT"]` | `["EVIDENCE_EXTRACT"]` | |
| `SUM` | `["LOGIC_STRUCT","KEYWORD_RATE"]` | `["KEYWORD_RATE"]` | |
| `PRD` | `["PREDICT_COMP","PREDICT_CRITICAL"]` | `["PREDICT_COMP"]` | |
| `CLR` | `["READ_COMPLETION","STAY_TIME"]` | `[]` | ⚠️ KPI 재설계 필요 |
| `LR` | `["ASR_RECOG","LINKED_SPEECH"]` | `[]` | ⚠️ KPI 재설계 필요 |
| `RRD` | `["READING_SPEED","ASR_ACCURACY"]` | `["READING_SPEED"]` | |
| `SCN` | `["SILENT_READING","KEYWORD_HIT"]` | `["KEYWORD_HIT"]` | |
| `FRT` | `["SPEAKING_RATE","ASR_ACCURACY","PROSODY_PATTERN","PRAGMATIC_RECOG","EXPRESSION_APPROP","RESPONSE_LATENCY"]` | `["SPEAKING_RATE","PROSODY_PATTERN","EXPRESSION_APPROP","RESPONSE_LATENCY"]` | |
| `SXP` | `["EXPRESSION_APPROP","SENTENCE_COMPLEX"]` | `["EXPRESSION_APPROP"]` | |
| `PWR` | `["ESSAY_STRUCT","SELF_CORRECTION"]` | `[]` | ⚠️ KPI 재설계 필요 |
| `GMN` | `["CONTEXT_INFER","INFER_VALIDITY"]` | `["INFER_VALIDITY"]` | |
| `WWB` | `["SEMANTIC_CONN","RELATION_APPROP"]` | `["RELATION_APPROP"]` | |

> WSP `["VOCAB_ACCURACY"]`, WSD `["VOCAB_ACCURACY","PRONUN_ACCURACY"]` — 변경 없음

---

## 4. 실행 순서

```
1. (백업) 2026-06-07_kpi-backup-before-cleanup.sql
2. (정비) 2026-06-07_kpi-catalog-cleanup.sql
```

SQL 파일 위치: `apps/api/prisma/manual-sql/`

---

## 5. KPI 재설계 필요 모듈

SQL 실행 후 아래 모듈은 KPI가 비어있어 별도 설계가 필요하다.

| module_code | name | 사유 |
|---|---|---|
| `CLR` | Clarification | READ_COMPLETION·STAY_TIME 모두 비활용 |
| `LR` | Listen & Repeat | ASR_RECOG·LINKED_SPEECH 모두 비활용 |
| `PWR` | Process Writing | ESSAY_STRUCT·SELF_CORRECTION 모두 비활용 |
| `SHR` | Story Question Type Analysis | 원래부터 KPI 미설정 |
| `WDR` | Word Decker Reading | 원래부터 KPI 미설정 |
| `WDS` | Word Decker Speaking | 원래부터 KPI 미설정 |

---

## 6. 실행 결과

> 실행 후 기록

| 항목 | 결과 |
|---|---|
| 실행일 | — |
| 실행자 | — |
| code_items 비활용 처리 | — |
| code_items INSERT | — |
| ai_modules 수정 | — |
| 이슈 | — |
