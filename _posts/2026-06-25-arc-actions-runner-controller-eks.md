---
title: "Configuring ARC (Actions Runner Controller) on EKS — GitOps Operations for GitHub Actions Self-hosted Runners"
date: 2026-06-25 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, arc, github-actions, self-hosted-runner, gitops, argocd, github-apps, kubernetes, pod-identity, terraform]
---

> **환경**: AWS EKS / ARC v0.10.1 / ArgoCD v3.4.2<br>
> **목적**: Private EKS 클러스터에서 GitHub Actions self-hosted runner 운영 및 CI/CD 배포 검증 자동화
{: .prompt-info }

---

## 1. ARC 도입 배경

### 왜 Self-hosted Runner가 필요한가

GitHub Actions의 기본 runner(GitHub-hosted runner)는 공용 인터넷에서 실행된다. **Private VPC 안에 위치한 EKS 클러스터의 API Server에는 직접 접근할 수 없다.**

```
[GitHub-hosted runner]
  ✅ Docker build, ECR push — 외부 인터넷 접근 가능
  ❌ kubectl rollout status — EKS API Server가 Private, 접근 불가

[ARC self-hosted runner (EKS 내부)]
  ✅ kubectl rollout status — VPC 내부에서 직접 접근 가능
  ✅ aws eks update-kubeconfig — Pod Identity로 IAM 자동 인증
```

따라서 CI/CD 파이프라인에서 역할을 분리했다.

| 단계 | Runner | 작업 |
|---|---|---|
| 1~3 | GitHub-hosted (`ubuntu-latest`) | 빌드, 품질 검사, ECR push |
| 4 | ARC self-hosted (`<prefix>-runner`) | 배포 완료 검증 (kubectl) |

### ARC란

ARC(Actions Runner Controller)는 Kubernetes 위에서 GitHub Actions self-hosted runner를 **오토스케일링 방식**으로 운영하기 위한 컨트롤러다.

```
job이 없을 때:  runner pod = 0개 (비용 최적화)
job 트리거:     controller가 runner pod 동적 생성 (ephemeral runner)
job 완료:       runner pod 즉시 삭제 (격리 보장)
```

> **Ephemeral runner의 보안 이점**<br>
> 매 job마다 새 pod를 생성하고 완료 후 즉시 삭제하므로, 이전 job의 환경 변수, 파일, 캐시가 다음 job에 남지 않는다. 공용 runner와 달리 환경 오염 위험이 없다.
{: .prompt-info }

### CI/CD 파이프라인에서의 역할

```
[GitHub Actions CI 파이프라인]

Step 1. 코드 품질 검사 (GitHub-hosted)
        TypeScript 타입체크 / ESLint / npm audit

Step 2. Docker 이미지 빌드 & ECR push (GitHub-hosted)
        태그: YYYYMMDD-{short_sha}

Step 3. infra 레포 이미지 태그 업데이트 커밋 (GitHub-hosted)
        → ArgoCD 변경 감지 → EKS 자동 배포 시작 (GitOps 트리거)

Step 4. 배포 완료 검증 (ARC self-hosted runner ← 여기!)
        aws eks update-kubeconfig
        kubectl rollout status deployment/... --timeout=5m
        → 성공: 파이프라인 완료 ✅
        → 실패: 배포 이상 감지 ❌
```

> **Step 4가 성공하면 보장되는 것**<br>
> 코드가 빌드/보안 검증을 통과하고 **실제 클러스터에 배포까지 완료됐음**을 자동으로 검증한다.<br>
> 단순히 "이미지가 ECR에 올라갔다"가 아니라 "파드가 새 이미지로 Rolling Update 완료됐다"를 확인한다.
{: .prompt-tip }

---

## 2. GitHub Apps vs PAT 인증

초기에는 PAT(Personal Access Token) 방식으로 구성했으나, 아래 이유로 GitHub Apps 방식으로 전환했다.

| 항목 | PAT | GitHub Apps |
|---|---|---|
| 연결 대상 | 개인 계정 종속 | 조직/레포 단위 설치 |
| 만료 정책 | 최대 1년, 수동 갱신 필요 | Private key 기반 단기 토큰 자동 발급 |
| 권한 범위 | 계정 전체 권한 가능 | 레포/권한 단위 세밀한 제어 |
| 보안 | 계정 탈취 시 전체 위험 | App 별 격리 |
| 운영 | 만료 시 runner 중단 | 자동 토큰 갱신, 중단 없음 |

> PAT 방식은 개인 계정에 종속되어 팀원이 퇴사하거나 계정이 변경될 때 모든 runner가 중단된다. 운영 환경에서는 반드시 GitHub Apps 방식을 사용해야 한다.
{: .prompt-warning }

---

## 3. 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│  Terraform (관리 영역)                                           │
│                                                                 │
│  aws_ecr_repository.arc_runner                                  │
│    └─ arc-runner (커스텀 이미지 — kubectl + aws cli 포함)        │
│                                                                 │
│  aws_secretsmanager_secret.arc_github_app       (Step 26-1)    │
│    └─ <prefix>-arc-github-app-apne2-secret                     │
│         github_app_id / installation_id / private_key          │
│                                                                 │
│  helm_release.arc_controller                    (Step 26-2)    │
│    └─ arc-system namespace                                      │
│         Deployment: arc-controller-gha-rs-controller           │
│         CRD x4: AutoscalingRunnerSet, EphemeralRunner, 등       │
│                                                                 │
│  aws_iam_role.arc_runner + Pod Identity         (Step 26)      │
│  aws_eks_access_entry.arc_runner (AmazonEKSViewPolicy)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ArgoCD (관리 영역)                                              │
│                                                                 │
│  arc-manifests App  →  kubernetes/arc-runners/                 │
│    └─ arc-runners namespace                                     │
│         ServiceAccount: arc-runner                              │
│                                                                 │
│  arc-runner App     →  helm/arc-runner/values.yaml             │
│    └─ arc-runners namespace                                     │
│         AutoscalingRunnerSet: <prefix>-runner                  │
│           minRunners: 0 / maxRunners: 3                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ESO (External Secrets Operator)                                │
│                                                                 │
│  ExternalSecret: github-runner-secret (arc-runners namespace)  │
│    └─ Secrets Manager → k8s Secret: github-runner-secret       │
│         github_app_id, installation_id, private_key            │
└─────────────────────────────────────────────────────────────────┘
```

### Namespace 구성

| 네임스페이스 | 역할 | 주요 리소스 |
|---|---|---|
| `arc-system` | ARC controller 운영 | Deployment, ServiceAccount, ClusterRole, CRD x4 |
| `arc-runners` | Runner pod 실행 공간 | AutoscalingRunnerSet, runner pod (job 시 생성) |

### Sync Wave 순서

```
wave 3  arc-manifests  → ServiceAccount 생성
wave 4  arc-runner     → AutoscalingRunnerSet 생성 (controller 필요)
```

> controller는 ArgoCD가 아닌 Terraform Helm provider로 관리한다. 이유는 [트러블슈팅 섹션](#9-트러블슈팅)에서 설명한다.
{: .prompt-info }

---

## 4. Terraform 구성

### ECR (arc-runner 커스텀 이미지)

```hcl
# terraform/envs/dev/main.tf
resource "aws_ecr_repository" "arc_runner" {
  name                 = "arc-runner"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

### IAM Role (Pod Identity)

```hcl
resource "aws_iam_role" "arc_runner" {
  name = "${local.prefix}-arc-runner-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action    = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}
```

> **IRSA 방식이 아닌 Pod Identity를 사용하는 이유**<br>
> Pod Identity는 OIDC Provider 없이 클러스터 레벨에서 SA-Role 매핑이 관리된다.<br>
> IRSA보다 설정이 단순하고, EKS 1.24+ 환경에서 AWS가 권장하는 방식이다.
{: .prompt-tip }

**IAM 정책 구성:**

| 정책 | 유형 | 목적 |
|---|---|---|
| `<prefix>-arc-runner-apne2-eks-policy` | inline | `eks:DescribeCluster` — EKS 클러스터 정보 조회 |
| `AmazonEC2ContainerRegistryReadOnly` | AWS managed | ECR에서 커스텀 이미지 pull |

### Pod Identity Association

```hcl
resource "aws_eks_pod_identity_association" "arc_runner" {
  cluster_name    = module.eks.cluster_name
  namespace       = "arc-runners"
  service_account = "arc-runner"
  role_arn        = aws_iam_role.arc_runner.arn
}
```

### EKS Access Entry — 최소 권한

runner pod가 `kubectl rollout status`를 실행할 수 있도록 k8s RBAC 권한을 부여한다. **최소 권한 원칙**에 따라 앱 네임스페이스에만 view 권한을 준다.

```hcl
resource "aws_eks_access_entry" "arc_runner" {
  cluster_name  = module.eks.cluster_name
  principal_arn = aws_iam_role.arc_runner.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "arc_runner" {
  cluster_name  = module.eks.cluster_name
  principal_arn = aws_iam_role.arc_runner.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["<app-namespace>"]   # 앱 네임스페이스만 허용
  }
}
```

> runner pod에 cluster-admin 수준의 권한을 주는 것은 보안상 위험하다.<br>
> 배포 검증에 필요한 것은 `kubectl rollout status`(read-only)뿐이므로 `AmazonEKSViewPolicy`로 충분하다.
{: .prompt-warning }

### GitHub Apps 시크릿

```hcl
resource "aws_secretsmanager_secret" "arc_github_app" {
  name                    = "<prefix>-arc-github-app-apne2-secret"
  description             = "ARC runner GitHub Apps authentication credentials"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "arc_github_app" {
  secret_id = aws_secretsmanager_secret.arc_github_app.id
  secret_string = jsonencode({
    github_app_id              = var.arc_github_app_id
    github_app_installation_id = var.arc_github_app_installation_id
    github_app_private_key     = var.arc_github_app_private_key
  })
  # ignore_changes 없음 — dev.secret.tfvars가 항상 단일 진실 공급원
}
```

```hcl
# terraform/envs/dev/variables.tf
variable "arc_github_app_id"              { type = string; sensitive = true }
variable "arc_github_app_installation_id" { type = string; sensitive = true }
variable "arc_github_app_private_key"     { type = string; sensitive = true }
```

```hcl
# terraform.secret.tfvars (gitignored)
arc_github_app_id              = "<app-id>"
arc_github_app_installation_id = "<installation-id>"
arc_github_app_private_key     = <<-EOT
  -----BEGIN RSA PRIVATE KEY-----
  ...
  -----END RSA PRIVATE KEY-----
EOT
```

### ARC Controller Helm Release (Terraform)

```hcl
resource "helm_release" "arc_controller" {
  name             = "arc-controller"
  namespace        = "arc-system"
  create_namespace = true
  repository       = "oci://ghcr.io/actions/actions-runner-controller-charts"
  chart            = "gha-runner-scale-set-controller"
  version          = "0.10.1"

  values = [file("${path.module}/../../../helm/arc-controller/values.yaml")]

  depends_on = [module.eks]
}
```

> ArgoCD 대신 Terraform Helm provider로 관리하는 이유는 ARC CRD 크기가 Kubernetes annotation 한도(256KB)를 초과하기 때문이다. 자세한 내용은 트러블슈팅 섹션을 참고.
{: .prompt-info }

---

## 5. Helm Values 구성

### Controller (`helm/arc-controller/values.yaml`)

```yaml
tolerations:
  - key: dedicated
    value: system
    effect: NoSchedule

nodeSelector:
  role: system
```

controller pod는 system node(`dedicated=system:NoSchedule` taint)에만 배치된다.

### Runner Scale Set (`helm/arc-runner/values.yaml`)

```yaml
githubConfigUrl: "https://github.com/<github-username>/<repo-name>"
githubConfigSecret: github-runner-secret

controllerServiceAccount:
  namespace: arc-system
  name: arc-controller-gha-rs-controller  # 자동탐색 실패 방지 (트러블슈팅 참고)

listenerTemplate:
  spec:
    containers:
      - name: listener     # CRD required value — 이름만 명시
    tolerations:
      - key: dedicated
        value: system
        effect: NoSchedule
    nodeSelector:
      role: system

runnerScaleSetName: "<prefix>-runner"   # GitHub Actions runs-on에 지정하는 레이블

minRunners: 0    # idle 상태 pod 미생성 (비용 최적화)
maxRunners: 3    # 최대 동시 3개 job

template:
  spec:
    serviceAccountName: arc-runner
    tolerations:
      - key: dedicated
        value: system
        effect: NoSchedule
    nodeSelector:
      role: system
    containers:
      - name: runner
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/arc-runner:<sha7>
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
```

| 설정 키 | 설명 |
|---|---|
| `githubConfigUrl` | runner가 수신할 GitHub 레포 URL |
| `githubConfigSecret` | GitHub Apps 인증 정보가 담긴 K8s Secret |
| `controllerServiceAccount` | controller 자동탐색 실패 방지 |
| `listenerTemplate` | listener pod가 system node에 스케줄링되도록 설정 |
| `runnerScaleSetName` | GitHub Actions `runs-on` 레이블에 지정하는 값 |
| `minRunners: 0` | idle 상태에서 runner pod 미생성 |

---

## 6. GitHub Apps 인증 흐름

### 시크릿 전달 경로

```
dev.secret.tfvars (gitignored)
  arc_github_app_id, installation_id, private_key
          │
          │ terraform apply
          ▼
AWS Secrets Manager
  <prefix>-arc-github-app-apne2-secret
          │
          │ ESO (1h refresh)
          ▼
Kubernetes Secret (arc-runners/github-runner-secret)
          │
          │ helm values.githubConfigSecret
          ▼
ARC Controller
  Private Key로 JWT 서명
  → GitHub API로 Installation Access Token 발급
  → runner 등록 및 job 수신
```

### ExternalSecret 정의

```yaml
# kubernetes/external-secrets/external-secret-github-runner.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: github-runner-secret
  namespace: arc-runners
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: github-runner-secret
    creationPolicy: Owner
  data:
    - secretKey: github_app_id
      remoteRef:
        key: <prefix>-arc-github-app-apne2-secret
        property: github_app_id
    - secretKey: github_app_installation_id
      remoteRef:
        key: <prefix>-arc-github-app-apne2-secret
        property: github_app_installation_id
    - secretKey: github_app_private_key
      remoteRef:
        key: <prefix>-arc-github-app-apne2-secret
        property: github_app_private_key
```

### GitHub Apps 필요 권한

| 권한 | 레벨 | 필요 이유 |
|---|---|---|
| Actions | Read & Write | runner 등록 및 job 수신 |
| Administration | Read & Write | runner registration token 발급 (필수) |
| Metadata | Read | 레포 기본 정보 조회 |

> **Administration 권한 주의사항**<br>
> GitHub Apps 권한을 변경하면 기존 installation에서 **별도로 승인(Review and accept)**을 해야 새 권한이 적용된다.<br>
> `https://github.com/settings/installations/{installation_id}` 에서 승인 후 controller를 재시작해야 한다.
{: .prompt-danger }

---

## 7. ArgoCD 앱 구성

### arc-manifests

```yaml
# kubernetes/argocd/application-arc-manifests.yaml
metadata:
  name: <prefix>-arc-manifests
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  source:
    path: kubernetes/arc-runners
  destination:
    namespace: arc-runners
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### arc-runner

```yaml
# kubernetes/argocd/application-arc-runner.yaml
metadata:
  name: <prefix>-arc-runner
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  sources:
    - repoURL: ghcr.io/actions/actions-runner-controller-charts
      chart: gha-runner-scale-set
      targetRevision: "0.10.1"     # controller와 반드시 동일 버전
      helm:
        releaseName: arc-runner
        valueFiles:
          - $values/helm/arc-runner/values.yaml
    - repoURL: git@github.com:<github-username>/<repo-name>.git
      targetRevision: main
      ref: values
  destination:
    namespace: arc-runners
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

> **controller 버전과 runner scale set 버전은 반드시 일치해야 한다.**<br>
> 버전이 다르면 JIT config 형식 불일치로 runner pod가 job을 받지 못하고 즉시 종료된다.
{: .prompt-danger }

---

## 8. arc-runner 커스텀 이미지

### Dockerfile

```dockerfile
# applications/arc-runner/Dockerfile
FROM ghcr.io/actions/actions-runner:2.335.1

USER root

ARG KUBECTL_VERSION=v1.29.0

RUN curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
      -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

RUN apt-get update -y \
    && apt-get install -y --no-install-recommends unzip \
    && curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
         -o /tmp/awscliv2.zip \
    && unzip /tmp/awscliv2.zip -d /tmp \
    && /tmp/aws/install \
    && rm -rf /tmp/aws /tmp/awscliv2.zip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER runner
```

### CI 파이프라인 — sha7 tag + writeback

`purina-nextjs`, `purina-api`와 동일하게 **sha7 tag + CI writeback** 패턴을 적용한다. CI가 `helm/arc-runner/values.yaml`의 image 태그를 직접 커밋하여 어떤 이미지가 배포됐는지 git 이력으로 추적 가능하다.

```yaml
# .github/workflows/<prefix>-gitactions-arc-runner-apne2-pipe.yml
jobs:
  build:
    runs-on: ubuntu-latest  # GitHub-hosted (chicken-and-egg 방지)
    permissions:
      id-token: write
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      - uses: aws-actions/amazon-ecr-login@v2

      - name: Compute image tag
        id: tag
        run: echo "tag=$(echo $GITHUB_SHA | cut -c1-7)" >> $GITHUB_OUTPUT

      - uses: docker/build-push-action@v5
        with:
          context: applications/arc-runner/
          push: true
          tags: |
            ${{ env.IMAGE_URI }}:${{ steps.tag.outputs.tag }}
            ${{ env.IMAGE_URI }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Update values.yaml image tag
        run: |
          sed -i "s|image: ${{ env.IMAGE_URI }}:.*|image: ${{ env.IMAGE_URI }}:${{ steps.tag.outputs.tag }}|" \
            helm/arc-runner/values.yaml
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add helm/arc-runner/values.yaml
          git diff --staged --quiet || git commit -m "ci: update arc-runner image tag to ${{ steps.tag.outputs.tag }}"
          git push
```

> arc-runner 이미지 빌드 workflow는 **GitHub-hosted runner**에서 실행한다.<br>
> self-hosted runner가 아직 없는 시점에 이미지를 처음 빌드해야 하므로, chicken-and-egg 문제가 없다.
{: .prompt-tip }

---

## 9. 트러블슈팅

### ARC CRD 256KB 초과 — Terraform으로 전환

**증상**

```
CustomResourceDefinition "autoscalingrunnersets.actions.github.com" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

**원인 분석**

ARC chart가 설치하는 CRD 4개 중 `autoscalingrunnersets.actions.github.com`의 스펙 크기가 **518KB**다. kubectl client-side apply는 전체 manifest를 `kubectl.kubernetes.io/last-applied-configuration` annotation에 저장하는데, Kubernetes의 annotation 최대 크기(256KB)를 2배 초과한다.

ArgoCD `ServerSideApply=true`를 적용해도 518KB 규모의 CRD는 서버 사이드 필드 추적에서도 동일 한도 초과가 발생한다.

**해결: Terraform Helm provider**

Helm provider는 Kubernetes API에 직접 chart를 설치하며, ArgoCD의 annotation 기반 상태 추적 구조를 거치지 않아 CRD 크기 제약이 없다.

```hcl
resource "helm_release" "arc_controller" {
  # ...ArgoCD가 아닌 Terraform으로 관리
}
```

---

### Controller 자동탐색 실패

**증상**

```
No gha-rs-controller deployment found using label
(app.kubernetes.io/part-of=gha-rs-controller).
Consider setting controllerServiceAccount.name in values.yaml.
```

**원인**

`gha-runner-scale-set` chart는 동일 클러스터 내 controller를 레이블로 자동 탐색한다. Terraform Helm release 이름이 변경되면서 레이블 패턴이 달라져 탐색에 실패했다.

**해결**

`controllerServiceAccount`를 명시적으로 설정한다.

```yaml
controllerServiceAccount:
  namespace: arc-system
  name: arc-controller-gha-rs-controller
```

---

### Listener Pod Pending — listenerTemplate containers 필수값

**증상**

```
Warning  FailedScheduling  karpenter  did not tolerate dedicated=web:NoSchedule;
                                       did not tolerate dedicated=system:NoSchedule
```

**원인**

Listener pod는 ARC controller가 내부적으로 생성하는 pod로, `listenerTemplate`을 명시하지 않으면 toleration이 없어 어느 노드에도 스케줄링되지 않는다.

**1차 시도 실패**

```yaml
# containers 없이 toleration만 추가
listenerTemplate:
  spec:
    tolerations:
      - key: dedicated ...
```

```
AutoscalingRunnerSet "..." is invalid:
spec.listenerTemplate.spec.containers: Required value
```

ARC CRD가 `listenerTemplate.spec.containers`를 필수값으로 요구한다.

**해결**

```yaml
listenerTemplate:
  spec:
    containers:
      - name: listener   # 이름만 명시, 실제 이미지/커맨드 변경 없음
    tolerations:
      - key: dedicated
        value: system
        effect: NoSchedule
    nodeSelector:
      role: system
```

---

### GitHub App Administration 권한 누락 (403 에러)

**증상**

```
ERROR: "failed to get runner registration token on refresh:
github api error: StatusCode 403:
{\"message\":\"Resource not accessible by integration\"}"
```

**원인**

runner registration token 발급 엔드포인트에는 **Repository permission: Administration — Read & Write**가 필요하다. 초기 App 생성 시 `Actions` 권한만 부여하여 발생했다.

**해결 절차**

```
1. GitHub App 설정 → Permissions → Administration: Read and write → Save
2. https://github.com/settings/installations/{installation_id} → Review and accept
3. controller pod 재시작 (캐시된 토큰 갱신)
   kubectl rollout restart deployment/arc-controller-gha-rs-controller -n arc-system
```

> GitHub Apps 권한을 변경하면 기존 installation에서 **별도 승인**이 필요하다. 승인 전에는 새 토큰이 발급되어도 동일한 403이 반환된다.
{: .prompt-danger }

---

### ExternalSecret Degraded — 시크릿 미존재

**증상**

```
error processing spec.data[0]:
  Secret does not exist: <prefix>-github-runner-apne2-secret
```

**원인**

PAT 방식용 시크릿이 Secrets Manager에 실제로 생성되지 않은 상태였다.

**해결**

PAT 방식을 완전히 제거하고 GitHub Apps 방식으로 전환하면서 동시에 해결했다.

---

## 10. GitHub Actions Workflow에서 ARC Runner 사용

```yaml
# .github/workflows/deploy.yml
jobs:
  smoke-test:
    runs-on: <prefix>-runner   # helm/arc-runner/values.yaml의 runnerScaleSetName
    timeout-minutes: 20        # 무한 대기 방지
    steps:
      - name: Configure kubectl
        run: aws eks update-kubeconfig --name <cluster-name> --region ap-northeast-2

      - name: Wait for ArgoCD sync
        run: |
          # ArgoCD가 새 이미지 감지할 때까지 폴링 (최대 10분)
          for i in $(seq 1 60); do
            CURRENT=$(kubectl get deployment <deployment-name> -n <namespace> \
              -o jsonpath='{.spec.template.spec.containers[0].image}')
            if [[ "$CURRENT" == *"$IMAGE_TAG"* ]]; then
              echo "ArgoCD sync completed"
              break
            fi
            echo "Waiting... ($i/60)"
            sleep 10
          done

      - name: Verify rollout
        run: |
          kubectl rollout status deployment/<deployment-name> \
            -n <namespace> \
            --timeout=5m
```

---

## 11. 장애 분석 1 — Runner Version Deprecated

### 증상

```
# GitHub Actions
Rollout Verification: Waiting for a runner to pick up this job... (무한 대기)

# EKS
kubectl get ephemeralrunner -n arc-runners
# STATUS: Failed — TooManyPodFailures
```

Runner pod가 생성되고 2~3초 만에 exit 0으로 종료됐다.

### 원인

GitHub이 v2.321.0 runner를 deprecated 처리하여 broker API에서 job 메시지 수신을 차단했다.

```
[RUNNER ERR] WRITE ERROR:
  Runner version v2.321.0 is deprecated and cannot receive messages.
```

### 디버깅 — Debug Wrapper 적용

ARC는 runner pod 종료 즉시 삭제하므로 `kubectl logs`로는 포착이 불가능하다. `values.yaml`에 임시 debug wrapper를 적용해 pod을 살려둔다.

```yaml
# helm/arc-runner/values.yaml (임시)
template:
  spec:
    containers:
      - name: runner
        command: ["/bin/bash", "-c"]
        args:
          - "/home/runner/run.sh 2>&1; echo exit:$? > /dev/termination-log; sleep 300"
```

runner가 종료되어도 pod이 300초 동안 유지되어 로그를 확인할 수 있다.

### 수정

```dockerfile
# applications/arc-runner/Dockerfile
# Before
FROM ghcr.io/actions/actions-runner:2.321.0

# After
FROM ghcr.io/actions/actions-runner:2.334.0
```

### Ghost Runner 해결

GitHub scale set 내부에 `totalBusyRunners: 1`, `totalRegisteredRunners: 0` stale 상태가 남았다. AutoscalingRunnerSet을 삭제하면 ArgoCD selfHeal이 즉시 재생성하면서 새 세션으로 재등록한다.

```bash
kubectl delete autoscalingrunnerset <prefix>-runner -n arc-runners
# ArgoCD selfHeal이 즉시 재생성 → GitHub에 새 scale set ID로 등록
```

---

## 12. 장애 분석 2 — ARC 0.9.3 + v2.335.1 JIT Session Conflict

### 증상

v2.334.0도 deprecated 처리되어 v2.335.1로 업그레이드 후 동일한 증상이 재발했다. 이번에는 로그가 **완전히 비어있었다.**

### 원인 — 2단계 복합 장애

**원인 1**: v2.334.0 deprecated

**원인 2**: ARC 0.9.3 + v2.335.1 JIT 비호환

> **JIT(Just-in-Time) 동작 원리**<br>
> ARC Controller가 GitHub API로 JIT Registration Token을 발급받아 `ACTIONS_RUNNER_INPUT_JITCONFIG` 환경변수로 runner pod에 주입한다.<br>
> ARC 0.9.3이 생성하는 JIT config 형식을 v2.335.1 runner가 처리하지 못해, 등록은 성공하지만 job 실행 없이 조용히 종료됐다.
{: .prompt-warning }

### 수정

**Fix 1 — ARC 0.9.3 → 0.10.1 업그레이드**

```hcl
# terraform/envs/dev/main.tf
resource "helm_release" "arc_controller" {
  version = "0.10.1"
}
```

```bash
terraform apply -var-file="dev.secret.tfvars" \
  -target=helm_release.arc_controller -auto-approve
```

```yaml
# kubernetes/argocd/application-arc-runner.yaml
targetRevision: "0.10.1"   # controller와 반드시 동일 버전
```

**Fix 2 — EphemeralRunner Deadlock 자동화 (CronJob)**

Failed EphemeralRunner가 `currentReplicas` 카운트에 포함되어 `current == desired`가 되면 controller가 새 runner를 만들지 않는 deadlock이 발생한다. 5분마다 자동 정리하는 CronJob을 배포했다.

```yaml
# kubernetes/arc-runners/ephemeralrunner-cleanup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ephemeralrunner-cleanup
  namespace: arc-runners
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: arc-runner-cleanup
          containers:
            - name: cleanup
              image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/arc-runner:latest
              command: ["/bin/bash", "-c"]
              args:
                - |
                  FAILED=$(kubectl get ephemeralrunner -n arc-runners -o json \
                    | jq -r '.items[] | select(.status.phase=="Failed") | .metadata.name')
                  if [ -n "$FAILED" ]; then
                    echo "$FAILED" | xargs kubectl delete ephemeralrunner -n arc-runners
                  fi
```

**Fix 3 — Workflow Concurrency 설정**

동일 브랜치에 연속 push 시 이전 run이 runner를 점유한 채 blocking하는 패턴을 방지한다.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### 현재 버전 구성

| 컴포넌트 | 버전 |
|---|---|
| ARC Controller | 0.10.1 |
| gha-runner-scale-set | 0.10.1 |
| runner base image | ghcr.io/actions/actions-runner:**2.335.1** |

---

## 13. Dependabot 자동 의존성 업데이트

### 도입 배경

runner 이미지 버전이 deprecated됐는데 수동으로 추적하지 못한 것이 장애의 근본 원인이었다. Dependabot을 통해 버전 드리프트를 조기에 감지하고 PR로 제어한다.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/applications/arc-runner"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Seoul"
    labels:
      - "dependencies"
      - "arc-runner"
    commit-message:
      prefix: "chore(arc)"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Seoul"
    labels:
      - "dependencies"
      - "github-actions"
    commit-message:
      prefix: "chore(ci)"
```

### Dependabot PR 처리 절차

> Dependabot PR이 올라오면 **무조건 바로 merge하지 않는다.**<br>
> ARC controller 버전과 runner 버전 간 호환성을 반드시 교차 확인해야 한다.
{: .prompt-danger }

```
Dependabot PR 수신
        │
        ▼
1. 새 runner 버전 확인 (예: 2.335.1 → 2.336.0)
        │
        ▼
2. ARC release notes 교차 확인
   https://github.com/actions/actions-runner-controller/releases
   → 현재 ARC 버전이 새 runner 버전을 지원하는지 확인
        │
        ├─ 호환됨 → PR merge → CI에서 ECR 이미지 자동 빌드
        │
        └─ 비호환 → ARC 먼저 업그레이드 필요
                    terraform apply -target=helm_release.arc_controller
                    application-arc-runner.yaml targetRevision 업데이트
                    → 그 후 Dependabot PR merge
```

---

## 14. 운영 가이드

### Controller 버전 업그레이드

controller와 runner scale set 버전은 반드시 일치해야 한다.

```bash
# Terraform으로 controller 업그레이드
cd terraform/envs/dev
terraform apply -var-file="dev.secret.tfvars" \
  -target=helm_release.arc_controller -auto-approve

# application-arc-runner.yaml의 targetRevision도 동일 버전으로 업데이트 후 git push
```

### GitHub Apps Private Key 교체

```bash
# 1. GitHub Apps 설정 → Private keys → Generate a private key
# 2. dev.secret.tfvars의 arc_github_app_private_key 교체
# 3. terraform apply
terraform apply -var-file="dev.secret.tfvars"

# 4. ESO 즉시 동기화 (기본 1시간 대기 없이)
kubectl annotate externalsecret github-runner-secret \
  -n arc-runners \
  force-sync=$(date +%s) --overwrite
```

### 모니터링 및 디버깅

```bash
# Controller 상태
kubectl get pods -n arc-system
kubectl logs -n arc-system \
  -l app.kubernetes.io/name=gha-rs-controller -f

# Runner 상태 (job 실행 중일 때만 pod 존재)
kubectl get pods -n arc-runners
kubectl get autoscalingrunnersets -n arc-runners
kubectl get ephemeralrunner -n arc-runners

# GitHub Apps 시크릿 동기화 상태
kubectl get externalsecret github-runner-secret -n arc-runners
```

### 장애 진단 플로우

```
runner pod가 2~3초 만에 Completed/Failed
        │
        ├─ kubectl logs 비어있음?
        │       → Debug wrapper 적용 (helm/arc-runner/values.yaml 임시 수정)
        │
        ├─ EphemeralRunner에 runnerId 있음?
        │       NO  → GitHub Apps 인증 문제 (Administration 권한 확인)
        │       YES → runner 등록은 성공, job 미실행
        │
        ├─ controller 로그에 "Runner exists in GitHub service"?
        │       YES → Stale 리스너 세션 → AutoscalingRunnerSet 삭제
        │
        ├─ runner 버전 deprecated?
        │       → github.com/actions/runner/releases 최신 버전 확인
        │         Dockerfile 업그레이드 → CI 빌드 트리거
        │
        └─ ARC 버전 비호환?
                → controller 버전과 runner scale set 버전 일치 여부 확인
                  terraform apply -target=helm_release.arc_controller
```

---

## 파일 구조

```
purina-kr-infra/
├── .github/
│   ├── dependabot.yml
│   └── workflows/
│       └── <prefix>-gitactions-arc-runner-apne2-pipe.yml
│
├── applications/
│   └── arc-runner/
│       └── Dockerfile
│
├── terraform/envs/dev/
│   ├── main.tf          (Step 26, 26-1, 26-2)
│   └── variables.tf     (arc_github_app_* 변수)
│
├── helm/
│   ├── arc-controller/
│   │   └── values.yaml
│   └── arc-runner/
│       └── values.yaml
│
└── kubernetes/
    ├── argocd/
    │   ├── application-arc-runner.yaml
    │   └── application-arc-manifests.yaml
    ├── arc-runners/
    │   ├── serviceaccount.yaml
    │   └── ephemeralrunner-cleanup.yaml
    └── external-secrets/
        └── external-secret-github-runner.yaml
```
