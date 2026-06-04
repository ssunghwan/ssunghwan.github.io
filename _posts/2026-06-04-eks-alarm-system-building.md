---
title: "Building a Production-Grade Alarm System for EKS E-commerce (Prometheus + AlertManager + Slack)"
date: 2026-06-04
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, prometheus, alertmanager, slack, loki, promtail, karpenter, hpa, external-secrets, monitoring, alerting]
toc: true
---

> **EKS 마이그레이션 시리즈**
> - 1편: [Infrastructure configuration and application packaging](/posts/legacy-php-ecommerce-docker-eks-migration/)
> - 2편: [Terraform Infrastructure Setup to Karpenter Configuration](/posts/eks-migration-terraform-karpenter/)
> - 3편: [VSCode Server Setup Guide for a Private Cluster](/posts/eks-private-vscode-server/)
> - 4편: [Installing ALBC, ESO, and ArgoCD components for configuring GitOps pipelines](/posts/eks-albc-eso-argocd-gitops/)
> - 5편: [From containerizing a test Next.js app to MSA separation](/posts/eks-migration-full-deployment/)
> - 6편: [Building a GitHub Actions + ArgoCD GitOps CI/CD Pipeline](/posts/eks-cicd-github-actions-argocd/)
> - 7편: [Building a PLG Monitoring Stack (Prometheus + Loki + Grafana)](/posts/eks-plg-monitoring-stack/)
> - 8편: [PLG Monitoring Stack Advancement (Dashboard + Notifications + Blackbox)](/posts/eks-plg-monitoring-advanced/)
> - **9편: Building a Production-Grade Alarm System for EKS E-commerce (현재 글)**

---

지난 8편에서 PLG 스택(Prometheus + Loki + Grafana)을 구축하고 Slack 알림, Blackbox Exporter, 커스텀 Alert Rule까지 기본 알람 시스템을 완성했다.

그런데 막상 배포하고 나서 찬찬히 들여다보니 문제가 한두 가지가 아니었다.

- `values.yaml` 한 파일에 Prometheus 설정, AlertManager 라우팅, Alert Rules가 전부 뒤섞여 있어서 알람 규칙 하나 바꾸려면 인프라 설정 파일 전체를 건드려야 했고
- Grafana admin 비밀번호가 git에 **평문으로** 올라가 있었고
- PodNotReady 알람은 kube-system Pod까지 전부 잡아서 **상시 firing** 상태였으며
- AWS API 메트릭을 참조하는 Alert Rule 2개는 해당 메트릭이 **아예 존재하지 않아서** 절대 울리지 않는 dead rule이었다

이번 편에서는 이 문제들을 하나씩 짚어가며 이커머스 EKS 환경에 실제로 적합한 프로덕션 수준의 알람 시스템으로 개선하는 전 과정을 다룬다.

---

## 1. 전체 아키텍처 개요

개선 후 알람 시스템의 전체 흐름은 아래와 같다.

```
[Karpenter Nodes]
  ├── system nodes → Prometheus, AlertManager, Grafana, Loki
  ├── web nodes    → purina-nextjs Pod
  └── api nodes    → purina-api Pod

[메트릭 수집]
  Prometheus ─── kube-state-metrics (Pod/Node/HPA 상태)
              ├── node-exporter (노드 리소스)
              ├── blackbox-exporter (외부 엔드포인트)
              └── ALB Controller (reconcile 메트릭)

[로그 수집]
  Promtail (DaemonSet) → Loki Gateway → Loki (S3 백엔드)
  ↳ CRI 파싱 + 앱 네임스페이스 JSON 파이프라인 (level/status label)

[알림]
  Prometheus → AlertManager → Slack
    ├── #purinapetcare-kubernetes-alerts [CRITICAL] (즉시, 1h repeat)
    ├── #purinapetcare-kubernetes-alerts [WARNING]  (4h repeat)
    └── #purinapetcare-kubernetes-alerts [INFO]     (참고용)
```

Alert Rule은 총 **19개**, 5개 카테고리로 분류된다.

| 카테고리 | Alert Rule 수 | 주요 감지 대상 |
|----------|:---:|------|
| Kubernetes Workload | 6 | CrashLoop, OOM, Restart, NotReady, Waiting, Replica 불일치 |
| Node / Storage | 4 | 메모리, 디스크, PVC 사용률 |
| External / Blackbox | 4 | 엔드포인트 다운, 응답 지연, SSL 만료 |
| HPA / Karpenter / ESO | 3 | 스케일링 한계, Pod Pending, 시크릿 동기화 오류 |
| ALB Controller | 2 | Reconcile 오류 |

---

## 2. values.yaml 파일 분리 — 관심사 분리

### 왜 분리해야 하는가?

기존에는 `kube-prometheus-stack/values.yaml` 한 파일에 Prometheus 스펙, AlertManager 라우팅 설정, 커스텀 Alert Rules가 전부 들어 있었다.

AlertManager Slack 메시지 템플릿을 수정하거나 Alert Rule 임계값 하나 바꾸려고 해도, Prometheus 리소스 설정이나 Grafana 설정이 포함된 메인 values 파일을 통째로 수정해야 했다. 리뷰 범위도 넓어지고, 협업 시 충돌 위험도 높다.

### 분리 구조

```
helm/monitoring/kube-prometheus-stack/
├── values.yaml                    # Base 인프라 설정 (Prometheus, Grafana, Node Exporter)
├── values-alertmanager.yaml       # AlertManager 라우팅 + Slack Receiver 3종
└── values-alert-rules-ops.yaml    # Ops 운영 Alert Rules (19개)
```

**values.yaml** — 인프라 담당자가 건드리는 영역. Prometheus retention, Grafana datasource, resource 설정 등.

**values-alertmanager.yaml** — 알람 운영 담당자 영역. Slack 채널 라우팅, 메시지 포맷, repeat_interval 등.

**values-alert-rules-ops.yaml** — SRE/Ops 담당자 영역. Alert 임계값, for 기간, 설명 문구 등.

### Helm 배포 시 적용 방법

세 파일을 `-f` 옵션으로 순서대로 넘기면 된다.

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
adminPassword: "NPPKvbflsk26@^"
```

git 히스토리에 비밀번호가 영구적으로 남는다. private repo라도 팀원 전체에게 노출되고, 레포 유출 시 즉시 위험해진다.

### 해결 전략 — B 방법: 별도 시크릿 분리

Grafana에는 이미 GitHub OAuth용 ExternalSecret(`grafana-github-oauth-secret`)이 있다. 여기에 admin 비밀번호까지 합치는 방법도 있지만, **OAuth 자격증명과 admin 비밀번호는 로테이션 주기와 접근 권한이 다르기 때문에** 별도 시크릿으로 분리하는 것이 현업 권장사항이다.

### 구성 흐름

```
[Terraform Step 19]
aws_secretsmanager_secret: nkor-dv-pupekrb2c-grafana-admin-apne2-secret
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

`lifecycle { ignore_changes = [secret_string] }` 설정 덕분에 Terraform이 최초 1회 시크릿을 생성한 뒤에는 더 이상 값을 건드리지 않는다. 이후 비밀번호 변경은 AWS 콘솔이나 CLI를 통해 직접 Secrets Manager 값을 수정하면 ESO가 자동으로 K8s Secret에 반영한다.

### Terraform 변경사항

```hcl
# terraform/envs/dev/variables.tf
variable "grafana_admin_password" {
  description = "Grafana admin user password"
  type        = string
  sensitive   = true
}

# terraform/envs/dev/main.tf — Step 19
resource "aws_secretsmanager_secret" "grafana_admin" {
  name        = "nkor-dv-pupekrb2c-grafana-admin-apne2-secret"
  description = "PurinaPetcare Grafana admin user credentials"
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
      key: nkor-dv-pupekrb2c-grafana-admin-apne2-secret
      property: admin-user
  - secretKey: admin-password
    remoteRef:
      key: nkor-dv-pupekrb2c-grafana-admin-apne2-secret
      property: admin-password
```

> 기존 `grafana-github-oauth-secret` ExternalSecret은 변경하지 않는다. OAuth 자격증명과 admin 비밀번호를 동일한 ExternalSecret으로 합치면 한쪽의 로테이션이 다른 쪽에 영향을 줄 수 있다.

---

## 4. Alert Rule 오탐 수정 — 3개

### 4-1. PodNotReady — system Pod까지 잡는 문제

```yaml
# Before ❌ — kube-system, monitoring 네임스페이스 Pod 전부 포함
expr: kube_pod_status_ready{condition="true"} == 0

# After ✅
expr: |
  kube_pod_status_ready{condition="true", namespace=~"nkor-dv-.*"} == 0
  and on(namespace, pod)
  kube_pod_status_phase{phase="Running"} == 1
```

두 가지를 동시에 고쳤다.

첫째, `namespace=~"nkor-dv-.*"` 필터로 앱 네임스페이스만 감시. kube-system, monitoring, karpenter 등의 시스템 Pod는 감시 대상에서 제외.

둘째, `phase="Running"` 조건 추가. Completed/Succeeded 상태의 Job Pod나 Terminating 중인 Pod는 Ready가 0이어도 정상이다. Running 상태인 Pod 중에서만 NotReady를 감지하도록 했다.

### 4-2. ContainerWaiting — CrashLoopBackOff 중복 감지 문제

```yaml
# Before ❌ — CrashLoopBackOff가 PodCrashLoopBackOff rule과 중복
expr: kube_pod_container_status_waiting_reason{reason!="ContainerCreating"} > 0

# After ✅
expr: |
  kube_pod_container_status_waiting_reason{
    reason!~"ContainerCreating|CrashLoopBackOff",
    namespace=~"nkor-dv-.*"
  } > 0
```

`CrashLoopBackOff`는 이미 별도의 `PodCrashLoopBackOff` rule이 감지하고 있다. 두 rule이 동일 상황에 동시에 firing하면 Slack에 중복 알람이 쏟아진다. `ContainerWaiting`은 실제로 잡아야 할 reason에 집중하도록 했다.

실제로 이 rule이 감지해야 할 주요 상황: `ErrImagePull`, `ImagePullBackOff`, `CreateContainerConfigError`, `CreateContainerError`, `InvalidImageName`

### 4-3. DeploymentReplicaMismatch — 롤링 업데이트 중 오탐

```yaml
# Before ❌ — for: 5m은 롤링 업데이트(보통 1~3분) 중에도 firing
expr: kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
for: 5m

# After ✅
expr: |
  kube_deployment_spec_replicas{namespace=~"nkor-dv-.*"}
  != kube_deployment_status_ready_replicas{namespace=~"nkor-dv-.*"}
for: 10m
```

`for: 5m`으로 설정했을 때, 배포를 할 때마다 롤링 업데이트 중에 알람이 울렸다. `for: 10m`으로 늘려서 정상적인 롤링 업데이트는 알람 없이 완료되고, 실제로 배포가 막혔을 때만 울리도록 했다.

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

`aws_api_call_permission_errors_total`, `aws_api_call_service_limit_exceeded_errors_total` — 이 두 메트릭은 AWS Load Balancer Controller가 실제로 export하지 않는다. Prometheus에 해당 메트릭 자체가 존재하지 않으니 이 rule들은 어떤 상황에서도 firing할 수 없다.

### 대체 rule 추가

ALB Controller가 실제로 export하는 메트릭은 `controller_runtime_reconcile_errors_total`이다. 기존 `ALBControllerReconcileErrorsHigh`(warning)에 더해 더 심각한 sustained 상태를 감지하는 critical rule을 추가했다.

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
```

IAM permission 오류, AWS API throttling 모두 reconcile error로 나타나므로 하나의 메트릭으로 통합 감지가 가능하다.

---

## 6. Promtail JSON 파이프라인 추가

### 문제

기존 Promtail 설정에는 파이프라인이 없었다. Next.js와 Express 앱이 JSON 형태로 로그를 출력해도 Loki는 그냥 raw text로만 저장했다. Grafana에서 `{level="error"}` 같은 label 기반 필터 쿼리를 쓸 수 없는 상태.

### EKS에서의 주의사항 — CRI vs Docker

EKS는 containerd를 런타임으로 사용한다(EKS 1.24부터 기본). containerd는 CRI(Container Runtime Interface) 포맷으로 로그를 작성하고, Docker의 JSON 포맷과 다르다.

```
# Docker 포맷 (docker stage로 파싱)
{"log":"...","stream":"stdout","time":"..."}

# CRI 포맷 (cri stage로 파싱)  ← EKS
2026-06-04T01:00:00.000000000Z stdout F {"level":"info","msg":"Server started"}
```

`docker: {}` stage 대신 반드시 `cri: {}` stage를 사용해야 한다.

### 파이프라인 구성

```yaml
# helm/monitoring/promtail/values.yaml
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
  snippets:
    pipelineStages:
      - cri: {}                           # CRI 포맷 파싱 (containerd)
      - match:
          selector: '{namespace=~"nkor-dv-.*"}'
          stages:
            - json:
                expressions:
                  level: level            # pino/winston의 level 필드
                  status: status          # HTTP status code 필드
            - labels:
                level:
                status:
```

`match` stage를 사용해서 앱 네임스페이스(`nkor-dv-.*`)에만 JSON 파싱을 적용했다. kube-system이나 monitoring 네임스페이스 로그까지 JSON 파싱을 시도하면 불필요한 CPU 사용이 발생하고, 파싱 실패 로그도 노이즈가 된다.

JSON 파싱 실패(plain text 로그)는 자동으로 무시되므로 `console.log`로 찍힌 비정형 로그가 섞여 있어도 로그 유실이 없다.

**적용 후 Loki 쿼리 예시**

```logql
# 에러 로그만 필터링
{namespace="nkor-dv-pupekrb2c-purinapp-kr", level="error"}

# HTTP 500 에러 로그
{namespace="nkor-dv-pupekrb2c-purinapp-kr", status="500"}

# 특정 앱의 에러 로그
{namespace="nkor-dv-pupekrb2c-purinapp-kr", container="nextjs", level="error"}
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

`namespace` 단위로 그룹핑하면 동일 원인의 알림을 하나로 묶는다. Pod 상세 정보는 알림 본문의 `{{ range .Alerts }}` 블록 안에서 이미 모두 표시되므로 group_by에서 제거해도 정보 손실이 없다.

---

## 8. Loki 컴포넌트 리소스 설정

### 문제

`write`, `read`, `backend`, `gateway` 컴포넌트에 resource requests/limits가 전혀 없었다. OOM이 발생하면 쿠버네티스 입장에서 eviction 우선순위를 판단할 근거가 없고, 같은 노드의 다른 Pod까지 영향을 줄 수 있다.

### 컴포넌트별 역할과 리소스 설정

| 컴포넌트 | 역할 | requests | limits |
|----------|------|----------|--------|
| `write` | 로그 수집 ingestion, WAL 처리 | cpu: 200m / mem: 512Mi | cpu: 1000m / mem: 1Gi |
| `read` | LogQL 쿼리 처리 | cpu: 200m / mem: 512Mi | cpu: 1000m / mem: 1Gi |
| `backend` | Compactor, Index-gateway | cpu: 100m / mem: 256Mi | cpu: 500m / mem: 512Mi |
| `gateway` | nginx 프록시 (경량) | cpu: 50m / mem: 64Mi | cpu: 200m / mem: 128Mi |

`chunksCache.allocatedMemory`도 2048MB에서 **1024MB**로 줄였다. 초기 이커머스 규모에서 2GB는 과다 할당이다.

---

## 9. 신규 Alert Rule 3개 추가

### 9-1. HPAMaxReplicasReached — 이커머스 트래픽 급증 대응

이커머스에서 가장 중요한 알람 중 하나다. HPA가 `maxReplicas`에 도달해 더 이상 스케일 아웃이 안 되는 상태는 곧 **트래픽 처리 용량 한계**를 의미한다.

```yaml
- alert: HPAMaxReplicasReached
  expr: |
    kube_horizontalpodautoscaler_status_condition{
      condition="ScalingLimited",
      status="true",
      namespace=~"nkor-dv-.*"
    } == 1
  for: 5m
  labels:
    severity: warning
    category: platform
  annotations:
    summary: "HPA가 최대 레플리카에 도달했습니다"
    description: |-
      [Impact]
      HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} 가 5분 이상
      최대 레플리카 한계에 도달해 있습니다.
      트래픽 급증 시 추가 스케일 아웃이 불가능한 상태입니다.
      ...
```

`kube_horizontalpodautoscaler_status_condition{condition="ScalingLimited"}` 메트릭을 사용했다. 레플리카 수를 직접 비교하는 방법(`currentReplicas >= maxReplicas`)보다 HPA 자체의 condition을 보는 것이 더 정확하다. HPA가 CPU 기준 이외에 다른 이유로도 ScalingLimited 상태가 될 수 있기 때문이다.

### 9-2. PodPendingTooLong — Karpenter 프로비저닝 실패 감지

Karpenter를 사용하는 환경에서 Pod가 Pending 상태가 10분 이상 지속되면 노드 프로비저닝에 문제가 있다는 신호다. EC2 Spot 용량 부족, IAM 권한 문제, NodePool 설정 오류 등이 원인일 수 있다.

```yaml
- alert: PodPendingTooLong
  expr: |
    kube_pod_status_phase{phase="Pending", namespace=~"nkor-dv-.*"} > 0
  for: 10m
  labels:
    severity: warning
    category: platform
  annotations:
    summary: "Pod가 10분 이상 Pending 상태입니다"
    description: |-
      [Recommended Action]
      1. kubectl describe pod {{ $labels.pod }} -n {{ $labels.namespace }}
      2. kubectl get nodeclaim -A  ← Karpenter 노드 프로비저닝 상태
      3. kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter
```

`for: 10m`으로 설정한 이유는 Karpenter가 노드를 프로비저닝하는 데 보통 1~3분이 걸리기 때문이다. 5분 이하로 설정하면 정상적인 스케일 아웃 중에도 알람이 울릴 수 있다.

### 9-3. ExternalSecretSyncError — 시크릿 동기화 실패

ESO가 Secrets Manager 동기화에 실패해도 **기존 K8s Secret이 삭제되지는 않는다**. 그래서 지금 당장 서비스에 영향이 없어 눈치채기 어렵다. 하지만 이 상태가 지속되면 시크릿 로테이션이 반영되지 않고, Pod 재시작 시 마운트 실패로 이어진다.

```yaml
- alert: ExternalSecretSyncError
  expr: |
    increase(controller_runtime_reconcile_errors_total{
      controller=~"externalsecret|clustersecretstore|secretstore"
    }[5m]) > 0
  for: 5m
  labels:
    severity: critical    # ← 즉각 대응 필요
    category: platform
```

`severity: critical`로 설정한 이유: 동기화 실패가 방치되면 시크릿 만료 또는 로테이션 이후 Pod 재시작 시 서비스 장애로 이어진다. 조용히 실패하는(silent failure) 특성이 있어 early detection이 특히 중요하다.

---

## 10. blackbox-exporter 중복 Alert Rule 제거

8편에서 blackbox-exporter를 구성할 때 `prometheusRule` 섹션에 `EndpointDown`, `SSLCertExpiringSoon`, `SSLCertExpiringCritical` rule을 직접 넣었다.

그런데 이번에 `values-alert-rules-ops.yaml`에도 동일한 rule들이 정의되어 있어서 같은 알람이 **두 번** 울리는 상황이 발생했다.

해결은 간단하다. blackbox-exporter 쪽의 `prometheusRule`을 비활성화하고, `values-alert-rules-ops.yaml`에서 통합 관리한다.

```yaml
# helm/monitoring/blackbox-exporter/values.yaml
prometheusRule:
  enabled: false  # values-alert-rules-ops.yaml 에서 통합 관리
```

---

## 11. AlertManager Slack 메시지 포맷 — 3단계 severity

AlertManager의 Slack receiver는 severity별로 3가지 포맷을 사용한다.

```
[CRITICAL] 🚨
  ├── color: #CC0000 (빨간 사이드바)
  ├── pretext: "즉시 대응 필요"
  ├── title_link → Grafana Dashboard
  └── body: 요약 + 설명 + Namespace/Pod/Container 컨텍스트 + Runbook 링크

[WARNING] ⚠️
  ├── color: #FFA500 (주황 사이드바)
  ├── pretext: "주의 필요"
  └── (CRITICAL과 동일 구조)

[INFO] ℹ️
  ├── color: #439FE0 (파란 사이드바)
  └── 참고용 이벤트 알림
```

**inhibit_rules** 설정으로 critical이 firing 중일 때 동일 `alertname + namespace`의 warning은 자동으로 억제된다. 같은 이슈에 대해 critical + warning 두 개가 동시에 울리는 노이즈를 방지한다.

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

신규 rule들이 실제로 Slack까지 잘 전달되는지 확인하기 위해 Alertmanager API를 직접 호출해서 테스트 알람을 발송했다. 클러스터에 실제 문제를 일으킬 필요 없이 전체 알림 경로를 검증하는 가장 안전한 방법이다.

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
        "namespace": "nkor-dv-pupekrb2c-purinapp-kr",
        "horizontalpodautoscaler": "hpa-nextjs"
      },
      "annotations": {
        "summary": "HPA가 최대 레플리카에 도달했습니다",
        "description": "..."
      },
      "endsAt": "2026-06-04T03:30:00Z"
    }
  ]'
```

`HPAMaxReplicasReached`(warning), `PodPendingTooLong`(warning), `ExternalSecretSyncError`(critical) 세 알람 모두 Slack에 정상 수신 확인.

---

## 13. 완료 체크리스트

| 완료 | 항목 | 비고 |
|------|------|------|
| ✅ | values.yaml 파일 3개로 분리 | 관심사 분리 |
| ✅ | Grafana adminPassword 하드코딩 제거 | Secrets Manager + ESO 분리 |
| ✅ | PodNotReady 오탐 수정 | namespace + Running phase 필터 |
| ✅ | ContainerWaiting 오탐 수정 | CrashLoopBackOff 중복 제거 |
| ✅ | DeploymentReplicaMismatch 오탐 수정 | for: 10m + namespace 필터 |
| ✅ | Dead Rule 2개 제거 | 존재하지 않는 메트릭 참조 |
| ✅ | ALBControllerReconcileErrorsSustained 추가 | critical 2단계 알람 |
| ✅ | Promtail CRI 파싱 + JSON 파이프라인 | level/status label 추출 |
| ✅ | AlertManager group_by 개선 | pod/instance 제거 |
| ✅ | Loki 컴포넌트 resource limits 설정 | chunksCache 2GB → 1GB |
| ✅ | blackbox-exporter 중복 rule 제거 | prometheusRule disabled |
| ✅ | HPAMaxReplicasReached 추가 | 트래픽 급증 감지 |
| ✅ | PodPendingTooLong 추가 | Karpenter 프로비저닝 실패 감지 |
| ✅ | ExternalSecretSyncError 추가 | ESO 동기화 오류 감지 |
| ✅ | Slack 알람 수신 검증 | 전체 3개 신규 알람 테스트 완료 |
| ⬜ | purina-api/nextjs ServiceMonitor | 앱 레벨 HTTP 메트릭 (추후 작업) |

---

## 마치며

이번 작업에서 가장 인상적이었던 부분은 **실제로 존재하지 않는 메트릭을 참조하던 dead rule**의 존재였다. 알람 시스템을 운영하면서 "이 rule은 절대 울리지 않는다"는 사실을 모른 채 지나쳤다면, 실제 IAM 권한 문제나 API throttling 상황을 놓쳤을 것이다.

Alert Rule을 작성할 때는 반드시 **Prometheus에 해당 메트릭이 실제로 존재하는지** 먼저 확인해야 한다. Grafana Explore → Metrics browser에서 메트릭 이름을 검색하거나, `kubectl exec`으로 Prometheus 컨테이너에서 직접 쿼리해보는 것이 좋다.

다음 편에서는 `purina-api`와 `purina-nextjs`에 `prom-client`를 추가해서 HTTP 에러율, P95 응답시간 같은 앱 레벨 메트릭을 Prometheus로 수집하고, 이를 기반으로 한 비즈니스 레벨 Alert Rule을 구성하는 것을 다룰 예정이다.

---

*이 글은 시리즈의 9번째 포스팅입니다.*  
*GitHub: [ssunghwan/purina-kr-infra](https://github.com/ssunghwan/purina-kr-infra)*
