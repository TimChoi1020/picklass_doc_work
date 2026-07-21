# 패널 레이아웃 설정 일반화 개발완료
## questionFlowMode 재정의 + uiTemplate 문항 패널 모드 도입 + passageMode 패널 제어 수정

> **작성일**: 2026-04-28  
> **완료일**: 2026-04-28  
> **연관 문서**: [SKM timed-select + 오케스트레이터 버그수정](./20260426_SKM_timed-select_오케스트레이터_버그수정.md)  
> **변경 파일**: `ai-module-data.ts`, `LessonSession.tsx`, `panels.tsx`, `MobileSplitLayout.tsx`, `seed.ts`, `register/page.tsx`, `agent-types.ts`

---

## 1. 배경 및 문제 요약

패널 레이아웃 제어가 DB 설정이 아닌 코드 내 파생 조건에 의존하고 있어 설정과 동작 간 불일치가 발생했다.

| 문제 | 원인 | 해결 |
|---|---|---|
| `passageMode: 'hidden'` 무효 | ContentPanel wrapper div 항상 렌더 → flex-1 공간 점유 | `passageMode !== 'hidden'` 조건부 렌더 적용 |
| 문항 패널 컴포넌트가 코드 파생 조건에 의존 | `isVoiceModule` (answerType 기반), `sentenceExplainMode` (answerType 기반) 하드코드 | `resolveUiTemplate()` 헬퍼로 교체 |
| `uiTemplate` DB 컬럼 미활용 | 전 모듈 `'standard'` 단일값이라 코드에서 무시됨 | 문항 패널 동작 모드로 재정의, 하위 호환 로직 구현 |
| `VoiceQuestionPanel` deck 미지원 | `questionFlowMode: 'deck'` 설정 무시, 전체 문장 누적 표시 | `questionFlowMode` prop 추가, deck 모드 구현 |

---

## 2. 설계 — 세 설정의 역할 분리

```
passageMode      → 지문 패널 동작 모드  (full / hidden / preview / timed-blur / timed-select)
uiTemplate       → 문항 패널 동작 모드  (standard / voice / embedded / hidden)
questionFlowMode → 문항 진행 방식       (sequential / deck / locked-steps)
```

### passageMode

| 값 | 지문 패널 DOM | 동작 |
|---|---|---|
| `full` | 렌더 | 처음부터 전체 공개 |
| `preview` | 렌더 | 첫 단락만 → 답변 후 전체 공개 |
| `timed-blur` | 렌더 | 타이머 읽기 → 읽기완료 후 blur 복원 |
| `timed-select` | 렌더 | 타이머 읽기 → 문장 선택 활성화 |
| `hidden` | **렌더 안 함** | 지문 패널 wrapper div 조건부 제거 |

### uiTemplate (재정의)

| 값 | 문항 패널 | 렌더 컴포넌트 |
|---|---|---|
| `standard` | 렌더 | `QuestionsPanel` |
| `voice` | 렌더 | `VoiceQuestionPanel` |
| `embedded` | **없음** | 지문에 문항 임베드 (CLR) |
| `hidden` | **없음** | 향후 확장용 |

### questionFlowMode (재정의)

| 값 | 적용 대상 | 동작 |
|---|---|---|
| `sequential` | **단문항 전용** | 문항 1개 표시 (기본값) |
| `deck` | **다문항 전용** | activeQuestionId 기준 카드 1개씩 순환 |
| `locked-steps` | **단문항 전용** | 전체 표시, 이전 완료 후 다음 활성화 |

```
questionCount: 'single'         →  questionFlowMode: 'sequential' | 'locked-steps'
questionCount: 'multi'          →  questionFlowMode: 'deck'
questionCount: 'content-driven' →  questionFlowMode 미사용 (CLR: uiTemplate=embedded)
```

---

## 3. 구현 내용

### Phase 1 — passageMode 'hidden' DOM 수정

**파일**: `apps/web/src/app/app/modules/[lessonId]/_components/LessonSession.tsx`

```tsx
// Before: ContentPanel wrapper 항상 렌더
<div className="flex-1 overflow-hidden">
  <ContentPanel previewOnly={!passageVisible} />
</div>

// After: passageMode 기준 조건부 렌더
{passageMode !== 'hidden' && (
  <div className="flex-1 overflow-hidden">
    <ContentPanel previewOnly={!passageVisible} />
  </div>
)}
```

> Mobile(`MobileSplitLayout.tsx`)은 `passageVisible=false` 시 탭 자체가 제거되므로 별도 수정 불필요.

---

### Phase 2 — resolveUiTemplate() + 문항 패널 라우팅

**파일**: `apps/web/src/lib/types/ai-module-data.ts`

```typescript
// Before
export type UiTemplate = 'standard';

// After
export type UiTemplate = 'standard' | 'voice' | 'embedded' | 'hidden';
```

**파일**: `LessonSession.tsx` (모듈 스코프에 추가)

```typescript
function resolveUiTemplate(
  uiTemplate: string,
  answerType: string,
): 'standard' | 'voice' | 'embedded' | 'hidden' {
  if (uiTemplate !== 'standard') return uiTemplate as 'voice' | 'embedded' | 'hidden';
  if (answerType === 'audio-record') return 'voice';
  if (answerType === 'sentence-explain') return 'embedded';
  return 'standard';
}
```

**`ModuleRunnerInner` 내 변경**:

```typescript
// Before
const isVoiceModule = moduleData.questions.every(q => q.type === 'audio-record');
// (line 587) const sentenceExplainMode = pedagogyProfile.answerType === 'sentence-explain';

// After
const effectiveUiTemplate = resolveUiTemplate(
  moduleData.uiTemplate ?? 'standard',
  pedagogyProfile.answerType,
);
const isVoiceModule = effectiveUiTemplate === 'voice';
// (line 587) const sentenceExplainMode = effectiveUiTemplate === 'embedded';
```

---

### Phase 3 — VoiceQuestionPanel deck 모드

**파일**: `apps/web/src/app/modules/[lessonId]/_components/panels.tsx`

- `questionFlowMode?: 'sequential' | 'deck' | 'locked-steps'` prop 추가
- `deck` 모드: `activeRecordingId` 기준 문장 1개만 카드형으로 표시, 헤더 `문장 N / M` 인디케이터
- `sequential` 모드(기본): 기존 전체 누적 표시 동작 유지

**파일**: `LessonSession.tsx`, `MobileSplitLayout.tsx`

```tsx
<VoiceQuestionPanel
  ...
  questionFlowMode={questionFlowMode}   // 추가
/>
```

---

### Phase 4 — 백오피스 수업모듈관리

#### seed.ts 변경 (`prisma/seed.ts`)

**UI_TEMPLATE code_items** (이름 수정 + 신규 3개 추가):
```typescript
UI_TEMPLATE: [
  { code: 'standard',  name: '기본 문항 패널',          sortOrder: 1 },
  { code: 'voice',     name: '음성 문항 패널',           sortOrder: 2 },
  { code: 'embedded',  name: '지문 임베드 (패널 없음)', sortOrder: 3 },
  { code: 'hidden',    name: '문항 패널 없음',           sortOrder: 4 },
],
```

**QUESTION_FLOW_MODE code_items** (이름 단문항/다문항 명시):
```typescript
QUESTION_FLOW_MODE: [
  { code: 'sequential',    name: '단문항 표시',          sortOrder: 1 },
  { code: 'deck',          name: '다문항 카드형 순환',    sortOrder: 2 },
  { code: 'locked-steps',  name: '단문항 단계 잠금형',   sortOrder: 3 },
],
```

**ai_modules 목표값 명시** (seed.ts만 업데이트, re-seed 미실행):

| 모듈 | 추가된 필드 |
|---|---|
| WRD, GMN, QAR, SWR | `questionFlowMode: 'deck'` |
| WSD, RRD, SHD, SHR | `uiTemplate: 'voice'` |
| CLR | `uiTemplate: 'embedded'` |
| RRD | `uiTemplate: 'voice'` + 기존 `questionFlowMode: 'deck'` 유지 |

#### register/page.tsx 변경

| 필드 | 변경 내용 |
|---|---|
| `uiTemplate` 레이블 | `화면 레이아웃 템플릿` → `문항 패널 동작 모드` |
| `uiTemplate` 설명 | standard/voice/embedded 역할 명시 |
| `questionFlowMode` 설명 | `단문항 전용 / 다문항 전용` 명시 |
| `passageMode` 설명 | `hidden=지문 패널 DOM 제거` 설명 추가 |

---

### 기타 — agent-types.ts 주석 정리

삭제된 모듈(WDR, WDS) 참조 주석 4건 현행화:

| 위치 | Before | After |
|---|---|---|
| `inputType` 테이블 english | `WDR·SWR` | `SWR` |
| `inputType` 테이블 audio | `WDS·RRD·SHR` | `RRD·SHD·WSD·WSP` |
| `vocabulary-context` | `WDR·WDS` | `WRD` |
| 재도전 모듈 목록 | `WDR, WDS, RRD, SHR` | `WRD, RRD, SHD, WSD, WSP` |

---

## 4. DB 처리 내역

### code_items 직접 수정 (Supabase SQL)

**UI_TEMPLATE** — `standard` 이름 업데이트 + `voice`/`embedded`/`hidden` 신규 추가  
**QUESTION_FLOW_MODE** — `sequential`/`deck`/`locked-steps` 이름 업데이트  
**레거시 항목 제거** — `chat`(id 629), `quiz`(id 630), `card`(id 631) 삭제
> 해당 값을 사용하는 ai_modules 없음 확인 후 삭제

### ai_modules — 수정 없음
> DB `ui_template` 전체 `'standard'` 유지. `resolveUiTemplate()` 하위 호환 로직으로 처리.

---

## 5. 활성 모듈 최종 상태

| 모듈 | question_count | passage_mode | ui_template (DB→실효) | question_flow_mode | 상태 |
|---|---|---|---|---|---|
| WRD | multi | full | standard→standard | deck | ✅ |
| WSD | multi | full | standard→**voice** | deck | ✅ |
| GMN | multi | full | standard→standard | deck | ✅ |
| WWB | multi | full | standard→standard | sequential | ⚠️ 부채 참고 |
| PRD | single | preview | standard→standard | sequential | ✅ |
| SCN | single | timed-blur | standard→standard | sequential | ✅ |
| SKM | single | timed-select | standard→standard | sequential | ✅ |
| CLR | content-driven | full | standard→**embedded** | — | ✅ |
| SUM | single | full | standard→standard | sequential | ✅ |
| QAR | multi | full | standard→standard | deck | ✅ |
| RRD | multi | **hidden** | standard→**voice** | deck | ✅ |
| SWR | multi | **hidden** | standard→standard | deck | ✅ |
| SHD | multi | full | standard→**voice** | sequential | ⚠️ 부채 참고 |
| WSP | multi | full | standard→**voice** | sequential | ⚠️ 부채 참고 |
| PWR | multi | full | standard→standard | sequential | ⚠️ 부채 참고 |

> `ui_template (DB→실효)`: DB는 모두 `standard`, `resolveUiTemplate()`이 `answerType` 기반으로 실효값 결정

---

## 6. 코드 규칙 (신규 모듈 추가 시)

### passageMode 선택 기준
```
지문 없음            → passageMode: 'hidden'
지문 항상 표시       → passageMode: 'full'
지문 일부 → 전체     → passageMode: 'preview'
타이머 읽기 (blur)   → passageMode: 'timed-blur'
타이머 읽기 (select) → passageMode: 'timed-select'
```

### uiTemplate 선택 기준
```
텍스트 입력 문항  → uiTemplate: 'standard'   (기본값, 생략 가능)
음성 입력 문항   → uiTemplate: 'voice'       (반드시 명시)
지문 임베드 방식  → uiTemplate: 'embedded'   (반드시 명시)
```

### questionFlowMode 선택 기준
```
questionCount: 'single'         → questionFlowMode 생략 (sequential 기본값)
questionCount: 'multi'          → questionFlowMode: 'deck' 반드시 명시
questionCount: 'content-driven' → questionFlowMode 미사용
```

---

## 7. 기술 부채

| 항목 | 내용 | 해소 조건 |
|---|---|---|
| `resolveUiTemplate` 하위 호환 로직 | DB `ui_template`이 전체 `'standard'`인 동안 `answerType`으로 실효값 결정 | DB `ui_template` 값이 실제 업데이트되면 제거 가능 |
| `uiTemplate` 컬럼명 의미 불일치 | DB 컬럼명은 `ui_template`이나 실질적으론 "문항 패널 동작 모드" | DB 컬럼 rename은 마이그레이션 비용 큼 → 문서로 관리 |
| WWB `questionFlowMode: sequential` | multi 모듈인데 sequential 유지 중 (DB값 보존 정책) | 별도 의사결정 후 deck 전환 |
| PWR `questionFlowMode: sequential` | multi 모듈인데 sequential 유지 중 (단계별 작문 특성) | 별도 의사결정 필요 |
| SHD, WSP `questionFlowMode: sequential` | voice 패널 + multi인데 sequential. deck 전환 시 VoiceQuestionPanel deck 동작 | 모듈 운영 방침 확정 후 전환 |
| 삭제 모듈 DB 잔류 | WDR, WDS, SHR(삭제예정), IMG, SNR — DB `status: active` 잔류. seed.ts 미등록, 소스 참조 없음 | 백오피스 UI에서 `status: inactive` 처리 권장 |
