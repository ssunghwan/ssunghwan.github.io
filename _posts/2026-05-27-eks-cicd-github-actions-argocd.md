---
title: "Building a GitHub Actions + ArgoCD GitOps CI/CD Pipeline"
date: 2026-05-27 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, argocd, gitops, github-actions, cicd, alb, karpenter, hpa, oidc, dex, oauth]
---

> EKS 마이그레이션 시리즈 여섯 번째 포스팅이다.<br>
> 앞선 포스팅에서 Next.js 샘플 애플리케이션을 EKS에 수동 배포하는 것까지 완료했다. 이번에는 코드 변경이 자동으로 배포되는 완전한 CI/CD 파이프라인을 구축한다.<br>
> GitHub Actions로 이미지를 빌드해 ECR에 푸시하고, ArgoCD가 Git 변경을 감지해 EKS에 자동 배포한다. 여기에 HPA, Karpenter 연동, ALB Ingress Group, GitHub OAuth까지 실제 현장에서 겪은 트러블슈팅을 그대로 담았다.
{: .prompt-info }

---

## 전체 아키텍처

```
개발자 코드 수정 (applications/app-name/**)
  │
  ▼ git push (main 브랜치)
GitHub Actions CI
  ├── AWS OIDC 인증 (IAM Role)
  ├── Docker build
  ├── ECR push (날짜+SHA 태그)
  └── kubernetes/apps/deployment-*.yaml image tag 업데이트 → git push
           │
           ▼ Git 변경 감지
        ArgoCD
           │ 자동 Sync
           ▼
     EKS 클러스터
           │
           ├── Rolling Update (다운타임 없음)
           ├── HPA (CPU/Memory 기반 자동 스케일링)
           └── Karpenter (노드 자동 프로비저닝)
```

---

## 1. 레포지토리 구조

```
infra-repo/
├── applications/
│   ├── purina-nextjs/        # Next.js 앱 소스 + Dockerfile
│   └── purina-api/           # Node.js API 소스 + Dockerfile
├── kubernetes/
│   ├── apps/
│   │   ├── deployment/
│   │   │   ├── deployment-nextjs.yaml
│   │   │   └── deployment-api.yaml
│   │   └── HorizontalPodAutoscaler/
│   │       ├── hpa-nextjs.yaml
│   │       └── hpa-api.yaml
│   ├── argocd/
│   │   ├── application.yaml              # ArgoCD Application 정의
│   │   └── repo-secret.yaml              # Git 레포 인증 (.gitignore)
│   ├── external-secrets/
│   │   └── external-secret-argocd.yaml
│   └── ingress/
│       ├── ingress.yaml                  # 앱 Ingress
│       └── ingress-argocd.yaml           # ArgoCD Ingress
├── helm/
│   └── argocd/
│       └── values.yaml
├── terraform/
└── .github/
    └── workflows/
        ├── gitactions-nextjs-pipe.yml
        └── gitactions-api-pipe.yml
```

> **모노레포 구조를 선택한 이유**<br>
> 앱 소스, K8s 매니페스트, Terraform, CI 워크플로우를 한 곳에서 관리한다.<br>
> 인프라 변경과 앱 변경 이력이 동일한 Git 히스토리에 남아 추적이 쉽고, ArgoCD가 `kubernetes/apps/` 디렉토리를 감시하므로 매니페스트 변경이 자동 배포를 트리거한다.
{: .prompt-tip }

---

## 2. GitHub Actions OIDC IAM Role (Terraform)

GitHub Actions에서 AWS ECR에 이미지를 푸시하려면 AWS 자격증명이 필요하다.

> **Access Key 대신 OIDC를 사용하는 이유**<br>
> Access Key를 GitHub Secrets에 저장하면 키 노출 위험이 있고, 만료 시 수동으로 교체해야 한다.<br>
> OIDC를 사용하면 GitHub Actions가 실행될 때마다 AWS에서 임시 자격증명을 발급받아 사용한다.<br>
> 장기 자격증명이 존재하지 않으므로 노출 자체가 불가능하다.
{: .prompt-warning }

```hcl
# modules/github-actions/main.tf

locals {
  github_oidc_url = "token.actions.githubusercontent.com"
}

# GitHub OIDC Provider 등록
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://${local.github_oidc_url}"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# IAM Role — GitHub Actions가 AssumeRole할 대상
resource "aws_iam_role" "github_actions" {
  name = "${var.prefix}-gha-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          # aud: GitHub Actions가 발급하는 토큰의 대상
          "${local.github_oidc_url}:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # sub: 특정 계정의 모든 레포, main 브랜치만 허용
          "${local.github_oidc_url}:sub" = "repo:<github-username>/*:ref:refs/heads/main"
        }
      }
    }]
  })
}

# ECR Push 권한만 부여 (최소 권한 원칙)
resource "aws_iam_role_policy" "github_actions_ecr" {
  name = "${var.prefix}-gha-apne2-ecr-policy"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # GetAuthorizationToken은 리소스 레벨 제어 불가 → * 불가피
        Effect   = "Allow"
        Action   = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        # 특정 ECR 레포지토리만 허용 (최소 권한)
        Resource = [
          "arn:aws:ecr:ap-northeast-2:<account-id>:repository/app-nextjs",
          "arn:aws:ecr:ap-northeast-2:<account-id>:repository/app-api"
        ]
      }
    ]
  })
}
```

> **IAM 설계 포인트**<br>
> `StringLike`로 와일드카드 사용 → 여러 레포에서 동일 Role 재사용 가능<br>
> `ref:refs/heads/main` → main 브랜치 push 시에만 인증 허용 (feature 브랜치 제외)<br>
> ECR 권한을 특정 레포지토리 ARN으로 제한 → 다른 레포의 이미지는 건드릴 수 없음
{: .prompt-info }

---

## 3. GitHub Actions CI 워크플로우

> **워크플로우를 앱별로 분리한 이유**<br>
> 하나의 워크플로우에 두 job을 넣으면 `applications/app-nextjs/**` 변경 시 api job도 트리거된다.<br>
> `paths` 필터는 워크플로우 레벨에만 적용되고 job 레벨의 `if` 조건은 정확히 매칭하지 못하기 때문이다.<br>
> 워크플로우를 완전히 분리하면 각 앱 경로 변경 시에만 해당 빌드가 트리거되어 불필요한 빌드를 방지한다.
{: .prompt-tip }

```yaml
# .github/workflows/gitactions-nextjs-pipe.yml
name: CI - Build and Push app-nextjs

on:
  push:
    branches: [main]
    paths:
      - 'applications/app-nextjs/**'   # 이 경로 변경 시에만 트리거

env:
  AWS_REGION: ap-northeast-2
  AWS_ACCOUNT_ID: "<account-id>"
  ROLE_ARN: arn:aws:iam::<account-id>:role/<prefix>-gha-apne2-role

jobs:
  build-nextjs:
    name: Build & Push app-nextjs
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # OIDC 토큰 발급에 필요
      contents: write    # manifest 업데이트 후 git push에 필요

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and Push
        id: build
        run: |
          # 날짜+SHA 7자리 형식 (예: 20260527-a9d63e3)
          IMAGE_TAG=$(date +%Y%m%d)-$(echo $GITHUB_SHA | cut -c1-7)
          IMAGE_URI=${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/app-nextjs:${IMAGE_TAG}
          docker build -t $IMAGE_URI applications/app-nextjs/
          docker push $IMAGE_URI
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT

      - name: Update image tag in manifest
        run: |
          IMAGE_TAG=${{ steps.build.outputs.image_tag }}
          # deployment yaml의 image 태그를 새 태그로 교체
          sed -i "s|<account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nextjs:.*|<account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nextjs:${IMAGE_TAG}|" \
            kubernetes/apps/deployment/deployment-nextjs.yaml
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add kubernetes/apps/deployment/deployment-nextjs.yaml
          git commit -m "ci: update app-nextjs image tag to ${IMAGE_TAG}"
          # 다른 CI 워크플로우가 먼저 push했을 경우를 대비해 rebase 후 push
          git pull --rebase origin main
          git push
```

### 이미지 태그 전략

| 방식 | 예시 | 장단점 |
|---|---|---|
| SHA 7자리 | `a9d63e3` | 간결하지만 날짜 파악 불가 |
| **날짜+SHA** | `20260527-a9d63e3` | 언제 빌드됐는지 즉시 파악 가능 ✅ |
| 시맨틱 버전 | `v1.0.3` | Git tag 기반, 수동 관리 필요 |

> **날짜+SHA를 선택한 이유**<br>
> ECR 콘솔에서 이미지 생성 날짜를 별도로 확인하지 않아도 태그만으로 파악 가능하다.<br>
> 장애 발생 시 "언제 빌드된 이미지인지"를 즉시 알 수 있어 롤백 대상을 빠르게 특정할 수 있다.<br>
> `git pull --rebase` 는 두 워크플로우가 동시에 같은 브랜치에 push할 때 충돌을 방지한다.
{: .prompt-tip }

---

## 4. ArgoCD Application 등록

### Private 레포 SSH 인증

레포지토리가 Private이므로 ArgoCD가 Git에 접근하려면 인증이 필요하다.

> **HTTPS 대신 SSH를 사용하는 이유**<br>
> HTTPS는 Personal Access Token이 필요하고 만료 기간이 있다.<br>
> SSH 키는 만료가 없고, Deploy Key로 특정 레포에만 읽기 권한을 부여할 수 있어 최소 권한 원칙에 부합한다.
{: .prompt-info }

```yaml
# kubernetes/argocd/repo-secret.yaml (.gitignore에 반드시 추가!)
apiVersion: v1
kind: Secret
metadata:
  name: argocd-repo-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository  # ArgoCD가 레포 인증 시크릿으로 인식
stringData:
  type: git
  url: git@github.com:<github-username>/<repo-name>.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

```bash
# SSH private key가 포함되어 있으므로 반드시 gitignore에 추가
echo "kubernetes/argocd/repo-secret.yaml" >> .gitignore
```

### ArgoCD Application

```yaml
# kubernetes/argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-argocd-apne2-pipe
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<github-username>/<repo-name>.git
    targetRevision: main          # 감시할 브랜치
    path: kubernetes/apps         # 감시할 디렉토리
    directory:
      recurse: true               # 하위 디렉토리까지 재귀 탐색
  destination:
    server: https://kubernetes.default.svc  # 현재 클러스터
    namespace: <app-namespace>
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스는 클러스터에서도 삭제
      selfHeal: true    # 클러스터 상태가 Git과 다르면 자동으로 원복
    syncOptions:
      - CreateNamespace=false   # 네임스페이스는 자동 생성하지 않음
```

> **`recurse: true`가 반드시 필요한 이유**<br>
> 이 옵션이 없으면 `kubernetes/apps/` 바로 아래 파일만 감시한다.<br>
> `deployment/`, `HorizontalPodAutoscaler/` 같은 하위 디렉토리의 파일을 감시하려면 반드시 필요하다.
{: .prompt-danger }

> **`prune: true` + `selfHeal: true` 조합**<br>
> `prune`: Git에서 매니페스트를 삭제하면 클러스터에서도 자동으로 삭제된다. 고아 리소스가 남지 않는다.<br>
> `selfHeal`: 누군가 `kubectl`로 직접 클러스터를 수정해도 ArgoCD가 Git 상태로 되돌린다. Git이 유일한 진실이 된다.
{: .prompt-info }

---

## 5. Deployment 매니페스트 — topologySpreadConstraints

### podAntiAffinity를 버린 이유

초기에는 `podAntiAffinity`로 AZ 분산을 설정했으나 Karpenter와 심각한 충돌이 발생했다.

```yaml
# Before — Karpenter와 충돌 발생
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: <prefix>-nextjs
      topologyKey: topology.kubernetes.io/zone
```

> **문제 원인**<br>
> Karpenter는 새 노드를 프로비저닝하기 전 시뮬레이션을 수행하는데, `required` 설정 시 시뮬레이션 단계에서 hard constraint처럼 처리한다.<br>
> 기존 Pod가 2a/2c에 각 1개씩 있으면, 롤링 업데이트 중 surge Pod를 놓을 AZ가 없다고 판단해 영구적으로 Pending 상태에 빠진다.
{: .prompt-danger }

### topologySpreadConstraints로 교체

```yaml
# kubernetes/apps/deployment/deployment-nextjs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <prefix>-nextjs-apne2-deploy
  namespace: <app-namespace>
spec:
  replicas: 2
  selector:
    matchLabels:
      app: <prefix>-nextjs
  template:
    metadata:
      labels:
        app: <prefix>-nextjs
    spec:
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
      - maxSkew: 1                                      # AZ 간 Pod 수 최대 차이
        topologyKey: topology.kubernetes.io/zone        # AZ 기준으로 분산
        whenUnsatisfiable: ScheduleAnyway               # 불가능하면 같은 AZ도 허용
        labelSelector:
          matchLabels:
            app: <prefix>-nextjs
      tolerations:
      - key: dedicated
        value: web
        effect: NoSchedule
      nodeSelector:
        role: web
      containers:
      - name: nextjs
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nextjs:1.0.0
        ports:
        - containerPort: 3000
        envFrom:
        - secretRef:
            name: mysql-secret
        env:
        - name: NODE_ENV
          value: "production"
        - name: SESSION_SECRET
          valueFrom:
            secretKeyRef:
              name: nextjs-secret
              key: SESSION_SECRET
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 3
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
```

### podAntiAffinity vs topologySpreadConstraints 비교

| 항목 | podAntiAffinity | topologySpreadConstraints |
|---|---|---|
| 기반 | Pod 간 관계 | 전체 Pod 분산 균형 |
| Karpenter 호환성 | 시뮬레이션 단계에서 hard constraint로 처리 | 호환성 좋음 ✅ |
| 롤링 업데이트 | surge Pod 스케줄 실패 가능 | 유연하게 허용 ✅ |
| 현업 권장 | Karpenter 미사용 환경 | **Karpenter 환경** ✅ |

> **`ScheduleAnyway` vs `DoNotSchedule`**<br>
> `ScheduleAnyway`: 가능하면 AZ 분산, 불가능하면 같은 AZ도 허용. 롤링 업데이트 중 surge Pod에 적합.<br>
> `DoNotSchedule`: 반드시 AZ 분산. ArgoCD Server처럼 HA가 중요한 시스템 컴포넌트에 적합.
{: .prompt-tip }

---

## 6. HPA (HorizontalPodAutoscaler)

> **HPA와 Karpenter의 역할 분리**<br>
> HPA는 **Pod 수**를 조절하고, Karpenter는 **노드 수**를 조절한다. 두 컴포넌트가 함께 동작해야 완전한 자동 스케일링이 된다.<br>
> HPA가 replicas를 늘려도 기존 노드에 리소스가 없으면 Pod는 Pending 상태가 된다. 이때 Karpenter가 새 노드를 자동으로 프로비저닝한다.
{: .prompt-info }

```yaml
# kubernetes/apps/HorizontalPodAutoscaler/hpa-nextjs.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <prefix>-nextjs-apne2-hpa
  namespace: <app-namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <prefix>-nextjs-apne2-deploy
  minReplicas: 2     # 최소 2개 (AZ 분산 보장)
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # CPU 60% 초과 시 스케일 아웃
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70    # Memory 70% 초과 시 스케일 아웃
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60     # 스케일 아웃 전 60초 안정화 대기
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60                # 60초마다 최대 2개씩 증가
    scaleDown:
      stabilizationWindowSeconds: 300    # 스케일 인 전 300초 안정화 대기 (급격한 축소 방지)
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120               # 120초마다 최대 1개씩 감소
```

> **`behavior` 설정을 세밀하게 조정한 이유**<br>
> **scaleUp**: 트래픽 급증 시 빠르게 대응. 60초 대기 후 60초마다 2개씩 증가.<br>
> **scaleDown**: 급격한 축소를 방지. 300초(5분) 안정화 후 120초마다 1개씩만 감소.<br>
> 이커머스는 프로모션 종료 후 트래픽이 급감하는데, 너무 빠르게 스케일 인하면 다음 트래픽 급증 시 대응이 늦어진다.
{: .prompt-warning }

### Karpenter + HPA 연동 흐름

```
트래픽 증가
  → HPA: replicas 2 → 4 → 6 (CPU/Memory 기준)
  → 기존 노드에 Pod 스케줄 불가 (리소스 부족)
  → Karpenter: 새 노드 자동 프로비저닝 (~30초)
  → topologySpreadConstraints: 새 노드에 AZ 분산 배포

트래픽 감소
  → HPA: replicas 6 → 4 → 2 (300초 안정화 후)
  → Karpenter: 빈 노드 30초 후 자동 제거 (consolidation)
```

---

## 7. ALB Ingress Group — 하나의 ALB로 여러 서비스 분리

> **Ingress Group을 사용하는 이유**<br>
> 앱과 ArgoCD를 별도 ALB로 운영하면 ALB 비용이 두 배가 된다. (ALB는 최소 시간당 약 $0.008)<br>
> `group.name`이 동일한 Ingress는 같은 ALB를 공유하며, ALB 리스너 규칙에서 Host 헤더로 트래픽을 분리한다.
{: .prompt-tip }

```
ALB (<prefix>-pub-apne2-alb)
  HTTPS:443 리스너
    규칙 1: Host = app.example.com    → nextjs Target Group
    규칙 2: Host = argocd.example.com → ArgoCD Target Group
    기본: 404 반환
```

### 앱 Ingress

```yaml
# kubernetes/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: <app-namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/load-balancer-name: <prefix>-pub-apne2-alb
    alb.ingress.kubernetes.io/group.name: <prefix>-pub-alb   # 그룹 식별자
    alb.ingress.kubernetes.io/healthcheck-path: /api/health
    external-dns.alpha.kubernetes.io/hostname: app.example.com
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <prefix>-nextjs-apne2-svc
            port:
              number: 80
```

### ArgoCD Ingress

```yaml
# kubernetes/ingress/ingress-argocd.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/load-balancer-name: <prefix>-pub-apne2-alb
    alb.ingress.kubernetes.io/group.name: <prefix>-pub-alb   # 앱과 동일한 그룹
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/success-codes: "200,307"
    external-dns.alpha.kubernetes.io/hostname: argocd.example.com
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

> **`group.name`이 달라지면 어떻게 되는가?**<br>
> AWS ALB 이름이 같아도 `group.name`이 다르면 ALBC는 새 ALB를 생성하려 한다.<br>
> 이미 같은 이름의 ALB가 있으면 `DuplicateLoadBalancerName` 에러가 발생한다.<br>
> 같은 ALB를 공유하려면 반드시 동일한 `group.name`을 사용해야 한다.
{: .prompt-danger }

> **`success-codes: "200,307"` 설정 이유**<br>
> ArgoCD server는 `server.insecure: true` 설정에서 일부 요청에 307 리다이렉트를 반환한다.<br>
> ALB 헬스체크가 307을 실패로 처리하면 ArgoCD 타겟이 Unhealthy로 처리되므로 200과 307 모두 성공으로 처리한다.
{: .prompt-info }

---

## 8. ArgoCD GitHub OAuth (Dex)

ArgoCD는 Dex를 내장 IdP로 사용해 외부 OAuth Provider와 연동한다. GitHub OAuth App을 만들어 개발자가 GitHub 계정으로 ArgoCD에 로그인할 수 있게 한다.

### GitHub OAuth App 생성

```
GitHub → Settings → Developer settings → OAuth Apps → New OAuth App

Application name: <prefix>-argocd
Homepage URL: https://argocd.example.com
Authorization callback URL: https://argocd.example.com/api/dex/callback
```

> `/api/dex/callback`이 Dex가 OAuth 코드를 받는 엔드포인트다. 이 경로가 정확하지 않으면 인증 후 리다이렉트가 실패한다.
{: .prompt-warning }

### Client Secret을 AWS Secrets Manager에 저장

```hcl
# terraform/envs/dev/main.tf
resource "aws_secretsmanager_secret" "argocd_github_oauth" {
  name = "<prefix>-github-oauth-apne2-secret"
}

resource "aws_secretsmanager_secret_version" "argocd_github_oauth" {
  secret_id = aws_secretsmanager_secret.argocd_github_oauth.id
  secret_string = jsonencode({
    clientID     = "<github-oauth-client-id>"
    clientSecret = var.argocd_github_oauth_client_secret  # terraform.secret.tfvars에서 주입
  })
}
```

### External Secrets로 K8s Secret 주입

```yaml
# kubernetes/external-secrets/external-secret-argocd.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-github-oauth-secret
  namespace: argocd
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: argocd-secret
    creationPolicy: Merge    # 기존 argocd-secret에 값 추가 (덮어쓰지 않음)
  data:
  - secretKey: oidc.github.clientSecret
    remoteRef:
      key: <prefix>-github-oauth-apne2-secret
      property: clientSecret
```

> **`creationPolicy: Merge`가 반드시 필요한 이유**<br>
> ArgoCD의 `argocd-secret`은 이미 존재하는 시크릿으로, 내부적으로 ArgoCD가 사용하는 키들이 있다.<br>
> `Replace`로 설정하면 기존 ArgoCD 내부 시크릿이 덮어씌워져 ArgoCD가 정상 동작하지 않는다.<br>
> `Merge`는 기존 키는 유지하고 `oidc.github.clientSecret`만 추가한다.
{: .prompt-danger }

### ArgoCD Helm values

```yaml
# helm/argocd/values.yaml
configs:
  cm:
    url: https://argocd.example.com
    dex.config: |
      connectors:
        - type: github
          id: github
          name: GitHub
          config:
            clientID: <github-oauth-client-id>
            clientSecret: $oidc.github.clientSecret   # argocd-secret에서 참조
            # orgs: GitHub Organization 멤버만 허용할 경우 추가
            # 개인 계정 사용 시 제거
            # orgs:
            #   - name: your-github-org
  rbac:
    policy.default: role:readonly    # 기본은 읽기 전용
    policy.csv: |
      g, <github-username>, role:admin   # 특정 사용자에게 admin 권한

  params:
    server.insecure: "true"    # ALB에서 TLS 종료 후 HTTP로 백엔드 통신

server:
  replicas: 2    # HA 구성
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule    # ArgoCD는 반드시 AZ 분산
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: argocd-server
```

> **`server.insecure: true` 설정 이유**<br>
> ALB에서 HTTPS를 종료하고 백엔드(ArgoCD server)와는 HTTP로 통신한다.<br>
> ArgoCD server가 HTTPS를 강제하면 ALB 헬스체크가 실패하므로 insecure 모드로 설정한다.<br>
> 외부 구간(인터넷 → ALB)은 HTTPS로 보호되므로 보안상 문제없다.
{: .prompt-info }

---

## 9. 트러블슈팅

### CI 워크플로우 paths 필터 — 앱별 분리 필요

| 항목 | 내용 |
|---|---|
| 증상 | `applications/app-nextjs/**` 변경 시 api 빌드도 트리거됨 |
| 원인 | `paths` 필터는 워크플로우 레벨에만 적용되고 job 레벨 `if` 조건은 정확히 매칭하지 못함 |
| 해결 | 워크플로우 파일을 앱별로 완전히 분리 |

```
.github/workflows/
├── gitactions-nextjs-pipe.yml  # paths: applications/app-nextjs/**
└── gitactions-api-pipe.yml     # paths: applications/app-api/**
```

---

### 롤링 업데이트 중 Pod Pending — podAntiAffinity 충돌

| 항목 | 내용 |
|---|---|
| 증상 | 이미지 태그 변경 후 새 Pod가 Pending 상태에서 벗어나지 못함 |
| 원인 | `podAntiAffinity: required` 설정 시 Karpenter 시뮬레이션 단계에서 hard constraint로 처리. 기존 Pod가 2a/2c에 각 1개씩 있으면 surge Pod를 놓을 AZ가 없다고 판단 |
| 해결 | `topologySpreadConstraints` + `ScheduleAnyway`로 교체 |

---

### ArgoCD selfHeal로 인한 구 ReplicaSet 부활

| 항목 | 내용 |
|---|---|
| 증상 | 구 ReplicaSet Pod를 삭제해도 ArgoCD selfHeal이 계속 살려냄 |
| 원인 | deployment yaml의 image tag가 구버전을 가리키고 있어 ArgoCD가 구 RS를 desired 상태로 인식 |
| 해결 | CI가 manifest를 업데이트하고 push한 후 로컬에서 pull하지 않아서 발생. ArgoCD Sync 상태와 클러스터의 실제 image tag를 동시에 확인 |

```bash
# ArgoCD가 바라보는 Git revision 확인
kubectl get application <prefix>-argocd-apne2-pipe -n argocd \
  -o jsonpath='{.status.sync.revision}'

# 클러스터에 실제 적용된 image tag 확인
kubectl get deployment <prefix>-nextjs-apne2-deploy \
  -n <app-namespace> \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

### GitHub OAuth — org 설정 오류

```
user "<github-username>" not in required orgs or teams
```

| 항목 | 내용 |
|---|---|
| 원인 | `orgs: [{name: <github-username>}]`을 설정했는데 개인 계정은 GitHub Organization이 아님 |
| 해결 | 개인 계정 사용 시 `orgs` 블록 제거. 조직 계정으로 전환 시 추가 |

```yaml
# 개인 계정 사용 시 → orgs 블록 없음
config:
  clientID: <client-id>
  clientSecret: $oidc.github.clientSecret

# GitHub Organization 사용 시
config:
  clientID: <client-id>
  clientSecret: $oidc.github.clientSecret
  orgs:
    - name: <your-github-org>
```

---

## 10. 최종 검증

### CI/CD 파이프라인 동작 확인

```bash
# 1. 앱 코드 변경 후 push
echo "# trigger" >> applications/app-nextjs/README.md
git add applications/app-nextjs/README.md
git commit -m "feat: trigger CI test"
git push

# 2. GitHub Actions에서 빌드 성공 확인
# → ECR에 날짜+SHA 태그 이미지 푸시
# → deployment-nextjs.yaml image tag 업데이트 커밋

# 3. ArgoCD Sync 확인
kubectl get application <prefix>-argocd-apne2-pipe -n argocd

# 4. 롤링 업데이트 확인
kubectl get pods -n <app-namespace> -w

# 5. 새 이미지 태그 확인
kubectl get deployment <prefix>-nextjs-apne2-deploy \
  -n <app-namespace> \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### HPA 확인

```bash
kubectl get hpa -n <app-namespace>

# NAME                   REFERENCE                TARGETS           MINPODS  MAXPODS  REPLICAS
# <prefix>-nextjs-hpa    Deployment/<prefix>-..   cpu: 5%/60%       2        10       2
```

---

## 마치며

이번 포스팅에서 구축한 CI/CD 파이프라인의 핵심은 **GitOps 원칙**이다. 클러스터 상태의 유일한 진실은 Git이고, ArgoCD는 Git과 클러스터를 항상 동기화한다. 개발자는 코드만 push하면 나머지는 파이프라인이 처리한다.

트러블슈팅 과정에서 가장 많은 시간을 소비한 부분은 Karpenter와 `podAntiAffinity`의 충돌이었다. **Karpenter 환경에서는 `topologySpreadConstraints`가 표준**이라는 점, 그리고 `ScheduleAnyway`와 `DoNotSchedule`의 차이를 명확히 이해하고 상황에 맞게 선택하는 것이 중요하다.

아직 개발 환경에서의 테스팅 이므로 GitHub의 OAuth를 사용하는 것으로 구성하였는데, 실제 프로덕션 환경에서는 Azure AD 기반의 인증 방식 등, 사내 컴플라이언스를 준수하여 구축하는 것을 권장한다.

> **EKS 마이그레이션 시리즈 진행사항**<br>
> 1편: 레거시 PHP 앱 컨테이너화 + To-Be 아키텍처 설계<br>
> 2편: Terraform VPC/EKS/Karpenter 구성<br>
> 3편: Karpenter 구성 검증 + Pod Identity<br>
> 4편: Private 클러스터 + VSCode Server 구축<br>
> 5편: ALBC/ESO/ArgoCD GitOps 파이프라인<br>
> 6편: PHP 앱 컨테이너화 → Next.js 재구현 → MSA 분리<br>
> **7편: GitHub Actions + ArgoCD CI/CD + HPA + ALB Ingress Group + GitHub OAuth**
{: .prompt-tip }
