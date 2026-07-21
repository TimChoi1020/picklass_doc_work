---
title: WSD 힌트 데이터 미로드 (미해결)
status: unresolved
date: 2026-06-09
---

## 증상 (Symptom)

WRD(Word Reading Decker)는 `hints.button` / `hints.direct` 데이터를 정상적으로 불러오는 반면,
WSD(Word Speaking Decker)는 동일한 데이터가 DB에 존재함에도 불구하고 로드되지 않는다.
결과적으로 힌트 버튼이 표시되지 않고 LLM 폴백으로 진행된다.

## 시도한 조치

1. **`question-generation.ts` 프롬프트 수정** (`apps/api/src/ai/prompts/question-generation.ts:145`)
   - `audio-record` 규칙의 `hints: null` → `hints: { button: "...", direct: "..." }` 로 변경
   - 신규 생성 문항에는 hints가 포함됨

2. **WSD 캐시 무효화**
   - `module_id = '5092e069-90f2-458b-8274-67677826d147'`인 active 레코드 55개를 archived 처리
   - SQL: `UPDATE module_questions SET status = 'archived' WHERE module_id = '5092e069-90f2-458b-8274-67677826d147' AND status = 'active'`
   - 이후 재생성 시 새 프롬프트 적용됨

3. 위 조치 후에도 문제가 해결되지 않음

## 미확인 원인 후보

- WSD 레슨의 `lesson.text_id` 가 null이어서 `module_questions` 대신 `ai_modules.questionData` JSONB 폴백이 사용되는지 여부
- `ai_modules.questionData` JSONB 필드에 hints 구조가 없을 가능성
- `lessons.service.ts:340` 분기: `if (lesson.text_id && lesson.text)` — text_id 없으면 module_questions를 조회하지 않음

## 다음 확인 사항

```sql
-- WSD 레슨에 text_id가 있는지 확인
SELECT cl.id, cl.text_id, cl.topic
FROM course_lessons cl
JOIN (
  SELECT DISTINCT course_lesson_id FROM module_histories WHERE module_code = 'WSD'
) mh ON cl.id = mh.course_lesson_id
LIMIT 10;
```

- `text_id`가 null이면 `ai_modules.questionData` 폴백 경로로 진입 → hints 없음
- `ai_modules.questionData` JSONB의 각 문항에 hints 필드 추가 필요 여부 검토
- `lessons.service.ts` `getModuleData` 의 폴백 경로(`text_id` 없을 때)에서도 hints를 포함하는 구조인지 확인

## 관련 파일

- `apps/api/src/ai/prompts/question-generation.ts:139-146` — audio-record 프롬프트 규칙
- `apps/api/src/lessons/lessons.service.ts:340-373` — text_id 유무 분기
- `apps/api/src/ai/question-generator.service.ts:51-70` — 캐시 조회 로직
- `apps/web/src/lib/adapters/GenericAdapter.ts:56,78` — hints 매핑
- `apps/web/src/lib/hooks/useModuleOrchestrator.ts:481-520` — 힌트 에스컬레이션 코드
