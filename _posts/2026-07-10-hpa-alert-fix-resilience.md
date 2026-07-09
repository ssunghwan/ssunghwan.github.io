---
title: "Enhancing EKS Workload Resilience and Analyzing Incidents Skewed Toward Specific Availability Zones"
date: 2026-07-09 09:00:00 +0900
categories: [Kubernetes, "Legacy PHP eCommerce - EKS Migration"]
tags: [eks, hpa, karpenter, pdb, graceful-shutdown, topologyspreadconstraints, consolidation, az, kubernetes]
---

> 이 포스팅은 이커머스 EKS 환경에서 워크로드 복원력을 높이기 위한 설계 가이드와, 실제 운영 중 발생한 **AZ 편중 사고 분석**을 함께 다룬다.<br>
> HPA 설계, PDB, Graceful Shutdown, AZ 분산 전략부터, Karpenter Consolidation과 `topologySpreadConstraints`의 소프트 제약이 맞물려 발생한 구조적 사고까지 실제 경험을 기반으로 정리했다.
{: .prompt-info }

> **관련 파일**<br>
> `kubernetes/apps/HorizontalPodAutoscaler/hpa-api.yaml`<br>
> `kubernetes/apps/HorizontalPodAutoscaler/hpa-nextjs.yaml`<br>
> `kubernetes/apps/HorizontalPodAutoscaler/pdb-api.yaml` _(신규)_<br>
> `kubernetes/apps/HorizontalPodAutoscaler/pdb-nextjs.yaml` _(신규)_<br>
> `kubernetes/apps/deployment/deployment-api.yaml`<br>
> `kubernetes/apps/deployment/deployment-nextjs.yaml`<br>
> `kubernetes/karpenter/nodepool-api-apne2*.yaml`<br>
> `kubernetes/karpenter/nodepool-web-apne2*.yaml`
{: .prompt-tip }

## 1. HPA 설계 가이드

### HPA란?

HPA(HorizontalPodAutoscaler)는 CPU/Memory 사용률 등 메트릭 기준으로 Deployment의 replicas를 자동으로 조절하는 Kubernetes 리소스다. Karpenter와 함께 동작하면 Pod 수 증가 → 노드 부족 → Karpenter가 노드 자동 프로비저닝까지 완전 자동화된다.

```
트래픽 증가
  → HPA: replicas 2 → 4 → 6
  → 기존 노드 리소스 부족
  → Karpenter: 새 노드 자동 프로비저닝
  → topologySpreadConstraints: AZ 분산 배포
```

### ScalingLimited 조건 이해

HPA를 운영하다 보면 `kubectl describe hpa`에서 아래와 같은 조건을 자주 보게 된다.

```
ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
```

`ScalingLimited` 조건은 두 가지 이유로 `True`가 될 수 있다.

| Reason | 의미 | 알림 대상 |
|---|---|---|
| `TooManyReplicas` | maxReplicas에 도달해서 더 이상 스케일 업 불가 | ✅ 진짜 알림 |
| `TooFewReplicas` | 계산된 replicas가 minReplicas보다 낮아서 하한선에 걸림 | ❌ 정상 동작 |

> **`TooFewReplicas`는 정상이다.**<br>
> CPU 사용률이 매우 낮으면 HPA가 minReplicas 미만으로 줄이려 하지만, `minReplicas` 설정으로 인해 제한된 상태다.<br>
> 스케일 다운이 막힌 것으로, 이커머스에서 AZ 분산을 위해 `minReplicas: 2`를 설정하면 평상시에는 항상 이 상태가 된다.
{: .prompt-info }

**현재 HPA 상태 확인:**

```bash
kubectl get hpa -n <app-namespace>

NAME                   TARGETS                        MINPODS  MAXPODS  REPLICAS
<prefix>-api-hpa       cpu: 1%/60%, memory: 20%/70%   2        10       2
<prefix>-nextjs-hpa    cpu: 1%/60%, memory: 18%/70%   2        10       2
```

### HPA Alert Expression 올바른 작성법

`ScalingLimited` 조건 기반 알림은 `TooFewReplicas`까지 잡아 오탐이 발생한다. **메트릭을 직접 비교하는 방식**을 사용해야 한다.

```yaml
# ❌ ScalingLimited=true 이면 이유 불문하고 발화 → 오탐 발생
kube_horizontalpodautoscaler_status_condition{
  condition="ScalingLimited",
  status="true",
  namespace=~"<app-namespace-prefix>.*"
} == 1

# ✅ 실제 replicas가 maxReplicas 이상일 때만 발화 → 정확한 알림
kube_horizontalpodautoscaler_status_current_replicas{namespace=~"<app-namespace-prefix>.*"}
  >= kube_horizontalpodautoscaler_spec_max_replicas{namespace=~"<app-namespace-prefix>.*"}
```

> 메트릭을 직접 비교하는 방식은 HPA 내부 상태(condition)가 아닌 실제 수치를 보기 때문에 훨씬 명확하고 오탐이 없다.
{: .prompt-tip }

### minReplicas와 AZ 분산 전략

`minReplicas: 2`는 단순히 최소 파드 수가 아니라 **AZ 분산 보장**의 의미를 갖는다. `topologySpreadConstraints`와 함께 설정해야 의도한 대로 동작한다.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway   # AZ 분산 권장, 불가능하면 허용
  labelSelector:
    matchLabels:
      app: <prefix>-api
```

### 현재 HPA 설정

**API 서비스**

```yaml
minReplicas: 2 / maxReplicas: 10
CPU target:    60%  (request 100m 기준 → 60m 초과 시 스케일 업)
Memory target: 70%  (request 128Mi 기준 → 90Mi 초과 시 스케일 업)
scaleUp:   stabilization 60s,  최대 2개/60s
scaleDown: stabilization 300s, 최대 1개/120s
```

**NextJS 서비스**

```yaml
minReplicas: 2 / maxReplicas: 10
CPU target:    60%  (request 200m 기준 → 120m 초과 시 스케일 업)
Memory target: 70%  (request 256Mi 기준 → 179Mi 초과 시 스케일 업)
scaleUp:   stabilization 60s,  최대 2개/60s
scaleDown: stabilization 300s, 최대 1개/120s
```

**Karpenter NodePool (AZ별)**

```
api-apne2a / api-apne2c: cpu limit 4, memory 16Gi
web-apne2a / web-apne2c: cpu limit 4, memory 16Gi
consolidateAfter: 30s
```

---

## 2. PodDisruptionBudget (PDB) 설계 가이드

### PDB란?

PDB(PodDisruptionBudget)는 자발적 중단(Voluntary Disruption) 시 동시에 종료될 수 있는 파드 수를 제한하는 정책이다.

> **자발적 중단(Voluntary Disruption)** 이란 노드 업그레이드, Karpenter consolidation, `kubectl drain` 등 의도된 중단을 의미한다.<br>
> OOM Kill, 노드 장애 같은 **비자발적 중단(Involuntary Disruption)** 은 PDB로 제어할 수 없다.
{: .prompt-info }

PDB가 없으면 Karpenter가 노드를 정리할 때 해당 노드의 파드를 한 번에 모두 종료할 수 있다. 2개 레플리카가 같은 노드에 있으면 순간적으로 서비스 중단이 발생한다.

### PDB 설정

```yaml
# pdb-api.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: <prefix>-api-apne2-pdb
  namespace: <app-namespace>
spec:
  minAvailable: 1    # 최소 1개는 항상 Running 상태 유지
  selector:
    matchLabels:
      app: <prefix>-api
```

적용 결과 확인:

```bash
kubectl get pdb -n <app-namespace>

NAME                   MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
<prefix>-api-pdb       1               N/A               1
<prefix>-nextjs-pdb    1               N/A               1
```

> **`ALLOWED DISRUPTIONS: 1`의 의미**<br>
> 현재 replicas가 2개이고 `minAvailable: 1`이므로, 동시에 중단 허용 가능한 파드는 1개다.<br>
> Karpenter가 노드를 consolidation 할 때 이 PDB를 존중하므로 한 번에 1개씩만 종료한다.
{: .prompt-tip }

### minAvailable 절대값 vs 비율

| 방식 | 설정 | 동작 |
|---|---|---|
| 절대값 | `minAvailable: 1` | replicas가 몇이든 항상 최소 1개 유지 |
| 비율 | `minAvailable: "50%"` | replicas의 50% 이상 항상 유지 |

> 개발계처럼 replicas가 고정적일 때는 절대값이 단순하다.<br>
> 운영계처럼 HPA로 replicas가 동적으로 변할 때는 비율이 더 안전하다.<br>
> replicas가 10개일 때 `minAvailable: 1`이면 9개가 동시에 종료될 수 있다.
{: .prompt-warning }

---

## 3. Graceful Shutdown 설계 가이드

### PreStop Hook

파드 종료 시 로드밸런서(ALB)가 엔드포인트 제거를 완료하기 전에 파드가 먼저 죽는 문제를 방지한다.

> **왜 필요한가?**<br>
> 쿠버네티스는 `SIGTERM` 전송과 동시에 엔드포인트 제거를 시작하지만, ALB가 이를 전파받는 데 수초가 소요된다.<br>
> 그 사이에 파드가 종료되면 ALB가 이미 죽은 파드로 요청을 보내 **502 에러**가 발생한다.<br>
> `preStop sleep 5s`로 ALB 연결 드레이닝이 완료될 시간을 확보한다.
{: .prompt-warning }

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

**종료 흐름:**

```
파드 종료 신호
  → preStop sleep 5s (ALB 연결 드레이닝 대기)
  → SIGTERM 전송 (앱에 종료 신호)
  → terminationGracePeriodSeconds: 30 내 처리 완료
  → SIGKILL (초과 시 강제 종료)
```

> **preStop 시간은 `terminationGracePeriodSeconds`에 포함된다.**<br>
> 현재 30초 중 5초를 preStop이 사용하므로, 실제 앱이 처리할 수 있는 시간은 최대 25초다.<br>
> 처리 시간이 긴 요청(주문, 결제)이 있다면 `terminationGracePeriodSeconds`를 충분히 늘려야 한다.
{: .prompt-danger }

---

## 4. AZ 분산 스케일링 동작 방식

트래픽 증가 시 HPA가 레플리카를 늘리면 `topologySpreadConstraints`(maxSkew: 1)에 의해 스케줄러가 AZ 균형을 맞춘다.

```
평상시:    2a:1  2c:1  (minReplicas 유지)
스케일 업: 2a:2  2c:1  → 다음 파드 → 2a:2  2c:2  (균형 복원)
           2a:2  2c:2  → 다음 파드 → 2a:3  2c:2  → ...
```

> `whenUnsatisfiable: ScheduleAnyway`이므로 한쪽 AZ 노드가 없으면 반대편에 임시 배치 가능하다. (소프트 제약)
{: .prompt-info }

---

## 5. 운영환경 전환 시 추가 적용 권고 사항

개발계에서는 트래픽이 낮아 체감 영향이 적지만, 운영 투입 전 반드시 검토해야 할 항목들이다.

### CPU/Memory Request 현실화 (우선순위: 높음)

HPA는 `실제 사용량 / request` 비율로 레플리카를 계산한다. Request가 낮으면 utilization %가 실제보다 높게 계산되어 불필요한 스케일 업이 발생한다.

```yaml
# 운영 권고값 (부하 테스트 결과 기반으로 조정 필요)
# API
resources:
  requests:
    cpu: 250m      # 100m → 250m
    memory: 256Mi  # 128Mi → 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

# NextJS (SSR)
resources:
  requests:
    cpu: 300m      # 200m → 300m
    memory: 384Mi  # 256Mi → 384Mi
  limits:
    cpu: 1000m
    memory: 768Mi
```

> VPA(Vertical Pod Autoscaler) recommendation 모드로 실제 사용량을 2주 이상 수집한 후 값을 결정하는 것을 권장한다.
{: .prompt-tip }

### scaleUp Policy 완화 (우선순위: 높음)

이커머스 특성상 타임딜, 플래시 세일 등 순간 트래픽 폭증이 발생할 수 있다. 현재 60초마다 2개씩 증가는 급격한 스파이크 대응에 부족하다.

```yaml
# 운영 권고
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0  # 스파이크 즉시 대응
    policies:
    - type: Percent
      value: 100       # 현재 replicas의 100%까지 한 번에 증가 가능
      periodSeconds: 60
    - type: Pods
      value: 4
      periodSeconds: 60
    selectPolicy: Max  # 두 policy 중 더 큰 값 선택
  scaleDown:
    stabilizationWindowSeconds: 600  # 10분 안정화 (섣부른 축소 방지)
    policies:
    - type: Pods
      value: 1
      periodSeconds: 180
```

### maxReplicas 및 Karpenter limit 상향 (우선순위: 높음)

```yaml
# HPA
maxReplicas: 20  # 10 → 20

# Karpenter NodePool
limits:
  cpu: 16      # 4 → 16
  memory: 64Gi # 16Gi → 64Gi
```

### PDB minAvailable 비율로 조정 (우선순위: 중간)

운영에서 replicas가 늘어날수록 minAvailable을 비율로 지정하는 것이 더 안전하다.

```yaml
spec:
  minAvailable: "50%"  # 절대값 1 → 비율 50%
  # replicas가 10이면 최소 5개 유지 보장
```

### PreStop 시간 및 terminationGracePeriod 상향 (우선순위: 중간)

운영에서는 처리 시간이 긴 요청(주문, 결제)이 존재하므로 종료 유예 시간을 늘려야 한다.

```yaml
terminationGracePeriodSeconds: 60  # 30 → 60

lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]  # 5s → 15s
```

### Karpenter consolidateAfter 조정 (우선순위: 중간)

개발계 30초는 비용 절감에 유리하지만, 운영에서는 트래픽이 짧은 주기로 오르내릴 경우 노드 재프로비저닝(1~3분) 비용이 더 클 수 있다.

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 5m  # 30s → 5m
```

### topologySpreadConstraints 하드 제약 검토 (우선순위: 중간)

현재 `whenUnsatisfiable: ScheduleAnyway`는 AZ 분산을 보장하지 않는다. 한 AZ에 장애 발생 시 모든 파드가 단일 AZ에 몰릴 수 있다.

```yaml
# 운영 권고 — AZ 분산 보장
whenUnsatisfiable: DoNotSchedule
# 단, Karpenter가 대상 AZ에 노드를 프로비저닝하지 못하면 Pending 발생.
# Karpenter NodePool이 해당 AZ를 커버하는지 반드시 확인 필요.
```

### 커스텀 메트릭 기반 HPA 검토 (우선순위: 낮음 — 장기)

CPU/Memory는 요청 처리 후 지연이 있어 이커머스 스파이크 대응이 느리다. KEDA를 통해 RPS 기반 스케일링 도입을 검토할 수 있다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-scaledobject
spec:
  scaleTargetRef:
    name: <prefix>-api-apne2-deploy
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring:9090
      metricName: http_requests_per_second
      query: sum(rate(nginx_ingress_controller_requests{namespace="<app-namespace>"}[1m]))
      threshold: "100"  # 초당 100 req 초과 시 스케일 업
```

### Scheduled Scaling 검토 (우선순위: 낮음 — 장기)

이커머스 트래픽 패턴이 예측 가능한 경우(점심, 저녁 피크) 사전에 minReplicas를 높이는 전략이다.

```yaml
triggers:
- type: cron
  metadata:
    timezone: Asia/Seoul
    start: "0 11 * * 1-5"   # 평일 11시 — 점심 피크 대비
    end:   "0 14 * * 1-5"
    desiredReplicas: "6"
- type: cron
  metadata:
    timezone: Asia/Seoul
    start: "0 18 * * 1-5"   # 평일 18시 — 저녁 피크 대비
    end:   "0 21 * * 1-5"
    desiredReplicas: "6"
```

---

## 6. 운영환경 전환 체크리스트

| 항목 | 우선순위 | 상태 |
|---|:---:|:---:|
| 부하 테스트 수행 후 CPU/Memory request 값 결정 | 높음 | ⬜ |
| maxReplicas 운영 트래픽 기준으로 상향 | 높음 | ⬜ |
| Karpenter NodePool limit 상향 | 높음 | ⬜ |
| scaleUp policy → Percent 방식으로 변경 | 높음 | ⬜ |
| scaleDown stabilizationWindowSeconds → 600s 이상 | 높음 | ⬜ |
| PDB minAvailable → 50% 비율로 변경 | 중간 | ⬜ |
| terminationGracePeriodSeconds → 60s 이상 | 중간 | ⬜ |
| preStop sleep → 15s 이상 | 중간 | ⬜ |
| topologySpreadConstraints whenUnsatisfiable → DoNotSchedule 검토 | 중간 | ⬜ |
| Karpenter consolidateAfter → 5m 이상 | 중간 | ⬜ |
| KEDA 도입 검토 (RPS 기반 스케일링) | 낮음 | ⬜ |
| Scheduled Scaling 검토 (트래픽 패턴 확인 후) | 낮음 | ⬜ |

---

---

## 7. AZ 편중 사고 분석 — Karpenter Consolidation과 TopologySpreadConstraints의 상호작용

> Stakater Reloader가 RDS 자격증명 로테이션을 감지해 정상적으로 자동 롤링 재시작을 수행했는데, 재시작 이후 API 파드 2개가 서로 다른 노드에 있음에도 **같은 AZ**에 몰려버리는 현상이 발생했다.<br>
> Karpenter Consolidation과 `topologySpreadConstraints`의 소프트 제약이 맞물려 발생한 구조적 문제를 분석하고 개선한 과정을 다룬다.
{: .prompt-info }

> **관련 포스팅**: [Stakater Reloader 도입 — Secret/ConfigMap 변경 시 파드 자동 롤링 재시작](/posts/stakater-reloader-secret-auto-restart/) — 이번 사고를 촉발한 재시작의 배경
{: .prompt-tip }


### 사고 요약

| 항목 | 내용 |
|---|---|
| 증상 | `<prefix>-api-apne2-deploy` 파드 2개가 서로 다른 파드 IP를 갖지만, 두 파드 모두 같은 물리 노드(`ap-northeast-2a`)에서 실행 중 — 실질적으로 **AZ 분산이 전혀 안 되고 있었음** |
| 대조군 | 같은 네임스페이스의 `nextjs` Deployment는 `ap-northeast-2a`/`ap-northeast-2c`에 정확히 1개씩 분산 — 두 Deployment의 YAML 설정은 **구조적으로 완전히 동일** |
| 근본 원인 | ① `whenUnsatisfiable: ScheduleAnyway`(소프트 제약)로 스케줄러가 같은 노드 배치를 허용 → ② 반대편 AZ(`ap-northeast-2c`)의 api 전용 노드가 파드 0개(empty)가 됨 → ③ Karpenter의 `WhenEmptyOrUnderutilized` + `consolidateAfter: 30s`가 그 빈 노드를 삭제 |
| 트리거 | Stakater Reloader가 `mysql-secret`(RDS 자격증명 로테이션) 감지해 수행한 정상적인 자동 롤링 재시작 — **재시작 자체는 의도된 정상 동작** |
| 결정적 증거 | Karpenter 컨트롤러 로그에서 `"reason":"empty","decision":"delete"`로 `api-apne2c-nodepool`의 노드를 삭제한 기록 확인 |
| 조치 완료 | `whenUnsatisfiable: DoNotSchedule` + `minDomains: 2`, `consolidateAfter: 30s → 2m` (2026-07-08 dev 반영) |

---

### 발견 경위

사용자가 아래 결과를 보고 의문을 제기했다.

```bash
kubectl get pods -n <app-namespace> -o wide

NAME                                    READY  STATUS   NODE
<prefix>-api-apne2-deploy-xxx-rnwvc     1/1    Running  ip-172-16-1-196...  # AZ-a
<prefix>-api-apne2-deploy-xxx-vw6h9     1/1    Running  ip-172-16-1-196...  # AZ-a ← 같은 노드!
<prefix>-nextjs-apne2-deploy-xxx-krfgm  1/1    Running  ip-172-16-2-84...   # AZ-c
<prefix>-nextjs-apne2-deploy-xxx-nqhc8  1/1    Running  ip-172-16-1-115...  # AZ-a
```

`api` 파드 2개가 **완전히 동일한 노드**에서 실행 중이었다. `nextjs`는 서로 다른 노드에 정확히 분산돼 있는데, 왜 `api`만 쏠렸는가가 조사의 출발점이었다.

---

### 조사 과정

**Step 1 — Deployment 스펙 확인**

```yaml
# api Deployment
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway   # ← 소프트 제약
  labelSelector:
    matchLabels:
      app: <prefix>-api
```

`nextjs` Deployment도 동일하게 `ScheduleAnyway`였다. 즉 **YAML 차이가 원인이 아님** 확인.

**Step 2 — 노드/AZ 전수 조사**

```bash
kubectl get nodes -o wide --show-labels

ip-172-16-1-115...  ap-northeast-2a  role=web
ip-172-16-1-153...  ap-northeast-2a  role=system
ip-172-16-1-196...  ap-northeast-2a  role=api   ← api 노드가 이것뿐
ip-172-16-2-208...  ap-northeast-2c  role=system
ip-172-16-2-84...   ap-northeast-2c  role=web
```

결정적 단서: **`role=api` 노드가 클러스터 전체에 `ap-northeast-2a`에 딱 1대뿐**이었다. `nodeSelector: role: api`로 스케줄링이 제한되는 이상, 노드가 1대뿐이면 두 replica가 물리적으로 같은 노드에 갈 수밖에 없다.

**Step 3 — Karpenter NodePool 확인**

```bash
# api NodePool 상태
api-apne2a-nodepool  nodes=1   # AZ-a 노드 1대 존재
api-apne2c-nodepool  nodes=0   # AZ-c 노드 없음 ← 문제
web-apne2a-nodepool  nodes=1
web-apne2c-nodepool  nodes=1
```

api용 NodePool은 `apne2a`/`apne2c` 둘 다 정의되어 있었다. 설정은 대칭인데, 실제 뜬 노드만 비대칭 — "무슨 일이 있어서 2c 노드가 사라졌다"는 히스토리 문제로 조사 방향을 전환했다.

**Step 4 — Karpenter 컨트롤러 로그 (결정적 증거)**

```bash
kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter \
  --since=200h | grep -i "api-apne2c"
```

```json
{
  "time": "2026-07-06T06:21:49Z",
  "message": "disrupting node(s)",
  "controller": "disruption",
  "reason": "empty",
  "decision": "delete",
  "nodes": ["ip-172-16-2-135 (api-apne2c-nodepool)"]
}
```

**`reason: "empty"`** — Reloader가 api 파드를 재시작하는 과정에서 롤링 업데이트 특성상 잠시 AZ-c 노드가 비었고, Karpenter의 `consolidateAfter: 30s`가 30초 만에 그 빈 노드를 삭제해버렸다. 그 이후 새 파드들은 AZ-a 노드 1대에 몰릴 수밖에 없었다.

---

### 근본 원인 상세 — 두 시스템의 정상 동작이 맞물린 결과

```
[Reloader] mysql-secret 변경 감지
  → api Deployment 롤링 재시작 트리거
        ↓
[K8s RollingUpdate] 순서:
  1. 새 파드(pod-A) 생성 요청 → 스케줄러: AZ-a 노드에 배치 (ScheduleAnyway이므로 허용)
  2. 기존 파드(pod-B, AZ-c) 종료
  3. 새 파드(pod-C) 생성 요청 → 스케줄러: AZ-c 노드가 비어있고, ScheduleAnyway이므로 AZ-a에도 허용
        ↓
[AZ-c api 노드] 파드 0개 = Empty 상태
        ↓
[Karpenter Consolidation] consolidateAfter: 30s → 30초 뒤 삭제
  "reason": "empty", "decision": "delete"
        ↓
[결과] api 파드 2개 모두 AZ-a에, AZ-c 노드 없음
       다음 재시작까지 이 상태 고정
```

> **`ScheduleAnyway`와 `WhenEmptyOrUnderutilized`는 각각 합리적인 기본값이지만, 같이 있으면 "한 번의 쏠림이 영구화"되는 조합이 된다.**<br>
> `nextjs`가 현재 잘 분산돼 있어 보이는 건 "구조적으로 보장"된 게 아니라 "다음 재시작 타이밍에 운이 좋았을 뿐"이다.
{: .prompt-danger }

---

### Reloader 재시작 자체는 문제가 아니다 — 책임 소재 명확화

이 사고를 보고 "Reloader 도입이 잘못됐다"고 해석하면 안 된다.

```
[정상 동작] Reloader → 롤링 재시작 (의도된 동작)
[정상 동작] Karpenter → 빈 노드 삭제 (의도된 동작)
[문제]      ScheduleAnyway → 쏠림을 막지 않음 (소프트 제약의 한계)
```

Reloader가 없었다면 사고가 안 났을까? 아니다. 다음 중 어떤 이벤트에서든 동일하게 재현된다.

| 이벤트 | 발생 주기 |
|---|---|
| RDS 자격증명 로테이션 | 매주 자동 |
| 배포(CI/CD) | PR merge 시마다 |
| Karpenter 노드 만료(`expireAfter: 720h`) | 30일마다 |
| AMI 업데이트(`al2023@latest`) | AWS AMI 릴리즈 시마다 |

Reloader는 이 이벤트들 중 하나를 오늘 트리거했을 뿐이다. 근본 원인은 `ScheduleAnyway` + Karpenter 공격적 통합 정책의 조합이다.

---

### 관련 YAML 필드 전수 해설

#### topologySpreadConstraints — ScheduleAnyway vs DoNotSchedule

| 값 | 동작 | 결과 |
|---|---|---|
| `ScheduleAnyway` (소프트 제약) | AZ 분산이 불가능해도 어딘가에 배치 | 쏠림 허용, 가용성 미보장 |
| `DoNotSchedule` (하드 제약) | AZ 분산이 불가능하면 Pending 유지 | 쏠림 불허, 일시적 Pending 가능 |

`minDomains` 필드와의 조합:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  minDomains: 2         # 최소 2개 AZ에 노드가 있어야 함을 스케줄러에 명시
  labelSelector:
    matchLabels:
      app: <prefix>-api
```

> **`minDomains: 2`를 추가하는 이유**<br>
> `DoNotSchedule`만 설정하면 스케줄러는 "현재 눈에 보이는 노드"를 기준으로 판단한다.<br>
> AZ-c 노드가 이미 삭제된 상태라면 "AZ-a에 1개 있으니 maxSkew=1 만족" → 허용해버린다.<br>
> `minDomains: 2`는 "가용한 도메인(AZ)이 최소 2개 미만이면 아예 스케줄 불가"를 추가로 강제해, Karpenter가 반대편 AZ에 노드를 새로 띄울 때까지 Pending으로 대기시킨다.
{: .prompt-info }

**`maxSkew`의 의미:**

```
maxSkew: 1 일 때:
  AZ-a: 1개, AZ-c: 1개 → skew=0 ✅
  AZ-a: 2개, AZ-c: 1개 → skew=1 ✅ (maxSkew 이하)
  AZ-a: 2개, AZ-c: 0개 → skew=2 ❌ → DoNotSchedule이면 Pending, ScheduleAnyway면 허용
```

---

#### Karpenter NodePool — 주요 필드 상세

현재 api/web NodePool의 전체 구성과 각 필드의 의미를 정리한다.

```yaml
# kubernetes/karpenter/nodepool-api-apne2a.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [t, m, c]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: [medium, large, xlarge]
        - key: topology.kubernetes.io/zone
          operator: In
          values: [ap-northeast-2a]
      taints:
        - key: dedicated
          value: api
          effect: NoSchedule
      expireAfter: 720h
  limits:
    cpu: 4
    memory: 16Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 2m   # 30s → 2m (2026-07-08 수정)
```

**필드별 표준 관점:**

| 필드 | 현재 값 | 이커머스 표준 관점 |
|---|---|---|
| `karpenter.sh/capacity-type` | `on-demand` | 결제/API처럼 갑작스러운 파드 종료가 장애로 이어지는 워크로드는 on-demand가 표준. 상태 없는 배치성 워크로드는 spot 혼용도 흔함 |
| `instance-category` | `t, m, c` | t(버스터블)/m(범용)/c(컴퓨트 최적화)를 허용해 Karpenter의 최적 인스턴스 선택 여지를 줌. 너무 좁으면 가용 용량 부족 위험 |
| `instance-size` | `medium, large, xlarge` | 합리적인 범위. small은 파드 스케줄링 밀도가 낮아 비효율, 2xlarge 이상은 단일 노드 장애 영향이 커짐 |
| `topology.kubernetes.io/zone` | `ap-northeast-2a` (AZ별 NodePool 분리) | **AZ별로 NodePool을 분리하는 패턴 자체가 표준** — AZ별 정책(limit, budget)을 독립적으로 관리 가능 |
| `taints` | `dedicated=api:NoSchedule` | Deployment의 toleration과 짝을 이루어 api 워크로드만 이 노드에 스케줄링되도록 격리 |
| `limits.cpu/memory` | `cpu: 4, memory: 16Gi` | dev 규모에 맞는 보수적 상한. 무한정 스케일아웃을 막는 안전핀 — 예상치 못한 폭주로 인한 과금 폭탄 방지 |
| `expireAfter` | `720h` (30일) | 명시하지 않으면 Karpenter가 동일한 기본값을 암묵적으로 적용. **git에 의도가 기록되지 않는다는 것 자체가 갭** — 노드 만료도 재배치 이벤트이므로 §6-1 하드 제약 없이는 30일마다 동일 사고가 재현될 잠재 위험 |

**`consolidationPolicy` 재검토:**

`WhenEmpty`로 바꾸면 이번 사고를 막을 수 있을까?

```
[이번 사고의 Karpenter 로그]
"reason": "empty"   ← 파드 0개인 노드를 삭제
"decision": "delete"
```

| 정책 | 파드 0개인 노드 | 저활용 노드 |
|---|---|---|
| `WhenEmpty` | 삭제 대상 ✅ | 건드리지 않음 |
| `WhenEmptyOrUnderutilized` | 삭제 대상 ✅ | 삭제 대상 |

두 정책 모두 **완전히 빈 노드는 동일하게 즉시 삭제 대상**이다. `WhenEmpty`로 바꿔도 이번 사고는 그대로 재현된다. 따라서 `consolidationPolicy` 변경은 기각했다.

**`consolidateAfter: 30s → 2m` 변경 이유:**

```
롤링 업데이트 타임라인 (api, replica 2):
  t=0s   새 파드-A 생성 (AZ-a)
  t=10s  기존 파드-B (AZ-c) 종료 → AZ-c 노드 Empty
  t=40s  consolidateAfter=30s → Karpenter가 AZ-c 노드 삭제 ← 이번 사고
  t=45s  새 파드-C 생성 요청 → AZ-c 노드 없음 → AZ-a에 배치 (ScheduleAnyway)

  consolidateAfter=2m 이었다면:
  t=10s  기존 파드-B 종료 → AZ-c 노드 Empty
  t=30s  새 파드-C 생성 요청 → DoNotSchedule이면 Pending
         → Karpenter가 AZ-c에 새 노드 프로비저닝 (~60s)
  t=90s  AZ-c 노드 준비 → 파드-C 배치
  t=120s consolidateAfter=2m → 아직 AZ-c 노드에 파드-C 있음 → 삭제 안 함 ✅
```

> `consolidateAfter` 연장은 `DoNotSchedule`(근본 원인 수정)을 보완하는 **안전 마진**이다.<br>
> `DoNotSchedule`만으로 이미 쏠림 경로가 차단되지만, 아직 파악하지 못한 엣지케이스를 대비해 여유 시간을 확보했다.
{: .prompt-tip }

**`disruption.budgets` — 암묵적 기본값의 위험:**

```yaml
# 명시하지 않으면 Karpenter가 자동으로 채우는 기본값
disruption:
  budgets:
  - nodes: "10%"
```

`nodes: "10%"`는 "한 번에 전체 NodePool 노드의 10%까지만 동시에 disruption 허용"을 의미한다. NodePool당 노드가 1대라면 10%는 0.1대 → 올림해서 **1대**, 즉 사실상 제한 없음과 같다.

이커머스 운영 표준은 여기에 `schedule`을 추가해 피크 시간대 Consolidation을 금지한다.

```yaml
# 운영 표준 권장 예시
disruption:
  budgets:
  - nodes: "20%"
  - nodes: "0"    # 피크 시간대 완전 차단
    schedule: "0 11 * * 1-5"   # 평일 11시~13시 (점심 피크)
    duration: 2h
  - nodes: "0"
    schedule: "0 18 * * 1-5"   # 평일 18시~21시 (저녁 피크)
    duration: 3h
```

> **현재 `budgets`가 git에 명시되어 있지 않다** — Karpenter 기본값이 적용 중이지만 코드에서 의도를 확인할 방법이 없다. "이 값을 왜 이렇게 뒀는지"가 git history에 전혀 기록되지 않는 것 자체가 유지보수 갭이다.
{: .prompt-warning }

---

#### Karpenter EC2NodeClass — 보안 관련 필드

```yaml
# kubernetes/karpenter/ec2nodeclass-api-apne2.yaml
spec:
  amiSelectorTerms:
    - alias: al2023@latest      # 항상 최신 AL2023 AMI 자동 추적
  instanceProfile: <prefix>-karpenter-apne2-node-profile
  subnetSelectorTerms:
    - tags:
        Name: <prefix>-private-apne2-01-sub   # 태그 기반 서브넷 선택
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/<cluster-name>: owned
  metadataOptions:
    httpEndpoint: enabled
    httpTokens: required          # IMDSv2 강제
    httpPutResponseHopLimit: 1
  tags:
    Environment: development
    ManagedBy: karpenter
```

**필드별 표준 관점:**

| 필드 | 현재 값 | 표준 관점 |
|---|---|---|
| `amiSelectorTerms: al2023@latest` | 최신 AMI 자동 추적 | 보안 패치 자동 반영 장점. 단 **새 AMI = 노드 교체 이벤트** → `DoNotSchedule` 없으면 이번 사고와 동일한 AZ 편중이 AMI 업데이트 시마다 발생 가능 |
| `subnetSelectorTerms` | 태그 기반 | Terraform이 서브넷을 재생성해도(ID가 바뀌어도) 깨지지 않는 표준 패턴. 하드코딩된 Subnet ID보다 유연 |
| `securityGroupSelectorTerms` | `kubernetes.io/cluster/<name>: owned` 태그 | EKS가 클러스터 생성 시 자동으로 붙이는 태그. 안전하고 표준적인 방식 |
| `httpTokens: required` | IMDSv2 강제 | AWS Well-Architected 보안 필러에서 명시적으로 권장. IMDSv1은 SSRF 취약점을 통한 인스턴스 메타데이터(IAM 자격증명 등) 탈취 위험이 있음 |
| `httpPutResponseHopLimit: 1` | 1홉 제한 | 컨테이너 내부에서 호스트의 IMDS를 우회 접근하는 것을 막는 심화 보안 설정. 기본값은 2인데 1로 낮춰야 컨테이너→호스트 IMDS 우회를 막을 수 있음 |

> **`httpPutResponseHopLimit: 1`이 중요한 이유**<br>
> 컨테이너 내부 → 호스트 네트워크 → IMDS(`169.254.169.254`) 경로는 TTL이 2홉 이상 필요하다.<br>
> `hopLimit: 1`로 설정하면 이 경로가 차단되어, 컨테이너가 탈취당해도 호스트의 IAM 자격증명을 가져갈 수 없다.
{: .prompt-info }

**AZ별 NodePool + EC2NodeClass 분리 패턴의 실익:**

```
[현재 구조]
nodepool-api-apne2a.yaml → ec2nodeclass-api-apne2a.yaml (subnet: apne2a 전용)
nodepool-api-apne2c.yaml → ec2nodeclass-api-apne2c.yaml (subnet: apne2c 전용)

[장점]
① AZ별 limit 독립 설정 가능
   apne2a: cpu=4, memory=16Gi
   apne2c: cpu=8, memory=32Gi  ← AZ별로 다른 capacity 할당 가능

② AZ별 budget 독립 설정 가능
   apne2a만 피크시간대 consolidation 금지 → apne2c는 허용 등

③ 서브넷이 AZ에 고정되어 있어 노드가 반드시 원하는 AZ에만 생성됨
   (단일 NodePool에 여러 AZ를 허용하면 Karpenter가 어느 AZ를 선택할지 예측 어려움)
```

---

### 현재 구성 vs 표준 — 필드별 갭 비교

| 필드 | 기존 값 | 개선 후 값 | 상태 |
|---|---|---|---|
| `whenUnsatisfiable` | `ScheduleAnyway` | `DoNotSchedule` | ✅ 2026-07-08 반영 |
| `minDomains` | 미설정 | `2` | ✅ 2026-07-08 반영 |
| `strategy.rollingUpdate` | 미명시(K8s 기본값) | `maxSurge: 25%, maxUnavailable: 25%` 명시 | ✅ 2026-07-08 반영 |
| `consolidationPolicy` | `WhenEmptyOrUnderutilized` | 변경 없음 (WhenEmpty로 변경해도 효과 없음) | 🚫 기각 |
| `consolidateAfter` | `30s` | `2m` | ✅ 2026-07-08 반영 |
| `disruption.budgets` | 미명시(기본값 `nodes: 10%`) | 명시 필요 | ❌ 미반영 (남은 과제) |
| NodePool AZ 수 | 2개 AZ | 3개 AZ | ❌ PROD 단계에서 해소 예정 |
| `pdb-api.yaml` minAvailable | `1` | `1` | ✅ 충족 |
| `metadataOptions.httpTokens` | `required` (IMDSv2 강제) | `required` | ✅ 충족 |

---

### 개선 결과 (2026-07-08 dev 반영 완료)

**반영 완료 — `whenUnsatisfiable: DoNotSchedule` + `minDomains: 2`**

```yaml
# kubernetes/apps/deployment/deployment-api.yaml (변경 후)
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule   # ScheduleAnyway → DoNotSchedule
  minDomains: 2                       # 신규 추가
  labelSelector:
    matchLabels:
      app: <prefix>-api

strategy:
  rollingUpdate:
    maxSurge: 25%          # 기존 K8s 기본값에 암묵 의존 → 명시
    maxUnavailable: 25%
```

트레이드오프: 노드 교체 직후 등 과도기엔 두 번째 파드가 일시적으로 Pending 상태가 된다(Karpenter가 새 노드를 띄우는 수십 초~수 분). Karpenter가 반대편 AZ에 노드를 자동으로 프로비저닝하므로 서비스는 1개 파드로 계속 운영된다.

**반영 완료 — `consolidateAfter: 30s → 2m`**

```yaml
# kubernetes/karpenter/nodepool-api-apne2a.yaml (및 2c, web 4개 파일 동일 변경)
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 2m   # 30s → 2m
```

**보류 — 3-AZ 확장**

dev 환경은 비용 트레이드오프로 2-AZ를 의도적으로 유지. PROD 승격 시점에 `ap-northeast-2b` NodePool 추가, HPA `minReplicas: 3` 상향을 별도 작업으로 진행.

---

### 남은 과제

- [ ] Karpenter api/web NodePool의 `disruption.budgets` 명시 (현재 암묵적 기본값 `nodes: 10%` 의존)
- [ ] 변경 후 재시작 이벤트를 인위적으로 유발해, AZ당 1개 노드 유지되는지 재검증 (저위험 시크릿으로)
- [ ] PROD 설계 시 3-AZ 확장 반영 (별도 작업)
- [ ] `expireAfter`(30일 노드 만료) + `al2023@latest`(자동 AMI 추적)도 재배치 이벤트이므로, 30일 뒤 정상 분산 확인

---

### 회고

> **"지금 우연히 정상으로 보인다"와 "구조적으로 보장된다"는 다르다.** `nextjs`는 현재 AZ에 잘 분산돼 있지만, `api`와 완전히 동일한 소프트 제약(`ScheduleAnyway`) + 공격적인 Karpenter 통합 정책을 갖고 있어 다음 재시작 타이밍에 따라 언제든 같은 방식으로 쏠릴 수 있다.
{: .prompt-warning }

> **두 시스템이 각각 "설계대로" 동작해도, 조합하면 의도하지 않은 결과가 나올 수 있다.** `ScheduleAnyway`와 `WhenEmptyOrUnderutilized`는 각각 합리적인 기본값이지만, 같이 있으면 "한 번의 쏠림이 영구화"되는 조합이 된다.
{: .prompt-danger }

> **"암묵적 기본값"은 git에 의도가 안 남는다.** `budgets`, `expireAfter`, `strategy.rollingUpdate` 전부 명시하지 않았는데도 실제로는 값이 채워져 동작한다. 동작 자체는 문제가 아니지만 "이 값을 왜 이렇게 뒀는지"가 git history에 기록되지 않는다는 건 향후 동일한 조사를 반복하게 만드는 갭이다.
{: .prompt-tip }

> **Karpenter 로그의 `"reason":"empty","decision":"delete"`처럼, 결정적 증거는 대개 컨트롤러 자신의 로그에 이미 남아있다.** 가설을 세우고 추측하기보다, 관련 컨트롤러의 로그를 먼저 훑는 게 더 빠른 경로였다.
{: .prompt-tip }
