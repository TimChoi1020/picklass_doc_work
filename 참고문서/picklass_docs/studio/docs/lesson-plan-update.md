# lesson-plan.md 업데이트 명세

**작성일**: 2026-04-22  
**근거 문서**: `20260421_지능형 수업 설계 자동화 로직.md`  
**대상 문서**: `lesson-plan.md`  
**범위**: 아래 3개 항목에 대한 기획 → 구현 갭 보완

---

## 1. 모듈 풀 부족 대응 방식 (Fallback Chain)

### 현재 상태 (`lesson-plan.md`)

- S2 필터 결과 모듈이 없거나 S3 Set Cover 후 KPI가 미커버되면 `UNCOVERED_KPI` warning 발행 후 부분 시퀀스 반환.
- 레벨 범위 자동 확장 없음. 엔진이 스스로 조건을 완화하는 로직 부재.

### 기획 요구사항 (`20260421_`)

S2 필터 전에 단계적 조건 완화 체인을 실행하여, 충분한 모듈 풀이 확보될 때까지 자동으로 조건을 넓힌다. 폴백으로 추가된 모듈은 응답에서 구분 표시.

```
1차 시도: 선택 KPI + 정확 레벨 매칭
    ↓ 권장 최소 모듈 수(RECOMMENDED_MIN_MODULES) 미달 시
2차 시도: 선택 KPI + 레벨 범위 ±1 확장
    ↓ 여전히 부족하면
3차 시도: 동일 skill 도메인, KPI 무관, 정확 레벨
    ↓ 여전히 부족하면
4차 시도: 동일 skill 도메인, KPI 무관, 레벨 ±1 확장
    ↓ 그래도 부족하면
가능한 만큼 반환 + FALLBACK_EXHAUSTED warning
```

### lesson-plan.md 에 추가/수정할 내용

#### § 4.3 S2 — Module Filter 수정

기존 단일 필터 함수를 폴백 체인 오케스트레이터로 교체.

```python
RECOMMENDED_MIN_MODULES = 4  # 권장 최소 모듈 수

def filter_with_fallback(all_modules, profile) -> tuple[list, str]:
    """
    단계적 폴백 체인으로 충분한 모듈 풀 확보.
    반환: (필터된 모듈 목록, 사용된 폴백 단계)
    """
    # 1차: 선택 KPI + 정확 레벨
    result = filter_modules(all_modules, profile, level_expand=0, kpi_required=True)
    if len(result) >= RECOMMENDED_MIN_MODULES:
        return result, "exact"

    # 2차: 선택 KPI + 레벨 ±1 확장
    result = filter_modules(all_modules, profile, level_expand=1, kpi_required=True)
    if len(result) >= RECOMMENDED_MIN_MODULES:
        warnings.add("FALLBACK_LEVEL_EXPANDED")
        return result, "level_expanded"

    # 3차: skill 도메인만, 정확 레벨
    result = filter_modules(all_modules, profile, level_expand=0, kpi_required=False)
    if len(result) >= RECOMMENDED_MIN_MODULES:
        warnings.add("FALLBACK_KPI_IGNORED")
        return result, "kpi_ignored"

    # 4차: skill 도메인만, 레벨 ±1 확장
    result = filter_modules(all_modules, profile, level_expand=1, kpi_required=False)
    if result:
        warnings.add("FALLBACK_LEVEL_EXPANDED")
        warnings.add("FALLBACK_KPI_IGNORED")
        return result, "full_fallback"

    # 완전 소진
    warnings.add("FALLBACK_EXHAUSTED")
    return [], "exhausted"


def filter_modules(modules, profile, level_expand=0, kpi_required=True) -> list:
    p_idx = CEFR_TO_INDEX[profile.passage_level]
    result = []
    for m in modules:
        m_min = int(m.suitable_level_min.split("_")[1]) - level_expand
        m_max = int(m.suitable_level_max.split("_")[1]) + level_expand
        level_ok = m_min <= p_idx <= m_max
        skill_ok = m.skill in profile.skill_filter
        kpi_ok = (not kpi_required or
                  bool(set(m.selected_kpi_codes) & profile.selected_kpi_codes))
        if level_ok and skill_ok and kpi_ok:
            result.append(m)
    return result
```

#### § 2.2 Response 수정

`sequence` 각 항목에 `is_fallback_module` 플래그 추가.

```json
{
  "order": 3,
  "module_code": "RRD",
  "is_fallback_module": true,          // 폴백으로 추가된 모듈 (KPI 미매칭)
  "fallback_reason": "kpi_ignored"     // 폴백 단계명
}
```

`meta` 에 폴백 정보 추가.

```json
"meta": {
  "fallback_stage": "level_expanded",  // 최종 사용된 폴백 단계 ("exact" | "level_expanded" | "kpi_ignored" | "full_fallback" | "exhausted")
  "fallback_module_count": 1           // 폴백으로 추가된 모듈 수
}
```

#### § 5 엣지 케이스 정책 추가

| 상황 | HTTP | 동작 |
|---|---|---|
| 권장 최소 모듈 수 미달, 레벨 ±1 확장으로 해소 | 200 | `FALLBACK_LEVEL_EXPANDED` warning, `is_fallback_module=false` (KPI는 유지) |
| 레벨 확장으로도 부족, KPI 조건 해제로 해소 | 200 | `FALLBACK_KPI_IGNORED` warning, 폴백 모듈은 `is_fallback_module=true` |
| 4차 시도 후에도 모듈 0개 | 200 | `FALLBACK_EXHAUSTED` warning, `sequence:[]` |

---

## 2. KPI 선택 구조

### 현재 상태 (`lesson-plan.md`)

- `selected_kpi_codes`는 analyzer 자체 설계의 **Layer 1 영문 코드 배열** (예: `s_fluency`, 26개).
- Layer 2(한글 10개)와 Layer 1(영문 26개) 간 KPI_ROLLUP 매핑이 필요한 2계층 구조.
- 백오피스의 `KPI_CATEGORY` 코드 체계와 **별개로 설계**되어 있어 불필요한 복잡도 발생.

### 문제 및 개선 방향

**Layer 1/2 구조는 불필요하다.**

백오피스에 이미 `KPI_CATEGORY` code group이 존재하며, `AiModule.selectedKpiCodes`는 이 코드를 직접 저장하고 있다. analyzer가 독자적으로 설계한 Layer 1/2 체계 대신, **백오피스 `KPI_CATEGORY`를 단일 코드 체계로 사용**하면 롤업 매핑 없이 동일한 코드로 UI 선택 → 모듈 매칭이 직결된다.

```
백오피스 KPI_CATEGORY 구조:
  code          → KPI 식별자 (예: "ERR_ANALYSIS")
  extraData.goal → 목표 설명 (예: "오류 인식 및 교정")
  name          → 측정 항목 (예: "문장 복잡도 성장")

AiModule.selectedKpiCodes → ["ERR_ANALYSIS", "FLUENCY_RATE", ...]
                              ↑ 동일한 KPI_CATEGORY.code 사용

기획 UI에서 KPI 선택:
  → KPI_CATEGORY 목록에서 extraData.goal 기준 중복 제거(distinct) 후 목표만 표시 (코드 미노출)
  → 선택한 code 배열 → selected_kpi_codes로 전달
  → S3 Set Cover: AiModule.selectedKpiCodes ∩ selected_kpi_codes
  → 동일 코드 체계이므로 롤업 변환 불필요
```

### lesson-plan.md 에 추가/수정할 내용

#### § 3.1 KPI 2계층 구조 → 단일 구조로 교체

기존 Layer 1/2 설명 전체를 아래로 교체.

> **KPI 코드 체계**: 백오피스 `KPI_CATEGORY` code group을 단일 소스로 사용한다. `ai_modules.selectedKpiCodes`에 저장된 코드와 UI에서 선택하는 코드가 동일한 체계이므로 별도 롤업 매핑이 필요 없다.
>
> `KPI_CATEGORY` 항목 구조:
> - `code`: KPI 식별자 — `selected_kpi_codes`에 사용되는 값 (UI에는 미노출)
> - `extraData.goal`: 목표 설명 — UI 선택 화면에 단독 노출 (goal 기준 distinct 처리)
> - `name`: 측정 항목 — UI 노출 제외

#### § 2.1 Request 수정

`selected_kpi_codes`를 백오피스 `KPI_CATEGORY.code` 배열로 재정의.

```json
{
  "passage_level": "C1",
  "selected_kpi_codes": ["ERR_ANALYSIS", "FLUENCY_RATE", "VOCAB_DEPTH"],  // KPI_CATEGORY.code 배열
  "duration_min": 30,
  "skill_filter": ["speaking", "vocabulary"]
}
```

#### § 4.1 S1 — Input Normalize 수정

Layer 롤업 로직 제거. `KPI_CATEGORY` 코드 유효성 검증만 수행.

```python
def normalize_kpi_codes(raw_codes: list[str]) -> set[str]:
    """
    백오피스 KPI_CATEGORY.code 유효성 검증.
    Layer 롤업 없음 — 단일 코드 체계.
    """
    valid_codes = load_kpi_category_codes()  # KPI_CATEGORY group의 code 집합
    unknown = [k for k in raw_codes if k not in valid_codes]

    if unknown:
        warnings.add(f"UNKNOWN_KPI_CODES:{unknown}")

    return {k for k in raw_codes if k in valid_codes}
```

#### § 3.2 필요 code groups 수정

Layer 1/2 관련 항목 정리.

| group_code | v3.1.0 필요성 | 상태 |
|---|---|---|
| `KPI_CATEGORY` | **필수** — UI 멀티셀렉트 소스 + `selected_kpi_codes` 유효성 검증 | ✅ 기존 (백오피스) |
| `KPI_ITEMS` (영문 26개) | ⬛ **제거** — `KPI_CATEGORY`로 대체 | 드롭 |
| `KPI_ROLLUP` | ⬛ **제거** — 단일 코드 체계로 불필요 | 드롭 |

#### § 2.3 부수 엔드포인트 수정 — `/kpi-catalog` 응답 단순화

Layer 구분 제거. `KPI_CATEGORY` 항목을 그대로 반환하되, 현재 조건의 가용 모듈 수를 포함.

```
GET /lesson-plan/kpi-catalog?passage_level=C1&skill_filter=speaking,vocabulary
```

```json
{
  "kpis": [
    {
      "code": "FLUENCY_RATE",
      "goal": "말하기 유창성 향상",
      "available_module_count": 5    // 현재 passage_level + skill_filter 조건에서 매칭 모듈 수
    },
    {
      "code": "ERR_ANALYSIS",
      "goal": "오류 인식 및 교정",
      "available_module_count": 2
    }
  ],
  "recommended_min_modules": 4      // 권장 최소 모듈 수 (UI 경고 기준)
}
```

`available_module_count`가 `recommended_min_modules` 미만이면 studio UI가 노란 경고, 0이면 빨간 차단으로 표시.

---

## 3. 수업 단계(Intro/Body/Closure) 결정 방식 — cognitive_level 활용

### 현재 상태 (`lesson-plan.md`)

- S5에서 `cognitive_level` 전역 정렬이 주 기준. `class_*` 플래그는 `cognitive_level`이 null일 때만 폴백으로 사용.
- 관리자가 명시적으로 설정한 `classBefore/classMiddle/classAfter`가 사실상 무시되는 구조.

### 개선 방향

`class_*` 플래그와 `cognitive_level`의 역할을 재정의한다.

```
1순위: class_* 플래그  → 모듈이 배치될 단계(Intro/Body/Closure) 결정 (관리자 의도)
2순위: cognitive_level → 같은 단계 내 순서 결정 (인지 부하 곡선)
3순위: priority        → 동일 cognitive_level 내 우선순위
```

**단일 플래그 모듈** (`classBefore` OR `classMiddle` OR `classAfter` 중 하나만 true):
→ 해당 플래그가 단계를 확정. 단계 내 순서는 `cognitive_level ASC → priority DESC`.

**다중 플래그 모듈** (예: `classBefore=true AND classMiddle=true`):
→ `cognitive_level`로 해당 단계 결정 (`1-2=intro`, `3-4=body`, `5-6=closure`).
→ 동일 `cognitive_level` 내 순서는 `priority DESC`.

이 방식의 핵심: `class_*` 플래그는 **관리자의 커리큘럼 설계 의도**를, `cognitive_level`은 **단계 내 인지 흐름**을, `priority`는 **운영 우선순위**를 각각 담당한다.

### lesson-plan.md 에 추가/수정할 내용

#### § 3.2.1 Bloom's Taxonomy 정의에 설계 원칙 추가

`Intro/Body/Closure 유추 규칙` 항목을 아래로 교체:

> **단계 결정 원칙**:
> - `class_*` 플래그가 **1순위** — 관리자가 설정한 단계 배치를 우선 존중한다. 단일 플래그 모듈은 해당 단계로 확정.
> - 다중 플래그 모듈은 `cognitive_level`로 적합한 단계를 결정하고, `priority`로 동순위를 해소한다.
> - `cognitive_level`은 **2순위** — 같은 단계 안에서 인지 부하가 낮은 모듈이 먼저 배치된다 (Bloom's Taxonomy 오름차순).
> - `priority`는 **3순위** — 동일 `cognitive_level` 내 운영팀 설정 우선순위.
>
> 수업 단계별 모듈 **개수는 미리 정하지 않는다**. `class_*` 플래그 배분과 시간 제약이 조합을 결정하며, 그 결과가 자연스러운 수업 흐름을 형성한다.

#### § 4.6 S5 — Stage Infer 수정

단일/다중 플래그 분기 처리 및 `cognitive_level` 기반 다중 플래그 해소 로직 추가.

```python
COGNITIVE_STAGE_BOUNDARIES = {
    "intro":   (1, 2),
    "body":    (3, 4),
    "closure": (5, 6),
}

def infer_stage(module) -> str:
    flags = []
    if module.class_before: flags.append("intro")
    if module.class_middle: flags.append("body")
    if module.class_after:  flags.append("closure")

    if len(flags) == 0:
        # 플래그 미설정: cognitive_level로 결정, 없으면 body 폴백
        return _stage_from_cognitive_level(module.cognitive_level) or "body"

    if len(flags) == 1:
        # 단일 플래그: 관리자 의도 확정
        return flags[0]

    # 다중 플래그: cognitive_level로 해소
    cl_stage = _stage_from_cognitive_level(module.cognitive_level)
    if cl_stage and cl_stage in flags:
        return cl_stage  # cognitive_level이 가리키는 단계가 허용된 단계 중 하나면 채택

    # cognitive_level이 없거나 플래그 범위 밖: priority 높은 단계 순으로 폴백
    # (closure > body > intro 순 — 마무리 성격이 가장 명확)
    for stage in ["closure", "body", "intro"]:
        if stage in flags:
            return stage

def _stage_from_cognitive_level(cl) -> str | None:
    if cl is None:
        return None
    for stage, (lo, hi) in COGNITIVE_STAGE_BOUNDARIES.items():
        if lo <= cl <= hi:
            return stage
    return None
```

#### § 4.7 S6 — Cognitive Sort 수정

전역 `cognitive_level` 단일 정렬에서 **단계별 정렬**로 변경.

```python
STAGE_ORDER = {"intro": 0, "body": 1, "closure": 2}

def sort_sequence(modules) -> list:
    # cognitive_level 결손 모듈 일괄 감지
    missing_cl = [m.code for m in modules if m.cognitive_level is None]
    if missing_cl:
        warnings.add(f"MISSING_COGNITIVE_LEVEL:{missing_cl}")

    # 정렬:
    #   1순위: 단계 순서 (intro=0, body=1, closure=2)
    #   2순위: cognitive_level ASC (단계 내 인지 흐름)
    #   3순위: priority DESC (동일 cognitive_level 내 운영 우선순위)
    #   4순위: code ASC (결정성 보장)
    primary = sorted(
        modules,
        key=lambda m: (
            STAGE_ORDER.get(m.inferred_stage, 1),
            m.cognitive_level or 99,
            -m.priority,
            m.code,
        )
    )
    return topological_sort_with_preserved_order(primary)
```

#### § 2.2 Response 수정 — stage_counts 추가

```json
"summary": {
  "cognitive_arc": [1, 2, 3, 4],
  "cognitive_arc_stages": ["intro", "intro", "body", "body"],
  "stage_counts": { "intro": 2, "body": 2, "closure": 0 },  // 단계별 모듈 수 (사전 고정 아님, 결과값)
  "missing_cognitive_level_count": 0
}
```

#### § 2.3 `/health` 엔드포인트 수정

```json
{
  "status": "ok",
  "cognitive_level_fill_rate": 0.72,
  "cognitive_level_missing_codes": ["PWR", "SCN"],
  "multi_flag_module_count": 3          // 다중 플래그로 인해 cognitive_level로 단계 해소된 모듈 수
}
```

---

## 우선순위 및 작업 순서

| 순서 | 항목 | 선행 조건 |
|---|---|---|
| 1 | `cognitive_level` DB 컬럼 추가 + 기존 모듈 시드값 적재 | Prisma 마이그레이션 |
| 2 | S5/S6 cognitive_level 기반 정렬 안정화 + `/health` 적재율 모니터링 | 1번 완료 |
| 3 | S1 `normalize_kpi_codes` — `KPI_CATEGORY` 코드 체계로 교체 (`KPI_ITEMS` 드롭) | 백오피스 `KPI_CATEGORY` 시드 확인 |
| 4 | `/kpi-catalog` 응답 단순화 — Layer 구분 제거, `available_module_count` 추가 | 3번 완료 |
| 5 | S2 폴백 체인 (`filter_with_fallback`) 구현 | 2번 완료 |
| 6 | Response에 `is_fallback_module`, `fallback_stage`, `stage_counts` 추가 | 5번 완료 |
