---
title: "Building a Production-Grade Alarm System for EKS E-commerce (Prometheus + AlertManager + Slack)"
date: 2026-06-04 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, prometheus, alertmanager, slack, loki, promtail, karpenter, hpa, external-secrets, monitoring, alerting]
---

> EKS 마이그레이션 시리즈 아홉 번째 포스팅이다.<br>
> 지난 8편에서 PLG 스택 고도화(대시보드 + Slack 알림 + Blackbox)까지 완료했다. 이번에는 막상 배포하고 나서 발견한 문제들을 하나씩 짚어가며 실제 프로덕션 수준의 알람 시스템으로 개선하는 전 과정을 다룬다.
{: .prompt-info }

---

## 문제 인식

배포 후 찬찬히 들여다보니 문제가 한두 가지가 아니었다.

- `values.yaml` 한 파일에 Prometheus 설정, AlertManager 라우팅, Alert Rules가 전부 뒤섞여 있어서 알람 규칙 하나 바꾸려면 인프라 설정 파일 전체를 건드려야 했고
- Grafana admin 비밀번호가 git에 **평문으로** 올라가 있었고
- `PodNotReady` 알람은 kube-system Pod까지 전부 잡아서 **상시 firing** 상태였으며
- AWS API 메트릭을 참조하는 Alert Rule 2개는 해당 메트릭이 **아예 존재하지 않아서** 절대 울리지 않는 dead rule이었다

---

## 1. 전체 아키텍처 개요

개선 후 알람 시스템의 전체 흐름은 아래와 같다.

```
[Karpenter Nodes]
  ├── system nodes → Prometheus, AlertManager, Grafana, Loki
  ├── web nodes    → nextjs Pod
  └── api nodes    → api Pod

[메트릭 수집]
  Prometheus ─── kube-state-metrics (Pod/Node/HPA 상태)
              ├── node-exporter (노드 리소스)
              ├── blackbox-exporter (외부 엔드포인트)
              └── ALB Controller (reconcile 메트릭)

[로그 수집]
  Promtail (DaemonSet) → Loki Gateway → Loki (S3 백엔드)
  └── CRI 파싱 + 앱 네임스페이스 JSON 파이프라인 (level/status label)

[알림]
  Prometheus → AlertManager → Slack
    ├── [CRITICAL] 즉시, 1h repeat
    ├── [WARNING]  4h repeat
    └── [INFO]     참고용
```

Alert Rule은 총 **19개**, 5개 카테고리로 분류된다.

| 카테고리 | 규칙 수 | 주요 감지 대상 |
|---|:---:|---|
| Kubernetes Workload | 6 | CrashLoop, OOM, Restart, NotReady, Waiting, Replica 불일치 |
| Node / Storage | 4 | 메모리, 디스크, PVC 사용률 |
| External / Blackbox | 4 | 엔드포인트 다운, 응답 지연, SSL 만료 |
| HPA / Karpenter / ESO | 3 | 스케일링 한계, Pod Pending, 시크릿 동기화 오류 |
| ALB Controller | 2 | Reconcile 오류 |

---

## 2. values.yaml 파일 분리 — 관심사 분리

### 왜 분리해야 하는가?

기존에는 `kube-prometheus-stack/values.yaml` 한 파일에 Prometheus 스펙, AlertManager 라우팅 설정, 커스텀 Alert Rules가 전부 들어 있었다.

AlertManager Slack 메시지 템플릿을 수정하거나 Alert Rule 임계값 하나 바꾸려고 해도 Prometheus 리소스 설정이나 Grafana 설정이 포함된 메인 values 파일을 통째로 수정해야 했다. 리뷰 범위도 넓어지고, 협업 시 충돌 위험도 높다.

> **관심사 분리(Separation of Concerns)** 원칙을 적용하면 각 담당자가 자신의 영역만 수정하면 된다. 인프라 담당자는 `values.yaml`, 알람 운영 담당자는 `values-alertmanager.yaml`, SRE는 `values-alert-rules-ops.yaml`만 건드리면 된다.
{: .prompt-tip }

### 분리 구조

```
helm/monitoring/kube-prometheus-stack/
├── values.yaml                    # Base 인프라 설정 (Prometheus, Grafana, Node Exporter)
├── values-alertmanager.yaml       # AlertManager 라우팅 + Slack Receiver 3종
└── values-alert-rules-ops.yaml    # Ops 운영 Alert Rules (19개)
```

### Helm 배포 시 적용 방법

세 파일을 `-f` 옵션으로 순서대로 넘기면 된다. **나중에 지정한 파일이 앞 파일을 오버라이드**하므로 순서가 중요하다.

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/monitoring/kube-prometheus-stack/values.yaml \
  -f helm/monitoring/kube-prometheus-stack/values-alertmanager.yaml \
  -f helm/monitoring/kube-prometheus-stack/values-alert-rules-ops.yaml
```

---

## 3. Grafana Admin Password 보안 강화 — Secrets Manager + ESO

### 문제

```yaml
# values.yaml — git에 평문으로 올라간 상태 ❌
adminPassword: "change-me-after-deploy"
```

git 히스토리에 비밀번호가 영구적으로 남는다. private repo라도 팀원 전체에게 노출되고, 레포 유출 시 즉시 위험해진다.

> **git 히스토리에 한 번 올라간 시크릿은 커밋을 삭제해도 완전히 제거되지 않는다.** 반드시 시크릿 자체를 로테이션해야 한다.
{: .prompt-danger }

### 왜 기존 OAuth Secret과 분리하는가?

Grafana에는 이미 GitHub OAuth용 ExternalSecret(`grafana-github-oauth-secret`)이 있다. 여기에 admin 비밀번호까지 합치는 방법도 있지만, **OAuth 자격증명과 admin 비밀번호는 로테이션 주기와 접근 권한이 다르기 때문에** 별도 시크릿으로 분리하는 것이 현업 권장사항이다.

| 시크릿 | 로테이션 주기 | 접근 권한 |
|---|---|---|
| GitHub OAuth Client Secret | GitHub에서 수동 재발급 | 인프라팀 |
| Grafana Admin Password | 보안 정책에 따라 주기적 변경 | 운영팀 |

### 구성 흐름

```
[Terraform]
aws_secretsmanager_secret: <prefix>-grafana-admin-apne2-secret
  └── secret_string: { "admin-user": "admin", "admin-password": "..." }
        lifecycle { ignore_changes = [secret_string] }  ← 수동 로테이션 보호
              ↓
[ESO: external-secret-grafana-admin.yaml]
ClusterSecretStore(aws-secrets-manager)
  └── K8s Secret: grafana-admin-secret
        ├── admin-user
        └── admin-password
              ↓
[values.yaml]
grafana:
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password
```

> **`lifecycle { ignore_changes = [secret_string] }` 설정의 의미**<br>
> Terraform이 최초 1회 시크릿을 생성한 뒤에는 더 이상 값을 건드리지 않는다.<br>
> 이후 비밀번호 변경은 AWS 콘솔/CLI로 직접 Secrets Manager 값을 수정하면 ESO가 자동으로 K8s Secret에 반영한다.<br>
> 이 설정이 없으면 `terraform apply` 할 때마다 비밀번호가 초기값으로 덮어씌워진다.
{: .prompt-warning }

### Terraform 변경사항

```hcl
# terraform/envs/dev/variables.tf
variable "grafana_admin_password" {
  description = "Grafana admin user password"
  type        = string
  sensitive   = true  # terraform plan 출력에서 마스킹
}

# terraform/envs/dev/main.tf
resource "aws_secretsmanager_secret" "grafana_admin" {
  name        = "<prefix>-grafana-admin-apne2-secret"
  description = "Grafana admin user credentials"
}

resource "aws_secretsmanager_secret_version" "grafana_admin" {
  secret_id = aws_secretsmanager_secret.grafana_admin.id
  secret_string = jsonencode({
    admin-user     = "admin"
    admin-password = var.grafana_admin_password
  })
  lifecycle {
    ignore_changes = [secret_string]
  }
}
```

### ExternalSecret 신규 생성

```yaml
# kubernetes/external-secrets/external-secret-grafana-admin.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: grafana-admin-secret
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: grafana-admin-secret
    creationPolicy: Owner
  data:
  - secretKey: admin-user
    remoteRef:
      key: <prefix>-grafana-admin-apne2-secret
      property: admin-user
  - secretKey: admin-password
    remoteRef:
      key: <prefix>-grafana-admin-apne2-secret
      property: admin-password
```

### values.yaml 변경

```yaml
grafana:
  # adminPassword 항목 완전 제거
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password
```

---

## 4. Alert Rule 오탐 수정

### PodNotReady — system Pod까지 잡는 문제

```yaml
# Before ❌ — kube-system, monitoring 네임스페이스 Pod 전부 포함
expr: kube_pod_status_ready{condition="true"} == 0

# After ✅
expr: |
  kube_pod_status_ready{condition="true", namespace=~"<app-namespace-prefix>.*"} == 0
  and on(namespace, pod)
  kube_pod_status_phase{phase="Running"} == 1
```

두 가지를 동시에 고쳤다.

**첫째**, namespace 필터로 앱 네임스페이스만 감시한다. kube-system, monitoring, karpenter 등의 시스템 Pod는 감시 대상에서 제외한다.

**둘째**, `phase="Running"` 조건을 추가한다.

> Completed/Succeeded 상태의 Job Pod나 Terminating 중인 Pod는 `Ready=0`이어도 정상이다.<br>
> `Running` 상태인 Pod 중에서만 NotReady를 감지하면 오탐을 크게 줄일 수 있다.
{: .prompt-info }

---

### ContainerWaiting — CrashLoopBackOff 중복 감지 문제

```yaml
# Before ❌ — CrashLoopBackOff가 PodCrashLoopBackOff rule과 중복 firing
expr: kube_pod_container_status_waiting_reason{reason!="ContainerCreating"} > 0

# After ✅
expr: |
  kube_pod_container_status_waiting_reason{
    reason!~"ContainerCreating|CrashLoopBackOff",
    namespace=~"<app-namespace-prefix>.*"
  } > 0
```

`CrashLoopBackOff`는 이미 별도의 `PodCrashLoopBackOff` rule이 감지하고 있다. 두 rule이 동일 상황에 동시에 firing하면 Slack에 중복 알람이 쏟아진다.

`ContainerWaiting`이 실제로 감지해야 할 reason: `ErrImagePull`, `ImagePullBackOff`, `CreateContainerConfigError`, `CreateContainerError`, `InvalidImageName`

---

### DeploymentReplicaMismatch — 롤링 업데이트 중 오탐

```yaml
# Before ❌ — for: 5m은 롤링 업데이트(보통 1~3분) 중에도 firing
expr: kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
for: 5m

# After ✅
expr: |
  kube_deployment_spec_replicas{namespace=~"<app-namespace-prefix>.*"}
  != kube_deployment_status_ready_replicas{namespace=~"<app-namespace-prefix>.*"}
for: 10m
```

> `for: 5m`으로 설정했을 때 배포를 할 때마다 롤링 업데이트 중에 알람이 울렸다.<br>
> `for: 10m`으로 늘려서 정상적인 롤링 업데이트는 알람 없이 완료되고, 실제로 배포가 막혔을 때만 울리도록 했다.
{: .prompt-tip }

---

## 5. Dead Rule 제거 및 ALB Controller 알람 개선

### 존재하지 않는 메트릭을 참조하던 rule 2개

```yaml
# ❌ 절대 울리지 않는 dead rule
- alert: AWSAPIPermissionErrorsDetected
  expr: increase(aws_api_call_permission_errors_total[5m]) > 0

- alert: AWSAPIThrottlingDetected
  expr: increase(aws_api_call_service_limit_exceeded_errors_total[5m]) > 0
```

`aws_api_call_permission_errors_total`, `aws_api_call_service_limit_exceeded_errors_total` — 이 두 메트릭은 AWS Load Balancer Controller가 실제로 export하지 **않는다**. Prometheus에 해당 메트릭 자체가 존재하지 않으니 어떤 상황에서도 firing할 수 없다.

> **Alert Rule을 작성할 때는 반드시 Prometheus에 해당 메트릭이 실제로 존재하는지 먼저 확인해야 한다.**<br>
> Grafana → Explore → Metrics browser에서 메트릭 이름을 검색하거나, `kubectl exec`으로 Prometheus 컨테이너에서 직접 쿼리해보는 것이 좋다.
{: .prompt-danger }

### 대체 rule 추가

ALB Controller가 실제로 export하는 메트릭은 `controller_runtime_reconcile_errors_total`이다. 기존 warning rule에 더해 더 심각한 sustained 상태를 감지하는 critical rule을 추가했다.

```yaml
# 기존 (warning): 5분 내 5회 이상
- alert: ALBControllerReconcileErrorsHigh
  expr: increase(controller_runtime_reconcile_errors_total{controller=~"ingress|service"}[5m]) > 5
  for: 5m
  labels:
    severity: warning

# 신규 (critical): 10분 내 3회 이상 지속
- alert: ALBControllerReconcileErrorsSustained
  expr: increase(controller_runtime_reconcile_errors_total{controller=~"ingress|service"}[10m]) > 3
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "ALB Controller Reconcile 오류 지속 발생"
    description: "IAM permission 오류, AWS API throttling 모두 reconcile error로 나타납니다. IRSA 설정과 IAM Policy를 확인하세요."
```

> IAM permission 오류, AWS API throttling 모두 reconcile error로 나타나므로 하나의 메트릭으로 통합 감지가 가능하다.
{: .prompt-info }

---

## 6. Promtail JSON 파이프라인 추가

### 문제

기존 Promtail 설정에는 파이프라인이 없었다. Next.js와 Express 앱이 JSON 형태로 로그를 출력해도 Loki는 그냥 raw text로만 저장했다. Grafana에서 `{level="error"}` 같은 label 기반 필터 쿼리를 쓸 수 없는 상태였다.

### EKS에서의 주의사항 — CRI vs Docker

> EKS는 containerd를 런타임으로 사용한다(EKS 1.24부터 기본). containerd는 CRI(Container Runtime Interface) 포맷으로 로그를 작성하며, Docker의 JSON 포맷과 다르다.
{: .prompt-warning }

```
# Docker 포맷 (docker stage로 파싱)
{"log":"...","stream":"stdout","time":"..."}

# CRI 포맷 (cri stage로 파싱)  ← EKS 환경
2026-06-04T01:00:00.000000000Z stdout F {"level":"info","msg":"Server started"}
```

`docker: {}` stage 대신 반드시 `cri: {}` stage를 사용해야 한다. 잘못 설정하면 로그 파싱이 전혀 되지 않거나 메타데이터가 누락된다.

### 파이프라인 구성

```yaml
# helm/monitoring/promtail/values.yaml
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
  snippets:
    pipelineStages:
      - cri: {}                           # containerd CRI 포맷 파싱
      - match:
          selector: '{namespace=~"<app-namespace-prefix>.*"}'
          stages:
            - json:
                expressions:
                  level: level            # pino/winston의 level 필드
                  status: status          # HTTP status code 필드
            - labels:
                level:
                status:
```

> **`match` stage로 앱 네임스페이스에만 JSON 파싱을 적용한 이유**<br>
> kube-system이나 monitoring 네임스페이스 로그까지 JSON 파싱을 시도하면 불필요한 CPU 사용이 발생하고, 파싱 실패 로그도 노이즈가 된다.<br>
> JSON 파싱 실패(plain text 로그)는 자동으로 무시되므로 `console.log`로 찍힌 비정형 로그가 섞여 있어도 로그 유실이 없다.
{: .prompt-tip }

**적용 후 Loki 쿼리 예시**

```logql
# 에러 로그만 필터링
{namespace="<app-namespace>", level="error"}

# HTTP 500 에러 로그
{namespace="<app-namespace>", status="500"}

# 특정 컨테이너의 에러 로그
{namespace="<app-namespace>", container="nextjs", level="error"}
```

---

## 7. AlertManager group_by 개선 — 알림 폭탄 방지

### 문제

```yaml
group_by: [alertname, namespace, pod, instance]
```

`pod`가 group_by에 포함되어 있으면 동일한 원인의 문제가 10개 Pod에서 동시에 발생할 때 **Slack에 10개의 개별 알림이 쏟아진다**. 배포 직후 롤링 업데이트 중 일시적으로 여러 Pod가 NotReady 상태가 되면 채널이 알림으로 도배된다.

```yaml
# After ✅
group_by: [alertname, namespace]
```

> `namespace` 단위로 그룹핑하면 동일 원인의 알림을 하나로 묶는다.<br>
> Pod 상세 정보는 알림 본문의 `{{ range .Alerts }}` 블록 안에서 이미 모두 표시되므로 group_by에서 제거해도 정보 손실이 없다.
{: .prompt-info }

---

## 8. Loki 컴포넌트 리소스 설정

### 문제

`write`, `read`, `backend`, `gateway` 컴포넌트에 resource requests/limits가 전혀 없었다.

> OOM이 발생하면 쿠버네티스 입장에서 eviction 우선순위를 판단할 근거가 없고, 같은 노드의 다른 Pod까지 영향을 줄 수 있다.<br>
> `requests` 없이 운영하면 BestEffort QoS Class로 분류되어 리소스 부족 시 가장 먼저 eviction된다.
{: .prompt-warning }

### 컴포넌트별 역할과 리소스 설정

| 컴포넌트 | 역할 | requests | limits |
|---|---|---|---|
| `write` | 로그 수신(ingestion), WAL 처리 | cpu: 200m / mem: 512Mi | cpu: 1000m / mem: 1Gi |
| `read` | LogQL 쿼리 처리 | cpu: 200m / mem: 512Mi | cpu: 1000m / mem: 1Gi |
| `backend` | Compactor, Index-gateway | cpu: 100m / mem: 256Mi | cpu: 500m / mem: 512Mi |
| `gateway` | nginx 프록시 (경량) | cpu: 50m / mem: 64Mi | cpu: 200m / mem: 128Mi |

> `chunksCache.allocatedMemory`도 2048MB에서 **1024MB**로 줄였다.<br>
> 초기 이커머스 규모에서 2GB는 과다 할당이다. 실제 로그 볼륨을 모니터링하면서 점진적으로 늘리는 것이 맞다.
{: .prompt-tip }

---

## 9. 신규 Alert Rule 3개 추가

### HPAMaxReplicasReached — 이커머스 트래픽 급증 대응

이커머스에서 가장 중요한 알람 중 하나다. HPA가 `maxReplicas`에 도달해 더 이상 스케일 아웃이 안 되는 상태는 곧 **트래픽 처리 용량 한계**를 의미한다.

```yaml
- alert: HPAMaxReplicasReached
  expr: |
    kube_horizontalpodautoscaler_status_condition{
      condition="ScalingLimited",
      status="true",
      namespace=~"<app-namespace-prefix>.*"
    } == 1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "HPA가 최대 레플리카에 도달했습니다"
    description: |
      HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }}가
      5분 이상 최대 레플리카 한계에 도달해 있습니다.
      트래픽 급증 시 추가 스케일 아웃이 불가능한 상태입니다.
      Deployment의 maxReplicas 값을 확인하고 필요시 상향하세요.
```

> **`ScalingLimited` condition을 사용하는 이유**<br>
> 레플리카 수를 직접 비교하는 방법(`currentReplicas >= maxReplicas`)보다 HPA 자체의 condition을 보는 것이 더 정확하다.<br>
> HPA가 CPU 기준 이외에 다른 이유로도 ScalingLimited 상태가 될 수 있기 때문이다.
{: .prompt-info }

---

### PodPendingTooLong — Karpenter 프로비저닝 실패 감지

Karpenter를 사용하는 환경에서 Pod가 Pending 상태가 10분 이상 지속되면 노드 프로비저닝에 문제가 있다는 신호다. EC2 Spot 용량 부족, IAM 권한 문제, NodePool 설정 오류 등이 원인일 수 있다.

```yaml
- alert: PodPendingTooLong
  expr: |
    kube_pod_status_phase{phase="Pending", namespace=~"<app-namespace-prefix>.*"} > 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Pod가 10분 이상 Pending 상태입니다"
    description: |
      [Recommended Action]
      1. kubectl describe pod {{ $labels.pod }} -n {{ $labels.namespace }}
      2. kubectl get nodeclaim -A  ← Karpenter 노드 프로비저닝 상태
      3. kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter
```

> **`for: 10m`으로 설정한 이유**<br>
> Karpenter가 노드를 프로비저닝하는 데 보통 1~3분이 걸린다.<br>
> `for: 5m` 이하로 설정하면 정상적인 스케일 아웃 중에도 알람이 울릴 수 있다.
{: .prompt-tip }

---

### ExternalSecretSyncError — 시크릿 동기화 실패

ESO가 Secrets Manager 동기화에 실패해도 **기존 K8s Secret이 삭제되지는 않는다**. 그래서 지금 당장 서비스에 영향이 없어 눈치채기 어렵다.

```yaml
- alert: ExternalSecretSyncError
  expr: |
    increase(controller_runtime_reconcile_errors_total{
      controller=~"externalsecret|clustersecretstore|secretstore"
    }[5m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "ExternalSecret 동기화 오류 발생"
    description: |
      ESO가 Secrets Manager와의 동기화에 실패하고 있습니다.
      현재 K8s Secret은 유지되지만, 시크릿 로테이션이 반영되지 않습니다.
      Pod 재시작 시 마운트 실패로 이어질 수 있습니다.
```

> **`severity: critical`로 설정한 이유**<br>
> 동기화 실패가 방치되면 시크릿 만료 또는 로테이션 이후 Pod 재시작 시 서비스 장애로 이어진다.<br>
> 조용히 실패하는(silent failure) 특성이 있어 early detection이 특히 중요하다.
{: .prompt-danger }

---

## 10. blackbox-exporter 중복 Alert Rule 제거

8편에서 blackbox-exporter를 구성할 때 `prometheusRule` 섹션에 `EndpointDown`, `SSLCertExpiringSoon`, `SSLCertExpiringCritical` rule을 직접 넣었다.

그런데 `values-alert-rules-ops.yaml`에도 동일한 rule들이 정의되어 있어서 같은 알람이 **두 번** 울리는 상황이 발생했다.

```yaml
# helm/monitoring/blackbox-exporter/values.yaml
prometheusRule:
  enabled: false  # values-alert-rules-ops.yaml 에서 통합 관리
```

> Alert Rule은 한 곳에서 통합 관리하는 것이 원칙이다. 분산되면 중복 알람이 발생하고, 임계값 변경 시 어디를 수정해야 하는지 혼란이 생긴다.
{: .prompt-warning }

---

## 11. AlertManager Slack 메시지 포맷 — 3단계 severity

```
[CRITICAL] 🚨
  ├── color: #CC0000 (빨간 사이드바)
  ├── pretext: "즉시 대응 필요"
  ├── title_link → Grafana Dashboard
  └── body: 요약 + 설명 + Namespace/Pod 컨텍스트

[WARNING] ⚠️
  ├── color: #FFA500 (주황 사이드바)
  └── (CRITICAL과 동일 구조)

[INFO] ℹ️
  ├── color: #439FE0 (파란 사이드바)
  └── 참고용 이벤트 알림
```

**inhibit_rules** 설정으로 critical이 firing 중일 때 동일 `alertname + namespace`의 warning은 자동으로 억제된다.

```yaml
inhibit_rules:
  - source_matchers:
      - severity = critical
    target_matchers:
      - severity = warning
    equal: [alertname, namespace]
```

---

## 12. 최종 검증 — Alertmanager API로 테스트 알람 발송

신규 rule들이 실제로 Slack까지 잘 전달되는지 확인하기 위해 Alertmanager API를 직접 호출해서 테스트 알람을 발송했다.

> 클러스터에 실제 문제를 일으킬 필요 없이 전체 알림 경로를 검증하는 가장 안전한 방법이다.
{: .prompt-tip }

```bash
# Alertmanager에 포트포워드
kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093 &

# 테스트 알람 발송 (HPAMaxReplicasReached)
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "HPAMaxReplicasReached",
        "severity": "warning",
        "namespace": "<app-namespace>",
        "horizontalpodautoscaler": "hpa-nextjs"
      },
      "annotations": {
        "summary": "HPA가 최대 레플리카에 도달했습니다",
        "description": "테스트 알람입니다."
      },
      "endsAt": "2026-06-04T03:30:00Z"
    }
  ]'
```

`HPAMaxReplicasReached`(warning), `PodPendingTooLong`(warning), `ExternalSecretSyncError`(critical) 세 알람 모두 Slack 정상 수신 확인.

---

## 13. 트러블슈팅

### values.yaml 블록 중복 정의 — 설정이 묵살되는 문제

| 항목 | 내용 |
|---|---|
| 증상 | 분명히 설정을 추가했는데 적용이 안 됨 |
| 원인 | YAML에서 동일한 키를 두 번 정의하면 두 번째가 첫 번째를 완전히 덮어씀. 파일 분리 후 `-f` 순서 실수 |
| 해결 | 각 파일에서 동일한 최상위 키를 중복 정의하지 않도록 구조 정리 |

---

### Loki label cardinality 과다 — 쿼리 느려짐

| 항목 | 내용 |
|---|---|
| 증상 | Grafana에서 Loki 쿼리 속도가 점점 느려짐 |
| 원인 | `status` label에 HTTP status code 값이 들어가면 200, 201, 204, 400, 401, 403, 404, 500, 502, 503... 수십 가지 고유값이 label로 쌓임 |
| 해결 | status label을 `2xx`, `4xx`, `5xx` 그룹으로 집계하거나, label 대신 `| json` 파이프라인으로 쿼리 시점에 파싱 |

> Loki에서 label은 **저카디널리티(low cardinality)** 값만 사용해야 한다. `pod`, `namespace`, `level` 같은 값은 적합하지만, `request_id`, `trace_id`, `user_id` 같은 고유값은 label로 쓰면 안 된다.
{: .prompt-danger }

---

## 14. 최종 체크리스트

| 항목 | 상태 |
|---|:---:|
| values.yaml 파일 3개로 분리 | ✅ |
| Grafana adminPassword 하드코딩 제거 | ✅ |
| PodNotReady 오탐 수정 (namespace + Running phase 필터) | ✅ |
| ContainerWaiting 오탐 수정 (CrashLoopBackOff 중복 제거) | ✅ |
| DeploymentReplicaMismatch 오탐 수정 (for: 10m + namespace 필터) | ✅ |
| Dead Rule 2개 제거 (존재하지 않는 메트릭 참조) | ✅ |
| ALBControllerReconcileErrorsSustained 추가 | ✅ |
| Promtail CRI 파싱 + JSON 파이프라인 (level/status label) | ✅ |
| AlertManager group_by 개선 (pod/instance 제거) | ✅ |
| Loki 컴포넌트 resource limits 설정 | ✅ |
| blackbox-exporter 중복 rule 제거 | ✅ |
| HPAMaxReplicasReached 추가 | ✅ |
| PodPendingTooLong 추가 | ✅ |
| ExternalSecretSyncError 추가 | ✅ |
| Slack 알람 수신 검증 (3개 신규 알람) | ✅ |

---

## 마치며

이번 작업에서 가장 인상적이었던 부분은 **실제로 존재하지 않는 메트릭을 참조하던 dead rule**의 존재였다. 알람 시스템을 운영하면서 "이 rule은 절대 울리지 않는다"는 사실을 모른 채 지나쳤다면, 실제 IAM 권한 문제나 API throttling 상황을 놓쳤을 것이다.

Alert Rule을 작성할 때는 반드시 **Prometheus에 해당 메트릭이 실제로 존재하는지** 먼저 확인해야 한다.

> **EKS 마이그레이션 시리즈**<br>
> 1편: As-Is 분석 + To-Be 아키텍처 설계<br>
> 2편: Terraform VPC/EKS/Karpenter 구성 및 검증<br>
> 3편: Private 클러스터 + VSCode Server 구축<br>
> 4편: ALBC/ESO/ArgoCD GitOps 파이프라인<br>
> 5편: PHP 앱 컨테이너화 → Next.js 재구현 → MSA 분리<br>
> 6편: GitHub Actions + ArgoCD CI/CD + HPA + ALB Ingress Group + GitHub OAuth<br>
> 7편: PLG Monitoring Stack (Prometheus + Loki + Grafana)<br>
> 8편(번외): PLG Monitoring Stack 고도화 (대시보드 + 알림 + Blackbox)<br>
> **9편: 프로덕션 수준 알람 시스템 구축 및 개선**
{: .prompt-tip }
