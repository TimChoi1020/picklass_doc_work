# 지문 난이도 분석기 연동 설계 (analyzer.picklass.com)

작성일: 2026-04-26
대상 시스템: studio.picklass.com (Next.js Web + NestJS API + Supabase Postgres)
연동 외부 API: https://analyzer.picklass.com (`POST /analyze`)

> **설계 원칙(2026-04-26 갱신)**
> 분석 결과는 컬럼으로 펼치지 않고 **JSON 한 컬럼(`result`)**에 보관한다. 분석기 종류를 식별하는 `analyzer_type` 디스크리미네이터를 두고, **하나의 지문에 여러 분석기 결과가 공존**할 수 있도록 `(text_id, analyzer_type, analyzer_version)` 단위로 행을 관리한다. UI는 분석기별 어댑터(노멀라이저)를 거쳐 동일한 표시 모델로 변환한다.

---

## 1. 배경 및 목표

### 1.1 현재 상태
- 지문 자세히 보기 모달(`apps/web/src/components/PassageDetailModal.tsx:96-132`)의 "지문 난이도 분석" 영역의 8개 지표(`B1+`, `B1`, `A2+` 등)가 **JSX에 하드코딩**되어 있어 어떤 지문을 열어도 동일한 값을 보여줌.
- NestJS 측에는 Gemini-2.5-flash 기반 분석(`apps/api/src/ai/ai.service.ts:71-124`)이 존재하며 결과를 `texts.analysis` JSONB 컬럼(`apps/api/prisma/schema.prisma:200`, migration `supabase/migrations/20260316000001_add_text_analysis.sql`)에 직접 덮어쓰는 구조. 그러나 프론트엔드는 이 결과를 사용하지 않음.
- `texts.level`은 **AI 생성 지문**(`text_type='A'`)일 때만 사용자가 직접 고른 CEFR 값으로 채워지고(`apps/web/src/components/CreatePassageModal.tsx:147-156`), **직접 입력 지문**(`text_type='I'`)은 `level`이 빈 값으로 저장됨(`apps/web/src/components/CreatePassageModal.tsx:189-194`).

### 1.2 목표
analyzer.picklass.com의 정량 분석 결과(8대 CEFR 지표)를 신규 테이블에 저장하고, 지문 생성 시점에 자동으로 호출하여 다음을 달성한다.

| 흐름 | `texts.level` | 분석 결과 (신규 테이블) |
|---|---|---|
| **AI 생성** (text_type='A') | 사용자 입력 그대로 유지 | analyzer 응답으로 채움 |
| **직접 입력** (text_type='I') | analyzer의 `final_level`로 채움 | analyzer 응답으로 채움 |
| **쌍둥이 생성** (text_type='T') | 원문 level 계승(현 동작 유지) | analyzer 응답으로 채움 |

추가로 `PassageDetailModal`의 하드코딩 8개 Badge를 신규 테이블 데이터로 바인딩한다.

### 1.3 확장성 요구사항 (이번 개정의 핵심)
- 향후 다른 분석기(예: Gemini 기반 정성 분석, 외부 lexile 분석기, 사내 신규 알고리즘)가 추가될 수 있다.
- 각 분석기는 **응답 스키마가 서로 다르며**, 노출하는 지표 개수·이름도 다를 수 있다.
- 한 지문에 대해 **여러 분석기 결과를 동시에 보존**해서 비교/대조하거나, 각 분석기의 버전 업그레이드 이력을 누적할 수 있어야 한다.
- DB 스키마는 새 분석기가 추가될 때 **마이그레이션 없이** 수용해야 한다(컬럼 추가 금지).

---

## 2. 외부 API 스펙 정리

### 2.1 엔드포인트
```
POST https://analyzer.picklass.com/analyze
Content-Type: application/json
Body: { "text": "<영어 본문>", "use_llm": false }
```

### 2.2 응답 (관심 필드만 발췌)

> **2026-04-26 README 갱신 반영**
> - `levels`/`scores`/`weighted_details`에 **`text_length` 키가 정식 포함**됨 (이전 누락 이슈 해소).
> - 종합 메타로 `level_score`, `difficulty_group`(A/B/C), `rough_level`, `boundary`(`upper_boundary` 등) 필드 추가.
> - 가중치 산정이 2단계 동적 모델로 변경됨 (Rough Scan → Fine-tuning). 자세한 표는 README의 [분석 원리](https://github.com/picklasslab/picklass-analyzer/blob/main/README.md#-분석-원리) 참조.

```json
{
  "total_score": 9.57,
  "level_score": 10,
  "final_level": "B2-",
  "level_description": "중상급 진입 (Pre-Upper Intermediate)",
  "text_info": { "length": 25, "avg_sentence_length": 8.3 },
  "metrics":  { "lti": 3.9, "ttr": 0.96, "sentence_length": 8.3, "text_length": 25, "sci": 2.17, "gws": 2.33, "ccr": 64.0, "kdi": 0.29 },
  "scores":   { "lti": 12, "ttr": 18, "sentence_length": 6, "text_length": 1, "sci": 8, "gws": 7, "ccr": 15, "kdi": 2 },
  "levels":   { "lti": "B2+", "ttr": "C2+", "sentence_length": "A2+", "text_length": "A1-", "sci": "B1", "gws": "B1-", "ccr": "C1+", "kdi": "A1" },
  "weighted_details": {
    "lti":             { "score": 12, "weight": 0.20, "weighted": 2.40 },
    "ttr":             { "score": 18, "weight": 0.15, "weighted": 2.70 },
    "sentence_length": { "score": 6,  "weight": 0.10, "weighted": 0.60 },
    "text_length":     { "score": 1,  "weight": 0.10, "weighted": 0.10 },
    "sci":             { "score": 8,  "weight": 0.15, "weighted": 1.20 },
    "gws":             { "score": 7,  "weight": 0.15, "weighted": 1.05 },
    "ccr":             { "score": 15, "weight": 0.05, "weighted": 0.75 },
    "kdi":             { "score": 2,  "weight": 0.10, "weighted": 0.20 }
  },
  "level_distribution": { "B2": 1, "C2": 1, "A2": 1, "A1": 2, "B1": 2, "C1": 1 },
  "difficulty_group": "B",
  "rough_level": "B1+",
  "boundary": "upper_boundary",
  "analysis_summary": "강점: lti, ttr 수준이 높음 | 개선점: sentence_length, text_length 보완 필요",
  "words": [ { "word": "rockets", "lemma": "rocket", "pos": "NNS", "cefr_level": "B1", ... } ]
}
```

### 2.3 8개 지표 ↔ 모달 라벨 매핑 (어댑터 책임)

이 매핑은 **DB 컬럼이 아니라 어댑터(5.1)에서 수행**한다. 8개 지표 모두 `levels.*`에서 직접 가져올 수 있어 별도 임계값 매핑/추정 로직이 필요 없다.

| 모달 라벨 (한글) | 정규화 키 | analyzer 응답 출처 |
|---|---|---|
| 어휘 난이도 | `vocab_difficulty` | `levels.lti` |
| 어휘 다양성 | `vocab_variety` | `levels.ttr` |
| 지문 길이 | `passage_length` | `levels.text_length` |
| 문장 길이 | `sentence_length` | `levels.sentence_length` |
| 문장 구조 | `sentence_structure` | `levels.sci` |
| 문법 다양성 | `grammar_variety` | `levels.gws` |
| 정보 밀도 | `information_density` | `levels.ccr` |
| 배경지식 의존도 | `background_knowledge` | `levels.kdi` |

> 보조 표시(툴팁, 상세 분해)에 사용할 수치는 `metrics.*`(원시값), `scores.*`(1~18 점수), `weighted_details.*`(가중치 적용 결과)에서 가져온다. `text_info.length`(단어 수), `text_info.avg_sentence_length`도 디버깅/리포트용으로 보관된 raw에 포함되어 있다.

---

## 3. 데이터베이스 설계 (신규 테이블)

기존 `texts.analysis` JSONB 컬럼은 deprecated 처리하되 마이그레이션 시 한 번에 제거하지 않고 후속 작업으로 미룬다(차이점·이력 보존).

### 3.1 테이블: `text_analyses`

> **✅ 적용 완료 (2026-04-26 확인)** — 운영 DB(`postgres.gwccmvkjxohtymujtgfm` / `aws-0-ap-northeast-2`)에 테이블이 생성되어 있음을 직접 접속 검증.
> - 컬럼 10개 (id, text_id, analyzer_type, analyzer_version, result, status, error, analyzed_at, created_at, updated_at) ✔
> - 인덱스 8개 (`text_analyses_pkey`, `uq_text_analyses_text_type_version`, `idx_text_analyses_1`~`6`) ✔
> - 제약 4개 (PK, FK→`texts(id) ON DELETE CASCADE`, UNIQUE 복합키, `status` CHECK) ✔
> - 트리거 0개 (애플리케이션 단 `updated_at` 갱신 컨벤션 준수) ✔
> - RLS 활성화 + 정책 3개 (view/insert/update) ✔
> - 현재 0행 — 후속 단계는 4절(NestJS 통합)부터 진행.

스키마는 **분석 결과 자체에 대한 컬럼을 두지 않는다**. 식별/조회용 메타와 결과 JSON만 보관한다.

```sql
-- 운영 DB에 직접 실행한 DDL (참조용 스펙).
-- 본 프로젝트는 supabase/migrations/ 파일로 tracked 마이그레이션을 만들지 않고
-- DB에 직접 적용하는 운영 방침을 따른다. 새 환경 재구성이 필요할 때 이 블록을 그대로 사용.

CREATE TABLE text_analyses (
  id                SERIAL PRIMARY KEY,
  text_id           INTEGER NOT NULL REFERENCES texts(id) ON DELETE CASCADE,

  -- 분석기 식별 (확장성의 핵심)
  analyzer_type     TEXT NOT NULL,                  -- 예: 'picklass-cefr', 'gemini-passage', 'lexile'
  analyzer_version  TEXT NOT NULL DEFAULT 'unknown',-- 예: 'v2.0.0' / 'gemini-2.5-flash-v1'

  -- 결과 본체 (스키마는 분석기별로 자유)
  result            JSONB NOT NULL,

  -- 상태 / 진단
  status            TEXT NOT NULL DEFAULT 'completed'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed')),
  error             TEXT,                           -- status='failed' 일 때 사유

  -- 메타
  analyzed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- 동일 (지문, 분석기 종류, 버전) 조합은 하나의 최신 결과만 유지
  CONSTRAINT uq_text_analyses_text_type_version
    UNIQUE (text_id, analyzer_type, analyzer_version)
);

-- 인덱스는 `idx_<테이블명>_<순번>` 컨벤션으로 명명한다.
CREATE INDEX idx_text_analyses_1 ON text_analyses(text_id);                    -- text_id 단독
CREATE INDEX idx_text_analyses_2 ON text_analyses(analyzer_type);              -- analyzer_type 단독
CREATE INDEX idx_text_analyses_3 ON text_analyses(text_id, analyzer_type);     -- 복합 (지문별 분석기 조회)
CREATE INDEX idx_text_analyses_4 ON text_analyses(status);                     -- 재시도/장애 식별
CREATE INDEX idx_text_analyses_5 ON text_analyses USING gin(result);           -- result JSONB 전체

-- 자주 조회하는 표시 필드(예: final_level)는 GIN 대신 expression index로 보강.
-- 분석기별로 키 위치가 다를 수 있어 분석기 단위로 만들고 순번을 이어 붙인다.
CREATE INDEX idx_text_analyses_6
  ON text_analyses ((result->>'final_level'))
  WHERE analyzer_type = 'picklass-cefr';

-- ※ updated_at 갱신은 DB 트리거 대신 애플리케이션(Prisma `@updatedAt`)에서 처리한다.
--   기존 `update_updated_at_column()` 트리거 패턴을 따르지 않는다.

ALTER TABLE text_analyses ENABLE ROW LEVEL SECURITY;

-- texts의 소유자만 읽기/쓰기
CREATE POLICY "Users can view own text_analyses" ON text_analyses
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM texts t WHERE t.id = text_analyses.text_id AND t.user_id = auth.uid())
  );
CREATE POLICY "Users can insert own text_analyses" ON text_analyses
  FOR INSERT WITH CHECK (
    EXISTS (SELECT 1 FROM texts t WHERE t.id = text_analyses.text_id AND t.user_id = auth.uid())
  );
CREATE POLICY "Users can update own text_analyses" ON text_analyses
  FOR UPDATE USING (
    EXISTS (SELECT 1 FROM texts t WHERE t.id = text_analyses.text_id AND t.user_id = auth.uid())
  );
```

### 3.2 설계 의도

- **결과 컬럼을 두지 않는 이유**: 분석기마다 응답 스키마가 다르고(예: Lexile 분석기는 단일 점수만, Gemini 정성 분석은 자유 서술) 새 분석기를 더할 때마다 컬럼/마이그레이션이 늘어나면 비용이 크다. JSONB 한 컬럼이면 분석기 종류와 무관하게 무한 확장 가능.
- **`analyzer_type` 디스크리미네이터**: 한 지문에 여러 분석기 결과를 동시에 보존하기 위한 핵심. UI/API 어디서든 `WHERE text_id=? AND analyzer_type=?`로 분기 조회.
- **`analyzer_version` 분리 보관**: 분석기는 알고리즘 개선으로 결과가 달라질 수 있다. 버전을 키에 포함하여 동일 지문에 대한 v1/v2 결과를 모두 비교 가능. 같은 버전을 재실행하면 upsert로 덮어쓴다.
- **`result` JSONB + GIN + expression index**: 자유 스키마를 보장하면서도 자주 쓰는 키(예: `result->>'final_level'`)는 partial expression index로 인덱싱하여 필터/정렬 성능을 확보. 새 분석기가 들어오면 그 분석기 전용 partial index만 추가하면 된다.
- **`status` / `error`**: 비동기화될 가능성과 외부 분석기 장애를 표현. 동기 호출 단계에서도 `failed` 행을 남겨 재시도 대상 식별에 사용.
- **`updated_at` 갱신은 애플리케이션 책임**: 기존 `update_updated_at_column()` 트리거를 사용하던 다른 테이블과 달리 본 테이블은 트리거를 두지 않는다. NestJS는 Prisma의 `@updatedAt`(자동 세팅) / 직접 작성하는 raw query에서 `updated_at = NOW()`를 명시한다. 이유: (1) 트리거가 모든 update 경로에 적용되어 백필/마이그레이션 시 의도치 않은 갱신을 일으킬 수 있고, (2) 분석 결과는 upsert 위주라 애플리케이션 단에서 명시적으로 시점을 통제하는 편이 일관성 있음.
- **RLS**: `texts`와 동일한 소유권 모델을 EXISTS 절로 위임.

### 3.3 Prisma 모델

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
  // idx_text_analyses_5 (GIN on result) / idx_text_analyses_6 (partial expression)는
  // Prisma가 직접 표현하지 못하므로 DB에 직접 DDL로 적용한다 (3.1 SQL 블록 참조).
}

model texts {
  // ... 기존 필드 유지
  text_analyses text_analyses[]   // 1:N
}
```

> 기존 `texts.analysis` 컬럼(JSONB) — 후속 PR에서 제거. 제거 전까지는 dual-write 하지 않고 신규 테이블만 갱신.

### 3.4 최신 결과 조회 패턴 (뷰 미채택, 10.7 결정)

`(text_id, analyzer_type)` 단위로 가장 최근 버전 한 행만 조회하는 일이 잦지만, **별도 뷰는 만들지 않는다**. NestJS에서 다음 두 패턴으로 처리:

```ts
// 1) 단일 (text, type) 최신 한 행 — Prisma 표준 API
prisma.text_analyses.findFirst({
  where: { text_id, analyzer_type, status: 'completed' },
  orderBy: { analyzed_at: 'desc' },
});

// 2) 한 지문의 분석기별 최신 묶음 — DISTINCT ON 필요, raw query
prisma.$queryRaw`
  SELECT DISTINCT ON (analyzer_type) *
  FROM text_analyses
  WHERE text_id = ${textId} AND status = 'completed'
  ORDER BY analyzer_type, analyzed_at DESC
`;
```

> 동일 raw 쿼리가 3곳 이상 반복되거나 권한 분리가 필요해지면 그때 `text_analyses_latest` 뷰로 추출 검토.

---

## 4. 백엔드 (NestJS) 변경

### 4.1 Analyzer 어댑터 레이어 (확장 포인트)

분석기 추가에 대비해 **레지스트리 + 어댑터 패턴**을 도입한다. 새 분석기를 붙일 때 변경은 어댑터 파일 하나 추가 + 레지스트리 등록 한 줄로 끝나야 한다.

```
apps/api/src/analyzer/
├── analyzer.module.ts
├── analyzer.registry.ts          # type → adapter 매핑
├── analyzer.types.ts             # 공용 인터페이스
├── adapters/
│   ├── analyzer.adapter.ts       # 추상 인터페이스
│   ├── picklass-cefr.adapter.ts  # https://analyzer.picklass.com
│   └── (추후) gemini-passage.adapter.ts, lexile.adapter.ts ...
└── normalizer/
    ├── ui-normalizer.ts          # 분석기별 result → 공통 UI 모델
    └── picklass-cefr.normalizer.ts
```

#### 어댑터 인터페이스

```ts
// analyzer.types.ts
export type AnalyzerType = 'picklass-cefr' | 'gemini-passage' | string; // 확장 허용
export interface AnalyzerRunResult {
  analyzerType: AnalyzerType;
  analyzerVersion: string;
  result: Record<string, unknown>;   // 그대로 JSONB에 저장
  // 선택: 어댑터가 추출해 주는 표준 필드 (텍스트 level 보강용 등)
  derived?: {
    finalLevel?: string;             // I 타입 보강 시 사용
    summary?: string;
  };
}

// adapters/analyzer.adapter.ts
export interface AnalyzerAdapter {
  readonly type: AnalyzerType;
  readonly version: string;
  analyze(content: string): Promise<AnalyzerRunResult>;
}
```

#### picklass-cefr 어댑터

```ts
// adapters/picklass-cefr.adapter.ts
@Injectable()
export class PicklassCefrAdapter implements AnalyzerAdapter {
  readonly type = 'picklass-cefr';
  // 분석기 알고리즘 버전. analyzer 응답에 version 필드가 추가되면 이 상수 제거.
  // 분석기 업그레이드 시 같은 PR에서 갱신 → git blame으로 변경 시점 추적 (10.3 결정).
  readonly version = 'v2.0.0';

  async analyze(content: string): Promise<AnalyzerRunResult> {
    const res = await fetch(`${process.env.ANALYZER_API_URL}/analyze`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: content, use_llm: false }),
    });
    if (!res.ok) throw new ServiceUnavailableException('analyzer.picklass.com 호출 실패');
    const json = await res.json();
    return {
      analyzerType: this.type,
      analyzerVersion: this.version,
      result: json,                          // 응답 전체 그대로 저장
      derived: { finalLevel: json.final_level, summary: json.analysis_summary },
    };
  }
}
```

#### 레지스트리

```ts
// analyzer.registry.ts
@Injectable()
export class AnalyzerRegistry {
  private readonly adapters = new Map<AnalyzerType, AnalyzerAdapter>();
  constructor(picklass: PicklassCefrAdapter /*, gemini: GeminiPassageAdapter ...*/) {
    this.register(picklass);
  }
  register(a: AnalyzerAdapter) { this.adapters.set(a.type, a); }
  get(type: AnalyzerType): AnalyzerAdapter {
    const a = this.adapters.get(type);
    if (!a) throw new BadRequestException(`Unknown analyzer: ${type}`);
    return a;
  }
  defaults(): AnalyzerType[] {
    // 지문 생성 시 기본으로 돌릴 분석기 목록 — env 또는 DB로 외부화 가능
    return (process.env.ANALYZER_DEFAULTS ?? 'picklass-cefr').split(',') as AnalyzerType[];
  }
}
```

새 분석기 추가 절차:
1. `adapters/<name>.adapter.ts` 작성 (Adapter 인터페이스 구현).
2. `AnalyzerRegistry` 생성자 DI 목록에 추가하고 `register()` 호출.
3. UI 표시가 필요하면 `normalizer/<name>.normalizer.ts` 작성(5.1 참조).
4. `ANALYZER_DEFAULTS` env에 추가하면 지문 생성 시 자동 실행.
**DB 마이그레이션 불필요**.

### 4.2 신규 서비스: `TextAnalysesService`

`apps/api/src/passages/text-analyses.service.ts`

```ts
@Injectable()
export class TextAnalysesService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly registry: AnalyzerRegistry,
  ) {}

  /** 단일 분석기 실행 후 upsert */
  async runAndStore(textId: number, content: string, type: AnalyzerType) {
    const adapter = this.registry.get(type);
    try {
      const out = await adapter.analyze(content);
      return await this.prisma.text_analyses.upsert({
        where: {
          uq_text_analyses_text_type_version: {
            text_id: textId,
            analyzer_type: out.analyzerType,
            analyzer_version: out.analyzerVersion,
          },
        },
        update: { result: out.result, status: 'completed', error: null, analyzed_at: new Date() },
        create: {
          text_id: textId,
          analyzer_type: out.analyzerType,
          analyzer_version: out.analyzerVersion,
          result: out.result,
          status: 'completed',
        },
      });
    } catch (e) {
      // 실패도 행으로 남겨 재시도 식별
      await this.prisma.text_analyses.upsert({
        where: { /* 동일 키 */ },
        update: { status: 'failed', error: String(e) },
        create: {
          text_id: textId, analyzer_type: type, analyzer_version: adapter.version,
          result: {}, status: 'failed', error: String(e),
        },
      });
      throw e;
    }
  }

  /** 기본 분석기 묶음 실행 (병렬, 일부 실패 허용) */
  async runDefaults(textId: number, content: string) {
    const types = this.registry.defaults();
    const settled = await Promise.allSettled(
      types.map((t) => this.runAndStore(textId, content, t)),
    );
    return settled;
  }

  /** 표시용 최신 결과 조회 */
  async findLatest(textId: number, type: AnalyzerType) {
    return this.prisma.text_analyses.findFirst({
      where: { text_id: textId, analyzer_type: type, status: 'completed' },
      orderBy: { analyzed_at: 'desc' },
    });
  }

  async findAllLatestByText(textId: number) {
    // 분석기 타입별 최신 한 행씩
    return this.prisma.$queryRaw`
      SELECT DISTINCT ON (analyzer_type) *
      FROM text_analyses
      WHERE text_id = ${textId} AND status = 'completed'
      ORDER BY analyzer_type, analyzed_at DESC
    `;
  }
}
```

### 4.3 지문 생성 흐름에 통합

`PassagesService.create()` (`apps/api/src/passages/passages.service.ts:81`)에서 **항상 기본 분석기 묶음을 실행**한다. 정책:

| text_type | 분석 호출 | `texts.level` 결정 로직 |
|---|---|---|
| `A` (AI 생성) | ✅ `runDefaults` | `dto.level` 그대로 사용 (사용자가 고른 CEFR 유지) |
| `I` (직접 입력) | ✅ `runDefaults` | `dto.level`이 비어 있으면 **권위 분석기**의 `derived.finalLevel`로 채움. 권위 분석기 실패 시 NULL 유지(10.4) — 지문 저장은 성공, UI는 `-` 표시(10.5). |
| `T` (쌍둥이) | ✅ `runDefaults` | 원문 level 계승 (현 동작) |

> "권위 분석기"는 env로 지정(`ANALYZER_LEVEL_AUTHORITY=picklass-cefr`). 새 분석기로 바꾸고 싶으면 env 변경만으로 가능.

```ts
async create(params): Promise<TextItem> {
  const text = await this.prisma.texts.create({ data: { ...params } });

  const settled = await this.textAnalyses.runDefaults(text.id, text.content);

  // 직접 입력 + level 미지정인 경우만 권위 분석기 결과로 보강
  if (params.textType === 'I' && !params.level) {
    const authority = process.env.ANALYZER_LEVEL_AUTHORITY ?? 'picklass-cefr';
    const ok = settled.find(
      (s) => s.status === 'fulfilled' && s.value.analyzer_type === authority,
    );
    if (ok && ok.status === 'fulfilled') {
      const derivedLevel = (ok.value.result as any)?.final_level;
      if (derivedLevel) {
        await this.prisma.texts.update({
          where: { id: text.id },
          data: { level: derivedLevel },
        });
        text.level = derivedLevel;
      }
    }
  }

  return text;
}
```

### 4.4 분석 조회/재실행 엔드포인트

기존 단일 `/passages/:id/analysis` 형태는 다중 분석기를 표현 못하므로 다음과 같이 확장한다.

| Method | Path | 설명 |
|---|---|---|
| `GET`  | `/passages/:id/analyses` | 해당 지문의 모든 분석기 최신 결과 (배열) |
| `GET`  | `/passages/:id/analyses/:type` | 특정 분석기 최신 결과 |
| `GET`  | `/passages/:id/analyses/:type/history` | 특정 분석기 버전별 이력 |
| `POST` | `/passages/:id/analyses/:type/refresh` | 특정 분석기 재실행 |
| `POST` | `/passages/:id/analyses/refresh-all` | 기본 분석기 묶음 재실행 |

기존 `GET /passages/:id/analysis` / `POST /passages/:id/analysis/refresh` (`passages.controller.ts:70-82`)는 **본 PR에서 즉시 삭제** (alias 없음, 10.6 결정). 호출자가 dead code 상태였음을 확인. 함께 프론트 래퍼 `aiApi.analyzePassage` / `aiApi.refreshAnalysis` (`apps/web/src/lib/api.ts:360-366`)도 삭제.

기존 `AiService.analyzePassage()`(Gemini 호출 + JSONB 저장)는 deprecated 처리. 향후 `gemini-passage` 어댑터로 재포장하면 동일 테이블에 함께 저장 가능.

### 4.5 (보류) 비동기 큐 적용 — 현재는 동기 유지 (10.2 결정)

> **현재 운영 방침은 동기 호출**이다(10.2 참조). 본 절은 향후 응답 시간 이슈가 관측될 때를 위한 단계적 전환 옵션 모음.

다수의 분석기를 병렬 실행하면 응답 시간이 가장 느린 분석기에 의해 결정된다. 외부 분석기 응답이 느리거나 일부 분석기가 LLM 기반(>5s)이면 다음 단계로 전환:

**1단계 — Vercel `waitUntil`** (가장 간단, 별도 인프라 불필요)
```ts
import { waitUntil } from '@vercel/functions';

waitUntil(this.textAnalyses.runDefaults(text.id, text.content));
return text;  // 즉시 응답, 분석은 백그라운드에서 함수 timeout(최대 300s) 안에 완료
```
사용자는 즉시 "저장 완료" 응답을 받고, 모달에서 8개 지표는 폴링/알림으로 채워짐.

**2단계 — Vercel Cron + `pending` 상태 폴링**
1. 지문 생성 시 `text_analyses` 행을 `status='pending'`으로 INSERT.
2. Vercel Cron이 매 1분 `pending` 행을 N개 가져와서 처리.
3. 기존 `async_tasks` 테이블(`supabase/migrations/20250710035319_initial_schema.sql:52`)을 활용하면 재시도/우선순위까지 표현 가능.

**3단계 — 외부 큐 서비스** (Inngest / Upstash QStash / Trigger.dev)
1. 지문 생성 시 이벤트 publish, 즉시 응답.
2. 큐 서비스가 webhook으로 워커 함수 호출 → analyzer 실행 → upsert + `notifications` 발행.

레지스트리에서 어댑터별로 `mode: 'sync' | 'async'`를 명시해 분석기마다 다른 전략을 쓸 수 있도록 확장 가능.

---

## 5. 공유 타입 / 프론트엔드 변경

### 5.1 노멀라이저(어댑터의 UI 짝)

UI는 분석기마다 다른 JSON을 직접 다루지 않고, **표시 모델로 변환된 결과**만 사용한다. **노멀라이저 함수는 `apps/api`에만 두고 백엔드가 응답 전에 변환**하며, 표시 모델 타입만 `packages/shared`에 두어 양쪽이 공유한다(10.8 결정).

```ts
// packages/shared/src/types/analysis.ts  ← 타입은 shared
export interface UiAnalysisIndicator {
  key: string;          // 'vocab_difficulty' 등
  label: string;        // '어휘 난이도'
  value: string;        // 'B1+' (분석기에 따라 점수일 수도)
  hint?: string;        // 툴팁용
}

export interface UiAnalysis {
  analyzerType: string;
  analyzerVersion: string;
  finalLabel?: string;        // 분석기가 종합 라벨을 내는 경우 (예: 'B2-')
  summary?: string;
  indicators: UiAnalysisIndicator[];
  analyzedAt: string;
}

export interface PassageAnalysesResponse {
  textId: number;
  analyses: UiAnalysis[];     // 분석기별 최신 결과들
}

// apps/api/src/analyzer/normalizer/picklass-cefr.normalizer.ts  ← 노멀라이저는 apps/api
import type { UiAnalysis } from '@classsnap/shared';
export function normalizePicklassCefr(raw: any, version: string, analyzedAt: string): UiAnalysis {
  const levels = raw?.levels ?? {};
  return {
    analyzerType: 'picklass-cefr',
    analyzerVersion: version,
    finalLabel: raw?.final_level,
    summary: raw?.analysis_summary,
    analyzedAt,
    indicators: [
      { key: 'vocab_difficulty',     label: '어휘 난이도',     value: levels.lti ?? '-' },
      { key: 'vocab_variety',        label: '어휘 다양성',     value: levels.ttr ?? '-' },
      { key: 'passage_length',       label: '지문 길이',       value: levels.text_length ?? '-' },
      { key: 'sentence_length',      label: '문장 길이',       value: levels.sentence_length ?? '-' },
      { key: 'sentence_structure',   label: '문장 구조',       value: levels.sci ?? '-' },
      { key: 'grammar_variety',      label: '문법 다양성',     value: levels.gws ?? '-' },
      { key: 'information_density',  label: '정보 밀도',       value: levels.ccr ?? '-' },
      { key: 'background_knowledge', label: '배경지식 의존도', value: levels.kdi ?? '-' },
    ],
  };
}
```

새 분석기를 붙일 때 `apps/api/src/analyzer/normalizer/`에 노멀라이저 한 파일만 추가하면 동일한 `UiAnalysis` 모양으로 UI에 노출 가능. 백엔드가 응답 전에 노멀라이즈해서 내려주는 것을 기본으로 하되(클라이언트는 표시 모델만 알면 됨), 전체 raw가 필요한 화면(상세 분해, 단어 하이라이트)에서는 별도 엔드포인트로 raw를 가져온다.

### 5.2 `PassageDetailModal` 데이터 바인딩

`apps/web/src/components/PassageDetailModal.tsx:96-132`에서:
- props에 `analyses?: UiAnalysis[]` 추가 (배열! 분석기 여러 개를 모두 받음).
- 우선 `analyses.find(a => a.analyzerType === 'picklass-cefr')`의 `indicators`를 8개 Badge에 매핑.
- 추후 다른 분석기가 추가되면 탭 또는 분석기 선택 드롭다운으로 노출 영역 분기.
- **데이터 없음/실패 처리 (10.5 결정)**: 8개 지표 각각의 `value`가 비어 있거나 `analyses`가 비어 있으면 해당 Badge에 `-` 한 글자만 표시. 별도의 "분석 중" / "재시도" 라벨이나 버튼은 두지 않는다. 명시적으로 다시 분석하려면 4.4절의 `POST /passages/:id/analyses/refresh-all` 엔드포인트를 호출 — 다만 이 트리거 UX는 본 모달 외부(예: 별도 관리 액션)에서 노출.

`apps/web/src/app/class/page.tsx`의 `selectedPassage`에 `analyses` 필드를 추가. `useText(id)`(`apps/web/src/hooks/useTexts.ts:46`)가 `/passages/:id`에서 `analyses` 배열까지 함께 받도록 NestJS `findOne`에 join을 추가.

### 5.3 생성 흐름 변경 포인트

프론트엔드는 분석 호출을 직접 하지 않는다. UX:
- `CreatePassageModal.handleConfirmPassage` / `handleManualCreate` / 쌍둥이 저장 후 토스트 표시 시점은 그대로.
- 상세 모달이 열릴 때 분석이 아직 진행 중이면 React Query polling 또는 `notifications` 구독으로 재페치.

---

## 6. 환경 변수 / 인프라

> **등록 채널 (10.1 확정)**: 신규 키는 **Vercel 프로젝트 환경변수에 직접 등록**한다. `apps/api`(NestJS)는 `english-ai/studio-picklass-com-api`, `apps/web`(Next.js)는 `english-ai/studio-picklass-com-web`. 로컬 동기화는 `vercel env pull .env.local --environment=development` 한 번으로 완료. `.env.local`은 `.gitignore` 처리되어 있음.

대상 키 (네 개 환경 — Development / Preview / Production 공통, 동일 값):

| 키 | 값 (예시) | 설명 |
|---|---|---|
| `ANALYZER_API_URL` | `https://analyzer.picklass.com` | 분석기 API base URL |
| `ANALYZER_API_TIMEOUT_MS` | `15000` | HTTP 타임아웃 (ms) |
| `ANALYZER_DEFAULTS` | `picklass-cefr` | 지문 생성 시 자동 실행할 어댑터 목록 (콤마 구분) |
| `ANALYZER_LEVEL_AUTHORITY` | `picklass-cefr` | `text_type='I'` && level 미지정 시 권위 분석기 |

> `analyzer_version`은 어댑터 코드 상수로 관리한다(10.3 결정). env 키 없음.

등록 절차 (각 환경 반복):
```bash
cd apps/api
vercel env add ANALYZER_API_URL          development   # 프롬프트에서 값 입력
vercel env add ANALYZER_API_URL          preview
vercel env add ANALYZER_API_URL          production
# ... 나머지 3개 키(ANALYZER_API_TIMEOUT_MS / ANALYZER_DEFAULTS / ANALYZER_LEVEL_AUTHORITY)도 동일하게
vercel env pull .env.local --environment=development
```

> 대상 프로젝트는 `apps/api`(NestJS)뿐. `apps/web`은 분석기를 직접 호출하지 않으므로 등록 불필요.

새 분석기 추가 시 `ANALYZER_DEFAULTS=picklass-cefr,gemini-passage`처럼 Vercel 환경변수만 갱신하면 활성화. 인증이 필요한 분석기는 어댑터 단위로 `<TYPE>_API_TOKEN` 같은 별도 키를 같은 채널로 등록.

---

## 7. 마이그레이션 / 롤아웃 순서

1. ✅ **DB DDL 적용 완료 (2026-04-26)**: `text_analyses` 테이블 + 인덱스 6개 + 제약 + RLS 정책 3개를 운영 DB에 직접 실행 후 접속 검증. 본 프로젝트는 `supabase/migrations/` 파일로 tracked 마이그레이션을 만들지 않으므로 별도 마이그레이션 커밋은 없음 — DDL 스펙은 본 문서 3.1 SQL 블록을 단일 출처로 유지한다.
2. **Prisma generate**: 모델 추가 후 `prisma db pull` 또는 수동 모델 작성 → `prisma generate`. (DB 스키마가 이미 존재하므로 `migrate dev` 대신 introspect 또는 baseline 처리)
3. **NestJS 분석 인프라**: `AnalyzerRegistry` + `PicklassCefrAdapter` + `TextAnalysesService` + 모듈 등록.
4. **`PassagesService.create` 통합**: `runDefaults` 호출 + I 타입 level 보강.
5. **신규 엔드포인트**: `/passages/:id/analyses[...]` 시리즈. 기존 `/analysis` 경로는 alias 없이 즉시 삭제 (10.6 결정).
6. **공유 타입(shared) + 노멀라이저(api)**: `packages/shared/src/types/analysis.ts`(타입) + `apps/api/src/analyzer/normalizer/picklass-cefr.normalizer.ts`(변환 함수). 10.8 결정.
7. **프론트 모달 바인딩**: `PassageDetailModal`이 `UiAnalysis[]` 사용.
8. ~~백필 스크립트~~: 백필 수행하지 않음 (10.5 결정). 기존 지문은 `text_analyses` 행이 없으면 UI에 `-` 표시.
9. **기존 Gemini 분석 deprecate**: `AiService.analyzePassage`, `texts.analysis` 컬럼 제거(후속 PR). 필요 시 `GeminiPassageAdapter`로 재구성하여 동일 테이블에 등록.

---

## 8. 영향 파일 요약

### 신규
- `apps/api/src/analyzer/analyzer.module.ts`
- `apps/api/src/analyzer/analyzer.registry.ts`
- `apps/api/src/analyzer/analyzer.types.ts`
- `apps/api/src/analyzer/adapters/analyzer.adapter.ts`
- `apps/api/src/analyzer/adapters/picklass-cefr.adapter.ts`
- `apps/api/src/passages/text-analyses.service.ts`
- `apps/api/src/analyzer/normalizer/picklass-cefr.normalizer.ts`

### 수정
- `apps/api/prisma/schema.prisma` (texts 모델 + text_analyses 모델)
- `apps/api/src/passages/passages.module.ts` (DI 등록)
- `apps/api/src/passages/passages.service.ts:81-187` (create 통합, getAnalysis 데이터 소스 교체)
- `apps/api/src/passages/passages.controller.ts:70-82` (신규 엔드포인트 추가 + 옛 `/analysis` 핸들러 삭제)
- `apps/web/src/lib/api.ts:359-367` (`aiApi.analyzePassage`/`refreshAnalysis` 래퍼 삭제 또는 신규 경로 호출로 재작성)
- `apps/api/src/app.module.ts` (AnalyzerModule import)
- `packages/shared/src/types/analysis.ts` (`UiAnalysis`, `PassageAnalysesResponse` 추가)
- `apps/web/src/hooks/useTexts.ts:46-72` (TextDetail 매핑에 `analyses` 추가)
- `apps/web/src/components/PassageDetailModal.tsx:11-32, 96-132` (props + Badge 바인딩, 다중 분석기 대응)
- `apps/web/src/app/class/page.tsx:128-307` (selectedPassage에 analyses 전달)

### 제거(후속 PR)
- `texts.analysis` JSONB 컬럼 + `idx_texts_analysis` 인덱스 — `ALTER TABLE texts DROP COLUMN analysis;`를 운영 DB에 직접 실행 (마이그레이션 파일 없이). 과거 추가는 `supabase/migrations/20260316000001_add_text_analysis.sql`에 기록되어 있으나 본 프로젝트는 더 이상 tracked 마이그레이션을 만들지 않음.
- `apps/api/src/ai/ai.service.ts:71-124` `analyzePassage` 메서드 (또는 어댑터로 재포장)

---

## 9. 결정사항 / 열린 이슈

1. **결과 저장 방식**: flat 컬럼 → JSONB(`result`) 단일 컬럼으로 통일 (분석기별 스키마 차이 흡수). ✔ (이번 개정에서 확정)
2. **다중 분석기**: `analyzer_type` + `analyzer_version` 디스크리미네이터로 1:N 관리. UNIQUE 키는 `(text_id, analyzer_type, analyzer_version)`. ✔
3. **분석 호출 동기 vs 비동기**: 1차 동기 (picklass-cefr) → 분석기 늘어나면 어댑터 단위로 비동기 전환 (4.5).
4. ~~`levels.text_length` 응답 여부~~ ✔ (2026-04-26 README 갱신으로 정식 포함 확인 — `levels`/`scores`/`weighted_details` 모두에 키 존재).
5. **AI 생성 시 사용자 선택 level과 analyzer `final_level`이 다르면?**: 사용자 선택값을 신뢰하고, 차이는 `result`에 모두 보존되어 있으므로 이후 검증 UX에서 비교 표기. `result.difficulty_group`/`rough_level`/`boundary`로 분석기의 판단 강도(경계선 여부)까지 함께 비교 가능.
6. **분석기 인증**: README상 명세 없음 — 운영 도메인 접근 토큰 필요 여부 확인 후 어댑터별 env 키 추가.
7. **인덱스 전략**: `result` 전체 GIN + 분석기별 partial expression index. 신규 분석기 추가 시 자주 쓰는 키만 별도 partial index를 추가하는 패턴으로 운영. 명명 컨벤션은 `idx_<테이블명>_<순번>` (예: `idx_text_analyses_7`, `idx_text_analyses_8` ...) — 신규 인덱스는 다음 순번을 이어 받는다.
8. **재시도/캐시**: 동일 `(text_id, analyzer_type, analyzer_version)` 조합은 upsert로 idempotent. 본문 해시(`md5(content)`)를 `result.meta.content_hash`로 저장해두면 본문 변경 감지에 활용 가능.

---

## 10. 컨펌 필요 항목 (사용자와 함께 확정)

> 이 섹션은 구현 착수 전 결정이 필요한 항목 모음입니다. 각 항목 옆 ☐를 ☑로 바꾸고 선택 안 / 메모를 남겨주시면 본문 해당 절을 그에 맞게 갱신합니다.

### 10.1 환경변수 등록 채널 ☑ (a)
- **이슈**: 문서 6절은 `apps/api/.env`에 신규 키(`ANALYZER_API_URL`, `ANALYZER_DEFAULTS`, ...)를 둔다고 적었으나, 실제 로컬은 `vercel env pull`로 받은 `apps/api/.env.local`이 단일 소스. 운영도 Vercel에서 주입.
- **결정 (2026-04-26)**: **(a) 채택** — Vercel(`english-ai/studio-picklass-com-api`)의 Development/Preview/Production 환경변수에 직접 등록하고, 로컬은 `vercel env pull .env.local`로 동기화한다. 본문 6절을 그에 맞게 갱신함.

### 10.2 동기 호출의 응답 시간 영향 ☑ 동기 채택
- **이슈**: `PassagesService.create()`가 `runDefaults`를 동기 await 하므로 지문 저장 응답 시간이 analyzer 응답 시간만큼 늘어남.
- **결정 (2026-04-26)**: **동기 호출로 진행**한다. 사용자가 지문 생성 직후 8개 지표를 즉시 보는 UX를 우선시하고, 큐 인프라(Vercel `waitUntil` / Cron / 외부 큐) 도입에 따른 복잡도를 회피한다. 운영 중 응답 시간 이슈가 관측되면 4.5절(비동기 큐 적용 옵션)을 후속으로 검토.
- **운영 모니터링 기준**: analyzer 호출 p95가 3초를 지속적으로 초과하거나 사용자 체감 불만이 나오면 (A) Vercel `waitUntil` → (B) Cron 폴링 → (C) 외부 큐 순으로 단계적 전환을 검토.

### 10.3 `analyzer_version` 운영 방식 ☑ (a)
- **이슈**: analyzer 응답에 version 필드가 없어 어댑터 측에서 라벨을 직접 결정.
- **결정 (2026-04-26)**: **(a) 채택** — 어댑터 코드 상수로 고정 (`PicklassCefrAdapter.version = 'v2.0.0'`). 분석기 업그레이드 시 같은 PR에서 상수 갱신 → git blame/PR 이력으로 변경 시점 추적 명확. `ANALYZER_API_VERSION` env는 사용하지 않음.

### 10.4 권위 분석기 실패 시 `texts.level` 처리 ☑ (a)
- **이슈**: `text_type='I'`이고 `dto.level` 비어 있는데 `ANALYZER_LEVEL_AUTHORITY` 분석기가 실패하면 `texts.level`은 NULL로 남음.
- **결정 (2026-04-26)**: **(a) 채택** — `texts.level`은 NULL 유지, `text_analyses`에는 `status='failed'` 행 기록. 지문 생성 자체는 성공으로 마무리. UI 표시는 10.5 정책에 따라 `-` 한 글자로 통일(별도 재시도 버튼 없음). 명시적 재실행은 4.4절 `POST /passages/:id/analyses/refresh-all` 호출.

### 10.5 백필 정책 ☑ 백필 없음 + 미존재 시 `-` 표시
- **결정 (2026-04-26)**: **백필 수행하지 않음**. 기존 지문은 `text_analyses` 행이 없는 상태로 유지. 항목별 레벨 데이터가 없으면 UI는 단순히 **`-` (하이픈)** 한 글자만 표시한다(미정/분석 중/재시도 같은 별도 라벨/버튼 없음). 사용자가 옛날 지문에서 분석 결과가 필요하면 4.4절의 `POST /passages/:id/analyses/refresh-all`로 명시적으로 재실행 가능.

### 10.6 4.4 deprecated alias 유지 기간 ☑ (a) — alias 없이 즉시 교체
- **호출자 검색 결과 (2026-04-26)**:
  - NestJS 컨트롤러 정의 (`apps/api/src/passages/passages.controller.ts:70-82`) — 이번 PR에서 신규 경로로 교체 대상.
  - 프론트 래퍼 정의 (`apps/web/src/lib/api.ts:360-366`)에 `aiApi.analyzePassage`, `aiApi.refreshAnalysis`가 존재하나, **`apps/web/src` 어디에서도 호출하지 않는 dead code**.
  - 외부 모바일/외부 서비스 호출 없음 (사내 BFF).
- **결정 (2026-04-26)**: **(a) 채택** — alias 없이 본 PR에서 즉시 신규 경로(`/passages/:id/analyses[...]`)로 교체. 옛 경로 컨트롤러 핸들러 + 프론트 래퍼 두 함수(`aiApi.analyzePassage`, `aiApi.refreshAnalysis`)를 함께 삭제한다. 4.4절 본문의 "한 사이클 동안 deprecated alias" 문구도 제거.

### 10.7 3.4 최신 결과 뷰(`text_analyses_latest`) 채택 여부 ☑ (a) — 미채택
- **결정 (2026-04-26)**: **(a) 채택** — `text_analyses_latest` 뷰는 만들지 않는다. NestJS는 Prisma `findFirst({ orderBy: { analyzed_at: 'desc' } })` / `$queryRaw`(`DISTINCT ON`)로 동등 처리. 향후 동일한 raw 쿼리가 3곳 이상에서 반복되거나 권한 분리가 필요해지면 그때 뷰로 추출 검토.

### 10.8 노멀라이저 위치 (shared vs api) ☑ (a) — apps/api 단독
- **결정 (2026-04-26)**: **(a) 채택** — 노멀라이저 함수는 `apps/api/src/analyzer/normalizer/`에만 둔다. 백엔드가 노멀라이즈해서 `UiAnalysis` 모양으로 응답하므로 프론트는 raw를 보지 않는다. 단, **타입 정의(`UiAnalysisIndicator`, `UiAnalysis`, `PassageAnalysesResponse`)는 응답 DTO이므로 `packages/shared/src/types/analysis.ts`에 유지**해 백엔드/프론트가 공유한다.

### 10.9 `result.words` 저장 정책 ☑ (a) — 전체 저장
- **결정 (2026-04-26)**: **(a) 채택** — analyzer 응답을 가공 없이 통째로 `result` JSONB에 저장한다(`words` 배열 포함). 단어 하이라이트 등 후속 UI를 별도 호출 없이 즉시 구현 가능. 저장 공간/GIN 인덱스 비용은 현 단계 우선순위가 아니며, 운영 중 `text_analyses` 행 크기가 문제로 관측되면 (b)/(c)로 전환 검토. 분리 시점에는 `result.meta.has_words=false` 플래그 등으로 응답 호환성을 유지한다.
