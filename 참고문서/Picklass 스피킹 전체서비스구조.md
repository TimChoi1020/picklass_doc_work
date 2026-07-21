## 3\. 전체 서비스 구조

### 3.1 서비스 아키텍처 개관 (도메인 · 채널)

Picklass는 **B2B 채널과 B2C 채널을 도메인으로 분리**한다. 서브도메인은 `admin.picklass.com` 외 신규 추가하지 않으며, studio.picklass.com은 **루트(/) \= B2B 홍보 랜딩**, **/app \= Studio 저작 앱**의 이중 용도로 운영한다.

| 구분 | 도메인 / 경로 | 역할 | 주 사용자 | 공개성 |
| :---- | :---- | :---- | :---- | :---- |
| **B2C 랜딩 · 학생 진입** | [**www.picklass.com**](http://www.picklass.com) | Phase 1: B2C 티저 \+ 학생 로그인·액세스코드 CTA Phase 2: 풀 B2C 브랜드 랜딩 | 학생 · 학부모 · 개인 학습자 | 공개 · SEO 집중 |
| **B2B 홍보 랜딩** | **studio.picklass.com/** (루트) | 제휴 문의 · 데모 신청 · 사례 소개 · 기관장·강사 로그인 | 학원 관리자 · 강사 · 파트너 | 공개 · B2B 키워드 SEO |
| **Studio 저작 앱** | **studio.picklass.com/app/** | 지문/과정/레슨 제작, 모듈 편집, 학생 관리 | 강사 · 기관장 | 로그인 필수 · noindex |
| **Admin 백오피스** | **admin.picklass.com** | 기관/사용자/콘텐츠/요금제/AI모듈/액세스코드 중앙 관리 | 시스템·학원관리자 | 내부 · noindex |
| **튜터링** | **tutoring.picklass.com** | AI 튜터 Pickle과의 개인 맞춤 학습, 학생 로그인 화면 | 학생 | 로그인/코드 필수 · noindex |
| **스피킹 튜터** | **speaking.tutoring.picklass.com** | 실시간 음성 대화 학습 (독립 앱 \+ 통합 모듈) | 학생 · 개인 구독자 | 로그인/코드 필수 · noindex |

**핵심 원칙**

- 학부모/학생에게 노출되는 공개 홍보 도메인은 **오직 [www.picklass.com](http://www.picklass.com)** 하나  
- 학원·강사·파트너 대상 홍보는 **studio.picklass.com/** 루트에서만 진행  
- Phase 1 초기에는 B2B 세일즈가 주력 → studio가 활성 홍보 채널, www는 티저 \+ 학생 진입점  
- Phase 2 B2C 확장 시 www가 메인 브랜드 채널로 전환 (도메인 이전/리다이렉트 재작업 불필요)

### 3.2 서비스 간 관계도

**진입 채널이 B2C(www)와 B2B(studio 루트)로 이원화**된다. 학생 로그인은 www에 CTA만 두고 클릭 시 `tutoring.picklass.com/login`으로 리다이렉트한다(옵션 B).

┌──────────────────────────────────────────────────────────────────┐

│                        \[진입 채널 분리\]                          │

├──────────────────────────────────────────────────────────────────┤

│                                                                  │

│   B2C / 학생 · 학부모                   B2B / 기관 · 강사        │

│        │                                       │                 │

│        ▼                                       ▼                 │

│  ┌─────────────────────┐             ┌──────────────────────┐   │

│  │ www.picklass.com    │             │ studio.picklass.com/ │   │

│  │ Phase1: B2C 티저    │             │  (루트)              │   │

│  │       \+ 학생 CTA    │             │  B2B 홍보 랜딩       │   │

│  │ Phase2: B2C 주력 ★  │             │  제휴/데모 문의       │   │

│  │ \[공개 SEO\]          │             │  강사·기관장 로그인  │   │

│  └─────────┬───────────┘             └───────┬──────────────┘   │

│            │                                 │                   │

│            │ \[학생 로그인\] CTA 클릭           │ 로그인             │

│            │   ↓ 리다이렉트                   │                   │

│            │                                 │                   │

│            ▼                                 ▼                   │

│  ┌─────────────────────────┐      ┌───────────────────────────┐ │

│  │ tutoring.picklass.com   │      │ studio.picklass.com/app/  │ │

│  │ /login  (학생 로그인)   │      │ (Studio 저작 앱)          │ │

│  │                         │      │  · 지문/레슨 제작         │ │

│  │ → 모듈식 레슨 진행      │      │  · 학생 배정              │ │

│  │   · Pickle AI 튜터      │      │  · 성과 모니터링          │ │

│  └─────────┬───────────────┘      └───────────────────────────┘ │

│            │                                                     │

│            │ Speaking 모듈 호출 or 독립 진입                      │

│            ▼                                                     │

│  ┌─────────────────────────────────────┐                         │

│  │ speaking.tutoring.picklass.com      │                         │

│  │  · 실시간 음성 대화 (Free-talking)  │                         │

│  │  · 통합 모듈 \+ 독립 앱 양쪽 지원    │                         │

│  └─────────────────────────────────────┘                         │

│                                                                  │

│  ┌─────────────────────────┐                                     │

│  │ admin.picklass.com      │   \[시스템/학원 관리자\]               │

│  │  · 기관/사용자/코드     │     ↑ studio에서 역할 확인 후        │

│  │  · 요금제/Billing       │       관리 권한으로 진입              │

│  │  · AI 모듈/시스템 설정  │                                     │

│  └─────────────────────────┘                                     │

│                                                                  │

└──────────────────────────────────────────────────────────────────┘

**주요 플로우 요약**

- **학생**: [www.picklass.com](http://www.picklass.com) → \[학생 로그인\] 또는 \[액세스코드 입장\] → `tutoring.picklass.com/login` → 학습  
- **강사/기관장**: studio.picklass.com/ 홍보 랜딩 → \[로그인\] → `studio.picklass.com/app`  
- **시스템/학원 관리자**: studio 로그인 후 역할 확인 → `admin.picklass.com`  
- **Speaking**: tutoring 내 모듈 호출 or `speaking.tutoring.picklass.com` 직접 진입

### 3.3 Speaking Tutor 이중 운영 모델

| 운영 모드 | 진입 경로 | 세션 단위 | 데이터 저장 위치 |
| :---- | :---- | :---- | :---- |
| **통합 모듈 (튜터링+스피킹)** | Tutoring 레슨 내 Speaking 모듈 슬롯 | LessonPlan의 PlannedModule | ModuleHistory에 호환 저장 |
| **독립 앱 (스피킹 단독)** | speaking.tutoring.picklass.com 직접 진입 | SpeakingSession (독립 엔터티) | SpeakingSession 테이블 |

두 경로 모두 동일한 **SpeakingConversationAgent**와 음성 인프라(STT/TTS)를 사용하되, **진입점/UI/인증/설정 경로/과금/리포트가 분기**된다.

#### 3.3.1 통합 모듈 플로우 (튜터링 → 스피킹)

학생이 Tutoring에 로그인한 상태에서 Speaking 모듈로 진입하는 경우, 인증·레벨·콘텐츠 설정이 **Tutoring에서 자동 전달**되어 학생이 별도 입력할 필요가 없다.

**사용자 여정**

\[튜터링 로그인\]

    ↓

\[수강(모듈) 목록\]

    ↓

\[스피킹 선택\]

    ↓

\[새 세션 시작 화면\]

  · 이름 (GET\_AUTH API 기본값, 수정 가능)

  · 마이크 선택·테스트

  · 튜터(Pickle 등) 선택

    ↓

\[세션 시작\]  ← 이 시점에 ONETIME TOKEN 소멸

    ↓

\[결과 화면\]

    ↓

\[수강(모듈) 목록\]  (원래 화면으로 복귀)

**핵심 인증·연동 메커니즘**

| 메커니즘 | 설명 |
| :---- | :---- |
| **ONETIME TOKEN** | Tutoring에서 Speaking 서비스로 페이지 이동 시 **일회용 토큰을 생성**하여 쿼리스트링 형태로 전달. 브라우저 이력·북마크로 재진입 불가하도록 보장. |
| **GET\_AUTH API** | Speaking 서비스가 ONETIME TOKEN을 키로 호출하는 인증·컨텍스트 조회 API. 응답 필드: **아이디, 이름, 인증코드, 레벨, 콘텐츠, 대화 분량, 음성 모드** |
| **이름 필드 정책** | GET\_AUTH 응답의 이름(기관에서 제공 가능)을 기본 설정하고, 세션 시작 화면에서 **사용자가 수정 가능**하도록 노출 |
| **토큰 소멸** | 세션 시작 시점에 ONETIME TOKEN 소멸. **소멸 API**가 Speaking 서비스 → Tutoring 방향으로 호출되어 토큰 상태를 무효화 |

**API 인터페이스 요약**

Tutoring → Speaking:

  GET https://speaking.tutoring.picklass.com/?token={ONETIME\_TOKEN}

Speaking → Tutoring:

  POST /api/auth/get-auth        (payload: { token })

       응답: { userId, name, accessCode, level, content, dialogLength, voiceMode }

  POST /api/auth/revoke-token    (payload: { token })

       세션 시작 시 호출, 토큰 소멸

#### 3.3.2 독립 앱 플로우 (스피킹 단독)

사용자가 Tutoring을 거치지 않고 Speaking 서비스에 **직접 진입**하는 경우. 모든 설정을 사용자가 직접 선택한다. 현재(v0.5 기준) 구현 플로우와 동일.

**사용자 여정**

\[스피킹 아이디 로그인\]

    ↓

\[진행중인 세션 목록\]

    ↓

\[새 세션 시작\]

    ↓

\[인증코드 입력\]

    ↓

\[새 세션 시작 화면\]

  · 레벨·콘텐츠·대화 분량·음성 모드 등 모든 설정 사용자가 직접 선택

  · 이름/마이크/튜터 선택

    ↓

\[세션 시작\]

    ↓

\[결과 화면\]

**핵심 특징**

- 자체 로그인 · 인증코드 · 세션 히스토리 모두 Speaking 서비스 내에서 관리  
- ONETIME TOKEN / GET\_AUTH API **사용하지 않음**  
- 세션 종료 후 Tutoring 측 수강 목록으로 돌아가지 않음 (Speaking 독립 목록으로 복귀)  
- 과금·리포트는 Speaking 구독 플랜 기준 (§5.3 연동)

#### 3.3.3 두 플로우 대조

| 항목 | 통합 모듈 (튜터링+스피킹) | 독립 앱 (스피킹 단독) |
| :---- | :---- | :---- |
| 진입 경로 | 튜터링 로그인 → 수강 목록 → 스피킹 선택 | 스피킹 직접 로그인 → 세션 목록 |
| 인증 방식 | ONETIME TOKEN \+ GET\_AUTH API | 스피킹 자체 아이디/인증코드 |
| 사전 설정 | Tutoring이 전달 (레벨/콘텐츠/분량/음성모드) | 사용자가 직접 선택 |
| 이름 필드 | GET\_AUTH 제공값 기본 \+ 수정 가능 | 사용자 직접 입력 |
| 세션 종료 후 복귀 | 튜터링 수강 목록 | 스피킹 세션 목록 |
| 세션 저장 | ModuleHistory (통합) \+ SpeakingSession 연결 | SpeakingSession 단독 |
| 과금 | 기관 플랜 번들 또는 Tutoring 구독 | Speaking 단독 구독/세션 추가 |

📎 통합 모듈의 상세 데이터 흐름과 UX 구성은 **§14.10 통합 모듈 연동**, 독립 앱 UX는 **§14.9 독립 앱 UX**를 참조한다.

### 3.4 도메인 맵

- **Identity**: User, Institution, AccessCode, Role, Session  
- **Content**: Passage, Lesson, Course, Module(Reading/Writing/Speaking)  
- **Learning**: ModuleHistory, LessonResult, KpiResult, SpeakingSession/Turn  
- **Commerce**: Plan, Subscription, Invoice, Payment, AIUsage  
- **System**: AIModule, Level(CEFR 18), Status, Duration

### 3.5 공통 인프라 및 도메인/SEO 정책

**공통 인프라**

- **인증**: 이메일 \+ 소셜 OAuth (Google / Kakao / Naver)  
- **DB**: PostgreSQL (Backoffice/Tutoring), Supabase (Studio/www)  
- **AI API**: Anthropic Claude(claude-haiku-4-5-20251001, Tool Use), Google GenAI  
- **음성**: Azure Speech (TTS 우선), Google STT, 추후 Azure STT 대체 검토

**도메인/SEO 정책**

| 도메인 / 경로 | 인덱싱 | 타겟 키워드 유형 | 근거 |
| :---- | :---- | :---- | :---- |
| [www.picklass.com](http://www.picklass.com) | **index 허용** | B2C (AI 영어 튜터, 영어 회화 학습 등) | Phase 2 브랜드 주력 채널 |
| studio.picklass.com/ (루트) | **index 허용** | B2B (학원 AI 교재, 영어 학원 솔루션 등) | 파트너 유입 필요 |
| studio.picklass.com/app/\*\* | **noindex** | \- | 앱 화면 공개 차단 |
| admin.picklass.com | **noindex** | \- | 관리자 전용, 공개 불필요 |
| tutoring.picklass.com | **noindex** | \- | 학생 로그인/학습 전용 |
| speaking.tutoring.picklass.com | **noindex** | \- | 학생 로그인/학습 전용 |

**인증 쿠키 스코프**

- www ↔ tutoring/speaking 간 SSO가 필요한 경우 `.picklass.com` 상위 도메인 쿠키 사용  
- studio ↔ admin 간 역할 검증 핸드오프는 서버 사이드 세션 또는 JWT 전달

**레거시 URL 리다이렉트 (Phase 1 전환)**

- 기존 [www.picklass.com의](http://www.picklass.com의) B2B 경로(`/partners`, `/demo`, `/pricing`, `/institution` 등) → `studio.picklass.com/*` 301 영구 이동  
- www 루트(`/`)는 **리다이렉트하지 않음** — B2C 티저/학생 진입점으로 유지

---

