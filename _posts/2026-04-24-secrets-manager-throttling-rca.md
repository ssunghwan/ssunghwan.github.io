---
title: "AWS Secrets Manager ThrottlingException"
date: 2026-04-24 00:00:00 +0900
categories: [Infrastructure, Root Cause Analysis]
tags: [aws, secrets-manager, throttling, rca, php, apcu, imdsv2, mod-php, php-fpm, sigv4]
---

> **환경**: AWS ap-northeast-2 / Production<br>
> **심각도**: High — 트래픽 집중 시 서비스 전면 장애 가능<br>
> **상태**: 정식 조치 진행 중
{: .prompt-danger }

---

## 1. 개요

### 발생 증상

운영 환경에서 AWS Secrets Manager API 호출 시 아래 오류가 간헐적으로 발생하였으며, 트래픽 집중 시 DB 연결 실패로 이어져 서비스 전면 장애 위험이 확인되었다.

```
Failed to get DB credentials: AWS API Error (HTTP 400):
{"__type":"ThrottlingException","message":"Request throttled due to congestion"}
```

### 영향 범위

| 항목 | 내용 |
|---|---|
| 장애 유형 | DB 연결 실패 → 서비스 전면 장애 |
| 발생 조건 | 트래픽 집중 시 (이벤트, 플래시 세일 등) |
| 현재 Fallback | **없음** — ThrottlingException 발생 시 즉시 `die()` 호출 |

### 핵심 요약

> 애플리케이션이 **사용자 HTTP 요청 1건당 Secrets Manager API를 1회 호출**하는 구조로, 캐싱이 전혀 없다.<br>
> EC2 3대가 각각 독립적으로 API를 호출하므로 트래픽 집중 시 AWS Secrets Manager 기본 TPS 한도(500 TPS)를 초과하여 ThrottlingException이 발생한다.
{: .prompt-warning }

---

## 2. 시스템 구성

### 인프라 구성

```
[사용자]
    │
    ▼
[CDN]
    │
    ▼
[ALB (Application Load Balancer)]
    │
    ├──────────────────┬──────────────────┐
    ▼                  ▼                  ▼
[EC2 #1]           [EC2 #2]           [EC2 #3]
Apache + mod_php   Apache + mod_php   Apache + mod_php
    │                  │                  │
    ├──────────────────┴──────────────────┤
    │              [EFS]                  │
    │       (사용자 세션 파일 공유)          │
    │                                     │
    ├─ curl ──▶ [EC2 Instance Metadata (IMDS)]
    │              IAM 임시 자격증명 획득
    │              ↑ 3대 각각 독립 호출 (캐싱 없음)
    │
    ├─ curl ──▶ [AWS Secrets Manager]     ← 문제 발생 지점
    │              DB Credential 조회
    │              ↑ 3대 각각 독립 호출 (캐싱 없음)
    │
    └─ TCP 3306 ──▶ [RDS Aurora MySQL 8.0]
```

> EC2 3대가 동일한 소스코드를 사용하며, 사용자 세션은 EFS를 통해 공유한다.<br>
> Secrets Manager credential은 3대 모두 동일한 값이지만, 각 서버가 독립적으로 매 요청마다 API를 호출하는 구조다.
{: .prompt-info }

### 애플리케이션 스택

| 항목 | 내용 |
|---|---|
| 웹 서버 | Apache HTTP Server |
| PHP 실행 방식 | **mod_php** (Apache 모듈로 내장) |
| EC2 대수 | **3대** (오토스케일링 없음, 고정 구성) |
| 소스코드 배포 | **동일 코드 복제** (3대 동일) |
| 세션 공유 | **EFS** (사용자 로그인 세션 파일 공유) |
| AWS SDK | **미사용** — SigV4 직접 서명 구현 |
| Composer | **미사용** |
| DB | RDS Aurora MySQL 8.0 |
| Secret 관리 | AWS Secrets Manager |

---

## 3. 오류 상세

### 요청당 API 호출 흐름

현재 구조에서 사용자 HTTP 요청 1건이 들어올 때 내부적으로 발생하는 호출:

```
사용자 요청 (1건)
    │
    ▼
Apache + mod_php
    │
    ▼
DB 설정 파일 로드 (require_once)
    │
    ├── curl #1 ──▶ IMDS — IAM Role 이름 조회
    │
    ├── curl #2 ──▶ IMDS — 임시 자격증명 조회
    │
    └── curl #3 ──▶ Secrets Manager GetSecretValue
```

> **사용자 요청 1건 = API 호출 3번 (IMDS 2회 + Secrets Manager 1회)**<br>
> 캐싱 없음 — 매 요청마다 반복
{: .prompt-danger }

### 현재 코드 구조 (문제 지점)

```php
// DB 연결 설정 파일 — 매 요청마다 실행됨
try {
    require_once '/path/to/lib.awsSecretsHelper.php';

    $helper = new AwsSecretsHelper();
    $creds  = $helper->getSecret($secretId, $secretARN);  // ← 매번 API 호출
} catch (Exception $e) {
    error_log("Failed to get DB credentials: " . $e->getMessage());
    die("Database configuration error.");  // ← Fallback 없이 즉시 서비스 중단
}
```

```php
// getSecret() 내부
public function getSecret($secretId, $secretARN) {
    // 캐싱 없음 — 항상 아래 실행
    $credentials = $this->getInstanceCredentials();  // IMDS 2회 curl
    // ... SigV4 서명
    // Secrets Manager API curl 1회
    return json_decode($result['SecretString'], true);
}
```

### 스로틀링 발생 조건

AWS Secrets Manager 기본 TPS 한도:

| API | 기본 TPS |
|---|---|
| `GetSecretValue` | 500 TPS |
| `DescribeSecret` | 50 TPS |
| `PutSecretValue` | 50 TPS |

> EC2 3대가 각각 독립적으로 API를 호출하므로 실제 호출량은 단일 서버 대비 3배다. ALB가 트래픽을 균등 분산하므로 서버당 약 167 TPS씩 발생하지만, 총합은 동일하게 500 TPS 한도에 수렴한다.<br>
> 이벤트나 플래시 세일 시 **동시 사용자 500명 기준 3대 합산 약 1,500 TPS**가 발생하여 한도를 즉시 초과한다.
{: .prompt-warning }

---

## 4. 근본 원인 분석 (RCA)

### 5 Whys 분석

| 단계 | Why | Answer |
|---|---|---|
| Why 1 | 왜 ThrottlingException이 발생했나? | Secrets Manager API 호출 횟수가 TPS 한도를 초과했기 때문 |
| Why 2 | 왜 API 호출 횟수가 한도를 초과했나? | 매 HTTP 요청마다 GetSecretValue를 호출하기 때문 |
| Why 3 | 왜 매 요청마다 API를 호출하나? | 캐싱 로직이 구현되어 있지 않기 때문 |
| Why 4 | 왜 캐싱이 없나? | AWS SDK 미사용 환경에서 캐싱 라이브러리를 별도 구현하지 않았기 때문 |
| Why 5 | 왜 SDK를 사용하지 않나? | Composer 미사용 환경으로 표준 SDK 설치가 불가능했기 때문 |

### 직접 원인

**캐싱 없는 Secrets Manager API 호출 구조**

`getSecret()` 함수가 호출될 때마다 아래를 반복한다.

- EC2 Instance Metadata 호출 (2회)
- Secrets Manager `GetSecretValue` API 호출 (1회)

DB 연결이 필요한 모든 페이지 요청에서 이 함수가 실행되므로, 트래픽에 비례하여 API 호출 횟수가 선형적으로 증가한다.

### 기여 원인

**① Fallback 로직 부재**

ThrottlingException 발생 시 `die()`를 즉시 호출하여 서비스가 중단된다. 캐시된 값이라도 반환하는 Fallback이 없다.

**② IMDSv1 사용**

현재 코드는 IMDSv1 방식(토큰 없이 직접 접근)으로 IMDS를 호출한다. IMDSv2에서는 PUT 요청으로 토큰을 먼저 발급받아야 하므로 보안상 IMDSv2 전환이 필요하다.

```php
// 현재 (IMDSv1) — 보안 취약
curl_init('http://169.254.169.254/latest/meta-data/iam/security-credentials/');

// 권장 (IMDSv2) — PUT 토큰 방식
curl_init('http://169.254.169.254/latest/api/token');  // PUT으로 토큰 먼저 발급
```

**③ mod_php 환경과 APCu**

mod_php는 Apache 프로세스에 PHP 인터프리터가 내장된 방식이다. APCu(APC User Cache)는 **동일 서버 내 Apache worker 간 공유 메모리**를 제공한다.

현재 환경(EC2 3대 고정, 오토스케일링 없음)에서는 아래와 같이 동작한다:

```
[EC2 #1] Apache worker들 → APCu 공유 메모리 (서버 내 공유) ✅
[EC2 #2] Apache worker들 → APCu 공유 메모리 (서버 내 공유) ✅
[EC2 #3] Apache worker들 → APCu 공유 메모리 (서버 내 공유) ✅

각 서버가 동일한 Secret 값을 독립적으로 캐싱
→ 서버 간 캐시 공유 불필요 (credential은 3대 모두 동일한 값)
```

> 오토스케일링 없이 3대 고정 구성이므로 서버 간 캐시 공유가 불필요하고, 각 서버 내 APCu 캐싱만으로 충분히 효과적이다.
{: .prompt-tip }

**④ AWS SDK 미사용**

AWS SDK for PHP는 기본적으로 재시도(Exponential Backoff), 캐싱 등의 기능을 제공하지만, Composer 미사용 환경으로 SDK를 적용하지 못하고 있다.

---

## 5. AWS Secrets Manager 동작 원리

### Secret 버전 스테이지

Secrets Manager는 Secret을 버전 단위로 관리하며, 각 버전에 스테이지 레이블을 부여한다.

```
AWSPREVIOUS  ←  이전 버전 (롤백용, Rotation 완료 후 자동 이동)
AWSCURRENT   ←  현재 활성 버전 (애플리케이션이 조회하는 버전)
AWSPENDING   ←  Rotation 진행 중인 버전 (완료 시 AWSCURRENT로 전환)
```

### Rotation(자동 교체) 동작 구조

```
createSecret → setSecret → testSecret → finishSecret
  새 PW 생성    DB에 PW 반영   접속 검증    버전 전환 완료
```

| 단계 | 역할 |
|---|---|
| `createSecret` | 랜덤 패스워드 생성 → `AWSPENDING` 버전으로 저장 |
| `setSecret` | `AWSCURRENT` 또는 `AWSPREVIOUS`로 DB 접속 → 패스워드 변경 |
| `testSecret` | `AWSPENDING` secret으로 DB 접속 검증 |
| `finishSecret` | `AWSPENDING` → `AWSCURRENT` 스테이지 전환 |

### 현재 Rotation 설정

| 항목 | 값 |
|---|---|
| Rotation 주기 | **90일** |
| 캐시 TTL (조치 후) | **24시간** |
| 안전 여부 | ✅ Rotation 후 최대 24시간 내 캐시 자동 갱신 |

> Rotation 주기(90일) >> 캐시 TTL(24시간)이므로 Rotation 후 별도 수동 캐시 삭제 불필요.
{: .prompt-info }

---

## 6. mod_php vs PHP-FPM 비교 분석

현재 환경은 `mod_php`를 사용하고 있다. 이번 이슈를 계기로 두 방식의 차이를 정리한다.

### 아키텍처 차이

**mod_php**

```
[Apache Process]
    ├── [PHP Interpreter] ← Apache 모듈로 내장
    ├── [PHP Interpreter]
    └── [PHP Interpreter]

특징: Apache worker와 PHP가 같은 프로세스에서 실행
     Apache가 재시작되면 PHP도 함께 재시작
```

**PHP-FPM (FastCGI Process Manager)**

```
[Apache Process]  ←→  [PHP-FPM Master]
                            ├── [PHP Worker Pool]
                            ├── [PHP Worker Pool]
                            └── [PHP Worker Pool]

특징: Apache와 PHP가 분리된 독립 프로세스
     PHP만 독립적으로 재시작 가능
     Worker Pool을 별도로 관리
```

### 상세 비교

| 항목 | mod_php | PHP-FPM |
|---|---|---|
| 실행 방식 | Apache 모듈로 내장 | 독립 FastCGI 프로세스 |
| 메모리 사용 | Apache worker당 PHP 메모리 상시 점유 | 요청 없을 때 PHP worker 유휴 가능 |
| **APCu 캐시 공유** | **worker 간 공유 가능** | **worker pool 내 안정적 공유** |
| 프로세스 격리 | Apache와 동일 프로세스 | 완전 분리 |
| 재시작 | Apache 전체 재시작 필요 | PHP-FPM만 독립 재시작 가능 |
| Worker Pool 설정 | Apache MPM 설정에 종속 | PHP-FPM 독립 설정 (pm.max_children 등) |
| 성능 (고트래픽) | Apache worker 수에 제한 | Worker Pool 동적 조정 가능 |
| Nginx 호환 | ❌ 불가 (Apache 전용) | ✅ Nginx + PHP-FPM 조합 가능 |
| 운영 복잡도 | 낮음 | 중간 |
| 현재 환경 적합성 | ✅ 현재 운영 중 | 마이그레이션 필요 |

### 캐싱 관점에서의 차이

**mod_php 환경 (현재)**

```
[EC2 #1]
Apache Worker #1 ─┐
Apache Worker #2 ─┼── APCu 공유 메모리 (서버 내 공유 가능) ✅
Apache Worker #3 ─┘

[EC2 #2] → 동일 구조 (독립 APCu)
[EC2 #3] → 동일 구조 (독립 APCu)

※ 3대 모두 동일한 credential → 서버 간 공유 불필요
→ 각 서버 내 APCu만으로 충분히 효과적
```

**PHP-FPM 환경 (전환 시)**

```
PHP-FPM Worker Pool
    ├── Worker #1 ─┐
    ├── Worker #2 ─┼── APCu 공유 메모리 (Pool 내 안정적 공유) ✅
    └── Worker #3 ─┘

→ Pool 설정으로 더 세밀한 제어 가능
→ Worker Pool 간 격리로 안정성 향상
```

### 현재 환경(3대 고정 + EFS)에서의 캐시 전략

| 항목 | 내용 |
|---|---|
| EC2 대수 | 3대 고정 (오토스케일링 없음) |
| 소스코드 | 동일 코드 복제 |
| 세션 공유 | EFS (사용자 로그인 세션 파일) |
| Credential 값 | 3대 모두 동일 |
| 캐시 공유 필요성 | **불필요** — 각 서버가 동일한 값을 독립 캐싱 |
| 권장 캐시 방식 | **APCu** — 메모리 기반, 서버 내 worker 간 공유 |

> EFS는 사용자 세션 파일 공유에만 사용되며 Secrets Manager 캐싱과는 무관하다.<br>
> Credential은 3대 모두 동일한 값이므로 EFS를 통한 캐시 공유 없이 각 서버가 독립적으로 APCu에 캐싱하는 것으로 충분하다.
{: .prompt-info }

### PHP-FPM 전환 시 얻을 수 있는 이점

| 이점 | 설명 |
|---|---|
| APCu 캐시 안정성 향상 | Worker Pool 단위 공유로 더 예측 가능한 동작 |
| 메모리 효율 향상 | 유휴 시 PHP worker 메모리 반환 가능 |
| 독립적 PHP 재시작 | 배포 시 Apache 재시작 없이 PHP-FPM만 reload 가능 |
| Nginx 전환 용이 | 향후 Apache → Nginx 전환 시 PHP-FPM 재사용 가능 |

### 현재 조치와의 관계

| 환경 | 권장 캐시 방식 | 비고 |
|---|---|---|
| 현재 (mod_php, 3대 고정) | **APCu** | 서버 내 worker 간 공유 가능, 3대 동일 credential로 충분 |
| PHP-FPM 전환 후 | **APCu** | Pool 단위 공유로 안정성 추가 향상 |

> 현재 환경에서도 APCu는 효과적이다. PHP-FPM 전환은 장기적 안정성과 운영 편의를 위한 것이며 당장 필수는 아니다.
{: .prompt-tip }

---

## 7. 조치 계획

### 임시 조치 (단기)

**AWS Support — TPS 한도 증가 요청**

```
Service Quota: Secrets Manager
Quota Name: GetSecretValue requests per second
Current: 500 TPS
Requested: 1,500 TPS
Region: ap-northeast-2
Reason: E-commerce platform experiencing ThrottlingException during peak traffic
```

### 정식 조치 — lib.awsSecretsHelper.php 수정

**변경 사항 요약**

| 항목 | 기존 | 변경 후 |
|---|---|---|
| 캐싱 방식 | 없음 | **APCu 메모리 캐시** TTL 86400초 (24시간) |
| Fallback | 없음 (die()) | ThrottlingException 시 에러 로그 기록 |
| IMDS 버전 | IMDSv1 | IMDSv2 (PUT 토큰) |
| 수정 파일 | — | `lib.awsSecretsHelper.php` **only** |
| db.php 수정 | — | **불필요** (호출 방식 변경 없음) |
| PHP 모듈 추가 | — | APCu 모듈 설치 필요 여부 사전 확인 필요 |
| 서버 재시작 | — | APCu 모듈 미설치 시 필요, 기설치 시 불필요 |

> **db.php는 수정 불필요**: `$helper->getSecret()` 호출 방식이 동일하므로 헬퍼 내부만 수정하면 된다.
{: .prompt-info }

**APCu 모듈 설치 여부 사전 확인**

```bash
# 3대 서버 모두 확인
php -m | grep apcu
# 또는
php -r "echo apcu_enabled() ? 'APCu enabled' : 'APCu not enabled';"
```

APCu 미설치 시 설치 필요 (서버 재시작 수반):

```bash
# Rocky Linux / Amazon Linux
sudo yum install -y php-pecl-apcu

# Ubuntu
sudo apt-get install -y php-apcu

# 설치 후 Apache 재시작
sudo systemctl restart httpd
```

**수정된 getSecret() 핵심 로직**

```php
public function getSecret($secretId, $secretARN) {
    // ── 1. APCu 캐시 확인 ──
    $cacheKey = 'aws_secret_' . md5($secretId);

    if (apcu_enabled() && apcu_exists($cacheKey)) {
        return apcu_fetch($cacheKey);  // 캐시 유효 → 즉시 반환 (API 호출 없음)
    }

    // ── 2. 캐시 miss — Secrets Manager API 호출 (기존 로직 동일) ──
    $credentials = $this->getInstanceCredentials();  // IMDSv2
    // ... SigV4 서명 및 curl

    $secret = json_decode($result['SecretString'], true);

    // ── 3. APCu 캐시 저장 (TTL 86400초 / 24시간) ──
    if (apcu_enabled()) {
        apcu_store($cacheKey, $secret, 86400);
    }

    return $secret;
}
```

**APCu 캐시 동작 (서버별)**

```
[EC2 #1 — APCu 메모리]
    Apache Worker #1 ─┐
    Apache Worker #2 ─┼── aws_secret_<md5> 공유 캐시 (TTL 24시간)
    Apache Worker #3 ─┘
    → 첫 요청만 Secrets Manager API 호출, 이후 메모리에서 즉시 반환

[EC2 #2 — APCu 메모리]  → 동일 구조
[EC2 #3 — APCu 메모리]  → 동일 구조

※ 3대 모두 동일한 credential 값 → 서버 간 공유 불필요
```

**수정된 전체 코드**

```php
<?php
/**
 * AWS Secrets Manager Helper (No SDK Required)
 * Uses AWS Signature Version 4 for authentication
 *
 * [변경 이력]
 * 2026-04-24 - APCu 메모리 캐시 추가 (TTL 86400초 / 24시간)
 *            - IMDSv1 → IMDSv2 (PUT 토큰 방식) 변경
 *            - ThrottlingException 발생 시 error_log 처리
 */

class AwsSecretsHelper {
    private $region   = 'ap-northeast-2';
    private $service  = 'secretsmanager';
    private $cacheTTL = 86400; // 24시간 (Rotation 주기 90일 대비 안전)

    public function getSecret($secretId, $secretARN) {
        // ── 1. APCu 캐시 확인 ──────────────────────────
        $cacheKey = 'aws_secret_' . md5($secretId);

        if (apcu_enabled() && apcu_exists($cacheKey)) {
            return apcu_fetch($cacheKey); // 캐시 유효 → 즉시 반환 (API 호출 없음)
        }

        // ── 2. 캐시 miss — Secrets Manager API 호출 ──
        $credentials  = $this->getInstanceCredentials();
        $host         = "secretsmanager.{$this->region}.amazonaws.com";
        $endpoint     = "https://{$host}/";
        $payload      = json_encode(['SecretId' => $secretId]);
        $signedHeaders = $this->signRequest('POST', $endpoint, $payload, [], $credentials);

        $ch = curl_init($endpoint);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
        curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $signedHeaders);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

        if (curl_errno($ch)) {
            throw new Exception("CURL Error: " . curl_error($ch));
        }
        curl_close($ch);

        if ($httpCode !== 200) {
            error_log("[AwsSecretsHelper] AWS API Error (HTTP {$httpCode}): {$response}");
            throw new Exception("AWS API Error (HTTP {$httpCode}): {$response}");
        }

        $result = json_decode($response, true);
        if (!isset($result['SecretString'])) {
            throw new Exception("Secret not found or invalid response");
        }

        $secret = json_decode($result['SecretString'], true);

        // ── 3. APCu 캐시 저장 ──────────────────────────
        if (apcu_enabled()) {
            apcu_store($cacheKey, $secret, $this->cacheTTL);
        }

        return $secret;
    }

    private function getInstanceCredentials() {
        // IMDSv2 — PUT 토큰 선발급
        $ch = curl_init('http://169.254.169.254/latest/api/token');
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 2);
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['X-aws-ec2-metadata-token-ttl-seconds: 21600']);
        $token = curl_exec($ch);
        curl_close($ch);

        if (empty($token)) {
            throw new Exception("Failed to retrieve IMDSv2 token");
        }

        // IAM Role 이름 조회
        $ch = curl_init('http://169.254.169.254/latest/meta-data/iam/security-credentials/');
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 2);
        curl_setopt($ch, CURLOPT_HTTPHEADER, ["X-aws-ec2-metadata-token: {$token}"]);
        $roleName = curl_exec($ch);
        curl_close($ch);

        if (empty($roleName)) {
            throw new Exception("No IAM role attached to EC2 instance");
        }

        // 임시 자격증명 조회
        $ch = curl_init("http://169.254.169.254/latest/meta-data/iam/security-credentials/{$roleName}");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 2);
        curl_setopt($ch, CURLOPT_HTTPHEADER, ["X-aws-ec2-metadata-token: {$token}"]);
        $credentialsJson = curl_exec($ch);
        curl_close($ch);

        $credentials = json_decode($credentialsJson, true);

        if (!isset($credentials['AccessKeyId'])) {
            throw new Exception("Failed to retrieve instance credentials");
        }

        return $credentials;
    }

    private function signRequest($method, $endpoint, $payload, $headers, $credentials) {
        $url      = parse_url($endpoint);
        $host     = $url['host'];
        $datetime = gmdate('Ymd\THis\Z');
        $date     = gmdate('Ymd');

        $canonicalHeaders  = "content-type:application/x-amz-json-1.1\n";
        $canonicalHeaders .= "host:{$host}\n";
        $canonicalHeaders .= "x-amz-date:{$datetime}\n";
        if (isset($credentials['Token'])) {
            $canonicalHeaders .= "x-amz-security-token:{$credentials['Token']}\n";
        }
        $canonicalHeaders .= "x-amz-target:secretsmanager.GetSecretValue\n";

        $signedHeaders = 'content-type;host;x-amz-date';
        if (isset($credentials['Token'])) {
            $signedHeaders .= ';x-amz-security-token';
        }
        $signedHeaders .= ';x-amz-target';

        $canonicalRequest = "POST\n/\n\n{$canonicalHeaders}\n{$signedHeaders}\n" . hash('sha256', $payload);
        $credentialScope  = "{$date}/{$this->region}/{$this->service}/aws4_request";
        $stringToSign     = "AWS4-HMAC-SHA256\n{$datetime}\n{$credentialScope}\n" . hash('sha256', $canonicalRequest);

        $kDate    = hash_hmac('sha256', $date,           'AWS4' . $credentials['SecretAccessKey'], true);
        $kRegion  = hash_hmac('sha256', $this->region,   $kDate,    true);
        $kService = hash_hmac('sha256', $this->service,  $kRegion,  true);
        $kSigning = hash_hmac('sha256', 'aws4_request',  $kService, true);
        $signature = hash_hmac('sha256', $stringToSign,  $kSigning);

        $authorization = "AWS4-HMAC-SHA256 Credential={$credentials['AccessKeyId']}/{$credentialScope}, SignedHeaders={$signedHeaders}, Signature={$signature}";

        $finalHeaders = [
            'Content-Type: application/x-amz-json-1.1',
            'Host: ' . $host,
            'X-Amz-Date: ' . $datetime,
            'X-Amz-Target: secretsmanager.GetSecretValue',
            'Authorization: ' . $authorization,
        ];

        if (isset($credentials['Token'])) {
            $finalHeaders[] = 'X-Amz-Security-Token: ' . $credentials['Token'];
        }

        return $finalHeaders;
    }
}
```

### 캐시 동작 흐름 (조치 후)

```
사용자 요청
    │
    ▼
getSecret() 호출
    │
    ├── APCu 캐시 유효? (TTL 24시간 이내)
    │       │
    │       ├── YES ──▶ APCu 메모리에서 즉시 반환 (API 호출 없음) ✅
    │       │
    │       └── NO
    │               │
    │               ▼
    │           Secrets Manager API 호출 (IMDSv2 + SigV4)
    │               │
    │               ├── 성공 ──▶ APCu 저장 후 반환 ✅
    │               │
    │               └── 실패 (ThrottlingException 등)
    │                       │
    │                       └── error_log 기록 후 Exception throw ❌
```

> APCu는 메모리 기반이므로 서버 재시작 시 캐시가 초기화된다. 재시작 직후 첫 요청에서 API를 호출하여 다시 캐싱된다.
{: .prompt-info }

---

## 8. 운영 적용 절차

> **수정 파일**: `lib.awsSecretsHelper.php` only (`db.php` 수정 불필요)<br>
> **서버 재시작**: APCu 기설치 시 불필요 / 미설치 시 Apache 재시작 필요<br>
> **대상 서버**: EC2 3대 모두 동일하게 적용
{: .prompt-tip }

### 사전 확인 (3대 모두)

```bash
# APCu 모듈 설치 여부 확인
php -m | grep apcu
php -r "echo apcu_enabled() ? 'APCu enabled' : 'APCu not enabled';"

# Apache 실행 계정 확인
ps aux | grep httpd | head -3

# PHP 버전 확인
php -v
```

### APCu 미설치 시 설치

```bash
# Rocky Linux / Amazon Linux
sudo yum install -y php-pecl-apcu

# 설치 확인
php -m | grep apcu

# Apache 재시작 (APCu 활성화)
sudo systemctl restart httpd
```

### 배포 절차 (3대 순차 적용)

```bash
# 1. 기존 파일 백업
cp /path/to/lib.awsSecretsHelper.php \
   /path/to/lib.awsSecretsHelper.php.bak.$(date +%Y%m%d)

# 2. 수정 파일 배포 후 APCu 캐시 동작 확인
php -r "
  apcu_store('test', 'ok', 60);
  echo apcu_fetch('test') === 'ok' ? 'APCu 정상' : 'APCu 오류';
"

# 3. 에러 로그 확인
tail -f /var/log/httpd/error_log | grep AwsSecretsHelper
```

### 롤백 절차

```bash
# 이슈 발생 시 즉시 롤백 (3대 모두)
cp /path/to/lib.awsSecretsHelper.php.bak.$(date +%Y%m%d) \
   /path/to/lib.awsSecretsHelper.php
```

---

## 9. 재발 방지

### 모니터링

**CloudWatch 알람 설정 권장**

| 메트릭 | 조건 | 알람 |
|---|---|---|
| `ThrottlingException` 발생 수 | > 0 / 5분 | SNS 알림 |
| Secrets Manager API 호출 TPS | > 400 TPS | SNS 알림 |

### 중장기 개선 방안

| 우선순위 | 방안 | 내용 |
|---|---|---|
| 높음 | **APCu 캐시 적용** | 이번 정식 조치 — 서버 내 메모리 캐싱으로 API 호출 대폭 감소 |
| 높음 | AWS TPS 한도 증가 | GetSecretValue 500 → 1,500 TPS 상향 (AWS Support) |
| 중간 | IMDSv2 전환 | 보안 강화 (이번 조치에 포함) |
| 중간 | CloudWatch 알람 | ThrottlingException 발생 시 즉시 SNS 알림 |
| 낮음 | PHP-FPM 전환 검토 | Worker Pool APCu 공유 안정성 향상, Nginx 전환 용이 |
| 낮음 | Composer 도입 | AWS SDK for PHP 표준 적용 |

### APCu vs 파일 캐시 비교

| 항목 | APCu (이번 조치) | 파일 캐시 (/tmp) |
|---|---|---|
| 저장 위치 | 서버 메모리 | 디스크 |
| 속도 | 매우 빠름 (메모리) | 상대적으로 느림 (I/O) |
| 서버 재시작 시 | 캐시 초기화 | 캐시 유지 |
| mod_php 환경 | ✅ worker 간 공유 가능 | ✅ 파일 시스템 공유 |
| 3대 고정 구성 적합성 | ✅ 각 서버 독립 캐싱으로 충분 | ✅ 가능하나 불필요한 I/O 발생 |
| 모듈 설치 | APCu 설치 필요 | 불필요 |

> 현재 환경(EC2 3대 고정, 동일 credential)에서는 **APCu가 더 적합**하다.
{: .prompt-tip }

---

## 10. 타임라인

| 시각 | 이벤트 |
|---|---|
| 2026-04-24 | ThrottlingException 이슈 최초 보고 |
| 2026-04-24 | RCA 분석 시작 — 코드 구조 확인 |
| 2026-04-24 | 근본 원인 확인 — 캐싱 부재 및 매 요청 API 호출 |
| 2026-04-24 | 정식 조치 코드 작성 완료 |
| 2026-04-24 | RCA 문서 작성 완료 |
| TBD | Dev 환경 검증 |
| TBD | Prod 배포 (APCu 기설치 여부에 따라 재시작 여부 결정) |
| TBD | AWS Support TPS 한도 증가 요청 |

---

## 참고

- [AWS Secrets Manager 할당량](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_limits.html)
- [AWS Secrets Manager Caching](https://docs.aws.amazon.com/secretsmanager/latest/userguide/use_caching_client.html)
- [IMDSv2 전환 가이드](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- [PHP APCu 공식 문서](https://www.php.net/manual/en/book.apcu.php)
- [PHP-FPM 공식 문서](https://www.php.net/manual/en/install.fpm.php)
- [AWS Signature Version 4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)
