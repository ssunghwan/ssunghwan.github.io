---
title: "패스워드는 바뀌었는데 파드는 몰랐다 - Stakater Reloader로 해결한 자동 롤링 재시작"
date: 2026-07-06 09:00:00 +0900
categories: [2. Kubernetes, Cloud Native Transformation]
tags: [eks, kubernetes, reloader, external-secrets, eso, argocd, secret-rotation, rds, gitops, prometheus]
---

> RDS 마스터 암호가 로테이션됐는데 API 파드가 며칠간 옛날 비밀번호로 DB 접속을 시도하다 로그인 장애가 발생했다.<br>
> 원인은 "K8s Secret이 갱신돼도 이미 떠있는 파드의 환경변수는 자동으로 바뀌지 않는다"는 Linux 프로세스의 근본적인 동작 원리였다.<br>
> 이 문제를 구조적으로 해결하기 위해 **Stakater Reloader**를 도입한 전 과정을 다룬다.
{: .prompt-info }

---

## 1. 요약

| 항목 | 내용 |
|---|---|
| 문제 | K8s Secret 갱신 후 기존 파드의 환경변수는 고정 → RDS 암호 로테이션 후 API 파드가 며칠간 구 비밀번호로 DB 접속 → 로그인 장애 |
| 도구 | [Stakater Reloader](https://github.com/stakater/Reloader) v1.4.19 (Helm chart 2.2.14) |
| 방식 | ArgoCD Multi-Source (공식 OCI 차트 + 이 저장소의 values.yaml) |
| 적용 범위 | `<prefix>-nextjs-apne2-deploy`, `<prefix>-api-apne2-deploy`에 `reloader.stakater.com/auto: "true"` annotation |
| 검증 | `cdn-secret` 값 변경 → 두 Deployment 모두 수 초 내 자동 롤링 재시작, 파드 내부 값까지 일치 확인 |
| 추가 보강 | 리소스 requests/limits, PodMonitor(Prometheus 연동) — 도입 당일 자체 점검에서 발견해 즉시 보강 |
| 부수 발견 | `nextjs-secret`이 실제로는 어느 Deployment에서도 참조되지 않는 죽은 시크릿이라는 사실 |

---

## 2. 배경 — DB 로그인 장애 전체 조사 과정

이 사고의 전체 조사 과정을 상세히 남겨둔다. 결론(Reloader 도입)만 보면 "당연한 해법"처럼 보이지만, 실제로는 여러 그럴듯한 가설을 하나씩 배제해나가는 과정이었고, 그 과정 자체가 "왜 이 도구가 필요한가"를 가장 설득력 있게 보여준다.

### 최초 증상

사용자가 "로그인이 안 되고 DB 에러가 뜬다"고 보고. API 파드 로그를 확인하자 다음 에러가 산발적으로 발견됐다.

```
Error: Access denied for user '<db-username>'@'172.16.x.x' (using password: YES)
  code: 'ER_ACCESS_DENIED_ERROR'
  errno: 1045
  sqlState: '28000'
```

로그 타임스탬프를 훑어보니 2026-07-03부터 2026-07-06까지 산발적으로(매 요청마다는 아니고 간헐적으로) 발생하고 있었다.

### 가설 1 — RDS 비밀번호 로테이션 자체가 실패했다 (기각)

`mysql-secret`(RDS 접속 정보)은 ESO가 관리하는데, `DB_USERNAME`/`DB_PASSWORD`는 RDS의 `manage_master_user_password = true`로 자동 관리하는 별도 시크릿에서 가져온다. "로테이션이 실패해서 Secrets Manager와 RDS 엔진의 비밀번호가 어긋난 게 아닌가" 의심했다.

**검증 시도**: API 파드에서 새 DB 커넥션을 직접 열어봄 → 100% 실패. 그런데 바로 직전에 같은 API를 통한 로그인은 정상 동작했다 — **이미 떠있는 커넥션 풀은 되는데 새 커넥션은 안 되는** 기묘한 패턴이 처음 관측됐다.

RDS 이벤트 로그를 조회하자 결정적 증거가 나왔다.

```
Message: "Reset master credentials"   Date: 2026-06-26T08:09:17
Message: "Reset master credentials"   Date: 2026-07-03T01:10:20
Message: "Reset master credentials"   Date: 2026-07-06T04:28:01
```

RDS가 "마스터 자격증명 재설정 완료"를 로그에 남기고 있었다. **로테이션은 매번 실제로 성공하고 있었다.** 가설 기각.

### 가설 2 — RDS 보안그룹이 로테이션 Lambda를 막고 있다 (기각)

AWS의 관리형 시크릿 로테이션은 고객 VPC 안에 Lambda를 배포해 실제로 DB에 접속해 새 비밀번호를 검증하는 구조다. RDS 보안그룹이 EKS/vscode SG만 허용하므로 "Lambda의 네트워크 경로가 막혔다"는 가설을 세웠다.

**검증**: Secrets Manager 콘솔의 "로테이션 구성" 탭 확인 → *"Amazon RDS는 이 보안 암호에 대한 교체를 관리하므로 Lambda 교체 함수를 선택할 필요가 없습니다"* 확인. **Lambda 자체가 없는, RDS가 완전히 내부적으로 처리하는 네이티브 로테이션**이었다. VPC 경로를 아예 안 타므로 보안그룹 이슈일 수가 없다. 기각.

### 가설 3 — RDS 인스턴스에 반영 대기 중인 변경사항이 있다 (기각)

RDS 콘솔에서 "보류 중인 변경사항" 배너 확인 → 없음. `PendingModifiedValues`도 빈 값(`{}`). 기각.

### 결정적 실험 — 관리자 권한으로 현재 값 직접 테스트

`aws secretsmanager get-secret-value`로 관리자 프로파일을 통해 **진짜 현재 값**을 직접 가져와서, ESO나 K8s Secret을 거치지 않고 그 값 그대로 DB에 연결해봤다.

```js
const conn = await mysql.createConnection({
  host: '...', port: 3306, user: '<db-username>',
  password: '<관리자 프로파일로 방금 가져온 실제 AWSCURRENT 값>',
  database: '<db-name>',
});
// → 성공
```

이 순간 **"AWS 쪽은 전부 정상이고, 문제는 우리 K8s 클러스터 안에 있다"**는 게 확정됐다.

### 범인 확정 — 파드 환경변수는 시작 시점에 고정된다

관리자 프로파일로 가져온 값과 K8s Secret에 저장된 값을 비교하니 **완전히 동일**했다. ESO도, K8s Secret도 전부 정확한 값을 갖고 있었다. 그런데도 `kubectl exec`으로 파드 안에서 `process.env.DATABASE_URL`을 읽어 연결하면 계속 실패했다.

**결론**: K8s Secret의 내용은 이미 며칠 전부터 정확했다. 문제는 **그 정확한 값을 담고 있는 Secret을, 그보다 먼저(로테이션 전에) 시작된 파드가 여전히 옛날 값 그대로 환경변수에 담고 있었다**는 것. `kubectl rollout restart deployment/<prefix>-api-apne2-deploy`로 즉시 해결됐다.

### 가설 배제 정리

| 가설 | 결과 |
|---|---|
| RDS 로테이션 자체 실패 | 기각 (RDS 이벤트 로그로 매번 성공 확인) |
| 로테이션 Lambda가 보안그룹에 막힘 | 기각 (Lambda 자체가 없는 완전 관리형 방식) |
| RDS 인스턴스 pending 변경사항 | 기각 (상태 `available`, pending 없음) |
| ESO의 시크릿 캐싱 버그 | 기각 (완전히 새로 재시작한 ESO 파드도 동일하게 실패) |
| **파드 환경변수가 시작 시점에 고정됨** | ✅ 확정된 근본 원인 |

---

## 3. 왜 이런 일이 근본적으로 가능한가

### Linux 프로세스의 환경변수는 exec() 시점에 딱 한 번 복사된다

Linux에서 새 프로세스를 시작할 때(`execve()` 시스템 콜), 커널은 부모 프로세스가 넘겨준 환경변수 배열(`envp`)을 새 프로세스의 메모리 공간에 복사한다. 이 복사는 **프로세스 시작 시점에 단 한 번만** 일어난다. 그 이후로는:

- 부모가 환경변수를 바꿔도 자식에게 전파되지 않는다.
- 외부에서 프로세스의 환경변수를 바꿀 수 있는 공식 방법이 없다 (`ptrace` 등 비정상적 방법 제외).
- 프로세스가 직접 `getenv()`로 읽는 순간에도, 그 값은 시작 시점에 복사된 자신의 메모리에서 나온다.

이는 OS 레벨의 근본적인 설계이지 버그가 아니다.

### Kubernetes의 `secretKeyRef`가 이 원리 위에서 동작한다

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: DB_PASSWORD
```

kubelet이 파드를 시작할 때 K8s API에서 `mysql-secret`의 `DB_PASSWORD` 값을 읽어 컨테이너 프로세스의 `envp`에 넣어준다. 이후로는 Linux 원리대로 — 컨테이너 프로세스는 자신의 메모리에 복사된 그 값만 본다.

K8s Secret이 갱신되어도(ESO가 로테이션된 값을 반영해도) **이미 실행 중인 파드는 전혀 모른다.** 새 파드가 시작될 때만 새 값을 받는다.

### 볼륨 마운트 방식은 왜 다른가

```yaml
# 환경변수 방식 — 재시작 필요 ❌
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: ...

# 볼륨 마운트 방식 — kubelet이 자동으로 파일을 갱신 ✅ (단, 앱이 파일을 다시 읽어야)
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
volumes:
  - name: secret-volume
    secret:
      secretName: mysql-secret
```

볼륨 마운트 방식에서는 kubelet이 Secret 변경을 감지해 마운트된 파일을 자동으로 업데이트한다(기본 60초 이내). 단, **앱이 해당 파일을 능동적으로 다시 읽는 로직이 있어야** 실제로 새 값이 사용된다.

> 이 앱은 Node.js 기반으로 `process.env`로 DB 접속 정보를 읽는다. `process.env`는 프로세스 시작 시 딱 한 번 복사된 환경변수 배열을 참조하므로, 볼륨 마운트로 바꿔도 앱 코드 수정 없이는 새 값이 사용되지 않는다.
{: .prompt-info }

---

## 4. Stakater Reloader란

**Stakater Reloader**는 Kubernetes Secret 또는 ConfigMap이 변경될 때 그것을 참조하는 Deployment/StatefulSet/DaemonSet 등을 **자동으로 롤링 재시작**해주는 오픈소스 컨트롤러다.

```
[ESO] Secret 갱신
         ↓
[Reloader] Secret 변경 감지
         ↓
[K8s API] Deployment에 annotation 추가 (lastUpdatedAt 타임스탬프)
         ↓
[K8s 컨트롤러] annotation 변경 → spec 변경으로 인식 → RollingUpdate 트리거
         ↓
[새 파드] 시작 시 최신 Secret 값 반영
```

핵심은 Reloader가 Deployment를 직접 삭제/생성하지 않는다는 점이다. `kubectl.kubernetes.io/restartedAt` annotation을 업데이트하면 K8s가 이를 spec 변경으로 인식해 자체 Rolling Update를 수행한다. 기존 RollingUpdate 전략(maxSurge, maxUnavailable), readinessProbe, PDB가 전부 그대로 작동한다.

### GitHub Stars & 운영 성숙도

Reloader는 2019년부터 개발되어 GitHub Stars 2,500+를 기록 중이며, CNCF Landscape에 등재된 프로젝트다. EKS 기반 프로덕션에서 ESO, ArgoCD와 함께 "3종 세트"로 자주 사용된다.

---

## 5. 동작 원리 상세

### 감지 메커니즘 — Kubernetes Informer

Reloader는 K8s API Server의 **Informer**(Watch + List) 메커니즘으로 Secret과 ConfigMap의 변경을 감지한다.

```
K8s API Server
    │  Watch /api/v1/secrets?watch=true
    ▼
Reloader Controller
    ├── Secret 변경 이벤트 수신
    ├── 해당 Secret을 참조하는 Deployment 목록 조회
    │   (annotation 필터링)
    └── 각 Deployment에 rollout annotation 업데이트
```

Polling이 아니라 Watch 기반이므로 변경 감지 지연이 수 초 이내다.

### 재시작 전략 — reloadStrategy: default

```yaml
reloadStrategy: default   # annotation 기반 (K8s 네이티브 RollingUpdate)
```

Reloader가 지원하는 재시작 전략은 두 가지다.

| 전략 | 동작 | 특징 |
|---|---|---|
| `default` | `kubectl.kubernetes.io/restartedAt` annotation 업데이트 | K8s 네이티브 RollingUpdate, readinessProbe·PDB 완전히 존중 |
| `patch` | 환경변수에 더미 값 직접 주입 | 이전 방식, 현재는 legacy |

`default` 전략을 사용하면 Reloader가 트리거한 재시작도 `kubectl rollout history`로 추적 가능하다.

### PDB와의 상호작용

```yaml
# PDB: 최소 1개 파드는 항상 Running 유지
spec:
  minAvailable: 1
```

Reloader가 RollingUpdate를 트리거할 때 K8s는 PDB를 존중한다. PDB가 없으면 2개 레플리카가 동시에 재시작될 수 있어 순간적인 서비스 중단이 발생할 수 있다.

> PDB 없이 Reloader를 운영하면 Secret 변경 시 모든 파드가 동시에 내려갈 위험이 있다. **Reloader 도입 전 PDB 설정 여부를 반드시 확인**해야 한다.
{: .prompt-danger }

---

## 6. 설치 및 설정

### ArgoCD Application (Multi-Source)

기존 클러스터의 Helm 차트 관리 컨벤션(ArgoCD Multi-Source)을 그대로 따른다.

```yaml
# kubernetes/argocd/application-reloader.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-reloader
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  sources:
    - repoURL: registry-1.docker.io/bitnamicharts
      chart: reloader
      targetRevision: "2.2.14"
      helm:
        releaseName: reloader
        valueFiles:
          - $values/helm/reloader/values.yaml
    - repoURL: git@github.com:<github-username>/<repo-name>.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

> **Sync Wave 2를 선택한 이유**<br>
> Reloader는 Deployment를 재시작하는 컨트롤러다. 재시작 대상 Deployment(`apps`, wave 6)보다 반드시 먼저 떠있어야 한다.<br>
> ESO(wave 1)가 먼저 Secret을 동기화한 뒤 Reloader(wave 2)가 뜨는 순서가 논리적으로 맞다.
{: .prompt-tip }

### Helm values

```yaml
# helm/reloader/values.yaml
reloader:
  watchGlobally: true        # 전체 네임스페이스 Secret/ConfigMap 감시
  reloadStrategy: default    # K8s 네이티브 annotation 기반 RollingUpdate

  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

  nodeSelector:
    role: system

  resources:
    requests:
      cpu: 10m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

  podMonitor:
    enabled: true            # Prometheus PodMonitor 활성화
    additionalLabels:
      release: kube-prometheus-stack
```

**`watchGlobally: true`를 선택한 이유:**

`watchGlobally: false`로 설정하면 Reloader가 자신이 배포된 네임스페이스(`monitoring`)의 Secret/ConfigMap만 감시한다. 재시작 대상 Deployment가 있는 `<app-namespace>`의 Secret을 감지하려면 `watchGlobally: true`가 필요하다.

> 실제 **재시작 대상**은 `reloader.stakater.com/auto: "true"` annotation이 붙은 워크로드로만 제한되므로, watch 범위가 넓어도 실질적인 blast radius는 annotation을 명시적으로 붙인 워크로드에만 한정된다.
{: .prompt-info }

### Deployment annotation 추가

```yaml
# kubernetes/apps/deployment/deployment-api.yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"   # 이 Deployment가 참조하는 모든 Secret/ConfigMap 변경 시 재시작
```

`auto: "true"`는 해당 Deployment의 `envFrom`, `env.valueFrom.secretKeyRef`, `volumes.secret` 등 **모든 참조 경로**를 자동으로 추적한다. 특정 Secret만 감시하려면 아래 방식을 사용한다.

```yaml
# 특정 Secret만 감시
annotations:
  secret.reloader.stakater.com/reload: "mysql-secret,cdn-secret"
```

---

## 7. 실제 배포된 리소스 확인

```bash
kubectl get all -n monitoring -l app.kubernetes.io/name=reloader

NAME                                        READY   STATUS    RESTARTS
pod/reloader-<hash>                         1/1     Running   0

NAME                                        TYPE        CLUSTER-IP
service/reloader-<hash>                     ClusterIP   10.x.x.x

NAME                                        READY   UP-TO-DATE
deployment.apps/reloader                    1/1     1
```

**RBAC 확인:**

```bash
kubectl get clusterrolebinding | grep reloader
# reloader-role-binding → ClusterRole: reloader-role
# 전체 네임스페이스의 secrets, configmaps watch/list/get 권한
```

---

## 8. 대안 도구 비교와 Reloader 선택 이유

| 도구 | 방식 | 장점 | 단점 | 우리 판단 |
|---|---|---|---|---|
| **Stakater Reloader** | Secret/ConfigMap 변경 → Rolling Restart | 배포 도구 독립, 설정 단순, ESO와 궁합 최적 | 재시작 자체가 솔루션 (실시간 반영 아님) | ✅ 채택 |
| **Helm lookup + Helm Operator** | Helm 릴리즈 값 변경으로 재배포 | Helm 생태계 일관성 | 우리 앱은 순수 Kustomize + plain YAML 사용 | ❌ 부적합 |
| **Vault Agent Injector** | 사이드카가 파일을 지속적으로 갱신 | 진짜 실시간 반영 가능 | HashiCorp Vault 전제, 앱이 파일 변경 감지 로직 직접 구현 필요 | ❌ Vault 미사용 |
| **수동 `kubectl rollout restart`** | 사람이 직접 | 별도 설정 불필요 | 이번 사고의 원인 그 자체 | ❌ 폐기 |

> **"GitOps(ArgoCD/Flux) + External Secrets Operator + Reloader"** 조합은 EKS 기반 프로덕션에서 매우 흔하게 보이는 3종 세트다.<br>
> ESO는 "시크릿을 최신으로 유지"까지만 책임지고, "그 최신 값을 파드에 실제로 반영"하는 건 Reloader의 역할이다. 둘 중 하나만 쓰면 절반만 해결된 상태로 남는다.
{: .prompt-tip }

### `autoReloadAll` vs annotation opt-in

Reloader는 `reloader.autoReloadAll: true`로 **모든** 워크로드를 무조건 감시하는 모드도 지원하지만, 공식 문서와 커뮤니티 모두 프로덕션에서는 **annotation 기반 opt-in**을 강력히 권장한다.

이유는 명확하다:

- 배치 작업을 처리 중인 워크로드가 작업 도중 재시작되면 데이터 정합성 문제가 생길 수 있다.
- 상태를 많이 들고 있는 StatefulSet이 의도치 않게 재시작되면 복구 시간이 길어진다.
- annotation이 코드로 명시되므로 "이 워크로드는 의도적으로 감시 대상"이라는 의도가 git 이력으로 추적된다.

현재 딱 2개(`nextjs`, `api`) Deployment에만 annotation을 붙였다.

---

## 9. 실제 동작 검증

운영 DB 자격증명(`mysql-secret`)으로 직접 테스트하는 건 위험하므로, nextjs/api 둘 다 참조하는 **저위험 시크릿**(`cdn-secret`)으로 검증했다.

### 첫 시도 — 잘못된 테스트 대상 선정

처음엔 `nextjs-secret`(`SESSION_SECRET`)으로 테스트를 시도했으나, 30초를 기다려도 아무 반응이 없었다. Reloader 로그를 확인해도 관련 이벤트가 전혀 없었다. 원인을 조사하다 **`nextjs-secret`을 참조하는 Deployment가 애초에 존재하지 않는다**는 걸 발견했다. Reloader가 고장난 게 아니라 테스트 대상 선정이 잘못됐던 것이다.

### 검증 절차

```bash
# 1. 테스트 전 파드 상태 기록
kubectl get pods -n <app-namespace> \
  -l 'app in (<prefix>-nextjs,<prefix>-api)'

# <prefix>-api-apne2-deploy-6df9d75658-bclwd    1/1  Running  0  26m
# <prefix>-api-apne2-deploy-6df9d75658-z8cqj    1/1  Running  0  26m
# <prefix>-nextjs-apne2-deploy-7cdcb4b745-9dr9c 1/1  Running  0  4d1h
# <prefix>-nextjs-apne2-deploy-7cdcb4b745-sc568 1/1  Running  0  4d1h

# 2. Secret 값 변경 (더미값으로 patch)
kubectl patch secret cdn-secret -n <app-namespace> \
  --type=json -p='[{"op":"replace","path":"/data/INTERNAL_API_KEY","value":"<더미값>"}]'
# secret/cdn-secret patched   # 06:20:09
```

### Reloader 감지 로그 (수 초 이내)

```
time="...06:20:09Z" level=info msg="Changes detected in 'cdn-secret' of type 'SECRET'
  in namespace '<app-namespace>';
  updated '<prefix>-api-apne2-deploy' of type 'Deployment' ..."
time="...06:20:09Z" level=info msg="Changes detected in 'cdn-secret' ...;
  updated '<prefix>-nextjs-apne2-deploy' ..."
```

같은 이벤트가 두 번씩 찍힌 건 레플리카 2개짜리 Deployment에 대해 Informer가 관련 리소스를 재확인하는 과정에서 나오는 정상적인 로그 중복이다.

### 자동 롤링 재시작 확인

```bash
kubectl get pods -n <app-namespace> \
  -l 'app in (<prefix>-nextjs,<prefix>-api)'

# <prefix>-api-apne2-deploy-56848d54cd-rnwvc    1/1  Running      0  34s   ← 새 ReplicaSet
# <prefix>-api-apne2-deploy-56848d54cd-vw6h9    1/1  Running      0  26s   ← 새 ReplicaSet
# <prefix>-api-apne2-deploy-6df9d75658-bclwd    1/1  Terminating  0  26m   ← 기존 파드
# <prefix>-api-apne2-deploy-6df9d75658-z8cqj    1/1  Terminating  0  26m
# <prefix>-nextjs-apne2-deploy-76776bcd7c-krfgm 1/1  Running      0  34s   ← 새 ReplicaSet
# <prefix>-nextjs-apne2-deploy-76776bcd7c-nqhc8 1/1  Running      0  20s
```

수동 개입 없이, Secret patch 하나만으로 **양쪽 Deployment 모두 새 ReplicaSet으로 전환**됐다.

### 원상복구 및 최종 일관성 확인

```bash
# Secret 삭제 → ESO가 즉시 재생성 (creationPolicy: Owner)
kubectl delete secret cdn-secret -n <app-namespace>

# K8s Secret 값 확인
kubectl get secret cdn-secret -n <app-namespace> \
  -o jsonpath='{.data.INTERNAL_API_KEY}' | base64 -d
# <실제 정상 값>

# 파드 내부 환경변수 확인 — K8s Secret과 100% 일치 ✅
kubectl exec -n <app-namespace> deploy/<prefix>-api-apne2-deploy \
  -- sh -c 'echo $INTERNAL_API_KEY'
# <실제 정상 값>
```

이 마지막 확인이 특히 중요하다. 단순히 "재시작이 일어났다"는 것만 확인한 게 아니라, **재시작된 파드가 실제로 최신 값을 정확히 물고 있는지**까지 확인해서 전체 사이클이 끝까지 제대로 동작함을 검증했다.

---

## 10. 현업 표준 부합 여부 체크리스트

도입 직후 자체 점검을 진행했고, 발견된 갭은 같은 날 즉시 보강했다.

| 항목 | 상태 | 설명 |
|---|---|---|
| Annotation 기반 opt-in (not `autoReloadAll`) | ✅ | 의도한 워크로드만 재시작 대상 |
| 안전한 롤아웃 전략 (`reloadStrategy: default`) | ✅ | K8s 네이티브 RollingUpdate, readinessProbe·PDB 존중 |
| Helm 차트 버전 고정 | ✅ | `targetRevision: "2.2.14"`, `latest` 미사용 |
| GitOps로 관리 (수동 설치 아님) | ✅ | 기존 ArgoCD Multi-Source 컨벤션 그대로 |
| 리소스 requests/limits | ✅ (도입 당일 보강) | 최초 배포 시 누락 → 자체 점검으로 발견 즉시 보강 |
| 모니터링 연동 (PodMonitor) | ✅ (도입 당일 보강) | Reloader 자체 `/metrics`가 Prometheus에 미연동 → 보강 |
| RBAC 범위 (`watchGlobally: true`) | ⚠️ 의도적 트레이드오프 | ClusterRole로 전체 네임스페이스 감시. 실제 재시작 대상은 annotation 명시 워크로드로 제한. 향후 다른 네임스페이스 확장 가능성을 열어두기 위해 유지 |
| 고가용성 (replica 1개) | ⚠️ 의도적 트레이드오프 | 크리티컬 서빙 경로가 아닌 재시작 트리거 도구라 단일 레플리카로 충분하다고 판단 |

`⚠️` 두 항목은 결함이 아니라 **의도적으로 인지하고 받아들인 트레이드오프**다. 실제로 문제가 되면 그때 `namespaceSelector`로 좁히거나 `enableHA: true`로 전환하면 된다.

---

## 11. 부수적으로 발견한 이슈 — 죽은 시크릿

검증 과정에서 `nextjs-secret`(`SESSION_SECRET`)을 첫 테스트 대상으로 골랐다가, **이 Secret을 참조하는 Deployment가 하나도 없다는 걸 발견했다.**

```bash
grep -rn "SESSION_SECRET\|session_secret" applications/<service>-nextjs/src
# (결과 없음)

grep -i "session\|iron\|cookie" applications/<service>-nextjs/package.json
# (결과 없음)
```

이 앱은 AT/RT 이중 토큰 방식(AT는 zustand 메모리, RT는 localStorage — 클라이언트가 토큰을 직접 보관)이라 **서버사이드 암호화 세션 쿠키 자체가 필요 없는 구조**다. `nextjs-secret`은 초기 설계 단계의 잔재로 보인다.

정리하려면 아래 세 곳이 얽혀있어 별도 작업으로 보류했다.

1. `kubernetes/external-secrets/external-secret-nextjs.yaml` (ExternalSecret 삭제)
2. `terraform/envs/dev/main.tf`의 관련 리소스 (실제 AWS Secrets Manager 리소스 삭제)
3. 루트 `README.md`의 Secrets Manager 표에서 해당 행 제거

---

## 12. 남은 과제

> - `nextjs-secret` 정리 — 별도 작업으로 보류 중
> - `namespaceSelector`로 watch 범위 좁히기 검토 — 현재는 `watchGlobally: true`로 전역 감시
> - argocd-server, grafana 등 다른 시스템 컴포넌트에도 Reloader annotation 적용 검토 (단, argocd-server는 `argocd-secret` 변경을 자체적으로 감지해 재시작하는 내장 기능이 있어 불필요할 수 있음)
{: .prompt-info }

---

## 13. 회고

> **같은 클래스의 문제가 2주 안에 2번 나오면, 그건 "운"이 아니라 "패턴"이다.** JWT 시크릿과 RDS 시크릿 둘 다 "Secret은 정상인데 파드가 옛날 값을 쓰고 있었다"는 동일한 근본 원인이었다. 반복되는 시점에 구조적 해법을 도입하는 판단이 필요했다.
{: .prompt-warning }

> **AWS 쪽을 의심하기 전에 우리 쪽(K8s)을 먼저 의심했어야 더 빨랐을 수 있다.** "관리형 서비스(RDS)는 보통 정상 동작한다"는 사전 확률을 좀 더 높게 잡았다면 조사 순서가 달라졌을 것이다.
{: .prompt-tip }

> **"일단 재시작 도구를 넣자"에서 끝내지 않고, 넣은 도구 자체를 우리 클러스터의 기존 컨벤션(리소스 제한, 모니터링)으로 다시 검증한 게 유효했다.** 새 컴포넌트를 급하게 추가할 때 이런 기본기가 누락되기 쉽고, 실제로 두 가지(resources, monitoring)가 빠져 있었다.
{: .prompt-tip }

> **검증 테스트의 대상 선정 자체도 조사의 일부다.** `nextjs-secret`으로 테스트했다가 "왜 반응이 없지?"를 파고든 게 결과적으로 죽은 시크릿을 찾아내는 부수 소득으로 이어졌다.
{: .prompt-info }
