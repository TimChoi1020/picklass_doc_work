# 과정 생성 — 모듈 UI 통일 + 시퀀싱 적용 계획

**작성일**: 2026-04-12  
**최종 수정**: 2026-04-24 (AI 회차도 시퀀싱 적용 — passageId 의존 제거)  
**범위**: `/course` 과정 생성 마법사  
**참고**: `/class/lesson-setup/[passageId]/page.tsx` (기준 UI 및 시퀀싱 로직)

---

## 0. 핵심 발견 (정책 재정립)

### analyzer 실제 요구사항

`apps/api/src/lesson-plan/lesson-plan.service.ts:37-63` 분석 결과:

```typescript
const analyzerReq: LessonPlanAnalyzerRequest = {
  passage_level: passage.level,        // ★ 필수
  selected_kpi_codes: dto.selected_kpi_codes,
  duration_min: dto.duration_min,
  passage_text: passage.content ?? undefined,  // optional
};
```

→ **analyzer는 `passage_level`만 있으면 시퀀싱 가능.** `passage_id`는 NestJS가 DB에서 level을 조회하는 우회 수단일 뿐, analyzer 자체는 level만 알면 됨.

### 결론

**AI 회차도 Step 1의 `wizardLevel`을 직접 사용하면 시퀀싱 가능.**  
이전 옵션 A의 "AI 회차는 시퀀싱 불가" 가정은 **잘못된 전제**. 폐기.

---

## 1. 확정 정책 (재정립)

| # | 항목 | 결정 |
|---|------|------|
| 1 | lesson-setup 인라인 JSX 리팩토링 | 하지 않음 |
| 2 | 시퀀싱 적용 범위 | **모든 회차 (AI / 기존 레슨 모두)** |
| 3 | 시퀀싱 입력 — 레벨 | 기존 레슨 회차: 레슨의 `level` 사용 / AI 회차: Step 1 `wizardLevel` 사용 |
| 4 | 시퀀싱 입력 — KPI | Step 1 `selectedObjectives` (모든 회차 공통) |
| 5 | 시퀀싱 입력 — duration | 회차별 입력 |
| 6 | duration UI 위치 | Step 2 회차별 (lesson-setup 동일 UI) |
| 7 | "n개 선택됨" 표시 | 삭제 |
| 8 | 시퀀싱 트리거 | 회차별 duration 선택 시 |
| 9 | 사전 채움 | 없음 — 빈 상태 시작 (lesson-setup 동일) |

---

## 2. 시퀀싱 트리거 조건 (회차별)

```
시퀀싱 호출 가능 (회차 idx) =
  (level 확보 가능: 기존 레슨이면 lesson.level / AI면 wizardLevel) AND
  selectedObjectives 1개 이상 AND
  topicsDuration[idx] ∈ [15, 20, 25, 30]
```

만족 시 호출. AI 회차의 경우 Step 1의 wizardLevel을 사용. duration 미선택이면 안내 메시지.

---

## 3. 백엔드 변경

### 3-1. DTO 변경 — passage_id를 optional로

**파일**: `apps/api/src/lesson-plan/dto/lesson-plan.dto.ts`

```typescript
export class GenerateLessonPlanDto {
  @IsOptional()                          // ← 변경
  @IsInt()
  passage_id?: number;

  @IsOptional()                          // ← 신규
  @IsString()
  passage_level?: string;                // ← 신규 (passage_id 없을 때 사용)

  @IsArray() @ArrayMinSize(1)
  @IsString({ each: true })
  selected_kpi_codes!: string[];

  @IsInt() @IsIn(ALLOWED_DURATIONS)
  duration_min!: AllowedDuration;
  // ...
}
```

서비스 측에서 `passage_id` XOR `passage_level` 검증 (둘 중 하나는 필수).

### 3-2. Service 변경 — passage_id 또는 level 분기

**파일**: `apps/api/src/lesson-plan/lesson-plan.service.ts`

```typescript
async generate(dto: GenerateLessonPlanDto): Promise<LessonPlanResponse> {
  let passageLevel: string;
  let passageText: string | undefined = undefined;

  if (dto.passage_id) {
    const passage = await this.prisma.texts.findUnique({
      where: { id: dto.passage_id },
      select: { level: true, content: true },
    });
    if (!passage) throw new NotFoundException(...);
    if (!passage.level) throw new BadRequestException(...);
    passageLevel = passage.level;
    passageText = passage.content ?? undefined;
  } else if (dto.passage_level) {
    passageLevel = dto.passage_level;
    // passage_text는 없음 (AI 회차 - 지문 미생성)
  } else {
    throw new BadRequestException('passage_id 또는 passage_level 중 하나는 필수입니다.');
  }

  const analyzerReq: LessonPlanAnalyzerRequest = {
    passage_level: passageLevel,
    selected_kpi_codes: dto.selected_kpi_codes,
    duration_min: dto.duration_min,
    skill_filter: dto.skill_filter,
    learner_id: dto.learner_id,
    passage_text: passageText,
  };

  return this.callAnalyzer(analyzerReq);
}
```

---

## 4. 프론트엔드 변경

### 4-1. API 클라이언트

**파일**: `apps/web/src/lib/api.ts`

`GenerateLessonPlanRequest`에 `passage_level?: string` 추가, `passage_id?` 로 변경.

### 4-2. 훅 시그니처 변경

**파일**: `apps/web/src/hooks/use-lesson-plan.ts`

```typescript
export function useLessonPlanSequence(
  input: { passageId: number | null; passageLevel: string | null },
  kpiCodes: string[],
  duration: number | null,
)
```

내부:
- enabled 조건: `(passageId || passageLevel)` AND kpiCodes ≥ 1 AND duration valid
- queryFn: passage_id 우선, 없으면 passage_level 전달

또는 기존 시그니처 유지하고 wrapper 함수 추가:

```typescript
useLessonPlanSequenceByLevel(level, kpiCodes, duration)
```

### 4-3. LessonModuleSection 변경

**파일**: `apps/web/src/components/LessonModuleSection.tsx`

Props 변경:
```typescript
interface Props {
  passageId: number | null;
  passageLevel: string | null;   // ← 신규 (AI 회차용)
  kpiCodes: string[];
  duration: number | null;
  // ...
}
```

내부에서 `useLessonPlanSequence({ passageId, passageLevel }, ...)` 호출.

조건 안내 메시지 단순화:
- KPI 0개 → "Step 1에서 학습 목표(KPI)를 선택하면 자동 시퀀싱이 적용됩니다"
- duration 미선택 → "수업 시간을 선택하면 자동 시퀀싱이 적용됩니다"
- AI 회차 안내는 **삭제** (시퀀싱 작동하므로)

### 4-4. course/page.tsx — passageLevel 전달

```typescript
<LessonModuleSection
  passageId={
    topicsSource[idx] === 'lesson'
      ? selectedLessonPerTopic[idx]?.id ?? null
      : null
  }
  passageLevel={
    topicsSource[idx] === 'lesson'
      ? selectedLessonPerTopic[idx]?.level ?? null
      : wizardLevel || null   // AI 회차: Step 1의 wizardLevel
  }
  kpiCodes={selectedObjectives}
  duration={topicsDuration[idx] ?? null}
  // ...
/>
```

---

## 5. 수정 파일 요약

| 파일 | 변경 |
|------|------|
| `apps/api/src/lesson-plan/dto/lesson-plan.dto.ts` | passage_id optional, passage_level 신규 |
| `apps/api/src/lesson-plan/lesson-plan.service.ts` | passage_id/level 분기 |
| `apps/web/src/lib/api.ts` | GenerateLessonPlanRequest 타입 확장 |
| `apps/web/src/hooks/use-lesson-plan.ts` | 훅 시그니처 (passage_level 받기) |
| `apps/web/src/components/LessonModuleSection.tsx` | passageLevel prop, 안내 메시지 정리 |
| `apps/web/src/app/course/page.tsx` | passageLevel 전달 (AI=wizardLevel, lesson=lesson.level) |

---

## 6. 영향 범위

### 변경 영향
- **lesson-setup 페이지**: 영향 없음 — 기존 `passage_id` 흐름 그대로 동작
- **course Step 2**: 모든 회차에서 시퀀싱 동작
- **백엔드**: API 스펙 확장(하위 호환). 기존 호출은 그대로 동작

### 변경 없음
- analyzer (Python 서버)
- DB 스키마

---

## 7. 검증 시나리오 (체크리스트 적용)

### 사용자 환경에서 직접 확인할 항목

| # | 시나리오 | 기대 동작 | 확인 방법 |
|---|---------|----------|----------|
| 1 | Step 2 진입 (AI 회차) | 모듈 빈 상태 + 안내 (KPI/duration 안내) | 화면 |
| 2 | AI 회차에서 duration 선택 | analyzer 호출 → sequence로 모듈 자동 채움 | Network 탭 + 화면 |
| 3 | 기존 레슨 회차에서 lesson 선택 → duration 선택 | analyzer 호출 (passage_id 사용) | Network 탭 |
| 4 | 회차 1, 2 모두 duration 선택 | 각 회차 독립 시퀀싱 호출 | Network 탭 (2회 호출) |
| 5 | 회차 1 duration 변경 | 회차 1만 재호출, 회차 2 영향 없음 | Network 탭 |
| 6 | KPI 0개 + duration 선택 | 안내 표시, API 미호출 | 화면 + Network |
| 7 | 시퀀싱 실패 → 다시 시도 | refetch 동작 | 화면 |

**모든 시나리오를 dev 서버에서 직접 확인 후 보고.**

---

## 8. 작업 순서

1. 백엔드 DTO/Service 확장 (passage_id optional + passage_level)
2. 프론트엔드 API 타입 + 훅 시그니처
3. LessonModuleSection passageLevel prop 추가
4. course/page.tsx에서 wizardLevel/lesson.level 전달
5. 안내 메시지 정리 (AI 안내 제거)
6. 빌드 검증 + `.next/` 정리
7. 시나리오 1~7 dev 서버 직접 확인
8. 보고 (Network 탭 호출 결과 포함)

---

## 9. 구현 결과 (2026-04-25 기준)

### 9-1. 완료된 작업

#### Phase A — "n개 선택됨" 삭제 ✅
- `apps/web/src/app/course/page.tsx` line 721-725 삭제

#### Phase 1 — 모듈 UI 통일 (ModuleChipList) ✅
- `apps/web/src/components/ModuleChipList.tsx` 신규 (lesson-setup과 동일 칩 스타일)
- `course/page.tsx` Step 2 모듈 영역 교체

#### Phase 2 — duration + 시퀀싱 (모든 회차) ✅
- 백엔드 DTO/Service: `passage_id` optional + `passage_level` 신규
- `useLessonPlanSequence({passageId, passageLevel}, ...)` 시그니처 변경 (객체 형태)
- `LessonModuleSection.tsx` 신규: 회차 단위 duration UI + 시퀀싱 호출 캡슐화
- `course/page.tsx`:
  - `topicsDuration` state 추가 + 4개 위치 동기화
  - 사전 채움(`getRecommendedModules`) 제거 — 빈 상태로 시작
  - lesson 모드: `passageLevel = lesson.level`
  - AI 모드: `passageLevel = wizardLevel`
- `lesson-setup` 호출도 객체 형태로 업데이트 (호환성 유지)

#### 추가 발견 + 수정 — 모듈 수정 모달 순서 버그 (Phase 3) ✅
**문제**: `course/[courseId]/page.tsx`의 모듈 수정 모달에서 시퀀스 번호가 DB 저장 순서가 아닌 카테고리 표시 순서로 부여되고 있었음.

```typescript
// Before (버그): moduleCategories 표시 순서로 재정렬
const selectedInOrder = Object.values(moduleCategories)
  .flatMap(c => c.modules)
  .filter(m => modulesToEdit.includes(m.code))
  .map(m => m.code);
const order = isSelected ? selectedInOrder.indexOf(module.code) + 1 : null;

// After (수정): modulesToEdit 자체의 순서 사용
const order = isSelected ? modulesToEdit.indexOf(module.code) + 1 : null;
```

**검증 데이터**:
- DB: `["WSD", "WRD", "IMG", "FRT"]` (시퀀싱 결과 그대로)
- 화면 (수정 전): `[FRT(1), WRD(2), IMG(3), WSD(4)]` ❌ 카테고리 순서로 재정렬됨
- 화면 (수정 후): `[WSD(1), WRD(2), IMG(3), FRT(4)]` ✅ DB 순서 그대로

### 9-2. DB 검증 결과

`course_lessons.skill_modules` 컬럼 (JSON 배열) — 시퀀싱 결과 순서 그대로 저장됨:

| 과정명 | 생성일 | 모듈 수 | 비고 |
|--------|--------|---------|------|
| 메이저리그 | 2026-04-25 16:11 | 4개 (시퀀스) | 시퀀싱 정상 작동 |
| 읽기 초급 과정 | 2026-04-25 09:43 | 21개 (전체) | 시퀀싱 적용 전 |
| 과정 10개 테스트 | 2026-04-23 09:59 | 21개 (전체) | 시퀀싱 적용 전 |

→ 시퀀싱 적용 이후(2026-04-25 16:11) 결과 배열만 DB에 저장됨.

### 9-3. 수정 파일 최종 목록

| 파일 | 변경 |
|------|------|
| `apps/api/src/lesson-plan/dto/lesson-plan.dto.ts` | `passage_id` optional + `passage_level` 신규 |
| `apps/api/src/lesson-plan/lesson-plan.service.ts` | passage_id/level 분기 처리 |
| `apps/web/src/lib/api.ts` | `GenerateLessonPlanRequest.passage_level` 추가 |
| `apps/web/src/lib/react-query.ts` | `lessonPlan.sequence(sourceKey, ...)` 키 형태 변경 |
| `apps/web/src/hooks/use-lesson-plan.ts` | 훅 시그니처 객체 형태로 변경 |
| `apps/web/src/components/ModuleChipList.tsx` | 신규 (Phase 1) |
| `apps/web/src/components/LessonModuleSection.tsx` | 신규 (Phase 2) |
| `apps/web/src/app/course/page.tsx` | "n개 선택됨" 삭제, duration state, LessonModuleSection 적용, passageLevel 전달 |
| `apps/web/src/app/course/[courseId]/page.tsx` | 모듈 수정 모달 순서 버그 수정 |
| `apps/web/src/app/class/lesson-setup/[passageId]/page.tsx` | 훅 객체 시그니처 적용 (호환성 유지) |

### 9-4. 빌드/배포

- API + Web build 모두 성공
- `.next/` 정리 완료 (dev 서버 충돌 방지)

### 9-5. 알려진 의존성

- analyzer 서버(Python) 정상 동작 필요
- analyzer가 자체 Supabase 연결 가능해야 함 (직접 연결 → pooler URL 권장)

---

## 10. 추가 작업 — Step 1에 수업 시간 추가 (2026-04-25)

### 10-1. 정책

- **Step 1 (1/2) 폼 8번 항목으로 "회차별 수업 시간" 추가** (장르 다음)
- 옵션: 15/20/25/30분 (lesson-setup 동일 UI)
- Step 1에서 선택한 값이 **Step 2 모든 회차에 디폴트로 적용**
- Step 2 진입 시 KPI + duration 충족 → **모든 회차 즉시 시퀀싱 자동 호출**
- 회차별로 개별 변경 가능 (Step 2의 LessonModuleSection)

### 10-2. 변경 사항

| 영역 | 변경 |
|------|------|
| state | `wizardDuration: number \| null` 추가 |
| Step 1 UI | 8번 항목 "회차별 수업 시간" UI 추가 (15/20/25/30분 버튼) |
| Step 1 validation | `wizardDuration ∈ [15,20,25,30]` 필수 검증 |
| `runGenerateAITopics` | `setTopicsDuration(Array(rounds).fill(wizardDuration))` |
| lessonRound 변경 핸들러 | 동일 (회차 수 변경 시 디폴트 채움) |
| reset 4곳 | `setWizardDuration(null)` 추가 |

### 10-3. 사용자 흐름

```
Step 1
├── 1. 과정명
├── 2. 과정 목표 (KPI)
├── 3. 과정주제
├── 4. 레슨 회차
├── 5. CEFR 레벨
├── 6. 지문 단어수
├── 7. 장르
└── 8. 회차별 수업 시간 ← 신규
        │
        ▼ "다음" 클릭 (validation)
        │
Step 2 (자동)
├── topicsDuration[*] = wizardDuration 채워짐
├── KPI(Step 1) + duration 충족 → 모든 회차 시퀀싱 자동 호출
└── 회차별로 duration 개별 변경 가능
```

### 10-4. 수정 파일

| 파일 | 변경 |
|------|------|
| `apps/web/src/app/course/page.tsx` | wizardDuration state, Step 1 UI, validation, 디폴트 채움, reset |

### 10-5. 빌드/검증

- Web build 성공
- `.next/` 정리 완료
