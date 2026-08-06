---
title: "낡은 EKS 인증 방식을 걷어내고 - EKS aws-auth ConfigMap에서 Access Entry로 전환기"
date: 2026-06-30 09:00:00 +0900
categories: [2. Kubernetes, Cloud Native Transformation]
tags: [eks, aws-auth, access-entry, iam, terraform, kubernetes, security, rbac, cloudtrail]
---

> EKS 클러스터 인증 구성을 점검하는 과정에서 `authenticationMode: API_AND_CONFIG_MAP`으로 설정되어 있음을 확인했다.<br>
> 두 가지 인증 메커니즘이 병용되는 상태의 문제점을 분석하고, AWS 권장 표준인 **Access Entry 전용(API 모드)**으로 전환한 전 과정을 다룬다.
{: .prompt-info }

---

## 1. 문제 인식

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --region ap-northeast-2 \
  --query 'cluster.accessConfig'

{
    "authenticationMode": "API_AND_CONFIG_MAP"
}
```

병용 상태의 문제점:

| 문제 | 설명 |
|---|---|
| **이중 관리 부담** | aws-auth ConfigMap과 Access Entry를 동시에 관리해야 함 |
| **감사 불가** | ConfigMap 수정은 CloudTrail에 변경 내용(diff)이 기록되지 않음 |
| **보안 위협면** | ConfigMap은 cluster-admin 권한만 있으면 수정 가능 |
| **드리프트 위험** | ConfigMap 직접 수정과 Terraform 코드 간 불일치 발생 가능 |
| **구식 방식** | AWS는 신규 클러스터에 Access Entry 방식을 표준으로 권장 |

**목표:**
- `authenticationMode: API_AND_CONFIG_MAP` → `API` 전환
- `aws-auth` ConfigMap 삭제
- 모든 인증 주체를 Terraform으로 코드화하여 GitOps로 관리

---

## 2. aws-auth ConfigMap 상세

### 등장 배경

EKS 출시 초기(2018년)에는 IAM과 Kubernetes RBAC을 연결하는 표준이 없었다. AWS는 임시방편으로 `kube-system` 네임스페이스에 `aws-auth`라는 ConfigMap을 만들어 IAM → Kubernetes 매핑 정보를 저장하는 방식을 도입했다.

EKS의 **AWS IAM Authenticator**가 이 ConfigMap을 읽는다. IAM Authenticator는 `kube-apiserver`의 **웹훅 토큰 인증기(Webhook Token Authenticator)**로 동작하며, `kubectl` 명령 시 전달된 AWS 토큰을 검증하고 ConfigMap의 매핑 테이블을 참조해 Kubernetes identity를 반환한다.

### 동작 원리

```
kubectl get pods
│
├─ 1. aws eks get-token 실행
│       → STS PreSignedURL 생성 (서명된 GetCallerIdentity 요청)
│
├─ 2. kube-apiserver에 Bearer Token으로 전달
│
├─ 3. kube-apiserver → IAM Authenticator 웹훅 호출
│
├─ 4. IAM Authenticator가 STS에 GetCallerIdentity 호출
│       → ARN 확인: arn:aws:iam::<account-id>:role/admin-role
│
├─ 5. aws-auth ConfigMap에서 ARN 매핑 검색
│       mapRoles:
│         - rolearn: arn:aws:iam::<account-id>:role/admin-role
│           username: admin
│           groups:
│             - system:masters
│
└─ 6. Kubernetes identity 반환
        UserInfo { username: "admin", groups: ["system:masters"] }
```

### ConfigMap 구조

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<account-id>:role/<node-role>
      username: system:node:{{EC2PrivateDNSName}}   # 템플릿 변수 사용 가능
      groups:
        - system:bootstrappers
        - system:nodes

    - rolearn: arn:aws:iam::<account-id>:role/<admin-role>
      username: admin
      groups:
        - system:masters
```

### aws-auth의 구조적 문제

**보안 취약점**

```bash
# aws-auth ConfigMap 수정은 K8s RBAC으로 제어
# system:masters 그룹이 있는 누구든 수정 가능
kubectl edit configmap aws-auth -n kube-system
# → 악의적 Role ARN 추가 가능
# → CloudTrail에는 "principal updated configmap" 만 기록 (누가 어떤 entry를 추가했는지 불명확)
```

**보안 취약점**

```bash
# aws-auth ConfigMap 수정은 K8s RBAC으로 제어
# system:masters 그룹이 있는 누구든 수정 가능
kubectl edit configmap aws-auth -n kube-system
# → 악의적 Role ARN 추가 가능
# → CloudTrail에는 "principal updated configmap"만 기록 (누가 어떤 entry를 추가했는지 불명확)
```

**YAML 오류 무관용**

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::<account-id>:role/node-role
    username: system:node:{{EC2PrivateDNSName}}
    groups:
    - system:bootstrappers
    - system:nodes
   - rolearn: ...   # ← 들여쓰기 오류 한 줄로 클러스터 전체 잠김
```

> 실제로 잘못된 `aws-auth` YAML 하나로 클러스터에 아무도 접근하지 못하는 **Lock-out 사고**가 AWS 커뮤니티에서 반복적으로 발생해왔다.
{: .prompt-danger }

**감사 불투명성**

| 변경 방식 | CloudTrail 기록 |
|---|---|
| aws-auth ConfigMap 직접 수정 | `UpdateConfigMap` — 어떤 role이 추가/삭제됐는지 diff 없음 |
| Access Entry 생성 | `CreateAccessEntry` — 정확히 어떤 ARN이 추가됐는지 명시 |
| Access Entry 삭제 | `DeleteAccessEntry` — 누가 언제 어떤 ARN을 제거했는지 명시 |

---

## 3. EKS Access Entry 상세

### 등장 배경

AWS는 2023년 11월 **EKS Access Entry**를 출시했다. aws-auth ConfigMap의 구조적 문제를 해결하기 위해 IAM → Kubernetes 매핑을 Kubernetes 리소스가 아닌 **EKS API 리소스**로 관리하는 방식이다.

### 동작 원리

```
kubectl get pods
│
├─ 1. aws eks get-token (동일)
│
├─ 2. kube-apiserver에 Bearer Token으로 전달
│
├─ 3. kube-apiserver → EKS 인증 레이어 (API 모드)
│       ConfigMap 조회 없이 EKS 컨트롤 플레인 내부에서 처리
│
├─ 4. IAM ARN 확인 → Access Entry 테이블 조회
│       cluster: <cluster-name>
│       principal: arn:aws:iam::<account-id>:role/admin-role
│       → type: STANDARD
│       → policy: AmazonEKSClusterAdminPolicy
│
└─ 5. Kubernetes identity 반환 (RBAC 적용)
```

### Access Entry 타입

| 타입 | 용도 | K8s 그룹 자동 부여 |
|---|---|---|
| `STANDARD` | 사람/서비스 계정 | 없음 (Policy Association 필요) |
| `EC2_LINUX` | Linux 워커 노드 | `system:bootstrappers`, `system:nodes` |
| `EC2_WINDOWS` | Windows 워커 노드 | `system:bootstrappers`, `system:nodes` |
| `FARGATE_LINUX` | Fargate 파드 | `system:bootstrappers`, `system:nodes` |

> `EC2_LINUX` 타입은 Policy Association 없이도 노드 Join에 필요한 K8s 그룹을 자동으로 부여한다.<br>
> aws-auth의 `groups: [system:bootstrappers, system:nodes]`와 동일한 효과다.
{: .prompt-info }

### Access Policy

`STANDARD` 타입 Entry에는 AWS 관리형 Access Policy를 연결한다.

| Policy ARN | 부여 권한 | 대응 K8s RBAC |
|---|---|---|
| `AmazonEKSClusterAdminPolicy` | 클러스터 전체 관리 | `cluster-admin` ClusterRole |
| `AmazonEKSAdminPolicy` | 대부분의 리소스 관리 | 준-관리자 |
| `AmazonEKSEditPolicy` | 네임스페이스 내 리소스 편집 | `edit` ClusterRole |
| `AmazonEKSViewPolicy` | 읽기 전용 | `view` ClusterRole |
| `AmazonEKSAdminViewPolicy` | Secret 포함 전체 읽기 | 민감 정보 포함 전체 조회 |

Policy는 `access_scope`로 적용 범위를 지정할 수 있다.

```hcl
# 클러스터 전체 적용
access_scope {
  type = "cluster"
}

# 특정 네임스페이스만 적용 (최소 권한 원칙)
access_scope {
  type       = "namespace"
  namespaces = ["<namespace-a>", "<namespace-b>"]
}
```

---

## 4. aws-auth vs Access Entry 비교

### 핵심 특성 비교

| 항목 | aws-auth ConfigMap | Access Entry |
|---|---|---|
| **저장 위치** | `kube-system` 네임스페이스 (etcd) | EKS 컨트롤 플레인 내부 |
| **관리 API** | Kubernetes API (`kubectl`) | AWS EKS API |
| **접근 제어** | Kubernetes RBAC | IAM 정책 (`eks:CreateAccessEntry` 등) |
| **감사 로그** | CloudTrail에 diff 없이 기록 | CloudTrail에 entry 단위로 명시적 기록 |
| **오류 영향** | YAML 오류 시 클러스터 전체 접근 불가 | 개별 entry 오류가 다른 entry에 영향 없음 |
| **Terraform 관리** | `kubernetes_config_map` 우회 필요 | `aws_eks_access_entry` 네이티브 리소스 |
| **가용성** | etcd 의존 | EKS 컨트롤 플레인 직접 처리 |
| **IAM 조건부 접근** | 불가 | 가능 |
| **역할 공유** | 불가 (클러스터별 별도 매핑 필요) | 하나의 Access Entry를 여러 클러스터 재사용 가능 |
| **출시 시점** | 2018년 (EKS 출시 초기) | 2023년 11월 |

### 보안 모델 상세 비교

**aws-auth 보안 모델**

```
IAM 권한이 있는 사용자
  └─ kubectl edit cm/aws-auth -n kube-system
       └─ ConfigMap 수정 → IAM ARN 추가/삭제 가능
            └─ CloudTrail: "UpdateConfigMap" (내용 변경 diff 없음)
```

`kubectl` 접근 권한만 있으면 ConfigMap을 직접 수정할 수 있다. 즉, 쿠버네티스 cluster-admin이 aws-auth도 수정 가능하다는 의미로, IAM과 Kubernetes 권한 경계가 분리되지 않는다.

**Access Entry 보안 모델**

```
IAM 권한이 있는 사용자
  └─ aws eks create-access-entry (IAM 정책으로 제어)
       └─ EKS API 호출 → Access Entry 생성
            └─ CloudTrail: "CreateAccessEntry"
                 ├─ principalArn: arn:aws:iam::<account-id>:role/new-role
                 ├─ clusterName: <cluster-name>
                 └─ type: STANDARD
```

Access Entry 생성/삭제는 `eks:CreateAccessEntry`, `eks:DeleteAccessEntry` IAM 권한으로 제어된다. 쿠버네티스 권한과 완전히 분리되어 있으며, CloudTrail에 어떤 ARN이 어떤 클러스터에 추가됐는지 명확히 기록된다.

### 운영 측면 비교

**aws-auth: 직접 수정의 위험성**

```bash
# 이 한 줄이 클러스터 전체를 잠글 수 있음
kubectl edit configmap aws-auth -n kube-system
# → vi 에디터에서 YAML 들여쓰기 오류 발생 시
# → 저장하면 aws-auth 파싱 실패 → 모든 사람이 클러스터 접근 불가

# 복구 방법: AWS Console에서 직접 접근하거나
# 클러스터 IAM 역할로 직접 수정 (복잡한 절차 필요)
```

**Access Entry: 개별 리소스 단위 관리**

```bash
# 잘못된 entry 생성해도 다른 entry에 영향 없음
aws eks create-access-entry \
  --cluster-name <cluster-name> \
  --principal-arn arn:aws:iam::<account-id>:role/wrong-role \
  --type STANDARD

# 삭제로 깔끔하게 원상복구
aws eks delete-access-entry \
  --cluster-name <cluster-name> \
  --principal-arn arn:aws:iam::<account-id>:role/wrong-role
```

### Terraform 관리 비교

| 항목 | aws-auth ConfigMap | Access Entry |
|---|---|---|
| **저장 위치** | `kube-system` 네임스페이스 (etcd) | EKS 컨트롤 플레인 내부 |
| **관리 API** | Kubernetes API (`kubectl`) | AWS EKS API |
| **접근 제어** | Kubernetes RBAC | IAM 정책 (`eks:CreateAccessEntry` 등) |
| **감사 로그** | CloudTrail에 diff 없이 기록 | CloudTrail에 entry 단위로 명시적 기록 |
| **오류 영향** | YAML 오류 시 클러스터 전체 접근 불가 | 개별 entry 오류가 다른 entry에 영향 없음 |
| **Terraform 관리** | `kubernetes_config_map` 우회 필요 | `aws_eks_access_entry` 네이티브 리소스 |
| **가용성** | etcd 의존 | EKS 컨트롤 플레인 직접 처리 |
| **IAM 조건부 접근** | 불가 | 가능 |
| **출시 시점** | 2018년 (EKS 출시 초기) | 2023년 11월 |

### Terraform 관리 비교

**aws-auth (구 방식)**

```hcl
# 방법 1: kubernetes provider로 직접 관리 (취약)
resource "kubernetes_config_map_v1_data" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }
  data = {
    mapRoles = yamlencode([{
      rolearn  = aws_iam_role.node.arn
      username = "system:node:{{EC2PrivateDNSName}}"
      groups   = ["system:bootstrappers", "system:nodes"]
    }])
  }
  force = true  # 수동 변경 덮어쓰기
}

# 방법 2: null_resource + kubectl (더 취약)
resource "null_resource" "aws_auth" {
  triggers = { config = local.aws_auth_configmap }
  provisioner "local-exec" {
    command = "kubectl apply -f aws-auth.yaml"
  }
}
```

두 방법 모두 Terraform state와 실제 ConfigMap 간 드리프트가 발생하기 쉽고, `force = true` 없이는 외부 변경이 덮어써지지 않는 문제가 있다.

**Access Entry (현재 방식)**

```hcl
# AWS 네이티브 Terraform 리소스 — 완전한 IaC 관리
resource "aws_eks_access_entry" "node" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.node.arn
  type          = "EC2_LINUX"
}

resource "aws_eks_access_entry" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope { type = "cluster" }
  depends_on = [aws_eks_access_entry.admin]
}
```

### 장단점 요약

**aws-auth ConfigMap**

장점: EKS 초기부터 존재해 오래된 가이드가 많음, `kubectl`만으로 관리 가능, `username`/`groups` 자유 커스터마이징

단점: YAML 오류 한 줄로 클러스터 전체 잠김 위험, CloudTrail 감사 불투명, K8s RBAC과 IAM 권한 경계 미분리, Terraform native 지원 없음, **AWS가 신규 기능 개발 중단(레거시)**

**Access Entry**

장점: IAM 정책으로 세밀한 접근 제어, CloudTrail entry 단위 명확한 감사 기록, Terraform `aws_eks_access_entry` 네이티브 지원, 개별 entry 오류 격리, **AWS 신규 표준으로 지속 개발 중**, Namespace 단위 권한 범위 지정 가능

단점: EKS 1.23+ 클러스터에서만 지원, `API` 모드 전환은 비가역적, AWS CLI/SDK 필요

---

## 5. 인증 모드(authenticationMode) 이해

| 모드 | aws-auth ConfigMap | Access Entry | 비고 |
|---|---|---|---|
| `CONFIG_MAP` | 사용 | 사용 불가 | EKS 초기 기본값, 레거시 |
| `API_AND_CONFIG_MAP` | 사용 (병용) | 사용 | 마이그레이션 중간 단계 |
| `API` | 무시 | 사용 | **현재 AWS 권장 표준** |

### 전환 방향 (단방향)

```
CONFIG_MAP  →  API_AND_CONFIG_MAP  →  API
    ↑                                    │
    └──────────── 되돌릴 수 없음 ─────────┘
```

> **`API` 모드로 전환한 이후에는 되돌릴 수 없다.**<br>
> 전환 전 모든 인증 주체가 Access Entry로 등록되어 있는지 반드시 확인해야 한다.
{: .prompt-danger }

### 병용 모드에서 우선순위

```
인증 요청 → Access Entry 조회
              ├─ 존재하면 → Access Entry 결과 사용
              └─ 없으면   → aws-auth ConfigMap 조회
                             ├─ 존재하면 → ConfigMap 결과 사용
                             └─ 없으면   → 인증 실패 (401)
```

---

## 6. 작업 전 현황 분석

### 클러스터 인증 주체 전체 목록

```
주체 1: 관리자 롤 (var.admin_role_arn)
├─ aws-auth: 없음
└─ Access Entry: ✅ 존재 (STANDARD + AmazonEKSClusterAdminPolicy)

주체 2: <prefix>-karpenter-apne2-node-role
├─ aws-auth: 없음
└─ Access Entry: ✅ 존재 (EC2_LINUX)

주체 3: <prefix>-eks-apne2-node-role  ← 문제의 1개
├─ aws-auth: ✅ 존재 (mapRoles에 등록)
└─ Access Entry: AWS가 자동 생성 (Terraform state에는 없음)
```

### aws-auth ConfigMap 내용

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<account-id>:role/<prefix>-eks-apne2-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
```

`<prefix>-eks-apne2-node-role`은 Managed Node Group이 사용하는 노드 IAM 롤이다.

이 롤이 aws-auth에 등록된 이유는 클러스터 초기 구성 시 `API_AND_CONFIG_MAP` 상태에서 MNG를 생성했기 때문이다. MNG를 생성하면 EKS가 해당 노드 롤의 Access Entry를 **자동으로 생성**하지만, 동시에 aws-auth에도 수동으로 등록해 이중 관리 상태가 됐다.

> **MNG 자동 Access Entry 생성 동작**<br>
> Managed Node Group을 `API_AND_CONFIG_MAP` 또는 `API` 모드 클러스터에서 생성하면, EKS가 해당 노드 IAM 롤에 대한 `EC2_LINUX` 타입 Access Entry를 **자동으로 생성**한다.<br>
> 이는 Terraform `aws_eks_node_group` 리소스에 명시되지 않은 **암묵적 동작**이다. 나중에 명시적으로 `aws_eks_access_entry`를 추가하면 중복 생성을 시도해 충돌이 발생한다.
{: .prompt-warning }

### 작업 전 Terraform 코드 현황

```hcl
# terraform/modules/eks/main.tf (작업 전)

resource "aws_eks_cluster" "this" {
  access_config {
    authentication_mode = "API_AND_CONFIG_MAP"  # ← 변경 대상
  }
}

# aws_eks_access_entry.node 존재하지 않음  # ← 추가 대상
# (실제로는 AWS가 자동 생성해두었지만 Terraform state에는 없음)

resource "aws_eks_access_entry" "admin" {  # ← 이미 존재
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  type          = "STANDARD"
}
```

---

## 7. 마이그레이션 절차

### 마이그레이션 전 체크리스트

```
✅ 클러스터 버전 1.23+ 확인 (Access Entry 지원 요건)
✅ 현재 aws-auth에 등록된 모든 주체 파악
✅ 해당 주체의 Access Entry 존재 여부 확인
✅ authenticationMode 전환은 비가역임을 인지
```

### 안전한 마이그레이션 순서

```
Step 1: Access Entry 생성 (노드 롤)       ← 반드시 먼저
         ↓
Step 2: authenticationMode → API 전환
         ↓
Step 3: aws-auth ConfigMap 삭제
         ↓
Step 4: 노드 Ready 상태 확인
```

> **Step 1을 반드시 Step 2 전에 완료해야 하는 이유**<br>
> `API` 모드로 전환하는 순간 aws-auth ConfigMap이 무시된다. 노드 롤에 Access Entry가 없으면 해당 롤을 사용하는 모든 노드가 EKS API 서버에 인증할 수 없게 되어 새로운 노드 Join이 전면 불가 상태가 된다.<br>
> 현재 실행 중인 파드는 영향을 받지 않지만, 노드 교체나 Karpenter 스케일아웃 시 노드가 Join되지 않아 파드 스케줄링이 멈춘다.
{: .prompt-danger }

### Terraform 코드 수정

```hcl
# terraform/modules/eks/main.tf

# 1. authenticationMode 변경
resource "aws_eks_cluster" "this" {
  # ...
  access_config {
    authentication_mode = "API"   # API_AND_CONFIG_MAP → API
  }
}

# 2. 노드 롤 Access Entry 추가
resource "aws_eks_access_entry" "node" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.node.arn
  type          = "EC2_LINUX"
  # EC2_LINUX 타입 자동 부여:
  #   groups: [system:bootstrappers, system:nodes]
  #   username: system:node:<NodeName>
}
```

### terraform plan 결과

```
# module.eks.aws_eks_access_entry.node will be created
# module.eks.aws_eks_cluster.this will be updated in-place
  ~ authentication_mode = "API_AND_CONFIG_MAP" → "API"

Plan: 1 to add, 7 to change, 0 to destroy.
```

### terraform apply — 1차 시도 (Access Entry 충돌)

```
Error: creating EKS Access Entry
  ResourceInUseException: The specified access entry resource
  is already in use on this cluster.
```

**원인**: EKS가 MNG 생성 시 노드 IAM 롤에 대한 `EC2_LINUX` Access Entry를 이미 자동 생성해두었다. Terraform state에는 없었지만 실제 AWS에는 존재하는 상태였다.

동시에 `authentication_mode` 변경은 성공적으로 적용됐다 (Terraform이 두 리소스를 병렬 처리).

> **Terraform apply가 에러로 종료됐어도 일부 리소스는 이미 변경되었을 수 있다.**<br>
> `terraform plan`으로 현재 state와 실제 인프라의 차이를 다시 확인하는 것이 중요하다.
{: .prompt-warning }

### terraform import (상태 동기화)

```bash
terraform import \
  -var-file="dev.secret.tfvars" \
  module.eks.aws_eks_access_entry.node \
  "<cluster-name>:arn:aws:iam::<account-id>:role/<prefix>-eks-apne2-node-role"

# Import successful!
```

기존에 AWS가 자동 생성한 Access Entry를 Terraform state로 가져와 코드와 state를 일치시켰다.

### terraform apply — 2차 시도

```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

state import 후 apply 시 이미 모든 리소스가 원하는 상태이므로 변경 없음.

### aws-auth ConfigMap 삭제

```bash
kubectl delete configmap aws-auth -n kube-system
# configmap "aws-auth" deleted from kube-system namespace
```

`API` 모드에서 aws-auth ConfigMap은 완전히 무시된다. 삭제하지 않아도 동작에 영향은 없지만, 오해를 방지하기 위해 삭제했다.

---

## 8. Terraform 최종 상태

```hcl
# terraform/modules/eks/main.tf

# EKS Cluster — API 전용 인증 모드
resource "aws_eks_cluster" "this" {
  access_config {
    authentication_mode = "API"
  }
  # ...
}

# EKS Access Entry - Managed Node Group 노드 (신규 추가, import로 state 동기화)
resource "aws_eks_access_entry" "node" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.node.arn
  type          = "EC2_LINUX"
}

# EKS Access Entry - 관리자 (기존 유지)
resource "aws_eks_access_entry" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope { type = "cluster" }
  depends_on = [aws_eks_access_entry.admin]
}
```

```hcl
# terraform/modules/karpenter/main.tf (기존 유지)

resource "aws_eks_access_entry" "karpenter_node" {
  cluster_name  = local.cluster_name
  principal_arn = aws_iam_role.karpenter_node.arn
  type          = "EC2_LINUX"
}
```

**최종 Access Entry 전체 목록:**

| Terraform 리소스 | IAM 롤 | 타입 | 관리 파일 |
|---|---|---|---|
| `aws_eks_access_entry.node` | `<prefix>-eks-apne2-node-role` | EC2_LINUX | `eks/main.tf` |
| `aws_eks_access_entry.admin` | `admin_role_arn` (var) | STANDARD | `eks/main.tf` |
| `aws_eks_access_entry.karpenter_node` | `<prefix>-karpenter-apne2-node-role` | EC2_LINUX | `karpenter/main.tf` |

---

## 9. 트러블슈팅

### ListAccessEntries 권한 없음

**증상**

```
AccessDeniedException: User: ... is not authorized to perform:
eks:ListAccessEntries on resource: ...
```

**원인**: VSCode 서버 인스턴스 롤에 `eks:ListAccessEntries` 권한이 없었다.

**해결**: `terraform apply`를 시도하면서 `ResourceInUseException`으로 Access Entry 존재를 간접 확인한 뒤, `terraform import`로 처리했다.

---

### terraform apply 병렬 실행 문제

1차 `terraform apply`에서 Terraform이 두 변경을 병렬로 처리했다.

- **Task A**: `aws_eks_access_entry.node` 생성 시도 → `ResourceInUseException` ❌
- **Task B**: `aws_eks_cluster.this` 업데이트 (`API_AND_CONFIG_MAP` → `API`) → 성공 ✅

Task A가 실패했지만 Task B는 이미 완료된 상태여서 클러스터는 `API` 모드가 됐다. `terraform import`로 Task A의 state를 동기화해 해결했다.

---

### MNG 노드 롤 Access Entry 자동 생성

| 항목 | 내용 |
|---|---|
| 원인 | MNG를 `API_AND_CONFIG_MAP` 또는 `API` 모드 클러스터에서 생성하면 EKS가 노드 IAM 롤에 대한 `EC2_LINUX` Access Entry를 **자동으로 생성**한다. Terraform `aws_eks_node_group` 리소스에 명시되지 않은 암묵적 동작이다. |
| 결과 | Terraform state에는 없는 리소스가 실제 AWS에 존재하는 불일치 발생 |
| 해결 | `terraform import`로 기존 리소스를 state에 가져와 코드와 state를 일치시킴 |

---

## 10. 검증 및 최종 상태

### 노드 상태 확인

```bash
kubectl get nodes
# 모든 노드 Ready 상태 확인
# MNG 노드(sys_2a/2c)와 Karpenter 노드 모두 정상
```

### 최종 인증 구성 확인

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --region ap-northeast-2 \
  --query 'cluster.accessConfig'
# { "authenticationMode": "API" }

kubectl get configmap aws-auth -n kube-system
# Error from server (NotFound): configmaps "aws-auth" not found
```

### 마이그레이션 전후 비교

**마이그레이션 전** — `authenticationMode: API_AND_CONFIG_MAP`

| IAM 롤 | aws-auth | Access Entry |
|---|---|---|
| admin_role_arn | ✗ | ✅ STANDARD |
| karpenter-node-role | ✗ | ✅ EC2_LINUX |
| eks-apne2-node-role | ✅ 수동 등록 | 자동 생성 (숨김) |

**마이그레이션 후** — `authenticationMode: API`

| IAM 롤 | aws-auth | Access Entry |
|---|---|---|
| admin_role_arn | 삭제됨 | ✅ STANDARD (`eks/main.tf`) |
| karpenter-node-role | 해당 없음 | ✅ EC2_LINUX (`karpenter/main.tf`) |
| eks-apne2-node-role | 삭제됨 | ✅ EC2_LINUX (`eks/main.tf`) |

### 마이그레이션 효과

| 항목 | 이전 | 이후 |
|---|---|---|
| 인증 방식 | aws-auth + Access Entry 병용 | Access Entry 단일 방식 |
| Terraform 관리 범위 | 일부 주체만 IaC 관리 | 전체 인증 주체 IaC 관리 |
| CloudTrail 감사 | ConfigMap 변경 로그 (diff 없음) | Entry 단위 명확한 기록 |
| 클러스터 잠김 위험 | YAML 오류 시 전체 잠김 가능 | 개별 entry 오류는 격리됨 |
| 보안 경계 | K8s RBAC과 IAM 권한 혼재 | IAM 정책으로 완전 분리 |
