---
title: "Terraform Infrastructure Setup to Karpenter Configuration"
date: 2026-05-12 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, terraform, aws, kubernetes, gitops, karpenter, nodepool, iac, irsa, pod-identity]
---

> 이커머스 플랫폼의 EKS 마이그레이션 시리즈 두 번째 포스팅이다.<br>
> Terraform으로 VPC, EKS 클러스터 기반을 구축하고, EKS Add-on 설치, Karpenter 구성, NodePool 설계까지 진행한 전체 과정을 담았다.<br>
> 단순히 클러스터를 하나 올리는 것이 아니라, GitOps 기반의 완전 자동화된 배포 파이프라인과 이커머스 트래픽 급증 대응을 염두에 두고 설계했다.
{: .prompt-info }

---

## 1. 왜 Terraform인가?

기존 인프라는 수동으로 구성된 단일 EC2 인스턴스였다. 인프라 변경 이력이 없고, 재현이 불가능하며, 개발자가 SFTP로 직접 배포하는 구조였다. 이는 실제로 운영 결함으로 이어졌다.

이번 마이그레이션의 핵심 목표 중 하나는 **인프라의 코드화(IaC)** 다. Terraform을 선택한 이유는 다음과 같다.

- AWS Provider 생태계가 성숙하고 레퍼런스가 풍부하다
- `plan → apply` 워크플로우로 변경 사항을 사전에 검토할 수 있다
- S3 + DynamoDB 기반의 Remote State로 팀 협업이 가능하다
- GitHub Actions와의 연동으로 GitOps 파이프라인 구성이 용이하다

---

## 2. 인프라 전체 설계

### GitOps 기반 배포 흐름

```
Git Push (infra-repo)
  ↓
GitHub Actions (CI)
  - terraform plan / apply (인프라 변경)
  - ECR 이미지 빌드 및 푸시 (앱 변경)
  ↓
ArgoCD (CD)
  - Dev: 자동 Sync
  - Prod: PR Approve 후 수동 Sync
```

개발자가 코드를 Git에 push하면 자동으로 인프라와 앱이 반영된다. Prod 환경은 반드시 승인 절차를 거친다. 기존에 SFTP로 직접 배포하던 방식을 완전히 제거하는 것이 목표다.

### 전체 컴포넌트 스택

```
[Internet]
    ↓
[WAF]
    ↓
[ALB] ← AWS Load Balancer Controller
    ↓  path 기반 라우팅
┌─────────────────────────┐
│ /       → Web Pod       │  Apache + PHP-FPM
│ /api/*  → WAS Pod       │  Tomcat (SSO)
└─────────────────────────┘
    ↓
[Karpenter 관리 노드]
  ├── HPA (파드 오토스케일링)
  ├── External Secrets Operator → Secrets Manager
  ├── EFS CSI Driver → EFS
  └── ArgoCD (GitOps)
    ↓
[RDS MySQL] → (추후 Aurora 전환)
[ElastiCache Redis] → (세션 스토리지)
```

> **Ingress 선택: ALB vs Ingress NGINX**<br>
> Ingress NGINX Controller도 고려했지만, AWS 환경에서는 ALB가 path 기반 라우팅을 네이티브로 지원하고 WAF와의 연동도 자연스럽다.<br>
> API Gateway는 Lambda 연동이나 외부 API 관리가 필요할 때 고려할 옵션이며, 단순 path 라우팅 수준에서는 오버엔지니어링이 된다.
{: .prompt-tip }

### 노드 구성 전략

노드는 역할별로 세 가지로 분리한다.

| 구분 | 용도 |
|---|---|
| Managed Node Group (`*-sys-mng`) | Karpenter, ArgoCD, External Secrets 등 시스템 파드 전용 |
| Karpenter NodePool (`*-web-nodepool`) | Apache + PHP-FPM 파드 |
| Karpenter NodePool (`*-was-nodepool`) | Tomcat / SSO 파드 |

PHP-FPM과 Apache는 하나의 앱을 구성하는 사이드카 관계이므로 같은 Pod에 컨테이너 두 개로 띄운다.

```
[Web Pod]
├── Container 1: Apache  (앞단 요청 처리)
└── Container 2: PHP-FPM (PHP 실행)
```

> MNG는 Karpenter가 설치되기 전 시스템 파드를 올리기 위한 부트스트랩 노드 역할을 한다. 실제 앱 파드는 Karpenter NodePool이 관리한다.<br>
> MNG에는 `dedicated=system:NoSchedule` taint를 걸어 앱 파드가 실수로 올라오지 않도록 격리했다.
{: .prompt-info }

### Karpenter를 선택한 이유

이커머스 플랫폼 특성상 프로모션, 세일 이벤트 시 트래픽이 급증한다. Managed Node Group의 Auto Scaling은 스케일아웃 속도가 느려 이런 상황에 대응하기 어렵다.

| 항목 | Cluster Autoscaler | Karpenter |
|---|---|---|
| 스케일링 단위 | 노드 그룹 (고정 타입) | 개별 노드 (최적 타입 자동 선택) |
| 응답 속도 | 수 분 | 수십 초 |
| Spot 활용 | 제한적 | 자동 다양화 |
| 이커머스 적합성 | 보통 | 높음 |

> 파드 스케일링은 HPA로 처리하고, 추후 KEDA를 도입해 DB 커넥션 수, 큐 메시지 수 같은 커스텀 메트릭 기반의 스케일링으로 확장할 계획이다.
{: .prompt-tip }

---

## 3. Terraform 디렉토리 구조

```
infra-repo/
└── terraform/
    ├── bootstrap/              # S3 + DynamoDB (최초 1회만 실행)
    │   └── main.tf
    ├── modules/
    │   ├── vpc/                # VPC, 서브넷, IGW, NAT GW, 라우팅
    │   ├── eks/                # EKS 클러스터, IAM, OIDC, MNG, Add-on
    │   ├── karpenter/          # Karpenter IAM, SQS, EventBridge
    │   ├── efs/                # 추후 추가
    │   ├── alb/                # 추후 추가
    │   └── external-secrets/   # 추후 추가
    └── envs/
        ├── dev/                # 개발 환경
        └── prod/               # 운영 환경 (추후 추가)
```

### 네이밍 컨벤션

모든 리소스는 다음 패턴을 따른다.

```
<company>-<env>-<project>-<service>-<region>(-NN)-<resource>
```

| 리소스 | 이름 예시 |
|---|---|
| VPC | `example-dv-myapp-vpc` |
| Public Subnet | `example-dv-myapp-public-apne2-01-sub` |
| Private Subnet (App) | `example-dv-myapp-private-apne2-01-sub` |
| Private Subnet (DB) | `example-dv-myapp-db-apne2-01-sub` |
| EKS Cluster | `example-dv-myapp-eks-apne2-cluster` |
| System MNG | `example-dv-myapp-sys-apne2-mng` |
| Web NodePool | `example-dv-myapp-web-apne2-nodepool` |
| WAS NodePool | `example-dv-myapp-was-apne2-nodepool` |

---

## 4. Terraform Remote State: Bootstrap

Terraform state를 S3에서 관리하려면 S3 버킷이 먼저 존재해야 한다. 하지만 S3를 Terraform으로 만들면 그 state는 어디에 저장하나? 이 닭과 달걀 문제를 해결하기 위해 `bootstrap` 디렉토리를 분리했다.

> **처음엔 local state로 S3와 DynamoDB를 생성하고, 이후 backend를 S3로 전환한다.**
{: .prompt-info }

```hcl
resource "aws_s3_bucket" "tfstate" {
  bucket = "example-dv-myapp-tfstate-apne2-s3"
  lifecycle { prevent_destroy = true }
}

resource "aws_dynamodb_table" "tfstate_lock" {
  name         = "example-dv-myapp-tfstate-apne2-ddb"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

> **DynamoDB Lock이 필요한 이유**<br>
> DynamoDB는 tfstate 파일의 동시 수정을 방지하는 잠금 역할을 한다.<br>
> 여러 엔지니어가 동시에 `terraform apply`를 실행할 경우 state 충돌이 발생할 수 있어 반드시 필요하다.
{: .prompt-warning }

```bash
# Step 1: S3 + DynamoDB 생성 (local state)
cd terraform/bootstrap && terraform init && terraform apply

# Step 2: local state를 S3로 마이그레이션
terraform init -migrate-state
```

---

## 5. VPC 설계

### 서브넷 레이어 구성

| 레이어 | CIDR 예시 | 용도 | 라우팅 |
|---|---|---|---|
| Public | `172.16.10-11.0/24` | ALB | IGW |
| Private (App) | `172.16.1-2.0/24` | EKS 노드 | NAT GW |
| Private (DB) | `172.16.20-21.0/24` | RDS, ElastiCache | 없음 |

> **DB 서브넷에 NAT Gateway 라우팅을 붙이지 않은 이유**<br>
> DB가 인터넷으로 나갈 이유가 없고, 보안상 차단하는 것이 맞다. 이커머스 환경에서 DB의 외부 통신 차단은 ISMS-P 통제 항목과도 연결된다.
{: .prompt-tip }

> **NAT Gateway를 AZ별로 각각 배치한 이유**<br>
> 단일 NAT Gateway를 사용하면 해당 AZ 장애 시 다른 AZ의 Private 서브넷도 아웃바운드가 끊긴다. 이커머스 환경에서 가용성은 타협할 수 없다.
{: .prompt-warning }

### AWS Load Balancer Controller를 위한 서브넷 태그

AWS Load Balancer Controller가 어느 서브넷에 ALB를 생성할지 자동으로 인식하기 위해 반드시 필요하다.

```hcl
# Public Subnet — 외부 ALB 생성 위치
"kubernetes.io/role/elb" = "1"

# Private Subnet — 내부 ALB 생성 위치
"kubernetes.io/role/internal-elb" = "1"

# 클러스터 소유 표시
"kubernetes.io/cluster/<cluster-name>" = "shared"
```

---

## 6. EKS 클러스터 설계

### terraform plan: 주요 리소스

| 모듈 | 주요 리소스 |
|---|---|
| VPC | VPC, 서브넷, IGW, NAT GW, 라우팅 테이블 |
| EKS | IAM Role, 클러스터, OIDC Provider, MNG |

### OIDC Provider (IRSA 기반)

OIDC Provider는 EKS 파드가 AWS IAM Role을 가정할 수 있도록 하는 신뢰 체계의 핵심이다. 이를 통해 `AWS_ACCESS_KEY_ID` 같은 자격증명을 코드나 환경변수에 넣지 않아도 파드가 AWS API를 호출할 수 있다.

```hcl
resource "aws_iam_openid_connect_provider" "this" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.this.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.this.identity[0].oidc[0].issuer
}
```

### MNG taint 설정

시스템 파드 전용 노드에 taint를 설정해 앱 파드가 올라오지 않도록 격리한다.

```hcl
resource "aws_eks_node_group" "sys" {
  node_group_name = "${local.prefix}-sys-apne2-mng"

  taint {
    key    = "dedicated"
    value  = "system"
    effect = "NO_SCHEDULE"
  }

  labels = { role = "system" }
}
```

### kubectl, helm 설치 및 kubeconfig 설정

```bash
# kubectl 설치 및 자동완성
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc

# helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kubeconfig 업데이트
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region ap-northeast-2 \
  --profile <aws-profile>
```

### EKS Access Entry

EKS 클러스터를 생성한 IAM Role과 kubectl을 실행하는 IAM Role이 다를 경우 별도로 Access Entry를 추가해야 한다. SSO 환경에서는 특히 주의가 필요하다.

```hcl
resource "aws_eks_access_entry" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = "<iam-role-arn>"
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = "<iam-role-arn>"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope { type = "cluster" }
  depends_on   = [aws_eks_access_entry.admin]
}
```

> **SSO Role ARN 확인 방법**<br>
> `aws sts get-caller-identity` 명령으로 현재 세션의 ARN을 확인할 수 있다.<br>
> SSO Role은 리전이 ARN에 포함되는 경우가 있으니 (`eu-central-1` 등) 정확한 ARN을 확인 후 입력해야 한다.
{: .prompt-warning }

---

## 7. EKS Add-on 구성

EKS 기본 Add-on 4개를 Terraform으로 명시적 관리한다.

| Add-on | 역할 |
|---|---|
| `coredns` | 클러스터 내부 DNS 해석 |
| `kube-proxy` | 노드 간 네트워크 라우팅 |
| `vpc-cni` | Pod에 VPC IP 직접 할당 |
| `eks-pod-identity-agent` | Pod Identity 방식의 IAM 인증 에이전트 |

```hcl
# CoreDNS — sys 노드 taint toleration 필수
resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "coredns"

  configuration_values = jsonencode({
    tolerations = [{
      key      = "dedicated"
      operator = "Equal"
      value    = "system"
      effect   = "NoSchedule"
    }]
  })
}
```

> **CoreDNS에 toleration이 반드시 필요한 이유**<br>
> sys MNG에 `dedicated=system:NoSchedule` taint가 걸려 있어, toleration 없이는 CoreDNS가 해당 노드에 스케줄링되지 않는다.<br>
> CoreDNS가 Pending 상태면 클러스터 내 DNS 해석 전체가 불가능해지고, 이를 의존하는 Karpenter 등 모든 시스템 파드가 연쇄적으로 장애를 일으킨다.
{: .prompt-danger }

---

## 8. Karpenter 구성

### IAM 설계 — Pod Identity + 최소 권한

Karpenter 컨트롤러는 파드로 동작하기 때문에 EC2 Node Role에 권한을 주면 안 된다. 노드에 올라가는 모든 파드가 그 권한을 공유하게 되어 보안상 취약해진다.

**Pod Identity** 방식을 선택했다.

| 구분 | IRSA | Pod Identity |
|---|---|---|
| 설정 복잡도 | OIDC thumbprint 관리 필요 | 단순 |
| AWS 권장 | 기존 방식 | 최신 권장 방식 |
| Karpenter 지원 | v0.32+ | v1.x 공식 지원 |

```hcl
resource "aws_iam_role" "karpenter_controller" {
  name = "${local.prefix}-karpenter-ctrl-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action    = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}

resource "aws_eks_pod_identity_association" "karpenter" {
  cluster_name    = local.cluster_name
  namespace       = "karpenter"
  service_account = "karpenter"
  role_arn        = aws_iam_role.karpenter_controller.arn
}
```

### IAM Policy 세분화

| Policy | 역할 |
|---|---|
| `KarpenterControllerNodeLifecyclePolicy` | EC2 인스턴스 생성/종료 |
| `KarpenterControllerResourceDiscoveryPolicy` | 서브넷/AMI/인스턴스 타입 조회 (읽기 전용) |
| `KarpenterControllerInterruptionPolicy` | SQS 메시지 수신/삭제 |
| `KarpenterControllerEKSIntegrationPolicy` | EKS 클러스터 조회 |

`ec2:RunInstances`는 리소스 타입별 ARN으로 세분화해 최소 권한을 적용했다.

```hcl
{
  Effect = "Allow"
  Action = ["ec2:RunInstances"]
  Resource = [
    "arn:aws:ec2:<region>::image/*",
    "arn:aws:ec2:<region>:*:instance/*",
    "arn:aws:ec2:<region>:*:subnet/*",
    "arn:aws:ec2:<region>:*:security-group/*",
    ...
  ]
}
```

### SQS + EventBridge — Spot 인터럽션 처리

Spot 인스턴스 사용 시 AWS로부터 인터럽션 알림을 받아 선제적으로 대응하기 위해 SQS + EventBridge를 구성한다.

| EventBridge Rule | 이벤트 | Karpenter 동작 |
|---|---|---|
| Spot 인터럽션 | EC2 Spot이 2분 후 종료 예고 | 미리 새 노드 프로비저닝 |
| 인스턴스 상태 변경 | EC2 stopping/terminated | 노드 빠르게 정리 |
| 인스턴스 재조정 권고 | AWS가 인스턴스 교체 권고 | 선제적 노드 교체 |
| 예약된 이벤트 | 유지보수/리타이어 예정 | 미리 워크로드 이전 |

### Instance Profile 방식

```yaml
spec:
  instanceProfile: <karpenter-node-instance-profile>
```

> **`role` 대신 `instanceProfile`을 지정하는 이유**<br>
> `role` 방식은 Karpenter가 Instance Profile을 직접 생성/삭제해야 해서 IAM 권한이 더 넓어진다.<br>
> Terraform으로 미리 만들어두고 지정하는 방식이 최소 권한 원칙에 부합한다.
{: .prompt-tip }

---

## 9. NodePool 설계

### EC2NodeClass

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: example-karpenter-nodeclass
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  instanceProfile: <karpenter-node-instance-profile>
  subnetSelectorTerms:
    - id: <private-subnet-id-az-a>
    - id: <private-subnet-id-az-b>
  securityGroupSelectorTerms:
    - id: <node-security-group-id>
```

### NodePool (web / was)

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: example-web-nodepool
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: example-karpenter-nodeclass
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["t", "m", "c"]
      taints:
        - key: dedicated
          value: web
          effect: NoSchedule
  limits:
    cpu: 8
    memory: 32Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
```

> **리소스 limits 설정 기준**<br>
> 현재 운영 중인 인스턴스 스펙(vCPU, Memory)을 기준으로 web/was 역할 비율을 고려해 설정했다.<br>
> 실제 운영 전 부하 테스트 후 조정할 예정이다.
{: .prompt-info }

---

## 10. 전체 진행 결과

| 항목 | 상태 |
|---|---|
| Terraform 디렉토리 구조 및 네이밍 컨벤션 수립 | ✅ |
| Bootstrap (S3 + DynamoDB) + state S3 전환 | ✅ |
| VPC 모듈 (서브넷 레이어 분리, NAT HA) | ✅ |
| EKS 클러스터 + OIDC Provider + MNG | ✅ |
| kubectl, helm 설치 + EKS Access Entry | ✅ |
| DB 서브넷 + RDS/ElastiCache 서브넷 그룹 | ✅ |
| EKS Add-on 4개 (CoreDNS, VPC CNI, kube-proxy, Pod Identity Agent) | ✅ |
| Karpenter IAM (Pod Identity, 최소 권한 Policy 4개) | ✅ |
| Karpenter SQS + EventBridge (4개 Rule) | ✅ |
| Karpenter Helm 설치 | ✅ |
| EC2NodeClass + NodePool (web, was) | ✅ |

```bash
kubectl get ec2nodeclass,nodepool

NAME                                          READY   AGE
ec2nodeclass.karpenter.k8s.aws/example-karpenter-nodeclass   True    14s

NAME                                               NODECLASS                      NODES   READY   AGE
nodepool.karpenter.sh/example-was-nodepool         example-karpenter-nodeclass    0       True    18m
nodepool.karpenter.sh/example-web-nodepool         example-karpenter-nodeclass    0       True    18m
```

---

## 11. 트러블슈팅

### SCP로 인한 `ec2:ModifySubnetAttribute` 차단

Public 서브넷에 `map_public_ip_on_launch = true`를 설정했는데 아래 에러가 발생했다.

```
Error: modifying EC2 Subnet MapPublicIpOnLaunch
api error UnauthorizedOperation: not authorized to perform: ec2:ModifySubnetAttribute
with an explicit deny in a service control policy
```

| 항목 | 내용 |
|---|---|
| 원인 | 조직 SCP에서 퍼블릭 IP 자동 할당을 명시적으로 차단 |
| 해결 | `map_public_ip_on_launch = true` 옵션 제거 |

---

### MNG 노드 EKS 등록 실패

MNG 노드가 20분이 넘도록 EKS 클러스터에 등록되지 않았다. EC2 시스템 로그를 확인했다.

```
nodeadm: SDK retrying request EC2/DescribeInstances, attempt 1~14
nodeadm: context deadline exceeded
FAILED to start nodeadm-config.service
```

| 항목 | 내용 |
|---|---|
| 원인 | SCP 에러로 apply가 중단되면서 NAT Gateway 생성이 누락 → 노드 기동 시 아웃바운드 경로 없음 |
| 해결 | SCP 이슈 해결 후 terraform apply 재실행 |

---

### kubectl 인증 실패

```
error: the server has asked for the client to provide credentials
```

| 항목 | 내용 |
|---|---|
| 원인 | EKS 클러스터를 생성한 IAM Role과 kubectl을 실행하는 IAM Role이 달라 Access Entry 없음 |
| 해결 | Terraform으로 `aws_eks_access_entry` + `aws_eks_access_policy_association` 추가 |

---

### CoreDNS Pending → Karpenter CrashLoopBackOff

Karpenter 설치 후 파드가 CrashLoopBackOff 상태가 됐다.

```
원인 추적:
CoreDNS Pending (toleration 없어 sys 노드에 스케줄링 불가)
  → DNS 해석 불가
  → Karpenter가 AWS API 호출 실패
  → Karpenter CrashLoopBackOff
```

| 항목 | 내용 |
|---|---|
| 원인 | CoreDNS Add-on에 sys 노드 taint toleration 미설정으로 인한 연쇄 장애 |
| 해결 | CoreDNS Add-on `configuration_values`에 toleration 추가 + `eks-pod-identity-agent` 함께 설치 |

---

## 12. Pod Identity 상세 동작 원리

### 배경

Kubernetes 파드가 AWS API를 호출하려면 IAM 자격증명이 필요하다. 예를 들어 Karpenter가 EC2 인스턴스를 프로비저닝하거나, External Secrets Operator가 Secrets Manager에서 시크릿을 읽으려면 AWS 권한이 있어야 한다.

과거에는 EC2 Node Role에 권한을 부여하는 방식을 많이 썼다. 하지만 이 방식은 심각한 보안 문제가 있다.

```
[잘못된 방식]
EC2 Node Role에 모든 권한 부여
    ↓
노드에 올라가는 모든 파드가 그 권한을 공유
    ↓
앱 파드, 시스템 파드 구분 없이 EC2 프로비저닝 권한을 가짐
```

### IRSA vs Pod Identity

AWS가 먼저 내놓은 방식이 IRSA다. OIDC Provider를 통해 특정 Kubernetes ServiceAccount에만 IAM Role을 바인딩하는 방식이다.

| 구분 | IRSA | Pod Identity |
|---|---|---|
| 설정 복잡도 | OIDC thumbprint 관리 필요 | 단순 |
| AWS 권장 여부 | 기존 방식 | 최신 권장 방식 |
| Karpenter 지원 | v0.32+ | v1.x 공식 지원 |
| 멀티클러스터 | 클러스터마다 OIDC 관리 | 간소화 |
| 필수 Add-on | 없음 | eks-pod-identity-agent 필요 |

Pod Identity는 `eks-pod-identity-agent` DaemonSet이 각 노드에서 실행되면서 파드의 자격증명 요청을 처리한다.

### Pod Identity 동작 원리

```
[파드 기동]
    ↓
eks-pod-identity-agent가 파드 감지
    ↓
Pod Identity Association 확인
(namespace: karpenter, serviceAccount: karpenter → Role ARN 매핑)
    ↓
파드 환경변수에 자동 주입
  AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
  AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/...
    ↓
파드가 AWS SDK 호출 시 해당 엔드포인트에서 임시 자격증명 발급
    ↓
IAM Role assume → AWS API 호출
```

### Trust Policy

Pod Identity에서는 `pods.eks.amazonaws.com`을 Principal로 설정한다. 이것이 일반 EC2 Role과의 핵심 차이다.

```hcl
assume_role_policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{
    Effect    = "Allow"
    Principal = { Service = "pods.eks.amazonaws.com" }
    Action    = ["sts:AssumeRole", "sts:TagSession"]
  }]
})
```

그리고 어떤 파드가 이 Role을 assume할 수 있는지는 Pod Identity Association으로 제한한다.

```hcl
resource "aws_eks_pod_identity_association" "karpenter" {
  cluster_name    = local.cluster_name
  namespace       = "karpenter"   # karpenter 네임스페이스만
  service_account = "karpenter"   # karpenter SA만
  role_arn        = aws_iam_role.karpenter_controller.arn
}
```

> `karpenter` 네임스페이스의 `karpenter` ServiceAccount를 사용하는 파드만 이 Role을 assume할 수 있다.<br>
> 다른 네임스페이스, 다른 ServiceAccount는 절대 이 권한을 가질 수 없다.
{: .prompt-tip }

---

## 13. Karpenter 구성 검증

### Pod Identity 연동 확인

**Pod Identity Association 등록 확인**

```bash
aws eks list-pod-identity-associations \
  --cluster-name <cluster-name> \
  --region ap-northeast-2
```

```json
{
    "associations": [
        {
            "clusterName": "<cluster-name>",
            "namespace": "karpenter",
            "serviceAccount": "karpenter",
            "associationArn": "arn:aws:eks:ap-northeast-2:<account-id>:podidentityassociation/..."
        }
    ]
}
```

**파드 환경변수 확인**

> Karpenter 컨트롤러 이미지는 Distroless 기반이라 shell이 없다. `kubectl describe`로 환경변수를 확인한다.
{: .prompt-info }

```bash
kubectl describe pod -n karpenter <karpenter-pod-name>
```

```
Environment:
  AWS_CONTAINER_CREDENTIALS_FULL_URI:     http://169.254.170.23/v1/credentials
  AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE: /var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token
```

`169.254.170.23`은 EKS Pod Identity Agent 엔드포인트다. 이 값이 주입되어 있다는 것은 Pod Identity가 정상 연동된 것을 의미한다.

**Karpenter 로그에서 AWS API 정상 호출 확인**

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=10 2>&1 | grep -E "ERROR|error"
```

에러 없이 아래와 같은 INFO 로그만 출력되면 AWS API를 정상 호출 중이다.

```json
{"level":"INFO","message":"discovered ssm parameter",
  "parameter":"/aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id",
  "value":"ami-xxxxxxxxxxxxxxxxx"}
```

---

### 노드 자동 프로비저닝 확인

web NodePool에 스케줄링되는 테스트 파드를 띄워 노드가 자동 생성되는지 확인했다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: karpenter-test
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: web
      effect: NoSchedule
  nodeSelector:
    role: web
  containers:
    - name: test
      image: public.ecr.aws/amazonlinux/amazonlinux:latest
      command: ["sleep", "300"]
EOF
```

```bash
kubectl get nodes -w
```

```
ip-172-16-1-xxx   Ready   4h  ← sys MNG 노드 (기존)
ip-172-16-2-xxx   Ready   4h  ← sys MNG 노드 (기존)
ip-172-16-1-yyy   NotReady  0s  ← Karpenter가 새로 프로비저닝
ip-172-16-1-yyy   Ready    31s  ← 31초 만에 Ready
```

> **동작 원리**<br>
> sys MNG 노드에는 `dedicated=system:NoSchedule` taint가 걸려있다.<br>
> 테스트 파드는 `dedicated=web:NoSchedule` toleration만 가지므로 sys 노드에 스케줄링되지 않는다.<br>
> Karpenter가 스케줄링 실패를 감지하고 web NodePool 기준으로 새 EC2를 프로비저닝했다.
{: .prompt-info }

```bash
kubectl get nodeclaims
```

```
NAME                       TYPE         CAPACITY    ZONE              NODE              READY
example-web-nodepool-xxx   t3a.medium   on-demand   ap-northeast-2a   ip-172-16-1-yyy   True
```

---

### Consolidation (노드 자동 제거) 확인

```bash
kubectl delete pod karpenter-test
kubectl get nodes -w
```

NodePool에 `consolidateAfter: 30s`로 설정했기 때문에 파드가 없어진 후 30초 뒤 노드가 자동으로 제거됐다.

```
ip-172-16-1-yyy  Ready    → 삭제됨 (Karpenter consolidation)
```

> 비어있는 노드를 자동으로 정리해 불필요한 비용이 발생하지 않도록 한다.
{: .prompt-tip }

---

### CoreDNS DNS 해석 확인

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"dedicated","operator":"Equal","value":"system","effect":"NoSchedule"}]}}' \
  -- nslookup kubernetes.default
```

```
Server:    10.100.0.10
Address 1: 10.100.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.100.0.1 kubernetes.default.svc.cluster.local
```

CoreDNS(`10.100.0.10`)가 정상 동작하고 클러스터 내부 DNS 해석이 완벽히 이루어졌다.

---

### SQS + EventBridge 파이프라인 확인

```bash
aws events list-rules \
  --name-prefix <prefix>-karpenter \
  --region ap-northeast-2 \
  --query 'Rules[*].{Name:Name,State:State}'
```

```json
[
    {"Name": "<prefix>-karpenter-instance-rebalance",  "State": "ENABLED"},
    {"Name": "<prefix>-karpenter-instance-state",      "State": "ENABLED"},
    {"Name": "<prefix>-karpenter-scheduled-change",    "State": "ENABLED"},
    {"Name": "<prefix>-karpenter-spot-interruption",   "State": "ENABLED"}
]
```

4개 Rule 모두 ENABLED 상태이며 동일한 SQS ARN으로 정상 연결되어 있다.

> 실제 Spot 인터럽션 E2E 테스트는 AWS FIS(Fault Injection Simulator)로 별도 진행할 예정이다.
{: .prompt-info }

---

### 최종 검증 결과

| 항목 | 방법 | 결과 |
|---|---|---|
| Pod Identity Association 등록 | `aws eks list-pod-identity-associations` | ✅ |
| Pod Identity 자격증명 주입 | `kubectl describe pod` 환경변수 확인 | ✅ |
| Karpenter AWS API 정상 호출 | 로그 ERROR 없음, SSM 조회 INFO 확인 | ✅ |
| 노드 자동 프로비저닝 | 테스트 파드 → 새 노드 31초 내 생성 | ✅ |
| NodeClaim READY | `kubectl get nodeclaims` | ✅ |
| Consolidation | 파드 삭제 후 30초 내 노드 자동 제거 | ✅ |
| CoreDNS DNS 해석 | `nslookup kubernetes.default` | ✅ |
| VPC CNI Pod IP 할당 | 파드 IP가 VPC 대역 내 정상 할당 | ✅ |
| SQS 큐 생성 | `aws sqs get-queue-url` | ✅ |
| EventBridge Rule 4개 ENABLED | `aws events list-rules` | ✅ |
| EventBridge → SQS 연결 (4개) | `aws events list-targets-by-rule` | ✅ |

---

## 14. 트러블슈팅: IAM 권한 부족으로 인한 403 에러

### 문제

Karpenter가 노드를 성공적으로 프로비저닝했음에도 불구하고 로그에서 지속적으로 403 에러가 발생했다.

```json
{
  "level": "ERROR",
  "controller": "nodeclaim.tagging",
  "error": "operation error EC2: CreateTags, StatusCode: 403,
    api error UnauthorizedOperation: not authorized to perform: ec2:CreateTags"
}
{
  "level": "ERROR",
  "error": "operation error EC2: DeleteLaunchTemplate, StatusCode: 403,
    api error UnauthorizedOperation: not authorized to perform: ec2:DeleteLaunchTemplate"
}
```

### 원인 분석

초기 IAM Policy 설계 시 `ec2:CreateTags`와 `ec2:DeleteLaunchTemplate` 권한이 누락됐다.

| 액션 | 목적 |
|---|---|
| `ec2:CreateTags` | 생성된 EC2 인스턴스에 Karpenter 관리 태그를 붙임 |
| `ec2:DeleteLaunchTemplate` | 노드 프로비저닝에 임시로 사용한 Launch Template을 정리 |

### 해결

```hcl
{
  Effect   = "Allow"
  Action   = ["ec2:CreateTags"]
  Resource = "*"
},
{
  Effect   = "Allow"
  Action   = ["ec2:DeleteLaunchTemplate"]
  Resource = "*"
  Condition = {
    StringEquals = {
      "aws:ResourceTag/kubernetes.io/cluster/${local.cluster_name}" = "owned"
    }
  }
}
```

> **`ec2:CreateTags`에 조건을 걸 수 없는 이유**<br>
> Karpenter가 태그 없는 기존 인스턴스에 태그를 붙이는 동작이라 `RequestTag` 조건을 걸 수가 없다.<br>
> `ec2:DeleteLaunchTemplate`은 `ResourceTag` 조건으로 Karpenter가 만든 Launch Template만 삭제할 수 있도록 제한했다.
{: .prompt-warning }

> IAM 정책을 변경해도 실행 중인 파드는 캐싱된 자격증명을 계속 사용한다. 파드를 재시작해야 새 권한이 적용된다.
{: .prompt-danger }

```bash
kubectl rollout restart deployment/karpenter -n karpenter
kubectl rollout status deployment/karpenter -n karpenter
```

---

## 15. 다음 단계

> 1. **AWS Load Balancer Controller** — ALB 기반 Ingress, path 라우팅 (`/` → web, `/api/*` → was)<br>
> 2. **External Secrets Operator** — Secrets Manager 연동<br>
> 3. **ArgoCD** — GitOps CD 파이프라인<br>
> 4. **GitHub Actions** — CI 파이프라인 (빌드, ECR 푸시, 매니페스트 업데이트)<br>
> 5. **AWS FIS** — Spot 인터럽션 E2E 테스트 (추후)
{: .prompt-info }
