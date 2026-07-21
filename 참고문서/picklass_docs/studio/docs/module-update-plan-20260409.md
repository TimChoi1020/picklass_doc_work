# 수업모듈 업데이트 계획 — DB 업데이트 + 하드코딩 제거

**작성일**: 2026-04-09  
**버전**: 1.0

---

## 1. 작업 개요

| 순서 | 작업 | 설명 |
|------|------|------|
| STEP 1 | DB 업데이트 | 테스트 데이터 삭제 + CSV 14개 모듈 속성 업데이트 |
| STEP 2 | 코드 수정 | 하드코딩 MODULE_CATEGORIES 삭제, ai_modules 테이블 단독 참조 |

---

## STEP 1: ai_modules 테이블 업데이트

### 1-A. 테스트 데이터 삭제 (2건)

```sql
DELETE FROM ai_modules WHERE code IN ('111', '11');
```

### 1-B. CSV 14개 모듈 속성 업데이트

skill명: CSV `voca` = DB `vocabulary` (변경 없음)  
CSV에 없는 모듈 9개(ORL, WSP, LR, SXP, SHD, SNR, WRD, IMG, WSD): **그대로 유지**

CSV → DB 컬럼 매핑:

| CSV 행 | DB 컬럼 | 비고 |
|--------|---------|------|
| scoringMode | scoring_mode | |
| answerType | answer_type | |
| passageExposureMode | passage_exposure_mode | |
| questionCount | question_count | |
| feedbackStyle | feedback_style | |
| questionMaxAttempts | question_max_attempts | 'undefined' → NULL |
| hintTypes | hint_types | |
| retryScope | retry_scope | |
| inputLanguage | input_language | |
| passageRole | passage_role | |
| Purpose | purpose | 비어있으면 기존값 유지 |
| Pedagogy Instruction | pedagogy_instruction | 비어있으면 기존값 유지 |

### 1-C. 모듈별 업데이트 값

#### vocabulary (4개)

| 컬럼 | WDR | WDS | GMN | WWB |
|------|-----|-----|-----|-----|
| scoring_mode | exact | pronunciation | holistic | holistic |
| answer_type | short-text | audio-record | essay | short-text |
| passage_exposure_mode | full | full | full | hidden |
| question_count | multi | multi | multi | multi |
| feedback_style | correct-wrong | correct-wrong | strengths-weaknesses | strengths-weaknesses |
| question_max_attempts | 3 | 3 | 1 | 1 |
| hint_types | spelling-prefix | audio-replaysyllable-guide | english-definition | relation-meaning |
| retry_scope | spelling-accuracy | pronunciation-accuracy | inference-accuracy | semantic-coverage |
| input_language | english | audio | korean | english |
| passage_role | context-reference | context-reference | context-reference | web-root |
| purpose | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 |
| pedagogy_instruction | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 |

#### reading (7개)

| 컬럼 | PRD | SCN | SKM | CLR | SUM | QAR | RRD |
|------|-----|-----|-----|-----|-----|-----|-----|
| scoring_mode | holistic | holistic | exact | holistic | holistic | exact | pronunciation |
| answer_type | essay | short-text | multiple-choice | essay | essay | mixed | audio-record |
| passage_exposure_mode | preview | hidden | full | full | full | full | full |
| question_count | single | single | single | multi | single | multi | multi |
| feedback_style | strengths-weaknesses | strengths-weaknesses | correct-wrong | strengths-weaknesses | strengths-weaknesses | correct-wrong | strengths-weaknesses |
| question_max_attempts | 1 | 1 | 1 | NULL | 1 | 1 | 3 |
| hint_types | bridge-question | korean-summary | none | none | topic-sentence | korean-translation | audio-replayfluency-points |
| retry_scope | inference-accuracy | keyword-hit-rate | topic-sentence-match | engagement-depth | structural-score | evidence-accuracy | pronunciation-accuracy |
| input_language | korean-or-english | korean-or-english | none (click) | none (click) | korean-or-english | none (click) | audio |
| passage_role | prediction-trigger | scan-target | scan-target | reading-material | reading-material | reading-material | fluency-practice-target |
| purpose | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | 기존값 유지 |
| pedagogy_instruction | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | CSV 텍스트 | 기존값 유지 |

#### speaking (1개)

| 컬럼 | SHR |
|------|-----|
| scoring_mode | pronunciation |
| answer_type | audio-record |
| passage_exposure_mode | full |
| question_count | multi |
| feedback_style | strengths-weaknesses |
| question_max_attempts | 3 |
| hint_types | audio-replayfluency-points |
| retry_scope | pronunciation-accuracy |
| input_language | audio |
| passage_role | fluency-practice-target |
| purpose | 기존값 유지 |
| pedagogy_instruction | 기존값 유지 |

#### writing (2개)

| 컬럼 | SWR | PWR |
|------|-----|-----|
| scoring_mode | holistic | holistic |
| answer_type | sentence-write | essay |
| passage_exposure_mode | full | full |
| question_count | multi | multi |
| feedback_style | strengths-weaknesses | strengths-weaknesses |
| question_max_attempts | 1 | 1 |
| hint_types | none | none |
| retry_scope | writing-quality | logical-coherence |
| input_language | english | korean-or-english |
| passage_role | writing-reference | writing-reference |
| purpose | 기존값 유지 | 기존값 유지 |
| pedagogy_instruction | 기존값 유지 | 기존값 유지 |

---

## STEP 2: 하드코딩 제거

### 2-A. shared 상수 삭제

**파일**: `packages/shared/src/constants/index.ts`

삭제 대상:
- `MODULE_CATEGORIES` 상수
- `ALL_MODULES` 상수
- `getModuleByCode()` 함수
- `getCategoryByModuleCode()` 함수

### 2-B. use-modules.ts 정리

**파일**: `apps/web/src/hooks/use-modules.ts`

- `MODULE_CATEGORIES` import 제거
- fallback 로직 삭제 — API 데이터 없으면 빈 객체/빈 배열 반환
- `SKILL_KOR_NAME` 수정: vocabulary, reading, speaking, writing

### 2-C. 프론트엔드 emoji 코드 제거

**파일**:
- `apps/web/src/app/course/page.tsx` — emoji 조건부 렌더링 삭제
- `apps/web/src/app/course/[courseId]/page.tsx` — emoji 조건부 렌더링 삭제

---

## 수정 대상 요약

| 대상 | 작업 |
|------|------|
| ai_modules 테이블 | 테스트 2건 삭제, CSV 14개 모듈 속성 업데이트 |
| `packages/shared/src/constants/index.ts` | MODULE_CATEGORIES 관련 코드 삭제 |
| `apps/web/src/hooks/use-modules.ts` | fallback 제거, DB 전용으로 단순화 |
| `apps/web/src/app/course/page.tsx` | emoji 렌더링 제거 |
| `apps/web/src/app/course/[courseId]/page.tsx` | emoji 렌더링 제거 |
