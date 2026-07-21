# 지문 난이도 분석기 연동 설계 (analyzer.picklass.com)

작성일: 2026-04-26
대상 시스템: tutoring.picklass.com (Next.js Web + NestJS API + Supabase Postgres)
연동 외부 API: https://analyzer.picklass.com (`POST /analyze`)
원본 설계: `studio.picklass.com/docs/analyzer.md` (이번 문서는 그 이식판)

> **이식 원칙**
> - DB는 **studio와 같은 Supabase 인스턴스**를 공유한다(`apps/api/prisma/schema.prisma:13` 주석 참조). 즉 `text_analyses` 테이블이 studio 측 작업으로 **이미 운영 DB에 존재**하므로, tutoring 쪽에서는 **테이블/RLS를 새로 만들지 않고 Prisma 모델만 introspect/추가**하여 사용한다.
> - tutoring은 사용자(학생)가 명시적인 "지문 생성 모달"을 거치지 않는다. 모든 지문은 "나만의 수업"(`POST /lessons/create-custom`)에서 자동 생성되며 `texts.text_type` = `'A'`만 발생한다. 따라서 studio 문서의 `I`(직접 입력)·`T`(쌍둥이) 분기와 권위 분석기 level 보강 로직은 **본 프로젝트에서는 제거**한다.
> - 이미 호출되고 있는 Gemini 기반 `analyzePassage`(`apps/api/src/ai/ai.service.ts:236`) 결과도 동일 테이블에 `analyzer_type='gemini-passage'`로 함께 저장하여, 정량(picklass-cefr)·정성(gemini-passage) 두 분석기를 한 곳에서 누적한다.

---

## 1. 배경 및 목표

### 1.1 현재 상태
- **지문 생성/저장**: `LessonsService.createCustomLesson` (`apps/api/src/lessons/lessons.service.ts:606-691`)
  1. `aiService.generatePassage` → 지문 생성(Gemini, L606)
  2. `prisma.texts.create` → `texts` 테이블에 저장 (L613, `text_type='A'`, `analysis` 컬럼 NULL)
  3. `aiService.analyzePassage` → Gemini 정성 분석(L626) — **결과를 응답 본문에만 실어 보내고 DB에 저장하지 않음**
  4. `lessonPlanService.generate` → 외부 analyzer `${ANALYZER_BASE_URL}/lesson-plan` 호출(L633) — 모듈 시퀀싱 전용
- **분석기 정량 호출(`POST /analyze`) 없음**: `lesson-plan.service.ts`는 analyzer 도메인을 사용하지만 `/lesson-plan` 엔드포인트만 호출. 8대 CEFR 지표는 현재 어디서도 산출되지 않는다.
- **Prisma `texts.analysis` JSONB 컬럼**: 스키마에는 있으나(`schema.prisma:56`) 어디서도 read/write 하지 않는 dead column.
- **분석 결과 표시 UI 없음**: tutoring 웹은 학생 풀이 화면(`apps/web/src/app/lessons/[lessonId]/...`)이 메인이며, studio처럼 "지문 자세히 보기 모달"이 없다. 분석 결과 노출 위치는 별도 결정 필요(§10.1).

### 1.2 목표
1. 지문 생성 시점에 **외부 analyzer 정량 분석(`POST /analyze`)을 자동 호출**하여 `text_analyses` 테이블에 저장.
2. 기존 Gemini 정성 분석(`analyzePassage`) 결과도 같은 테이블에 `analyzer_type='gemini-passage'`로 저장하여 createCustom 응답과 별개로 영속화.
3. 추후 학생/강사용 분석 표시 UI를 추가할 때 곧바로 `text_analyses`를 조회하면 되도록 데이터 인프라를 먼저 갖춘다(이번 PR은 표시 UI까지는 포함하지 않음 — §10.1).

### 1.3 비목표 (studio 문서와의 차이)
- ❌ `texts.level` 자동 보강 로직(studio §4.3 표): tutoring은 `text_type='A'`만 발생하고 사용자가 선택한 CEFR 인덱스가 이미 `parseLevelToIndex` 흐름으로 결정됨(`lessons.service.ts:592-598`). 분석기 결과로 level을 덮지 않는다.
- ❌ 쌍둥이/직접입력 분기.
- ❌ 본 PR에서는 web UI 변경 없음(타입 정의·노멀라이저는 미리 추가).

---

## 2. 외부 API 스펙

studio 문서 §2 와 동일하므로 요점만:

```
POST https://analyzer.picklass.com/analyze
Content-Type: application/json
Body: { "text": "<영어 본문>", "use_llm": false }
```

응답에서 본 프로젝트가 의미 있게 사용하는 키: `final_level`, `level_score`, `levels.*`(8대 지표), `scores.*`, `weighted_details.*`, `metrics.*`, `text_info`, `analysis_summary`, `difficulty_group`, `rough_level`, `boundary`, `words[]`. 응답 전체를 `result` JSONB에 그대로 저장한다.

8대 지표 ↔ 라벨 매핑(어댑터/노멀라이저 책임): studio 문서 §2.3 표 그대로 재사용.

---

## 3. 데이터베이스

### 3.1 공유 Supabase 인스턴스 — 신규 DDL 미적용

studio·tutoring 두 프로젝트는 **같은 Supabase Postgres**(`postgres.gwccmvkjxohtymujtgfm` / `aws-0-ap-northeast-2`)를 공유한다(`apps/api/.env`의 `DATABASE_URL` 동일). studio 측에서 2026-04-26에 `text_analyses` 테이블 + 인덱스 6개 + RLS 정책 3개를 운영 DB에 직접 적용 완료한 상태이므로, **tutoring에서는 DDL을 다시 실행하지 않는다**.

확인이 필요한 항목 (착수 전 1회):
- 운영 DB에 `text_analyses` 테이블이 실제로 존재하는지 직접 점검(`\d text_analyses`).
- RLS 정책이 `texts.user_id = auth.uid()` 기반이므로, tutoring API의 호출 컨텍스트(`Service Role` 키 사용 vs anon JWT)에 따라 INSERT 가능 여부 검증 필요(§10.2).

### 3.2 Prisma 모델 추가

studio가 정의한 모델과 동일 스펙으로 `apps/api/prisma/schema.prisma`에 추가한다. **migration 디렉토리에 새 파일을 만들지 않는다** — DB는 이미 적용 상태이므로 `prisma migrate resolve --applied <hash>` 로 baseline 처리하거나 `prisma db pull`로 introspect 후 수동 정리. 본 프로젝트는 Prisma migration 패턴을 사용 중이므로(`apps/api/prisma/migrations/` 3개), studio처럼 "마이그레이션 파일 없이 DB 직접 적용" 방침을 강제 적용할지 결정 필요(§10.3).

```prisma
model text_analyses {
  id               Int      @id @default(autoincrement())
  text_id          Int
  analyzer_type    String
  analyzer_version String   @default("unknown")
  result           Json
  status           String   @default("completed")
  error            String?
  analyzed_at      DateTime @default(now()) @db.Timestamptz(6)
  created_at       DateTime @default(now()) @db.Timestamptz(6)
  updated_at       DateTime @updatedAt        @db.Timestamptz(6)

  text texts @relation(fields: [text_id], references: [id], onDelete: Cascade)

  @@unique([text_id, analyzer_type, analyzer_version], map: "uq_text_analyses_text_type_version")
  @@index([text_id],                          map: "idx_text_analyses_1")
  @@index([analyzer_type],                    map: "idx_text_analyses_2")
  @@index([text_id, analyzer_type],           map: "idx_text_analyses_3")
  @@index([status],                           map: "idx_text_analyses_4")
}

model texts {
  // ... 기존 필드 유지
  text_analyses text_analyses[]
}
```

### 3.3 기존 `texts.analysis` 컬럼 처리

read/write 호출자가 없는 dead column. studio 측에서도 같은 결정으로 후속 PR에서 제거 예정. 본 프로젝트는 이번 PR에서 만지지 않는다(컬럼 그대로 둠).

---

## 4. 백엔드 (NestJS) 변경

### 4.1 분석기 어댑터 레이어 (확장 포인트)

studio §4.1 구조를 그대로 도입한다. 디렉토리:

```
apps/api/src/analyzer/
├── analyzer.module.ts
├── analyzer.registry.ts
├── analyzer.types.ts
├── adapters/
│   ├── analyzer.adapter.ts          # 추상 인터페이스
│   ├── picklass-cefr.adapter.ts     # https://analyzer.picklass.com/analyze
│   └── gemini-passage.adapter.ts    # 기존 AiService.analyzePassage 재포장
└── normalizer/
    ├── picklass-cefr.normalizer.ts
    └── gemini-passage.normalizer.ts
```

**기존 `LessonPlanService`(`apps/api/src/lesson-plan/lesson-plan.service.ts`)와의 관계**: lesson-plan 호출은 모듈 시퀀싱(`/lesson-plan`) 전용으로 어댑터 구조와 별개. 같은 도메인을 쓰지만 엔드포인트와 책임이 다르므로 `LessonPlanService`는 그대로 유지한다. 단, 환경변수는 §6 결정에 따라 통합 가능.

#### 어댑터 인터페이스

studio §4.1 의 `AnalyzerAdapter` / `AnalyzerRunResult` 그대로 차용. picklass-cefr 어댑터의 fetch 호출 부분만 발췌:

```ts
// adapters/picklass-cefr.adapter.ts
@Injectable()
export class PicklassCefrAdapter implements AnalyzerAdapter {
  readonly type = 'picklass-cefr';
  readonly version = 'v2.0.0';

  constructor(private readonly config: ConfigService) {}

  async analyze(content: string): Promise<AnalyzerRunResult> {
    const base = this.config.get<string>('ANALYZER_BASE_URL') ?? 'https://analyzer.picklass.com';
    const res = await fetch(`${base}/analyze`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: content, use_llm: false }),
    });
    if (!res.ok) throw new ServiceUnavailableException('analyzer.picklass.com 호출 실패');
    const json = await res.json();
    return {
      analyzerType: this.type,
      analyzerVersion: this.version,
      result: json,
      derived: { finalLevel: json.final_level, summary: json.analysis_summary },
    };
  }
}
```

#### gemini-passage 어댑터

기존 `AiService.analyzePassage`(`ai.service.ts:236`)를 호출만 하는 얇은 wrapper.

```ts
// adapters/gemini-passage.adapter.ts
@Injectable()
export class GeminiPassageAdapter implements AnalyzerAdapter {
  readonly type = 'gemini-passage';
  readonly version = 'gemini-2.5-flash-v1';

  constructor(private readonly aiService: AiService) {}

  async analyze(content: string): Promise<AnalyzerRunResult> {
    // tutoring의 analyzePassage는 title/wordCount까지 받으므로 wrapper에서 채워준다.
    const out = await this.aiService.analyzePassage({
      title: '',                  // 필요 시 어댑터 호출자가 별도 옵션으로 전달
      content,
      wordCount: content.split(/\s+/).filter(Boolean).length,
    });
    return {
      analyzerType: this.type,
      analyzerVersion: this.version,
      result: out as unknown as Record<string, unknown>,
    };
  }
}
```

> ⚠️ `analyzePassage` 시그니처가 `title/wordCount`를 요구하는데 어댑터 인터페이스는 `content`만 받는다. 두 가지 해결 방법: (a) 어댑터 인터페이스에 옵셔널 `meta` 인자 추가, (b) 호출자가 항상 `runAndStore`에 raw 컨텍스트를 함께 넘기게 시그니처 확장. §10.4 에서 결정.

### 4.2 신규 서비스: `TextAnalysesService`

`apps/api/src/analyzer/text-analyses.service.ts` (§10.5 결정).

studio §4.2 의 `runAndStore`/`runDefaults`/`findLatest`/`findAllLatestByText` 4개 메서드를 그대로 이식. raw query는 Prisma 5.x 기준 동일 문법.

### 4.3 지문 생성 흐름 통합

`LessonsService.createCustomLesson` (`lessons.service.ts:606-691`) 변경 포인트:

```ts
// 변경 전
const passage = await this.aiService.generatePassage({ ... });               // L606
const text    = await this.prisma.texts.create({ ... });                    // L613
const analysis = await this.aiService.analyzePassage({ ... });              // L626 — 결과 버려짐
const planResp = await this.lessonPlanService.generate({ ... });            // L633

// 변경 후 (제안)
const passage = await this.aiService.generatePassage({ ... });
const text    = await this.prisma.texts.create({ ... });
// (1) 분석기 묶음 실행 + 영속화. analysis 변수는 응답 호환을 위해 picklass-cefr 결과의 derived/summary로 대체하거나, gemini-passage 결과로 복원.
const analyzed = await this.textAnalyses.runDefaults(text.id, passage.content, {
  title: passage.title,
  wordCount: passage.wordCount,
});
const planResp = await this.lessonPlanService.generate({ ... });
```

**응답 호환성** (§10.6 결정 — (c) 채택): createCustom 응답의 `analysis` 필드는 그대로 유지(Gemini 정성 분석 결과). `runDefaults` 결과 배열에서 `gemini-passage` 항목을 추출해 채운다. 정확한 코드는 §10.6 본문 참조.

**동기 + 병렬 호출** (§10.7 결정 — (a) 채택): `runDefaults` 내부에서 `Promise.allSettled([picklass-cefr, gemini-passage])` 로 두 어댑터를 병렬 실행하여 추가 latency 를 두 어댑터 중 느린 쪽 한 번분으로 흡수. 일부 어댑터 실패는 허용(영속화 + lesson 생성 성공). 운영 모니터링 기준은 §10.7 참조.

### 4.4 조회/재실행 엔드포인트 (§10.8 결정 — `PassagesController` 신설)

`apps/api/src/passages/passages.controller.ts` 를 신설하여 다음 2개를 노출. 본 PR 은 §10.1 결정에 따라 표시 UI 를 동봉하지 않으므로 엔드포인트도 최소 2개로 제한.

| Method | Path | 설명 |
|---|---|---|
| `GET` | `/passages/:textId/analyses` | 해당 지문의 분석기별 최신 결과 묶음(`UiAnalysis[]` 노멀라이즈 응답) |
| `POST` | `/passages/:textId/analyses/refresh-all` | 기본 분석기 묶음 재실행 (관리자/디버그용) |

권한 검증: §10.2 결정에 따라 컨트롤러에서 `authenticatedUserId === texts.user_id` 일치 여부를 확인하고 불일치 시 `ForbiddenException`. RLS 우회 상태이므로 application layer 검증 필수.

기존 `POST /ai/analyze-passage`(`ai.controller.ts:108-113`) 컨트롤러 핸들러는 본 PR 에서 **즉시 삭제**(§10.9 결정). 단 `AiService.analyzePassage` 메서드는 `GeminiPassageAdapter` 가 호출하므로 유지.

### 4.5 (보류) 비동기 큐 — 본 PR 동기 + 병렬 채택(§10.7). 향후 latency 이슈 관측 시 단계적 전환 옵션은 studio §4.5 패턴 그대로 차용 가능.

---

## 5. 공유 타입 / 노멀라이저

### 5.1 타입 위치

tutoring 모노레포는 `packages/types/src/index.ts` 에 공유 타입을 모아둔다(`@tutoring/types`). 분석 관련 타입은 §10.10 결정에 따라 **`packages/types/src/analysis.ts` 별도 파일**로 분리하고 `index.ts` 는 `export * from './analysis';` 한 줄만 추가한다.

```ts
// packages/types/src/analysis.ts
export interface UiAnalysisIndicator {
  key: string;
  label: string;
  value: string;
  hint?: string;
}

export interface UiAnalysis {
  analyzerType: string;
  analyzerVersion: string;
  finalLabel?: string;
  summary?: string;
  indicators: UiAnalysisIndicator[];
  analyzedAt: string;
}

export interface PassageAnalysesResponse {
  textId: number;
  analyses: UiAnalysis[];
}
```

### 5.2 노멀라이저

`apps/api/src/analyzer/normalizer/`에 둔다(studio §10.8 결정과 동일). 백엔드가 응답 직전에 raw → `UiAnalysis` 로 변환하여 내려준다. picklass-cefr 노멀라이저는 studio 문서의 코드 그대로.

### 5.3 프론트 변경

본 PR 범위 외(§10.1). 다만 createCustom 응답의 `analysis` 필드를 어떻게 다룰지(§10.6)에 따라 web의 호출부(`apps/web/src/...lessons/create-custom...`) 한 줄 정도 수정될 수 있다.

---

## 6. 환경 변수

본 프로젝트는 `LessonPlanService`(시퀀싱)와 신규 `PicklassCefrAdapter`(정량 분석) 둘 다 **기존 키 `ANALYZER_BASE_URL` 하나**를 공유한다(§10.11 결정). 새 키를 도입하지 않는다.

| 키 | 값 (예시) | 설명 |
|---|---|---|
| `ANALYZER_BASE_URL` | `https://analyzer.picklass.com` | 분석기 base URL — `/lesson-plan`(시퀀싱) + `/analyze`(정량 분석) 공용 |
| `ANALYZER_DEFAULTS` | `picklass-cefr,gemini-passage` | 지문 생성 시 자동 실행할 어댑터 목록(콤마 구분). 미설정 시 `picklass-cefr` |

> `ANALYZER_BASE_URL` 은 이미 `apps/api/.env` / `apps/api/.env.local` 에 등록되어 있다(직전 세션에서 `https://analyzer.picklass.com` 으로 설정 완료). 추가 등록은 `ANALYZER_DEFAULTS` 한 키만.
> 타임아웃은 별도 env 없이 `fetch` 기본값 또는 어댑터 코드 상수로 관리. 운영 중 필요 시점에 추가 검토.

등록 채널: Vercel `english-ai/tutoring-picklass-com-api` 프로젝트 환경변수에 `ANALYZER_DEFAULTS` 추가하고 로컬은 `vercel env pull apps/api/.env.local` 로 동기화.

---

## 7. 마이그레이션 / 롤아웃 순서

1. **공유 DB 점검**: 운영 DB에 `text_analyses` 테이블 + RLS 정책이 실제로 존재하는지 검증(§3.1). 미존재 시 studio 문서 §3.1 SQL 블록을 직접 실행.
2. **Prisma 모델 동기화**: `schema.prisma`에 `text_analyses` 모델 + `texts.text_analyses` 역참조 추가, `prisma generate`. migration 파일 생성 여부는 §10.3.
3. **분석기 인프라 도입**: `apps/api/src/analyzer/` 디렉토리 일괄 추가 + `AnalyzerModule`, `AppModule` 등록.
4. **기존 Gemini 호출 wrapping**: `AiService.analyzePassage`는 그대로 유지하되 `GeminiPassageAdapter`가 호출하도록 wrapping (§10.4).
5. **`LessonsService.createCustomLesson` 통합**: §4.3 의 변경 적용. 응답 `analysis` 필드 처리 §10.6 결정대로.
6. **신규 엔드포인트**: §4.4 2개. 기존 `POST /ai/analyze-passage` 처리 §10.9.
7. **웹 변경**: 본 PR 범위 외(§10.1). 후속 PR에서 표시 UI 추가.

---

## 8. 영향 파일 요약

### 신규
- `apps/api/src/analyzer/analyzer.module.ts`
- `apps/api/src/analyzer/analyzer.registry.ts`
- `apps/api/src/analyzer/analyzer.types.ts`
- `apps/api/src/analyzer/adapters/{analyzer,picklass-cefr,gemini-passage}.adapter.ts`
- `apps/api/src/analyzer/normalizer/{picklass-cefr,gemini-passage}.normalizer.ts`
- `apps/api/src/analyzer/text-analyses.service.ts`
- `packages/types/src/analysis.ts`
- (선택) `apps/api/src/passages/passages.controller.ts` — §4.4 2개 엔드포인트

### 수정
- `apps/api/prisma/schema.prisma` (`text_analyses` 모델 + `texts` 역참조)
- `apps/api/src/lessons/lessons.service.ts:606-691` (createCustomLesson 통합)
- `apps/api/src/lessons/lessons.module.ts` (DI 등록)
- `apps/api/src/app.module.ts` (`AnalyzerModule` import)
- `packages/types/src/index.ts` (`export * from './analysis';` 한 줄 추가)
- (가능성) `apps/api/src/ai/ai.controller.ts:108-113` — `POST /ai/analyze-passage` 처리 (§10.9)
- (가능성) `apps/web/src/...` — createCustom 응답 `analysis` 필드 처리 변경 (§10.6)

### 제거
- 본 PR에서는 없음. `texts.analysis` 컬럼은 그대로 유지.

---

## 9. 결정사항 / 열린 이슈 (확정된 것만)

1. **DB DDL 미실행**: studio 측에서 이미 같은 Supabase에 적용했으므로 tutoring은 모델 추가만. ✔ (§3.1)
2. **결과 컬럼 유지 정책**: studio와 동일하게 `result` JSONB 단일. ✔
3. **두 분석기 모두 같은 테이블**: picklass-cefr + gemini-passage. ✔ (§4.1)
4. **본 PR은 표시 UI 미포함**: 인프라 + 데이터 영속화까지만. ✔ (§1.3)

---

## 10. 컨펌 필요 항목 (사용자와 함께 확정)

> 각 항목 옆 ☐를 ☑ 로 바꾸고 결정/메모를 남겨주시면 본문 해당 절을 그에 맞게 갱신합니다.

### 10.1 분석 결과 표시 UI 위치 ☑ (c) — 본 PR 범위 외
- **결정 (2026-04-26)**: **(c) 채택** — 본 PR은 데이터 영속화 인프라까지만 포함한다. 학생/강사용 분석 결과 표시 UI는 후속 PR에서 디자인 검토 후 별도로 추가한다. 이에 따라 §5.3(프론트 변경) 및 §7-7(웹 변경)은 본 PR에서 수행하지 않으며, 본 PR 산출물은 다음 두 가지에 한정:
  - (1) `text_analyses` 테이블에 picklass-cefr·gemini-passage 결과가 매 지문 생성 시 저장되는 것
  - (2) 후속 UI가 곧바로 사용할 수 있도록 `GET /passages/:textId/analyses` 조회 엔드포인트 + 노멀라이즈된 `UiAnalysis` 응답 형태가 준비되는 것

### 10.2 RLS 정책과 API 호출 컨텍스트 ☑ (a) — RLS 우회 + 애플리케이션 단 owner 검증
- **사실 확인 (2026-04-26)**:
  - `DATABASE_URL` 은 `postgresql://postgres.<project>:...@...pooler.supabase.com` 형태로 **Supabase `postgres` 역할** 사용 (`apps/api/.env`). 이 역할은 `BYPASSRLS` 속성을 가지므로 RLS가 **자동 우회**된다.
  - tutoring API에는 `createClient`/`SUPABASE_SERVICE` 등 Supabase JS 사용처가 없다(grep 0건). 모든 DB 접근이 **Prisma 단일 경로**.
  - 이미 `LessonsService` 등에서 `authenticatedUserId`로 application-layer 권한 체크를 수행 중(`lessons.service.ts:182, 236`). tutoring 전반의 컨벤션.
- **결정 (2026-04-26)**: **(a) 채택** — RLS는 우회 상태로 둔다(이미 그렇게 동작 중). studio가 만든 `text_analyses` RLS 정책 3개는 PostgREST/Supabase JS 경로의 안전장치로만 의미가 있고, NestJS 경로에는 영향이 없다. 따라서 본 PR에서 RLS·정책 수정 작업은 **불필요**.
- **본 PR이 추가로 해야 할 일**: `TextAnalysesService` 의 조회/재실행 메서드에서 owner 검증을 application layer로 수행. 예: `findLatest`/`findAllLatestByText` 호출 시 호출자가 넘긴 `authenticatedUserId` 와 `texts.user_id` 일치 여부를 확인하고 불일치면 `ForbiddenException`. INSERT 경로(`runAndStore`)는 호출자(`LessonsService.createCustomLesson`)가 이미 `targetUserId` 컨텍스트에서 호출하므로 추가 검증 불필요.

### 10.3 Prisma 마이그레이션 파일 ☑ (b) — 마이그레이션 파일 미생성
- **결정 (2026-04-26)**: **(b) 채택** — `apps/api/prisma/migrations/` 에 새 파일을 추가하지 않는다. studio 측에서 운영 DB에 이미 적용 완료된 상태이므로 tutoring은 **schema.prisma 모델 추가만** 수행한다.
- **권장 작업 순서**:
  1. `cd apps/api && npx prisma db pull` 로 운영 DB의 `text_analyses` 테이블을 introspect 하여 schema.prisma 에 자동 반영.
  2. (선택) introspect 결과를 본 문서 §3.2 의 모델 정의와 대조하여 컬럼/인덱스/제약 명명이 일치하는지 검증. 차이가 있으면 schema.prisma 쪽을 수동 정리(예: `@@unique` 의 `map` 명, `@@index` 명).
  3. `npx prisma generate` 로 클라이언트 재생성.
  4. `texts` 모델에 `text_analyses text_analyses[]` 역참조를 수동 추가(introspect 가 양방향 관계를 누락할 수 있음).
- **운영 노트**:
  - 본 결정은 **`text_analyses` 테이블에만 한정**된 예외. tutoring 의 일반 마이그레이션 컨벤션(`prisma migrate dev`)은 다른 테이블에서는 그대로 유지한다.
  - 향후 다른 개발자가 `prisma migrate dev` 를 실행할 때 schema.prisma 에는 있고 마이그레이션 히스토리에는 없는 `text_analyses` 모델로 인해 drift 가 감지될 수 있다. 이 경우 **새 마이그레이션 파일을 생성하지 말고** 동료에게 본 문서 §10.3 결정 사항을 전달하거나, 한 번만 `npx prisma migrate resolve --applied <existing_baseline>` 로 무시 처리.
  - 신규 환경(예: 로컬 DB 재구성, Preview)에서 빈 DB 로 시작할 일이 생기면 studio 측 `docs/analyzer.md §3.1` 의 SQL 블록을 직접 실행하여 테이블을 동일 스펙으로 만든다.

### 10.4 `AnalyzerAdapter.analyze` 시그니처 — 메타 인자 ☑ (a)
- **결정 (2026-04-26)**: **(a) 채택** — 어댑터 시그니처에 옵셔널 `meta` 인자를 추가한다. 분석기마다 필요한 부가 정보 차이를 한 곳에서 흡수하고, 분석기가 `meta`를 사용하지 않으면 무시하면 되도록 한다.
- **확정 인터페이스**:
  ```ts
  // analyzer.types.ts
  export interface AnalyzerInputMeta {
    title?: string;
    wordCount?: number;
  }

  // adapters/analyzer.adapter.ts
  export interface AnalyzerAdapter {
    readonly type: AnalyzerType;
    readonly version: string;
    analyze(content: string, meta?: AnalyzerInputMeta): Promise<AnalyzerRunResult>;
  }
  ```
- **호출 경로 변경**:
  - `TextAnalysesService.runAndStore(textId, content, type, meta?)` 로 시그니처 확장.
  - `runDefaults(textId, content, meta?)` 도 동일 인자 추가.
  - `LessonsService.createCustomLesson` 의 호출(§4.3)은 `runDefaults(text.id, passage.content, { title: passage.title, wordCount: passage.wordCount })` 형태로 채워 넘긴다.
  - `picklass-cefr.adapter.ts` 는 `meta` 를 무시(현재 응답에 영향 없음).
  - `gemini-passage.adapter.ts` 는 `meta?.title` / `meta?.wordCount` 를 그대로 `aiService.analyzePassage` 에 전달. 미전달 시 `content` 로부터 fallback(`title=''`, `wordCount = content.split(/\s+/).filter(Boolean).length`).

### 10.5 `TextAnalysesService` 디렉토리 위치 ☑ (a)
- **결정 (2026-04-26)**: **(a) 채택** — `apps/api/src/analyzer/text-analyses.service.ts` 에 둔다. 어댑터·레지스트리·노멀라이저와 같은 `AnalyzerModule` 안에 응집하여 한 모듈로 export. `LessonsService` 는 `AnalyzerModule` 을 import 해서 `TextAnalysesService` 를 주입받는다.
- **모듈 구조 변경**:
  ```
  apps/api/src/analyzer/
  ├── analyzer.module.ts          # provides: registry, adapters, TextAnalysesService / exports: TextAnalysesService
  ├── analyzer.registry.ts
  ├── analyzer.types.ts
  ├── text-analyses.service.ts    # ← 여기
  ├── adapters/
  │   ├── analyzer.adapter.ts
  │   ├── picklass-cefr.adapter.ts
  │   └── gemini-passage.adapter.ts
  └── normalizer/
      ├── picklass-cefr.normalizer.ts
      └── gemini-passage.normalizer.ts
  ```
- **§4.2 본문 갱신 사항**: 서비스 경로 표기 `apps/api/src/passages/text-analyses.service.ts` → `apps/api/src/analyzer/text-analyses.service.ts` 로 정정.
- **§4.4 엔드포인트 컨트롤러 위치**: `PassagesController` 는 §10.8 에서 결정. `TextAnalysesService` 위치와 분리하여 다룬다(서비스는 analyzer 모듈에, 컨트롤러는 §10.8 결정에 따름).
- **§8 영향 파일 갱신 사항**: `apps/api/src/passages/text-analyses.service.ts` 를 `apps/api/src/analyzer/text-analyses.service.ts` 로 정정.

### 10.6 createCustom 응답의 `analysis` 필드 ☑ (c) — 현 동작 보존
- **사실 확인 (2026-04-26)**: web에서 `analysis` 사용처는 단 두 줄 (`apps/web/src/app/class/lesson-setup/custom/page.tsx:407-408`):
  ```tsx
  <span>유형: {result.analysis.type}</span>
  <span>읽기: {result.analysis.estimatedReadingMinutes}분</span>
  ```
  - `type` / `estimatedReadingMinutes` 는 **Gemini `analyzePassage` 의 정성 분석 결과**.
  - **picklass-cefr 응답에는 이 두 필드가 없음** (정량 지표 위주). 따라서 (b) 대체 시 `undefined`로 회귀.
- **결정 (2026-04-26)**: **(c) 채택** — `LessonsService.createCustomLesson` 응답의 `analysis` 필드를 **그대로 유지**한다(Gemini 정성 분석 결과). picklass-cefr / gemini-passage 두 분석기의 결과는 §4.3 의 `runDefaults` 호출로 `text_analyses` 테이블에 영속화만 수행. 응답 형태 무변경 → web 변경 0건.
- **§4.3 본문 갱신 사항**: createCustom 변경 후의 `analyzed` 변수에서 `analysis` 필드 채움 방식 정정.
  ```ts
  // 변경 후 (확정)
  const passage = await this.aiService.generatePassage({ ... });
  const text    = await this.prisma.texts.create({ ... });

  // 분석기 묶음 실행 + 영속화 (picklass-cefr + gemini-passage)
  const analyzed = await this.textAnalyses.runDefaults(text.id, passage.content, {
    title: passage.title,
    wordCount: passage.wordCount,
  });

  // 응답 호환: gemini-passage 결과를 그대로 추출하여 기존 analysis 필드에 실어준다.
  // - 이전엔 aiService.analyzePassage()를 직접 호출했지만, 이제 GeminiPassageAdapter 안에서 동일 호출이 일어남.
  // - runDefaults는 settled 배열을 반환하므로 gemini-passage 항목을 찾아 result 를 꺼낸다.
  const geminiSettled = analyzed.find(
    (s) => s.status === 'fulfilled' && s.value.analyzer_type === 'gemini-passage',
  );
  const analysis =
    geminiSettled && geminiSettled.status === 'fulfilled'
      ? (geminiSettled.value.result as PassageAnalysisResult)
      : null;
  // gemini-passage 가 실패해도 lesson 생성 자체는 성공시킨다(analysis=null). web은 옵셔널로 처리하도록 §5.3 변경 또는 그대로 두고 type 가드.

  const planResp = await this.lessonPlanService.generate({ ... });
  ```
- **후속 PR 가이드**: §10.1 의 UI 추가 PR 에서 응답에 옵셔널 `analyses?: UiAnalysis[]` 필드를 추가하여 picklass-cefr 정량 결과를 함께 내려줄 수 있다(폴링 회피). 기존 `analysis` 필드는 그대로 유지.

### 10.7 분석 호출 동기/비동기 ☑ (a) — 동기 + 병렬
- **결정 (2026-04-26)**: **(a) 채택** — `TextAnalysesService.runDefaults` 내부에서 `Promise.allSettled` 로 두 어댑터(picklass-cefr · gemini-passage)를 **병렬 실행**한다. createCustom 의 추가 latency 는 두 어댑터 중 느린 쪽 한 번 분만 가산.
- **구현 노트**:
  - 기존 코드에서 `aiService.analyzePassage` 직렬 호출(L626) 자리는 `runDefaults` 호출로 대체. 즉 추가 latency 는 picklass-cefr 한 번분(약 1~3초 예상) — gemini-passage 가 더 느리면 그쪽이 latency 결정자.
  - 일부 어댑터 실패 허용: studio §4.2 의 `Promise.allSettled` 패턴 그대로. 한 어댑터가 실패해도 다른 분석은 영속화 + lesson 생성 성공.
  - `gemini-passage` 결과를 응답 `analysis` 자리에 채우는 처리는 §10.6 코드대로 `settled.find(...)` 로 추출.
- **운영 모니터링 기준**: createCustom 전체 p95 가 12초를 지속적으로 초과하면 §4.5 단계적 비동기 전환(Vercel `waitUntil` → Cron → 외부 큐) 검토.

### 10.8 신규 엔드포인트 prefix ☑ (a) — `PassagesController` 신설
- **결정 (2026-04-26)**: **(a) 채택** — `apps/api/src/passages/passages.controller.ts` 를 신설하여 `/passages/:textId/analyses` 시리즈를 노출. studio 의 컨트롤러 prefix 와 정합.
- **본 PR 노출 엔드포인트 (§10.1 결정에 맞춰 최소화)**:
  | Method | Path | 설명 |
  |---|---|---|
  | `GET` | `/passages/:textId/analyses` | 해당 지문의 분석기별 최신 결과 묶음(노멀라이즈된 `UiAnalysis[]`) |
  | `POST` | `/passages/:textId/analyses/refresh-all` | 기본 분석기 묶음 재실행 (관리자/디버그용) |
- **모듈 구성**: `PassagesModule` 신설 → `PassagesController` 만 보유, 서비스 의존성은 `AnalyzerModule` 에서 주입(`TextAnalysesService`). `LessonsModule` 과는 독립.
- **권한 검증**: §10.2 결정에 따라 컨트롤러에서 `authenticatedUserId` 와 `texts.user_id` 일치 여부를 확인하고 불일치 시 `ForbiddenException`. RLS 가 우회 상태이므로 application layer 검증이 필수.
- **§4.4 본문 갱신 사항**: 표 상단의 prefix 결정 미정 안내문 제거하고 위 2개 엔드포인트로 확정.

### 10.9 기존 `POST /ai/analyze-passage` 처리 ☑ (a) — 즉시 삭제
- **사실 확인 (2026-04-26)**: web grep 결과 `apps/web/src/...` 어디에서도 `/ai/analyze-passage` 를 호출하지 않음 (createCustom 내부에서만 `aiService.analyzePassage` 메서드를 직접 호출). 외부 호출자 없음.
- **결정 (2026-04-26)**: **(a) 채택** — 본 PR 에서 `AiController.analyzePassage` 핸들러(`ai.controller.ts:108-113`) 를 즉시 삭제. alias 미유지. studio §10.6 결정과 동일 패턴.
- **함께 처리할 것**:
  - `AiService.analyzePassage` 메서드(`ai.service.ts:236`) 자체는 **유지**한다. `GeminiPassageAdapter` 가 이 메서드를 호출하므로 (§10.4 결정). 컨트롤러 핸들러만 제거.
  - `apps/api/src/ai/ai.controller.ts` 에서 관련 import / DTO 도 함께 정리.

### 10.10 공유 타입 파일 분리 여부 ☑ (b)
- **결정 (2026-04-26)**: **(b) 채택** — `packages/types/src/analysis.ts` 새 파일에 분석 관련 타입(`UiAnalysisIndicator`, `UiAnalysis`, `PassageAnalysesResponse`)을 두고 `index.ts` 에서 barrel re-export(`export * from './analysis';`).
- **§5.1 본문 갱신 사항**: 타입 위치 표기 `packages/types/src/index.ts` → `packages/types/src/analysis.ts` 로 정정. `index.ts` 는 re-export 한 줄만 추가.
- **§8 영향 파일 갱신 사항**: `packages/types/src/index.ts` 항목을 다음 두 줄로 분리:
  - 신규: `packages/types/src/analysis.ts`
  - 수정: `packages/types/src/index.ts` (barrel re-export 한 줄 추가)

### 10.11 환경변수 통합 ☑ — 단일 변수 `ANALYZER_BASE_URL` 유지
- **결정 (2026-04-26)**: **변수 하나로 통합하되, 기존 키 `ANALYZER_BASE_URL` 을 그대로 재사용**한다. 새 키(`ANALYZER_API_URL`)를 도입하지 않는다. 같은 분석기 인프라이므로 base URL 한 개면 충분하고, 기존 `LessonPlanService` 코드(`lesson-plan.service.ts:31`)와 `apps/api/.env`·`apps/api/.env.local` 라인을 그대로 사용 가능 → 변경 0건.
- **사용 형태**:
  - `LessonPlanService` 는 `${ANALYZER_BASE_URL}/lesson-plan` (현 동작 유지)
  - `PicklassCefrAdapter` 는 `${ANALYZER_BASE_URL}/analyze` (신규)
- **§6 본문 갱신 사항**: 환경변수 표에서 `ANALYZER_API_URL` 행 제거하고 `ANALYZER_BASE_URL` 한 행으로 통일. 키 이름과 설명을 같이 정정.
- **§4.1 본문 갱신 사항**: `PicklassCefrAdapter` 코드의 `process.env.ANALYZER_API_URL` → `process.env.ANALYZER_BASE_URL` 로 정정.

### 10.12 백필 ☑ (a) — 백필 안 함
- **결정 (2026-04-26)**: **(a) 채택** — 본 PR 시점 이전에 만들어진 `texts` 행은 `text_analyses` 행이 없는 상태로 둔다. 새로 생성되는 지문만 picklass-cefr · gemini-passage 분석 결과가 영속화된다.
- **운영 영향**: 후속 UI(§10.1) 가 도입될 때 기존 지문은 분석 결과가 없으므로 `-` 또는 빈 상태로 표시(studio §10.5 와 동일 정책). 명시적으로 다시 분석하려면 §4.4 의 `POST /passages/:textId/analyses/refresh-all` 호출.
- **§7 본문 갱신 사항**: 롤아웃 순서에 백필 단계 없음을 명시.

---

> 위 12개 항목 중 사용자가 결정한 내용을 반영해 본문 §3~§7 을 갱신할 예정. ☐ 항목이 모두 ☑ 로 바뀌면 구현 착수.
