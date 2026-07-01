---
title: "Implementing SMS OTP Authentication on EKS — Integrating Valkey (Redis) for Processing and Storage"
date: 2026-06-24 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, redis, valkey, solapi, sms, otp, elasticache, sentinel, ioredis, kubernetes, argocd]
---

> 회원가입 플로우에 휴대폰 본인 인증(SMS OTP)을 도입하는 작업이다.<br>
> 단순 SMS 발송에서 끝나지 않고, EKS `replicas: 2` 환경에서도 인증이 깨지지 않도록 **분산 OTP 저장소**까지 구축하는 전 과정을 다룬다.<br>
> AWS SNS SMS 시도 및 실패 → Solapi 전환 → 멀티 파드 OTP 문제 발견 → Valkey(Redis 호환) Sentinel HA 구성까지의 여정을 그대로 담았다.
{: .prompt-info }

---

## 1. 요구사항 정의

```
회원가입 시 휴대폰 번호 입력
  → 인증번호 발송 버튼 클릭 → 6자리 숫자 SMS 수신
  → 3분(180초) 내에 인증번호 입력 → 확인
  → 인증 완료 → 회원가입 진행

조건: EKS replicas: 2 환경에서 어느 파드가 요청을 받더라도 동일하게 동작
```

---

## 2. AWS SNS SMS 시도 및 실패

### 왜 SNS를 먼저 고려했나

기존 인프라가 AWS이므로 별도 외부 서비스 없이 SNS SMS를 우선 검토했다.

**설계 방향:**
- IAM Role에 `sns:Publish` 권한 추가
- `@aws-sdk/client-sns` SDK로 `PublishCommand` 호출
- `ap-northeast-2` 리전 SNS SMS Sandbox에 발신번호 등록

### 실패 원인 1: Terraform AWS Provider v5 미지원

SNS SMS Sandbox 번호 등록을 Terraform으로 관리하려 했으나 오류가 발생했다.

```
Error: The provider hashicorp/aws does not support resource type
       "aws_sns_sms_sandbox_phone_number"
```

AWS Provider v5에서 해당 리소스 타입이 제거된 상태였다. Terraform 코드를 revert하고 CLI로 직접 등록했다.

```bash
# CLI 직접 등록
aws sns create-sms-sandbox-phone-number \
  --phone-number "+8210xxxxxxxx" \
  --region ap-northeast-2

# 수신된 OTP로 검증
aws sns verify-sms-sandbox-phone-number \
  --phone-number "+8210xxxxxxxx" \
  --one-time-password "913633" \
  --region ap-northeast-2
```

### 실패 원인 2: 기업 VPN이 SNS SMS 발신 트래픽 차단

Sandbox 등록 및 `PublishCommand` 호출까지는 성공했으나 실제 문자가 수신되지 않았다.

> **원인**: AWS SNS SMS는 AWS 글로벌 인프라를 통해 발신되는데, 기업 망 내부에서 해당 트래픽이 필터링되고 있었다.<br>
> 기업 VPN 정책이 AWS SNS SMS 발신 트래픽을 차단하는 케이스는 생각보다 흔하다.<br>
> 국내 SMS 전문 사업자 API는 일반 HTTPS 아웃바운드로 나가므로 VPN 영향을 받지 않는다.
{: .prompt-danger }

### SNS 완전 제거

더 이상 진행이 불가능하므로 SNS 관련 코드를 전부 제거했다.

- `terraform/modules/cloudfront/main.tf` → `sns:Publish` IAM 정책 statement 삭제
- `applications/purina-api/package.json` → `@aws-sdk/client-sns` 의존성 삭제
- `applications/purina-api/src/routes/auth.js` → SNS 코드 전체 삭제

---

## 3. Solapi SMS 연동

### Solapi를 선택한 이유

국내 SMS 전문 사업자인 Solapi는 EKS NAT Gateway를 통해 외부로 나가는 **HTTPS API 방식**이므로 VPN 차단 영향을 받지 않는다.
임시 api 테스팅을 위해 Solapi를 사용하고, 추 후에는 당사의 써드파티 업체를 활용할 예정이다.

**사전 설정:**
- 발신번호 등록 (사전 심사 완료)
- Solapi API Key 보안 정책상 NAT Gateway 3개 IP를 허용 IP로 등록

> NAT Gateway IP는 EKS 파드가 외부로 나갈 때 사용하는 고정 아웃바운드 IP다. 각 AZ별로 별도 NAT Gateway가 있으므로 3개 모두 등록해야 한다.
{: .prompt-tip }

### Secrets Manager 저장 (Terraform)

Solapi API Key/Secret은 코드에 하드코딩하지 않고 Secrets Manager → ESO → K8s Secret 체계로 관리한다.

```hcl
# terraform/envs/dev/main.tf
resource "aws_secretsmanager_secret" "solapi" {
  name        = "<prefix>-solapi-apne2-secret"
  description = "Solapi SMS API 인증 정보"
}

resource "aws_secretsmanager_secret_version" "solapi" {
  secret_id = aws_secretsmanager_secret.solapi.id
  secret_string = jsonencode({
    api_key       = var.solapi_api_key
    api_secret    = var.solapi_api_secret
    sender_number = var.solapi_sender_number
  })
  lifecycle {
    ignore_changes = [secret_string]  # 콘솔 직접 수정 시 Terraform이 덮어쓰지 않음
  }
}
```

```hcl
# terraform/envs/dev/variables.tf
variable "solapi_api_key"       { type = string; sensitive = true }
variable "solapi_api_secret"    { type = string; sensitive = true }
variable "solapi_sender_number" { type = string; sensitive = true }
```

```hcl
# terraform.secret.tfvars (git 미추적)
solapi_api_key       = "<your-api-key>"
solapi_api_secret    = "<your-api-secret>"
solapi_sender_number = "<your-phone-number>"
```

### External Secret (ESO)

```yaml
# kubernetes/external-secrets/external-secret-solapi.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: solapi-secret
  namespace: <app-namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: solapi-secret
    creationPolicy: Owner
  data:
  - secretKey: SOLAPI_API_KEY
    remoteRef:
      key: <prefix>-solapi-apne2-secret
      property: api_key
  - secretKey: SOLAPI_API_SECRET
    remoteRef:
      key: <prefix>-solapi-apne2-secret
      property: api_secret
  - secretKey: SOLAPI_SENDER_NUMBER
    remoteRef:
      key: <prefix>-solapi-apne2-secret
      property: sender_number
```

### K8s Deployment 업데이트

```yaml
# kubernetes/apps/deployment/deployment-api.yaml
containers:
- name: api
  envFrom:
  - secretRef:
      name: mysql-secret
  - secretRef:
      name: solapi-secret    # 신규 추가
  env:
  - name: NODE_ENV
    value: "production"      # dev 전용 로직 제어용
```

### API 구현

```javascript
// applications/purina-api/src/routes/auth.js
const { SolapiMessageService } = require("solapi");

const solapi = new SolapiMessageService(
  process.env.SOLAPI_API_KEY,
  process.env.SOLAPI_API_SECRET
);

router.post("/send-sms", async (req, res) => {
  const { phone } = req.body;
  const digits = phone.replace(/\D/g, "");

  const code = String(Math.floor(100000 + Math.random() * 900000));

  await solapi.send({
    to: digits,
    from: process.env.SOLAPI_SENDER_NUMBER,
    text: `성환펫월드 인증번호입니다.\n3분내에 입력해주세요.\n인증번호: ${code}`,
  });

  res.json({ success: true });
});
```

> **Solapi SDK v6 트러블슈팅**<br>
> 문서를 잘못 해석해서 초기에 `sendOne({ message: { ... } })` 중첩 구조로 구현했다가 오류가 발생했다.<br>
> SDK v6 기준으로 `sendOne`은 deprecated이며, `send()`에 flat 구조로 전달해야 한다.
{: .prompt-warning }

```javascript
// ❌ 잘못된 구현
await solapi.sendOne({ message: { to: digits, from: ..., text: ... } });

// ✅ 올바른 구현
await solapi.send({ to: digits, from: ..., text: ... });
```

---

## 4. 멀티 파드 OTP 문제 발견 및 분석

### 증상

SMS 발송은 정상이나 인증번호 입력 후 확인 버튼 클릭 시:

```
인증 요청이 없거나 만료되었습니다.
```

방금 받은 인증번호인데 만료됐다는 응답이 반복됐다.

### 원인 분석 — 서버 메모리 저장의 함정

초기 구현에서 OTP를 서버 메모리(`Map`)에 저장했다.

```javascript
// 문제가 있던 코드
const otpStore = new Map();

// POST /send-sms → Pod A가 처리
const code = "382941";
otpStore.set(phone, { code, expiresAt: Date.now() + 180000 });

// POST /verify-sms → Pod B가 처리 → Pod B의 otpStore에는 해당 키가 없음
const entry = otpStore.get(phone); // undefined → "만료" 에러
```

```
[사용자 요청 흐름]
POST /auth/send-sms   → kube-proxy → Pod A (Map에 저장: otp["010..."] = "382941")
POST /auth/verify-sms → kube-proxy → Pod B (Map에 없음 → "만료" 에러)
```

> **kube-proxy의 iptables 기반 로드밸런싱**은 같은 클라이언트를 반드시 같은 파드로 보내지 않는다.<br>
> 특히 두 요청 사이에 시간이 있는 OTP 플로우에서는 다른 파드로 라우팅될 가능성이 높다.<br>
> 이처럼 파드 간 공유되지 않는 로컬 상태를 갖는 것을 **"Stateful 안티패턴"** 이라 한다.
{: .prompt-danger }

### 해결 방향 비교

| 방법 | 결론 |
|---|---|
| Sticky Session (IP Hash) | K8s ClusterIP에서 구현 어려움, Pod 재시작 시 세션 소실 |
| MySQL OTP 테이블 | 가능하나 인메모리 대비 레이턴시 높음 |
| HMAC 서명 (저장 없음) | 보안 복잡도 증가, 즉시 무효화 어려움 |
| **Redis (외부 공유 저장소)** | 멀티 파드 공유, TTL 네이티브 지원, 업계 표준 ✅ |

> **Redis를 선택한 이유**<br>
> `SETEX key TTL value` 하나로 저장 + 만료를 원자적으로 처리한다.<br>
> 멀티 파드 공유 저장소로 업계 표준이며, 이후 세션, 장바구니 캐싱, Rate Limiting 등으로 확장하기 용이하다.
{: .prompt-info }

---

## 5. ElastiCache 도입 시도 및 SCP 차단

### Terraform 코드 설계

Security Group은 기존 `terraform/modules/security-groups/` 모듈에서 관리하는 원칙에 따라 인라인으로 넣지 않고 모듈에 추가한 뒤 output으로 참조했다.

```hcl
# terraform/modules/security-groups/main.tf 추가분
resource "aws_security_group" "redis" {
  name        = "${local.prefix}-redis-apne2-sg"
  description = "ElastiCache Redis - allow 6379 from EKS nodes"
  vpc_id      = var.vpc_id
}

resource "aws_vpc_security_group_ingress_rule" "redis_from_vpc" {
  security_group_id = aws_security_group.redis.id
  cidr_ipv4         = var.vpc_cidr   # VPC 내부에서만 접근 허용
  from_port         = 6379
  to_port           = 6379
  ip_protocol       = "tcp"
}
```

```hcl
# terraform/envs/dev/main.tf
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "${local.prefix}-redis-apne2"
  description          = "OTP 분산 저장"
  node_type            = "cache.t4g.micro"
  num_cache_clusters   = 1
  engine_version       = "7.1"
  port                 = 6379

  subnet_group_name  = module.vpc.cache_subnet_group_name
  security_group_ids = [module.security_groups.redis_sg_id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = false

  depends_on = [module.vpc, module.security_groups]
}
```

> **`aws_elasticache_cluster` vs `aws_elasticache_replication_group`**<br>
> 단일 노드라도 `replication_group`을 사용하는 것이 권장된다.<br>
> `aws_elasticache_cluster`에서 Redis는 deprecated 방향이며, `replication_group`이 Multi-AZ 전환 등 확장에 유연하다.
{: .prompt-tip }

### SCP 차단

```
Error: creating ElastiCache Replication Group:
AccessDenied: User: arn:aws:sts::<account-id>:assumed-role/AWSReservedSSO_...
is not authorized to perform: elasticache:CreateReplicationGroup
with an explicit deny in a service control policy
```

AWS Organization 수준의 SCP가 `elasticache:CreateReplicationGroup`을 **명시적으로 차단**하고 있었다. `explicit deny`는 어떤 역할로도 우회가 불가능하다.

> **SCP(Service Control Policy)란?**<br>
> AWS Organizations에서 계정 단위로 특정 AWS 서비스/작업을 허용/차단하는 정책이다.<br>
> IAM 권한이 아무리 높아도 SCP의 `explicit deny`는 Override할 수 없다.<br>
> 기업 환경에서는 비용 관리, 보안 정책, 컴플라이언스 이유로 특정 서비스를 차단하는 경우가 많다.
{: .prompt-warning }

Terraform 코드는 커밋된 상태이므로 **SCP 승인 후 `terraform apply` 한 번으로 즉시 적용** 가능하다.

---

## 6. Redis vs Valkey — 기술 선택 배경

### Redis 라이선스 변경 이슈

2024년 Redis Ltd.가 라이선스를 **BSD → SSPL + RSALv2**로 변경했다. 클라우드 사업자가 Redis를 managed service로 제공할 수 없게 만든 사실상의 상업적 제한이었다.

이에 AWS, Google, Oracle, Ericsson 등이 반발해 Redis 7.2를 포크, **Linux Foundation** 산하에 **Valkey** 프로젝트를 출범시켰다.

| 항목 | Redis 7.4+ | Valkey |
|---|---|---|
| 라이선스 | SSPL + RSALv2 (비 OSI) | BSD-3-Clause (OSI 오픈소스) |
| 관리 주체 | Redis Ltd. | Linux Foundation |
| AWS 지원 | ElastiCache 레거시 | ElastiCache / MemoryDB 신규 기본값 |
| 클라이언트 호환 | — | ioredis 등 Redis 클라이언트 그대로 사용 |
| 프로토콜 | — | Redis 완전 호환 (drop-in replacement) |

### ElastiCache for Valkey도 SCP 차단 대상

ElastiCache for Valkey는 Valkey를 엔진으로 쓰는 것일 뿐, API는 동일한 `elasticache:CreateReplicationGroup`이다. SCP 차단이 Valkey에도 동일하게 적용된다.

```hcl
# ElastiCache for Valkey (SCP 승인 후 적용 예정)
resource "aws_elasticache_replication_group" "redis" {
  engine         = "valkey"   # Redis → Valkey 변경만 하면 됨
  engine_version = "7.2"
  ...
}
```

### Bitnami Valkey on EKS 선택

Valkey 오픈소스를 EKS 내부에 직접 배포하면 ElastiCache API를 전혀 사용하지 않으므로 SCP 무관하다.

| 방식 | SCP 필요 | HA | 운영 부담 |
|---|:---:|:---:|:---:|
| ElastiCache for Redis | ✅ 필요 | AWS 관리 | 없음 |
| ElastiCache for Valkey | ✅ 필요 | AWS 관리 | 없음 |
| Amazon MemoryDB for Valkey | ✅ 필요 | AWS 관리 + 무손실 | 없음 |
| **Valkey on EKS (Bitnami)** | ❌ 불필요 | Sentinel 구성 | 직접 관리 |

> ElastiCache SCP가 승인되면 `REDIS_SENTINEL_HOSTS` 환경변수를 `REDIS_URL`로 교체하는 것만으로 마이그레이션이 완료된다. `ioredis` 클라이언트 코드 변경 없음.
{: .prompt-tip }

---

## 7. Valkey Helm 배포 구성 (ArgoCD GitOps)

### ArgoCD GitOps 방식으로 Helm 관리

기존 Helm 차트들은 직접 `helm upgrade` 명령으로 관리되고 있었다. 이번 Valkey를 계기로 **ArgoCD Application + Helm source 방식**으로 전환했다.

> **직접 `helm install` 방식의 문제점**<br>
> Git에 values.yaml이 있어도 클러스터 실제 상태와 일치 보장이 없다.<br>
> ArgoCD selfHeal 없이 누군가 파드를 직접 수정하면 그대로 유지된다.<br>
> Valkey부터 ArgoCD가 관리하도록 하고 이후 추가되는 Helm 차트는 동일 방식으로 통일한다.
{: .prompt-info }

### ArgoCD Multiple Sources 활용

ArgoCD v3.4.3은 multiple sources를 지원한다. Helm 차트는 Bitnami 레지스트리에서, values.yaml은 Git에서 관리한다.

```yaml
# kubernetes/argocd/application-valkey.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <prefix>-valkey
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  sources:
  - repoURL: https://charts.bitnami.com/bitnami
    chart: valkey
    targetRevision: "6.1.11"
    helm:
      releaseName: valkey           # 반드시 명시 (미지정 시 Application 이름이 사용됨)
      valueFiles:
      - $values/helm/valkey/values.yaml
  - repoURL: git@github.com:<github-username>/<repo-name>.git
    targetRevision: main
    ref: values                     # git ref 이름 (valueFiles에서 $values로 참조)
  destination:
    server: https://kubernetes.default.svc
    namespace: <app-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

### Sentinel HA 구성

```yaml
# helm/valkey/values.yaml
architecture: replication

auth:
  enabled: false  # VPC 내부 접근, 네트워크 레벨 보안으로 대체

sentinel:
  enabled: true
  primarySet: myprimary
  quorum: 2         # 3개 sentinel 중 2개 합의 시 failover

primary:
  tolerations:
    - key: dedicated
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system
  persistence:
    enabled: true
    storageClass: gp3
    size: 2Gi
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

replica:
  replicaCount: 3
  tolerations:
    - key: dedicated
      value: system
      effect: NoSchedule
  nodeSelector:
    role: system
  persistence:
    enabled: true
    storageClass: gp3
    size: 2Gi
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

**Sentinel HA 구조:**

```
valkey-node-0  (primary)  → system 노드 AZ-c  + gp3 EBS 2Gi
valkey-node-1  (replica)  → system 노드 AZ-a  + gp3 EBS 2Gi
valkey-node-2  (replica)  → system 노드 AZ-c  + gp3 EBS 2Gi

Sentinel 프로세스: 각 파드 sidecar 컨테이너 (26379 포트)
  → Primary 장애 감지 시 quorum(2/3) 합의 후 Replica 자동 승격
```

> **Sentinel 동작 원리**<br>
> Sentinel 3개가 Primary를 지속적으로 모니터링한다.<br>
> Primary가 응답하지 않으면 quorum(2/3 이상) 합의 후 Replica를 새 Primary로 승격한다.<br>
> 클라이언트(ioredis)는 Sentinel에 먼저 연결해 현재 Primary 주소를 조회하므로, Primary가 바뀌어도 자동으로 새 Primary에 연결된다.
{: .prompt-info }

---

## 8. 트러블슈팅: releaseName 미지정 문제

### 증상

최초 배포 시 `releaseName`을 지정하지 않아 ArgoCD Application 이름이 Helm release name으로 사용됐다.

```
# 실제 생성된 리소스 (잘못된 상태)
Pod:     <prefix>-valkey-node-0
Service: <prefix>-valkey-headless

# deployment-api.yaml에 설정한 sentinel 호스트
valkey-node-0.valkey-headless:26379  ← 실제 서비스명과 불일치 → 연결 실패
```

### 해결

```yaml
helm:
  releaseName: valkey   # 명시적 지정
```

`releaseName: valkey` 추가 후 ArgoCD가 기존 리소스를 삭제하고 올바른 이름으로 재생성했다.

```bash
# 정상화 확인
kubectl get pods -n <app-namespace> | grep valkey
# valkey-node-0   Running
# valkey-node-1   Running
# valkey-node-2   Running

# Sentinel 정상 동작 확인
kubectl exec -n <app-namespace> valkey-node-0 \
  -c valkey -- valkey-cli -p 26379 SENTINEL masters
# name: myprimary
# flags: master
```

> **ArgoCD에서 Helm chart 배포 시 `releaseName`을 반드시 명시해야 한다.**<br>
> 미지정 시 `metadata.name`이 release name으로 사용되어 리소스 이름이 예상과 달라진다.
{: .prompt-danger }

---

## 9. api Redis OTP 구현

### ioredis Sentinel 연결

단순 `REDIS_URL` (단일 엔드포인트) 방식으로는 Sentinel failover 시 자동 재연결이 되지 않는다. ioredis의 **Sentinel 모드**를 사용해야 Primary가 바뀔 때 자동으로 새 Primary에 연결된다.

```javascript
// applications/purina-api/src/routes/auth.js
const Redis = require("ioredis");
const { SolapiMessageService } = require("solapi");

// 환경별 분기: Sentinel 환경변수가 있으면 Sentinel 모드, 없으면 단일 연결
const redis = process.env.REDIS_SENTINEL_HOSTS
  ? new Redis({
      sentinels: process.env.REDIS_SENTINEL_HOSTS.split(",").map((h) => {
        const [host, port] = h.split(":");
        return { host, port: parseInt(port, 10) };
      }),
      name: process.env.REDIS_SENTINEL_MASTER || "myprimary",
    })
  : new Redis(process.env.REDIS_URL || "redis://localhost:6379");

const solapi = new SolapiMessageService(
  process.env.SOLAPI_API_KEY,
  process.env.SOLAPI_API_SECRET
);

// ── POST /auth/send-sms ────────────────────────────────────────────
router.post("/send-sms", async (req, res) => {
  try {
    const { phone } = req.body;
    if (!phone) return res.status(400).json({ error: "휴대폰 번호를 입력해주세요." });

    const digits = phone.replace(/\D/g, "");
    if (digits.length < 10 || digits.length > 11) {
      return res.status(400).json({ error: "올바른 휴대폰 번호를 입력해주세요." });
    }

    const code = String(Math.floor(100000 + Math.random() * 900000));

    // 모든 파드가 공유하는 Valkey에 180초 TTL로 저장
    await redis.setex(`otp:${phone}`, 180, code);

    try {
      await solapi.send({
        to: digits,
        from: process.env.SOLAPI_SENDER_NUMBER,
        text: `성환펫월드 인증번호입니다.\n3분내에 입력해주세요.\n인증번호: ${code}`,
      });
    } catch (solapiErr) {
      console.error("Solapi send error:", solapiErr);
      if (process.env.NODE_ENV === "production") {
        return res.status(500).json({ error: "SMS 발송에 실패했습니다." });
      }
      // 로컬 dev에서는 로그로 OTP 확인
      console.log(`[DEV] OTP for ${phone}: ${code}`);
    }

    res.json({ success: true });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "SMS 발송 실패" });
  }
});

// ── POST /auth/verify-sms ──────────────────────────────────────────
router.post("/verify-sms", async (req, res) => {
  const { phone, code } = req.body;
  if (!phone || !code)
    return res.status(400).json({ error: "휴대폰 번호와 인증번호를 입력해주세요." });

  // 어느 파드가 요청을 받더라도 동일한 Valkey에서 조회
  const stored = await redis.get(`otp:${phone}`);
  if (!stored)
    return res.status(400).json({ error: "인증 요청이 없거나 만료되었습니다." });
  if (stored !== String(code))
    return res.status(400).json({ error: "인증번호가 일치하지 않습니다." });

  // 인증 성공 즉시 삭제 (재사용 방지)
  await redis.del(`otp:${phone}`);
  res.json({ verified: true });
});
```

**보안 설계 포인트:**

| 항목 | 설계 | 이유 |
|---|---|---|
| 인증 성공 시 즉시 삭제 | `redis.del()` | 동일 코드 재사용 방지 |
| 180초 TTL | `SETEX` | 만료 후 자동 삭제, 별도 cleanup 불필요 |
| 키 형식 `otp:{phone}` | prefix 구분 | 다른 Redis 키와 네임스페이스 분리 |
| 네트워크 접근 제한 | VPC CIDR 6379 허용만 | 외부 인터넷에서 Redis 직접 접근 불가 |

### Deployment 환경변수

```yaml
# kubernetes/apps/deployment/deployment-api.yaml
env:
- name: REDIS_SENTINEL_HOSTS
  value: "valkey-node-0.valkey-headless.<app-namespace>.svc.cluster.local:26379,valkey-node-1.valkey-headless.<app-namespace>.svc.cluster.local:26379,valkey-node-2.valkey-headless.<app-namespace>.svc.cluster.local:26379"
- name: REDIS_SENTINEL_MASTER
  value: "myprimary"
```

**환경별 연결 방식:**

| 환경 | 설정 | 동작 |
|---|---|---|
| K8s (현재) | `REDIS_SENTINEL_HOSTS` | Sentinel HA 모드 |
| 로컬 dev | `REDIS_URL=redis://localhost:6379` | 단일 연결 |
| ElastiCache 마이그레이션 후 | `REDIS_SENTINEL_HOSTS` 제거 + `REDIS_URL` | 단일 연결 |

---

## 10. 트러블슈팅: package-lock.json 불일치

### 문제

`ioredis`를 `package.json`에 **직접 텍스트 편집**으로 추가했다. 이 경우 `package-lock.json`이 업데이트되지 않아 CI 빌드가 실패했다.

```dockerfile
# applications/purina-api/Dockerfile
RUN npm ci --only=production
# npm ci는 package-lock.json과 package.json이 정확히 일치해야 함
```

```
Error: npm ci can only install packages when your package.json and
package-lock.json are in sync.
```

### 해결

```bash
# ❌ 잘못된 방법 — package.json 직접 텍스트 편집
# package-lock.json이 업데이트되지 않아 npm ci 실패

# ✅ 올바른 방법
cd applications/purina-api
npm install ioredis   # package.json + package-lock.json 동시 업데이트
git add package.json package-lock.json
git commit -m "feat: add ioredis dependency"
```

> 패키지 추가/제거는 반드시 `npm install <package>` 명령으로 진행해야 한다. 직접 텍스트 편집은 절대 금지.
{: .prompt-danger }

---

## 11. Valkey 데이터 저장 구조

### 물리적 저장 위치

Valkey는 두 곳에 동시에 데이터를 저장한다.

```
[api] → redis.setex("otp:010...", 180, "382941")
                    ↓
         ┌──────────────────────────┐
         │      Valkey 파드 내부     │
         │                          │
         │  ① RAM (메모리)           │ ← 모든 읽기/쓰기 (us 단위 응답)
         │        ↓ 주기적 스냅샷    │
         │  ② EBS gp3 디스크         │ ← /bitnami/valkey/data/dump.rdb
         └──────────────────────────┘
```

**① RAM** — 실제 서빙 레이어. 모든 `GET`/`SET` 명령이 메모리에서 처리된다. Valkey가 빠른 이유다.

**② EBS gp3** — 영속성 레이어. 주기적으로 메모리 전체를 디스크에 스냅샷(RDB)으로 저장한다. 파드가 재시작되면 `dump.rdb`를 읽어 메모리를 복구한다.

```bash
# 컨테이너 내부에서 확인
kubectl exec -n <app-namespace> valkey-node-0 \
  -c valkey -- ls /bitnami/valkey/data/
# dump.rdb
```

### RDB vs AOF 영속성 방식

| 방식 | 동작 | 특성 |
|---|---|---|
| **RDB** (현재) | 일정 간격으로 메모리 전체 스냅샷 | 가볍고 빠름, 마지막 스냅샷 이후 데이터 유실 가능 |
| **AOF** | 모든 쓰기 명령을 로그에 순차 기록 | 유실 없음, 파일 크기 크고 복구 느림 |

> OTP는 TTL 180초짜리 임시 데이터라 유실되어도 재발급으로 해결 가능하므로 RDB로 충분하다.<br>
> 세션/장바구니처럼 유실이 민감한 데이터가 추가되면 AOF 활성화를 검토해야 한다.
{: .prompt-info }

### 키 네이밍 컨벤션

Redis/Valkey는 테이블이 없는 **flat key-value 스토어**다. `:` 구분자로 논리적 네임스페이스를 구분하는 것이 업계 표준이다.

```
현재 사용 중:
  otp:{phone}              → "382941"        TTL 180초

향후 확장 계획:
  session:{userId}         → JSON 유저 정보   TTL 7일
  cart:{userId}            → JSON 장바구니    TTL 30일
  ratelimit:sms:{phone}    → "3"             TTL 60초 (SMS 스팸 방지)
  blacklist:{token}        → "1"             TTL = JWT 만료 시간 (강제 로그아웃)
```

### 향후 Valkey 확장 계획

**SMS Rate Limiting (스팸 방지)**

```javascript
const key = `ratelimit:sms:${phone}`;
const count = await redis.incr(key);
if (count === 1) await redis.expire(key, 60); // 첫 요청 시 1분 TTL 설정
if (count > 3) return res.status(429).json({ error: "1분에 3회까지만 발송 가능합니다." });
```

**JWT 블랙리스트 (강제 로그아웃)**

```javascript
// 로그아웃 또는 계정 정지 시
await redis.setex(`blacklist:${token}`, 60 * 60 * 24 * 7, "1");

// 미들웨어에서 매 요청 검증
const isBlacklisted = await redis.get(`blacklist:${token}`);
if (isBlacklisted) return res.status(401).json({ error: "만료된 토큰입니다." });
```

**장바구니 캐싱**

```javascript
// MySQL 왕복 없이 빠른 장바구니 업데이트
await redis.hset(`cart:${userId}`, itemId, JSON.stringify({ qty, price }));
await redis.expire(`cart:${userId}`, 60 * 60 * 24 * 30);

// 결제 시점에 MySQL로 영구 저장
const cart = await redis.hgetall(`cart:${userId}`);
await db.query("INSERT INTO orders ...");
await redis.del(`cart:${userId}`);
```

---

## 12. 회원가입 OTP 인증 성공 확인

모든 수정 완료 후 실제 테스트 성공.

```
1. 휴대폰 번호 입력 → POST /auth/send-sms
2. api Pod A → ioredis Sentinel → Valkey Primary에 OTP 저장 (180초 TTL)
   redis.setex("otp:010...", 180, "382941")
3. SMS 수신 (Solapi 발송)
4. 인증번호 입력 → POST /auth/verify-sms
5. api Pod B → ioredis Sentinel → 동일 Valkey Primary에서 OTP 조회
   → 멀티 파드 문제 해결 ✅
6. 인증 성공 → redis.del("otp:010...") → 재사용 방지
7. 회원가입 완료
```

---

## 13. 현재 상태 및 후속 작업

### 현재 상태

| 항목 | 상태 | 비고 |
|---|---|---|
| Solapi SMS 발송 | ✅ 정상 | NAT Gateway IP 허용 등록 완료 |
| Secrets Manager (Solapi) | ✅ 완료 | ESO로 K8s Secret 동기화 중 |
| Valkey Sentinel HA | ✅ 정상 | node-0/1/2 모두 Running |
| gp3 EBS Persistence | ✅ 정상 | 각 노드 2Gi EBS 볼륨 연결 |
| 회원가입 OTP 인증 | ✅ 성공 | 멀티 파드 환경에서 정상 동작 확인 |
| ElastiCache Terraform 코드 | ✅ 완료 | SCP 승인 후 `terraform apply` 대기 |

### 후속 작업

> **1단계: SCP 예외 신청**<br>
> `elasticache:CreateReplicationGroup`, `DescribeReplicationGroups`, `DeleteReplicationGroup`, `ModifyReplicationGroup` 허용 요청
{: .prompt-info }

> **2단계: SCP 승인 시 ElastiCache 마이그레이션**<br>
> `terraform apply` → endpoint 확인 → `REDIS_SENTINEL_HOSTS` 제거 + `REDIS_URL` 추가 → `helm uninstall valkey`
{: .prompt-tip }

> **3단계: Toss Payments 결제 API 개발**<br>
> Redis와 독립적으로 진행 가능. 재고 감소 로직의 atomic INCR/DECR은 Redis 연동 후 고도화 예정.
{: .prompt-tip }

> **4단계: 카카오/네이버 소셜 로그인 (간편 로그인) 도입**<br>
> OAuth 2.0 Authorization Code Flow 기반으로 구현 예정.<br>
> 카카오 로그인: `https://kauth.kakao.com/oauth/authorize` → callback → JWT 발급<br>
> 네이버 로그인: `https://nid.naver.com/oauth2.0/authorize` → callback → JWT 발급<br>
> 기존 이메일/비밀번호 로그인과 동일한 JWT 체계를 유지하고, `users` 테이블에 `provider`, `provider_id` 컬럼을 추가하는 방식으로 통합할 예정.
{: .prompt-info }

---

## 변경된 파일 목록

```
applications/purina-api/
  package.json                               # ioredis 추가, @aws-sdk/client-sns 제거
  package-lock.json                          # npm install로 동기화
  src/routes/auth.js                         # SNS 제거, Solapi + Valkey OTP 전면 재작성

terraform/envs/dev/
  main.tf                                    # Solapi Secret, ElastiCache (SCP 대기)
  outputs.tf                                 # redis_primary_endpoint 출력 추가
  variables.tf                               # solapi 변수 추가

terraform/modules/security-groups/
  main.tf                                    # Redis Security Group 추가
  outputs.tf                                 # redis_sg_id 출력 추가

terraform/modules/cloudfront/
  main.tf                                    # sns:Publish IAM 정책 제거

kubernetes/
  external-secrets/external-secret-solapi.yaml   # 신규 생성
  argocd/application-valkey.yaml                 # 신규 생성 (ArgoCD Helm Application)
  apps/deployment/deployment-api.yaml            # REDIS_SENTINEL_HOSTS, solapi-secret 추가
  helm/valkey/values.yaml                        # 신규 생성 (Sentinel HA values)
```
