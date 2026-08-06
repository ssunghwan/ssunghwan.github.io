---
title: "흩어진 배포를 하나로 - ArgoCD App of Apps 아키텍처 전환"
date: 2026-06-06 09:00:00 +0900
categories: [2. Kubernetes, Cloud Native Transformation]
tags: [eks, argocd, gitops, app-of-apps, sync-wave, kubernetes, devops, iac, helm, multi-source, server-side-apply]
---

> **환경**: AWS EKS / ArgoCD v3.4.2<br>
> **Phase 1**: 수동 kubectl apply 기반 운영 → 완전한 GitOps 아키텍처로 전환 (2026-06-06)<br>
> **Phase 2**: 고아 Helm 릴리스 8개 GitOps 전환 및 동기화 안정화 (2026-06-26)
{: .prompt-info }

---

## 1. 배경 및 문제 정의

### 전환 이전 상태

ArgoCD가 도입되어 있었지만 `kubernetes/apps` 디렉토리(Deployment, HPA)만 관리하고 있었고, 나머지 인프라 리소스는 **모두 수동 `kubectl apply`로 운영**되고 있었다.

```bash
# 수동 관리 영역 — Git과 클러스터가 언제든 달라질 수 있음
kubectl apply -f kubernetes/ingress/          # 누가 언제 바꿨는지 추적 불가
kubectl apply -f kubernetes/external-secrets/
kubectl apply -f kubernetes/karpenter/
kubectl apply -f kubernetes/monitoring/
kubectl apply -f kubernetes/storage/
```

### 이것이 왜 문제인가

| 문제 | 설명 |
|---|---|
| **드리프트** | 클러스터 상태와 Git이 달라져도 아무도 모름 |
| **추적 불가** | 누가 어떤 변경을 언제 했는지 알 수 없음 |
| **재현 불가** | 클러스터가 날아가면 수동으로 순서 맞춰 재적용 필요 |
| **롤백 어려움** | 문제 발생 시 이전 상태로 되돌리는 절차가 없음 |
| **온보딩 비용** | 새 팀원이 "어디에 뭘 apply해야 하나" 파악에 시간 소요 |

> 이커머스 환경에서는 세일/이벤트 시 클러스터에 급격한 변경이 발생한다.<br>
> 이 상황에서 수동 관리는 장애 위험을 높이고 대응 속도를 떨어뜨린다.
{: .prompt-danger }

---

## 2. GitOps 핵심 원칙

GitOps는 Git을 **Single Source of Truth(단일 진실 공급원)**로 삼는 운영 방식이다.

```
개발자가 PR → merge → ArgoCD 자동 감지 → 클러스터 반영
                              ↑
                      Git 상태 = 클러스터 상태
```

### 4가지 원칙

**선언적(Declarative)**: 시스템의 상태를 코드로 선언 (명령어 X)

**버전 관리(Versioned)**: 모든 변경이 Git 커밋으로 기록됨

**자동 적용(Automated)**: Git 변경 → 클러스터 자동 동기화

**지속적 검증(Continuously reconciled)**: ArgoCD가 드리프트를 감지하고 자동 복구

### selfHeal이 중요한 이유

```yaml
syncPolicy:
  automated:
    selfHeal: true   # 누군가 클러스터에서 직접 수정해도 Git 상태로 되돌림
    prune: true      # Git에서 삭제된 리소스는 클러스터에서도 삭제
```

> **`selfHeal: true`가 없으면** 누군가 `kubectl edit`으로 직접 바꿨을 때 ArgoCD가 모르고 넘어간다.<br>
> 이것이 드리프트의 가장 흔한 원인이다. 운영 환경에서는 반드시 활성화하는 것이 권장된다.
{: .prompt-warning }

---

## 3. App of Apps 패턴 이해

### 단일 Application의 한계

ArgoCD Application 하나로 전체를 관리하면 여러 문제가 발생한다.

- 하나의 디렉토리에 모든 리소스가 뒤섞임
- 배포 순서를 제어할 수 없음
- 레이어별 다른 sync 정책 적용 불가
- 장애 영향 범위가 넓어짐

### App of Apps 패턴

**루트 Application이 다른 Application들을 관리**하는 계층 구조다.

```
[Git: kubernetes/argocd/]
app-of-apps.yaml
  ├── application-storage.yaml
  ├── application-karpenter.yaml
  ├── application-external-secrets.yaml
  ├── application-ingress.yaml
  ├── application-monitoring.yaml
  └── application-apps.yaml

[ArgoCD]
<prefix>-appsync-apne2-cluster  (루트)
  ├── <prefix>-storage
  ├── <prefix>-karpenter
  ├── <prefix>-external-secrets
  ├── <prefix>-ingress
  ├── <prefix>-monitoring
  └── <prefix>-apps
```

### Self-Managed 구조

루트 Application 자신도 Git에 있으며 ArgoCD가 관리한다. `include: "application-*.yaml"` 패턴으로 자기 자신(`app-of-apps.yaml`)은 제외하고 하위 Application 파일들만 읽는다.

```yaml
directory:
  recurse: false
  include: "application-*.yaml"   # app-of-apps.yaml은 패턴에서 자동 제외
```

> **새로운 레이어를 추가하려면 `application-xxx.yaml` 파일 하나만 추가하면 된다.**<br>
> ArgoCD가 자동으로 감지하여 새 Application을 생성한다. 이것이 App of Apps 패턴의 핵심 장점이다.
{: .prompt-tip }

---

## 4. Before / After 아키텍처

### Before

```
Git Repository
├── kubernetes/
│   ├── apps/                    ← ArgoCD 관리 ✅
│   │   ├── deployment/
│   │   └── HorizontalPodAutoscaler/
│   ├── argocd/
│   │   └── application-nextjs.yaml   ← 단일 Application 정의
│   ├── external-secrets/            ← 수동 kubectl apply ❌
│   ├── ingress/                     ← 수동 kubectl apply ❌
│   ├── karpenter/                   ← 수동 kubectl apply ❌
│   ├── monitoring/                  ← 수동 kubectl apply ❌
│   └── storage/                     ← 수동 kubectl apply ❌

ArgoCD Applications: 1개
  └── <prefix>-argocd-apne2-pipe (kubernetes/apps)
```

### After

```
Git Repository
├── kubernetes/
│   ├── apps/
│   ├── argocd/
│   │   ├── app-of-apps.yaml                    ← 루트 (self-managed)
│   │   ├── application-storage.yaml            ← wave 1
│   │   ├── application-karpenter.yaml          ← wave 2
│   │   ├── application-external-secrets.yaml   ← wave 3
│   │   ├── application-ingress.yaml            ← wave 4
│   │   ├── application-monitoring.yaml         ← wave 5
│   │   └── application-apps.yaml               ← wave 6
│   ├── external-secrets/   ← ArgoCD 관리 ✅
│   ├── ingress/            ← ArgoCD 관리 ✅
│   ├── karpenter/          ← ArgoCD 관리 ✅
│   ├── monitoring/         ← ArgoCD 관리 ✅
│   └── storage/            ← ArgoCD 관리 ✅

ArgoCD Applications: 7개
  └── <prefix>-appsync-apne2-cluster (루트)
        ├── <prefix>-storage
        ├── <prefix>-karpenter
        ├── <prefix>-external-secrets
        ├── <prefix>-ingress
        ├── <prefix>-monitoring
        └── <prefix>-apps
```

---

## 5. 레이어 설계 전략

### 레이어 분리 기준

| 레이어 | 디렉토리 | 내용 | 변경 빈도 |
|---|---|---|---|
| storage | `kubernetes/storage/` | StorageClass, VolumeSnapshot CRD | 매우 낮음 |
| karpenter | `kubernetes/karpenter/` | NodePool, EC2NodeClass | 낮음 |
| external-secrets | `kubernetes/external-secrets/` | ExternalSecret, ClusterSecretStore | 낮음 |
| ingress | `kubernetes/ingress/` | Ingress, Service | 중간 |
| monitoring | `kubernetes/monitoring/` | ALB metrics 등 | 낮음 |
| apps | `kubernetes/apps/` | Deployment, HPA | **높음** |

> **변경 빈도가 높은 apps 레이어와 인프라 레이어를 분리**함으로써 애플리케이션 배포가 인프라 sync에 영향을 주지 않는다.<br>
> 개발팀이 `kubernetes/apps/` 경로만 PR로 변경하면 인프라팀의 승인 없이도 배포가 가능하다.
{: .prompt-info }

### Namespace 전략

리소스가 여러 namespace에 걸쳐 있는 경우(external-secrets, ingress), 각 리소스 파일에 namespace를 명시하고 Application의 `destination.namespace`는 fallback 값으로 설정한다.

```
external-secrets 리소스 namespace 분포:
  - monitoring      (grafana, alertmanager 시크릿)
  - argocd          (argocd 시크릿)
  - <app-namespace> (앱 시크릿)

ingress 리소스 namespace 분포:
  - monitoring      (grafana ingress)
  - argocd          (argocd ingress)
  - <app-namespace> (purina ingress)
```

> ArgoCD는 리소스 파일에 명시된 namespace를 우선 적용하므로, `destination.namespace`는 namespace가 없는 리소스의 fallback으로만 사용된다.
{: .prompt-tip }

---

## 6. 디렉토리 구조

```
kubernetes/
├── argocd/
│   ├── app-of-apps.yaml                    ← 루트 Application (kubectl apply로 최초 1회만 등록)
│   ├── application-storage.yaml
│   ├── application-karpenter.yaml
│   ├── application-external-secrets.yaml
│   ├── application-ingress.yaml
│   ├── application-monitoring.yaml
│   └── application-apps.yaml
│
├── storage/
│   ├── storageclass-gp3.yaml
│   └── volume-snapshot/
│       └── *.yaml
│
├── karpenter/
│   ├── ec2nodeclass-*.yaml
│   └── nodepool-*.yaml
│
├── external-secrets/
│   ├── cluster-secret-store.yaml
│   └── external-secret-*.yaml
│
├── ingress/
│   ├── namespace.yaml
│   ├── ingress.yaml
│   ├── ingress-argocd.yaml
│   ├── ingress-grafana.yaml
│   └── service.yaml
│
├── monitoring/
│   └── albc-metrics.yaml
│       (*.json 파일은 application-monitoring.yaml에서 exclude 처리)
│
└── apps/
    ├── deployment/
    │   ├── deployment-api.yaml
    │   └── deployment-nextjs.yaml
    └── HorizontalPodAutoscaler/
        ├── hpa-api.yaml
        └── hpa-nextjs.yaml
```

---

## 7. 구현 상세

### 루트 Application — app-of-apps.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-appsync-apne2-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<github-username>/<repo-name>.git
    targetRevision: main
    path: kubernetes/argocd
    directory:
      recurse: false
      include: "application-*.yaml"   # app-of-apps.yaml 자신은 제외
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true       # application-*.yaml 파일 삭제 시 해당 Application도 삭제
      selfHeal: true    # 수동 변경 시 Git 상태로 복구
    syncOptions:
      - CreateNamespace=false
```

> **`include: "application-*.yaml"` 패턴의 핵심**<br>
> `app-of-apps.yaml` 자신은 이 패턴에 해당하지 않으므로 자동으로 제외된다.<br>
> 만약 자기 자신을 읽으려 한다면 무한 루프가 발생하는데, 이 패턴 덕분에 안전하다.
{: .prompt-info }

### 하위 Application 공통 구조

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-{layer}
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "{N}"   # 배포 순서 제어
spec:
  project: default
  source:
    repoURL: git@github.com:<github-username>/<repo-name>.git
    targetRevision: main
    path: kubernetes/{layer}
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: {fallback-namespace}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

### storage — ServerSideApply 옵션

```yaml
# application-storage.yaml
syncOptions:
  - CreateNamespace=false
  - ServerSideApply=true
```

> **`ServerSideApply=true`가 필요한 이유**<br>
> VolumeSnapshot CRD처럼 스키마가 큰 리소스는 클라이언트 사이드 apply 시 `kubectl.kubernetes.io/last-applied-configuration` annotation이 너무 커져 오류가 발생한다.<br>
> `ServerSideApply=true`로 annotation 크기 제한을 우회한다.
{: .prompt-warning }

### monitoring — JSON 파일 제외

```yaml
# application-monitoring.yaml
source:
  path: kubernetes/monitoring
  directory:
    recurse: true
    exclude: "*.json"    # Grafana dashboard JSON은 K8s manifest가 아니므로 제외
```

> `kubernetes/monitoring/*.json`은 Grafana 대시보드 정의 파일이다.<br>
> ArgoCD가 Kubernetes 리소스로 apply하려고 하면 오류가 발생하므로 반드시 제외해야 한다.
{: .prompt-danger }

---

## 8. Sync Wave — 배포 순서 제어

### 왜 순서가 중요한가

인프라 레이어 간에는 명확한 의존성이 있다.

```
storage CRD 없이    → PVC 생성 불가
Karpenter 없이      → 노드 프로비저닝 불가
ExternalSecret 없이 → 앱 Secret 없음 → 파드 기동 실패
Ingress 없이        → 외부 트래픽 진입 불가
```

### Sync Wave 설계

```
Wave 1 — storage
  StorageClass(gp3), VolumeSnapshot CRD
  → 클러스터의 가장 기반이 되는 스토리지 계층

Wave 2 — karpenter
  EC2NodeClass, NodePool
  → 노드 자동 확장 정책. 앱 배포 전에 노드 프로비저닝 준비

Wave 3 — external-secrets
  ClusterSecretStore, ExternalSecret
  → 앱이 필요로 하는 Secret을 먼저 생성

Wave 4 — ingress
  Ingress, Service
  → 트래픽 진입 경로. 앱 배포 전에 라우팅 준비

Wave 5 — monitoring
  ALB metrics 등 관측성 리소스
  → 앱 배포 전 관측 준비

Wave 6 — apps
  Deployment, HPA
  → 모든 인프라가 준비된 후 마지막에 애플리케이션 배포
```

### Sync Wave 동작 원리

```
ArgoCD sync 시작
    │
    ├── Wave 1 리소스 apply → Healthy 대기
    ├── Wave 2 리소스 apply → Healthy 대기
    ├── Wave 3 리소스 apply → Healthy 대기
    ├── Wave 4 리소스 apply → Healthy 대기
    ├── Wave 5 리소스 apply → Healthy 대기
    └── Wave 6 리소스 apply → 완료
```

> 각 wave는 이전 wave의 모든 리소스가 **Healthy 상태가 된 후에** 시작된다.<br>
> wave 중 하나가 실패하면 이후 wave는 실행되지 않으므로 의존성 위반을 방지할 수 있다.
{: .prompt-info }

---

## 9. 적용 절차

### 최초 1회 — 루트 Application 등록

App of Apps 패턴에서 루트 Application은 **클러스터에 직접 한 번만 등록**한다. 이후부터는 Git 변경으로 모든 것이 관리된다.

```bash
# 1. 파일 확인
cat kubernetes/argocd/app-of-apps.yaml

# 2. 클러스터에 직접 등록 (최초 1회)
kubectl apply -f kubernetes/argocd/app-of-apps.yaml

# 3. 하위 Applications 자동 생성 확인
kubectl get applications -n argocd

# 4. git push 후 ArgoCD가 최신 상태 반영
git push origin main

# 5. 필요 시 hard refresh 트리거
kubectl annotate application <prefix>-appsync-apne2-cluster \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### 기존 Application 전환 시 주의사항

기존에 수동으로 생성된 Application을 App of Apps로 전환할 때 반드시 `--cascade=orphan`으로 삭제해야 한다.

```bash
# ❌ 잘못된 방법 — 관리 중인 Deployment, Service 등이 함께 삭제됨
kubectl delete application old-app -n argocd

# ✅ 올바른 방법 — ArgoCD Application 객체만 삭제, K8s 리소스는 유지
kubectl delete application old-app -n argocd --cascade=orphan
```

> **`--cascade=orphan`을 빠뜨리면** ArgoCD가 Application 삭제 시 관리 중이던 Deployment, Service 등을 함께 삭제한다. 운영 환경에서는 즉각적인 서비스 장애로 이어진다.
{: .prompt-danger }

---

## 10. 이커머스 현업 관점 인사이트

### GitOps가 이커머스에서 특히 중요한 이유

이커머스는 **세일/이벤트 기간에 트래픽이 급증**하는 특성이 있다. 이 시점에 인프라 변경이 자주 발생하는데, 수동 운영은 다음 위험을 낳는다.

- 긴장 상태에서 실수로 잘못된 리소스에 apply
- 누가 어떤 변경을 했는지 사후 추적 불가
- 문제 발생 시 롤백 경로가 불분명

GitOps는 이 모든 상황을 **PR 기반 승인 + 자동 apply**로 통제한다.

### 레이어 분리의 실무적 의미

```
[팀 구조와 레이어 매핑]

인프라팀  → storage, karpenter, external-secrets 레이어 소유
플랫폼팀  → ingress, monitoring 레이어 소유
개발팀    → apps 레이어 소유 (배포 자율성)
```

레이어가 분리되어 있으면 개발팀이 `kubernetes/apps/` 경로만 PR로 변경하고 플랫폼팀의 승인 없이도 배포가 가능하다. 동시에 인프라 레이어에 대한 변경은 인프라팀이 리뷰한다.

### prune: true 운영 시 주의사항

```yaml
syncPolicy:
  automated:
    prune: true    # Git에서 파일을 삭제하면 클러스터에서도 삭제됨
```

`prune: true`는 강력하지만 위험하다. 실수로 파일을 삭제하거나 잘못 커밋하면 Deployment가 삭제될 수 있다.

**현업에서의 보호 전략:**

```
1. main 브랜치 보호 규칙 설정 (GitHub Branch Protection)
   - PR 없이 직접 push 차단
   - 최소 1명 리뷰어 승인 필수

2. apps 레이어는 별도 CI 파이프라인 통과 필수
   (테스트, 이미지 검증 후에만 merge)

3. 삭제 변경은 별도 PR로 분리 (다른 변경과 묶지 않음)
```

### selfHeal과 긴급 패치의 충돌

프로덕션 장애 시 `kubectl edit`으로 즉시 수정하는 경우가 있다. `selfHeal: true`가 설정되어 있으면 ArgoCD가 이를 되돌린다.

> **긴급 패치 후 Git을 반드시 업데이트해야 한다.**<br>
> 그렇지 않으면 sync 복원 시 hotfix가 되돌아간다.
{: .prompt-danger }

**현업 긴급 패치 절차:**

```bash
# 1. 긴급 패치 적용 전 — ArgoCD sync 일시 중단
kubectl patch application <prefix>-apps -n argocd \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. 긴급 kubectl 수정 진행
kubectl set image deployment/api api=<image>:hotfix-tag -n <namespace>

# 3. Git에 동일한 변경 커밋 & Push (반드시 수행)
git commit -m "hotfix: ..."
git push origin main

# 4. ArgoCD 자동 sync 복원
kubectl patch application <prefix>-apps -n argocd \
  --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

### SG ID 하드코딩 문제

현재 구조에서 `alb.ingress.kubernetes.io/security-groups` annotation에 SG ID가 하드코딩되어 있다. Terraform으로 SG를 재생성하면 ID가 변경되므로, 현업에서는 다음 패턴으로 해결한다.

```
[권장 패턴: Terraform → SSM → External Secret → Kustomize]

1. Terraform에서 SG 생성 후 SSM Parameter에 ID 저장
   resource "aws_ssm_parameter" "pub_alb_sg_id" {
     name  = "/<prefix>/sg/pub-alb-id"
     value = aws_security_group.pub_alb.id
   }

2. ExternalSecret으로 ConfigMap에 주입
   kind: ExternalSecret
   target:
     name: sg-ids-config
     template:
       data:
         pub-alb-sg-id: "{{ .pubAlbSgId }}"

3. Kustomize replacements로 Ingress에 주입
   replacements:
   - source:
       kind: ConfigMap
       name: sg-ids-config
       fieldPath: data.pub-alb-sg-id
     targets:
     - select:
         kind: Ingress
       fieldPaths:
       - metadata.annotations.[alb.ingress.kubernetes.io/security-groups]
```

### 현업에서 많이 쓰는 App of Apps 추가 패턴

**환경별 분기 (dev/staging/prod)**

```
kubernetes/
├── argocd/
│   ├── app-of-apps-dev.yaml    → targetRevision: dev
│   ├── app-of-apps-stg.yaml    → targetRevision: staging
│   └── app-of-apps-prod.yaml   → targetRevision: main
```

**ApplicationSet으로 멀티 클러스터 관리**

```yaml
# 클러스터가 여러 개일 때 하나의 ApplicationSet으로 모든 클러스터에 배포
kind: ApplicationSet
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: production
  template:
    spec:
      source:
        path: kubernetes/apps
```

---

## 11. 일상 운영 가이드

### 새 리소스 추가

```bash
# 1. 파일 생성
vi kubernetes/ingress/ingress-new-service.yaml

# 2. Git commit & push
git add kubernetes/ingress/ingress-new-service.yaml
git commit -m "feat(ingress): new-service 인그레스 추가"
git push origin main

# 3. ArgoCD 자동 감지 및 적용 (약 3분 이내)
kubectl get applications <prefix>-ingress -n argocd
# → Synced / Healthy 확인
```

### 새 레이어 추가

```bash
# 예: cert-manager 레이어 추가
vi kubernetes/argocd/application-cert-manager.yaml

git add kubernetes/argocd/application-cert-manager.yaml
git push origin main

# app-of-apps가 자동으로 새 Application 생성
```

### 리소스 삭제

```bash
# 파일 삭제 후 push하면 prune: true에 의해 클러스터에서도 자동 삭제
git rm kubernetes/ingress/ingress-old-service.yaml
git commit -m "chore(ingress): old-service 인그레스 제거"
git push origin main
```

### 동기화 상태 확인

```bash
# 전체 Application 상태 한눈에 확인
kubectl get applications -n argocd

# 특정 Application 상세 확인
kubectl describe application <prefix>-apps -n argocd

# 드리프트 감지 (OutOfSync 상태인 것 확인)
kubectl get applications -n argocd | grep -v Synced

# 수동 sync 트리거 (자동 sync 대기 없이 즉시 반영)
kubectl annotate application <prefix>-apps \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

## 12. 트러블슈팅

### app-of-apps가 Synced인데 하위 Application이 생성되지 않음

| 항목 | 내용 |
|---|---|
| 원인 | ArgoCD가 아직 git repo를 최신으로 fetch하지 않음 |
| 해결 | hard refresh 트리거 |

```bash
kubectl annotate application <prefix>-appsync-apne2-cluster \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

### CRD apply 시 "too long" 오류

| 항목 | 내용 |
|---|---|
| 원인 | `kubectl.kubernetes.io/last-applied-configuration` annotation 크기 초과 |
| 해결 | `application-storage.yaml`에 `ServerSideApply=true` syncOption 추가 |

---

### multi-namespace 리소스가 엉뚱한 namespace에 배포됨

| 항목 | 내용 |
|---|---|
| 원인 | 리소스 파일에 namespace가 명시되지 않아 `destination.namespace`로 배포됨 |
| 해결 | 각 리소스 파일에 `metadata.namespace` 명시 |

---

### selfHeal로 인해 긴급 패치가 되돌아감

| 항목 | 내용 |
|---|---|
| 원인 | selfHeal이 활성화된 상태에서 kubectl로 직접 수정 |
| 해결 | sync 일시 중단 → 패치 → Git 업데이트 → sync 복원 (위 긴급 패치 절차 참고) |

---

### 기존 Application 전환 시 리소스가 삭제됨

| 항목 | 내용 |
|---|---|
| 원인 | `--cascade=orphan` 없이 Application을 삭제 |
| 해결 | 리소스 재생성 필요. 이후에는 반드시 `kubectl delete application <name> -n argocd --cascade=orphan` 사용 |

---

---

## Phase 2 — 고아 Helm 릴리스 GitOps 전환 (2026-06-26)

---

## 14. 고아 Helm 릴리스란

Phase 1에서 `kubernetes/` 디렉토리의 manifest들을 ArgoCD로 관리하게 됐지만, 클러스터에 직접 `helm install`로 설치된 릴리스들은 여전히 ArgoCD의 GitOps 범위 밖에 있었다.

```bash
# 고아 상태 — GitOps 관리 없음
helm install argocd argo/argo-cd -n argocd
helm install kube-prometheus-stack ... -n monitoring
```

| 문제 | 설명 |
|---|---|
| **드리프트 감지 불가** | 클러스터 실제 values가 git과 얼마나 다른지 알 수 없음 |
| **변경 이력 없음** | 누가 언제 `helm upgrade`했는지 추적 불가 |
| **복구 절차 불명확** | 장애 시 어떤 values로 재설치해야 하는지 문서 없음 |
| **업그레이드 위험** | 이전 상태로 롤백할 기준이 없음 |

### 대상 클러스터 현황 (작업 전)

```bash
$ helm list -A

NAMESPACE          NAME                        CHART VERSION
argocd             argocd                      argo-cd-9.5.19              # 고아
external-secrets   external-secrets            external-secrets-2.5.0      # 고아
kube-system        aws-load-balancer-controller albc-3.3.0                 # 고아
kube-system        metrics-server              metrics-server-3.13.0       # 고아
monitoring         kube-prometheus-stack       kube-prometheus-stack-86.2.0 # 고아
monitoring         loki                        loki-7.0.0                  # 고아
monitoring         blackbox-exporter           blackbox-exporter-11.10.0   # 고아
monitoring         alloy                       alloy-1.10.0                # 고아
```

---

## 15. 이전 대상 선정 — 현업 표준 기준

모든 Helm 릴리스는 GitOps(ArgoCD)로 관리하는 것이 원칙이다. 단, 아래 예외 기준이 있다.

| 구분 | 관리 방식 | 근거 |
|---|---|---|
| **Karpenter Helm install** | Terraform 유지 | EKS 클러스터 생성 직후 의존성 (node 부팅 전) |
| **Karpenter CRs** (NodePool, EC2NodeClass) | ArgoCD 관리 | 노드 정책 변경은 GitOps로 감사 필요 |
| **기타 모든 Helm 릴리스** | ArgoCD 관리 | 드리프트 감지, 변경 이력, 자동 복구 |

```
✅ ArgoCD로 이전
  - argocd (self-management)
  - external-secrets-operator
  - aws-load-balancer-controller (ALBC)
  - metrics-server
  - kube-prometheus-stack
  - loki / alloy / blackbox-exporter

❌ 이전 제외 (Terraform 유지)
  - karpenter (helm install)
```

---

## 16. Multi-source Helm 패턴

### 개념

ArgoCD의 **multi-source** 기능을 사용하면 Helm chart repo와 values 파일을 별도 소스에서 가져올 수 있다.

```yaml
spec:
  sources:
    # Source 1: Helm chart (외부 레포)
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: "86.2.0"
      helm:
        releaseName: kube-prometheus-stack
        valueFiles:
          - $values/helm/monitoring/kube-prometheus-stack/values.yaml

    # Source 2: Values 파일 (자체 git 레포)
    - repoURL: git@github.com:<github-username>/<repo-name>.git
      targetRevision: main
      ref: values             # "$values" 변수로 참조
```

| 장점 | 설명 |
|---|---|
| **chart 버전 고정** | `targetRevision: "86.2.0"` — 의도치 않은 업그레이드 방지 |
| **values 버전 관리** | git에서 values 변경 이력 관리 |
| **분리된 관심사** | chart 업그레이드와 values 변경이 독립적 |
| **롤백 용이** | git revert로 values 변경 즉시 롤백 |

### 전환 절차

각 릴리스마다 다음 2단계를 수행했다.

```bash
# Step 1: 배포된 values 추출 (반드시 이 방법으로)
helm get values <release-name> -n <namespace> -o yaml > helm/<chart>/values.yaml
```

> **`helm get values`로 실제 배포 상태를 추출해야 한다.**<br>
> Helm chart의 default values를 사용하면 기존 클러스터 설정과 차이가 생겨 예상치 못한 변경이 발생한다.
{: .prompt-danger }

### Values 파일 디렉토리 구조

```
helm/
├── argocd/
│   └── values.yaml
├── alloy/
│   └── values.yaml
├── aws-load-balancer-controller/
│   └── values.yaml
├── external-secrets/
│   └── values.yaml
├── metrics-server/
│   └── values.yaml
└── monitoring/
    ├── blackbox-exporter/
    │   └── values.yaml
    ├── kube-prometheus-stack/
    │   └── values.yaml
    └── loki/
        └── values.yaml
```

---

## 17. Sync Wave 순서 재설계

Phase 2에서 Helm 릴리스가 추가되면서 Sync Wave 순서를 재설계했다.

```
Wave 1: metrics-server, external-secrets-operator, argocd (self)
  └── ESO가 먼저 떠야 ExternalSecret CR이 처리 가능
  └── ArgoCD self-management는 조기 wave에서 관리

Wave 2: ALBC
  └── ESO가 있어야 AWS 시크릿을 가져올 수 있음

Wave 3: ESO CRs (ExternalSecret, SecretStore)
  └── ESO Operator가 있어야 CRD가 존재함

Wave 4: kube-prometheus-stack
  └── 모니터링 스택은 기반 인프라 이후 설치

Wave 5: loki, alloy, blackbox-exporter
  └── Prometheus가 먼저 떠야 scrape target 등록 가능

Wave 6: apps (Deployment, HPA)
  └── 모든 인프라 완료 후 애플리케이션 배포
```

---

## 18. ArgoCD Self-management (자기 관리)

### 개념

ArgoCD가 자기 자신의 Helm 차트를 ArgoCD Application으로 관리하는 패턴이다. ArgoCD 자체 업그레이드, values 변경도 GitOps 흐름으로 통제된다.

```
git push (helm/argocd/values.yaml 수정)
  → App-of-Apps 감지
  → application-argocd.yaml 적용
  → ArgoCD가 자신의 Helm 차트를 sync
  → ArgoCD Deployment 롤링 업데이트
```

### argocd-secret — 다중 관리자 문제

`argocd-secret`은 세 주체가 동시에 관여한다.

| 주체 | 관여 방식 | 관리 필드 |
|---|---|---|
| ArgoCD Helm 차트 | Secret 생성 | 구조(키 목록) |
| ArgoCD 런타임 | 값 생성 | admin.password, server.secretkey, tls.crt/key |
| ESO (ExternalSecret) | creationPolicy: Merge | oidc.github.clientSecret |

> Helm 차트의 desired state와 실제 Secret 상태가 항상 다를 수밖에 없다.<br>
> `ignoreDifferences`로 런타임 생성 필드를 **구체적 경로**로 명시해야 한다.
{: .prompt-warning }

### application-argocd.yaml 최종 구성

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-argocd
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  sources:
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argo-cd
      targetRevision: "9.5.19"
      helm:
        releaseName: argocd
        valueFiles:
          - $values/helm/argocd/values.yaml
    - repoURL: git@github.com:<github-username>/<repo-name>.git
      targetRevision: main
      ref: values
  destination:
    namespace: argocd
  ignoreDifferences:
    - group: ""
      kind: Secret
      name: argocd-secret
      namespace: argocd
      jqPathExpressions:
        # kubectl apply 잔존 annotation
        - '.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]'
        # ESO reconcile hash
        - '.metadata.annotations["reconcile.external-secrets.io/data-hash"]'
        # ESO ExternalSecret tracking-id
        - '.metadata.annotations["argocd.argoproj.io/tracking-id"]'
        # ArgoCD 런타임 생성 값
        - '.data["admin.password"]'
        - '.data["admin.passwordMtime"]'
        - '.data["server.secretkey"]'
        # ESO가 주입하는 값
        - '.data["oidc.github.clientSecret"]'
        # ArgoCD TLS 런타임 인증서
        - '.data["tls.crt"]'
        - '.data["tls.key"]'
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - ServerSideApply=true
```

> **안티패턴 — 절대 하지 말 것**<br>
> `.metadata.annotations` 전체나 `.data` 전체를 무시하면 의도치 않은 변경도 감지하지 못하게 된다.<br>
> 반드시 구체적인 경로(`'.data["admin.password"]'`)만 지정해야 한다.
{: .prompt-danger }

---

## 19. 트러블슈팅 — Phase 2

### App-of-Apps 지속적 OutOfSync

**원인 1: `kustomize: {}` 빈 필드**

`application-apps.yaml`에 아무 설정도 없는 `kustomize: {}` 필드가 있었다. live Application CR에는 이 필드가 없어서 diff 발생.

```yaml
# 수정 전 (diff 원인)
source:
  path: kubernetes/apps
  kustomize: {}   # ← 빈 값이지만 live CR에 없어서 diff

# 수정 후
source:
  path: kubernetes/apps
```

**원인 2: `last-applied-configuration` annotation**

App-of-Apps가 client-side apply를 사용하는 동안 annotation이 Application CR에 설정됐다. SSA로 전환하면 이 annotation이 새로 설정되지는 않지만, 이미 존재하는 annotation이 git 파일에는 없으므로 diff로 감지된다.

```yaml
# app-of-apps.yaml에 추가
ignoreDifferences:
  - group: argoproj.io
    kind: Application
    jqPathExpressions:
      - .metadata.annotations["argocd.argoproj.io/tracking-id"]
      - .metadata.annotations["argocd.argoproj.io/sync-requested"]
      - .metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

---

### kube-prometheus-stack ComparisonError

```
ComparisonError: failed to calculate diff:
  error building typed value from config resource:
  .spec.hostNetwork: field not declared in schema
```

**원인 분석**

`monitoring.coreos.com/v1` 그룹의 `Prometheus`와 `Alertmanager` CRD가 `spec.hostNetwork: false` 필드를 포함한다. ArgoCD v3.x의 번들 스키마에 이 필드가 선언되지 않아 diff 계산 단계에서 exception이 발생한다.

> **`ignoreDifferences` 단독으로는 해결 불가**<br>
> `ignoreDifferences`는 diff *결과*를 무시하지만, diff *계산* 자체가 실패하면 효과가 없다.<br>
> `RespectIgnoreDifferences=true`를 함께 설정해야 비교 단계(comparison phase)에서도 무시가 적용된다.
{: .prompt-warning }

```yaml
ignoreDifferences:
  - group: apps
    kind: DaemonSet
    jqPathExpressions:
      - .spec.template.spec.hostNetwork
      - .spec.template.spec.containers[].ports[].hostPort
  - group: monitoring.coreos.com
    kind: Prometheus
    jqPathExpressions:
      - .spec.hostNetwork
  - group: monitoring.coreos.com
    kind: Alertmanager
    jqPathExpressions:
      - .spec.hostNetwork
  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    jqPathExpressions:
      - .webhooks[]?.clientConfig.caBundle
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    jqPathExpressions:
      - .webhooks[]?.clientConfig.caBundle
syncPolicy:
  syncOptions:
    - ServerSideApply=true
    - RespectIgnoreDifferences=true   # 비교 단계에서도 무시 적용
```

---

### external-secrets-operator CRD annotation 크기 초과

```
CustomResourceDefinition "clustersecretstores.external-secrets.io"
is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

**원인 분석**

```bash
# CRD 크기 측정
clustersecretstores.external-secrets.io: 335KB
secretstores.external-secrets.io:        334KB
```

Kubernetes의 annotation 크기 제한은 **262144 bytes (256KB)** 다. Client-side apply는 `kubectl.kubernetes.io/last-applied-configuration` annotation에 전체 리소스 JSON을 저장하므로 335KB CRD는 무조건 초과한다.

> **Server-side apply는 `last-applied-configuration`을 사용하지 않는다.**<br>
> 대신 `managedFields`로 소유권을 추적하므로 annotation 크기 제한을 우회할 수 있다.<br>
> CRD가 있는 차트(`external-secrets`, `kube-prometheus-stack`, `cert-manager` 등)는 모두 `ServerSideApply=true`를 설정해야 한다.
{: .prompt-info }

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

> **SSA 전환 후 첫 sync는 수동 트리거 필요**<br>
> Application CR 업데이트 후 ArgoCD가 "Synced" 상태를 유지하면 `selfHeal`이 작동하지 않는다. 직접 sync operation을 주입해야 한다.
{: .prompt-warning }

```bash
kubectl patch application <prefix>-external-secrets-operator -n argocd \
  --type merge -p '{
    "operation": {
      "initiatedBy": {"username": "admin"},
      "sync": {
        "revision": "HEAD",
        "syncOptions": ["CreateNamespace=true","ServerSideApply=true"]
      }
    }
  }'
```

**SSA가 필요한 차트 목록:**

| 차트 | 이유 |
|---|---|
| kube-prometheus-stack | 대형 CRD + MutatingWebhookConfiguration |
| external-secrets-operator | ~335KB CRD (clustersecretstores, secretstores) |
| cert-manager | WebhookConfiguration |
| ArgoCD self | 대형 Deployment spec |

---

## 20. 최종 상태 검증

```bash
kubectl get applications -n argocd \
  -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status"

NAME                                          SYNC     HEALTH
<prefix>-albc                                 Synced   Healthy
<prefix>-alloy                                Synced   Healthy
<prefix>-apps                                 Synced   Healthy
<prefix>-appsync-apne2-cluster                Synced   Healthy   ← App-of-Apps
<prefix>-argocd                               Synced   Healthy   ← Self-management
<prefix>-blackbox-exporter                    Synced   Healthy
<prefix>-external-secrets                     Synced   Healthy
<prefix>-external-secrets-operator            Synced   Healthy
<prefix>-ingress                              Synced   Healthy
<prefix>-karpenter                            Synced   Healthy
<prefix>-kube-prometheus-stack                Synced   Healthy
<prefix>-loki                                 Synced   Healthy
<prefix>-metrics-server                       Synced   Healthy
<prefix>-monitoring                           Synced   Healthy
<prefix>-storage                              Synced   Healthy
<prefix>-valkey                               Synced   Healthy

# 전체 Synced + Healthy ✅
```

### 현업 표준 체크리스트

| 항목 | 상태 |
|---|:---:|
| App-of-Apps 패턴 (`application-*.yaml` glob 자동 감지) | ✅ |
| Multi-source Helm (chart 버전 고정 + git values 분리) | ✅ |
| `ignoreDifferences` 최소 범위 (전체 무시 대신 구체적 경로만 지정) | ✅ |
| `ServerSideApply=true` (대형 차트 CRD 포함 전체 적용) | ✅ |
| `RespectIgnoreDifferences=true` (CRD 스키마 미선언 필드 오류 방지) | ✅ |
| Sync Wave 순서 (의존성 순서 보장 1→2→3→4→5→6) | ✅ |
| `automated.selfHeal + prune` (드리프트 자동 복구, 고아 리소스 정리) | ✅ |
| Karpenter Terraform 유지 (helm install은 Terraform, CRs는 ArgoCD) | ✅ |
| values 파일 git 관리 (`helm get values`로 실 배포 상태 추출 후 커밋) | ✅ |

---

## 21. 향후 개선 로드맵

**보안 강화 (HIGH)**

> **securityContext 미설정**<br>
> 현재 api, nextjs Deployment 모두 `podSecurityContext` 및 `containerSecurityContext` 미설정.<br>
> `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false` 적용 필요.<br>
> 단, `readOnlyRootFilesystem` 적용 전 앱이 쓰는 경로를 먼저 확인하고 `emptyDir` 볼륨으로 마운트해야 한다.
{: .prompt-danger }

> **NetworkPolicy 부재**<br>
> 클러스터 전체에 NetworkPolicy가 거의 없다. 네임스페이스 default-deny ingress 정책 + 필요한 통신만 허용하는 방식으로 설계 필요.<br>
> 잘못 설계 시 Prometheus scrape 단절, ArgoCD 접근 불가, 앱 간 통신 단절이 발생할 수 있어 위험도가 높다.
{: .prompt-warning }

**운영 개선 (MEDIUM)**

> - ArgoCD Image Updater 도입: CI에서 새 이미지 태그 push 시 Deployment 자동 업데이트<br>
> - ApplicationSet 도입: dev/staging/prod 환경 분기 자동화<br>
> - main 브랜치 보호 규칙: PR 필수 + 최소 1명 승인 + CI 통과 후 merge<br>
> - SG ID 변수화: Terraform → SSM → External Secret → Kustomize 패턴 적용
{: .prompt-tip }

---

## 참고

- [ArgoCD App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [ArgoCD Multiple Sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
- [ArgoCD ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [GitOps Principles](https://opengitops.dev/)
