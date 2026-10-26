# 18090(쿠버네티스) 팀 CI/CD 실습 가이드

작성: 2026-10-26 (개정: 개인 실습안 → 팀 전체 실습안)
대상: 원격 서버(seoulit.pe.kr, 계정 guest4)의 minikube 클러스터, namespace `his`
저장소: `https://github.com/SEOUL-IT-HIS/his-cicd` (실습 전용, 팀 프로젝트 규칙 미적용)
성격: **최종 배포와 무관한 팀 학습.** 18080(최종 배포본, 터미널 실행)은 그대로 둔다
목표: **각자 본인 Jenkins로 본인 담당 MSA를 직접 빌드·배포한다.** 인프라·nginx·프론트는 한두 명이 맡는다
근거: `06_배포·인프라\k8s_원격배포\` 의 가이드와 실제 적용한 YAML(`common.yaml`, `hisfront-nginx-v1.0.yaml`, `labimaging.yaml`, `pharmacy.yaml`)

---

## 0. 먼저 결론

### Q0. 팀원마다 Jenkins가 따로(각자 PC) 있고 노드도 각자 만들었다. 실습 환경으로 괜찮나?

**괜찮습니다. 이번 목표("내 서비스는 내가 배포")에는 오히려 맞는 구조입니다.**

| 관점 | 각자 Jenkins (지금) | Jenkins 하나를 같이 씀 (실무에 더 흔함) |
|---|---|---|
| 내 파이프라인을 내가 만들고 고침 | 그대로 됨 | 권한을 나눠 줘야 함 |
| 준비 | 이미 되어 있음 | 모두가 접속할 수 있는 서버에 Jenkins를 새로 띄워야 함(지금 Jenkins는 각자 PC의 localhost) |
| 누가 무엇을 배포했는지 한눈에 | 안 보임 → `rollout history`로 보완(이 가이드의 Jenkinsfile이 기록을 남김) | Jenkins 화면에서 보임 |
| 한 사람 PC가 꺼지면 | 그 사람 서비스의 자동 배포만 멈춤. 떠 있는 Pod는 그대로 | Jenkins 서버가 꺼지면 전원이 멈춤 |
| 비밀번호(토큰) | 각자 본인 Docker Hub 토큰을 본인 Jenkins에 | 한곳에 모임 |

배포 대상(클러스터)은 **하나를 같이** 쓰므로, 따로 쓰는 Jenkins끼리 부딪히지 않게 **규칙 6가지**를 지킵니다(3장). 이 규칙은 Jenkinsfile과 job 설정에 대부분 들어 있어서, 각자 지켜야 할 것은 "내 폴더만 고친다" 정도입니다.

### Q1. 원격에 돌고 있는 클러스터부터 멈추고 시작해야 하나?

**아니요. 클러스터(minikube)는 멈추지도 지우지도 않습니다.**

| 이유 | 설명 |
|---|---|
| 파이프라인의 배포 대상이 그 클러스터다 | 배포 단계는 `kubectl apply`로 **켜져 있는 클러스터**에 "이 상태로 맞춰 줘"라고 요청한다 |
| 18090 연결이 클러스터에 붙어 있다 | `18090 → 30080`은 `minikube start --ports=18090:30080`으로 클러스터를 만들 때 정해졌다. `minikube delete` 하면 사라진다 |
| 남겨야 할 것이 클러스터 안에 있다 | 비밀번호(Secret 11개)와 SeaweedFS 저장 공간(PVC)은 파이프라인이 만들지 않는다 |

"18090을 완전히 내린다"는 **클러스터가 아니라 그 안의 앱(Deployment·Service·ConfigMap)을 지우는 것**으로 합니다(8장).

### Q2. 기존에 돌고 있는 것을 그대로 두어도 CI/CD가 적용되나?

**예.** `kubectl apply`는 "없으면 만들고, 있으면 파일대로 맞춘다"는 명령이라, 각자 파이프라인이 자기 Deployment에 새 이미지를 apply하면 **새 Pod가 준비된 뒤 옛 Pod를 내리는 롤링 업데이트**가 됩니다(`maxUnavailable: 0`, 무중단). 그래서 각자 **아무 때나 자기 서비스부터** 옮겨 탈 수 있습니다(7장).

| 알아둘 점 | 내용 |
|---|---|
| 손으로 바꾼 값은 파일 값으로 되돌아간다 | `kubectl scale`·`edit`으로 바꾼 값은 apply 때 YAML 값으로 덮인다. 그래서 5-4에서 **지금 실제로 돌고 있는 값**으로 YAML을 만든다 |
| 경고 한 줄 | apply가 아닌 방법으로 만든 오브젝트에 처음 apply하면 `missing the kubectl.kubernetes.io/last-applied-configuration annotation` 경고. 자동으로 붙이므로 무시 |
| 이미지 주인이 바뀐다 | 각 서비스 이미지가 **담당자 본인 Docker Hub의 `his-<서비스>:b<빌드번호>-<커밋>`** 으로 바뀐다 |

### 18080(최종 배포본)에 영향이 있나?

18080은 쿠버네티스 밖의 터미널 프로세스라 **직접 상관이 없습니다.** 다만 같이 쓰는 것이 있습니다.

| 같이 쓰는 것 | 조심할 점 |
|---|---|
| **Oracle DB** (`seoulit.pe.kr:1521/freepdb1`, 같은 스키마) | 18090 테스트 데이터가 18080에도 보인다. **`spring.jpa.hibernate.ddl-auto`가 `create`/`create-drop`인 서비스가 18090에 뜨면 18080의 테이블까지 지워진다.** 각 담당자가 5-5에서 반드시 확인 |
| **터미널 SeaweedFS** (`192.168.49.1:18891`) | nginx의 `/files/`와 admin 직원 사진이 18080 쪽 SeaweedFS를 쓴다 |
| **서버 CPU·메모리** | 빌드는 minikube 밖(서버 자체)에서 돈다. 10명이 동시에 빌드하면 Oracle·18080까지 느려진다 → 노드 executor 1, 재구축 날은 순서대로(9장) |

---

## 1. 구조

```
[GitHub] SEOUL-IT-HIS/<각 서비스 저장소>   (소스. 읽기만)
         SEOUL-IT-HIS/his-cicd              (Jenkinsfile + k8s YAML. 담당 폴더별로 나뉨)
              ↑ 5분마다 "내 폴더"와 "내 서비스 저장소"의 변경 확인
[팀원 A PC Jenkins]  [팀원 B PC Jenkins]  ...  [강진기 PC Jenkins]        ← 일을 시키는 쪽 (각자)
        ↓ 각자 만든 원격 노드(에이전트). 모두 서버의 guest4 계정으로 돈다
[원격 서버 guest4]  git clone → gradle bootJar → docker build → docker push(본인 Docker Hub)
                    → kubectl apply (내 서비스만) → rollout status
        ↓
[minikube · ns his] nginx ─ hisfrontend-nginx ─ 서비스 10개 ─ redis · kafka · seaweedfs
```

- **CI**: 코드가 바뀌면 자동으로 빌드해 이미지로 만든다. **CD**: 만든 이미지를 클러스터에 올리고 떴는지 확인한다
- 에이전트가 모두 guest4라 `kubectl`이 guest4의 `~/.kube/config`를 그대로 쓴다. kubeconfig 설정이 따로 필요 없다
- 각자 PC가 꺼지면 **그 사람 서비스의 자동 배포만** 멈춘다

---

## 2. 역할 분담

| 역할 | 폴더 | 담당 | 하는 일 |
|---|---|---|---|
| 인프라 | `infra/` | (정하기) | 공통 ConfigMap `his-common-config` + Redis·Kafka·SeaweedFS. **팀 전원에게 영향**이 있어 자동 실행 없이 공지 후 직접 실행 |
| nginx | `nginx/` | (정하기, 인프라 담당 겸임 권장) | 18090 입구. 라우팅 설정 변경·재시작 |
| 프론트 | `frontend/` | (정하기) | Next.js 이미지 빌드·배포 |
| admin, patient | `services/admin/`, `services/patient/` | 1명이 둘 다 | job 2개 |
| lab-imaging, pharmacy | `services/labimaging/`, `services/pharmacy/` | 강진기 | job 2개 |
| billing, reception, surgery, inpatient, outpatient, emergency | `services/<서비스>/` | 각 담당 | job 1개씩 |

한 사람이 서비스 둘을 맡으면 본인 Jenkins에 job을 둘 만들면 됩니다. 노드·Credentials는 하나로 같이 씁니다.

---

## 3. 같은 클러스터를 같이 쓰기 위한 규칙

| # | 규칙 | 지키게 하는 장치 |
|---|---|---|
| 1 | **`his-cicd`에서 내 폴더만 고친다** | 사람 약속. 서비스 job의 Included Regions(6장) 덕분에 남이 내 폴더를 안 건드리는 한 내 job은 안 돈다 |
| 2 | **push 전 `git pull --rebase`** | 여럿이 `main`에 바로 push → 안 하면 `rejected (non-fast-forward)` |
| 3 | **내 서비스 리소스만 배포** | 서비스 Jenkinsfile의 "남의 리소스 검사" 단계. `k8s.yaml`에 `<서비스>-config`, `<서비스>` Deployment·Service 말고 다른 것이 있으면 배포 전에 멈춘다 |
| 4 | **공통 설정·인프라·nginx는 담당자만** | 서비스 Jenkinsfile은 `infra/`, `nginx/`를 apply하지 않는다. 바꿀 것이 있으면 담당자에게 요청 |
| 5 | **비밀번호·토큰은 저장소에 넣지 않는다** | 비밀번호는 서버 Secret, 토큰은 각자 Jenkins Credentials. 저장소가 Public이다 |
| 6 | **본인 노드는 executor 1, 작업 폴더는 본인 것** | 5-2 노드 설정 |

---

## 4. 쿠버네티스 설정 규칙 (재구축 후에도 그대로)

### 4-1. Service 종류와 이름·포트

| 대상 | Service 이름 | 종류 | 포트 | 담당 |
|---|---|---|---|---|
| nginx | `nginx` | **NodePort** | 8080, **nodePort 30080** | nginx |
| 프론트 | `hisfrontend-nginx` | ClusterIP | 3000 | 프론트 |
| admin | `admin` | ClusterIP | 9191 | admin·patient |
| billing | `billing` | ClusterIP | 8081 | 각 담당 |
| pharmacy | `pharmacy` | ClusterIP | 8082 | 강진기 |
| reception | `reception` | ClusterIP | 8083 | 각 담당 |
| surgery | `surgery` | ClusterIP | 8084 | 각 담당 |
| lab-imaging | `labimaging` | ClusterIP | 8085 | 강진기 |
| inpatient | `inpatient` | ClusterIP | 8086 | 각 담당 |
| patient | `patient` | ClusterIP | 8087 | admin·patient |
| outpatient | `outpatient` | ClusterIP | 8088 | 각 담당 |
| emergency | `emergency` | ClusterIP | 8089 | 각 담당 |
| Redis / Kafka / SeaweedFS | `redis` / `kafka` / `seaweedfs` | ClusterIP | 6379 / 9092 / 8888·8333·9333 | 인프라 |

- **이름과 포트를 바꾸지 않는다.** nginx 라우팅(`patient.his.svc.cluster.local:8087` 등)과 서비스끼리의 설정(`http://patient:8087`)이 이 이름으로 찾는다
- **NodePort는 nginx 하나만.** 서비스를 NodePort로 열어도 서버 방화벽(18000~18999만 열림)과 minikube 포트 연결이 없어 바깥에서 닿지 않는다
- nginx는 짧은 이름을 못 찾아서 전체 이름과 `resolver 10.96.0.10`(kube-dns)을 쓴다. nginx job이 실행할 때마다 이 값을 출력해서 확인한다

### 4-2. 바깥에서 들어오는 길

```
브라우저 → seoulit.pe.kr:18090 → (minikube --ports=18090:30080) → Service nginx(30080→8080) → nginx Pod
        → /api/<서비스>/** : 각 서비스    / : hisfrontend-nginx:3000    /files/** : 터미널 SeaweedFS
```

### 4-3. Secret과 PVC — 파이프라인 밖, 지우지 않는다

- 지금 있는 것: `redis-secret`, `admin-secret` … `surgery-secret` (서비스 10개 모두 `<서비스>-secret`), PVC `seaweedfs-data`
- 파이프라인은 Secret을 **만들지 않고 있는지만 확인**한다. 8장(내리기)에서도 지우지 않는다
- 새로 만들어야 할 때만 서버에서(guest4는 셸 기록을 모두가 같이 보므로 비밀번호를 명령에 직접 치지 않는다):
  ```bash
  read -rsp "DB 비밀번호: " P; echo
  kubectl -n his create secret generic labimaging-secret --from-literal=DB_PASSWORD="$P"; unset P
  ```

### 4-4. 이미지 이름과 태그

| 대상 | 이미지 |
|---|---|
| 서비스 | `<담당자 Docker Hub>/his-<서비스>:b<빌드번호>-<커밋7자리>` (예: `jinki238/his-labimaging:b3-1d0c648`) |
| 프론트 | `<프론트 담당 Docker Hub>/hisfrontend:b<빌드번호>-<커밋>` |
| nginx·Redis·Kafka·SeaweedFS | 공식 이미지 그대로 |

- `latest`를 쓰지 않는다. 태그가 매번 달라야 쿠버네티스가 바뀐 것을 알아챈다
- 커밋을 붙인 이유: Jenkins job을 지웠다 다시 만들면 빌드번호가 1로 돌아간다. 같은 태그가 다시 나오면 서버에 남은 옛 이미지가 그대로 쓰일 수 있다
- YAML의 image 자리는 `__IMAGE__`. 파이프라인이 채워서 apply한다. **`his-cicd`의 YAML을 사람이 직접 `kubectl apply` 하지 않는다**

---

## 5. 준비 (각자)

### 5-1. 서버 확인 (한 번, 아무나)

2026-10-26 강진기가 확인 완료: kubectl(`~/bin`)·docker·java 17·git 있음, docker 권한 있음, Deployment·Service 15개, Secret 11개, PVC Bound, minikube Running, 18090 열림, Docker 데이터는 `/mnt/data`(여유 충분), DiskPressure 없음.

### 5-2. 본인 Jenkins의 원격 노드 설정

Jenkins 관리 → Nodes → 본인 노드 → Configure

| 항목 | 값 | 이유 |
|---|---|---|
| Number of executors | **1** | 공용 서버. 한 사람이 여러 빌드를 동시에 돌리지 않게 |
| Remote root directory | **`/home/guest4/jenkins-<이니셜>`** (예: `/home/guest4/jenkins-kjk`) | 모두 같은 guest4 계정이다. 두 사람이 같은 폴더를 쓰면 작업 폴더·에이전트 파일이 섞인다. 바꾸면 노드를 다시 연결(Disconnect → Launch) |
| Labels | 본인이 정한 이름(예: `remote-linux-node`) | 내 Jenkinsfile의 `CFG.agent`에 그대로 적는다 |
| Tool Locations → Git | `/usr/bin/git` | Jenkins가 Windows라 기본값 `git.exe`로는 리눅스에서 실패한다 |

### 5-3. 본인 Jenkins의 Credentials

Jenkins 관리 → Credentials → System → Global credentials → Add Credentials

| ID | Kind | 값 |
|---|---|---|
| **`dockerhub`** (철자 그대로) | Username with password | Username = 본인 Docker Hub 아이디, Password = **본인 Docker Hub 액세스 토큰**(Account settings → Personal access tokens → Generate, 권한 Read & Write) |

- GitHub 토큰은 필요 없다. `his-cicd`와 서비스 저장소가 모두 Public이다
- 토큰을 Jenkinsfile·YAML·채팅에 적지 않는다. 기한을 두면 학습이 끝난 뒤 저절로 못 쓰게 된다
- **서버에서 손으로 `docker login`하지 않는다.** guest4 공용 `~/.docker`에 토큰이 남아 접속한 누구나 내 계정으로 push할 수 있다. Jenkinsfile은 로그인 정보를 그 빌드 폴더에만 두고 끝나면 지운다

### 5-4. 내 서비스의 `k8s.yaml` 만들기 (서비스 담당)

`labimaging`, `pharmacy`는 이미 들어 있다. 나머지 담당자는 **클러스터에 남아 있는 "마지막으로 apply한 원본"을 꺼내** 만든다.

```bash
# 서버에서 (S 에 내 서비스 이름)
S=billing
mkdir -p ~/export-$S && chmod 700 ~/export-$S
{
  kubectl -n his apply view-last-applied configmap/$S-config -o yaml; echo '---'
  kubectl -n his apply view-last-applied deployment/$S -o yaml;      echo '---'
  kubectl -n his apply view-last-applied service/$S -o yaml
} > ~/export-$S/k8s.yaml
kubectl -n his get deploy $S -o custom-columns=IMAGE:.spec.template.spec.containers[0].image,REPLICAS:.spec.replicas
```

- 꺼낸 파일에서 **image 줄만 `image: __IMAGE__`로 바꾼다**
- replicas는 **1**(재처리 스케줄러·캐시 때문에 정한 팀 규칙, minikube 메모리 한도)
- **admin은 initContainer(wait-for-admin)가 없어야 한다**(자기 자신을 기다리게 된다)
- `view-last-applied`가 "not found"면 apply 외 방법으로 만든 것. `kubectl -n his get <종류>/<이름> -o yaml`을 보고 `services/labimaging/k8s.yaml`을 복사해 값만 바꾼다(기존 가이드 6-3 "바꿀 곳" 표)
- ConfigMap 이름이 `<서비스>-config`가 아니면 Jenkinsfile의 "남의 리소스 검사" 패턴도 그 이름에 맞게 고친다
- FileZilla로 내려받아 `his-cicd/services/<서비스>/k8s.yaml`에 넣고, 서버의 `~/export-$S`는 지운다

### 5-5. 내 서비스 저장소 점검 (서비스 담당)

| 확인 | 왜 | 문제일 때 |
|---|---|---|
| 루트에 `Dockerfile` | 파이프라인이 `docker build .` | 없으면 labimaging과 같은 4줄짜리: `FROM eclipse-temurin:17-jre` / `COPY build/libs/*-SNAPSHOT.jar /app/app.jar` / `EXPOSE <포트>` / `ENTRYPOINT ["java","-jar","app.jar"]` |
| `build.gradle` version이 `...-SNAPSHOT` | Dockerfile이 `*-SNAPSHOT.jar`를 복사 | 다르면 COPY 패턴을 맞춘다 |
| `gradlew`가 저장소에 있다 | `./gradlew`로 빌드 | 없으면 추가 |
| **`ddl-auto`** | 18080과 같은 스키마 | `create`/`create-drop`이면 **절대 배포 금지**. `none`/`validate` 권장 |
| properties에 평문 비밀번호·키 | Docker Hub 저장소가 기본 **공개**로 만들어진다 | 평문이 있으면 Docker Hub 저장소를 private으로 바꾸고 imagePullSecret(기존 가이드 4-7) 또는 평문 제거 |

```bash
grep -rniE 'ddl-auto|password|secret|key' src/main/resources/*.properties
```

### 5-6. 내 Jenkinsfile의 CFG 채우기

`services/<서비스>/Jenkinsfile` 맨 위 `CFG`만 고친다(나머지는 팀 공통).

```groovy
def CFG = [
    svc       : 'billing',                                            // 바꾸지 않는다
    repo      : 'https://github.com/SEOUL-IT-HIS/<서비스 저장소>.git',  // TODO 를 실제 주소로
    branch    : 'develop',
    dockerhub : '<본인 Docker Hub 아이디>',
    agent     : '<본인 노드 Label>',
    owner     : '<이름>',
]
```

`TODO`가 하나라도 남아 있으면 job이 노드를 기다리기 전에 바로 멈추고 어느 칸인지 알려 준다. 인프라·nginx는 `agent`만, 프론트는 `dockerhub`·`agent`·`owner`(+ 필요하면 `branch`)를 채운다.

### 5-7. 올리기

```bash
cd /c/his/his-cicd            # 처음이면: git clone https://github.com/SEOUL-IT-HIS/his-cicd.git
git pull --rebase
git add services/<서비스>/
grep -rniE 'password|token|pat_|secret' services/<서비스>/ | grep -viE 'secretRef|secretKeyRef|-secret|REDIS_PASSWORD$'   # 실제 값이 나오면 올리지 않는다
git commit -m "<서비스> CI/CD 설정"
git push
```

`rejected`가 나면 `git pull --rebase` 후 다시 `git push`. 내 폴더만 고쳤다면 충돌은 나지 않는다.

---

## 6. 본인 Jenkins에 job 만들기

새 Item → **Pipeline** → Pipeline 항목 **Definition: Pipeline script from SCM**

| 칸 | 값 |
|---|---|
| SCM | Git |
| Repository URL | `https://github.com/SEOUL-IT-HIS/his-cicd.git` |
| Credentials | - none - |
| Branch Specifier | `*/main` |
| Script Path | 아래 표 |
| **Additional Behaviours → Add → "Polling ignores commits in certain paths" → Included Regions** | 아래 표 (**꼭 설정**) |

| job 이름(권장) | Script Path | Included Regions |
|---|---|---|
| `his-<서비스>` 예: `his-labimaging`, `his-pharmacy` | `services/<서비스>/Jenkinsfile` | `services/<서비스>/.*` |
| `his-frontend` | `frontend/Jenkinsfile` | `frontend/.*` |
| `his-infra` | `infra/Jenkinsfile` | `infra/.*` |
| `his-nginx` | `nginx/Jenkinsfile` | `nginx/.*` |

- **Included Regions가 없으면** 누가 `his-cicd` 어디든 push할 때마다 팀 전원의 job이 한꺼번에 돈다(10명 동시 빌드 → 서버 과부하). 이 설정이 있으면 내 폴더가 바뀔 때만 돈다
- 내 서비스 **소스 저장소**의 변경은 이 설정과 상관없이 감시된다(소스가 바뀌면 배포되는 게 목적)
- 자동 실행(`pollSCM`)은 Jenkinsfile에 들어 있어 **job을 한 번 실행한 뒤부터** 등록된다. 인프라·nginx에는 일부러 넣지 않았다(전원 영향)
- 한 사람이 두 서비스를 맡으면 job 2개. 두 번째는 새 Item → "Copy from"에 첫 job 이름을 넣고 Script Path와 Included Regions만 바꾼다
- 저장 후 `Build Now`를 누르기 전에 7장을 읽는다

---

## 7. 1단계 — 각자 무중단 교체 (내리기 전)

목적: **18090을 내리기 전에** 각자 파이프라인이 되는지 확인. 실패해도 지금 Pod가 계속 서비스한다. 담당자마다 **편할 때 각자** 해도 된다(인프라·nginx 담당 제외 — 이 둘은 9장에서).

1. 본인 job → Build Now
2. 콘솔 단계별 정상 모습

   | 단계 | 정상 |
   |---|---|
   | 준비 | `which` 결과 3줄, 에러 없음 |
   | 남의 리소스 검사 | `configmap/<서비스>-config`, `deployment.apps/<서비스>`, `service/<서비스>` 세 줄 |
   | 소스 받기 | `Checking out Revision ... (refs/remotes/origin/develop)` |
   | jar 빌드 | `BUILD SUCCESSFUL` |
   | 이미지 빌드·push | `Login Succeeded`, `... b1-xxxxxxx: digest: sha256:...` |
   | 배포 | `configured`, `successfully rolled out` |

3. 서버에서 확인
   ```bash
   kubectl -n his rollout history deployment/<서비스>     # 마지막 줄에 "<이미지> by <이름> (Jenkins #1)"
   kubectl -n his get deploy -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
   ```
4. 브라우저 `http://seoulit.pe.kr:18090`에서 내 서비스 화면 확인

**팀 전원이 1단계를 통과한 뒤** 8장으로 갑니다. 1단계만으로도 "각자 CI/CD로 배포"는 이미 된 것이고, 8~9장은 "파이프라인만으로 처음부터 다시 세울 수 있는가"를 확인하는 팀 행사입니다.

---

## 8. 2단계 — 18090 내리기 (팀 행사, 인프라 담당 1명이 실행)

### 8-1. 먼저

- 날짜·시간을 정해 공지: "○시~○시 18090 내리고 각자 파이프라인으로 다시 올림(18080 영향 없음)"
- 전원이 7장을 통과했는지, 각자 job이 준비됐는지 확인
- 그 시간 동안 **각자 job의 자동 실행이 멋대로 돌지 않게** 하려면 job을 잠시 Disable 해 둔다(9장 순서가 오면 Enable 후 실행)

### 8-2. 기록과 백업

```bash
kubectl -n his get deploy,svc,cm,secret,pvc
B=~/his-backup-$(date +%Y%m%d-%H%M); mkdir -p "$B"; chmod 700 "$B"
kubectl -n his get deploy,svc,cm -o yaml > "$B/before-cicd.yaml"   # Secret 은 백업하지 않는다
```

### 8-3. 내리기

```bash
kubectl -n his delete deployment --all
kubectl -n his delete service --all
kubectl -n his get configmap -o name | grep -v kube-root-ca.crt | xargs -r kubectl -n his delete
```

### 8-4. 확인

```bash
kubectl -n his get all              # No resources found
kubectl -n his get secret,pvc       # Secret 11개, seaweedfs-data 그대로
```

---

## 9. 3단계 — 각자 파이프라인으로 재구축 (순서가 중요)

| 순서 | 누가 | job | 왜 이 순서 | 다음 사람에게 넘기는 신호 |
|---|---|---|---|---|
| 1 | 인프라 담당 | `his-infra` | 모든 서비스가 `his-common-config`·Redis·Kafka를 쓴다. 서비스 Jenkinsfile은 `his-common-config`가 없으면 멈춘다 | 콘솔 `successfully rolled out` ×3 |
| 2 | admin·patient 담당 | `his-admin` | 다른 서비스의 initContainer가 `admin:9191`을 기다린다 | admin 배포 성공 |
| 3 | 나머지 서비스 담당 전원 | `his-<서비스>` | 동시에 해도 되지만 **2~3명씩** 나눠서(공용 서버 CPU·메모리) | 각자 성공 |
| 4 | 프론트 담당 | `his-frontend` | `npm ci` + `next build`가 무겁다. 서비스 빌드가 끝난 뒤 | 성공 |
| 5 | nginx 담당 | `his-nginx` | 프론트가 없으면 화면이 502. 18090 응답 확인까지 한다 | `/nginx-health -> 200` |

- 걸리는 시간: 서비스 하나 3~5분, 프론트 5~15분. 2~3명씩 나누면 전체 1시간 안팎
- 하나가 실패해도 다른 사람은 자기 것을 계속 진행해도 된다. 단 admin이 실패하면 나머지 서비스는 Init 상태에서 기다리다 시간 초과될 수 있으니 admin을 먼저 해결한다
- 단톡방에 "1 끝", "2 끝"처럼 신호를 남기면 순서가 꼬이지 않는다

---

## 10. 확인

```bash
kubectl -n his get pods                     # 15개 안팎 Running, READY 1/1
kubectl -n his get deploy -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
#   → 서비스·프론트 이미지가 각 담당자 Docker Hub 의 ...:b<번호>-<커밋>
kubectl -n his get svc                      # nginx 만 NodePort 8080:30080
curl -s http://localhost:18090/nginx-health # ok
for s in admin billing pharmacy reception surgery labimaging inpatient patient outpatient emergency; do
  echo "== $s"; kubectl -n his rollout history deployment/$s | tail -1     # 누가 어떤 빌드로 배포했는지
done
```

브라우저 `http://seoulit.pe.kr:18090` → 로그인 → 각 메뉴 → 직원 사진까지. **502는 nginx 뒤의 서비스가 없거나 안 뜬 것**입니다.

---

## 11. 그다음: CI/CD처럼 쓰기

### 11-1. 자동 배포

```
내 서비스 저장소 develop 에 병합 → (5분 안에) 내 Jenkins 가 발견 → 빌드 → push → 롤링 업데이트
his-cicd 의 내 폴더(k8s.yaml 등) 수정 후 push → 같은 흐름
```

18090에 올라가는 것은 **develop 브랜치** 코드다. 18080 최종본과 다를 수 있다.

### 11-2. 되돌리기 (내 서비스만)

```bash
kubectl -n his rollout history deployment/<서비스>
kubectl -n his rollout undo deployment/<서비스>                 # 바로 이전으로
kubectl -n his rollout undo deployment/<서비스> --to-revision=2
```

다음 파이프라인이 돌면 다시 최신 코드로 배포된다. 계속 막으려면 코드를 고치거나 내 job을 Disable.

### 11-3. 설정 바꾸기

| 바꿀 것 | 누가 | 방법 |
|---|---|---|
| 내 서비스 ConfigMap | 서비스 담당 | `services/<서비스>/k8s.yaml` 수정 → push → 내 job이 다시 돈다(새 이미지 태그라 Pod가 새로 떠서 새 값을 읽는다) |
| `his-common-config`, Redis·Kafka·SeaweedFS | 인프라 담당 | `infra/common.yaml` 수정 → push → 공지 → `his-infra` 직접 실행. ConfigMap만 바꿨다면 각 서비스가 새 값을 읽도록 담당자들에게 재배포(또는 `rollout restart`)를 요청 |
| nginx 라우팅 | nginx 담당 | `nginx/k8s.yaml` 수정 → push → `his-nginx` 직접 실행 |
| Secret | 해당 담당 | 파이프라인 밖. 지우고 4-3대로 다시 만든 뒤 `kubectl -n his rollout restart deployment/<서비스>` |

### 11-4. 정리

| 위치 | 방법 |
|---|---|
| 서버 Docker의 빌드 이미지 | 파이프라인이 push 후 자동 삭제 |
| minikube 안에 받아 둔 이미지 | 가끔 `minikube image ls \| grep his-<서비스>`로 보고 내 옛 태그만 `minikube image rm <이미지:태그>` |
| Docker Hub 태그 | 각자 웹에서 오래된 태그 삭제 |
| Jenkins 빌드 기록 | job마다 10개만 남음 |

**`docker system prune`, `docker image prune`은 쓰지 않는다**(서버에 다른 팀 컨테이너·이미지가 있다).

---

## 12. 막혔을 때

| 증상 | 원인과 조치 |
|---|---|
| 빌드가 시작하자마자 `CFG.xxx 를 채운다` | 내 Jenkinsfile의 `CFG`에 `TODO`가 남음(5-6) |
| `Waiting for next available executor on '...'` | `CFG.agent`가 내 노드 Label과 다르거나 노드가 끊김. 노드 화면에서 연결 상태와 Label 확인 |
| `kubectl: not found` | `sh` 첫 줄 `export PATH="$HOME/bin:$PATH"` 확인 |
| `docker: permission denied` | 에이전트가 docker 권한 없이 떠 있음. 노드 재연결 |
| `git.exe` 관련 오류 | 노드 Tool Locations → Git = `/usr/bin/git`(5-2) |
| "남의 리소스 검사"에서 멈춤 | `k8s.yaml`에 내 것이 아닌 리소스(다른 이름, Namespace, PVC 등)가 있음. 내 ConfigMap 이름이 `<서비스>-config`가 아니면 검사 패턴을 고친다 |
| `his-common-config 가 없다` | 인프라가 아직 안 올라옴(9장 순서 1) |
| `docker push` `denied` | 토큰 권한이 Read 전용이거나 Credentials Username이 Docker Hub 아이디와 다름 |
| `rollout status` 시간 초과 | `kubectl -n his get pods -l app=<서비스>` 후 아래 |
| `Init:0/1` 멈춤 | admin을 기다리는 중. admin 상태 확인 |
| `CreateContainerConfigError` | ConfigMap·Secret 이름 불일치. `kubectl -n his describe pod <pod>` 맨 아래 Events |
| `ImagePullBackOff` | 이미지 이름 오타, 또는 Docker Hub 저장소가 private |
| `CrashLoopBackOff` | `kubectl -n his logs <pod> --previous`. DB 접속·설정 키 오류가 흔함 |
| `OOMKilled` | `limits.memory` 부족 |
| 남이 push할 때마다 내 job이 돈다 | Included Regions 미설정(6장) |
| `git push` `rejected (non-fast-forward)` | `git pull --rebase` 후 다시 push |
| 18090 화면 502 | 프론트나 서비스가 없음. `kubectl -n his get pods`, `kubectl -n his logs deploy/nginx` |
| 18090 응답 없음 | `kubectl -n his get svc nginx`의 nodePort 30080, `ss -ltn \| grep 18090` |

---

## 13. 하지 말 것

| 하지 말 것 | 이유 |
|---|---|
| `minikube stop`, `minikube delete` | 팀 전원의 배포 대상. delete는 18090 연결·Secret·PVC까지 사라진다 |
| `kubectl delete namespace his`, PVC 삭제, `kubectl delete -f common.yaml` | Secret·SeaweedFS 파일이 사라진다 |
| 남의 Deployment·ConfigMap을 `kubectl`로 직접 수정 | 그 사람 파이프라인이 다음 배포 때 덮어쓰고, 서로 원인을 모르게 된다 |
| `his-cicd`의 남의 폴더 수정 | 그 사람 job이 자동으로 돈다 |
| `docker system prune`, `docker image prune`, `docker rm $(docker ps -aq)` | 공용 서버의 다른 팀 것까지 지운다 |
| 서버에서 손으로 `docker login` | 공용 `~/.docker`에 토큰이 남는다 |
| 비밀번호·토큰을 YAML·Jenkinsfile·Git에 | 저장소가 Public이다 |
| `latest` 태그, replicas 2 이상 | 변경 감지 실패 / 팀 규칙·메모리 한도 |
| `ddl-auto=create` 서비스 배포 | 18080과 같은 스키마의 테이블이 지워진다 |
| 18080 쪽 터미널 프로세스, 서버 포트 18080·18891 | 최종 배포본과 18090 직원 사진이 같이 끊긴다 |
