# Speaking 기획서 §6 · 정책·규칙

> 참고문서(`참고문서/picklass_docs/speaking/`) 기반. 적용 대상은 §4(화면)·§5(기능) ID로 표기.

---

## 0. 정책 카탈로그

| 정책ID | 분류 | 규칙 요약 | 적용 대상 |
|---|---|---|---|
| SP-P-001 | 게이팅 | 레벨테스트 진입 게이팅(hard_block/soft_prompt) | SP-F-071 |
| SP-P-002 | 레벨 | 18단계 레벨 시스템(CEFR 6밴드×3) | SP-F-012, 050, 060 |
| SP-P-003 | 레벨 | P-ALT 루브릭·PT/AT 체인 | SP-F-070 |
| SP-P-004 | 학습지표 | 발화량·이수율·수료조건 | SP-F-011, 030, 040 |
| SP-P-005 | 인증 | 통합모듈 ONETIME vs 독립앱, JWT | SP-F-061 |
| SP-P-006 | 수강권 | 수강권 2모델(access_codes / 파고다) | SP-F-020, 051 |
| SP-P-007 | 학습설정 | 마이크 모드 | SP-F-022, 052, 060 |
| SP-P-008 | 알림 | 알림·방해금지(DND) | SP-F-052 |
| SP-P-009 | 마스터리 | 문장 마스터리 엔진(레벨·망각곡선) | SP-F-031~032 |

---

## 1. 레벨테스트 게이팅 · 레벨 시스템

### SP-P-001 · 게이팅 규칙
- `GET /levels/status`(JwtAuthGuard) → `{requiresTest, testType:'PT'|'AT'|null, reason:'new_user'|'renewal'|'after_lesson', enforcement:'hard_block'|'soft_prompt', hasNickname, hasLevel}`
- **판정 우선순위**: ① 신규(완료진단 0건 & 레벨없음) → PT / **hard_block** / new_user  ② 갱신 후 PT 미완료 → PT / soft_prompt / renewal(Phase2)  ③ 마지막수업완료 > 마지막진단 → AT / soft_prompt / after_lesson
- **hard_block은 신규 PT에만**: `LessonAccessGuard` → 수업 시작 엔드포인트 `403 {code:'LEVEL_TEST_REQUIRED', testType:'PT'}` (→ §7 SP-E-005)
- **스누즈**: hard_block=매 홈 진입 노출 / soft_prompt=`localStorage.levelPromptSnoozeUntil = now+24h`(서버 무호출)
- ⚠️ Phase1: `picklass_level` 부재를 new_user 프록시로 사용(`alt_sessions.user_id` integer vs `users.id` UUID 타입 불일치 — 미결)

### SP-P-002 · 18단계 레벨 시스템
- **18단계 = CEFR 6밴드 × 3세분**: `A1-,A1,A1+,...,C2-,C2,C2+`(picklass_rank 1~18). 단일 권위 = `words_18levels_6000.picklass_rank`
- 공인시험 환산: L1-3=A1(OPIc NL) / L7-9=B1(IL-IH) / L13-15=C1(AL) / L16-18=C2(AH)
- 레벨 저장: 스냅샷 `user_learning_profiles.picklass_level`(SMALLINT 1-18) + 원장 `alt_level_history`(append-only), `LevelService.recordLevelChange`(트랜잭션). `cefr = labelForRank(level)`(18라벨)
- **5대 KPI 색상(불변)**: 발음 `#185FA5` · 유창성 `#534AB7` · 문법 `#BA7517` · 화용 `#D4537E` · 발화량 `#0F6E56`

### SP-P-003 · P-ALT 루브릭 · PT/AT 체인
- **파트**: P1 Vocab(4지선다 CAT) · P2 문장낭독(3~7문장 CAT) · P3 일상대화(6~10턴) · P4 주제발표(조건부, 30초 준비+40~90초 발화)
- **P4 진입 조건**: P3 6축 평균 L13↑(심화)/L7-L12(표준)/L6↓(생략)/화용성 L4↓(생략)
- **6대 루브릭**: 어휘·유창성·발음·문법·표현·화용성. 최종레벨 = `(어휘×1.5+유창성×1.5+발음×1.0+문법×1.5+표현×1.0+화용성×1.0)/7.5`
- **PT/AT 체인(v8.1)**: PT 1회 + AT 체인(각 AT는 `previous_session_id`와 비교, **하한 보호** = 직전 진단 이하 하락 금지). P3·P4 LLM 단일 호출 4축 동시 채점

---

## 2. 학습 지표

### SP-P-004 · 발화량·이수율·수료조건
- **발화량 2미터**(주간 리듬 대시보드) `GET /challenge/utterance`: 주간(월~일, 200)/월간(1일~말일, 800). 탭모드 ×0.6 가중. 소스 = `utterance_logs`. ⚠️ 구 4미터 중 **일일·페르소나 미터는 폐기**(B-3/B-4 확정) — 일일 누적은 주간 미터에 실시간 반영(별도 미터 아님), 페르소나 미터는 v1.3 미채택. (근거: 본편 §14.4.6 B-3/B-4)
- **수료조건 유형**: `utterance_sentences`/`utterance_words`/`course_completion_pct`/`lesson_count`/`study_days`(복합 AND). `GET /config/program`(backoffice `institution_settings` raw SQL READ)
- **이수율** = `lesson_results.completed_at IS NOT NULL` 비율. `default_completion_minutes`(수업시간) 2026-06-24 연동. `dailyGoalMinutes` 기본 20
- ⚠️ **기관 평균 비교 KPI API 부재** → 홈 레이더·리포트 모두 Mock/제거(후속)

---

## 3. 인증 · 수강권

### SP-P-005 · 인증 (통합 vs 독립)
- **통합 모듈(튜터링+스피킹)**: ONETIME TOKEN(쿼리스트링, 재진입 불가) → `POST /api/auth/get-auth {token}` → `{userId,name,accessCode,level,content,dialogLength,voiceMode}` → 세션 시작 시 `POST /api/auth/revoke-token`(토큰 소멸). 저장 = ModuleHistory
- **독립 앱**: 자체 로그인·인증코드·세션히스토리. ONETIME TOKEN/GET_AUTH 미사용. 저장 = SpeakingSession 단독
- **JWT**: `{userId,email,roleCode,institutionId,partnerId}`. partnerId = 로그인 시 재귀 CTE 1회(`type='partner'` 조상) → carry-forward(refresh 시 재계산 안 함). `JWT_EXPIRES_IN=1h`

### SP-P-006 · 수강권 2모델
- **픽클래스 B2B** = `access_codes`(과정당 1코드, `usage_start/end_date`, `status_code='active'`)
- **파고다 파트너** = 카테고리 전체(정규상품 레슨N회 / 무제한상품). 나만의수업: 정규=레슨풀 차감 / 무제한=20회 cap
  - D1(파고다 grant, 픽클래스 consumption) / D2(최초 이수완료 시 1회 차감·재수강 무차감) / D3(상품기간당 20회)
- ⚠️ 신규 원장 `enrollment_usage`(설계, 미착수)

---

## 4. 학습 설정 · 알림

### SP-P-007 · 마이크 모드
- **PTT**(라이트, 명시적 누름) / **서브웨이 모드**(조용한 모드/키보드 입력, 3-strike 트리거 시 안내). ~~Always-On(노멀, 자동 인식)~~ **폐기** — PTT로 학습 흐름 통제권 확보(본편 §14.4.6). RPB·RPF는 PTT 방식(학습 흐름 통제)
- 탭 모드(마이크 OFF) 발화 **×0.6 가중**
- 허용 마이크 모드는 백오피스 기관 수업설정(`allowed_mic_modes`)에서 강제(BO 연동)

### SP-P-008 · 알림 · 방해금지
- **학습 알림(MY-008)**: `user_learning_profiles.reminder_enabled/reminder_time/reminder_phone_mode`(정규화 컬럼)
- **광의 푸시**: `users.notification_settings` JSONB — push 9종(mission/streak/challenge/intimacy/omp_reminder/utterance/character/admin/return) + `dnd_start/dnd_end`(기본 22:00~08:00) + kakao/email/marketing_opt_in. `GET/PATCH /users/notification-settings`(deep-merge)
- **파이프라인 게이트 순서**: ① 카테고리 OFF → 전체 skip ② Record 항상(공유 `notifications`, type=`speaking.*`) ③ DND → 푸시만 억제·내역 기록. 스케줄러=외부 크론(`X-Cron-Secret`)
- ⚠️ 알림 파이프라인 **미구현 → UI 잠정 숨김**

---

## 5. 문장 마스터리 엔진

### SP-P-009 · 레벨·망각곡선
- **마스터리 레벨**: Lv0(0)/Lv1(1~2)/Lv2(3~4)/Lv3(5~7)/**Lv4 마스터(8+)**. `mastery_target=8`(문장 누적 발화수 기준)
- **망각곡선 간격**: `[1, 2, 4, 7, 16, 35]`일. Quiz 세션당 **6문장**(5~8), 문장별 1~2발화, 하루 1세션 권장, due 초과분 이월
- **저장**: 단일 `user_expressions`. `source` = `learning`(Quiz, `texts.core_expressions` 편입) / `note`(사용자) / `preset`(backoffice 카탈로그, 로드맵 P5)
- **API 분리**: 기존 `POST /:id/review` → `utterance`(count++ + `utterance_logs` insert) + `complete-session`(망각곡선 전진)

---

## 6. 오픈 이슈 (정책 미확정 / 미착수)

| # | 이슈 | 영향 |
|---|---|---|
| O-1 | `alt_sessions.user_id`(integer) vs `users.id`(UUID) 타입 불일치 | SP-P-001, 003 |
| O-2 | 기관 평균 KPI 비교 API 부재 → Mock/제거 | SP-P-004, SP-F-012, 040 |
| O-3 | 파고다 수강연동 원장 `enrollment_usage` 미착수 | SP-P-006 |
| O-4 | 알림 파이프라인/스케줄러 미구현 | SP-P-008 |
| O-5 | preset 표현 카탈로그(backoffice) 로드맵 P5 | SP-P-009 |
| O-6 | 레벨테스트 게이팅 renewal(soft_prompt) Phase2(access_codes READ 필요) | SP-P-001 |
