# P-ALT 콘텐츠 개발 계획

> 기준 기획서: [`./P-ALT_Speaking_v8.md`](./P-ALT_Speaking_v8.md)  
> DB 설계: [`./P-ALT_DB_설계.md`](./P-ALT_DB_설계.md)  
> 작성일: 2026-07-03

---

## 1. 개요

P-ALT 4개 파트(P1–P4)에서 출제되는 모든 문항·문장·주제 카드를 준비하는 계획.  
Gemini를 사용해 초안을 대량 생성하고, 사람이 레벨 적합성·문장 자연스러움·함정 옵션 품질을 검토하는 **생성 → 검토 → 입력** 3단계 워크플로우로 진행한다.

---

## 2. 파트별 물량 목표

| 파트 | 항목 | 레벨 범위 | 레벨당 | 총 목표 | 근거 |
|------|------|----------|--------|--------|------|
| **P1** | 4지선다 문항 | L1–L18 | 40문항 | **720문항** | CAT 재노출 방지 + 재응시 대비 |
| **P2** | 낭독 문장 | L1–L18 | 12문장 | **216문장** | 최대 7문장 CAT + 여유 5문장 |
| **P3** | 대화 주제 | — | 공통(범용) | **50주제** | 주제 고갈·반복 방지. AT 전용 제작 안 함(§6.1) |
| **P3** | 주제별 probe(측정 의도) | — | 주제당 4개 | **약 200 probe** | opener 1 + follow_up/deep_dive 3. 원문이 아닌 의도만 작성 |
| **P4** | 주제 발표 카드 | L7+ | 공통(범용) | **50카드** | 재응시 중복 방지. AT 전용 제작 안 함(§7.3) |

**전체 합계**: 문항류 약 1,236개

> **P3는 "질문 원문"이 아니라 "측정 의도(probe)"를 작성한다.** 챗봇이 사용자 답변에 반응해 실시간으로 질문을 생성하므로(기획서 §8-1), 고정 원문을 강제하면 대화가 부자연스러워지고 화용성 측정이 훼손된다. 상세 배경은 DB 설계 문서 [§4.4](./P-ALT_DB_설계.md) 및 본 문서 §6 참조.

---

## 3. 레벨 기준 참조표

### 3.1 레벨(picklass_rank)·WPM·문장 길이 대응 (P1·P2 공통) — L1~L18

> **기준 개정 배경 (2026-07-03)**
>
> 1. **WPM 하향**: 기획서 §7-1은 6밴드(L1 80~) 기준이었으나 **실제 학습자 결과상 WPM 목표가 과도하게 높다**는 관측에 따라, L1=50 / L18=148을 앵커로 18레벨 선형 보간(레벨당 약 +5.8)하여 하향 조정한다.
> 2. **레벨 축 = CEFR(picklass_rank)로 통일**: 어휘 레벨링은 빈도(COCA)가 아니라 **`words_18levels_6000.picklass_rank`(CEFR 6밴드 × -/기본/+ = 18단계)** 를 단일 권위 기준으로 삼는다. 이는 어휘 테이블 설계 문서 [§3.7](./2026-07-03_어휘테이블-정리-및-18단계설계.md)이 "한국 EFL 체감 난이도는 원어민 코퍼스 빈도가 아니라 CEFR·EFL 노출순서에 부합한다"며 빈도 단일축을 명시적으로 폐기한 결정과 정합한다.
> 3. **P-ALT 레벨 L1~L18 = picklass_rank 1~18** 로 1:1 대응한다.

| 레벨 | picklass_rank | picklass_level | CEFR | 목표 WPM | 문장 길이(단어) |
|:---:|:---:|:---:|:---:|:---:|:---:|
| L1 | 1 | A1- | A1 | 50 | 5–6 |
| L2 | 2 | A1 | A1 | 56 | 6–7 |
| L3 | 3 | A1+ | A1 | 62 | 7–8 |
| L4 | 4 | A2- | A2 | 67 | 8–9 |
| L5 | 5 | A2 | A2 | 73 | 9–10 |
| L6 | 6 | A2+ | A2 | 79 | 10–11 |
| L7 | 7 | B1- | B1 | 85 | 11–12 |
| L8 | 8 | B1 | B1 | 90 | 12–13 |
| L9 | 9 | B1+ | B1 | 96 | 13–14 |
| L10 | 10 | B2- | B2 | 102 | 14–15 |
| L11 | 11 | B2 | B2 | 108 | 15–16 |
| L12 | 12 | B2+ | B2 | 113 | 16–17 |
| L13 | 13 | C1- | C1 | 119 | 17–19 |
| L14 | 14 | C1 | C1 | 125 | 19–20 |
| L15 | 15 | C1+ | C1 | 131 | 20–22 |
| L16 | 16 | C2- | C2 | 136 | 22–24 |
| L17 | 17 | C2 | C2 | 142 | 24–26 |
| L18 | 18 | C2+ | C2 | 148 | 26+ |

> **⚠ 목표 WPM·문장 길이는 P2(낭독) 전용 지표다.** P1(어휘)은 WPM을 측정하지 않으며, 문장 길이도 §3.1 range를 따르지 않고 **별도 규칙(≤10단어)** 을 쓴다(§4.1 배경 참조). 두 파트에 공통으로 적용되는 것은 **레벨 축(`picklass_rank`/`picklass_level`/CEFR)** 뿐이다.

**P2 지표의 형태가 다른 이유**

| 지표 | 형태 | 이유 |
|------|------|------|
| **목표 WPM** | 단일 center 값 | §3.3 채점 판정(중=목표 ±15%, 하=±20% 이탈)이 min/max를 파생 → range 별도 저장 불필요. `alt_p2_sentences.target_wpm` / `alt_p2_responses.target_wpm`에 center 값 저장 |
| **문장 길이** | range | 채점 지표가 아닌 **P2 문장 작성 가이드**. 실제 문장의 `alt_p2_sentences.word_count`는 실측값으로 저장 |

> **어휘 레벨은 `picklass_rank`로 검증한다 (COCA 아님)**: 문항의 어휘 레벨 적합성은 COCA 빈도 밴드가 아니라 `words_18levels_6000.picklass_rank`로 판정한다. P1은 target word의 `picklass_rank = 레벨`, P2는 문장이 **`picklass_rank = 레벨`인 단어를 최소 1개 포함**하고 **나머지 내용어는 `picklass_rank ≤ 레벨`**. 상세 파이프라인은 §3a 참조.
>
> **`coca_max_rank`는 참고 지표**: `alt_p2_sentences.coca_max_rank`(= 테이블 `frequency_rank` 최댓값)는 레벨 판정 기준이 아니라 **부가 참고값**으로만 저장한다(corpus 단어만 존재, supplement는 NULL).
>
> **⚠ WPM 실측 재검증 필요**: 목표를 하향했으므로 §5.3 Azure TTS 실측 시 **새 목표값 기준**으로 재확인한다. 특히 저레벨(L1 50 WPM)은 TTS가 그만큼 느린 자연스러운 낭독을 합성할 수 있는지 확인이 선행되어야 한다.

### 3.2 P1 CAT 판정 기준

| 판정 | 조건 | 다음 문항 |
|------|------|----------|
| 상 (Exceeds) | 정답 + 응답 3초 이내 | +2 상향 |
| 중 (Meets) | 정답 + 응답 3초 초과 | +1 상향 |
| 하 (Below) | 오답 | PT: -1 / AT: 1회 유지→연속 2회시 -1 |

### 3.3 P2 복합 점수 판정 기준

| 판정 | WPM | 단어 일치율 | Confidence | 다음 문장 |
|------|-----|-----------|-----------|----------|
| 상 | 목표 초과 | 95%+ | 0.85+ | +2 (3개 모두 충족) |
| 중 | 목표 ±15% | 80–94% | 0.70–0.84 | +1 |
| 하 | 목표 20% 이탈 | 79%↓ | 0.70↓ | PT: -1 / AT: 1회 유지→연속 2회 -1 |

---

## 3a. words_18levels_6000 기반 출제 파이프라인 (P1·P2 공통)

> 원천 테이블 상세: [`2026-07-03_어휘테이블-정리-및-18단계설계.md`](./2026-07-03_어휘테이블-정리-및-18단계설계.md)  
> 산출 SQL: `speaking.picklass.com/apps/api/prisma/manual-sql/2026-07-03_words-18levels-6000*.sql`

P1·P2 콘텐츠는 사내 어휘 코퍼스 `words_18levels_6000`(6,532행)을 기반으로 생성한다. 이 테이블의 `picklass_rank`(1~18)가 **P-ALT 레벨 L1~L18과 1:1 대응**하므로, 레벨링을 새로 계산하지 않고 그대로 활용한다.

### 3a.1 공통 — picklass_rank로 레벨별 랜덤 추출

```sql
-- 레벨별로 필요한 만큼 단어를 랜덤 추출 (P1: target word / P2: 허용 어휘 풀 확인용)
SELECT word, picklass_level, pos          -- pos는 있으면(supplement) 오답 생성 힌트로 활용
FROM words_18levels_6000
WHERE picklass_rank = :level
ORDER BY random()
LIMIT :count;
```

- 레벨 내 난이도 스펙트럼을 고르게 가져가려면 `random()` 대신 `difficulty_order` 순번 구간 샘플링으로 대체 가능.
- `def_kr`·`pos`는 supplement 533행에만 존재하므로 **정답·오답을 테이블에 의존하지 않는다**(아래 §3a.2 참조). def_kr 전체 백필은 별도 자산 과제이며 본 파이프라인의 전제가 아니다.

### 3a.2 P1 — Gemini가 문항 전체 생성 (def_kr 불필요)

테이블은 **target word만 공급**하고, 문장·정답·오답은 Gemini가 생성한다.

> **P1 문항 스타일과 "안다"의 정의 (배경)**
>
> P1 문항은 **밑줄 친 target word를 포함한 문장 + 4지선다(한국어 뜻)** 형식이다. 수험자는 밑줄 단어의 뜻을 4개 보기에서 고른다. 이 형식은 어휘를 **재인(recognition)** 수준에서 측정한다 — 뜻을 명확히 알든 어렴풋이 느끼든 정답을 고르면 "아는 단어"로 간주한다. 이는 P-ALT가 측정하려는 **Vocab Size(잠재 어휘량)** 의 정의와 일치한다(기획서 §6-1).

> **⭐ 문장은 "비정의 캐리어(non-defining carrier)" — 문맥 단서 누출 차단 (핵심 배경, 2026-07-03 실제 생성에서 확립)**
>
> P1은 **어휘력 테스트**이지 독해력 테스트가 아니다. 문장이 target word의 뜻을 **문맥으로 유추 가능하게** 만들면(예: "The camel lives in the hot **desert**." → camel·hot이 desert를 알려줌), 단어를 몰라도 정답을 맞혀 측정이 무효가 된다. 이를 **문맥 단서 누출(context clue leakage)** 이라 한다.
>
> 해결책은 세계 표준 어휘 크기 시험(**Paul Nation's Vocabulary Size Test**)의 방식 — **비정의 캐리어 문장**이다. 문장은 **품사와 문법 자리만** 보여주고 **의미 단서는 0**이 되도록 중립 프레임으로 만든다:
> - 명사: `This is a X.` / `I saw the X.` / `They have a X.`
> - 동사: `They X it.` / `People X this.` / `We will X it.`
> - 형용사: `It is X.` / `It was very X.`
> - 부사: `He did it X.`
>
> 규칙: 주변을 **중립 대명사(I, they, it, this, people)** 로만 채우고, target과 **의미적으로 연관된 단어를 절대 넣지 않는다.** 그러면 뜻은 오직 단어를 알아야만 알 수 있다. 문장은 짧게(≤10단어) 유지한다.

```
입력: word = "desert", picklass_level = "A2"(=L5)

Gemini 출력 (비정의 캐리어):
{
  sentence: "This is a desert.",   // camel/hot 같은 의미 단서 없음
  target_word: "desert",
  correct: "사막",
  distractors: ["숲", "디저트", "도시"]
}
```

**생성 시 배제 규칙 (2026-07-03 실측으로 확립)**

- **고유명사 배제**: 코퍼스에 소문자로 섞인 국가·브랜드명(brazil 등)은 어휘 측정에 부적절. Gemini가 고유명사면 `is_bad`(또는 `skip`) 플래그로 배제 → 같은 레벨에서 대체 단어 재추출.
- **외래어(음차 정답) 배제**: 정답이 영어발음을 한글로 옮긴 음차(재즈·티셔츠·위스키 등)면 단어를 읽을 줄만 알아도 트리비얼하게 맞힘. 이런 target은 배제하고 **비(非)외래어 단어로 교체**. Gemini가 "정답이 음차면 `is_bad`"로 판정.
- **마크업 금지**: `<u>`, `_`, `*` 등 어떤 강조 표기도 문장에 넣지 않는다(밑줄은 프론트가 target_word로 렌더링). 생성 후 잔여 태그는 제거.
- `pos`가 있는 단어는 프롬프트에 전달 → **같은 품사 오답** 유도로 함정 품질↑.
- `alt_p1_questions.coca_rank`에 테이블 `frequency_rank` 복사(corpus 단어만, supplement는 NULL).

**검증 게이트 (전수 자동 QA)**

| 게이트 | 방법 | 처리 |
|--------|------|------|
| **클로즈 누출 테스트** | target을 빈칸(`____`)으로 가리고 문장+4지선다를 LLM에 제시 → "문맥만으로 정답 유추 가능?" | 유추 가능(누출)이면 문장 재생성. 수렴까지 라운드 반복 |
| **가산성·관사 문법** | 캐리어 템플릿이 불가산/복수명사에 `a/an`을 붙이는 오류(예: "a money") 검사 | 비문이면 가산성 준수 프롬프트로 재생성 |
| **결정론적 QA** | ≤10단어 / 밑줄단어 문장 포함(굴절 표면형 일치) / 옵션 4개 유일 / 정답 위치 균등 | 위반 시 보정 |

> **굴절형 표면형 일치**: Gemini가 단어를 굴절시켜 쓰면(charity→charities) DB `target_word`를 **문장 표면형으로 맞춘다**(밑줄 렌더링 정합성). 캐리어는 기본형을 쓰므로 대부분 자동 해소.
>
> **기능어 한계**: 순수 비교 기능어(than 등)는 구조=의미라 어떤 문장도 뜻이 드러나 완전 누출 차단이 불가능. 소수(1~2건)는 문서화된 한계로 수용하거나 내용어로 교체.
>
> **실적 (2026-07-03, 720문항)**: 클로즈 누출 14→4→2건 수렴, 문법 오류 12건 수정, 음차 37건·고유명사 배제. 최종 정답 위치 180×4 균등.

### 3a.3 P2 — 레벨 단어를 먼저 선정 → 문장 생성 → picklass_level로 검증

> **레벨 단어를 seed로 먼저 뽑는 이유 (배경)**
>
> P2는 **리딩 플루언시(낭독 유창성) 테스트**다. 레벨 N 문장이라면 **`picklass_rank = N`인 단어를 최소 1개 이상 포함**해야 한다 — 그렇지 않으면 문장 내 모든 단어가 N 미만이 되어, 실제로는 더 낮은 레벨의 문장이 되어버린다(레벨 미달). 따라서 "CEFR 밴드만 주고 생성"하면 저레벨 단어로만 이뤄진 문장이 나올 위험이 있다. 이를 막기 위해 **레벨 N 단어를 랜덤으로 먼저 선정(seed)** 하고, 그 단어를 반드시 포함하도록 문장을 생성한다.

```
1. seed 선정: words_18levels_6000에서 picklass_rank = :level 단어 1~2개 랜덤 추출(§3a.1)

2. 생성: Gemini에 seed 단어 + picklass_level + 문장 길이 range(§3.1) 지정
   → seed 단어를 반드시 포함하는 문장 생성

3. ★검증 (테이블 조회, picklass_level 기준):
   - 문장을 lemma 토큰화 → 각 content word를 words_18levels_6000에서 조회
   - (a) picklass_rank = :level 단어가 1개 이상 존재  (레벨 도달 확인)
   - (b) 모든 content word의 picklass_rank ≤ :level     (레벨 초과 없음)
   - (a)·(b) 모두 만족하면 통과, 아니면 재생성/단어 교체

4. coca_max_rank = 문장 내 content word의 max(frequency_rank)   // 참고용, 판정 기준 아님
```

- 검증 축은 **`picklass_level`(CEFR)이며 `coca_max_rank`가 아니다.** 어휘 테이블 §3.7이 빈도 단일축을 폐기했으므로 정합적이다.
- seed 단어가 문장에 남았는지도 검증 (a) 단계에서 함께 확인된다.
- 함수어(the/is/of)·고유명사·변화형은 화이트리스트 또는 lemma 정규화 후 조회(테이블은 표제어 lemma).

### 3a.4 이 방식의 이점

- 레벨링을 재발명하지 않음 — `picklass_rank`를 그대로 사용.
- P1 오답이 **같은 품사·인접 레벨 실제 어휘** 기반이라 무작위 오답보다 정교.
- P2가 **레벨 단어를 반드시 포함**(seed)하여 "레벨 미달 문장"을 원천 차단. 어휘 레벨이 **주관 판단 → 테이블 조회로 객관화**되고, §5.3 WPM 실측과 함께 이중 검증(어휘·유창성).

---

## 4. P1 — Vocab Size 문항

### 4.1 문항 구조

```
문장: This is a desert.          (비정의 캐리어 — 의미 단서 없음)
          ‾‾‾‾‾‾ (밑줄 = target_word)
① 사막  ② 숲  ③ 디저트  ④ 도시
정답: ①
```
> 보기는 한국어 뜻(4지선다). target_word는 문장에 그대로 노출되고 밑줄로 표시된다.

**측정 철학**: P1은 어휘를 **재인(recognition)** 수준에서 측정한다 — 밑줄 단어의 뜻을 명확히 알든, 어렴풋이 느끼든 4지선다에서 정답을 고르면 "아는 단어"로 간주한다(§3a.2 배경).

- **target word는 `words_18levels_6000`에서 `picklass_rank = 레벨`로 추출**한다(§3a.1). 레벨 적합성은 테이블이 보장.
- **문장은 비정의 캐리어(≤10단어)** — 문맥으로 뜻이 유추되지 않는 중립 프레임(§3a.2 ⭐ 배경). Nation VST 방식.
- **고유명사·외래어(음차 정답) target 배제**, 마크업 금지(§3a.2).
- 오답 3개는 **매력적인 함정** — 음운 유사어·의미 혼동어·레지스터 불일치어. `pos`가 있으면 **같은 품사** 뜻으로 구성.

### 4.2 Gemini 생성 프롬프트 가이드

> target word는 테이블에서 공급되고, Gemini는 캐리어 문장·정답·오답만 생성한다(§3a.2).

```
[시스템]
You design a vocabulary-size test in the style of Paul Nation's VST.
This is a VOCABULARY test, not a reading test. For each target word, build ONE
"find the meaning of the underlined word" item with a NON-DEFINING carrier
sentence: short, grammatical, giving ZERO clue to the word's meaning.

RULES:
- If the word is a PROPER NOUN, or a LOANWORD whose Korean meaning would just be a
  transliteration (재즈 jazz, 티셔츠 t-shirt, 위스키 whiskey), set "is_bad": true and skip.
- Sentence MAX 7-10 words, plain text (NO markup: <u>, _, *), word appears verbatim.
- Use only neutral filler (I/you/he/she/they/it/this/that/we/people/is/are/was/will/
  have/do/the/a/my/some/very). NEVER include a word semantically related to the target.
- COUNTABILITY: for uncountable/plural nouns do NOT use "a/an"
  (use "This is X." / "They have X.").
- POS frames: noun "This is a X." · verb "They X it." · adj "It is X." · adv "He did it X."

[사용자]
target_word: "desert"
pos: "noun"                 // may be null; if present, keep distractors same POS

Produce:
1. sentence: a non-defining carrier sentence containing target_word verbatim
2. correct: a GENUINE Korean meaning (translation), NEVER a transliteration
3. distractors: 3 plausible-but-wrong Korean meanings, same POS

Output as JSON.
```

### 4.3 검토 체크리스트 (전수 자동 QA — §3a.2 검증 게이트)

- [ ] 밑줄 단어가 문장에 그대로(verbatim) 포함 (굴절 시 target_word를 표면형으로 교정)
- [ ] **클로즈 누출 테스트 통과** — 단어 빈칸 시 문맥만으로 정답 유추 불가
- [ ] 문장 ≤10단어, 마크업(`<u>`/`_`/`*`) 없음
- [ ] **가산성·관사 문법 정상** ("a money" 같은 비문 없음)
- [ ] 정답이 음차(외래어 발음)가 아닌 실제 뜻인가
- [ ] 오답 3개가 매력적 함정이고 유일(중복 없음), pos 있으면 같은 품사
- [ ] 정답 위치가 레벨별 균등(①②③④ 각 10/40)

### 4.4 배치 계획

| 우선순위 | 레벨 | 사유 |
|---------|------|------|
| 1차 | L6–L12 | PT 시작점(L9) 중심, 가장 높은 CAT 출제 빈도 |
| 2차 | L1–L5 | 저레벨 수험자 대비 |
| 3차 | L13–L18 | 고레벨 수험자 대비 |

---

## 5. P2 — 문장 낭독 CAT

### 5.1 문장 품질 기준

- **레벨 기준 = `picklass_level`(CEFR)**. `picklass_rank = 레벨`인 seed 단어를 먼저 뽑아 문장에 반드시 포함시키고(§3a.3), 검증으로 (a) 레벨 단어 1개 이상 존재 + (b) 모든 내용어 `picklass_rank ≤ 레벨`을 보장한다.
- 단어 수가 해당 레벨 §3.1 range에 맞아야 한다.
- 내용이 중립적이고 문화적으로 민감하지 않아야 한다.
- 연음(liaison)은 L7 이상부터 허용, L10 이상은 포함 권장.
- 학술/전문 어휘(AWL 포함)는 L16 이상에만 사용.

### 5.2 Gemini 생성 프롬프트 가이드 (seed → 생성 → 검증)

> §3a.3의 3단계 흐름: ① `picklass_rank=N` seed 단어 랜덤 추출 → ② seed를 포함하는 문장 **생성** → ③ 테이블 조회로 `picklass_level` 기준 **검증**.

```
[시스템 · 2단계 생성]
You are designing reading-aloud sentences for a Korean EFL speaking test.
Sentences will be read aloud and assessed by Azure Speech SDK.
Level is defined by CEFR (picklass_level), NOT by corpus frequency.

Level [N] criteria (use exact values from §3.1 table):
- picklass_level / CEFR: [e.g. B1- / B1]
- Word count range: [e.g. L7 → 11–12]
- Target WPM (single center value): [e.g. L7 → 85]  // for TTS check only
- Liaison: [not required / optional / recommended]

[사용자]
seed_words: ["observe", "microscope"]   // picklass_rank = N words, MUST appear
Generate 6 reading-aloud sentences for CEFR [band].
Requirements:
- Each sentence MUST contain at least one seed_word verbatim
- Natural, complete sentences within the word-count range
- All other vocabulary at or below the CEFR band (no words above the band)
- Varied topics (daily life, nature, work, technology, etc.)
- Avoid tongue twisters or unusual proper nouns

Output JSON: [{sentence, word_count, seed_used, has_liaison}]
```

**3단계 검증 (테이블 조회, `picklass_level` 기준, §3a.3)**: 생성 문장을 lemma 토큰화 → 각 content word를 `words_18levels_6000`에서 조회 → **(a)** `picklass_rank = N` 단어 1개 이상 존재 + **(b)** 모든 단어 `picklass_rank ≤ N` 을 확인. 둘 중 하나라도 실패 시 재생성. `coca_max_rank`는 참고값으로만 기록(**검증 기준 아님**).

### 5.3 WPM 실측 검증 절차 (어휘 검증과 별개, 3단계)

§3a.3 어휘 레벨 검증을 통과한 문장에 대해, Azure TTS로 낭독 WPM 적정성을 추가 확인한다.

1. Azure TTS로 각 문장을 §3.1 목표 WPM으로 합성
2. 원어민 수준 발음으로 읽을 때 실제 WPM이 목표 ±15% 내에 드는지 확인
3. 이탈 문장은 단어 수 조정 또는 교체 (조정 후 §3a.3 어휘 검증 재실행)

> **이중 검증**: 어휘 레벨(§3a.3 테이블 조회, picklass_level) + 유창성/WPM(§5.3 TTS 실측). 두 검증을 모두 통과해야 P2 문장으로 확정한다.

### 5.4 배치 계획

P1과 동일한 우선순위. L7–L11 먼저 제작 후 양 끝 레벨 확장.

---

## 6. P3 — 일상 대화 주제·probe

> **작성 대상은 "질문 원문"이 아니라 "측정 의도(probe)"다 — 설계 배경**
>
> 기획서 §8-1은 P3를 "챗봇이 응답 수준을 **실시간으로 판단하여 질문 복잡도를 조정**하는 자유 대화"로 규정한다. 여기에 완성된 질문 문장을 미리 써두고 그대로 출제하면 두 가지 문제가 생긴다:
>
> - **부자연스러움**: 사용자가 이미 말한 내용을 무시하고 대본상 다음 질문을 던지면 대화가 어긋난다.
> - **측정 타당성 훼손**: 이 어긋남은 UX 문제에 그치지 않는다. 대화 흐름이 깨지면 사용자가 당황하고, 그 혼란이 **화용성(pragmatics) 점수를 부당하게 깎는다.** 자연스러운 대화 자체가 화용성 측정의 전제이기 때문이다.
>
> 그래서 콘텐츠 팀이 작성하는 것은 **각 턴에서 끌어내야 할 측정 의도(probe_intent)** 와 **참고용 예시 문장(example_question)** 이다. 실제 발화 질문은 LLM이 사용자 답변에 반응하며 자유롭게 생성하고, probe의 의도만 대화에 자연스럽게 녹인다. 예시 문장은 "이런 식으로 물으면 된다"는 참고일 뿐 강제가 아니므로, **완성도 높게 폴리싱할 필요가 없어 제작 부담이 낮다.**
>
> 이 방식으로 **자연스러운 대화 · 루브릭 측정 통제 · AT ↔ 직전 진단 비교 공정성**이 동시에 성립한다. 상세는 DB 설계 [§4.4 `alt_p3_probes`](./P-ALT_DB_설계.md) 참조.

### 6.1 주제 유형

| 유형 | test_type | 설명 | 목표 수 |
|------|-----------|------|--------|
| 범용 일상 | PT / BOTH | 직업·취미·여행·음식·가족 등 | **50주제** (전량) |
| ~~학습 콘텐츠 연계~~ | ~~AT~~ | — | **미제작 (아래 결정 참조)** |

> **AT용 "학습 콘텐츠 연계 주제"는 사전 제작하지 않는다 (2026-07-03 결정)**
>
> AT용 주제를 학습 모듈과 **정적으로 사전 매핑하지 않는다.** 대신 **PT/BOTH 범용 주제(50주제)만 제작**하여 PT·AT 공통으로 사용한다.
>
> **향후 진행 시 바람직한 방향**: AT용 P3는 정적 주제 풀 매핑이 아니라, **응시 시점에 해당 사용자가 실제로 학습한 콘텐츠를 검토하여 그에 맞는 P3 문항(probe/질문)을 런타임 동적 생성**하는 방식이 바람직하다. 사용자마다 학습 내용이 다르므로 개인화된 연계가 사전 주제 풀보다 타당하다. (전제: speaking 학습 모듈 이력 데이터 접근 — §12 미결과 연동)

### 6.2 주제·probe 구조

각 주제는 **opener → follow_up → deep_dive** 순서의 probe 흐름을 가진다.  
opener만 예시 문장을 원문 그대로 출제할 수 있고(첫 턴은 맥락이 없어 자연스러움), 나머지는 의도 기반으로 LLM이 실시간 생성한다.

```
주제: Daily Routines (level_min: 1, level_max: 9)

[opener · 원문 고정 가능]
  probe_intent:     하루 일과를 개방형으로 서술하게 해 기본 시제·빈도부사 사용을 관찰
  example_question: "Can you tell me about your typical morning routine?"
  target_rubric:    grammar

[follow_up · 의도 기반 실시간 생성]
  probe_intent:     사용자가 언급한 활동의 구체적 사례를 파고들어 어휘 다양성을 관찰
  example_question: "What do you usually have for breakfast?"
  target_rubric:    vocab

[deep_dive · 의도 기반 실시간 생성]
  probe_intent:     선호·이유를 묻는 의견 질문으로 논리 연결·표현 수준을 관찰
  example_question: "Do you prefer a fixed routine or a flexible schedule? Why?"
  target_rubric:    expression

[deep_dive · 의도 기반 실시간 생성]
  probe_intent:     과거-현재 변화를 비교 서술하게 해 화용적 확장·자발성을 관찰
  example_question: "How has your routine changed over the past few years?"
  target_rubric:    pragmatics
```

### 6.3 Gemini 생성 프롬프트 가이드

콘텐츠 생성 단계에서는 **probe 세트**를 만든다. (런타임에 LLM이 실제 질문을 생성하는 것은 별개이며, CAT 엔진 개발 시 다룬다.)

```
[시스템]
You are designing conversation "probes" for a Korean EFL speaking assessment 
modeled after IELTS Speaking Part 1.

A probe is NOT a fixed question. It defines what each conversational turn 
should elicit, so a live chatbot can generate a natural question that reacts 
to the learner's previous answer while still covering the measurement goal.

Topic requirements:
- Suitable for Korean adult learners
- General-purpose topic usable in BOTH PT and AT (no learning-content-specific topics)
- level range: [min–max]
- Avoid politically sensitive, religious, or overly personal topics

[사용자]
Generate 1 P3 topic with 4 probes for the P-ALT test.

Each probe:
- turn_order, question_role: "opener" | "follow_up" | "deep_dive"
- probe_intent: what this turn must elicit (the measurement goal), in Korean
- example_question: ONE reference English question (not mandatory at runtime)
- target_rubric: "grammar" | "vocab" | "expression" | "pragmatics"

Ensure the 4 probes together cover grammar, vocab, expression, and pragmatics.

Output:
{
  title: "...",
  category: "...",
  level_min: N,
  level_max: N,
  probes: [
    {turn_order: 1, question_role: "opener",
     probe_intent: "...", example_question: "...", target_rubric: "grammar"},
    ...
  ]
}
```

### 6.4 주제 카테고리 배분 (범용 50주제 기준)

> PT·AT 공통 범용 주제 전량(§2·§6.1). 이전 "PT 30 + AT 20" 분리 배분표를 **범용 50주제 단일 배분**으로 갱신(2026-07-03). CAT 시작점(L9)·학습자 밀도가 높은 중저레벨(일상·직업·취미, L1–L12)에 증분을 집중해 재출제 없이 세션당 주제를 확보한다.
>
> **아래는 실제 생성·DB 적재된 배분(2026-07-03 기준)이다.** 전 레벨 구간에서 §10.1 최소 3주제 조건을 충족한다(L1: 일상 12 / L9 중심: 일상12+직업9+취미9+여행8 / L18: 사회7+기술5=12).

| 카테고리 | 주제 수 | 레벨 범위 |
|---------|--------|----------|
| 일상·생활 | 12 | L1–L9 |
| 직업·학업 | 9 | L5–L12 |
| 취미·여가 | 9 | L5–L12 |
| 여행·장소 | 8 | L7–L15 |
| 사회·트렌드 | 7 | L10–L18 |
| 기술·미래 | 5 | L12–L18 |

### 6.5 검토 체크리스트

- [ ] 한국 문화 맥락에서 자연스러운 주제인가
- [ ] opener probe의 예시 문장이 Yes/No로 끝나지 않는가 (개방형)
- [ ] deep_dive probe가 의견·이유·비교를 요구하는가
- [ ] 주제당 4개 probe가 grammar·vocab·expression·pragmatics를 **골고루 커버**하는가
- [ ] `probe_intent`가 "무엇을 측정할지"를 명확히 서술하는가 (단순 질문 재기술 금지)
- [ ] 주제가 PT·AT 공통으로 쓸 수 있는 범용 주제인가 (AT 전용 학습연계 주제는 제작하지 않음, §6.1)

---

## 7. P4 — 주제 발표 카드

### 7.1 카드 구조

IELTS Speaking Part 2 포맷. 화면에 표시되는 구성 요소:

```
키워드: [환경] [재활용] [생활 습관]

발표 질문:
Talk about a small change you made in your daily life
to help the environment.
Include what the change was, why you made it,
and how it has affected your life.
```

### 7.2 Gemini 생성 프롬프트 가이드

```
[시스템]
Design IELTS Speaking Part 2-style topic cards for Korean EFL learners.
Each card has 3 keywords and 1 speaking question.
Target: learners at P-ALT Level 7 and above (B1–C2).

[사용자]
Generate 5 P4 topic cards.
Each card:
- keyword_1~3: 3 Korean keywords shown on screen (2–4 syllables each)
- question: English speaking prompt (2–3 sentences)
  Must include: what/who + why/how + impact/reflection structure
- category: topic domain (general-purpose, usable in both PT and AT)

Output as JSON array.
```

### 7.3 카테고리 배분 (50카드 기준)

| 카테고리 | 카드 수 | 비고 |
|---------|--------|----------|
| 환경·사회 | 12 | 공통(범용) |
| 기술·디지털 | 10 | 공통(범용) |
| 교육·경험 | 10 | 공통(범용) |
| 직업·꿈 | 9 | 공통(범용) |
| 문화·여행 | 9 | 공통(범용) |
| ~~학습 콘텐츠 연계~~ | — | ~~AT 전용~~ 미제작 (§6.1 결정과 동일) |

> **P4 AT 전용 카드도 사전 제작하지 않는다** — P3와 동일 결정(§6.1). 범용 카드 50장만 제작해 PT·AT 공통 사용하고, 학습 연계는 향후 런타임 동적 생성 방향. 정적 카드는 전부 범용이라 `alt_p4_topics`에 `test_type` 컬럼 없음.

### 7.4 검토 체크리스트

- [ ] 1분 내 의미 있는 발화가 가능한 주제인가
- [ ] 키워드 3개가 질문 내용과 직접 연결되는가
- [ ] "tell me about a time when..." 형식보다 의견·분석형인가
- [ ] L7 수준에서도 시도할 수 있는 난이도인가

---

## 8. 생성 워크플로우

```
단계 0 — 원천 추출 (§3a.1)
  ↓  words_18levels_6000에서 picklass_rank=레벨로 단어 랜덤 추출
  ↓  P1: target word 세트 / P2: seed 단어(레벨 N) 세트

단계 1 — Gemini 생성
  ↓  P1: target word별 짧은 문장(≤10단어)+정답+오답 생성 (§3a.2)
  ↓  P2: seed 단어를 포함하는 문장 생성 (§3a.3, picklass_level+문장길이)

단계 2 — 검증·보정
  ↓  어휘 레벨 검증
  ↓    · P1: target word의 picklass_rank = 레벨 (테이블 보장)
  ↓    · P2: seed(레벨 단어) 1개↑ 포함 + 모든 내용어 picklass_rank ≤ 레벨 (§3a.3, 실패 시 재생성)
  ↓  자연스러운 영어 표현 검토 · P1 오답 매력도(같은 pos) 확인
  ↓  P2 WPM 실측 (Azure TTS, §5.3)
  ↓  탈락 항목 교체 또는 레벨 이동

단계 3 — DB 입력
  ↓  검증 완료된 JSON → INSERT SQL 생성
  ↓  Supabase Studio에서 실행
  ↓  is_active = true로 활성화
```

---

## 9. 단계별 제작 일정

| 주차 | 작업 | 산출물 |
|------|------|--------|
| 1주차 | P1 L6–L12 (40문항×7레벨 = 280문항) | DB 입력 완료 |
| 2주차 | P1 L1–L5 + L13–L18 (40×11 = 440문항) | DB 입력 완료, P1 총 720문항 완성 |
| 3주차 | P2 전 레벨 (12×18 = 216문장) + WPM 실측 | DB 입력 완료 |
| 4주차 | P3 주제 50개 + probe 약 200개 | DB 입력 완료 |
| 5주차 | P4 카드 50개 | DB 입력 완료 |
| 6주차 | 콘텐츠 QA (전체 샘플 검증) + 미달 레벨 보충 | 최종 확인 완료 |

---

## 10. 품질 기준 및 QA

### 10.1 레벨별 최소 문항 수 (서비스 오픈 기준)

CAT 엔진이 동작하려면 각 레벨에 **최소 10문항(P1)·6문장(P2)**이 있어야 한다.  
전체 720·216이 확보되기 전에도 아래 최소값이 갖춰지면 파일럿 가능.

| 파트 | 레벨당 최소 | 비고 |
|------|-----------|------|
| P1 | 10문항 | CAT 5턴 + 중복 방지 여유 |
| P2 | 6문장 | CAT 3–7문장 + 여유 |
| P3 | 3주제 (각 4 probe) | 동일 주제 재출제 방지 |
| P4 | 5카드 | 재응시 중복 방지 |

### 10.2 QA 체크 포인트

- **중복 단어 확인**: 동일 target_word가 같은 레벨에 2개 이상 없는지
- **문화 중립성**: 특정 종교·정치·성별 편향 표현 없는지
- **P2 발음 가능성**: 한국 학습자가 발음하기 극단적으로 어려운 조합(sh/th 연속 등) 과도하지 않은지
- **P3/P4 중복 주제**: 같은 맥락 주제가 2개 이상 연속 출제되지 않는 로직 (서비스 레이어에서 처리)

---

## 11. 결정된 사항

| 항목 | 결정 | 근거 |
|------|------|------|
| **P3 질문 생성 방식** | **의도(probe) 기반 하이브리드**. 콘텐츠는 probe(측정 의도)만 작성하고, 런타임에 LLM이 사용자 답변에 반응해 실제 질문을 생성. opener만 예시 문장 원문 출제 허용 | 고정 원문은 대화를 부자연스럽게 만들어 화용성 측정을 훼손. §6 설계 배경 및 DB [§4.4](./P-ALT_DB_설계.md) 참조 |
| **AT용 P3·P4 학습연계 콘텐츠** | **사전 제작하지 않음.** PT/BOTH 범용 주제·카드만 제작해 PT·AT 공통 사용 (§6.1, §7.3) | 학습 내용은 사용자마다 다름 → 정적 주제 풀 매핑보다 **런타임에 개인 학습 이력 검토 후 동적 생성**이 타당 |

## 12. 미결 사항

| 항목 | 내용 | 결정 필요 시점 |
|------|------|--------------|
| P3 런타임 질문 생성 프롬프트 | probe_intent → 실제 질문 생성 프롬프트 설계·품질 검증 | CAT 엔진 구현 시 |
| **AT 학습연계 P3·P4 동적 생성** | (진행 시) 사용자가 **실제 학습한 콘텐츠를 검토해 P3 probe·P4 카드를 런타임 생성**. 전제: speaking 학습 모듈 이력 데이터 접근 | 향후 개인화 고도화 시 |
| 영어 전문가 검토 | Gemini 초안의 표현·난이도 최종 검토 주체 | 콘텐츠 QA 전 |
| 문항 노출 이력 관리 | 같은 사용자에게 동일 문항 재출제 방지 테이블 설계 | AT 구현 전 |
