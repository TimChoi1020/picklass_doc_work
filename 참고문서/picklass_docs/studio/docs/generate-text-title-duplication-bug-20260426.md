# 지문 생성 시 `texts.title`이 `texts.content`에 중복 저장되는 버그 — 분석 + 수정

**작성일**: 2026-04-26
**대상 파일**: [apps/web/src/app/api/generate-text/route.ts](../apps/web/src/app/api/generate-text/route.ts)
**증상**: AI 지문 생성 후 `texts` 테이블 저장 시, `title` 컬럼에 들어간 제목이 `content` 컬럼의 본문 앞부분에도 그대로 포함되어 저장됨.

---

## 1. 데이터 흐름

```
[클라이언트]
  text-creation-modal.tsx / CreatePassageModal.tsx /
  class/page.tsx / use-courses.ts
    ↓ POST /api/generate-text
[서버 - route.ts]
  Gemini 2.5 Flash 호출 (스트리밍)
    ↓ fullResponse 누적
  fullResponse를 split('\n') 후 라인 단위 파싱
    - line.startsWith('Title:') → title
    - line.includes('Content:') 이후 → content
    ↓
  removeMarkdown(title), removeMarkdown(content)
    ↓ JSON 응답 { title, content, wordCount, actualWordCount }
[클라이언트]
  useCreateText 훅 → POST /passages
    ↓
  passages.service.create → prisma.texts.create({ title, content })
```

호출처는 4곳이며 모두 응답에서 `title`/`content`만 사용한다.

- `apps/web/src/app/class/page.tsx:337`
- `apps/web/src/components/CreatePassageModal.tsx:85`
- `apps/web/src/components/ui/text-creation-modal.tsx:100`
- `apps/web/src/hooks/use-courses.ts:168`

---

## 2. 버그 원인

원인은 **프롬프트의 의도가 아니라 서버측 파싱 로직의 결함**이다. 프롬프트는 분명히 분리하라고 지시한다.

```
Response Format:
Title: [Appropriate title]

Content:
[Text content - plain text only, no markdown]

Please separate the title and content clearly...
```

### 2-1. 파싱 로직이 너무 엄격 (수정 전 `route.ts:218-236`)

```typescript
for (const line of lines) {
  if (line.startsWith('Title:')) {            // ← 앞 공백 1칸만 있어도 실패
    title = line.replace('Title:', '').trim();
  } else if (line.includes('Content:') || isContent) {
    if (line.includes('Content:')) {
      isContent = true;
      continue;
    }
    if (isContent && line.trim()) {
      content += line + '\n';
    }
  }
}
```

Gemini가 다음과 같이 마크다운/형식을 살짝 어기면 매칭이 모두 실패한다.

- `**Title:** My Story` (굵은 글씨)
- `## Title: My Story` (헤더)
- `*Title:* My Story` (기울임)
- `  Title: My Story` (앞 공백)
- `Content: Hello world` (마커와 본문이 같은 줄)

### 2-2. 결정적 버그 — fallback이 전체 응답을 그대로 사용 (수정 전 `route.ts:243-246`)

```typescript
// 본문이 없으면 전체 응답을 본문으로 사용
if (!content.trim()) {
  content = fullResponse;   // ← 제목 라인 "Title: ..." 까지 통째로 들어감
}
```

content 파싱이 실패하면 **fullResponse 전체**를 content로 덮어쓴다. fullResponse에는 `Title: My Story\n\nContent:\n본문...` 형태가 그대로 남아 있어 **제목이 content 안에 그대로 중복 저장된다.**

게다가 title 쪽에서는 `if (!title)` fallback이 `${genre} about ${topic}`를 채우므로 (`route.ts:239-241`), 결과적으로:

- `texts.title` = `"essay about climate change"` (기본값)
- `texts.content` = `"Title: My Story\n\nContent:\n본문..."` (전체 응답)

이라는 어색한 데이터가 저장된다.

### 2-3. 부수 문제 — `removeMarkdown` 호출 순서

```typescript
const cleanedContent = removeMarkdown(content.trim());   // 파싱 *이후* 실행
```

마크다운 제거가 파싱 이후에 일어나므로, 마크다운 섞인 응답은 이미 파싱 단계에서 실패한 뒤다. 마커 주변의 마크다운만 파싱 *전에* 정리했다면 매칭이 성공했을 것이다.

---

## 3. 수정 내용

### 3-1. 파싱 전 마커 주변 마크다운만 정리 (`route.ts:218-222`)

본문의 마크다운 처리(기존 `removeMarkdown`)는 그대로 유지하고, **`Title:` / `Content:` 마커를 감싼 마크다운만** 먼저 벗겨낸다.

```typescript
const normalizedResponse = fullResponse
  .replace(/\*\*\s*(Title|Content)\s*:\s*\*\*/gi, '$1:')   // **Title:** → Title:
  .replace(/^#{1,6}\s+(Title|Content)\s*:/gim, '$1:')      // ## Title:   → Title:
  .replace(/^\s*\*\s*(Title|Content)\s*:\s*\*/gim, '$1:'); // *Title:*    → Title:
```

### 3-2. 파싱 정규식 관대화 (`route.ts:230-243`)

```typescript
for (const line of lines) {
  const titleMatch = !isContent && line.match(/^\s*Title\s*:\s*(.*)$/i);
  const contentMatch = line.match(/^\s*Content\s*:\s*(.*)$/i);

  if (titleMatch) {
    title = titleMatch[1].trim();
  } else if (contentMatch) {
    isContent = true;
    const rest = contentMatch[1].trim();
    if (rest) content += rest + '\n';   // 'Content: 본문' 인라인 패턴 처리
  } else if (isContent && line.trim()) {
    content += line + '\n';
  }
}
```

- 앞 공백 허용 (`^\s*`)
- 대소문자 무시 (`/i`)
- `Content: 본문` 처럼 마커와 본문이 같은 줄에 있어도 본문에 포함
- `!isContent` 가드로 본문 영역에 우연히 등장한 "Title:" 문자열은 본문으로 처리

### 3-3. fallback 안전화 (`route.ts:250-257`) — **핵심 수정**

```typescript
if (!content.trim()) {
  content = normalizedResponse
    .split('\n')
    .filter(line => !/^\s*Title\s*:/i.test(line) && !/^\s*Content\s*:\s*$/i.test(line))
    .join('\n')
    .trim();
}
```

본문 파싱이 끝까지 실패해도 **`Title:` 라인과 단독 `Content:` 라인을 제거**하고 사용하므로 제목 중복이 발생하지 않는다.

---

## 4. 호환성 / 영향 범위

| 항목 | 영향 |
|---|---|
| API 응답 스키마 (`{title, content, wordCount, actualWordCount}`) | **변경 없음** |
| 호출처 4곳 | **영향 없음** |
| 기존 정상 응답 케이스 | 새 정규식이 기존 패턴을 모두 포함 → 동일하게 동작 |
| 이미 DB에 저장된 중복 데이터 | 본 수정 대상 아님 (필요 시 별도 마이그레이션 필요) |
| 타입 체크 (`tsc --noEmit`) | 통과 |

---

## 5. 후속 과제 (선택)

1. **이미 저장된 중복 데이터 정리**
   `texts.content`가 `"Title: ..."` 또는 `"Content:"` 마커로 시작하는 행을 찾아 1회성 마이그레이션으로 정리.

2. **JSON responseSchema로 전환 (2단계)**
   `responseMimeType: 'application/json'` + `responseSchema`를 사용하면 텍스트 파싱 자체가 불필요해진다. 다만 스트리밍 처리와 응답 형식이 모두 바뀌므로 별도 작업 단위로 분리.

3. **`removeMarkdown`의 `_text_` 정규식**
   본문에 변수명(`user_id`, `my_var`)이 등장하면 underscore가 잘려 나갈 수 있다. 본 버그와 별개이지만 같은 함수라 함께 알아둘 필요가 있다.
