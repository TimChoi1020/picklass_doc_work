# useKpiItems — ai_modules 불필요한 의존 및 역방향 매핑 문제

| 발생일 | 영향 | 해결 |
|--------|------|------|
| 2026-06-07 | studio 과정 생성 시 KPI 목록 로드에 ai_modules 쿼리 추가 발생 | SKILL_CODE_MAP 도입 + ai_modules 의존 제거 |

## 증상 (Symptom)

`studio.picklass.com3` 과정 생성 페이지(`apps/web/src/hooks/use-modules.ts`)의 `useKpiItems`가
KPI 목록을 불러올 때 **불필요하게 `ai_modules` 전체를 조회**하고, `selected_kpi_codes`를 역방향으로
순회해 `skills[]`를 조립한다.

- 과정목표 칩 UI에 필요한 데이터는 `KPI_CATEGORY` 하나로 충분
- 그럼에도 `useModulesList()`(→ `GET /modules`)가 추가로 실행됨
- `modulesData`가 로드되기 전까지 `skills: []`인 채로 렌더링됨 (필터 오작동 가능)

## 원인 (Root Cause)

`KPI_CATEGORY` 코드 항목의 `extra_data.skill` 필드로 스킬 정보를 직접 얻을 수 있음에도,
이를 간과하고 `ai_modules.selected_kpi_codes` 역방향 매핑으로 `skills[]`를 조립하는 방식을 택했다.

```ts
// 기존 — ai_modules 전체를 받아서 역방향 순회
const kpiToSkills = new Map<string, Set<string>>();
for (const mod of modulesData.data) {
  for (const kpiCode of mod.selected_kpi_codes ?? []) {
    kpiToSkills.get(kpiCode)!.add(mod.skill);  // 역방향 매핑
  }
}
```

추가로, `extra_data.skill`은 **단일 문자** (`'r'`, `'s'`)이고
`ai_modules.skill`은 **풀네임** (`'reading'`, `'speaking'`)이어서
UI 상수(`SKILL_SORT_ORDER`, `SKILL_ABBR`)와 포맷이 달랐다.
이 포맷 불일치가 직접 사용을 어렵게 만들어 역방향 매핑 방식을 선택한 원인이 됐다.

## 해결 (Resolution)

`SKILL_CODE_MAP`을 추가해 단일문자 → 풀네임 변환을 처리하고,
`queryFn` 내 goal 그룹핑 시 `skills`를 직접 수집한다.
`useModulesList()` 호출과 `select` 콜백을 제거한다.

```ts
// 추가할 변환 맵
const SKILL_CODE_MAP: Record<string, string> = {
  r: 'reading',
  l: 'listening',
  s: 'speaking',
  w: 'writing',
  v: 'vocabulary',
};

export function useKpiItems() {
  // useModulesList() 호출 제거

  return useQuery({
    queryKey: queryKeys.commonCodes.items('KPI_CATEGORY'),
    queryFn: async () => {
      const items = await commonCodesApi.getItemsByGroupCode('KPI_CATEGORY');
      const active = items
        .filter((item) => item.is_active)
        .sort((a, b) => a.sort_order - b.sort_order);

      const groupOrder: string[] = [];
      const grouped: Record<string, { goal: string; codes: string[]; skills: Set<string> }> = {};

      for (const item of active) {
        const goal = ((item.extra_data?.goal as string) || item.name || '').trim();
        if (!goal) continue;

        const skillChar = item.extra_data?.skill as string | undefined;
        const skill = skillChar ? (SKILL_CODE_MAP[skillChar] ?? skillChar) : undefined;

        if (!grouped[goal]) {
          grouped[goal] = { goal, codes: [], skills: new Set() };
          groupOrder.push(goal);
        }
        if (!grouped[goal].codes.includes(item.code)) {
          grouped[goal].codes.push(item.code);
        }
        if (skill) grouped[goal].skills.add(skill);
      }

      return groupOrder.map((goal): KpiItem => ({
        code: grouped[goal].codes[0]!,
        goal,
        codes: grouped[goal].codes,
        skills: Array.from(grouped[goal].skills),
      }));
    },
    staleTime: 5 * 60 * 1000,
    // select 콜백 제거
  });
}
```

### 동작 변화

| 항목 | 기존 | 변경 후 |
|---|---|---|
| 쿼리 수 | 2개 (KPI + modules) | 1개 (KPI만) |
| skills 로딩 타이밍 | modules 로드 완료 후 | KPI 로드 즉시 완성 |
| 모듈 없는 KPI | skills:[] → 필터 제외 | extra_data.skill 기반 → 필터 포함 |

모듈 없는 KPI가 필터에 포함되는 동작 변화는 **의도된 것**이다.
KPI 카탈로그에 정의된 KPI는 모듈 구현 여부와 무관하게 과정목표 선택지에 노출되는 것이 맞다.

## 재발 방지 (Prevention)

- `code_items.extra_data` 스키마(`kpi-catalog.md` 참고)를 먼저 확인한다.
  필요한 정보가 이미 `extra_data` 안에 있으면 외부 테이블 역방향 조회를 추가하지 않는다.
- `extra_data`의 포맷(단일문자 vs 풀네임 등)이 UI 상수와 다를 경우,
  역방향 매핑 대신 **변환 맵(SKILL_CODE_MAP 패턴)** 을 추가해 직접 변환한다.
- KPI 관련 훅 작성 시 체크리스트:
  - [ ] `KPI_CATEGORY.extra_data` 에서 필요한 값을 직접 얻을 수 있는가?
  - [ ] 외부 테이블을 추가로 조회해야 한다면, 그 이유가 명확한가?
  - [ ] 포맷 불일치가 있다면 변환 맵으로 해결하는가?

## 참고

- 수정 대상 파일: `studio.picklass.com3/apps/web/src/hooks/use-modules.ts`
- KPI 카탈로그 (extra_data 스키마): `picklass_docs/shared/conventions/kpi-catalog.md`
