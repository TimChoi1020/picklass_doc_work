# Backoffice 기획서 §7 · 예외·상태

> 참고문서(`참고문서/picklass_docs/backoffice/`) 기반. 화면 §4·기능 §5·정책 §6 참조.
> 구조: ①공통 상태 표준 → ②화면별 매트릭스 → ③예외 카탈로그.

---

## 0. 공통 화면 상태 표준

| 상태 | 표준 UI | 행동 |
|---|---|---|
| Loading | 스켈레톤/스피너 | 조작 비활성 |
| Empty | 안내 문구 + (해당 시) CTA | "검색 결과가 없습니다" 등 |
| Error | 오류 배너/토스트 | 재시도 |
| **권한없음 Tier1** | "접근 권한이 없습니다." 표시 후 `/admin/dashboard` 리다이렉트 | 라우트 차단 |
| **권한없음 Tier2** | 필터 LOCK(pre-fill+disable), scope=[] 시 빈 결과 | 데이터 격리 |
| 준비중 | UI 노출 + 미연동 표식 | 백엔드 대기 |

---

## 1. 화면별 상태 매트릭스

| 화면ID | Loading | Empty | Error | 권한없음/제약 |
|---|---|---|---|---|
| BO-S-010 조직목록 | 스피너 | 목록 없음 | 오류 | academy_admin 등록버튼 없음(Tier2) |
| BO-S-020 사용자목록 | 스피너 | 목록 없음 | 오류 | partnerName/groupName 컬럼 항상 `-`(HARD 미연동) |
| BO-S-031 과정목록 | 스피너 | "과정이 없습니다" | 오류 | 생성/편집 액션 없음(읽기전용) |
| BO-S-033 액세스코드 | 스피너 | 통계 0 | 생성 실패 토스트 | completed/expired 전환버튼 없음(E-008) |
| BO-S-041 기관학습현황 | 스피너 | 결과 없음 | 오류 | academy_admin 접근 불가(E-004) |
| BO-S-043 출석률 | 스피너 | 결과 없음 | 오류 | 소급 불가 안내(로그 이전) |
| BO-S-055 API키관리 | — | — | — | 미연동(Phase G) |
| BO-S-070/071 API로그 | — | — | — | 전체 목업 |

---

## 2. 예외 시나리오 카탈로그

### BO-E-001 · 입력검증 실패
- **발생**: 필수 누락·형식/범위 위반(등록·생성 폼)
- **문구 예**: "필수 항목을 입력해주세요", "등록 만료일은 오늘 이후로 설정해주세요", "학생 선택 시 제공할 과정을 선택해주세요", 계약 시작일 ≥ 종료일 거부
- **처리**: 클라 검증 + 서버 재검증

### BO-E-002 · 이메일/아이디 중복
- **발생 화면/기능**: BO-S-011·021 / BO-F-011, 021
- **문구**: "이미 사용 중인 이메일입니다" / "사용 가능한 이메일입니다" / (미확인 시) "이메일 중복 확인을 완료해주세요"
- **처리**: `POST /users/check-duplicate` 통과 전 등록 차단

### BO-E-003 · 개수 상한 초과 (+클램핑 버그)
- **발생 화면/기능**: BO-S-023·034 / BO-F-024, 034
- **조건**: 개수 > 1000
- **문구**: "최대 1,000개까지 생성 가능합니다"
- ⚠️ **[버그]** 서비스 내부 `Math.min(count, 100)` — UI에서 1000 입력해도 실제 100개만 생성. UI 한도(1000)와 서버 한도(100) 불일치 **정합 필요**(BO-P-010, O-2)

### BO-E-004 · 권한없음 (Tier 1)
- **발생**: SYS 전용 화면에 하위 역할 접근(URL 직접 포함)
- **처리**: `AdminRoleGate` → "접근 권한이 없습니다." → `router.replace('/admin/dashboard')`
- **연관 정책**: BO-P-002

### BO-E-005 · 권한없음 (Tier 2)
- **발생**: 접근은 되나 데이터 범위 밖
- **처리**: 필터 LOCK(pre-fill+disable), 백엔드 scope=[] → 빈 결과. 쿠키 변조는 무효(JWT가 진실의 원천)
- **연관 정책**: BO-P-003~006

### BO-E-006 · scope 단건조회 갭
- **발생 화면/기능**: BO-S-032 등 상세 / BO-F-031
- **조건**: `GET /:id`가 scope 확인 없이 반환
- **위험**: URL 직접 접근 시 타 기관 데이터 조회 가능 — **알려진 보안 갭**(O-3)

### BO-E-007 · 자동 로그인 실패
- **발생 화면/기능**: BO-S-020 / BO-F-023
- **문구**: "자동 로그인에 실패했습니다"
- **처리**: 핸드오프 토큰 60초 만료 재사용 불가(BO-P-012)

### BO-E-008 · 최종상태 코드 전환 시도
- **발생 화면/기능**: BO-S-033 / BO-F-035
- **조건**: completed/expired 코드
- **처리**: 상태 전환 버튼 미노출(BO-P-020)

### BO-E-009 · resolveUserScope userId↔UUID 혼동 (과거 버그)
- **발생 화면/기능**: BO-S-031 / BO-F-030 (course.service)
- **원인**: `where:{userId}`(로그인 문자열)로 조회 → partner/group/academy 로그인 시 과정 목록 완전 공백(`data:[]`)
- **수정**: JWT `payload.userId = user.id`(UUID)이므로 `where:{id: userId}`. **재발 방지**: id=UUID / userId=로그인문자열 구분, resolveUserScope 중복 구현 금지(InstitutionService 재사용, BO-P-004)

### BO-E-010 · 코드관리 트랜잭션 타임아웃
- **발생 화면/기능**: BO-S-050 / BO-F-050 (saveGroupItems)
- **원인**: 트랜잭션 내 N개 순차 update → Prisma 5초 초과 "Transaction not found"(Vercel 콜드스타트 빈발)
- **수정**: createMany/updateMany 배치 전환 + 임시 timeout 30000. (Promise.all도 단일 커넥션이라 직렬)

### BO-E-011 · 카테고리 배정 위반
- **발생 화면/기능**: BO-S-035 / BO-F-037
- **조건**: L1 단독 배정 / 교차 파트너 카테고리
- **처리**: BadRequest / 파트너 가드 차단(BO-P-031)

---

## 3. 에러코드 ↔ 사용자 메시지

| 상황 | 메시지 | 후속 |
|---|---|---|
| 이메일 중복 | "이미 사용 중인 이메일입니다" | 차단 |
| 중복확인 미완료 | "이메일 중복 확인을 완료해주세요" | 차단 |
| 개수 초과 | "최대 1,000개까지 생성 가능합니다" | 차단 |
| expiryDate 과거 | "등록 만료일은 오늘 이후로 설정해주세요" | 차단 |
| 학생 코드 과정 미선택 | "학생 선택 시 제공할 과정을 선택해주세요" | 차단 |
| Tier1 위반 | "접근 권한이 없습니다." | dashboard 리다이렉트 |
| 자동 로그인 실패 | "자동 로그인에 실패했습니다" | 토스트 |
| 파트너 API 오류(외부) | MEMBER_NOT_FOUND / NO_ENROLLMENT / INVALID_OPTIONS | — |

---

## 4. 미구현 / 준비중 (Known Gaps)

| 영역 | 내용 | 정책/기능 |
|---|---|---|
| API키관리 | UI완료·백엔드 미연동(Phase G), Webhook 필드 Phase C | BO-F-053 |
| 과정 카테고리 | `bulk-assign-category`·파트너 상세 탭 미구현 | BO-F-037 |
| 파트너 API 연동 | 내 연동 설정 Step2(Phase F-3), 연동 테스트 Step3(Phase F-2), 외부 Pull `/b2b/results` Phase B-2 | BO-F-061, 062 |
| API 로그 | enrollments/members 전체 목업 | BO-S-070, 071 |
| 통계 HARD 필터 | 상위파트너/상위그룹 UI 고정만(parent_id 스키마 대기), 출석률 courseId UI 필드 없음 | BO-F-040, 041 |
| 사용자관리 | partnerName/groupName 컬럼 항상 `-`, Scope 백엔드 미연동(문서 불일치 O-1) | BO-F-020 |
| scope 단건조회 | `GET /:id` scope 미확인 보안 갭 | BO-E-006 |
| 보안 | isTempPassword 강제변경 미구현, generatedPassword 평문 저장 | BO-F-024 |
| 데이터 정합 | 카테고리 DB CHECK 제약 미채택(앱 가드만), student_count/deployment_count 컬럼 제거 미결 | BO-P-033 |

> **Scope 연동 상태는 문서 간 불일치**(O-1)가 있어 실제 코드 확인 후 확정 필요.
