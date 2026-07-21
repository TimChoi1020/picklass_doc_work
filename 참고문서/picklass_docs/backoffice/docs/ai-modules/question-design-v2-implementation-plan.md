# 문항설계2 탭 추가 개발 계획

**작성일**: 2026-04-02  
**대상 버전**: picklass-backoffice  
**담당자**: AI 에이전트

---

## 1. 개요

수업모듈 코드관리 페이지와 모듈등록 페이지에 **문항설계2 탭**을 추가하여, 다음 5가지 항목을 관리하고자 함.

| 항목명 | 영문명 | 설명 |
|--------|--------|------|
| 힌트 타입 | hintTypes | 문항 힌트 제공 방식 관리 (**다중 선택**, 쉼표 구분 저장) |
| 재시도 범위 | retryScope | 재시도 시 채점 범위 지정 |
| 입력 언어 | inputLanguage | 학습자 입력 시 사용 가능한 언어 |
| 지문 역할 | passageRole | 지문이 수행하는 역할 정의 |
| 최대 제출 횟수 | questionMaxAttempts | 직접 숫자 입력 (무제한/1회/3회 등 자유 설정) |

---

## 2. 작업 범위 및 구조

### 2.1 영향받는 파일

#### Frontend (Next.js)
- `apps/admin/frontend/src/app/(admin)/admin/ai-modules/code-management/page.tsx` - 코드관리 페이지
- `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx` - 모듈등록 페이지

#### Backend (NestJS)
- `apps/admin/backend/src/ai-modules/ai-modules.controller.ts`
- `apps/admin/backend/src/ai-modules/ai-modules.service.ts`
- `packages/core/src/ai-modules/ai-modules.service.ts` - 비즈니스 로직
- `packages/types/src/index.ts` - DTO/인터페이스

#### Database (Prisma)
- `prisma/schema.prisma` - AiModule 모델
- `prisma/seed.ts` - 시드 데이터 (코드 그룹 + 코드 항목 추가)
- `prisma/migrations/` - 마이그레이션 파일

---

## 3. 상세 작업 계획

### 3.1 [Database 변경] Prisma 스키마 추가

**파일**: `prisma/schema.prisma`

AiModule 모델에 다음 4개 필드 추가:

```prisma
// ── 문항설계2 (탭3.5) ──
hintTypes          String    @default("") @map("hint_types") @db.VarChar(200)
retryScope         String    @default("") @map("retry_scope") @db.VarChar(200)
inputLanguage      String    @default("") @map("input_language") @db.VarChar(200)
passageRole        String    @default("") @map("passage_role") @db.VarChar(200)
questionMaxAttempts Int?      @map("question_max_attempts")
```

**스토리**: 각 필드는 쉼표로 구분된 문자열로 여러 값을 저장하거나, 단일 선택 코드를 저장

---

### 3.2 [Database 전환] Prisma 마이그레이션

**작업**:

> **주의**: Supabase 환경에서 `prisma migrate dev`는 cross-schema 참조(`auth.users`) 오류로 실행 불가.  
> 아래와 같이 **직접 SQL 실행** 후 `prisma generate`로 Client를 재생성한다.

```bash
# 1) 직접 SQL로 컬럼 추가 (포트 5432 = Direct 연결)
DATABASE_URL="postgresql://..." npx prisma db execute --stdin <<'SQL'
ALTER TABLE "ai_modules" ADD COLUMN IF NOT EXISTS "hint_types" VARCHAR(200) NOT NULL DEFAULT '';
ALTER TABLE "ai_modules" ADD COLUMN IF NOT EXISTS "retry_scope" VARCHAR(200) NOT NULL DEFAULT '';
ALTER TABLE "ai_modules" ADD COLUMN IF NOT EXISTS "input_language" VARCHAR(200) NOT NULL DEFAULT '';
ALTER TABLE "ai_modules" ADD COLUMN IF NOT EXISTS "passage_role" VARCHAR(200) NOT NULL DEFAULT '';
ALTER TABLE "ai_modules" ADD COLUMN IF NOT EXISTS "question_max_attempts" INTEGER;
SQL

# 2) Prisma Client 재생성
npx prisma generate
```

---

### 3.2.1 [Database 시드] 코드 그룹 및 코드 항목 추가

문항설계2 탭의 드롭다운(select)은 `code_groups` → `code_items` 테이블에서 동적으로 로드된다.  
**코드 그룹 4개**와 각 그룹의 **코드 항목**을 시드 데이터로 추가해야 한다.

#### 파일: `prisma/seed.ts`

**Step 1**: `codeGroups` 배열에 4개 그룹 추가 (기존 `SKILL` 항목 뒤)

```typescript
// 문항설계2 관련 코드 그룹
{
  code: 'HINT_TYPES',
  name: '힌트 타입',
  description: '문항 힌트 제공 방식 관리',
},
{
  code: 'RETRY_SCOPE',
  name: '재시도 범위',
  description: '재시도 시 채점 범위 지정',
},
{
  code: 'INPUT_LANGUAGE',
  name: '입력 언어',
  description: '학습자 입력 시 사용 가능한 언어',
},
{
  code: 'PASSAGE_ROLE',
  name: '지문 역할',
  description: '지문이 수행하는 역할 정의',
},
```

**Step 2**: `codeItemsByGroup` 객체에 항목 데이터 추가

> 모듈 스프레드시트에서 실제 사용되는 **모든 코드 값**을 포함한다.

```typescript
// ─── 문항설계2 코드 항목 ───

HINT_TYPES: [
  { code: 'bridge-question', name: '다리 질문', sortOrder: 1 },
  { code: 'audio-replay', name: '음성 재생', sortOrder: 2 },
  { code: 'fluency-points', name: '유창성 점수', sortOrder: 3 },
  { code: 'spelling-prefix', name: '스펠링 접두사 힌트', sortOrder: 4 },
  { code: 'audio-replaysyllable-guide', name: '음성 재생 + 음절 가이드', sortOrder: 5 },
  { code: 'english-definition', name: '영어 정의 제공', sortOrder: 6 },
  { code: 'relation-meaning', name: '관계/의미 힌트', sortOrder: 7 },
  { code: 'korean-summary', name: '한국어 요약', sortOrder: 8 },
  { code: 'topic-sentence', name: '주제문 힌트', sortOrder: 9 },
  { code: 'korean-translation', name: '한국어 번역', sortOrder: 10 },
  { code: 'audio-replayfluency-points', name: '음성 재생 + 유창성 점수', sortOrder: 11 },
],

RETRY_SCOPE: [
  { code: 'inference-accuracy', name: '추론 정확도', sortOrder: 1 },
  { code: 'pronunciation-accuracy', name: '발음 정확도', sortOrder: 2 },
  { code: 'spelling-accuracy', name: '스펠링 정확도', sortOrder: 3 },
  { code: 'semantic-coverage', name: '의미 커버리지', sortOrder: 4 },
  { code: 'keyword-hit-rate', name: '키워드 적중률', sortOrder: 5 },
  { code: 'topic-sentence-match', name: '주제문 매칭', sortOrder: 6 },
  { code: 'engagement-depth', name: '참여 깊이', sortOrder: 7 },
  { code: 'structural-score', name: '구조 점수', sortOrder: 8 },
  { code: 'evidence-accuracy', name: '근거 정확도', sortOrder: 9 },
  { code: 'writing-quality', name: '작문 품질', sortOrder: 10 },
  { code: 'logical-coherence', name: '논리적 일관성', sortOrder: 11 },
],

INPUT_LANGUAGE: [
  { code: 'korean-or-english', name: '한국어 또는 영어', sortOrder: 1 },
  { code: 'audio', name: '음성 입력', sortOrder: 2 },
  { code: 'english', name: '영어', sortOrder: 3 },
  { code: 'korean', name: '한국어', sortOrder: 4 },
],

PASSAGE_ROLE: [
  { code: 'prediction-trigger', name: '예측 트리거', sortOrder: 1 },
  { code: 'fluency-practice-target', name: '유창성 연습 대상', sortOrder: 2 },
  { code: 'context-reference', name: '문맥 참조', sortOrder: 3 },
  { code: 'web-root', name: '웹 루트', sortOrder: 4 },
  { code: 'scan-target', name: '스캔 대상', sortOrder: 5 },
  { code: 'reading-material', name: '읽기 자료', sortOrder: 6 },
  { code: 'writing-reference', name: '작문 참조', sortOrder: 7 },
],
```

**Step 3**: 시드 실행

```bash
# .env 파일에 DATABASE_URL 설정 필요 (Direct 연결, 포트 5432)
npx prisma db seed
```

> **참고**: 시드는 `upsert`로 동작하므로 기존 데이터에 영향 없이 안전하게 재실행 가능.  
> 코드 관리 페이지 UI에서 직접 추가/수정도 가능하지만, 초기 데이터는 시드로 넣는 것이 운영 일관성을 보장한다.

---

### 3.3 [Frontend - 타입] DTO 및 인터페이스 추가

**파일**: `packages/types/src/index.ts`

#### 기존 인터페이스 확장
```typescript
export interface AiModuleResponse {
  // ... 기존 필드
  
  // ── 문항설계2 (새로 추가) ──
  hintTypes?: string;
  retryScope?: string;
  inputLanguage?: string;
  passageRole?: string;
  questionMaxAttempts?: number;
}

export interface CreateAiModuleDto {
  // ... 기존 필드
  
  // ── 문항설계2 (새로 추가) ──
  hintTypes?: string;
  retryScope?: string;
  inputLanguage?: string;
  passageRole?: string;
  questionMaxAttempts?: number;
}
```

---

### 3.4 [코드관리] 탭 추가 및 코드 그룹 설정

**파일**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/code-management/page.tsx`

#### 작업 내용

**Step 1**: `buildTabConfig()` 함수에 새 탭 추가

위치: "문항설계" 탭과 "레벨/인지" 탭 사이

```typescript
{
  key: 'question-design-v2',
  label: '문항설계2',
  groups: [
    {
      groupCode: 'HINT_TYPES',
      title: 'Hint Types',
      columns: nameCodeColumns,
    },
    {
      groupCode: 'RETRY_SCOPE',
      title: 'Retry Scope',
      columns: nameCodeColumns,
    },
    {
      groupCode: 'INPUT_LANGUAGE',
      title: 'Input Language',
      columns: nameCodeColumns,
    },
    {
      groupCode: 'PASSAGE_ROLE',
      title: 'Passage Role',
      columns: nameCodeColumns,
    },
  ],
}
```

#### 관리할 코드 예시

| 그룹 코드 | 코드 | 설명 |
|----------|------|------|
| HINT_TYPES | bridge-question | 다리 질문 |
| HINT_TYPES | audio-replay, fluency-points | 음성 재생, 유창성 점수 |
| RETRY_SCOPE | inference-accuracy | 추론 정확도 |
| RETRY_SCOPE | pronunciation-accuracy | 발음 정확도 |
| INPUT_LANGUAGE | korean-or-english | 한국어 또는 영어 |
| INPUT_LANGUAGE | audio | 음성 입력 |
| PASSAGE_ROLE | prediction-trigger | 예측 트리거 |
| PASSAGE_ROLE | fluency-practice-target | 유창성 연습 대상 |

**주의**: `questionMaxAttempts`는 코드 관리 항목이 아니라 모듈 레지스트리에서 직접 입력하는 숫자 필드입니다. 코드 그룹 관리에 포함되지 않습니다.

---

### 3.5 [모듈등록] 폼 데이터 구조 확장

**파일**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

#### Step 1: ModuleFormData 인터페이스 수정

기존 `ModuleFormData` 인터페이스에 다음 필드 추가:

```typescript
interface ModuleFormData {
  // ... 기존 필드 (skill, moduleName, answerType 등)
  
  // ── 문항설계2 필드 (새로 추가) ──
  hintTypes: string;
  retryScope: string;
  inputLanguage: string;
  passageRole: string;
  questionMaxAttempts: number | undefined;
}
```

#### Step 2: useState 초기값 설정

```typescript
const [formData, setFormData] = useState<ModuleFormData>({
  // ... 기존 필드들
  
  // ── 문항설계2 필드 ──
  hintTypes: '',
  retryScope: '',
  inputLanguage: '',
  passageRole: '',
  questionMaxAttempts: undefined,
});
```

#### Step 3: 모듈 조회 시 데이터 로딩 (useEffect)

`loadModule()` 함수의 `setFormData` 호출에 다음 추가:

```typescript
setFormData((prev) => ({
  ...prev,
  hintTypes: data.hintTypes ?? '',
  retryScope: data.retryScope ?? '',
  inputLanguage: data.inputLanguage ?? '',
  passageRole: data.passageRole ?? '',
  questionMaxAttempts: data.questionMaxAttempts,
}));
```

---

### 3.6 [모듈등록] UI 탭 추가 및 폼 필드 구성

**파일**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

#### Step 1: 탭 버튼 추가

현재 탭 버튼 목록:
- 기본 정보
- 커리큘럼 배치 설정
- 문항설계(QuestionData)
- **문항설계2** ← **새롭게 추가**
- 성과KPI설정
- AI 동작 설정

#### Step 2: activeTab 상태 확장

```typescript
const [activeTab, setActiveTab] = useState<
  'basicInfo' 
  | 'courseSettings' 
  | 'questionDataDesign' 
  | 'questionDesignV2'  // ← 새로 추가
  | 'kpiSettings' 
  | 'contentConfig'
>('basicInfo');
```

#### Step 3: 탭 버튼 UI 추가

기존의 "문항설계(QuestionData)" 버튼 다음에 추가:

```typescript
<button
  type="button"
  onClick={() => setActiveTab('questionDesignV2')}
  style={{
    padding: '12px 20px',
    fontSize: '14px',
    fontWeight: 600,
    border: 'none',
    background: 'none',
    cursor: 'pointer',
    borderBottom: activeTab === 'questionDesignV2' ? '3px solid #667eea' : 'none',
    color: activeTab === 'questionDesignV2' ? '#667eea' : '#999',
  }}
>
  문항설계2
</button>
```

#### Step 4: 탭 콘텐츠 UI 구현

기존의 문항설계 탭 콘텐츠 다음에 추가:

```typescript
{/* 문항설계2 탭 */}
{activeTab === 'questionDesignV2' && (
  <div>
    <h3 style={{ marginBottom: '16px', fontSize: '18px', fontWeight: 700, color: '#333' }}>
      문항설계2
    </h3>

    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(2, 1fr)', gap: '20px', marginBottom: '20px' }}>
      {/* Hint Types — 다중 선택 (체크박스 태그) */}
      <div>
        <label style={{ display: 'block', fontWeight: 600, marginBottom: '8px' }}>Hint Types (다중 선택)</label>
        <div style={{ display: 'flex', flexWrap: 'wrap', gap: '8px', padding: '10px', border: '1px solid #ddd', borderRadius: '4px', minHeight: '42px' }}>
          {(codeMap['HINT_TYPES'] ?? []).map((item) => {
            const selected = formData.hintTypes.split(',').filter(Boolean);
            const isChecked = selected.includes(item.code);
            return (
              <label key={item.code} style={{
                display: 'inline-flex', alignItems: 'center', gap: '4px',
                padding: '4px 10px', borderRadius: '4px', cursor: 'pointer', fontSize: '13px',
                border: `1px solid ${isChecked ? '#667eea' : '#ddd'}`,
                backgroundColor: isChecked ? '#f5f7ff' : '#fff',
                color: isChecked ? '#667eea' : '#333',
              }}>
                <input
                  type="checkbox"
                  checked={isChecked}
                  onChange={(e) => {
                    const next = e.target.checked
                      ? [...selected, item.code]
                      : selected.filter((c) => c !== item.code);
                    setFormData((prev) => ({ ...prev, hintTypes: next.join(',') }));
                  }}
                  style={{ display: 'none' }}
                />
                {item.code}
              </label>
            );
          })}
        </div>
        <div style={{ marginTop: '6px', fontSize: '12px', color: '#777' }}>
          문항별 힌트 제공 방식을 정의합니다. 여러 개 선택 가능, 쉼표로 구분 저장됩니다.
        </div>
      </div>

      {/* Retry Scope */}
      <div>
        <label style={{ display: 'block', fontWeight: 600, marginBottom: '8px' }}>Retry Scope</label>
        <select
          name="retryScope"
          value={formData.retryScope}
          onChange={handleInputChange}
          style={FIELD_STYLE}
        >
          <option value="">선택하세요</option>
          {(codeMap['RETRY_SCOPE'] ?? []).map((item) => (
            <option key={item.code} value={item.code}>
              {item.code} - {item.name}
            </option>
          ))}
        </select>
        <div style={{ marginTop: '6px', fontSize: '12px', color: '#777' }}>
          재시도 시 채점 범위를 지정합니다.
        </div>
      </div>

      {/* Input Language */}
      <div>
        <label style={{ display: 'block', fontWeight: 600, marginBottom: '8px' }}>Input Language</label>
        <select
          name="inputLanguage"
          value={formData.inputLanguage}
          onChange={handleInputChange}
          style={FIELD_STYLE}
        >
          <option value="">선택하세요</option>
          {(codeMap['INPUT_LANGUAGE'] ?? []).map((item) => (
            <option key={item.code} value={item.code}>
              {item.code} - {item.name}
            </option>
          ))}
        </select>
        <div style={{ marginTop: '6px', fontSize: '12px', color: '#777' }}>
          학습자가 사용 가능한 입력 언어를 지정합니다.
        </div>
      </div>

      {/* Passage Role */}
      <div>
        <label style={{ display: 'block', fontWeight: 600, marginBottom: '8px' }}>Passage Role</label>
        <select
          name="passageRole"
          value={formData.passageRole}
          onChange={handleInputChange}
          style={FIELD_STYLE}
        >
          <option value="">선택하세요</option>
          {(codeMap['PASSAGE_ROLE'] ?? []).map((item) => (
            <option key={item.code} value={item.code}>
              {item.code} - {item.name}
            </option>
          ))}
        </select>
        <div style={{ marginTop: '6px', fontSize: '12px', color: '#777' }}>
          지문의 역할을 정의합니다.
        </div>
      </div>

      {/* Question Max Attempts */}
      <div>
        <label style={{ display: 'block', fontWeight: 600, marginBottom: '8px' }}>최대 제출 횟수</label>
        <input
          type="number"
          min="0"
          value={formData.questionMaxAttempts ?? ''}
          onChange={(e) => {
            const val = e.target.value;
            setFormData((prev) => ({
              ...prev,
              questionMaxAttempts: val === '' ? undefined : Number(val),
            }));
          }}
          placeholder="예시: 1 또는 3 (비워두면 무제한)"
          style={FIELD_STYLE}
        />
        <div style={{ marginTop: '6px', fontSize: '12px', color: '#777' }}>
          • 무제한: 빈 값 (학생 제출 개념 없음 - CLR 등)
          <br />
          • 1: 단일 제출 후 AI 피드백만 (PRD, GMN, WWB, SCN, SKM, SUM, QAR, SWR, PWR 등)
          <br />
          • 3: 3회 재도전 허용 (WDR, WDS, RRD, SHR 등)
          <br />
          • 기타: 프로젝트 요구에 맞게 자유롭게 입력 가능
        </div>
      </div>
    </div>
  </div>
)}
```

---

### 3.7 [모듈등록] 저장 함수 수정

**파일**: `apps/admin/frontend/src/app/(admin)/admin/ai-modules/register/page.tsx`

`handleSubmit()` 함수의 `CreateAiModuleDto` 구성 부분에 4개 필드 추가:

```typescript
const dto: CreateAiModuleDto = {
  // ... 기존 필드들
  
  // ── 문항설계2 필드 (새로 추가) ──
  hintTypes: formData.hintTypes,
  retryScope: formData.retryScope,
  inputLanguage: formData.inputLanguage,
  passageRole: formData.passageRole,
  questionMaxAttempts: formData.questionMaxAttempts,
};
```

---

### 3.8 [Backend - API] 컨트롤러 및 서비스 수정

**파일**: 
- `apps/admin/backend/src/ai-module/ai-module.controller.ts`
- `packages/core/src/ai-module/ai-module.service.ts` - 비즈니스 로직
- `packages/types/src/index.ts` - DTO/인터페이스

#### 작업 내용

1. **CreateAiModuleDto / UpdateAiModuleDto** 업데이트: 5개 필드 추가
2. **packages/core** (비즈니스 로직):
   - `create()` / `update()` / `toResponse()` 메서드에 필드 매핑

### 3.9 [Backend] VALID_SKILLS 확장

**파일**: `packages/core/src/ai-module/ai-module.service.ts`

현재 `VALID_SKILLS = ['vocabulary', 'reading', 'speaking']` → `writing` 추가:

```typescript
const VALID_SKILLS = ['vocabulary', 'reading', 'speaking', 'writing'];
```

> **확정**: 스프레드시트의 `voca` = `vocabulary`로 등록. `pronunciation` skill은 존재하지 않음.

---

## 4. [모듈 데이터 등록] 14개 모듈 일괄 등록 계획

### 4.1 사전 조건

모듈 등록 전 아래 코드 항목이 **code_items에 존재**해야 한다.  
시드 또는 코드관리 UI에서 먼저 추가할 것.

#### 문항설계 탭 코드 (기존 탭, 시드에 미정의 — UI에서 수동 등록 여부 확인 필요)

| 그룹 | 필요한 코드 |
|------|------------|
| SCORING_MODE | `exact`, `holistic`, `pronunciation` |
| ANSWER_TYPE | `short-text`, `audio-record`, `essay`, `multiple-choice`, `mixed`, `sentence-write` |
| PASSAGE_EXPOSURE_MODE | `full`, `hidden`, `preview` |
| QUESTION_COUNT | `single`, `multi` |
| FEEDBACK_STYLE | `correct-wrong`, `strengths-weaknesses` |

#### 문항설계2 탭 코드 (3.2.1절 시드 데이터로 등록)

| 그룹 | 필요한 코드 |
|------|------------|
| HINT_TYPES | `spelling-prefix`, `audio-replaysyllable-guide`, `english-definition`, `relation-meaning`, `bridge-question`, `korean-summary`, `topic-sentence`, `korean-translation`, `audio-replayfluency-points` |
| RETRY_SCOPE | `spelling-accuracy`, `pronunciation-accuracy`, `inference-accuracy`, `semantic-coverage`, `keyword-hit-rate`, `topic-sentence-match`, `engagement-depth`, `structural-score`, `evidence-accuracy`, `writing-quality`, `logical-coherence` |
| INPUT_LANGUAGE | `english`, `audio`, `korean`, `korean-or-english` |
| PASSAGE_ROLE | `context-reference`, `web-root`, `prediction-trigger`, `scan-target`, `reading-material`, `fluency-practice-target`, `writing-reference` |

### 4.2 모듈 등록 데이터 (14개)

> `none` 값은 빈 문자열(`''`)로 저장한다.

| 코드 | name | skill | scoringMode | answerType | passageExposureMode | questionCount | feedbackStyle | questionMaxAttempts | hintTypes | retryScope | inputLanguage | passageRole |
|------|------|-------|-------------|------------|---------------------|---------------|---------------|---------------------|-----------|------------|---------------|-------------|
| WDR | Word Decker Reading | vocabulary | exact | short-text | full | multi | correct-wrong | 3 | spelling-prefix | spelling-accuracy | english | context-reference |
| WDS | Word Decker Speaking | vocabulary | exact | audio-record | full | multi | correct-wrong | 3 | audio-replaysyllable-guide | pronunciation-accuracy | audio | context-reference |
| GMN | Guessing Meaning | vocabulary | holistic | essay | full | multi | strengths-weaknesses | 1 | english-definition | inference-accuracy | korean | context-reference |
| WWB | Word Web | vocabulary | holistic | short-text | hidden | multi | strengths-weaknesses | 1 | relation-meaning | semantic-coverage | english | web-root |
| PRD | Prediction | reading | holistic | essay | preview | single | strengths-weaknesses | 1 | bridge-question | inference-accuracy | korean-or-english | prediction-trigger |
| SCN | Scanning | reading | holistic | short-text | hidden | single | strengths-weaknesses | 1 | korean-summary | keyword-hit-rate | korean-or-english | scan-target |
| SKM | Skimming | reading | exact | multiple-choice | full | single | correct-wrong | 1 | | topic-sentence-match | | scan-target |
| CLR | Clarification | reading | holistic | essay | full | multi | strengths-weaknesses | *(null)* | | engagement-depth | | reading-material |
| SUM | Summarizing | reading | holistic | essay | full | single | strengths-weaknesses | 1 | topic-sentence | structural-score | korean-or-english | reading-material |
| QAR | Question-Answer Relationship | reading | exact | mixed | full | multi | correct-wrong | 1 | korean-translation | evidence-accuracy | | reading-material |
| RRD | Repeated Reading | reading | pronunciation | audio-record | full | multi | strengths-weaknesses | 3 | audio-replayfluency-points | pronunciation-accuracy | audio | fluency-practice-target |
| SHR | Story Question Type Analysis | speaking | holistic | audio-record | full | multi | strengths-weaknesses | 3 | audio-replayfluency-points | pronunciation-accuracy | audio | fluency-practice-target |
| SWR | Sentence Writing | writing | holistic | sentence-write | full | multi | strengths-weaknesses | 1 | | writing-quality | english | writing-reference |
| PWR | Paragraph Writing | writing | holistic | essay | full | multi | strengths-weaknesses | 1 | | logical-coherence | korean-or-english | writing-reference |

### 4.3 Purpose / Pedagogy Instruction (장문 텍스트)

각 모듈별 `purpose`와 `pedagogyInstruction`은 스프레드시트에 별도 텍스트로 존재.  
모듈 등록 시 해당 텍스트를 함께 입력한다.

| 코드 | purpose 요약 | pedagogyInstruction 시작 |
|------|-------------|------------------------|
| WDR | 단어 이미지 → 스펠링 타이핑 | [모듈: Word Decker Reading (WDR) — 시각 반복 플래시카드] |
| WDS | 음성 듣고 단어 발음 녹음, ASR 발음 정확도 | [모듈: Word Decker Speaking (WDS) — 발화 중심 어휘 각인] |
| GMN | Context Clues로 유추, 논리적 근거 연습 | [모듈: Guessing Meaning (GMN) — 문맥 유추 전략] |
| WWB | 학생 지문 속 어휘를 동의어/관계 시각화 | [모듈: Word Web (WWB) — 어휘망 확장] |
| PRD | 지문 제목/삽화로 내용 예측 전략 | [모듈: Prediction (PRD) — 내용 예측 전략] |
| SCN | 특정 정보(숫자, 이름, 날짜) 빠르게 찾기 | [모듈: Scanning (SCN) — 특정 정보 색출] |
| SKM | 지문 전체적 대의(Gist) 빠르게 파악 | [모듈: Skimming (SKM) — 대의 파악 전략] |
| CLR | 읽기 도중 이해 안 되는 구간 질문하여 명료화 | [모듈: Clarification (CLR) — 의미 명료화] |
| SUM | 읽은 내용을 핵심 키워드로 요약 | ## 모듈 목적 |
| QAR | 질문-답 관계 유형 분류 전략 | ## 모듈 목적 |
| RRD | 지문 음독, 발음/유창성 측정 | ## 모듈 목적 |
| SHR | 질문의 유형(Right There, Think and Search, Author and Me, On My Own)을 분석 | ## 모듈 목적 |
| SWR | 그 지문의 어법에 재구성하는 작문 연습 | ## 모듈 목적 |
| PWR | 문자 지문의 관계를 이해하여 전략적 문제 해결력 향상 | ## 모듈 목적 |

### 4.4 등록 방식

**시드 스크립트** (`prisma/seed.ts`)로 일괄 등록한다.

- AiModule의 `code` 필드는 `@unique`이므로 **upsert** 패턴 사용
- 이미 존재하는 모듈 → 데이터 업데이트
- 존재하지 않는 모듈 → 새로 생성
- 기존 코드 그룹/항목과 동일한 패턴 (`prisma.aiModule.upsert`)

```typescript
// 시드 upsert 패턴
for (const module of aiModules) {
  await prisma.aiModule.upsert({
    where: { code: module.code },
    update: {
      // code 제외 모든 필드 업데이트
      skill: module.skill,
      name: module.name,
      scoringMode: module.scoringMode,
      // ... 나머지 필드
    },
    create: {
      // 전체 필드
      code: module.code,
      skill: module.skill,
      name: module.name,
      // ... 나머지 필드
    },
  });
}
```

> **주의**: `purpose`, `pedagogyInstruction`은 이미 수동 입력된 값이 있을 수 있으므로,  
> update 시 빈 문자열이면 기존 값을 덮어쓰지 않도록 조건부 업데이트를 고려한다.

### 4.5 확정 사항

| 항목 | 결정 |
|------|------|
| skill 코드 | 스프레드시트 `voca` → `vocabulary`로 등록 |
| `pronunciation` skill | 존재하지 않음. RRD의 scoringMode `pronunciation`은 별개 (skill은 `reading`) |
| `none` 처리 | hintTypes/inputLanguage의 `none` → 빈 문자열(`''`) |
| `none (click)` 처리 | inputLanguage의 `none (click)` → 빈 문자열(`''`) |
| 모듈 name 필드 | Pedagogy Instruction에서 추출 (예: WDR → Word Decker Reading) |
| 등록 방식 | 시드 스크립트 (`prisma/seed.ts`) |

---

## 5. 구현 순서

### Phase 1: 기반 작업 (완료)
1. ~~Prisma 스키마 수정 → 마이그레이션 실행~~
2. ~~타입 정의 (`packages/types`) → 빌드~~
3. ~~코드관리 페이지 탭 추가~~
4. ~~모듈등록 페이지 폼 추가~~
5. ~~Backend API 필드 매핑 추가 (`packages/core`) → 빌드~~

### Phase 2: 수정 작업 (신규)
6. **시드 데이터 확장** — 문항설계2 코드 항목 전체 추가 → `npx prisma db seed`
7. **hintTypes UI 변경** — 단일 select → 다중 선택 체크박스 태그
8. **VALID_SKILLS 확장** — `writing`, `pronunciation` 추가
9. **문항설계 탭 코드 항목 확인** — SCORING_MODE, ANSWER_TYPE 등 기존 탭 코드 존재 여부 확인/추가

### Phase 3: 모듈 데이터 등록
10. **미결정 사항 확정** (skill 코드, none 처리 등)
11. **모듈 14개 등록** (방식 확정 후 실행)
12. **통합 검증** — 등록된 모듈 조회/수정 동작 확인

---

## 6. 검증 항목

- [ ] Prisma 마이그레이션 성공 및 DB 스키마 변경 확인 (5개 컬럼 추가)
- [ ] 시드 실행 후 `code_groups` 4개 그룹 + 전체 `code_items` 생성 확인
- [ ] 코드관리 페이지에서 4개 코드 그룹에 대한 CRUD 원활함
- [ ] hintTypes 다중 선택 UI 정상 동작 (선택/해제/저장/로드)
- [ ] 모듈 신규 등록 시 5개 필드(4개 코드그룹 + questionMaxAttempts) 저장 확인
- [ ] 모듈 수정 시 5개 필드 로드 및 저장 확인
- [ ] questionMaxAttempts 숫자 입력 필드 동작 확인 (빈 값/1/3/기타 값)
- [ ] VALID_SKILLS 확장 후 writing/pronunciation skill 모듈 등록 가능 확인
- [ ] 14개 모듈 등록 후 목록 조회 정상 확인
- [ ] 빌드 오류 및 타입 에러 없음 (`pnpm run build`)
- [ ] API 응답 포맷 확인

---

## 7. 참고사항

### 명칭 규칙
- **영문 필드명** (코드): snake_case (예: `hint_types`, `retry_scope`)
- **UI 표시명**: 한글 (예: "힌트 타입", "재시도 범위")
- **코드 그룹**: UPPER_SNAKE_CASE (예: `HINT_TYPES`, `RETRY_SCOPE`)

### 기존 패턴 유지
- `handleInputChange()` 훅 활용 (단일 선택 필드)
- hintTypes는 체크박스 전용 핸들러 사용 (쉼표 구분 문자열 join/split)
- `nameCodeColumns` 포맷 사용
- `codeMap` 동적 로드
- 저장 시 nullable한 값은 명시적으로 빈 문자열('') 또는 null 처리

### 후속 작업
- 다른 모듈별 코드 관리 페이지의 통일성 검토
- API 문서 업데이트 (OpenAPI/Swagger)
- 마이그레이션 롤백 계획 사전 수립

