---
title: "관측의 시작, 직접 쌓는 모니터링 - Prometheus, Loki, Grafana(PLG) 스택 구축기"
date: 2026-05-28 09:00:00 +0900
categories: [2. Kubernetes, Operations]
tags: [eks, prometheus, loki, grafana, promtail, plg, monitoring, karpenter, irsa, s3, ebs-csi]
---

> EKS 마이그레이션 시리즈 일곱 번째 포스팅이다.<br>
> 앞선 포스팅에서 GitHub Actions + ArgoCD GitOps CI/CD 파이프라인을 완성했다. 이번에는 클러스터와 애플리케이션의 상태를 한눈에 파악할 수 있는 모니터링 스택을 구축한다.<br>
> ELK 대신 PLG(Promtail + Loki + Grafana)를 선택한 이유부터, EBS CSI Driver 설치, S3 백엔드 구성, GitHub OAuth SSO 연동까지 실제 현장에서 겪은 트러블슈팅을 그대로 담았다.
{: .prompt-info }

---

## 왜 PLG인가?

이커머스 EKS 환경에서 모니터링 스택을 선택할 때 가장 많이 비교하는 조합이 ELK(Elasticsearch + Logstash + Kibana)와 PLG다.

| 항목 | ELK | PLG |
|---|---|---|
| 리소스 | Elasticsearch 메모리 과다 사용 | 가벼움 |
| 비용 | 높음 (ES 노드 비용) | 낮음 (S3 스토리지) |
| 로그 검색 | 강력한 전문 검색 | 레이블 기반 필터링 |
| 메트릭 통합 | 별도 구성 필요 | Grafana 하나로 통합 |
| 운영 복잡도 | 높음 | 낮음 |

> **PLG를 선택한 핵심 이유**<br>
> Loki는 Elasticsearch와 달리 로그 내용을 인덱싱하지 않고 레이블만 인덱싱한다.<br>
> 덕분에 S3 같은 저렴한 오브젝트 스토리지를 백엔드로 사용할 수 있어 비용이 획기적으로 낮다.<br>
> 로그 볼륨이 수 TB에 달하거나 복잡한 전문 검색이 필요하다면 ELK가 적합하지만, 그렇지 않다면 Grafana 하나에서 메트릭과 로그를 함께 볼 수 있는 PLG가 훨씬 실용적이다.
{: .prompt-tip }

### 전체 아키텍처

```
Promtail (DaemonSet, 모든 노드)
  └── /var/log/pods/ 로그 수집
        └──→ Loki Gateway (nginx)
                ├── Loki Write  ──→ S3 (<prefix>-loki-apne2-s3, KMS 암호화)
                ├── Loki Read   ←── S3
                ├── Loki Backend (compactor, ruler, index-gateway)
                ├── chunks-cache (Memcached)
                └── results-cache (Memcached)

Prometheus
  ├── Node, Pod, K8s 오브젝트 메트릭 수집
  └── AlertManager → Slack / Email

Grafana
  ├── Datasource: Prometheus (메트릭)
  ├── Datasource: Loki (로그)
  └── GitHub OAuth SSO
          │
          ▼
    ALB → grafana.example.com
```

---

## 1. 사전 준비

### EBS CSI Driver 설치

Prometheus, Grafana, AlertManager는 재시작 시에도 데이터가 유지되어야 하므로 PV(Persistent Volume)가 필요하다. EKS에서 EBS 볼륨을 동적으로 프로비저닝하려면 EBS CSI Driver가 필요하다.

> **EKS Addon으로 설치하는 이유**<br>
> Helm으로도 설치 가능하지만 EKS 버전 업그레이드 시 Addon 방식이 버전 관리가 훨씬 편하다.<br>
> AWS가 EKS 버전과 호환되는 CSI Driver 버전을 자동으로 관리해준다.
{: .prompt-info }

**IRSA 생성 (Terraform)**

```hcl
# modules/eks/main.tf

resource "aws_iam_role" "ebs_csi" {
  name = "${local.prefix}-ebs-csi-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.this.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          # kube-system 네임스페이스의 ebs-csi-controller-sa SA만 허용
          "${local.oidc_provider_url}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
          "${local.oidc_provider_url}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# AWS 관리형 정책 연결
resource "aws_iam_role_policy_attachment" "ebs_csi" {
  role       = aws_iam_role.ebs_csi.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}

# KMS 커스텀 키 사용 시 추가 권한 필요
# AmazonEBSCSIDriverPolicy에는 KMS 권한이 포함되어 있지 않음
resource "aws_iam_role_policy" "ebs_csi_kms" {
  name = "${local.prefix}-ebs-csi-kms-apne2-policy"
  role = aws_iam_role.ebs_csi.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "kms:CreateGrant", "kms:ListGrants", "kms:RevokeGrant",
        "kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*",
        "kms:GenerateDataKey*", "kms:DescribeKey"
      ]
      Resource = "<ebs-kms-key-arn>"
    }]
  })
}
```

**EKS Addon으로 설치**

```hcl
resource "aws_eks_addon" "ebs_csi" {
  cluster_name                = aws_eks_cluster.this.name
  addon_name                  = "aws-ebs-csi-driver"
  service_account_role_arn    = aws_iam_role.ebs_csi.arn
  resolve_conflicts_on_update = "OVERWRITE"

  # system 노드에만 배포
  configuration_values = jsonencode({
    controller = {
      tolerations = [{
        key      = "dedicated"
        value    = "system"
        effect   = "NoSchedule"
        operator = "Equal"
      }]
      nodeSelector = { role = "system" }
    }
  })

  depends_on = [aws_eks_node_group.sys]
}
```

설치 확인:

```bash
kubectl get pods -n kube-system | grep ebs-csi
# ebs-csi-controller-*   6/6   Running  (system 노드)
# ebs-csi-node-*         3/3   Running  (전체 노드 DaemonSet)
```

### StorageClass (gp3) 생성

```yaml
# kubernetes/storage/storageclass-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # Pod 스케줄 후 같은 AZ에 볼륨 생성
allowVolumeExpansion: true               # PVC 용량 확장 허용
reclaimPolicy: Retain                    # PVC 삭제 시 볼륨 보존
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: <ebs-kms-key-arn>
```

> **주요 설정 설명**<br>
> `WaitForFirstConsumer`: PVC 생성 즉시 볼륨을 만들지 않고 Pod가 특정 AZ에 스케줄될 때까지 기다린다. Pod와 볼륨이 같은 AZ에 생성되어 마운트 실패를 방지한다.<br>
> `Retain`: PVC를 삭제해도 실제 EBS 볼륨은 삭제되지 않는다. 실수로 PVC를 삭제해도 데이터를 복구할 수 있어 운영계 필수 설정이다.<br>
> `encrypted + kmsKeyId`: EBS 볼륨을 CMK(Customer Managed Key)로 암호화한다.
{: .prompt-warning }

```bash
kubectl apply -f kubernetes/storage/storageclass-gp3.yaml

# 기존 gp2 default 제거
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl get storageclass
# NAME   PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE
# gp3    ebs.csi.aws.com   Retain          WaitForFirstConsumer   (default)
```

### Loki S3 버킷 + IRSA (Terraform)

```hcl
# Loki 전용 KMS Key
resource "aws_kms_key" "loki" {
  description             = "KMS key for Loki S3 bucket encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true  # 연 1회 자동 로테이션
}

resource "aws_kms_alias" "loki" {
  name          = "alias/<prefix>-loki-apne2-kms"
  target_key_id = aws_kms_key.loki.key_id
}

# S3 버킷
resource "aws_s3_bucket" "loki" {
  bucket = "<prefix>-loki-apne2-s3"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "loki" {
  bucket = aws_s3_bucket.loki.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.loki.arn
    }
    bucket_key_enabled = true  # KMS API 호출 횟수 감소 → 비용 절감
  }
}

resource "aws_s3_bucket_public_access_block" "loki" {
  bucket                  = aws_s3_bucket.loki.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IRSA
resource "aws_iam_role" "loki" {
  name = "<prefix>-loki-apne2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = module.eks.oidc_provider_arn }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          # monitoring 네임스페이스의 loki SA만 허용
          "${module.eks.oidc_provider_url}:sub" = "system:serviceaccount:monitoring:loki"
          "${module.eks.oidc_provider_url}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "loki" {
  name = "<prefix>-loki-apne2-policy"
  role = aws_iam_role.loki.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"]
        Resource = [aws_s3_bucket.loki.arn, "${aws_s3_bucket.loki.arn}/*"]
      },
      {
        Effect = "Allow"
        Action = ["kms:GenerateDataKey", "kms:Decrypt", "kms:DescribeKey"]
        Resource = aws_kms_key.loki.arn
      }
    ]
  })
}
```

**Helm Repo 추가 및 네임스페이스 생성**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
```

---

## 2. kube-prometheus-stack

kube-prometheus-stack은 Prometheus, Grafana, AlertManager, Node Exporter, kube-state-metrics를 한 번에 설치하는 Helm Chart다.

```bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 70.4.2 \
  -f helm/monitoring/kube-prometheus-stack/values.yaml
```

### values.yaml

```yaml
fullnameOverride: "kube-prometheus-stack"

# -----------------------------------------------
# Prometheus
# -----------------------------------------------
prometheus:
  prometheusSpec:
    tolerations:
      - key: dedicated
        operator: Equal
        value: system
        effect: NoSchedule
    nodeSelector:
      role: system
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: prometheus

    # 메트릭 보존 정책
    # retention과 retentionSize 중 먼저 도달하는 조건에서 오래된 데이터 삭제
    retention: 30d
    retentionSize: "50GB"

    # PV 설정
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi  # retentionSize와 일치시켜 볼륨 초과 방지

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi

    # 수집/평가 간격
    # 운영계: 15s 권장 / 개발계: 30s로 부하 감소
    scrapeInterval: 30s
    evaluationInterval: 30s

# -----------------------------------------------
# AlertManager
# -----------------------------------------------
alertmanager:
  alertmanagerSpec:
    tolerations:
      - key: dedicated
        operator: Equal
        value: system
        effect: NoSchedule
    nodeSelector:
      role: system
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi

# -----------------------------------------------
# Grafana
# -----------------------------------------------
grafana:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 512Mi

  # ESO로 주입된 Secret을 환경변수로 로드
  # ${clientID}, ${clientSecret} 형식으로 grafana.ini에서 참조
  envFromSecret: grafana-github-oauth-secret

  grafana.ini:
    server:
      root_url: https://grafana.example.com
    auth.github:
      enabled: true
      client_id: ${clientID}
      client_secret: ${clientSecret}
      scopes: user:email,read:org
      auth_url: https://github.com/login/oauth/authorize
      token_url: https://github.com/login/oauth/access_token
      api_url: https://api.github.com/user
      allow_sign_up: true
      allowed_users: <github-username>
      role_attribute_path: "'Admin'"

  # Loki datasource 자동 등록
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local
      access: proxy
      isDefault: false

  adminPassword: "change-me-after-deploy"

# -----------------------------------------------
# Node Exporter
# DaemonSet이므로 모든 노드 taint에 toleration 필요
# 하나라도 빠지면 해당 노드의 메트릭이 수집되지 않음
# -----------------------------------------------
nodeExporter:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
    - key: dedicated
      operator: Equal
      value: web
      effect: NoSchedule
    - key: dedicated
      operator: Equal
      value: api
      effect: NoSchedule

# -----------------------------------------------
# kube-state-metrics
# -----------------------------------------------
kube-state-metrics:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# Prometheus Operator
# admissionWebhooks.patch도 같은 블록에 설정해야 함
# 블록을 두 번 정의하면 두 번째가 첫 번째를 덮어씀 → 주의!
# -----------------------------------------------
prometheusOperator:
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system
  admissionWebhooks:
    enabled: true
    patch:
      enabled: true
      tolerations:
        - key: dedicated
          operator: Equal
          value: system
          effect: NoSchedule
      nodeSelector:
        role: system
    job:
      tolerations:
        - key: dedicated
          operator: Equal
          value: system
          effect: NoSchedule
      nodeSelector:
        role: system
```

> **`retention`과 `retentionSize`를 함께 설정하는 이유**<br>
> `retention: 30d`만 설정하면 30일치 데이터가 쌓여도 삭제되지 않아 볼륨이 꽉 찰 수 있다.<br>
> `retentionSize: "50GB"`를 함께 설정하면 두 조건 중 먼저 도달하는 시점에 오래된 데이터를 자동으로 삭제한다.<br>
> `storage: 50Gi`와 `retentionSize: "50GB"`를 일치시켜야 볼륨 초과를 방지할 수 있다.
{: .prompt-warning }

> **`scrapeInterval: 30s` — 개발계와 운영계 차이**<br>
> 운영계에서는 15s로 설정해 더 촘촘하게 메트릭을 수집하는 것이 권장된다.<br>
> 개발계에서는 30s로 설정해 Prometheus의 CPU/메모리 부하를 줄인다.
{: .prompt-info }

---

## 3. Loki

### 배포 모드 선택

| 모드 | 구성 | 적합한 환경 |
|---|---|---|
| SingleBinary | 단일 Pod | 테스트/소규모 |
| **SimpleScalable** | read/write/backend 분리 | **중소규모 운영 (권장)** ✅ |
| Distributed | 완전 분산 | 대규모 운영 |

> SimpleScalable은 구성 요소가 분리되어 있어 병목 지점만 독립적으로 스케일링할 수 있다.<br>
> 예를 들어 로그 쓰기가 느리면 write replicas만 늘리면 된다. SingleBinary는 전체를 스케일해야 한다.
{: .prompt-tip }

```bash
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --version 6.30.1 \
  -f helm/monitoring/loki/values.yaml
```

### values.yaml

```yaml
loki:
  # 멀티 테넌시 비활성화
  # true 설정 시 모든 요청에 X-Scope-OrgID 헤더 필수
  # 단일 팀/클러스터 환경에서는 false가 일반적
  auth_enabled: false

  commonConfig:
    replication_factor: 2  # 데이터 복제본 수 (write replicas와 일치시킬 것)

  storage:
    type: s3
    bucketNames:
      chunks: <prefix>-loki-apne2-s3  # 실제 로그 청크 저장
      ruler: <prefix>-loki-apne2-s3   # 알림 규칙 저장
      admin: <prefix>-loki-apne2-s3   # 관리 데이터 저장
    s3:
      region: ap-northeast-2
      insecure: false

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb          # TSDB 인덱스 (현재 권장 방식)
        object_store: s3
        schema: v13          # 현재 최신 스키마
        index:
          prefix: loki_index_
          period: 24h        # 24시간 단위로 인덱스 분리

# IRSA: ServiceAccount에 IAM Role ARN annotation 추가
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<prefix>-loki-apne2-role

deploymentMode: SimpleScalable

# -----------------------------------------------
# Loki Write
# 로그 수신 → WAL → S3 저장
# -----------------------------------------------
write:
  replicas: 2  # replication_factor와 일치
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# Loki Read
# LogQL 쿼리 처리 → S3에서 청크 조회
# -----------------------------------------------
read:
  replicas: 2
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# Loki Backend
# Compactor: 청크 압축 및 정리 (스토리지 비용 절감)
# Ruler: 알림 규칙 평가
# Index Gateway: TSDB 인덱스 관리
# -----------------------------------------------
backend:
  replicas: 2
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# Gateway (nginx)
# Promtail, Grafana의 단일 진입점 역할
# /loki/api/v1/push → write 컴포넌트로 라우팅
# /loki/api/v1/query → read 컴포넌트로 라우팅
# -----------------------------------------------
gateway:
  enabled: true
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# chunks-cache (Memcached)
# S3에서 읽은 로그 청크를 메모리에 캐싱
# 동일한 시간대 로그 재조회 시 S3 요청 없이 즉시 반환
# → 쿼리 속도 향상, S3 API 비용 절감
# 기본값: 8192MB(8GB) → 개발계: 2048MB(2GB)로 축소 가능
# -----------------------------------------------
chunksCache:
  enabled: true
  allocatedMemory: 2048  # MB 단위
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

# -----------------------------------------------
# results-cache (Memcached)
# 동일한 LogQL 쿼리 결과를 캐싱
# Grafana 대시보드 반복 조회 시 read 컴포넌트와 S3 부하 감소
# -----------------------------------------------
resultsCache:
  enabled: true
  tolerations:
    - key: dedicated
      operator: Equal
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system

minio:
  enabled: false  # S3를 사용하므로 비활성화
```

> **`replication_factor: 2`와 write replicas를 일치시켜야 하는 이유**<br>
> Loki는 로그를 수신할 때 `replication_factor` 수만큼의 write 컴포넌트에 복제 저장한다.<br>
> write replicas가 `replication_factor`보다 적으면 쓰기가 실패한다. 반드시 일치시켜야 한다.
{: .prompt-danger }

---

## 4. Promtail

Promtail은 DaemonSet으로 클러스터의 모든 노드에 배포되어 Pod 로그를 수집하고 Loki로 전송한다.

```bash
helm upgrade --install promtail grafana/promtail \
  --namespace monitoring \
  --version 6.16.6 \
  -f helm/monitoring/promtail/values.yaml
```

### values.yaml

```yaml
# Loki Gateway로 로그 전송
# Gateway가 내부에서 write 컴포넌트로 라우팅
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push

# DaemonSet이므로 모든 노드에 배포되어야 함
# 모든 taint에 대한 toleration을 설정해야 함
# 하나라도 빠지면 해당 노드의 로그가 수집되지 않음
tolerations:
  - key: dedicated
    operator: Equal
    value: system
    effect: NoSchedule
  - key: dedicated
    operator: Equal
    value: web
    effect: NoSchedule
  - key: dedicated
    operator: Equal
    value: api
    effect: NoSchedule

resources:
  requests:
    cpu: 50m      # 개발계 (운영계: 100m~200m, 로그 볼륨에 비례)
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

> **Promtail에 nodeSelector를 설정하면 안 되는 이유**<br>
> nodeSelector를 설정하면 해당 노드에만 배포된다. 나머지 노드의 로그는 수집되지 않는다.<br>
> DaemonSet의 목적 자체가 모든 노드에 배포하는 것이므로 nodeSelector 없이 toleration만 설정해야 한다.
{: .prompt-warning }

---

## 5. Grafana GitHub OAuth (SSO)

Grafana는 ArgoCD(Dex 경유)와 달리 자체적으로 GitHub OAuth를 지원한다.

### GitHub OAuth App 생성

```
GitHub → Settings → Developer settings → OAuth Apps → New OAuth App

Application name: <prefix>-grafana
Homepage URL: https://grafana.example.com
Authorization callback URL: https://grafana.example.com/login/github
```

> **ArgoCD의 callback URL(`/api/dex/callback`)과 다르다**는 점에 주의.<br>
> Grafana는 `/login/github`를 사용한다. 잘못 설정하면 인증 후 리다이렉트가 실패한다.
{: .prompt-danger }

### Client Secret을 AWS Secrets Manager에 저장

```hcl
resource "aws_secretsmanager_secret" "grafana_github_oauth" {
  name        = "<prefix>-grafana-github-oauth-apne2-secret"
  description = "Grafana GitHub OAuth Client ID/Secret"
}

resource "aws_secretsmanager_secret_version" "grafana_github_oauth" {
  secret_id = aws_secretsmanager_secret.grafana_github_oauth.id
  secret_string = jsonencode({
    clientID     = "<github-oauth-client-id>"
    clientSecret = var.grafana_github_oauth_client_secret
  })
  lifecycle {
    ignore_changes = [secret_string]  # 수동 업데이트 보호
  }
}
```

### External Secrets로 K8s Secret 주입

```yaml
# kubernetes/external-secrets/external-secret-grafana.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: grafana-github-oauth-secret
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: grafana-github-oauth-secret
    creationPolicy: Owner  # ESO가 Secret을 소유 (삭제 시 같이 삭제)
  data:
  - secretKey: clientID
    remoteRef:
      key: <prefix>-grafana-github-oauth-apne2-secret
      property: clientID
  - secretKey: clientSecret
    remoteRef:
      key: <prefix>-grafana-github-oauth-apne2-secret
      property: clientSecret
```

> **ArgoCD의 `Merge`와 달리 `Owner`를 사용하는 이유**<br>
> ArgoCD는 기존 `argocd-secret`에 키를 추가해야 했으므로 `Merge` 방식이 필요했다.<br>
> Grafana는 새 Secret을 ESO가 직접 생성하므로 `Owner`가 적합하다.<br>
> `Owner`는 ExternalSecret이 삭제될 때 K8s Secret도 함께 삭제되어 고아 리소스가 남지 않는다.
{: .prompt-info }

그 후 Helm values에서 `envFromSecret`으로 로드하면 `${clientID}`, `${clientSecret}` 형식으로 `grafana.ini`에서 참조할 수 있다.

---

## 6. Ingress 설정

Grafana는 기존 pub ALB의 Ingress Group에 추가한다. 같은 `group.name`을 사용하면 하나의 ALB에서 host-header로 트래픽을 분리한다.

```yaml
# kubernetes/ingress/ingress-grafana.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/load-balancer-name: <prefix>-pub-apne2-alb
    alb.ingress.kubernetes.io/group.name: <prefix>-pub-alb  # 기존 ALB 그룹과 동일
    alb.ingress.kubernetes.io/healthcheck-path: /api/health
    external-dns.alpha.kubernetes.io/hostname: grafana.example.com
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80
```

현재 ALB 라우팅 구조:

```
<prefix>-pub-apne2-alb
  ├── app.example.com     → Next.js
  ├── argocd.example.com  → ArgoCD
  └── grafana.example.com → Grafana   ← 이번에 추가
```

---

## 7. 트러블슈팅

### admissionWebhooks Job Pending

```
0/2 nodes are available: 2 node(s) had untolerated taint(s).
```

| 항목 | 내용 |
|---|---|
| 원인 | `kube-prometheus-stack-admission-create` Job Pod에 toleration 없음 |
| 잘못된 접근 | `prometheusOperator` 블록을 values.yaml에 두 번 정의하면 두 번째가 첫 번째를 덮어써서 `admissionWebhooks` 설정이 무시됨 |
| 해결 | `prometheusOperator` 블록 **하나**에 모든 설정을 통합 |

```yaml
# 올바른 설정 — 하나의 블록에 통합
prometheusOperator:
  tolerations:
    - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
  nodeSelector:
    role: system
  admissionWebhooks:
    patch:
      tolerations:
        - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
      nodeSelector:
        role: system
    job:
      tolerations:
        - { key: dedicated, operator: Equal, value: system, effect: NoSchedule }
      nodeSelector:
        role: system
```

---

### chunks-cache Pending (메모리 부족)

```
0/2 nodes are available: Insufficient memory.
```

| 항목 | 내용 |
|---|---|
| 원인 | chunks-cache 기본 메모리가 8GB(`allocatedMemory: 8192`)인데 system 노드 가용 메모리 부족 |
| 해결 | 개발계에서는 2GB로 축소. 운영계 전환 시 노드 스펙 올리고 원복 |

```yaml
chunksCache:
  enabled: true
  allocatedMemory: 2048  # 8192 → 2048
```

---

### Promtail CPU 부족

```
0/2 nodes are available: Insufficient cpu.
```

| 항목 | 내용 |
|---|---|
| 원인 | system 노드에 이미 많은 컴포넌트가 올라가 있어 CPU 여유 없음 |
| 해결 | Promtail CPU Request 축소. 개발계 저 로그 볼륨에서는 50m으로 충분 |

```yaml
resources:
  requests:
    cpu: 50m  # 100m → 50m
```

---

### Loki IRSA 권한 오류 — AccessDenied

```
AccessDenied: s3:PutObject on resource: arn:aws:s3:::<prefix>-loki-apne2-s3/...
```

| 항목 | 내용 |
|---|---|
| 원인 | ServiceAccount에 IAM Role ARN annotation이 없거나 Trust Policy의 SA 이름/네임스페이스 불일치 |
| 해결 | values.yaml의 serviceAccount annotation과 Terraform Trust Policy를 동시에 확인 |

```bash
# ServiceAccount annotation 확인
kubectl get sa loki -n monitoring -o yaml | grep role-arn

# Pod 환경변수 확인 (Pod Identity Agent 주입 여부)
kubectl describe pod loki-write-0 -n monitoring | grep AWS_CONTAINER
```

---

### Grafana OAuth 콜백 URL 오류

```
The redirect_uri is not associated with the application.
```

| 항목 | 내용 |
|---|---|
| 원인 | GitHub OAuth App의 callback URL이 `/login/github`가 아닌 다른 경로로 설정됨 |
| 해결 | GitHub OAuth App 설정에서 `Authorization callback URL`을 `https://grafana.example.com/login/github`로 수정 |

---

## 8. 최종 확인

```bash
# 전체 Pod 상태 확인
kubectl get pods -n monitoring

# PVC 확인 (Prometheus, Grafana, AlertManager)
kubectl get pvc -n monitoring

# Loki S3 연동 확인
kubectl logs -n monitoring loki-write-0 | grep -i "s3\|error" | tail -10

# Promtail → Loki 전송 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=promtail --tail=5

# Grafana 헬스체크
curl -s -o /dev/null -w "%{http_code}" https://grafana.example.com/api/health
# 200
```

---

## 마치며

PLG 스택 구축에서 가장 신경 써야 할 부분은 두 가지였다.

첫 번째는 **DaemonSet(Node Exporter, Promtail)의 toleration**이다. 모든 노드에 배포되어야 하는 DaemonSet은 클러스터에 존재하는 모든 taint에 대한 toleration을 설정해야 한다. 하나라도 빠지면 해당 노드의 메트릭/로그가 수집되지 않는다.

두 번째는 **Loki의 S3 IRSA 설정**이다. EKS IRSA는 ServiceAccount, 네임스페이스, IAM Role이 정확히 매핑되어야 한다. Loki의 경우 `monitoring` 네임스페이스의 `loki` ServiceAccount로 고정되어 있으므로 Terraform Trust Policy에 정확히 명시해야 한다.

> **EKS 마이그레이션 시리즈**<br>
> 1편: As-Is 분석 + To-Be 아키텍처 설계<br>
> 2편: Terraform VPC/EKS/Karpenter 구성 및 검증<br>
> 3편: Private 클러스터 + VSCode Server 구축<br>
> 4편: ALBC/ESO/ArgoCD GitOps 파이프라인<br>
> 5편: PHP 앱 컨테이너화 → Next.js 재구현 → MSA 분리<br>
> 6편: GitHub Actions + ArgoCD CI/CD + HPA + ALB Ingress Group + GitHub OAuth<br>
> **7편: PLG Monitoring Stack (Prometheus + Loki + Grafana)**
{: .prompt-tip }
