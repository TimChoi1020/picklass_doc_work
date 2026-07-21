# GMN contentGenerationInstruction 추가

> **작성일**: 2026-04-26  
> **대상 파일**: `prisma/seed.ts` (GMN 항목)  
> **연관 문서**: [AI모듈필드재분류_완료](./20260426_AI모듈필드재분류_완료.md)  
> **영향 범위**: 백오피스 seed / Supabase ai_modules (튜터링 연동)

---

## 1. 사용자 흐름 (User Flow)

변경 없음. seed.ts 데이터 정의 변경으로, 학습자/관리자 화면에 직접 영향 없음.

튜터링 내부 흐름:

```
GMN 모듈 진입
  → getOrGenerate() → DB 캐시 미스
  → buildQuestionGenerationPrompt({ contentGenerationInstruction, ... })
       ↓ (기존: contentGenerationInstruction 없음)
       → buildInstructRules() fallback: "핵심 주제를 반영한 과제 지시문 1건"
       ↓ (변경 후: contentGenerationInstruction 설정됨)
       → 단어별 문맥 유추 문항 3~8건 생성
  → module_questions 저장
  → 각 문항: type='context-clue', text='…단어 X의 의미를 유추하세요', hint='영어 정의'
```

---

## 2. IA 구조 및 기능 정의

### 변경된 파일

| 파일 | 변경 내용 |
|---|---|
| `prisma/seed.ts` (GMN 항목, ~L619) | `contentGenerationInstruction` 필드 추가 |

### contentGenerationInstruction 역할

`buildQuestionGenerationPrompt()`에서 사용:

```typescript
const contentBlock = params.contentGenerationInstruction?.trim()
  ? params.contentGenerationInstruction.trim()      // ← 이 경로로 진입
  : params.strategy === 'extract'
    ? buildExtractRules(params.answerType)
    : buildInstructRules();                          // ← 기존 fallback (범용 과제 지시문)
```

---

## 3. 정책 (Policy / Business Rules)

### GMN 문항 생성 정책

| 항목 | 내용 |
|---|---|
| 문항 단위 | 단어 1개 = 문항 1개 (절대로 여러 단어를 하나의 문항에 묶지 않음) |
| 문항 수 | 3~8개 (questionMinCount~questionMaxCount) |
| 단어 선정 | 문맥 단서(정의/예시/대조/동격)로 유추 가능한 단어, 고유명사·수사·조동사 제외 |
| question type | `"context-clue"` |
| text 형식 | `"다음 지문에서 밑줄 친 단어 '[단어]'의 의미를 문맥을 통해 유추하여 한국어로 설명하세요."` |
| answer | `""` (빈 문자열, holistic 채점) |
| hint | 해당 단어의 영어 정의 1문장 (영영사전 스타일, 한국어 번역 포함 금지) |

### hintTypes 연계

GMN `hintTypes: ['english-definition']` 설정과 일치:  
→ contentGenerationInstruction의 hint 필드도 영어 정의 형태로 생성.

---

## 4. 개발자 추가 작업

- [ ] **Supabase ai_modules 동기화**: GMN 레코드의 `content_generation_instruction` 컬럼 업데이트  
  - 값: seed.ts GMN 항목의 `contentGenerationInstruction` 문자열 그대로
  - 방법: Supabase 대시보드 또는 REST API PATCH `?code=eq.GMN`
- [ ] **module_questions 캐시 무효화**: 기존 생성된 GMN module_questions 레코드가 있다면 삭제  
  → 다음 진입 시 새 instruction으로 재생성
- [ ] **검증**: GMN 모듈 진입 후 module_questions 테이블에 3~8개 문항이 `context-clue` 타입으로 생성되는지 확인

---

## 5. 코드 규칙 (Coding Rules)

### contentGenerationInstruction 작성 규칙

- **역할 선언**: `[역할]` 블록으로 시작 — Gemini에게 맥락 제공
- **모듈 특성**: `[모듈 특성]` 블록 — 해당 모듈의 학습 방식 설명
- **생성 방식**: 각 레코드 필드를 명시적으로 열거 (question_number, type, text, options, answer, hint)
- **answer 처리**: holistic 모듈은 반드시 `answer: ""` 고정 명시
- **hint 처리**: hintTypes에 맞는 형식 명시 (또는 null 명시)
- **레벨별 분기**: 필요 시 targetLevel 범위에 따른 어휘 수준 지침 포함 (CLR 참고)

### 금지 패턴

- 여러 단어를 묶은 종합 과제 지시문 → instruct 전략의 `buildInstructRules()` fallback이 이 방식 (GMN에 부적합)
- hint에 한국어 번역 포함 → `hintTypes: ['english-definition']` 취지 위배
- answer에 실제 정답 기입 → holistic 모듈은 `""` 고정

---

## 6. 기술 부채 / 알려진 문제 (Known Issues)

| 항목 | 내용 |
|---|---|
| 사전 생성 힌트 미활용 | `module_questions.hint`에 영어 정의가 저장되지만, `giveHint` case가 항상 `/ai/hint` API를 재호출함. Phase 2(힌트 시스템 개선) 이전까지는 낭비 발생 |
| Supabase 수동 동기화 필요 | seed.ts ↔ Supabase ai_modules는 자동 동기화 없음 — 변경 시마다 수동 PATCH 필요 |
| `contentGenerationInstruction` 미설정 모듈 | WWB 등 instruct 전략 다문항 holistic 모듈 중 미설정 항목 존재 — `buildInstructRules()` fallback 사용 중 |

---

## 7. 컴포넌트/훅 의존성 (Dependencies)

```
백오피스 seed.ts (GMN.contentGenerationInstruction)
  → Supabase ai_modules.content_generation_instruction
    → 튜터링 GET /lessons/module-meta → AiModuleData.contentGenerationInstruction
      → buildQuestionGenerationPrompt(params.contentGenerationInstruction)
        → Gemini → module_questions (3~8건)
          → 튜터링 프론트엔드 input.questions
```

---

## 8. DB/API 구조 (Data Contract)

### 백오피스 ai_modules 컬럼

```
content_generation_instruction: TEXT | NULL
```

### 튜터링 API 계약

```typescript
// GET /lessons/module-meta?moduleCode=GMN
// AiModuleData
{
  contentGenerationInstruction: string | null;  // seed의 값 그대로
}
```

### 생성되는 module_questions 레코드 형태 (GMN)

```typescript
{
  question_number: number;         // 1~8
  type: "context-clue";
  text: "다음 지문에서 밑줄 친 단어 '[단어]'의 의미를 문맥을 통해 유추하여 한국어로 설명하세요.";
  options: null;
  answer: "";                      // holistic — 빈 문자열
  hint: string;                    // 영어 정의 1문장 (예: "to feel or show great happiness")
}
```
