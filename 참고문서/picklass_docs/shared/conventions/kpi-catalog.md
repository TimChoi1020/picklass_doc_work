# KPI 코드 카탈로그

> **데이터 출처**: `code_groups` / `code_items` 테이블 (Supabase 공유 DB)
> **최종 확인**: 2026-06-08
> **변경 이력**: [kpi-migration-2026-06.md](./kpi-migration-2026-06.md)

---

## 1. 구조 개요

KPI 메타데이터는 3개의 `code_group`으로 관리된다.

| group_code | group_name | 역할 | 소스 사용 여부 |
|---|---|---|---|
| `KPI_CATEGORY` | 성과KPI 분류 | 실제 측정 지표 코드 — **유일한 운영 테이블** | ✅ 사용 |
| `KPI_ITEMS` | 성과KPI 항목 | 영역별 대분류 — 문서 참고용 (코드 비연동) | ❌ 미사용 |
| `KPI_GRAMMAR` | 성과KPI: 문법 | 문법 영역 세부 분류 — 문서 참고용 (코드 비연동) | ❌ 미사용 |

> **2026-06-07 변경**: `extra_data.skill` 필드 도입으로 KPI_ITEMS 역참조가 제거됐다.
> 소스는 `KPI_CATEGORY`만 조회하며, skill은 `extra_data.skill`에서 직접 읽는다.

`ai_modules.selected_kpi_codes`에는 `KPI_CATEGORY.code` 값이 배열로 저장된다.

---

## 2. KPI_CATEGORY — 측정 지표 (운영 테이블)

### 2.1 extra_data 스키마

각 KPI_CATEGORY 항목의 `extra_data` JSON 구조:

```jsonc
{
  "skill":         "s",                    // 스킬 코드: r | l | s | w | v
  "goal":          "말하기 유창성",          // 목표 표시명 (자유 텍스트)
  "measureUnit":   "WPM",                  // 측정 단위
  "measureTool":   "STT",                  // 측정 도구: 측정 | ASR | STT | LLM | STT+LLM
  "measureMethod": "WPM 변별 분석 (원어민 속도 대비)" // 측정 방법 설명
}
```

#### skill 코드표

| skill | 영역 | 비고 |
|---|---|---|
| `r` | 읽기 (Reading) | |
| `l` | 듣기 (Listening) | |
| `s` | 말하기 (Speaking) | |
| `w` | 쓰기 (Writing) | |
| `v` | 어휘 (Vocabulary) | |

#### 소스에서의 정렬·조회 방식

```ts
// 정렬: r(0) → l(1) → s(2) → w(3) → v(4) → goal localeCompare
const SKILL_ORDER = { r: 0, l: 1, s: 2, w: 3, v: 4 };

// skill 레이블 변환
const SKILL_LABEL = { r: '읽기', l: '듣기', s: '말하기', w: '쓰기', v: '어휘' };

// 조회 예시
const skill = item.extraData?.skill;           // 'r' | 'l' | 's' | 'w' | 'v'
const label = SKILL_LABEL[skill] ?? '-';       // '읽기' | ...
```

---

### 2.2 활성 코드 목록 (skill · goal 순)

#### 읽기 (`r`)

| code | name | goal | measureUnit | measureTool |
|---|---|---|---|---|
| `READING_COMPLETE` | 읽기 완료율 | 독해 | % | 측정 |
| `READING_SPEED` | 읽기 속도 (WPM) | 읽기 유창성 | WPM | 측정 |
| `READ_ACCURACY` | 읽기 정확성 | 읽기 유창성 | WCPM | ASR |
| `PREDICT_COMP` | 예측 타당성 점수 — 이해전략 | 읽기 이해 전략 | % | LLM |
| `INFER_VALIDITY` | 유추 타당성 점수 | 읽기 이해 전략 | % | LLM |
| `KEYWORD_HIT` | 키워드 적중률 | 읽기 이해 전략 | % | 측정 |
| `TOPIC_SELECTION` | 주제문 선택 정확도 | 비판적 읽기 | % | 측정 |
| `EVIDENCE_EXTRACT` | 근거 추출 능력 | 비판적 읽기 | % | 측정 |

#### 듣기 (`l`)

| code | name | goal | measureUnit | measureTool |
|---|---|---|---|---|
| `LISTEN_COMPLETE` | 듣기 완료도 | 듣기 이해 | % | 측정 |
| `MAIN_IDEA` | 요지 청취 정답률 | 청취 이해 | % | 측정 |
| `INFER_LISTENING` | 추론적 청취 정답률 | 청취 이해 | % | 측정 |

#### 말하기 (`s`)

| code | name | goal | measureUnit | measureTool |
|---|---|---|---|---|
| `MLU_LENGTH` | MLU — 평균 발화 길이 | 발화량 | Count | STT |
| `TOTAL_UTTERANCE` | 총 발화 Sentence Count | 발화량 | Sentence Count | STT |
| `PRONUN_ACCURACY` | 발음 정확도 ASR 인식률 | 발음 정확도 | % | ASR |
| `PROSODY_PATTERN` | 억양 ASR 인식률 | 발음 정확도 | % | ASR |
| `SPEAKING_RATE` | 말하기 속도 (WPM) | 말하기 유창성 | WPM | STT |
| `RESPONSE_LATENCY` | 즉각 반응 속도 | 말하기 유창성 | ms/sec | 측정 |
| `SILENCE_RATIO` | 머뭇거림 빈도 / 침묵 비율 | 말하기 유창성 | % | STT |
| `EXPRESSION_APPROP` | 표현 적절성 | 말하기 문법 정확성 | % | LLM |
| `GRAMMAR_ERROR` | 문법 오류율 | 말하기 문법 정확성 | % | LLM |
| `EXPRESSION_COM` | 표현 이해 정답률 | 표현 이해 | % | 측정 |
| `INTERACTION_ACT` | 화행 적절성 | 대화하기 | % | LLM |
| `PRESENT_STRUCT` | 발표 구성 완성도 | 발표하기 | % | LLM |
| `PRESENT_CONTENT` | 내용 전달 명확성 | 발표하기 | % | LLM |
| `PRESENT_FLUENCY` | 발표 유창성 | 발표하기 | WPM / % | STT+LLM |

#### 쓰기 (`w`)

| code | name | goal | measureUnit | measureTool |
|---|---|---|---|---|
| `WORD_ORDER` | 어순 정확성 | 문장 쓰기 | % | 측정 |
| `STRUCT_COMPLEX` | 문장 정확성 | 문장 쓰기 | % | LLM |
| `KEYWORD_RATE` | 핵심 키워드 충족률 | 문단 쓰기 | % | LLM |
| `OUTLINE_COMPLETE` | 개요 완성도 | 문단 쓰기 | % | LLM |
| `WRITING_ERROR` | 문법 오류율 | 글 쓰기 과정 | % | LLM |
| `VOCAB_DIVERSITY` | 표현 다양성 (TTR) | 글 쓰기 과정 | Ratio (TTR) | 측정 |
| `LOGIC_COHESION` | 논리적 연결성 | 글 쓰기 과정 | % | LLM |

#### 어휘 (`v`)

| code | name | goal | measureUnit | measureTool |
|---|---|---|---|---|
| `VOCAB_RECOG` | 어휘 인지 정답률 | 어휘 확장 | % | 측정 |
| `RECOG_SPEED` | 단어 인식 속도 | 어휘 확장 | ms | 측정 |
| `VOCAB_ACCURACY` | 어휘 발화 정확도 | 어휘 발화 | % | ASR |
| `VOCAB_GUESS` | 어휘 유추 정확도 | 어휘 유추 | % | LLM |
| `RELATION_APPROP` | 관계어 연관 적절성 | 어휘 관계 | % | LLM |

---

## 3. KPI_ITEMS — 영역별 대분류 (문서 참고용)

> **소스 비연동**: 2026-06-07 `extra_data.skill` 도입 이후 코드에서 조회하지 않는다.
> `KPI_CATEGORY.extra_data.goal` 텍스트의 표준 참조 목록으로만 유지한다.

### 읽기 (Reading)
| code | name |
|---|---|
| `r_comprehension` | 독해 |
| `r_decoding_fluency` | 읽기 유창성 |
| `r_comprehension_strategies` | 읽기 이해 전략 |
| `r_critical_reading` | 비판적 읽기 |

### 듣기 (Listening)
| code | name |
|---|---|
| `l_comprehension` | 듣기 이해 |
| `l_listening` | 청취 이해 |

### 말하기 (Speaking)
| code | name |
|---|---|
| `s_utterance_volume` | 발화량 |
| `s_pronunciation_prosody` | 발음 정확도 |
| `s_fluency` | 말하기 유창성 |
| `s_accuracy` | 말하기 문법 정확성 |
| `s_expression` | 표현 이해 |
| `s_interaction_skills` | 대화하기 |
| `s_presentation` | 발표하기 |

### 쓰기 (Writing)
| code | name |
|---|---|
| `w_sentence_construction` | 문장 쓰기 |
| `w_discourse_organization` | 문단 쓰기 |
| `w_writing_process` | 글 쓰기 과정 |

### 어휘 (Vocabulary)
| code | name |
|---|---|
| `v_vocab_expansion` | 어휘 확장 |
| `v_vocab_speaking` | 어휘 발화 |
| `v_word_guessing` | 어휘 유추 |
| `v_word_relationships` | 어휘 관계 |

비활용 처리 (2026-06-07): `r_text_types`, `l_sound_recognition`, `l_realtime_processing`, `l_discourse_types`, `w_genre_writing`  
비활용 처리 (2026-06-08): `v_recognition_meaning` (→ 어휘 확장·어휘 발화로 세분화)

---

## 4. KPI_GRAMMAR — 문법 세부 분류 (문서 참고용)

> **소스 비연동**: 소스에서 조회된 적 없음. 기획 단계 설계 기록으로만 보존.

| code | name | 상태 |
|---|---|---|
| `morphology_syntax` | 형태 및 통사 | 활성 |
| `tense_modality_voice` | 시제, 양태, 태 | 활성 |
| `grammar_in_context` | 맥락적 문법 | 비활용 |
| `error_analysis` | 오류 인식 및 교정 | 비활용 |
