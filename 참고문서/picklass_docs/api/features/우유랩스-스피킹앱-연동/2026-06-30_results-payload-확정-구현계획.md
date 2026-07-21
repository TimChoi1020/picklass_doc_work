# results 페이로드 확정 — measures 키 표준 + 문항별 상세 구현계획

- 날짜: 2026-06-30
- 대상 저장소: `../../../../api.picklass.com`
- 근거 원문: [DocB PDF](file:///C:/Users/MS/Desktop/api/스피킹앱__픽클래스_연동_규격_(8개_모듈)__콘텐츠__API.pdf) §2.3(results 본문)·§3.4(모듈별 measures)·§5(체크리스트)
- 범위: [구현계획 §8 잔여 미정](./2026-06-28_구현계획.md#L117)의 **"results payload 최종 범위(measures 키 표준, 문항별 상세 포함 여부)"** 항목을 PDF 규격대로 **확정·타입화**한다.

## 0. 전제 — 미정/확정 경계

PDF §5 체크리스트 중 본 계획이 **확정**하는 것과 **보류**하는 것을 명시한다.

| 항목 | 출처 | 처리 |
|---|---|---|
| results 본문 골격(level/passRate/retakeCount/scores/results[]) | §2.3 | **확정** — 타입화 |
| measures 모듈별 키 표준 | §3.4 | **확정** — 본 문서 §1 표 |
| 문항별 상세 results[] 포함·구조 | §2.3 | **확정** — 포함, DTO 고정 |
| **대표 score 산식** | (PDF 없음) | **보류(미정)** — 현행 `scores 평균` 임시 유지 |
| **콘텐츠 형식**(오디오 출처·OMP 공인시험 환산·LRN 자막) | §5 A/B/E | **보류(미정)** — 필드 null·measures에서 제외 |

> 핵심: PDF가 주는 건 5축 점수(`scores`)와 **5축을 넘어서는 모듈별 추가 측정값**(`measures`)이다. 5축은 이미 top-level `scores`에 있으므로 `measures`에는 **모듈 고유 지표만** 담는다(중복 금지).

## 1. measures 키 표준 (확정)

§3.4 "모듈별 내부 측정값"의 지표명을 camelCase 키로 고정. 저장 위치 = `module_histories.kpis.measures`(JsonB). 값은 number(0~100 또는 0~1; **단위는 우유랩스 송신값 그대로 보존**, 우리 환산 없음).

| 모듈 | measures 키 | 의미(§3.4) | 비고 |
|---|---|---|---|
| **LRN** | `staySeconds`, `readCompletion` | 영상 머문시간(초) · 완독률 | staySeconds 정수(초), readCompletion 0~1 |
| **VLM** | — (measures 생략/null) | 측정·결과저장 제외 | §3.4 "없음(자기보고)" → results 호출 자체 선택 |
| **SHD** | — (5축만) | 5축 외 추가 없음 | scores로 충분 |
| **SFB** | `missionUsage` | 미션 사용 | |
| **SMK** | `expressionApprop` | 표현 적절성 | answer.exprApropThreshold와 대응 |
| **RPB** | `missionUsage` | 미션 사용 | |
| **RPF** | `missionUsage`, `interactionAct` | 미션 사용 · 상호작용 | |
| **OMP** | `logicStructure`, `reutterance`, `keywordUsage`, `ompFour{logic,expression,pronunciation,time}` | 논리구조·재발화·키워드 사용률·OMP 4영역 | **`officialExamConversion`(공인시험 환산)은 미정 → 제외**(§5 E) |

- 송신측이 표준 외 키를 추가로 보내도 **거부하지 않고 그대로 보존**한다(measures는 forbidNonWhitelisted 비적용, 확장 여지). 표준 키는 **Swagger·DTO 주석으로 계약 고정**.
- VLM은 §3.4상 측정 없음 → results 호출이 와도 measures=null로 저장(거부 안 함).

## 2. 문항별 상세 results[] (확정)

PDF §2.3 `results[]` 원소를 DTO로 고정. 저장 위치 = `module_histories.answers`(JsonB, 현행 유지).

```jsonc
results: [{
  questionId: string,      // 필수
  seq: number,             // 필수 (0-base)
  passed: boolean,         // 필수
  attempt: number,         // 필수 (1-base)
  retakeCycle: number,     // 필수 (0/1)
  durationMs: number,      // 필수
  pronunciation?: {        // 발화(audio-record) 모듈만. 탭(audio-listen: LRN/VLM)은 생략
    accuracy: number, fluency: number, completeness: number,
    prosody: number, score: number, errorWords: number
  }
}]
```

- `pronunciation`은 **옵셔널**(탭 모듈 LRN/VLM 생략). 존재 시 6개 키 전부 number.
- results[] 자체도 옵셔널(§2.3 "선택, 협의" → **포함으로 확정**하되 미전송 허용).

## 3. 구현 작업

### 3.1 DTO 타입화 — `src/external/dto/results.dto.ts`

현행 `scores?: Record<string,number>` / `measures?: Record<string,unknown>` / `results?: unknown[]` 를 중첩 DTO로 강화.

- `ScoresDto`: `pronunciation/fluency/grammar/appropriateness/volume` (각 `@IsOptional @IsNumber`). `forbidNonWhitelisted` 적용해 5축 외 키 차단.
- `ResultItemDto`: §2 구조. `@ValidateNested` + `@Type(() => ResultItemDto)`. `pronunciation`은 `@IsOptional @ValidateNested @Type(() => PronunciationDto)`.
- `PronunciationDto`: 6개 number 필수.
- `measures`: **타입화하지 않음**(모듈별 가변·확장). `@IsOptional @IsObject` 유지 + 클래스 주석에 §1 표를 명시. (per-module 조건부 검증은 과설계 → 계약은 문서/Swagger로 고정)
- `results`: `@IsOptional @IsArray @ValidateNested({each:true}) @Type(() => ResultItemDto)`.

> `main.ts` ValidationPipe는 이미 `whitelist/forbidNonWhitelisted/transform` 활성 → 중첩 DTO 자동 검증.

### 3.2 저장 매핑 — `external.service.ts > saveResults`

- `kpis.measures` = `dto.measures ?? null` (현행 유지, 이제 §1 키 표준 계약 하에 저장).
- `answers` = `dto.results`(타입화된 ResultItemDto[]) — 현행 유지.
- `score`(대표 점수) = **현행 `scores 평균` 그대로**(미정, 본 계획 범위 밖). 산식 확정 시 별도 1줄 교체 지점만 주석으로 표기.
- 변경 없음: course_lesson_id/student_id/module_code/completed_at.

### 3.3 Swagger 주석

`@ApiBody`로 results 예시(SHD 1건 + measures) 동봉, `ResultsDto` 필드에 §1 measures 키 표 요약을 description으로. `/docs`에서 우유랩스가 계약 확인 가능하도록.

### 3.4 검증(e2e, read-only 원복)

기존 인증 e2e 패턴(전용 테스트 토큰) 재사용:
1. 모듈별(SHD/SFB/SMK/RPB/RPF/OMP) 샘플 payload POST → `module_histories.kpis.measures` 가 §1 키로 저장되는지.
2. results[] 포함 payload → `answers`에 §2 구조로 저장되는지, `pronunciation` 누락(LRN/VLM 모사) 케이스 통과.
3. scores 외 키·measures 표준 외 키 → scores는 400(forbidNonWhitelisted), measures는 보존(통과) 확인.
4. 테스트 데이터 원복(delete).

## 4. 단계별 작업

1. `results.dto.ts`: ScoresDto/PronunciationDto/ResultItemDto 추가 + ResultsDto 중첩 검증.
2. `external.service.ts`: saveResults 주석 보강(measures 키 계약·score 산식 미정 지점 표기). 저장 로직은 사실상 유지.
3. Swagger `@ApiBody` 예시 + description.
4. build·typecheck·lint → e2e(§3.4) → 데이터 원복.
5. 문서: 본 계획 결과를 [구현계획 §8](./2026-06-28_구현계획.md)에 "measures 키 표준·문항별 상세 = 확정(2026-06-30)"으로 반영. 잔여 미정에는 **대표 score 산식·콘텐츠 형식**만 남긴다.

## 4.1 구현·검증 결과 (2026-06-30 완료)

- **구현**: `results.dto.ts` 중첩 DTO(`ScoresDto`/`PronunciationDto`/`ResultItemDto`) + `ResultsDto` 검증. `external.service.ts` saveResults 주석(score 산식 미정 지점·measures 키 계약). `external.controller.ts` `@ApiBody`(SHD/RPF/LRN 예시 + measures 키 표 description). DB·저장 경로 변경 없음(기존 kpis/answers 재사용).
- **정적 체크**: `pnpm typecheck`·`pnpm lint`·`pnpm build` 통과(exit 0).
- **정합성 체크(전역 ValidationPipe 동일 옵션 whitelist+forbidNonWhitelisted+transform, 11케이스 ALL PASS)**:
  - scores 5축 엄격(외 키·>100 거부), results[] 구조 엄격(외 키·잘못된 타입·pronunciation 부분누락 거부), pronunciation 옵셔널(LRN 탭 통과), measures 자유 키 보존(확장 허용), 필수 누락·passRate>1 거부.
- **체크리스트(PDF §5) 매핑**: "결과 페이로드 — measures 포함 범위·키 이름, 문항별 상세 포함 여부" → **확정·구현 완료**. (나머지 §5 A/B/E 콘텐츠·score 산식은 §5 미정 유지)
- **라이브 DB e2e**: results 저장 왕복(kpis/answers)은 2026-06-28 세션에서 검증 완료, 본 변경은 저장 경로 불변 + DTO 검증층만 추가 → 정합성 테스트로 커버.

## 5. 남는 미정(본 계획 이후)

- **대표 score 산식**: `module_histories.score`(int) 채움 규칙. 현재 scores 평균. → 우유랩스/내부 협의.
- **콘텐츠 형식**: LRN videoUrl·자막(§5 A), 오디오 음원 출처 TTS vs 파일(§5 B), OMP 공인시험 환산식(§5 E).
- measures **값 단위·범위**(0~100 vs 0~1) 표준: 현재 송신값 보존만. 필요 시 우유랩스 샘플 수신 후 확정.
