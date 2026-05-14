---
title: "VSCode Server Setup Guide for a Private Cluster"
date: 2026-05-14 18:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, vscode-server, private-endpoint, terraform, ssm, secrets-manager, aws, vpc-endpoint]
---

> EKS 마이그레이션 시리즈 세 번째 포스팅이다.<br>
> 앞선 포스팅에서 EKS 클러스터와 Karpenter를 구성했다. 이번에는 EKS API 엔드포인트를 완전 Private으로 전환하면서 발생한 문제들을 해결하고, 이를 위해 VPC 내부에서 Terraform과 kubectl을 실행할 수 있는 VSCode Server를 구축한 과정을 다룬다.
{: .prompt-info }

---

## 1. 왜 VSCode Server가 필요했나?

### EKS Private Endpoint 전환 결정

처음 EKS를 구성할 때는 `endpoint_public_access = true`로 설정했다. 개발 편의성을 위해서였다. 그러나 보안 검토 과정에서 문제가 명확해졌다.

```
현재 상태:
외부 인터넷 → EKS API 서버 (퍼블릭 + 프라이빗)
  └── 외부에서 kubectl, terraform 가능

목표 상태:
외부 인터넷 → 접근 불가
VPC 내부    → EKS API 서버 (프라이빗 전용)
```

> ISMS-P 관점에서 관리 포트(EKS API, 6443/443)를 인터넷에 노출하는 것은 접근통제 위반이다.<br>
> 따라서 `endpoint_public_access = false`로 전환이 필수였다.
{: .prompt-danger }

### 문제: 외부에서 kubectl/terraform 불가

EKS를 Private으로 전환하면 WSL(로컬 개발환경)에서 더 이상 `kubectl`, `helm`, `terraform apply`(Helm provider 포함)를 실행할 수 없다.

```
WSL (외부)
  └── kubectl get nodes             → Connection timeout ❌
  └── terraform apply (helm_release) → dial tcp timeout  ❌
```

### 해결책: VPC 내부에 관리용 서버 배치

VPC 내부에 관리용 서버를 두고, 그 서버에서 모든 인프라 작업을 수행하는 방식을 선택했다.

```
WSL (외부)
  └── SSM Session Manager → VSCode Server (VPC 내부)
                              └── kubectl, terraform, helm 실행
                              └── EKS API 서버 접근 가능 ✅
```

> 관리용 서버로 VSCode Server를 선택한 이유는 웹 브라우저로 접근 가능한 IDE 환경을 제공하기 때문이다.<br>
> SSH 없이 SSM Session Manager 포트포워딩만으로 브라우저에서 풀 IDE 환경을 쓸 수 있다.
{: .prompt-tip }

---

## 2. 아키텍처 설계

### 보안 원칙

| 항목 | 결정 | 이유 |
|---|---|---|
| 퍼블릭 IP | 없음 | 외부 직접 접근 차단 |
| SSH 접근 | 없음 | SSM Session Manager로 대체 |
| 인바운드 SG | 없음 | 아웃바운드만 허용 (SSM은 아웃바운드로 통신) |
| EBS 암호화 | 활성화 | ISMS-P 암호화 요건 |
| 패스워드 관리 | Secrets Manager | 평문 하드코딩 금지 |
| KMS | 활성화 | SSM 세션 암호화 |

### IAM 최소 권한

```
AmazonSSMManagedInstanceCore (관리형 정책)
  └── SSM Session Manager 접근

인라인 정책
  ├── S3: tfstate 버킷만 (GetObject, PutObject, DeleteObject, ListBucket)
  ├── DynamoDB: tfstate lock 테이블만 (GetItem, PutItem, DeleteItem)
  ├── EKS: DescribeCluster, ListClusters (kubeconfig 업데이트용)
  ├── Secrets Manager: VSCode 패스워드 secret ARN만 (GetSecretValue)
  └── KMS: SSM 세션 암호화 키만 (Decrypt, GenerateDataKey)
```

### 모듈 의존성 구조

VSCode Server 모듈 도입으로 EKS 모듈과의 순환 참조 문제가 발생했다. 이를 해결하기 위해 `security-groups` 모듈을 분리하여 단방향 의존성 구조를 만들었다.

```
module.vpc
  ↓
module.vscode           (vpc만 참조)
  ↓
module.security_groups  (vpc + vscode_sg_id 참조)
  ↓
module.eks              (security_groups 참조)
  ↓
module.karpenter
```

---

## 3. Terraform 모듈 구조

### 디렉토리 구조

```
terraform/
├── bootstrap/
│   └── main.tf
├── envs/
│   └── dev/
│       ├── versions.tf
│       ├── providers.tf
│       ├── locals.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── main.tf
│       └── terraform.tfvars
└── modules/
    ├── vpc/
    ├── vscode-server/      # 신규
    ├── security-groups/    # 신규 (순환 참조 해결)
    ├── vpc-endpoints/      # 신규
    ├── eks/
    └── karpenter/
```

### vscode-server 모듈

```hcl
# Secrets Manager — 패스워드 관리
resource "aws_secretsmanager_secret" "vscode" {
  name                    = "${local.prefix}-vscode-apne2-secret"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "vscode" {
  secret_id     = aws_secretsmanager_secret.vscode.id
  secret_string = jsonencode({ password = "change-me-after-deploy" })

  lifecycle {
    ignore_changes = [secret_string]  # 수동 업데이트 보호
  }
}

# VSCode Server EC2
resource "aws_instance" "vscode" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.private_subnet_id
  associate_public_ip_address = false  # 퍼블릭 IP 없음

  root_block_device {
    encrypted = true  # EBS 암호화
  }

  user_data_replace_on_change = true  # user_data 변경 시 자동 재생성
}
```

### user_data — 자동 설치 스크립트

EC2 부팅 시 필요한 도구들을 자동으로 설치하고 VSCode Server를 구동한다.

```bash
#!/bin/bash
export HOME=/root  # HOME 환경변수 명시 (미설정 시 오류 발생)

# 필수 도구 설치
# curl은 Amazon Linux 2023 기본 포함 → 별도 설치 불필요
# AWS CLI v2도 기본 포함 → 별도 설치 불필요
dnf update -y
dnf install -y git unzip tar jq

# kubectl 설치
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mv kubectl /usr/local/bin/

# Terraform 설치
TERRAFORM_VERSION="1.10.0"
curl -LO "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/\
terraform_${TERRAFORM_VERSION}_linux_amd64.zip"
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
mv terraform /usr/local/bin/

# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# VSCode Server 설치
curl -fsSL https://code-server.dev/install.sh | sh

# Secrets Manager에서 패스워드 읽기
VSCODE_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id <prefix>-vscode-apne2-secret \
  --region ap-northeast-2 \
  --query SecretString \
  --output text | jq -r '.password')

# VSCode Server 설정
mkdir -p /home/ec2-user/.config/code-server
cat > /home/ec2-user/.config/code-server/config.yaml <<VSCONFIG
bind-addr: 127.0.0.1:8080
auth: password
password: $VSCODE_PASSWORD
cert: false
VSCONFIG

systemctl enable --now code-server@ec2-user
```

### security-groups 모듈

EKS API 접근을 VSCode Server로만 제한하기 위한 추가 보안 그룹이다.

```hcl
resource "aws_security_group" "eks_additional" {
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = var.allowed_security_group_ids  # VSCode SG만
    description     = "HTTPS from allowed resources to EKS API"
  }
}
```

이 보안 그룹을 EKS 클러스터의 `security_group_ids`에 추가하면 VSCode Server에서만 EKS API(443)에 접근할 수 있다.

---

## 4. VPC Endpoint 설계

EKS Private 환경에서는 AWS 서비스 접근을 위해 VPC Endpoint가 필수다.

### Endpoint SG 인바운드 정책

> 초기에는 보안 그룹 기반으로 인바운드를 제한했으나, EKS 노드가 ECR에서 이미지를 pull할 때 노드 보안 그룹에서 Endpoint SG로의 443 허용이 필요했다.<br>
> 순환 참조 문제로 보안 그룹 기반 대신 **VPC CIDR 기반**으로 변경했다.
{: .prompt-warning }

```hcl
ingress {
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = [var.vpc_cidr]  # 172.16.0.0/16
  description = "HTTPS from within VPC"
}
```

> VPC CIDR 전체를 허용하되, Private Subnet 내부 트래픽만 가능하므로 외부에서의 접근은 차단된다.
{: .prompt-info }

### 생성된 Endpoint 목록

| Endpoint | 유형 | 용도 |
|---|---|---|
| ssm | Interface | SSM Session Manager |
| ssmmessages | Interface | SSM Session Manager |
| ec2messages | Interface | SSM Session Manager |
| secretsmanager | Interface | VSCode 패스워드 읽기 |
| ecr.api | Interface | ECR 이미지 관리 |
| ecr.dkr | Interface | ECR 이미지 pull |
| s3 | Gateway | ECR 레이어 pull, tfstate |
| eks | Interface | EKS API |
| logs | Interface | CloudWatch Logs |
| sts | Interface | Pod Identity 토큰 |

> **EKS 노드 ImagePull 문제**<br>
> EKS `depends_on`에 `module.vpc_endpoints`를 추가하지 않으면 노드가 먼저 생성되고 ECR Endpoint가 나중에 생성되어 `ImagePullBackOff`가 발생한다.
{: .prompt-danger }

```hcl
module "eks" {
  depends_on = [module.security_groups, module.vpc_endpoints]  # 순서 중요
}
```

---

## 5. 배포 전략

### WSL vs VSCode Server 역할 분리

EKS Private 전환 후 Helm provider가 EKS API에 접근해야 하는데 WSL(외부)에서는 불가능하다. 따라서 배포를 두 단계로 나눴다.

**WSL에서 (EKS 이전까지):**

```bash
terraform apply -auto-approve \
  -target=module.vpc \
  -target=module.vscode \
  -target=module.vpc_endpoints \
  -target=module.security_groups
```

**VSCode Server에서 (나머지 전체):**

```bash
terraform apply -auto-approve
```

> VSCode Server가 VPC 내부라 EKS API와 Helm 모두 접근 가능하다.
{: .prompt-tip }

---

## 6. VSCode Server 접근 방법

### SSM 포트포워딩

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}' \
  --region ap-northeast-2 \
  --profile <aws-profile>
```

브라우저에서 `http://localhost:8080` 접속 후 Secrets Manager에 설정한 패스워드로 로그인한다.

### 패스워드 설정

배포 후 Secrets Manager에서 수동으로 패스워드를 업데이트한다.

```bash
aws secretsmanager put-secret-value \
  --secret-id <prefix>-vscode-apne2-secret \
  --secret-string '{"password":"강력한-패스워드"}' \
  --region ap-northeast-2 \
  --profile <aws-profile>
```

이후 VSCode Server 내부에서 패스워드를 갱신한다.

```bash
VSCODE_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id <prefix>-vscode-apne2-secret \
  --region ap-northeast-2 \
  --query SecretString \
  --output text | jq -r '.password')

sed -i "s/^password:.*/password: $VSCODE_PASSWORD/" \
  /home/ec2-user/.config/code-server/config.yaml

systemctl restart code-server@ec2-user
```

---

## 7. 트러블슈팅

### Amazon Linux 2023 curl 패키지 충돌

```
package curl-minimal conflicts with curl provided by curl-8.17.0
```

| 항목 | 내용 |
|---|---|
| 원인 | Amazon Linux 2023은 `curl-minimal`이 기본 설치되어 있어 `curl` 패키지와 충돌 |
| 해결 | `dnf install`에서 `curl` 제거. Amazon Linux 2023은 curl이 이미 포함되어 있음 |

---

### AWS CLI 중복 설치

```
Found preexisting AWS CLI installation
```

| 항목 | 내용 |
|---|---|
| 원인 | Amazon Linux 2023에 AWS CLI v2가 기본 설치되어 있음 |
| 해결 | user_data에서 AWS CLI 설치 스크립트 제거 |

---

### HOME 환경변수 미설정

```
main: line 238: HOME: unbound variable
```

| 항목 | 내용 |
|---|---|
| 원인 | user_data 실행 환경에서 `HOME` 환경변수가 설정되지 않은 상태로 VSCode Server 설치 스크립트 실행 |
| 해결 | user_data 최상단에 `export HOME=/root` 추가 |

---

### KMS 권한 부족

```
Unable to retrieve data key, AccessDeniedException: not authorized to perform: kms:Decrypt
```

| 항목 | 내용 |
|---|---|
| 원인 | VSCode Server IAM Role에 KMS 복호화 권한 누락 |
| 해결 | IAM Policy에 KMS 권한 추가 |

```hcl
{
  Effect   = "Allow"
  Action   = ["kms:Decrypt", "kms:GenerateDataKey"]
  Resource = var.kms_key_arn
}
```

---

### Secrets Manager 삭제 대기

```
InvalidRequestException: You can't create this secret because a secret
with this name is already scheduled for deletion.
```

| 항목 | 내용 |
|---|---|
| 원인 | `recovery_window_in_days = 7` 설정으로 7일간 삭제 대기 상태 |
| 해결 | 강제 즉시 삭제 |

```bash
aws secretsmanager delete-secret \
  --secret-id <prefix>-vscode-apne2-secret \
  --force-delete-without-recovery \
  --region ap-northeast-2 \
  --profile <aws-profile>
```

---

### EKS 노드 ImagePullBackOff

| 항목 | 내용 |
|---|---|
| 원인 | EKS 노드가 먼저 생성되고 ECR VPC Endpoint가 나중에 생성되어 노드 부팅 시 ECR 접근 불가 |
| 해결 | `module.eks`의 `depends_on`에 `module.vpc_endpoints` 추가 |

```hcl
module "eks" {
  depends_on = [module.security_groups, module.vpc_endpoints]
}
```

---

### Helm provider EKS API 접근 불가 (WSL)

```
Kubernetes cluster unreachable: dial tcp 172.16.x.x:443: i/o timeout
```

| 항목 | 내용 |
|---|---|
| 원인 | EKS가 Private Endpoint 전용이라 VPC 외부(WSL)에서 접근 불가 |
| 해결 | VSCode Server(VPC 내부)에서 `terraform apply` 실행 |

---

## 8. 최종 검증

### 구성된 인프라

| 구분 | 리소스 | 상태 |
|---|---|---|
| Network | VPC Endpoints (10개) | ✅ |
| Compute | EKS Private Endpoint | ✅ |
| Compute | VSCode Server (SSM 접근) | ✅ |
| Security | Secrets Manager 패스워드 관리 | ✅ |
| Security | KMS 세션 암호화 | ✅ |

### EKS Private 동작 확인

VSCode Server에서 kubectl 접근 정상:

```bash
kubectl get nodes

NAME                                              STATUS   ROLES    AGE
ip-172-16-1-xxx.ap-northeast-2.compute.internal   Ready    <none>   35m
ip-172-16-2-xxx.ap-northeast-2.compute.internal   Ready    <none>   35m
```

WSL(외부)에서 접근 불가 — 의도된 동작:

```
dial tcp 172.16.x.x:443: i/o timeout
```

---

## 9. 다음 단계

> 1. **AWS Load Balancer Controller** — ALB 기반 Ingress 구성<br>
> 2. **External Secrets Operator** — Secrets Manager 연동<br>
> 3. **ArgoCD** — GitOps CD 파이프라인<br>
> 4. **GitHub Actions** — CI/CD 파이프라인<br>
> 5. **ECR 레포지토리** — web, was 이미지 저장소
{: .prompt-info }
