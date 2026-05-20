---
title: "Installing ALBC, ESO, and ArgoCD components for configuring GitOps pipelines"
date: 2026-05-20 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [aws, eks, albc, argocd, external-secrets, gitops, kubernetes, helm, irsa, eso]
---

> EKS 마이그레이션 시리즈 네 번째 포스팅이다.<br>
> 앞선 포스팅에서 EKS 클러스터, Karpenter, VSCode Server까지 구성했다. 이번에는 실제 GitOps 파이프라인을 구성하기 위한 핵심 컴포넌트들을 설치한다.<br>
> AWS Load Balancer Controller로 ALB를 자동 생성하고, External Secrets Operator로 Secrets Manager 연동을 구성했으며, ArgoCD로 GitOps CD 기반을 마련했다.
{: .prompt-info }

---

## 1. 전체 구성 흐름

이번 포스팅에서 구성하는 컴포넌트는 다음과 같다.

```
AWS Load Balancer Controller (ALBC)  → Ingress 기반 ALB 자동 생성
External Secrets Operator (ESO)      → Secrets Manager → K8s Secret 동기화
ArgoCD                               → GitOps CD
GitHub Actions OIDC Role             → ECR 푸시 권한 (CI 준비)
```

각 컴포넌트는 **IAM은 Terraform**, **설치는 Helm** 패턴을 일관되게 적용했다.

| 구분 | 도구 | 이유 |
|---|---|---|
| EKS 클러스터, VPC, IAM/IRSA | Terraform | 변경 주기 느림, 인프라 코드화 |
| ALBC, ESO, ArgoCD | Helm | 변경 주기 빠름, 쿠버네티스 네이티브 |

---

## 2. AWS Load Balancer Controller (ALBC)

### ALBC란?

EKS에서 Ingress 리소스를 배포하면 ALBC가 이를 감지해 AWS ALB를 자동으로 생성하고 관리한다. 수동으로 ALB를 만들고 타겟 그룹을 연결하는 작업이 필요 없다.

```
Ingress YAML 배포
      ↓
AWS Load Balancer Controller (EKS에서 실행)
      ↓
AWS ALB 자동 생성/관리
```

### IRSA 구성

ALBC가 AWS API를 호출해 ALB를 생성/수정/삭제하려면 IAM 권한이 필요하다.

> **IRSA vs Pod Identity**<br>
> Karpenter는 Pod Identity를 사용했지만, ALBC는 현재 공식적으로 **IRSA(IAM Role for Service Account)** 만 지원하므로 IRSA 방식으로 구성했다.
{: .prompt-warning }

```hcl
# modules/loadbalancer-controller/main.tf
resource "aws_iam_role" "albc" {
  name = "${var.prefix}-albc-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = var.oidc_provider_arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${var.oidc_provider_url}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
          "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "albc" {
  name   = "${var.prefix}-albc-apne2-policy"
  role   = aws_iam_role.albc.id
  policy = file("${path.module}/albc-iam-policy.json")
}
```

AWS 공식 IAM Policy JSON은 아래 명령으로 다운로드한다.

```bash
curl -o terraform/modules/loadbalancer-controller/albc-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

`modules/eks/outputs.tf`에 OIDC output이 있으면 별도 추가 없이 바로 참조할 수 있다.

```hcl
output "oidc_provider_arn" {
  value = aws_iam_openid_connect_provider.this.arn
}
output "oidc_provider_url" {
  value = replace(aws_eks_cluster.this.identity[0].oidc[0].issuer, "https://", "")
}
```

`envs/dev/main.tf`에 모듈을 추가한다.

```hcl
module "albc" {
  source = "../../modules/loadbalancer-controller"

  prefix            = "<prefix>"
  oidc_provider_arn = module.eks.oidc_provider_arn
  oidc_provider_url = module.eks.oidc_provider_url

  depends_on = [module.eks]
}
```

### Helm으로 ALBC 설치

IRSA 생성 후 Helm으로 ALBC를 설치한다. sys 노드에 올라가야 하므로 toleration을 함께 지정한다.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=<albc-role-arn>" \
  --set region=ap-northeast-2 \
  --set vpcId=<vpc-id> \
  --set "tolerations[0].key=dedicated" \
  --set "tolerations[0].operator=Equal" \
  --set "tolerations[0].value=system" \
  --set "tolerations[0].effect=NoSchedule"
```

설치 확인:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

```
aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0
aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0
```

### Ingress 배포 및 ALB 생성

```yaml
# kubernetes/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app>-ingress
  namespace: <namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/load-balancer-name: <alb-name>  # 32자 이하
spec:
  rules:
  - host: <your-domain>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: apache-service
            port:
              number: 80
```

> **`load-balancer-name` annotation 주의사항**<br>
> ALB 이름은 **32자 제한**이 있다. 초과 시 아래 에러가 발생한다.<br>
> `load balancer name cannot be longer than 32 characters`
{: .prompt-danger }

> **ACM 인증서 자동 연결**<br>
> `certificate-arn` annotation에 ACM 인증서 ARN을 지정하면 ALBC가 HTTPS 리스너에 자동으로 연결한다.<br>
> 인증서가 동일한 ARN으로 갱신되는 경우 별도 작업 없이 자동 반영된다.
{: .prompt-tip }

```bash
kubectl apply -f kubernetes/ingress/
kubectl get ingress -n <namespace>
```

---

## 3. External Secrets Operator (ESO)

### ESO란?

AWS Secrets Manager에 저장된 시크릿을 Kubernetes Secret으로 자동 동기화해주는 오퍼레이터다.

```
AWS Secrets Manager (시크릿 저장)
        ↓
ClusterSecretStore (연결 설정)
        ↓
ExternalSecret (어떤 시크릿을 가져올지 정의)
        ↓
Kubernetes Secret (자동 생성)
        ↓
Pod에서 환경변수로 사용
```

> **왜 ESO를 사용하는가?**<br>
> PHP 코드에서 Secrets Manager를 직접 호출하는 커스텀 헬퍼 코드를 제거할 수 있다.<br>
> ESO가 자동으로 K8s Secret을 생성하면, 파드는 환경변수로 주입받아 사용하면 된다.<br>
> `refreshInterval` 설정으로 주기적으로 최신 시크릿을 동기화하므로 Rotation과도 자연스럽게 연동된다.
{: .prompt-info }

### IRSA 구성

ESO도 Secrets Manager API를 호출해야 하므로 IRSA를 구성한다.

```hcl
# modules/external-secrets-operator/main.tf
resource "aws_iam_role" "eso" {
  name = "${var.prefix}-eso-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = var.oidc_provider_arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${var.oidc_provider_url}:sub" = "system:serviceaccount:external-secrets:external-secrets"
          "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "eso" {
  name = "${var.prefix}-eso-apne2-policy"
  role = aws_iam_role.eso.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = "arn:aws:secretsmanager:ap-northeast-2:<account-id>:secret:*"
    }]
  })
}
```

### Helm으로 ESO 설치

ESO는 3개의 컴포넌트(operator, cert-controller, webhook)로 구성된다. 각각 toleration을 지정해야 한다.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<eso-role-arn> \
  --set tolerations[0].key=dedicated \
  --set tolerations[0].operator=Equal \
  --set tolerations[0].value=system \
  --set tolerations[0].effect=NoSchedule \
  --set certController.tolerations[0].key=dedicated \
  --set certController.tolerations[0].operator=Equal \
  --set certController.tolerations[0].value=system \
  --set certController.tolerations[0].effect=NoSchedule \
  --set webhook.tolerations[0].key=dedicated \
  --set webhook.tolerations[0].operator=Equal \
  --set webhook.tolerations[0].value=system \
  --set webhook.tolerations[0].effect=NoSchedule
```

### ClusterSecretStore 및 ExternalSecret 배포

> **ESO v2.5.0 API 버전 변경**<br>
> ESO v2.5.0부터 API 버전이 `v1beta1` → `v1`으로 변경됐다. 구버전 예제를 그대로 사용하면 적용되지 않으니 주의.
{: .prompt-warning }

```yaml
# kubernetes/external-secrets/cluster-secret-store.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

```yaml
# kubernetes/external-secrets/external-secret-mysql.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mysql-secret
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: mysql-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: <secret-name>  # Secrets Manager의 시크릿 이름
```

정상 동기화 확인:

```bash
kubectl get clustersecretstore
```

```
NAME                  AGE   STATUS   CAPABILITIES   READY
aws-secrets-manager   8s    Valid    ReadWrite      True
```

```bash
kubectl get externalsecret -n <namespace>
```

```
NAME           STORETYPE            STORE                 REFRESH INTERVAL   STATUS         READY
mysql-secret   ClusterSecretStore   aws-secrets-manager   1h                 SecretSynced   True
```

---

## 4. ArgoCD

### GitOps CD

ArgoCD는 Git 레포지토리를 지속적으로 모니터링하고, 변경사항을 감지하면 클러스터에 자동으로 반영한다.

```
git push
    ↓
ArgoCD가 변경 감지
    ↓
클러스터 자동 배포
```

### Helm values 파일 관리

ArgoCD는 다수의 컴포넌트로 구성되며, 각각 toleration을 지정해야 한다.

> `--set` 플래그를 반복하는 대신 values 파일로 관리하는 것이 유지보수에 훨씬 좋다.
{: .prompt-tip }

```yaml
# helm/argocd/values.yaml
global:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

controller:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

dex:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

redis:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

redisSecretInit:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

server:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

repoServer:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

applicationSet:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule

notifications:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f helm/argocd/values.yaml
```

최종 확인:

```bash
kubectl get pods -n argocd
```

```
argocd-application-controller-0                     1/1     Running
argocd-applicationset-controller-xxxxxxxxxx-xxxxx   1/1     Running
argocd-dex-server-xxxxxxxxxx-xxxxx                  1/1     Running
argocd-notifications-controller-xxxxxxxxxx-xxxxx    1/1     Running
argocd-redis-xxxxxxxxxx-xxxxx                       1/1     Running
argocd-repo-server-xxxxxxxxxx-xxxxx                 1/1     Running
argocd-server-xxxxxxxxxx-xxxxx                      1/1     Running
```

---

## 5. GitHub Actions OIDC Role

### 왜 OIDC인가?

GitHub Actions에서 AWS 리소스에 접근하려면 자격증명이 필요하다.

| 방식 | 장단점 |
|---|---|
| Access Key (GitHub Secrets) | 간단하지만 장기 자격증명 노출 위험 |
| OIDC | 임시 자격증명 발급, 키 노출 위험 없음 ✅ |

### OIDC Provider 및 IAM Role 구성

```hcl
# modules/github-actions/main.tf
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

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
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # 특정 레포지토리, 특정 브랜치만 허용
          "token.actions.githubusercontent.com:sub" = "repo:<github-org>/*:ref:refs/heads/master"
        }
      }
    }]
  })
}
```

> **`GetAuthorizationToken` 권한 주의**<br>
> ECR 로그인에 필요한 `ecr:GetAuthorizationToken`은 리소스 레벨 제어가 불가능해 `Resource: "*"`로 지정해야 한다.<br>
> 나머지 ECR 작업(Push, Pull 등)은 레포지토리 ARN으로 최소 권한 원칙을 적용한다.
{: .prompt-warning }

---

## 6. 리포지토리 디렉토리 구조

이번 포스팅까지 완성된 `infra-repo` 레포지토리 구조다.

```
infra-repo/
├── helm/
│   └── argocd/
│       └── values.yaml
├── kubernetes/
│   ├── karpenter/
│   │   ├── ec2nodeclass.yaml
│   │   ├── nodepool-web.yaml
│   │   └── nodepool-was.yaml
│   ├── ingress/
│   │   ├── namespace.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── external-secrets/
│       ├── cluster-secret-store.yaml
│       └── external-secret-mysql.yaml
└── terraform/
    ├── envs/
    │   └── dev/
    └── modules/
        ├── eks/
        ├── karpenter/
        ├── loadbalancer-controller/
        ├── external-secrets-operator/
        ├── github-actions/
        └── ...
```

---

## 7. 트러블슈팅

### StatefulSet toleration 미반영

ArgoCD `argocd-application-controller`는 StatefulSet으로 배포된다. `helm upgrade` 후에도 기존 Pod에 변경사항이 반영되지 않는 경우가 있다.

| 항목 | 내용 |
|---|---|
| 원인 | StatefulSet은 rolling update 시 기존 Pod를 즉시 교체하지 않는 경우가 있음 |
| 해결 | Pod를 직접 삭제해 재생성 |

```bash
kubectl delete pod argocd-application-controller-0 -n argocd
```

---

### ALB 이름 32자 초과

```
load balancer name cannot be longer than 32 characters
```

| 항목 | 내용 |
|---|---|
| 원인 | `alb.ingress.kubernetes.io/load-balancer-name` annotation 값이 32자 초과 |
| 해결 | ALB 이름을 32자 이하로 줄임 |

---

### ESO ClusterSecretStore API 버전 오류

```
no matches for kind "ClusterSecretStore" in version "external-secrets.io/v1beta1"
```

| 항목 | 내용 |
|---|---|
| 원인 | ESO v2.5.0부터 API 버전이 `v1beta1` → `v1`으로 변경됨 |
| 해결 | `apiVersion: external-secrets.io/v1`으로 변경 |

---

## 8. 다음 단계

> 1. **ArgoCD Internal ALB 구성** — Route53 Private Hosted Zone 연동<br>
> 2. **애플리케이션 컨테이너화** — Dockerfile 작성<br>
> 3. **GitHub Actions CI 구성** — Checkmarx SAST 연동<br>
> 4. **ArgoCD Application 설정** — GitOps 배포 테스트
{: .prompt-info }
