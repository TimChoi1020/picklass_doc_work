# Picklass 전체 서비스 기획서

> **문서 버전:** v0.21 (정합성 일괄 정정 — 구 장번호 cross-ref 수정 · Speaking 6모듈(v0.15) 잔여 반영 · 모듈 총수 통일 · 좌석 배분(v0.17 폐기) 잔재 제거 · 변경 이력 갱신)
> **작성일:** 2026-05-08 (v0.21 갱신: 2026-07-17)
> **문서 유형:** 통합 서비스 기획서 (Product Planning Document)
> **상태:** Draft
> **작성자:** Tim (tim.hy.choi@gmail.com)

---

## 목차 (Table of Contents)

- **0. 문서 상태 표기 범례**
- **1. 문서 개요**
  - 1.1 문서 목적과 범위
  - 1.2 대상 독자
  - 1.3 용어 정의 / 약어표
- **2. 서비스 개요 (Executive Summary)**
  - 2.1 제품 비전
  - 2.2 핵심 가치 제안
  - 2.3 시장 포지셔닝 및 차별점
  - 2.4 현재 상태
  - 2.5 ❌ 핵심 성과 지표 요약
  - 2.6 On-demand 콘텐츠 생성 원칙
- **3. 전체 서비스 구조**
  - 3.1 서비스 아키텍처 개관 (도메인 · 채널)
  - 3.2 서비스 간 관계도
  - 3.3 Speaking Tutor 이중 운영 모델
  - 3.4 도메인 맵
  - 3.5 공통 인프라 및 도메인/SEO 정책
- **4. Actor & 조직 모델**
  - 4.1 Actor Taxonomy 개관
  - 4.2 조직 계층 (Organization Hierarchy)
  - 4.3 Organization Entities 정의
  - 4.4 Person Actors 정의 (8종)
  - 4.5 Role 권한 매트릭스 (관리 Scope × 기능)
  - 4.6 계정 모델 — 단일 User + 복수 Membership
  - 4.7 B2B/B2C 채널 분리 원칙
  - 4.8 Organization/사용자 상태 체계
  - 4.9 Speaking 단독 사용자 시나리오
  - 4.10 경계 케이스 (Edge Cases)
- **5. 비즈니스 모델**
  - 5.1 B2B 3채널 모델 (Academy / Enterprise / K12)
  - 5.2 중소학원 SaaS 구독 모델
  - 5.3 Speaking Tutor 단독 구독/부가 옵션 모델
  - 5.4 무료 체험 정책
  - 5.5 요금제 구조
  - 5.6 Speaking 이용 시간/세션 기반 과금 고려사항
  - 5.7 추가 과금 항목
  - 5.8 환불 및 해지 정책
- **6. 랜딩 페이지 (B2C / B2B 분리 운영)**
  - 6.A www.picklass.com — B2C 티저 · 학생 진입
  - 6.B studio.picklass.com/ — B2B 홍보 랜딩
  - 6.C studio.picklass.com/app — Studio 저작 앱 진입
  - 6.D 공통 정책
- **7. 백오피스 (Admin)**
  - 7.1 페이지 맵 및 IA
  - 7.2 대시보드
  - 7.3 Organization 관리 (Partner / Group / Institution)
  - 7.3A 기존 명칭 (Institute) 호환 (Deprecated)
  - 7.4 사용자관리 (Users)
  - 7.5 액세스코드 관리
  - 7.6 수업 모듈 관리 (AI Modules)
  - 7.7 Billing (청구 관리)
  - 7.7.1 플랜 상품 구성 옵션 (Product Configuration)
  - 7.7.2 ❌ 좌석 배분 정책
  - 7.8 시스템관리
  - 7.9 권한 및 감사 로그
  - 7.10 기관 브랜딩 (Institution Branding)
- **8. 스튜디오 (Studio · 강사 운영 레이어)**
  - 8.1 서비스 개요 및 역할
  - 8.2 과정 / 레슨 / 지문 계층 구조 (아키텍처 맵)
  - 8.3 Course Hub
  - 8.4 지문 라이브러리 UX
  - 8.5 레슨 편집 UX
  - 8.6 모듈 라이브러리 활용
  - 8.7 Speaking 시나리오/주제 편집 도구
  - 8.8 학생 배정 및 배포
  - 8.9 학생 성과 모니터링
  - 8.10 권한·공유 정책
- **9. 콘텐츠 생성·분석 엔진 (Content Generation & Analysis Engine)**
  - 9.1 개요 및 책임 범위
  - 9.2 입력 파라미터 스키마
  - 9.3 생성 파이프라인
  - 9.4 분석 파이프라인
  - 9.5 강사 검수 워크플로우
  - 9.6 콘텐츠 라이브러리 (Content Repository)
  - 9.7 미디어 라이브러리 (Media Library)
  - 9.8 버전 관리 및 재사용
  - 9.9 비용·성능 관리
  - 9.10 KPI 및 관찰성
  - 9.11 데이터 모델 (요약)
  - 9.12 API 엔드포인트
  - 9.13 리스크 및 완화
- **10. 모듈 시퀀싱 엔진 (Module Sequencing Engine)**
  - 10.1 개요 및 책임 범위
  - 10.1.1 실제 구현체 — `analyzer` 서버
  - 10.1.2 알고리즘 4단계
  - 10.2 입력 스키마
  - 10.3 필터링 알고리즘 (Module Filtering)
  - 10.4 시퀀싱 알고리즘 (Module Sequencing)
  - 10.5 시간 예산 조정
  - 10.6 강사 오버라이드 UI (§8.5 연동)
  - 10.7 출력 스키마: LessonPlan
  - 10.8 검증 및 품질 관리
  - 10.9 AI 호출 최적화
  - 10.10 버전 관리
  - 10.11 데이터 모델 (요약)
  - 10.12 API 엔드포인트
  - 10.13 KPI 및 관찰성
  - 10.14 리스크 및 완화
  - 10.15 Speaking 콘텐츠 생성 모드 (3종)
- **11. 튜터링 (Tutoring · 학생용)**
  - 11.0 상품 운영 모델
  - 11.1 서비스 개요
  - 11.2 레슨 진입 → 모듈 진행 → 완료 여정
  - 11.3 Pickle 상호작용 모델
  - 11.4 실시간 피드백 시스템
  - 11.5 성과카드 및 KPI 시각화
  - 11.6 Speaking 모듈 임베드 방식
  - 11.7 모바일/데스크탑 UI 적응
  - 11.8 나만의 수업 — 학생 자유 학습
  - 11.9 수업 운영 (수강 형태)
  - 11.10 수료 기준 (Completion Criteria)
  - 11.11 게이미피케이션 (Gamification)
  - 11.12 푸시·알림 (Push & Notifications)
  - 11.13 ❌ B2B 제휴 운영 모델
- **12. AI 튜터링 엔진 (Agent 아키텍처)**
  - 12.1 2단계 Agent 구조 개관
  - 12.2 CurriculumPlannerAgent
  - 12.3 ModuleOrchestratorAgent
  - 12.4 FeedbackGenerationAgent (텍스트 SSE)
  - 12.5 SpeakingConversationAgent (신설)
  - 12.6 프롬프트 엔지니어링
  - 12.7 컨텍스트 윈도우 관리
  - 12.8 에러 처리 · Fallback · Guardrail
  - 12.9 프롬프트 인젝션 방어
- **13. 통합 모듈 시스템 (Unified Module System)**
  - 13.1 마스터 모듈 레지스트리
  - 13.2 서비스 범위 분류 (Service Scope)
  - 13.3 스킬 분류
  - 13.4 역할 / 단계 분류 — 두 축 병행
  - 13.5 모듈 프로퍼티 스키마
  - 13.6 문항 유형 및 채점 방식
  - 13.7 네이밍 규칙·Variant·시퀀싱 그룹
  - 13.8 공통 규약 (Module Common Rules)
  - 13.9 호환성 매트릭스 (요약)
  - 13.10 상태 및 라이프사이클
  - 13.11 모듈 버전 관리 및 마이그레이션
  - 13.12 관련 산출물 (Companion Artifacts)
  - 13.13 미디어 라이브러리
- **14. Speaking Tutor**
  - 14.1 제품 개요 및 포지셔닝
  - 14.2 레벨 시스템 (1–18, 6그룹)
  - 14.3 Free Talking(FRT) 대화 루프 설계
  - 14.4 Pick-Speak Method — Speaking 학습 흐름 및 6 교수법 모듈
  - 14.5 레벨별 Adaptive Interaction 매트릭스
  - 14.6 KPI 측정 체계
  - 14.7 음성 인프라 (Speech Stack)
  - 14.8 API 호출 최적화 및 비용
  - 14.9 독립 앱 UX (Speaking 전용 UI/UX)
  - 14.10 통합 모듈 연동 (튜터링 레슨 내 삽입)
  - 14.11 구현 우선순위 및 일정
  - 14.12 리스크
- **15. 학습 진단·평가 — 레벨 테스트 체계**
  - 15.1 핵심 원칙
  - 15.2 Level Test (학습 시작 시 진단)
  - 15.3 Achievement Test (학습 종료 시 성취도 평가)
  - 15.4 KPI 통합 진단 엔진 — 개인 절대 수준 측정 (Picklass Proficiency Diagnostics, PPD)
  - 15.5 정밀 진단 (Advanced Diagnostics)
  - 15.6 전사 일원화 로드맵
  - 15.7 데이터 모델 (요약)
  - 15.8 운영 KPI
  - 15.9 리스크 및 완화
- **16. B2B 제휴 운영 모델 (Partner Operating Model)**
  - 16.0 핵심 가치 — 성과 데이터 운영 지원
  - 16.1 운영 책임 분리 원칙
  - 16.2 책임 분리 매트릭스
  - 16.3 인터페이스 (제휴사 ↔ 픽클래스)
  - 16.4 운영 데이터 흐름 (수료 판정 예시)
  - 16.5 진입 채널별 B2B 제휴 모드 적용
  - 16.6 회원 계정 연동 (Account Linking)
  - 16.7 서비스 인증 연동 (Service Authentication)
  - 16.8 상품 구성 ↔ 학습 운영 매핑
  - 16.9 파트너사 운영 도구 지원
  - 16.10 채널별 적용 상세
  - 16.11 데이터 모델 (요약)
  - 16.12 운영 KPI
  - 16.13 리스크 및 완화
- **17. 핵심 도메인 정책**
  - 17.1 액세스코드 생성·활성화·만료 규칙
  - 17.2 요금제별 기본값 및 자동 채움
  - 17.3 비용 정책
  - 17.4 계약 상태 전환 정책
  - 17.5 데이터 검증 정책
- **18. 사용자 플로우 (통합)**
  - 18.1 비로그인 → 랜딩 → 가입
  - 18.2 학습자 여정 (읽기 중심 레슨)
  - 18.3 학습자 여정 (Speaking 독립 앱)
  - 18.4 강사 여정 (Studio)
  - 18.5 학원관리자 여정 (L2 InstitutionAdmin)
  - 18.6 본부·지주사 여정 (L1 GroupAdmin)
  - 18.7 파트너 여정 (L0 PartnerAdmin)
  - 18.8 학부모 여정 (Phase 1 — 속성 기반)
  - 18.9 시스템관리자 여정
- **19. 기술 아키텍처**
  - 19.1 Admin 스택
  - 19.2 Studio / www 스택
  - 19.3 Tutoring 스택
  - 19.4 Speaking Tutor 스택
  - 19.5 API 엔드포인트 맵
  - 19.6 실시간 인프라
  - 19.7 비동기 태스크 시스템
  - 19.8 배포 및 CI/CD
- **20. 로드맵**
  - 20.1 Phase 1 (~2026 Q2)
  - 20.2 Phase 2 (2026 Q3)
  - 20.3 Phase 3 (2026 Q4+)
  - 20.4 마일스톤 및 의존성
  - 20.5 회의록 260413 기반 고도화 트랙
- **21. 리스크 및 완화 전략**
  - 21.1 기술 리스크
  - 21.2 비즈니스/교육 리스크
  - 21.3 운영 리스크
- **22. 보안 및 컴플라이언스**
  - 22.1 인증 및 세션 관리
  - 22.2 데이터 보존 및 삭제
  - 22.3 개인정보 처리 및 이용약관
  - 22.3.1 인덱싱 및 공개성 정책
  - 22.4 감사 로그
  - 22.5 외부 LLM 보안 검토
- **23. 부록**
  - 23.1 화면 와이어프레임 / 목업
  - 23.2 디자인 가이드 요약
  - 23.3 파트너(파고다) 계약 마일스톤 요약
  - 23.4 참고 문서 인덱스
  - 23.5 변경 이력 및 승인

---

## 0. 문서 상태 표기 범례 *(v0.10 도입, v0.11 갱신)*

이 문서의 각 섹션·항목에는 **개발 진척도 마커**가 붙는다.  
v0.10은 "기획" vs "실제 구현" 정합성 보고가 목적이었고, v0.11은 그 위에 **회의록 260413 기반 신규 기획 8개 항목**(상품 패키지·콘텐츠 생성 방식·수업 운영·수료 기준·레벨 테스트·게이미피케이션·푸시 알림·미디어 라이브러리)을 추가한다.

| 마커 | 의미 |
|---|---|
| ✅ **구현 완료** | 실제 코드·DB·운영 환경에 반영되어 동작 중 |
| 🟢 **MVP** | 일부 기능 구현, 운영 검증 진행 중 |
| 📋 **기획 단계** | 문서·설계만 정의, 미구현 |
| ⚠️ **재정의** | 초기 기획과 실제 구현이 일부 달라 갱신됨 |
| ❌ **폐기** | 구버전 기획, 현재 시스템에서 제거됨 |

마커가 표기되지 않은 항목은 별도 검증 전이므로 임시로 "📋 기획 단계"로 간주한다.

> 📎 **참고 자료**: 본 v0.10은 다음 개발 docs를 참조함 — `picklass-backoffice/docs/ai-modules/20260426_AI모듈필드재분류_완료.md`, `picklass-backoffice/docs/users/20260424_사용자일괄생성.md`, `studio.picklass.com3/docs/20260421_지능형 수업 설계 자동화 로직.md`, `studio.picklass.com3/docs/20260422_KPI기반_수업설정_과정목표_개선.md`, `tutoring.picklass.com/docs/나만의수업_시퀀싱통합_20260425.md`, `tutoring.picklass.com/docs/20260428_questionFlowMode_재정의_개발완료.md`, `tutoring.picklass.com/docs/외부API_명세서_20260409.md`. v0.11은 추가로 `요청사항/AI 회의록_260413.docx`(파고다 합동 회의록, 2026-04-13)를 반영한다.

---

## 1. 문서 개요

### 1.1 문서 목적과 범위

본 문서는 Picklass(ClassSnap) 플랫폼의 **전체 서비스 구조·비즈니스 모델·핵심 도메인·AI 엔진·기술 아키텍처**를 통합적으로 정리한 기획서이다. 폴더 내에 흩어져 있던 다음 자료들을 단일 프레임에 통합한다.

- 백오피스(Admin) 문서군 (`picklass-backoffice/docs/*`)
- 스튜디오(Studio) 문서군 (`studio.picklass.com3/docs/*`)
- 튜터링(Tutoring) 기획 문서 (`tutoring.picklass.com/docs/picklass-tutoring-planning.md`)
- 스피킹 튜터 설계 자료 (`speaking.picklass.com/*`)
- 랜딩 페이지 및 공통 정책 (`www.picklass.com`)
- 파고다 개발 마일스톤 계약 및 스펙 PDF

본 문서는 **기획 관점의 통합본**으로, 개별 기능의 상세 개발 스펙은 각 서비스별 docs 폴더를 참조한다.

### 1.2 대상 독자

| 독자 | 주 활용 장 |
|---|---|
| 경영진 / 투자자 | §2, §5, §15, §20, §21 |
| 프로덕트 매니저 | §3 ~ §17 |
| 개발 / AI 엔지니어 | §9, §10, §12, §14, §15, §18, §19 |
| 디자이너 | §6, §8, §11, §14.9 |
| 파트너사(파고다 등) | §2, §5, §20.4, §23.3 |

### 1.3 용어 정의 / 약어표

| 용어 | 정의 |
|---|---|
| **LessonPlan** | 지문 1개 + 모듈 시퀀스로 구성된 학습 계획 |
| **PlannedModule** | LessonPlan 내 개별 모듈 슬롯 |
| **ModuleData / ModuleResult** | 실제 학습 콘텐츠 / 모듈 완료 결과 |
| **OrchestratorContext / ToolCall** | Agent 입력 컨텍스트 / 출력(도구 호출 결정) |
| **PassageAnalysis** | CurriculumPlannerAgent의 지문 분석 결과 |
| **ModulePedagogyProfile** | 모듈의 교수법 특성(채점/피드백/지문노출 모드) |
| **ContentConfig** | UI 배치 정책 |
| **KpiResult** | 모듈별 성과 지표 |
| **STT / TTS** | Speech-to-Text / Text-to-Speech |
| **WPM** | Words Per Minute (발화 속도) |
| **SRS** | Spaced Repetition System (간격 반복 학습) |
| **CEFR** | Common European Framework of Reference (A1~C2) |
| **Tool Use** | Anthropic Claude API의 구조화 출력 기능 |
| **SSE** | Server-Sent Events (스트리밍 방식) |
| **Guardrail** | Agent 출력 유효성 검증 레이어 |
| **Rolling Window** | 최근 N개 메시지만 유지하는 컨텍스트 관리 전략 |
| **TTFT** | Time To First Token (스트리밍 첫 토큰 지연) |

---

## 2. 서비스 개요 (Executive Summary)

### 2.1 제품 비전

> **"AI 기반 영어 4기능 통합 교육 플랫폼 — ClassSnap(Picklass)"**

Picklass는 Reading · Writing · Listening · Speaking 4기능을 아우르는 B2B 중심 영어 교육 플랫폼이다. 검증된 교수법과 생성형 AI를 결합하여 학원/어학원/학교/기업교육 시장에 **"영어 수업 준비, 바로 딱!"** 경험을 제공한다.

### 2.2 핵심 가치 제안

1. **B2B 기관 중심** — 학원 관리자가 강사/학생을 일괄 관리하고, 액세스코드 배포를 통해 빠르게 온보딩
2. **AI 맞춤형 학습** — CurriculumPlannerAgent가 지문·목표·레벨을 보고 레슨을 자동 설계
3. **모듈식 교수법** — 지문 1개 + 교수법 모듈 N개 조합으로 유연한 레슨 구성
4. **음성 기반 대화 학습** — Speaking Tutor(Free-talking)를 통한 실시간 양방향 음성 세션
5. **데이터 기반 성장** — 모듈별 KPI / 레벨별 WPM / 5지표 스피킹 KPI로 정량적 진도 추적

### 2.3 시장 포지셔닝 및 차별점

| 구분 | 기존 LMS | 기존 AI 영어앱 | **Picklass** |
|---|---|---|---|
| 주 고객 | 기관 | 개인(B2C) | **기관(B2B) 우선 + SaaS 병행** |
| 콘텐츠 | 강사 제작 | 고정 커리큘럼 | **AI 자동 생성 + 강사 편집** |
| 평가 방식 | 객관식 위주 | 음성 평가 중심 | **4기능 통합 + 교수법 모듈** |
| 피드백 | 사후 리포트 | 즉시 채점 | **실시간 적응형(Orchestrator)** |

### 2.4 현재 상태

- **Reading 베타 운영 중** (지문 관리, 유창성 읽기, 전략적 읽기)
- **Writing / Listening / Speaking 순차 출시 예정**
- Speaking은 별도 독립 앱(speaking.picklass.com)으로도 운영 계획

### 2.5 ❌ 핵심 성과 지표 요약 *(v0.17 삭제 — §20 로드맵·§21 성공 지표로 일원화)*

> 본 절은 §20·§21로 통합 이관되어 v0.17에서 폐기되었다.

### 2.6 On-demand 콘텐츠 생성 원칙

> Picklass는 수업 콘텐츠(레슨 구성·문항·피드백·추천 주제)를 **사용자가 모듈에 진입하는 시점에 실시간으로 생성**하는 On-demand 방식을 원칙으로 한다. 지문 등록 시점에 레슨·문항을 미리 만들어 두는 선생성(Pre-generation) 방식은 채택하지 않는다.

| 항목 | 내용 |
|---|---|
| 생성 시점 | **모듈 진입 시점** (레슨·수업 진행 중) |
| 선생성 여부 | ❌ 없음 — 지문 등록 시 레슨·문항·피드백을 미리 만들지 않음 |
| 적용 위치 | Tutoring Lesson Player, Studio 수업 미리보기, Speaking 세션 |
| 목적 | **비용 효율화** (사용되지 않는 콘텐츠 생성 낭비 제거) + **최신 컨텍스트 반영** (AI 모델 최신 버전·지문 수정사항·학습자 이력 즉시 반영) |
| 예외 | 지문 자체 텍스트(§9.3)와 레벨 분석 결과(§9.4)는 사전 분석 후 저장 — 모듈 단위 콘텐츠만 on-demand |

**이 원칙의 설계적 파급**
- **§9 콘텐츠 생성·분석 엔진**: 지문은 저장하되, 모듈별 활동 콘텐츠는 진입 시 생성
- **§10 모듈 시퀀싱 엔진**: LessonPlan은 저장하되, 개별 문항·피드백은 런타임 생성
- **§12 Orchestrator Agent**: 학습자 상태에 따라 매 상호작용마다 다음 콘텐츠 결정
- **§14 Speaking Tutor**: 3-Phase 대화 루프가 완전한 on-demand 구조로 동작

**실패 처리 (후속 과제)** — On-demand 생성 실패 시 폴백(캐시된 기본 콘텐츠 or 재시도 + 사용자 대기 UI) 정책은 후속 과제로 등록 (§20 로드맵 연동).

---

## 3. 전체 서비스 구조

### 3.1 서비스 아키텍처 개관 (도메인 · 채널)

Picklass는 **B2B 채널과 B2C 채널을 도메인으로 분리**한다. 서브도메인은 `admin.picklass.com` 외 신규 추가하지 않으며, studio.picklass.com은 **루트(/) = B2B 홍보 랜딩**, **/app = Studio 저작 앱**의 이중 용도로 운영한다.

| 구분 | 도메인 / 경로 | 역할 | 주 사용자 | 공개성 |
|---|---|---|---|---|
| **B2C 랜딩 · 학생 진입** | **www.picklass.com** | Phase 1: B2C 티저 + 학생 로그인·액세스코드 CTA<br>Phase 2: 풀 B2C 브랜드 랜딩 | 학생 · 학부모 · 개인 학습자 | 공개 · SEO 집중 |
| **B2B 홍보 랜딩** | **studio.picklass.com/** (루트) | 제휴 문의 · 데모 신청 · 사례 소개 · 기관장·강사 로그인 | 학원 관리자 · 강사 · 파트너 | 공개 · B2B 키워드 SEO |
| **Studio 저작 앱** | **studio.picklass.com/app/** | 지문/과정/레슨 제작, 모듈 편집, 학생 관리 | 강사 · 기관장 | 로그인 필수 · noindex |
| **Admin 백오피스** | **admin.picklass.com** | 기관/사용자/콘텐츠/요금제/AI모듈/액세스코드 중앙 관리 | 시스템·학원관리자 | 내부 · noindex |
| **튜터링** | **tutoring.picklass.com** | AI 튜터 Pickle과의 개인 맞춤 학습, 학생 로그인 화면 | 학생 | 로그인/코드 필수 · noindex |
| **스피킹 튜터** | **speaking.picklass.com** | 실시간 음성 대화 학습 (독립 앱 + 통합 모듈) | 학생 · 개인 구독자 | 로그인/코드 필수 · noindex |

**핵심 원칙**
- 학부모/학생에게 노출되는 공개 홍보 도메인은 **오직 www.picklass.com** 하나
- 학원·강사·파트너 대상 홍보는 **studio.picklass.com/** 루트에서만 진행
- Phase 1 초기에는 B2B 세일즈가 주력 → studio가 활성 홍보 채널, www는 티저 + 학생 진입점
- Phase 2 B2C 확장 시 www가 메인 브랜드 채널로 전환 (도메인 이전/리다이렉트 재작업 불필요)

### 3.2 서비스 간 관계도

**진입 채널이 B2C(www)와 B2B(studio 루트)로 이원화**된다. 학생 로그인은 www에 CTA만 두고 클릭 시 `tutoring.picklass.com/login`으로 리다이렉트한다(옵션 B).

```
┌──────────────────────────────────────────────────────────────────┐
│                        [진입 채널 분리]                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   B2C / 학생 · 학부모                   B2B / 기관 · 강사        │
│        │                                       │                 │
│        ▼                                       ▼                 │
│  ┌─────────────────────┐             ┌──────────────────────┐   │
│  │ www.picklass.com    │             │ studio.picklass.com/ │   │
│  │ Phase1: B2C 티저    │             │  (루트)              │   │
│  │       + 학생 CTA    │             │  B2B 홍보 랜딩       │   │
│  │ Phase2: B2C 주력 ★  │             │  제휴/데모 문의       │   │
│  │ [공개 SEO]          │             │  강사·기관장 로그인  │   │
│  └─────────┬───────────┘             └───────┬──────────────┘   │
│            │                                 │                   │
│            │ [학생 로그인] CTA 클릭           │ 로그인             │
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
│  │ speaking.picklass.com      │                         │
│  │  · 실시간 음성 대화 (Free-talking)  │                         │
│  │  · 통합 모듈 + 독립 앱 양쪽 지원    │                         │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  ┌─────────────────────────┐                                     │
│  │ admin.picklass.com      │   [시스템/학원 관리자]               │
│  │  · 기관/사용자/코드     │     ↑ studio에서 역할 확인 후        │
│  │  · 요금제/Billing       │       관리 권한으로 진입              │
│  │  · AI 모듈/시스템 설정  │                                     │
│  └─────────────────────────┘                                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**주요 플로우 요약**

- **학생**: www.picklass.com → [학생 로그인] 또는 [액세스코드 입장] → `tutoring.picklass.com/login` → 학습
- **강사/기관장**: studio.picklass.com/ 홍보 랜딩 → [로그인] → `studio.picklass.com/app`
- **시스템/학원 관리자**: studio 로그인 후 역할 확인 → `admin.picklass.com`
- **Speaking**: tutoring 내 모듈 호출 or `speaking.picklass.com` 직접 진입

### 3.3 Speaking Tutor 이중 운영 모델 *(+ 외부 임베드 보조 채널)*

> ⚠️ **재정의 (v0.10)**: 기획상 "이중 운영 모델"은 유지하되, 외부 시스템(예: 파고다SCS) 연동을 위한 **외부 임베드 채널이 이미 구현·운영 중**임을 명시. 별도 신규 채널이 아닌 기존 외부 API의 재사용 형태.

| 운영 모드 | 진입 경로 | 세션 단위 | 데이터 저장 위치 | 상태 |
|---|---|---|---|---|
| **통합 모듈 (튜터링+스피킹)** | Tutoring 레슨 내 Speaking 모듈 슬롯 | LessonPlan의 PlannedModule | ModuleHistory에 호환 저장 | 📋 기획 단계 |
| **독립 앱 (스피킹 단독)** | speaking.picklass.com 직접 진입 | SpeakingSession (독립 엔터티) | SpeakingSession 테이블 | 📋 기획 단계 |
| **외부 임베드 (이미 구현)** | 외부 시스템이 `X-Access-Token` + `X-Module-Code`로 `GET /external/verify` 호출 → 임베드 위젯 또는 별도 도메인에서 학습 진행 | 외부 시스템이 발급한 lessonId 기반 세션 | 외부 시스템 + 픽클래스 자체 학습 로그 | ✅ **구현 완료** (`SNR`, `FRT` 모듈 대상) |

**3가지 모드 모두** 동일한 SpeakingConversationAgent와 음성 인프라(STT/TTS)를 사용하되, **진입점/UI/인증/설정 경로/과금/리포트가 분기**된다.

> 📎 외부 임베드 케이스의 상세 인증 메커니즘과 API 응답 스키마는 **§14.10.4 ONETIME TOKEN + 외부 verify** 참조.

#### 3.3.1 통합 모듈 플로우 (튜터링 → 스피킹)

학생이 Tutoring에 로그인한 상태에서 Speaking 모듈로 진입하는 경우, 인증·레벨·콘텐츠 설정이 **Tutoring에서 자동 전달**되어 학생이 별도 입력할 필요가 없다.

**사용자 여정**
```
[튜터링 로그인]
    ↓
[수강(모듈) 목록]
    ↓
[스피킹 선택]
    ↓
[새 세션 시작 화면]
  · 이름 (GET_AUTH API 기본값, 수정 가능)
  · 마이크 선택·테스트
  · 튜터(Pickle 등) 선택
    ↓
[세션 시작]  ← 이 시점에 ONETIME TOKEN 소멸
    ↓
[결과 화면]
    ↓
[수강(모듈) 목록]  (원래 화면으로 복귀)
```

**핵심 인증·연동 메커니즘**

| 메커니즘 | 설명 |
|---|---|
| **ONETIME TOKEN** | Tutoring에서 Speaking 서비스로 페이지 이동 시 **일회용 토큰을 생성**하여 쿼리스트링 형태로 전달. 브라우저 이력·북마크로 재진입 불가하도록 보장. |
| **GET_AUTH API** | Speaking 서비스가 ONETIME TOKEN을 키로 호출하는 인증·컨텍스트 조회 API. 응답 필드: **아이디, 이름, 인증코드, 레벨, 콘텐츠, 대화 분량, 음성 모드** |
| **이름 필드 정책** | GET_AUTH 응답의 이름(기관에서 제공 가능)을 기본 설정하고, 세션 시작 화면에서 **사용자가 수정 가능**하도록 노출 |
| **토큰 소멸** | 세션 시작 시점에 ONETIME TOKEN 소멸. **소멸 API**가 Speaking 서비스 → Tutoring 방향으로 호출되어 토큰 상태를 무효화 |

**API 인터페이스 요약**
```
Tutoring → Speaking:
  GET https://speaking.picklass.com/?token={ONETIME_TOKEN}

Speaking → Tutoring:
  POST /api/auth/get-auth        (payload: { token })
       응답: { userId, name, accessCode, level, content, dialogLength, voiceMode }

  POST /api/auth/revoke-token    (payload: { token })
       세션 시작 시 호출, 토큰 소멸
```

#### 3.3.2 독립 앱 플로우 (스피킹 단독)

사용자가 Tutoring을 거치지 않고 Speaking 서비스에 **직접 진입**하는 경우. 스피킹 단독 앱은 **두 가지 경로**로 새 세션을 시작할 수 있다.

**(공통) 로그인 직후 화면 구조**

```
[스피킹 아이디 로그인]
    ↓
┌────────────────────────────────────────────────┐
│ 대시보드                                        │
│  ├─ 진행중인 세션 목록 (이어하기)              │
│  ├─ 수강 중인 과정 (My Courses)                │
│  │     └─ 레슨 선택 → 경로 A 또는 B 진입       │
│  └─ 수강할 수 있는 과정 (Available Courses)    │
│        └─ [인증코드 입력] → 수강 목록에 추가    │
└────────────────────────────────────────────────┘
```

##### 경로 A. 사전 세팅 과정 기반 수업

과정 제작자(기관·강사)가 **레벨·콘텐츠·대화 분량·음성 모드**를 사전에 지정한 레슨을 수강하는 경우. 사용자는 세션 개시에 관련된 최소 항목만 선택한다.

```
[수강 중인 과정] → [레슨 선택]
    ↓
[새 세션 시작 화면]
  · 이름 / 마이크 / 튜터 선택
  · 레벨·콘텐츠·대화 분량·음성 모드 = 과정 사전 설정값 (수정 불가)
    ↓
[세션 시작]
    ↓
[결과 화면]
```

**특징**
- 사용자가 전체 설정을 직접 하지 않음 — 과정 제작자의 의도가 반영됨
- 동일 과정 내 여러 레슨을 순차 수강 가능 (학습 곡선 설계)
- **인증코드는 "과정 단위"로 발급·사용**된다 — 인증코드 1개 = 과정 1개 수강권. 수강할 수 있는 과정에서 인증코드 입력 시 해당 과정이 "수강 중인 과정"으로 승격되며, 이후 과정 내 모든 레슨은 추가 인증 없이 수강 가능

##### 경로 B. 개인 관심사 기반 수업 (플랜 옵션)

사용자의 **현재 플랜에 "개인 관심사 수업" 상품 옵션이 포함된 경우**에만 활성화된다. 이 옵션은 **Admin이 플랜 레벨에서 상품 구성 옵션으로 일괄 관리**한다 (§7 참조). 사용자는 관심 주제를 직접 선택하여 자유 대화 세션을 생성하되, 레벨·음성 모드·대화 분량 등 학습 파라미터는 과정 기본값을 따른다.

```
[수강 중인 과정] (개인 관심사 수업 옵션 노출)
    ↓
[인증]  ← 수강 자격 확인 (플랜 권한)
    ↓
[관심 영역 등록]
  · AI 자동 생성 추천 주제 리스트 (최신 뉴스·이슈·트렌드 기반)
  · 사용자 자유 입력도 허용
    ↓
[새 세션 시작 화면]
  · 이름 / 마이크 / 튜터 선택
  · 레벨·음성 모드·대화 분량 = 과정 기본값 (수정 불가)
    ↓
[세션 시작]
    ↓
[결과 화면]
```

**특징**
- 과정에 귀속되지만 **주제는 사용자가 선택** — 흥미 유지·장기 학습 동기 부여
- **레벨·음성 모드·대화 분량은 과정 기본값으로 고정** — 경로 A와 학습 난이도·경험 일관성 유지
- **추천 주제는 AI가 최신 뉴스·이슈·트렌드를 실시간 수집하여 자동 생성** (일정 주기 갱신 및 캐시)
  - 데이터 소스: 뉴스 API, 소셜 트렌드, 공개 시사 이슈
  - 필터: 연령·레벨·과정 맥락에 맞는 주제로 정제 (성인/미성년 안전 분류 포함)
- 플랜에 옵션이 없는 사용자에게는 경로 B 진입점(UI) 자체가 노출되지 않음

##### 공통 특징 (경로 A·B)

- 자체 로그인 · 인증코드 · 세션 히스토리 모두 Speaking 서비스 내에서 관리
- ONETIME TOKEN / GET_AUTH API **사용하지 않음** (Tutoring 연동 없음)
- 세션 종료 후 Tutoring 수강 목록으로 돌아가지 않음 (Speaking 대시보드로 복귀)
- 과금·리포트는 Speaking 구독 플랜 기준 (§5.3 연동)
- "진행중인 세션 목록"에서 미완료 세션을 이어서 진행 가능

##### 과정 · 레슨 · 세션 계층 구조 (Speaking 독립)

```
Plan (플랜 · Admin에서 상품 구성)
 │   └─ [개인 관심사 수업] 옵션 (플랜 레벨 토글)         ← 경로 B 활성화 조건
 │
 └─ Course (과정 · 인증코드 1개로 수강권 부여)
     ├─ Lesson 1 (레벨/콘텐츠/분량/음성모드 사전 설정)    ← 경로 A
     ├─ Lesson 2
     └─ (플랜 옵션 ON 시) 개인 관심사 수업 진입점         ← 경로 B
           └─ AI 추천 주제 선택 시 Session 생성
```

> 📎 "개인 관심사 수업" 옵션은 **Admin에서 플랜(상품) 레벨 토글로 관리**된다 (§7 참조). 해당 옵션이 포함된 플랜을 보유한 사용자에게만 경로 B UI가 노출된다.

#### 3.3.3 세 가지 플로우 + B2B 제휴 모드 대조 *(v0.16 보강)*

독립 앱의 두 경로(A/B)를 통합 모듈과 함께 비교하며, **B2B 제휴 모드**(§16)는 별도 운영 형태로 모든 플로우 위에 적용 가능한 메타 모드이다.

> 🔁 **v0.16 — B2B 제휴 모드 도입**: 위 3개 플로우는 픽클래스 자체 운영 기준이다. **B2B 제휴 모드**가 적용되면 (a) 진입은 제휴사 사이트에서 시작 (b) 인증은 외부 토큰(`X-Access-Token` + `X-Module-Code`, §14.10.5) (c) 운영(상품·과금·수료)은 제휴사 자체 기획 (d) 수업 진행만 픽클래스 임베드 실행. 상세는 **§16** 참조.

| 항목 | 통합 모듈 (튜터링+스피킹) | 독립 앱 — 경로 A (과정 기반) | 독립 앱 — 경로 B (개인 관심사) |
|---|---|---|---|
| 진입 경로 | 튜터링 로그인 → 수강 목록 → 스피킹 선택 | 스피킹 로그인 → 수강 중인 과정 → 레슨 선택 | 스피킹 로그인 → 수강 중인 과정(개인관심사 UI 노출) → 인증 → 관심 영역 등록 |
| 인증 방식 | ONETIME TOKEN + GET_AUTH API | 스피킹 자체 아이디 + 인증코드 (**과정 단위**) | 스피킹 자체 아이디 + 수강 자격 확인 |
| 사전 설정 (레벨/콘텐츠/분량/음성모드) | Tutoring이 GET_AUTH로 전달 | **과정 제작자가 사전 지정** (수정 불가) | **과정 기본값으로 고정** (사용자 수정 불가) |
| 사용자 선택 항목 | 이름 · 마이크 · 튜터 (+ Tutoring 기본값 수정) | 이름 · 마이크 · 튜터 | **관심 주제(AI 추천)** · 이름 · 마이크 · 튜터 |
| 이름 필드 | GET_AUTH 제공값 기본 + 수정 가능 | 사용자 직접 입력 (프로필 값 기본) | 사용자 직접 입력 |
| 세션 종료 후 복귀 | 튜터링 수강 목록 | Speaking 대시보드 (수강 중인 과정) | Speaking 대시보드 |
| 세션 저장 | ModuleHistory (통합) + SpeakingSession 연결 | SpeakingSession (과정·레슨 FK 포함) | SpeakingSession (과정 FK + 관심 주제) |
| 옵션 노출 조건 | Tutoring 레슨에 Speaking 모듈 슬롯 존재 | 기본 경로 | **사용자 플랜에 "개인 관심사 수업" 상품 옵션 포함** |
| 옵션 관리 주체 | Studio 강사 (레슨 편집) | Studio 강사 (과정 편집) | **Admin (플랜 레벨 상품 구성)** |
| 과금 | 기관 플랜 번들 또는 Tutoring 구독 | Speaking 구독/**과정 단위 과금** (인증코드 1건 = 과정 1건) | Speaking 구독 중 옵션 포함 플랜 이용 |

> 📎 통합 모듈의 상세 데이터 흐름과 UX 구성은 **§14.10 통합 모듈 연동**, 독립 앱 UX는 **§14.9 독립 앱 UX**를 참조한다. "개인 관심사 수업" 옵션은 **§7 Admin의 플랜 상품 구성**에서 관리되며, 과정 단위 인증코드 정책은 **§7.5 액세스코드**의 Speaking 확장 항목을 참조한다.

#### 3.3.4 1:1 회화 채널 (파고다 외부 시스템 연계) *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 파고다 합동 회의(2026-04-13)에서 확정된 채널 운영 정책. 외부 임베드 채널의 **상위 운영 형태**를 정의한다.
>
> 📎 **v0.19 cross-ref**: 본 §3.3.4는 **§16. B2B 제휴 운영 모델**의 **첫 번째 표준 사례**이다. 운영 책임 분리·인증·상품 매핑·운영 도구 등 일반화된 모델은 §16 참조, 1:1 회화 채널 특수 정책(앱 설치 선택권·전환경 지원·라인업 등)은 본 절에서 상세 다룬다. §16.10.3에서 본 절을 cross-ref로 인용한다.

회의록 결정 사항을 다음 4개 정책으로 정리한다.

**① 채널 명칭 변경**

- 파고다 측 사이트(전화외국어 도메인)의 상단 서비스명을 **"전화외국어"** → **"1:1 회화"** 로 변경.
- 픽클래스 측 외부 임베드 채널은 이 1:1 회화 사이트의 **AI 상품 라인업** 위치에 배치된다.

**② 상품 라인업 구성**

| 라인업 구분 | 운영 주체 | 픽클래스 연관 |
|---|---|---|
| 전화 회화 | 파고다 강사 매칭 | 없음 |
| 화상 회화 | 파고다 강사 매칭 | 없음 |
| **AI 회화 (신규)** | 픽클래스 임베드 | ✅ §3.3 외부 임베드 모드, §14.10.5 GET /external/verify |

AI 회화 라인업의 학습 모듈은 픽클래스 측에서 **`SNR`(Speaking N-Reply, 본 대화 학습) · `FRT`(Free Talking, 자유 대화)** 기준으로 임베드되며, 외부 시스템이 발급한 액세스 토큰으로 인증된다(§14.10.5).

**③ 플랫폼 제공 범위 — 전환경 지원**

| 환경 | 지원 여부 | 설명 |
|---|---|---|
| PC Web | 📋 기획 | 데스크탑 브라우저 임베드 |
| Mobile Web | 📋 기획 | 모바일 브라우저 임베드 |
| Native App (iOS/Android) | 📋 기획 | 파고다 앱 또는 1:1 회화 전용 앱 |

**④ 앱 설치 선택권 (Soft App-Install)**

학습자가 AI 회의실 입장 시 다음 분기를 갖는다.

```
[1:1 회화 사이트 → AI 회화 입장]
    ↓
[앱 설치 여부 감지]
    ├─ 설치된 경우  → 앱으로 자동 연결 (Universal Link / Deep Link)
    └─ 미설치인 경우 → 다음 두 옵션을 학습자에게 제공
                       (a) 브라우저에서 계속 진행
                       (b) 앱 다운로드 후 참가
```

> ⚠️ **확인 필요**: 회의록 기록상 "오이지 측 확인 필요" — 앱 미설치 시 브라우저 계속 진행/앱 참가 분기 UX 구현 가능성 확인. 결과에 따라 (a)/(b) 분기를 자동/수동 모드로 운영.

**⑤ 피드백 페이지 정책 (참고)**

회의록에서 결정된 피드백 노출 정책 — 학습자 페이지(파고다 사이트 내 노출)에서 다음 기준으로 데이터를 표시한다.

| 표기 단위 | 기준 | 설명 |
|---|---|---|
| **일간 학습성취** | **레벨별 상대값** | 학습자 레벨 대비 상대 진척률 (예: A2 기준 110%) |
| **월간 학습성취** | **절대값** | 누적 발화량·시간·미션 완료 등 절대 수치 |

상세 수료 기준 및 학습자 페이지 노출 항목은 **§11.10 수료 기준** 참조.

**⑥ 상품 라인업·환경 지원 매트릭스**

| 상품 라인업 \\ 환경 | PC Web | Mobile Web | App | 인증 방식 |
|---|---|---|---|---|
| 전화 회화 | ⭕ | ⭕ | ⭕ | 파고다 자체 |
| 화상 회화 | ⭕ | ⭕ | ⭕ | 파고다 자체 |
| **AI 회화 (픽클래스 임베드)** | 📋 | 📋 | 📋 | **`X-Access-Token` + `X-Module-Code`** (§14.10.5) |

> 📎 외부 임베드 인증 메커니즘은 **§14.10.5 외부 시스템 임베드 GET /external/verify**, 콘텐츠 생성 방식 3종(교재/주제+레벨/프리토킹)은 **§10.5 Speaking 콘텐츠 생성 모드** 참조.

### 3.4 도메인 맵

- **Identity**: User, Institution, AccessCode, Role, Session
- **Content**: Passage, Lesson, Course, Module(Reading/Writing/Speaking)
- **Learning**: ModuleHistory, LessonResult, KpiResult, SpeakingSession/Turn
- **Commerce**: Plan, Subscription, Invoice, Payment, AIUsage
- **System**: AIModule, Level(CEFR 18), Status, Duration

### 3.5 공통 인프라 및 도메인/SEO 정책

**공통 인프라**
- **인증**: 이메일 + 소셜 OAuth (Google / Kakao / Naver)
- **DB**: PostgreSQL (Backoffice/Tutoring), Supabase (Studio/www)
- **AI API**: Anthropic Claude(claude-haiku-4-5-20251001, Tool Use), Google GenAI
- **음성**: Azure Speech (TTS 우선), Google STT, 추후 Azure STT 대체 검토

**도메인/SEO 정책**

| 도메인 / 경로 | 인덱싱 | 타겟 키워드 유형 | 근거 |
|---|---|---|---|
| www.picklass.com | **index 허용** | B2C (AI 영어 튜터, 영어 회화 학습 등) | Phase 2 브랜드 주력 채널 |
| studio.picklass.com/ (루트) | **index 허용** | B2B (학원 AI 교재, 영어 학원 솔루션 등) | 파트너 유입 필요 |
| studio.picklass.com/app/** | **noindex** | - | 앱 화면 공개 차단 |
| admin.picklass.com | **noindex** | - | 관리자 전용, 공개 불필요 |
| tutoring.picklass.com | **noindex** | - | 학생 로그인/학습 전용 |
| speaking.picklass.com | **noindex** | - | 학생 로그인/학습 전용 |

**인증 쿠키 스코프**
- www ↔ tutoring/speaking 간 SSO가 필요한 경우 `.picklass.com` 상위 도메인 쿠키 사용
- studio ↔ admin 간 역할 검증 핸드오프는 서버 사이드 세션 또는 JWT 전달

**레거시 URL 리다이렉트 (Phase 1 전환)**
- 기존 www.picklass.com의 B2B 경로(`/partners`, `/demo`, `/pricing`, `/institution` 등) → `studio.picklass.com/*` 301 영구 이동
- www 루트(`/`)는 **리다이렉트하지 않음** — B2C 티저/학생 진입점으로 유지

---

## 4. Actor & 조직 모델

> **v0.6 전면 재편:** B2B 고객을 학원뿐 아니라 기업·학교까지 포괄하기 위해, 조직 계층을 **3-Tier 고정 구조**로 정의하고 Person Actor를 **8종**으로 재정의한다. 계정은 **단일 User + 복수 Membership** 모델로 통일한다.

### 4.1 Actor Taxonomy 개관

Picklass의 사용자 체계는 **4개 축**으로 정의된다.

| 축 | 분류 | 예시 |
|---|---|---|
| **A. Entity vs Person** | 조직(엔터티) / 사람(역할) | Institution = Entity / 강사 = Person |
| **B. 조직 계층 (Depth)** | Partner(L0) / Group(L1) / Institution(L2) — 3단 고정 | 파트너사 → 프랜차이즈 본부 → 가맹학원 |
| **C. 채널** | B2B (파트너/본부/기관) / B2C (개인) | 기관 소속 학생 / 개인 구독자 |
| **D. 역할(Role)** | 8종 Person Role | 시스템관리자 ~ Consumer |

### 4.2 조직 계층 (Organization Hierarchy)

#### 4.2.1 3-Tier 고정 구조

조직은 최대 3단 고정 계층으로 구성된다. 4단 이상 세분화가 필요한 경우 Institution 내부의 논리적 분류(팀·반·부서)로 처리하며, 조직 엔터티 자체는 확장하지 않는다.

```
[L0] Partner (파트너사)           ◇ 선택적
  │   여러 Group 또는 Institution 횡단 관리
  │
  ├─[L1] Group (본부 / 지주사)    ◇ 선택적
  │    산하 Institution 등록·플랜 배분·통계
  │
  └─[L2] Institution (학원/계열사/학교) ■ 필수
       실제 수업·결제·액세스코드 발급 단위
```

#### 4.2.2 필수/선택 레벨 규칙

| 고객 유형 | L0 Partner | L1 Group | L2 Institution |
|---|---|---|---|
| 단독 소규모 학원 | — | — | ■ |
| 프랜차이즈 학원 | 선택 | ■ | ■ |
| 단일 법인 기업 | — | — | ■ |
| 대기업 그룹 | 선택 | ■ | ■ |
| 파트너 중개 (단일 Institution) | ■ | — | ■ |
| 파트너 중개 (복수 Group) | ■ | ■ | ■ |

**핵심 원칙**
- **Institution(L2)은 항상 필수** — 실제 학습·결제·액세스코드 발급의 단위
- Partner는 **Group을 건너뛰고 Institution 직접 관리 가능** (작은 단일 계약을 위한 유연성)
- Partner 단일로는 운영 불가 — 반드시 산하에 Group 또는 Institution 1개 이상

#### 4.2.3 Sector별 매핑 (Academy / Enterprise / K12)

Institution.sector 속성에 따라 **UI 레이블이 동적 전환**된다. 내부 DB 필드와 코드 상수는 영어 기반 일관 유지.

| 내부 개념 | Academy (학원) | Enterprise (기업) | K12 (학교) |
|---|---|---|---|
| **L1 Group** | 프랜차이즈 본부 | 지주사 / 본사 | 교육청 / 재단 |
| **L2 Institution** | 가맹학원 (단독 학원 포함) | 계열사 / 지사 | 학교 |
| **Group Admin** | 본부 관리자 | **HR 총괄** | 교육청 관리자 |
| **Institution Admin (공통 용어)** | **학원관리자** | HR 담당자 | 학교 관리자 |
| **Instructor** | 강사 | 사내강사 · 외부강사 | 교사 |
| **Learner** | 학생 | 임직원 | 학생 |
| **Guardian** | 학부모 | (필요 시 사용 — 가족돌봄 교육 등) | 학부모 |

> **용어 확정(v0.6)**
> - "기관장" 용어 폐기 → **"학원관리자"로 통일** (내부 코드: `InstitutionAdmin`)
> - L1 Group Admin의 기업 UI 기본값 = **"HR 총괄"**
> - Guardian은 주로 Academy·K12용이며, Enterprise에서도 필요 시 사용 허용

#### 4.2.4 Sector 속성과 UI 레이블 동적 전환

```typescript
enum Sector {
  ACADEMY = 'academy',
  ENTERPRISE = 'enterprise',
  K12 = 'k12',
}

// UI 레이블 매핑 예시 (i18n 상수)
const LABELS = {
  academy:    { L1: '프랜차이즈 본부', L2: '가맹학원', learner: '학생', instructor: '강사' },
  enterprise: { L1: '지주사',         L2: '계열사',   learner: '임직원', instructor: '사내강사' },
  k12:        { L1: '교육청',         L2: '학교',     learner: '학생',  instructor: '교사' },
};
```

Institution.sector는 L2에서 지정하며, 상위 Group의 sector와 불일치할 경우 경고(혼합 섹터 운영은 허용하되 UI 통일을 위해 Group sector가 기본값).

### 4.3 Organization Entities 정의

| Entity | 계층 | 계정 | 필수 여부 | 주 관리 Role |
|---|---|---|---|---|
| **Platform** | — | 시스템 기본 | — | 시스템관리자 |
| **Partner** | L0 | ❌ (대표자가 role 보유) | 선택 | Partner Admin |
| **Group** | L1 | ❌ | 선택 | Group Admin |
| **Institution** | L2 | ❌ | **필수** | 학원관리자 |
| **Plan / Product** | — | ❌ | 기본 제공 | 시스템관리자 (§7.7.1) |

**Organization 테이블 통합 모델 (§16 연동)**

```typescript
interface Organization {
  id: string;
  level: 'Partner' | 'Group' | 'Institution';
  parentId: string | null;           // 상위 조직 참조
  name: string;
  sector?: Sector;                   // Institution에서 필수, Group은 옵션
  status: 'active' | 'inactive' | 'suspended' | 'withdrawn';
  contractInfo?: object;
  createdAt: Date;
}
```

계층 무결성: Partner → Group → Institution 순서만 허용. Partner의 parentId는 null, Group의 parentId는 Partner 또는 null, Institution의 parentId는 Group/Partner/null 중 하나.

### 4.4 Person Actors 정의 (8종) *(v0.17 보강 — B2B 제휴 데이터 처리 + L2+ 이메일 정책)*

| # | Role (내부 코드) | UI 명칭 (Academy / Enterprise / K12) | 주 소속 | 핵심 책임 |
|---|---|---|---|---|
| 1 | **PlatformAdmin** | 시스템관리자 | Platform | 전체 운영, 감사 로그 |
| 2 | **PartnerAdmin** | 파트너 담당자 | Partner (L0) | 산하 Group/Institution 계약 |
| 3 | **GroupAdmin** | 본부 관리자 / **HR 총괄** / 교육청 관리자 | Group (L1) | 산하 Institution 등록, 플랜 배분, 통계 |
| 4 | **InstitutionAdmin** | **학원관리자** / HR 담당자 / 학교 관리자 | Institution (L2) | 강사·학습자 초대, 액세스코드 발급, 수강 관리 |
| 5 | **Instructor** | 강사 / 사내강사 / 교사 | Institution (L2) | 과정·레슨 제작, 배정 학습자 모니터링 |
| 6 | **Learner** | 학생 / 임직원 | Institution (B2B) or — (B2C) | 학습 진행 |
| 7 | **Guardian** | 학부모 | Learner 계정 연결 | 자녀 진도 조회, 결제 — **Phase 1은 속성만** |
| 8 | **Consumer** | 개인 구독자 | 없음 (B2C) | 개인 학습, 구독 결제 |

#### 4.4.1 B2B 제휴 데이터 처리 정책 *(v0.17 신설)*

B2B 제휴 모드(§16)에서는 **파트너사가 자체 회원 시스템을 운영**하므로, picklass는 학습 진행에 필요한 최소 데이터만 외부에서 수신·저장한다.

| 항목 | 정책 |
|---|---|
| 데이터 흐름 | **파트너사 → picklass** 단방향 (학습 진입 시점에 외부 토큰과 함께 전달) |
| 저장 범위 | **학습 진행에 필요한 항목만 저장** — 학생 식별자(외부 ID), 표시명, 레벨, 수강 과정 매핑. 결제·연락처·민감 정보는 저장 X |
| 동기화 주기 | 토큰 verify 시점(§14.10.5) 또는 파트너사가 명시적으로 푸시할 때만 갱신 |
| 데이터 보존 | 파트너 계약 종료 후 **30일 내 자동 삭제** (학습 로그는 익명화 후 보존) |
| 회원 인증 | picklass 자체 비밀번호 미발급 — **외부 토큰 기반 SSO** (§14.10.5 GET /external/verify) |

#### 4.4.2 이메일 아이디 정책 *(v0.17 신설)*

| 대상 | 정책 |
|---|---|
| **Institution(L2) 이상 관리자** (PlatformAdmin / PartnerAdmin / GroupAdmin / InstitutionAdmin / Instructor) | **확인된 이메일 아이디 사용 필수** — 회원가입 시 이메일 인증 완료해야 활성화. 이메일이 곧 로그인 ID 역할 |
| Learner (B2B) | 기관 발급 액세스코드 + 임의 ID(이메일 미인증 허용) |
| Learner (B2C / Consumer) | 이메일 인증 권장 (필수는 아님, 단 결제 발생 시 인증 강제) |
| Guardian | Learner 속성으로 등록 시 이메일 미인증 허용. 독립 Membership 승격 시(Phase 2+) 인증 필수 |
| **B2B 제휴 진입자** | picklass 이메일 인증 불필요 — 파트너사 인증 신뢰 (§4.4.1) |

이 정책은 **운영 책임 격차**(관리자 = 신원 책임 高 / 학습자 = 학습 자유도 高)를 반영한다.

**Guardian 운영 규칙 (v0.6)**
- **Phase 1**: Learner 계정의 **보호자 속성**(이름·이메일·연락처)로 존재. 독립 로그인 없음.
- **Phase 2+**: 독립 Membership 도입, 자녀 진도 읽기 전용 + B2C 결제 주체 역할
- **Enterprise 섹터**: 기본 미사용이나 "가족돌봄 교육" 등 복지 프로그램 요구 시 활성화 (옵션)

### 4.5 Role 권한 매트릭스 (관리 Scope × 기능)

| 기능 \ Role | Platform | Partner | Group | 학원관리자 | Instructor | Learner | Guardian | Consumer |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Platform 설정 | ✅ | — | — | — | — | — | — | — |
| Partner 계약·수수료 | ✅ | 자기 Partner | — | — | — | — | — | — |
| Group 생성·편집 | ✅ | ✅ 산하 | 자기 Group | — | — | — | — | — |
| Institution 등록 | ✅ | ✅ 산하 | ✅ 산하 | 초기 세팅만 | — | — | — | — |
| 플랜 구독·결제 | ✅ | 산하 대리 | 산하 대리 | ✅ 자기 기관 | — | — | 자녀 대리 | ✅ |
| 액세스코드 발급 | ✅ | 산하 일괄 | 산하 일괄 | ✅ 자기 기관 | — | — | — | — |
| 강사·학습자 초대 | ✅ | 산하 | 산하 | ✅ 자기 기관 | — | — | — | — |
| 과정·레슨 제작 | 조회 | 조회 | 조회 | ✅ 자기 기관 | ✅ 배정 | — | — | — |
| 학습 진행 | — | — | — | 조회 | 조회(배정) | ✅ 본인 | 자녀 조회 | ✅ 본인 |
| 통계·리포트 | ✅ 전체 | 산하 | 산하 | ✅ 자기 기관 | 자기 담당 | 본인 | 자녀 | 본인 |

**권한 상속 원칙**: 상위 Organization Role은 하위의 모든 권한을 **위임 실행 가능**. 단, 감사 로그(§22.4)에 "위임 실행자" 기록 필수.

### 4.6 계정 모델 — 단일 User + 복수 Membership

한 사람 = 한 계정, 그 위에 **여러 역할 컨텍스트(Membership)**를 두는 모델을 채택한다.

```
User (alice@example.com)
  │
  ├─ Membership #1: Institution X에서 "Instructor"       ← 주 업무
  ├─ Membership #2: Institution Y에서 "Learner"         ← 교사 연수
  ├─ Membership #3: Partner P에서 "PartnerAdmin"        ← 파트너 담당자 겸직
  │
  ├─ GuardianLink #1: 자녀 Learner Z 연결                ← 보호자 역할
  │
  └─ Subscription #1: B2C "Consumer" (Speaking 단독 플랜)
```

**Membership 스키마 (개념)**

```typescript
interface Membership {
  id: string;
  userId: string;
  orgType: 'Partner' | 'Group' | 'Institution';
  orgId: string;
  role: 'PartnerAdmin' | 'GroupAdmin' | 'InstitutionAdmin'
      | 'Instructor' | 'Learner';
  status: 'active' | 'inactive' | 'suspended' | 'withdrawn';
  // seatSource: ❌ v0.17 폐기 — 청구는 활성 카운트 + 사용량 기반으로 단순화
  joinedAt: Date;
  revokedAt?: Date;
}
```

> ❌ **v0.17 삭제 — 좌석 배분(seatSource) 혼합 모델**: v0.6에서 채택했던 seatSource(`institution-paid` / `group-allocated` / `partner-allocated`) 분류 모델은 v0.17에서 폐기되었다. 좌석 단위 추적은 더 이상 핵심 청구 모델이 아니며, B2B 제휴(§16) 운영 모델에서는 제휴사가 자체 결제·과금을 관리하므로 picklass 측 seatSource 추적은 불필요하다. **Membership.seatSource 필드 폐기**, 청구는 **Membership 활성 카운트 + 사용량 기반**으로 단순화.

### 4.7 B2B/B2C 채널 분리 원칙

| 채널 | 포함 Actor | 진입 도메인 | 결제 주체 | 소속 Organization |
|---|---|---|---|---|
| B2B-파트너 | PartnerAdmin | admin (제한) | Partner 수수료·정산 | L0 Partner |
| B2B-본부/지주 | GroupAdmin | studio → admin (제한) | Group 일괄 or 산하 개별 | L1 Group |
| B2B-기관 | InstitutionAdmin / Instructor / Learner(B2B) / Guardian(B2B) | studio · www → tutoring | Institution | L2 Institution |
| B2C | Consumer / Guardian(B2C) | www → tutoring/speaking | 개인 | 없음 |
| Platform | PlatformAdmin | admin | — | Platform |

### 4.8 Organization/사용자 상태 체계

| 상태 | 코드 | User 의미 | Organization 의미 |
|---|---|---|---|
| 활성 | active | 정상 활동 | 정상 운영 |
| 휴회 | inactive | 일시적 비활성 (재활성 가능) | 일시적 운영 중단 |
| 정지 | suspended | 관리자 조치 정지 | 계약 위반·미납 등 임시 정지 |
| 탈퇴 | withdrawn | 영구 탈퇴 (복원 불가) | 계약 종료·해지 |

**상태 전환 규칙**
- 활성 ↔ 휴회/정지: 가역
- 탈퇴: 불가역 (데이터 아카이빙)
- Organization 상태 변경 시 **산하 Membership·하위 조직 일괄 영향** — 정책에 따라 연쇄 처리

### 4.9 Speaking 단독 사용자 시나리오

Speaking Tutor를 B2C 경로로 이용하는 Consumer의 주요 페르소나 4종 (기존 §4.6 유지):

1. **초급 학습자(A1–A2)** — 발화 습관 형성, 즉각 교정 중심
2. **중·고급 학습자(B1–C2)** — 논리 전개/유창성 개선
3. **시험 준비생** — 토익 스피킹, 오픽 대비 (Role-Play, QAR 집중)
4. **비즈니스 영어 학습자** — 업무 상황극·Key Expressions 중심

> B2B Learner(기관 소속)의 Speaking 이용은 §3.3.1(튜터링↔스피킹 연동) 및 §3.3.2 경로 A/B 참조.

### 4.10 경계 케이스 (Edge Cases)

| 케이스 | 처리 방식 |
|---|---|
| **강사의 자기 학습** | 같은 Institution에 Learner Membership 추가(활성 Membership 카운트에 반영) 또는 B2C Subscription 병행. Instructor role 단독으로는 학습 기록 미생성(미리보기 샌드박스만 제공) |
| **학부모 결제 (B2C)** | Guardian이 자녀 Consumer 계정을 대신 결제. Phase 1은 Consumer 계정 하위 "결제 담당자" 속성으로 처리, Phase 2에 Guardian 독립 Membership화 |
| **복수 소속 (강사+학생 겸직)** | 단일 User + 2개 Membership. 로그인 후 컨텍스트 선택 UI |
| **B2B→B2C 전환 (졸업 후 개인 학습)** | B2B Membership 만료 → Consumer Subscription 신규. 학습 이력 본인 계정에 계승 |
| **파트너가 복수 Group 관리** | 단일 PartnerAdmin 계정이 복수 L1 Group을 산하로 관리. 대시보드에서 Group 선택 후 드릴다운 |
| **파트너가 Group 없이 Institution 직접 관리** | Partner → Institution 직접 연결 (L1 생략). 중소형 단일 계약에 적용 |
| **Group의 좌석 일괄 배분** *(❌ v0.17 폐기 — Membership 활성 카운트 + 사용량 기반으로 단순화)* | 본부가 1,000석 구매 → 산하 10개 기관에 100석씩 자동/수동 배분 (개념 유지, seatSource 추적은 미사용) |
| **단독 소규모 학원** | L0/L1 없이 L2만 활성화. sector=`academy` |
| **단일 법인 기업교육** | L0/L1 없이 L2만 활성화. sector=`enterprise`, Guardian 미사용 |
| **대기업 그룹의 교차 계열사 이동** | User 동일, Membership 2개(이전 계열사 withdrawn, 신규 계열사 active). 학습 기록 계승 |
| **교사 연수 (학원이 학원에 위탁)** | 위탁 학원(B)이 피연수 학원(A)의 강사를 학생으로 받음. 강사가 A의 Instructor + B의 Learner 2개 Membership 보유 |

---

## 5. 비즈니스 모델

### 5.1 B2B 3채널 모델 (Academy / Enterprise / K12)

**진입 채널: `studio.picklass.com/` (B2B 홍보 랜딩)**

B2B는 **파트너 / 본부·지주사 / 기관** 3개 층위에서 계약이 이뤄지며, 각 섹터(Academy·Enterprise·K12)에 따라 UI·과금 모델이 분기된다.

| B2B 하위 채널 | Organization Level | 결제·좌석 | 주 의사결정자 |
|---|---|---|---|
| 파트너 중개 | L0 Partner → L1/L2 | 파트너 일괄 + 수수료 정산 | PartnerAdmin |
| 본부·지주 일괄 | L1 Group → L2 | Group 일괄 구매 후 산하 기관 이용 *(좌석 추적은 v0.17 폐기 — 활성 Membership 카운트 기준)* | GroupAdmin |
| 기관 개별 | L2 Institution 단독 | Institution 자체 예산 | 학원관리자 |



```
[파트너 학원/어학원]
     │
     │ 1. studio.picklass.com 접속 → 제휴/데모 문의
     ▼
[Picklass 세일즈] ─ 계약 체결 ─► [admin.picklass.com 기관 등록]
     │                               │
     │     ◄──── 액세스코드 발급 ─────┘
     │
     ├─► 강사: studio.picklass.com/ 로그인 → /app 저작 앱 → 7일 체험
     └─► 학생: 학원이 배포한 코드 → www.picklass.com의 [액세스코드 입장]
                → tutoring/speaking 접근
```

**핵심 특징:**
- Admin이 기관 등록 시 플랜·학생 수·계약 기간 확정
- 액세스코드 대량 생성 후 강사·학생에게 배포
- 결제는 기관 단위 구독 (월납/연납)
- **학부모에게 노출되는 공개 홍보 도메인은 www뿐** — B2B 홍보(studio 루트)는 학생·학부모가 직접 접할 이유가 없음

### 5.2 중소학원 SaaS 구독 모델

**진입 채널: `studio.picklass.com/` (B2B 홍보 랜딩에서 자유 가입)**

- 액세스코드 없이 studio 루트에서 **자유 가입** → `/app`으로 진입
- 7일 무료 체험 → Lite / Pro / Enterprise 중 선택
- 신용카드 직접 결제 (월간/연간)
- Phase 2부터 **개인 학습자(B2C) 자유 가입은 www.picklass.com**에서 별도로 운영

### 5.3 Speaking Tutor 단독 구독/부가 옵션 모델

| 옵션 | 설명 | 과금 방식 |
|---|---|---|
| **번들 포함** | Tutoring 플랜(Pro/Enterprise)에 Speaking 모듈 포함 | 플랜 월정액에 포함 |
| **단독 구독** | Speaking 전용 앱만 이용 | 월 구독 (개인 요금제) |
| **세션 추가** | 기본 한도 초과 시 | 분당/세션당 과금 |

### 5.4 무료 체험 정책

모델별로 무료 체험 제공 여부가 다르다. **B2B 제휴 모델은 체험 없이 액세스코드 발급으로 즉시 활성화**된다.

| 모델 | 체험 제공 | 상세 |
|---|---|---|
| **B2B 제휴 모델** | ❌ **없음** | 기관이 이미 계약 체결·결제 완료 상태. 액세스코드 발급으로 즉시 활성화. 코드 만료 시 재발급으로 이용 관리. |
| **SaaS 구독 모델 (강사)** | ✅ 7일 | Studio 자가 가입 시점 자동 활성. Studio 기본 강의 제작 기능 제한적 이용 가능. 체험 종료 후 신용카드 결제 필요(월간/연간). |
| **B2C 개인 구독 (Consumer)** | ✅ 7일 | www.picklass.com 자가 가입 시점 자동 활성. 무료 체험 과정만 접근 가능. 체험 종료 후 결제 필요. |

**공통 알림 정책 (SaaS·B2C)**
- 가입 직후 체험 시작 안내
- D-3: 이메일 · 앱 푸시
- 종료 당일 최종 안내
- 지속 이용 시 자동 결제 또는 대기 상태 전환

**환불 정책**
- 체험 기간 내 취소: 즉시 환불
- 결제 후 취소: 월간 7일, 연간 30일 환불 보장

> 📎 PDF 초안 v1.0(2026-03-05)에서 정의된 "B2B 제휴 체험 없음" 원칙을 v0.7에서 공식 채택하여, 기존 v0.6의 "B2B/SaaS 양쪽 7일 체험"을 수정한다.

### 5.5 요금제 구조

| 항목 | Starter | Pro | Enterprise | 제휴 |
|---|---|---|---|---|
| 월비용 | 100,000원 | 1,000,000원 | 협의 | 협의 |
| 연납할인 | 5% | 10% | 0% | - |
| 기본학생수 | 10명 | 50명 | 100명 | - |
| 학생당 단가 | 5,000원 | 10,000원 | 협의 | - |
| 최대학생수 | 50명 | 500명 | 무제한 | - |
| 관리계정수 | 2 | 5 | 10 | - |
| API 연동 | 미연동 | 연동 | 연동 | 협의 |
| API 비용 | 무료 | 30,000원 | 협의 | - |

### 5.6 Speaking 이용 시간/세션 기반 과금 고려사항

- **STT/TTS/LLM API 비용** 3요소가 세션당 실제 원가 구성
- 세션당 평균 비용 추정: ~$0.049 (통합 AI 처리 최적화 후)
- 과금 단위 후보: **세션 수**, **음성 시간(분)**, **LLM 토큰**
- 권장: 플랜별 월 세션 한도 → 초과 시 분당/세션당 과금

### 5.7 추가 과금 항목

- 기술지원비 (부가 서비스)
- 콘텐츠 IP 전환비 (기존 교재 → AI 모듈화 시 일회성)
- API 연동비 (Pro 전용 일회성)
- **음성 API 초과 과금** (Speaking 전용, 상기 5.6)

### 5.8 환불 및 해지 정책

- 체험 기간 내: 즉시 환불
- 결제 후: 7일 환불 보장(월간), 30일(연간)
- 계약 중도 해지: 잔여 기간 환산 환불 + 위약금(협의)

---

## 6. 랜딩 페이지 (B2C / B2B 분리 운영)

본 장은 **§6.A www (B2C 티저·학생 진입)**, **§6.B studio 루트 (B2B 홍보)**, **§6.C studio /app (Studio 저작 앱 진입)**, **§6.D 공통 정책**으로 나뉜다.

---

### 6.A www.picklass.com — B2C 티저 · 학생 진입

#### 6.A.1 Phase 1 구성 (현재)

- **목적**: B2C 브랜드 토대 구축 + 기관 소속 학생의 진입점
- **콘텐츠**: 최소 티저 수준 (오픈 예정 안내, 브랜드 스토리 요약)
- **노출 CTA**: 오직 2종
  1. **[학생 로그인]** 버튼 → 클릭 시 `tutoring.picklass.com/login`으로 리다이렉트
  2. **[액세스코드 입장]** 버튼 → 클릭 시 코드 입력 모달 → 검증 후 tutoring으로 이동
- **보조**: 정식 오픈 알림 이메일 수집 폼
- **푸터 안내**: "학원·기관 제휴 문의는 [studio.picklass.com](https://studio.picklass.com)"

```
[Hero]  "영어 실력이, 딱! 맞춰집니다"
        Picklass AI Tutor — 정식 오픈 준비 중

[CTA]   [ 학생 로그인 ]   [ 액세스코드 입장 ]

[Sub]   소속 학원에서 받은 액세스코드로 입장하세요.
        [정식 오픈 알림 받기]  (이메일)

[Footer] 학원·기관 제휴 문의 → studio.picklass.com
         회사 정보 · 약관 · 개인정보처리방침
```

#### 6.A.2 Phase 2 확장 (B2C 본격 런칭)

- 풀 B2C 마케팅 랜딩으로 확장
- 섹션: Hero / Why Picklass / 4기능 소개 / Speaking Tutor / 요금제 / FAQ / 블로그 연동
- 개인 구독자 직접 회원가입 CTA 추가

#### 6.A.3 학생 로그인 플로우 (옵션 B)

**www에는 로그인 폼을 직접 두지 않는다.** CTA 버튼만 배치하며, 실제 로그인은 tutoring 도메인에서 수행한다.

```
www.picklass.com
    │
    │ [학생 로그인] 클릭
    ▼
tutoring.picklass.com/login
    │
    │ 이메일/소셜 인증
    ▼
tutoring.picklass.com (학습 홈)
```

**채택 이유**
- 학부모 인식: www는 홍보 채널, tutoring이 "학습 공간"이라는 역할 분리 유지
- 보안/개발: 공유 쿠키·CORS 설계 부담 최소화
- Phase 2 전환 시 www 랜딩 재설계에 방해되지 않음

#### 6.A.4 액세스코드 입장 플로우

- www.picklass.com에서 [액세스코드 입장] 클릭 → 코드 입력 모달 표시
- 6자리 코드 입력 (I, O, 0, 1 제외), 자동 대문자 변환
- 검증: 형식 → 존재 → 상태(미사용) → 유효기간
- 성공: 기관 연결된 tutoring으로 진입 + 축하 메시지
- 실패: 오류 메시지 + "소속 학원에 문의하기" 링크

#### 6.A.5 비로그인 접근 정책

- 접근 가능: www 랜딩 전체, 학생 로그인 CTA, 액세스코드 모달, 정식 오픈 알림 신청
- 접근 제한: tutoring/speaking/studio/admin 실제 앱 화면은 로그인/코드 필수
- **B2B 관련 콘텐츠(제휴/데모/학원용 기능)는 www에서 일체 노출하지 않음** → studio 루트로 유도

---

### 6.B studio.picklass.com/ — B2B 홍보 랜딩

#### 6.B.1 페이지 구성

- 헤더 (로고 / 네비 / 기관장·강사 로그인 / 데모 신청)
- Hero: "학원 운영을 가볍게, 강사 업무를 똑똑하게"
- 주요 기능 (기관 관리 / 강사 저작 / AI 자동 레슨 생성 / 학생 성과 리포트)
- 플랜별 요금제 (Starter / Pro / Enterprise / 제휴)
- 도입 사례 (Phase 1은 파고다 외 초기 파트너 중심)
- 데모 신청 폼 / 제휴 문의 폼
- 푸터 (회사 정보, 약관, 개인정보처리방침)

#### 6.B.2 로그인 / 회원가입 플로우

**공통 단계 (기관장·강사)**
1. 사용자 타입 선택 (기관장 / 강사)
2. 인증 방법 선택 (이메일 / 소셜 3종)
3. 인증 정보 입력
4. **로그인 후 추가 정보 입력 모달** (타입별 필수 필드)
   - 기관장: 기관명, 기관유형, 성명, 연락처
   - 강사: 닉네임, 소속 기관(코드 연결)
5. 완료 → `studio.picklass.com/app`으로 이동

#### 6.B.3 데모/제휴 문의 플로우

- 비로그인 상태에서도 문의 폼 제출 가능
- 수집: 기관명, 담당자명, 연락처, 이메일, 운영 형태, 예상 학생 수
- 제출 시 세일즈팀 알림 + 자동 응답 메일

#### 6.B.4 학생·학부모 차단 원칙

- studio 루트는 **기관·강사 대상 콘텐츠만** 노출
- "학생이신가요? www.picklass.com으로 이동하세요" 배너를 명시 (헤더 또는 푸터)
- 검색엔진은 B2B 키워드로만 노출되도록 메타 태그 관리

---

### 6.C studio.picklass.com/app — Studio 저작 앱 진입

- 로그인 필수, 인덱싱 차단(noindex)
- 강사·기관장이 지문/과정/레슨을 제작·편집
- 상세 기능은 **§8 스튜디오** 참조

---

### 6.D 공통 정책

#### 6.D.1 이용약관 및 개인정보 수집 동의

- 필수: 이용약관 + 개인정보 수집
- 선택: 마케팅 정보 수신
- 수집 항목: 이메일, 사용자 타입, 인증 정보, 타입별 추가 정보

#### 6.D.2 Speaking 체험 진입점

- 비로그인 상태에서 **맛보기 대화 1회 (3분)** 제공 여부는 Phase 2에서 본격 검토
- Phase 1에는 로그인/코드 입장 이후에만 Speaking 제공
- 체험 시 음성 녹취는 기본 미저장(프라이버시 우선)

---

## 7. 백오피스 (Admin)

### 7.1 페이지 맵 및 IA

```
/admin
├── dashboard
├── institute        (기관관리)
│   ├── register
│   └── [id]/edit
├── users            (사용자관리)
├── accesscode       (액세스코드)
│   └── generate
├── ai-modules       (AI 모듈)
├── billing          (청구)
└── system           (시스템 상수)
```

### 7.2 대시보드

- 등록 기관 수, 활성 사용자, 월 매출
- 플랜별 기관 분포
- **AI 모듈 사용량** (LLM 토큰/STT·TTS 분)
- **음성 세션 사용량** (Speaking 전용 그래프)

### 7.3 Organization 관리 (Partner / Group / Institution)

> **v0.6 재편:** 기존 "기관관리(Institute)"를 **Organization 3-Tier 관리**로 확장. Academy / Enterprise / K12 섹터 공통 UI에서 L0–L2 전 레벨의 조직 등록·편집·상태 관리를 제공한다.

#### 7.3.1 Organization 목록 및 계층 뷰

- 트리 뷰: Partner → Group → Institution 계층 구조 시각화
- 필터: 섹터(Academy/Enterprise/K12) · 레벨(L0/L1/L2) · 상태
- 검색: 조직명·담당자·연락처·계약번호

#### 7.3.2 Partner 등록 (L0, 선택)

- 파트너사명, 대표 담당자, 계약 유형(중개·리셀·화이트라벨)
- 수수료·정산 정책
- 관리 범위(산하 Group/Institution 설정 권한)

#### 7.3.3 Group 등록 (L1, 선택)

- Group명, sector(Academy/Enterprise/K12), 상위 Partner(있을 경우)
- 학원: 프랜차이즈 본부 / 기업: 지주사·본사 / K12: 교육청
- GroupAdmin(본부 관리자 / **HR 총괄** / 교육청 관리자) 초기 담당자
- Group 단위 플랜 일괄 구매 여부(선택)

#### 7.3.4 Institution 등록 (L2, 필수)

기존 4섹션 구조 유지, sector·parentId 필드 추가:

1. **가입정보** — 아이디(이메일), 초기 비밀번호, **Institution명**, **sector**, **parentId(Partner/Group)**, 담당자
2. **부가정보** — 지점 수, 운영 형태(직영/가맹/직접운영), 학습자 규모
3. **라이선스·요금제** — 플랜, 월비용, 기본좌석수, 단가, API 연동 *(seatSource 기본값은 v0.17에서 폐기)*
4. **계약정보** — 계약상태, 시작/종료일, 자동갱신, 갱신조건

**플랜 선택 시 자동 채움**: monthlyPrice, annualDiscount, baseStudents, pricePerStudent, maxAdminAccounts, apiIntegration이 DEFAULT_PLANS 값으로 초기화되며, 이후 수동 편집 가능.

#### 7.3.5 조직 상태 연쇄 처리

상위 Organization 상태 변경 시 산하 조직과 Membership에 대한 영향 범위를 선택:
- **연쇄 적용(Cascade)** — 산하 전체 동일 상태로 전환
- **독립 유지(Isolate)** — 산하 상태 그대로, 상위만 변경
- **경고만(Warn)** — 상태 변경하되 산하는 별도 재검토 큐로 이동

### 7.3A 기존 명칭 (Institute) 호환 (Deprecated)

기존 `/admin/institute` 경로는 L2 Institution 전용 뷰로 리다이렉트. 새로운 `/admin/organizations`가 3-Tier 통합 관리 진입점.

### 7.4 사용자관리 (Users) *(⚠️ v0.10 정합화)*

- 역할별 통계 카드 (학원관리자/강사/학생)
- 테이블 컬럼: 역할·소속·사용자명·아이디·액세스코드·상태·활성일·작업
- **등록 방식**: 관리자는 institute에서, 강사/학생은 액세스코드 사전 생성
- **아이디 동시 생성 옵션**: `{prefix}{seq}@{domain}.pickle` 형식

#### 7.4.1 사용자 일괄 생성 워크플로 *(✅ 구현 완료 — 2026-04-24)*

> 기관 단위로 강사/학생 계정을 **대량 생성하여 즉시 로그인 가능한 상태**로 배포하는 기능. v0.10에서 정식 워크플로로 등재.
>
> **참조**: `picklass-backoffice/docs/users/20260424_사용자일괄생성.md`

**진입 경로**

```
/admin/users
   └─ "사용자 일괄 생성" 버튼 클릭
        ↓
   /admin/users/access-code (페이지 제목: "아이디 일괄 생성")
        ↓
   사용자 유형 (강사/학생) + 소속 기관 + 생성 개수(1~1000) + 고유코드(4자리)
        ↓
   "생성" 클릭 → 액세스코드 + 사용자 계정 동시 생성
        ↓
   "초기 비밀번호는 아이디와 동일합니다" alert
        ↓
   /admin/users 리다이렉트
```

**자동 생성 규칙**

| 항목 | 규칙 |
|---|---|
| `userId` | `{고유코드소문자}{3자리일련번호}@{고유코드소문자}.pick` 형식 (예: `abcd001@abcd.pick`) |
| `password` | `userId와 동일` (임시, `isTempPassword: true`) |
| 일련번호 시작값 | 동일 도메인 기존 사용자 수 + 1 (재실행 시 충돌 방지) |
| `registrationExpiry` | 생성일 + **365일** 고정 |
| `usagePeriodDays` | **365** 고정 |
| 사용자 statusCode | **`active`** (즉시 활성, 별도 활성화 플로우 불필요) |
| 액세스코드 statusCode | **`active`** |
| 액세스코드 동시 생성 | **항상 함께 생성** (체크박스 옵션 제거됨) |

**기술 부채 / 알려진 제한**

| 항목 | 상태 |
|---|---|
| UI는 1,000개 허용, 서비스 내부 `Math.min(..., 100)` 클램핑 | ⚠️ 100개 상한 버그 — `access-code.service.ts:53` 수정 필요 |
| `isTempPassword: true` 강제 변경 로직 | 📋 미구현 — 첫 로그인 시 비밀번호 변경 강제 라우팅 후속 작업 |
| `generated_password` 평문 DB 저장 | ⚠️ 보안 검토 필요 — 배포 확인 후 null 처리 검토 |
| 성공 메시지 `alert()` | 📋 toast UI로 교체 예정 |

**API 계약**

```typescript
// POST /access-codes
interface CreateAccessCodeDto {
  roleCode: 'teacher' | 'student';
  institutionId: string;            // UUID
  count: number;                    // 1~1000 (서비스 내부는 100 클램핑 — 버그)
  registrationExpiry: string;       // ISO date (생성일 +365일 고정)
  usagePeriodDays: number;          // 365 고정
  statusCode?: string;              // 'active' 고정
  createUserSimultaneously?: boolean; // true 고정
  userIdPrefix?: string;            // 고유코드 소문자
  userIdDomain?: string;            // '{고유코드소문자}.pick'
}

interface AccessCodeResponse {
  id, code, institutionId, institutionName, roleCode,
  userId, courseId, statusCode,
  registrationExpiry, usagePeriodDays,
  generatedUserId: string;          // 평문 (배포용)
  generatedPassword: string;        // 평문 (배포용)
  activatedAt, createdAt;
}
```

### 7.5 액세스코드 관리

- **형식**: 6자리 (A-Z [I, O 제외] + 2-9 [0, 1 제외] = 32종 조합)
- **혼동 방지**: I↔1, O↔0 혼동 문자 제거
- **고유성**: 시스템 내 중복 생성 절대 불가
- **상태**: 활성 / 예약됨 / 사용됨 / 만료됨
- **초기 상태**: 생성 직후 `미사용` 고정 (등록 대기)
- **유효기간**: 1M / 3M / 6M / 12M (발급일 + 일수)
- **생성 단위**: **1~1,000개 일괄 생성** (v0.7 확정)
- **학생용**: 제공 과정 선택 필수 (강사는 과정 선택 미표시)
- **재발급**: 만료·오류 코드는 폐기 후 신규 생성

> 📎 **일괄 생성 상한 변경 이력**: PDF 초안 v1.0(2026-03-05)은 100개 상한, v0.7(2026-04-21)에서 **1,000개로 확정**. 대량 배포 기관(프랜차이즈 본부·기업 지주사) 시나리오 대응을 위한 확장.

**Speaking 독립 앱 확장 (v0.5.3)**
- **과정 단위 인증코드**: Speaking 단독 앱 경로 A에서 사용. 인증코드 1건 = 과정 1건 수강권. 사용자가 입력하면 해당 과정이 "수강 중인 과정" 목록에 추가되며, 과정 내 모든 레슨을 추가 인증 없이 수강 가능.
- **사용 기록 단위**: 인증코드 ↔ 사용자 ↔ 과정 3축으로 매핑되며, 동일 과정 재사용(연장) 시 신규 코드 발급 필요.

### 7.6 수업 모듈 관리 (AI Modules)

#### 7.6.1 Reading/Writing 모듈 마스터
| 모듈ID | 이름 | 상태 | 포함 플랜 | 월 한도 |
|---|---|---|---|---|
| ai-generate | AI 생성 | 활성 | Pro/Ent | 무제한 |
| tts | TTS | 활성 | Pro/Ent | 10,000자 |
| strategic-reading | 전략적 읽기 | 활성 | Pro/Ent | 무제한 |
| word-analysis | 단어 분석 | 활성 | 모든 플랜 | 무제한 |
| stt | STT | 개발중 | - | - |

#### 7.6.2 Speaking 6모듈 마스터 (§14 상세 연결) *(v0.21 — v0.15 6모듈 정렬)*
- Learn & Study (LRN), Vocabulary Listening & Meaning (VLM), Expression Drill (EDR, 3-stage), Role-Play (RPL, PTT), Free Talking (FRT), One Minute Presentation (OMP). *구 7모듈(WSP/LR/SHD/SXP/RLP/FRT/OMS)의 재설계·통합·코드 변경 이력은 §13.1.6 폐기 모듈 표 참조.*

#### 7.6.3 플랜 포함 여부 · 월 한도 · 초과 과금
- 기본 포함: 플랜 월정액에 포함
- 초과 사용: 모듈별 초과 요금 자동 계산 (Usage 테이블 기록)

#### 7.6.4 음성 세션별 사용량 집계
- 세션 단위: STT 초, TTS 글자, LLM 토큰 3축 집계
- 기관별·사용자별·일별 대시보드

### 7.7 Billing (청구 관리)

- 요금제별 가입 현황 / 월 매출 추이 (최근 3개월)
- 미납 처리 워크플로 (미납해결 버튼)
- 플랜 변경 시 역사적 가격 보존(기존 계약 유지)
- 페이지네이션: 20행/페이지

### 7.7.1 플랜 상품 구성 옵션 (Product Configuration)

> **v0.5.3 신설**: 플랜 단위로 활성/비활성화하는 기능 토글을 중앙에서 관리한다. 여기에서 설정한 옵션은 해당 플랜을 보유한 모든 사용자·기관에 일괄 반영된다.

| 옵션 코드 | 옵션명 | 설명 | 적용 플랜(예) |
|---|---|---|---|
| `opt.personal-interest-speaking` | **개인 관심사 수업** | Speaking 독립 앱 경로 B(개인 관심사 기반 세션) 활성화. 옵션 ON인 플랜 사용자에게만 UI 노출. | Pro·Enterprise 또는 별도 요금제 |
| `opt.speaking-unlimited` | Speaking 무제한 세션 | 월 세션 한도 해제 | Enterprise |
| `opt.ai-topic-recommender` | AI 추천 주제 엔진 | 최신 뉴스·이슈 기반 주제 자동 생성 (경로 B 전제 기능) | Pro·Enterprise |
| ... | (향후 추가) | | |

**운영 규칙**
- 옵션 토글은 Admin 시스템관리자만 편집 가능
- 기존 플랜 구독자에 대해 옵션을 회수할 경우: 다음 갱신 주기부터 적용 (현재 주기는 유지)
- 옵션 조합의 비정상 상태(예: 경로 B 옵션 ON인데 AI 추천 엔진 OFF) 자동 검증

### 7.7.2 ❌ 좌석 배분 정책 *(v0.17 삭제)*

> ❌ **v0.17 폐기 — 좌석 배분(seatSource) 정책**: v0.6에서 신설했던 좌석 단위 추적·배분 모델은 v0.17에서 폐기되었다. 청구 모델은 **활성 Membership 카운트 + 사용량 기반**으로 단순화되었으며, B2B 제휴 운영(§16)에서는 제휴사가 자체 결제·과금을 관리하므로 picklass 측 좌석 추적이 불필요하다. `seat_allocations` 테이블 폐기, `Membership.seatSource` 필드 폐기.

### 7.8 시스템관리

4개 섹션:
1. **사용자 상태** (active/inactive/suspended/withdrawn)
2. **액세스코드 상태·유효기간**
3. **CEFR 18단계 레벨 시스템**
4. **레벨 ↔ WPM 매핑표** (Speaking 연동)

#### CEFR 18단계 ↔ WPM 매핑

| 카테고리 | 레벨 | CEFR | WPM (L = 80 + (N−1) × 4) |
|---|---|---|---|
| Starter | 1–3 | A1−, A1, A1+ | 80, 84, 88 |
| Beginner | 4–6 | A2−, A2, A2+ | 92, 96, 100 |
| Intermediate | 7–9 | B1−, B1, B1+ | 104, 108, 112 |
| Upper-Intermediate | 10–12 | B2−, B2, B2+ | 116, 120, 124 |
| Advanced | 13–15 | C1−, C1, C1+ | 128, 132, 136 |
| Proficient | 16–18 | C2−, C2, C2+ | 140, 144, 148 |

### 7.9 권한 및 감사 로그

- 모든 기관·액세스코드·상태 변경 기록 보관
- 변경자·시간·이전/이후 값 3종 필드
- 보안 감시 로그는 일정 기간 보관

### 7.10 기관 브랜딩 (Institution Branding) *(v0.17 신설)*

> 📋 **기획 단계** — 거래 학원·기업이 **자기 기관의 로고·테마 컬러·스킨**을 학습자 화면에 적용할 수 있도록 하는 UI 커스터마이징 기능. B2B 영업의 핵심 요구사항으로 **"우리 회사 영어 교육 플랫폼"으로 보이게 하는 화이트 라벨 효과**를 제공한다.

#### 7.10.1 적용 범위

| 적용 대상 | 학습자 화면 | 강사 화면 | 관리자 화면 |
|---|---|---|---|
| **기관 로고** | ✅ 우상단 + 로딩 화면 | ✅ 헤더 | ✅ 헤더 |
| **테마 컬러 (primary·accent)** | ✅ 버튼·진행바·하이라이트 | ✅ 일부 | ✅ 일부 |
| **스킨 (배경·카드 스타일)** | ✅ 학습자 메인·세션 화면 | ⚠️ 선택적 | ❌ (관리 일관성) |
| **도메인 (서브도메인 별칭)** | ⚠️ 옵션 (`{기관}.tutoring.picklass.com`) | ⚠️ 옵션 | ❌ |
| **이메일 발송 from 주소** | ✅ "{기관명} 영어 학습" | ✅ | — |
| **수료증·리포트 PDF 헤더** | ✅ 기관 로고+이름 | ✅ | ✅ |

> ⚠️ Pickle Agent 캐릭터·UI 패턴(모바일 3분할 등) 등 **picklass 핵심 학습 UX**는 브랜딩 변경 대상에서 제외. 학습 효과·접근성 일관성을 위함.

#### 7.10.2 브랜딩 설정 권한

| 권한 | 설정 가능 항목 |
|---|---|
| PlatformAdmin | 전체 |
| GroupAdmin | 산하 Institution 일괄 적용 |
| **InstitutionAdmin** | **자기 기관 로고·테마·이메일 from·PDF 헤더** |
| Instructor | (없음 — 조회만) |

#### 7.10.3 데이터 모델 (요약 — 골격)

```sql
table institutions (
  id pk,
  name varchar(255),
  ...
  brand_config jsonb null,    -- v0.17 신설
  ...
);

-- brand_config 구조 (예시)
{
  "logo_url": "https://cdn.../org_123/logo.svg",
  "logo_dark_url": "https://cdn.../org_123/logo-dark.svg",
  "favicon_url": "https://cdn.../org_123/favicon.ico",
  "theme": {
    "primary_color": "#0066CC",
    "accent_color": "#FF6B35",
    "background_style": "subtle-pattern" | "solid" | "gradient"
  },
  "skin_id": "default" | "warm" | "cool" | "minimal",
  "subdomain_alias": "abc-academy",   // 옵션
  "email_from_name": "ABC 어학원 영어 학습",
  "email_from_address": "noreply@abc-academy.com",  // SPF/DKIM 검증 후
  "pdf_header": {
    "logo_url": "...",
    "institution_name": "ABC 어학원",
    "footer_text": "..."
  }
}
```

#### 7.10.4 적용 정책

| 정책 | 내용 |
|---|---|
| **렌더링 방식** | 학습자 진입 시 액세스코드 또는 도메인으로 기관 식별 → `brand_config` 동적 주입 (CSS 변수 + 이미지 URL 교체) |
| **캐싱** | 브랜드 자산은 CDN 캐싱(1시간), 변경 시 즉시 invalidate |
| **승인** | 로고 업로드 시 자동 검수(파일 크기·형식·해상도) + 부적절 콘텐츠 자동 차단 |
| **B2C 영역** | `www.picklass.com` 등 picklass 자체 도메인은 brand_config 미적용 |
| **이메일 도메인 검증** | 사용자 정의 from 주소 사용 시 SPF·DKIM 레코드 검증 필수 |

#### 7.10.5 B2B 제휴와의 관계 (§16 연동)

B2B 제휴 모드에서는 일반적으로 제휴사가 **자체 사이트에 픽클래스 학습 모듈을 임베드**하므로, 외부 사이트 자체가 브랜딩 컨테이너 역할을 한다. 이 경우 picklass 측 brand_config는 보조적으로만 사용된다.

| 운영 형태 | brand_config 역할 |
|---|---|
| 픽클래스 자체 운영 (B2C·일반 B2B 직판) | **주요** — 학습자 화면 전체 브랜딩 |
| Tutoring 통합 임베드 (§3.3.1) | **보조** — picklass 도메인 진입 시에만 적용 |
| 외부 임베드 (§3.3.4, §16) | **보조** — 임베드 위젯 내부 영역만 적용 (외부 사이트 헤더는 제휴사가 브랜딩) |

---

## 8. 스튜디오 (Studio · 강사 운영 레이어)

> **역할 재정의 (v0.4):** Studio는 강사·기관장이 **화면에서 직접 조작하는 저작·운영 레이어**이다. 지문 생성·분석의 엔진 상세는 **§9 콘텐츠 생성·분석 엔진**, 레슨 자동 설계의 엔진 상세는 **§10 모듈 시퀀싱 엔진**에서 별도로 다룬다.

### 8.1 서비스 개요 및 역할

- **도메인**: `studio.picklass.com/app`
- **대상**: 강사·기관장
- **미션**: "5분 내에 오늘 수업 준비 완료" — 지문 생성·검수 → 레슨 자동 설계 → 학생 배정까지 단일 워크플로우 제공

### 8.2 과정 / 레슨 / 지문 계층 구조 (아키텍처 맵)

```
Course (과정)
 └─ Lesson (레슨)
     ├─ Passage (지문 1개)                    ← 엔진: §9
     └─ ModuleSequence (모듈 N개)             ← 엔진: §10
         └─ PlannedModule (order, role, ...)
```

- **Course**: 장기 학습 단위 (예: "10주 비즈니스 영어 기초")
- **Lesson**: 1회 수업 분량 (15~45분)
- **Passage**: 레슨 1개당 1개 지문 (§9에서 생성·분석)
- **ModuleSequence**: 해당 레슨에 배정된 교수법 모듈 N개 (§10에서 시퀀싱)

### 8.3 Course Hub *(v0.17 — 구 Course Wizard에서 명칭 변경)*

Studio 수업 구성 전체(My Library → 레슨 → 과정 → 배포 4-STEP)를 **단일 통합 UI에서 한 번에 진행**하는 진입점이다. 강사가 각 메뉴를 순회하지 않고 빠르게 과정을 만들 수 있도록 설계되었으며, 숙련도에 따라 3가지 모드를 제공한다.

#### 8.3.1 3가지 모드

| 모드 | 사용 대상 | 특징 |
|---|---|---|
| **Quick Start** | 처음 사용하는 강사, 템플릿 미선택 | 최소 입력(지문 1개 + 기본 모듈 구성)으로 즉시 배포 가능 |
| **Template** | 반복 운영 강사 | 사전 준비된 템플릿(학년·학습 목표·과정 유형별)을 선택 후 변수만 입력 |
| **Advanced** | 숙련 강사, 세밀한 제어 필요 | 각 STEP의 전 필드 노출, 모듈 커스터마이징·조건부 규칙·시간 예산 정밀 제어 |

#### 8.3.2 진행 단계 — STEP 1~4 통합

Course Hub 단일 화면에서 순차 진행하되, 각 STEP 사이 건너뛰기·되돌리기를 허용한다. STEP 내부 동작은 본문의 해당 엔진 절을 호출한다.

| STEP | 명칭 | 주 동작 | 연동 |
|---|---|---|---|
| **STEP 1** | 지문 입력 | 텍스트 직접 입력 / 파일 업로드 / Hub 복제(향후) | §9 콘텐츠 생성·분석 엔진 |
| **STEP 2** | 레슨 구성 | AI 자동 시퀀싱 or 수동 모듈 선택 | §10 모듈 시퀀싱 엔진 |
| **STEP 3** | 과정 구성 | 레슨 묶기, 순서 지정 | §8.2 계층 구조 |
| **STEP 4** | 배포 | 액세스 코드 발급, 대상·기간 설정 | §7.5 액세스코드 |

#### 8.3.3 경로 및 진입점
- URL: `studio.picklass.com/app/course-hub` *(구 `/wizard` 호환 리다이렉트)*
- 진입점: Studio 대시보드의 **"새 과정 만들기"** CTA, 상단 네비게이션의 **"Course Hub"** 메뉴, 빈 My Library에서 안내되는 **"지금 시작하기"** 링크
- **On-demand 원칙(§2.6) 연동**: Course Hub는 지문과 LessonPlan만 저장하고, 실제 문항·피드백은 학생이 Tutoring에서 모듈에 진입할 때 생성됨

#### 8.3.4 모드 간 전환
- 초안 저장 후 언제든 모드 전환 가능 (Quick Start → Template → Advanced 단계적 심화)
- Advanced에서 저장된 설정은 Template으로 역변환 가능 (개인 템플릿화)

#### 8.3.5 B2B 교재 업로드 워크플로 *(v0.16 신설)*

B2B 제휴사·기관이 **자체 보유 교재(PDF/DOC)를 업로드**하여 과정으로 변환하는 워크플로. Course Hub 진입 시 STEP 1 위에 "교재 업로드" 진입점이 추가된다.

**5단계 자동화 흐름**

```
[Step 0] 교재 파일 업로드 (PDF / DOC / DOCX)
   ↓
[Step 1] 자동 분할 (단원/지문 단위)
   ├─ 페이지·헤더·소단원 마크 기반 자동 분할
   └─ 강사가 분할 결과 검수·수정 가능
   ↓
[Step 2] 단원별 교재 의도 메타데이터 자동 추출 (§9.6.3)
   ├─ key_expressions[]   : 핵심 표현 5개
   ├─ key_vocabulary[]    : 핵심 어휘 N개
   ├─ grammar_focus[]     : 문법 포인트
   └─ learning_objective  : 단원 학습 목표
   ↓
[Step 3] 각 단원에 대해 §10 모듈 시퀀싱 엔진 호출
   ├─ 의도 메타데이터를 LLM 프롬프트에 주입 (콘텐츠 일치성 확보)
   └─ 단원별 LessonPlan 자동 생성
   ↓
[Step 4] 단원 LessonPlan을 묶어 과정으로 등록
   ↓
[Step 5] 강사 검수 + 배포 (액세스코드 발급)
```

| 항목 | 정책 |
|---|---|
| 지원 포맷 | PDF, DOC, DOCX (이미지 PDF는 OCR 후 처리) |
| 단원 자동 분할 정확도 | 80% 이상 (틀린 분할은 강사가 수동 조정) |
| 의도 메타데이터 추출 정확도 | 핵심 표현 70%↑, 어휘 80%↑ — 강사 검수 단계 필수 |
| 권한 | 기관 관리자·강사만 업로드 가능 (학생 X) |
| 저작권 | 업로드 전 "교재 권리 보유 확인" 체크박스 강제 |

> 📎 데이터 모델: `content_items.textbook_intent`(jsonb), `content_items.textbook_source`(jsonb) — §9.6.3 참조.

#### 8.3.6 과정 생성 3종 모드 *(v0.16 신설 — Speaking·Tutoring 이원화 대응)*

과정 생성 초기에 **활용할 모듈 범위**를 3종 중 선택한다. 선택에 따라 STEP 2 모듈 시퀀싱이 사용 가능한 모듈 풀이 결정된다.

| 모드 | 활용 모듈 | 용도 | 사용 모듈 카탈로그 |
|---|---|---|---|
| **A. 스피킹 only** | Pick-Speak Method 6 모듈 | Speaking 회화 중심 과정 (1:1 회화·B2C·외부 임베드) | LRN, VLM, EDR, RPL, FRT, OMP |
| **B. 튜터링 only** | Tutoring 22 모듈 (V/R/W) | 4기능 교과형 과정 (학원·학교·B2B 정규) | Vocabulary 7 + Reading 8 + Writing 6 + 일부 Speaking 발음 모듈 (선택적) |
| **C. 통합** | 전체 28 모듈 | 종합 영어 과정 (4기능 + 회화) | 전 모듈 |

**모드 결정 시점** — Course Hub 진입 직후, 교재 업로드 분기 이전

```
[Course Hub 진입]
   ↓
[과정 생성 모드 선택 (A/B/C)] ← v0.16 신규 단계 / v0.17 위치 명확화
   ↓
[교재 업로드 여부 선택]
   ├─ 업로드 ✓ → §8.3.5 5단계 워크플로 진입 (선택 모드의 모듈 풀로 제한)
   └─ 업로드 ✗ → 일반 STEP 1 진입
   ↓
[STEP 2 모듈 시퀀싱] (선택 모드의 모듈 풀로 제한)
```

> 🔁 **v0.17 명확화**: 모드 선택은 **Course Hub 진입 직후**에 결정되며, 이후 교재 업로드·STEP 1·STEP 2 모든 단계에서 모드의 모듈 풀로 제한된다. 모드는 과정 생성 진행 중 변경 불가 (변경 시 새 과정으로 재시작).

**모드별 시퀀싱 엔진 동작 차이**

| 모드 | 시퀀싱 엔진 동작 | 학습자 진입 경로 |
|---|---|---|
| A. 스피킹 only | §10 시퀀싱 엔진 + Pick-Speak Method 5단계 흐름 | Speaking 독립 앱 (`speaking.picklass.com`) 또는 Tutoring 임베드 |
| B. 튜터링 only | §10 시퀀싱 엔진 + 4-Role/3-Stage 분류 기반 | Tutoring (`tutoring.picklass.com`) |
| C. 통합 | 두 엔진 병행 + 단원별 모드 가변 가능 | Tutoring + Speaking 통합 진입 |

> 📎 §13.2 서비스 범위 분류와 직접 연동. 과정 모드는 `courses.module_scope` 컬럼(enum: 'speaking_only', 'tutoring_only', 'unified')으로 저장.

### 8.4 지문 라이브러리 UX

지문 라이브러리의 UI/UX 경험만 다룬다. 생성·분석 엔진 내부 동작은 §9 참조.

- **리스트 뷰**: 주제·스킬·난이도·유형·출처 필터
- **카드 뷰**: 미리보기 + 태그 + CEFR 레벨 배지
- **즐겨찾기 / 최근 사용**
- **공유 범위 인디케이터**: 개인 / 기관 / 플랫폼 (§8.10)
- **"이 지문으로 레슨 만들기"** 원클릭 CTA → §10 Planner 호출

### 8.5 레슨 편집 UX

레슨 편집 화면의 UI/UX만 다룬다. 자동 설계 알고리즘은 §10 참조.

- **3-모드 토글**: AI 자동 / 수동 / 하이브리드
- **타임라인 뷰**: 모듈 순서·소요시간을 바 차트로 시각화
- **Planner Rationale 패널**: §10 엔진이 제공한 "이 모듈을 선택한 이유" 표시
- **드래그 앤 드롭**: 모듈 순서 변경
- **실시간 제약 경고**: 선행조건·양립불가·시간 초과 위반 시 인라인 표시
- **A/B 제안 탭**: §10의 2안 제안을 비교 선택

### 8.6 모듈 라이브러리 활용

- 역할별 필터: warming / passage-use / practice / output
- 스킬별 필터: Reading / Writing / Listening / Speaking
- 적정 레벨·예상 소요시간 배지
- "라이브러리에서 바로 추가" — 레슨 편집 화면에서 드래그
- Speaking 모듈 선택 시 음성 세션 UI가 자동 연결됨 (§14)

### 8.7 Speaking 시나리오/주제 편집 도구

- 프리토킹용 주제풀 관리 (영화, 여행, 비즈니스 등)
- 난이도 태깅 (레벨 1~18)
- 대화 예시·힌트 3단계 사전 정의
- Role-Play 시나리오 편집 (파트 A/B 스크립트)
- 엔진 내부 동작은 §14.4 스피킹 7 교수법 모듈과 §14.5 레벨별 Adaptive Interaction 매트릭스 참조

### 8.8 학생 배정 및 배포 *(v0.17 보강)*

- **학생 단위 배정** 또는 **기관 전체 단위 배정** (학원관리자가 일괄 적용)
- 링크 또는 액세스코드 방식 배포
- 마감일·최대 시도 횟수 설정

> ⚠️ **picklass는 LMS의 "분반(Class)" 등의 그룹 설정 기능을 두지 않는다**. 일반 LMS는 학생을 "반(Class)" 단위로 묶어 일괄 관리하지만, picklass의 배정 단위는 **(a) 학생 개별** 또는 **(b) 기관 전체** 두 종류로 단순화된다. 분반·학년·세션 그룹화가 필요한 경우, 기관 자체 LMS·HR 시스템에서 그룹을 관리하고 picklass에는 **그룹별로 다른 액세스코드를 발급**하는 방식으로 운용한다(§7.5 액세스코드 정책).

### 8.9 학생 성과 모니터링

- 학생별 진행도 테이블
- 모듈별 정답률·KPI
- 말하기 5지표(유창성/정확성/복잡성/상호작용성/발음) 시각화
- 주간/월간 리포트 PDF 내보내기
- §14.6 KPI 측정 체계와 데이터 모델 연동

### 8.10 권한·공유 정책

| 공유 범위 | 설명 | 편집 권한 |
|---|---|---|
| 개인(private) | 강사 본인만 접근 | 본인 |
| 기관(institution) | 같은 기관 강사 공유 | 본인 + 기관관리자 |
| 플랫폼(public) | 전체 강사에게 공개 | 본인 (관리자 승인 필요) |

- 지문·레슨·모듈 시퀀스 각각에 공유 범위 설정 가능
- 공유 변경 이력 감사 로그 (§22.4 연동)

---

## 9. 콘텐츠 생성·분석 엔진 (Content Generation & Analysis Engine)

> Studio의 지문(Passage) 파이프라인 전체를 담당한다. AI 생성 → 분석 → 검수 → 저장 → 재사용 루프로 구성되며, Studio(§8) UX에서 호출되고 §10 모듈 시퀀싱 엔진의 입력(PassageAnalysis)을 생성한다.
>
> **On-demand 원칙 연동 (§2.6 참조)**: 본 엔진은 **지문 텍스트와 레벨 분석 결과만 저장**하고, 모듈별 활동(문항·피드백·추천 주제)은 학습자 진입 시점에 런타임 생성한다. 지문 자체의 사전 분석은 강사 검수 UX를 위해 필요하므로 예외적으로 저장한다.

### 9.1 개요 및 책임 범위

- **엔진 미션**: "강사가 5분 내에 학습 가능한 지문 1개를 확보"
- **입력**: 강사가 지정한 주제·레벨·길이·유형 (§9.2)
- **출력**: 승인된 지문(Passage) + 분석 결과(PassageAnalysis)
- **경계(Scope)**: 지문 생성·분석·검수·라이브러리 저장까지 담당. 레슨 설계(모듈 시퀀싱)는 §10에서 처리.

### 9.2 입력 파라미터 스키마

```typescript
interface GenerationRequest {
  topic: string;                    // 주제 (자유 텍스트)
  targetLevel: number;              // 목표 CEFR (1~18)
  length: "short"|"medium"|"long";  // 짧음(~100) / 중간(~250) / 김(~500) 단어
  type: "narrative"|"expository"|"argumentative"|"descriptive";
  subjectArea?: string;             // 교과 영역 (선택)
  learnerInterests?: string[];      // 학습자 관심사 (개인화)
  style?: "formal"|"casual"|"academic"|"conversational";
  constraints?: {
    bannedWords?: string[];         // 금칙어
    mustIncludeWords?: string[];    // 포함 어휘
    politySafety?: boolean;         // 정치·종교·폭력 필터 (기본 on)
  };
}
```

### 9.3 생성 파이프라인

#### 9.3.1 프롬프트 템플릿 (모델/버전 관리)
- 템플릿 자산: `prompt_templates/passage_gen/v{n}.md`
- 유형·레벨별 분기 템플릿 (12개 이상)
- 템플릿 버전은 DB에 저장되어 생성 로그와 연결

#### 9.3.2 LLM 호출 전략
| 상황 | 모델 선택 |
|---|---|
| 단일 고품질 지문 | Claude(claude-haiku-4-5) 홀리스틱 생성 |
| 대량 배치(주제 확장) | Google GenAI 병렬 생성 |
| 짧은 재생성(부분 수정) | Claude 스트리밍 + 부분 교정 프롬프트 |

#### 9.3.3 스트리밍 생성 UX
- SSE 기반 실시간 미리보기 (강사가 생성 도중에도 읽기 시작 가능)
- 첫 토큰 지연 목표 < 1.5초 (TTFT)

#### 9.3.4 재시도 및 품질 필터
- 길이 이탈(±30%) 시 자동 재생성
- 목표 레벨과 추정 레벨 편차 > 2 시 재시도 (최대 3회)
- 부적절 콘텐츠 감지 시 즉시 폐기

### 9.4 분석 파이프라인

#### 9.4.1 CEFR 레벨 추정 — 8개 정량 지표 + LLM 하이브리드

정량 지표 8종과 LLM 정성 분석을 결합한 **하이브리드 레벨 추정 엔진**이 지문을 분석해 CEFR 1–18 레벨과 신뢰도 점수를 산출한다. 정량 지표로 포착하기 어려운 문맥·주제·배경지식 요소를 LLM 정성 분석이 보완한다.

**(a) 정량 지표 8종 및 산출식**

| # | 지표 | 측정 기준 | 산출식 |
|---|---|---|---|
| 1 | 어휘 난이도 **LTI** (Lexical Threshold Index) | 어휘 빈도 기반 난이도 점수 | 어휘 난이도 측정 알고리즘 + 18개 레벨 매핑 기준표 (별첨1) |
| 2 | 어휘 다양성 **TTR** (Type-Token Ratio) | 중복 제거 단어 / 전체 단어 비율 | `TTR = Type / Token` |
| 3 | 지문 길이 | 전체 단어 수 | `Word Count` |
| 4 | 평균 문장 길이 | 문장당 단어 수 | `Avg Sentence Length = 총 단어 수 / 총 문장 수` |
| 5 | 문장 구조 **SCI** (Sentence Complexity Index) | 접속사·관계사·특수구두점 기반 복잡도 | `SCI = (접속사 수 × 1.5) + (관계사 수 × 2.0) + (특수구두점(;) 수 × 1.0) + 1` |
| 6 | 문법 다양성 **GWS** (Grammar Weight Score) | 문법 요소별 가중치 합 | `GWS = Σ(문법 요소별 가중치) / 전체 문장 수` |
| 7 | 정보 밀도 **CCR** (Content Concentration Ratio) | 실질어 비율 | `CCR = (실질어 수 / 전체 단어 수) × 100` |
| 8 | 배경지식 의존도 **KDI** (Knowledge Dependency Index) | 고유명사 비율 + LTI 결합 | `KDI = (고유명사 수 / 전체 단어 수 × 0.7) + (LTI / 5.0 × 0.3)` |

> 8개 지표의 상세 계산 방법 및 사례 분석은 **별첨2. 8 지표 계산 방법 및 사례 분석** 문서 참조 (컴패니언 산출물로 분리 작성 예정).

**(b) LLM 정성 분석 (보완 레이어)**

정량 지표로 포착하기 어려운 다음 요소를 **Claude Tool Use 호출**로 분석하여 구조화 출력한다.

- 문맥적 난이도 (context difficulty)
- 주제 복잡성 (topic complexity)
- 배경지식 요구 수준 (prior knowledge requirement)
- 관용 표현 · 문학적 수사 밀도
- 학습자 레벨 대비 문화적 생소함

출력: `{ contextDifficulty: number, topicComplexity: number, ..., rationale: string }`

**(c) 하이브리드 통합 로직**

```
1. 정량 8지표 계산 (로컬, ~200ms)
2. 가중치 기반 1차 CEFR 점수 산출 (Quantitative Score)
3. LLM 정성 분석 호출 (Claude Tool Use, ~1s)
4. 정성 분석 결과로 2차 보정 (특히 주제·맥락 편차 교정)
5. 최종 CEFR 레벨 1~18 + 신뢰도(0~1) 출력
```

**(d) 출력 스키마**

```typescript
interface PassageLevelAnalysis {
  cefrLevel: number;        // 1~18
  confidence: number;       // 0~1
  quantitativeScores: {
    LTI: number; TTR: number; wordCount: number;
    avgSentenceLength: number; SCI: number; GWS: number;
    CCR: number; KDI: number;
  };
  qualitativeAnalysis: {
    contextDifficulty: number;
    topicComplexity: number;
    priorKnowledgeRequirement: number;
    rationale: string;      // LLM 근거 주석
  };
  topContributingIndicators: string[]; // 레벨 상승 기여 상위 3개
  targetLevelDelta?: number;  // targetLevel 입력 시 편차
}
```

**(e) 부속 산출물**
- **레벨 기여 지표 TOP-3** — 강사 검수 화면에서 "왜 B2 레벨인가?"를 즉시 설명
- **LLM 근거 주석** — 하이브리드 판정 과정의 투명성 확보
- **targetLevel 편차 경고** — 강사가 원한 레벨과 실제 추정 레벨 차이가 ±2 이상이면 재생성 권장 알림

#### 9.4.2 유형 분류
- 4종: 서사(narrative) / 설명(expository) / 논증(argumentative) / 묘사(descriptive)
- 분류기: LLM 제로샷 + 규칙 보정

#### 9.4.3 핵심 어휘 추출
- TF-IDF 기반 빈도 가중 + 학습자 레벨 상대 난이도
- 상위 N개(기본 10개) 키 어휘 반환

#### 9.4.4 예상 읽기 시간
- 공식: `ceil(words / WPM(level))`
- WPM은 §7.8의 레벨-WPM 매핑 사용

#### 9.4.5 어려운 문장 하이라이트 (향후)
- 레벨 대비 어려운 문장 자동 표시

#### 9.4.6 저작권·편향 위험 점검 (향후)
- 기존 지문과의 유사도(예: 코사인 유사도) 검사
- 편향 키워드 감지기

### 9.5 강사 검수 워크플로우

```
[AI 생성] → [자동 분석] → [강사 검수 화면]
                               │
     ┌─────────────────────────┼─────────────────────────┐
     ▼                         ▼                         ▼
  [승인]                   [부분 수정]              [재생성 요청]
     │                         │                         │
     ▼                         ▼                         ▼
 승인본 저장           수정 후 재분석             프롬프트 조정
                                                 → 9.3 루프
```

- 분석 결과 표시 → 강사가 수정·승인 → 저장
- 레벨 재추정 버튼, 어휘 재추출 버튼
- 변경 전/후 비교 UI

### 9.6 콘텐츠 라이브러리 (Content Repository) *(⚠️ v0.16 — passage→passage+video 다형 콘텐츠 확장)*

> v0.16부터 단일 `passage` 콘텐츠 모델을 **다형 콘텐츠 모델**(passage / video / media-clip)로 확장한다. B2B 제휴사가 영상 학습 자산을 보유한 경우, 영상 URL 등록 + 자막/전사 텍스트 분석 → 지문과 동일하게 레벨 추정 후 모듈 시퀀싱에 활용 가능.

#### 9.6.1 콘텐츠 타입

| 타입 | 입력 | 분석 방식 | 사용처 |
|---|---|---|---|
| **passage** | 텍스트 지문 | §9.4 지문 분석 파이프라인 직접 적용 | Reading·Writing·Speaking 모듈 전체 |
| **video** *(v0.16 신설)* | 영상 URL + 자막/전사 + 메타(duration·source) | **자막·전사 텍스트 추출 → §9.4.1 지문 레벨 추정 8 정량 지표 재사용** | LRN(LEARN 모듈), Speaking 시나리오, 미디어 클립 큐레이션 |
| **media-clip** *(← §13.13 통합)* | 드라마/영화/유튜브 클립 + 시작·종료 시점 | 자막 기반 표현 매핑 + media_expressions 테이블 | 학습 후 추천 (성과카드, KPI 달성 시) |

#### 9.6.2 영상 콘텐츠 등록·레벨링 워크플로 *(v0.16 신설)*

```
[강사·관리자가 영상 URL 등록]
     ↓
[자막/전사 자동 추출]
   ├─ YouTube: youtube-transcript-api
   ├─ Vimeo·자체 호스팅: 첨부 자막(SRT/VTT) 또는 Whisper 전사
   └─ 자막 없음 → Whisper STT로 자동 전사 (§14.7.6 듀얼 트랙 재사용)
     ↓
[자막 텍스트 → §9.4.1 지문 분석 파이프라인 입력]
   ├─ LTI (어휘 빈도)
   ├─ TTR (어휘 다양도)
   ├─ SCI (문장 복잡도)
   ├─ GWS (학년 점수)
   ├─ CCR (문맥 일관성)
   └─ KDI (지식 밀도)
     ↓
[CEFR 레벨 추정 + 학습 가능 표현 5개 자동 추출]
     ↓
[콘텐츠 라이브러리에 video 타입으로 등록]
```

> 영상 메타데이터(`video.duration`, `video.source`, `video.subtitle_lang`)와 분석 결과(`leveled_at`, `cefr`, `extracted_expressions`)는 `content_items` 테이블에 단일 레코드로 통합 저장.

#### 9.6.3 교재 의도 메타데이터 (Textbook Intent Metadata) — 골격 *(v0.16 신설)*

B2B 교재 기반 과정 생성(§8.3) 시, 교재 본문의 학습 의도(핵심 표현·어휘·학습 목표)를 별도 메타데이터로 보존한다. AI가 모듈 콘텐츠를 생성할 때 LLM 프롬프트에 이 메타데이터를 주입해 **교재 의도 일치성**을 확보한다.

**문제 정의**

기존 AI 콘텐츠 생성은 지문 텍스트만 입력받아 LLM이 자유롭게 모듈 콘텐츠를 생성한다. 그러나 교재는 단원별로 **사전 정의된 학습 표현·어휘·문법 포인트**가 있고, 이를 무시한 자동 생성은 교재의 교수 의도와 어긋나는 결과를 만든다.

**해결 — 교재 의도 메타데이터**

```
[B2B 교재 PDF/DOC 업로드]
     ↓
[자동 분할 — 단원/지문 단위]
     ↓
[각 단원에 대해 다음 메타데이터 추출]
   ├─ key_expressions[]   : 학습 목표 핵심 표현 5개
   ├─ key_vocabulary[]    : 학습 목표 핵심 어휘 N개
   ├─ grammar_focus[]     : 문법 포인트 (예: "현재완료", "조건문")
   ├─ learning_objective  : 단원 학습 목표 한 줄 (예: "비즈니스 미팅 시작 표현 익히기")
   └─ pedagogy_note       : 강사용 메모 (선택)
     ↓
[지문 + 의도 메타데이터를 함께 콘텐츠 레지스트리에 저장]
     ↓
[모듈 콘텐츠 생성 시 LLM 프롬프트에 의도 메타데이터 주입]
   "다음 표현이 반드시 등장해야 함: {{key_expressions}}"
   "어휘는 {{key_vocabulary}} 중심으로 사용"
```

**데이터 모델 (골격)**

```sql
table content_items (
  id pk,
  type enum('passage','video','media-clip'),
  source_url text null,            -- video·media-clip
  body text null,                   -- passage 본문
  cefr enum,
  duration_sec int null,            -- video
  -- 교재 의도 메타데이터 (B2B 교재 업로드 시 자동 채움)
  textbook_intent jsonb null,       -- { key_expressions[], key_vocabulary[],
                                    --   grammar_focus[], learning_objective,
                                    --   pedagogy_note }
  -- 출처 추적
  textbook_source jsonb null,       -- { textbook_id, unit_no, page_range }
  created_by fk,
  created_at timestamp
);
```

> ⚠️ 본 v0.16에는 골격 데이터 모델만 정의한다. 상세 스키마·검증 규칙·LLM 프롬프트 통합 사양은 v0.17 이후 후속 작업.

#### 9.6.4 라이브러리 운영 정책

- 태그 체계: 주제·스킬·난이도·유형·**콘텐츠 타입**·출처
- 검색·필터·즐겨찾기 (콘텐츠 타입 필터 추가)
- 공유 범위: 개인 / 기관 / 플랫폼 (§8.10과 연동)
- 정렬: 최근 사용 / 인기도 / 평가
- 영상 콘텐츠 검수: 자막 정확도 80% 이상 도달 시 published 상태

### 9.7 미디어 라이브러리 (Media Library) *(← §13.13 통합 이관, v0.11 신설 → v0.16 §9 통합)*

> 📎 v0.11에서 §13.13으로 신설했던 미디어 라이브러리(드라마·영화·유튜브 클립 자동 큐레이션)를 v0.16에서 **§9 콘텐츠 라이브러리로 통합 이관**한다. 모든 콘텐츠가 §9 파이프라인을 통해 등록·분석·재사용되도록 통일.

#### 9.7.1 컨셉

학습자가 모듈 학습 중 **특정 핵심 표현**을 만났을 때, 해당 표현이 **실제로 사용된 드라마·영화·유튜브 클립**을 자동 큐레이션하여 학습자에게 제공.

#### 9.7.2 처리 파이프라인

```
[학습 표현 감지 — 모듈 결과 또는 KPI 추적]
     ↓
[Media Library 검색]
   ├─ 자체 인덱싱 미디어 DB (자막+표현 매핑)
   └─ 외부 검색 (YouTube API, OpenSubtitles)
     ↓
[순위화 — 관련성·길이·화자 발음 명확성]
     ↓
[학습자에게 클립 추천 — 학습 후 제공 카드]
```

#### 9.7.3 데이터 모델 (요약)

```sql
table media_assets (
  id pk,
  content_item_id fk,             -- §9.6.1 콘텐츠 모델 통합 (type='media-clip')
  external_id varchar(255),
  title varchar(500),
  language enum('en','en-US','en-UK',...),
  duration_sec int,
  thumbnail_url text,
  embed_url text,
  rights_status enum('cleared','cc','fair_use','restricted'),
  created_at timestamp
);

table media_expressions (
  id pk,
  media_asset_id fk,
  expression_text text,
  start_ms int,
  end_ms int,
  speaker varchar(100) null,
  cefr_level enum('A1',...,'C1+'),
  search_vector tsvector
);
```

#### 9.7.4 저작권·법적 이슈

| 항목 | 정책 |
|---|---|
| 직접 호스팅 | ❌ 불가 |
| 임베드 (YouTube 등) | ✅ 가능 (퍼블리셔 임베드 정책 준수) |
| 자막 기반 표현 매핑 | 자체 인덱싱은 합법, 자막 원문 노출은 라이선스 검토 필요 |
| 클립 시점 표시 | 시작/종료 시점만 노출 → 원본 시청 유도 |

> ⚠️ 법무 검토 후 진행. 본 절은 기능 명세이며 운영 전 라이선스 정책 확정 필요.

#### 9.7.5 통합 지점

| 통합 위치 | 활용 |
|---|---|
| §11.5 성과카드 | "이번 주 학습한 표현이 사용된 영상" 카드 노출 |
| §14.6 KPI | KEYWORD_HIT 등 표현 학습 지표 달성 시 클립 추천 |
| §14.4 LRN 모듈 | LEARN 단계의 보조 콘텐츠 (영상 + 미디어 클립 동시 노출 가능) |
| 학습자 홈 | 일일 학습 후 1~2개 클립 자동 노출 |

#### 9.7.6 추천 알고리즘 (개요)

| 신호 | 가중치 |
|---|---|
| 표현 정확 매칭 | 1.0 |
| 학습자 CEFR 레벨 ±1 | 0.7 |
| 클립 길이 5~15초 | 0.6 |
| 발음 명확도 | 0.5 |
| 신선도(신규 미디어) | 0.3 |

→ 모듈 종료 시점에 위 신호 합산 후 상위 3개 클립 추천.

### 9.8 버전 관리 및 재사용

- 상태 플로우: draft → approved → published
- 수정 이력 및 롤백
- 레슨에 임베드된 지문의 동기화 정책
  - 기본: published 버전 고정 참조 (변경되어도 기존 레슨 영향 없음)
  - 옵션: "최신 버전 자동 동기화" 선택 가능

### 9.9 비용·성능 관리

- 생성 1건당 토큰 예산: 기본 2,000 토큰
- 분석 1건당 시간 목표: < 2초
- 배치 생성, 캐싱, 사전 생성(pre-fetch) 전략
- 기관 단위 월 생성 한도 및 초과 과금 (§5.7 연동)

### 9.10 KPI 및 관찰성

| 지표 | 목표 |
|---|---|
| 생성 성공률 (첫 시도 승인) | ≥ 70% |
| 재생성 비율 | ≤ 30% |
| 평균 검수 시간 | < 3분 |
| 분석 레벨 정확도 (강사 수정 비율 역지표) | ≥ 85% |
| 강사 만족도 | 4.0/5.0 이상 |

### 9.11 데이터 모델 (요약)

```sql
passages (id, text, type, level_estimated, level_target, status,
          author_id, institution_id, share_scope, created_at)
passage_analyses (id, passage_id, level_estimated, level_confidence,
                  type_classification, key_vocab JSONB, reading_min, analyzed_at)
passage_versions (id, passage_id, version, text, changed_by, changed_at, reason)
passage_tags (id, passage_id, tag_type, tag_value)
passage_usages (id, passage_id, lesson_id, used_at)
generation_logs (id, request JSONB, prompt_version, tokens_used, cost, status)
```

### 9.12 API 엔드포인트

```
POST /api/passage/generate            (§9.3 생성)
POST /api/passage/generate/stream     (SSE)
POST /api/passage/analyze             (§9.4 분석 단독 호출)
POST /api/passage/{id}/approve        (§9.5 승인)
POST /api/passage/{id}/version        (§9.7 새 버전)
GET  /api/passage?filter=...          (§9.6 라이브러리)
```

### 9.13 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| LLM 환각 (부정확한 정보) | 학습 방해 | 강사 검수 강제, 사실성 검증 프롬프트 |
| 부적절한 어휘·표현 | 불쾌감, 컴플라이언스 | 금칙어 필터, 안전 분류기 |
| 편향(문화·성·정치) | 학습자 영향 | 편향 감지기, 강사 리뷰 |
| 저작권 유사성 | 법적 리스크 | 유사도 검사 (§9.4.6) |
| 레벨 오추정 | 학습 부적합 | 강사 검수 + 신뢰도 표시 |
| 비용 폭증 | 수익성 | 캐싱, 재시도 상한, 월 한도 |

---

## 10. 모듈 시퀀싱 엔진 (Module Sequencing Engine)

> CurriculumPlannerAgent의 핵심. §9에서 생성된 지문 + 학습 목표 + 시간 예산을 입력으로 받아 **LessonPlan(모듈 시퀀스)**을 자동 생성한다. Studio(§8) 레슨 편집 UX에서 호출된다.
>
> ⚠️ **재정의 (v0.10)**: 본 장의 "CurriculumPlannerAgent"는 **추상 명칭**으로 유지하되, **실제 구현체는 별도 마이크로서비스 `analyzer` 서버**이다 (✅ 구현 완료). LLM Tool Use 기반 Agent로의 향후 전환 가능성은 §10.9 참조.

### 10.1 개요 및 책임 범위

- **엔진 미션**: "주어진 지문과 15분으로 최적의 교수 설계 산출"
- **3-모드**: 자동 생성 / 수동 편집 / 하이브리드
- **경계**: LessonPlan 생성까지. 실시간 학습 중 의사결정은 §12 ModuleOrchestratorAgent가 담당.

### 10.1.1 실제 구현체 — `analyzer` 서버 *(✅ 구현 완료)*

| 항목 | 내용 |
|---|---|
| 서비스 명 | `analyzer` (별도 마이크로서비스) |
| API 엔드포인트 | `POST /lesson-plan` (NestJS API Gateway → analyzer 서버 호출) |
| 호출 주체 | **Studio + Tutoring 양쪽** (단일 진실 원천 → 결과 동등성 parity 보장) |
| 호출 위치 | studio: `/class/lesson-setup/[passageId]`, `/course-hub` / tutoring: `/class/lesson-setup/custom` (나만의 수업) |
| 핵심 알고리즘 | **KPI 집합 커버(Set Cover) + Stage Diversity + 인지 부하 곡선(cognitiveLevel) + 시간 제약** |
| 폴백 정책 | Gemini 폴백 **폐지(2026-04-23)** — analyzer 장애 시 명확한 에러 응답 |
| 데이터 소스 | 운영 Supabase DB (폴리필·목업 모두 제거 완료) |

**요청 스키마 (실제)**

```typescript
// POST /lesson-plan
interface LessonPlanAnalyzerRequest {
  passage_level: string;            // CEFR (예: 'B1', 'C1')
  selected_kpi_codes: string[];     // KPI_CATEGORY 코드 배열
  duration_min: 15 | 20 | 25 | 30;  // 4개 옵션
  passage_text?: string;            // optional, 향후 정밀도 향상용
  passage_id?: number;              // optional, 백엔드가 DB에서 level 조회 시 사용
}
```

**응답 스키마 (실제)**

```typescript
interface LessonPlanResponse {
  module_sequence: PlannedModule[];
  total_estimated_minutes: number;
  sequencing: {
    coverage_score: number;          // KPI 커버리지 점수
    stage_distribution: { before: number, middle: number, after: number };
    fallback_modules: string[];      // 보충된 모듈 (KPI 부족 시)
  };
}
```

### 10.1.2 알고리즘 4단계 *(✅ 구현 완료)*

```
[입력]
  passage_level + selected_kpi_codes + duration_min
        ↓
[1단계] 모듈 풀 필터링 (Module Filtering)
  - suitableLevelMin ≤ passage_level ≤ suitableLevelMax
  - 도메인 매칭 (목적에 부합하는 영역)
        ↓
[2단계] KPI 커버리지 + 시간 제약 시퀀싱 (Set Cover Knapsack)
  - 신규 KPI를 추가로 커버하는 모듈을 우선 선택
  - 이미 커버된 KPI만 담당하는 모듈은 우선순위 하향
  - sum(estimatedMinutes) ≤ duration_min 제약
  - 시간 초과 시 신규 KPI 기여도 낮은 모듈부터 제거
        ↓
[3단계] 단계 결정 + 정렬 (3순위 체계)
  - 1순위: classBefore/Middle/After 플래그 → Stage 결정
  - 2순위: cognitiveLevel ASC → Stage 내 인지 부하 순서
  - 3순위: priority DESC → 동일 cognitiveLevel 내 운영 우선순위
        ↓
[4단계] prerequisites / incompatibleWith 제약 검증·보정
        ↓
[출력] LessonPlan (moduleSequence)
```

> 📎 알고리즘 상세 의사코드는 `studio.picklass.com3/docs/20260421_지능형 수업 설계 자동화 로직.md` 참조.

### 10.2 입력 스키마

```typescript
interface PlanningRequest {
  passageAnalysis: PassageAnalysis;     // §9.4 출력
  learningGoal: string | GoalTag[];     // 자연어 또는 구조화된 목표
  targetLevel: number;                  // CEFR 1~18
  timeBudget: number;                   // 분
  availableModules: ModulePlanningMeta[]; // 라이브러리 카탈로그
  learnerProfile?: {                    // 선택 (누적 학습 이력)
    recentModules: string[];            // 중복 회피
    weakSkills: string[];               // 약점 보강
    avgCompletionRate: number;
  };
}
```

### 10.3 필터링 알고리즘 (Module Filtering)

#### 10.3.1 레벨 적합성
- `suitableLevels.min ≤ targetLevel ≤ suitableLevels.max` 교집합 통과한 모듈만 선택

#### 10.3.2 교수 목표 매칭
- learningGoal → module roles 매핑 테이블
- 예: "표현력 향상" → output 역할 가산

#### 10.3.3 양립불가 제거
- `incompatibleWith` 제약 충족 모듈만 잔존

#### 10.3.4 선행조건 검증
- `prerequisites` 배열의 모듈이 이미 포함될 수 있는 경우만 허용

#### 10.3.5 학습자 이력 반영
- 최근 사용 모듈은 가중치 감점 (중복 회피)
- 약점 스킬 매칭 모듈은 가중치 가산

### 10.4 시퀀싱 알고리즘 (Module Sequencing)

#### 10.4.1 역할 순서
- 기본 순서: warming → passage-use → practice → output
- 예외: Speaking 단독 세션에서는 warming 생략 가능

#### 10.4.2 인지 부하 곡선 최적화
- Bloom's level을 완만히 상승시키도록 배치
- 급격한 난이도 점프 방지

#### 10.4.3 지문 노출 일관성
- 한 레슨 내에서 passage exposure가 before → during → after 순으로 진행되도록 검증
- 역전(after 후 before 노출 금지)

#### 10.4.4 문항 유형 다양성
- 같은 `answerType` 모듈 3개 이상 연속 배치 방지
- 강제: essay 연속 2회 후 다른 유형 삽입

### 10.5 시간 예산 조정

```
total_estimated = Σ moduleSequence[i].estimatedMinutes
```

- total > timeBudget + 10%: 우선순위 낮은 모듈 제거 or 대체
- total < timeBudget − 10%: 심화/반복 모듈 추가
- 반복(최대 3회) 후 수렴 실패 시 강사 알림

### 10.6 강사 오버라이드 UI (§8.5 연동)

#### 10.6.1 Planner 결과 시각화
- 타임라인(바 차트) + 각 모듈 Rationale 팝오버

#### 10.6.2 드래그 기반 모듈 추가/삭제/순서 변경

#### 10.6.3 조건부 규칙 편집
- 예: "이전 모듈 점수 > 90 → 다음 모듈 스킵"
- 예: "정답률 < 50 → 재시도 모듈 자동 삽입"

#### 10.6.4 실시간 제약 위반 경고
- incompatible/missing prereq/시간 초과 인라인 표시

### 10.7 출력 스키마: LessonPlan

```json
{
  "lessonId": "lesson-123",
  "passageId": "passage-456",
  "passageAnalysis": { ... },
  "moduleSequence": [
    {
      "order": 1,
      "moduleCode": "PRD",
      "role": "warming",
      "passageExposed": false,
      "estimatedMinutes": 3,
      "rationale": "B2 레벨 학습자의 배경지식 활성화를 위해 예측하기 모듈 우선 배치"
    },
    { "order": 2, "moduleCode": "RRD", ... },
    { "order": 3, "moduleCode": "SWR", ... }
  ],
  "totalEstimatedMinutes": 12,
  "generatedAt": "2026-04-17T00:00:00Z",
  "plannerVersion": "v1.0",
  "editedByTeacher": false
}
```

### 10.8 검증 및 품질 관리

- **자동 검증**: 시간 합산, 역할 순서, 제약 준수 일괄 체크
- **A/B 생성**: 같은 입력에 대해 2안 제시 (10.12 API 참조)
- **학습 결과 피드백 루프**: 완료율·정답률 기반 모듈 가중치 주기적 조정

### 10.9 AI 호출 최적화

- 필터링(10.3)은 로컬 룰 기반 (비용 0)
- 시퀀싱(10.4)은 LLM Tool Use로 구조화 출력
- 프롬프트 스냅샷 저장 → 재현성 보장
- 캐싱: 동일 입력 해시 1회 이상 → 결과 재사용 (TTL 24h)

### 10.10 버전 관리

- LessonPlan 수정 이력 (Planner 자동 / 강사 편집 구분)
- 모듈 버전 변경 시 기존 LessonPlan 영향 평가 (§13.11 연동)

### 10.11 데이터 모델 (요약)

```sql
lessons (id, passage_id, learning_goal, target_level, module_sequence JSONB,
         total_minutes, created_at)
lesson_plans (id, lesson_id, version, generated_by, edited_by_teacher, payload JSONB)
planned_modules (id, lesson_plan_id, order, module_code, role, passage_exposed,
                 estimated_minutes, rationale)
planner_runs (id, request JSONB, response JSONB, tokens_used, cost,
              duration_ms, status, created_at)
```

### 10.12 API 엔드포인트

```
POST /api/lesson/plan              (자동 생성)
POST /api/lesson/plan/stream       (SSE)
POST /api/lesson/{id}/plan/edit    (강사 편집 저장)
POST /api/lesson/{id}/plan/ab      (A/B 제안)
GET  /api/lesson/{id}/plan/history (버전 이력)
```

### 10.13 KPI 및 관찰성

| 지표 | 목표 |
|---|---|
| 자동 수용률 (수정 없이 승인) | ≥ 50% |
| 평균 제안 시간 | < 5초 |
| 토큰 사용량 (1회 제안당) | < 3,000 |
| 학생 최종 완료율 (Planner 품질의 역지표) | ≥ 70% |
| 시간 예산 수렴률 (±10% 내) | ≥ 95% |

### 10.14 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 모듈 라이브러리 빈약 | 시퀀싱 다양성 저하 | v0.15 기준 28개 모듈 운영, Phase 2에서 미작성 11개 + 추가 5개 모듈 단계적 합류 (§13.1 레지스트리 참조) |
| LLM 비용 폭증 | 수익성 | 룰 기반 1차 필터 + LLM 최종 선별, 캐싱 |
| 강사 불신 | 자동 수용률 저하 | Rationale 상세 제공, 수정 자유도 보장, A/B 제안 |
| 레벨·제약 충돌 | 유효 모듈 0개 | 완화 규칙 (제약 가중치화), 강사 알림 후 수동 모드 전환 |
| 버전 호환성 | 기존 LessonPlan 파손 | 모듈 버전 고정 참조(§9.7과 같은 패턴) |

### 10.15 Speaking 콘텐츠 생성 모드 (3종) *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — Speaking AI 회화 상품의 **레슨 콘텐츠 생성 방식 3종**을 정의한다. 이는 §10.1 Tutoring 시퀀싱 엔진(passage→module sequence)과는 다른 별개의 흐름이며, **Speaking 단일 도메인 내 dialogue/mission 자동 생성 파이프라인**이다.

**기본 원칙: 사전 미션 기반 진행 (Pre-Set Mission Mode)**

회의에서 다음 두 가지 후보 중 **후자**로 확정:

| 안 | 진행 방식 | 채택 여부 |
|---|---|---|
| 안 A. **동적 변경** | 학습자 수준에 따라 수업 진행 중 콘텐츠가 실시간으로 변경됨 | ❌ 미채택 |
| 안 B. **사전 미션 기반** | 콘텐츠 생성 후에는 사전에 설정된 미션 시퀀스에 따라 진행 | ✅ **채택** |

→ 모든 콘텐츠 생성 모드(아래 3종)는 **수업 시작 전 단일 콘텐츠/미션 세트를 확정**하고, 이후 수업 중에는 그 미션을 따라간다. 학습자 수준 미스매치 발생 시 **사후 분석을 통해 다음 회차 콘텐츠에 반영**(피드백 루프).

#### 10.15.1 모드 ①: 교재 기반 자동 생성

| 항목 | 정책 |
|---|---|
| 입력 | 학습자가 사용 중인 **교재 (textbook)** 의 단원/챕터 |
| 레벨 | **교재 자체의 레벨 메타데이터를 그대로 사용** (별도 레벨 진단 불필요) |
| 출력 | 해당 단원의 핵심 표현·어휘·대화 시나리오 → SpeakingSession 미션 |
| 강점 | 학원·기관에서 사용 중인 기존 교재와 일치 — 강사 검수 부담 ↓ |
| 사용 예 | "Pagoda Power Speaking Vol. 2, Unit 5 — Travel" → AI가 Travel 시나리오 dialogue 생성 |

#### 10.15.2 모드 ②: 주제 & 레벨 선택 기반 자동 생성

| 항목 | 정책 |
|---|---|
| 입력 | **주제(Topic) + 학습자 레벨** |
| 레벨 | 학습자의 **현재 레벨**에 맞춰 어휘·문법·발화량 조정 |
| 출력 | 해당 주제·레벨에 적합한 dialogue + 미션 |
| 강점 | 교재 없이 빠른 시작, 흥미 기반 주제 선정 가능 |
| 사용 예 | "Job Interview, A2" → A2 수준 어휘로 면접 dialogue 생성 |

#### 10.15.3 모드 ③: 프리토킹 (Free Talking) — **보조 수단**

| 항목 | 정책 |
|---|---|
| 입력 | 명확한 주제 없이 자유 대화 |
| 레벨 | 학습자 레벨 (시스템 추정값) |
| 출력 | 미션 없이 자유 대화, 사후에 발화 데이터로부터 KPI 추출 |
| ⚠️ 제약 | **고레벨이 아닌 학습자에겐 학습 효과가 떨어질 수 있음** — 회의록 명시 |
| 권장 사용 | 정규 레슨이 아닌 **보조적 수단** (예: 주말 free 학습 슬롯, 워밍업) |
| 게이트 | B2 이상 학습자에게 우선 노출, A1~A2는 기본 비노출 (⚠️ 정책 검토 중) |

#### 10.15.4 3종 통합 비교

| 비교 축 | 모드 ① 교재 기반 | 모드 ② 주제+레벨 | 모드 ③ 프리토킹 |
|---|---|---|---|
| 레벨 결정 | 교재 메타데이터 | 학습자 현재 레벨 | 학습자 추정 레벨 |
| 미션 사전 설정 | ✅ | ✅ | ❌ (자유 대화) |
| 진행 방식 | 사전 미션 시퀀스 | 사전 미션 시퀀스 | 자유 대화 후 KPI 측정 |
| 수업 중 변경 | ❌ (사전 확정) | ❌ (사전 확정) | N/A |
| 정규 레슨 가능 | ✅ | ✅ | ⚠️ 보조 수단 |
| 권장 학습자 | 교재 사용자 | 모든 레벨 | B2+ |

> 📎 콘텐츠 생성 후 사전 미션의 **레벨 일치 검증/조정**은 §15 레벨 테스트(Achievement Test) 결과를 다음 회차에 반영하는 형태로 수행한다(피드백 루프).

---

## 11. 튜터링 (Tutoring · 학생용)

### 11.0 상품 운영 모델 *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 본 절은 Tutoring·Speaking 두 채널의 **운영·과금·수료·게이미피케이션 등 상품-수업 운영 정책의 통합 헤드라인**이다. 각 세부 정책은 §11.0~§11.12에 분산된다.

#### 11.0.1 운영 모델 개요

회의록(260413) 결정에 따라 픽클래스의 학습자 운영 모델은 다음 5축으로 정의된다.

| 운영 축 | 정의 | 세부 절 | 상태 |
|---|---|---|---|
| **상품 형태** | 1:1 회화(Speaking) / 통합 튜터링(R/W/L/S) / 외부 임베드 (§3.3.4) | §3.3, §3.3.4 | 📋/✅ |
| **콘텐츠 생성 방식** | 교재 기반 / 주제+레벨 / 프리토킹 (§10.15) + B2B 교재 업로드 (§8.3.5) | §10.15, §8.3.5 | 📋 |
| **과정 생성 모드** | 스피킹 only / 튜터링 only / 통합 (§8.3.6) — v0.16 신설 | §8.3.6 | 📋 |
| **수업 운영 형태** | 정규수강(요일 지정) / 자유수강(분수 한도) | §11.9 | 📋 |
| **수료·평가** | 4지표 기반 수료 + Level Test/Achievement Test | §11.10, §15 | 📋 |
| **운영 주체 (v0.16 신설)** | **픽클래스 자체 운영** / **B2B 제휴 모드** (§16) | §16 | 📋 |

> 🔁 **v0.16 추가 — B2B 제휴 분기**: 픽클래스 자체 운영(B2C·일반 B2B 직판)과 **B2B 제휴 모드**(제휴사가 운영을 자체 기획·개발, 픽클래스는 수업 모듈만 임베드 실행) 두 분기로 운영된다. B2B 제휴 모드의 책임 분리·인터페이스·콜백 정책은 **§16 B2B 제휴 운영 모델** 참조.

#### 11.0.2 학습자가 보는 상품 라인업 (예시)

| 라인업 | 운영 형태 | 콘텐츠 생성 | 수료 기준 | 노출 채널 |
|---|---|---|---|---|
| **A. 정규 1:1 회화 (교재 기반)** | 정규수강 (주 N회 요일) | 모드 ① 교재 기반 | 월 발화량+미션+출석 | 1:1 회화 사이트, 픽클래스 임베드 |
| **B. 자유 1:1 회화 (주제 기반)** | 자유수강 (분수 한도) | 모드 ② 주제+레벨 | 월 발화량+누적시간 | 1:1 회화 사이트, 픽클래스 임베드 |
| **C. 프리토킹 보조 슬롯** | 자유수강 부가 | 모드 ③ 프리토킹 | 부가 KPI 기록만 | 자유수강 옵션 (B2+) |
| **D. 통합 튜터링 (R/W/L/S)** | 정규수강 또는 자유수강 | passage 기반 시퀀싱 (§10) | 모듈 완료율 + KPI | 픽클래스 Tutoring 정규 |

#### 11.0.3 동적 변경 vs 사전 미션 — 정책 확정

> ✅ **확정 (회의록 260413)**: 콘텐츠 생성 후에는 **사전에 설정된 미션 기반으로 진행**된다. 수업 진행 중 학습자 수준에 따라 **동적으로 변경되지 않는다**.

학습자 수준 미스매치는 **사후 분석 → 다음 회차 콘텐츠 생성 시 반영**(피드백 루프) 형태로 보정한다. 이는 다음 두 가지 시스템 단순성 확보를 위함이다.

1. **콘텐츠 검수 가능성**: 사전 확정된 미션은 강사·기관이 사전 검수 가능 → 품질 보장
2. **운영 안정성**: 실시간 동적 변경은 LLM 비용·지연·일관성 리스크 ↑

> ⚠️ 이 정책은 **튜터링 시퀀싱 엔진(§10.4)** 과는 별개이다. 시퀀싱 엔진의 분기·완화 규칙은 **수업 시작 전** 단계에서만 적용된다.

### 11.1 서비스 개요

학생이 AI 튜터 **Pickle**의 안내를 받으며 모듈식 레슨을 진행하는 웹 기반 학습 플랫폼.

### 11.2 레슨 진입 → 모듈 진행 → 완료 여정

```
/modules/[lessonId] 진입
    ↓
LessonPlan 로드 → 첫 모듈 시작
    ↓
[ModuleOrchestratorAgent 실시간 의사결정 루프]
    ↓
모듈 완료 → 성과카드
    ↓
다음 모듈(3~4회 반복) → 레슨 완료
```

### 11.3 Pickle 상호작용 모델

- 격려 메시지, 스캐폴딩(LV1→LV4 힌트)
- 4요소 통제: UI 노출 / 콘텐츠 깊이 / 피드백 타이밍 / 다음 액션
- Tool 호출 단일 의도 원칙

### 11.4 실시간 피드백 시스템

| 유형 | 트리거 | 동작 |
|---|---|---|
| 즉시 교정 | 객관식 오답 | "틀렸어요" + 재시도 |
| 단계적 힌트 | 2회 이상 반복 오답 | LV1 → LV2 → LV3 → 정답 공개 |
| 홀리스틱 | Essay 제출 | Claude AI 평가 + 강점/개선 |
| 발음 평가 | Audio 제출 | 단어별 점수 + 평균 |

### 11.5 성과카드 및 KPI 시각화

- 모듈명 / 정답률 / 모듈별 KPI 차트
- "다음 모듈" CTA
- 최종 완료 시 평균 정답률 + 모듈 목록

### 11.6 Speaking 모듈 임베드 방식

- LessonPlan 내 Speaking 모듈 슬롯 추가 지원
- Orchestrator → SpeakingConversationAgent 핸드오프
- 완료 후 ModuleHistory에 세션 요약 저장(§14.10 참조)

### 11.7 모바일/데스크탑 UI 적응

- 데스크탑: 2컬럼 (지문/문항 + 피드백)
- 모바일: 탭 전환 (content / voice / feedback)

### 11.8 나만의 수업 — 학생 자유 학습 *(✅ 구현 완료 — v0.10 신설)*

> 학생이 강사가 만든 정규 과정에 의존하지 않고 **본인이 KPI/시간/레벨/주제를 직접 선택해 자유롭게 학습**하는 워크플로. 2026-04-25 백엔드 통합 완료.
>
> **참조**: `tutoring.picklass.com/docs/나만의수업_시퀀싱통합_20260425.md`

#### 11.8.1 진입 경로

```
[Tutoring 학생 메뉴]
   └─ "나만의 수업" 진입
        ↓
   /class/lesson-setup/custom (또는 동등 경로)
        ↓
   학생이 옵션 입력 → AI 지문 생성 + analyzer 시퀀싱 + 자동 과정 생성
        ↓
   "나만의 수업" 과정에 lesson 자동 추가 → 즉시 학습 진입 가능
```

#### 11.8.2 학생 입력 옵션 폼

| 항목 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| 주제 (topic) | text | (필수 입력) | 학생이 자유롭게 입력 |
| 단어 수 (wordCount) | number | 150 | 지문 길이 |
| **KPI 선택** | multi-select | `['VOCAB_RECOG']` | KPI_CATEGORY goal 멀티셀렉트 |
| **수업 시간 (durationMin)** | enum | 30 | `15 / 20 / 25 / 30분` |
| **CEFR 레벨 (targetCefrLevel)** | enum | (추정 레벨 자동 pre-fill) | `PreA1, A1, A2, B1, B2, C1, C2` |

**API 요청**

```typescript
// POST /lessons/create-custom
interface CreateCustomLessonRequest {
  topic: string;
  targetLevel?: number;            // 1~18 (기존)
  wordCount?: number;              // 기존
  selectedKpiCodes?: string[];     // 신규 (default: ['VOCAB_RECOG'])
  durationMin?: 15 | 20 | 25 | 30; // 신규 (default: 30)
  targetCefrLevel?: string;        // 신규 (default: 'B1')
}
```

**API 응답**

```typescript
interface CreateCustomLessonResponse {
  lessonId: string;
  courseId: string;             // "나만의 수업" 과정 ID
  passage: { title, wordCount, ... };
  analysis: { estimatedLevel, type, ... };
  moduleSequence: PlannedModule[];
  totalEstimatedMinutes: number;
  sequencing: {
    coverageScore: number;
    stageDistribution: { ... };
    fallbackModules: string[];
  };
}
```

#### 11.8.3 시퀀싱 일원화 — Studio와 동등성(Parity) 보장

`POST /lessons/create-custom`은 내부에서 **Studio와 동일한 analyzer 서버**를 호출. 같은 입력 → 같은 모듈 시퀀스가 보장됨.

- 변경 전: Gemini가 모듈 시퀀스 생성 (지문 생성과 모듈 결정이 함께)
- 변경 후: **지문은 Gemini, 모듈 시퀀스는 analyzer** 분리. 폴백 정책 폐지(2026-04-23).

#### 11.8.4 추정 레벨 시스템 *(✅ 구현 완료)*

학생의 학습 이력을 분석하여 **CEFR 레벨을 자동 추정**, 옵션 폼에 디폴트로 노출. 학생이 override 가능.

| 입력 데이터 | 산출 |
|---|---|
| 학생 누적 KPI 점수 | 추정 CEFR 레벨 (PreA1~C2) |
| 최근 학습 모듈 적중률 | 신뢰도 점수 |
| 직전 N회 세션 정답률 | 추세 (상승/유지/하락) |

**운영 KPI**: 추정 레벨에 대한 **학생 override 비율 < 30%** (시스템 정확도 검증 지표).

#### 11.8.5 학생 단순화 vs Studio 강사 UX 차이

| 항목 | Studio (강사용 lesson-setup) | Tutoring 나만의 수업 (학생용) |
|---|---|---|
| 옵션 폼 | KPI + duration + 모듈 직접 선택 | KPI + duration + CEFR (모듈 미노출, 자동 시퀀싱만 표시) |
| 모듈 칩 번호 배지 UI | ✓ 표시 | 미표시 (결과만 학습 진입) |
| 사전 채움 | 없음 (빈 상태 시작) | 추정 레벨 자동 pre-fill |
| analyzer 호출 트리거 | KPI + duration 모두 선택 시 | 학생 "수업 시작" 클릭 시 1회 |

> 📎 학생용 모듈 미리보기 UI는 v0.11 후속 검토 대상 (현재는 학습 결과만 표시).

### 11.9 수업 운영 (수강 형태) *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 회의록 결정에 따라 학습자 수강 형태는 **정규수강**과 **자유수강** 두 가지로 운영된다.

#### 11.9.1 정규수강 (Regular Class)

학습자가 **고정된 학습 요일**을 지정하고, 해당 요일에 정해진 회차의 수업을 진행하는 모드.

| 항목 | 정책 |
|---|---|
| 학습 단위 | 회차 (1회차 = 1 lesson) |
| 진행 일정 | 요일 지정 (예: 월·수·금) |
| 출석 체크 | **해당 차시를 해당 요일에 진행해야 출석으로 인정** |
| 수료 기준 | §11.10의 4지표 (출석률 포함) |
| 알림 | 학습 시간 직전 푸시·알림톡 자동 발송 (§11.12) |
| 결강 처리 | 결강 시 해당 회차는 미출석 처리, 보강 정책은 기관별 결정 |

**적합 대상**: 학원·기업 임직원 등 **루틴 학습이 중요한 B2B 학습자**.

#### 11.9.2 자유수강 (Self-Paced Class)

학습자가 **제공된 분수(시간) 한도** 내에서 자유롭게 수업을 진행하는 모드.

| 항목 | 정책 |
|---|---|
| 학습 단위 | 분 단위 누적 (예: 월 600분 한도) |
| 진행 일정 | 자유 (요일/시간 미지정) |
| 출석 체크 | **출석률 미적용** (수료 기준에서 제외) |
| 수료 기준 | §11.10의 3지표 (월 누적 시간/발화량/미션 완료율) |
| 알림 | 미진행 N일 경과 시 독려 푸시·알림톡 (§11.12) |
| 잔여 시간 | 학습자 페이지에서 실시간 노출 |

**적합 대상**: 자율 학습 가능한 **고급 학습자, B2C, 일부 K12 어드밴스드**.

#### 11.9.3 정규수강 ↔ 자유수강 비교

| 비교 축 | 정규수강 | 자유수강 |
|---|---|---|
| 일정 강제성 | 강 (요일 지정) | 약 (자유) |
| 단위 | 회차 | 분 |
| 출석률 KPI | ✅ 포함 | ❌ 미적용 |
| 알림 시점 | 사전 (수업 직전) | 사후 (미진행 누적) |
| 콘텐츠 모드 (§10.15) | 모드 ① 또는 ② | 모드 ②, ③ 가능 |
| 게이미피케이션 (§11.11) | 출석 streak | 누적 시간 보상 |

> 📎 정규/자유 선택은 **상품 라인업 단위**(§11.0.2)로 사전 결정된다. 학습자가 임의로 전환할 수 없으며, 기관 관리자가 §7.7 플랜 구성에서 정의한다.

### 11.10 수료 기준 (Completion Criteria) *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 회의록 결정에 따라 4개 정량 지표 기반 수료 시스템을 도입한다.

#### 11.10.1 4개 수료 지표

| 코드 | 지표명 | 측정 단위 | 데이터 소스 |
|---|---|---|---|
| **CMP_TIME** | **월 누적 수업시간** | 분 (Min) | SpeakingSession.duration 합산 (월 단위) |
| **CMP_UTT** | **발화량(문장 수)** | 문장 (Sent) | KpiResult.MLU_LENGTH × Turn 카운트 (월 단위) |
| **CMP_MISSION** | **미션 완료율** | % (0~100) | 사전 미션 N개 중 완료 M개 → M/N |
| **CMP_ATTEND** | **출석률** | % (0~100) | 정규수강 회차 중 출석 비율 (※자유수강 미적용) |

> ⚠️ 회의록 원안의 "발화시간"은 **"월 누적 수업시간"**으로 정정 (회의록 ~~취소선~~ 명시).

#### 11.10.2 고객사·레벨별 차등 적용

기관(고객사)은 **위 4지표를 다중 선택**하여 자체 수료 기준을 설정할 수 있으며, **레벨별 차등** 임계값을 지정할 수 있다.

**예시 1: A업체 - 단일 기준**

```
조건: 100문장 이상 발화 AND 미션 70% 이상 진행
→ 수료
```

**예시 2: A업체 - 레벨별 차등 (수업시간 기준)**

| 레벨 | 월 누적 수업시간 임계값 | 발화량 추가 조건 |
|---|---|---|
| 초급 (PreA1, A1) | 50분 이상 | + 미션 60% |
| 중급 (A2, B1) | 80분 이상 | + 미션 70% |
| 고급 (B2, C1+) | 100분 이상 | + 미션 80% |

#### 11.10.3 운영 흐름 — 파고다 ↔ 픽클래스 (오이지)

회의록 명시 흐름:

```
[파고다] 업체별/레벨별 수료 기준값 정의
    ↓ (기준값 전달)
[픽클래스(오이지)] 학습 데이터 측정
    ↓ (결과값 전송)
[파고다 학습자 페이지] 수료 기준 + 현재 진척도 실시간 노출
```

#### 11.10.4 학습자 페이지 노출 형식 (예시)

회의록 명시 예시 — A업체 Y학생, 초급:

```
📊 수료 진척도

수료 기준: 50분 이상 발화 & 미션 60% 이상 진행
─────────────────────────────────────────
현재 진척:
  · 누적 수업시간: ████████░░  42 / 50 분  (84%)
  · 미션 완료율:   █████████░  90 / 60 %   (150%) ✓
  · 발화량:        ████░░░░░░  78 / —      (참고)

⏱ 수료까지 8분 남음
```

#### 11.10.5 데이터 모델 (요약)

```sql
-- 기관별 수료 기준 설정
table completion_policies (
  id pk,
  institution_id fk,
  level_band enum('PreA1','A1','A2','B1','B2','C1+'),
  min_total_minutes int null,           -- CMP_TIME 임계
  min_utterance_sentences int null,     -- CMP_UTT 임계
  min_mission_complete_rate decimal(5,2) null,  -- CMP_MISSION 임계 (%)
  min_attendance_rate decimal(5,2) null,        -- CMP_ATTEND 임계 (%)
  combine_logic enum('ALL','ANY') default 'ALL',  -- AND or OR
  effective_from date,
  effective_to date null
);

-- 학습자별 월간 진척도 스냅샷
table completion_progress (
  id pk,
  user_id fk,
  policy_id fk,
  ym char(7),  -- '2026-04'
  total_minutes int,
  utterance_sentences int,
  mission_complete_rate decimal(5,2),
  attendance_rate decimal(5,2),
  is_completed bool,
  completed_at timestamp null,
  unique (user_id, policy_id, ym)
);
```

> 📎 외부 시스템(파고다)으로의 진척도 노출은 **§14.10.5 외부 verify** 흐름의 응답 페이로드에 포함되어 전달된다.

### 11.11 게이미피케이션 (Gamification) *(v0.11 신설 — 회의록 260413, 고도화)*

> 📋 **기획 단계 (고도화)** — 회의록 "추후 고도화 안건"에 기록된 게이미피케이션 3대 축. 참고 레퍼런스: 맥스AI, 헤이링.

#### 11.11.1 포인트 시스템 (Points)

| 항목 | 정책 |
|---|---|
| 포인트 적립 | 수업 완료 / 미션 완료 / 출석 streak / 특정 KPI 달성 |
| 포인트 차감 | 결강(정규수강) — 기관 옵션 |
| 외부 스토어 제휴 | **포인트 → 외부 파트너 스토어 상품 교환** (학습 유인책) |
| 만료 정책 | 적립 후 N개월 (기관별 설정) |

> ⚠️ 외부 스토어 제휴는 별도 BD(Business Development) 트랙으로 진행되며, 본 문서 v0.11에는 정책 골격만 기재.

#### 11.11.2 캐릭터 성장 (Character Growth — Tamagotchi 형태)

| 항목 | 정책 |
|---|---|
| 캐릭터 형식 | 다마고치 형태의 **AI 튜터 캐릭터(Pickle 또는 사용자 선택)** |
| 성장 트리거 | 수료율, 누적 학습시간, 발화량 등 KPI |
| 표정 변화 | **수료율에 따라 캐릭터 표정 변화** (행복/평온/우울) |
| 학습 동기 | 학습자가 캐릭터를 돌보는 동기 → 학습 지속성 향상 |
| 단계 | 알·새끼·청년·성숙 등 다단계 (UX 디자인 별도) |

#### 11.11.3 경쟁 요소 (Ranking — 기업별)

| 항목 | 정책 |
|---|---|
| 랭킹 단위 | **기업별(Institution 단위)** 학습 랭킹 |
| 랭킹 지표 | 누적 발화량 / 미션 완료율 / 수료율 / 학습 streak |
| 노출 범위 | 기업 내 학습자에게만 공개 (개인 식별성 옵션) |
| 보상 | 월간/분기 1·2·3위 포인트 보너스 (옵션) |
| 공정성 | 레벨별 분리 랭킹 (초급/중급/고급) — 옵션 |

> 📎 게이미피케이션 데이터는 §11.10 수료 시스템의 KPI를 재사용. 별도 신규 측정 지표는 도입하지 않는다.

### 11.12 푸시·알림 (Push & Notifications) *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계 (일부 가능 / 일부 OS 제약 확인 필요)** — 회의록 "수업 참여 독려 자동화" 항목.

#### 11.12.1 알림 채널

| 채널 | 트리거 | 가능 여부 |
|---|---|---|
| **앱 푸시 (FCM/APNS)** | 학습 직전, 미진행 누적, 수료 임박 등 | ✅ 표준 가능 |
| **알림톡 (카카오)** | 동일 | ✅ 가능 (별도 발송 비용) |
| **SMS** | 알림톡 미수신자 fallback | ✅ 가능 |
| **앱 푸시 (전화 수신 형태)** | 수업 시작 직전 | ⚠️ **OS 제약 확인 필요** (회의록 명시) |

#### 11.12.2 자동 트리거 룰 (예시)

| 트리거 ID | 조건 | 채널 | 메시지 예시 |
|---|---|---|---|
| `T_REGULAR_BEFORE` | 정규수강 수업 30분 전 | 앱 푸시 | "오늘은 영어 회화 수업 날입니다 🎯" |
| `T_FREE_INACTIVE` | 자유수강 3일 미진행 | 앱 푸시 + 알림톡 | "이번 달 잔여 200분! 오늘 5분이라도 학습해볼까요?" |
| `T_COMPLETION_NEAR` | 수료 기준 90% 도달 | 앱 푸시 | "수료까지 8분 남았습니다 ⭐" |
| `T_RANK_UP` | 기업 내 랭킹 5위 이상 도약 | 앱 푸시 | "이번 주 회사 내 영어 학습 TOP 3 진입! 🏆" |

#### 11.12.3 OS 제약 — 전화 수신 형태 푸시

회의록 명시 검토 사항:

> "APP 푸시를 전화 수신 형태로 자동 실행 가능 여부 문의 (OS 제약사항 확인 필요)"

| OS | 전화 수신 형태 푸시 | 설명 |
|---|---|---|
| iOS | ❌ 제약 | CallKit은 실제 통화·VoIP에만 허용. 학습 알림 용도 사용 시 Apple 심사 거부 위험 |
| Android | ⚠️ 부분 가능 | Full-screen Intent + Notification은 가능하나 Android 14+ 권한 사용자 동의 필요 |

→ **현실적 운영안**: 전화 수신 형태가 아닌 **풀스크린 푸시 + 진동·소리 강화**로 대체. **수료율에 따른 독려 앱 푸시·알림톡 자동발송은 현재 가능** (회의록 명시).

#### 11.12.4 학습자 옵트인 / 옵트아웃

- 사용자별 채널·트리거 단위 ON/OFF (학습자 페이지 설정)
- 기본값: 정규수강자는 사전 알림 ON, 자유수강자는 미진행 알림 ON
- 광고성 알림은 별도 동의 (마케팅 정보 수신 동의)

### 11.13 ❌ B2B 제휴 운영 모델 *(v0.19 챕터 승격 — §16 참조)*

> ❌ **v0.19에서 §16. B2B 제휴 운영 모델 챕터로 승격 이전**되었다. cross-cutting 운영 모델임이 명확해졌고 영업·계약 사용성이 강화됐다. 기존 §16.0~12 서브섹션은 §16.0~16.9, §16.11~16.13으로 이동, §16.10 채널별 적용 절(Tutoring·Speaking·외부 임베드)이 신규 추가되었다.

> 📎 상세는 **§16. B2B 제휴 운영 모델** 참조.

---

## 12. AI 튜터링 엔진 (Agent 아키텍처)

### 12.1 2단계 Agent 구조 개관

```
CurriculumPlannerAgent (Macro, 1회 호출)
    ↓ LessonPlan 생성
ModuleOrchestratorAgent (Micro, 매 상호작용)
    ↓ 4요소 통제 + Tool Call
[UI/콘텐츠/피드백/다음액션]
```

### 12.2 CurriculumPlannerAgent

- **입력**: PassageData + LearningGoal + TargetLevel + 시간제약
- **도구**: analyzePassage, filterModules, sequenceModules
- **출력**: LessonPlan (moduleSequence)

### 12.3 ModuleOrchestratorAgent

#### 12.3.1 4요소 통제
UI 노출 / 콘텐츠 선택 / 피드백 타이밍 / 다음 액션.

#### 12.3.2 Rule-Based 우선순위 (9단계)
1. IdleCheck (120초 무응답) → checkEngagement
2. Disengaged (참여도 낮음 + 180초) → signalReplan
3. RepeatWrongAnswer (2회+ 오답) → provideFeedback(힌트 상승)
4. HolisticFeedback (essay 제출) → provideFeedback(AI)
5. PronunciationFeedback (audio 제출) → provideFeedback(발음)
6. WrongAnswerFeedback (객관식 오답) → provideFeedback
7. CorrectAnswer (정답) → presentQuestion or celebrate
8. InitialEntry (최초 진입) → showPassage or presentQuestion
9. PresentNextQuestion (기본) → presentQuestion

#### 12.3.3 Tool Use 스키마
showPassage / presentQuestion / provideFeedback / checkEngagement / signalReplan / celebrate

### 12.4 FeedbackGenerationAgent (텍스트 SSE)

- Essay/Audio 피드백 스트리밍 (Server-Sent Events)
- 첫 토큰 지연(TTFT) < 1.5초 목표
- 실시간 타이핑 효과로 체감 지연 감소

### 12.5 SpeakingConversationAgent (신설)

#### 12.5.1 3-Phase 대화 플로우

| Phase | 단계 | 처리 방식 | 소요 |
|---|---|---|---|
| 1. 진입 | 세션 초기화 | 로컬 | 즉시 |
| 1. Opening | 첫 질문 + 힌트 3단계 생성 | AI API 1회 | ~800ms |
| 1. TTS | 첫 질문 음성 출력 | 로컬 TTS | 레벨별 WPM |
| 2. 순환 루프 | Step A~E 반복 | - | - |
| 3. 종료 | Session Summary | 로컬 | 즉시 |
| 3. KPI | 5지표 분석 | AI API 1회 | ~1,000ms |
| 3. 저장 | 대화 기록 DB | DB | ~200ms |

#### 12.5.2 통합 AI 처리 (1회 호출)
Step C에서 단일 API 호출로 다음 6개를 동시 수행:
1. 상황 판단 (답변/역질문/도움요청)
2. 문법/어휘/표현 분석
3. 피드백 생성
4. 응답 생성
5. 다음 질문 생성
6. 힌트 3단계 생성

**최적화 효과**: 2회/턴 → 1회/턴(50%↓), 응답시간 1,300ms → 800ms(38%↓), 세션당 비용 $0.057 → $0.049(14%↓)

#### 12.5.3 추임새 / 즉시 피드백 / 따라 읽기
- **300ms 내 추임새** ("Great!") → 즉각 반응 체감
- 문법 오류 있음: 텍스트 교정 + TTS 재생 + 따라 읽기(L1–12)
- 문법 오류 없음: 메인 응답만 TTS

#### 12.5.4 침묵 감지 · STT Confidence · 로컬 Fallback

| 이벤트 | 임계값 | 동작 | LLM 호출 |
|---|---|---|---|
| 침묵 1차 | 5초 | 💡 Hint 1 | 없음 |
| 침묵 2차 | 10초 | 📝 Hint 2 | 없음 |
| 침묵 3차 | 15초 | 🔄 Hint 3 + 질문 변경 | 없음 |
| STT Confidence | < 0.5 | "못 들었어요" 재요청 | 없음 |

#### 12.5.5 꼬리 질문 vs 질문 유도 전환
- AI 질문 3–4회 누적 후, L4+ 학생에게 질문 유도(주도권 역전)
- 레벨별 주도권 배분: L1–3 AI 100% → L16–18 학습자 90%

### 12.6 프롬프트 엔지니어링

**시스템 프롬프트 계층**:
1. 역할 정의 (You are Pickle…)
2. 교수법 지시문 (pedagogyInstruction, 모듈별 동적 삽입)
3. 피드백 스타일 (feedbackStyle)
4. 컨텍스트 스냅샷 (구조화, 자연어 최소화)
5. 도구 목록 (Tool Use 형식)

**프롬프트 스냅샷**: 세션 시작 시 고정, 진행 중 변경 미적용 → 일관성 보장.

### 12.7 컨텍스트 윈도우 관리

| 전략 | 적용 시점 | 방법 |
|---|---|---|
| Rolling Window | 항상 | 최근 N=20 메시지 유지 |
| 점진적 요약 | 메시지 > 20 | 오래된 블록 Claude 요약 |
| 구조화 컨텍스트 | 매 요청 | LearnerState 구조체 전달 |

**토큰 예산**: System ~500 / PassageAnalysis ~200 / History ~1,500 / 답안 ~300 / Tools ~400 → 합계 ~2,900

### 12.8 에러 처리 · Fallback · Guardrail

| 시나리오 | Fallback |
|---|---|
| API 타임아웃 (>5초) | Rule-Based 자동 전환 |
| Rate Limit | 지수 백오프 (1s→2s→4s, 최대 3회) |
| 잘못된 Tool Call | 스키마 검증 실패 → 기본 액션 |
| 빈 응답/파싱 실패 | 이전 결과 재사용 or 재시도 안내 |
| 컨텍스트 초과 | 강제 Rolling Window 축소 |

### 12.9 프롬프트 인젝션 방어

- 학생 입력을 user role로 분리 (system에 직접 삽입 금지)
- 입력 길이 제한: essay 2,000자 / 채팅 500자
- 특수 패턴("ignore previous instructions", XML 태그) 이스케이프

---

## 13. 통합 모듈 시스템 (Unified Module System)

> **v0.5 재편:** Tutoring과 Speaking에서 사용되는 모든 모듈을 **단일 레지스트리**로 통합 관리한다. 각 모듈은 서비스 범위(Tutoring/Speaking/Both), 스킬, 역할을 메타데이터로 가지며, §11 튜터링 LessonPlan과 §14 Speaking Tutor 세션 양쪽에서 이 레지스트리를 참조한다.
>
> **운영 컴패니언**: 실제 모듈 추가·수정 워크플로우는 엑셀 파일 `Picklass_모듈_카탈로그.xlsx` (필터·정렬·버전 이력 관리)로 운영한다. 본 장은 그 스냅샷·원칙·정책을 담는다.

### 13.1 마스터 모듈 레지스트리 *(⚠️ v0.12 — Module List v1.0 정렬)*

> ✅ **확정** — 본 §13.1은 **`Picklass_Module_List_v1.0.docx`**(2026-04-29 확정)를 단일 진실 공급원(Single Source of Truth)으로 한다. **총 28개 모듈** (v1.0 원본 29개 → v0.15 Speaking 7→6 통합 개편 반영, §13.1.5 참조), **4개 영역**(Vocabulary / Reading / Speaking / Writing). v0.9에서 신설했던 **Listening 독립 영역은 폐기**되었다(§13.1.6 폐기 모듈 표 참조).

> 📎 각 모듈의 **상세 학습 시나리오**와 **레벨별 학습량·콘텐츠·힌트 매트릭스**는 컴패니언 문서 **`Picklass_모듈_시나리오_상세.md`**에 분리 수록한다(v0.12에 맞춰 갱신 예정).
> 📎 각 모듈에 연결된 **KPI 코드**와 **레벨별 목표 벤치마크**는 **§14.6 KPI 측정 체계**와 컴패니언 문서 **`Picklass_KPI_체계_상세.md`** 참조.

**컬럼 범례** *(v0.12 정렬)*

| 컬럼 | 정의 | 값 |
|---|---|---|
| 코드 | 모듈 식별자 (3글자 자음 패턴) | 예: WRD, PRD, SUM |
| answerType | 답안 유형 | short-text / audio-record / click-match / essay / keyword-chips / sentence-select / sentence-explain / sentence-write / mixed / drag-drop |
| scoringMode | 채점 방식 | exact / holistic / pronunciation |
| passageMode | 지문 노출 모드 | full / hidden / preview / timed-blur / timed-select |
| qCount | 문항 수 | single / multi / content-driven |
| feedbackStyle | 피드백 양식 *(v0.12 부활)* | correct-wrong / strengths-weaknesses / sentence-explain / wpm-pronunciation |
| 작성 | DB·구현 상태 | DB등록 / 완료 / 미작성 / 추가 / 미정 / 제외 |

#### A. Vocabulary 계열 (7개)

| 코드 | 명칭 | answerType | scoringMode | passageMode | qCount | feedbackStyle | 작성 |
|---|---|---|---|---|---|---|---|
| **WRD** | Word Decker Reading | short-text | exact | full | multi | correct-wrong | ✅ 완료 |
| **WSD** | Word Decker Speaking | audio-record | pronunciation | full | multi | correct-wrong | 미작성 |
| **GMN** | Guessing Meaning | short-text | holistic | full | multi | strengths-weaknesses | ✅ 완료 |
| **WWB** | Word Web | short-text | holistic | full | multi | strengths-weaknesses | 미작성 |
| **IMG** | Image Match | click-match | exact | full | multi | correct-wrong | 추가 |
| **WDN** | Word DNA | — | — | — | — | — | 추가 *(상세 미정)* |
| **WFT** | Word Family Tree | short-text | exact | full | multi | correct-wrong | 추가 |

#### B. Reading 계열 (8개 active)

| 코드 | 명칭 | answerType | scoringMode | passageMode | qCount | feedbackStyle | 작성 |
|---|---|---|---|---|---|---|---|
| **PRD** | Prediction | essay | holistic | preview | single | strengths-weaknesses | ✅ 완료 |
| **SCN** | Scanning | keyword-chips | holistic | timed-blur | single | strengths-weaknesses | ✅ 완료 |
| **SKM** | Skimming | sentence-select | holistic | timed-select | single | strengths-weaknesses | ✅ 완료 |
| **CLR** | Clarification | sentence-explain | holistic | full | content-driven | sentence-explain | ✅ 완료 |
| **SUM** | Summarizing | essay | holistic | full | single | strengths-weaknesses | ✅ 완료 |
| **QAR** | QAR | mixed | holistic | full | multi | strengths-weaknesses | ✅ 완료 |
| **RRD** | Repeated Reading *(← Speaking RPR 이동)* | audio-record | pronunciation | hidden | multi | wpm-pronunciation | ✅ 완료 |
| **ORL** | Oral Reading | — | — | full | single | — | 미작성 |

> 🔁 **영역 이동**: 기존 Speaking 공용에 있던 `RPR(Repeated Reading)`이 **Reading 영역**으로 이동하여 코드 `RRD`로 변경되었다. 모듈 본질이 "지문 기반 반복 낭독"이라는 점에서 Reading 분류가 더 적합한 것으로 재분류됨.

#### C. Speaking 계열 (6개) *(⚠️ v0.15 — 모듈별 학습 시나리오 v1.1 반영)*

| 코드 | 명칭 | Pick-Speak 단계 | answerType | scoringMode | passageMode | qCount | feedbackStyle | 작성 |
|---|---|---|---|---|---|---|---|---|
| **LRN** | Learn & Study *(✦ 신규)* | 2단계 LEARN | — | — | full | — | — | 미작성 |
| **VLM** | Vocabulary Listening & Meaning *(← WSP 재설계)* | 3-1 어휘 워밍업 | click-match | exact | full | multi | correct-wrong | 미작성 |
| **EDR** | Expression Drill *(✦ LR/SHD/SXP 통합 + 3-stage)* | 3-2 정밀 드릴 | mixed | holistic | full | multi | strengths-weaknesses | 미작성 |
| **RPL** | Role-Play *(← RLP 코드 변경, PTT 방식)* | 4-1 APPLY Phase 1 | mixed | holistic | full | single | strengths-weaknesses | 미작성 |
| **FRT** | Free Talking | 4-2 APPLY Phase 2 | mixed | holistic | full | single | wpm-pronunciation | 미작성 |
| **OMP** | One Minute Presentation *(← OMS 명칭 통일)* | 4-3 APPLY Phase 3 | mixed | holistic | full | single | strengths-weaknesses | 미작성 |

> 🔁 **v0.15 핵심 변경 — 7→6 모듈로 통합 개편**:
> - **WSP → VLM 재설계**: 단어 스피킹의 ASR 정확도 한계와 학습 효과 약함을 해소하기 위해 **듣고 의미 확인 방식**(스피킹 X)으로 변경. 청각 입력 + 의미 인지 조합.
> - **LR/SHD/SXP → EDR 단일 모듈로 통합**: 3개 분리 모듈을 **EDR(Expression Drill) 단일 모듈**로 통합하고 내부에 **3-stage 진행**(Read → Fill-in → Expand) 도입. §13.7.1 시퀀싱 그룹 개념 폐기.
> - **RLP → RPL 코드 변경 + PTT 방식 도입**: Always-On 자동 인식 대신 Push-to-Talk 방식으로 학습 흐름 통제권 확보.
> - **OMS → OMP 명칭 통일**: One Minute **Presentation** (Topic 표현 폐기, 발표 본질 명확화).
> - **LRN(Learn & Study) 신규 모듈**: 기존 시스템 단계였던 LEARN이 **콘텐츠 입력 + 표현 5개 카드 흡수**의 모듈로 정식화.
> - **RST(Response Training) ❌ 폐기 유지** (v0.14 결정).
> - 총 모듈 수: **7→6**

> 📎 6개 모듈의 7단계 상세 학습 시나리오, 0단계 데이터 준비, 멘트 사전 생성 데이터, 레벨별 매트릭스는 컴패니언 문서 **`Picklass_모듈_시나리오_상세.md`** v1.1 참조.

> 📎 Pick-Speak Method 5단계 학습 흐름·6 모듈 매핑·시퀀싱 알고리즘·6대 차별 메커니즘·**5축 KPI**(발음정확도·유창성·문법정확성·화용성·**발화량 추가**)·모바일 3분할 화면 표준 상세는 **§14.4** 참조.

#### D. Writing 계열 (6개)

| 코드 | 명칭 | answerType | scoringMode | passageMode | qCount | feedbackStyle | 작성 |
|---|---|---|---|---|---|---|---|
| **UNS** | Unscramble | drag-drop | exact | full | multi | correct-wrong | 추가 |
| **CPW** | Copy Writing | short-text | exact | full | multi | correct-wrong | 미작성 |
| **SCP** | Sentence Completion | mixed | exact | full | multi | correct-wrong | 미작성 |
| **SWR** | Sentence Writing | sentence-write | holistic | hidden | multi | strengths-weaknesses | ✅ 완료 |
| **PPR** | Paraphrasing | essay | holistic | full | multi | strengths-weaknesses | 추가 |
| **PWR** | Process Writing | essay | holistic | full | multi | strengths-weaknesses | 미작성 |

#### 13.1.5 전체 통계 (v0.15 기준)

- **총 모듈 수: 28개** (Vocabulary 7 / Reading 8 active / Speaking 6 / Writing 6 + 제외 1)
- **Listening 영역**: ❌ **폐기** (v0.9 신설 → v0.12 제거)
- **v0.15 변경**: Speaking 영역 7→6 모듈 통합 개편 — WSP→VLM 재설계, LR/SHD/SXP→EDR 통합, RLP→RPL 코드 변경 + PTT, OMS→OMP, **LRN(Learn & Study) 신규**
- **작성 상태별**:

| 상태 | 개수 | 모듈 |
|---|---|---|
| ✅ **완료** (DB 등록 + 구현 완료) | 10 | WRD, GMN, PRD, SCN, SKM, CLR, SUM, QAR, RRD, SWR |
| **미작성** (등록 필요) | 11 | WSD, WWB, ORL, **LRN**, **VLM**, **EDR**, **RPL**, FRT, **OMP**, CPW, SCP, PWR — Speaking 6 + 기타 |
| **추가** (신규 합류) | 5 | IMG, WDN, WFT, UNS, PPR |
| **미정** | 0 | (v0.14의 OMS 미정 → v0.15에서 OMP로 사양 확정) |
| **제외** (폐기 결정 명시) | 2 | SHR, PRC |

#### 13.1.6 폐기 모듈 표 *(v0.12 신설, v0.14·v0.15 갱신)*

다음 모듈은 **Module List v1.0 / Pick-Speak Method / 학습 시나리오 v1.1 정합화 결과 폐기 또는 통합 흡수**되었다.

| 폐기 코드 | 영역 | 명칭 | 폐기 시기 | 폐기·통합 사유 |
|---|---|---|---|---|
| **COL** | Vocabulary | Collocation Trainer | v0.12 | 모듈 우선순위 재조정 — v1.0 미포함 |
| **SHR** | Reading | Shadow Reading | v0.12 | "낭독 반복" 개념이 RRD(Repeated Reading)로 통합·대체 |
| **PWT** | Writing | Prewriting (= Summarizing 쓰기 측면) | v0.12 | SUM(Summarizing)과 기능 중첩 → 통합 |
| **TTS-SIM** | Speaking 독립 | Test-Taking Simulation | v0.12 | Speaking 핵심 모듈 우선순위 재조정 |
| **FIB** | Speaking 공용 | Fill-in-the-Blank | v0.12 | Writing SCP(Sentence Completion)와 기능 중첩 |
| **PIC** | Speaking 공용 | Picture Description | v0.12 | 시나리오 기반 FRT/RPL에 통합 가능 |
| **KED** | Speaking 공용 | Key Expressions Drill | v0.12 | EDR(LR/SHD/SXP 통합) 발화 훈련 모듈로 분산 |
| **QAR-S** | Speaking 공용 | QAR (Speaking) | v0.12 | QAR(Reading)에 음성 응답 옵션으로 통합 가능 |
| **LGS** | Listening | Listening for Gist | v0.12 | Listening 영역 폐기에 따라 |
| **DIC** | Listening | Dictation | v0.12 | Listening 영역 폐기에 따라 (Writing CPW에 일부 흡수 가능) |
| **LQR** | Listening | Listening QAR | v0.12 | Listening 영역 폐기에 따라 |
| **DLT** | Listening | Discriminative Listen | v0.12 | Listening 영역 폐기에 따라 |
| **PRC** | Writing | Paragraph Construction | v0.12 | Module List v1.0 명시 제외 |
| **RST** | Speaking | Response Training | v0.14 | Pick-Speak Method 5단계 흐름에 위치 없음 |
| **WSP** | Speaking | Word Speaking | **v0.15 (재설계)** | **VLM(Vocabulary Listening & Meaning)으로 재설계** — 단어 ASR 한계 → 듣고 의미 확인 방식 |
| **LR** | Speaking | Listen & Repeat | **v0.15 (통합)** | **EDR(Expression Drill) 단일 모듈로 통합 흡수** (3-stage Read 단계에 기능 포함) |
| **SHD** | Speaking | Shadowing | **v0.15 (통합)** | **EDR 단일 모듈로 통합 흡수** (Read 단계의 정확 발음 평가에 기능 포함) |
| **SXP** | Speaking | Sentence Expansion | **v0.15 (통합)** | **EDR 단일 모듈로 통합 흡수** (3-stage Expand 단계에 기능 포함) |
| **RLP** | Speaking | Role-Playing | **v0.15 (코드 변경)** | **RPL(Role-Play)로 코드 변경 + PTT 방식 도입** — 모듈은 활성 유지, 코드만 변경 |
| **OMS** | Speaking | One-Minute Topic Speaking | **v0.15 (코드 변경)** | **OMP(One Minute Presentation)로 명칭 통일** — 모듈은 활성 유지, 코드만 변경 |

> 🔁 **v0.15 핵심 변경 — 통합 vs 코드 변경 구분**:
> - **재설계**(WSP→VLM): 모듈 본질 자체가 바뀜 (스피킹 → 듣기·의미 확인)
> - **통합 흡수**(LR/SHD/SXP→EDR): 3개 모듈이 1개 모듈의 3-stage 내부 단계로 흡수 (단순 폐기 아님)
> - **코드 변경**(RLP→RPL, OMS→OMP): 모듈 활성 유지, 명칭/방식만 갱신
> 
> EDR Stage 매핑: **Stage 1 Read** = LR + SHD 정확 발화 측면 / **Stage 2 Fill-in** = (신규 블랭크 채우기) / **Stage 3 Expand** = SXP 문장 확장 측면

> 📎 폐기·통합된 모듈의 KPI는 §14.6 KPI 코드 레지스트리에서 **EDR/VLM/RPL/OMP의 통합 KPI로 재할당**된다. 상세 매핑은 컴패니언 `Picklass_KPI_체계_상세.md` v0.15 갱신 시 정리.

#### 13.1.7 v0.12 주요 변경 요약

| 변경 유형 | 내용 | 개수 |
|---|---|---|
| **모듈 폐기** | COL, SHR, PWT, TTS-SIM, FIB, PIC, RLP, KED, QAR-S, LGS, DIC, LQR, DLT, PRC | 14 |
| **모듈 신규** | WSP(Word Speaking), FRT(Scenario Based Free Talking — CMP 재정의 포함) | 2 |
| **영역 이동** | RPR(Speaking) → RRD(Reading) | 1 |
| **코드 표준화 (3글자 자음 패턴)** | WDR→WRD, WDS→WSD, GSM→GMN, WFA→WFT, SMZ→SUM, ORD→ORL, QAR-R→QAR, LRP→LR, SEX→SXP, SHR-S→SHD, RSP→RST, M1T→OMS, USC→UNS, CPY→CPW, STC→SCP, PAR→PPR | 16 |
| **영역 폐기** | Listening 독립 영역 (4개 모듈 동시 폐기) | 1영역 |
| **스키마 확장** | `feedbackStyle` 부활 (4종), `answerType` 10종 enum 표준화, `scoringMode` 3종 신규, 작성 상태 6분류 | — |

> 📊 **엑셀 카탈로그 연동**: 본 표의 모든 컬럼은 `Picklass_모듈_카탈로그.xlsx` (v0.5)의 **Master** 시트와 1:1 동기화된다. 영역별 필터 뷰(Vocabulary / Reading / Speaking / Writing) 및 폐기 모듈 시트, 작성 상태 시트가 함께 제공된다.

### 13.2 서비스 범위 분류 (Service Scope) *(v0.12 정렬)*

2가지 범위 체계로 모듈의 사용처를 명시한다. v0.11의 "Speaking(독립)/Speaking(공용)" 구분은 v1.0에서 단순화되어, **Speaking 모듈은 모두 통합 임베드 + 독립 운영 모두 가능**한 형태로 운영된다.

| 서비스 범위 | 사용처 | 모듈 (28개 기준, v0.15) |
|---|---|---|
| **Tutoring** (21) | Tutoring LessonPlan 내에서만 | Vocabulary 7 (WRD·WSD·GMN·WWB·IMG·WDN·WFT), Reading 8 (PRD·SCN·SKM·CLR·SUM·QAR·RRD·ORL), Writing 6 (UNS·CPW·SCP·SWR·PPR·PWR) |
| **Speaking** (6) | Tutoring LessonPlan 임베드 + 독립 앱 전용 모드 모두 운영 | **LRN, VLM, EDR, RPL, FRT, OMP** *(v0.15: WSP→VLM 재설계, LR/SHD/SXP→EDR 통합, RLP→RPL, OMS→OMP, LRN 신규)* |

**임베드 규칙 (v0.12 단순화)**
- 모든 **Speaking 모듈**은 Tutoring LessonPlan의 `moduleSequence`에 포함 가능하며, 진입 시 §12.5 `SpeakingConversationAgent`로 제어권 핸드오프된다.
- **FRT(Scenario Based Free Talking)** 등 시나리오 기반 모듈은 독립 앱에서 단독 실행도 가능하다 (대화 결과는 `SpeakingSession` 테이블에 저장).
- `Tutoring` 전용 모듈(Vocabulary/Reading/Writing 21개)은 Speaking 독립 앱에서 호출 불가.

### 13.3 스킬 분류 *(v0.12 정렬)*

한 모듈이 **복수 스킬**을 가질 수 있다 (예: RRD = Reading + Speaking).

| 스킬 | 주요 모듈 | 합계 |
|---|---|---|
| **Reading** | PRD, SCN, SKM, CLR, SUM, QAR, RRD, ORL, GMN, WDN | 10 |
| **Vocabulary** | WRD, WSD, GMN, WWB, IMG, WDN, WFT | 7 |
| **Writing** | UNS, CPW, SCP, SWR, PPR, PWR, SUM | 7 |
| **Speaking** | WSD, LRN, VLM, EDR, **RPL**, FRT, OMP, RRD, ORL | 9 |

> Vocabulary는 Reading의 하위 태그로 운영(상호 중복 허용). 모듈은 주 스킬 + 부 스킬 조합으로 태깅된다. **Listening 독립 스킬은 v0.12에서 폐기**되었으며, 듣기 활동은 Speaking·Reading 내 보조 활동으로 흡수된다.

### 13.4 역할 / 단계 분류 — 두 축 병행 *(⚠️ 재정의)*

> **재정의 (v0.10)**: 모듈 분류 축은 **2축 병행 운영**한다. 4-Role(교수법 학술 분류)과 3-Stage(실제 운영 분류)는 보는 관점이 다르며, 둘 다 유지한다.

#### 13.4.1 4-Role (교수법 학술 분류) *(📋 기획)* *(v0.12 갱신)*

학습 흐름 안에서의 **교수법적 기능**을 4가지로 분류. CurriculumPlannerAgent의 시퀀싱 알고리즘에서 인지 부하 곡선의 이론적 근거로 사용.

| 역할 | 목적 | 모듈 (28개 기준, v0.15) |
|---|---|---|
| **warming** | 도입 (어휘·배경지식·예측·파닉스) | PRD, WRD, WSD, IMG, UNS, CPW, LRN, VLM |
| **passage-use** | 본격 읽기·듣기 (스킬 적용) | SCN, SKM, ORL, RRD |
| **practice** | 연습 (이해도 확인·드릴·분석) | GMN, WWB, WDN, WFT, QAR, CLR, SCP, EDR |
| **output** | 산출 (창작·발화·요약) | SUM, SWR, PPR, PWR, RPL, FRT, OMP |

#### 13.4.2 3-Stage (실제 운영 분류) *(✅ 구현 완료)* *(v0.12 갱신)*

실제 DB·analyzer에서 운영되는 분류. **`classBefore` / `classMiddle` / `classAfter` Boolean 플래그 3개**가 모듈마다 부여되며, 시퀀싱 알고리즘은 이 3-Stage를 **1순위 정렬 키**로 사용한다.

| 수업 단계 | DB 플래그 | 의미 | 모듈 (28개 기준, v0.15) |
|---|---|---|---|
| **Pre-class** | `classBefore: true` | 수업 전 준비·도입·파닉스 | IMG, WRD, WSD, UNS, CPW, PRD, LRN, VLM |
| **During-class** | `classMiddle: true` | 수업 중 본격 학습·드릴 | GMN, WWB, WDN, WFT, SCN, SKM, QAR, CLR, ORL, RRD, EDR, SCP |
| **Post-class** | `classAfter: true` | 수업 후 정리·산출·확장 | SUM, SWR, PPR, PWR, RPL, FRT, OMP |

> 한 모듈이 복수 Stage Boolean을 동시에 `true`로 가질 수도 있음 (예: `classBefore=true, classMiddle=true` → 도입·중반 어디서든 활용 가능).

#### 13.4.3 4-Role ↔ 3-Stage 자동 매핑

4-Role을 3-Stage로 변환할 때 다음 규칙을 따른다:

```
warming      ──► Pre-class       (직선 매핑)
passage-use  ─┐
              ├──► During-class  (병합 — 두 역할이 같은 단계에 들어감)
practice     ─┘
output       ──► Post-class      (직선 매핑)
```

**역방향(3-Stage → 4-Role)은 불가능**하다. During-class만 알고는 그것이 passage-use인지 practice인지 결정할 수 없으므로, 4-Role 정보를 별도 보존해야 한다.

| 운영 시나리오 | 사용하는 분류 |
|---|---|
| analyzer 시퀀싱 1순위 정렬 | **3-Stage** |
| 강사 교수법 가이드·문서 | **4-Role** |
| 모듈 카탈로그 검색 필터 | 둘 다 노출 |
| 학생 진도 대시보드 | **3-Stage** (이해 직관성 우선) |

> 📎 두 분류의 개념 차이는 §13.4 도입부 정의를 참조하고, 실제 DB 컬럼 매핑은 **`Picklass_모듈_카탈로그.xlsx` v0.4의 Master 시트**에서 두 축 모두 표시.

### 13.5 모듈 프로퍼티 스키마 *(⚠️ 재정의 — v0.12 Module List v1.0 정렬)*

> v0.10에서 ❌폐기로 분류했던 `feedbackType`이 **`feedbackStyle`로 부활**(v1.0 결정). 폐기 결정이 번복된 것이 아니라, 구형 stage 시스템 잔재였던 `feedbackType`은 폐기되고, **새로운 표준화 필드 `feedbackStyle`이 4종 enum**(correct-wrong / strengths-weaknesses / sentence-explain / wpm-pronunciation)으로 도입된 형태이다. `answerType` 또한 자유 문자열에서 **10종 enum**으로 표준화되었다.

```typescript
interface AiModule {
  // ── 식별·메타 ────────────────────────────────────
  code: string;                   // 'PRD', 'QAR', 'SHD', 'IMG' 등 — v0.12 3글자 자음 패턴
  name: string;
  skill: string;                  // 'Reading' | 'Vocabulary' | 'Speaking' | ...
  serviceScope: "Tutoring" | "Speaking-standalone" | "Speaking-shared";

  // ── 단계 분류 (3-Stage Boolean ✅ 구현 완료) ─────
  classBefore: boolean;           // Pre-class 적합
  classMiddle: boolean;           // During-class 적합
  classAfter: boolean;            // Post-class 적합

  // ── 4-Role 분류 (📋 기획 — 학술용 별도 보존) ────
  roles: string[];                // ['warming', 'practice', ...]

  // ── 지문 노출 모드 ✅ 구현 완료 (v0.10 재정의) ───
  passageMode:
    | "full"           // 처음부터 전체 공개 (기본)
    | "hidden"         // 지문 패널 없음 (WRD, WSD)
    | "preview"        // 첫 단락만 → 답변 후 전체 공개 (PRD)
    | "timed-blur"     // N초 전체 노출 → blur 처리 (SCN)
    | "timed-select";  // N초 후 문장 선택 모드 (SKM 등 예정)

  // ── 문항 설계 ✅ 구현 완료 (v0.10 신설) ──────────
  questionCount: "single" | "multi" | "content-driven";  // content-driven: CLR 등 지문 전체 추출
  questionMinCount: number;
  questionMaxCount: number;
  questionMaxAttempts: number;    // 문항당 재시도 횟수

  // ── 힌트 (✅ 구현 완료 · v0.10 다중 선택으로 변경) ─
  hintTypes: string[];            // ['syllable-guide','meaning-hint','english-definition' ...]
                                  // 빈 배열 또는 ['none'] = 힌트 없음

  // ── UI/문항 진행 (✅ 구현 완료 — v0.10 신설) ──────
  uiTemplate:
    | "standard"   // 기본 QuestionsPanel
    | "voice"      // VoiceQuestionPanel (audio-record)
    | "embedded"   // 지문에 문항 임베드 (CLR sentence-explain)
    | "hidden";    // 향후 확장용
  questionFlowMode:
    | "sequential"   // 단문항 전용 — 1개씩 표시 (기본)
    | "deck"         // 다문항 전용 — activeQuestionId 기준 카드 1개씩 순환
    | "locked-steps"; // 단문항 전용 — 전체 표시, 이전 완료 후 다음 활성화

  // ── 콘텐츠 생성 지시 (✅ 구현 완료 — v0.10 신설) ─
  contentGenerationInstruction: string;  // 모듈별 LLM 프롬프트 instruction
                                         // (예: GMN의 "단어 1개 = 문항 1개" 규칙)
  questionGenerationStrategy: "extract" | "instruct";

  // ── 채점·피드백 (✅ v0.12 enum 표준화) ─────────────
  answerType:                     // v1.0 표준화 10종 enum
    | "short-text"
    | "audio-record"
    | "click-match"
    | "essay"
    | "keyword-chips"
    | "sentence-select"
    | "sentence-explain"
    | "sentence-write"
    | "mixed"
    | "drag-drop";
  scoringMode: "exact" | "holistic" | "pronunciation";  // v1.0 3종 enum
  feedbackStyle:                  // ✅ v0.12 부활 — v1.0 4종 enum
    | "correct-wrong"             // 객관식·매칭형 — 즉시 정오답 표시
    | "strengths-weaknesses"      // 서술·요약형 — AI 강점/약점 분석
    | "sentence-explain"          // CLR — 문장 단위 LLM 설명
    | "wpm-pronunciation";        // 발화·낭독형 — WPM + 발음 점수
  inputLanguage?: string;         // 'korean-or-english' 등
  passageRole?: string;           // 지문이 모듈 내에서 어떤 역할을 하는지

  // ── KPI 매핑 (✅ 구현 완료 — courses.kpi_codes 연동) ─
  selectedKpiCodes: string[];     // KPI_CATEGORY 코드 배열 (예: ['FLUENCY_RATE','PRAGMATICS'])

  // ── 시퀀싱 메타 ───────────────────────────────────
  cognitiveLevel: number;         // Bloom's 1-6
  suitableLevels: { min, max };   // CEFR 1–18
  estimatedMinutes: { min, max };
  prerequisites: string[];        // 선행 모듈 코드
  incompatibleWith: string[];     // 양립불가 모듈 코드
  priority: number;               // 동일 cognitiveLevel 내 운영 우선순위

  // ── 라이프사이클 ──────────────────────────────────
  status: "MVP" | "활성" | "예정" | "단종";
  phase: "P1" | "P2" | "P3";
  version: string;                // 'v1.0'
  contextObjective: string;       // "현실적 학습 목표"
  pedagogyInstruction: string;    // AI 시스템 프롬프트 삽입

  // ── ❌ 폐기 필드 (v0.10 명시 + v0.12 갱신) ─────────
  // passageExposureMode: ❌ DROP — passageMode로 통합 (GenericAdapter에 브릿지 코드만 잔존)
  // feedbackType:        ❌ DROP — 구형 stage 시스템 잔재 → feedbackStyle로 재정의
  // retryThreshold:      ❌ DROP — questionMaxAttempts로 대체
  // learning_objectives: ❌ DROP — KPI_CATEGORY로 통합
}
```

**신설 속성 상세 — v0.10 + v0.12 갱신**

| 필드 | 상태 | 설명 |
|---|---|---|
| `passageMode` | ✅ 구현 완료 | 지문 노출 5종 모드. v0.9의 `passageExposureMode`를 대체 |
| `hintTypes[]` | ✅ 구현 완료 | 다중 선택 (이전: CSV 단일 문자열) |
| `questionCount: 'content-driven'` | ✅ 구현 완료 | CLR 등 지문 전체 문장 추출 모듈용 |
| `uiTemplate` | ✅ 구현 완료 | 문항 패널 동작 모드 (standard/voice/embedded/hidden) |
| `questionFlowMode` | ✅ 구현 완료 | 문항 진행 방식 (sequential/deck/locked-steps) |
| `contentGenerationInstruction` | ✅ 구현 완료 | 모듈별 LLM 프롬프트 instruction (GMN, CLR 등에 사용) |
| `selectedKpiCodes` | ✅ 구현 완료 | KPI_CATEGORY 코드 배열, `courses.kpi_codes`와 연동 |
| `classBefore/Middle/After` | ✅ 구현 완료 | 3-Stage 분류 (§13.4.2) |
| **`answerType` (10종 enum)** | **⚠️ v0.12 표준화** | 자유 문자열 → enum 표준화 (Module List v1.0) |
| **`scoringMode` (3종 enum)** | **⚠️ v0.12 표준화** | exact / holistic / pronunciation 3종으로 축소 |
| **`feedbackStyle` (4종 enum)** | **✅ v0.12 부활** | v0.10에서 ❌폐기된 `feedbackType` 자리에 표준화 양식으로 재도입 |

**v0.9에 있었으나 변경/축소된 필드**

| v0.9 필드 | v0.10 상태 | 비고 |
|---|---|---|
| `classPhase` | ⚠️ 재정의 | 단일 enum이 아닌 **3-Stage Boolean 3개**로 운영 |
| `passageExposureMode` | ❌ 폐기 | DB DROP, GenericAdapter 브릿지로만 잔존 (점진 제거 예정) |
| `automationLevel` | 📋 기획 단계 | 본문 정의는 유지, 실제 DB 컬럼 미반영 (KPI별 측정 도구 분류로 대체) |

#### 13.5.1 AiModuleFlags — Orchestrator 추상화 레이어 *(✅ 구현 완료)*

Tutoring `useModuleOrchestrator.ts`가 `buildModuleFlags(profile)`로 모듈 속성을 읽기 쉬운 Boolean 플래그 셋으로 변환. Orchestrator 규칙에서 직접 `profile.scoringMode === 'holistic'` 비교를 피하고 `flags.hasRightWrongConcept` 같은 플래그를 참조하도록 추상화.

```typescript
interface AiModuleFlags {
  isPassageHidden: boolean;       // passageMode === 'hidden'
  isPassagePreview: boolean;      // passageMode === 'preview'
  isPassageTimedBlur: boolean;    // passageMode === 'timed-blur'
  isVoiceModule: boolean;         // answerType === 'audio-record'
  isMultiQuestion: boolean;       // questionCount !== 'single'
  hasRightWrongConcept: boolean;  // scoringMode === 'exact'
  hasRetry: boolean;              // questionMaxAttempts > 1
  hasHints: boolean;              // hintTypes.length > 0 && !['none']
  // (계획 대비 일부 미구현 — passageVisible/isAudioInput 등은 후속 작업)
}
```

> 📎 미구현 flags 및 기술부채는 `picklass-backoffice/docs/ai-modules/20260426_AI모듈필드재분류_완료.md` §6 참조.

### 13.6 문항 유형 및 채점 방식 *(⚠️ 재정의 — v0.12 Module List v1.0 정렬)*

**문항 유형 (answerType) — v1.0 표준화 10종 enum**

| 유형 | 설명 | 사용 모듈 (v0.12 기준) |
|---|---|---|
| `short-text` | 단답형 (단어·짧은 표현) | WRD, GMN, WWB, WFT, CPW |
| `audio-record` | 음성 녹음 | WSD, RRD |
| `click-match` | 이미지/카드 매칭 클릭 | IMG |
| `essay` | 서술·요약형 | PRD, SUM, PPR, PWR |
| `keyword-chips` | 키워드 칩 선택 (스캐닝) | SCN |
| `sentence-select` | 문장 선택 (스키밍) | SKM |
| `sentence-explain` | 문장 단위 설명 (uiTemplate=embedded) | CLR |
| `sentence-write` | 영작문 한 문장 단위 | SWR |
| `mixed` | 혼합 (객관식+발화 또는 객관식+서술 등) | QAR, EDR, RPL, FRT, OMP, SCP |
| `drag-drop` | 드래그앤드롭 배열 | UNS |
| _(미정/미작성)_ | 속성 결정 대기 | LRN, ORL, WDN |

**채점 방식 (scoringMode) — v1.0 3종 enum**

| 방식 | 설명 | 사용 모듈 |
|---|---|---|
| `exact` | 문자열 정확 비교 (객관식·단답·드래그·매칭) | WRD, IMG, WFT, UNS, CPW, SCP |
| `holistic` | AI 평가 (서술·요약·다관점 분석) | GMN, WWB, PRD, SCN, SKM, CLR, SUM, QAR, SWR, PPR, PWR, EDR, RPL, FRT, OMP |
| `pronunciation` | 발음 점수 (ASR + 음향 분석) | WSD, RRD |

> v0.10의 `mixed` 채점은 v0.12에서 제거되었다. 혼합 채점이 필요한 경우 `answerType: mixed`로 답안만 혼합하고 채점은 `holistic`이 일괄 처리한다.

**피드백 양식 (feedbackStyle) — v1.0 4종 enum (✅ v0.12 부활)**

| 양식 | 설명 | 사용 모듈 |
|---|---|---|
| `correct-wrong` | 즉시 정오답 표시 (객관식·단답·매칭형 적합) | WRD, WSD, IMG, WFT, UNS, CPW, SCP |
| `strengths-weaknesses` | AI 강점/약점 분석 (서술·요약·다관점) | GMN, WWB, PRD, SCN, SKM, SUM, QAR, SWR, PPR, PWR |
| `sentence-explain` | 문장 단위 LLM 설명 (CLR uiTemplate=embedded) | CLR |
| `wpm-pronunciation` | WPM(분당 단어 수) + 발음 점수 (낭독·발화 적합) | RRD, FRT |

> v0.10 changelog에서 `feedbackType`을 ❌폐기했던 결정은 **v0.12에서 부활**된 것이 아니라, 구형 stage 시스템 잔재였던 `feedbackType`은 폐기되고 **새로 표준화된 `feedbackStyle` 필드(4종 enum)** 가 도입된 것이다.

**❌ 폐기된 필드 (v0.10 명시 + v0.12 유지)**

| 폐기 필드 | 폐기 시기 | 대체 |
|---|---|---|
| `feedbackType` | 2026-04-26 | `feedbackStyle` (v1.0 표준화 4종 enum) |
| `retryThreshold` | 2026-04-26 | `questionMaxAttempts` |
| `passageExposureMode` | 2026-04-26 (DB DROP) | `passageMode` (GenericAdapter 브릿지로 점진 제거) |
| `learning_objectives` | 2026-04-22 | KPI_CATEGORY (`courses.kpi_codes`) |

### 13.7 네이밍 규칙·Variant·시퀀싱 그룹 *(⚠️ v0.12·v0.14 갱신)*

**3글자 자음 패턴 (v1.0 표준)**

Module List v1.0에서 코드 명명 규칙이 **3글자 자음 패턴**으로 표준화되었다 (예: WRD, GMN, SUM, PPR, OMS).

| 패턴 | 규칙 | 예시 |
|---|---|---|
| 3글자 자음 | 영어 명칭의 자음 또는 핵심 음절 추출 | `Word Decker Reading → WRD`, `Summarizing → SUM`, `Paraphrasing → PPR` |
| 2글자 단축 | 짧은 명칭은 2글자도 허용 | `Listen & Repeat → LR` |

**v0.5의 접미사 -R/-S 규칙 폐기 (v0.12)**

| 구버전 | v0.12 | 비고 |
|---|---|---|
| QAR-R | **QAR** | -R 접미사 폐기 |
| QAR-S | ❌ 폐기 | Speaking QAR 모듈 자체 폐기 |
| SHR-S | **SHD** | -S 접미사 폐기, Shadowing 코드로 단순화 |

이로써 모든 모듈은 영역 구분 없이 **3글자 자음 코드 단일 체계**로 운영된다. 영역(Vocabulary/Reading/Speaking/Writing)은 코드가 아닌 **영역 분류 메타데이터**로만 관리한다.

#### 13.7.1 시퀀싱 그룹 운영 *(❌ v0.15 폐기 — EDR 단일 모듈 통합으로 불필요)*

> ❌ **v0.15에서 시퀀싱 그룹 개념 폐기**: v0.14에서 도입한 "Expression Drill = LR·SHD·SXP 시퀀싱 그룹" 개념은 **모듈별 학습 시나리오 v1.1**에서 **EDR(Expression Drill) 단일 모듈로 통합**되며 폐기되었다. 그룹 내 동적 선택 대신 **모든 학습자가 동일한 EDR 모듈을 진행하되, 모듈 내부에서 3-stage(Read → Fill-in → Expand) 점진 진행**으로 대체.

**v0.15 기준 — Pick-Speak 단계 ↔ 모듈 1:1 매핑**

| Pick-Speak 단계 | 매핑 모듈 | 비고 |
|---|---|---|
| 1단계 PICK | (시스템 단계, 모듈 없음) | 시퀀싱 알고리즘이 결정 |
| 2단계 LEARN | **LRN** (Learn & Study) | 단일 모듈 |
| 3-1 어휘 워밍업 | **VLM** (Vocabulary Listening & Meaning) | 초·중급만, B2+ 자동 스킵 |
| 3-2 정밀 드릴 | **EDR** (Expression Drill) | **3-stage 내부 진행** (Read → Fill-in → Expand) |
| 4-1 APPLY Phase 1 | **RPL** (Role-Play, PTT) | 필수 |
| 4-2 APPLY Phase 2 | **FRT** (Free Talking) | B1+ 활성, 시퀀싱 결정 |
| 4-3 APPLY Phase 3 | **OMP** (One Minute Presentation) | B1+ 시험·비즈니스·면접 학습자, 시퀀싱 결정 |
| 5단계 MEASURE | (시스템 단계, 모듈 없음) | §15.4 PPD 산출 |

→ Speaking 6 모듈은 각각 Pick-Speak 단계의 명확한 위치를 가지며, 단계 내 동적 선택은 시퀀싱 알고리즘(§14.4.3)이 단계 활성화 여부로 결정한다.

### 13.8 공통 규약 (Module Common Rules)

모든 모듈이 공통으로 적용받는 UX·타이머·개입 정책. 모듈별 상세 시나리오는 컴패니언 `Picklass_모듈_시나리오_상세.md` 참조.

#### 13.8.1 다문항 모듈 표준 UX

여러 문항(단어·문장·문제)을 순환 학습하는 모듈은 다음 UX 패턴을 공통 준수한다.

| 요소 | 규칙 |
|---|---|
| 프로그레스 바 | 상단에 필수 노출, 현재/전체 문항 위치 표시 |
| 재도전 기준 | 모듈 전체 정답률 **80% 미만**일 경우 재도전 유도 |
| 분기 버튼 | `[다시할게요]` (80% 미만 시 활성) / `[이해했어요]` (다음 단계) |
| 완료 버튼 | 학습 완료 후 `[다음]` 버튼으로 다음 모듈로 이동 |

**적용 모듈**: WRD, WSD, QAR, VLM, EDR(Stage별 순환) 등 다문항 구조 모듈

#### 13.8.2 타이머·지연·힌트 표준 파라미터 *(v0.12 갱신)*

모듈별 지연 기준 및 자동 스탑 규칙은 아래 체계를 따른다.

| 모듈 군 | 지연 기준 | 자동 스탑 | 힌트 개입 |
|---|---|---|---|
| Word Decker (WRD/WSD) | **10초** | — | 스펠링 20% 노출 (최소 2자) 또는 발음 힌트 |
| Prediction (PRD) / Guessing Meaning (GMN) / Word Web (WWB) | **20초** | — | 브릿지 질문 / 영문 정의 / 관계어 뜻 |
| Clarification (CLR) | **15~30초** (레벨별) | — | 힌트 없음 (1:1 대화 기반) |
| Summarizing (SUM) | **60초** | — | 주제문/요지문 제공 |
| Scanning (SCN) / Skimming (SKM) | **20초** | — | 한글 요약 / 주제문 위치 힌트 |
| QAR | **20초** | — | 한글 번역 질문 |
| Vocabulary Listening & Meaning (VLM) | — | **어휘당 6초 자동 진행** | 슬로우 재생 (발화 없음) |
| Expression Drill (EDR) | — | **문장 길이 × 1.0~1.5초** (Stage별) | Stage 1: 다시 들려주기 + 음절 분리 / Stage 2: 블랭크 힌트 / Stage 3: 한글 가이드 + 핵심 단어 |
| Repeated Reading (RRD) | **10초** | — | 끊어 읽기 가이드 + 유창성 포인트 |
| Role-Play (RPL) | — | 무발화 자동 종료 없음 (PTT) | 한국어 안전망 + 미사용 표현 유도 |
| Scenario Free Talking (FRT) | **5초 화이트노이즈** | 무응답 자동 종료 | 레벨별 (초급: 문장 완성형 / 중급: AI 리프레이즈 / 고급: 서브 코멘트 패널) |
| One Minute Presentation (OMP) | — | **60초 고정** | 구조 템플릿 6단계 가이드 |

#### 13.8.3 재도전 표준 로직

```
모듈 전체 완료 후:
  ├─ 정답률 ≥ 80%  →  [이해했어요] 버튼만 노출  →  다음 모듈
  └─ 정답률 < 80%  →  [다시할게요] 버튼 활성화
                        ↓ 사용자 선택
                        ├─ [다시할게요]  →  오답 문항만 or 전체 재도전
                        └─ [이해했어요]  →  다음 모듈
```

#### 13.8.4 문항당 재시도 횟수

단문항 반복 모듈(WRD, WSD, RRD — EDR은 Stage당 3회, §14.4.2)은 **문항당 최대 3회** 재시도 허용. 3회 소진 시:
- 정답 자동 공개
- 문항별 심층 피드백(사전 준비 데이터) 제공
- 다음 문항으로 자동 이동

#### 13.8.5 통과 기준 *(v0.12 갱신)*

| 모듈 유형 | 통과 기준 |
|---|---|
| Word Decker (타이핑) | 정답 문자열 정확 일치 |
| 발음 평가 (WSD, RRD) | **ASR 인식률 70% 이상** |
| 속도 복합 (EDR Stage 1, RRD) | ASR 70% + 원어민 속도 충족 |
| 객관식·매칭형 (QAR, IMG, VLM) | 정답 선택 + (QAR) **근거 문장 일치(Evidence Match)** |
| 홀리스틱 평가 (PRD, GMN, WWB, FRT, EDR Stage 3, OMP 등) | LLM 평가 점수 (문맥·구조·타당성 기준) |

### 13.9 호환성 매트릭스 (요약) *(v0.12 갱신)*

전체 호환성은 엑셀 카탈로그의 **호환성 매트릭스** 시트 참조. v1.0 기준 제약 요약:

| 모듈 A | 제약 | 모듈 B | 사유 |
|---|---|---|---|
| PRD | incompatibleWith | RRD | 같은 세션 내 교수 목표 중복 (예측 vs 반복 낭독) |
| PWR | prerequisites | SWR | 문장 작성 능력 선행 필요 |
| FRT | recommended-prior | EDR | Expression Drill(Stage 3 Expand) 선행 시 FRT 효과 극대 |
| OMP | recommended-prior | SUM | Summarizing 선행 시 1분 발표 구조화 향상 |

> ⚠️ 제약 위반 시 §10.4 시퀀싱 엔진이 자동 필터링하며, 강사 수동 편집에서도 실시간 경고를 표시한다. v0.11까지 정의되었던 KED/SHR 제약은 해당 모듈 폐기로 함께 제거됨. **RPL(구 RLP, v0.14 부활 → v0.15 코드 변경)**은 §14.4 APPLY Phase 1 핵심 모듈로 운영된다.

### 13.10 상태 및 라이프사이클

모듈은 다음 상태를 순차적으로 거친다.

```
[기획 draft]
   ↓
[예정]  ──── Phase 배정
   ↓
[개발 중]
   ↓
[MVP]  ──── 내부 테스트
   ↓
[활성]  ──── 프로덕션 노출
   ↓
[단종]  ──── 레거시 레슨에서만 참조 허용
```

- **MVP**: 기능 미완성이지만 동작 가능, 피드백 수집 단계
- **활성**: 프로덕션 공개, LessonPlan에 자동 선정 가능
- **단종**: 신규 레슨에 사용 금지, 기존 LessonPlan은 버전 고정으로 유지 (§13.10)

### 13.11 모듈 버전 관리 및 마이그레이션

**버전 체계**
```
v1.0 (초기 릴리즈)
 ├─ v1.1 (문항·프롬프트 수정 — Backward Compatible)
 │    └─ v1.2 (채점 파라미터 튜닝)
 └─ v2.0 (구조 변경 — Breaking Change)
```

**마이그레이션 규칙**
- **Backward Compatible** (minor): 기존 LessonPlan 자동 적용, 기존 답변 데이터 호환
- **Breaking Change** (major): 신규 LessonPlan만 적용, 기존 레슨은 v1.x 버전 고정 참조
- **단종**: 신규 사용 금지, 기존 레슨은 단종 직전 버전 스냅샷 참조

**Speaking 모듈 특별 규정 (v0.12)**
- Speaking 모듈의 변경은 **Tutoring LessonPlan과 Speaking 독립 세션 양쪽**에 영향을 미치므로, 호환성 테스트 매트릭스에 양 경로 검증 필수.

### 13.12 관련 산출물 (Companion Artifacts) *(v0.12 갱신)*

| 산출물 | 위치 | 용도 |
|---|---|---|
| **`Picklass_Module_List_v1.0.docx`** | 외부 개발 문서 | **단일 진실 공급원(SSoT)** — 본 §13 정렬 기준 |
| **`Picklass_모듈_카탈로그.xlsx`** (v0.5) | 요청사항 폴더 | 운영용 모듈 카탈로그 — 29 모듈 + 폐기 시트 + 작성 상태 시트 |
| **`Picklass_모듈_시나리오_상세.md`** | 요청사항 폴더 | 모듈별 학습 시나리오 (v0.12에 맞춰 갱신 예정) |
| **`Picklass_KPI_체계_상세.md`** | 요청사항 폴더 | 70 KPI 코드 + 모듈×KPI 매핑 (폐기 모듈의 KPI 재할당 예정) |
| 본 기획서 §13 | v0.12 | 정책·원칙·분류 체계 |
| `src/lib/constants.ts` | 코드베이스 | TUTORING_MODULES / SPEAKING_MODULES 상수 (v0.12 코드 변경 반영 필요) |
| §10 모듈 시퀀싱 엔진 | v0.12 | 레지스트리를 활용한 자동 설계 |
| §14 Speaking Tutor | v0.12 | 교수법 해설 (모듈 카탈로그는 본 §13 참조) |

### 13.13 미디어 라이브러리 *(❌ v0.16 폐기 — §9.7로 통합 이관)*

> ❌ **v0.16에서 §9.7 미디어 라이브러리(§9 콘텐츠 라이브러리 통합)** 로 이관되었다. 모든 콘텐츠가 §9 단일 파이프라인을 통해 등록·분석·재사용되도록 통일하기 위함이다. 본 절의 컨셉·처리 파이프라인·데이터 모델·저작권·통합 지점·추천 알고리즘은 모두 **§9.7로 이전**되었으며, 본 §13.13은 cross-reference용 placeholder만 유지한다.

> 📎 미디어 라이브러리 상세는 **§9.7 미디어 라이브러리** 참조.

---

## 14. Speaking Tutor

### 14.1 제품 개요 및 포지셔닝

#### 14.1.1 이중 운영 구조

Speaking Tutor는 **동일한 엔진·데이터 모델·AI 에이전트**를 공유하되, 2개 진입 채널로 제공된다.

| 채널 | 진입 방식 | UI/UX | 과금 |
|---|---|---|---|
| 통합 모듈 | Tutoring 레슨 내 Speaking 슬롯 | Tutoring 공통 UI | 플랜 번들 |
| 독립 앱 | speaking.picklass.com 직접 진입 | Speaking 전용 UI | 단독 구독/추가 옵션 |

#### 14.1.2 타겟 사용자

- 영어 초급 학습자 (A1–A2)
- 중급 학습자 (B1–B2)
- 고급 학습자 (C1–C2)
- 시험 준비생 (토익 스피킹, 오픽)
- 비즈니스 영어 학습자

#### 14.1.3 Reading/Writing과의 차별점

| 항목 | Reading/Writing | **Speaking** |
|---|---|---|
| 입출력 | 텍스트/터치 | **실시간 양방향 음성** |
| 세션 단위 | 문항 | **대화 턴(turn)** |
| 지연 민감도 | 중 | **매우 높음 (TTFT 중요)** |
| 데이터 | Answer, Score | **Audio, STT Transcript, Confidence, WPM** |

### 14.2 레벨 시스템 (1–18, 6그룹)

#### 14.2.1 CEFR 매핑

| 그룹 | 레벨 | CEFR | 대화 전략 |
|---|---|---|---|
| Starter | L1–3 | A1−, A1, A1+ | 발화 습관 형성 |
| Beginner | L4–6 | A2−, A2, A2+ | 문장 구조 안착 |
| Intermediate | L7–9 | B1−, B1, B1+ | 의미 단위 연결 |
| Upper-Intermediate | L10–12 | B2−, B2, B2+ | 논리 전개 고도화 |
| Advanced | L13–15 | C1−, C1, C1+ | 비판적 사고/토론 |
| Proficient | L16–18 | C2−, C2, C2+ | 유창성/지적 파트너십 |

#### 14.2.2 WPM 공식
**Level N = 80 + (N−1) × 4** (80 기준값은 코드로 관리)

#### 14.2.3 레벨별 대화 전략 / 개입 강도 / 피드백 깊이

| 그룹 | 개입/피드백 핵심 |
|---|---|
| Starter | 즉각 교정: 단어를 문장으로 조립, 정서적 지지 |
| Beginner | 즉각 교정: 기본 문법 및 패턴 반복 |
| Intermediate | 확장 피드백: 접속사 활용 및 유의어 추천 |
| Upper-Inter. | 확장 피드백: 문장 결합도 및 세련된 표현 |
| Advanced | 지연 피드백: 논리적 완결성 및 수사적 교정 |
| Proficient | 지연 피드백: 뉘앙스 분석 및 비판적 반박 |

### 14.3 Free Talking(FRT) 대화 루프 설계 *(⚠️ v0.14 — Pick-Speak APPLY Phase 2 상세)*

> 🔁 **v0.14 정합화**: 본 절은 v0.13까지 "Free-talking 세션 설계 (3-Phase)"였으나, **§14.4 Pick-Speak APPLY 3-Phase**(롤플레이 / FRT / 1MP)와 명칭이 충돌하여 "Free Talking 대화 루프"로 재정의. 본 절은 **APPLY Phase 2(FRT) 모듈 내부의 대화 처리 루프**만 다룬다.

FRT 모듈 진입 시 동작하는 대화 처리 흐름은 다음 3단계 루프로 구성된다.

#### 14.3.1 진입 · Opening

- **진입**: §14.4 APPLY Phase 1 미션 5/5 완료 자동 전환 또는 학습자 수동 전환 ("충분해요, 자유 대화로" 버튼)
- **Opening**: 시나리오 컨텍스트 + 첫 질문 + 힌트 3단계 생성 (AI API 1회, ~800ms)
- **TTS**: 첫 질문 음성 출력 (레벨별 WPM, §14.2.2 공식)
- **발화량 챌린지 미터**: 화면 상단 등장 ("지난 7일 평균 28문장 → 오늘 목표 30문장")

#### 14.3.2 순환 루프

```
Step A: AI 질문 생성 (레벨 맞춤 + 힌트 3단계) → TTS
Step B: 사용자 발화 대기
    Case 1: 침묵 5/8/10초 → Hint 1/2/3 (로컬)
    Case 2: STT Conf < 0.5 → 재요청 (로컬)
    Case 3: 한국어 발화 1회 → 한국어 안전망 (§14.4.6 차별점 3) 활성화
    Case 4: 정상 입력 → Step C
Step C: 통합 AI 처리 (상황+문법+응답+힌트, 1회 호출)
Step D: 실시간 피드백 (추임새 300ms + 교정/따라읽기) + 발화량 미터 충전
Step E: 다음 액션 (꼬리 질문 or 질문 유도) → Step A
```

#### 14.3.3 종료

- 세션 시간 만료(레벨별 1~2분, §14.5.7 참조) 또는 학습자 수동 종료
- Session Summary (로컬 마무리 인사)
- KPI 생성: 5지표 분석 → §14.6.1 + §15.4 PPD 갱신 (AI API 1회, ~1,000ms)
- 데이터 저장: 대화 기록, 오류 리스트, 오디오 로그
- §14.4 APPLY Phase 3(OMS) 활성 학습자는 자동 연계, 비활성 학습자는 §14.4 Step 5 MEASURE로 전환

### 14.4 Pick-Speak Method — Speaking 학습 흐름 및 6 교수법 모듈 *(⚠️ v0.15 갱신 — 학습 시나리오 v1.1 반영)*

> 본 절은 Speaking 학습의 **5단계 상위 프레임워크(Pick-Speak Method)** 와 그 안에 배치되는 **6 원자 모듈**을 통합 기술한다.
>
> 📌 모듈 코드·스킬·레벨·역할 등 구조화 메타데이터는 §13.1 마스터 모듈 레지스트리 및 `Picklass_모듈_카탈로그.xlsx` v0.7 참조. 모듈별 7단계 상세 학습 시나리오·0단계 데이터 준비·멘트 사전 생성 데이터는 `Picklass_모듈_시나리오_상세.md` v1.1 컴패니언 문서 참조.

#### 14.4.0 설계 철학 — "정확하게 + 많이 + 측정 가능하게"

Speaking 학습 설계의 5대 원칙.

| 원칙 | 내용 |
|---|---|
| 1. **3계 구조** | 모든 레슨은 **입력(LEARN) → 출력(DRILL/APPLY) → 측정(MEASURE)** 의 3계 구조를 갖는다 |
| 2. **사전 미션 기반** | 회의록 확정 사항(§10.15)에 따라 **사전 설정 미션 기반 진행**, 미션 내부에서만 AI 자유도 부여 |
| 3. **공공장소 사용성** | 출퇴근·카페·도서관 환경 우선 설계 — Silent Drill Mode (§14.4.6 차별점 2) |
| 4. **정량 측정** | **5축 즉시 피드백** (발음정확도·유창성·문법정확성·**화용성**·**발화량**), §15.4 PPD와 직접 연동 |
| 5. **약정 없는 신뢰 구조** | 학습 자체에 보상을 내재화 (§14.4.9 동기 부여) |

#### 14.4.0.1 모바일 3분할 화면 표준 *(v0.15 신설)*

모든 6 Speaking 모듈은 **모바일 우선 상단·중간·하단 세로 3분할** UI 표준을 따른다 (기존 데스크톱형 좌상/좌하/우측 분할 폐기).

| 영역 | 역할 | 표시 요소 |
|---|---|---|
| **상단 (Top)** | 진도·헤더·메타 | 5단계 진행 인디케이터, 모듈명, 시간/카운트다운, 시나리오 정보 |
| **중간 (Middle)** | 메인 학습 콘텐츠 | 문장, 카드, 영상, 대화 흐름, 차트, **5축 피드백 그리드** |
| **하단 (Bottom)** | 입력·액션 | 마이크/녹음 버튼, **\[💡 힌트\] 버튼**, 모드 토글, 다음/이전 |

**Pickle Agent 멘트 노출**: 상단 작은 말풍선 또는 중간 영역 위 토스트 (별도 채팅 패널 없음). 학습자 발화 흐름을 방해하지 않는 비침습적 노출.

#### 14.4.0.2 힌트 버튼 사용자 요청형 정책 *(v0.15 — AI 선제 개입 폐지)*

기존 자동 타이머 기반 AI 선제 개입을 **힌트 버튼 사용자 요청형**으로 전환했다.

| 변경 전 (v0.14) | 변경 후 (v0.15) |
|---|---|
| 타이머 경과 시 자동 힌트 제공 | **\[💡 힌트\] 버튼 상시 노출, 학습자 직접 클릭 시에만 힌트 제공** |
| 자동 개입으로 학습 흐름 끊김 | 학습자 자율성 우선 |
| 힌트 사용 횟수 = 감점 | **힌트 사용 횟수는 측정·기록만**, 점수에 직접 반영 X |

**예외 — 특수 트리거 유지**: FRT 무발화 5초·OMP 60초 발화 정체 등 학습 진행에 필수적인 자동 트리거는 유지.

#### 14.4.0.3 멘트 사전 생성·저장 구조 *(v0.15 신설)*

기존 단계별 멘트 가이드 표를 **세션 진입 시 사전 생성·저장하는 멘트 데이터 구조**로 전환.

**0단계 데이터 준비 시점**에 LLM이 학습자 컨텍스트(이름·레벨·직전 학습 결과·페르소나)를 반영해 **모든 트리거의 멘트를 사전 생성**한다. 트리거 발생 시 즉시 재생되어 응답 지연이 없다.

```javascript
// 세션 멘트 데이터 구조 (예시)
{
  session_id: "ses_20260507_001",
  module_code: "EDR",
  user_context: { name: "Tim", level: "B1", weak_axis: "화용성" },
  ments: [
    { id: "M01_INTRO",        trigger: "MODULE_ENTRY",        variants: [...3종] },
    { id: "M02_HINT",         trigger: "USER_HINT_CLICK",     variants: [...5종] },
    { id: "M03_PASS",         trigger: "FEEDBACK_PASS",       variants: [...5종] },
    { id: "M04_RETRY_1",      trigger: "FEEDBACK_RETRY_1",    variants: [...3종] },
    { id: "M05_RETRY_FINAL",  trigger: "FEEDBACK_RETRY_FINAL",variants: [...3종] },
    { id: "M06_3STRIKE",      trigger: "3STRIKE_ALERT",       variants: [...2종] },
    { id: "M07_OUTRO",        trigger: "MODULE_COMPLETE",     variants: [...3종] }
  ]
}
```

**핵심 정책**

| 정책 | 내용 |
|---|---|
| 변형 수 | 트리거당 **2~5종 변형** 사전 생성 |
| 노출 방식 | **무작위 노출** (반복 학습 시 신선도 유지) |
| 컨텍스트 변수 | `{{name}}`, `{{level}}`, `{{weakAxis}}`, `{{score}}`, `{{topic}}`, `{{remaining}}` 등 학습자별 슬롯 |
| 페르소나 톤 | 친근한 존댓말 + 격려 톤 (모든 모듈 공통) |
| 개인화 예시 | "Tim님, 어제 화용성 +3점 올랐어요\!" |

#### 14.4.1 Pick-Speak Method 5단계 학습 흐름 *(v0.15 — 6 모듈)*

```
[1] PICK     (1~2분)   ─ 목표 픽업                              (시스템 단계)
     ↓
[2] LEARN    (3~5분)   ─ 입력 학습                              → LRN
     ↓
[3] DRILL    (5~8분)   ─ 정밀 드릴
     ├─ 3-1. 어휘 워밍업       (0:30, 초·중급만)               → VLM (듣고 의미 확인)
     └─ 3-2. 5축 정밀 드릴     (5~7분, 3-stage 진행)           → EDR (Read → Fill-in → Expand)
     ↓
[4] APPLY    (5~10분)  ─ 적용·자유화 (3-Phase)
     ├─ Phase 1. 롤플레이                                       → RPL (PTT 방식)
     ├─ Phase 2. 자유 확장 (Free Talking)                       → FRT (Always-On + 챌린지 미터)
     └─ Phase 3. 1 Minute Presentation                          → OMP (30초 준비 + 1분 발화)
     ↓
[5] MEASURE  (2~3분)   ─ 측정·복습 (§15.4 PPD 산출)             (시스템 단계)
```

**합계 — 골든타임 18~22분** (시퀀싱 결정에 따라 8~25분 동적 변동)

> 🔁 **v0.14 → v0.15 변경**: 7 모듈 → **6 모듈 통합**. WSP→VLM(재설계), LR/SHD/SXP→**EDR 단일 모듈**(3-stage 통합), RLP→RPL(PTT), OMS→OMP(명칭 통일), **LRN(Learn & Study) 신규**.

#### 14.4.2 단계별 상세

##### Step 1. PICK (목표 픽업) — 1~2분

학습자가 오늘 배울 표현·미션을 선택하거나 AI가 자동 추천한다.

| 항목 | 정책 |
|---|---|
| 추천 로직 | 어제 학습한 약점 + 학습자 직무·관심사 + CEFR 레벨 난이도 + §15.4 PPD 약점 영역 |
| 화면 노출 | 미션 제목 / 예상 소요 시간 / 완료 보상 / **오늘의 학습 구성** (§14.4.3 시퀀싱 결과 노출) |
| 정규수강·자유수강 | §11.9 분기 적용 |

##### Step 2. LEARN (입력 학습) — 3~5분 → **LRN (Learn & Study)** ✦ v0.15 모듈화

기관 교재·교안 자산을 디지털화한 짧은 비디오 강의로 핵심 표현·패턴을 학습. v0.15에서 **LRN 단일 모듈로 정식화**.

| 항목 | 정책 |
|---|---|
| 모듈 | **LRN (Learn & Study)** |
| 형식 | 짧은 비디오 강의 + 핵심 표현 5개 카드 적층 |
| 카드 구성 | 영어 표현 + 한국어 의미 + 사용 상황 + TTS + 즐겨찾기 |
| 자막 | 한국어/영어/없음 3종 토글 (왕초보 진입장벽 ↓) |
| 재생 속도 | 0.7× / 1.0× / 1.3× |
| 카드 호출 | 핵심 표현 5개는 이후 EDR/RPL/FRT/OMP에서 재호출됨 |
| 콘텐츠 출처 | §10.15 콘텐츠 생성 모드 ① 교재 기반 / ② 주제+레벨 |
| 측정 지표 | 카드 관람 완료율, 자막 사용 패턴, 즐겨찾기 표현 수 |

**레벨별 매트릭스**

| CEFR | 영상 길이 | 표현 카드 수 | 자막 기본값 |
|---|---|---|---|
| Pre-A1~A1 | 1.5분 | 5개 (단순) | 한국어 ON |
| A2 | 2분 | 5개 | 한/영 토글 |
| B1 | 2.5분 | 5개 (관용 1개) | 영어 ON |
| B2 | 3분 | 5개 (격식 차이) | 영어 ON |
| C1/C2 | 3분 | 5개 (수사 표현) | OFF |

##### Step 3. DRILL (정밀 드릴) — 5~8분

본격 발화 훈련 단계. 워밍업 + 메인의 2단계 구성.

**3-1. 어휘 워밍업 (Warmup) — 0:30 (초·중급만)** → **VLM** ✦ v0.15 재설계

LRN 학습 표현 속 핵심 어휘 5개를 **듣고 의미를 확인**하는 30초 워밍업. v0.14의 WSP(단어 스피킹)는 ASR 정확도 한계와 학습 효과 약함으로 폐기되고, **청각 입력 + 의미 인지** 방식으로 재설계.

| 항목 | 정책 |
|---|---|
| 모듈 | **VLM (Vocabulary Listening & Meaning)** |
| 흐름 | 듣기(1.5초) → 의미 확인(3.5초) → 다음 카드(1초). 어휘당 6초, 총 30초 |
| 의미 확인 방식 | A1~A2 한국어 의미 자동 노출 / A2 한국어 3지선다 / B1~B2 영어 정의 3지선다 |
| 학습자 행동 | **마이크 발화 없음** — 듣고 보고 (필요시 탭) 진행. ASR 부담 0 |
| 레벨별 노출 | A1~A2 5개 전부 / B1~B2 즐겨찾기 안 한 어휘만 1~3개 / B2+~C1 자동 스킵 |
| 측정 지표 | 의미 선택 정답률, 카드 관람 완료율, 슬로우 재생 사용 횟수 |

**3-2. 5축 정밀 드릴 (Main) — 5~7분 ★ 핵심 차별점** → **EDR (Expression Drill)** ✦ v0.15 통합 + 3-stage

LRN 표현을 **3-stage 점진적 발화**(읽기 → 블랭크 → 확장)로 체득하며 **5축 KPI 즉시 피드백**을 받는 픽클래스 핵심 차별 모듈. v0.14의 LR/SHD/SXP 3개 모듈이 EDR 단일 모듈로 통합.

| 항목 | 정책 |
|---|---|
| 모듈 | **EDR (Expression Drill)** |
| **5축 즉시 피드백** | **발음정확도 · 유창성 · 문법정확성 · 화용성 · 발화량** 동시에 그래프로 표시 (v0.14 4축 → 5축으로 확장) |
| Silent Drill Mode | 옵션 — 마이크 감도 극대화 + 키보드 타이핑 답변 허용 |
| 진행 모드 | PTT(라이트) / Always-On(노멀) / Silent Drill |
| 콘텐츠 강화 | **레벨별 관용표현(이디엄·숙어) 비중 1~5개 차등** (초급 1개 → 고급 5개) |
| 재시도 | Stage당 최대 3회 |
| 어휘 재호출 | VLM 학습 어휘는 발화 막힘·발음 60점 이하 시 화면 하단 인라인 미니 카드로 자동 재호출 |

**EDR 3-stage 내부 진행 — 표현당 Stage 1→2→3 순차 통과**

| Stage | 명칭 | 학습자 행동 | 5축 측정 포커스 |
|---|---|---|---|
| **Stage 1 — Read** | 문장 읽기 | 원문 문장(관용표현 강조) 그대로 발화 + IPA 발음 가이드 | 발음정확도, 유창성 |
| **Stage 2 — Fill-in** | 블랭크 채우기 | 핵심 표현 부분 블랭크 (예: "Let's _____ the meeting.") 채워 전체 문장 발화 | 문법정확성, 발화량 |
| **Stage 3 — Expand** | 문장 확장 | 같은 표현을 활용한 새 문장 만들기 (한국어 가이드 + 핵심 단어 힌트) | **화용성**, 문법정확성, 발화량 |

→ 5표현 × 3 Stage = 표현 1개당 Stage 1→2→3 모두 통과해야 다음 표현으로 이동. 총 **15회 발화 사이클**.

**EDR 통합 — v0.14 LR/SHD/SXP의 기능 흡수 매핑**

| v0.14 모듈 | v0.15 EDR 흡수 위치 |
|---|---|
| LR (Listen & Repeat) | Stage 1 Read의 청각 입력 + 모방 발화 측면 |
| SHD (Shadowing) | Stage 1 Read의 정확 발음 평가 측면 (ASR + WPM 복합 분석) |
| SXP (Sentence Expansion) | Stage 3 Expand의 문장 확장 미션 측면 |

##### Step 4. APPLY (적용·자유화) — 5~10분 [3-Phase]

자유 롤플레이를 단일 모드에서 **3-Phase 구조**로 재설계.

**Phase 1 — 롤플레이 (Role-Play) — 4~5분 (필수)** → **RPL** ✦ v0.15 PTT 방식

| 항목 | 정책 |
|---|---|
| 모듈 | **RPL (Role-Play)** — v0.14의 RLP에서 코드 변경 |
| AI 캐릭터 | Sarah, Mike 등 사전 구성 (이름·아바타·음성·성격 사전 설정) |
| 시나리오 | 4~6턴 대화 사전 설계 (비즈니스 미팅·면접·여행 등) |
| **PTT 방식** | **마이크 버튼을 누르고 있는 동안만 녹음** (떼면 즉시 인식 종료) — Always-On 자동 인식 폐기로 학습 흐름 통제권 확보 |
| 미션 | LRN의 표현 5개를 실제 대화 흐름 속에서 **모두 사용**해야 미션 완료 |
| UX | 사용한 표현은 우측 카드에 ✓ 표시, 미사용은 AI가 자연스럽게 유도 질문 |
| 안전망 | 한국어 안전망 (1회 한국어 발화 시 AI가 영어로 풀어주고 권장 표현 가르침), 레벨별 횟수 제한 |
| 무발화 자동 종료 | 없음 (PTT 특성) |

**Phase 2 — 자유 확장 (Free Talking) — 1~2분** → **FRT**

| 항목 | 정책 |
|---|---|
| 모듈 | **FRT (Free Talking)** |
| 전환 조건 | Phase 1 미션 5/5 완료 시 자동 / "충분해요, 자유 대화로" 버튼 수동 |
| 핵심 메커니즘 | **발화량 챌린지 미터** — "지난 7일 평균 28문장 → 오늘 목표 30문장", 발화 시 실시간 충전 |
| 무발화 5초 (특수 트리거) | 화이트노이즈 5초 → 자동 빈 말풍선 + 레벨별 힌트 (초급 문장 완성형 / 중급 AI 리프레이즈 / 고급 자율) |
| 게이미피케이션 | 미션 카드 없음, **발화량 자체가 보상 신호** |
| 활성화 | A1~A2 자동 비활성, A2 보조, B1+ 활성 |

**Phase 3 — 1 Minute Presentation — 약 3분 ✦ 시그니처 모듈** → **OMP** ✦ v0.15 명칭 통일

| 항목 | 정책 |
|---|---|
| 모듈 | **OMP (One Minute Presentation)** — v0.14의 OMS(One-Minute Topic Speaking)에서 명칭 통일 |
| 본질 | "구조화된 모놀로그" 발화 능력 측정 — 자유 대화의 즉흥성 ↔ 롤플레이 정형성의 중간 지점 |
| **모듈 내부 흐름** | **(1) 30초 준비** — LRN 콘텐츠 기반 발표 주제 + 핵심 키워드 + Intro-Body-Conclusion 3단 구조 가이드 / **(2) 1분 발화** — 카운트다운 + 발화량 미터 + **5축 누적 평가** + 30초 마무리 토스트 + 50초 결론 토스트 / **(3) 1~1.5분 종합 피드백** — 5영역(발음정확도·유창성·문법정확성·화용성·발화량) 평가 + AI 추천 더 좋은 발표 구조 |
| 메모 템플릿 6항목 | The topic is about / It mainly says that / In my opinion / I agree because / For example / So overall |
| 활성화 정책 | 시퀀싱 알고리즘이 학습자 목적·레벨·시간으로 결정. **A1~A2 항상 비활성**, A2 보조, B1+ 동적 활성. 시험 대비·비즈니스·면접 학습자에게는 Phase 2 대체 또는 추가 |
| 차별 가치 | OPIc 1번·토익S 4·5번 답변 형식과 정확히 일치. 비즈니스 발표·면접 자기소개에도 동일 활용 |
| 시험 학습자 한정 | 종료 시 "이 페이스라면 OPIc IH 도달 예상\!" 등 등급 예측 멘트 |

##### Step 5. MEASURE & REVIEW (측정·복습) — 2~3분

세션 종료 즉시 **레슨 카드(Lesson Card)** 생성 + §15.4 PPD 갱신.

| 노출 항목 | 내용 |
|---|---|
| **5축 지표 점수 + 변화 추이** | 발음정확도·유창성·문법정확성·**화용성**·**발화량** |
| 사용 못한 표현 리스트 | LRN의 표현 5개 중 미사용 |
| AI 제안 고급 표현 3개 | 학습자 레벨 +1 단계 표현 |
| 다음 추천 레슨 | §15.4 PPD 약점 기반 |
| 추가 시각화 | 어휘 활용도(VLM) · 5축 누적(EDR) · 자유 발화량(FRT) · 1분 발표 점수(OMP) |
| 복습 스케줄 | 망각곡선 기반 1일·3일·7일·14일 푸시 (§11.12) |
| 월간 | Achievement Test (§15.3) 자동 실행 |
| 블렌디드 연동 | 데이터를 화상 1:1 강사에게 자동 전달 |

#### 14.4.3 시퀀싱 알고리즘

학습자가 PICK 단계에서 시작 버튼을 누르는 순간 시퀀싱 알고리즘이 실행된다. analyzer 서버(§10.1.1)가 호스팅.

**입력값 7종**

```
1. 학습자 CEFR 레벨
2. 학습 목적 (시험 / 비즈니스 / 여행 / 일상 / 면접 등 다중 선택)
3. 가용 시간 (분)
4. 직전 학습의 5축 약점 (발음·유창·문법·화용·발화량)
5. 직전 학습의 미사용 표현
6. 망각곡선 복습 트리거 여부
7. 학습 환경 (공공장소 시 Silent Mode 자동 ON)
```

**출력값 4종**

```
1. 활성 모듈 리스트 (예: LRN + VLM + EDR + RPL + OMP)
2. 각 모듈의 시간 할당 (예: 4:00 / 0:30 / 6:00 / 4:30 / 3:00)
3. 모듈 내부 콘텐츠 선택 (시나리오 / 표현 카드 / 어휘 / 발표 주제)
4. 진행 모드 옵션 (Silent Mode / 자막 기본값)
```

**시퀀싱 노출 정책**

알고리즘 결과는 학습자에게 강제되지 않는다. **PICK 화면 상단 "오늘의 학습 구성"** 영역에 미리 노출되며, 학습자가 모듈 ON/OFF 변경 가능.

> 예: "오늘은 학습 콘텐츠 + 어휘 워밍업 + 표현 드릴 + 롤플레이 = 18분"

#### 14.4.4 시퀀싱 매트릭스 (학습 목적 × 레벨) *(v0.15 — 6 모듈 코드 갱신)*

LRN과 EDR은 모든 세션에서 활성화(필수). VLM·RPL·FRT·OMP는 시퀀싱 결정.

| 학습 목적 | A1~A2 (초급) | B1~B2 (중급) | B2+~C1 (고급) |
|---|---|---|---|
| **시험 대비** | LRN + VLM + EDR | LRN + EDR + **OMP** | LRN + **OMP** + EDR (OMP 우선) |
| **비즈니스 영어** | LRN + VLM + RPL | LRN + EDR + RPL + **OMP** | LRN + EDR + RPL + **OMP** |
| **여행 영어** | LRN + VLM + RPL | LRN + EDR + RPL | LRN + RPL + FRT |
| **일상 회화** | LRN + VLM + EDR | LRN + EDR + FRT | LRN + FRT + EDR (FRT 우선) |
| **이직·면접 영어** | LRN + VLM + RPL | LRN + EDR + RPL + **OMP** | LRN + RPL + **OMP** + FRT |
| **레벨 진단 (첫 학습)** | LRN + EDR (단축) | LRN + EDR (표준) | LRN + EDR + **OMP** |

> 시험 대비·면접 영어 학습자에게 **OMP(1MP)가 FRT를 대체하는 시그니처 모듈**로 작동한다.

#### 14.4.5 시퀀싱 매트릭스 (가용 시간) *(v0.15)*

| 가용 시간 | 활성 모듈 패턴 | 비고 |
|---|---|---|
| **8분 (Quick)** | LRN + EDR (단축) | 통근 짧은 구간 |
| **12분 (Light)** | LRN + VLM + EDR / LRN + EDR + RPL | 어휘 또는 표현 집중 |
| **18분 (Standard)** | LRN + VLM + EDR + RPL | 일반 골든타임 |
| **22분 (Full)** | LRN + VLM + EDR + RPL + (FRT 또는 OMP) | 표준 |
| **25분 (Deep)** | LRN + VLM + EDR + RPL + FRT + OMP | 6개 모듈 풀 활성화 (드뭄) |
| **30분 (Premium)** | 6개 모듈 + 화상 1:1 연계 | 프리미엄 사용자 |

#### 14.4.6 6가지 차별 메커니즘 *(v0.15 — 4축 → 5축 확장)*

Picklass Speaking이 두 경쟁사 대비 갖는 시스템 차별점.

**차별점 1 — 5축 즉시 피드백 시스템 ✦ v0.15 발화량 추가**

| 항목 | 내용 |
|---|---|
| 측정 축 | **발음정확도 · 유창성 · 문법정확성 · 화용성 · 발화량** 5축 동시 시각화 (v0.14의 4축에서 **발화량 추가**) |
| 위치 | EDR Stage별 5축 그리드 (2×3 — 4축 + 발화량 통합 셀) |
| §15.4 연결 | 5축 = PPD의 Pronunciation·Fluency·Grammar·**Pragmatics**·Speaking 역량과 직접 매핑 |
| 모듈 적용 | EDR(필수), RPL/FRT/OMP(누적 평가) |

**차별점 2 — Silent Drill Mode (조용한 모드)**

| 항목 | 내용 |
|---|---|
| 핵심 기능 | 마이크 감도 극대화 + 키보드 응답 + 작은 음성 안내 |
| 사용 환경 | 출퇴근·카페·도서관 |
| 활성 단계 | 주로 DRILL (EDR) |

**차별점 3 — 한국어 안전망 (Korean Safety Net)**

| 항목 | 내용 |
|---|---|
| 트리거 | 학습자가 영어 답변에 막혀 한국어로 1회 발화할 때 |
| 처리 | AI가 한국어를 영어로 자연스럽게 풀어주고, 영어 표현을 가르침 |
| 활성 단계 | APPLY Phase 1 (RPL) 핵심, Phase 2 (FRT) 보조 |
| 횟수 제한 | A1~A2 무제한 / A2 2회 / B1~B2 1회 / C1~C2 없음 |

**차별점 4 — 블렌디드 1:1 화상 연동**

| 항목 | 내용 |
|---|---|
| 적용 티어 | 프리미엄 — 월 1회 화상 1:1 수업 제공 |
| 데이터 흐름 | AI 학습 데이터(약점·미사용 표현·5축 추이) → 강사에게 자동 전달 |
| 강사 화면 | Studio §8 또는 별도 화상 코칭 패널 |

**차별점 5 — 발화량 챌린지 미터**

| 항목 | 내용 |
|---|---|
| 활성 단계 | APPLY Phase 2 (FRT) |
| 메커니즘 | 개인 발화량 추이 기반 동적 목표 실시간 시각화 |
| 게이미피케이션 | "어제보다 더 많이 말했다" 즉각적 성취감 |

**차별점 6 — 1 Minute Presentation 모듈 ✦ 시그니처**

| 항목 | 내용 |
|---|---|
| 활성 단계 | APPLY Phase 3 (OMP 모듈) |
| 시장 공략 | 시험 대비(OPIc·토익S) + 비즈니스(회의 발표·아이디어 피칭) + 면접(자기소개) 통합 |
| 차별 가치 | 두 경쟁사 모두 별도 모놀로그 모듈 부재 — 픽클래스 고유 차별 무기 |
| 시험 학습자 한정 | 종료 시 등급 예측 멘트 (예: "이 페이스라면 OPIc IH 도달 예상\!") |

#### 14.4.7 페르소나별 학습 플로우 *(v0.15 — 6 모듈 코드 갱신)*

| 페르소나 | 시퀀싱 결과 | 1일 학습 시간 | 핵심 모듈 |
|---|---|---|---|
| 통근족 직장인 (B1~B2) | LRN + EDR + RPL (Silent Mode) | 18분 (편도) | EDR |
| 비즈니스 영어 (B1~B2) | LRN + EDR + RPL + OMP | 22분 | RPL, **OMP** |
| **OPIc·토익S 수험생 (B2+)** | **LRN + EDR + OMP 또는 OMP 단독** | 15~22분 | **OMP** (시그니처) |
| 왕초보 입문자 (A1~A2) | LRN + VLM + EDR (한국어 자막) | 16분 | VLM |
| **이직·면접 준비** | **LRN + RPL + OMP** | 22분 | **OMP** (자기소개·답변) |
| 일상 회화 학습자 (B1+) | LRN + EDR + FRT | 20분 | FRT (발화량 챌린지) |
| 프리미엄 사용자 | 4~5개 모듈 + 화상 1:1 월 1회 | 22분 + 30분/월 | 전 모듈 |

> **OPIc·토익S 수험생과 이직·면접 준비 페르소나에서 OMP(1MP)가 시그니처 모듈**로 작동하며, 시험 대비·면접 시장이 픽클래스의 신규 공략 영역으로 추가된다.

#### 14.4.8 1세션 시간·발화량 목표

| 단계 | 표준 시간 | 비고 |
|---|---|---|
| 1. PICK | 1~2분 | 전체 세션 동일 |
| 2. LEARN | 3~5분 | 콘텐츠 길이에 따라 가변 |
| 3. DRILL | 5~8분 | 워밍업 0:30(초·중급) + 메인 5~7분 |
| 4. APPLY | 5~10분 | Phase 1 필수 + Phase 2/3 시퀀싱 선택 |
| 5. MEASURE | 2~3분 | 전체 세션 동일 |
| **합계** | **16~28분** | 평균 18~22분 골든타임, 시험 대비형 22~25분 |

**1세션 발화량 목표** — 모듈 시퀀스에 따라 30~150문장까지 변동.

| 학습자 유형 | 표준 발화량 |
|---|---|
| 일반 학습자 | 80~100문장 |
| 시험 대비형 (OMP 포함) | 90~120문장 |
| 프리미엄 풀 시퀀스 | 130~150문장 |

#### 14.4.9 학습 동기 부여 시스템

현금 환급 대신 **학습 자체에 보상을 내재화**.

| 보상 형태 | 예시 |
|---|---|
| 일일 스트릭 | 연속 학습일 가시화 |
| 주간 챌린지 | 주차별 발화량·새 표현 도전 |
| 월간 리포트 무료 발송 | §15.4 PPD 진단 카드 PDF |
| 4축 향상 시 등급 상승 | 자동 등급 부여 |
| 등급 혜택 | 무료 화상 수업 1회 추가, 프리미엄 콘텐츠 해금 등 학습 가치형 보상 |

발화량 챌린지 미터(Phase 2)와 1분 발표 점수(Phase 3)가 동기 부여 시스템의 **핵심 일일 트리거**로 작동한다.

#### 14.4.10 Speaking 6 모듈 — 단계별 배치 요약 *(v0.15)*

| Pick-Speak 단계 | 모듈 코드 | 명칭 | 핵심 메커니즘 | 핵심 KPI |
|---|---|---|---|---|
| 2단계 LEARN | **LRN** | Learn & Study | 콘텐츠 입력 + 표현 5개 카드 | VOCAB_EXPOSURE, FAVORITE_RATE |
| 3-1 어휘 워밍업 | **VLM** | Vocabulary Listening & Meaning | 듣고 의미 확인 (스피킹 X) | VOCAB_RECOG, MEANING_HIT |
| 3-2 정밀 드릴 | **EDR** | Expression Drill | 5축 피드백 + 3-stage 진행 (Read→Fill-in→Expand) | PRONUNCIATION, FLUENCY_WPM, GRAMMAR_ACCURACY, **PRAGMATICS**, **UTTERANCE_VOL** |
| 4-1 APPLY Phase 1 | **RPL** | Role-Play | PTT 기반 시나리오 대화 | INTERACTION_ACT, MISSION_USAGE, KOREAN_FALLBACK |
| 4-2 APPLY Phase 2 | **FRT** | Free Talking | Always-On 자유 대화 + 챌린지 미터 | TOTAL_UTTERANCE, SILENCE_RATIO, CHALLENGE_HIT |
| 4-3 APPLY Phase 3 | **OMP** | One Minute Presentation | 30초 준비 + 1분 발표 + 5영역 평가 | FLUENCY_WPM, LOGIC_STRUCT, TIME_USAGE, KEYWORD_RATE |

> 📎 PICK·MEASURE는 시퀀싱 대상 모듈이 아니며, 모든 세션에서 시스템 단계로 진행된다 (시간·콘텐츠는 가변).

**통합적 교수법 원리**

1. **점진적 복잡성 증가** — 콘텐츠 흡수(LRN) → 어휘 의미(VLM) → 표현 정밀(EDR Read→Fill-in→Expand) → 역할극(RPL) → 자유 대화(FRT) → 모놀로그(OMP) 단계적 발전
2. **다중 감각 활용** — 시청각(LRN) → 청각·인지(VLM) → 발화·평가(EDR) → 전 감각 통합(RPL·FRT·OMP) 순 자극
3. **자동화에서 창의성으로** — 기계적 반복(EDR Stage 1·2)에서 자유로운 표현(FRT)·구조화 모놀로그(OMP)로 전환
4. **개별화 학습** — 시퀀싱 알고리즘 + 레벨별 맞춤 전략(§14.5) + 멘트 사전 생성 + AI 실시간 피드백
5. **측정 가능성** — 모든 모듈이 §15.4 PPD에 정량 기여 (5축 즉시 + 누적 KPI)
6. **사용자 자율성** — 자동 개입 폐지, 힌트 버튼 사용자 요청형 (§14.4.0.2)

### 14.5 레벨별 Adaptive Interaction 매트릭스

#### 14.5.1 Opening 전략

| 그룹 | 전략 |
|---|---|
| Starter | 한글 가이드 + 표현 안내 |
| Beginner | 한글+영어 혼합 |
| Intermediate | 영어로 주제 배경 |
| Upper-Inter. | 영어로 토론 방향 |
| Advanced | 도발적 화두 |
| Proficient | 사회적 담론 |

#### 14.5.2 침묵 대응

| 그룹 | 1차 (5초) | 2차 (8초) | 3차 (10초) |
|---|---|---|---|
| Starter | 정답 제공 | - | - |
| Beginner | 문장 시작 | - | - |
| Intermediate | - | 소재 제공 | - |
| Upper-Inter. | - | 확장 유도 | - |
| Advanced | - | - | 논리 힌트 |
| Proficient | - | - | 반론 제시 |

#### 14.5.3 Feedback 깊이

Starter→Proficient 순으로 **즉시 교정 → 확장 → 지연/수사적** 순 심화.

#### 14.5.4 따라 읽기 강제 범위
- L1–12: 문법 오류 시 따라 읽기 **강제**
- L13+: 따라 읽기 **없음** (성인 학습자 존중)

#### 14.5.5 Next Action 주도권

| 그룹 | AI 주도 | 학습자 주도 |
|---|---|---|
| Starter | 100% | 0% |
| Beginner | 90% | 10% |
| Intermediate | 70% | 30% |
| Upper-Inter. | 50% | 50% |
| Advanced | 30% | 70% |
| Proficient | 10% | 90% |

#### 14.5.6 Summary / KPI 리포트

- Starter–Beginner: 기본 Report + 오류 리스트
- Intermediate–Upper: KPI 기본~상세
- Advanced–Proficient: Feedback + KPI 심층

#### 14.5.7 Free Talking(FRT) 레벨별 대화 전략 *(v0.12 — 코드 갱신)*

Free Talking 세션은 레벨에 따라 **질문 유형·개입 방식·피드백 언어·세션 시간**이 전면 다르게 설계된다. (v0.11의 CMP는 v0.12에서 시나리오 기반 FRT로 재정의되었다.)

| 항목 | 초급 (L1–6, A1–A2) | 중급 (L7–12, B1–B2) | 고급 (L13–18, C1–C2) |
|---|---|---|---|
| **첫 질문 유형** | Closed / Semi-open (Yes·No, 2~3어절 응답 유도) | Open-ended (이유·경험 제시 요구) | 의견 충돌·전략적 질문 (주장·반론 대응) |
| **예시 질문** | "Do you like working late?" | "What would you do if your boss asked you to stay late suddenly?" | "How should teams handle sudden deadline changes fairly?" |
| **개입 방식** | 답변 지연 시 **힌트 문장 완성형 제안** (한국어 포함) 예: "남을까요? I'd stay or I'd not stay로 시작해보세요." | 발화 직후 5초 이내 **AI 리프레이즈** (의미 연결 중심) 예: "would + 동사 형태로 가정법을 써야 자연스러워요." | 대화 방해 없이 **서브 코멘트 사이드 패널** 표시 (논리흐름·표현대안·반론대응) |
| **무응답 힌트** | 제공 (문장 완성형) | 제공 | **제공 안 함** |
| **피드백 언어** | **바이링구얼** (한국어+영어 혼합 말풍선) | **바이링구얼** (한국어+영어) | **영어 메타 피드백만** |
| **세션 제한 시간** | **3분** | **5분** | **7분** |
| **재질문 스타일** | 쉬운 후속 질문 + 즉각적 긍정 피드백 | 의견 → 이유 → 경험 연결 질문 | 반론 제시 / 재질문 / 논리 전개 요구 |
| **피드백 말풍선 색** | 일반 대화와 구분되는 색 | 일반 대화와 구분되는 색 | 일반 대화 속 자연스러운 서브 패널 |

**대화 종료 조건**: 레벨별 제한 시간 만료 시 자동 종료 메시지 출력
> "Time's up for today, but great conversation as always. I'll send a summary to help you review!"

### 14.6 KPI 측정 체계 *(⚠️ v0.10 운영 정합화)*

> **확장 요약**: v0.8까지는 Speaking FRT(Free Talking, 구 CMP) 전용 5지표였으나, v0.9에서는 **40개 모듈을 아우르는 70 KPI 코드 레지스트리**로 확장. v0.12에서 모듈은 29개로 정리되어 일부 KPI는 고아 상태(§13.1.6 폐기 모듈 KPI 재할당 후속 과제). 8대 역량 분류, 2-depth 계층(역량→하위목표→측정항목), 측정 도구 구분, 레벨별 목표 벤치마크, 모듈×KPI 매핑을 통합.
>
> ⚠️ **v0.10 정합화**: 70 KPI 레지스트리(📋 기획)는 유지하되, **실제 운영 단위는 `KPI_CATEGORY` 코드그룹의 `extraData.goal` 멀티셀렉트(✅ 구현 완료)**임을 명시. 두 체계의 매핑·운용 방식을 §14.6.0에 정의.
>
> 📎 **상세 컴패니언 문서**: 70 KPI 전체 속성 및 모듈×KPI 매핑 매트릭스는 `Picklass_KPI_체계_상세.md` 참조.
>
> 📊 **엑셀 카탈로그 연동**: `Picklass_모듈_카탈로그.xlsx` (v0.4)의 시트 — `KPI 코드` / `모듈×KPI 매핑` / `레벨 벤치마크` 3종 참조.

#### 14.6.0 운영 모델 — `KPI_CATEGORY` 코드그룹 *(✅ 구현 완료)*

실제 시스템에서는 KPI를 **백오피스 코드그룹 `KPI_CATEGORY`**로 관리하며, 사용자(강사·학생)에게 노출되는 단위는 **`extraData.goal` 텍스트**이다.

```
[코드그룹] KPI_CATEGORY
  ├─ code: 'FLUENCY_RATE'
  │   extraData: { goal: '말하기 유창성 향상' }
  ├─ code: 'PRAGMATICS'
  │   extraData: { goal: '화용적 의미 이해' }
  ├─ code: 'PRONUNC'
  │   extraData: { goal: '발음 정확도' }
  └─ ...
```

**운영 흐름**

```
[강사·학생 UI]
  goal 텍스트 멀티셀렉트 (예: "말하기 유창성", "화용성")
        ↓
[Studio·Tutoring 프론트]
  내부적으로 goal → code 변환 (FLUENCY_RATE, PRAGMATICS)
        ↓
[NestJS API → analyzer 서버]
  selected_kpi_codes: ['FLUENCY_RATE', 'PRAGMATICS']
        ↓
[저장]
  courses.kpi_codes TEXT[] = ARRAY['FLUENCY_RATE', 'PRAGMATICS']
  ai_modules.selectedKpiCodes = [...]
```

**운영 데이터 위치 (✅ 구현 완료)**

| 항목 | 위치 |
|---|---|
| KPI 마스터 정의 | 백오피스 시스템관리 → 코드관리 → `KPI_CATEGORY` 코드그룹 |
| 모듈 ↔ KPI 매핑 | `ai_modules.selectedKpiCodes: TEXT[]` |
| 과정 ↔ KPI 매핑 | `courses.kpi_codes: TEXT[]` (2026-04-23 신설, NOT NULL DEFAULT '{}') |
| analyzer 시퀀싱 입력 | `selected_kpi_codes: string[]` |
| 학생 화면 표시 | `goal` 텍스트 (코드 미노출) |

> **❌ 폐기**: 이전의 `ai_modules.learning_objectives TEXT[]` 컬럼은 2026-04-22 마이그레이션으로 DROP. 모듈 학습 목표는 KPI_CATEGORY 코드 매핑으로 대체.

#### 14.6.0+ KPI_CATEGORY ↔ 70 KPI 레지스트리 정합화

| 항목 | KPI_CATEGORY 운영 모델 | 70 KPI 레지스트리 (기획) |
|---|---|---|
| 단위 | 코드 + goal 텍스트 (1:1) | 코드 + 8대 역량 + 2-depth 계층 |
| 코드 수 | (📋 운영 코드 수 별도 집계) | 70 (`Picklass_KPI_체계_상세.md` §2) |
| 사용자 노출 | goal 텍스트 멀티셀렉트 | (현 시점 미노출) |
| 시퀀싱 호환성 | `selected_kpi_codes` 직접 전달 | 동일 코드 체계 사용 시 호환 |
| 정합화 후속 | KPI_CATEGORY 운영 코드를 70 레지스트리와 1:1 매핑하는 백필 작업 필요 | `picklass-backoffice/docs/ai-modules/20260423_KPI기반_시퀀싱_작업목록.md` §10 (P1 미착수) |

#### 14.6.1 Speaking 5지표 (FRT Free Talking 전용 · v0.12 코드 갱신)

Free Talking(FRT) 세션의 사용자 피드백·리포트에 우선 적용되는 5지표 (기존 v0.8 정의 유지, v0.12에 코드만 갱신).

| 지표 | 정의 (개략) | 측정 방법 |
|---|---|---|
| 유창성 (Fluency) | 발화 속도, 끊김 횟수 | WPM + 침묵 구간 분석 |
| 정확성 (Accuracy) | 문법·어휘 오류율 | AI 분석 |
| 복잡성 (Complexity) | 문장 구조 다양성 | 구문 분석 |
| 상호작용성 (Interactivity) | 질문·호응 빈도 | 턴 분석 |
| 발음 (Pronunciation) | 단어별 발음 점수 | STT + Phonetic 분석 |

> 이는 Free Talking 세션의 **요약 대시보드용 대표 지표**이며, 하위의 세분화된 70 KPI 코드 중 주요 항목을 묶은 상위 뷰이다.

#### 14.6.2 통합 KPI 레지스트리 — 8대 역량 분류 *(v0.9 신설)*

전체 KPI는 **8대 역량**으로 대분류되며, 각 역량 아래 **하위 목표(Sub-goal)**가 있고, 하위 목표 아래 **측정 항목(Measurement Item)**이 있는 2-depth 구조.

| 대분류 (역량) | 하위 목표 (Sub-goal) | 측정 항목 수 | 대표 KPI 코드 |
|---|---|---|---|
| **읽기 (Reading)** | 해독 및 유창성, 이해 전략, 비판적 읽기, 다양한 텍스트 유형 | 11 | SILENT_READING, KEYWORD_HIT, READING_SPEED, TOPIC_SELECTION, TYPE_ACCURACY, EVIDENCE_EXTRACT, READ_COMPLETION, STAY_TIME, READ_ACCURACY, CHUNKING_APPROP, FLUENCY_GROWTH, PREDICT_COMP, PREDICT_CRITICAL, INFER_VALIDITY |
| **듣기 (Listening)** | 음성 인식, 청취 이해, 실시간 처리, 다양한 담화 유형 | 8 | LINKED_SPEECH, DICTATION_ACC, MAIN_IDEA, TYPE_SCORE, DETAIL_LISTENING, INFER_LISTENING, PRAGMATIC_RECOG, PHONEME_DISCRIM, LIAISON_RECOG |
| **말하기 (Speaking)** | 발화량, 발음 및 운율, 유창성, 상호작용 기술 | 14 | ASR_ACCURACY, ASR_RECOG, PROSODY_PATTERN, SPEAKING_RATE, RESPONSE_LATENCY, HESITATION_FREQ, MLU_LENGTH, SILENCE_RATIO, TOTAL_UTTERANCE, INTERACTION_ACT, FLUENCY_WPM |
| **어휘 (Vocabulary)** | 어휘 인지 및 의미, 어휘 관계, 어휘 사용, 어휘 학습 전략 | 13 | VOCAB_RECOG, RECOG_SPEED, VOCAB_ACCURACY, PRONUN_ACCURACY, IMAGE_MATCHING, COLLOC_RECOG, VOCAB_PROPRIETY, CONTEXT_INFER, SEMANTIC_CONN, RELATION_APPROP, DERIV_RECOG, ROOT_ANALYSIS, ADV_VOCAB, VOCAB_DIVERSITY |
| **문법 (Grammar)** | 문법 정확성, 맥락적 문법, 오류 인식 및 교정, 형태 및 통사 | 11 | BLANK_ACCURACY, GRAMMAR_ITEM, EXPRESSION_APPROP, SENTENCE_COMPLEX, GRAMMAR_ERROR, ACCURACY_SCORE, RE_UTTERANCE, WORD_ORDER, SENTENCE_ACCURACY, ERROR_DETECTION, GRAMMAR_ASSIST |
| **쓰기 (Writing)** | 문장 구성, 쓰기 과정, 문단 및 담화 조직, 장르별 쓰기 | 10 | HANDWRITING_MATCH, KEYWORD_BLANK, STRUCT_COMPLEX, WRITING_ERROR, SEMANTIC_IDENTITY, KEYWORD_RATE, LOGIC_STRUCT, OUTLINE_COMPLETE, LOGIC_COHESION, ESSAY_STRUCT, SELF_CORRECTION |
| **실시간 처리 (Real-time)** | 응답 지연, 화용 이해 | 3 | ASR_ACCURACY, RESPONSE_LATENCY, PRAGMATIC_RECOG |
| **총합** | | **70개** (일부 KPI는 복수 역량 교차 매핑) | |

> 📎 **KPI 코드 전수 목록(70개) · 측정 유닛 · 측정 방법 · 측정 도구**는 `Picklass_KPI_체계_상세.md` §2 참조.

#### 14.6.3 측정 도구 분류

| 도구 | 설명 | 비용 | KPI 수 |
|---|---|---|---|
| **측정 (measured)** | 룰 기반 자동 채점 (단어 매칭, 클릭 정답, 타이머, WPM 계산 등) | $0 (로컬) | ~40개 (57%) |
| **LLM** | AI 분석 (문맥 유추, 의미 관계, 구조 완성도 등) | API 비용 발생 | ~30개 (43%) |
| **mixed** | 복합 (일부 측정 + 일부 LLM) | 일부 비용 | 일부 모듈 전용 |

이 분류는 `ModulePlanningMeta.automationLevel` 속성과 연동되어 **Planner가 시간/비용 예산에 따라 자동 최적화** 가능.

#### 14.6.4 측정 유닛 표준

| 유닛 | 용도 | 예시 KPI |
|---|---|---|
| `%` | 정답률·정확도 | VOCAB_RECOG, KEYWORD_HIT, TYPE_ACCURACY |
| `ms` | 반응 시간 | RECOG_SPEED, RESPONSE_LATENCY |
| `WPM` | 단어/분 (읽기·말하기 속도) | SILENT_READING, SPEAKING_RATE, READING_SPEED |
| `WPM Delta` | 회차별 WPM 변화 | FLUENCY_GROWTH |
| `Score (1-5)` | 루브릭 5점 척도 (LLM 평가) | VOCAB_PROPRIETY, INFER_VALIDITY, LOGIC_STRUCT |
| `Score (0-1)` | 확률/유사도 | SEMANTIC_IDENTITY |
| `Count` | 빈도·카운트 | READ_COMPLETION, HESITATION_FREQ, MLU_LENGTH |
| `Sec/Min` | 체류·발화 시간 | STAY_TIME, TOTAL_UTTERANCE |
| `Ratio (TTR)` | 비율 지표 | VOCAB_DIVERSITY |
| `Length/Level` | 길이·레벨 증감 | SENTENCE_COMPLEX |

#### 14.6.5 레벨별 목표 벤치마크 *(v0.9 신설, 재검토 후 확정 예정)*

엑셀 원자료 기반 초안 수치. **실제 서비스 런칭 전 재검토·확정 필요** (v0.9 태그 = `DRAFT`).

| KPI | PreA1 | A1 | A2 | B1 | B2 | C1+ | 단위 |
|---|---|---|---|---|---|---|---|
| **읽기 속도** (SILENT_READING, READING_SPEED) | 40–60 | 60–90 | 90–120 | 120–150 | 150+ | 150+ | WPM |
| **말하기 속도** (SPEAKING_RATE, FLUENCY_WPM) | — | 40–60 | 60–90 | 90–120 | 120–150 | 150+ | WPM |
| **MLU 평균 발화 길이** (MLU_LENGTH) | — | 3–5 | 5–8 | 8–12 | 12+ | 12+ | 단어/발화 |
| **문장 복잡도** (SENTENCE_COMPLEX) | — | 5–8 | 8–12 | 12–15 | 15+ | 15+ | 단어/문장 |
| **문법 오류율** (GRAMMAR_ERROR, ACCURACY_SCORE) | — | — | 10%↑ 개선 | 5–10% 보통 | 5%↓ 우수 | 5%↓ 우수 | % |
| **침묵 비율** (SILENCE_RATIO) | — | — | — | 20%↓ 유창 | 20%↓ 유창 | 20%↓ 유창 | % |
| **총 발화 시간** (TOTAL_UTTERANCE) | — | 30–60 | 60–90 | 90+ | 90+ | 90+ | 초 |
| **읽기 정확성 ASR** (READ_ACCURACY) | 80%↓ 부적절 | 80–90% 지도 필요 | 90%↑ 독립 | 90%↑ 독립 | 90%↑ 독립 | 90%↑ 독립 | % |
| **ASR 인식률** (ASR_ACCURACY, ASR_RECOG) | — | — | 80%↑ 이해 가능 | 80%↑ | 80%↑ | 80%↑ | % |
| **핵심 키워드 충족률** (KEYWORD_RATE) | — | — | — | 80%↑ 합격 | 80%↑ 합격 | 80%↑ 합격 | % |

> 📎 **전체 벤치마크 표**(70 KPI × 6 레벨 × 경고/합격/우수 3단계 기준)는 `Picklass_KPI_체계_상세.md` §4 참조.

#### 14.6.6 모듈 × KPI 매핑 요약

각 모듈은 **1~3개의 핵심 KPI**를 주 측정 지표로 가진다. §13.1 레지스트리의 `핵심 KPI` 컬럼과 1:1 대응.

| 영역 | 모듈 예시 | 대표 KPI |
|---|---|---|
| Vocabulary | WRD | VOCAB_RECOG + RECOG_SPEED |
| Vocabulary | IMG | IMAGE_MATCHING |
| Reading | SCN | SILENT_READING + KEYWORD_HIT |
| Reading | QAR | TYPE_ACCURACY + EVIDENCE_EXTRACT |
| Reading | RRD | FLUENCY_GROWTH + ASR_ACCURACY *(영역 이동 v0.12)* |
| Speaking | EDR *(구 SHD 흡수)* | PROSODY_PATTERN + SPEAKING_RATE |
| Speaking | FRT | TOTAL_UTTERANCE + SILENCE_RATIO + INTERACTION_ACT |
| Writing | SWR | STRUCT_COMPLEX + WRITING_ERROR |
| Writing | PWR | ESSAY_STRUCT + SELF_CORRECTION |

> 📎 **v0.14 Pick-Speak Method 정렬에 따라 29 모듈 × 70 KPI 매트릭스로 재구성 진행 중**. 폐기 모듈(LGS, DIC, LQR, DLT, COL, SHR, PWT, TTS-SIM, FIB, PIC, KED, QAR-S, **RST**)의 KPI는 고아 상태이며, 후속 작업에서 (a) 다른 모듈로 재할당 또는 (b) KPI 자체 폐기 결정. RPL(구 RLP)은 INTERACTION_ACT·SPEAKING_RATE 등 기존 KPI를 그대로 사용. 상세 매핑은 `Picklass_KPI_체계_상세.md` §3 또는 엑셀 `모듈×KPI 매핑` 시트 참조.

#### 14.6.7 대시보드·리포트 활용

| 화면 | 표시 범위 | 주체 |
|---|---|---|
| Tutoring 학생 대시보드 | 본인 KPI 추이 (스킬별 요약) | 학생 |
| Speaking FRT 세션 종료 리포트 | 5지표(§14.6.1) + 상세 KPI | 학생 |
| Studio 강사 모니터링 | 배정 학생의 모듈별·KPI별 집계 | 강사 |
| Admin Billing/성과 | 기관 전체 평균 KPI, 벤치마크 대비 | 학원관리자·본부 |

### 14.7 음성 인프라 (Speech Stack)

#### 14.7.1 STT
- **Primary**: Azure Speech (koreacentral/eastus)
- **Secondary**: Google STT
- Confidence 임계값 0.5 기준 재요청

#### 14.7.2 TTS
- **Primary**: Azure Neural TTS (WPM 제어 가능)
- 레벨별 말속도 조절 (§14.2.2 공식)

#### 14.7.3 스트리밍
- LLM 응답 첫 청크 도착 즉시 TTS 시작 → 체감 지연 87%↓
- SSE 기반 토큰 단위 전송

#### 14.7.4 마이크 권한 · 노이즈 · 끊김
- 브라우저 권한 요청 UX
- 배경 소음 필터
- 네트워크 끊김 시 로컬 상태 보존 후 재접속 복구

#### 14.7.5 로컬 우선 처리
- 침묵 힌트·Confidence 재요청은 LLM 미호출 (비용↓)

#### 14.7.6 듀얼 트랙 STT (Whisper + Azure) *(v0.11 신설 — 회의록 260413, 프로토타입 공유)*

> 📋 **기획 단계 (검토 중)** — Azure 단일 STT의 인식률 한계를 보완하기 위한 **이중 트랙 음성 처리** 전략. 회의록 "프로토타입 공유" 항목에 기록된 검토 내용 반영.

**문제 정의**

| 문제 | 영향 |
|---|---|
| Azure STT 인식률 한계 | 한국 학습자 발화(L1 간섭, 비원어민 억양)에서 ASR 오류율 상승 |
| 발화 종료 → AI 응답 생성까지 딜레이 | 대화 몰입 저해, 학습 흥미 감소 |
| 분석 정확도와 응답 속도 트레이드오프 | 정확도↑ → 속도↓ 또는 그 반대 |

**해결 전략 — Whisper(STT) + Azure(분석) 듀얼 트랙**

| 트랙 | 역할 | 엔진 | 비고 |
|---|---|---|---|
| **트랙 A. STT (음성→텍스트)** | 학습자 발화의 1차 텍스트화 | **OpenAI Whisper** (또는 호환 모델) | 한국어 화자 영어 발화 인식률 우수, 라벨 자유도↑ |
| **트랙 B. 분석 (발음·억양·운율)** | 발화 품질 KPI 측정 | **Azure Speech Pronunciation Assessment** | 기존 KPI 산출 파이프라인 그대로 재사용 |

**처리 흐름**

```
[학습자 발화]
    ↓
[병렬 전송]
  ├─ 트랙 A: Whisper STT → 텍스트 → SpeakingConversationAgent (즉시 응답 생성)
  └─ 트랙 B: Azure 분석 → KPI(READ_ACCURACY, ASR_ACCURACY, FLUENCY_WPM 등) → KpiResult
    ↓
[AI 응답 TTS 시작] ← 트랙 A 결과로 빠르게 응답 시작
[KPI 누적]      ← 트랙 B 결과는 세션 종료 후 분석 시 활용
```

**응답 딜레이 단축 전략 (회의록 명시)**

| 단계 | 기존 | 개선안 |
|---|---|---|
| 학습자 발화 종료 감지 | VAD(Voice Activity Detection)가 침묵 1.5초 후 확정 | 침묵 임계값 0.7초로 단축, 발화 종료 예측 모델 추가 검토 |
| STT 처리 | Azure STT 단일 (서버 라운드트립) | Whisper(병렬) — 첫 토큰 더 빠름 |
| AI 응답 생성 | STT 완료 후 LLM 호출 | STT 부분 결과(streaming) 도착 즉시 LLM 프리워밍 |
| **학습 분석** | 발화 즉시 KPI 산출 | **대화 종료 후 일괄 분석** — 응답 지연에 영향 없음 |

> ⚠️ **검토 사항**: 듀얼 트랙은 추가 비용(LLM 토큰 외 STT API 비용)이 발생한다. **회의록 명시**: 현재 10분당 150~250원 수준 원가가 사용 모델 변경 시 변동 가능. **§22 보안 및 컴플라이언스** 참조.

### 14.8 API 호출 최적화 및 비용

| 항목 | Before | After | 개선 |
|---|---|---|---|
| API 호출 횟수 | 2회/턴 | 1회/턴 | 50%↓ |
| 응답 시간 | ~1,300ms | ~800ms | 38%↓ |
| 세션당 비용 | $0.057 | $0.049 | 14%↓ |

**추가 최적화**: 캐싱(공통 피드백), 배치 처리, claude-haiku 우선 사용.

### 14.9 독립 앱 UX (Speaking 전용 UI/UX)

#### 14.9.1 세션 시작 화면
- 레벨 선택 (1–18) / 주제 / 세션 시간
- 목표·힌트 안내
- **모드 선택 (FRT 전용)**: `Chatting Mode` / `Audio Only` 토글 (§14.9.5)

#### 14.9.2 대화 화면
- 음성 파형 시각화
- 자막 on/off 토글
- 힌트 버튼 (레벨/상황별 3단계)

#### 14.9.3 실시간 피드백 패널
- 추임새(즉각) → 교정 텍스트 → 따라 읽기(L1–12)

#### 14.9.4 세션 종료 리포트
- KPI 5지표 차트
- 대화 스크립트 + 오류 하이라이트
- 다음 권장 주제/레벨

#### 14.9.5 Chatting Mode / Audio Only 토글 *(v0.8 신설)*

Free Talking(FRT) 세션은 두 가지 모드로 운영된다. 사용자가 세션 시작 화면에서 선택.

| 모드 | 인터페이스 | 사용 상황 | 특징 |
|---|---|---|---|
| **Chatting Mode** | 채팅창 + 음성 말풍선 병행 | 초·중급 학습자, 자막 필요 시 | AI·학습자 발화 텍스트 말풍선으로 표시, 피드백 말풍선 색상 구분(일반 대화와 다른 색), 레벨별 바이링구얼 피드백(초·중급: 한국어+영어) |
| **Audio Only** | 음성 파형 + 최소 UI | 고급 학습자, 실전 회화 훈련 | 텍스트 최소화, 대화 몰입 극대화, 발화 감지 시 파형 시각화, 고급 전용 서브 코멘트 사이드 패널(논리흐름·표현대안·반론대응) |

**공통 규칙**
- AI 원어민 발화 완료 후 마이크 자동 활성화
- 무응답 5초(화이트노이즈만 입력) 시 자동 종료 또는 레벨별 개입
- 발화 말풍선/파형은 학습자 [Stop] 클릭 또는 제한 시간 만료 시 종료

> 📎 레벨별 개입 전략(초급 힌트 / 중급 AI 리프레이즈 / 고급 서브 코멘트)은 §14.5 참조.

### 14.10 통합 모듈 연동 (튜터링 레슨 내 삽입)

> 📎 전체 사용자 플로우·인증 메커니즘은 **§3.3.1 통합 모듈 플로우** 참조. 본 절은 엔진·데이터 레이어 상세.

#### 14.10.1 LessonPlan 내 스피킹 모듈 슬롯
- ModuleSequence에 Speaking 모듈 code 추가
- `passageExposureMode = "full"` (지문 기반 대화 시)

#### 14.10.2 오케스트레이터 ↔ SpeakingConversationAgent 핸드오프
- ModuleOrchestratorAgent가 Speaking 모듈 진입 시 세션 시작 신호
- Speaking 세션 종료 시 KpiResult를 ModuleResult에 포함

#### 14.10.3 결과 데이터 표준화
- ModuleHistory 테이블에 호환 저장
- SpeakingSession 테이블에 상세 턴/오디오/교정 저장 (외래키 연결)

#### 14.10.4 ONETIME TOKEN 기반 크로스도메인 인증 (§3.3.1 연동)

Tutoring(`tutoring.picklass.com`) → Speaking(`speaking.picklass.com`) 도메인 이동 시 **단일 사용 토큰**으로 인증과 설정을 전달한다.

**토큰 라이프사이클**

```
[1] Tutoring: 사용자가 "스피킹 선택" 클릭
      ↓ 서버에서 ONETIME_TOKEN 발급 (UUID, TTL 5분)
      ↓ 토큰·사용자ID·레벨·콘텐츠·대화분량·음성모드 레코드 생성
[2] Tutoring → Speaking: 쿼리스트링으로 토큰 전달
      https://speaking.picklass.com/?token={TOKEN}
[3] Speaking 세션 시작 화면 진입 시:
      POST /api/auth/get-auth { token }
        응답: { userId, name, accessCode, level, content, dialogLength, voiceMode }
      → 이름·설정 자동 채움 (이름은 사용자 수정 가능)
[4] 사용자 "세션 시작" 클릭:
      POST /api/auth/revoke-token { token }
      → 토큰 상태를 소멸 처리 (이후 동일 토큰 재사용 불가)
[5] 세션 진행·종료 → ModuleHistory 및 SpeakingSession 기록
```

**보안 요구사항**
- 토큰 TTL ≤ 5분 (만료 시 자동 무효)
- 토큰 1회 사용 강제 (GET_AUTH 호출 후 세션 시작 시점에 즉시 소멸)
- IP/User-Agent 핑거프린팅으로 도용 탐지 (선택)
- 토큰 로그는 감사 로그(§22.4)에 기록

**실패 처리**
- 만료·사용된 토큰: Speaking 측 "세션 초기화 실패" 안내 → Tutoring 복귀 CTA
- GET_AUTH API 장애: 5초 타임아웃 후 재시도(최대 2회), 실패 시 사용자에게 수동 재진입 유도

**API 스키마 (요약)**

```typescript
// GET_AUTH 요청
interface GetAuthRequest { token: string; }

// GET_AUTH 응답
interface GetAuthResponse {
  userId: string;
  name: string;           // 기관 제공값 (수정 가능)
  accessCode: string;
  level: number;          // CEFR 1–18
  content: string;        // 주제/지문 식별자
  dialogLength: number;   // 대화 분량 (분 또는 턴 수)
  voiceMode: string;      // TTS 음성 선택
}

// REVOKE 요청
interface RevokeTokenRequest { token: string; }
```

#### 14.10.5 외부 시스템 임베드 — `GET /external/verify` *(✅ 구현 완료)*

> §3.3 외부 임베드 채널의 **실제 운영 API**. 2026-04-09자 외부 API 명세서에 정의되어 있으며 이미 동작 중.
>
> 📎 **v0.19 cross-ref**: 본 verify API는 **§16. B2B 제휴 운영 모델**의 표준 인증 방식 중 하나(§16.7 ONETIME TOKEN)이며, **§16.10.2 Speaking 채널 적용**에서 Pick-Speak Method 임베드의 핵심 진입점으로 사용된다. 본 절은 API 명세 중심, §16.10.2.6은 운영 흐름 중심으로 다룬다.

외부 시스템(예: 파고다SCS)이 자체 사용자·과정을 관리한 채로 픽클래스의 Speaking 학습 화면만 임베드 형태로 제공하기 위한 API.

**인증 방식**

| 헤더 | 필수 | 설명 |
|---|---|---|
| `X-Access-Token` | ✓ | 픽클래스에서 발급한 원타임 토큰. 토큰 유효 시간 **60분** |
| `X-Module-Code` | ✓ | 조회할 모듈 코드 (현재: `SNR` Scenario Talking, `FRT` Free Talking) |

**엔드포인트**

```
GET https://tutoring.picklass.com/api/external/verify
```

**응답 구조 (200 OK)**

```typescript
interface ExternalVerifyResponse {
  lessonId: string;            // UUID
  user: {
    userId: string;            // 외부 시스템 사용자 ID
    name: string;
  };
  module: {
    code: string;              // 'SNR', 'FRT'
    name: string;
    skill: string;             // 'speaking'
    uiTemplate: string;        // §13.5 신규 필드 — 'standard'|'voice'|'embedded'|'hidden'
    answerType: string;
    scoringMode: string;
    passageMode: string;       // §13.5 — 'full'|'hidden'|'preview'|'timed-blur'|'timed-select'
    feedbackStyle: string;
    questionCount: string;     // §13.5 — 'single'|'multi'|'content-driven'
    questionMinCount: number;
    questionMaxCount: number;
    hintTypes: string[];       // §13.5 — 다중 선택 (이전 CSV 단일 문자열에서 변경)
  };
  lesson: {
    id: string;
    topic: string;
    lessonOrder: number;
    text: {
      title: string;
      content: string;
      level: string;
      category: string;
      wordCount: number;
    };
  };
}
```

**사용 흐름**

```
[외부 시스템 (예: 파고다SCS)]
    │
    │ 1. 픽클래스에 미리 발급받은 ONETIME TOKEN을 자체 학습자에게 제공
    ▼
[학습자 → 외부 시스템 페이지]
    │
    │ 2. "스피킹 학습 시작" 클릭 → 픽클래스 임베드 위젯/페이지로 이동
    │    (token + module-code를 헤더로 전달)
    ▼
[픽클래스 Tutoring API]
    │
    │ 3. GET /external/verify (토큰 검증 + 사용자·모듈·지문 정보 반환)
    │ 4. 임베드 위젯이 응답 데이터로 학습 화면 구성
    ▼
[학습 진행 → 학습 결과 저장 (외부 시스템 + 픽클래스 자체 로그 동시)]
```

**v0.11에서 다룰 후속 (회의록 8개 항목 본편)**: 외부 시스템과의 학습 결과 회신 webhook, 수료 기준 자동 전송 정책 등은 v0.11에서 추가 설계 예정.

### 14.11 구현 우선순위 및 일정

Phase 1 (MVP, ~2026 Q2):
1. SpeakingConversationAgent (3-Phase 플로우)
2. FRT (Free Talking) 모듈 최우선
3. 레벨 1–9 지원 → 단계적 10–18 확장
4. 독립 앱 세션 화면 + 종료 리포트

Phase 2 (2026 Q3):
- 6모듈 순차 구현 (LRN → VLM → EDR → RPL → FRT 고도화 → OMP) — v0.15 Pick-Speak 정렬
- 5축 KPI 고도화 (발음정확도·유창성·문법정확성·화용성·발화량)
- 통합 모듈 연동 완성

### 14.12 리스크

| 리스크 | 영향 | 완화책 |
|---|---|---|
| 음성 지연 | UX 치명적 | 스트리밍 + 추임새 우선 재생 |
| API 비용 폭증 | 수익성↓ | 통합 처리, 캐싱, 로컬 Fallback |
| 프라이버시 | 법적 리스크 | 기본 비저장, 명시적 동의 시만 저장 |
| 미성년 음성 수집 동의 | 컴플라이언스 | 보호자 동의 체계, COPPA/PIPA 검토 |
| 음성 인식 오류 | 학습 방해 | Confidence 임계값 + 재요청 UX |

---

## 15. 학습 진단·평가 — 레벨 테스트 체계 *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계** — 회의록 결정에 따라 픽클래스의 **레벨 진단·성취도 평가** 체계를 일원화한다. 본 장은 (a) 학습 시작 시점의 **Level Test**, (b) 학습 종료 시점의 **Achievement Test**, (c) 진단 결과의 활용, (d) 전사(B2B/B2C/전화/인강/출강) 일원화 로드맵을 정의한다.

### 15.1 핵심 원칙

회의록(260413) 결정 사항을 5개 원칙으로 정리한다.

| 원칙 | 내용 | 출처 |
|---|---|---|
| **P1. 별도 레벨 테스트 미강제** | **레벨 확인 수업 1회**를 통해 별도 레벨 테스트 없이 학습자 수준 자동 분석 | 회의록 "레벨 설계" |
| **P2. 점수 표기 이중화** | **CEFR 기준 점수** + **파고다 레벨**을 병행 표기 | 회의록 "레벨 설계" |
| **P3. Level/Achievement Test 동일 방식** | 학습 전 Level Test와 학습 종료 후 Achievement Test는 **동일 방식**이어도 무관 | 회의록 "추후 고도화 1)" |
| **P4. 정밀 진단 (고도화)** | 체계적인 루브릭에 기반한 **정밀 레벨테스트 결과**, MBTI 형태의 개인 성향·학습 환경 반영 | 회의록 "추후 고도화 1)" |
| **P5. 전사 일원화** | B2B, B2C, 전화/인강/출강 등 **전사 레벨테스트 체계 일원화** | 회의록 "추후 고도화 6)" |

### 15.2 Level Test (학습 시작 시 진단)

#### 15.2.1 진입 경로

| 진입 채널 | 흐름 |
|---|---|
| 신규 학습자 (B2C/B2B 공통) | 회원가입 → "레벨 확인 수업" 자동 배정 (P1) |
| 기존 학습자 신규 과정 등록 | 첫 레슨이 자동으로 Level Test 모드로 진행 |
| 강사·관리자 수동 트리거 | Studio/Admin에서 "레벨 재진단" 발급 (재시험 시) |

#### 15.2.2 진행 방식

**기본 — 레벨 확인 수업 1회 (P1)**

별도의 객관식 시험 없이, 학습자가 1회 수업을 진행하면서 발생한 **발화·청해·읽기·쓰기 데이터**를 자동 수집·분석하여 레벨 추정. 회의록 P1 원칙에 따라 학습자에게 "시험"이라는 부담을 주지 않는다.

**측정 데이터 (Level Test 모드)**

| 영역 | 측정 데이터 | 산출 KPI |
|---|---|---|
| Speaking | 발화량·MLU·WPM·문법 오류율·어휘 다양도 | SPEAKING_RATE, MLU_LENGTH, GRAMMAR_ERROR, VOCAB_DIVERSITY |
| Listening | 음성 콘텐츠 이해도 (Q&A) | LISTEN_GIST, LISTEN_DETAIL |
| Reading | 읽기 속도·이해도 | SILENT_READING, READ_COMPLETION |
| Writing | 작문 길이·정확도·표현력 | TYPE_ACCURACY, VOCAB_PROPRIETY |

> 📎 KPI 코드 정의는 §14.6, `Picklass_KPI_체계_상세.md` 참조.

#### 15.2.3 레벨 산출 알고리즘 (개요)

```
[레벨 확인 수업 종료]
    ↓
[KPI 누적 — 영역별 점수]
    ↓
[CEFR 분류 모델 (LLM + 룰 기반 하이브리드)]
   - PreA1 / A1 / A2 / B1 / B2 / C1 / C2
    ↓
[CEFR ↔ 파고다 레벨 매핑 (P2)]
   - Pagoda L1 ~ L18 (Picklass §14.2 정의)
    ↓
[학습자 페이지 노출]
   - "당신의 CEFR 레벨: A2"
   - "파고다 레벨: L5 (CEFR A2 중반)"
```

#### 15.2.4 CEFR ↔ 파고다 레벨 매핑 (참조)

§14.2 정의를 본 장에서 일원화 표로 명시.

| CEFR | 파고다 그룹 | 파고다 레벨 |
|---|---|---|
| PreA1 | Beginner | L1 |
| A1 | Beginner | L2~L3 |
| A2 | Elementary | L4~L6 |
| B1 | Intermediate | L7~L10 |
| B2 | Upper-Intermediate | L11~L14 |
| C1 | Advanced | L15~L17 |
| C2 | Master | L18 |

### 15.3 Achievement Test (학습 종료 시 성취도 평가)

#### 15.3.1 트리거

| 트리거 ID | 조건 |
|---|---|
| `ACH_COURSE_END` | 과정(course) 모든 레슨 완료 시 |
| `ACH_MONTHLY` | 월말 정기 평가 (옵션) |
| `ACH_MANUAL` | 강사/관리자 수동 발급 |

#### 15.3.2 동일 방식 원칙 (P3)

회의록 명시 — Level Test와 Achievement Test는 **동일한 진행 방식**으로 운영해도 무방. 즉:

| 측면 | Level Test | Achievement Test |
|---|---|---|
| 진행 형식 | 1회 학습 세션 | 1회 학습 세션 (동일) |
| 측정 영역 | 4기능 (S/L/R/W) | 4기능 (S/L/R/W) |
| 출력 | CEFR + 파고다 레벨 | CEFR + 파고다 레벨 + **변화량(Delta)** |
| 주요 차이 | 베이스라인 측정 | **베이스라인 대비 향상도 측정** |

→ Level Test 결과를 베이스라인으로 저장하고, Achievement Test에서 동일 측정 후 `Delta = Achievement - Level` 산출.

#### 15.3.3 결과 노출

| 항목 | 표시 형태 |
|---|---|
| 시작 레벨 | "학습 시작 시: A1 / Pagoda L3" |
| 현재 레벨 | "현재: A2 / Pagoda L5" |
| 변화량 | "📈 1단계 향상 (CEFR A1 → A2, +2 Pagoda 레벨)" |
| 영역별 변화 | Speaking +12 WPM, Listening +15%, Reading +30 WPM, Writing +5pt |

### 15.4 KPI 통합 진단 엔진 — 개인 절대 수준 측정 (Picklass Proficiency Diagnostics, PPD) *(v0.13 신설)*

> 📋 **기획 단계 — 본 절은 Level Test(§15.2) / Achievement Test(§15.3) / 수료 기준(§11.10) / 게이미피케이션(§11.11) / 푸시·알림(§11.12) / 개인·그룹 리포트가 공통으로 사용하는 측정 엔진**이다. 각 모듈이 측정하는 다수의 KPI(§14.6, 70 코드)를 개인별로 정리해 **현재 수준에 대한 절대값 진단**을 산출한다.

#### 15.4.1 문제 정의

각 모듈은 자체 KPI를 측정한다(§13.1, §14.6.6). 학습이 누적되면 학습자 1명당 **수십~수백 개의 KPI 측정값**이 쌓인다. 이를 그대로 두면 다음 문제가 발생한다.

| 문제 | 결과 |
|---|---|
| KPI 단위·범위가 다름 | WPM, %, ms, count, score 등을 **단일 비교 척도로 환산 필요** |
| KPI 수가 너무 많음 | 학습자·강사·관리자가 한눈에 이해하기 어려움 — **압축 필요** |
| 모듈별·기간별 편향 | 자주 한 모듈만 한 학습자가 과대평가될 수 있음 — **균형 필요** |
| 절대 수준 비교 불가 | "이 학습자가 어느 수준인가?"에 대한 **단일 답이 없음** |

**해결 목표 — 절대값 진단 (Absolute Level Diagnosis)**

학습자 1명에 대해 다음 3개 출력을 **레벨/모듈/기간 무관한 절대값**으로 산출한다.

1. **Picklass Proficiency Score (PPS)** — 0~100 단일 종합 점수
2. **8대 역량별 점수** — Speaking/Reading/Listening/Writing/Vocabulary/Grammar/Fluency/Pragmatics 각 0~100
3. **CEFR + 파트너 레벨** — PPS로부터 자동 매핑

#### 15.4.2 진단 파이프라인 5단계

```
[Step 1] Raw KPI Collection           ← 모듈 세션 종료 시 KpiResult 적재
   ↓
[Step 2] Normalization                 ← 단위 통일 (raw → 0~100 normalized)
   ↓
[Step 3] Weighting                     ← KPI별 중요도 × 측정 신뢰도 × 시간 감쇠
   ↓
[Step 4] Capability Aggregation        ← 8대 역량별 가중 평균
   ↓
[Step 5] Composite Score (PPS)         ← 8대 역량 가중 종합 + 절대 레벨 매핑
```

#### 15.4.3 Step 1 — Raw KPI 적재

모듈 세션 종료 시점에 `kpi_results` 테이블에 적재. (§14.6.0 운영 모델 참조)

```sql
table kpi_results (
  id pk,
  user_id fk,
  session_id fk,           -- ModuleHistory 또는 SpeakingSession
  module_code varchar(10), -- 측정 모듈 (WRD/SUM/FRT 등)
  kpi_code varchar(40),    -- 70 KPI 코드 중 1
  raw_value numeric,       -- 단위별 원값
  unit varchar(20),        -- %, WPM, ms, count, score, ratio
  level_band enum,         -- 측정 시점 학습자 레벨
  measured_at timestamp,
  index (user_id, kpi_code, measured_at)
);
```

#### 15.4.4 Step 2 — KPI 정규화 (Normalization, raw → 0~100)

각 KPI는 단위가 다르므로 **0~100 정규화 점수**로 환산. **§14.6.5 레벨 벤치마크**의 floor(PreA1)·ceiling(C2) 기준값을 활용.

| 단위 | 정규화 공식 | 예시 |
|---|---|---|
| **% (정확도/완료율)** | `score = raw_value` (직접 매핑) | 정답률 80% → 80점 |
| **WPM (속도)** | `score = clamp((raw - floor) / (ceiling - floor) × 100, 0, 100)` | A2 floor=60·C2 ceiling=150, raw=80 → (80-60)/(150-60)×100 = **22점** |
| **ms (반응 속도, 짧을수록 우수)** | `score = clamp(100 - (raw - target) / target × 100, 0, 100)` | target=1000ms, raw=1500 → 100-(500/1000×100) = **50점** |
| **count (빈도)** | KPI별 룩업 테이블 (예: hesitation 0회=100, 5회=60, 10회=20) | hesitation 5회 → **60점** |
| **score 1-5 (루브릭)** | `score = raw × 20` | 4점 → **80점** |
| **ratio (TTR 등 0~1)** | KPI별 정규화 함수 (S자 곡선 등) | TTR 0.5 → **65점** |

> 📎 정규화 floor/ceiling은 §14.6.5 레벨 벤치마크 표의 PreA1~C2 6개 구간에서 추출하며, KPI 카탈로그(`kpi_definitions` 테이블)에 저장된다.

**정규화 결과 의미**
- **0~9**: 측정 자체 부진 (PreA1 수준 미만)
- **10~24**: A1 수준
- **25~39**: A2 수준
- **40~59**: B1 수준
- **60~79**: B2 수준
- **80~94**: C1 수준
- **95~100**: C2 수준 도달

#### 15.4.5 Step 3 — 가중치 적용 (Weighting)

각 KPI 측정값에는 3종 가중치가 곱해진다.

**(1) 중요도 가중치 (Importance, 1~5점)**

KPI 카탈로그에 사전 정의. 핵심 KPI는 5점, 보조 KPI는 1점.

| KPI 예시 | 중요도 | 사유 |
|---|---|---|
| FLUENCY_WPM | 5 | Speaking 핵심 측정값 |
| GRAMMAR_ERROR | 4 | 문법 정확성 핵심 |
| HESITATION_FREQ | 2 | 보조 신호 |
| RE_UTTERANCE | 1 | 부수 정보 |

**(2) 측정 신뢰도 (Confidence, 0~1)**

측정 횟수 N이 많을수록 신뢰 ↑.

```
confidence = min(1.0, N / N_min)
```
- N_min: KPI별 최소 측정 횟수 (보통 3~5회)
- N=0 → 0 (측정 없음 — 진단 불가)
- N=N_min → 1.0 (충분)

**(3) 시간 감쇠 (Recency Weight)**

오래된 측정값은 영향력 감소.

| 측정 시점 | 가중치 |
|---|---|
| 0~30일 이내 | **1.0** |
| 31~90일 | **0.7** |
| 91~180일 | **0.4** |
| 181일 이상 | **0.2** |

**최종 가중치**
```
weight = importance × confidence × recency
```

#### 15.4.6 Step 4 — 8대 역량별 점수 산출

각 KPI는 8대 역량(§14.6.2) 중 하나에 매핑된다. 역량별로 해당 KPI들의 **가중 평균**을 산출.

```
역량 점수 = Σ(정규화 점수 × 최종 가중치) / Σ(최종 가중치)
```

**예시 — "Fluency(발화 유창성)" 역량 산출**

| KPI | 정규화 | 중요도 | 신뢰도 | 시간 가중 | 최종 가중치 | 기여 |
|---|---|---|---|---|---|---|
| FLUENCY_WPM | 72 | 5 | 0.9 | 1.0 | 4.5 | 324 |
| SILENCE_RATIO | 85 | 4 | 0.8 | 1.0 | 3.2 | 272 |
| HESITATION_FREQ | 60 | 2 | 1.0 | 0.7 | 1.4 | 84 |
| **합계** |  |  |  |  | **9.1** | **680** |

→ Fluency 역량 점수 = 680 / 9.1 ≈ **74.7점**

#### 15.4.7 Step 5 — Picklass Proficiency Score (PPS)

8대 역량 점수의 **가중 종합**. 역량 가중치는 학습자의 학습 목적별로 차등.

**학습 목적별 역량 가중치 매트릭스**

| 학습 목적 | Speaking | Reading | Listening | Writing | Vocabulary | Grammar | Fluency | Pragmatics |
|---|---|---|---|---|---|---|---|---|
| **일반 (균형)** | 1.5 | 1.5 | 1.5 | 1.0 | 1.0 | 1.0 | 1.5 | 1.0 |
| **회화 중심** | 2.0 | 0.5 | 1.5 | 0.3 | 1.0 | 1.0 | 2.0 | 1.5 |
| **시험 대비** | 1.0 | 2.0 | 2.0 | 1.5 | 1.5 | 2.0 | 0.5 | 0.5 |
| **비즈니스** | 1.5 | 1.5 | 2.0 | 2.0 | 1.5 | 1.5 | 1.0 | 2.0 |
| **여행 영어** | 2.0 | 0.5 | 1.5 | 0.3 | 1.5 | 0.5 | 1.5 | 2.0 |

**PPS 공식**

```
PPS = round(Σ(역량 점수 × 역량 가중치) / Σ(역량 가중치))
```

> 학습 목적은 §15.5 정밀 진단 결과 또는 학습자 자가 설정에서 결정된다. 미설정 시 "일반(균형)" 적용.

#### 15.4.8 절대 레벨 매핑 (PPS → CEFR / 파트너 레벨)

PPS 0~100을 **CEFR 7단계** 및 **파트너 레벨(L1~L18)** 두 체계에 매핑.

| PPS 구간 | CEFR | 파트너 레벨 (예시: L1~L18) | 의미 |
|---|---|---|---|
| 0–9 | **PreA1** | L1 | 학습 시작 단계 |
| 10–24 | **A1** | L2~L3 | 기본 표현 가능 |
| 25–39 | **A2** | L4~L6 | 일상 의사소통 |
| 40–59 | **B1** | L7~L10 | 익숙한 주제 자유 표현 |
| 60–79 | **B2** | L11~L14 | 복잡한 주제 능숙 표현 |
| 80–94 | **C1** | L15~L17 | 학술·전문 영어 활용 |
| 95–100 | **C2** | L18 | 원어민 수준 |

> 매핑 임계값은 학습자 데이터 누적 후 분포 기반 calibration (Phase 2 이후). 초기에는 위 고정 임계값을 사용한다.

#### 15.4.9 측정 충분성 (Sufficiency) 및 진단 신뢰도

PPD 출력은 **측정 데이터 충분성**에 따라 3단계 신뢰도 등급을 가진다.

| 등급 | 조건 | UI 표기 |
|---|---|---|
| **✅ 확정 (Confirmed)** | 8대 역량 모두 N ≥ N_min, 최근 30일 데이터 ≥ 1회 | "PPS: 73 (확정)" |
| **🟡 추정 (Estimated)** | 8대 역량 중 6개 이상 N ≥ N_min | "PPS: 73 ± 5 (추정 — 데이터 일부 부족)" |
| **⚠️ 불충분 (Insufficient)** | 6개 미만 충족 | "측정 데이터 부족 — 추가 학습 후 진단 가능" |

신규 학습자는 §15.2 Level Test 1회 세션 직후 보통 "추정" 등급. 4~6주 학습 누적 후 "확정" 등급 진입.

#### 15.4.10 진단 카드 (Diagnostic Card)

PPD 산출 결과를 청자별 표준 카드 형식으로 노출.

**학습자용 진단 카드 (학생 페이지)**

```
📊 당신의 영어 종합 능력 (Picklass Proficiency Score)

PPS: ████████░░  73 / 100   ✅ 확정
레벨: B2 (Upper-Intermediate) / Pagoda L12

8대 역량 (레이더 차트)
  Speaking      ━━━━━━━━●━━ 78
  Reading       ━━━━━━━●━━━ 70
  Listening     ━━●━━━━━━━━ 62
  Writing       ━━━━━━●━━━━ 68
  Vocabulary    ━━━━━●━━━━━ 65
  Grammar       ━━━●━━━━━━━ 55  ← 약점
  Fluency       ━━━━━━━━●━━ 75  ← 강점
  Pragmatics    ━━━━━━━●━━━ 72

📈 강점 TOP 3: Speaking · Fluency · Pragmatics
⚠️ 약점 TOP 3: Grammar · Listening · Vocabulary

🎯 다음 추천 학습:
  · Expression Drill (EDR, Stage 3 Expand) — 문법 정확도 향상
  · Process Writing (PWR) — 문법·구조 동시 강화
  · Vocabulary Listening & Meaning (VLM) — 듣기 보강
🎯 4주 후 목표: Grammar 65점 (+10) · Listening 70점 (+8)
```

**강사용 진단 카드 (Studio 학생 상세 페이지)**

```
📋 [학생 #45 / 김OO / 4월 진단 스냅샷]

PPS: 73 (확정) | 베이스라인 65 → +8 (4주)
약점 1순위: Grammar 55점 — 자세히
  · 관련 KPI: GRAMMAR_ERROR (74점), TENSE_ACC (40점), PREP_USAGE (51점)
  · 약점 모듈 데이터: SCP 평균 60%, EDR 평균 55%
  · 처방 권장:
    - 다음 레슨 EDR 모듈 1개 추가 (사후 분석 70%↑ 효과 예상)
    - 한국어 grammar hint 활성화 (현재 영문만 사용 중)
    - PWR 4주 후 도전 권장 (현재 prerequisite 미달)
```

**관리자용 진단 카드 (Admin 학생 목록 행)**

```
[김OO]  PPS 73  B2  📈+8  ✅확정  강점Speaking  약점Grammar  [상세]
```

#### 15.4.11 데이터 모델

```sql
-- KPI 정의 카탈로그 (정규화·가중치 사전 정의)
table kpi_definitions (
  kpi_code pk,                  -- 'FLUENCY_WPM' 등 70 코드
  capability enum,              -- 8대 역량 (Speaking/Reading/.../Pragmatics)
  unit varchar(20),             -- %, WPM, ms, count, score, ratio
  importance int,               -- 1~5
  min_measurement_count int,    -- N_min
  benchmark_floor numeric,      -- §14.6.5 PreA1 기준
  benchmark_ceiling numeric,    -- §14.6.5 C2 기준
  normalization_fn varchar(20)  -- 'percent' | 'wpm-linear' | 'ms-inverse' | 'count-lookup' | 'score-x20' | 'ratio-sigmoid'
);

-- 학습자별 진단 스냅샷 (배치 + 실시간 갱신)
table proficiency_snapshots (
  id pk,
  user_id fk,
  computed_at timestamp,
  source_period varchar(20),         -- 'last_30d' | 'last_90d' | 'lifetime'
  goal_profile varchar(20),          -- '일반' | '회화' | '시험' | '비즈니스' | '여행'
  capability_scores jsonb,           -- {speaking:78, reading:70, ...} 0~100
  pps int,                           -- 0~100
  cefr enum,                         -- PreA1 ~ C2
  partner_level int,                 -- L1 ~ L18
  confidence_status enum('confirmed','estimated','insufficient'),
  measurement_count_summary jsonb,   -- 역량별 측정 N
  computed_runtime_ms int            -- 산출 시간 (성능 모니터링)
);

-- 산출 감사 로그 (재현·디버그용)
table proficiency_calc_log (
  id pk,
  snapshot_id fk,
  kpi_code,
  raw_value numeric,
  normalized_score numeric,
  importance numeric,
  confidence numeric,
  recency_weight numeric,
  final_weight numeric,
  contribution_to_capability numeric,
  capability enum
);
```

**산출 주기**
- **실시간**: 모듈 세션 종료 직후 해당 학습자의 PPD 재산출 (~ 200ms 목표)
- **일별 배치**: 매일 00:30 KST 전체 활성 학습자 PPD 재산출 (시간 감쇠 갱신)
- **월말 결산**: 매월 1일 03:00 KST 월간 리포트용 스냅샷 생성

#### 15.4.12 사용처 매트릭스

PPD 엔진은 다음 8개 영역에서 재사용된다. **단일 진단 엔진 — 다중 사용처** 패턴.

| 사용처 | PPD 활용 방식 | 출력 |
|---|---|---|
| §15.2 Level Test | 첫 1회 학습 후 PPD 산출 | CEFR + 파트너 레벨 + 진단 카드 (확정/추정 표기) |
| §15.3 Achievement Test | 학습 N개월 후 PPD 재산출, baseline과 Delta 비교 | ΔPPS + ΔCEFR + 영역별 변화량 |
| §11.10 수료 기준 | PPS·역량 점수 임계값 충족 시 자동 수료 | 수료 진척도 게이지 |
| §11.11 게이미피케이션 | PPS 향상량 → 캐릭터 성장·랭킹 | 캐릭터 표정·기업 랭킹 순위 |
| §11.12 푸시·알림 | PPS 정체 N주, 약점 KPI 임계 등 트리거 | 자동 발송 알림 |
| **개인 리포트** (예정) | 진단 카드 메인 위젯 | 주간/월간 PDF |
| **그룹 리포트** (예정) | 학생별 PPS 매트릭스 + 분포 | 반·기관·기업 종합 |
| §10 시퀀싱 엔진 | 약점 역량 → 다음 모듈 추천 가중치 | LessonPlan 우선순위 |

#### 15.4.13 운영 KPI (PPD 자체 성능 지표)

| KPI | 목표 |
|---|---|
| 진단 산출 시간 (실시간) | ≤ 200 ms |
| Level Test ↔ 강사 평가 일치율 | ≥ 80% |
| Achievement Test 향상 검증 신뢰도 | 학습 3개월 후 평균 ΔPPS ≥ +5 (학습 효과 검증) |
| "확정" 등급 진입 평균 기간 | ≤ 6주 학습 후 |
| 측정 부족 케이스 비율 | ≤ 10% (런칭 6개월 후) |

#### 15.4.14 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 정규화 공식의 부정확성 | 레벨 오판정 | Phase 2 이후 실제 학습자 분포 기반 calibration, 분기 1회 검증 |
| KPI 가중치 공정성 | 특정 모듈 학습자 편향 | 분기 1회 가중치 리뷰 위원회, 8대 역량 균형 점검 |
| 측정 데이터 부족 | 신규 학습자 진단 불가 | "추정 + 신뢰구간" 표기로 점진 정확도 향상, Level Test 안내 강화 |
| 시간 감쇠 과도 | 우수 학습자가 휴면 후 등급 하향 | Achievement Test 주기 안내, 휴면 30일 후 자동 알림 |
| 점수 게이밍 (PPS만 올리려는 행위) | 학습 본질 왜곡 | 다축 측정·일정 시간 학습 요건 부과, 8대 역량 균형 보너스 |

> 📎 본 PPD 엔진의 전체 KPI 코드 정의·정규화 함수·중요도는 컴패니언 문서 **`Picklass_KPI_체계_상세.md`** §5(신설 예정)에 표 형태로 수록한다.

### 15.5 정밀 진단 (Advanced Diagnostics) *(고도화)*

> 📋 **기획 단계 (고도화)** — P4 원칙에 따라 단순 CEFR/파고다 레벨을 넘어 **MBTI형 성향·학습 환경 반영**.

#### 15.5.1 추가 진단 차원

| 차원 | 측정 항목 |
|---|---|
| **학습 성향 (MBTI 형식)** | 청각형 vs 시각형, 분석형 vs 직관형, 도전형 vs 신중형 등 |
| **학습 환경** | 학습 가능 시간대, 학습 디바이스(모바일/PC), 1회 평균 학습시간 |
| **약점 영역** | 4기능 중 가장 취약한 영역, 세부 KPI 약점 (예: 발음·억양·문법) |
| **목표** | 시험 대비, 비즈니스 영어, 일상 회화, 여행 영어 등 |

#### 15.5.2 결과 활용

- **개인화 학습 로드맵 자동 생성**: AI가 진단 결과를 기반으로 추천 모듈 시퀀스 + 추천 콘텐츠 생성 모드(§10.15) + 추천 수강 형태(§11.9) 제안
- **튜터 매칭** (1:1 회화 채널): 학습자 성향에 맞는 AI 튜터 페르소나 추천
- **콘텐츠 톤 조정**: 분석형은 문법 설명 더 많이, 직관형은 빠른 대화 더 많이

#### 15.5.3 루브릭 (Rubric) — 향후 정의

| 영역 | 루브릭 차원 | 점수 척도 |
|---|---|---|
| Speaking | 유창성 · 정확성 · 발음 · 어휘력 · 상호작용 | 1~5점 |
| Listening | 핵심 파악 · 세부 이해 · 추론 | 1~5점 |
| Reading | 읽기 속도 · 어휘력 · 추론 · 비판적 읽기 | 1~5점 |
| Writing | 문장 구조 · 문법 · 어휘 · 응집성 | 1~5점 |

→ Phase 2 이후 상세 루브릭 정의 (§20 로드맵 연동).

### 15.6 전사 일원화 로드맵 *(고도화)*

회의록 "추후 고도화 6)" — B2B/B2C/전화/인강/출강 등 전사의 레벨테스트 체계를 일원화한다.

#### 15.6.1 현재 상태 (Baseline)

| 채널 | 현재 레벨 진단 방식 | 결과 호환성 |
|---|---|---|
| Picklass B2B | 레벨 확인 수업 1회 (P1) | CEFR + 파고다 레벨 |
| Picklass B2C | 동일 | CEFR + 파고다 레벨 |
| 1:1 회화 (전화/화상) | 자체 강사 평가 | 파고다 자체 |
| 인강 (이러닝) | 사전 객관식 테스트 | 자체 |
| 출강 | 강사 인터뷰 | 자체 |

#### 15.6.2 통합 비전

```
[모든 채널의 레벨 진단 데이터]
    ↓
[픽클래스 통합 레벨 진단 엔진 (Picklass Unified Level System, PULS)]
    ↓
[CEFR + 파고다 레벨 + 학습 성향 + 약점 영역]
    ↓
[모든 채널에서 동일 레벨 적용]
   - 학습자가 채널을 옮겨도 동일 레벨로 시작
   - 한 채널의 학습 성과가 다른 채널에 반영
```

#### 15.6.3 단계별 추진 (예정)

| Phase | 시기 | 목표 |
|---|---|---|
| **Phase A** | 2026 Q3 | Picklass B2B/B2C 채널 일원화 (CEFR + 파고다 레벨 표준화) |
| **Phase B** | 2026 Q4 | 1:1 회화(전화/화상) 채널과 양방향 데이터 연동 (§20 블렌디드 학습) |
| **Phase C** | 2027 Q1 | 인강·출강 채널까지 통합 |

### 15.7 데이터 모델 (요약)

```sql
-- 레벨 진단 결과 (Level Test + Achievement Test 공통)
table level_assessments (
  id pk,
  user_id fk,
  type enum('LEVEL_TEST','ACHIEVEMENT_TEST','MANUAL'),
  source_session_id fk,        -- 측정에 사용된 SpeakingSession 또는 LessonResult
  cefr enum('PreA1','A1','A2','B1','B2','C1','C2'),
  pagoda_level int,             -- L1 ~ L18
  scores jsonb,                 -- 영역별 점수 {speaking: 65, listening: 72, ...}
  rubric jsonb null,            -- 정밀 루브릭 (Phase 2+)
  diagnostic jsonb null,        -- MBTI형 성향, 약점, 목표 등 (Phase 2+)
  baseline_id fk null,          -- ACHIEVEMENT_TEST일 때 비교 베이스라인 LEVEL_TEST id
  delta jsonb null,             -- baseline 대비 변화량
  created_at timestamp
);

-- 채널별 통합 매핑 (Phase A+)
table cross_channel_levels (
  user_id pk,
  channel enum('picklass_b2b','picklass_b2c','phone','elearning','outclass'),
  cefr enum('PreA1','A1','A2','B1','B2','C1','C2'),
  pagoda_level int,
  last_assessed_at timestamp,
  source_assessment_id fk
);
```

### 15.8 운영 KPI

| KPI | 목표 |
|---|---|
| Level Test 1회 세션 완료율 | ≥ 95% (학습자가 진단을 완료하는 비율) |
| Level Test ↔ 강사 평가 일치율 | ≥ 80% (강사가 학습자를 직접 평가했을 때의 일치도) |
| Achievement Test 향상 검증 | 학습 3개월 후 평균 ΔCEFR ≥ 0.5단계 |
| 통합 레벨 매핑 정확도 (Phase B+) | 채널 간 매핑 오류율 ≤ 5% |

### 15.9 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 1회 세션의 측정 신뢰도 부족 | Level Test 정확도 ↓ | KPI 다축 측정 + LLM 하이브리드 보정 (§15.2.3) |
| 채널 간 레벨 정의 충돌 | 일원화 실패 | CEFR을 표준으로 고정, 채널별 매핑은 변환 함수로 정의 |
| 정밀 루브릭 과부하 | 운영 복잡도 ↑ | Phase 2 이후 점진 도입, Phase 1에는 CEFR만 사용 |
| 학습자 거부감 | 진단 회피 | "시험"이 아닌 "확인 수업" UX (P1) |

> 📎 본 장과 연관된 KPI 코드 정의는 §14.6, 학습자 페이지 노출 형식은 §11.10.4, 운영 모드 분류(정규/자유)는 §11.9 참조.

---

## 16. B2B 제휴 운영 모델 (Partner Operating Model) *(v0.19 신설 — 구 §11.13에서 챕터 승격)*

> 📋 **기획 단계** — B2B 제휴 중심 사업 진행에 따른 **제휴사 ↔ 픽클래스 책임 분리 운영 모델**. §3.3.4 1:1 회화 채널·§14.10.5 외부 verify의 일반화·확장 모델로 작동한다. v0.17에서 **성과 데이터 운영 지원 가치를 핵심 정책으로 격상**.

### 16.0 핵심 가치 — 성과 데이터 운영 지원 *(v0.17 격상)*

> 💡 **B2B 제휴의 본질**: picklass는 단순히 학습 모듈을 제공하는 것이 아니라, **수업 진행을 통해 산출한 성과 데이터를 파트너사에 전달하여 고객사의 운영을 적극 지원**한다. 이는 단방향 콜백을 넘어선 **운영 지원 파트너십**이다.

**3대 가치**

| 가치 | 설명 |
|---|---|
| **① 학습 엔진 임베드** | picklass의 검증된 모듈·KPI 측정·PPD 진단 엔진을 파트너사가 즉시 활용 |
| **② 성과 데이터 운영 지원** ✦ | 5축 KPI·PPD 진단·수료 진척도 데이터를 파트너사에 표준 인터페이스로 전달 → 파트너사가 자사 학습자에게 더 나은 운영(수료 판정·맞춤 케어·CS·갱신 영업)을 할 수 있도록 지원 |
| **③ 운영 책임 분리** | 상품·과금·CS는 파트너사 자율, 학습 엔진은 picklass 책임 |

**성과 데이터 운영 지원의 구체 가치**

| 파트너사 활용 | picklass가 제공하는 데이터 |
|---|---|
| 수료 판정 자동화 | 5축 KPI 누적·PPD·진척도 (실시간/배치) |
| 학습자 맞춤 케어 | 약점 영역·정체 학습자 식별·미사용 표현 |
| CS 응대 | 학습자별 상세 학습 로그·5축 추이 |
| 갱신 영업 | 학습 효과 향상도·CEFR 변화·만족도 지표 |
| 코호트 분석 | 기관 평균 vs 산업 평균 벤치마크 (옵션) |
| HR 보고 | 임직원 학습 성과 표준 리포트 |

### 16.1 운영 책임 분리 원칙

B2B 제휴 사업에서는 **제휴사가 자체 비즈니스 로직을 기획·개발**하고, **픽클래스는 수업 모듈 임베드 실행 + 성과 데이터 운영 지원**을 담당하는 명확한 책임 분리를 원칙으로 한다. 이는 (a) 제휴사의 사업 자율성 보장 (b) 픽클래스의 학습 엔진 집중 (c) 양사 인터페이스 단순화 (d) **파트너사 운영 효율 극대화**를 위함이다.

**1줄 요약**

> 운영(상품 구성·과금·수료 정책·마케팅) = **제휴사 자체 기획·개발** / 학습 + 성과 데이터 = **픽클래스 임베드 실행 + 운영 지원**

### 16.2 책임 분리 매트릭스

| 영역 | 제휴사 책임 | 픽클래스 책임 |
|---|---|---|
| **상품 구성** | 상품 라인업 정의, 가격, 할인 정책, 구독·일회성 상품 | (없음 — 제휴사 정책 그대로 수용) |
| **과금·결제** | 결제 시스템, 환불 정책, 영수증, 세금계산서 | 학습 모듈 사용량 정산 데이터만 제공 |
| **수료 기준** | 수료 임계값 설정 (시간·발화량·미션·출석률), 수료증 발급 | KPI 측정값 실시간 전달 (§11.10) |
| **마케팅·CRM** | 광고, 프로모션, 가입 유치, CS, 알림톡 | (없음) |
| **사용자 가입** | 회원가입, 인증, 결제 플로우 | (없음 — 제휴사 회원이 외부 토큰으로 진입) |
| **학습 화면** | (없음) | 모듈 UI/UX, Pickle Agent, 5축 피드백, Pick-Speak Method |
| **콘텐츠 생성** | 제휴사 보유 교재 제공 (§8.3.5 업로드) | AI 콘텐츠 생성 엔진 (§9), 모듈 시퀀싱 (§10) |
| **수업 진행** | (없음) | 학생 학습 세션 운영 (§14 Speaking, §11 Tutoring) |
| **KPI 측정·분석** | (없음 — 데이터 수신만) | §14.6 70 KPI · §15.4 PPD 산출 |
| **학습자 페이지 노출** | 제휴사 사이트 내 진척도 위젯 운영 | 진척도 데이터 제공 (실시간 또는 배치) |
| **학습 데이터 보존** | 제휴사 자체 백업·열람 정책 | 픽클래스 측 학습 로그 보존 |
| **법적 책임** | 사용자 계약·이용약관·개인정보처리방침 | 학습 엔진 운영 안정성·SLA |

### 16.3 인터페이스 (제휴사 ↔ 픽클래스)

```
[제휴사 사이트] (자체 회원 가입·결제 완료)
       ↓
[액세스 코드 발급 API] ←─ 픽클래스 (제휴사가 호출)
       ↓
[학습자가 학습 진입 시점에 액세스 토큰 생성]
       ↓
[GET /external/verify] (X-Access-Token + X-Module-Code) ─→ 픽클래스
       ↓
[픽클래스 학습 모듈 임베드 실행]
       ↓
[세션 종료 시 KPI·진척도 콜백] ─→ 제휴사
       ↓
[제휴사 사이트] (수료 판정·진척도 노출)
```

**API 인터페이스 요약**

| 방향 | API | 설명 |
|---|---|---|
| 제휴사 → 픽클래스 | `POST /external/access-codes` | 학생용 액세스 코드 일괄 발급 (§7.5) |
| 제휴사 → 픽클래스 | `GET /external/verify` | 토큰 검증 + 모듈 진입 (§14.10.5) |
| 픽클래스 → 제휴사 | `POST <partner>/callbacks/session-completed` | 세션 종료 시 KPI·진척도 콜백 |
| 제휴사 → 픽클래스 | `GET /external/students/{id}/progress` | 학생 진척도 조회 (수료 기준 판정용) |
| 제휴사 → 픽클래스 | `POST /external/courses/upload` | 교재 업로드 (§8.3.5) |

### 16.4 운영 데이터 흐름 (수료 판정 예시)

```
[픽클래스] 매일 00:30 KST 배치
   - 학생별 5축 KPI 누적 → §15.4 PPD 산출
   - 학생별 4지표(누적 시간·발화량·미션·출석) 산출 (§11.10)
       ↓
   - 제휴사 callback URL로 진척도 데이터 전송
       ↓
[제휴사] 자체 시스템에서 수료 기준 판정
   - 제휴사가 정의한 임계값과 비교
   - 수료 시 자체 수료증 발급
   - 학생 페이지에 진척도 표시
```

### 16.5 진입 채널별 B2B 제휴 모드 적용

| 진입 채널 | 제휴 모드 | 비고 |
|---|---|---|
| 1:1 회화 (외부 임베드, §3.3.4) | ✅ 적용 (사례) | Speaking 모듈 임베드, 제휴사가 상품·과금 운영 |
| Tutoring 통합 임베드 (§3.3.1) | ⚠️ 일부 적용 | 제휴사가 Tutoring 화면을 자사 도메인에서 임베드하는 경우만 |
| Speaking 독립 앱 (§3.3.2) | ❌ 미적용 | 픽클래스 자체 운영 (B2C·일반 B2B 직판) |
| 외부 임베드 — 학습 모듈만 | ✅ 적용 | 가장 간결한 형태 — 제휴사 사이트 내 위젯 임베드 |

### 16.6 회원 계정 연동 (Account Linking) *(v0.18 신설)*

파트너사 회원 ↔ picklass 학습자를 어떻게 매핑할 것인가의 정책. **§4.4.1 B2B 제휴 데이터 처리 정책**(파트너 회원의 picklass 측 최소 저장 원칙)을 전제로 3종 매핑 모드를 제공한다.

##### 16.6.1 매핑 모드 3종

| 모드 | 시점 | 매핑 방식 | 사용 상황 |
|---|---|---|---|
| **A. 1:1 사전 매핑 (Pre-Provisioning)** | 파트너사가 회원 가입 시점 | API로 사전 매핑 생성 — 파트너 회원 ID → picklass user 1:1 매칭 | 파트너 회원 = picklass 학습자가 100% 일치 (대형 제휴) |
| **B. 액세스코드 매핑 (Access Code)** | 학습자가 picklass 진입 시점 | 파트너가 발급한 코드를 학습자가 picklass에서 입력 → 매핑 | 파트너가 learner를 단계적으로 합류시키는 경우 (가맹·기업) |
| **C. SSO 즉시 매핑 (Just-in-Time)** | 매 학습 진입 시점 | 외부 토큰(§16.7)에 사용자 식별자 포함 → picklass가 자동 user 생성·매칭 | **권장 — 1:1 회화·외부 임베드 표준** (§3.3.4, §14.10.5) |

##### 16.6.2 매핑 흐름 (모드 C SSO 즉시 매핑 — 권장)

```
[파트너 회원 학습 클릭]
   ↓
[파트너 서버: 외부 토큰 발급 (§16.7)]
  payload: { external_user_id, name, level, course_id, partner_id }
   ↓
[picklass GET /external/verify (X-Access-Token + X-Module-Code)]
   ↓
[picklass: partner_user_mappings 조회]
  ├─ 매핑 존재 → 기존 picklass user 사용
  └─ 매핑 없음 → 신규 picklass user 자동 생성 + 매핑 등록 (Just-in-Time)
   ↓
[picklass: §4.4.1 정책에 따라 최소 데이터만 저장]
   ↓
[학습 모듈 진입]
```

##### 16.6.3 데이터 모델

```sql
table partner_user_mappings (
  id pk,
  partner_id fk,                       -- partner_organizations.id
  external_user_id varchar(255),       -- 파트너 측 회원 ID (해시 저장 권장)
  picklass_user_id fk,                 -- users.id
  mapping_method enum('pre_provision','access_code','sso_jit'),
  display_name varchar(100) null,      -- 학습 화면 표시명 (수정 가능, §3.3.1 이름 정책)
  level_at_link enum null,             -- 매핑 시점 학습자 레벨
  external_metadata jsonb null,        -- 파트너가 푸시한 추가 메타 (직무·부서 등, 학습 설계용)
  created_at timestamp,
  last_synced_at timestamp,
  unique (partner_id, external_user_id)
);
```

##### 16.6.4 동기화 정책

| 트리거 | 동작 |
|---|---|
| 외부 토큰 verify 시점 (§16.7) | `last_synced_at` 갱신, 메타데이터 비교 후 변경분만 갱신 |
| 파트너사 명시적 푸시 (`POST /external/users/sync`) | 즉시 매핑 갱신 |
| 학습자 자기 정보 수정 (display_name) | picklass 측만 갱신, 파트너로 역방향 푸시 X |
| 회원 탈퇴·계약 종료 | §4.4.1에 따라 30일 후 자동 익명화·삭제 |

##### 16.6.5 명칭·식별자 정책

| 항목 | 정책 |
|---|---|
| picklass 내부 user_id | UUID v4 (외부 비공개) |
| external_user_id | 파트너가 정의 (일반적으로 파트너 측 사용자 PK) — picklass는 해시·암호화 후 저장 권장 |
| 표시명 (display_name) | 파트너에서 푸시 가능 + 학습자가 학습 진입 시 수정 가능 (§3.3.1 GET_AUTH 이름 필드 정책 일관) |
| 이메일 | 파트너 인증을 신뢰하므로 picklass 측 이메일 인증 미강제 (§4.4.2) |

### 16.7 서비스 인증 연동 (Service Authentication) *(v0.18 신설)*

파트너사 ↔ picklass 인증을 어떻게 처리할 것인가. picklass는 4종 인증 방식을 지원하며, 사용처별로 권장 방식이 다르다.

##### 16.7.1 인증 방식 4종

| 방식 | 사용처 | 보안 수준 | 토큰 수명 |
|---|---|---|---|
| **ONETIME TOKEN** *(§3.3.1·§14.10.5)* | 학습자 단일 진입 (1회 사용 후 소멸) | 🟢 매우 높음 | 5분 (TTL) |
| **OAuth 2.0 / OIDC** | 표준 SSO — 파트너가 OIDC IdP 운영하는 경우 | 🟢 매우 높음 | 액세스 1시간 / 리프레시 30일 |
| **API Key** | 서버-서버 (파트너 서버 → picklass) — 액세스코드 발급·진척도 조회 등 | 🟡 보통 (HMAC 서명 필수) | 영속 (회전 권장) |
| **JWT (자체 발급)** | 파트너가 자체 JWT 발급 시 picklass가 공개키로 검증 | 🟢 높음 | 파트너 정의 (≤ 1시간 권장) |

##### 16.7.2 사용처 매트릭스

| 시나리오 | 권장 인증 | 비고 |
|---|---|---|
| 학습자가 학습 모듈 진입 (1회) | **ONETIME TOKEN** | 가장 보편적, 모든 채널 표준 |
| 학습자가 학습 모듈 진입 (반복, SSO 표준) | OAuth/OIDC | 파트너가 IdP 운영 시 |
| 학습자가 학습 모듈 진입 (자체 JWT) | JWT | 파트너 인프라가 IdP 미운영, 자체 토큰 발급 시 |
| 파트너 → picklass 액세스코드 발급 | API Key | 서버-서버 |
| 파트너 → picklass 학습자 진척도 조회 | API Key | 서버-서버 |
| picklass → 파트너 콜백 (세션 완료·진척도) | HMAC 서명 (파트너 측 verify) | 본질은 파트너가 picklass 발신을 검증 |

##### 16.7.3 ONETIME TOKEN 라이프사이클 (표준)

```
[1] 파트너 서버가 ONETIME TOKEN 발급 요청 (서버 내부)
       payload: { external_user_id, course_id, module_code, level, duration }
       (파트너의 인증 — API Key + HMAC)
       ↓
[2] picklass: TOKEN 생성 (UUID, TTL 5분), partner_token_log 기록
       ↓
[3] 파트너가 학습자 브라우저에 TOKEN을 쿼리스트링·페이로드로 전달
       ↓
[4] 학습자 브라우저 → picklass: GET /external/verify (X-Access-Token: TOKEN, X-Module-Code: ...)
       ↓
[5] picklass: TOKEN 검증 + §16.6 매핑 + 모듈 진입
       ↓
[6] picklass: TOKEN 즉시 소멸 (1회용)
```

##### 16.7.4 보안 정책

| 정책 | 내용 |
|---|---|
| **API Key 회전** | 분기 1회 자동 회전 권장 (관리자 알림), 강제 회전은 보안 사고 시 |
| **HMAC 서명** | 모든 서버-서버 호출은 HMAC-SHA256 서명 필수 (`X-Signature` 헤더) |
| **TLS 1.2+** | 모든 통신은 TLS 1.2 이상 필수 |
| **IP 화이트리스트** | 옵션 — 파트너별 picklass 인입 IP 제한 |
| **재사용 방지** | ONETIME TOKEN은 1회 사용 후 즉시 무효화, 동일 토큰 재사용 시도는 보안 로그 |
| **속도 제한 (Rate Limit)** | 파트너별 API 호출 한도 (분당 1,000건 기본, 협의 조정) |
| **감사 로그** | 모든 인증 이벤트 90일 보존 (§22.4 감사 로그) |

##### 16.7.5 데이터 모델 (요약)

```sql
table partner_auth_methods (
  id pk,
  partner_id fk,
  method enum('onetime_token','oauth_oidc','api_key','jwt'),
  status enum('active','rotating','revoked'),
  -- OAuth/OIDC
  oidc_issuer varchar(255) null,
  oidc_client_id varchar(255) null,
  -- API Key (HMAC)
  api_key_hash varchar(255) null,
  hmac_secret_hash varchar(255) null,
  -- JWT
  jwt_public_key text null,             -- 파트너 공개키 (PEM)
  jwt_issuer varchar(255) null,
  -- 공통
  ip_whitelist jsonb null,
  rate_limit_per_minute int default 1000,
  created_at timestamp,
  rotated_at timestamp null
);

table partner_token_log (
  id pk,
  partner_id fk,
  token_hash varchar(255),
  external_user_id varchar(255),
  issued_at timestamp,
  consumed_at timestamp null,
  ip_address varchar(45),
  user_agent text null,
  result enum('issued','consumed','expired','invalid','rate_limited')
);
```

### 16.8 상품 구성 ↔ 학습 운영 매핑 *(v0.18 신설)*

파트너사 자체 상품 라인업과 picklass 학습 운영 정책을 매핑하는 표준 모델. 파트너사의 다양한 상품 구성에 따라 **picklass 측 운영(과정 모드·수강 형태·시퀀싱 페르소나·수료 기준)이 자동 결정**된다.

##### 16.8.1 매핑 축 4종

| 매핑 축 | picklass 운영 정책 | 참조 |
|---|---|---|
| **모듈 풀 (Module Scope)** | 스피킹 only / 튜터링 only / 통합 | §8.3.6 과정 생성 3종 모드 |
| **수강 형태** | 정규수강 (요일 지정) / 자유수강 (분 한도) | §11.9 |
| **시퀀싱 페르소나** | 시험 대비 / 비즈니스 / 여행 / 일상 / 면접 / 레벨 진단 | §14.4.4 |
| **수료 기준** | 4지표 임계값 (월 누적 시간·발화량·미션·출석) | §11.10 |

##### 16.8.2 표준 매핑 사례 4종

| 파트너 상품 라인업 | 모듈 풀 | 수강 형태 | 시퀀싱 페르소나 | 수료 기준 |
|---|---|---|---|---|
| **A. 정규 1:1 회화 코스** (정규반 60일) | 스피킹 only (Pick-Speak 6모듈) | 정규수강 (주 3회) | 일상 회화 | 월 80분+발화 100문장+미션 70%+출석 80% |
| **B. AI 자유 회화** (자유 이용권) | 스피킹 only | 자유수강 (월 600분) | 일상 회화 | 월 누적 시간+발화량 (출석 미적용) |
| **C. 비즈니스 영어 정규** | 통합 (28모듈) | 정규수강 (주 5회) | 비즈니스 영어 | 월 100분+발화 120문장+미션 80%+출석 90% |
| **D. 시험 대비 (OPIc·토익S)** | 통합, OMP 시그니처 | 자유수강 (시험 직전 집중) | 시험 대비 | 월 누적 시간+OMP 점수 향상도+미션 80% |

##### 16.8.3 데이터 모델

```sql
table partner_product_mappings (
  id pk,
  partner_id fk,
  external_product_id varchar(255),     -- 파트너 측 상품 ID
  external_product_name varchar(255),
  -- 운영 매핑 4축
  module_scope enum('speaking_only','tutoring_only','unified'),
  course_mode enum('regular','self_paced'),
  sequencing_persona enum('exam','business','travel','daily','interview','level_test'),
  completion_policy_id fk null,         -- 별도 수료 정책 테이블 참조
  -- 추가 옵션
  default_level_band enum null,
  default_session_duration_min int null,
  enable_signature_module enum('OMP','FRT',null),
  -- 메타
  created_at timestamp,
  effective_from date,
  effective_to date null,
  unique (partner_id, external_product_id)
);
```

##### 16.8.4 매핑 적용 흐름

```
[파트너 회원이 상품 구매]
   ↓
[파트너 서버: 외부 토큰 발급 시 external_product_id 포함]
       payload: { external_user_id, external_product_id, course_id, ... }
   ↓
[picklass GET /external/verify]
   ↓
[picklass: partner_product_mappings 조회 → 매핑 4축 자동 적용]
   ↓
[Course Hub 자동 진입 또는 학습 모듈 직접 진입 (§8.3 / §14.4)]
```

##### 16.8.5 운영 정책

| 정책 | 내용 |
|---|---|
| 매핑 등록 | 파트너 온보딩 시 `POST /external/products/upsert`로 일괄 등록 |
| 매핑 변경 | `effective_from`·`effective_to`로 버전 관리, 기존 학습 세션은 etag 고정 |
| 누락 매핑 | 매핑 없는 product_id 진입 시 기본값(통합/자유수강/일상/표준 수료) 적용 + 알림 |
| 검증 | 매핑 등록 시 module_scope ↔ sequencing_persona 호환성 자동 검증 |

### 16.9 파트너사 운영 도구 지원 *(v0.18 신설)*

파트너사가 자체 운영(고객 케어·CS·갱신 영업·HR 보고)을 효과적으로 수행할 수 있도록 picklass가 제공하는 5종 운영 지원 도구.

##### 16.9.1 운영 도구 5종

| 도구 | 형태 | 사용자 | 주요 기능 |
|---|---|---|---|
| **A. 운영 콘솔 (Partner Operation Console)** | 웹 대시보드 (`partner-console.picklass.com`) | 파트너 운영자·CS | 학습자 검색, 진척도 조회, 수료 진단, 약점 케어 추천 |
| **B. 조회 API** | REST API | 파트너 백엔드 | 학습자 진척도·5축 KPI·PPD·세션 이력 조회 |
| **C. 데이터 다운로드** | CSV/Excel/PDF 정기 리포트 | 파트너 운영자 | 월간·주간 코호트 리포트, 학습자별 수료 현황 |
| **D. Webhook (실시간 이벤트)** | HTTP POST (서명 포함) | 파트너 백엔드 | 세션 완료·수료 도달·이상 학습자 감지 등 실시간 알림 |
| **E. 분석 대시보드 (BI 연동)** | Tableau·Power BI·Looker 등 | 파트너 데이터 분석가 | 데이터 마트 직접 연결 (Snowflake / BigQuery 옵션) |

##### 16.9.2 운영 콘솔 (도구 A) 핵심 기능

| 화면 | 기능 |
|---|---|
| **학습자 목록** | 파트너 소속 전체 학습자 검색·필터(레벨·상품·진척도·이상 학습자) |
| **학습자 상세** | 5축 KPI 추이, PPD 진단 카드(§15.4), 모듈별 학습 이력, 수료 진척도 게이지 |
| **이상 학습자 알림** | 7일 미접속, 정체, 약점 누적 등 자동 식별 → 케어 추천 |
| **CS 응대 지원** | 학습자 클릭 → 즉시 학습 로그·세션 기록 노출 |
| **수료 모니터링** | 임박/완료/미달 학습자 분리 노출 |
| **리포트 생성** | 학습자별 PDF 리포트 일괄 발송 (§15.4 진단 카드) |

##### 16.9.3 조회 API (도구 B) 엔드포인트

| 엔드포인트 | 설명 |
|---|---|
| `GET /external/students` | 파트너 소속 학습자 목록 (페이징) |
| `GET /external/students/{ext_id}/progress` | 학습자 진척도 (4지표 + PPD) |
| `GET /external/students/{ext_id}/sessions` | 학습 세션 이력 |
| `GET /external/students/{ext_id}/kpi-trends?period=30d` | 5축 KPI 추이 |
| `GET /external/cohorts/{cohort_id}/summary` | 코호트(상품·기간) 요약 통계 |
| `POST /external/reports/generate` | 비동기 리포트 생성 요청 (PDF/CSV) |

##### 16.9.4 Webhook (도구 D) 이벤트 타입

| 이벤트 | 발화 시점 |
|---|---|
| `session.completed` | 학습 세션 종료 직후 |
| `progress.threshold` | 4지표 중 어느 하나의 임계값 도달 |
| `completion.achieved` | 수료 기준 100% 충족 |
| `learner.dropout_risk` | 7일 미접속 또는 약점 누적 감지 |
| `learner.level_up` | CEFR 레벨 상승 |
| `daily.summary` | 매일 00:30 KST 일별 요약 |

Webhook 페이로드는 HMAC-SHA256 서명 포함, 5xx 응답 시 자동 재시도 (지수 백오프, 최대 7일 보존).

##### 16.9.5 데이터 다운로드·분석 (도구 C·E)

**도구 C — 정기 리포트**
- 매월 1일 03:00 KST 자동 생성
- 형식: CSV (raw 데이터) + Excel (서식·차트) + PDF (요약 보고서)
- 보존: 12개월

**도구 E — BI 연동 (옵션, 엔터프라이즈)**
- Snowflake / BigQuery / Redshift 데이터 마트 직접 연결
- 일별 ETL 동기화 (학습 로그 → 데이터 마트)
- 파트너 자체 BI 도구(Tableau·Power BI·Looker)로 시각화

##### 16.9.6 도구 사용 권한

| 도구 | 파트너 측 권한 |
|---|---|
| 운영 콘솔 | 파트너 운영 관리자 (역할별 접근 제한 — 전체 / CS만 / 보고서만) |
| 조회 API | 서버-서버 (API Key + HMAC) |
| 데이터 다운로드 | 운영 콘솔 권한자 + 정기 발송 이메일 수신자 |
| Webhook | 파트너 콜백 URL 단일 (또는 이벤트 타입별 분리 가능) |
| BI 연동 | 별도 계약 (엔터프라이즈) |

### 16.10 채널별 적용 상세 *(v0.19 신설)*

§16.0~16.9에서 정의한 공통 운영 모델을 **3개 채널별로 어떻게 적용하는가**를 다룬다. 각 채널은 학습 콘텐츠·KPI·임베드 방식·SLA가 다르므로 채널별 보강이 필요하다.

| 채널 | 절 | 핵심 차이 |
|---|---|---|
| Tutoring | §16.10.1 | 모듈 시퀀싱(§10) + 학생용 화면 임베드 + 4기능 통합 |
| Speaking | §16.10.2 | Pick-Speak Method 5단계 + 5축 KPI + 6 모듈 + 음성 인프라 SLA |
| 외부 임베드 (1:1 회화) | §16.10.3 | §3.3.4 1:1 회화 채널의 cross-ref + 운영 표준화 |

#### 16.10.1 Tutoring 채널 적용

Tutoring(§11) 학생용 학습 모듈을 외부 파트너사 사이트에 임베드하는 케이스. 4기능(V/R/W + Speaking 임베드) 통합 학습이 가능하다.

**임베드 방식**

| 항목 | 정책 |
|---|---|
| 임베드 URL | `tutoring.picklass.com/modules/[lessonId]?token=...` |
| 인증 | §16.7 ONETIME TOKEN 또는 OAuth/JWT |
| 운영 정책 | §16.8 상품-운영 매핑 — 보통 `module_scope: tutoring_only` 또는 `unified` |
| 시퀀싱 | §10 모듈 시퀀싱 엔진이 `analyzer` 서버에서 호출됨 |

**Tutoring 채널 콜백 페이로드 (예시)**

```json
{
  "event": "session.completed",
  "channel": "tutoring",
  "external_user_id": "partner_user_12345",
  "lesson_id": "lsn_abc",
  "modules_completed": ["PRD", "SCN", "QAR", "SUM"],
  "module_results": [
    { "code": "PRD", "score": 85, "duration_sec": 180, "kpi": { "PREDICT_COMP": 0.85 } },
    { "code": "SCN", "score": 92, "duration_sec": 120, "kpi": { "SILENT_READING": 134 } }
    /* ... */
  ],
  "lesson_total_score": 86,
  "completion_progress": {
    "monthly_minutes": 42,
    "mission_complete_rate": 0.80,
    "attendance_rate": 0.85
  },
  "ppd_snapshot": { /* §15.4 PPD */ },
  "completed_at": "2026-05-08T10:30:00Z"
}
```

**4기능 통합 학습 시 모듈 풀**

| 영역 | 활용 가능 모듈 |
|---|---|
| Vocabulary | WRD, GMN, WWB, IMG, WDN, WFT (WSD 제외 — 발음 평가는 Speaking 측에서) |
| Reading | PRD, SCN, SKM, CLR, SUM, QAR, RRD, ORL |
| Writing | UNS, CPW, SCP, SWR, PPR, PWR |
| Speaking 임베드 | LRN, VLM, EDR, RPL, FRT, OMP (선택적 — `unified` 모드) |

**Tutoring 특수 정책**

| 정책 | 내용 |
|---|---|
| 나만의 수업(§11.8) 사용 | B2B 제휴 모드에서는 **기본 비활성** (파트너 상품 외 자유 학습 차단) — 옵션으로 활성화 가능 |
| 게이미피케이션(§11.11) | 파트너가 자체 보상 시스템 운영 시 picklass 측 미적용 |
| 푸시·알림(§11.12) | 파트너가 자체 발송 → picklass는 데이터만 전달 |
| 수료 기준(§11.10) | 파트너 정의 임계값 사용, picklass는 4지표 측정만 |

#### 16.10.2 Speaking 채널 적용

Speaking(§14)·Pick-Speak Method 학습을 외부 파트너사 사이트에 임베드하는 케이스. **B2B 제휴의 핵심 사용 패턴**(예: 1:1 회화 채널 §3.3.4)이며, 가장 정교한 운영 정책이 적용된다.

##### 16.10.2.1 Pick-Speak Method B2B 임베드 흐름

§14.4 Pick-Speak Method 5단계를 외부 임베드 환경에서 실행. 모든 단계는 picklass 임베드 위젯 내부에서 진행되며, 시작·종료 시점에 파트너 측에 콜백 발송.

```
[파트너 사이트] 학습자가 학습 시작 클릭
       ↓
[파트너 → picklass] ONETIME TOKEN 발급 + GET /external/verify (§14.10.5)
       ↓
[picklass 임베드 위젯 시작]
       ├─ PICK (1~2분)   ← analyzer 시퀀싱 결과 노출
       ├─ LEARN (3~5분)  ← LRN 모듈
       ├─ DRILL (5~8분)  ← VLM + EDR (3-stage)
       ├─ APPLY (5~10분) ← RPL + FRT + OMP
       └─ MEASURE (2~3분)← §15.4 PPD 산출
       ↓
[파트너 ← picklass] session.completed Webhook (콜백)
       ↓
[파트너 사이트] 학습자에게 결과 노출 + 자체 운영 처리
```

##### 16.10.2.2 5축 KPI 콜백 페이로드 (Speaking 한정)

5축 KPI는 §14.4 발음정확도·유창성·문법정확성·화용성·발화량. Speaking 세션 종료 시 파트너에게 다음 페이로드로 전달.

```json
{
  "event": "session.completed",
  "channel": "speaking",
  "external_user_id": "partner_user_12345",
  "course_id": "course_business_b1",
  "module_sequence": ["LRN", "VLM", "EDR", "RPL", "OMP"],
  "session_duration_sec": 1320,
  "five_axis_kpi": {
    "pronunciation_accuracy": 82,
    "fluency": 75,
    "grammar_accuracy": 68,
    "pragmatics": 73,
    "utterance_volume": 88
  },
  "module_results": [
    {
      "code": "EDR",
      "stage_results": [
        { "stage": "Read", "score": 84 },
        { "stage": "Fill-in", "score": 79 },
        { "stage": "Expand", "score": 73 }
      ],
      "five_axis_avg": { /* 5축 평균 */ },
      "expressions_mastered": 4
    },
    {
      "code": "FRT",
      "challenge_meter_hit": true,
      "total_utterances": 32,
      "korean_safety_count": 1
    },
    {
      "code": "OMP",
      "topic": "Describe a recent business meeting",
      "preparation_time_sec": 30,
      "speech_duration_sec": 58,
      "five_area_score": {
        "pronunciation": 80,
        "fluency": 75,
        "grammar": 70,
        "pragmatics": 78,
        "volume": 85
      },
      "test_grade_predict": "OPIc IH"
    }
  ],
  "ppd_snapshot": { /* §15.4 PPD */ },
  "completed_at": "2026-05-08T10:30:00Z"
}
```

##### 16.10.2.3 6 모듈 임베드 정책

각 모듈을 외부 임베드 시 활성화 옵션은 파트너 상품 매핑(§16.8)으로 결정.

| 모듈 | 단계 | 외부 임베드 활성화 정책 |
|---|---|---|
| **LRN** Learn & Study | LEARN | ✅ 항상 활성 (콘텐츠 입력 필수) — 콘텐츠 출처: 파트너 교재(§8.3.5) 또는 picklass 자동 |
| **VLM** Vocabulary Listening & Meaning | DRILL 워밍업 | A1~A2 활성 / B1~B2 즐겨찾기 안 한 어휘만 / B2+ 자동 스킵 (§14.4.2) |
| **EDR** Expression Drill | DRILL 메인 | ✅ 항상 활성 (5축 측정 핵심) — 3-stage Read→Fill-in→Expand |
| **RPL** Role-Play | APPLY P1 | ✅ 항상 활성 — 파트너가 시나리오 제공 가능 (옵션) |
| **FRT** Free Talking | APPLY P2 | A1~A2 자동 비활성 / B1+ 활성 — 파트너 옵션으로 ON/OFF 가능 |
| **OMP** One Minute Presentation | APPLY P3 | A1~A2 자동 비활성 / B1+ 시퀀싱 결정 — **시험 대비·비즈니스·면접 학습자 시그니처** |

##### 16.10.2.4 음성 인프라 SLA (Whisper + Azure 듀얼 트랙)

§14.7.6 듀얼 트랙 STT를 B2B 제휴 환경에서 운영할 때의 SLA.

| SLA 항목 | 표준 | 엔터프라이즈 |
|---|---|---|
| STT 인식률 | ≥ 90% (Whisper + Azure 합성) | ≥ 92% (전용 모델 옵션) |
| 응답 지연 (발화 종료 → AI 응답 시작) | ≤ 1.2초 (p95) | ≤ 0.8초 (p95) |
| TTS 첫 토큰 지연 (TTFT) | ≤ 200ms | ≤ 150ms |
| 음성 품질 | 22kHz 16-bit | 44.1kHz 16-bit (옵션) |
| 가용성 | 99.5% | 99.9% |
| 데이터 보존 | 30일 | 협의 (최대 12개월) |

##### 16.10.2.5 추가 운영 옵션 (파트너별 ON/OFF)

| 옵션 | 기본값 | 용도 |
|---|---|---|
| 한국어 안전망 (Korean Safety Net) | ON (A1~A2) / OFF (B1+) | RPL Phase 1에서 한국어 발화 허용 횟수 (§14.4.6 차별점 3) |
| Silent Drill Mode | OFF | EDR 모듈에서 마이크 감도 극대화 + 키보드 입력 (§14.4.6 차별점 2) |
| 발화량 챌린지 미터 | ON | FRT Phase 2 게이미피케이션 |
| 캐릭터 다마고치 | OFF | 파트너가 자체 게이미피케이션 운영 시 OFF |
| 모바일 3분할 화면 | ON (강제) | §14.4.0.1 표준 — 파트너가 끌 수 없음 |
| 멘트 사전 생성 | ON (강제) | §14.4.0.3 — 파트너가 학습자 컨텍스트 추가 푸시 가능 |

##### 16.10.2.6 §14.10.5 외부 verify와의 통합

§14.10.5 GET /external/verify (`X-Access-Token` + `X-Module-Code`)는 본 §16.10.2 Speaking 채널의 **표준 진입 인증 방식**이다. 인증 흐름은 §16.7 ONETIME TOKEN과 동일하되, Speaking 모듈 코드(`X-Module-Code: LRN|VLM|EDR|RPL|FRT|OMP|PICK_SPEAK_FULL`)가 추가로 전달된다.

```
[파트너 → picklass]
GET /external/verify
Headers:
  X-Access-Token: {ONETIME_TOKEN}
  X-Module-Code: PICK_SPEAK_FULL    # 또는 개별 모듈 코드
       ↓
[picklass]
- 토큰 검증 (§16.7)
- 매핑 조회 (§16.6)
- 상품-운영 매핑 (§16.8)
- 모듈 활성화 정책 적용 (§16.10.2.3)
       ↓
[Pick-Speak Method 5단계 실행]
```

#### 16.10.3 외부 임베드 채널 (1:1 회화) — §3.3.4 cross-ref

> 📎 **§3.3.4 1:1 회화 채널 (파고다 외부 시스템 연계)** 참조 — §3.3.4는 1:1 회화 채널의 진입 정책·상품 라인업·전환경 지원·앱 설치 선택권 등 채널 운영 정책을 다룬다. 본 §16.10.3은 §3.3.4가 §16 B2B 제휴 운영 모델의 **표준 사례·표준 인스턴스**임을 명시하는 cross-ref 절이다.

##### 16.10.3.1 §3.3.4와 §16의 관계

| 측면 | §3.3.4 | §16 |
|---|---|---|
| 위치·역할 | 외부 채널 운영 정책 (학습자 진입·UX) | B2B 제휴 운영 모델 (책임 분리·인증·데이터·도구) |
| 추상화 레벨 | 채널 인스턴스 (1:1 회화 1개 채널) | 운영 모델 (모든 B2B 제휴 일반화) |
| 관계 | §3.3.4 = §16의 **첫 번째 표준 사례** | §16 = §3.3.4를 일반화·표준화한 모델 |

##### 16.10.3.2 1:1 회화 채널 → §16 모델 매핑

| §3.3.4 항목 | §16 대응 절 |
|---|---|
| 채널 명칭 변경 (전화외국어 → 1:1 회화) | §16 적용 사례 |
| AI 회화 라인업 추가 | §16.8 상품-운영 매핑 |
| PC/Mobile/App 전환경 지원 | §16.10.2.4 SLA + 자체 채널 정책 |
| 앱 설치 선택권 (Soft App-Install) | §3.3.4.4 (§16에 일반화 안 됨 — 파트너 자율) |
| 피드백 페이지 정책 (일간 레벨별 / 월간 절대값) | §16.10.2.2 콜백 페이로드 + §16.9 운영 콘솔 |
| 외부 verify (`X-Access-Token` + `X-Module-Code`) | §16.7 + §14.10.5 + §16.10.2.6 |

##### 16.10.3.3 향후 확장 (다른 외부 채널)

§16 B2B 제휴 운영 모델은 1:1 회화 외에도 다음 채널에 적용 가능:

| 후보 채널 | 적용 시 §16.10.3 확장 |
|---|---|
| 다른 어학원 자체 사이트 임베드 | §16 표준 그대로 |
| 기업 HR 시스템(Cornerstone, SAP SuccessFactors 등) 임베드 | §16.7 OAuth/OIDC 인증 + §16.9 BI 연동 |
| 인강·MOOC 플랫폼 임베드 | §16.10.1 Tutoring 채널 + §16.8 시험 대비 매핑 |
| 정부·공공 영어 교육 사업 | §16 + 별도 보안 검토(§22.5) |

> 📎 본 §16.10.3은 §3.3.4 외에 추가되는 모든 외부 임베드 채널의 표준 진입점이 된다. 신규 파트너 온보딩 시 본 절을 출발점으로 사용한다.

### 16.11 데이터 모델 (요약)

```sql
table partner_organizations (
  id pk,
  name varchar(255),
  contract_status enum('draft','active','suspended','terminated'),
  callback_url text,                    -- 진척도 콜백 받을 URL
  api_key_hash varchar(255),            -- 픽클래스 API 호출용
  module_scope enum('speaking_only','tutoring_only','unified'),
  completion_policy_strategy enum('partner_managed','picklass_managed'),
  textbook_intent_required bool default false,  -- 교재 의도 메타데이터 강제 여부
  created_at timestamp
);

table partner_callbacks_log (
  id pk,
  partner_id fk,
  callback_type enum('session_completed','progress_updated','completion_achieved'),
  payload jsonb,
  status enum('queued','sent','failed','retried'),
  sent_at timestamp,
  response_code int,
  retry_count int
);
```

### 16.12 운영 KPI

| KPI | 목표 |
|---|---|
| 콜백 전송 성공률 | ≥ 99% |
| 콜백 평균 지연 (세션 종료 후) | ≤ 60초 |
| API 가용성 (verify) | ≥ 99.9% |
| 제휴사 신규 온보딩 평균 시간 | ≤ 2주 |

### 16.13 리스크 및 완화

| 리스크 | 영향 | 완화 |
|---|---|---|
| 제휴사 콜백 URL 다운 | 진척도 전달 실패 | 큐 기반 재시도 (지수 백오프), 7일 보존 후 폐기 |
| 책임 경계 모호 (학습자 CS) | 제휴사 ↔ 픽클래스 책임 핑퐁 | 계약서에 1·2차 CS 책임 명문화, FAQ 분기 안내 |
| 제휴사별 커스터마이징 요구 누적 | 운영 복잡도 ↑ | 표준 인터페이스 고수, 커스터마이징은 `partner_organizations.config` jsonb로 한정 |
| 데이터 동기화 지연 | 수료 판정 오차 | 실시간(Webhook) + 배치(매일 00:30) 이중화 |

> 📎 본 모델은 §3.3.4 1:1 회화 채널을 일반화·표준화한 것으로, 향후 추가되는 모든 B2B 제휴는 본 §16 모델을 기준으로 운영된다. **§16.10 채널별 적용**에서 Tutoring·Speaking·외부 임베드 3개 채널의 적용 상세를 다룬다.

---

---

## 17. 핵심 도메인 정책

### 17.1 액세스코드 생성·활성화·만료 규칙

- 6자리 (A–Z [I, O 제외] + 2–9 [0, 1 제외])
- 유효기간: 1M/3M/6M/12M
- 생명주기: 생성(비활성) → 할당(비활성) → 사용(활성) → 만료 → 탈퇴
- 재발급: 기존 폐기 후 신규 생성

### 17.2 요금제별 기본값 및 자동 채움

- DEFAULT_PLANS 상수 기반 자동 초기화
- 수정 후 개별 기관 커스텀 값 저장
- 플랜 변경 시 관련 필드 일괄 갱신, 기존 계약 가격 보존

### 17.3 비용 정책

- 기본 포함: 플랜 월정액
- 초과 사용료: 모듈별 정의 (AI 생성 1,000원/요청, TTS 50원/1,000자 등)
- 일회성 비용: API 연동비, 기술지원비, IP 전환비
- **음성 세션 초과 과금**: Speaking 플랜별 한도 초과 시 분당/세션당

### 17.4 계약 상태 전환 정책

- 협의중 → 계약완료 → 활성 → 만료/해지
- 자동갱신(Y): 종료일 도래 시 +1년 자동 연장
- 해지 시 잔여 환불 + 위약금 협의

### 17.5 데이터 검증 정책

- 이메일: 표준 형식, 최대 254자
- 패스워드: 8자 이상, 최대 128자
- 연락처: 010-0000-0000
- 기관명: 1–100자
- 숫자 필드: 음수 불가
- 날짜 필드: 시작일 < 종료일

---

## 18. 사용자 플로우 (통합)

**역할별 진입 도메인 요약**

| 역할 | Phase 1 진입 도메인 | 비고 |
|---|---|---|
| 학생 (학부모 포함) | `www.picklass.com` → `tutoring.picklass.com/login` | Speaking 독립 앱은 `speaking.picklass.com` 직접 진입 가능 |
| 강사 | `studio.picklass.com/` → `studio.picklass.com/app` | |
| 기관장 (학원관리자) | `studio.picklass.com/` → `studio.picklass.com/app` → 관리 기능 | 권한 확인 후 admin으로 승격 |
| 시스템관리자 | `studio.picklass.com/` 로그인 → `admin.picklass.com` | |

### 18.1 비로그인 → 랜딩 → 가입

- **학생/학부모**: `www.picklass.com` → [학생 로그인] or [액세스코드 입장] → tutoring 이동
- **기관장·강사**: `studio.picklass.com/` 홍보 랜딩 → 회원가입 모달 → 타입/인증방식 선택 → 타입별 정보 입력 → 약관 동의 → 완료
- **제휴 문의(비로그인)**: `studio.picklass.com/` 데모 신청 폼 제출

### 18.2 학습자 여정 (읽기 중심 레슨)

1. `www.picklass.com` 접속
2. [학생 로그인] 클릭 → `tutoring.picklass.com/login` 리다이렉트
3. 이메일/소셜 로그인 → tutoring 학습 홈 진입
4. `/modules/[lessonId]` → LessonPlan 로드
5. ModuleRunner 첫 모듈 시작
6. Orchestrator 실시간 의사결정 루프
7. 모듈 완료 → 성과카드 → 다음 모듈
8. 최종 완료 → 평균 정답률

### 18.3 학습자 여정 (Speaking 독립 앱)

1. `speaking.picklass.com` 직접 접속 (또는 tutoring 학습 홈에서 이동)
2. 레벨/주제/시간 선택
3. PHASE 1 Opening → TTS 재생
4. PHASE 2 순환 대화 루프
5. PHASE 3 KPI 리포트 + 저장
6. 다음 세션 추천

### 18.4 강사 여정 (Studio)

1. `studio.picklass.com/` B2B 홍보 랜딩 접속 (또는 기관에서 받은 코드로 자유 가입)
2. 회원가입 → 역할 "강사" 선택 → 소속 기관 코드 연결
3. `studio.picklass.com/app` 진입
4. 지문 생성/AI 분석
5. 레슨 자동 생성 (CurriculumPlanner) 또는 수동 조합
6. 학생 배정
7. 성과 모니터링

### 18.5 학원관리자 여정 (L2 InstitutionAdmin)

1. `studio.picklass.com/` 접속 → 제휴 문의 제출 (또는 자유 가입)
2. 계약 체결 후 세일즈가 admin에 Institution 등록 (sector=academy/enterprise/k12)
3. `studio.picklass.com/` 로그인 → 권한 확인 → `admin.picklass.com` 이동
4. 라이선스 구매 / 플랜 선택 (자기 Institution만)
5. 강사/학습자용 액세스코드 대량 생성 (과정 단위)
6. 코드 배포 → 기관 사용자 온보딩
7. Billing/Usage 모니터링

### 18.6 본부·지주사 여정 (L1 GroupAdmin)

**학원 프랜차이즈 본부 시나리오**
1. Partner 또는 세일즈 경로로 Group 등록 (sector=academy, 프랜차이즈 본부)
2. `studio.picklass.com/` 로그인 → GroupAdmin 권한 확인 → admin(제한 뷰)
3. 산하 가맹학원(Institution) 일괄 등록 또는 기존 가맹점 연결
4. **Group 단위 플랜 일괄 구매** → 산하 기관에 이용 권한 배분 *(좌석 단위 추적은 v0.17 폐기 — 활성 Membership 카운트 + 사용량 기준)*
5. 산하 기관별 통계·리포트 열람 (정답률, 세션 수, 완료율)
6. 표준 과정·템플릿 Group 공유 설정

**기업 HR 총괄 시나리오 (sector=enterprise)**
1. 계약 후 Group(지주사) 등록, HR 총괄이 GroupAdmin
2. 산하 계열사(Institution)를 연결, 과정·이용 권한 배분
3. 임직원 일괄 온보딩 (임직원 DB 연동 시 API, 미연동 시 액세스코드)
4. 계열사별 교육 이수율·성과 대시보드
5. 연간 재계약·갱신 협상 데이터 수집

### 18.7 파트너 여정 (L0 PartnerAdmin)

1. 세일즈 경로로 Partner 계약 체결, `admin.picklass.com` 초청
2. PartnerAdmin 계정 발급 → 산하 Group/Institution 등록 권한 획득
3. **복수 프랜차이즈 본부 또는 지주사 관리** (수평 포트폴리오)
4. Partner 단위 계약 후 산하에 이용 권한 배분 *(좌석 단위 추적은 v0.17 폐기)*
5. 수수료·정산 리포트 확인
6. 신규 계약 확장, 기존 계약 갱신

### 18.8 학부모 여정 (Phase 1 — 속성 기반)

1. 자녀가 학원 액세스코드로 Learner 계정 생성 시 보호자 이메일·연락처 입력
2. 진도 요약 이메일·SMS 자동 수신 (주 1회 등)
3. 학습 이슈 알림(미완료·저조 등) 수신
4. (B2C 결제 필요 시) 자녀 계정 공유 로그인으로 결제
5. **Phase 2+**: 학부모 독립 Membership → 자녀 진도 실시간 조회, 결제 주체 분리

### 18.9 시스템관리자 여정

1. `studio.picklass.com/` 로그인 (역할: 시스템관리자) → `admin.picklass.com` 이동
2. 모든 Organization(Partner/Group/Institution) 관리
3. 사용자·Membership·코드·모듈 관리
4. 플랜 상품 구성(§7.7.1) *(좌석 배분 정책 §7.7.2는 v0.17에서 폐기)*
5. AI 모듈 사용량·비용 모니터링
6. 감사 로그 조회

## 19. 기술 아키텍처

### 19.1 Admin 스택

- Frontend: Next.js 16 (App Router), TypeScript, Tailwind, shadcn/ui, Zustand
- Backend: NestJS, PostgreSQL, Prisma ORM
- Monorepo: pnpm + Turborepo
- 포트: Frontend 4100, Backend 4101, DB 5432

### 19.2 Studio / www 스택

- Next.js 15, React 19, TypeScript
- Supabase (DB/Auth/Storage/Realtime)
- Google GenAI, Azure Speech TTS
- Tailwind CSS, Radix UI, Lucide React

### 19.3 Tutoring 스택

- Next.js 15 (App Router)
- NestJS backend
- Anthropic Claude API (claude-haiku-4-5-20251001, Tool Use)
- Google Speech-to-Text (음성 평가)

### 19.4 Speaking Tutor 스택

- Next.js 15 (실시간 UI)
- **WebSocket/SSE** 기반 양방향 스트리밍
- Azure Speech (STT + Neural TTS)
- Anthropic Claude API (통합 처리 모드)
- 상태머신: Zustand + XState 고려
- 오디오 파이프라인: MediaRecorder + Web Audio API

### 19.5 API 엔드포인트 맵

```
Lesson:
  GET  /api/lesson/{id}/plan
  GET  /api/lesson/{id}/history

Module:
  POST /api/module/{code}/data
  POST /api/module/{code}/evaluate
  POST /api/module/{code}/history

Chat / Agent:
  POST /api/chat/feedback
  POST /api/chat/feedback/stream   ← SSE
  POST /api/chat/message
  POST /api/agent/orchestrator/decide

Speech Session:  ← 신설
  POST /api/speaking/session            (세션 시작)
  POST /api/speaking/turn               (턴 처리: STT + 통합 AI)
  POST /api/speaking/session/{id}/end   (세션 종료 + KPI)
  GET  /api/speaking/session/{id}       (세션 조회)
```

### 19.6 실시간 인프라

- Supabase Realtime (Studio/www): 태스크 상태, 알림
- SSE (Tutoring/Speaking): 스트리밍 피드백/대화 응답
- 오디오 스트림: PCM/Opus, WebSocket 기반

### 19.7 비동기 태스크 시스템

- `async_tasks` 테이블: 상태 추적(pending/running/done/failed)
- 페이지 새로고침 시 상태 복구
- 실패 시 자동 재시도 및 사용자 알림

### 19.8 배포 및 CI/CD

- Docker + Docker Compose
- GitHub Actions
- AWS EC2 / RDS / S3
- Blue-Green 배포 전략

---

## 20. 로드맵

### 20.1 Phase 1 (~2026 Q2)

**목표**: 핵심 Agent 기반 튜터링 엔진 완성 + **Speaking MVP** + **도메인 채널 분리 전환**

**T0 도메인 전환 작업 (Phase 1 착수 직후, 2~3주 내)**
- 📋 `studio.picklass.com/`에 B2B 홍보 랜딩 신규 구축 (기존 www B2B 콘텐츠 이관)
- 📋 기존 `www.picklass.com/partners`, `/demo`, `/pricing` 등 B2B URL → studio로 301 리다이렉트
- 📋 `www.picklass.com` 루트를 B2C 티저 + 학생 진입 UI로 교체 ([학생 로그인] / [액세스코드 입장] CTA)
- 📋 `admin.picklass.com` 도메인 설정 (백오피스 이관)
- 📋 검색엔진 재색인 요청, `robots.txt` 및 메타 태그 정비

**엔진 및 모듈 개발 (v0.12 정렬)**
✅ ModulesPage + LessonSession
✅ ModuleRunner 어댑터 (PRD, SCN, SKM, CLR, SUM, QAR, RRD, SWR — 완료 10개 중 일부)
✅ ModuleOrchestratorAgent Rule 엔진
🔄 CurriculumPlannerAgent Claude 연결
🔄 FeedbackGenerationAgent Claude 연결
🔄 발음 평가 API
📋 백오피스 레슨 편집 UI
📋 **Speaking Free Talking MVP (FRT — Scenario Based Free Talking)**
📋 Speaking 독립 앱 세션 화면

### 20.2 Phase 2 (2026 Q3)

**목표**: 모듈 라이브러리 확대 + **Speaking 7모듈 순차 구현** + **www B2C 본격 확장**

- 추가 Vocabulary·Writing 모듈 미작성·추가 합류 (WSD, WWB, IMG, WDN, WFT, UNS, CPW, SCP, PPR, PWR)
- Speaking 6개 미작성 모듈 완성 (**LRN, VLM, EDR, RPL, FRT, OMP**) — v0.15 통합 개편 (§13.1.6) 반영
- Orchestrator Rule→LLM 전환 (Claude Tool Use)
- KPI 5지표 고도화 (FRT 기준)
- Agent 관찰성(Observability) 구축
- 백오피스 모듈 라이브러리 CRUD
- 모듈 버전 관리 시스템
- **www.picklass.com 풀 B2C 브랜드 랜딩으로 확장** (Hero/Features/Pricing/FAQ/블로그)
- **개인 학습자 B2C 자유 가입 플로우** (www에서 구독 결제까지)

### 20.3 Phase 3 (2026 Q4+)

- 그룹 커리큘럼 (반별 맞춤)
- 동료 학습 기능
- AI 튜터 성격 커스터마이징
- 멀티모달 입출력 (비디오/이모지)
- 학습 게임화 (배지/랭킹/미션)
- 부모/관리자 정기 리포트

### 20.4 마일스톤 및 의존성

파고다 개발 마일스톤 계약(`파고다_개발_마일스톤_계약내용.pdf`)과 연동하여 세부 일정 관리. 주요 의존성:
- Azure Speech 계정 · 지역 선택
- Anthropic API 할당량
- 파고다 스펙 PDF(`픽클래스튜터링_스피킹_파고다_개발스펙.pdf`) 요건

### 20.5 회의록 260413 기반 고도화 트랙 *(v0.11 신설)*

> 📋 **고도화 (Phase 2 ~ Phase 3+)** — 회의록 "추후 고도화 안건"의 6개 트랙. 참고 레퍼런스: 맥스AI, 헤이링.

| 트랙 | 내용 | 시기 | 의존 |
|---|---|---|---|
| **T1. 맞춤형 레벨진단·로드맵** | 정밀 루브릭 + MBTI형 성향·학습 환경 분석 → AI 학습 로드맵 자동 생성 | Phase 2 (2026 Q3) | §15 정밀 진단, §10 시퀀싱 엔진 |
| **T2. 전화/화상 블렌디드 학습** | 전화/화상 수업 피드백(취약점) ↔ AI 학습 데이터 양방향 연동, 수업 녹취/녹화 분석 → 부족 영역 기반 AI 학습 자동 구성 | Phase 2~3 | §15.6 채널 일원화, 외부 시스템 인증 (§14.10.5) |
| **T3. 게이미피케이션 풀세트** | 포인트 외부 스토어 제휴, 캐릭터 다마고치 성장, 기업별 학습 랭킹 시스템 | Phase 3 | §11.11 게이미피케이션 골격 |
| **T4. 수업 참여 독려 자동화** | 수료율 기반 푸시·알림톡 자동 발송, OS 제약 검토 후 전화 수신 형태 푸시 | Phase 2 (일부 가능) | §11.12 푸시·알림 |
| **T5. 미디어 라이브러리** | 핵심 표현 사용된 드라마/영화/유튜브 클립 자동 큐레이션 | Phase 3+ | §13.13 미디어 라이브러리, 라이선스 검토 |
| **T6. 전사 레벨테스트 일원화** | B2B/B2C/전화/인강/출강 전사 레벨테스트 체계 일원화 | Phase A(Q3)→B(Q4)→C(2027 Q1) | §15.6 단계별 추진 |

#### 20.5.1 트랙 간 우선순위·의존 다이어그램

```
[Phase 1 (~Q2)]  현재 v0.10 정합성 + v0.11 신규 8개 항목 기획 확정
                   │
                   ▼
[Phase 2 (Q3)]  T1 정밀 진단 ─────────► T6 Phase A (B2B/B2C 일원화)
                T4 푸시 자동화 (가능 부분)
                   │
                   ▼
[Phase 3 (Q4+)] T2 블렌디드 (1:1 회화 채널 연동) ─► T6 Phase B
                T3 게이미피케이션 풀세트
                T5 미디어 라이브러리 (라이선스 검토 후)
                   │
                   ▼
[2027 Q1+]      T6 Phase C (인강·출강 통합)
```

#### 20.5.2 KPI (고도화 트랙)

| 트랙 | 성공 지표 | 목표 (Phase 종료 시) |
|---|---|---|
| T1 | 맞춤형 로드맵 채택률 | 학습자 50% 이상 시스템 추천 로드맵 사용 |
| T2 | 블렌디드 학습 활성화율 | 전화/화상 수강자 중 30% AI 학습 병행 |
| T3 | DAU↑ (게이미피케이션 효과) | 캐릭터 도입 후 DAU 20% ↑ |
| T4 | 푸시 → 학습 진입률 | 푸시 발송 대비 학습 진입 5% 이상 |
| T5 | 미디어 클립 노출 후 표현 정착률 | 클립 노출된 표현의 재발화율 +30% |
| T6 | 채널 간 레벨 매핑 정확도 | 5% 미만 오차 |

> 📎 참고 레퍼런스(맥스AI, 헤이링)에 대한 상세 분석은 §23 부록에서 추가 정리 예정 (v0.12+).

## 21. 리스크 및 완화 전략

### 21.1 기술 리스크

| 리스크 | 영향 | 확률 | 완화 |
|---|---|---|---|
| Claude API 비용 초과 | 비용 증가 | M | 배치/캐싱/claude-haiku 우선 |
| 음성 평가 정확도 저하 | 불만족 | M | Azure + Google 이중화 |
| DB 성능 저하 | 응답 지연 | M | 파티셔닝/인덱싱/읽기 복제본 |
| 프론트엔드 복잡도 | 유지보수 | M | Zustand/XState, 테스트 커버리지 |
| 프롬프트 인젝션 | 비정상 응답 | L | user role 분리, 이스케이프 |
| AI 환각 | 잘못된 피드백 | M | Tool Use 구조화, Guardrail |
| 컨텍스트 초과 | Agent 오작동 | L | Rolling Window + 요약 |
| **음성 실시간 지연** | UX 치명적 | M | **스트리밍 + 추임새 우선 재생** |

### 21.2 비즈니스/교육 리스크

| 리스크 | 영향 | 완화 |
|---|---|---|
| AI 튜터 오답 | 학생 신뢰도↓ | 교사 검토, A/B 테스트 |
| 모듈 과다 설계 | 중도 포기 | 시간 예측 검증, UT |
| 이종 브라우저 | 일부 배제 | Cross-browser 테스트 |
| **미성년 음성 동의** | **컴플라이언스** | **보호자 동의 체계, PIPA/COPPA 검토** |

### 21.3 운영 리스크

| 리스크 | 대응 |
|---|---|
| 모듈 버전 충돌 | 엄격한 버전 정책, 마이그레이션 테스트 |
| 데이터 손실 | 1일 1회 백업, replication |
| 배포 중단 | CI/CD 자동화, Blue-Green |
| **음성 인프라 장애** | **이중화 + graceful degradation** |

---

## 22. 보안 및 컴플라이언스

### 22.1 인증 및 세션 관리

- 이메일/비밀번호 (8자+), 소셜 OAuth (Google/Kakao/Naver)
- 2FA 계획 (향후)
- 세션 토큰, 만료 정책
- **도메인 간 인증 핸드오프**: studio 로그인 → admin 이동 시 서버 사이드 세션 또는 JWT 전달. www ↔ tutoring 간 공유가 필요한 경우에만 `.picklass.com` 상위 도메인 쿠키 사용

### 22.2 데이터 보존 및 삭제

- 회원 정보: 계정 삭제 시까지 보관
- 로그인 로그: 보안 목적 일정 기간
- **음성 녹취 보관 정책**:
  - 기본: 세션 종료 후 학습 리포트용으로 30일 보관
  - 명시적 동의 시: 장기 보관 (학습 추이 분석)
  - 사용자 요청 시 즉시 삭제
  - 미성년: 보호자 동의 없이 장기 보관 금지

### 22.3 개인정보 처리 및 이용약관

- 수집 항목·목적·보관 기간 명시
- 마케팅 수신 동의는 선택
- 제3자 제공 시 별도 동의

### 22.3.1 인덱싱 및 공개성 정책

- **공개 인덱싱 허용**: `www.picklass.com`, `studio.picklass.com/` (루트)
- **noindex 필수**: `studio.picklass.com/app/**`, `admin.picklass.com`, `tutoring.picklass.com`, `speaking.picklass.com`
- `robots.txt` 및 `<meta name="robots" content="noindex">`로 이중 차단
- B2B 랜딩(studio 루트)에는 "학부모·학생 대상 콘텐츠 없음" 원칙을 명시하여 파트너 신뢰 확보

### 22.4 감사 로그

- 관리자 작업 이력 (누가/언제/무엇을)
- 상태 변경 전/후 기록
- 최소 1년 보관

### 22.5 외부 LLM 보안 검토 *(v0.11 신설 — 회의록 260413)*

> 📋 **기획 단계 (지속 검토)** — 회의록 "보안 사항" 항목. 외부 LLM(생성형 AI 엔진) 연동 시 발생하는 보안 이슈를 6개 검토 축으로 정리.

#### 22.5.1 검토 항목 6축 (회의록 명시)

| 검토 축 | 내용 | 진행 상태 |
|---|---|---|
| **① 보안 이슈 없는 엔진 사용 여부** | 제휴사 보안 정책에 따라 **허용 LLM 기준** 확인 필요 | 📋 검토 중 |
| **② 방화벽·데이터 암호화** | 외부 LLM 연동 시 방화벽 정책, 전송 구간/저장 구간 암호화 | 📋 검토 중 |
| **③ 사용 엔진 공유 가능 여부** | 사용 중인 LLM 모델명 공유 가능 여부 + 엔진별 보안 수준 | 📋 검토 중 |
| **④ 특정 LLM 제외 운영** | 고객사 요구에 따라 **특정 LLM 제외**(예: GPT만 허용, Claude 제외) 등 선택적 운영 | 📋 검토 중 |
| **⑤ 비용·모델 변경** | 현재 **10분당 150~250원** 수준 원가, 사용 모델 변경 시 비용 변경 가능 | ✅ 현재 기준 확정 |
| **⑥ 지속적 보안 검토** | 외부 LLM 연동 보안 대책은 **지속적으로 검토**하기로 함 (회의 결정) | 📋 ongoing |

#### 22.5.2 LLM 사용 정책 — 허용 엔진 매트릭스 (예시)

| 엔진 | 보안 수준 | 픽클래스 사용 | B2B 고객사 옵션 |
|---|---|---|---|
| OpenAI GPT-4o (Azure OpenAI) | 🟢 Azure 자체 컴플라이언스 | ✅ 사용 중 | ✅ 기본 옵션 |
| OpenAI GPT-4o (OpenAI 직접) | 🟡 검토 필요 | ⚠️ 제한적 | ❌ 보안 검토 후 |
| Anthropic Claude (AWS Bedrock) | 🟢 AWS 자체 컴플라이언스 | ✅ 사용 중 | ✅ 옵션 가능 |
| Anthropic Claude (Anthropic API 직접) | 🟡 검토 필요 | ⚠️ 제한적 | ❌ |
| Google Gemini | 🟡 검토 필요 | ❌ 미사용 | ❌ |
| 오픈소스 LLM (자체 호스팅) | 🟢 자체 통제 | 📋 향후 검토 | ✅ 추후 옵션 |

> ⚠️ 위 매트릭스는 본 v0.11 시점의 작성자 추정 — **고객사·법무·보안팀 검토 후 확정 필요**.

#### 22.5.3 비용 모델 (LLM 토큰 + 음성)

| 항목 | 단가 (현재 기준) | 비고 |
|---|---|---|
| **음성 처리 합산 (10분당)** | **150~250원** | 회의록 명시. STT + LLM + TTS 통합 |
| LLM 토큰 (입력+출력) | 변동 (모델별) | 모델 변경 시 ±50% 폭 변동 가능 |
| Whisper STT 추가 (§14.7.6) | +10~30원/10분 | 듀얼 트랙 적용 시 추가 |

→ 사용 모델 변경 시 원가 변경 가능성 있음 (회의록 명시).

#### 22.5.4 운영 정책

| 정책 | 내용 |
|---|---|
| **데이터 마스킹** | LLM 호출 전 학습자 PII(이름·이메일 등) 마스킹 처리 |
| **로그 보관** | LLM 요청/응답 로그는 **30일 보관 후 자동 삭제** (감사 목적), 학습자 식별자 분리 저장 |
| **Opt-out** | 고객사가 자사 데이터의 LLM 학습 사용 거부 시, "no-train" 헤더 또는 자체 배포 모델 사용 |
| **장애 대응** | LLM 장애 시 fallback 엔진 자동 전환 (Azure OpenAI ↔ AWS Bedrock) |
| **재검토 주기** | 분기 1회 보안 검토, 연 1회 외부 감사 |

> 📎 음성 인프라(STT/TTS)의 듀얼 트랙 정책은 §14.7.6 참조. KPI 측정 기록의 보존 정책은 §22.2 참조.

---

## 23. 부록

### 23.1 화면 와이어프레임 / 목업

> 별도 디자인 파일(Figma) 링크로 대체 예정.

### 23.2 디자인 가이드 요약

- 파고다 B2B 디자인 가이드(`파고다B2B_디자인가이드_디자인팀_v1(251211).pdf`) 연동
- 역할별 색상 팔레트: 시스템관리자(연두) / 학원관리자(파랑) / 강사(주황) / 학생(자주)
- 상태 배지: 활성(초록) / 비활성(노랑) / 정지(빨강) / 탈퇴(회색)

### 23.3 파트너(파고다) 계약 마일스톤 요약

- `파고다_개발_마일스톤_계약내용.pdf` 기준
- `픽클래스튜터링_스피킹_파고다_개발스펙.pdf` 기반 스피킹 요건 반영
- `픽클래스_전체구조_스튜디오_개발스펙.pdf` 기반 스튜디오 스펙 반영

### 23.4 참고 문서 인덱스

#### 23.4.1 Backoffice docs 매핑

| 문서 | 본 기획서 반영 장 |
|---|---|
| `docs/index.md` | §6 랜딩 플로우 |
| `docs/users/users.md` | §4, §7.4 |
| `docs/institute/institute.md` | §7.3 |
| `docs/accesscode/accesscode.md` | §7.5, §17.1 |
| `docs/ai-modules/ai-modules.md` | §7.6 |
| `docs/billing/billing.md` | §5, §7.7 |
| `docs/system/system.md` | §7.8, §4.4 |

#### 23.4.2 Studio docs 매핑

| 문서 | 본 기획서 반영 장 |
|---|---|
| `README.md` | §2.3, §8.1 |
| `docs/course-*`, `course-detail-*` | §8.2–§8.4 |
| `docs/module-db-integration-plan-*` | §13, §18 |

#### 23.4.3 Tutoring docs 매핑

| 문서 | 본 기획서 반영 장 |
|---|---|
| `docs/picklass-tutoring-planning.md` | §11, §12, §13 |
| `docs/agent-architecture-context-*` | §12 |
| `docs/modules-lessonId-*` | §9.2 |
| `docs/외부API_명세서_*` | §19.5 |

#### 23.4.4 Speaking 폴더 매핑

| 문서 | 본 기획서 반영 장 |
|---|---|
| `스피킹 교수법 조사 ...md` | §14.4 7모듈 근거, §1.3 용어 |
| `md.Picklass Speaking Tutor - Free-talking.docx` | §14.1–§14.10 전반 |
| `스피킹대화수업로직_20260214.pdf` | §14.3 3-Phase, §12.5 |
| `Picklass Tutoring 수업모듈 SB (1).pdf` | §13, §14.4 |
| `픽클래스_파고다_킥오프미팅.docx` | §20.4, §23.3 |

### 23.5 변경 이력 및 승인

| 버전 | 작성자 | 날짜 | 승인 |
|---|---|---|---|
| v0.1 | Tim | 2026-04-17 | Draft |
| v0.2 | Tim | 2026-04-17 | Draft (Speaking 반영) |
| v0.6 | Tim | — | Draft (Actor·조직 3-Tier 전면 재편) |
| v0.10~v0.11 | Tim | — | Draft (구현 정합성 보고 + 회의록 260413 신규 8항목) |
| v0.12 | Tim | — | Draft (Module List v1.0 정렬 — 29모듈·코드 표준화) |
| v0.13~v0.15 | Tim | — | Draft (PPD 진단 엔진 + Pick-Speak Method + Speaking 6모듈 통합 개편) |
| v0.16~v0.18 | Tim | — | Draft (B2B 교재 업로드·다형 콘텐츠 + 좌석 모델 폐기·기관 브랜딩 + 계정/인증/상품매핑/운영도구) |
| v0.19 | Tim | — | Draft (§16 B2B 제휴 운영 모델 챕터 승격) |
| v0.20 | Tim | 2026-05-08 | Draft (목차 신설 + 구 §1.4·§19·§22 삭제 + 도메인 정리) |
| **v0.21** | Tim | **2026-07-17** | Draft (**정합성 일괄 정정** — 구 장번호 cross-ref 11곳, Speaking 6모듈 잔여 반영, 모듈 총수 28 통일, 좌석 배분 잔재 제거, 변경 이력 갱신) |

---

**문서 끝.**

> 본 기획서는 Picklass 플랫폼의 전체 구조를 기획 관점에서 통합 정리한 문서(v0.21)이다. 개별 도메인의 상세 개발 스펙은 각 서비스 `docs/` 폴더를 우선 참조하며, 본 문서는 **의사결정·일정·리스크 관리의 상위 기준**으로 활용한다.
