---
title: "E-commerce EKS Workload Resilience Enhancement Guide"
date: 2026-06-08 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, hpa, karpenter, pdb, graceful-shutdown, msa, crossplane, gitops, argocd, kubernetes]
---

> 이번 포스팅은 이커머스 EKS 환경에서 워크로드 복원력을 높이기 위한 설계 가이드다.<br>
> HPA 설계 및 Alert Expression 올바른 작성법, PDB를 통한 자발적 중단 제어, Graceful Shutdown 설계, AZ 분산 전략, 그리고 운영 환경 전환 시 체크리스트와 MSA 전환 장기 로드맵까지 다룬다.
{: .prompt-info }

> **관련 파일**<br>
> `kubernetes/apps/HorizontalPodAutoscaler/hpa-api.yaml`<br>
> `kubernetes/apps/HorizontalPodAutoscaler/hpa-nextjs.yaml`<br>
> `kubernetes/apps/HorizontalPodAutoscaler/pdb-api.yaml` _(신규)_<br>
> `kubernetes/apps/HorizontalPodAutoscaler/pdb-nextjs.yaml` _(신규)_<br>
> `kubernetes/apps/deployment/deployment-api.yaml`<br>
> `kubernetes/apps/deployment/deployment-nextjs.yaml`<br>
> `helm/monitoring/kube-prometheus-stack/values-alert-rules-ops.yaml`
{: .prompt-tip }

---

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

## 7. 배포 흐름 (GitOps)

이 프로젝트는 ArgoCD로 GitOps를 구성하고 있으며, 모든 변경은 Git을 통해서만 적용해야 한다.

```
파일 수정 → git commit & push → ArgoCD 자동 감지 → 자동 배포 (automated + selfHeal)
```

> **주의**: `kubectl apply` 또는 `helm upgrade` 직접 실행 시 ArgoCD가 selfHeal로 되돌릴 수 있다.<br>
> 긴급 상황이 아니면 반드시 Git을 통해 변경할 것.
{: .prompt-danger }

---

## 8. 모놀리식 API → MSA 전환 전략 (장기 로드맵)

현재 구조는 단일 `api` 파드가 모든 도메인(상품, 장바구니, 주문, 결제, 회원 등)을 처리하는 **모놀리식 백엔드 on Kubernetes** 구조다. MSA가 반드시 정답은 아니지만, 서비스가 성장하면서 자연스럽게 분리 압력이 생긴다.

### 분리 시작 신호 (When to Split)

아래 항목 중 하나라도 지속적으로 발생한다면 해당 도메인의 분리를 검토해야 한다.

| 신호 | 의미 |
|---|---|
| 특정 API 엔드포인트가 CPU/Memory 사용량의 80% 이상을 차지 | 해당 기능만 독립 스케일 필요 |
| 배포 시 특정 기능 때문에 전체 롤링 업데이트 지연 | 배포 단위 분리 필요 |
| 두 팀 이상이 같은 코드베이스에서 충돌 | 팀 경계 = 서비스 경계 |
| 특정 기능의 SLA가 나머지와 다름 (결제 99.99% vs 상품 99.9%) | 독립적 운영 필요 |
| 특정 기능의 응답 지연이 다른 기능에 영향 | 장애 격리 필요 |

### 분리 우선순위

```
1. 결제 서비스    — SLA/보안 요구사항이 나머지와 다름. PCI DSS 범위 축소 효과
2. 상품/검색 서비스 — 트래픽이 가장 많고 캐싱 전략이 독립적
3. 주문 서비스    — 복잡한 비즈니스 로직, 독립 배포 요구 높음
4. 장바구니 서비스 — Redis 기반 세션 관리로 독립화 용이
5. 회원/인증 서비스 — 모든 서비스가 의존하므로 안정화 후 마지막
6. 알림 서비스    — 비동기로 분리하기 가장 쉬운 도메인
```

### 단계별 전환 로드맵

**Phase 0 — 현재 (모놀리스 on Kubernetes)**

```
[NextJS] → [API 모노리스] → [RDS MySQL]
```

해야 할 것: 코드 내부에서 도메인 경계를 명확히 분리 (패키지/모듈 단위). 나중에 서비스 경계로 쓸 수 있도록 준비.

**Phase 1 — Strangler Fig 패턴 적용 (API Gateway 도입)**

```
[NextJS] → [API Gateway]
                ↓             ↓
         [API 모노리스]   [결제 서비스] ← 첫 분리
                              ↓
                        [RDS PostgreSQL]
```

> **Strangler Fig 패턴**이란 기존 모놀리스를 죽이지 않고, 앞단에 API Gateway를 놓고 새 서비스로 트래픽을 점진적으로 이전하는 패턴이다.
{: .prompt-info }

**인프라 변경 사항 — 결제 서비스 신규 리소스:**

```yaml
# 결제 서비스 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-deploy
  namespace: <prefix>-payment-kr
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-service
  template:
    spec:
      containers:
      - name: payment
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/payment-service:latest
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # 결제는 더 길게
      terminationGracePeriodSeconds: 60
---
# 결제 서비스 전용 HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: <prefix>-payment-kr
spec:
  minReplicas: 2
  maxReplicas: 20    # 결제는 더 높게
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # 결제는 더 보수적으로
---
# 결제 서비스 PDB — 무중단 필수
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: <prefix>-payment-kr
spec:
  minAvailable: "50%"  # 결제는 절대값이 아닌 비율
  selector:
    matchLabels:
      app: payment-service
```

**API Gateway 라우팅 예시 (Kong):**

```yaml
# /api/v1/payments/* → payment-service
# /api/v1/* (나머지) → api-monolith (기존 유지)
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: payment-route
proxy:
  path: /api/v1/payments
```
```

**Phase 2 — 상품/검색 서비스 분리**

상품 서비스는 읽기 트래픽이 압도적이므로 **읽기/쓰기 분리(CQRS)**와 **캐싱** 전략이 핵심이다.

```yaml
# 상품 서비스는 읽기 부하가 높으므로 HPA를 더 공격적으로
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 200       # 트래픽 급증 시 3배까지 즉시 확장
      periodSeconds: 30
```

**Phase 3 — 이벤트 기반 비동기 분리 (주문/알림)**

서비스 간 동기 호출이 늘어날수록 장애 전파 위험이 커진다. 주문 완료 후 알림 발송 같은 흐름은 **메시지 큐(Amazon SQS)**로 분리한다.

```
[주문 서비스] → SQS → [알림 서비스]
                    → [정산 서비스]
                    → [재고 서비스]
```

KEDA로 SQS 메시지 수 기반 스케일링:

```yaml
triggers:
- type: aws-sqs-queue
  metadata:
    queueURL: https://sqs.ap-northeast-2.amazonaws.com/<account-id>/order-notification-queue
    queueLength: "10"   # 큐에 메시지 10개당 파드 1개 추가
    awsRegion: ap-northeast-2
```

**Phase 4 — 서비스 메시 도입 (Istio / Linkerd)**

서비스 수가 10개 이상이 되면 서비스 간 통신 제어가 필요해진다.

```
서비스 메시가 제공하는 것:
- mTLS: 서비스 간 통신 암호화 (결제 서비스 필수)
- Circuit Breaker: 특정 서비스 장애 시 연쇄 장애 차단
- Retry / Timeout: 코드 변경 없이 정책으로 제어
- Traffic Splitting: 카나리 배포 (상품 서비스 신버전을 10%만)
- 분산 트레이싱: Jaeger/Zipkin 연동
```

### 데이터베이스 분리 전략

```
Phase 0: 모노리스 → 단일 RDS MySQL
Phase 1: 결제 서비스 분리 → 결제 전용 RDS (PCI DSS 격리)
Phase 2: 상품 서비스 분리 → RDS(쓰기) + ElasticSearch(검색) + Redis(캐시)
Phase 3: 나머지 서비스 → 도메인별 RDS / DynamoDB 선택
```

> **분산 트랜잭션 문제**<br>
> 단일 DB에서는 트랜잭션으로 해결되던 것이 MSA에서는 복잡해진다.<br>
> 주문 생성 + 재고 차감 + 결제 승인이 각각 다른 DB에 있을 때 결제 실패 시 롤백 방법이 필요하다.<br>
> **Saga 패턴**(Choreography 또는 Orchestration)으로 보상 트랜잭션(Compensating Transaction)을 구현해야 한다.
{: .prompt-warning }

**Saga 패턴 — Choreography (이벤트 기반):**

```
정상 흐름:
  주문생성 → OrderCreated 이벤트 발행
           → 재고서비스 차감 → StockReduced 이벤트 발행
           → 결제서비스 승인 → PaymentApproved 이벤트 발행
           → 주문 최종 확정

실패 시 보상 트랜잭션(Compensating Transaction):
  결제실패 → PaymentFailed 이벤트 발행
          → 재고서비스 복구 → StockReleased 이벤트 발행
          → 주문 취소     → OrderCancelled 이벤트 발행
```

> 각 서비스는 이벤트를 구독하고 처리 결과를 새 이벤트로 발행한다. 중앙 조율자 없이 서비스 간 독립성을 유지하는 것이 Choreography Saga의 핵심이다.
{: .prompt-info }

---

## 9. MSA 전환 시 인프라 GitOps 아키텍처

### 현재 구조의 한계

서비스가 늘어날수록 Terraform 수동 관리가 병목이 된다.

```
# 현재
Terraform (수동/CI) → EKS, VPC, RDS x1, IAM

# MSA 이후 서비스당 필요한 리소스
payment  → RDS, IAM Role(IRSA), SQS, KMS
product  → RDS, ElasticSearch, ElastiCache, IAM Role
order    → RDS, SQS, IAM Role
...
```

### 인프라 GitOps 도구 비교

| 도구 | 방식 | ArgoCD 연동 | 특징 |
|---|---|---|---|
| **tf-controller** | Terraform CRD | 별도 (Flux 기반) | 기존 Terraform 코드 재사용 가능 |
| **Crossplane** | K8s-native CRD | 완벽 통합 | ArgoCD 하나로 K8s+AWS 통합 관리 |
| **ACK** | K8s-native CRD | 완벽 통합 | AWS 공식, AWS 리소스에 특화 |
| **Atlantis** | PR → plan/apply | 없음 | 가장 단순, 기존 Terraform 유지 |

### 권고 아키텍처: 2단계 전환

**단기 (서비스 1~5개): Atlantis 도입**

Terraform 코드는 그대로 두되, `terraform apply`를 PR 기반으로 자동화한다.

```
Developer opens PR
  → Atlantis runs terraform plan
  → Posts plan result as PR comment
  → PR approved + comment "atlantis apply"
  → terraform apply runs automatically
```

**중장기 (서비스 5개 이상): Crossplane 도입**

Kubernetes CRD로 AWS 리소스를 선언하고, ArgoCD가 이를 sync한다.

```yaml
# 결제 서비스 전용 RDS를 Crossplane CRD로 선언
apiVersion: rds.aws.crossplane.io/v1alpha1
kind: DBInstance
metadata:
  name: payment-rds
  namespace: <payment-namespace>
spec:
  forProvider:
    region: ap-northeast-2
    dbInstanceClass: db.t3.medium
    engine: mysql
    engineVersion: "8.0"
    multiAZ: true
  writeConnectionSecretsToRef:
    name: payment-db-secret  # 접속 정보를 K8s Secret으로 자동 저장
```

### 전체 GitOps 아키텍처 (MSA + Crossplane 도입 후)

```
+-----------------------------------------------+
|                    GitHub                     |
|  kubernetes/                                  |
|  ├── apps/payment/    (Deployment, HPA, PDB)  |
|  ├── apps/product/    (Deployment, HPA, PDB)  |
|  └── infra/           (Crossplane CRDs)       |
|      ├── payment-rds.yaml                     |
|      ├── payment-sqs.yaml                     |
|      └── product-elasticache.yaml             |
+-------------------+---------------------------+
                    │ git push
                    ▼
+-----------------------------------------------+
|                   ArgoCD                      |
|  <prefix>-payment-app  →  apps/payment/       |
|  <prefix>-product-app  →  apps/product/       |
|  <prefix>-infra-app    →  infra/              |
+----------+--------------------+---------------+
           │                    │
           │ K8s resources      │ Crossplane CRDs
           ▼                    ▼
+------------------+   +--------------------+
|   EKS Cluster    |   |    Crossplane      |
|  payment-pod     |   │   (AWS Provider)   |
|  product-pod     |   └──── AWS API call ──→ RDS / SQS / IAM
+------------------+
```

### Karpenter NodePool — 서비스 특성별 분리

```yaml
# 결제 서비스 — On-Demand 전용, Spot 금지
- key: karpenter.sh/capacity-type
  operator: In
  values: ["on-demand"]   # 결제는 절대 Spot 사용 금지

# 상품/검색 서비스 — 읽기 부하, Spot 허용
- key: karpenter.sh/capacity-type
  operator: In
  values: ["spot", "on-demand"]

# 알림/정산 서비스 — 비동기, Spot 최대 활용
- key: karpenter.sh/capacity-type
  operator: In
  values: ["spot"]
```

### 도입 시점 판단 기준

| 단계 | 서비스 수 | 팀 수 | 권고 |
|---|:---:|:---:|---|
| 현재 | 1 (모노리스) | 1 | 현행 유지 |
| MSA 초기 | 2~5 | 1~2 | Atlantis 도입 |
| MSA 성장기 | 5+ | 3+ | Crossplane 도입 |
| MSA 성숙기 | 10+ | 5+ | Crossplane Composition (플랫폼 팀) |

> **Crossplane Composition 성숙기 예시**<br>
> 플랫폼팀이 표준 서비스 패키지를 정의하면, 개발팀은 아래 선언 하나로 RDS + SQS + IAM Role + Secret을 자동 생성할 수 있다.
{: .prompt-tip }

```yaml
apiVersion: platform.example.com/v1alpha1
kind: EcommerceService
metadata:
  name: payment-infra
spec:
  dbInstanceClass: db.t3.medium
  queueCount: 2
  region: ap-northeast-2
# 위 선언 하나로 RDS + SQS + IAM Role + Secret 자동 생성
```

### MSA 전환 시 EKS 인프라 변경 사항

**Namespace 전략**

```yaml
# 현재: 단일 namespace
<app-namespace>

# MSA 전환 후: 서비스별 namespace (팀 단위 격리)
<prefix>-payment-kr     # 결제팀 관리
<prefix>-product-kr     # 상품팀 관리
<prefix>-order-kr       # 주문팀 관리
<prefix>-member-kr      # 회원팀 관리
```

**NetworkPolicy — 서비스 간 통신 명시적 허용**

MSA에서는 서비스 간 통신을 명시적으로 허용해야 한다. 기본적으로 모든 통신을 차단하고 필요한 것만 열어주는 것이 보안 원칙이다.

```yaml
# 결제 서비스는 주문 서비스에서만 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-ingress-policy
  namespace: <prefix>-payment-kr
spec:
  podSelector:
    matchLabels:
      app: payment-service
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: <prefix>-order-kr
    ports:
    - port: 8080
```

**서비스별 독립 ArgoCD Application**

```yaml
# 현재: 단일 app-of-apps
<prefix>-apps → 모든 워크로드

# MSA 전환 후: 서비스별 독립 Application
<prefix>-payment-app  → kubernetes/apps/payment/
<prefix>-product-app  → kubernetes/apps/product/
<prefix>-order-app    → kubernetes/apps/order/
# 팀마다 자신의 Application만 배포 가능
```

---

### 관찰 가능성 (Observability) 강화

MSA에서는 요청이 여러 서비스를 거치므로 기존 로그/메트릭만으로는 장애 추적이 불가능하다.

```
현재: Loki(로그) + Prometheus(메트릭) + Grafana(시각화)

MSA 필요 추가: Jaeger 또는 Grafana Tempo (분산 트레이싱)
```

**분산 트레이싱 흐름**

```
사용자 요청
  → NextJS (trace-id: abc-123 생성)
  → API Gateway (trace-id 전파)
  → 주문 서비스 (span 기록)
  → 결제 서비스 (span 기록)
  → 알림 서비스 (span 기록)

Grafana Tempo에서 trace-id 하나로 전체 흐름 추적 가능
```

> Grafana Tempo는 기존 PLG 스택과 통합이 자연스러워 MSA 분산 트레이싱 도구로 권장한다.
{: .prompt-tip }

**서비스별 알림 분리**

```yaml
# MSA 전환 후 알림은 서비스별로 severity 구분
- alert: PaymentServiceDown
  expr: up{job="payment-service"} == 0
  labels:
    severity: critical   # 결제는 critical
    team: payment

- alert: ProductServiceDown
  expr: up{job="product-service"} == 0
  labels:
    severity: warning    # 상품은 warning (캐시로 일부 서빙 가능)
    team: product
```

---

### MSA 전환 체크리스트

**분리 전 준비 (코드 레벨)**

```
□ 모노리스 내부 도메인 패키지 경계 명확히 정리
□ 도메인 간 직접 메서드 호출 → 인터페이스로 추상화
□ 공유 DB 테이블 현황 파악 (분리 시 가장 어려운 부분)
□ 서비스 간 트랜잭션 흐름 문서화
□ API 명세 (OpenAPI/Swagger) 정리
```

**인프라 레벨 준비**

```
□ API Gateway 도입 (Kong / AWS API GW)
□ 서비스별 ECR 저장소 분리
□ 서비스별 Namespace 설계
□ Karpenter NodePool 서비스 특성별 분리
□ NetworkPolicy 설계 (서비스 간 통신 허용 목록)
□ 서비스별 독립 RDS 또는 스키마 분리
□ 분산 트레이싱 도구 도입 (Grafana Tempo 권장)
□ 서비스 메시 도입 검토 (Istio / Linkerd)
```

**분리 후 검증**

```
□ 각 서비스 독립 배포 확인 (다른 서비스 영향 없음)
□ 한 서비스 장애 시 다른 서비스 정상 동작 확인 (Circuit Breaker 테스트)
□ 분산 트레이싱으로 전체 요청 흐름 추적 가능 확인
□ 서비스별 HPA 독립 동작 확인
□ 카오스 엔지니어링으로 장애 격리 검증 (선택)
```

---

### IRSA 전략 — Crossplane으로 자동화

MSA에서는 서비스별 최소 권한 IAM Role이 필수다. Crossplane으로 IRSA까지 자동화할 수 있다.

```yaml
# 결제 서비스 IAM Role — SQS, KMS만 허용
apiVersion: iam.aws.crossplane.io/v1beta1
kind: Role
metadata:
  name: payment-service-irsa-role
spec:
  forProvider:
    assumeRolePolicyDocument: |
      {
        "Statement": [{
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::<account-id>:oidc-provider/..."
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "...:sub": "system:serviceaccount:<payment-namespace>:payment-service-sa"
            }
          }
        }]
      }
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: <payment-namespace>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/payment-service-irsa-role
```

> Crossplane이 IAM Role을 Git 선언만으로 자동 생성하고, ServiceAccount에 annotation을 자동으로 달아준다. 더 이상 Terraform으로 IRSA를 수동 관리할 필요가 없다.
{: .prompt-tip }

---

### 흔한 실수와 주의사항

> **1. 너무 이른 MSA 전환**<br>
> 팀 규모, 트래픽이 받쳐주지 않는데 MSA를 도입하면 운영 복잡도만 증가한다. 모놀리스의 개발 속도를 MSA가 따라가려면 상당한 인프라 투자가 필요하다.
{: .prompt-warning }

> **2. 분산 모놀리스(Distributed Monolith)**<br>
> 서비스를 물리적으로 분리했지만 DB를 공유하거나 동기 호출이 과도한 경우. MSA의 장점은 없고 복잡도만 증가하는 최악의 케이스다.
{: .prompt-danger }

> **3. 서비스 경계를 기술이 아닌 비즈니스 도메인으로**<br>
> 잘못된 예: DB 레이어 서비스, 인증 레이어 서비스 (기술 계층 분리)<br>
> 올바른 예: 결제 서비스, 주문 서비스, 상품 서비스 (비즈니스 도메인 분리)
{: .prompt-tip }

> **4. 서비스 간 공유 라이브러리 남용**<br>
> 공통 유틸, DTO를 shared-library로 묶으면 다시 강결합이 발생한다.<br>
> 서비스마다 독립적인 모델을 가지고, 필요 시 API 계약(OpenAPI)으로만 통신해야 한다.
{: .prompt-warning }
