# Backoffice 기획서 §4 · IA·화면구조

> 참고문서(`참고문서/picklass_docs/backoffice/`) 기반. **SSOT**: `menu-redesign/20260527_메뉴구조_권한모델_개발계획.md`
> 이 문서가 정의하는 **화면 ID(BO-S-###)** 를 §5(기능)·§6(정책)·§7(예외)가 참조합니다.

---

## 0. 개요 & IA 설계 배경

**Backoffice는 Picklass 플랫폼 전체를 운영·관리하는 통합 관리자 도구**입니다. **관리 대상별 대메뉴**(조직·사용자·콘텐츠·통계·시스템)와 **계층 조직 × 역할 기반 2-Tier 접근제어**가 IA의 두 축입니다.

- **조직 계층**: 플랫폼 > 파트너 > 그룹 > 기관
- **2-Tier 접근제어**(§6 핵심): 같은 메뉴도 역할에 따라 **메뉴 자체가 차단(Tier 1)** 되거나 **데이터 범위가 자기 조직으로 제한(Tier 2)** 됩니다.

---

## 1. 네비게이션 / 메뉴 트리

범례: ★=SYS 전용(Tier1) / 🤝=PARTNER_ABOVE / ⬢=ORG_ABOVE / (무표기)=ALL_ADMIN

```
조직관리                /admin/institute
사용자 관리             /admin/users
콘텐츠 플랫폼
 ├ 지문 관리 ★          /admin/speaking-core-data
 ├ 과정 목록            /admin/courses
 ├ 액세스코드           /admin/accesscode
 └ 과정 카테고리 🤝     /admin/content/categories
통계 / 리포트
 ├ 플랫폼 현황 ★        /admin/stats/platform
 ├ 기관 학습 현황 ⬢     /admin/stats/institutions
 ├ 학습자 현황          /admin/stats/learners
 ├ 출석률 ⬢            /admin/stats/attendance
 └ B2B 학습데이터 ★     /admin/b2b/results
시스템 ★
 ├ 코드관리 / 수업모듈 / 수업모듈 코드관리 / 스피킹 지문 프롬프트
 └ 시스템관리자 계정 / API키관리
파트너 API 연동 🤝
 ├ API 연동 가이드 / 내 연동 설정 / 연동 테스트(★)
API 로그 ★
 └ 수강로그 / 로그인로그
```

---

## 2. 화면 인벤토리 (Screen Inventory)

> 접근권한은 §6 BO-P-001~002 기준. 유형: Page / Modal / Sub(탭·드릴다운).

| 화면ID | 화면명 | 경로 | 접근권한 | 연관 기능(§5) | 상태 |
|---|---|---|---|---|---|
| BO-S-001 | 대시보드 | `/admin/dashboard` | ALL | — | ✅ |
| **조직관리** | | | | | |
| BO-S-010 | 조직 목록 | `/admin/institute` | ALL | BO-F-010 | ✅ Scope연동 |
| BO-S-011 | 조직 등록 | `/admin/institute/register` | ALL(등록:ORG↑) | BO-F-011 | ✅ |
| BO-S-013 | 조직 상세(탭) | `/admin/institute/[id]` | ALL | BO-F-012~013 | ✅ |
| **사용자 관리** | | | | | |
| BO-S-020 | 사용자 목록 | `/admin/users` | ALL | BO-F-020 | ⚠️ Scope UI만 |
| BO-S-021 | 사용자 등록 | `.../register` | ALL | BO-F-021 | ✅ |
| BO-S-023 | 아이디 일괄생성 | `/admin/users/access-code` | ALL | BO-F-024 | ✅ |
| **콘텐츠 플랫폼** | | | | | |
| BO-S-030 | 지문 관리 | `/admin/speaking-core-data` | ★SYS | — | ✅ |
| BO-S-031 | 과정 목록 | `/admin/courses` | ALL | BO-F-030, 032 | ✅ 읽기전용 |
| BO-S-032 | 과정 상세(탭) | `/admin/courses/[id]` | ALL | BO-F-031 | ✅ |
| BO-S-033 | 액세스코드 목록 | `/admin/accesscode` | ALL | BO-F-033, 035 | ✅ |
| BO-S-034 | 액세스코드 생성 | `.../generate` | ALL | BO-F-034 | ✅ |
| BO-S-035 | 과정 카테고리 | `/admin/content/categories` | 🤝 | BO-F-036~037 | ⚠️ 일부 미연동 |
| **통계 / 리포트** | | | | | |
| BO-S-040 | 플랫폼 현황 | `/admin/stats/platform` | ★SYS | BO-F-043 | ✅ |
| BO-S-041 | 기관 학습 현황 | `/admin/stats/institutions` | ⬢ | BO-F-040 | ✅ |
| BO-S-042 | 학습자 현황 | `/admin/stats/learners` | ALL | — | ✅ |
| BO-S-043 | 출석률 | `/admin/stats/attendance` | ⬢ | BO-F-041 | ✅ |
| BO-S-044 | B2B 학습데이터 | `/admin/b2b/results` | ★SYS | BO-F-042 | ✅ |
| **시스템** ★ | | | | | |
| BO-S-050 | 코드관리 | `/admin/system` | ★SYS | BO-F-050 | ✅ |
| BO-S-051 | 수업모듈 | `/admin/ai-modules` | ★SYS | BO-F-051 | ✅ |
| BO-S-052 | 수업모듈 코드관리 | `/admin/ai-modules/code-management` | ★SYS | BO-F-050 | ✅ |
| BO-S-053 | 스피킹 지문 프롬프트 | `/admin/speaking-passage-prompts` | ★SYS | — | ✅ |
| BO-S-054 | 시스템관리자 계정 | `/admin/system/admins` | ★SYS | BO-F-052 | ✅ |
| BO-S-055 | API키관리 | `/admin/b2b/api-keys` | ★SYS | BO-F-053 | ⚠️ 미연동(Phase G) |
| **파트너 API 연동** 🤝 | | | | | |
| BO-S-060 | API 연동 가이드 | `/admin/partner/api-guide` | 🤝 | BO-F-060 | ⚠️ Step1 |
| BO-S-061 | 내 연동 설정 | `/admin/partner/settings` | 🤝 | BO-F-061 | 🔲 Step2 |
| BO-S-062 | 연동 테스트 | `/admin/partner/test` | ★SYS | BO-F-062 | 🔲 Step3 |
| **API 로그** ★ | | | | | |
| BO-S-070 | 수강로그 | `/admin/b2b/enrollments` | ★SYS | — | ⚠️ 목업 |
| BO-S-071 | 로그인로그 | `/admin/b2b/members` | ★SYS | — | ⚠️ 목업 |

---

## 3. 화면 전환 관계

- **사이드바**(`filterByRole`): 역할별 재귀 필터링으로 접근 가능한 메뉴만 노출
- **BO-S-013 조직 상세**: partner·group = `[기본정보]` 탭만 / institution = `[기본정보]` `[수업설정]` 탭 → 목록 "수업설정" 버튼 = `?tab=수업설정`
- **BO-S-013 → BO-S-031**: `[이 기관의 과정 보기 →]` 딥링크(type별 `?partnerId|groupId|institutionId=`)
- **BO-S-031 과정 상세**: 탭 `[기본정보]` `[액세스코드(N)]`
- **BO-S-041 기관 학습 현황**: 3단 드릴다운 리스트 → 레슨 상세 → 모듈(FRT) 상세
- **BO-S-043 출석률**: 기관별 → 학습자 → 월별 캘린더 드릴다운
- **BO-S-020 사용자 → 자동로그인(handoff)**: teacher→Studio, student→Tutoring 새 탭

---

## 4. 레이아웃 & 가드

- 이중 래핑: `AdminAuthGate`(로그인 여부) → `AdminRoleGate`(role 권한, Tier1 라우트 차단 → `/admin/dashboard` 리다이렉트)
- orgContext: `GET /auth/me` → Zustand `useAuthStore` 전 페이지 공유(§6 BO-P-005)

---

## 5. 근거 문서
- `menu-redesign/20260527_메뉴구조_권한모델_개발계획.md` · `20260529_권한별-데이터-스코프-구현계획.md`
- `docs/users/20260528_계정로그인_개발계획.md`(handoff·resolveUserScope) · `docs/report/20260525_학습결과_리포트_메뉴_개발계획.md`
- `features/기관상세페이지/기관상세-탭구조-정리.md` · `features/b2b/20260527_파트너API연동_포털_개발계획.md`

> 기존 `IA_정보구조_Backoffice.md`를 본 문서(§4)로 승격했습니다.
