---
title: "Grafana Dashboard Improvements, Alloy WAL Persistence, and Slack Alert Standardization"
date: 2026-06-30 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, grafana, prometheus, loki, alloy, alertmanager, slack, karpenter, valkey, arc, monitoring]
---

> 이 포스팅은 하루 동안 진행한 모니터링 스택 전면 개편 작업을 기록한다.<br>
> ALB/Overview 대시보드 버그 수정, Executive 신규 대시보드 생성, Alloy WAL 영속화, Loki 파이프라인 표준화, Slack 알림 규칙 개선, ARC Runner PrometheusRule 신규 추가까지 총 12개 항목을 다룬다.
{: .prompt-info }

---

## 1. 작업 전체 요약

| # | 항목 | 결과 |
|---|---|---|
| 1 | ALB 대시보드 — targetGroupBinding 컨트롤러 누락 수정 | ✅ |
| 2 | Overview — PVC Unhealthy 26 false positive 수정 | ✅ |
| 3 | Executive 대시보드 신규 생성 (커스텀 브랜드) | ✅ |
| 4 | Active Firing Alerts "No data" 처리 | ✅ |
| 5 | Node Exporter MacOS/AIX 내장 대시보드 비활성화 | ✅ |
| 6 | Alloy WAL 영속화 (hostPath) | ✅ |
| 7 | Loki 파이프라인 EKS 이커머스 표준화 | ✅ |
| 8 | Loki 대시보드 전면 개편 | ✅ |
| 9 | 고아 대시보드 "Loki Kubernetes Logs" 삭제 | ✅ |
| 10 | Slack 알림 규칙 EKS 이커머스 표준 개선 | ✅ |
| 11 | ARC Runner PrometheusRule 3개 신규 추가 | ✅ |

---

## 2. ALB 대시보드 — targetGroupBinding 컨트롤러 누락 수정

### 배경 — AWS LBC와 Karpenter가 메트릭을 공유하는 이유

AWS Load Balancer Controller(LBC)는 Go 기반 Kubernetes 컨트롤러로, `controller-runtime` 프레임워크를 사용한다. `controller-runtime`은 Reconcile 루프의 성능 지표를 자동으로 Prometheus 메트릭으로 노출하는데, 이 메트릭 이름이 `controller_runtime_reconcile_total`이다.

Karpenter 역시 동일한 `controller-runtime` 프레임워크를 사용한다. 따라서 **두 컴포넌트가 동일한 메트릭 이름을 공유**하며, `controller` 레이블로 어느 컨트롤러의 Reconcile인지 구분한다.

```
controller_runtime_reconcile_total{controller="ingress"}            ← AWS LBC: ALB Ingress
controller_runtime_reconcile_total{controller="service"}            ← AWS LBC: NLB Service
controller_runtime_reconcile_total{controller="targetGroupBinding"} ← AWS LBC: TargetGroupBinding CRD
controller_runtime_reconcile_total{controller="nodeclaim"}          ← Karpenter
controller_runtime_reconcile_total{controller="nodepool"}           ← Karpenter
```

ALB 대시보드는 LBC 관련 컨트롤러만 보여야 하므로 `nodeclaim`, `nodepool`은 제외하고 `ingress`, `service`, `targetGroupBinding`만 포함해야 한다.

### TargetGroupBinding이란?

TargetGroupBinding은 AWS LBC가 제공하는 **CRD**다. ALB/NLB의 Target Group과 Kubernetes Service를 직접 연결하는 역할을 한다.

```
[외부 트래픽]
    │
    ▼
[ALB Target Group]  ←──── TargetGroupBinding CRD
    │                      (AWS LBC가 이 CRD를 Reconcile)
    ▼
[Kubernetes Service]
    │
    ▼
[Pod]
```

Target Group에 파드 IP를 직접 등록하거나, Ingress 생성 시 Target Group을 구성하는 과정에서 LBC가 TargetGroupBinding을 Reconcile한다. 이 컨트롤러의 메트릭이 없으면 **LBC의 핵심 동작(파드 → Target Group 등록/해제)을 모니터링하지 못하게 된다.**

### 문제 발견

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
curl -s "http://localhost:9090/api/v1/label/controller/values" | jq '.data[]'
```

```
"ingress"
"nodeclaim"
"nodepool"
"service"
"targetGroupBinding"   ← 이것이 대시보드 필터에서 빠져 있었음
```

기존 필터 `controller=~"ingress|service"`는 `targetGroupBinding`을 누락해 Reconcile Success Rate, Reconcile Errors, Reconciliation Rate, Duration p99, Duration p50 — **5개 패널 전부**에서 TargetGroupBinding 메트릭이 보이지 않았다.

### 수정 내용

5개 패널 모두 정규식에 `targetGroupBinding` 추가:

```diff
- controller=~"ingress|service"
+ controller=~"ingress|service|targetGroupBinding"
```

```promql
# Reconcile Success Rate (예시)
sum(rate(
  controller_runtime_reconcile_total{
    namespace="kube-system",
    controller=~"ingress|service|targetGroupBinding",
    result="success"
  }[5m]
)) by (controller)
```

---

## 3. Overview 대시보드 — PVC Unhealthy 26 false positive 수정

### 배경 — kube-state-metrics의 Phase 메트릭 구조

`kube-state-metrics`는 Kubernetes 리소스의 상태를 Prometheus 메트릭으로 변환해 노출하는 컴포넌트다. `kube_persistentvolumeclaim_status_phase` 메트릭은 직관적이지 않은 구조를 가진다.

PVC의 Phase는 세 가지다: `Bound`, `Lost`, `Pending`. kube-state-metrics는 **PVC 하나당 Phase 종류 수만큼 시리즈를 생성**하며, 현재 Phase인 것만 값이 `1`이고 나머지는 `0`이다.

```
PVC: data-valkey-node-0 (현재 상태: Bound)

kube_persistentvolumeclaim_status_phase{phase="Bound"}   = 1  ← 현재 상태
kube_persistentvolumeclaim_status_phase{phase="Lost"}    = 0
kube_persistentvolumeclaim_status_phase{phase="Pending"} = 0
```

즉, PVC 1개 = 시리즈 3개. PVC 13개 = 시리즈 39개.

### 원인 분석

```promql
# PVC Unhealthy (기존 — 문제)
count(kube_persistentvolumeclaim_status_phase{phase!="Bound"})
```

`phase!="Bound"` 조건은 `Lost`와 `Pending` 시리즈를 모두 선택한다. 모든 PVC가 `Bound` 상태일 때도 `Lost`(=0), `Pending`(=0) 시리즈는 **항상 존재**하므로 `count()`가 이들을 그대로 센다.

PVC 13개 × `phase!="Bound"` 해당 Phase 2종(`Lost`, `Pending`) = **26**

실제로는 Unhealthy PVC가 하나도 없는데 26으로 표시된 것이다.

### 수정 내용

`== 1` 필터를 추가해 **실제로 해당 Phase인 시리즈(값=1)만** 집계한다.

```diff
# PVC Healthy
- count(kube_persistentvolumeclaim_status_phase{phase="Bound"})
+ count(kube_persistentvolumeclaim_status_phase{phase="Bound"} == 1) or vector(0)

# PVC Unhealthy
- count(kube_persistentvolumeclaim_status_phase{phase!="Bound"})
+ count(kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1) or vector(0)
```

> **`or vector(0)`를 붙이는 이유**<br>
> Unhealthy PVC가 0건일 때 Grafana에서 "No data"가 아닌 **숫자 0**으로 표시하기 위함이다.<br>
> 쿼리 결과 행이 0건이면 Grafana는 패널 레벨 "No data" 오버레이를 보이는데, `vector(0)`는 항상 값 0을 반환하는 더미 시리즈 역할을 한다.
{: .prompt-tip }

---

## 4. Executive 대시보드 신규 생성

### 생성 배경

기존 모니터링 대시보드는 운영자(SRE) 관점으로 설계되어 있어 상세한 메트릭 지식이 없는 사람이 보기 어려웠다. 비즈니스 담당자나 개발팀이 한눈에 서비스 전체 현황을 파악할 수 있는 **단일 화면(Single Screen) 대시보드**가 필요했다.

요구사항:
- 커스텀 브랜드 헤더 (로고 + 서비스명 + 클러스터 정보)
- 서비스 가용성, 인프라 리소스, ArgoCD 상태를 단일 화면에서 확인
- Dark theme 기반, Auto-refresh 30s
- 비운영자도 이해할 수 있는 직관적 레이블

### 커스텀 브랜드 HTML 헤더 패널

Grafana는 기본적으로 Text 패널에서 HTML/CSS를 sanitize(제거)한다. XSS 공격 방지 목적이다.

```yaml
# helm/monitoring/kube-prometheus-stack/values.yaml
grafana:
  grafana.ini:
    security:
      disable_sanitize_html: true
```

> 이 설정은 **관리자가 통제하는 내부 대시보드에서만** 사용하며, 외부 사용자가 대시보드 콘텐츠를 편집할 수 없는 환경이어야 한다.
{: .prompt-warning }

헤더 패널은 좌측에 브랜드 로고 + 서비스명, 우측에 클러스터 정보를 표시한다.

```html
<div style="
  background: #1a1a2e;
  padding: 14px 24px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  gap: 20px;
  border-left: 4px solid #E2001A;
">
  <img src="<brand-logo-url>" height="36" style="opacity:0.9;" />
  <span style="color:white; font-size:20px; font-weight:700;">
    <서비스명> platform
  </span>
  <span style="color:rgba(255,255,255,0.5); font-size:15px;">
    |  Health Dashboard
  </span>
  <span style="color:rgba(255,255,255,0.35); font-size:12px; margin-left:auto;">
    EKS Cluster · ap-northeast-2 · Auto-refresh 30s
  </span>
</div>
```

### 실제 대시보드 레이아웃 (스크린샷 기준)

**섹션 1 — 상단 핵심 지표**

상단 행은 좌측 2/3을 timeseries 2개가 차지하고, 우측에 gauge 2개가 세로로 배치된다.

| 패널 | 타입 | 측정값 |
|---|---|---|
| Memory / CPU | timeseries | memory ~20%, cpu ~10% |
| Blackbox Probe Rate | timeseries | ~80 ops/s (1h 전 비교선 포함) |
| Memory | gauge | 24.4% |
| P95 Latency | gauge | 221 ms |

좌측 timeseries 아래 행에는 stat 패널 2개가 위치한다.

| 패널 | 값 |
|---|---|
| Container Restarts (1h) | 0 |
| Running Pods | 87 |

`probe rate(-1h)` 비교선을 함께 표시해 현재 probe 빈도가 평소와 달라졌는지 한눈에 확인할 수 있다.

**섹션 2 — 네임스페이스별 리소스 현황**

좌측은 stacked timeseries, 우측은 bar chart로 나란히 배치된다.

CPU Usage by Namespace (stacked timeseries):

| Namespace | CPU |
|---|---|
| monitoring | 0.253 cores |
| `<app-namespace>` | 0.138 cores |
| kube-system | 0.062 cores |
| argocd | 0.041 cores |
| karpenter | 0.035 cores |
| external-secrets | 0.002 cores |
| arc-system / arc-runners | < 0.001 cores |

Top Namespaces Memory (bar chart):

| Namespace | Memory |
|---|---|
| monitoring | 3.42 GB |
| argocd | 1.13 GB |
| kube-system | 1.02 GB |
| `<app-namespace>` | 300 MB |
| karpenter | 217 MB |

**섹션 3 — HTTP 응답 시간 분포**

좌측은 HTTP Probe Response Time timeseries, 우측은 Percentile 테이블이다. 평상시 100ms 이하를 유지하나 일시적으로 600ms+ spike 구간이 발생하는 것이 확인된다.

| Percentile | 응답시간 |
|---|---|
| P25 | 87.6 ms |
| P50 | 92.8 ms |
| P75 | 149 ms |
| P90 | 211 ms |
| P95 | 221 ms |

> **HTTP Probe Response Time에서 600ms+ spike가 보인다.**<br>
> Blackbox Exporter의 probe interval 또는 timeout 설정을 확인해야 한다. DNS lookup latency나 TLS handshake 지연일 가능성이 있다.
{: .prompt-warning }

**섹션 4 — Uptime & Service Status**

좌측에 Uptime 24h stat 패널(`<서비스명>` 100%), 우측에 Service Status 테이블이 배치된다.

| URL | Status |
|---|---|
| https://argocd.`<domain>` | ✅ |
| https://grafana.`<domain>` | ✅ |
| https://`<service>.<domain>` | ✅ |

**섹션 5 — ArgoCD 상태**

stat 패널 3개가 가로로 나열된다. 값이 0이면 green, 1 이상이면 자동으로 red로 변하도록 Threshold를 설정했다.

| 패널 | 값 | 색상 |
|---|---|---|
| ArgoCD Total Apps | 20 | green |
| ArgoCD OutOfSync Apps | 0 | green |
| ArgoCD Degraded Apps | 0 | green |

**섹션 6 — SSL 만료 & PVC 사용률**

SSL Certificate Expiry 테이블과 PVC Usage 테이블이 나란히 배치된다. SSL 테이블은 `blackbox-exporter`가 각 엔드포인트의 인증서 만료일을 보여주고, PVC 테이블은 각 노드의 PVC 사용률을 표시한다.

**섹션 7 — ArgoCD Applications 전체 현황**

전체 20개 Application의 Namespace / Health / Application명 / Sync Status를 테이블로 표시한다. Health가 `Degraded`이거나 Sync Status가 `OutOfSync`인 행은 붉은색으로 강조된다.

### 서비스 상태(Uptime) — Blackbox Exporter

서비스 URL의 24h 가용성은 `probe_success` 메트릭의 평균으로 계산한다.

```promql
# Uptime 24h (%)
avg_over_time(probe_success{instance="https://<service-domain>"}[24h]) * 100
```

**Probe Rate (실시간 + 1h 비교):**

```promql
# 현재 probe rate
rate(probe_success_total[1m])

# 1시간 전 probe rate (비교선)
rate(probe_success_total[1m] offset 1h)
```

### HTTP 응답 시간 분포 — Percentile Table

P95 단일 값이 아닌 P25~P95 전체 분포를 테이블로 함께 제공해 응답 지연의 성격을 파악할 수 있게 한다.

```promql
histogram_quantile(0.25, rate(probe_duration_seconds_bucket[5m])) * 1000  # ms
histogram_quantile(0.50, rate(probe_duration_seconds_bucket[5m])) * 1000
histogram_quantile(0.75, rate(probe_duration_seconds_bucket[5m])) * 1000
histogram_quantile(0.90, rate(probe_duration_seconds_bucket[5m])) * 1000
histogram_quantile(0.95, rate(probe_duration_seconds_bucket[5m])) * 1000
```

현재 측정값 (P25: 87.6ms / P50: 92.8ms / P75: 149ms / P90: 211ms / P95: 221ms)은 이커머스 기준으로 양호한 수치다.

### ArgoCD stat 패널 Threshold

```json
"thresholds": {
  "steps": [
    { "color": "green", "value": null },
    { "color": "red",   "value": 1 }
  ]
}
```

`OutOfSync Apps`, `Degraded Apps` 패널은 값이 0이면 green, 1 이상이면 red로 자동 변경된다.

---

## 5. Active Firing Alerts — No data 처리

### 배경 — Prometheus ALERTS 메트릭의 특수한 동작

Prometheus에는 특수 메트릭 `ALERTS`가 있다. 이 메트릭은 **알림이 실제로 발화(firing) 중일 때만 시리즈가 생성**된다.

```
# 발화 중인 알림이 있을 때
ALERTS{alertname="PodCrashLooping", alertstate="firing", severity="warning"} = 1

# 발화 중인 알림이 없을 때
→ ALERTS 메트릭 자체가 존재하지 않음 (시리즈 0개)
```

일반 메트릭이라면 값이 0인 시리즈라도 존재하겠지만, `ALERTS`는 비어 있는 것이 "정상"을 의미한다.

### Grafana Table 패널의 "No data" 문제

| 상황 | 해결책 |
|---|---|
| 쿼리가 시리즈를 반환하지만 특정 셀 값이 없음 | `fieldConfig.defaults.noValue` 설정 |
| 쿼리가 **행 자체를 0개 반환** | `noValue`로 처리 불가 — 패널 레벨 오버레이 표시 |

`ALERTS` 메트릭이 없을 때는 쿼리 결과 행이 0개이므로 두 번째 상황에 해당한다.

### 해결 — Synthetic "All Clear" 행

Target A와 Target B 두 개의 쿼리를 병합하는 방식으로 해결한다.

**Target A** — 실제 발화 알림 쿼리:
```promql
ALERTS{alertstate="firing", alertname!="Watchdog"}
```

> `alertname!="Watchdog"`: Prometheus가 정상 동작 중임을 나타내는 `Watchdog` 알림(항상 발화 상태)을 제외한다.
{: .prompt-info }

**Target B** — Synthetic "All Clear" 쿼리:
```promql
label_replace(
  label_replace(
    label_replace(
      (count(ALERTS{alertstate="firing", alertname!="Watchdog"}) or vector(0)) == 0,
      "alertname", "✓ All Clear", "", ""),
    "severity", "healthy", "", ""),
  "namespace", "—", "", "")
```

단계별 동작:

```
1. count(ALERTS{alertstate="firing", alertname!="Watchdog"})
   → 발화 중인 알림 수 집계 / 알림 없으면 결과 없음

2. ... or vector(0)
   → 결과가 없으면 0으로 대체 (항상 값을 반환하도록 보장)

3. ... == 0
   → 알림 수가 0일 때만 값 1 반환 / 알림이 있으면 결과 없음

4. label_replace × 3
   → alertname = "✓ All Clear"
   → severity  = "healthy"
   → namespace = "—"
```

**Transformation 체인:**

```
① Merge          → Target A + Target B 결과를 하나의 테이블로 합침
② Filter by value → Target B 값 컬럼 == 1인 행 제거 (알림이 있을 때 B행 제거)
③ Organize fields → 표시 컬럼 순서 정리 (alertname, severity, namespace)
```

결과:
- 알림 없음 → "✓ All Clear" 행만 표시
- 알림 있음 → 실제 알림 행 표시, "All Clear" 행은 Filter에 의해 제거

---

## 6. Node Exporter MacOS/AIX 내장 대시보드 비활성화

### 문제 현상

Grafana 대시보드 목록에 `Node Exporter MacOS` 대시보드가 존재했지만 데이터가 전혀 없었다.

### 원인

`kube-prometheus-stack` Helm 차트는 기본적으로 다양한 운영체제를 위한 Node Exporter 대시보드 ConfigMap을 생성한다.

```bash
kubectl get configmap -n monitoring | grep "node-exporter\|nodes"

kube-prometheus-stack-nodes          # Linux 노드용 (정상)
kube-prometheus-stack-nodes-darwin   # macOS 노드용 (EKS에 불필요)
kube-prometheus-stack-nodes-aix      # IBM AIX 서버용 (EKS에 불필요)
```

EKS 노드는 모두 Amazon Linux 2023 기반이므로 darwin, aix 대시보드는 영구적으로 데이터가 없다.

### 수정

```yaml
# helm/monitoring/kube-prometheus-stack/values.yaml
nodeExporter:
  operatingSystems:
    linux:
      enabled: true    # EKS Amazon Linux 2023 — 유지
    aix:
      enabled: false   # IBM AIX 서버 없음
    darwin:
      enabled: false   # macOS 노드 없음
```

ArgoCD 동기화 후 관련 ConfigMap이 삭제되어 Grafana 대시보드 목록에서 제거됨을 확인했다.

---

## 7. Alloy — WAL 영속화 및 로그 파이프라인 표준화

### 배경 — Grafana Alloy와 loki.source.kubernetes

현재 스택은 Promtail을 **Grafana Alloy**로 교체한 상태다. Alloy의 `loki.source.kubernetes` 컴포넌트는 노드 파일시스템 접근 없이 **Kubernetes API Server의 Log Streaming 엔드포인트**를 통해 로그를 수집한다.

```
[Alloy DaemonSet - 각 노드]
    │
    │  ① discovery.kubernetes "pods" — 현재 노드의 파드 목록 조회
    │
    ▼
loki.source.kubernetes
    │
    │  ② GET /api/v1/namespaces/{ns}/pods/{pod}/log?follow=true&sinceTime={WAL 위치}
    │
    ▼
loki.process "pipeline"   → 필터링 / 레이블 추출
    │
    ▼
loki.write "default"       → Loki 전송
```

`sinceTime` 파라미터가 핵심이다. Alloy는 WAL에 저장된 마지막 읽은 위치를 이 파라미터로 전달해 중복 없이 이어서 읽는다.

### WAL 유실로 인한 로그 지연 문제

#### 증상

```
Loki 최신 nkor 로그: 2026-06-29 14:59:36
현재 시각:          2026-06-30 05:41:31
지연:               약 14시간 42분
```

#### 원인 분석

```bash
kubectl get daemonset -n monitoring alloy \
  -o jsonpath='{.spec.template.spec.containers[0].args}'

["run", "/etc/alloy/config.alloy",
 "--storage.path=/tmp/alloy",    ← 문제: /tmp는 컨테이너 재시작 시 초기화됨
 ...]
```

`/tmp`는 컨테이너의 임시 파일시스템이다. 파드가 재시작되면 WAL이 사라진다.

Alloy 로그에서 실제 증거를 확인했다.

```
# WAL이 없는 경우 (재시작 후) — 파드 생성 시점부터 다시 읽기 시작
opened log stream  target=nkor-dv.../api
  start_time=2026-06-28T04:47:01.110Z   ← 파드 생성 시점 (2일 전)

# WAL이 없을 때 Go zero time
start_time=0001-01-01T00:00:00.000Z    ← WAL 없음, Kubernetes API가 파드 생성 시점부터 전체 로그 반환
```

### 해결: hostPath WAL 영속화

#### 왜 PVC가 아닌 hostPath인가?

DaemonSet은 `volumeClaimTemplates`를 지원하지 않는다. 이는 StatefulSet 전용 기능이다.

| 방법 | 장점 | 단점 |
|---|---|---|
| `hostPath` | 구성 단순, PVC 불필요, 즉시 적용 | 노드 교체 시 WAL 유실 |
| `local-path-provisioner + PVC` | 노드별 PVC 자동 생성 가능 | 추가 컴포넌트 설치 필요 |

Alloy DaemonSet은 노드 교체(Karpenter Scale-In)가 발생해도 `older_than = "168h"` 필터가 과도한 백로그 전송을 막아주므로 **hostPath로 충분**하다.

```yaml
# helm/alloy/values.yaml

alloy:
  storagePath: /var/lib/alloy        # /tmp/alloy → 영속 경로로 변경

  mounts:
    extra:
    - name: alloy-wal
      mountPath: /var/lib/alloy

controller:
  volumes:
    extra:
    - name: alloy-wal
      hostPath:
        path: /var/lib/alloy-wal
        type: DirectoryOrCreate      # 디렉토리 없으면 자동 생성
```

**영속화 후 동작:**

```
Alloy 파드 재시작
    │
    ▼
/var/lib/alloy-wal 마운트 (노드 파일시스템에서 기존 WAL 복구)
    │
    ▼
WAL에서 마지막 읽은 위치 확인
    │
    ▼
해당 위치부터 API 스트리밍 재개 (백로그 없음)
```

### 백로그 즉시 처리 — older_than 임시 변경

WAL 영속화를 적용해도 첫 번째 재시작 직후에는 WAL이 없으므로 이미 쌓인 14시간 백로그를 처리해야 했다. `older_than` 값을 임시로 줄여 백로그를 즉시 skip했다.

```yaml
# 임시 적용
stage.drop {
  older_than          = "30m"
  drop_counter_reason = "log_too_old"
}
```

```
Alloy 재시작 (WAL 없음)
    │
    ▼
6/28 04:47 부터 로그 스트리밍 시작
    │
    ├── 6/28 ~ 6/30 05:10 로그 (30분보다 오래됨) → 즉시 DROP
    └── 6/30 05:11 ~ 현재 로그 (30분 이내)       → Loki 전송
    │
    ▼
수 분 내 catch-up 완료
```

Alloy 메트릭으로 처리 현황 확인:
```
loki_process_dropped_lines_total{reason="log_too_old"} 456  ← 백로그 drop 수
loki_write_sent_entries_total                          157   ← 최근 로그 전송 수
```

작업 완료 후 정상 설정인 `168h`로 복원했다.

### EKS 이커머스 표준 파이프라인 구성

**변경 전후 비교:**

| 항목 | Before | After |
|---|---|---|
| 헬스체크 노이즈 필터 범위 | `kube-system` 네임스페이스만 | 전 클러스터 공통 적용 |
| 필터 대상 패턴 | `/healthz`, `/readyz`, `/livez`, `/metrics` | 위 + `/health/`, `/ping`, `kube-probe`, `"GET / HTTP"` |
| nkor JSON 추출 필드 | `level`, `status` | `level`, `status`, `method`, `statusCode` |

#### 헬스체크 노이즈 필터 전역화

```alloy
stage.drop {
  expression          = "(?i)kube-probe|/healthz|/health/|/readyz|/livez|/metrics|/ping|\\\"GET / HTTP"
  drop_counter_reason = "probe_noise"
}
```

#### nkor NestJS 구조화 로그 필드 추출

```alloy
stage.match {
  selector = `{namespace=~"nkor-dv-.*"}`

  stage.json {
    expressions = {
      level      = "level",
      status     = "status",
      method     = "method",
      statusCode = "statusCode",
    }
  }

  stage.labels {
    values = {
      level  = ""
      status = ""
      method = ""
      # statusCode는 레이블로 승격하지 않음 (카디널리티 폭발 방지)
    }
  }
}
```

> **`statusCode`를 레이블로 승격하지 않는 이유**<br>
> 레이블 값이 많아질수록(고카디널리티) Loki 인덱스 크기가 폭발적으로 증가한다.<br>
> 숫자 필드(200, 201, 400, 404, 500…)는 LogQL 파이프라인(`| json | statusCode >= 500`)으로 처리한다.
{: .prompt-warning }

---

## 8. Loki 대시보드 — EKS 이커머스 표준 전면 개편

### 개편 전 문제점

**문제 1 — 네임스페이스 범위가 nkor-.\*로 고정**

nkor 앱이 idle 상태이면 로그가 생성되지 않아 항상 "No data"가 됐다.

```diff
{
  "name": "namespace",
- "regex": "nkor-.*",
+ "regex": "",        # 전체 네임스페이스 표시
- "allValue": "nkor-.*",
+ "allValue": ".+"    # All 선택 시 전체
}
```

**문제 2 — app 변수 없이 pod만 있어 드릴다운 불편**

`app` 레이블을 중간 단계로 추가해 계층적 필터링이 가능하게 했다.

```json
{
  "name": "app",
  "type": "query",
  "query": {
    "label": "app",
    "stream": "{namespace=~\"$namespace\"}",
    "type": 1
  },
  "allValue": ".+",
  "includeAll": true,
  "multi": true
}
```

드릴다운 흐름:
```
All namespaces
  → [<app-namespace> 선택]
    → All apps
      → [<app-namespace>-api 선택]
        → All pods → [specific pod]
          → All containers → 로그 조회
```

**문제 3 — 에러 패턴이 텍스트 로그에만 맞춰져 있음**

```diff
- |~ "(?i)error|panic|fatal"
+ |~ "(?i)error|panic|fatal|exception|\"level\":\"error\"|level=error"
```

### 신규 패널 추가

#### HTTP 5xx 에러 추적

```logql
sum(count_over_time(
  {namespace=~"$namespace", app=~"$app"}
    |~ "\"statusCode\":5|status[=: ]5[0-9][0-9]| 5[0-9][0-9] "
  [$__range]
))
```

세 가지 패턴을 동시에 감지:
- `"statusCode":5` — NestJS JSON 로그
- `status[=: ]5[0-9][0-9]` — 텍스트 로그 (`status=500`, `status: 500`)
- ` 5[0-9][0-9] ` — nginx access log

Threshold: `0 → 🟢` / `1~9 → 🟠` / `10+ → 🔴`

#### Error Rate Trend

```logql
sum by (namespace)(
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}
      |~ "(?i)error|panic|fatal|\"level\":\"error\"|level=error"
    [1m]
  )
)
/
sum by (namespace)(
  count_over_time({namespace=~"$namespace", app=~"$app"} [1m])
)
```

평시에 에러 비율이 1% 미만이라면, 5%로 급등하는 시점을 스파이크로 즉시 감지할 수 있다.

#### Live Logs prettifyLogMessage 활성화

```diff
- "prettifyLogMessage": false,
+ "prettifyLogMessage": true,
```

JSON 로그를 Grafana가 자동으로 파싱해 들여쓰기된 형태로 표시한다.

#### 기본 시간 범위 확장

```diff
- "time": {"from": "now-30m", "to": "now"}
+ "time": {"from": "now-1h",  "to": "now"}
```

---

## 9. 고아 대시보드 삭제 — Loki Kubernetes Logs

### 근본 원인 분석

Grafana API로 상세 조회한 결과 3가지 문제를 발견했다.

**문제 1 — 변수 쿼리가 Prometheus 방식**

```json
// 잘못된 방식 (Prometheus용)
"query": "label_values(namespace)"

// Loki용 올바른 방식
"query": { "label": "namespace", "stream": "", "type": 1 }
```

`label_values(namespace)`는 Prometheus datasource를 대상으로 동작하며, Loki datasource에서는 지원되지 않는다.

**문제 2 — 존재하지 않는 `stream` 레이블**

구버전 Loki/Promtail은 `stream=stdout` / `stream=stderr`를 레이블로 사용했다. 현재 Alloy 기반 스택은 이 레이블을 생성하지 않으므로 쿼리 결과가 항상 0건이다.

**문제 3 — 수동 import로 GitOps 관리 밖**

`provisioned: false`는 이 대시보드가 ConfigMap(GitOps)이 아닌 Grafana UI에서 직접 import됐음을 의미한다.

### 처리

구조적으로 현재 환경과 호환되지 않으며, 동일한 역할의 `Kubernetes / Logs / Overview` 대시보드가 이미 존재하므로 삭제했다.

```bash
curl -X DELETE \
  -u "admin:${ADMIN_PASS}" \
  "http://localhost:3001/api/dashboards/uid/o6-BGgnnk"

# {"message":"Dashboard Loki Kubernetes Logs deleted","uid":"o6-BGgnnk"}
```

---

## 10. 앱 로그 부재 — 향후 개선 방향

이번 작업 중 nkor API / Next.js 파드가 정상 운영 중에도 stdout 로그를 거의 생성하지 않음을 확인했다. 이는 **HTTP access log를 stdout으로 출력하는 미들웨어가 없기 때문**이다.

**NestJS API:**
```typescript
// main.ts
import morgan from 'morgan';
app.use(morgan('combined'));  // stdout으로 access log 출력
```

**Next.js:**
```javascript
// server.js
const morgan = require('morgan');
app.use(morgan('combined'));
```

미들웨어 추가 후 아래 형태의 로그가 Loki에 저장된다.

```json
{"method":"GET","url":"/api/products","status":200,"responseTime":45}
```

Loki 대시보드에서 메서드별, 상태코드별, 응답시간 분포를 모니터링할 수 있게 된다.

---

## 11. Slack 알림 규칙 EKS 이커머스 표준 개선

### 배경 — values.yaml 단일 파일 원칙

kube-prometheus-stack ArgoCD 앱은 multi-source로 구성되어 있으며, ArgoCD가 참조하는 values 파일은 `values.yaml` 단 하나다. `values-alertmanager.yaml`, `values-alert-rules-ops.yaml` 등 다른 파일은 ArgoCD가 마운트하지 않아 **모든 alert rule은 `values.yaml` 내 `additionalPrometheusRulesMap` 블록에 인라인으로 작성해야 한다.**

### 기존 룰 문제 식별

| # | 룰 | 문제 |
|---|---|---|
| 1 | PodNotReady 등 4개 | `namespace=~"nkor-dv-.*"` — monitoring/argocd/karpenter Pod 장애 무감지 |
| 2 | EndpointSlowResponse | 임계값 3초 — 이커머스 허용 지연(2초)보다 느슨함 |
| 3 | ALBControllerReconcileErrorsHigh | warning 기준 `> 5` — 너무 느슨함 |
| 4 | ALBControllerReconcileErrorsSustained | critical 기준 `> 3` — warning보다 낮아 역전 현상 |
| 5 | ArgoCDAppMissing | `for: 5m` — ArgoCD 재시작 시 false positive |
| 6 | Karpenter/Valkey | 알림 없음 — 이커머스 핵심 인프라 모니터링 공백 |

### namespace 필터 전역 확장

```diff
- namespace=~"nkor-dv-.*"
+ namespace!~"kube-system"
```

`monitoring`, `argocd`, `external-secrets`, `karpenter` 네임스페이스의 Pod 장애는 플랫폼 전체 관측성 손실로 직결된다. `kube-system`만 제외하는 이유는 CoreDNS, kube-proxy 등 시스템 컴포넌트가 EKS 서비스 SLA로 관리되기 때문이다.

### EndpointSlowResponse 임계값 조정

```diff
- expr: probe_duration_seconds > 3
+ expr: probe_duration_seconds > 2
```

이커머스 업계 표준 SLA는 "첫 화면 로드 3초 이내"이며 이를 달성하려면 백엔드 API 응답은 2초 이내여야 한다. Blackbox Exporter의 `probe_duration_seconds`는 DNS + TCP connect + TLS handshake + HTTP response 전체 소요시간이므로 2초가 적절하다.

### ALB Controller 오류 임계값 역전 수정

```diff
# ALBControllerReconcileErrorsHigh (warning)
- expr: increase(controller_runtime_reconcile_errors_total[5m]) > 5
+ expr: increase(controller_runtime_reconcile_errors_total[5m]) > 3

# ALBControllerReconcileErrorsSustained (critical)
- expr: increase(controller_runtime_reconcile_errors_total[10m]) > 3
+ expr: increase(controller_runtime_reconcile_errors_total[10m]) > 10
```

> **기준 설정 근거**<br>
> `warning (> 3 in 5m)`: 분당 0.6회 오류. Ingress 변경 작업 중 일시적으로 발생 가능한 수준.<br>
> `critical (> 10 in 10m)`: 분당 1회 오류가 10분 지속. IAM 권한 오류, SCP 차단, TargetGroup 충돌 등 구조적 장애를 의미한다.
{: .prompt-info }

### ArgoCDAppMissing `for` 조정

```diff
- for: 5m
+ for: 10m
```

`argocd_app_info` 메트릭은 ArgoCD ApplicationController가 재시작하는 동안 잠시 사라진다. 정상 재시작 소요 시간은 통상 3~5분이므로 `for: 5m`에서는 재시작마다 false positive가 발생했다.

### 신규 알림 — Karpenter

#### KarpenterCloudProviderErrors

```yaml
alert: KarpenterCloudProviderErrors
expr: increase(karpenter_cloudprovider_errors_total[5m]) > 0
for: 5m
labels:
  severity: critical
  category: platform
annotations:
  summary: Karpenter가 EC2 노드 프로비저닝에 실패하고 있습니다
  description: |
    [Impact]
    Karpenter CloudProvider에서 EC2 API 오류가 발생했습니다.
    신규 노드 프로비저닝이 실패하여 Pending Pod가 해소되지 않을 수 있습니다.
    트래픽 급증 시 스케일 아웃이 불가능한 상태입니다.

    [Likely Cause]
    - EC2 용량 부족 (InsufficientInstanceCapacity)
    - IAM 권한 오류 (karpenter-apne2-ctrl-role)
    - AWS 서비스 쿼터 초과
    - 잘못된 EC2NodeClass / NodePool 설정

    [Recommended Action]
    1. kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter | grep -i error
    2. kubectl get nodeclaim -A
    3. kubectl get ec2nodeclass, nodepool
    4. AWS CloudTrail에서 EC2 RunInstances 오류 확인
```

> `increase([5m]) > 0` — Karpenter 오류는 단 1건만 발생해도 배치 프로비저닝 전체 실패를 의미할 수 있으므로 즉시 감지한다.
{: .prompt-info }

#### KarpenterNodePoolAtLimit

```yaml
alert: KarpenterNodePoolAtLimit
expr: |
  karpenter_nodepools_usage
    / karpenter_nodepools_limit > 0.90
for: 5m
labels:
  severity: warning
annotations:
  summary: Karpenter NodePool 리소스 한도에 근접했습니다
  description: |
    NodePool {{ $labels.nodepool }} 의 {{ $labels.resource }} 사용량이 한도의 90%를 초과했습니다.
    한도 도달 시 신규 노드를 프로비저닝할 수 없습니다.

    [Recommended Action]
    1. kubectl get nodepool -o yaml
    2. NodePool limits 상향 조정 검토
```

> `> 0.90` — 100%에서는 이미 신규 프로비저닝 불가 상태이므로 10% 버퍼를 두어 선제 대응한다.
{: .prompt-tip }

### 신규 알림 — Valkey (Redis)

Valkey 구성: StatefulSet `valkey-node` (3개 노드), `redis-exporter`로 Prometheus 메트릭 수집.

#### ValkeyDown

```yaml
alert: ValkeyDown
expr: redis_up == 0
for: 1m
labels:
  severity: critical
```

| 값 | 의미 |
|---|---|
| `1` | Exporter가 Redis에 정상 연결됨 |
| `0` | Exporter가 실행 중이나 Redis에 연결 불가 |
| `absent` | Exporter 자체가 다운됨 |

#### ValkeyMemoryHigh / ValkeyMemoryCritical

```yaml
# warning: 80% 초과 5분 지속
expr: (redis_memory_used_bytes / redis_memory_max_bytes > 0.80) and redis_memory_max_bytes > 0
for: 5m

# critical: 90% 초과 3분 지속
expr: (redis_memory_used_bytes / redis_memory_max_bytes > 0.90) and redis_memory_max_bytes > 0
for: 3m
```

> `and redis_memory_max_bytes > 0` — `maxmemory 0` (무제한) 설정 시 `0/0` NaN 방지.
{: .prompt-tip }

#### ValkeyConnectionsSaturated

```yaml
expr: redis_connected_clients / redis_config_maxclients > 0.80
for: 5m
labels:
  severity: warning
```

`redis_config_maxclients`는 `CONFIG GET maxclients` 값(기본 10,000). 80% 기준으로 연결 포화 전에 감지.

#### ValkeyEvictionRateHigh

```yaml
expr: increase(redis_evicted_keys_total[5m]) > 100
for: 5m
labels:
  severity: warning
```

5분 내 100건 이상 eviction은 캐시 히트율 저하로 DB 부하를 직접 유발한다.

#### ValkeyRejectedConnectionsHigh

```yaml
expr: increase(redis_rejected_connections_total[5m]) > 0
for: 1m
labels:
  severity: critical
```

연결 거부가 발생한다는 것은 이미 `maxclients` 한도에 도달한 상태다. 단 1건이라도 즉시 critical 처리한다.

### 전체 변경 요약

| 룰 | 변경 유형 | 변경 전 | 변경 후 |
|---|---|---|---|
| PodNotReady 등 4개 | namespace 필터 | `namespace=~"nkor-dv-.*"` | `namespace!~"kube-system"` |
| EndpointSlowResponse | 임계값 | `> 3s` | `> 2s` |
| ALBControllerReconcileErrorsHigh | 임계값 | `[5m] > 5` | `[5m] > 3` |
| ALBControllerReconcileErrorsSustained | 임계값 | `[10m] > 3` | `[10m] > 10` |
| ArgoCDAppMissing | for 기간 | `for: 5m` | `for: 10m` |
| KarpenterCloudProviderErrors | 신규 | — | `increase[5m] > 0`, critical |
| KarpenterNodePoolAtLimit | 신규 | — | `usage/limit > 0.90`, warning |
| ValkeyDown | 신규 | — | `redis_up == 0`, critical |
| ValkeyMemoryHigh | 신규 | — | `used/max > 0.80`, warning |
| ValkeyMemoryCritical | 신규 | — | `used/max > 0.90`, critical |
| ValkeyConnectionsSaturated | 신규 | — | `clients/maxclients > 0.80`, warning |
| ValkeyEvictionRateHigh | 신규 | — | `increase[5m] > 100`, warning |
| ValkeyRejectedConnectionsHigh | 신규 | — | `increase[5m] > 0`, critical |

### 적용 후 상태

```
ops groups: 8  (기존 7 → 8)
total rules: 34 (기존 26 → 34)

[ops-albc-alerts]                 2 rules  ✅
[ops-argocd-alerts]               3 rules  ✅
[ops-blackbox-alerts]             4 rules  ✅
[ops-kubernetes-workload-alerts]  6 rules  ✅
[ops-monitoring-stack-alerts]     2 rules  ✅
[ops-node-storage-alerts]         6 rules  ✅
[ops-scaling-eso-alerts]          5 rules  ✅  ← Karpenter 2개 추가
[ops-valkey-alerts]               6 rules  ✅  ← 신규 그룹
```

Alertmanager webhook URL로 synthetic alert 2개를 주입해 라우팅 검증:
- `[TEST] ValkeyDown` (critical) → `#critical-alerts` ✅
- `[TEST] KarpenterNodePoolAtLimit` (warning) → `#warning-alerts` ✅

---

## 12. ARC Runner PrometheusRule — CI/CD 파이프라인 장애 감지

### 도입 배경

ARC 0.10.1 업그레이드 장애 이후 **runner가 조용히 죽어있어도 다음 CI 실행 전까지 감지가 불가능**한 문제가 드러났다. runner가 중단된 상태에서 GitHub Actions job이 트리거되면 무한 대기에 빠지고, 개발팀은 한참 뒤에야 인지하게 된다.

3개의 PrometheusRule을 추가하여 runner 이상 상태를 사전에 Slack으로 알린다.

### 알람 설계 원칙

| 원칙 | 내용 |
|---|---|
| 빠른 감지 | `interval: 30s` — Prometheus 기본 1분보다 빠르게 평가 |
| False positive 억제 | `for:` 조건으로 일시적 상태 변화 필터링 |
| 조치 즉시 확인 | `description`에 kubectl 명령 직접 기술 |
| Alertmanager 라우팅 연동 | `severity: critical/warning` → 기존 Slack 채널 자동 라우팅 |

### PrometheusRule 전체 구성

```yaml
# kubernetes/monitoring/arc-runner-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: arc-runner-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack   # Prometheus Operator 감지 필수 레이블
spec:
  groups:
    - name: arc-runner
      interval: 30s
      rules:
        - alert: ARCRunnerTooManyFailures
          ...
        - alert: ARCRunnerPodPending
          ...
        - alert: ARCListenerDown
          ...
```

> **`release: kube-prometheus-stack` 레이블이 없으면 Prometheus Operator가 이 PrometheusRule을 무시한다.**
{: .prompt-danger }

### 알람 1: ARCRunnerTooManyFailures

**증상**: `arc-runners` 네임스페이스에 Failed 상태 runner pod가 3개 이상 누적

**원인**: EphemeralRunner deadlock — Failed runner가 `currentReplicas`에 포함되어 controller가 새 runner를 생성하지 못하는 상태

```yaml
alert: ARCRunnerTooManyFailures
expr: |
  count(
    kube_pod_status_phase{namespace="arc-runners", phase="Failed"}
    * on(pod) group_left()
    kube_pod_labels{namespace="arc-runners", label_app_kubernetes_io_component="runner"}
  ) > 2
for: 2m
labels:
  severity: warning
  team: platform
annotations:
  summary: "ARC runner pod 실패 누적 ({{ $value }}개)"
  description: |
    조치: kubectl delete ephemeralrunner -n arc-runners --all
```

| 항목 | 값 | 이유 |
|---|---|---|
| `for: 2m` | 2분 | 일시적 pod restart와 구분 |
| `> 2` | 2개 초과 | 1~2개 Failed는 정상 범위, 3개부터 deadlock 위험 |
| `severity: warning` | warning | 즉시 중단은 아니지만 다음 job 실패 예고 |

**조치:**
```bash
kubectl delete ephemeralrunner -n arc-runners --all
# CronJob이 5분마다 자동 정리 (kubernetes/arc-runners/ephemeralrunner-cleanup.yaml)
```

### 알람 2: ARCRunnerPodPending

**증상**: `arc-runners` 네임스페이스에 Pending 상태 runner pod가 5분 이상 지속

**원인**: system 노드 용량 부족, taint/toleration 불일치, ECR 이미지 pull 실패

```yaml
alert: ARCRunnerPodPending
expr: |
  count(
    kube_pod_status_phase{namespace="arc-runners", phase="Pending"}
    * on(pod) group_left()
    kube_pod_labels{namespace="arc-runners", label_app_kubernetes_io_component="runner"}
  ) > 0
for: 5m
labels:
  severity: warning
```

| 항목 | 값 | 이유 |
|---|---|---|
| `for: 5m` | 5분 | Karpenter 노드 프로비저닝 시간(~2분) 감안 |

**조치:**
```bash
kubectl describe pod -n arc-runners <pod-name>
kubectl describe node -l role=system | grep -A 5 "Allocatable"
```

### 알람 3: ARCListenerDown (Critical)

**증상**: `arc-system`에 listener pod가 3분 이상 부재

**원인**: ARC listener는 GitHub 브로커와 항상 연결을 유지하며 job을 감지한다. listener가 없으면 GitHub job이 트리거되어도 runner가 전혀 생성되지 않아 **CI 파이프라인 전체가 무한 대기** 상태가 된다.

```yaml
alert: ARCListenerDown
expr: |
  absent(
    kube_pod_status_phase{
      namespace="arc-system",
      pod=~"<prefix>-runner-.*-listener",
      phase="Running"
    }
  )
for: 3m
labels:
  severity: critical
  team: platform
annotations:
  summary: "ARC listener pod 없음 — runner 전체 중단"
  description: |
    조치:
    kubectl get pods -n arc-system
    kubectl describe autoscalingrunnerset -n arc-runners
    kubectl delete autoscalingrunnerset -n arc-runners <prefix>-runner
    # ArgoCD selfHeal이 즉시 재생성
```

| 항목 | 값 | 이유 |
|---|---|---|
| `for: 3m` | 3분 | ARC controller가 listener pod 재시작까지 ~60초 소요 |
| `severity: critical` | critical | listener 부재 = CI 파이프라인 완전 중단 |

### 트러블슈팅: kube_pod_labels 레이블 미export 문제

**문제**: 초기 `ARCListenerDown` 표현식에서 pod 커스텀 레이블로 필터를 시도했으나 항상 firing 상태가 됐다.

```yaml
# 초기 버전 (오작동)
kube_pod_labels{
  namespace="arc-system",
  label_app_kubernetes_io_component="runner-scale-set-listener"
}
```

**원인**: kube-state-metrics는 기본 설정에서 pod의 모든 레이블을 `kube_pod_labels` 메트릭으로 export하지 않는다. `metricLabelsAllowList`에 명시된 레이블만 export한다.

```bash
# 실제 export 확인
curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=kube_pod_labels{namespace="arc-system", label_app_kubernetes_io_component="runner-scale-set-listener"}'
# → results: 0  (레이블 미export)
```

**해결**: label join 제거, pod name regex로 직접 필터링.

```yaml
# 수정 후 (정상 동작)
kube_pod_status_phase{
  namespace="arc-system",
  pod=~"<prefix>-runner-.*-listener",
  phase="Running"
}
```

> `kube_pod_status_phase`의 `pod` 레이블은 항상 export된다. pod name 패턴 `{prefix}-runner-{hash}-listener`는 ARC가 고정 형식으로 생성하므로 regex 매칭이 안정적이다.
{: .prompt-tip }

### Alertmanager 라우팅

기존 Alertmanager 라우팅 설정이 `severity` 레이블 기준으로 동작하므로 추가 설정 없이 자동 라우팅된다.

```
ARCListenerDown (critical)
  → slack-critical receiver
  → group_wait: 10s, repeat_interval: 1h

ARCRunnerTooManyFailures / ARCRunnerPodPending (warning)
  → slack-warning receiver (default route)
  → group_wait: 30s, repeat_interval: 4h
```

### 동작 검증

```bash
# 1. listener pod 강제 삭제
kubectl delete pod -n arc-system <prefix>-runner-754b578d-listener

# 2. Prometheus 상태 확인 (30초 후)
# → ARCListenerDown: state=firing ✅

# 3. ARC controller가 listener 자동 재시작 (57초 소요)

# 4. 복구 후 상태 확인
# → ARCListenerDown: state=inactive ✅
```

**최종 상태:**

| 알람 | 상태 |
|---|---|
| ARCRunnerTooManyFailures | inactive (Failed runner 없음) |
| ARCRunnerPodPending | inactive (Pending runner 없음) |
| ARCListenerDown | inactive (listener 정상 Running) |

---

## 변경 파일 요약

| 파일 | 변경 내용 |
|---|---|
| `helm/alloy/values.yaml` | WAL 영속화 (hostPath), 헬스체크 필터 전역화, nkor JSON 필드 추가 |
| `helm/monitoring/kube-prometheus-stack/values.yaml` | `disable_sanitize_html`, Node Exporter OS 비활성화, 알림 규칙 개선 |
| `kubernetes/monitoring/grafana-dashboard-ingress-alb.yaml` | 5개 패널 controller 필터에 `targetGroupBinding` 추가 |
| `kubernetes/monitoring/grafana-dashboard-overview.yaml` | PVC 쿼리 `== 1` 필터 + `or vector(0)` 추가 |
| `kubernetes/monitoring/grafana-dashboard-executive.yaml` | 신규 생성 — 커스텀 Executive 단일 화면 대시보드 |
| `kubernetes/monitoring/loki-k8s-logs-overview-dashboard.yaml` | namespace 전체 공개, app 변수 추가, HTTP 5xx/Error Rate 패널 추가 |
| `kubernetes/monitoring/arc-runner-alerts.yaml` | 신규 생성 — ARC Runner PrometheusRule 3개 |
