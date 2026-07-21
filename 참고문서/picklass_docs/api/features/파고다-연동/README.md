# 파고다 ↔ 오이지(픽클래스) 연동 (Doc A · 10 API)

**계획 변경(2026-07-08)**: 파고다-오이지 연동 명세(Doc A)를 **문서 그대로, 10개 API 전부** 대응하기로 결정. 픽클래스가 Doc A의 **'오이지' 역할을 직접 수행**한다. (기존 [우유랩스 스피킹앱 연동](../우유랩스-스피킹앱-연동/)의 "Doc A는 픽클래스 당사자 아님 / 8·9만 Pull" 전제를 대체)

## 문서

- [2026-07-08_인터페이스-테이블-설계-API1-7.md](./2026-07-08_인터페이스-테이블-설계-API1-7.md) — **#1~7 인터페이스 테이블 설계**. 픽클래스 코어 무변경 + `pagoda_*` 참조 레이어 5종. v1.2 반영, 확정/보류, §10 구현 진척(#1~6·로그인 provisioning 완료)
- [2026-07-08_네이티브-투영-전략-결정대기.md](./2026-07-08_네이티브-투영-전략-결정대기.md) — **분기 없이** 파고다 데이터를 네이티브(`access_codes`·`institution_settings`)로 투영하는 전략. **결정 3건 확정(2026-07-11)** + 투영·게이트·classId 매핑 구현·e2e 통과(§7)
- [2026-07-11_결과송신-Push-재작업-설계.md](./2026-07-11_결과송신-Push-재작업-설계.md) — **#8·#9·#10 Push 재작업 설계**. 현행 GET Pull 분석 + 명세 대조 + 필드 매핑표 + 아웃박스 아키텍처 + 단계 계획. 착수 전(협의 의존 다수)

## 기준 명세

- **파고다-오이지 연동 API 명세서 v1.2** (2026-07-07, `C:\Users\MS\Desktop\api\`)
  - v1.1→v1.2: **#4에 `classInfo.classId`(필수) 추가**. 그 외 동일. (micMode는 v1.1에서 삭제)

## 방향 (10 API)

```
#1~6  파고다 → 오이지(픽클래스 수신)      : 수강신청·기수변경·환불·기간조정·종료일·설문공지
#7    오이지(픽클래스) → 파고다           : 로그인 인증 (아웃바운드)
#8~10 오이지(픽클래스) → 파고다           : 출석완료·수료완료·평가 (결과 송신)
```

- **#8·#9** 는 현재 `api.picklass.com`에 **GET Pull(방향 반대)** 로 임시 구현됨 → **Push 재작업 대상**. **#10** 미구현.

## 확정 (2026-07-08)

| 항목 | 결정 |
|---|---|
| classId 매핑 | **`courses.id`**(과정). ⚠️ 명세 v1.2 재확인(2026-07-11): 파고다는 **자기 콘텐츠 코드**(processId/detailId=`Y2GK7A` 형식 6자)만 보냄 — 우리 식별자 미포함. **매핑표 필수**(`pagoda_content_map`: detailId→`course_lessons.id`), course는 `course_lessons.course_id`로 파생. "detailId=우리 코드 직접연결" 전제는 **폐기** |
| attendance | 진도율(차시 완료율)로 계산 — 별도 출석 컬럼·요구처 없음 |
| #1~6 수신 인증 | `partner_api_keys`(Bearer) 우선, 검증 로직 강화는 추후 |
| 횟수(quota) | 파고다 전용 — grant=`class_cnt`(#1 부여·**#4 서비스티켓 갱신**), 소진=`enrollment_usage`. 네이티브=무제한 |
| comCd | 픽클래스 기관과 **매핑 불요** — `com_cd` 자립 앵커 |
| 신원 반영 | **로그인(#7) upsert** — #1~6은 미러 적재만, 로그인 시 users 있으면 update·없으면 insert |
| provisioning | `user_id`=`pagoda:{comCd}:{loginId}` · `institution_id`=`0364cb92-…`(파고다 고정 기관) · `name`=loginId · `password_hash`=sentinel(로컬 로그인 차단) |
| 수강 자격 | `access_codes` 실체화 안 함 — `pagoda_enrollments` read-through(EnrollmentProvider) |
| 소유 | `pagoda_*` 테이블은 **backoffice** 마이그레이션 소유, api/speaking 참조 |

## 추후 협의

- #7 인증 엔드포인트·auth_secret (전 API URI "미정") / 1~6 수신 인증 / processId·detailId 매핑표 / #8~10 결과 송신 인터페이스(별도 문서)

## 관련

- [우유랩스 스피킹앱 연동](../우유랩스-스피킹앱-연동/) — Doc B(3 API), 기존 전체분석
- 원문 파일: `C:\Users\MS\Desktop\api\` (파고다-오이지 v1.2 PDF 외)
