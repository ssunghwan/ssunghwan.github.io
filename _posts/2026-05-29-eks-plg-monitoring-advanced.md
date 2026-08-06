---
title: "모니터링, 한 단계 더 - PLG 스택 고도화: 데쉬보드, 알림, BlackBox"
date: 2026-05-29 09:00:00 +0900
categories: [Kubernetes, Operations]
tags: [eks, prometheus, loki, grafana, alertmanager, slack, blackbox, plg, monitoring, albc, volumesnapshot]
---

> EKS 마이그레이션 시리즈 여덟 번째 포스팅이다.<br>
> 앞선 포스팅에서 PLG 기본 스택(Prometheus + Loki + Grafana)을 구축했다. 이번에는 대시보드 확장, Loki 버그 해결, Alertmanager Slack 연동, Blackbox Exporter, ALB Controller 메트릭까지 실제 운영에 필요한 고도화 작업을 다룬다.
{: .prompt-info }

---

## 전체 구성 흐름

```
Promtail (DaemonSet, 모든 노드)
  └── 로그 수집 → Loki Gateway
                        ├── Loki Write → S3 (30일 보존)
                        └── Loki Read  ← Grafana 조회

Prometheus
  ├── Node / Pod / K8s 오브젝트 메트릭 수집
  ├── Blackbox Exporter (HTTP/SSL 외부 프로빙)
  ├── ALB Controller 메트릭
  └── AlertManager → Slack (#alerts)

Grafana
  ├── dotdc 대시보드 (Kubernetes 뷰)
  ├── Loki 로그 대시보드
  ├── Blackbox / ALB 대시보드
  └── GitHub OAuth SSO
```

### 데이터 보존 정책

| 데이터 | 보존 기간 | 저장소 |
|---|---|---|
| Prometheus 메트릭 | 30일 (50GB 상한) | EBS gp3 PVC |
| Loki 로그 | 30일 (자동 삭제) | S3 |
| Alertmanager 상태 | 영구 | EBS gp3 PVC |
| Grafana 설정 | 영구 | EBS gp3 PVC |

---

## 1. Grafana 대시보드 확장 (dotdc)

### 기본 대시보드로는 부족한 이유

kube-prometheus-stack에 기본 포함된 **kubernetes-mixin** 대시보드는 Kubernetes SIG에서 만든 표준 대시보드다. 정확하고 신뢰할 수 있지만, 운영자 관점에서 아쉬운 점이 있다.

- **Pod 상태를 한눈에 보기 어려움**: CPU/메모리를 Namespace 단위로만 보여주고, 개별 Pod의 OOM/Restart/Throttle 상태를 직관적으로 표시하지 않음
- **네임스페이스별 리소스 비교가 어려움**: 팀별로 얼마나 리소스를 쓰고 있는지 한 화면에서 비교 불가

### dotdc 대시보드란?

[dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes)는 커뮤니티에서 가장 인기 있는 Kubernetes 대시보드 프로젝트다. kube-prometheus-stack과 100% 호환되며, 기본 mixin 대시보드를 **대체하지 않고 보완**한다.

| 대시보드 | 핵심 패널 |
|---|---|
| `k8s-views-global` | 클러스터 전체 노드 수, CPU/메모리 사용률 |
| `k8s-views-namespaces` | NS별 CPU/메모리 Req/Limit/실사용 비교 |
| `k8s-views-nodes` | 노드별 Pod 수, 리소스 압력 |
| `k8s-views-pods` | **CPU Throttle, OOM, Restart 한눈에** |
| `k8s-system-api-server` | 요청 레이턴시, 에러율 |
| `k8s-system-coredns` | DNS 쿼리율, 캐시 히트율 |
| `k8s-addons-prometheus` | Prometheus 자체 스크래핑 상태 |
| `k8s-addons-trivy-operator` | 보안 스캔 (Trivy 없으면 No data) |

### 설치 방법

```bash
# 다운로드
git clone https://github.com/dotdc/grafana-dashboards-kubernetes.git

# Grafana port-forward
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring &

# admin 비밀번호 확인
GRAFANA_PASS=$(kubectl get secret kube-prometheus-stack-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d)

# 전체 대시보드 한 번에 Import
cd ~/grafana-dashboards-kubernetes/dashboards
for f in *.json; do
  echo "Importing $f ..."
  curl -s -X POST "http://localhost:3000/api/dashboards/db" \
    -H "Content-Type: application/json" \
    -u "admin:$GRAFANA_PASS" \
    -d "{\"dashboard\": $(cat $f), \"overwrite\": true}" \
    | jq -r '.status // .message'
done
```

> **회사 방화벽으로 Grafana UI Import가 안 되는 경우**<br>
> Grafana UI에서 ID로 Import 시 grafana.com에서 JSON을 받아오는데, 방화벽이 막혀있으면 `Failed to fetch` 에러가 발생한다.<br>
> JSON 파일을 직접 다운로드하여 **Upload dashboard JSON file** 방식이나 위 API 방식을 사용하면 된다.
{: .prompt-tip }

### k8s-views-pods 대시보드 읽는 법

이 대시보드에서 **No data**인 패널이 있으면 당황할 수 있는데, 오히려 좋은 신호다.

| 패널 | No data 의미 | 데이터가 나타나는 경우 |
|---|---|---|
| CPU Throttled seconds | 쓰로틀링 없음 ✅ | CPU limits에 걸려서 속도 제한 중 |
| OOM Events | OOM 없음 ✅ | 메모리 limits 초과로 컨테이너 kill |
| Container Restart | 재시작 없음 ✅ | CrashLoopBackOff, OOMKilled 등 |

---

## 2. Loki 업그레이드 및 에러 해결

### 문제 발견

Grafana Explore에서 Loki 로그를 조회했을 때, **error 로그가 비정상적으로 많이** 보였다.

```
error: 1,220건 / info: 776건 / warning: 10건 (5분 기준)
```

실제 에러 내용:

```
msg="negative structured metadata bytes received" userID=fake
retentionHours= isAggregatedMetric=false policyName= size=0
```

### 원인 분석: Loki 3.5.0의 알려진 버그

이 에러는 [GitHub Issue #17371](https://github.com/grafana/loki/issues/17371)에 보고된 Loki 3.5.0의 버그다.

**발생 조건**: Promtail이 structured metadata 없이 로그를 보내면, Loki의 push handler에서 `size=0`인 경우도 에러로 처리한다.

> **왜 심각한가?**<br>
> 로그 수집 자체에는 영향이 없지만, 에러 로그가 분당 수백 건씩 쌓이면서 Promtail이 이 에러 로그를 다시 수집 → Loki에 저장하는 **무한 루프**가 발생한다. S3 비용이 증가하고, Grafana에서 실제 에러와 구분이 어려워진다.
{: .prompt-danger }

### 해결 Step 1: Loki 버전 업그레이드

```bash
# 현재 버전 확인
helm list -n monitoring | grep loki
# → loki  monitoring  3  loki-6.30.1  3.5.0

# 업그레이드
helm repo update
helm upgrade loki grafana/loki \
  -n monitoring \
  -f helm/monitoring/loki/values.yaml \
  --version 6.55.0

# 확인
kubectl get pods -n monitoring \
  -o jsonpath='{.items[?(@.metadata.name=="loki-write-0")].spec.containers[*].image}'
# → docker.io/grafana/loki:3.6.7 ✅
```

### 해결 Step 2: Structured Metadata 비활성화

업그레이드 후에도 에러가 계속됐다. `push.go:202`의 다른 코드 경로에서 같은 버그가 남아있었기 때문이다.

**Structured Metadata란?**

로그에 고카디널리티(high cardinality) 메타데이터를 인덱싱 없이 붙이는 기능이다.

```
# 일반 라벨 (저카디널리티 → 인덱싱됨)
{namespace="monitoring", pod="grafana-xxx"}

# Structured Metadata (고카디널리티 → 인덱싱 안 됨, 쿼리 시 필터 가능)
{namespace="monitoring"} | trace_id="abc123" | request_id="req-456"
```

| 사용 케이스 | 필요 여부 |
|---|---|
| OpenTelemetry(OTLP) 로그 수집 | ✅ 필수 |
| trace_id, request_id 검색 | ✅ 권장 |
| Promtail 기본 구성 (현재 환경) | ❌ 불필요 |

현재 Promtail로 수집하고 있으므로 비활성화해도 로그 수집/검색에 전혀 영향이 없다.

```yaml
# helm/monitoring/loki/values.yaml
loki:
  limits_config:
    allow_structured_metadata: false    # 추가
```

### Before / After

| 항목 | Before (3.5.0) | After (3.6.7 + 설정) |
|---|---|---|
| 에러 로그 (5분) | **1,220건** | **0건** |
| S3 불필요 누적 | 분당 수 MB | 없음 |
| Loki 버전 | 3.5.0 | 3.6.7 |
| Chart 버전 | 6.30.1 | 6.55.0 |

> Loki를 업그레이드할 때는 반드시 **Changelog**를 확인하고, Grafana Explore에서 에러 로그를 확인하는 습관을 들이자. 사용하지 않는 기능은 명시적으로 비활성화하는 것이 안전하다.
{: .prompt-tip }

---

## 3. Loki 로그 대시보드 구성

### PLG인데 L(oki)이 빠졌다?

kube-prometheus-stack을 설치하면 **Prometheus 메트릭 대시보드는 수십 개**가 자동으로 설치된다. 하지만 Loki 로그 대시보드는 **하나도 없다**. Loki는 별도 Helm 차트로 설치했기 때문이다.

모니터링의 핵심은 **"메트릭에서 이상 감지 → 로그에서 원인 분석"** 흐름인데, 로그 대시보드가 없으면 이 흐름이 끊긴다.

### Loki 데이터소스 확인

kube-prometheus-stack values.yaml에서 Loki를 Grafana 데이터소스로 자동 등록한다.

```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local
      access: proxy
      isDefault: false
```

> **URL 설명**: `loki-gateway`는 Loki의 nginx 리버스 프록시다. 모든 Loki 요청은 Gateway를 통해 read/write 컴포넌트로 라우팅된다.
{: .prompt-info }

### 커뮤니티 대시보드 (ID: 15141)

```
Grafana → Dashboards → New → Import → ID: 15141 → Load → Loki 선택 → Import
```

### 커스텀 에러 트래킹 대시보드

커뮤니티 대시보드는 로그 조회에 특화되어 있지만, 에러 통계/추이/Top Pod 같은 운영 뷰가 부족하다.

**Row 1: Overview**

| 패널 | LogQL | 설명 |
|---|---|---|
| 🔴 Errors | `sum(count_over_time({namespace=~"$namespace"} \|~ "(?i)error\|panic\|fatal" [$__range]))` | 전체 에러 건수 |
| 🟡 Warnings | `sum(count_over_time({namespace=~"$namespace"} \|~ "(?i)warn" [$__range]))` | 경고 건수 |
| 📊 Total Logs | `sum(count_over_time({namespace=~"$namespace"} [$__range]))` | 전체 로그 라인 수 |
| Log Level Distribution | `sum by (detected_level)(count_over_time(...))` | error/warn/info 비율 |

**Row 2: Error Tracking**

| 패널 | 설명 |
|---|---|
| Top Error-Producing Pods | 에러를 가장 많이 뱉는 Pod Top 10 (Table) |
| Error Distribution by NS | 네임스페이스별 에러 비율 (Donut Chart) |
| Recent Error Logs | 최신 에러 로그 스트림 (Logs 패널) |

**Variables (드롭다운)**

```
datasource → namespace → pod → container → search (텍스트)
```

`namespace` 선택 → 해당 namespace의 `pod` 목록 자동 갱신 → 해당 pod의 `container` 목록 자동 갱신

> **운영 팁**: PLG 스택 자체가 많은 로그를 생성한다. 실제 워크로드 에러만 보려면 namespace 필터에서 `monitoring`을 제외하자.
{: .prompt-tip }

---

## 4. VolumeSnapshot CRD 설치

### 문제 발견

Grafana 로그 대시보드에서 `kube-system` 네임스페이스의 에러를 확인했을 때, `ebs-csi-controller` Pod에서 **30초~1분 간격으로** 에러가 반복되고 있었다.

```
E0528 07:43:22 reflector.go:204] "Failed to watch"
  err="failed to list *v1.VolumeSnapshotClass:
  the server could not find the requested resource
  (get volumesnapshotclasses.snapshot.storage.k8s.io)"
```

### 원인: CRD 미설치

EBS CSI Driver를 EKS Add-on으로 설치하면 `csi-snapshotter` 사이드카 컨테이너가 기본 포함된다. 이 사이드카는 VolumeSnapshot CRD를 찾으려 하지만, **CRD가 클러스터에 없으면** 계속 에러를 발생시킨다.

> **VolumeSnapshot이란?**<br>
> Kubernetes에서 PVC(EBS 볼륨)의 스냅샷을 선언적으로 관리하는 기능이다.<br>
> CRD를 설치해도 스냅샷이 자동으로 생성되지는 않는다. CRD = "기능 등록", VolumeSnapshot 리소스 생성 = "실제 스냅샷 찍기"다.<br>
> **카메라 앱을 설치했다고 사진이 자동으로 찍히는 것이 아닌 것과 같다.**
{: .prompt-info }

### CRD 설치

```bash
cd kubernetes/storage/volume-snapshot

curl -sLO https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
curl -sLO https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
curl -sLO https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

kubectl apply -f .
```

### 검증

```bash
kubectl logs -n kube-system <ebs-csi-controller-pod> -c csi-snapshotter --tail=5
```

```
# Before
E0528 "Failed to watch" type="*v1.VolumeSnapshotClass"     ❌

# After
I0528 "Caches populated" type="*v1.VolumeSnapshotClass"    ✅
I0528 "Caches populated" type="*v1.VolumeSnapshotContent"  ✅
```

---

## 5. Alertmanager Slack 연동

### 왜 알림이 핵심인가?

모니터링에서 가장 흔한 **안티패턴**은 "대시보드를 만들어 놓고 아무도 안 보는 것"이다.

```
❌ 잘못된 모니터링: 대시보드 → 사람이 주기적으로 확인 → 문제 놓침
✅ 올바른 모니터링: 메트릭 수집 → 임계값 초과 → 자동 알림 → 즉시 대응
```

### Slack Webhook을 Secrets Manager로 관리하는 이유

Slack Webhook URL은 민감 정보다. URL만 알면 누구든 해당 채널에 메시지를 보낼 수 있다.

| 방법 | 보안 | Git 안전 | 운영 편의 |
|---|:---:|:---:|:---:|
| values.yaml에 평문 | ❌ | ❌ 노출 | ⭐ 쉬움 |
| K8s Secret 수동 생성 | ✅ | ✅ | ⚠️ 클러스터 재생성 시 유실 |
| **Secrets Manager + ESO** | ✅ | ✅ | ✅ 자동 동기화 |

현재 프로젝트에서 Grafana OAuth, ArgoCD OAuth도 동일한 패턴으로 관리하고 있으므로 **일관성**을 위해 같은 패턴을 사용한다.

### Terraform — Secrets Manager 리소스

```hcl
resource "aws_secretsmanager_secret" "alertmanager_slack" {
  name        = "<prefix>-alertmanager-slack-apne2-secret"
  description = "Alertmanager Slack Webhook URL"
}

resource "aws_secretsmanager_secret_version" "alertmanager_slack" {
  secret_id = aws_secretsmanager_secret.alertmanager_slack.id
  secret_string = jsonencode({
    webhook_url = var.alertmanager_slack_webhook_url
  })
  lifecycle {
    ignore_changes = [secret_string]  # 수동 변경 시 Terraform이 덮어쓰지 않음
  }
}
```

`terraform.secret.tfvars`에 실제 URL 저장 (`.gitignore`에 포함):

```hcl
alertmanager_slack_webhook_url = "https://hooks.slack.com/services/..."
```

### External Secrets로 K8s Secret 주입

```yaml
# kubernetes/external-secrets/external-secret-alertmanager.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: alertmanager-slack-webhook
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: alertmanager-slack-webhook
    creationPolicy: Owner
  data:
  - secretKey: webhook-url
    remoteRef:
      key: <prefix>-alertmanager-slack-apne2-secret
      property: webhook_url
```

### Alertmanager 라우팅 구조

Alertmanager의 라우팅은 **Tree 구조**다. 알림이 들어오면 root route → child route 순으로 매칭된다.

```yaml
# kube-prometheus-stack values.yaml
alertmanager:
  config:
    global:
      slack_api_url_file: "/etc/alertmanager/secrets/alertmanager-slack-webhook/webhook-url"

    route:
      receiver: "slack-alerts"
      group_by: ["alertname", "namespace", "pod"]
      group_wait: 30s        # 같은 그룹의 알림을 30초간 모음
      group_interval: 5m     # 그룹에 새 알림 추가 시 5분 후 재전송
      repeat_interval: 4h    # 동일 알림 지속 시 4시간마다 반복

      routes:
        - receiver: "slack-critical"
          matchers:
            - severity = critical
          group_wait: 10s        # critical은 10초만 대기 (긴급)
          repeat_interval: 1h    # 1시간마다 반복

    receivers:
      - name: "slack-alerts"
        slack_configs:
          - channel: "#alerts"
            api_url_file: "/etc/alertmanager/secrets/alertmanager-slack-webhook/webhook-url"
            title: '{{ if eq .Status "firing" }}🚨 {{ .CommonLabels.severity | toUpper }} 🚨{{ end }} {{ .CommonAnnotations.summary }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    inhibit_rules:
      - source_matchers:
          - severity = critical
        target_matchers:
          - severity = warning
        equal: ["alertname", "namespace"]
```

> **각 파라미터가 실제로 의미하는 것**<br>
> `group_wait: 30s` — Pod 3개가 동시에 OOM → 30초 기다렸다가 한 번에 전송<br>
> `group_interval: 5m` — 5분 후 4번째 Pod도 OOM → 기존 그룹에 추가해서 재전송<br>
> `repeat_interval: 4h` — 4시간째 해결 안 됨 → 리마인더 알림 재전송
{: .prompt-info }

> **`api_url_file` vs `api_url`**<br>
> `api_url`은 URL이 ConfigMap에 평문으로 노출된다. `api_url_file`은 마운트된 Secret 파일에서 읽어 노출되지 않는다.
{: .prompt-warning }

> **inhibit_rules의 역할**<br>
> 노드 메모리가 92%일 때 `NodeMemoryHigh` (warning, 80%+)와 `NodeMemoryCritical` (critical, 90%+)이 동시에 firing된다.<br>
> inhibit rule이 없으면 2개 알림이 동시에 Slack에 온다. 있으면 critical만 전송된다. 이미 90%인데 80% 경고는 의미가 없기 때문이다.
{: .prompt-tip }

### Helm 적용 및 테스트

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/monitoring/kube-prometheus-stack/values.yaml

# 테스트 알림 발송
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring &

curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "status": "firing",
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "namespace": "monitoring"
    },
    "annotations": {
      "summary": "Alertmanager Slack 연동 테스트",
      "description": "이 알림이 Slack에 도착하면 연동 성공입니다!"
    }
  }]'
```

---

## 6. Custom Alert Rules (PrometheusRules)

### 왜 커스텀 Alert Rule이 필요한가?

kube-prometheus-stack 기본 Alert Rule은 대부분 **플랫폼 레벨** 알림이다 (KubeletDown, NodeNotReady 등). 이커머스 운영에 필요한 **애플리케이션 레벨** 알림은 직접 추가해야 한다.

### 각 Alert Rule 상세 설명

**Pod CrashLoopBackOff (Critical)**

```yaml
- alert: PodCrashLoopBackOff
  expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "Pod CrashLoopBackOff 감지"
    description: "Pod {{ $labels.pod }} ({{ $labels.namespace }})가 CrashLoopBackOff 상태입니다."
```

> 배포 직후 일시적 crash는 정상일 수 있으므로 `for: 3m`으로 3분간 지속될 때만 알림을 발송한다.
{: .prompt-info }

**Pod OOMKilled (Critical)**

```yaml
- alert: PodOOMKilled
  expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Pod OOMKilled 감지"
    description: "Pod {{ $labels.pod }} ({{ $labels.namespace }})가 OOMKilled 됐습니다. 메모리 limits를 확인하세요."
```

> OOM은 즉각 대응이 필요하므로 `for: 1m`으로 설정한다. 이커머스 프로모션 트래픽 급증 시 메모리 limits 초과가 발생할 수 있다.
{: .prompt-warning }

**NodeMemoryHigh (Warning) / NodeMemoryCritical (Critical)**

```yaml
- alert: NodeMemoryHigh
  expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.8
  for: 5m
  labels:
    severity: warning

- alert: NodeMemoryCritical
  expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
  for: 3m
  labels:
    severity: critical
```

> **2단계 알림 전략**: 80%에서 warning → 조치할 시간 확보 → 90%에서 critical → 즉시 대응<br>
> inhibit_rules와 연계해 90% 초과 시 critical만 전송, 80% warning은 억제한다.
{: .prompt-tip }

**PVCUsageHigh (Critical)**

```yaml
- alert: PVCUsageHigh
  expr: kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.9
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "PVC 사용량 90% 초과"
    description: "{{ $labels.persistentvolumeclaim }} PVC가 90% 이상 사용 중입니다."
```

> PVC는 늘릴 수는 있지만 줄일 수 없다. Prometheus PVC(50Gi)가 꽉 차면 메트릭 write가 실패하고, 알림도 오지 않는다 (아이러니). 90%에서 미리 감지해야 한다.
{: .prompt-danger }

### 전체 Alert 커버리지

| 알림 | 심각도 | 조건 |
|---|---|---|
| PodCrashLoopBackOff | 🔴 critical | 3분 이상 |
| PodOOMKilled | 🔴 critical | 1분 이상 |
| NodeMemoryCritical | 🔴 critical | 90%+ |
| PVCUsageHigh | 🔴 critical | 90%+ |
| EndpointDown | 🔴 critical | 2분 이상 다운 |
| SSLCertExpiringCritical | 🔴 critical | 7일 이내 만료 |
| PodRestartRateHigh | 🟡 warning | 1시간 내 5회+ |
| PodNotReady | 🟡 warning | 5분 이상 |
| DeploymentReplicaMismatch | 🟡 warning | 5분 이상 |
| NodeMemoryHigh | 🟡 warning | 80%+ |
| NodeDiskHigh | 🟡 warning | 85%+ |
| SSLCertExpiringSoon | 🟡 warning | 30일 이내 만료 |
| EndpointSlowResponse | 🟡 warning | 응답 3초 초과 |

---

## 7. Loki 로그 보존 정책

### 왜 보존 정책이 필요한가?

보존 정책 없이 Loki를 운영하면 S3에 **무기한으로** 로그가 쌓인다.

| 일일 로그 | 1개월 | 6개월 | 1년 |
|---|---|---|---|
| 1GB/일 | 30GB | 180GB | 365GB |
| 5GB/일 | 150GB | 900GB | 1.8TB |
| 10GB/일 | 300GB | 1.8TB | **3.6TB** |

> S3 Standard 기준 1TB당 약 $23/월. 불필요한 로그 때문에 1년이면 **수십~수백 달러**를 낭비하게 된다.
{: .prompt-warning }

### Compactor의 동작 원리

Loki의 보존 정책은 **Compactor** 컴포넌트가 담당한다. SimpleScalable 모드에서는 `loki-backend` Pod에서 실행된다.

```
S3 Index (TSDB)
    ↓ 10분마다 스캔
Compactor
    ├── 인덱스 병합 (스토리지 최적화)
    └── 만료 로그 탐지
            ↓
        삭제 마킹 (S3에 기록)
            ↓
        2시간 대기 (delete_delay)
            ↓
        실제 삭제 (S3 chunks)
```

### 설정 상세 설명

```yaml
loki:
  limits_config:
    retention_period: 30d              # 로그 보존 기간

  compactor:
    retention_enabled: true            # 보존 정책 활성화
    retention_delete_delay: 2h         # 삭제 전 대기 시간
    delete_request_store: s3           # 삭제 요청 저장소 (storage.type과 일치)
    compaction_interval: 10m           # Compactor 실행 주기
```

| 옵션 | 값 | 이유 |
|---|---|---|
| `retention_period` | `30d` | 개발 환경에서 30일이면 충분. 운영은 90d 권장 |
| `retention_delete_delay` | `2h` | 2시간 내 오삭제 발견 시 복구 가능 |
| `delete_request_store` | `s3` | `storage.type`과 반드시 일치해야 함 |
| `compaction_interval` | `10m` | 너무 자주 하면 S3 API 비용 증가 |

> **`delete_request_store` 누락 시 CrashLoopBackOff**<br>
> `retention_enabled: true` 설정 시 `delete_request_store`를 누락하면 Loki backend Pod가 crash한다.<br>
> `compactor.delete-request-store should be configured when retention is enabled`
{: .prompt-danger }

---

## 8. Blackbox Exporter 배포

### 내부 메트릭 vs 외부 프로빙

```
내부 메트릭 (Prometheus):
  Pod CPU 정상 ✅ + Pod 메모리 정상 ✅ + 컨테이너 Running ✅
          ↓
  하지만 ALB가 unhealthy? DNS 장애? SSL 만료?
          ↓
  실제 사용자: "사이트가 안 열려요" 😱
```

| 원인 | 내부 메트릭 | 외부 프로빙 |
|---|:---:|:---:|
| ALB Target Group unhealthy | 감지 불가 ❌ | DOWN ✅ |
| DNS 설정 오류 | 감지 불가 ❌ | DOWN ✅ |
| SSL 인증서 만료 | 감지 불가 ❌ | 감지 ✅ |
| CDN/WAF 장애 | 감지 불가 ❌ | DOWN ✅ |

Blackbox Exporter는 **사용자와 동일한 관점**에서 HTTP 요청을 보내 응답을 확인한다.

### HTTP Probe의 각 Phase 이해

Blackbox Exporter는 HTTP 요청의 각 단계를 **개별 메트릭으로 분리**하여 측정한다.

```
DNS Resolve → TCP Connect → TLS Handshake → Processing → Transfer
  0.001s         0.005s         0.020s          0.050s      0.010s

probe_http_duration_seconds{phase="resolve"}     = 0.001
probe_http_duration_seconds{phase="connect"}     = 0.005
probe_http_duration_seconds{phase="tls"}         = 0.020
probe_http_duration_seconds{phase="processing"}  = 0.050
probe_http_duration_seconds{phase="transfer"}    = 0.010
```

| Phase | 느려지는 원인 |
|---|---|
| `resolve` | DNS 서버 응답 지연, DNS 캐시 미스 |
| `connect` | 네트워크 지연, TCP 재전송 |
| `tls` | 인증서 체인이 길거나 OCSP stapling 지연 |
| `processing` | 서버 응답 시간 (DB 쿼리, 외부 API 호출 등) |
| `transfer` | 응답 본문이 크거나 네트워크 대역폭 부족 |

### SSL 인증서 모니터링

ACM(AWS Certificate Manager)은 자동 갱신이지만, 간혹 실패하는 경우가 있다. SSL이 만료되면 브라우저에서 접속이 차단된다.

```yaml
# 30일 전 warning
- alert: SSLCertExpiringSoon
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
  labels:
    severity: warning

# 7일 전 critical
- alert: SSLCertExpiringCritical
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 7
  labels:
    severity: critical
```

### 모니터링 대상 및 설치

| 타겟 | URL | 체크 내용 |
|---|---|---|
| 앱 | `https://app.example.com` | HTTP 200 + SSL |
| Grafana | `https://grafana.example.com` | HTTP 200 + SSL |
| ArgoCD | `https://argocd.example.com` | HTTP 200 + SSL |

```bash
helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  -n monitoring \
  -f helm/monitoring/blackbox-exporter/values.yaml \
  --version 11.10.0
```

---

## 9. ALB Controller 메트릭 모니터링

### Kubernetes Controller의 Reconciliation Loop

모든 Kubernetes Controller는 **Reconciliation Loop** 패턴으로 동작한다.

```
① Watch: Ingress 리소스 변경 감지
      ↓
② Reconcile: 현재 상태 vs 원하는 상태 비교
      ↓
③ Act: AWS API 호출 (ALB 생성/수정/삭제)
      ↓
④ Status: Ingress status 업데이트
      ↓
①로 돌아감
```

AWS Load Balancer Controller는 이 과정에서 발생하는 **모든 메트릭을 포트 8080에서 노출**한다.

| 메트릭 | 의미 |
|---|---|
| `controller_runtime_reconcile_total` | reconcile 총 횟수 (성공/실패) |
| `controller_runtime_reconcile_time_seconds` | reconcile 소요 시간 |
| `aws_api_calls_total` | AWS API 호출 수 |
| `aws_api_call_duration_seconds` | AWS API 레이턴시 |
| `aws_api_call_permission_errors_total` | IAM 권한 에러 |
| `aws_api_call_service_limit_exceeded_errors_total` | API 쓰로틀링 |

### 왜 메트릭 Service가 없었나?

ALBC의 Helm 차트는 **Webhook Service(443)만 기본 생성**한다. 메트릭 Service(8080)는 선택사항이라 생성되지 않는다. Prometheus가 스크래핑하려면 Service → ServiceMonitor 두 개가 필요하다.

### Service + ServiceMonitor 생성

```yaml
# kubernetes/monitoring/albc-metrics.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: aws-load-balancer-controller-metrics
  namespace: kube-system
  labels:
    app.kubernetes.io/name: aws-load-balancer-controller
spec:
  selector:
    app.kubernetes.io/name: aws-load-balancer-controller
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  labels:
    release: kube-prometheus-stack    # Prometheus가 인식하는 라벨
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/name: aws-load-balancer-controller
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

> **`release: kube-prometheus-stack` 라벨의 중요성**<br>
> Prometheus Operator는 이 라벨이 있는 ServiceMonitor만 인식한다.<br>
> 라벨이 없으면 ServiceMonitor를 만들어도 Prometheus가 스크래핑 대상에 추가하지 않는다.
{: .prompt-danger }

### 핵심 모니터링 지표

| 지표 | 정상 | 이상 |
|---|---|---|
| Reconcile Success Rate | 99%+ | 90% 이하면 Ingress 변경이 ALB에 반영 안 됨 |
| Reconcile Duration p99 | < 5초 | 30초+ 이면 AWS API 지연 또는 리소스 충돌 |
| AWS API Throttling | 0 | 0 이상이면 AWS API 할당량 확인 필요 |
| Permission Errors | 0 | 0 이상이면 IRSA/IAM 정책 확인 필요 |

---

## 10. EKS 관리형 컴포넌트 비활성화

### 왜 No data인가?

EKS에서 Grafana의 **etcd, Scheduler, Controller Manager** 대시보드가 No data인 것은 **정상**이다.

EKS는 관리형 Kubernetes이므로 컨트롤 플레인을 **AWS가 관리**한다.

| 컴포넌트 | 메트릭 수집 | 이유 |
|---|:---:|---|
| CoreDNS | ✅ | 워커 노드에서 Pod로 실행 |
| kubelet | ✅ | 워커 노드에서 실행 |
| API Server | ✅ | 메트릭 엔드포인트 공개 |
| **etcd** | ❌ | AWS 관리, 비공개 |
| **kube-scheduler** | ❌ | AWS 관리, 비공개 |
| **controller-manager** | ❌ | AWS 관리, 비공개 |

```yaml
# kube-prometheus-stack values.yaml
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeControllerManager:
  enabled: false
```

> 이 설정으로 Prometheus가 존재하지 않는 Target을 스크래핑하려는 시도를 멈추고, 불필요한 scrape error가 사라진다.
{: .prompt-info }

---

## 11. 트러블슈팅

### admissionWebhooks 블록 중복 정의

| 항목 | 내용 |
|---|---|
| 증상 | `prometheusOperator` 설정이 일부만 적용됨 |
| 원인 | values.yaml에 `prometheusOperator` 블록을 두 번 정의하면 두 번째가 첫 번째를 덮어씀 |
| 해결 | `prometheusOperator` 블록 **하나**에 모든 설정을 통합 |

```yaml
# 올바른 설정 — 하나의 블록에 통합
prometheusOperator:
  tolerations:
    - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
  nodeSelector:
    role: system
  admissionWebhooks:
    patch:
      tolerations:
        - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
    job:
      tolerations:
        - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
```

---

### Loki Compactor CrashLoopBackOff

```
CONFIG ERROR: compactor.delete-request-store should be configured when retention is enabled
```

| 항목 | 내용 |
|---|---|
| 원인 | `retention_enabled: true` 설정 시 `delete_request_store` 누락 |
| 해결 | `delete_request_store: s3` 반드시 함께 설정 |

---

### Grafana OAuth 콜백 URL 오류 (Grafana vs ArgoCD)

| 항목 | 내용 |
|---|---|
| 증상 | GitHub 인증 후 `redirect_uri is not associated with the application` |
| 원인 | ArgoCD callback URL(`/api/dex/callback`)과 Grafana callback URL(`/login/github`)을 혼동 |
| 해결 | Grafana OAuth App의 callback URL을 `https://grafana.example.com/login/github`로 설정 |

---

## 최종 구성 체크리스트

| 항목 | 상태 |
|---|:---:|
| Grafana 대시보드 확장 (dotdc 8개) | ✅ |
| Loki 3.5.0 → 3.6.7 업그레이드 | ✅ |
| Structured metadata 에러 해결 | ✅ |
| Loki 로그 대시보드 (커뮤니티 + 커스텀) | ✅ |
| VolumeSnapshot CRD 설치 | ✅ |
| Alertmanager Slack 연동 (Secrets Manager + ESO) | ✅ |
| Custom Alert Rules 13개 | ✅ |
| Loki 로그 보존 정책 (30d) | ✅ |
| Blackbox Exporter (HTTP/SSL 프로브) | ✅ |
| ALB Controller 메트릭 모니터링 | ✅ |
| EKS 관리형 컴포넌트 비활성화 | ✅ |

---

> **EKS 마이그레이션 시리즈**<br>
> 1편: As-Is 분석 + To-Be 아키텍처 설계<br>
> 2편: Terraform VPC/EKS/Karpenter 구성 및 검증<br>
> 3편: Private 클러스터 + VSCode Server 구축<br>
> 4편: ALBC/ESO/ArgoCD GitOps 파이프라인<br>
> 5편: PHP 앱 컨테이너화 → Next.js 재구현 → MSA 분리<br>
> 6편: GitHub Actions + ArgoCD CI/CD + HPA + ALB Ingress Group + GitHub OAuth<br>
> 7편: PLG Monitoring Stack (Prometheus + Loki + Grafana)<br>
> **8편(번외): PLG Monitoring Stack 고도화 (대시보드 + 알림 + Blackbox)**
{: .prompt-tip }
