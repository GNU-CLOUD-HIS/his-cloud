# his-cicd — 18090(쿠버네티스) 팀 CI/CD 실습 저장소

> **팀 CI/CD 실습 전용 저장소입니다.** 최종 배포(18080)와 무관합니다. 팀 프로젝트 규칙(PR·리뷰)은 적용하지 않고 `main`에 바로 push 합니다.
> 각자 **본인 Jenkins**에서 **본인 담당 폴더**의 Jenkinsfile 로 job 을 만들어, 같은 클러스터(원격 서버 minikube, namespace `his`)에 배포합니다.

전체 절차: `CICD_팀실습_가이드.md` (이 저장소에 함께 있음)

## 담당표

| 대상 | 폴더 | 담당자 | 자동 배포(pollSCM) |
|---|---|---|---|
| 인프라: 공통 설정 + Redis·Kafka·SeaweedFS | `infra/` | (정하기) | 없음, 담당자가 공지 후 직접 실행 |
| nginx (18090 입구) | `nginx/` | (정하기, 인프라 담당 겸임 권장) | 없음, 담당자가 직접 실행 |
| 프론트 | `frontend/` | (정하기) | 있음 |
| admin | `services/admin/` | 권수근 | 있음 |
| patient | `services/patient/` | 홍예진 | 있음 |
| lab-imaging | `services/labimaging/` | 강진기 | 있음 |
| pharmacy | `services/pharmacy/` | 강진기,이우성 | 있음 |
| billing | `services/billing/` | 정대훈 | 있음 |
| reception | `services/reception/` | 최인경 | 있음 |
| surgery | `services/surgery/` | (담당자) | 있음 |
| inpatient | `services/inpatient/` | 김가영 | 있음 |
| outpatient | `services/outpatient/` | 서호령 | 있음 |
| emergency | `services/emergency/` | 박상민 | 있음 |

## 팀 규칙 (6가지)

1. **내 폴더만 고친다.** 다른 사람 폴더는 열어 보기만 한다
2. **commit 한 뒤, push 전에 `git pull --rebase`.** 여럿이 `main`에 바로 push 하므로 안 하면 `rejected (non-fast-forward)`가 난다. 커밋 전에 하면 `You have unstaged changes`로 거절된다
3. **내 서비스 리소스만 배포한다.** 서비스 Jenkinsfile 은 `k8s.yaml`에 내 것(`<서비스>-config`, `<서비스>` Deployment·Service) 말고 다른 것이 있으면 배포 전에 멈춘다
4. **공통 설정(`his-common-config`)·Redis·Kafka·SeaweedFS·nginx 는 담당자만 바꾼다.** 필요하면 담당자에게 요청
5. **비밀번호·토큰은 이 저장소에 넣지 않는다.** 서버 Secret 과 각자 Jenkins Credentials 에만
6. **본인 Jenkins 의 job 설정에 Included Regions(내 폴더)를 건다.** 안 걸면 남이 push 할 때마다 내 job 도 돈다

## 폴더 구조

```
infra/      Jenkinsfile, common.yaml
nginx/      Jenkinsfile, k8s.yaml
frontend/   Jenkinsfile, k8s.yaml
services/<서비스>/   Jenkinsfile (CFG 6줄만 채움), k8s.yaml (각 담당자가 서버에서 꺼내 넣음)
```

`k8s.yaml`의 `__IMAGE__`는 파이프라인이 채웁니다. 이 파일들을 직접 `kubectl apply` 하지 않습니다.
