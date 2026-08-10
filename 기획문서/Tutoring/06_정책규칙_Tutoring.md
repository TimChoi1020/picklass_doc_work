# Tutoring 기획서 §6 · 정책·규칙

> 참고문서(`참고문서/picklass_docs/tutoring/`) 기반. 적용 대상은 §4(화면)·§5(기능) ID로 표기.
> 학습 화면의 **데이터드리븐 렌더링(TU-P-004)** 과 **오케스트레이터 규칙(TU-P-005)** 이 Tutoring IA의 핵심입니다.

---

## 0. 정책 카탈로그

| 정책ID | 분류 | 규칙 요약 | 적용 대상 |
|---|---|---|---|
| TU-P-001 | 인증 | JWT·보호경로(middleware 미구현) | 전역 |
| TU-P-002 | 수강권 | 수강 상태 판정(usage_end_date) | TU-F-020, 040 |
| TU-P-003 | 나만의수업 | canUseCustom·기관설정 강제·CEFR 추정 | TU-F-030 |
| TU-P-004 | 렌더링 | 데이터드리븐 3축(passageMode/uiTemplate/questionFlowMode) | TU-F-050 |
| TU-P-005 | 오케스트레이터 | Rule 우선순위 엔진 | TU-F-050 |
| TU-P-006 | 결과데이터 | 레슨완료 저장·약점KPI·중복방지 | TU-F-051 |

---

## 1. 인증 · 수강권

### TU-P-001 · JWT · 보호경로
- JWT 단일 토큰(`JWT_EXPIRES_IN` 기본 1h), `localStorage('picklass_auth_token')`(httpOnly 아님), `Authorization: Bearer`(authFetch)
- `GET /auth/me` → `touchActivity()`(활동시간 갱신 + `refreshToken`/`X-Refresh-Token` 헤더로 갱신). `IDLE_TIMEOUT_SECONDS`(3600s) 초과 → 401. 별도 `/auth/refresh` 없음
- 장시간 레슨: `useSessionKeepalive`(30분마다 `/auth/me`)
- ⚠️ **`middleware.ts` 미구현**: 보호경로 `/modules/*, /report, /my-classes` 리디렉션 없음(모든 페이지 비로그인 접근 가능). 이유: localStorage 토큰이라 SSR middleware가 못 읽음 → 쿠키 이중저장 선결 필요
- 소셜 OAuth(google/kakao/naver) 전부 미구현(`alert("준비 중")`)

### TU-P-002 · 수강 상태 판정
- 계약 기간 기반(DB status 불변, 조회 시점 계산 — 백오피스 `effectiveStatus()` 정합):
  - 수강중 = `usage_end_date IS NULL OR >= 오늘`
  - 수강종료 = `usage_end_date < 오늘`
- access_codes status: `active`=수강중, `inactive`=미사용(구 관례 `inactive+user_id="등록완료"` 폐기, 마이그레이션 전환)
- **course_type 서비스 필터**: tutoring=`tutoring,integrated`(+NULL) / speaking=`speaking,integrated`. NULL은 `integrated`로 backfill
- 등록: `registerAccessCode()` → `registration_expiry` 만료 체크, 이미사용 판별 `status_code==='active'`, UPDATE `status='active', activated_at=NOW(), usage_start/end=COALESCE(기존, NOW()[+usage_period_days])`

---

## 2. 나만의 수업

### TU-P-003 · canUseCustom · 기관설정 강제 · CEFR 추정
- **canUseCustom** = `activeCourses.length > 0`(수강중 일반 과정 1개↑). false면 잠금 배너 + 카드 비활성
- **기관설정 강제**(공유 `institution_settings`):
  - `institutionId==null`(B2C) → 제약 없음 / 소속인데 설정행 없음 → **permissive**(무제한·허용) / 설정행 있음 → 값 적용
  - `free_learning_enabled=false` → 생성 차단(서버 403 "소속 기관에서 나만의 수업을 허용하지 않습니다.", 프론트 error step)
  - `max_total_class_minutes` → 초과 프리셋 disabled + 서버 400. 기본 30이 상한 초과 시 허용 최대로 자동 하향, 전 프리셋 비활성(max<15) 시 CTA 비활성
  - `max_frt_module_minutes/allowed_mic_modes/mic_mode_user_change/default_completion_*`는 나만의수업 UI에 입력 없어 범위 밖
  - JWT `institutionId` 캐싱(로그인 1회 리졸브, carry-forward). 구 토큰(미포함) → null → permissive
- **CEFR 추정**(`GET /lessons/me/level-estimate`, JWT 401 강제):
  - 데이터: `lesson_results` JOIN `texts.level`, 최근 30일(`RECENT_DAYS=30`), `SAMPLE_LIMIT=10`
  - 가중: `scoreAdjust=(score-50)/50×1.5`, `adjustedLevel=clamp(0,18,levelIdx+scoreAdjust)`, `recencyWeight=exp(-daysSince/14)`, `durationWeight=clamp(0.3,1.0, 0.3+0.7×duration/300)`, `weight=recency×duration`
  - `inferredIndex=round(가중평균)`, confidence=`clamp(0,1, sample×0.5+std×0.4+recency×0.1)`, n<3이면 conf≤0.4
  - Cold-start: 0건→B1(idx=8)/conf=0 / 1~2건→conf≤0.4 / ≥3건→정상
- 학습시간 프리셋 15/20/25/30(기본 30), CEFR 19단계(PreA1=0 ~ C2+=18)
- create-custom 에러: 400(KPI 0개/duration 외/지문레벨 분석실패)·401(레벨추정 미인증)·502(analyzer 미응답)·503(analyzer 5xx). **analyzer 장애 시 Gemini 폴백 없음**

---

## 3. 데이터드리븐 렌더링

### TU-P-004 · DB 3축 렌더링
학습 화면은 `ai_modules`의 3필드로 동작이 결정됩니다(코드 분기 없음, DB 등록만으로 렌더링 구성).

**passageMode 5종**
| 값 | 동작 | 예 |
|---|---|---|
| `full` | 전체 공개 | 대부분 |
| `hidden` | DOM 미렌더(flex 미점유) | RRD, SWR |
| `preview` | 도입 2문장→답변 후 전체(`revealPassage`) | PRD |
| `timed-blur` | 타이머 읽기→"읽기완료"→blur 복원 | SCN |
| `timed-select` | 타이머 읽기→문장 선택 즉시 활성, 첫 제출 시 자동 done | SKM |

(구값 `preview-then-reveal→preview`, `timed-then-blur→timed-blur`)

**uiTemplate 4종**: `standard`(QuestionsPanel) / `voice`(VoiceQuestionPanel; WSD/RRD/SHD/WSP) / `embedded`(패널 없음, CLR) / `hidden`(확장용)

**questionFlowMode 3종**(카드 렌더): `deck`(activeQuestionId 1개만) / `sequential`(전체 동시, 자유 답변) / `locked-steps`(전체 동시, 이전 미완료 🔒; PWR)

**questionCount 3종**(진행도/버튼): `single`(진행도·버튼 숨김) / `multi`(N/M + 다음 버튼) / `content-driven`(지문 문장 수 기반; CLR)
> 역할 분리: flowMode=렌더, count=진행도/버튼

**모듈 전체 설정표**
| 모듈 | count | passageMode | uiTemplate | flowMode |
|---|---|---|---|---|
| WRD | multi | full | standard | deck |
| WSD | multi | full | voice | deck |
| GMN | multi | full | standard | deck |
| WWB | multi | full | standard | sequential |
| PRD | single | preview | standard | sequential |
| SCN | single | timed-blur | standard | sequential |
| SKM | single | timed-select | standard | sequential |
| CLR | content-driven | full | embedded | — |
| SUM | single | full | standard | sequential |
| QAR | multi | full | standard | deck |
| RRD | multi | hidden | voice | deck |
| SWR | multi | hidden | standard | deck |
| SHD | multi | full | voice | sequential |
| WSP | multi | full | voice | sequential |
| PWR | multi | full | standard | locked-steps |

- 채점 `scoringMode` 4종: `exact/holistic/pronunciation/writing-eval`. answerType: multiple-choice/essay/short-text/sentence-write/audio-record + keyword-chips(SCN)/sentence-select(SKM)/sentence-explain(CLR)
- 정규화(2026-07-02): 정답비교 `.trim().toLowerCase()`. `multiple-choice`의 `answer`는 `"③"` 단순문자열(구 JSON → 정규화, 부가데이터 `explanation` JSONB)

---

## 4. AI 오케스트레이터

### TU-P-005 · Rule 우선순위 엔진
첫 non-null 반환 규칙 실행. `useModuleOrchestrator`.

| 순위 | Rule | 조건 → 액션 |
|---|---|---|
| 0 | ruleGreet | `greetingShown===false` → greet(purpose 단일) |
| 1 | ruleIdleCheck | idle ≥ 120초 → checkEngagement |
| 2 | ruleDisengaged | idle ≥ 180초 & low → signalReplan |
| 3 | ruleRepeatWrongAnswer | 오답 1회(exact,maxAttempts≥2) → giveHint(direct) / 2회↑ → revealAnswer. audio/sentence-write 제외 |
| 3b | ruleOfferHint | idle ≥ 20초 & 미답변/오답 → offerHint 버튼 |
| 4a | ruleHolisticFeedback | holistic 제출 직후 → giveFeedback(holistic) LLM |
| SHD-C | rulePronunciationFeedback | audio-record 제출 → giveFeedback(pronunciation, Azure) |
| SWR-A | ruleWritingFeedback | writing-eval → giveFeedback(writing) |
| 4c | ruleWrongAnswerFeedback | exact 첫 오답 → giveFeedback(correctness) |
| 6 | ruleCompleteAfterCelebrate | celebrate+전체 답변 → completeModule |
| SHD-A/B | rulePlayModelAudio / ruleStartRecordingAfterAudio | 발음 모듈 |
| 5 | ruleCorrectAnswer | 정답 판정, 마지막 문항 → celebrate |
| 7 | ruleInitialEntry | greetingShown+무상호작용 → passageMode별 showPassage |
| 7b | ruleClrIdle | sentence-explain → 항상 null |
| 8 | rulePresentNextQuestion | 미답변 문항 → askQuestion |

- **힌트 3레벨**: Level 0 button(학생 클릭) / Level 1 direct(오답 1회 자동) / Level 2 answer(오답 2회 revealAnswer). `questionMaxAttempts=Math.min(값,2)`(실질 최대 2회). SHD(발음): score<70 1차 button, 2차+ direct
- **타임아웃**: idle 120초→참여도확인, 180초+저참여→리플래닝, offerHint 20초
- **문항 표시 게이트**: 모든 모드에서 `activeQuestionId !== null` 전까지 미표시("Pickle AI가 문항을 준비 중이에요...")
- ⚠️ `retryThreshold`(모듈 재도전 %) **미구현**(점수 무관 항상 완료). "오답률 50%" 판정은 planning 성공지표(모듈실패 score<50% ≤20%) 근거

---

## 5. 결과 데이터 · 이수조건

### TU-P-006 · 레슨완료 저장·약점KPI·중복방지
- **저장 테이블**: `module_histories`(모듈완료마다 1행: answers·chat_messages·score 0–100·kpis JSONB·attempt_counts·hint_used_ids·engagement_level·time_spent_sec) / `lesson_results`(레슨당 1행 upsert: average_score·module_results·total_duration·is_complete·estimated_level·weak_kpi_codes·summary/strengths/improvements/next_steps) / `module_kpi_results`(1 KPI=1행)
- **저장 흐름**: 모듈완료 → `POST /lessons/:id/modules/:code/history` → `saveModuleHistory` `$transaction`(module_histories + module_kpi_results + lesson_results upsert). 커밋 후(외부) LLM KPI 평가(pending) + description 생성 + kpis UPDATE
- **집계**: average_score=전체 이력 평균, total_duration=time_spent_sec 합, `is_complete = moduleResults.length >= skill_modules.length`
- **약점 KPI**(computeWeakKpiCodes): `WEAK_THRESHOLD=60`. 일반 KPI 평균<60 약점 / 역방향(`GRAMMAR_ERROR,WRITING_ERROR,RECOG_SPEED,RESPONSE_LATENCY`)은 >60 약점
- **재학습 중복 방지**(방안 C): `lesson_results.is_complete=true` OR 동일 module_histories 존재 → 트랜잭션 진입 전 종료, 기존 kpis 반환(`skipped:true`)
- studentId undefined → `UnauthorizedException`(폴백 UUID 제거 — 다른 계정 저장 버그 수정)
- 외부 스피킹 토큰: 레슨플랜 조회 시 SNR/FRT 감지 → `external_access_tokens.create`(expiresAt 기본 60분)

---

## 6. 오픈 이슈 (정책 미확정 / 미구현)

| # | 이슈 | 영향 |
|---|---|---|
| O-1 | `middleware.ts` 보호경로 미구현(비로그인 접근 가능) | TU-P-001 |
| O-2 | `retryThreshold` 모듈 재도전 판정 미구현(항상 완료) | TU-P-005 |
| O-3 | analyzer 장애 시 폴백 없음(명확한 에러만) | TU-P-003 |
| O-4 | 튜터링-백오피스 `ai_modules` DB 이중화(수동 Supabase PATCH 동기화) | TU-P-004 |
| O-5 | CLR `options`→`explanation` JSONB 마이그레이션 미완(양쪽 폴백) | TU-P-004 |
| O-6 | `/report` 상세 미구현, `window.location.reload()` → `router.refresh()` 교체 필요 | §4, TU-F-040 |
