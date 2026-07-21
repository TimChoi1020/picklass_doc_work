# Speaking IA (정보구조)

> 참고문서(`참고문서/picklass_docs/speaking/`)를 기반으로 정리한 Speaking(스피킹 앱) 서비스의 정보구조 문서입니다.
> **핵심 근거**: `architecture/2026-06-03_스피킹앱-IA구조정리-개발계획.md`, `architecture/피클 스피킹앱 화면 구조 - 시트1.tsv`
> 명칭 기준: 2026-07-10 재디자인 반영 (원안 TSV/IA문서와 일부 차이는 §6 참조)

---

## 0. 서비스 개요 & IA 설계 배경

**Speaking은 학습자가 직접 말하기를 연습하는 모바일 중심 학습 앱**입니다. Studio·Backoffice가 강사·관리자용 도구라면, Speaking은 **학생이 매일 접속해 학습을 반복하는 소비자(B2C 스타일) 앱**이라는 점이 IA 설계의 출발점입니다.

**IA 설계의 3가지 배경 원리:**

**① "일상 반복 영역(탭)"과 "몰입 학습 영역(풀스크린)"의 분리.** 하단 5개 탭(홈·수업·챌린지·리포트·MY)은 앱을 켤 때마다 오가는 상시 네비게이션이고, 실제 말하기 학습이 진행되는 플레이어 화면(`/learn-flow`, `/player/*`)은 탭바를 숨긴 **풀스크린 몰입 모드**로 분리됩니다. 학습에 집중할 때 하단 탭이 방해하지 않도록 한 구조입니다.

**② 화면 ID 체계로 전 화면을 관리.** 모든 화면에 `PKS-{영역}-{번호}` 코드를 부여(예: `PKS-HM-001` 홈)해 기획·개발·QA가 동일한 화면을 지칭합니다. 접두어만 봐도 어느 탭 소속인지 알 수 있습니다.

**③ 두 갈래 진입 경로(통합 모듈 vs 독립 앱).** Speaking은 두 방식으로 열립니다 — (a) **통합 모듈**: Tutoring 과정 안의 한 모듈로 호출(1회용 토큰으로 세션 시작), (b) **독립 앱**: `speaking.tutoring.picklass.com`에 스피킹 아이디로 직접 로그인. 같은 학습 콘텐츠지만 진입·인증 방식이 다릅니다(§3.5).

**학습 흐름의 큰 그림**: 신규 학습자는 먼저 **레벨테스트(P-ALT)로 게이팅**된 뒤(§3.1), 홈에서 오늘의 학습을 시작하고, 챌린지·리포트로 동기부여와 피드백을 받는 루프를 반복합니다.

---

## 1. 네비게이션 구조

### 화면 ID 접두어
`PKS-FL`(풀스크린 오버레이) · `PKS-HM`(홈) · `PKS-CL`(수업) · `PKS-CH`(챌린지) · `PKS-RP`(리포트) · `PKS-MY`(MY)

### 하단 탭 5종 (BottomNav) — `app/(tabs)/`
5개 탭은 각각 학습 사이클의 한 국면을 담당합니다: **홈(오늘 할 일)·수업(과정 학습)·챌린지(복습·동기부여)·리포트(성과 확인)·MY(설정)**.

| 순번 | 탭명 | ID | 경로 | 역할 |
|---|---|---|---|---|
| 1 | 홈 | PKS-HM-001 | `/home` | 오늘의 학습 진입점·현황 요약 |
| 2 | 수업 | PKS-CL-001 | `/learn` | 정규 과정 학습·과정 탐색 |
| 3 | 챌린지 | PKS-CH-001 | `/challenge` | 복습 퀴즈·표현노트·발화량·배지 |
| 4 | 리포트 | PKS-RP-001 | `/feedback` | 학습 성과 분석·약점 피드백 |
| 5 | MY | PKS-MY-001 | `/my` | 계정·수강내역·학습/앱 설정 |

- 루트 `/` → `/home` 리다이렉트. 인증 가드 + BottomNav 공통 레이아웃.

### 탭 바깥 풀스크린 (탭바 없음 = 몰입 학습 영역)
- `/onboarding/*` — PKS-FL-001 (로그인 / purpose / persona / level-test)
- `/learn-flow/*` — PKS-FL-002 정규 학습 Player (5단계)
- `/player/*` — PKS-FL-003 FRT / PKS-FL-004 퀴즈 / PKS-FL-005 표현노트
- 공통: `player/layout.tsx` 우상단 X → `router.back()`

### 공통 오버레이
- `BottomSheet`(아래서 올라옴, `layer=1`) · `Drawer`(오른쪽 슬라이드, `layer=1`) — 둘 다 `createPortal`로 PhoneFrame에 마운트

---

## 2. 탭·화면별 기능 설명

### 홈 (PKS-HM-001) `/home`
**"오늘 무엇을 학습할지 즉시 알려주고, 내 학습 현황을 요약해 보여주는 대시보드"**. 앱을 켰을 때 가장 먼저 만나는 액션 허브입니다.
- 헤더: 시간대 인사 `Good Morning/Afternoon/Evening, {firstName}!`
- 상단 오버레이: 레벨테스트 진입 팝오버(`LevelTestPopover`, 진단 필요 시)
- **섹션1 오늘의 학습**(가장 중요한 CTA): ① 정규 카드(제목·챕터·분·진행바) → `/learn-flow/pick` / ② Talk with Pickle(FRT 자유대화) → `/player/frt`
- **섹션2 학습 현황**: 기간칩 + 이수율(`N/전체강`+바) · 발화량(`current/Goal`) — 목표 대비 진척 요약
- **섹션3 스피킹 밸런스**: SVG 레이더 5축(발음·유창성·문법·화용성·발화량), 사용자+기관평균+범례 — 내 실력의 균형을 시각화

### 수업 (PKS-CL-001) `/learn`
**"현재 수강 중인 정규 과정을 이어서 학습하고, 새 과정을 찾는 학습의 메인 무대"**.
- 섹션1 상태 요약: 진척률 · 남은 수업 수 · 발화량(→ `/challenge`)
- 섹션2 현재 과정: 과정명(→ CL-002 과정 변경) + 레슨 목록(→ `/learn-flow/pick?chapterId=`)
- 섹션3 자유수업(FRT) → `/player/frt`
- 섹션4 하단 버튼: 과정 변경(→ CL-002) / 과정 탐색하기(→ CL-003)
- **PKS-CL-002** 수강 과정 선택(BottomSheet, `hasProgress` 과정만) · **PKS-CL-003** `/learn/explore`(추천·카테고리·검색)

### 챌린지 (PKS-CH-001) `/challenge`
**"복습과 동기부여를 담당하는 탭"**. 정규 학습이 '새로 배우기'라면 챌린지는 '반복·게임화'로, 발화량 통계·랭킹·배지로 꾸준한 학습을 유도합니다.
- 섹션1 발화량 통계: 기간 필터(주간/월간) + 막대 차트 + 평균선 + 랭킹 pill
- 섹션2 복습 퀴즈 → `/player/quiz` (PKS-FL-004) — 배운 문장 섀도잉 복습
- 섹션3 표현 노트 → `/player/expressions` (PKS-FL-005) — 저장한 표현 관리·연습
- 섹션4 배지 컬렉션 → `/challenge/badges` (PKS-CH-003)
- 섹션5 AI 캐릭터 챗 (v2.0 예정)
- **PKS-CH-002** 발화량 랭킹(BottomSheet, 닉네임 마스킹) · **PKS-CH-003** `/challenge/badges`(획득/미획득)

### 리포트 (PKS-RP-001) `/feedback`
**"내 학습 성과를 정량·정성으로 분석하고 약점을 다음 학습으로 연결해주는 피드백 탭"**.
- 섹션1 현재 수강 기간 카드(2개 이상 시 → 수강기간 선택 BottomSheet)
- 섹션2 종합 리포트(주/월/90일): Quality(발음·유창성·문법·화용) / Quantity(발화·듣기·섀도잉) / Growth(5축 변화량) → "레슨별 리포트 보기" `/feedback/lessons`
- **섹션3 약점 분석**(피드백 루프의 핵심): 추천에 따라 → `/learn`(Quality 약점) 또는 `/challenge`(Quantity 약점)로 유도
- 섹션4 1MP(1분 발표) 갤러리(3열): → `/feedback/1mp/[id]`, 전체보기 `/feedback/1mp`
- **PKS-RP-002** `/feedback/lessons` · **PKS-RP-003** `/feedback/1mp` · **PKS-RP-004** 레슨 상세(Drawer) · **PKS-RP-005** `/feedback/1mp/[id]`

### MY (PKS-MY-001) `/my`
**"계정·수강권·학습 설정을 관리하는 마이페이지"**. 2026-07-10 재디자인으로 상단 히어로 + 카드 리스트 구조로 재구성됨.
- **헤더 히어로**(블루 그라디언트): 아바타(준비중) / 이름+연필(→ MY-002 닉네임 수정) / 레벨 배지 + 레벨테스트 리포트(준비중) / 수강권 드롭다운(→ `/learn?courseId=`) / 로그아웃(→ `/home`)
- **본문 카드**: 수강내역(→ `/my/history`) / 공지사항 N(→ `/my/notifications`) / 설정
- **BottomSheet(설정류)**: MY-002 닉네임 수정 · MY-005 앱 설정(볼륨) · MY-007 학습 시간 설정 · MY-008 학습 알림 설정 · MY-009 AI 페르소나(Sarah/James/Mia)
- **푸시 페이지**: MY-003 `/my/history`(수강 내역, 종료 카드 → `/feedback?enrollmentId=`) · MY-004 `/my/settings` · MY-006 `/my/notifications` · MY-010 `/my/notifications/[id]`

### 풀스크린 플레이어 (PKS-FL) — 몰입 학습 영역
**실제 말하기 학습이 진행되는 전용 화면군**. 탭바를 숨기고 학습에만 집중하도록 설계.

| ID | 경로 | 내용 | 진입 |
|---|---|---|---|
| FL-001 | `/onboarding/...` | 온보딩(login/purpose/persona/level-test) | 첫 로그인 |
| FL-002 | `/learn-flow/...` | 정규 학습 5단계 | 홈 정규 카드, 수업 레슨 |
| FL-003 | `/player/frt` | FRT 자유 대화 | 홈 FRT, 수업 FRT |
| FL-004 | `/player/quiz` | 복습 퀴즈(섀도잉) | 챌린지 섹션2 |
| FL-005 | `/player/expressions` | 표현 노트 | 챌린지 섹션3 |

**정규 학습 Player `/learn-flow` 5단계**(하나의 지문을 배우는 표준 흐름): `pick`(미션 선택) → `learn`(학습) → `drill`(반복 훈련, 5축 즉시 피드백/Azure 발음평가) → `apply`(실전 적용, role-play / free-talking / presentation 리다이렉트) → `measure`(레이더 결산)

**모듈 학습 9개 모듈**(정규 과정 시퀀싱 단위): LRN(강의영상) → VLM(어휘) → SHD(쉐도잉) → SFB(빈칸채워말하기) → SMK(문장만들기·LLM) → RPB(롤플레이 기본) → RPF(롤플레이 프리) → OMP(1분 발표·LLM) / FRT(자유대화·별도). 각 모듈은 6단계 표준(데이터준비→수업시작→문항제시→답안수행→피드백+재도전→완료)을 따름.

**문장 마스터리(퀴즈/표현노트)**: S1 진입점 → S2 Quiz 섀도잉 세션(`/player/quiz`) → S3 세션 종료 요약 / S4 표현노트 목록(`/player/expressions`) → S5 표현 추가 시트 / S6 단일 문장 섀도잉

---

## 3. 주요 사용자 플로우

### 3.1 레벨테스트 게이팅 (신규 사용자)
**신규 학습자는 자기 레벨을 모른 채 학습할 수 없으므로, 첫 진입 시 레벨테스트를 강제**합니다. 상황별로 강도가 다릅니다.
1. 홈 로드 → `GET /levels/status`
2. `requiresTest=true` → 팝오버(reason: new_user/renewal/after_lesson)
3. [시작하기] → `/onboarding/level-test`(PT 플로우) / [나중에] → 닫힘
- **하드블록**(신규 PT): 수업 카드 잠금 + 시작 엔드포인트 403 `LEVEL_TEST_REQUIRED`, 매 진입 재노출 — 반드시 봐야 통과
- **soft_prompt**(갱신 PT·AT): 수업 허용, 24h 스누즈. PT 완료 시 닉네임+레벨 동시 저장

### 3.2 레벨테스트(P-ALT) 내부 플로우
**Placement Test(신규 진단)와 Achievement Test(학습 후 성취도)가 체인으로 이어지는 적응형 진단**입니다.
- 유형: PT(Placement, 신규 자동 1회+원할 때) ─ AT(Achievement, 학습 후) 체인
- 파트: P1 Vocab Size(4지선다 CAT) → P2 문장 낭독 CAT → P3 일상 대화(IELTS P1형 6~10턴) → P4 주제 발표(IELTS P2형, 조건부)
- 최종: 6대 루브릭 가중합산 → 레벨 산출 → 리포트(6섹션, 공인시험 예상 구간 병기)

### 3.3 정규 학습 시작~완료
홈 정규 카드 or 수업 레슨 → `/learn-flow/pick?chapterId=` → learn → drill → apply(role-play/free-talking/presentation) → measure → 탭 복귀

### 3.4 복습 퀴즈(문장 마스터리)
챌린지 Quiz 카드 → S2 세션(문장별 듣기→발화→평가→리워드, `POST /expressions/:id/utterance`) → `complete-session` → S3 요약 → 챌린지 복귀

### 3.5 서비스 진입/도메인 (통합 vs 독립)
같은 학습이라도 **어디서 진입하느냐에 따라 인증 방식이 달라집니다.**
- 학생: `www.picklass.com` → 로그인/액세스코드 → `tutoring.picklass.com/login` → 학습
- **통합 모듈**: 튜터링 로그인 → 수강 목록 → 스피킹 선택 → 새 세션 화면 → 세션 시작(**ONETIME TOKEN 소멸**) → 결과 → 목록 복귀 (인증: ONETIME TOKEN + GET_AUTH API)
- **독립 앱**(`speaking.tutoring.picklass.com`): 스피킹 로그인 → 진행중 세션 목록 → 새 세션 → 인증코드 입력 → 세션 시작 → 결과 (ONETIME TOKEN 미사용)

---

## 4. 화면 전환 매트릭스 (주요)

| 출발 | 도착 |
|---|---|
| 홈 정규 카드 | `/learn-flow/pick` (FL-002) |
| 홈 FRT 카드 | `/player/frt` (FL-003) |
| 홈 팝오버 [시작하기] | `/onboarding/level-test` |
| 수업 과정명 | CL-002 (BottomSheet) |
| 수업 레슨 | `/learn-flow/pick?chapterId=` |
| 수업 과정 탐색 | `/learn/explore` (CL-003) |
| 챌린지 복습 퀴즈 | `/player/quiz` (FL-004) |
| 챌린지 표현 노트 | `/player/expressions` (FL-005) |
| 챌린지 배지 더보기 | `/challenge/badges` (CH-003) |
| 리포트 레슨별 리포트 | `/feedback/lessons` (RP-002) |
| 리포트 lessons 레슨 | RP-004 (Drawer) |
| 리포트 약점분석 추천 | `/learn` 또는 `/challenge` |
| 리포트 1MP 썸네일/전체보기 | `/feedback/1mp/[id]` / `/feedback/1mp` |
| MY 수강내역 | `/my/history` (MY-003) |
| MY 수강권 드롭다운 항목 | `/learn?courseId=` |
| `/my/history` 종료 카드 | `/feedback?enrollmentId=` |
| 플레이어 X 버튼 | `router.back()` |

---

## 5. 구현 상태 (2026-07-10 기준)
- ✅ 5탭 UI 전체(Mock 다수), 공통 BottomSheet·Drawer, 인증(로그인·회원가입 실데이터)
- ✅ 실연동: 홈 발화량·이수율바·스피킹밸런스 사용자폴리곤·오늘의 학습 / 수업 탭 수강과정(`enrolled-courses`) / MY 수강내역·프로필 / 레벨테스트 게이팅
- ⚠️ 스켈레톤: `/learn-flow`(5단계), `/player/*`, `/onboarding`(purpose/persona/level-test)
- ⚠️ Mock 잔존: 홈 이수율 개수·기관평균 폴리곤·기간칩, 챌린지 시계열·배지

---

## 6. 명칭 차이 주의
문서 버전 간 명칭·구성이 달라 혼동하기 쉬운 지점입니다.
- 홈 섹션2: 원안 "발화량·이수율·수업시간 3열" → 재디자인 "이수율·발화량 2행(수업시간 제거)"
- MY: 원안 5섹션(계정/수강내역/수업설정/앱설정/알림) → 재디자인 "헤더 히어로 + 설정 단일카드" 재구성
- 레벨테스트 PT 플로우는 문서에 따라 `PKS-LV-002`로도 지칭됨

---

## 7. 근거 문서
- `참고문서/picklass_docs/speaking/architecture/2026-06-03_스피킹앱-IA구조정리-개발계획.md` — IA 구조 핵심
- `.../architecture/피클 스피킹앱 화면 구조 - 시트1.tsv` — 화면 구조 원본 기획
- `.../architecture/P-ALT_Speaking_v8.md` — 레벨테스트(P-ALT) 기획서
- `참고문서/Picklass 스피킹 전체서비스구조.md` — 도메인/서비스 진입 구조
- `.../features/홈탭/2026-07-10_홈탭-재디자인.md` · `.../features/챌린지/2026-07-10_*.md`
- `.../features/마이페이지/2026-07-10_*.md` · `.../features/레벨테스트/2026-07-09_레벨테스트-진입-게이팅.md`
- `.../features/모듈학습/20260615_스피킹 모듈 시나리오.md` · `.../features/learn페이지-실API연동/20260619_*.md`
