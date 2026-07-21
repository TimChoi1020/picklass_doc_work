# shared / conventions

picklass 조직의 표준 코딩 룰·컨벤션.

## 마스터 룰 위치

조직 표준 룰 마스터 문서는 현재 backoffice 저장소 내에 있다:

- 경로: `D:\Project\picklass-backoffice\.agent\rules\project_rules.md`
- 내용: 언어 정책, 타입 안전성(`any`/`@ts-ignore` 금지), 데이터 처리(parseInt, nullable FK),
  단방향 의존성, 작업 프로세스, 문서화 의무, 오류 디버깅 워크플로우.

## 사용

- 새 picklass 프로젝트 셋업 시 첫 참조 대상.
- 각 프로젝트 CLAUDE.md는 이 룰을 자체 컨텍스트에 맞춰 차용 후, 충돌이 있을 때만 자체 룰 우선.

## 향후 이전

이 마스터 문서는 단계적으로 `picklass_docs/shared/conventions/`로 이전할 예정.
이전 시 backoffice 저장소에는 본 위치를 가리키는 링크만 남긴다.
