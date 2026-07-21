# P-ALT DB 설계

> 기준 기획서: [`P-ALT_Speaking_v8.md`](./P-ALT_Speaking_v8.md)  
> 리포트 레이아웃: `p_alt_report_v8.html`  
> 작성일: 2026-07-03  
> DDL 위치: `speaking.picklass.com/apps/api/prisma/manual-sql/`

---

## 1. 설계 원칙

- speaking은 tutoring과 **동일 Supabase 인스턴스를 공유**한다. 모든 테이블에 `alt_` 접두사를 붙여 네임스페이스를 분리한다.
- `prisma migrate` / `db push` **사용 금지**. DDL은 수동 SQL로 작성하고 Supabase Studio에서 직접 실행한다 (DB 마이그레이션 정책 참조: `../../shared/infrastructure/db-migration-policy.md`).
- `users` 테이블은 기존 공통 테이블을 그대로 참조한다. speaking 전용 컬럼은 추가하지 않는다.
- JSONB는 **쿼리가 필요 없는 순수 표시용 데이터**에만 허용한다. 비교·집계가 필요한 필드는 별도 컬럼 또는 별도 테이블로 정규화한다.

---

## 2. 테이블 그룹 개요

```
[정적 Lookup]
  alt_exam_conversion      -- P-ALT 레벨 ↔ 공인시험 예상 점수 환산표
  alt_level_benchmarks     -- 동일레벨 사용자 루브릭 백분위 (주기 갱신)

[문항·콘텐츠]
  alt_p1_questions         -- P1 Vocab Size 4지선다 문항
  alt_p2_sentences         -- P2 문장 낭독 CAT 문장 풀
  alt_p3_topics            -- P3 대화 주제
  alt_p3_probes            -- P3 대화 측정 의도(probe) — 질문 원문이 아닌 의도 저장
  alt_p4_topics            -- P4 주제 발표 카드

[세션·응답]
  alt_sessions             -- 시험 세션 (PT / AT-1 / AT-2)
  alt_p1_responses         -- P1 문항별 응답
  alt_p2_responses         -- P2 낭독별 응답
  alt_p3_turns             -- P3 대화 턴별 기록
  alt_p4_responses         -- P4 발표 응답

[결과·리포트]
  alt_results              -- 최종 레벨·루브릭 집계
  alt_script_errors        -- P3·P4 발화 오류 문장 (정규화)
```

---

## 3. 정적 Lookup 테이블

### 3.1 `alt_exam_conversion`

P-ALT 레벨과 공인시험 예상 점수 환산. 리포트 섹션 6 렌더링에 사용.  
파일럿 100건 누적 후 실측 기반으로 보정하며, 보정 이력은 `updated_at`으로 추적한다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `level` | INTEGER PK | 1–18 |
| `cefr` | VARCHAR(4) | 'A1'–'C2' |
| `opic_min` | VARCHAR(8) | 예: 'NL', 'IM2' |
| `opic_max` | VARCHAR(8) | |
| `toeic_s_min` | INTEGER | |
| `toeic_s_max` | INTEGER | |
| `ielts_min` | DECIMAL(3,1) | 예: 5.5 |
| `ielts_max` | DECIMAL(3,1) | |
| `toefl_s_min` | INTEGER | |
| `toefl_s_max` | INTEGER | |
| `updated_at` | TIMESTAMPTZ | 보정 시점 |

초기 데이터는 기획서 §3-2 환산표 기준으로 18행 INSERT.

### 3.2 `alt_level_benchmarks`

동일레벨 사용자 루브릭 백분위. 리포트 섹션 3의 "상위 %" 계산 원천.  
배치 작업으로 주기 갱신하거나, 응시자 100명 미만인 초기에는 하드코딩 시드값으로 운영.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `level` | INTEGER | 최종 레벨 기준 |
| `rubric_key` | VARCHAR(20) | 'vocab' / 'fluency' / 'pronunciation' / 'grammar' / 'expression' / 'pragmatics' |
| `avg_level` | DECIMAL(4,2) | 해당 루브릭 평균 레벨 |
| `p25` | DECIMAL(4,2) | 25 백분위 |
| `p50` | DECIMAL(4,2) | 50 백분위 (중앙값) |
| `p75` | DECIMAL(4,2) | 75 백분위 |
| `sample_count` | INTEGER | 집계에 사용된 세션 수 |
| `updated_at` | TIMESTAMPTZ | |

---

## 4. 문항·콘텐츠 테이블

### 4.1 `alt_p1_questions` — Vocab Size 문항

CAT 방식. 응답 시간 3초 기준으로 상(Exceeds)/중(Meets) 판정.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `level` | INTEGER | 1–18 |
| `sentence` | TEXT | 밑줄 단어가 포함된 문장 |
| `target_word` | VARCHAR(100) | 밑줄 단어 |
| `option_1`–`option_4` | VARCHAR(100) | 선택지 (정답 포함) |
| `correct_option` | SMALLINT | 1–4 |
| `coca_rank` | INTEGER | target_word의 빈도 순위(= `words_18levels_6000.frequency_rank` 복사). **참고값**, 레벨 판정 기준 아님. supplement 단어는 NULL |
| `response_threshold_ms` | INTEGER | 상/중 판정 기준 (기본 3000) |
| `is_active` | BOOLEAN | false면 출제 제외 |
| `created_at` | TIMESTAMPTZ | |

> **레벨 판정은 `picklass_rank`로 한다 (COCA 아님)**: `level`은 `words_18levels_6000.picklass_rank`(CEFR 기반, 1~18)와 1:1 대응한다. target word는 `WHERE picklass_rank = level`로 추출하므로 레벨 적합성이 테이블에서 보장된다. `coca_rank`는 부가 참고값일 뿐이다. 출제 파이프라인은 콘텐츠 문서 [§3a](./P-ALT_콘텐츠_개발.md) 참조.

**목표 물량**: 레벨당 40문항 × 18레벨 = 720문항 ([콘텐츠 개발 문서](./P-ALT_콘텐츠_개발.md) 참조)

### 4.2 `alt_p2_sentences` — 낭독 문장

CAT 방식. 복합 점수 = WPM(50%) + 단어 일치율(30%) + ASR confidence(20%).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `level` | INTEGER | 1–18 |
| `sentence` | TEXT | 낭독 대상 문장 |
| `word_count` | SMALLINT | 실제 단어 수 (레벨별 문장 길이 range 내, 콘텐츠 §3.1) |
| `target_wpm` | SMALLINT | 해당 레벨 목표 WPM(단일 center 값, 콘텐츠 §3.1). 채점 시 ±15% 판정 밴드 파생 |
| `coca_max_rank` | INTEGER | 문장 내 내용어의 max(`words_18levels_6000.frequency_rank`). **참고값**, 레벨 판정 기준 아님 (supplement 단어 제외) |
| `cefr` | VARCHAR(4) | 'A1'–'C2' (= 레벨의 picklass_level 밴드) |
| `has_liaison` | BOOLEAN | 연음 포함 여부 (L7+ true) |
| `is_active` | BOOLEAN | |
| `created_at` | TIMESTAMPTZ | |

> **WPM은 단일 center 값**: 레벨별 목표 WPM은 하나의 값(콘텐츠 §3.1)이며, 상/중/하 판정은 이 값에 ±15%/±20%를 적용해 파생한다(기획서 §7-2). 따라서 min/max 컬럼을 두지 않고 `target_wpm` 하나만 저장한다. 응답 테이블 `alt_p2_responses.target_wpm`도 이 값을 복사한다.
>
> **어휘 레벨 검증은 `picklass_rank`로 한다 (COCA 아님)**: 문장의 레벨 적합성은 (a) `picklass_rank = level`인 단어를 **최소 1개 포함**(레벨 도달) + (b) 모든 내용어의 `picklass_rank ≤ level`(레벨 초과 없음)로 검증한다. 레벨 N 단어를 seed로 먼저 뽑아 문장에 포함시킨다(콘텐츠 [§3a.3](./P-ALT_콘텐츠_개발.md)). `coca_max_rank`는 부가 참고값일 뿐이다.

**목표 물량**: 레벨당 12문장 × 18레벨 = 216문장

### 4.3 `alt_p3_topics` — 대화 주제

> **정적 주제는 전부 PT·AT 공통(범용)** — AT 전용 학습연계 주제는 사전 제작하지 않고 향후 런타임 동적 생성(콘텐츠 §6.1). 따라서 `test_type` 컬럼을 두지 않는다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `title` | VARCHAR(200) | 주제명 |
| `level_min` | INTEGER | 적합 최소 레벨 |
| `level_max` | INTEGER | 적합 최대 레벨 |
| `category` | VARCHAR(50) | '일상' / '직업' / '취미' / '여행' 등 |
| `is_active` | BOOLEAN | |
| `created_at` | TIMESTAMPTZ | |

### 4.4 `alt_p3_probes` — 대화 측정 의도(probe)

> **설계 배경 — 왜 "질문 원문 풀"이 아니라 "의도 풀"인가**
>
> 기획서 §8-1은 P3를 "챗봇이 응답 수준을 **실시간으로 판단하여 발화 난이도와 질문 복잡도를 조정**하는 자유 대화"로 규정한다. 이 요구사항과 **정해진 질문 문장을 그대로 출제하는 방식은 근본적으로 충돌**한다.
>
> - **부자연스러움**: 사용자가 이미 말한 내용을 무시하고 대본상 다음 질문을 던지면 대화가 어긋난다.
> - **측정 타당성 훼손**: 이 어긋남은 단순 UX 문제가 아니다. 대화 흐름이 깨지면 사용자가 당황하고, 그 혼란이 **화용성(pragmatics) 점수를 부당하게 깎는다.** 자연스러운 대화 자체가 화용성 측정의 전제이기 때문이다.
>
> 따라서 이 테이블은 **말할 문장(verbatim)이 아니라, 각 턴에서 끌어내야 할 측정 의도(probe)** 를 저장한다. LLM은 실제 발화 문장을 사용자 답변에 반응하며 자유롭게 생성하되, 정해진 의도만 대화 안에 자연스럽게 녹인다. 실제 IELTS Part 1 시험관도 토픽 프레임은 고정하되 문장은 응시자에 맞춰 조율하는데, 그와 동일한 구조다.
>
> 이 설계로 세 가지가 동시에 성립한다:
> 1. **자연스러움** — LLM이 사용자 발화에 반응하며 문장을 생성 (기획서 "실시간 조정" 충족)
> 2. **측정 통제** — `probe_intent`가 각 루브릭 축의 근거를 반드시 수집하게 보장 (점수 방어 가능)
> 3. **AT 공정성** — AT와 직전 진단이 같은 *의도 세트*를 커버하면, 문장이 달라도 "동일 난이도에서 성장" 주장이 성립

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `topic_id` | INTEGER FK → alt_p3_topics | |
| `turn_order` | SMALLINT | 1–10 권장 순서 |
| `question_role` | VARCHAR(20) | 'opener' / 'follow_up' / 'deep_dive' |
| `probe_intent` | TEXT | **이 턴에서 끌어내야 할 측정 목표.** LLM 프롬프트에 주입되어 실제 질문 생성의 기준이 된다 |
| `example_question` | TEXT | LLM 참고용 예시 문장. 그대로 써도 되고, 사용자 맥락에 맞게 변형해도 됨 (강제 아님) |
| `target_rubric` | VARCHAR(20) | 이 probe가 겨냥하는 루브릭 축: 'grammar' / 'expression' / 'pragmatics' / 'vocab' |
| `is_active` | BOOLEAN | |

> **opener만 예외적으로 원문 고정 허용**: 첫 턴은 이전 대화 맥락이 없어 예시 문장을 그대로 써도 부자연스럽지 않고, 오히려 모든 세션의 출발점을 통제하는 이점이 있다. 이 경우 `example_question`을 그대로 출제한다.
>
> **실제 출제된 질문**은 `alt_p3_turns.question_text`에 기록되어, LLM이 생성한 최종 문장이 무엇이었는지 추적된다.

**목표 물량**: 50주제 × probe 4개(opener 1 + follow_up/deep_dive 3) = 약 200 probe.  
질문 원문을 완성도 높게 다듬을 필요가 없어(예시는 참고용) 고정 원문 방식(주제당 8문항) 대비 제작 부담이 절반 수준으로 감소한다.

### 4.5 `alt_p4_topics` — 주제 발표 카드

IELTS P2형. P3 평균 L7 이상 수험자에게만 제공.  
정적 카드는 전부 PT·AT 공통(범용) — AT 전용은 사전 제작 안 함(§6.1). `test_type` 컬럼 없음.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `keyword_1`–`keyword_3` | VARCHAR(100) | 화면 표시 키워드 3개 |
| `question` | TEXT | 발표 질문 1개 |
| `level_min` | INTEGER | 기본 7 |
| `category` | VARCHAR(50) | '사회' / '환경' / '기술' / '문화' 등 |
| `is_active` | BOOLEAN | |
| `created_at` | TIMESTAMPTZ | |

**목표 물량**: PT 30카드 + AT 20카드 = 50카드

---

## 5. 세션·응답 테이블

### 5.1 `alt_sessions`

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `user_id` | INTEGER FK → users | |
| `test_type` | VARCHAR(4) | **'PT' / 'AT'** (2종) |
| `previous_session_id` | INTEGER FK → alt_sessions | **AT는 직전 완료 진단(PT 또는 직전 AT)을 참조. PT는 NULL** |
| `status` | VARCHAR(20) | 'in_progress' / 'completed' / 'abandoned' |
| `p4_executed` | BOOLEAN | P4 실행 여부 |
| `p1_final_level` | INTEGER | P1 CAT 수렴 레벨 |
| `p2_final_level` | INTEGER | |
| `p3_final_level` | INTEGER | |
| `p4_final_level` | INTEGER | NULL if P4 skipped |
| `started_at` | TIMESTAMPTZ | |
| `completed_at` | TIMESTAMPTZ | |

> **테스트 유형 체계 — "PT 1회 + AT 체인" (2026-07-03 확정)**
>
> 기존 PT/AT-1/AT-2 3분류를 **PT/AT 2종**으로 단순화한다. "AT-1(Pre)/AT-2(Post)"라는 고정 짝을 없애고, **각 AT가 직전 진단과 비교되는 체인** 구조로 정리했다. 배경: 신규 사용자가 PT 직후 다시 AT-1을 봐야 하는 억지와 AT-1 시작점 모호성을 제거하기 위함. (기획서 §1-1 v8.1 개정과 정합)
>
> ```
> PT ─── AT ─── AT ─── AT ─── ...
>  │      │      │      │
>  └──▶ baseline ◀──────┘   (각 AT는 previous_session_id로 직전 진단을 가리킴)
> ```
>
> - **PT**: 전 구간(L1–L18) 배치. **최초 진단은 자동 PT**, 이후에도 **원하면 언제든 재응시(재배치)** 가능. `previous_session_id = NULL`(재-앵커 지점).
> - **AT**: 직전 완료 진단(PT 또는 직전 AT)을 baseline으로 참조. 시작점 = baseline 레벨, 범위 = baseline 레벨 ±4.
> - **성장** = `이번 AT.final_level − previous_session.final_level`. 학습 기간 = `이번 AT.started_at − previous_session.completed_at`.
> - **하한 보호**: 이번 AT `final_level ≥ previous_session.final_level` (직전 진단 이하로 하락 금지).
> - 첫 AT의 baseline이 PT이면 교차 설정(전 구간 vs 제한 범위) 비교이나, **6대 루브릭 축은 공통**이라 성장 측정은 유효. 2번째 AT부터는 AT-vs-AT 동일 설정.
> - `§6.2 alt_script_errors` 오류 지속/해소 비교도 `previous_session_id` 체인으로 동작(PT 포함).

### 5.2 `alt_p1_responses`

문항 단위 응답. 리포트 섹션 1 CAT 궤적 차트의 데이터 원천.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions | |
| `question_id` | INTEGER FK → alt_p1_questions | |
| `item_sequence` | SMALLINT | 차트 x축 순서 (1부터) |
| `selected_option` | SMALLINT | 1–4 |
| `is_correct` | BOOLEAN | |
| `response_time_ms` | INTEGER | |
| `judgment` | VARCHAR(10) | '상' / '중' / '하' |
| `level_at_time` | INTEGER | 이 문항 직후 추정 레벨 |
| `level_upper` | INTEGER | 신뢰 구간 상한 (차트 밴드용) |
| `level_lower` | INTEGER | 신뢰 구간 하한 |
| `answered_at` | TIMESTAMPTZ | |

### 5.3 `alt_p2_responses`

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions | |
| `sentence_id` | INTEGER FK → alt_p2_sentences | |
| `item_sequence` | SMALLINT | |
| `audio_url` | TEXT | 녹음 파일 URL |
| `actual_wpm` | DECIMAL(6,2) | 실측 WPM |
| `target_wpm` | INTEGER | 문장 기준 목표 WPM (복사 저장) |
| `wpm_ratio` | DECIMAL(5,3) | actual / target (예: 1.10 = 110%) |
| `word_match_rate` | DECIMAL(5,3) | 단어 일치율 0–1 |
| `asr_confidence` | DECIMAL(5,3) | Azure ASR confidence 0–1 |
| `composite_score` | DECIMAL(5,3) | WPM×0.5 + 일치율×0.3 + conf×0.2 |
| `judgment` | VARCHAR(10) | '상' / '중' / '하' |
| `level_at_time` | INTEGER | |
| `level_upper` | INTEGER | |
| `level_lower` | INTEGER | |
| `answered_at` | TIMESTAMPTZ | |

### 5.4 `alt_p3_turns`

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions | |
| `topic_id` | INTEGER FK → alt_p3_topics | |
| `turn_number` | SMALLINT | 1–10 |
| `question_text` | TEXT | 실제 출제된 질문 (LLM 생성분 포함) |
| `user_transcript` | TEXT | ASR 전사 결과 |
| `audio_url` | TEXT | |
| `is_spontaneous_expansion` | BOOLEAN | 자발적 주제 확장 여부 (LLM 판정) |
| `response_latency_ms` | INTEGER | 질문→발화 시작까지 지연 |
| `answered_at` | TIMESTAMPTZ | |

### 5.5 `alt_p4_responses`

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions | |
| `topic_id` | INTEGER FK → alt_p4_topics | |
| `audio_url` | TEXT | |
| `transcript` | TEXT | ASR 전사 결과 |
| `duration_seconds` | INTEGER | 실제 발화 지속 시간 |
| `active_vocab_rate` | DECIMAL(5,3) | LLM 산출 — 고급 어휘 사용률 |
| `answered_at` | TIMESTAMPTZ | |

---

## 6. 결과·리포트 테이블

### 6.1 `alt_results`

세션 완료 시 생성. 리포트 전체의 핵심 원천.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions UNIQUE | |
| `user_id` | INTEGER FK → users | 조회 편의 (세션 통해서도 접근 가능) |
| **루브릭 레벨 (6개)** | | |
| `rubric_vocab` | INTEGER | 어휘 레벨 |
| `rubric_fluency` | INTEGER | 유창성 레벨 |
| `rubric_pronunciation` | INTEGER | 발음 정확성 레벨 |
| `rubric_grammar` | INTEGER | 문법 레벨 |
| `rubric_expression` | INTEGER | 표현 레벨 |
| `rubric_pragmatics` | INTEGER | 화용성 레벨 |
| **최종 레벨** | | |
| `final_level` | INTEGER | 가중 합산 결과 |
| `cefr` | VARCHAR(4) | |
| **공인시험 예상** | | |
| `opic_range` | VARCHAR(20) | 예: 'IM-IH' |
| `toeic_s_range` | VARCHAR(20) | 예: '130-140' |
| `ielts_range` | VARCHAR(20) | 예: '5.5-6.5' |
| `toefl_s_range` | VARCHAR(20) | 예: '18-21' |
| **요약 메트릭** | | |
| `rubric_avg` | DECIMAL(4,2) | 6개 루브릭 단순 평균 (리포트 헤더용) |
| `rubric_stddev` | DECIMAL(4,2) | 스킬 편차 |
| `cat_efficiency` | DECIMAL(5,3) | CAT 수렴 효율 (최소 필요 문항 / 실제 사용 문항) |
| `p4_skipped_reason` | VARCHAR(50) | 'level_too_low' / 'pragmatics_too_low' / NULL |
| **처방** | | |
| `prescription` | JSONB | 처방 3개 배열 [{rank, title, desc, module}] — 표시 전용 |
| `created_at` | TIMESTAMPTZ | |

> **가중 합산 공식**  
> final_level = (vocab×1.5 + fluency×1.5 + pronunciation×1.0 + grammar×1.5 + expression×1.0 + pragmatics×1.0) / 7.5  
> AT 하한 보호: final_level ≥ 직전 진단(previous_session) final_level

### 6.2 `alt_script_errors`

P3·P4 발화 오류 문장. JSONB 대신 정규화하여 AT ↔ 직전 진단 비교 쿼리를 지원.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | |
| `session_id` | INTEGER FK → alt_sessions | |
| `part` | VARCHAR(4) | 'P3' / 'P4' |
| `turn_number` | SMALLINT | P3의 경우 몇 번째 턴. P4는 NULL |
| `error_num` | SMALLINT | 리포트 표시 순서 |
| `error_type` | VARCHAR(50) | 13개 유형 중 하나 (아래 참조) |
| `original_text` | TEXT | 발화 원문 전체 문장 |
| `corrected_text` | TEXT | 교정 제안 |
| `error_start_idx` | INTEGER | 원문 내 오류 시작 인덱스 (밑줄 렌더링용) |
| `error_end_idx` | INTEGER | |
| `is_resolved` | BOOLEAN | AT-2에서 동일 유형 오류가 사라졌으면 true |
| `created_at` | TIMESTAMPTZ | |

**오류 유형 13개** (error_type 허용값):  
`시제 오류` / `관사 누락` / `주술 호응` / `전치사 오류` / `수 일치` / `어순 오류` / `연어 오류` / `레지스터 부적절` / `부정문 오류` / `비교급 오류` / `수동태 오류` / `관계사 오류` / `조동사 오류`

> **AT ↔ 직전 진단 비교 쿼리 패턴**  
> ```sql
> -- 이번 AT에서 지속된 오류: 직전 진단과 동일 error_type이 이번 AT에도 존재
> SELECT e1.error_type
> FROM alt_script_errors e1                                    -- 직전 진단(baseline)의 오류
> JOIN alt_sessions s2 ON s2.previous_session_id = e1.session_id  -- 그 baseline을 참조한 이번 AT
> JOIN alt_script_errors e2 ON e2.session_id = s2.id
>   AND e2.error_type = e1.error_type
> WHERE e1.session_id = :baseline_session_id
> ```

---

## 7. 인덱스 계획

```sql
-- 세션 조회
CREATE INDEX idx_alt_sessions_user_type ON alt_sessions(user_id, test_type);
CREATE INDEX idx_alt_sessions_previous ON alt_sessions(previous_session_id);

-- 응답 조회 (세션 단위 집계)
CREATE INDEX idx_alt_p1_responses_session ON alt_p1_responses(session_id, item_sequence);
CREATE INDEX idx_alt_p2_responses_session ON alt_p2_responses(session_id, item_sequence);
CREATE INDEX idx_alt_p3_turns_session ON alt_p3_turns(session_id, turn_number);

-- 결과 조회
CREATE INDEX idx_alt_results_user ON alt_results(user_id, created_at DESC);
CREATE INDEX idx_alt_results_session ON alt_results(session_id);

-- 오류 비교
CREATE INDEX idx_alt_script_errors_session ON alt_script_errors(session_id, error_type);

-- 문항 출제 (레벨 기반 랜덤)
CREATE INDEX idx_alt_p1_questions_level ON alt_p1_questions(level, is_active);
CREATE INDEX idx_alt_p2_sentences_level ON alt_p2_sentences(level, is_active);
CREATE INDEX idx_alt_p3_topics_active ON alt_p3_topics(is_active, level_min, level_max);
CREATE INDEX idx_alt_p4_topics_active ON alt_p4_topics(is_active, level_min);
```

---

## 8. 테이블 관계 다이어그램

```
users
  └── alt_sessions (user_id)
        ├── alt_sessions (previous_session_id) ← AT-2→AT-1 자기 참조
        ├── alt_p1_responses (session_id) → alt_p1_questions
        ├── alt_p2_responses (session_id) → alt_p2_sentences
        ├── alt_p3_turns (session_id) → alt_p3_topics → alt_p3_probes
        ├── alt_p4_responses (session_id) → alt_p4_topics
        ├── alt_results (session_id)
        └── alt_script_errors (session_id)

[Lookup]
  alt_exam_conversion (level)
  alt_level_benchmarks (level × rubric_key)
```

---

## 9. DDL 작성 및 적용 절차

1. DDL 파일 위치: `speaking.picklass.com/apps/api/prisma/manual-sql/YYYY-MM-DD_P-ALT-tables.sql`
2. 작성 순서: Lookup → 문항 테이블 → alt_sessions → 응답 테이블 → 결과 테이블
3. 각 테이블은 BEGIN/COMMIT 트랜잭션으로 래핑
4. 적용: Supabase Studio SQL Editor에서 직접 실행
5. 적용 후 `pnpm --filter @speaking/api prisma:pull` 로 schema.prisma 동기화
6. `pnpm --filter @speaking/api build` 로 Prisma Client 재생성

---

## 10. 결정된 사항

| 항목 | 결정 | 근거 |
|------|------|------|
| **P3 질문 생성 방식** | **의도(probe) 기반 하이브리드**. opener는 원문 고정, follow_up/deep_dive는 `probe_intent`를 LLM에 주입해 실시간 생성 | 기획서 §8-1 "실시간 난이도 조정" 요구 + 고정 원문의 부자연스러움이 화용성 측정을 훼손하는 문제 회피. §4.4 설계 배경 참조 |

## 11. 미결 사항

| 항목 | 내용 | 결정 필요 시점 |
|------|------|--------------|
| 벤치마크 갱신 주기 | 배치 주기 및 최소 샘플 수 | 파일럿 이후 |
| 오디오 스토리지 | Supabase Storage vs Azure Blob | 인프라 결정 |
| 문항 노출 이력 | 재응시 시 동일 문항 제외 정책 (별도 테이블 필요 여부) | AT 구현 전 |

---

## 12. 사용자 레벨·프로필 확장 (2026-07-09 추가)

> DDL: `speaking.picklass.com/apps/api/prisma/manual-sql/2026-07-09_speaking-profile-and-level.sql`
> 기능 문서: [`../features/마이페이지/2026-07-09_마이페이지-프로필-레벨-저장설계.md`](../features/마이페이지/2026-07-09_마이페이지-프로필-레벨-저장설계.md)

my 페이지의 **현재 레벨·영어 닉네임·학습 알림**을 저장한다. `alt_results`는 PT/AT 세션별 결과만 담아
"현재 레벨 단일 소스"와 "학습 중 조정(무세션) 이력"을 담지 못하는 공백을 메운다.

### 12.1 설계 — 원장(ledger) + 스냅샷(snapshot)

레벨 변화는 이벤트 스트림(PT → 학습조정 → AT → …)이다.

- **스냅샷(현재값)**: speaking 소유 `user_learning_profiles`에 컬럼 추가 → my 페이지·콘텐츠 서빙이 PK 1행 O(1) 조회. 신규 테이블 안 만듦(재사용).
- **원장(이력)**: 신규 `alt_level_history` — 모든 변화 1행씩 append-only. 궤적·성장 리포트 원천.

| 대안 | 기각 사유 |
|------|-----------|
| `users`에 컬럼 추가 | 공유 테이블 + 이력 소실 (§1) |
| `alt_results`만 사용 | 학습 조정(무세션) 못 담음, 현재값 두 소스 union 필요 |
| JSONB 배열 이력 | 궤적 정렬·집계 불리 (§1 JSONB 원칙 위반) |
| 원장만 두고 매번 MAX 조회 | my 페이지·서빙이 매 요청 스캔 → 스냅샷 1컬럼으로 회피 |

### 12.2 `user_learning_profiles` 확장 컬럼

speaking 소유 테이블이라 공유 협의 불필요. (기존: `cefr_level`·`daily_goal_min`·`persona_code` 등)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `english_nickname` | VARCHAR(16), CHECK `^[A-Za-z0-9 ]+$` | 원어민 호명·랭킹. 영문 전용, 유일성 없음. NULL이면 앱에서 `users.name` 폴백 |
| `picklass_level` | SMALLINT, CHECK 1–18 | 현재 레벨 (표시·서빙 단일 소스). `== words_18levels_6000.picklass_rank` |
| `cefr_level` | VARCHAR (기존) | **18변형 라벨 저장**(A1-~C2+, 2026-07-09 결정). `labelForRank(picklass_level)`. 파트너 매핑 키. `alt_level_history.cefr`도 동일 |
| `level_source` | VARCHAR(20) | 'PT'/'AT'/'learning'/'manual'/'onboarding' (신규 소스는 값만 추가) |
| `level_updated_at` | TIMESTAMPTZ | 현재 레벨 최종 설정 시각 |
| `reminder_enabled` | BOOLEAN DEFAULT true | 학습 알림 on/off |
| `reminder_time` | TIME | 매일 지정 시각 (예 09:00) |
| `reminder_phone_mode` | BOOLEAN DEFAULT false | 방해금지(전화 수신 차단) |

부분 인덱스: `idx_ulp_reminder ON (reminder_time) WHERE reminder_enabled` — 스케줄러가 "지금 시각 대상자"만 조회.

> **학습 알림 경계**: 매일 리마인더(시각·전화모드)는 스케줄러가 시각으로 조회하므로 정규화 컬럼으로 분리한다.
> 광의 푸시 카테고리 on/off·수신동의·DND 윈도우는 기존 `users.notification_settings`(JSONB, 공유) 유지.
> 볼륨 등 기기별 취향은 DB에 두지 않고 클라이언트 로컬 저장.

### 12.3 `alt_level_history` — 레벨 변화 이력 원장

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | UUID PK | |
| `user_id` | UUID FK → users | |
| `level` | SMALLINT, CHECK 1–18 | 변화 후 결과 레벨 |
| `previous_level` | SMALLINT | 직전 레벨 (첫 행 NULL) |
| `delta` | SMALLINT | level − previous_level (성장 표시) |
| `cefr` | VARCHAR(4) | 표시용 denormalize |
| `source` | VARCHAR(20) | 'PT'/'AT'/'learning'/'manual'/'onboarding' |
| `session_id` | INTEGER FK → alt_sessions | PT/AT일 때만. 학습조정은 NULL |
| `ref_type` / `ref_id` | VARCHAR(30) / UUID | 학습조정 참조(lesson/module) |
| `reason` | TEXT | 사람이 읽는 사유 |
| `created_at` | TIMESTAMPTZ | |

인덱스: `idx_alt_level_hist_user ON (user_id, created_at DESC)`.

### 12.4 쓰기 흐름 (트랜잭션 1개)

1. `INSERT alt_level_history (... source, session_id 또는 ref_id, previous_level, delta ...)`
2. `UPDATE user_learning_profiles SET picklass_level, cefr_level, level_source, level_updated_at`

- **PT/AT 완료**: `source='PT'|'AT'`, `session_id`=완료 세션, `level`=`alt_results.final_level`.
  헤드라인 레벨만 원장에, 루브릭 상세는 `alt_results` 유지(중복 없음).
- **학습 중 조정**: `source='learning'`, `ref_type/ref_id`, `session_id` NULL.
- AT baseline은 여전히 `alt_sessions.previous_session_id` 체인으로 리졸브(§5.1). 원장은 이 로직 무관.

### 12.5 미결

- ⚠️ `alt_sessions.user_id`가 `integer`로 선언됨(§5.1). `users.id`는 UUID → 타입 불일치. `alt_level_history.user_id`는 UUID로 맞췄으나 `alt_*` 세션 계열 정합성은 별도 처리 필요.
- `alt_*` 모델 전반이 `schema.prisma` 미등록 — 적용 후 모델 직접 작성(`prisma:pull` 금지).
