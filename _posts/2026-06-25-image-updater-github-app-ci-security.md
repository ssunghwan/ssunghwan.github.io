---
title: "Adoption of ArgoCD Image Updater + Switch to GitHub App HTTPS + Strengthening CI Security"
date: 2026-06-25 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, argocd, image-updater, github-app, gitops, cicd, trivy, ecr, kustomize, pod-identity, security]
---

> 기존 CI에서 git push로 이미지 태그를 직접 업데이트하던 방식을 ArgoCD Image Updater 기반으로 전환했다.<br>
> SSH deploy key 기반 레포 인증을 GitHub App HTTPS 인증으로 교체하고, Trivy 사전 스캔과 npm audit 개선으로 CI 보안도 강화했다.
{: .prompt-info }

---

## 1. 도입 배경 — 기존 CI git push 방식의 문제

### 기존 방식

```
GitHub Actions CI
  └── 1. infra repo checkout
  └── 2. sed -i "s/old-tag/new-tag/" kubernetes/apps/kustomization.yaml
  └── 3. git commit + git push origin main
  └── 4. ArgoCD가 변경 감지 → sync → 배포
```

단순하고 직관적이지만, 실제 운영에서는 아래 문제들이 반복된다.

**레이스 컨디션**

nextjs PR과 api PR이 거의 동시에 merge되면 두 CI가 각자 `kustomization.yaml`을 수정 후 push한다. 선행 push가 ArgoCD에 반영되기 전에 후행 push가 덮어씌우면 하나의 배포가 누락된다.

```
CI-nextjs: sed s/old-tag/new-nextjs-tag/ → push
CI-api:    sed s/old-tag/new-api-tag/    → push (nextjs 변경 덮어씌움)
                                          → nextjs는 배포 안 됨 ❌
```

**CI와 인프라의 강결합**

CI의 역할이 "빌드 + ECR push"를 넘어 "인프라 레포 상태 변경"까지 확장된다. 단일 책임 원칙(SRP)을 위반한다.

**GitOps 원칙 위반**

"클러스터 상태는 항상 git에 의해서만 변경된다"는 GitOps 원칙에서, CI가 git을 직접 조작하면 누가 언제 무엇을 변경했는지 추적하기 어려워진다.

### 목표 아키텍처

```
GitHub Actions CI
  └── 1. 코드 품질/보안 검사 (quality gate)
  └── 2. Docker 이미지 빌드 + Trivy 스캔
  └── 3. ECR push (스캔 통과 시에만)
  └── 4. smoke-test (Image Updater 배포 완료 대기)

ArgoCD Image Updater (in-cluster, 2분 간격)
  └── ECR에서 newest-build 태그 감지
  └── kustomization.yaml 자동 업데이트
  └── GitHub App으로 main 브랜치에 git commit + push

ArgoCD
  └── kustomization.yaml 변경 감지 → sync → kubectl apply
```

> **역할 분리의 핵심**<br>
> CI의 역할 = **ECR에 이미지를 올리는 것**<br>
> 이미지 태그 관리와 배포 = **Image Updater + ArgoCD**<br>
> CI는 더 이상 인프라 레포를 건드리지 않는다.
{: .prompt-tip }

---

## 2. ArgoCD Image Updater란

### v0.x(annotation) vs v1.x(CRD) 차이

2024년 이후 v1.x 계열(CRD 기반)이 출시되어 기존 v0.x(annotation 기반)와 설계가 완전히 달라졌다.

| 항목 | v0.x (annotation 기반) | v1.x (CRD 기반) |
|---|---|---|
| 설정 위치 | ArgoCD Application의 annotation | `ImageUpdater` CRD 별도 리소스 |
| 설정 예시 | `argocd-image-updater.argoproj.io/image-list: ...` | `kind: ImageUpdater` → `applicationRefs` |
| 권장 여부 | Deprecated | 현재 권장 (프로덕션 표준) |
| Helm chart 버전 | 0.9.x, 0.10.x | 1.2.x |

> **v0.x 문서와 혼용하면 설정이 전혀 동작하지 않는다.**<br>
> 인터넷에 v0.x 예제가 훨씬 많으므로 버전 확인이 매우 중요하다. 이 프로젝트에서는 **chart 1.2.2 (app v1.2.1)** 을 사용한다.
{: .prompt-danger }

### newest-build 전략

Image Updater가 지원하는 업데이트 전략 중 `newest-build`는 **ECR에 push된 시간 기준으로 가장 최신 이미지를 선택**한다.

이 프로젝트의 이미지 태그 형식: `20260625-ace308b` (날짜-SHA 앞 7자)

`semver` 전략과 달리 `newest-build`는 태그 문자열의 의미를 분석하지 않고, ECR의 `imagePushedAt` 타임스탬프를 기준으로 정렬한다. 태그 형식에 관계없이 가장 최근에 push된 이미지를 항상 선택하므로 이 프로젝트의 날짜-SHA 태그 형식과 완벽하게 호환된다.

---

## 3. Terraform 모듈 설계

### 모듈 분리 배경

초기 구현에서는 IAM Role, ECR 정책, Pod Identity Association, ConfigMap을 모두 `main.tf`에 inline으로 작성했다. 다른 컴포넌트들(`module "eso"`, `module "albc"` 등)은 모두 별도 모듈로 분리되어 있으므로 일관성을 위해 동일 패턴으로 분리했다.

### 모듈 파일 구조

```
terraform/modules/argocd-image-updater/
├── main.tf       — IAM Role, ECR Policy, Pod Identity, ECR creds ConfigMap
├── variables.tf  — prefix, cluster_name, account_id, region
└── outputs.tf    — role_arn
```

### main.tf 상세 설명

```hcl
# IAM Role — EKS Pod Identity 방식
# pods.eks.amazonaws.com이 Principal인 이유:
#   IRSA는 sts:AssumeRoleWithWebIdentity를 사용하지만
#   Pod Identity는 sts:AssumeRole + sts:TagSession을 사용하는 새 방식이다.
resource "aws_iam_role" "this" {
  name = "${var.prefix}-image-updater-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action    = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}

# ECR 읽기 정책
resource "aws_iam_role_policy" "ecr" {
  name = "${var.prefix}-image-updater-ecr-apne2-policy"
  role = aws_iam_role.this.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # GetAuthorizationToken: Resource = "*" 필수
        # ECR 토큰은 레지스트리 레벨 작업이라 특정 repository ARN으로 범위 지정 불가
        Effect   = "Allow"
        Action   = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability",   # newest-build 전략에 필요
          "ecr:DescribeImages",
          "ecr:DescribeRepositories",
          "ecr:GetDownloadUrlForLayer",         # newest-build 전략에 필요
          "ecr:ListImages"
        ]
        Resource = "arn:aws:ecr:${var.region}:${var.account_id}:repository/*"
      }
    ]
  })
}

# Pod Identity Association
# service_account = "argocd-image-updater": Helm chart가 이 이름으로 SA를 생성한다.
# Terraform이 먼저 Association을 설정해도 SA가 없으면 pending 상태로 대기하다가
# Helm 설치 후 자동으로 연결된다.
resource "aws_eks_pod_identity_association" "this" {
  cluster_name    = var.cluster_name
  namespace       = "argocd"
  service_account = "argocd-image-updater"
  role_arn        = aws_iam_role.this.arn
}

# ECR Credential Helper ConfigMap
# Terraform으로 관리하는 이유:
#   ArgoCD Application(GitOps)으로 관리하면 sync가 비동기라 순서 보장이 없다.
#   Helm Pod가 먼저 시작될 경우 ConfigMap이 아직 없어 FailedMount 에러 발생.
#   Terraform의 depends_on으로 "ConfigMap 생성 → Helm Pod 시작" 순서를 보장한다.
resource "kubernetes_config_map" "ecr_creds" {
  metadata {
    name      = "argocd-image-updater-ecr-creds"
    namespace = "argocd"
  }

  data = {
    "ecr-creds.sh" = <<-EOT
      #!/bin/sh
      echo "AWS:$(aws ecr get-login-password --region ap-northeast-2)"
    EOT
  }
}
```

### terraform/envs/dev/main.tf 호출부

```hcl
module "argocd_image_updater" {
  source       = "../../modules/argocd-image-updater"
  prefix       = local.prefix
  cluster_name = module.eks.cluster_name
  account_id   = local.account_id
  region       = local.region
  depends_on   = [module.eks]
}

resource "helm_release" "argocd_image_updater" {
  name       = "argocd-image-updater"
  namespace  = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argocd-image-updater"
  version    = "1.2.2"
  values     = [file("${path.module}/../../../helm/argocd-image-updater/values.yaml")]

  # module.argocd_image_updater 안의 kubernetes_config_map.ecr_creds가
  # 반드시 먼저 생성된 후 Helm Pod가 시작돼야 FailedMount 없이 기동된다.
  depends_on = [module.argocd_image_updater]
}
```

---

## 4. ECR 인증 — Pod Identity + credential helper

### 인증 흐름

```
argocd-image-updater Pod
  │
  ├── Pod Identity Association 설정됨
  │   → AWS_CONTAINER_CREDENTIALS_RELATIVE_URI 자동 주입
  │
  ├── ECR API 호출 전: credentials: ext:/scripts/ecr-creds.sh 실행
  │     aws ecr get-login-password --region ap-northeast-2
  │       └── Pod Identity → IAM Role assume → ECR 토큰(12시간) 발급
  │     echo "AWS:<token>"
  │
  └── Image Updater가 "AWS:<token>" 파싱
        username = "AWS"
        password = <token>
        → ECR API 인증 성공
```

### credential helper 방식을 선택한 이유

| 방법 | 설명 | 문제점 |
|---|---|---|
| `ext:/scripts/ecr-creds.sh` | 스크립트 실행 결과로 인증 | 없음 (현재 방식) ✅ |
| `secret:네임스페이스/시크릿명` | K8s 시크릿의 docker config 사용 | ECR 토큰은 12시간 만료 → 갱신 CronJob 필요 |
| `env:AWS_ACCESS_KEY_ID` | 환경변수 기반 장기 키 | 장기 자격증명 노출 위험 |

> `ext:` 방식은 Pod Identity와 결합하면 **장기 자격증명이 전혀 없고** 토큰 갱신도 자동이다. 가장 보안이 높고 운영 부담이 없는 방식이다.
{: .prompt-tip }

### ConfigMap `defaultMode: 0755` 필수

```yaml
volumes:
  - name: ecr-creds-script
    configMap:
      name: argocd-image-updater-ecr-creds
      defaultMode: 0755   # 실행 권한 없으면 Permission denied 에러 발생
```

> ConfigMap으로 마운트된 파일은 기본적으로 실행 권한(`x`)이 없다.<br>
> `ext:` credential 방식은 해당 파일을 직접 실행(`execve`)하므로 반드시 `0755` 이상을 설정해야 한다.
{: .prompt-warning }

---

## 5. Helm values 구성

```yaml
# helm/argocd-image-updater/values.yaml

config:
  registries:
    - name: ECR-apne2
      # ECR API endpoint
      api_url: https://<account-id>.dkr.ecr.ap-northeast-2.amazonaws.com
      # 이 prefix로 시작하는 이미지를 이 레지스트리로 라우팅
      prefix: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com
      # ECR은 /v2/ ping endpoint가 없어 true로 설정하면 오류 발생
      ping: false
      insecure: false
      # ext: 방식으로 스크립트 실행 결과를 인증에 사용
      credentials: ext:/scripts/ecr-creds.sh
      # ⚠️ credsExpire 필드는 v1.2.x에서 제거됨 — 절대 추가하지 말 것

  # git write-back 커밋 설정
  git.user: argocd-image-updater
  git.email: noreply@github.com
  git.commit-message-template: "ci: auto-update image {{range .AppImages}}{{.Image.String}} {{end}}"

# ECR creds 스크립트 마운트
volumes:
  - name: ecr-creds-script
    configMap:
      name: argocd-image-updater-ecr-creds
      defaultMode: 0755

volumeMounts:
  - name: ecr-creds-script
    mountPath: /scripts

tolerations:
  - key: dedicated
    operator: Equal
    value: system
    effect: NoSchedule

nodeSelector:
  role: system

# aws cli가 올바른 리전을 사용하도록 환경변수 주입
# AWS_REGION과 AWS_DEFAULT_REGION 모두 설정 — SDK 버전별로 참조하는 변수가 다름
extraEnv:
  - name: AWS_REGION
    value: ap-northeast-2
  - name: AWS_DEFAULT_REGION
    value: ap-northeast-2
```

### v0.x vs v1.x values 주요 차이

| v0.x 키 | v1.x 키 | 비고 |
|---|---|---|
| `config.gitCommitUser` | `config.git.user` | 키 이름 변경 |
| `config.gitCommitMail` | `config.git.email` | 키 이름 변경 |
| `config.interval` | `extraArgs: ["--interval", "2m"]` | 기본값 2m이므로 생략 가능 |
| `registries[].credsExpire` | **제거됨** | v1.2.x에서 RegistryConfiguration 구조체에서 삭제 |

---

## 6. GitOps 구조

### App of Apps 패턴에서의 위치

```
<prefix>-appsync-apne2-cluster (App of Apps)
  └── kubernetes/argocd/application-*.yaml
        ├── application-external-secrets-operator.yaml  (wave 1)
        ├── application-albc.yaml                       (wave 2)
        ├── application-kube-prometheus-stack.yaml      (wave 4)
        ├── application-apps.yaml                       (wave 6) ← 앱 워크로드
        └── application-image-updater.yaml              (wave 7) ← Image Updater CRD
```

> **Image Updater는 반드시 apps(wave 6) 이후에 배포돼야 한다.**<br>
> `<prefix>-apps` Application이 먼저 존재해야 Image Updater가 해당 Application을 감시 대상으로 등록할 수 있기 때문이다.
{: .prompt-warning }

### application-image-updater.yaml

```yaml
# kubernetes/argocd/application-image-updater.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-image-updater
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "7"
spec:
  project: default
  source:
    repoURL: https://github.com/<github-username>/<repo-name>.git
    targetRevision: main
    path: kubernetes/image-updater
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

> **파일명 규칙 주의**<br>
> `appsync`는 `application-*.yaml` glob 패턴으로 파일을 탐색한다.<br>
> `kubernetes/image-updater/image-updater-cr.yaml`은 Application 파일이 아닌 **ImageUpdater CRD 리소스 자체**이므로 별도 경로에 있어도 된다.
{: .prompt-info }

### ImageUpdater CRD 리소스 상세

```yaml
# kubernetes/image-updater/image-updater-cr.yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: <prefix>-apps
  namespace: argocd
spec:
  writeBackConfig:
    method: git
    gitConfig:
      branch: main
      writeBackTarget: "kustomization:."
      # "kustomization:." 설명:
      #   Image Updater 최종 경로 = [App spec.source.path] + [writeBackTarget 경로]
      #   App source.path = "kubernetes/apps"
      #   writeBackTarget = "kustomization:."  → 경로 = "."
      #   최종 경로 = "kubernetes/apps" ← 올바른 kustomization.yaml 위치
      #
      #   ❌ writeBackTarget = "kustomization:./kubernetes/apps"
      #   최종 경로 = "kubernetes/apps/kubernetes/apps" ← 이중 경로, 오류

  applicationRefs:
    - namePattern: <prefix>-apps
      images:
        - alias: nextjs
          imageName: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-nextjs
          commonUpdateSettings:
            updateStrategy: newest-build
          manifestTargets:
            kustomize:
              # kustomization.yaml의 images[].name 과 정확히 일치해야 함
              name: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-nextjs

        - alias: api
          imageName: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-api
          commonUpdateSettings:
            updateStrategy: newest-build
          manifestTargets:
            kustomize:
              name: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-api
```

**`manifestTargets.kustomize.name` 필드 설명:**

```yaml
# kubernetes/apps/kustomization.yaml — Image Updater가 자동으로 업데이트하는 파일
images:
  - name: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-nextjs  # ← name으로 매칭
    newTag: 20260625-ace308b                                                  # ← 이 값을 업데이트
```

---

## 7. GitHub App HTTPS 인증 전환

### 기존 상태와 문제

기존 ArgoCD 레포 시크릿은 SSH deploy key 기반이었다.

```
type: git
url: git@github.com:<owner>/<repo>.git   ← SSH
sshPrivateKey: <deploy key>               ← read-only
```

GitHub deploy key는 기본적으로 read-only다. Image Updater가 kustomization.yaml을 커밋 + push하려고 시도하면:

```
ERROR: The key you are authenticating with has been marked as read only.
fatal: Could not read from remote repository.
```

### 해결 방안 — GitHub App HTTPS 인증

이미 ARC runner가 사용 중인 Secrets Manager 시크릿을 재사용한다.

| 방법 | 장점 | 단점 |
|---|---|---|
| Write 권한 새 SSH deploy key | 간단 | 장기 키 관리 필요 |
| **GitHub App HTTPS** | 네이티브 자동 갱신, 기존 App 재사용 ✅ | GitHub App `Contents: write` 권한 필요 |

### GitHub App 네이티브 인증 동작 원리

ArgoCD repo-server는 GitHub App 자격증명을 인식하면 다음 과정을 **자동으로** 수행한다.

```
1. Private Key로 JWT 생성 (RS256, 만료 10분)
2. GitHub API /app/installations/{id}/access_tokens 호출
3. Installation Access Token 발급 (유효시간 60분)
4. https://x-access-token:<token>@github.com/... 형태로 git 인증
5. 토큰 만료 전 자동 갱신
```

> 이전에 구현했던 CronJob(50분마다 Python 스크립트로 토큰 갱신)이 하던 일을 ArgoCD repo-server가 완전히 대신한다.<br>
> CronJob, RBAC, ServiceAccount 등 관련 리소스 전체를 제거할 수 있다.
{: .prompt-tip }

### GitHub App Contents: write 권한 필요

Image Updater가 kustomization.yaml을 커밋 + push하려면 레포지토리 쓰기 권한이 필요하다.

```
GitHub → Settings → Developer settings → GitHub Apps → 앱 선택
→ Permissions & events → Repository permissions
→ Contents: Read and write → Save
→ 조직/레포 설치 승인 (Review and accept)
```

> 권한을 변경한 뒤 설치 레벨에서 **별도 승인**이 필요하다.<br>
> 승인 전에는 새 토큰이 발급되어도 push 시 403이 반환된다.
{: .prompt-danger }

### ESO로 ArgoCD 레포 시크릿 생성

ARC runner가 사용하는 동일한 Secrets Manager 시크릿을 재사용한다.

```yaml
# kubernetes/external-secrets/external-secret-image-updater.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-repo-github-app
  namespace: argocd
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: repo-purina-kr-infra-github-app
    creationPolicy: Owner
    template:
      metadata:
        labels:
          # ArgoCD가 이 레이블이 붙은 Secret을 레포지토리 자격증명으로 인식
          argocd.argoproj.io/secret-type: repository
      data:
        type: git
        url: https://github.com/<github-username>/<repo-name>.git
        githubAppID: "{{ .github_app_id }}"
        githubAppInstallationID: "{{ .github_app_installation_id }}"
        githubAppPrivateKey: "{{ .github_app_private_key }}"
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

> **ARC runner와 동일한 Secrets Manager 시크릿 공유**<br>
> 시크릿 로테이션 시에도 한 곳만 업데이트하면 Image Updater와 ARC runner 모두에 자동 반영된다.
{: .prompt-info }

### 전환 순서 (부트스트랩 문제 방지)

SSH 크레덴셜이 살아있는 동안 HTTPS 크레덴셜을 먼저 만들어야 한다.

```
Step 1: external-secret-image-updater.yaml 추가 → git push
        (기존 SSH Application들이 ESO 파일 sync → HTTPS 레포 시크릿 생성)

Step 2: HTTPS 레포 시크릿 생성 확인
        kubectl get secret -n argocd repo-purina-kr-infra-github-app

Step 3: 모든 application-*.yaml의 repoURL SSH → HTTPS 변경 + CronJob 제거 → git push
        (이제 SSH 크레덴셜 없어도 HTTPS로 정상 동작)
```

> **순서를 지키지 않으면?**<br>
> Application들이 HTTPS URL로 전환된 상태에서 HTTPS 크레덴셜이 없으면 모든 ArgoCD Application이 레포 접근 불가 상태가 된다. 반드시 HTTPS 크레덴셜을 먼저 만들어야 한다.
{: .prompt-danger }

---

## 8. 트러블슈팅

### Helm chart 버전 not found

**증상**: `Error: chart version "0.10.4" not found`

**원인**: 0.x 문서를 참조해 존재하지 않는 버전을 지정했다.

**해결**: `helm search repo argo/argocd-image-updater`로 실제 버전 확인 후 `1.2.2`로 변경.

---

### `credsExpire` 필드로 인한 CrashLoopBackOff

**증상**: Pod 시작 즉시 exit code 1, usage 텍스트 출력 후 종료

```
level=error msg="could not load registry configuration"
error="yaml: unmarshal errors:
  line 4: field credsExpire not found in type registry.RegistryConfiguration"
```

**원인**: v1.2.x에서 `RegistryConfiguration` Go 구조체에서 `credsExpire` 필드가 제거됐다. 바이너리가 시작 시 `registries.conf`를 파싱하다가 strict unmarshal 에러를 내고 종료한다.

> 에러 후 usage를 출력하므로 "왜 help가 출력되는가?" 혼란스러울 수 있다. 핵심은 에러 메시지 첫 줄이다.
{: .prompt-tip }

**해결**: `values.yaml`에서 `credsExpire: 10h` 라인 삭제.

---

### ECR 403 — `ecr:GetDownloadUrlForLayer` 권한 없음

**증상**: Pod 정상 기동, ECR 인증 성공, 그러나 이미지 메타데이터 조회 시 403

```
error fetching metadata for purina-api:...:
denied: User: ... is not authorized to perform: ecr:GetDownloadUrlForLayer
```

**원인**: `newest-build` 전략은 이미지 push 시각을 정확히 판단하기 위해 `ecr:GetDownloadUrlForLayer`와 `ecr:BatchCheckLayerAvailability`도 필요하다.

**해결**: IAM 정책에 두 액션 추가 → `terraform apply`.

---

### ConfigMap FailedMount

**증상**: Helm Pod Pending/CrashLoop

```
FailedMount: configmap "argocd-image-updater-ecr-creds" not found
```

**원인**: ConfigMap을 ArgoCD Application으로 관리했을 때, ArgoCD sync는 비동기이므로 순서가 보장되지 않는다. Helm Pod가 ConfigMap보다 먼저 생성되면 마운트 실패한다.

**해결**: ConfigMap을 Terraform 리소스로 이동 + `helm_release`에 `depends_on` 추가. Terraform은 의존성 그래프 기반으로 순서를 보장한다.

---

### writeBackTarget 이중 경로

**증상**: 이미지 감지 성공, kustomization.yaml 찾기 실패

```
could not find kustomization in /tmp/git-.../kubernetes/apps/kubernetes/apps
```

**원인**: Image Updater 최종 경로 = `[App spec.source.path]` + `[writeBackTarget 경로]`

- App `spec.source.path` = `kubernetes/apps`
- `writeBackTarget: "kustomization:./kubernetes/apps"` → 최종 경로 = `kubernetes/apps/kubernetes/apps` ❌

**해결**: `writeBackTarget: "kustomization:."` — App path 기준 현재 디렉토리를 사용.

---

### SSH 키 read-only — git push 실패

**증상**: 이미지 감지 → kustomization.yaml 수정 → push 실패

```
ERROR: The key you are authenticating with has been marked as read only.
```

**해결**: GitHub App HTTPS 인증으로 전환 (7절 참조).

---

### GitHub App 403 — Contents write 권한 없음

**증상**: HTTPS 전환 후에도 push 실패

```
remote: Write access to repository not granted.
```

**원인**: GitHub App의 `Contents` 권한이 `Read-only`였다.

**해결**: GitHub Apps → Permissions → Contents: `Read and write` → Save → 설치 승인.

---

### CronJob RBAC — create verb와 resourceNames 충돌

> 이미 CronJob은 GitHub App HTTPS 전환으로 제거됐지만, 동일한 실수를 방지하기 위해 기록한다.
{: .prompt-info }

**증상**: `cannot use resourceNames for verb 'create'`

**원인**: K8s RBAC에서 `create` verb는 아직 존재하지 않는 리소스에 대한 작업이므로 `resourceNames`로 범위를 제한할 수 없다.

**해결**: Role을 두 개 rule로 분리

```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create"]                              # resourceNames 없음
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["argocd-image-updater-secret"]
    verbs: ["get", "patch", "update"]
```

---

## 9. CI 보안 강화

### npm audit — `--omit=dev` 추가

#### 기존 방식의 문제

```bash
npm audit --audit-level=high   # dev 의존성 포함 전체 검사
```

Next.js 15.x에서 `eslint-config-next` → `@next/eslint-plugin-next` → `glob`(10.2.0~10.4.5) 의존성 체인에 HIGH 취약점이 존재한다.

```
glob  10.2.0 - 10.4.5
Severity: high
glob CLI: Command injection via -c/--cmd executes matches with shell:true
```

이 취약점은 개발자가 **로컬에서 glob CLI를 직접 `-c/--cmd` 플래그와 함께 실행**하는 특수한 상황에서만 발동한다.

#### 왜 프로덕션 배포에는 해당 없는가

`eslint-config-next`는 `devDependencies`다. 프로덕션 Docker 이미지는 multi-stage build로 dev 패키지를 포함하지 않는다.

```dockerfile
FROM node:20 AS builder
RUN npm ci          # devDependencies 포함 (ESLint, TypeScript 등)
RUN npm run build

FROM node:20-slim AS runner
COPY --from=builder /app/.next/standalone ./
# glob, eslint → 이 이미지에 없음
```

실제 서버에서 실행되지 않는 코드의 취약점은 공격자가 악용할 수 없다.

#### 현업 표준 접근법

```bash
# 프로덕션 의존성만 검사 — 파이프라인 차단 조건
npm audit --omit=dev --audit-level=high

# dev 의존성 취약점은 Dependabot 등으로 PR 단위 알림 (차단이 아닌 경고)
```

> dev 취약점을 완전히 무시하는 것이 아니라, **배포 차단 기준**에서만 제외하고 별도 채널로 모니터링한다.<br>
> 공급망 공격(supply chain attack) 관점에서 dev 도구 취약점은 CI 서버 침해 벡터가 될 수 있으므로 완전 방치는 위험하다.
{: .prompt-warning }

---

### Trivy — ECR push 전 로컬 이미지 스캔

#### 기존 방식: push 후 스캔

```
Build → ECR push → Trivy scan
```

CRITICAL 취약점이 발견돼도 이미 ECR에 이미지가 올라가 있다. ECR은 이미 push된 이미지를 자동으로 삭제하지 않으므로 실수로 배포될 위험이 있다.

#### 변경 후: push 전 스캔

```
Build (load: true) → Trivy scan → 통과 시에만 ECR push
```

CRITICAL 취약점이 있는 이미지는 **ECR에 올라가지조차 않는다.**

```yaml
# Docker Buildx로 로컬에만 이미지 빌드
- name: Build image (local only)
  uses: docker/build-push-action@v5
  with:
    context: applications/purina-nextjs/
    push: false      # ECR push 안 함
    load: true       # 로컬 Docker daemon에 로드
    tags: ${{ env.IMAGE_URI }}:${{ steps.tag.outputs.tag }}
    cache-from: type=gha
    cache-to: type=gha,mode=max

# 로컬 이미지 Trivy 스캔
- name: Container image security scan (Trivy)
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE_URI }}:${{ steps.tag.outputs.tag }}
    format: table
    exit-code: '1'          # CRITICAL 발견 시 exit 1 → 이후 스텝 실행 안 됨
    severity: CRITICAL
    ignore-unfixed: true    # 패치가 없는 CVE는 제외

# Trivy 통과 후에만 ECR push
- name: Push to ECR
  run: docker push ${{ env.IMAGE_URI }}:${{ steps.tag.outputs.tag }}
```

> **`ignore-unfixed: true` 이유**<br>
> 패치가 없는 취약점은 개발팀이 당장 대응할 수 없다. 이를 포함하면 파이프라인이 영구적으로 차단되는 상황이 발생한다.<br>
> 실제로 대응 가능한(패치 버전이 존재하는) 취약점만 차단 조건으로 삼는 것이 현실적인 운영 기준이다.
{: .prompt-info }

---

### Next.js 14.2.30 → 15.5.19 업그레이드

#### 14.2.30의 런타임 취약점

| CVE | 설명 | 심각도 |
|---|---|---|
| GHSA-h64f-5h5j-jqjh | DoS in Image Optimization API — 악의적 이미지 URL로 무한 루프 유발 | HIGH |
| GHSA-c4j6-fc7j-m34r | SSRF in WebSocket upgrades — 내부 서버로 요청 포워딩 가능 | HIGH |
| GHSA-wfc6-r584-vfw7 | Cache poisoning in RSC responses — 다른 사용자에게 잘못된 응답 전달 | HIGH |
| GHSA-36qx-fr4f-26g5 | Middleware/Proxy bypass in i18n — i18n 설정 시 보안 미들웨어 우회 | HIGH |

이 4개는 모두 **프로덕션 런타임**에서 실제 공격자가 악용할 수 있는 취약점이다. `next` 자체가 production dependency이므로 `--omit=dev`를 적용해도 계속 차단된다.

14.x 계열 내에서는 이 취약점들의 backport 패치가 없어 14.2.35까지 동일하게 취약하다.

#### 15.5.19 선택 이유

- 4개 런타임 취약점이 15.x에서 수정됨
- 2026년 6월 기준 15.x stable 최신 버전
- 14 → 15 마이그레이션에서 `next.config.mjs` 등 주요 설정 형식이 호환됨
- React 18을 계속 지원 (React 19 강제 업그레이드 없음)

업그레이드 후 `npm audit --omit=dev --audit-level=high` 기준 **0개 차단 취약점**.

---

## 10. 최종 검증

### Image Updater 상태 확인

```bash
# Pod 정상 기동 확인
kubectl get pod -n argocd -l app.kubernetes.io/name=argocd-image-updater
# NAME                                               READY   STATUS    RESTARTS   AGE
# argocd-image-updater-controller-7b7dc4667f-m92l4   1/1     Running   0          ...

# 최신 폴링 결과 확인 (2분 간격)
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=5
# level=info msg="Processing results: applications=1 images_considered=2 images_updated=1 errors=0"
```

`images_updated=1 errors=0` — 정상 동작.

### git write-back 확인

```bash
git log --oneline -3
# 9262cd8 build: update of application <prefix>-apps   ← Image Updater 자동 커밋
# 871feff feat(gitops): ArgoCD 레포 인증을 GitHub App HTTPS로 전환 + CronJob 제거
# 950e19b feat(eso): ArgoCD 레포 시크릿을 GitHub App HTTPS 인증으로 교체

cat kubernetes/apps/kustomization.yaml
# images:
#   - name: .../purina-nextjs
#     newName: .../purina-nextjs
#     newTag: 20260625-ace308b    ← Image Updater가 자동 업데이트
#   - name: .../purina-api
#     newName: .../purina-api
#     newTag: 20260624-b7038b1
```

### 완성된 CI/CD 파이프라인

```
1. 개발자 코드 push
        ↓
2. GitHub Actions: quality gate (TypeScript, ESLint, npm audit --omit=dev)
        ↓ 통과 시
3. GitHub Actions: Docker build (load: true, push: false)
        ↓
4. GitHub Actions: Trivy scan (로컬 이미지, CRITICAL 차단)
        ↓ 통과 시
5. GitHub Actions: ECR push
        ↓
6. Image Updater (2분 이내): newest-build 태그 감지
        ↓
7. Image Updater: kustomization.yaml 업데이트 → GitHub App으로 git commit+push
        ↓
8. ArgoCD: 변경 감지 → sync → kubectl apply (rolling update)
        ↓
9. GitHub Actions: smoke-test — rollout status 확인
        ↓
10. 파이프라인 성공 ✅
```

---

## 11. 후속 작업 — Multi-Source 전환과 이중관리 해소 (2026-07-01 ~ 07-02)

### Multi-Source 전환 배경

기존 구조에서 `helm/argocd-image-updater/values.yaml`을 한 줄만 고쳐도 `terraform apply`가 필요했다. 대부분의 워크로드는 git push만으로 반영되는데, Image Updater만 예외적으로 수동 개입이 필요한 게 불편하다고 판단해 ArgoCD Multi-source Application으로 전환을 시도했다.

### Multi-Source Helm 패턴

ArgoCD의 multi-source는 "Helm 차트를 외부 저장소에서 가져오되, values 파일만 이 저장소(GitOps repo)에서 가져오고 싶다"는 요구를 해결하기 위한 공식 패턴이다.

```yaml
# kubernetes/argocd/application-image-updater.yaml (Multi-source 전환 후)
spec:
  sources:
    # Source 1: Helm 차트 (외부 저장소)
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argocd-image-updater
      targetRevision: 1.2.2
      helm:
        releaseName: argocd-image-updater
        valueFiles:
          - $values/helm/argocd-image-updater/values.yaml  # $values로 git repo 참조

    # Source 2: values 파일 소스 (이 저장소)
    - repoURL: https://github.com/<github-username>/<repo-name>.git
      targetRevision: main
      ref: values   # "values"라는 이름표로 source 1이 $values로 참조

    # Source 3: ImageUpdater CR (기존과 동일)
    - repoURL: https://github.com/<github-username>/<repo-name>.git
      targetRevision: main
      path: kubernetes/image-updater
      directory:
        recurse: true
  syncOptions:
    - ServerSideApply=true  # Helm CRD와 CR이 동시에 적용될 때 필드 소유권 충돌 방지
```

**각 source의 역할:**

- **source[0]**: Helm 차트 정의(템플릿, CRD, 기본 values)가 있는 곳. ArgoCD repo-server가 `helm template`으로 매니페스트를 렌더링한다.
- **source[1]**: 이 저장소를 체크아웃하되 `ref: values` 이름표만 붙여서 source[0]이 `$values`로 참조할 수 있게 한다. 그 자체로는 아무 리소스도 만들지 않는다.
- **source[2]**: 기존과 동일하게 `ImageUpdater` CR을 관리하는 plain directory source.

> **`ServerSideApply=true`를 추가한 이유**<br>
> 한 Application 안에서 Helm 차트가 생성하는 CRD(`ImageUpdater` CRD 자체)와 그 인스턴스인 `ImageUpdater` CR이 동시에 적용된다.<br>
> Server-side apply는 API 서버가 각 필드마다 "누가(manager) 이 값을 썼는지"를 추적하므로 필드 충돌을 구조적으로 방지한다.
{: .prompt-info }

### `credsexpire: 10h` 추가

ECR의 `GetAuthorizationToken`이 발급하는 토큰은 **12시간**만 유효하다. Image Updater 내부적으로 이 토큰을 캐시하는 시간(`credsexpire`)이 실제 토큰 수명보다 길면, 만료된 토큰으로 ECR API를 호출해 인증 실패(`401`)가 발생한다. 12시간보다 짧은 **10시간**으로 설정해 여유 있게 재발급되도록 했다.

```yaml
registries:
  - name: ECR-apne2
    credentials: ext:/scripts/ecr-creds.sh
    credsexpire: 10h   # ECR 토큰 수명(12h)보다 짧게 설정
```

---

### 이중관리 문제 발견

Multi-source 전환 이후 `terraform/envs/dev/main.tf`의 `helm_release.argocd_image_updater` 리소스가 **제거되지 않고 그대로 남아있었다.**

```
[Terraform]  helm_release "argocd_image_updater"  ──┐
                                                      ├─► 같은 Helm release
[ArgoCD]     application sources[0] (Helm chart)  ──┘   같은 namespace, 같은 리소스들
```

당시 `terraform plan`은 `No changes`였다. 두 컨트롤러가 참조하는 값이 우연히 완전히 같았기 때문에 드리프트가 겉으로 드러나지 않고 있었을 뿐, 실질적으로는 이미 위험한 상태였다.

### 이중관리가 왜 위험한가

**① 핑퐁(reconciliation 충돌)**

```
t0: values.yaml 수정 → git push
t1: ArgoCD가 감지해 새 값 반영 (클러스터에 적용됨)
t2: 다른 이유로 terraform apply 실행
    → Terraform이 자신의 state 기준으로 "이전 값"으로 되돌림
t3: ArgoCD가 다음 reconciliation에서 다시 git 기준으로 되돌림
    → 어느 쪽이 마지막으로 적용했느냐에 따라 실제 운영 상태가 결정되는 비결정적 상태
```

**② `terraform destroy` 시 실제 삭제**

Terraform이 "자신이 소유"한다고 믿고 있으므로, `.tf`에서 이 리소스 블록을 지우고 `apply`하면 `helm uninstall argocd-image-updater`에 해당하는 삭제가 실제로 수행된다. Image Updater 컨트롤러가 사라지면 CI/CD 이미지 자동 갱신 파이프라인 전체가 멈춘다.

**③ Server-Side Apply 필드 소유권 충돌**

`argocd-controller`와 Terraform Helm provider가 같은 리소스의 같은 필드에 서로 다른 field manager 이름으로 값을 쓰면 다음과 같은 에러가 발생할 수 있다.

```
Apply failed with 1 conflict: conflict with "helm" using apps/v1:
  .spec.template.spec.containers[name="argocd-image-updater"].resources
```

**④ `helm history`/`helm rollback`이 실제와 다른 이력**

ArgoCD는 Helm 릴리즈 이력(`sh.helm.release.v1.*` Secret)을 갱신하지 않는다. 운영자가 `helm rollback`을 실행하면 ArgoCD가 최근 적용한 변경 사항이 전부 사라지는 혼란스러운 장애로 이어진다.

---

### 해결 방향 — EKS 현업 표준 패턴

**핵심 원칙**: GitOps 엔진이 의존하는 플랫폼 컨트롤러는 **Terraform으로 부트스트랩**하고, 그 위에서 도는 애플리케이션 레벨 설정만 **ArgoCD(GitOps)**가 관리한다.

> 이미 이 패턴이 프로젝트 내에 적용되어 있었다.
> - ARC Controller: Terraform `helm_release` — "CRD 518KB 초과로 ArgoCD 관리 불가 → Terraform 사용"
> - Karpenter: Terraform `helm_release` — 노드 프로비저닝을 담당하는 클러스터 최하위 레벨 컨트롤러
> - **Image Updater**: Multi-source 전환으로 일시적으로 이탈했다가 원복
{: .prompt-info }

**최종 소유권 구조:**

| 컴포넌트 | 관리 주체 | 근거 |
|---|---|---|
| Helm 차트 설치 (Deployment, SA, RBAC 등) | **Terraform** | 순환 의존성 방지. GitOps가 의존하는 컨트롤러는 Terraform이 부트스트랩 |
| IAM Role / Pod Identity / ECR 인증 ConfigMap | **Terraform** | AWS 클라우드 리소스. Helm Pod 시작 전에 반드시 존재해야 하는 선행 리소스 |
| `ImageUpdater` CR (어떤 ECR 이미지를 감시할지) | **ArgoCD** | 애플리케이션 레벨 설정. 변경 빈도가 높고 git push로 바로 반영 |

---

### 무중단 마이그레이션 — 단계별 절차

단순히 ArgoCD Application에서 Helm 소스만 빼면, `prune: true`가 걸려있어 ArgoCD가 다음 sync에서 관련 리소스를 즉시 삭제한다. 이를 피하기 위해 4단계로 나눠 진행했다.

**1단계 — `prune: false` 안전장치와 함께 source 원복**

source 변경과 `prune: false` 설정을 **같은 커밋**에 함께 반영했다. 삭제가 발생할 수 있는 타이밍 자체가 존재하지 않도록 원자적으로 처리했다.

```yaml
# kubernetes/argocd/application-image-updater.yaml
spec:
  source:                              # sources[] → 단일 source로 원복
    repoURL: https://github.com/<github-username>/<repo-name>.git
    targetRevision: main
    path: kubernetes/image-updater
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: false   # ← 전환기 안전장치
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

sync 후 10개 리소스에 `requiresPruning: true`가 표시됐지만 `prune: false`라 실제 삭제는 일어나지 않았다. 컨트롤러는 무중단으로 유지됐다.

```bash
kubectl get deploy argocd-image-updater-controller -n argocd
# NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
# argocd-image-updater-controller   1/1     1            1           6d21h  ← 변화 없음
```

**2단계 — ArgoCD 추적(tracking) annotation 제거**

ArgoCD가 이 10개 리소스를 완전히 "잊게" 하려면 `argocd.argoproj.io/tracking-id` annotation을 제거해야 한다. 리소스 자체는 전혀 손대지 않는다.

```bash
# 예시: Deployment
kubectl patch deployment argocd-image-updater-controller -n argocd \
  --type=json -p='[{"op":"remove","path":"/metadata/annotations/argocd.argoproj.io~1tracking-id"}]'

# ClusterRole/ClusterRoleBinding은 -n 플래그 없이
kubectl patch clusterrole argocd-image-updater \
  --type=json -p='[{"op":"remove","path":"/metadata/annotations/argocd.argoproj.io~1tracking-id"}]'
```

> **JSON Patch 경로 이스케이프 규칙**<br>
> annotation key의 슬래시(`/`)는 `~1`로 이스케이프한다.<br>
> `argocd.argoproj.io/tracking-id` → `argocd.argoproj.io~1tracking-id`
{: .prompt-tip }

패치 후 hard refresh로 재평가하면 Application이 `Synced`/`Healthy`가 되고 `ImageUpdater` CR 하나만 관리하게 된다.

**3단계 — `prune: true` 복구**

이 시점부터는 Application이 `ImageUpdater` CR 하나만 관리하므로 `prune: true`로 되돌려도 안전하다.

```yaml
syncPolicy:
  automated:
    prune: true   # 복구
    selfHeal: true
```

**4단계 — stale 상태(history, operationState) 정리**

Multi-source 시절의 sync 기록이 `status.history`에 남아있어 ArgoCD UI에서 에러가 표시됐다.

```bash
# 문제의 history 항목 제거
kubectl patch application <prefix>-image-updater -n argocd \
  --type=json -p='[{"op":"remove","path":"/status/history/4"}]'

# stale operationState 제거
kubectl patch application <prefix>-image-updater -n argocd \
  --type=json -p='[{"op":"remove","path":"/status/operationState"}]'

# 새 sync operation 트리거 — 깨끗한 operationState 재생성
kubectl patch application <prefix>-image-updater -n argocd \
  --type=merge -p='{"operation":{"initiatedBy":{"username":"admin"},"sync":{"prune":true}}}'
```

> **ArgoCD Application CRD는 `status`를 subresource로 분리하지 않는다.**<br>
> 따라서 `--subresource=status` 없이 일반 patch로 `status` 필드를 직접 수정할 수 있다.<br>
> 단, 이는 UI 표시 문제 정리용으로만 사용해야 한다.
{: .prompt-warning }

---

### 최종 검증

```bash
# Application 상태
kubectl get application <prefix>-image-updater -n argocd -o json \
  | jq '.status.sync.status, .status.health.status, .spec.source.path'
# "Synced"
# "Healthy"
# "kubernetes/image-updater"

# 컨트롤러 무중단 확인
kubectl get deploy argocd-image-updater-controller -n argocd
# READY   AVAILABLE   AGE
# 1/1     1           6d21h   ← 마이그레이션 내내 재시작 없음

# ImageUpdater CR 정상 감시 확인
kubectl get imageupdater -n argocd
# NAME           APPS   IMAGES   LAST CHECKED   READY
# <prefix>-apps  1      2        63s            True   ← nextjs, api 이미지 2개 정상 감시
```

---

### 회고

> **"Multi-source 전환으로 편의성을 높인다"는 판단 자체가, 이미 확립된 아키텍처 컨벤션과 충돌하는지 먼저 점검했어야 했다.**<br>
> `arc_controller`/`karpenter`가 왜 Terraform으로 관리되는지 그 이유(순환 의존성 방지)가 Image Updater에도 그대로 적용되는데, "values.yaml 수정 편의성"만 보고 이 컨벤션을 놓쳤다.
{: .prompt-danger }

> **`terraform plan`에 diff가 없다고 안전한 게 아니다.**<br>
> 두 컨트롤러가 같은 리소스를 관리해도 값이 우연히 같으면 며칠간 드리프트가 겉으로 드러나지 않는다.<br>
> 이중 소유권 자체가 문제지, 현재 값이 같은지 다른지는 별개 문제다.
{: .prompt-warning }

> **GitOps 리소스를 다른 컨트롤러로 이관할 때는 "삭제 후 재생성"이 아니라 "추적 정보만 떼어내기"가 표준 기법이다.**<br>
> `prune: false`로 삭제를 임시로 막고, tracking annotation만 제거해 소유권을 이전하면 실제 리소스를 단 한 번도 재시작시키지 않고 컨트롤러를 교체할 수 있다.
{: .prompt-tip }
