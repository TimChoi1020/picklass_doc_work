# 액세스코드 관리 시스템 - 개발 가이드 (Developer Guide)

## 목차

1. [개요](#개요)
2. [기술 스택](#기술-스택)
3. [프로젝트 구조](#프로젝트-구조)
4. [타입 정의](#타입-정의)
5. [API 설계](#api-설계)
6. [컴포넌트 구조](#컴포넌트-구조)
7. [상태 관리](#상태-관리)
8. [데이터베이스](#데이터베이스)
9. [구현 가이드](#구현-가이드)
10. [에러 처리](#에러-처리)
11. [테스트 가이드](#테스트-가이드)
12. [배포 체크리스트](#배포-체크리스트)

---

## 개요

액세스코드 관리 시스템은 Picklass 플랫폼에서 강사 및 학생의 접근을 제어하는 핵심 기능입니다.

**핵심 기능:**
- 액세스코드 일괄 생성
- 코드 상태 관리 (사용/미사용)
- 기간 및 과정 기반 접근 제어

**개발 기간:** 2주
**팀 구성:** 백엔드 2명, 프론트엔드 1명
**의존성:** Next.js 14+, React 18+, TypeScript 5+

---

## 기술 스택

### 프론트엔드
- **프레임워크**: Next.js 14 (App Router)
- **언어**: TypeScript
- **상태 관리**: React Hooks (useState, useContext)
- **스타일링**: inline CSS + CSS modules

### 백엔드
- **런타임**: Node.js 18+
- **프레임워크**: Express.js 또는 Next.js API Routes
- **데이터베이스**: PostgreSQL / MySQL
- **ORM**: Prisma 또는 TypeORM
- **캐싱**: Redis (선택)

### 기타
- **검증**: Zod, Yup
- **로깅**: Winston, Pino
- **테스트**: Jest, Vitest

---

## 프로젝트 구조

```
picklass-react/
├── app/
│   └── admin/
│       └── accesscode/
│           ├── page.tsx                    # 액세스코드 관리 페이지
│           ├── generate/
│           │   └── page.tsx               # 액세스코드 생성 페이지
│           └── layout.tsx
│
├── components/
│   └── admin/
│       ├── AccessCodeTable.tsx            # 테이블 컴포넌트
│       ├── AccessCodeStatsCards.tsx       # 통계 카드
│       ├── AccessCodeSearchFilter.tsx     # 검색 필터
│       └── Pagination.tsx                 # 페이지네이션
│
├── lib/
│   ├── accesscode.service.ts             # 비즈니스 로직
│   ├── accesscode.hooks.ts               # 커스텀 훅
│   ├── accesscode.validation.ts          # 유효성 검사
│   └── accesscode.utils.ts               # 유틸리티 함수
│
├── api/
│   ├── accesscode/
│   │   ├── route.ts                      # POST, GET
│   │   ├── [id]/
│   │   │   └── route.ts                  # PUT, DELETE
│   │   └── search/
│   │       └── route.ts                  # POST
│   └── health.ts
│
├── types/
│   └── accesscode.ts                     # 타입 정의
│
├── hooks/
│   └── useAccessCode.ts                  # 커스텀 훅
│
├── styles/
│   └── accesscode.module.css             # 스타일
│
└── docs/
    └── accesscode.md                     # 이 파일
```

---

## 타입 정의

### 1. 기본 타입 (`types/accesscode.ts`)

```typescript
// 액세스코드 상태
export enum AccessCodeStatus {
  UNUSED = 'UNUSED',      // 미사용
  ACTIVE = 'ACTIVE',      // 사용
}

// 사용자 유형
export enum UserType {
  TEACHER = 'teacher',
  STUDENT = 'student',
}

// 사용 기간
export enum UsagePeriod {
  ONE_MONTH = '1month',
  THREE_MONTHS = '3month',
  SIX_MONTHS = '6month',
  ONE_YEAR = '1year',
  UNLIMITED = 'unlimited',
}

// 과정 ID
export enum CourseId {
  COURSE_1 = 'course1',    // 영어 기초
  COURSE_2 = 'course2',    // 영어 심화
  COURSE_3 = 'course3',    // 비즈니스 영어
  COURSE_ALL = 'all',      // 전체 과정
}

// 기관 ID
export enum InstitutionId {
  INST_001 = 'inst001',    // 기술학원
  INST_002 = 'inst002',    // 개발자 학습소
  INST_003 = 'inst003',    // 언어교육전문학원
}

// ===== 데이터 모델 =====

// 액세스코드 생성 요청
export interface CreateAccessCodeRequest {
  userType: UserType
  institution: InstitutionId
  codeCount: number        // 1-1000
  expiryDate: string       // YYYY-MM-DD
  usagePeriod: UsagePeriod
  selectedCourse?: CourseId // 학생 선택 시 필수
}

// 액세스코드 생성 응답
export interface CreateAccessCodeResponse {
  success: boolean
  message: string
  data: {
    codes: AccessCodeItem[]
    createdCount: number
    skippedCount?: number
  }
}

// 액세스코드 항목
export interface AccessCodeItem {
  id: string                    // UUID
  code: string                  // 6자리 코드 (예: ABC123)
  status: AccessCodeStatus
  userType: UserType
  institution: InstitutionId
  expiryDate: string           // YYYY-MM-DD
  expiryDateFormatted: string  // 2026-05-01
  usagePeriod: UsagePeriod
  usagePeriodLabel: string     // "3개월"
  assignedCourse?: CourseId
  activateDate: string | null
  createdAt: string            // ISO 8601
  updatedAt: string
  createdBy: string            // 관리자 ID
}

// 액세스코드 목록 조회 응답
export interface AccessCodeListResponse {
  success: boolean
  data: {
    items: AccessCodeRecord[]
    total: number
    page: number
    pageSize: number
    totalPages: number
  }
  pagination: {
    currentPage: number
    totalPages: number
    hasNextPage: boolean
    hasPreviousPage: boolean
  }
}

// 테이블에 표시될 액세스코드 레코드
export interface AccessCodeRecord {
  code: string
  status: AccessCodeStatus
  role: string              // "강사" | "학생"
  institution: string       // 기관명
  userName: string | null
  expiryDate: string
  usagePeriod: string
  activateDate: string | null
}

// 상태 전환 요청
export interface UpdateAccessCodeStatusRequest {
  id: string
  newStatus: AccessCodeStatus
  reason?: string          // 변경 사유 (선택)
}

// 상태 전환 응답
export interface UpdateAccessCodeStatusResponse {
  success: boolean
  message: string
  data: {
    id: string
    code: string
    previousStatus: AccessCodeStatus
    newStatus: AccessCodeStatus
    updatedAt: string
  }
}

// 필터/검색 요청
export interface AccessCodeFilterRequest {
  status?: AccessCodeStatus
  userType?: UserType
  institution?: InstitutionId
  keyword?: string              // 코드 또는 사용자명
  expiryDateFrom?: string      // YYYY-MM-DD
  expiryDateTo?: string        // YYYY-MM-DD
  page: number
  pageSize: number
  sortBy?: 'createdAt' | 'expiryDate' | 'code'
  sortOrder?: 'asc' | 'desc'
}

// 통계 응답
export interface AccessCodeStatsResponse {
  success: boolean
  data: {
    total: number
    activeCount: number        // 사용 중
    unusedCount: number        // 미사용
    expiredCount: number       // 만료됨
    expiringCount: number      // 곧 만료 (30일 이내)
    byUserType: {
      teacher: number
      student: number
    }
  }
}

// ===== 폼 데이터 =====

// 생성 폼 상태
export interface AccessCodeFormState {
  userType: UserType
  institution: InstitutionId
  codeCount: string
  expiryDate: string
  usagePeriod: UsagePeriod
  selectedCourse: CourseId | null
}

// 폼 유효성 검사 에러
export interface ValidationError {
  field: string
  message: string
  code: string  // 에러 코드 (예: REQUIRED, INVALID_RANGE)
}

export interface ValidationResult {
  isValid: boolean
  errors: ValidationError[]
}
```

---

## API 설계

### 1. 액세스코드 생성 API

**Endpoint:** `POST /api/accesscode`

**요청:**
```typescript
// POST /api/accesscode
{
  "userType": "teacher",
  "institution": "inst001",
  "codeCount": 50,
  "expiryDate": "2026-05-01",
  "usagePeriod": "3month",
  "selectedCourse": null
}
```

**응답 (성공):**
```typescript
// 201 Created
{
  "success": true,
  "message": "50개의 강사 액세스코드가 생성되었습니다.",
  "data": {
    "codes": [
      {
        "id": "uuid-1",
        "code": "ABC123",
        "status": "UNUSED",
        "userType": "teacher",
        "institution": "inst001",
        "expiryDate": "2026-05-01",
        "usagePeriod": "3month",
        "createdAt": "2026-03-10T10:30:00Z"
      },
      // ... 더 많은 코드
    ],
    "createdCount": 50,
    "skippedCount": 0
  }
}
```

**응답 (실패):**
```typescript
// 400 Bad Request
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "유효성 검사 실패",
    "details": [
      {
        "field": "codeCount",
        "message": "최대 1,000개까지 생성 가능합니다."
      }
    ]
  }
}
```

### 2. 액세스코드 목록 조회 API

**Endpoint:** `GET /api/accesscode?page=1&pageSize=10&status=ACTIVE`

**쿼리 파라미터:**
```typescript
{
  page?: number              // 기본값: 1
  pageSize?: number          // 기본값: 10
  status?: 'ACTIVE' | 'UNUSED'
  userType?: 'teacher' | 'student'
  institution?: string
  keyword?: string           // 코드 또는 사용자명
  expiryDateFrom?: string   // YYYY-MM-DD
  expiryDateTo?: string     // YYYY-MM-DD
  sortBy?: 'createdAt' | 'expiryDate'
  sortOrder?: 'asc' | 'desc'
}
```

**응답:**
```typescript
// 200 OK
{
  "success": true,
  "data": {
    "items": [
      {
        "code": "ABC123",
        "status": "ACTIVE",
        "role": "강사",
        "institution": "기술학원",
        "userName": "Lee Teacher",
        "expiryDate": "2026-05-01",
        "usagePeriod": "3개월",
        "activateDate": "2026-02-01"
      }
      // ...
    ],
    "total": 245,
    "page": 1,
    "pageSize": 10,
    "totalPages": 25
  },
  "pagination": {
    "currentPage": 1,
    "totalPages": 25,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

### 3. 액세스코드 상태 업데이트 API

**Endpoint:** `PUT /api/accesscode/{id}`

**요청:**
```typescript
{
  "newStatus": "UNUSED",
  "reason": "사용자 요청"  // 선택사항
}
```

**응답:**
```typescript
// 200 OK
{
  "success": true,
  "message": "ABC123의 상태를 '미사용'으로 변경했습니다.",
  "data": {
    "id": "uuid-1",
    "code": "ABC123",
    "previousStatus": "ACTIVE",
    "newStatus": "UNUSED",
    "updatedAt": "2026-03-10T14:30:00Z"
  }
}
```

### 4. 통계 조회 API

**Endpoint:** `GET /api/accesscode/stats`

**응답:**
```typescript
// 200 OK
{
  "success": true,
  "data": {
    "total": 500,
    "activeCount": 320,
    "unusedCount": 150,
    "expiredCount": 20,
    "expiringCount": 10,
    "byUserType": {
      "teacher": 200,
      "student": 300
    }
  }
}
```

### 에러 코드 정의

| HTTP 상태 | 에러 코드 | 설명 |
|----------|----------|------|
| 400 | VALIDATION_ERROR | 유효성 검사 실패 |
| 400 | INVALID_RANGE | 범위 초과 (codeCount > 1000) |
| 400 | INVALID_DATE | 잘못된 날짜 형식 |
| 401 | UNAUTHORIZED | 인증 실패 |
| 403 | FORBIDDEN | 권한 없음 |
| 404 | NOT_FOUND | 리소스 찾음 없음 |
| 409 | CONFLICT | 중복 데이터 |
| 500 | INTERNAL_ERROR | 서버 오류 |

---

## 컴포넌트 구조

### 1. AccessCodeGeneration Page

**파일:** `app/admin/accesscode/generate/page.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createAccessCode } from '@/lib/accesscode.service'
import { validateCreateRequest } from '@/lib/accesscode.validation'
import type { CreateAccessCodeRequest, UserType } from '@/types/accesscode'

export default function AccessCodeGenerationPage() {
  const router = useRouter()
  const [formData, setFormData] = useState<Partial<CreateAccessCodeRequest>>({
    userType: 'teacher' as UserType,
    institution: undefined,
    codeCount: '',
    expiryDate: '',
    usagePeriod: undefined,
    selectedCourse: undefined,
  })
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError(null)

    // 유효성 검사
    const validation = validateCreateRequest(formData)
    if (!validation.isValid) {
      setError(validation.errors[0]?.message || '유효성 검사 실패')
      return
    }

    setLoading(true)
    try {
      const result = await createAccessCode(formData)
      alert(`${result.data.createdCount}개의 액세스코드가 생성되었습니다.`)
      router.push('/admin/accesscode')
    } catch (err) {
      setError(err instanceof Error ? err.message : '생성 실패')
    } finally {
      setLoading(false)
    }
  }

  return (
    <>
      <form onSubmit={handleSubmit}>
        {/* 폼 필드들 */}
        {error && <div className="error">{error}</div>}
        <button type="submit" disabled={loading}>
          {loading ? '생성 중...' : '생성'}
        </button>
      </form>
    </>
  )
}
```

### 2. AccessCodeTable Component

**파일:** `components/admin/AccessCodeTable.tsx`

```typescript
import type { AccessCodeRecord } from '@/types/accesscode'

interface Props {
  records: AccessCodeRecord[]
  onStatusToggle: (code: string) => Promise<void>
}

export default function AccessCodeTable({ records, onStatusToggle }: Props) {
  return (
    <table>
      <thead>
        <tr>
          <th>액세스코드</th>
          <th>코드상태</th>
          <th>역할</th>
          <th>소속</th>
          <th>사용자명</th>
          <th>등록만료일</th>
          <th>사용기간</th>
          <th>활성일</th>
          <th>작업</th>
        </tr>
      </thead>
      <tbody>
        {records.map((record) => (
          <tr key={record.code}>
            <td>{record.code}</td>
            <td>
              <Badge status={record.status} />
            </td>
            {/* ... 다른 셀들 ... */}
            <td>
              <ActionButton 
                status={record.status}
                onToggle={() => onStatusToggle(record.code)}
              />
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}

// 헬퍼 컴포넌트
function Badge({ status }: { status: string }) {
  const colors = {
    'ACTIVE': '#4CAF50',
    'UNUSED': '#FFC107',
  }
  return <span style={{ backgroundColor: colors[status] }}>{status}</span>
}

function ActionButton({ status, onToggle }: any) {
  return status === 'ACTIVE' ? (
    <button onClick={onToggle}>미사용 전환</button>
  ) : (
    <button onClick={onToggle}>사용 전환</button>
  )
}
```

---

## 상태 관리

### 1. Context API를 이용한 전역 상태

**파일:** `lib/accesscode.context.ts`

```typescript
import { createContext, useContext } from 'react'
import type { AccessCodeRecord } from '@/types/accesscode'

interface AccessCodeContextType {
  codes: AccessCodeRecord[]
  loading: boolean
  error: string | null
  fetchCodes: (filters?: any) => Promise<void>
  updateStatus: (id: string, newStatus: string) => Promise<void>
}

const AccessCodeContext = createContext<AccessCodeContextType | undefined>(undefined)

export function useAccessCode() {
  const context = useContext(AccessCodeContext)
  if (!context) {
    throw new Error('useAccessCode must be used within AccessCodeProvider')
  }
  return context
}
```

### 2. 커스텀 훅

**파일:** `hooks/useAccessCode.ts`

```typescript
import { useState, useCallback } from 'react'
import type { AccessCodeRecord, CreateAccessCodeRequest } from '@/types/accesscode'

export function useAccessCodeManagement() {
  const [codes, setCodes] = useState<AccessCodeRecord[]>([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const fetchCodes = useCallback(async (filters?: any) => {
    setLoading(true)
    try {
      const query = new URLSearchParams(filters)
      const response = await fetch(`/api/accesscode?${query}`)
      const data = await response.json()
      if (data.success) {
        setCodes(data.data.items)
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to fetch')
    } finally {
      setLoading(false)
    }
  }, [])

  const updateStatus = useCallback(async (id: string, newStatus: string) => {
    try {
      const response = await fetch(`/api/accesscode/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ newStatus }),
      })
      const data = await response.json()
      if (data.success) {
        // UI 업데이트
        setCodes(prev => 
          prev.map(code => 
            code.code === data.data.code 
              ? { ...code, status: newStatus }
              : code
          )
        )
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to update')
    }
  }, [])

  return { codes, loading, error, fetchCodes, updateStatus }
}
```

---

## 데이터베이스

### 1. 스키마 설계 (PostgreSQL)

```sql
-- 액세스코드 테이블
CREATE TABLE access_codes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code VARCHAR(10) UNIQUE NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'UNUSED',
  user_type VARCHAR(20) NOT NULL,        -- 'teacher' | 'student'
  institution_id VARCHAR(50) NOT NULL,
  expiry_date DATE NOT NULL,
  usage_period VARCHAR(50) NOT NULL,
  assigned_course VARCHAR(50),
  activated_by_user_id UUID,
  activated_at TIMESTAMP,
  created_by UUID NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (created_by) REFERENCES users(id),
  FOREIGN KEY (activated_by_user_id) REFERENCES users(id),
  INDEX idx_code (code),
  INDEX idx_status (status),
  INDEX idx_expiry_date (expiry_date),
  INDEX idx_institution (institution_id),
  INDEX idx_created_at (created_at)
)

-- 액세스코드 변경 이력 테이블
CREATE TABLE access_code_audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  access_code_id UUID NOT NULL,
  previous_status VARCHAR(20),
  new_status VARCHAR(20) NOT NULL,
  changed_by UUID NOT NULL,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  reason TEXT,
  
  FOREIGN KEY (access_code_id) REFERENCES access_codes(id),
  FOREIGN KEY (changed_by) REFERENCES users(id),
  INDEX idx_access_code_id (access_code_id),
  INDEX idx_changed_at (changed_at)
)
```

### 2. Prisma Schema

```prisma
model AccessCode {
  id                String    @id @default(cuid())
  code              String    @unique
  status            String    @default("UNUSED")    // "ACTIVE" | "UNUSED"
  userType          String                           // "teacher" | "student"
  institutionId     String
  expiryDate        DateTime
  usagePeriod       String                           // "1month", "3month", etc
  assignedCourse    String?
  activatedByUserId String?
  activatedAt       DateTime?
  createdBy         String
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  
  auditLogs         AuditLog[]
  
  @@index([code])
  @@index([status])
  @@index([expiryDate])
}

model AuditLog {
  id              String       @id @default(cuid())
  accessCodeId    String
  accessCode      AccessCode   @relation(fields: [accessCodeId], references: [id])
  previousStatus  String?
  newStatus       String
  changedBy       String
  changedAt       DateTime     @default(now())
  reason          String?
  
  @@index([accessCodeId])
  @@index([changedAt])
}
```

---

## 구현 가이드

### 1. 액세스코드 생성 로직

**파일:** `lib/accesscode.service.ts`

```typescript
import type { CreateAccessCodeRequest, CreateAccessCodeResponse } from '@/types/accesscode'

// 액세스코드 생성
export async function createAccessCode(
  request: CreateAccessCodeRequest
): Promise<CreateAccessCodeResponse> {
  // 1. 유효성 검사
  const validation = validateCreateRequest(request)
  if (!validation.isValid) {
    throw new Error(validation.errors[0]?.message)
  }

  // 2. 액세스코드 생성
  const codes = generateAccessCodes(request.codeCount)

  // 3. 데이터베이스에 저장
  const savedCodes = await db.accessCode.createMany({
    data: codes.map(code => ({
      code,
      status: 'UNUSED',
      userType: request.userType,
      institutionId: request.institution,
      expiryDate: new Date(request.expiryDate),
      usagePeriod: request.usagePeriod,
      assignedCourse: request.selectedCourse,
      createdBy: getCurrentUserId(),
    }))
  })

  return {
    success: true,
    message: `${request.codeCount}개의 ${request.userType === 'teacher' ? '강사' : '학생'} 액세스코드가 생성되었습니다.`,
    data: {
      codes: savedCodes.map(mapToAccessCodeItem),
      createdCount: savedCodes.length,
    }
  }
}

// 액세스코드 생성 알고리즘
function generateAccessCodes(count: number): string[] {
  const codes: string[] = []
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
  
  for (let i = 0; i < count; i++) {
    let code = ''
    for (let j = 0; j < 6; j++) {
      code += chars.charAt(Math.floor(Math.random() * chars.length))
    }
    // 중복 확인
    if (!codes.includes(code)) {
      codes.push(code)
    } else {
      i-- // 재시도
    }
  }
  
  return codes
}

// 상태 업데이트
export async function updateAccessCodeStatus(
  id: string,
  newStatus: string
) {
  const previousCode = await db.accessCode.findUnique({ where: { id } })
  
  const updated = await db.accessCode.update({
    where: { id },
    data: { 
      status: newStatus,
      updatedAt: new Date(),
    }
  })

  // 감시 로그 기록
  await db.auditLog.create({
    data: {
      accessCodeId: id,
      previousStatus: previousCode?.status,
      newStatus,
      changedBy: getCurrentUserId(),
    }
  })

  return updated
}
```

### 2. 유효성 검사

**파일:** `lib/accesscode.validation.ts`

```typescript
import { z } from 'zod'
import type { ValidationResult, CreateAccessCodeRequest } from '@/types/accesscode'

const createAccessCodeSchema = z.object({
  userType: z.enum(['teacher', 'student']),
  institution: z.string().min(1, '기관을 선택하세요'),
  codeCount: z.number()
    .min(1, '최소 1개 이상 입력하세요')
    .max(1000, '최대 1,000개까지 생성 가능합니다'),
  expiryDate: z.string()
    .refine(
      (date) => new Date(date) > new Date(),
      '등록 만료일은 미래 날짜여야 합니다'
    ),
  usagePeriod: z.enum(['1month', '3month', '6month', '1year', 'unlimited']),
  selectedCourse: z.string().optional()
    .refine(
      (course, ctx) => {
        if (ctx.parent.userType === 'student' && !course) {
          return false
        }
        return true
      },
      '학생 선택 시 과정을 선택하세요'
    ),
})

export function validateCreateRequest(data: any): ValidationResult {
  try {
    createAccessCodeSchema.parse(data)
    return { isValid: true, errors: [] }
  } catch (error) {
    if (error instanceof z.ZodError) {
      return {
        isValid: false,
        errors: error.errors.map(err => ({
          field: err.path.join('.'),
          message: err.message,
          code: err.code,
        }))
      }
    }
    return {
      isValid: false,
      errors: [{ field: 'unknown', message: '검증 중 오류 발생', code: 'UNKNOWN' }]
    }
  }
}
```

---

## 에러 처리

### 1. 에러 타입 정의

```typescript
export class AccessCodeError extends Error {
  constructor(
    public code: string,
    public message: string,
    public details?: any
  ) {
    super(message)
    this.name = 'AccessCodeError'
  }
}

export class ValidationError extends AccessCodeError {
  constructor(details: any) {
    super('VALIDATION_ERROR', '유효성 검사 실패', details)
  }
}

export class NotFoundError extends AccessCodeError {
  constructor(resource: string) {
    super('NOT_FOUND', `${resource}를 찾을 수 없습니다`)
  }
}

export class ConflictError extends AccessCodeError {
  constructor(message: string) {
    super('CONFLICT', message)
  }
}
```

### 2. 에러 핸들링 미들웨어

```typescript
export function errorHandler(error: Error, context: any) {
  if (error instanceof ValidationError) {
    return {
      status: 400,
      body: {
        success: false,
        error: {
          code: error.code,
          message: error.message,
          details: error.details,
        }
      }
    }
  }

  if (error instanceof NotFoundError) {
    return {
      status: 404,
      body: {
        success: false,
        error: {
          code: error.code,
          message: error.message,
        }
      }
    }
  }

  // 기본 에러
  console.error('[ERROR]', error)
  return {
    status: 500,
    body: {
      success: false,
      error: {
        code: 'INTERNAL_ERROR',
        message: '서버 오류가 발생했습니다.',
      }
    }
  }
}
```

---

## 테스트 가이드

### 1. 단위 테스트

**파일:** `lib/__tests__/accesscode.service.test.ts`

```typescript
import { describe, it, expect, vi } from 'vitest'
import { createAccessCode, generateAccessCodes } from '@/lib/accesscode.service'

describe('AccessCode Service', () => {
  describe('generateAccessCodes', () => {
    it('should generate unique codes', () => {
      const codes = generateAccessCodes(10)
      expect(codes).toHaveLength(10)
      expect(new Set(codes).size).toBe(10) // 중복 없음
    })

    it('should generate correct format codes', () => {
      const codes = generateAccessCodes(5)
      codes.forEach(code => {
        expect(code).toMatch(/^[A-Z0-9]{6}$/)
      })
    })
  })

  describe('createAccessCode', () => {
    it('should create codes successfully', async () => {
      const request = {
        userType: 'teacher',
        institution: 'inst001',
        codeCount: 5,
        expiryDate: '2026-12-31',
        usagePeriod: '3month',
      }

      const result = await createAccessCode(request)
      expect(result.success).toBe(true)
      expect(result.data.createdCount).toBe(5)
    })

    it('should reject invalid codeCount', async () => {
      const request = {
        userType: 'teacher',
        institution: 'inst001',
        codeCount: 1001,  // 초과
        expiryDate: '2026-12-31',
        usagePeriod: '3month',
      }

      expect(() => createAccessCode(request)).rejects.toThrow()
    })
  })
})
```

### 2. 통합 테스트

```typescript
describe('Access Code API Integration', () => {
  it('POST /api/accesscode should create codes', async () => {
    const response = await fetch('/api/accesscode', {
      method: 'POST',
      body: JSON.stringify({
        userType: 'teacher',
        institution: 'inst001',
        codeCount: 10,
        expiryDate: '2026-12-31',
        usagePeriod: '3month',
      })
    })

    expect(response.status).toBe(201)
    const data = await response.json()
    expect(data.success).toBe(true)
    expect(data.data.codes).toHaveLength(10)
  })
})
```

### 3. 테스트 체크리스트

- [ ] 액세스코드 생성 성공
- [ ] 1000개 이상 생성 거절
- [ ] 과거 날짜 거절
- [ ] 학생 선택 시 과정 필수 확인
- [ ] 상태 전환 성공
- [ ] 페이지네이션 동작
- [ ] 필터링 동작
- [ ] 검색 동작

---

## 배포 체크리스트

### 사전 검사
- [ ] 모든 테스트 통과 (`npm test`)
- [ ] 린팅 통과 (`npm run lint`)
- [ ] 빌드 성공 (`npm run build`)
- [ ] 환경 변수 설정 확인
- [ ] 데이터베이스 마이그레이션 완료

### 배포 단계
- [ ] 백업 생성
- [ ] 데이터베이스 스키마 업데이트
- [ ] API 서버 재시작
- [ ] 프론트엔드 번들 배포
- [ ] 캐시 무효화
- [ ] 스모크 테스트 수행

### 배포 후
- [ ] 에러 로그 모니터링
- [ ] 성능 메트릭 확인
- [ ] 사용자 피드백 수집
- [ ] 롤백 계획 준비

---

## 환경 변수

```env
# .env.local

# API
NEXT_PUBLIC_API_URL=http://localhost:3000
API_SECRET_KEY=your-secret-key

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/picklass
DB_LOG_LEVEL=info

# Cache
REDIS_URL=redis://localhost:6379

# Monitoring
SENTRY_DSN=https://xxx@sentry.io/xxx
LOG_LEVEL=debug
```

---

## 트러블슈팅

### 문제: "액세스코드 생성 속도 느림"
**해결:** 배치 생성 최적화, Redis 캐싱 추가

### 문제: "중복 코드 발생"
**해결:** 데이터베이스 UNIQUE 제약조건 확인, 생산 알고리즘 개선

### 문제: "페이지네이션 오류"
**해결:** OFFSET/LIMIT 계산 확인, 인덱스 추가

---

## 과정(Course) 데이터 연동 구현 계획

### 현황 분석

현재 액세스코드 생성 페이지(`apps/admin/frontend/src/app/(admin)/admin/accesscode/generate/page.tsx`)에서 "제공할 과정" 필드는 하드코딩된 옵션(`영어 기초`, `영어 심화`, `비즈니스 영어`, `전체 과정`)을 사용하고 있으며, DB에 Course 테이블이 존재하지 않는다.

**변경 목표:** 소속 기관(Institution)을 선택하면 해당 기관이 보유한 과정(Course) 목록을 API로 조회하여 동적으로 표시한다.

---

### 1단계: DB 스키마 — Course 모델 추가

**파일:** `prisma/schema.prisma`

```prisma
model Course {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  institutionId   String    @map("institution_id") @db.Uuid
  name            String    @db.VarChar(200)
  description     String?   @db.Text
  statusCode      String    @default("active") @map("status_code") @db.VarChar(50)
  sortOrder       Int       @default(0) @map("sort_order")
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt       DateTime? @map("deleted_at") @db.Timestamptz

  // Relations
  institution Institution @relation(fields: [institutionId], references: [id])

  @@index([institutionId])
  @@index([statusCode])
  @@map("courses")
}
```

**Institution 모델 변경:**
```prisma
// Institution에 courses 관계 추가
model Institution {
  // ... 기존 필드 ...
  courses     Course[]     // 추가
}
```

**AccessCode 모델 변경:**
```prisma
// AccessCode에 course 관계 추가
model AccessCode {
  // ... 기존 필드 ...
  course  Course? @relation(fields: [courseId], references: [id])  // 추가
}

// Course 모델에 역방향 관계 추가
model Course {
  // ... 기존 필드 ...
  accessCodes AccessCode[]  // 추가
}
```

**마이그레이션:**
```bash
pnpm prisma migrate dev --name add_course_model
```

---

### 2단계: 공유 타입 정의

**파일:** `packages/types/src/index.ts`

```typescript
// 과정 응답 DTO
export interface CourseResponse {
  id: string;
  institutionId: string;
  name: string;
  description: string | null;
  statusCode: string;
  sortOrder: number;
}

// 기관별 과정 조회 쿼리
export interface CourseQueryDto {
  institutionId: string;
  statusCode?: string;  // 기본값: 'active'
}
```

---

### 3단계: Backend — Course API 구현

#### 3-1. Core 모듈 (`packages/core`)

**파일:** `packages/core/src/course/course.module.ts`
```typescript
@Module({
  imports: [PrismaModule],
  providers: [CourseService],
  exports: [CourseService],
})
export class CourseModule {}
```

**파일:** `packages/core/src/course/course.service.ts`
```typescript
@Injectable()
export class CourseService {
  constructor(private prisma: PrismaService) {}

  // 기관별 과정 목록 조회
  async findByInstitution(institutionId: string, statusCode = 'active'): Promise<Course[]> {
    return this.prisma.course.findMany({
      where: {
        institutionId,
        statusCode,
        deletedAt: null,
      },
      orderBy: { sortOrder: 'asc' },
    });
  }
}
```

#### 3-2. Backend Controller (`apps/admin/backend`)

**파일:** `apps/admin/backend/src/course/course.controller.ts`

```typescript
@Controller('courses')
export class CourseController {
  constructor(private courseService: CourseService) {}

  // GET /courses?institutionId=xxx
  @Get()
  async getCoursesByInstitution(
    @Query('institutionId') institutionId: string,
  ): Promise<CourseResponse[]> {
    if (!institutionId) {
      throw new BadRequestException('institutionId는 필수입니다.');
    }
    return this.courseService.findByInstitution(institutionId);
  }
}
```

**신규 API 엔드포인트:**

| Method | Path | 설명 | Query Params |
|--------|------|------|-------------|
| GET | `/courses` | 기관별 과정 목록 조회 | `institutionId` (필수), `statusCode` (선택) |

**응답 예시:**
```json
[
  {
    "id": "uuid-course-1",
    "institutionId": "uuid-inst-1",
    "name": "영어 기초 리딩",
    "description": "초급자를 위한 리딩 과정",
    "statusCode": "active",
    "sortOrder": 1
  },
  {
    "id": "uuid-course-2",
    "institutionId": "uuid-inst-1",
    "name": "중급 리딩",
    "description": null,
    "statusCode": "active",
    "sortOrder": 2
  }
]
```

---

### 4단계: Frontend — 과정 동적 로딩

#### 4-1. API 함수 추가

**파일:** `apps/admin/frontend/src/lib/api/course.ts`

```typescript
import { fetchApi } from '../api';
import type { CourseResponse } from '@repo/types';

export async function getCoursesByInstitution(institutionId: string): Promise<CourseResponse[]> {
  return fetchApi<CourseResponse[]>('/courses', {
    params: { institutionId },
  });
}
```

**파일:** `apps/admin/frontend/src/lib/api.ts` — re-export 추가
```typescript
export * from './api/course';
```

#### 4-2. Generate 페이지 수정

**파일:** `apps/admin/frontend/src/app/(admin)/admin/accesscode/generate/page.tsx`

**변경 사항:**

1. **state 추가:** `courses` 배열, `coursesLoading` 로딩 상태
2. **institution 변경 시 과정 조회:** `useEffect`로 `institution` 값이 바뀔 때마다 `getCoursesByInstitution(institution)` 호출
3. **하드코딩 옵션 제거:** 기존 `<option value="course1">영어 기초</option>` 등을 동적 렌더링으로 교체
4. **과정 초기화:** 기관 변경 시 `selectedCourse`를 빈 값으로 리셋
5. **createAccessCodes 호출에 courseId 전달:** 선택된 과정의 실제 UUID를 전송

**수정 코드 스케치:**

```typescript
// 1. import 추가
import { getCoursesByInstitution } from '@/lib/api';
import type { CourseResponse } from '@repo/types';

// 2. state 추가
const [courses, setCourses] = useState<CourseResponse[]>([]);
const [coursesLoading, setCoursesLoading] = useState(false);

// 3. institution 변경 시 과정 조회
useEffect(() => {
  if (!institution) {
    setCourses([]);
    setSelectedCourse('');
    return;
  }
  setCoursesLoading(true);
  setSelectedCourse('');  // 기관 변경 시 과정 초기화
  getCoursesByInstitution(institution)
    .then(setCourses)
    .catch(console.error)
    .finally(() => setCoursesLoading(false));
}, [institution]);

// 4. 과정 select 렌더링 (하드코딩 → 동적)
{userType === 'student' && (
  <div>
    <label style={LABEL_STYLE}>제공할 과정 *</label>
    <select
      value={selectedCourse}
      onChange={(e) => setSelectedCourse(e.target.value)}
      required
      disabled={coursesLoading || courses.length === 0}
      style={FIELD_STYLE}
    >
      <option value="">
        {coursesLoading ? '과정 불러오는 중...' : courses.length === 0 ? '해당 기관에 등록된 과정이 없습니다' : '-- 과정을 선택하세요 --'}
      </option>
      {courses.map((c) => (
        <option key={c.id} value={c.id}>{c.name}</option>
      ))}
    </select>
  </div>
)}

// 5. submit 시 courseId 전달
await createAccessCodes({
  // ... 기존 필드
  courseId: selectedCourse || undefined,  // UUID 전달
});
```

---

### 5단계: 작업 순서 및 체크리스트

| 순서 | 작업 | 위치 | 의존성 |
|------|------|------|--------|
| 1 | Course 모델 추가 + 마이그레이션 | `prisma/schema.prisma` | 없음 |
| 2 | 공유 타입(CourseResponse, CourseQueryDto) 정의 | `packages/types` | 없음 |
| 3 | CourseService 구현 | `packages/core/src/course/` | 1 |
| 4 | CourseController + Module 등록 | `apps/admin/backend/src/course/` | 3 |
| 5 | `packages/core` 빌드 + 백엔드 재시작 | CLI | 3, 4 |
| 6 | 프론트엔드 API 함수 추가 | `apps/admin/frontend/src/lib/api/course.ts` | 2 |
| 7 | Generate 페이지 수정 (동적 과정 로딩) | `generate/page.tsx` | 4, 6 |
| 8 | 시드 데이터 추가 (테스트용 과정) | `prisma/seed.ts` | 1 |
| 9 | 통합 테스트 | - | 전체 |

### 주의사항

- `createAccessCodes` DTO에 `courseId` 필드는 이미 optional로 존재하므로 추가 수정 불필요
- `AccessCode` 스키마의 `courseId` 컬럼도 이미 존재 — Course FK 관계만 연결하면 됨
- 기관에 과정이 없는 경우 안내 메시지 표시 필요 (빈 select 방지)
- `packages/core` 수정 후 반드시 빌드(`pnpm run build`) + 백엔드 재시작

---

## 참고 문서

- [Next.js API Routes](https://nextjs.org/docs/api-routes/introduction)
- [Prisma Documentation](https://www.prisma.io/docs/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [Jest Testing Guide](https://jestjs.io/docs/)

