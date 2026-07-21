# Tutoring 기획서 §7 · 예외·상태

> 참고문서(`참고문서/picklass_docs/tutoring/`) 기반. 화면 §4(TU-S-###)·기능 §5(TU-F-###)·정책 §6(TU-P-###) 참조.
> 구조: ①공통 상태 표준 → ②화면별 매트릭스 → ③예외 카탈로그.

---

## 0. 공통 화면 상태 표준

| 상태 | 표준 UI | 행동 |
|---|---|---|
| Loading | "학습을 불러오는 중..." / 문항 준비 문구 | 조작 대기 |
| Empty | 섹션별 빈 상태 | 유도 |
| Error | 토스트/에러 step | 재시도 |
| 문항 미준비 | "Pickle AI가 문항을 준비 중이에요..." | activeQuestionId=null |
| 스켈레톤(완료) | summaryFeedback/pending KPI 스켈레톤 유지 | 숨기지 않음 |
| 세션만료(401) | 로그아웃 → 로그인 | 유휴 3600s |

---

## 1. 화면별 상태 매트릭스

| 화면ID | Loading | Empty | 특이 상태 |
|---|---|---|---|
| TU-S-010 나의수업 | 스피너 | 섹션별 빈 상태 | custom 0개면 섹션 숨김 |
| TU-S-020 나만의수업 | generating step | KPI 로드 실패 시 빈목록+에러 | free 차단 error step, 시간 초과 disabled |
| TU-S-040 모듈학습 | "학습을 불러오는 중..." | 문항 미준비 문구 | timed-blur 3상태(before/reading/done) |
| TU-S-041 레슨완료 | 스켈레톤 | KPI 없는 모듈 아코디언 빈상태 | summaryFeedback=null 스켈레톤 유지, estimatedLevel=null 뱃지 숨김 |
| TU-S-050 리포트 | — | — | 미구현 |

---

## 2. 예외 시나리오 카탈로그

### TU-E-001 · 입력검증 / 인증 실패
- **발생 화면/기능**: TU-S-030/060 / TU-F-040, 001
- **문구**: "액세스코드를 입력해주세요"(빈 값), "액세스코드 등록 기간이 만료되었습니다.", "유효하지 않은..."
- **처리**: 인라인 error, 로딩 중 닫기/취소 disabled

### TU-E-002 · 나만의수업 제약 위반
- **발생 화면/기능**: TU-S-020 / TU-F-030
- **조건·처리**:
  - `free_learning_enabled=false` → 403 "소속 기관에서 나만의 수업을 허용하지 않습니다." → error step 안내화면
  - 학습시간 > `max_total_class_minutes` → 초과 프리셋 disabled + 서버 400 "학습 시간은 최대 N분까지 가능합니다."
  - KPI 0개/duration 외 → 400, 레벨추정 미인증 → 401
  - analyzer 미응답 → 502 / analyzer 5xx → 503 (**Gemini 폴백 없음**)
- **정책**: TU-P-003

### TU-E-003 · 마이크 권한 / 발음 평가
- **발생 화면/기능**: TU-S-040(voice 모듈) / TU-F-050
- **처리**: Azure Speech SDK 점수(0–100) 직접, score<70 시 힌트 에스컬레이션(button→direct). AudioFile은 model/student 구분 S3 저장
- ⚠️ 마이크 권한(getUserMedia) 거부 예외 UI는 **문서상 미기재**(설계 필요)
- **VoiceQuestionPanel "준비 중" 버그**: 문항 완료 후 activeRecordingId null 리셋 시 오버레이 덮임 → `doneCount>0`(deck)/`doneCount===0`(sequential)로 미시작 vs 완료대기 구분

### TU-E-004 · 네트워크 / 세션(401) / LLM 장애
- **발생**: TU-S-040, 041 / TU-F-050, 051
- **처리**:
  - 401: 유휴 3600s 초과 `/auth/me` 401 → 로그아웃. saveModuleHistory studentId 없으면 401
  - 레슨완료 AI 피드백: 마운트 즉시 fetch + **5초 후 재조회**로 갱신(Gemini fire-and-forget). 네트워크 실패 시 React state 폴백
  - LLM 장애: API 타임아웃(>5초) → Rule 기반 전환, Rate limit → 지수 백오프(3회, 1s→2s→4s), 잘못된 Tool Call → presentNextQuestion 폴백
  - `pendingOrchestrateRef`: LLM 대기 중 답안 제출 → pending 마킹 → 완료 후 재실행

### TU-E-005 · 학습 화면 상태 전이
- **발생 화면/기능**: TU-S-040 / TU-F-050 (`usePassageMode`)
- **timed-blur(SCN)** 3상태: `before`(blur + Floating Timer "지문읽기") → `reading`(지문 표시 + inline 타이머 "읽기 중" + "읽기완료") → `done`(blur 복원, 첫 제출 전). `blurActive = before || (done && !firstAnswerSubmitted)`
- **timed-select(SKM)**: reading에서 문장 선택 즉시 활성, done 전환=첫 제출 자동(완료 버튼 없음), blur 없음(`blurActive=before`만)
- **full**: `passageVisibleOverride=true`(preview 플래시 방지) / **preview**: undefined(orchestrator revealPassage) / **hidden**: override=false, readingPhase='done'

### TU-E-006 · 빈 상태 / 데이터 없음
- 나의수업: 섹션별 빈 상태. 레슨완료: summaryFeedback=null 스켈레톤 유지, pending KPI 스켈레톤 바, KPI 없는 모듈 빈 아코디언, 다음 레슨 없으면 홈 버튼만
- KPI 로드 실패(옵션폼): 빈 목록 + 에러 메시지, institutions/settings 실패 → permissive 폴백

---

## 3. 문항 입력 타입별 처리

| 타입 | 처리 |
|---|---|
| `short-text` | `<input>`(한 줄, Enter 자동 제출) |
| `essay`/`sentence-write` | `<textarea>` |
| `keyword-chips`(SCN) | chip 추가/삭제/중복방지, Enter=추가(제출 아님), "제출하기"=일괄 제출, 후 읽기전용 pill |
| `sentence-select`(SKM) | ContentPanel 인라인 `<span>` 클릭, 칩 렌더 |
| `sentence-explain`(CLR) | SentenceExplainPopover(인라인). 완료=`passage.content` 파싱 문장 80% 탐색 → "완료" 버튼(자동완료 아님), "지문의 80% 이상을 탐색했어요." (`questions.length` 사용 금지 — 즉시완료 버그) |

- ✓/✗ 표시: `scoringMode==='exact'`일 때만
- "다음문항으로 →"=`isAnswered && !isLastQuestion`, "이해했어요 👍"=`awaitingFeedbackConfirm && isLastQuestion`

---

## 4. 에러코드 ↔ 사용자 메시지

| 상황 | HTTP | 메시지 |
|---|---|---|
| 액세스코드 빈 값 | — | "액세스코드를 입력해주세요" |
| 등록기간 만료 | — | "액세스코드 등록 기간이 만료되었습니다." |
| 나만의수업 미허용 | 403 | "소속 기관에서 나만의 수업을 허용하지 않습니다." |
| 학습시간 초과 | 400 | "학습 시간은 최대 N분까지 가능합니다." |
| analyzer 미응답/오류 | 502/503 | (명확한 에러, 폴백 없음) |
| 유휴 초과 | 401 | 로그아웃 |

---

## 5. 미구현 / 준비중 (Known Gaps)

| 영역 | 내용 | 정책/기능 |
|---|---|---|
| 인증 | `middleware.ts`(보호경로), 소셜 OAuth(google/kakao/naver) | TU-P-001 |
| 리포트 | `/report` 상세 미구현 | TU-S-050 |
| 액세스코드 | 성공 메시지 미표시(`setSuccess` 미호출), `window.location.reload()` → `router.refresh()` 교체 | TU-F-040 |
| 모듈 | 계획 모듈 일부(QAR/VCB/WSD/SWR/PWR 등), PWR sentence-write holistic 예외 미구현 | TU-P-004 |
| 학습 로직 | `retryThreshold` 재도전 판정 미구현, `scheduleCheck`(현 20초 하드코딩) | TU-P-005 |
| 데이터 | CLR `options`→`explanation` 마이그레이션 미완, WSD 힌트 데이터 미로드 미해결 | TU-P-004 |
| 파이프라인 | `student_level_snapshots`/speaking 미러링 미구현, ai_modules DB 이중화 수동 동기화 | TU-P-006 |
| 마이크 | getUserMedia 권한 거부 예외 UI 미기재 | TU-E-003 |
