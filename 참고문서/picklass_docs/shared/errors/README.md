# shared / errors

여러 서비스가 공유하는 인프라/계약 영역에서 발생한 오류 기록.

## 어디에 둘 것인가

- 한 서비스에서만 재현되는 오류 → 해당 서비스의 `errors/` (예: `speaking/errors/`)
- 인프라(Supabase, analyzer, Vercel) 또는 공유 계약(JWT, 공통코드)에서 발생 → 여기

## 형식

서비스별 `errors/`와 동일:

```markdown
## 증상 (Symptom)
## 원인 (Root Cause)
## 해결 (Resolution)
## 재발 방지 (Prevention)
```

영향 받은 서비스 목록을 문서 상단에 명시.
