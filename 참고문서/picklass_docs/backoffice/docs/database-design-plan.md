# Picklass 데이터베이스 설계 계획서

## 1. 개요

### 1.1 문서 정보
- **작성일**: 2026-03-08
- **버전**: v1.0
- **데이터베이스**: PostgreSQL
- **참조 문서**:
  - `picklass-playground/Picklass_policy_260306.md`
  - `picklass-playground/admin/users.html`
  - `picklass-playground/admin/institute.html`
  - `picklass-backoffice/docs/publishing-plan.md`

### 1.2 설계 규칙
| 항목 | 규칙 |
|------|------|
| 테이블명 | 알파벳 소문자 + snake_case |
| 컬럼명 | 알파벳 소문자 + snake_case |
| PK | `id` (UUID 또는 BIGSERIAL) |
| 외래키 | `{테이블명}_id` |
| 생성일시 | `created_at` (TIMESTAMPTZ) |
| 수정일시 | `updated_at` (TIMESTAMPTZ) |
| 삭제 | Soft Delete (`deleted_at` TIMESTAMPTZ NULL) |
| Boolean | `is_` prefix (예: `is_active`) |
| Enum | Common Code 테이블로 관리 |

---

## 2. 공통 코드 관리 테이블

### 2.1 설계 목적
- Select Box 옵션들을 하드코딩하지 않고 DB에서 관리
- 코드 추가/수정 시 애플리케이션 재배포 없이 변경 가능
- 다국어 지원 확장 가능
- 정렬 순서 지정 가능

### 2.2 테이블 구조

#### 2.2.1 `code_groups` (코드 그룹 테이블)

| 컬럼명 | 데이터 타입 | 제약조건 | 설명 |
|--------|-------------|----------|------|
| id | BIGSERIAL | PK | 자동 생성 ID |
| code | VARCHAR(50) | UNIQUE, NOT NULL | 그룹 코드 (영문) |
| name | VARCHAR(100) | NOT NULL | 그룹명 (한글) |
| description | TEXT | NULL | 그룹 설명 |
| is_active | BOOLEAN | DEFAULT TRUE | 활성화 여부 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 생성일시 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 수정일시 |

**인덱스**:
- `idx_code_groups_code` ON `code` (UNIQUE)

#### 2.2.2 `code_items` (코드 항목 테이블)

| 컬럼명 | 데이터 타입 | 제약조건 | 설명 |
|--------|-------------|----------|------|
| id | BIGSERIAL | PK | 자동 생성 ID |
| group_id | BIGINT | FK → code_groups.id, NOT NULL | 소속 그룹 |
| code | VARCHAR(50) | NOT NULL | 항목 코드 (영문) |
| name | VARCHAR(100) | NOT NULL | 항목명 (한글) |
| description | TEXT | NULL | 항목 설명 |
| sort_order | INTEGER | DEFAULT 0 | 정렬 순서 |
| extra_data | JSONB | NULL | 추가 데이터 (색상 등) |
| is_active | BOOLEAN | DEFAULT TRUE | 활성화 여부 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 생성일시 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 수정일시 |

**인덱스**:
- `idx_code_items_group_code` ON (`group_id`, `code`) (UNIQUE)
- `idx_code_items_group_id` ON `group_id`

**제약조건**:
- `fk_code_items_group` FOREIGN KEY (`group_id`) REFERENCES `code_groups`(`id`)

### 2.3 초기 데이터

#### 코드 그룹 (code_groups) 

| code | name | description |
|------|------|-------------|
| USER_ROLE | 사용자 역할 | 시스템관리자, 학원관리자, 강사, 학생 |
| USER_STATUS | 사용자 상태 | 활성, 비활성, 정지, 탈퇴 |
| INSTITUTION_TYPE | 기관 유형 | 개인 학원, 프랜차이즈, 어학원, 공교육, 기업교육 |
| OPERATION_TYPE | 운영 형태 | 직영, 가맹 |
| PLAN_TYPE | 요금제 플랜 | Lite, Pro, Enterprise, 제휴 |
| BILLING_CYCLE | 청구 주기 | 월납, 연납 |
| CONTRACT_STATUS | 계약 상태 | 협의중, 계약완료, 활성, 만료, 해지 |
| ACCESS_CODE_STATUS | 액세스코드 상태 | 활성, 비활성, 탈퇴 |

#### 코드 항목 (code_items)

**USER_ROLE (사용자 역할)**

| code | name | sort_order | extra_data |
|------|------|------------|------------|
| system_admin | 시스템관리자 | 1 | `{"color": "#8BC34A", "badge_bg": "#8BC34A"}` |
| academy_admin | 학원관리자 | 2 | `{"color": "#2196F3", "badge_bg": "#2196F3"}` |
| teacher | 강사 | 3 | `{"color": "#FF9800", "badge_bg": "#FF9800"}` |
| student | 학생 | 4 | `{"color": "#9C27B0", "badge_bg": "#9C27B0"}` |

**USER_STATUS (사용자 상태)**

| code | name | sort_order | extra_data |
|------|------|------------|------------|
| active | 활성 | 1 | `{"color": "#4CAF50", "icon": "check"}` |
| inactive | 비활성 | 2 | `{"color": "#FFC107", "icon": "clock"}` |
| suspended | 정지 | 3 | `{"color": "#f44336", "icon": "ban"}` |
| withdrawn | 탈퇴 | 4 | `{"color": "#9E9E9E", "icon": "user-x"}` |

**INSTITUTION_TYPE (기관 유형)**

| code | name | sort_order |
|------|------|------------|
| private_academy | 개인 학원 | 1 |
| franchise | 프랜차이즈 | 2 |
| language_school | 어학원 | 3 |
| public_education | 공교육 | 4 |
| corporate_training | 기업교육 | 5 |

**OPERATION_TYPE (운영 형태)**

| code | name | sort_order |
|------|------|------------|
| direct | 직영 | 1 |
| franchise | 가맹 | 2 |

**PLAN_TYPE (요금제 플랜)**

| code | name | sort_order | extra_data |
|------|------|------------|------------|
| lite | Lite | 1 | `{"price_per_student": 12000, "description": "월 10만원"}` |
| pro | Pro | 2 | `{"price_per_student": "8000-12000", "description": "월 100만원, 구간할인"}` |
| enterprise | Enterprise | 3 | `{"price_per_student": null, "description": "제휴 협의"}` |
| affiliate | 제휴 | 4 | `{"price_per_student": null, "description": "제휴 협의"}` |

**BILLING_CYCLE (청구 주기)**

| code | name | sort_order |
|------|------|------------|
| monthly | 월납 | 1 |
| annually | 연납 | 2 |

**CONTRACT_STATUS (계약 상태)**

| code | name | sort_order | extra_data |
|------|------|------------|------------|
| negotiating | 협의중 | 1 | `{"color": "#FFC107"}` |
| contracted | 계약완료 | 2 | `{"color": "#2196F3"}` |
| active | 활성 | 3 | `{"color": "#4CAF50"}` |
| expired | 만료 | 4 | `{"color": "#9E9E9E"}` |
| terminated | 해지 | 5 | `{"color": "#f44336"}` |

**ACCESS_CODE_STATUS (액세스코드 상태)**

| code | name | sort_order | extra_data |
|------|------|------------|------------|
| active | 활성 | 1 | `{"color": "#4CAF50"}` |
| inactive | 비활성 | 2 | `{"color": "#FFC107"}` |
| withdrawn | 탈퇴 | 3 | `{"color": "#9E9E9E"}` |

---

## 3. 기관 관리 테이블

### 3.1 테이블 구조

#### 3.1.1 `institutions` (기관 테이블)

| 컬럼명 | 데이터 타입 | 제약조건 | 설명 | 출처 |
|--------|-------------|----------|------|------|
| id | UUID | PK, DEFAULT gen_random_uuid() | 기관 고유 ID | - |
| **--- 가입정보 ---** | | | | |
| name | VARCHAR(200) | NOT NULL | 기관명 | 1. 가입정보 |
| institution_type_code | VARCHAR(50) | NOT NULL | 기관 유형 코드 (FK → code_items) | 1. 가입정보 |
| contact_name | VARCHAR(100) | NOT NULL | 담당자 성명 | 1. 가입정보 |
| contact_phone | VARCHAR(20) | NOT NULL | 담당자 연락처 | 1. 가입정보 |
| contact_email | VARCHAR(255) | NULL | 담당자 이메일 | 추가 |
| **--- 부가정보 ---** | | | | |
| branch_count | INTEGER | DEFAULT 0 | 지점 수 | 2. 부가정보 |
| operation_type_code | VARCHAR(50) | NOT NULL | 운영 형태 코드 | 2. 부가정보 |
| current_students | INTEGER | DEFAULT 0 | 현재 수강생 규모 | 2. 부가정보 |
| **--- 라이선스 & 요금제 ---** | | | | |
| plan_code | VARCHAR(50) | NOT NULL | 플랜 코드 | 3. 라이선스 |
| price_per_student | DECIMAL(10,2) | NULL | 학생당 단가 | 3. 라이선스 |
| max_students | INTEGER | NOT NULL | 최대 허용 학생수 | 3. 라이선스 |
| max_admin_accounts | INTEGER | DEFAULT 1 | 허용 관리계정수 | 3. 라이선스 |
| billing_cycle_code | VARCHAR(50) | NOT NULL | 청구 주기 코드 | 3. 라이선스 |
| annual_discount_rate | DECIMAL(5,2) | DEFAULT 0 | 연납 할인율 (%) | 3. 라이선스 |
| api_integration_fee | DECIMAL(12,2) | DEFAULT 0 | API 연동비 | 3. 라이선스 |
| tech_support_fee | DECIMAL(12,2) | DEFAULT 0 | 기술지원비 | 3. 라이선스 |
| content_ip_fee | DECIMAL(12,2) | DEFAULT 0 | 콘텐츠 IP 전환비 | 3. 라이선스 |
| notes | TEXT | NULL | 기타 유의사항 | 3. 라이선스 |
| **--- 계약 정보 ---** | | | | |
| contract_status_code | VARCHAR(50) | NOT NULL | 계약 상태 코드 | 4. 계약정보 |
| contract_start_date | DATE | NOT NULL | 계약 시작일 | 4. 계약정보 |
| contract_end_date | DATE | NOT NULL | 계약 종료일 | 4. 계약정보 |
| is_auto_renewal | BOOLEAN | DEFAULT FALSE | 자동갱신 여부 | 4. 계약정보 |
| renewal_conditions | TEXT | NULL | 갱신 조건 | 4. 계약정보 |
| **--- 시스템 ---** | | | | |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 생성일시 | - |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 수정일시 | - |
| deleted_at | TIMESTAMPTZ | NULL | 삭제일시 (Soft Delete) | - |

**인덱스**:
- `idx_institutions_name` ON `name`
- `idx_institutions_plan_code` ON `plan_code`
- `idx_institutions_contract_status` ON `contract_status_code`
- `idx_institutions_deleted_at` ON `deleted_at` WHERE `deleted_at IS NULL`

### 3.2 ERD (기관 관련)

```
code_groups (1) ──────< code_items (N)
                            │
                            │ (institution_type_code)
                            │ (operation_type_code)
                            │ (plan_code)
                            │ (billing_cycle_code)
                            │ (contract_status_code)
                            ▼
                      institutions (N)
```

---

## 4. 사용자 관리 테이블

### 4.1 테이블 구조

#### 4.1.1 `users` (사용자 테이블)

| 컬럼명 | 데이터 타입 | 제약조건 | 설명 | 출처 |
|--------|-------------|----------|------|------|
| id | UUID | PK, DEFAULT gen_random_uuid() | 사용자 고유 ID | - |
| institution_id | UUID | FK → institutions.id, NULL | 소속 기관 ID | 모달 폼 |
| role_code | VARCHAR(50) | NOT NULL | 역할 코드 | 모달 폼 |
| user_id | VARCHAR(255) | UNIQUE, NOT NULL | 로그인 아이디 | 모달 폼 |
| password_hash | VARCHAR(255) | NOT NULL | 비밀번호 해시 | 모달 폼 |
| name | VARCHAR(100) | NOT NULL | 사용자 이름 | 모달 폼 |
| email | VARCHAR(255) | NULL | 이메일 주소 | 추가 |
| phone | VARCHAR(20) | NULL | 연락처 | 추가 |
| status_code | VARCHAR(50) | NOT NULL DEFAULT 'active' | 상태 코드 | 모달 폼 |
| is_temp_password | BOOLEAN | DEFAULT TRUE | 임시 비밀번호 여부 | Policy 5.3 |
| last_login_at | TIMESTAMPTZ | NULL | 마지막 로그인 일시 | 추가 |
| activated_at | TIMESTAMPTZ | NULL | 활성화 일시 | 테이블 표시 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 생성일시 | - |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 수정일시 | - |
| deleted_at | TIMESTAMPTZ | NULL | 삭제일시 (Soft Delete) | - |

**인덱스**:
- `idx_users_user_id` ON `user_id` (UNIQUE)
- `idx_users_institution_id` ON `institution_id`
- `idx_users_role_code` ON `role_code`
- `idx_users_status_code` ON `status_code`
- `idx_users_email` ON `email`
- `idx_users_deleted_at` ON `deleted_at` WHERE `deleted_at IS NULL`

**제약조건**:
- `fk_users_institution` FOREIGN KEY (`institution_id`) REFERENCES `institutions`(`id`)

### 4.2 테이블 구조

#### 4.2.1 `access_codes` (액세스코드 테이블)

| 컬럼명 | 데이터 타입 | 제약조건 | 설명 | 출처 |
|--------|-------------|----------|------|------|
| id | UUID | PK, DEFAULT gen_random_uuid() | 액세스코드 고유 ID | - |
| code | CHAR(6) | UNIQUE, NOT NULL | 6자리 액세스코드 | Policy 5.3 |
| institution_id | UUID | FK → institutions.id, NOT NULL | 소속 기관 ID | 모달 폼 |
| role_code | VARCHAR(50) | NOT NULL | 역할 코드 (teacher/student) | 모달 폼 |
| user_id | UUID | FK → users.id, NULL | 연결된 사용자 ID | Policy 5.3 |
| course_id | UUID | FK → courses.id, NULL | 과정 ID (학생만) | 모달 폼 |
| status_code | VARCHAR(50) | NOT NULL DEFAULT 'inactive' | 코드 상태 | 모달 폼 |
| registration_expiry | DATE | NOT NULL | 등록 유효기간 | 모달 폼 |
| usage_period_days | INTEGER | NOT NULL | 사용가능기간 (일) | 모달 폼 |
| usage_start_date | DATE | NULL | 사용 시작일 | Policy 5.3 |
| usage_end_date | DATE | NULL | 사용 종료일 (계산값) | Policy 5.3 |
| generated_user_id | VARCHAR(255) | NULL | 자동 생성된 아이디 | 모달 폼 |
| generated_password | VARCHAR(255) | NULL | 자동 생성된 임시 비밀번호 | 모달 폼 |
| activated_at | TIMESTAMPTZ | NULL | 활성화 일시 | Policy 5.3 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 생성일시 | - |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 수정일시 | - |

**인덱스**:
- `idx_access_codes_code` ON `code` (UNIQUE)
- `idx_access_codes_institution_id` ON `institution_id`
- `idx_access_codes_user_id` ON `user_id`
- `idx_access_codes_status_code` ON `status_code`
- `idx_access_codes_registration_expiry` ON `registration_expiry`

**제약조건**:
- `fk_access_codes_institution` FOREIGN KEY (`institution_id`) REFERENCES `institutions`(`id`)
- `fk_access_codes_user` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`)
- `chk_access_codes_code_format` CHECK (`code ~ '^[A-HJ-NP-Z2-9]{6}$'`) -- I,O,0,1 제외

### 4.3 ERD (사용자 관련)

```
                     code_items
                         │
                         │ (role_code, status_code)
                         ▼
institutions (1) ────< users (N)
      │                  │
      │                  │ (user_id)
      │                  ▼
      └────────< access_codes (N)
```

---

## 5. 전체 ERD

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CODE MANAGEMENT                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐         ┌──────────────┐                                 │
│  │ code_groups  │ (1)──<  │ code_items   │                                 │
│  │──────────────│         │──────────────│                                 │
│  │ id           │         │ id           │                                 │
│  │ code (UK)    │         │ group_id (FK)│                                 │
│  │ name         │         │ code         │                                 │
│  │ description  │         │ name         │                                 │
│  │ is_active    │         │ sort_order   │                                 │
│  └──────────────┘         │ extra_data   │                                 │
│                           │ is_active    │                                 │
│                           └──────────────┘                                 │
│                                  │                                          │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │
           ┌───────────────┬───────┴───────┬────────────────┐
           │               │               │                │
           ▼               ▼               ▼                ▼
  (institution_type) (operation_type)  (plan)      (contract_status)
  (billing_cycle)
           │               │               │                │
           └───────────────┴───────┬───────┴────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            INSTITUTION MANAGEMENT                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────┐                               │
│  │ institutions                             │                               │
│  │─────────────────────────────────────────│                               │
│  │ id (PK)                                 │                               │
│  │ name                                    │ ◀── 가입정보                  │
│  │ institution_type_code (FK → code_items) │                               │
│  │ contact_name, contact_phone             │                               │
│  │─────────────────────────────────────────│                               │
│  │ branch_count                            │ ◀── 부가정보                  │
│  │ operation_type_code (FK → code_items)   │                               │
│  │ current_students                        │                               │
│  │─────────────────────────────────────────│                               │
│  │ plan_code (FK → code_items)             │ ◀── 라이선스                  │
│  │ max_students, max_admin_accounts        │                               │
│  │ billing_cycle_code (FK → code_items)    │                               │
│  │ annual_discount_rate, api_integration_fee│                              │
│  │─────────────────────────────────────────│                               │
│  │ contract_status_code (FK → code_items)  │ ◀── 계약정보                  │
│  │ contract_start_date, contract_end_date  │                               │
│  │ is_auto_renewal, renewal_conditions     │                               │
│  └─────────────────────────────────────────┘                               │
│                          │                                                  │
└──────────────────────────┼──────────────────────────────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER MANAGEMENT                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐│
│  │ users                           │    │ access_codes                    ││
│  │─────────────────────────────────│    │─────────────────────────────────││
│  │ id (PK)                         │◀───│ user_id (FK, NULL)              ││
│  │ institution_id (FK)             │    │ id (PK)                         ││
│  │ role_code (FK → code_items)     │    │ code (UK, 6자리)                ││
│  │ user_id (UK)                    │    │ institution_id (FK)             ││
│  │ password_hash                   │    │ role_code (teacher/student)     ││
│  │ name                            │    │ course_id (FK, NULL)            ││
│  │ status_code (FK → code_items)   │    │ status_code (FK → code_items)   ││
│  │ is_temp_password                │    │ registration_expiry             ││
│  │ activated_at                    │    │ usage_period_days               ││
│  │ last_login_at                   │    │ generated_user_id               ││
│  └─────────────────────────────────┘    │ activated_at                    ││
│                                         └─────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Prisma Schema 설계

### 6.1 schema.prisma

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// =====================
// 공통 코드 관리
// =====================

model CodeGroup {
  id          BigInt     @id @default(autoincrement())
  code        String     @unique @db.VarChar(50)
  name        String     @db.VarChar(100)
  description String?    @db.Text
  isActive    Boolean    @default(true) @map("is_active")
  createdAt   DateTime   @default(now()) @map("created_at") @db.Timestamptz
  updatedAt   DateTime   @updatedAt @map("updated_at") @db.Timestamptz

  items       CodeItem[]

  @@map("code_groups")
}

model CodeItem {
  id          BigInt     @id @default(autoincrement())
  groupId     BigInt     @map("group_id")
  code        String     @db.VarChar(50)
  name        String     @db.VarChar(100)
  description String?    @db.Text
  sortOrder   Int        @default(0) @map("sort_order")
  extraData   Json?      @map("extra_data") @db.JsonB
  isActive    Boolean    @default(true) @map("is_active")
  createdAt   DateTime   @default(now()) @map("created_at") @db.Timestamptz
  updatedAt   DateTime   @updatedAt @map("updated_at") @db.Timestamptz

  group       CodeGroup  @relation(fields: [groupId], references: [id])

  @@unique([groupId, code])
  @@index([groupId])
  @@map("code_items")
}

// =====================
// 기관 관리
// =====================

model Institution {
  id                    String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid

  // 가입정보
  name                  String    @db.VarChar(200)
  institutionTypeCode   String    @map("institution_type_code") @db.VarChar(50)
  contactName           String    @map("contact_name") @db.VarChar(100)
  contactPhone          String    @map("contact_phone") @db.VarChar(20)
  contactEmail          String?   @map("contact_email") @db.VarChar(255)

  // 부가정보
  branchCount           Int       @default(0) @map("branch_count")
  operationTypeCode     String    @map("operation_type_code") @db.VarChar(50)
  currentStudents       Int       @default(0) @map("current_students")

  // 라이선스 & 요금제
  planCode              String    @map("plan_code") @db.VarChar(50)
  pricePerStudent       Decimal?  @map("price_per_student") @db.Decimal(10, 2)
  maxStudents           Int       @map("max_students")
  maxAdminAccounts      Int       @default(1) @map("max_admin_accounts")
  billingCycleCode      String    @map("billing_cycle_code") @db.VarChar(50)
  annualDiscountRate    Decimal   @default(0) @map("annual_discount_rate") @db.Decimal(5, 2)
  apiIntegrationFee     Decimal   @default(0) @map("api_integration_fee") @db.Decimal(12, 2)
  techSupportFee        Decimal   @default(0) @map("tech_support_fee") @db.Decimal(12, 2)
  contentIpFee          Decimal   @default(0) @map("content_ip_fee") @db.Decimal(12, 2)
  notes                 String?   @db.Text

  // 계약 정보
  contractStatusCode    String    @map("contract_status_code") @db.VarChar(50)
  contractStartDate     DateTime  @map("contract_start_date") @db.Date
  contractEndDate       DateTime  @map("contract_end_date") @db.Date
  isAutoRenewal         Boolean   @default(false) @map("is_auto_renewal")
  renewalConditions     String?   @map("renewal_conditions") @db.Text

  // 시스템
  createdAt             DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt             DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt             DateTime? @map("deleted_at") @db.Timestamptz

  // Relations
  users                 User[]
  accessCodes           AccessCode[]

  @@index([name])
  @@index([planCode])
  @@index([contractStatusCode])
  @@map("institutions")
}

// =====================
// 사용자 관리
// =====================

model User {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  institutionId   String?   @map("institution_id") @db.Uuid
  roleCode        String    @map("role_code") @db.VarChar(50)
  userId          String    @unique @map("user_id") @db.VarChar(255)
  passwordHash    String    @map("password_hash") @db.VarChar(255)
  name            String    @db.VarChar(100)
  email           String?   @db.VarChar(255)
  phone           String?   @db.VarChar(20)
  statusCode      String    @default("active") @map("status_code") @db.VarChar(50)
  isTempPassword  Boolean   @default(true) @map("is_temp_password")
  lastLoginAt     DateTime? @map("last_login_at") @db.Timestamptz
  activatedAt     DateTime? @map("activated_at") @db.Timestamptz
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt       DateTime? @map("deleted_at") @db.Timestamptz

  // Relations
  institution     Institution? @relation(fields: [institutionId], references: [id])
  accessCodes     AccessCode[]

  @@index([institutionId])
  @@index([roleCode])
  @@index([statusCode])
  @@index([email])
  @@map("users")
}

model AccessCode {
  id                  String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  code                String    @unique @db.Char(6)
  institutionId       String    @map("institution_id") @db.Uuid
  roleCode            String    @map("role_code") @db.VarChar(50)
  userId              String?   @map("user_id") @db.Uuid
  courseId            String?   @map("course_id") @db.Uuid
  statusCode          String    @default("inactive") @map("status_code") @db.VarChar(50)
  registrationExpiry  DateTime  @map("registration_expiry") @db.Date
  usagePeriodDays     Int       @map("usage_period_days")
  usageStartDate      DateTime? @map("usage_start_date") @db.Date
  usageEndDate        DateTime? @map("usage_end_date") @db.Date
  generatedUserId     String?   @map("generated_user_id") @db.VarChar(255)
  generatedPassword   String?   @map("generated_password") @db.VarChar(255)
  activatedAt         DateTime? @map("activated_at") @db.Timestamptz
  createdAt           DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt           DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relations
  institution         Institution @relation(fields: [institutionId], references: [id])
  user                User?       @relation(fields: [userId], references: [id])

  @@index([institutionId])
  @@index([userId])
  @@index([statusCode])
  @@index([registrationExpiry])
  @@map("access_codes")
}
```

---

## 7. 구현 순서

| 단계 | 작업 | 파일 |
|------|------|------|
| 1 | Prisma 스키마 생성 | `prisma/schema.prisma` |
| 2 | 마이그레이션 실행 | `npx prisma migrate dev` |
| 3 | 초기 데이터 시드 | `prisma/seed.ts` |
| 4 | Prisma Client 생성 | `npx prisma generate` |

---

## 8. 검증 체크리스트

### 8.1 테이블 생성 검증
- [ ] `code_groups` 테이블 생성 확인
- [ ] `code_items` 테이블 생성 확인
- [ ] `institutions` 테이블 생성 확인
- [ ] `users` 테이블 생성 확인
- [ ] `access_codes` 테이블 생성 확인

### 8.2 초기 데이터 검증
- [ ] 8개 코드 그룹 생성 확인
- [ ] 각 그룹별 코드 항목 생성 확인
- [ ] extra_data JSON 형식 검증

### 8.3 제약조건 검증
- [ ] PK/FK 제약조건 동작 확인
- [ ] UNIQUE 제약조건 동작 확인
- [ ] 액세스코드 형식 CHECK 제약조건 확인

### 8.4 인덱스 검증
- [ ] 검색 성능 테스트 (기관명, 사용자명 등)
- [ ] FK 조회 성능 테스트

---

## 9. 추후 확장 계획

| 테이블 | 설명 | 우선순위 |
|--------|------|----------|
| courses | 과정 관리 | 높음 |
| lessons | 레슨 관리 | 높음 |
| passages | 지문 관리 | 높음 |
| ai_modules | AI 수업 모듈 관리 | 중간 |
| billing_history | 청구 내역 | 중간 |
| learning_logs | 학습 로그 | 낮음 |

---

*본 문서는 데이터베이스 설계 초안이며, 개발 진행 시 협의를 통해 업데이트됩니다.*
