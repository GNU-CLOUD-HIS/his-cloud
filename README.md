# his-cicd — 18090(쿠버네티스) 배포를 Jenkins CI/CD로 돌리기 위한 저장소

> **개인 CI/CD 실습 전용 저장소입니다.** 최종 배포(18080)·팀 프로젝트 규칙과 무관하며, PR 없이 `main`에 바로 커밋합니다.
> 이 저장소에 push하면 Jenkins가 18090에 자동 배포하므로, 담당자(강진기) 외에는 수정하지 말아 주세요.

자세한 절차: 프로젝트 문서 `claude/18090_CICD전환_작업가이드_20261026.md`

| 파일 | Jenkins job 이름 | 하는 일 |
|---|---|---|
| Jenkinsfile.infra | his-infra | 공통 ConfigMap + Redis·Kafka·SeaweedFS 적용 |
| Jenkinsfile.backend | his-svc-<서비스명> (10개) | jar 빌드 → 이미지 push → 배포 |
| Jenkinsfile.frontend | his-frontend | 프론트 이미지 빌드 → push → 배포 |
| Jenkinsfile.nginx | his-nginx | Nginx 설정·배포 → 18090 응답 확인 |
| Jenkinsfile.all | his-all | 위 job 을 순서대로 실행(전체 재구축) |

- `k8s/` 의 `__IMAGE__` 는 파이프라인이 채운다. 이 파일들을 직접 `kubectl apply` 하지 않는다.
- 비밀번호(Secret)는 이 저장소에 넣지 않는다. 서버에서 한 번만 `read -rsp` 로 만든다.
- `k8s/services/` 에는 labimaging·pharmacy 만 있다. 나머지 8개는 가이드 3-4 대로 만든다.
