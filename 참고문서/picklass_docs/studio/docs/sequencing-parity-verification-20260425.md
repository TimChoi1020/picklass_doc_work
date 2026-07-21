# lesson-setup ↔ course 시퀀싱 동등성 검증 플랜

**작성일**: 2026-04-25  
**목적**: 두 페이지(`/class/lesson-setup/[passageId]`, `/course` Step 2)의 **모듈 선정 + 시퀀싱 로직이 완전히 동일한 결과를 내는지** 반복 검증할 수 있는 절차 문서.  
**용도**: 향후 시퀀싱 관련 코드 변경 시 회귀 테스트로 사용. 본 문서의 절차를 그대로 따라가며 결과를 기록하면 차이를 즉시 발견할 수 있음.

---

## 1. 검증 대상 — "동일해야 하는" 항목

### 1-1. 입력

| 항목 | lesson-setup | course Step 2 회차 (기존 레슨 모드) | course Step 2 회차 (AI 모드) |
|------|-------------|-----------------------------------|----------------------------|
| passage_id | URL param | `selectedLessonPerTopic[idx].id` | (없음) |
| passage_level | DB(texts.level) | DB(texts.level) | Step 1 `wizardLevel` |
| KPI 코드 | `selectedKpis` | `selectedObjectives` | `selectedObjectives` |
| duration | `durationMinutes` | `topicsDuration[idx]` | `topicsDuration[idx]` |

### 1-2. 출력 (analyzer 응답)

| 항목 | 둘 다 같아야 |
|------|------------|
| `sequence` (모듈 코드 배열) | ✓ |
| `sequence` 순서 | ✓ |
| `summary`, `meta` | ✓ |

### 1-3. UI 표시

| 항목 | lesson-setup | course |
|------|-------------|--------|
| 카테고리 그룹 | skill별 (`vocabulary`, `reading`, `speaking`, `writing`) | 동일 |
| 칩 스타일 | 코드 + 이름, 선택 시 녹색, 우상단 번호 배지 | `ModuleChipList` (lesson-setup 동일) |
| 비활성 모듈 (sequence 외) | 회색 배경 | 동일 |
| 시퀀싱 중 인디케이터 | spinner + "시퀀싱 분석 중..." | 동일 (`LessonModuleSection`) |
| 시퀀싱 실패 오버레이 | 에러 메시지 + 다시 시도 | 동일 |

---

## 2. 사전 준비

### 2-1. 검증용 지문 (고정)

검증을 반복 가능하게 하려면 고정된 지문 1개를 사용. 매 검증마다 동일한 지문 선택.

**기준 지문 (변경 시 본 문서도 갱신):**

| 필드 | 값 |
|------|-----|
| ID | `___` (검증 시 채움) |
| 제목 | `___` |
| level | `___` (예: A2) |
| category | `___` (예: 설명문) |
| word_count | `___` |

선택 기준:
- DB에 실제 존재
- `level` 컬럼 NOT NULL (없으면 lesson-setup이 거부)
- 본인(현재 로그인 사용자) 소유 텍스트 (본 정책상 본인 외 조회 불가)

### 2-2. 환경

| 항목 | 값 |
|------|-----|
| analyzer 서버 | 정상 동작 (Supabase 연결 OK) |
| dev 서버 | `pnpm dev` 실행 중 |
| `.next/` 캐시 | 정리됨 (필요 시 `rm -rf apps/web/.next`) |
| 브라우저 | Chrome DevTools 열기 (Network + Console 탭) |
| 로그인 | `sniper4457@naver.com` 등 teacher 계정 |

### 2-3. 검증용 KPI / duration 조합

여러 시나리오를 반복 가능하게 고정.

| 시나리오 | KPI | duration |
|---------|-----|----------|
| A | 1개 — 가장 첫 번째 KPI 선택 | 15분 |
| B | 1개 — 가장 첫 번째 KPI 선택 | 30분 |
| C | 2개 — 첫 2개 KPI | 20분 |
| D | 모든 KPI | 30분 |

---

## 3. 검증 절차

### 3-1. lesson-setup에서 결과 캡처 (시나리오 A 기준)

1. 브라우저: `/class/lesson-setup/{기준 지문 ID}` 진입
2. KPI: 시나리오 A의 KPI 선택
3. duration: 15분 선택
4. DevTools Network 탭에서 `POST /lesson-plan` 요청 확인
   - **Request Body 캡처**: `passage_id`, `selected_kpi_codes`, `duration_min`
   - **Response Body 캡처**: `sequence` 배열의 `module_code` 순서
5. 화면의 모듈 칩 순서 캡처: `[1] AAA, [2] BBB, [3] CCC, ...`
6. 결과 기록 (§4 템플릿)

### 3-2. course에서 동일 결과 캡처 (시나리오 A 기준)

#### 케이스 1: 기존 레슨 모드 (passage_id 사용 — lesson-setup과 동일 경로)

1. 브라우저: `/course` → "과정 생성" 모달
2. Step 1 입력:
   - 과정명: `검증A`
   - 과정 목표: 시나리오 A의 KPI
   - 과정주제: `검증`
   - 레슨 회차: 1
   - CEFR 레벨: 기준 지문과 동일 (예: A2)
   - 단어수: 200
   - 장르: 기준 지문과 동일
   - 회차별 수업 시간: 15분
3. "다음" 클릭 → Step 2
4. 회차 1: "기존 레슨" 토글 → 기준 지문 선택
5. 자동 시퀀싱 호출 확인 (Network 탭)
   - **Request Body 캡처**: `passage_id`, `selected_kpi_codes`, `duration_min`
   - **Response Body 캡처**: `sequence`
6. 회차의 모듈 칩 순서 캡처
7. 결과 기록 (§4 템플릿)

#### 케이스 2: AI 모드 (passage_level 사용)

1. 위 Step 1 동일 입력 (단, 레벨은 기준 지문과 동일하게)
2. 회차 1: "AI 생성" 모드 (디폴트)
3. 자동 시퀀싱 호출 확인
   - **Request Body 캡처**: `passage_level` (passage_id 없음), `selected_kpi_codes`, `duration_min`
   - **Response Body 캡처**: `sequence`
4. 회차의 모듈 칩 순서 캡처
5. 결과 기록

> **참고**: passage_level만 보낼 경우 analyzer가 passage_text 없이 분석하므로, **passage_id 케이스와 sequence가 약간 다를 수 있음** (정상). 이 차이가 정책상 허용 범위인지 확인 필요.

---

## 4. 결과 기록 템플릿

### 시나리오 A (KPI 1개, 15분)

| 항목 | lesson-setup | course (passage_id) | course (passage_level) |
|------|-------------|---------------------|----------------------|
| Request: passage_id | ___ | ___ | (없음) |
| Request: passage_level | (없음) | (없음) | ___ |
| Request: selected_kpi_codes | ___ | ___ | ___ |
| Request: duration_min | 15 | 15 | 15 |
| Response sequence (코드 순서) | ___ | ___ | ___ |
| 화면 칩 번호 순서 | ___ | ___ | ___ |
| **passage_id 끼리 동일?** | — | □ | — |
| **passage_level은 차이 허용 범위?** | — | — | □ |

### 시나리오 B/C/D — 동일 형식으로 반복

---

## 5. 비교 체크리스트

### 5-1. 입력 일치

- [ ] lesson-setup과 course(passage_id)가 보낸 `selected_kpi_codes` 배열 = 동일 (정렬 순서 무관, set 일치)
- [ ] `duration_min` 동일
- [ ] `passage_id` 동일

### 5-2. 출력 일치 (passage_id 케이스끼리)

- [ ] `sequence`의 `module_code` 배열 순서 일치
- [ ] `summary` 필드 값 일치
- [ ] `meta` 필드 값 일치

### 5-3. UI 표시 일치

- [ ] 카테고리 그룹 동일 (vocabulary/reading/speaking/writing)
- [ ] 칩 스타일 (선택/추천/비활성) 색상 동일
- [ ] 번호 배지 모듈 코드 일치
- [ ] 번호 배지 순서 일치 (1,2,3...)

### 5-4. 동작 일치

- [ ] duration 변경 시 둘 다 재호출
- [ ] KPI 변경 시 둘 다 재호출
- [ ] 디바운스(300ms) 동일
- [ ] 시퀀싱 실패 시 둘 다 오버레이 + 다시 시도 버튼

### 5-5. lesson-setup 단독 검증 (회귀)

- [ ] passage_id 흐름 정상 동작 (기존 코드 회귀 없음)
- [ ] passage 레벨 누락 시 400 에러 메시지 정상

### 5-6. course 단독 검증

- [ ] 다중 회차에서 모든 회차에 sequence 적용 (race 없음)
- [ ] 회차별 duration 변경 시 해당 회차만 재호출
- [ ] AI 회차 (passage_level) → passage_id 회차로 전환 시 사용자 입력 보존

---

## 6. 차이 발생 시 진단 가이드

### Case A: Request Body가 다름

| 차이 | 원인 추정 | 확인 위치 |
|------|----------|----------|
| `selected_kpi_codes` 다름 | KPI 매핑 로직 차이 | lesson-setup `handleKpiClick` vs course Step 1 KPI 선택 로직 |
| `duration_min` 다름 | duration 전달 경로 차이 | lesson-setup `durationMinutes` vs course `topicsDuration[idx]` / `wizardDuration` |
| `passage_id` vs `passage_level` 분기 | 의도된 분기 (course AI 모드만 level) | `LessonModuleSection` props 결정 로직 |

### Case B: Response sequence 다름 (passage_id끼리)

- analyzer 응답이 비결정적인지 확인 (같은 입력 = 같은 출력이어야)
- backend가 두 호출에서 다른 데이터를 보냈는지 (예: 캐시 stale)
- React Query 캐시 충돌 확인 (queryKey가 정확히 같은지)

### Case C: UI 표시 다름 (sequence 같은데 화면이 다름)

- `ModuleChipList`의 `sequenceOrder` / `activeCodes` 전달 누락
- `selectedCodes`에 sequence 외 모듈이 섞임 → 끝번호로 빠짐 (번호 어긋남)
- CSS 클래스 차이 (lesson-setup 인라인 vs ModuleChipList)

### Case D: 회차 일부만 sequence 반영 (course)

- **stale closure / setState race** — 모든 회차 setter가 functional updater인지 확인
- React Query의 `placeholderData=keepPreviousData` 작동 여부

---

## 7. 자동화 가능한 부분 (향후)

수동 검증을 자동화로 전환하는 방안:

1. **Network mock 테스트**: lesson-setup과 course의 Request body를 캡처해 비교하는 e2e 테스트 (Playwright)
2. **Snapshot 테스트**: ModuleChipList 렌더 결과 스냅샷 비교
3. **백엔드 단위 테스트**: 같은 입력에 같은 analyzerReq 빌드 검증

본 문서의 §3 절차를 Playwright로 옮기면 매 PR에서 자동 검증 가능.

---

## 8. 검증 이력

| 일자 | 시나리오 | 결과 | 차이 / 메모 | 작업자 |
|------|---------|------|------------|--------|
| 2026-04-25 | duration 15/20/25/30 (KPI 5개 — `어휘 인지 및 의미`) | ✓ | passage_id(145) vs passage_level(A2-) 양 경로 모두 `WSD,WRD,IMG,FRT` 동일 시퀀스. 기준 지문: id=145 "My Online Art Teacher" A2- 에세이 | claude |

---

## 9. 부록 — 코드 위치 (변경 시 동시 검토)

| 파일 | 역할 |
|------|------|
| `apps/web/src/app/class/lesson-setup/[passageId]/page.tsx` | lesson-setup 진입 + 시퀀싱 로직 |
| `apps/web/src/app/course/page.tsx` | course 마법사 (Step 1/2 + 회차) |
| `apps/web/src/components/LessonModuleSection.tsx` | course의 회차별 모듈 영역 (시퀀싱 호출 캡슐화) |
| `apps/web/src/components/ModuleChipList.tsx` | 칩 UI (course에서만 사용) |
| `apps/web/src/hooks/use-lesson-plan.ts` | `useLessonPlanSequence` 훅 |
| `apps/web/src/lib/api.ts` | `lessonPlanApi.generate` |
| `apps/api/src/lesson-plan/lesson-plan.service.ts` | NestJS → analyzer 호출 |
| `apps/api/src/lesson-plan/dto/lesson-plan.dto.ts` | DTO (passage_id / passage_level) |

코드 변경 시 **§5 체크리스트로 회귀 검증** 후 §8에 결과 기록.

---

## 10. 핵심 한 줄

> **두 페이지의 시퀀싱 결과가 같음을 보장하려면, "같은 입력"인지 매번 직접 확인해야 한다. 같은 query는 같은 응답이지만, 같은 입력을 보내고 있는지는 호출 시점마다 다를 수 있다. Network 탭의 Request Body가 정확히 일치하는지가 검증의 출발점.**
