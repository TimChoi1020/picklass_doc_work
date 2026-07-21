# Studio 메뉴 구조 마스터 문서

> **작성일**: 2026-06-05
> **최종 수정**: 2026-06-05
> **단일 진실의 원천**: 이 문서가 studio.picklass.com 메뉴 구조의 기준.
> 새 메뉴 추가·변경·삭제 시 이 문서를 먼저 수정한다.
> **코드 위치**: `studio.picklass.com3/apps/web/src/components/oizi/StudioHeader.tsx`

---

## 수정 이력

| 회차 | 일자 | 변경 내용 |
|:---:|------|----------|
| 1 | 2026-06-05 | 초안 작성 — 현재 구현 실측 + Enrollment 신설·CourseSidebar 재편 계획 반영 |

---

## 1. 설계 원칙

- **헤더 메뉴**: 최상위 업무 단위. 강사의 작업 흐름 순서대로 배치.
- **사이드바**: 해당 메뉴 내부 탐색 전용. 다른 메뉴의 업무를 혼재시키지 않는다.
- **영문 명칭**: 헤더 메뉴명은 영문 사용 (`My Library`, `Course Hub`, `Enrollment`, `Class Report`).
- **업무 흐름 순서**:
  ```
  지문 만들기 → 과정 구성 → 수강자 배정 → 결과 확인
  My Library    Course Hub   Enrollment    Class Report
  ```

---

## 2. 헤더 네비게이션

### 2.1 현재 구현 상태

```
My Library  |  Course Hub  |  Class Report
```

| 메뉴 | 경로 | 상태 |
|------|------|------|
| My Library | `/class` | ✅ 구현 |
| Course Hub | `/course` | ✅ 구현 |
| Class Report | `/report` | ✅ 구현 |

### 2.2 목표 구조

```
My Library  |  Course Hub  |  Enrollment  |  Class Report
```

| 메뉴 | 경로 | 상태 |
|------|------|------|
| My Library | `/class` | ✅ 구현 |
| Course Hub | `/course` | ✅ 구현 |
| **Enrollment** | `/enrollment` | 🔲 미구현 (신설 예정) |
| Class Report | `/report` | ✅ 구현 |

---

## 3. 메뉴별 상세 구조

### 3.1 My Library — `/class`

강사가 수업에 사용할 지문(텍스트)을 보관·관리하는 공간.

**사이드바 (`My Library` 고정)**

| 항목 | 경로 | 상태 |
|------|------|------|
| 전체 보기 | `/class` | ✅ |
| My folder | `/class` (필터) | ✅ (UI만, 실제 폴더 기능 미구현) |
| 새 폴더 만들기 | — | 🔲 미구현 |

**하위 페이지**

| 페이지 | 경로 | 상태 |
|--------|------|------|
| 지문 목록 | `/class` | ✅ |
| 수업 준비 | `/class/lesson-setup/[passageId]` | ✅ |
| 수업 진행 | `/class/lesson/[passageId]` | ✅ |

---

### 3.2 Course Hub — `/course`

과정을 생성·편집·관리하는 공간.

**사이드바 (현재)**

| 항목 | 경로 | 상태 |
|------|------|------|
| 모든 과정 | `/course` | ✅ |
| 학생 아이디 관리 | `/course/students` | ✅ → Enrollment로 이동 예정 |
| 액세스코드 관리 | `/course/accesscode` | ✅ → Enrollment로 이동 예정 |

**사이드바 (목표)**

| 항목 | 경로 | 상태 |
|------|------|------|
| 전체 보기 | `/course` | ✅ |
| 카테고리 트리 (L1/L2) | `/course` (필터) | 🔲 미구현 (Phase 4 예정) |
| 미분류 | `/course` (필터) | 🔲 Phase 4 연동 시 추가 |

> 학생 아이디 관리·액세스코드 관리는 `Enrollment` 메뉴로 이동. CourseSidebar는 과정 탐색에만 집중.

**하위 페이지**

| 페이지 | 경로 | 상태 |
|--------|------|------|
| 과정 목록 | `/course` | ✅ |
| 과정 상세·편집 | `/course/[courseId]` | ✅ |

---

### 3.3 Enrollment — `/enrollment` (**신설 예정**)

강사가 수강자를 등록하고 과정 접근 코드를 관리하는 공간.

**사이드바**

| 항목 | 경로 | 상태 |
|------|------|------|
| Students | `/enrollment/students` | 🔲 (현재 `/course/students`에서 이동) |
| Access Codes | `/enrollment/access-codes` | 🔲 (현재 `/course/accesscode`에서 이동) |

**하위 페이지**

| 페이지 | 경로 | 기존 경로 | 상태 |
|--------|------|----------|------|
| Students | `/enrollment/students` | `/course/students` | 🔲 경로 이동 |
| Access Codes | `/enrollment/access-codes` | `/course/accesscode` | 🔲 경로 이동 |

> 기존 경로(`/course/students`, `/course/accesscode`)는 redirect로 하위 호환 유지.

---

### 3.4 Class Report — `/report`

수업 결과 및 학습 현황 리포트.

**하위 페이지**

| 페이지 | 경로 | 상태 |
|--------|------|------|
| 리포트 목록 | `/report` | ✅ |
| 학습자 상세 리포트 | `/report/[studentId]` | ✅ |

---

## 4. 전체 라우트 맵

```
/                               홈 (로그인 후 리다이렉트)
/login                          로그인
/signup                         회원가입
/legal/[document]               법적 문서

/class                          My Library — 지문 목록
/class/lesson-setup/[passageId] 수업 준비
/class/lesson/[passageId]       수업 진행

/course                         Course Hub — 과정 목록
/course/[courseId]              과정 상세·편집
/course/students                → /enrollment/students 로 redirect 예정
/course/accesscode              → /enrollment/access-codes 로 redirect 예정

/enrollment                     Enrollment (신설) — 인덱스
/enrollment/students            학생 아이디 관리
/enrollment/access-codes        액세스코드 관리

/report                         Class Report — 리포트 목록
/report/[studentId]             학습자 상세 리포트
```

---

## 5. 미구현 항목 (우선순위순)

| 우선순위 | 항목 | 작업 내용 | 관련 문서 |
|:-------:|------|----------|----------|
| 🔴 | **Enrollment 메뉴 신설** | StudioHeader에 링크 추가, `/enrollment` 라우트 생성, 기존 페이지 이동, redirect 처리 | — |
| 🔴 | **CourseSidebar 카테고리 트리** | L1/L2 폴더 UI, 과정 목록 필터 연동 | [`studio/features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md`](../features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md) |
| 🔴 | **과정 편집 페이지 카테고리 배정** | L1/L2 Select UI, PATCH 저장 | 동 위 |
| 🟡 | **CourseSidebar 정리** | 학생·액세스코드 항목 제거 (Enrollment 이동 후) | — |
| 🟡 | My Library 폴더 기능 | 실제 폴더 생성·이동 기능 | — |

---

## 6. 변경 시 체크리스트

새 메뉴를 추가하거나 기존 메뉴를 변경할 때:

- [ ] 이 문서(§2~§5) 업데이트
- [ ] `StudioHeader.tsx` 링크 추가·수정
- [ ] active 조건(`isXxxPage`) 추가
- [ ] 라우트 폴더 및 `page.tsx` 생성
- [ ] 기존 경로 → 새 경로 redirect 처리 (경로 변경 시)
- [ ] 이 문서 §8 수정 이력에 기록

---

## 7. 관련 문서

- 과정 카테고리 CourseSidebar 연동: [`studio/features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md`](../features/과정카테고리/20260605_과정카테고리-2depth-개발계획.md)
- IA 구조 정리: [`studio/architecture/2026-06-03_스피킹앱-IA구조정리-개발계획.md`](./2026-06-03_스피킹앱-IA구조정리-개발계획.md)
