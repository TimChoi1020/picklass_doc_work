# Class 페이지 수정 내역

## 📋 개요
2026년 3월 11일에 진행된 클래스 페이지 및 관련 모달 수정 사항 정리

---

## 🎨 1. CEFR 레벨 선택 개선

### 변경 파일
- `apps/web/src/components/AIGenerateModal.tsx`

### 변경 사항
#### Before
```javascript
const CEFR_LEVELS = ['A1', 'A2', 'B1', 'B2', 'C1', 'C2'];
// 6개 레벨, 1줄 레이아웃, 버튼 크기: h-11, w-[50px]
```

#### After
```javascript
const CEFR_LEVELS = ['A1-', 'A1', 'A1+', 'A2-', 'A2', 'A2+', 'B1-', 'B1', 'B1+', 'B2-', 'B2', 'B2+', 'C1-', 'C1', 'C1+', 'C2-', 'C2', 'C2+'];
// 18개 레벨, 2줄 레이아웃 (grid-cols-9), 버튼 크기: h-9
```

### 기술 상세
- **레이아웃**: Flexbox → CSS Grid (`grid-cols-9`)
- **버튼 높이**: h-11 → h-9 (줄어들음)
- **버튼 폭**: w-[50px] (고정) → 자동 (grid 내에서 균등 분배)
- **텍스트 크기**: text-[15px] → text-xs
- **간격**: gap-3 → gap-1.5

### 개발자 추가 작업
- [ ] CEFR 레벨 확장에 따른 API 스키마 검증
  - `/api/generate-text` API에서 18개 레벨 모두 지원하는지 확인
  - 백엔드 CEFR 레벨 검증 로직 업데이트 필요
- [ ] 모달 높이 재조정 (버튼 크기 축소로 인한 모달 전체 높이 변경)
- [ ] 모바일/태블릿 반응형 처리
  - 작은 화면에서 grid-cols-9가 적절한지 테스트
  - 필요시 grid-cols-6 또는 다른 breakpoint 추가

---

## 🎨 2. 텍스트 색상 가시성 개선

### 변경 파일
- `apps/web/src/app/class/page.tsx`

### 변경 사항

#### 2-1. 인사말 텍스트 (라인 407)
```jsx
// Before
<div className="text-3xl leading-[38px]">

// After
<div className="text-3xl leading-[38px] text-gray-900">
```

#### 2-2. 검색 필드 입력 (라인 584)
```jsx
// Before
<Input
  placeholder="지문 찾기"
  className="h-10 rounded-full border-gray-400 pl-10 text-sm"
/>

// After
<Input
  placeholder="지문 찾기"
  className="h-10 rounded-full border-gray-400 bg-white pl-10 text-sm text-gray-900 placeholder:text-gray-400"
/>
```

#### 2-3. 필터 버튼들 (라인 640-655)
```jsx
// Before
<span className="text-sm font-semibold">CEFR</span>

// After
<span className="text-sm font-semibold text-gray-900">CEFR</span>
```

- CEFR, 단어수, 장르, 최신순 모두 동일 처리

#### 2-4. 정렬 아이콘 (라인 650)
```jsx
// Before
<ArrowUpDown className="h-5 w-5" />

// After
<ArrowUpDown className="h-5 w-5 text-gray-900" />
```

### 개발자 추가 작업
- [ ] 다크모드 지원 시 색상 변경 - Tailwind dark: prefix 추가
  ```jsx
  text-gray-900 dark:text-white
  placeholder:text-gray-400 dark:placeholder:text-gray-300
  ```

---

## 📊 3. 지문 난이도 분석 섹션 추가

### 변경 파일
- `apps/web/src/components/PassageDetailModal.tsx`

### 변경 사항
PassageDetailModal의 Tags Row 아래에 "지문 난이도 분석" 섹션 추가

#### 새로 추가된 구조
```jsx
{/* Analysis Indicators */}
<div className="flex flex-col gap-3">
  <p className="text-xs font-semibold text-gray-600">지문 난이도 분석</p>
  <div className="flex flex-wrap gap-2">
    {/* 8개 지표 배지 */}
    <Badge variant="secondary" className="bg-gray-200 text-gray-600">
      <span className="text-[11px]">어휘 난이도</span>
      <span className="ml-1 font-semibold">B1+</span>
    </Badge>
    {/* ... 반복 */}
  </div>
</div>
```

#### 8개 지문 난이도 분석 지표
1. **어휘의 난이도** → 현재값: B1+
2. **어휘 다양성** → 현재값: B1
3. **지문의 길이** → 현재값: A2+
4. **문장 길이** → 현재값: B1-
5. **문장 구조** → 현재값: A2
6. **문법 다양성** → 현재값: B1
7. **정보 밀도** → 현재값: B2-
8. **배경지식 의존도** → 현재값: A2+

### 스타일 상세
- **배경색**: bg-gray-200 (밝은 회색)
- **텍스트색**: text-gray-600 (어두운 회색)
- **배지 높이**: 기본 Badge 컴포넌트
- **텍스트 크기**: 지표명 text-[11px], CEFR 레벨 font-semibold
- **간격**: gap-3 (섹션), gap-2 (배지 간)

### 개발자 추가 작업
- [ ] **백엔드 API 개발 필수**
  - 지문 분석 API 엔드포인트 생성 필요 (예: `/api/analyze-passage`)
  - 입력: 지문 내용
  - 출력: 8개 지표에 대한 CEFR 레벨

- [ ] **지문 분석 로직 구현**
  - 지문 데이터베이스에 다음 컬럼 추가 필요:
    ```sql
    ALTER TABLE texts ADD COLUMN analysis JSONB;
    -- 예시 구조:
    {
      "vocab_difficulty": "B1+",
      "vocab_variety": "B1",
      "passage_length": "A2+",
      "sentence_length": "B1-",
      "sentence_structure": "A2",
      "grammar_variety": "B1",
      "information_density": "B2-",
      "background_knowledge_dependency": "A2+"
    }
    ```

- [ ] **PassageDetailModal props 확장**
  ```typescript
  interface PassageDetailModalProps {
    // ... 기존 props
    analysisData?: {
      vocab_difficulty: string;
      vocab_variety: string;
      passage_length: string;
      sentence_length: string;
      sentence_structure: string;
      grammar_variety: string;
      information_density: string;
      background_knowledge_dependency: string;
    };
  }
  ```

- [ ] **동적 데이터 바인딩**
  - 현재는 하드코딩된 값 → 실제 분석 데이터로 교체 필요
  - `page.tsx`에서 PassageDetailModal로 analysisData prop 전달

- [ ] **AI 생성 지문의 자동 분석**
  - `/api/generate-text` API에서 지문 생성 시 동시에 분석 실행
  - 생성된 지문의 난이도 분석 자동 저장

## 🔍 4. 필터링 및 검색 기능 구현

### 변경 파일
- `apps/web/src/app/class/page.tsx`
- `packages/shared/src/constants/index.ts` (새로운 상수 추가)

### 4-1. 공유 상수 추가 (WORD_COUNT_RANGES)

#### packages/shared/src/constants/index.ts
```typescript
// ============================================
// 단어수 범위 시스템
// ============================================

export const WORD_COUNT_RANGES = [
  { label: '0~100', value: '0~100' },
  { label: '100~200', value: '100~200' },
  { label: '200~300', value: '200~300' },
  // ... (20개 범위, 100씩 증가)
  { label: '1900~2000', value: '1900~2000' },
] as const;

export type WordCountRange = typeof WORD_COUNT_RANGES[number]['value'];
```

**특징**:
- 0~2000까지 100씩 증가 (총 20개 범위)
- course/[courseId]/page.tsx와 class/page.tsx에서 공유
- 타입 안전성: `WordCountRange` 타입 제공

### 4-2. 클래스 필터링 UI 구현

#### State 추가
```typescript
const [filterLevel, setFilterLevel] = useState<string>('');
const [filterGenre, setFilterGenre] = useState<string>('');
const [filterWordCount, setFilterWordCount] = useState<string>('');
const [searchText, setSearchText] = useState<string>('');
const [openLevelDropdown, setOpenLevelDropdown] = useState(false);
const [openGenreDropdown, setOpenGenreDropdown] = useState(false);
const [openWordCountDropdown, setOpenWordCountDropdown] = useState(false);
```

#### 필터링 로직
```typescript
const filteredLibraryTexts = useMemo(() => {
  return libraryTexts.filter((text) => {
    if (searchText && !text.title.toLowerCase().includes(searchText.toLowerCase())) return false;
    if (filterLevel && text.level !== filterLevel) return false;
    if (filterGenre && text.category !== filterGenre) return false;
    if (filterWordCount && text.wordCountRange !== filterWordCount) return false;
    return true;
  });
}, [libraryTexts, searchText, filterLevel, filterGenre, filterWordCount]);
```

### 4-3. 검색 기능

#### 검색창 구현
```jsx
<Input
  placeholder="지문 찾기"
  value={searchText}
  onChange={(e) => {
    setSearchText(e.target.value);
    setCurrentPage(1);  // 검색 시 첫 페이지로 리셋
  }}
  className="h-10 rounded-full border-gray-400 bg-white pl-10 text-sm text-gray-900 placeholder:text-gray-400"
/>
```

**동작**:
- 실시간 제목 검색 (대소문자 구분 안 함)
- 필터와 함께 사용 가능
- 검색 시 페이지를 자동으로 1로 리셋

### 4-4. 드롭다운 필터 UI

#### CEFR 필터 (예시)
```jsx
<div className="relative">
  <Button
    variant="ghost"
    className="h-10 gap-2 px-2"
    onClick={() => {
      setOpenLevelDropdown(!openLevelDropdown);
      setOpenGenreDropdown(false);
      setOpenWordCountDropdown(false);
    }}
  >
    <span className="text-sm font-semibold text-gray-900">
      {filterLevel || 'CEFR'}
    </span>
    <ChevronDown className="h-5 w-5 text-gray-400" />
  </Button>
  {openLevelDropdown && (
    <div className="absolute top-full left-0 mt-1 w-40 rounded-lg border border-gray-200 bg-white shadow-lg z-50">
      <button
        className="block w-full px-4 py-2 text-left text-sm text-gray-700 hover:bg-gray-100 border-b"
        onClick={() => {
          setFilterLevel('');
          setOpenLevelDropdown(false);
          setCurrentPage(1);
        }}
      >
        전체
      </button>
      {LEVEL_SYSTEM.map((item) => (
        <button key={item.cefrLevel} /* ... */>
          {item.cefrLevel}
        </button>
      ))}
    </div>
  )}
</div>
```

**UI 특징**:
- 드롭다운 자동 닫기 (다른 필터 열 때)
- 선택값 버튼에 표시
- "전체" 옵션으로 필터 해제 가능
- 필터 선택 시 자동으로 페이지 1로 리셋

### 4-5. WordCount 관련 함수

#### 단어수 처리 분리
```typescript
// 필터링용 범위 계산
const getWordCountRange = (content: string): string => {
  const wordCount = content.trim().split(/\s+/).length;
  for (const range of WORD_COUNT_RANGES) {
    const [min, max] = range.value.split('~').map(Number);
    if (wordCount >= min && wordCount <= max) {
      return range.value;
    }
  }
  return WORD_COUNT_RANGES[WORD_COUNT_RANGES.length - 1].value;
};

// 표시용 형식 지정
const formatWordCount = (content: string): string => {
  const wordCount = content.trim().split(/\s+/).length;
  return `${wordCount} Words`;
};
```

**중요**:
- `wordCount`: 테이블에 표시되는 **실제 단어수** ("234 Words")
- `wordCountRange`: 필터링에 사용되는 **범위** ("200~300")

### 4-6. libraryTexts 구조 변경

```typescript
const libraryTexts = useMemo(() => {
  return texts.map((text) => ({
    ...text,
    wordCount: formatWordCount(text.content),        // 표시용: "234 Words"
    wordCountRange: getWordCountRange(text.content), // 필터용: "200~300"
    classCount: 0,
    author: getAuthorName(text),
    createdAt: formatDate(text.created_at),
    type: text.text_type === 'A' ? 'ai' as const
      : text.text_type === 'T' ? 'teamwork' as const
        : 'upload' as const,
    text_type: text.text_type || 'A',
    origin_id: text.origin_id,
  }));
}, [texts]);
```

### 개발자 추가 작업
- [ ] 필터 초기화 기능 추가
  - "필터 전체 초기화" 버튼 (모든 필터 한 번에 해제)
- [ ] 필터 조합 저장 기능
  - 자주 사용하는 필터 조합을 프리셋으로 저장
- [ ] 필터 결과 표시
  - "검색 결과: ○개" 텍스트 추가
- [ ] 정렬 기능 구현 (현재는 버튼만 있음)
  - "최신순", "가나다순" 등 정렬 로직 추가
- [ ] 반응형 드롭다운
  - 모바일에서 드롭다운이 화면 밖으로 나가는 이슈 처리

---

**영향 범위**:
- AI 지문 생성 시 더 세밀한 난이도 조절 가능
- 사용자가 원하는 정확한 난이도의 지문 생성 가능
- 지문 분석 결과도 18단계로 표현

### 2. 지문 정보 표시 정책 변경
**기존**: Title, Level, WordCount, Genre, Author, CreatedAt, ClassCount만 표시
**변경**: 위 정보 + 8가지 난이도 분석 지표 추가

**사용자 경험 개선**:
- 사용자가 지문을 클릭했을 때 더 상세한 분석 정보 제공
- 자신의 수준에 맞는 지문 선택에 도움
- 지문 난이도의 구성 요소를 이해 가능

### 3. 모달 UI/UX 정책 변경
- **AI 지문 생성 폼**: 버튼 크기 축소로 모달 높이 감소 → 모바일 친화적
- **지문 상세 정보**: 분석 정보 추가로 모달 높이 증가 → 스크롤 필요할 수 있음

---

## 🧪 테스트 체크리스트

### 기능 테스트
- [ ] AI 지문 생성 모달에서 18개 CEFR 레벨 모두 선택 가능한지 확인
- [ ] 각 레벨 선택 후 지문 생성 API 정상 작동 확인
- [ ] PassageDetailModal에서 8개 지표가 모두 표시되는지 확인
- [ ] 지문 상세 보기 모달의 스크롤 동작 확인

### UI/UX 테스트
- [ ] 밝은 배경에서 텍스트 가시성 확인 (Hello, Oizi, Pick your class)
- [ ] 검색 필드 입력 텍스트 가시성 확인
- [ ] 필터 버튼 텍스트 가시성 확인
- [ ] 분석 지표 배지가 오버플로우되지 않는지 확인
- [ ] 반응형 화면 크기에서 레이아웃 확인

### 호환성 테스트
- [ ] 브라우저 호환성 (Chrome, Safari, Firefox, Edge)
- [ ] 모바일 기기 호환성
- [ ] 섹션 508 접근성 기준 확인

---

## 📝 참고사항

### 관련 파일 경로
```
apps/web/src/
├── app/class/
│   └── page.tsx (메인 클래스 페이지)
├── components/
│   ├── AIGenerateModal.tsx (AI 지문 생성 폼)
│   ├── PassageDetailModal.tsx (지문 상세 정보)
│   ├── PassageInfoBadges.tsx (뱃지 컴포넌트)
│   └── ui/badge.tsx (Badge 기본 컴포넌트)
```

### 향후 고려사항
1. **지문 분석의 정확성**: 현재 하드코딩된 값을 실제 분석 엔진으로 교체
2. **캐싱**: 분석 결과 캐싱으로 성능 최적화
3. **실시간 분석**: 지문 생성 후 즉시 분석 결과 반영
4. **다국어 지원**: 지표명을 다국어로 지원할 경우 i18n 추가 필요

---

**작성일**: 2026-03-11
**수정자**: AI Assistant
**상태**: 완료 (개발자 추가 작업 필요)
