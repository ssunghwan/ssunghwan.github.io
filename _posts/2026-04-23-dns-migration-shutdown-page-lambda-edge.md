---
title: "서버가 죽어도 공지는 떠야한다 - CloudFront+S3+Lambda로 만든 무중단 점검 페이지"
date: 2026-04-23 00:00:00 +0900
categories: [Cloud & Infrastructure, Architecture]
tags: [aws, cloudfront, lambda-edge, s3, dns, route53, waap]
---

> **작업 목적**: Route53 → 글로벌 DNS 이관 시 서비스 종료 안내 페이지 구성 및 관계자 IP 기반 접근 제어
{: .prompt-info }

---

## 1. 배경 및 문제 상황

### 현재 아키텍처

```
사용자
  → example.co.kr (Route53)
  → WAAP 방화벽
  → CloudFront
  → ALB
  → 운영 서버 (EC2)
```

### DNS 이관 계획

```
Route53 → 글로벌 DNS
이후 Akamai CDN 온보딩 예정
```

### 핵심 문제: DNS 이관 → Akamai 온보딩 순서 의존성

이번 작업의 배경은 **서비스를 의도적으로 종료**하면서 고객들에게 서비스 종료 안내 페이지를 보여주는 것이다. DNS 이관 후 Akamai 온보딩 전까지 **공백 구간**이 반드시 발생하는데, 이 구간에도 고객이 사이트에 접근했을 때 자연스럽게 서비스 종료 안내 페이지로 연결되어야 한다.

```
[현재]
example.co.kr → Route53 → WAAP → CloudFront → ALB → 운영 서버

[DNS 전파 중]
일부 유저: Route53 참조 → 기존대로 동작 (CloudFront → S3 셧다운 페이지)
일부 유저: 글로벌 DNS 참조 → CloudFront → S3 셧다운 페이지

[이관 완료 후 Akamai 온보딩 전 공백 구간]
example.co.kr → 글로벌 DNS → CloudFront → S3 셧다운 페이지
```

### Akamai가 DNS 이관 전 온보딩 불가한 이유

- Akamai 온보딩 시 도메인 소유권 DNS 검증 필요
- 검증을 위해서는 글로벌 DNS에 레코드가 먼저 등록되어야 함
- 즉, **DNS 이관 → Akamai 온보딩** 순서가 강제됨

### 글로벌 팀 팔로업 지연 문제

- 글로벌 DNS 등록은 글로벌 IT 팀이 처리
- 팔로업이 느려 공백 구간이 길어질 가능성 있음
- 공백 구간 동안에도 고객이 접근 시 **서비스 종료 안내 페이지**가 표시되어야 함

### 추가 요구사항

- 글로벌 팀 대기 없이 인프라 담당자가 독립적으로 처리 가능해야 함
- 서비스 종료 안내 페이지를 띄우는 동안 관계자(내부 테스트 담당자)는 실제 사이트에 접근 가능해야 함
- 일반 사용자는 서비스 종료 안내 페이지 표시
- URL이 외부 도메인으로 변경되지 않아야 함 (`example.co.kr` 유지)

---

## 2. 해결 방안 검토

### 검토한 옵션들

**옵션 1: ALB Fixed Response + 서브도메인 redirect**

```
example.co.kr → ALB (302 Redirect) → shutdown.example.co.kr → S3
```

- 문제: `shutdown.example.co.kr` 서브도메인도 글로벌 DNS 등록 필요 → 글로벌 팀 의존

**옵션 2: CloudFront + S3 (선택)**

```
example.co.kr → 글로벌 DNS → CloudFront → S3 (서비스 종료 안내 페이지)
```

- 글로벌 DNS에 `example.co.kr` → CloudFront 도메인만 등록 요청
- CloudFront 오리진을 S3로 교체하면 바로 서비스 종료 안내 페이지 서빙
- 장점: 글로벌 팀 요청 최소화, 인프라 담당자가 독립적으로 처리 가능

### Apex 도메인 CNAME 제약 해결

```
example.co.kr (apex 도메인) → CNAME 불가 (DNS 표준)
→ CloudFront 앞에 두면 HTTPS 처리 + apex 문제 해결
→ 글로벌 DNS에서 A 레코드로 CloudFront IP 지정 또는 별도 처리
```

### DNS 전파 구간 양쪽 트래픽 처리

```
[DNS 전파 중간 구간]
구 유저: Route53 → CloudFront (기존 참조)
신 유저: 글로벌 DNS → CloudFront (새로운 참조)

→ 양쪽 모두 동일한 CloudFront로 도달
→ CF 오리진을 S3로 교체하면 양쪽 모두 서비스 종료 안내 페이지 서빙
→ DNS 전파 구간 별도 처리 불필요
```

### 최종 결정: 기존 CloudFront 활용 + Lambda@Edge IP 분기

```
일반 사용자:
example.co.kr → CloudFront → Lambda@Edge → 서비스 종료 안내 HTML 반환 (URL 유지)

관계자:
example.co.kr → CloudFront → Lambda@Edge → ALB → 운영 서버
```

---

## 3. S3 셧다운 페이지 구성

### 3-1. S3 버킷 생성

```bash
# 버킷 생성
aws s3 mb s3://{bucket-name} --region ap-northeast-2

# 퍼블릭 액세스 차단 해제
aws s3api put-public-access-block \
  --bucket {bucket-name} \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# 버킷 정책 설정 (퍼블릭 읽기)
aws s3api put-bucket-policy \
  --bucket {bucket-name} \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::{bucket-name}/*"
    }]
  }'

# Static Website 호스팅 활성화
aws s3 website s3://{bucket-name} \
  --index-document index.html \
  --error-document index.html
```

### 3-2. 파일 구성

**index.html**:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>서비스 종료 안내</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background: #f5f5f5;
    }
    .container { max-width: 800px; width: 95%; }
    .popup-image { width: 100%; display: block; }
  </style>
</head>
<body>
  <div class="container">
    <!-- S3 절대 경로 사용 필수 -->
    <img
      class="popup-image"
      src="https://{bucket-name}.s3.ap-northeast-2.amazonaws.com/shutdown-notice.jpg"
      alt="서비스 종료 안내"
    />
  </div>
</body>
</html>
```

> img src는 반드시 S3 절대 경로 사용. 상대 경로 사용 시 CloudFront 경유 시 이미지 로드 실패
{: .prompt-warning }

```bash
aws s3 cp index.html s3://{bucket-name}/ --content-type "text/html; charset=utf-8"
aws s3 cp shutdown-notice.jpg s3://{bucket-name}/ --content-type "image/jpeg"
```

---

## 4. CloudFront 구성

### 4-1. S3 오리진 추가

**CloudFront 콘솔 → Origins 탭 → Create origin**:

```
Origin domain: {bucket-name}.s3-website.ap-northeast-2.amazonaws.com
Protocol: HTTP only
```

> S3 REST 엔드포인트(`.s3.amazonaws.com`) 말고 반드시 **S3 Website 엔드포인트(`.s3-website.amazonaws.com`)** 사용.<br>
> REST 엔드포인트는 index.html 자동 서빙 불가
{: .prompt-danger }

### 4-2. 서비스 종료 당일 Default Behavior 변경

```
Behaviors 탭 → Default(*) 편집
원본: ALB → S3 오리진으로 교체
캐시 정책: CachingDisabled
```

### 4-3. CloudFront Invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id {cf-distribution-id} \
  --paths "/*"
```

---

## 5. Lambda@Edge IP 기반 접근 제어

### 5-1. 구조

```
허용 IP → CloudFront → Lambda@Edge → request 그대로 통과 → ALB → 운영 서버
비허용 IP → CloudFront → Lambda@Edge → 서비스 종료 안내 HTML 직접 반환 (URL 변경 없음)
```

### 5-2. Lambda 함수 생성

> Lambda@Edge는 반드시 **us-east-1 리전**에 생성
{: .prompt-danger }

```
함수명: {function-name}
리전: us-east-1
런타임: Node.js 20.x
파일명: index.mjs
```

> 파일명이 `.mjs`이면 반드시 ES Module 방식으로 작성<br>
> - ❌ `exports.handler = async (event) => {` (CommonJS)<br>
> - ✅ `export const handler = async (event) => {` (ES Module)
{: .prompt-warning }

**함수 코드 (index.mjs)**:

```javascript
export const handler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;
    
    const allowedIPs = [
        "xxx.xxx.xxx.xxx",  // 허용 IP 목록
    ];
    
    // WAAP 방화벽이 CF 앞에 있는 경우
    // event.viewer.ip 는 방화벽 IP로 인식되므로
    // x-forwarded-for 헤더에서 실제 클라이언트 IP 추출
    let clientIP = request.clientIp;
    if (headers['x-forwarded-for']) {
        clientIP = headers['x-forwarded-for'][0].value.split(',')[0].trim();
    }
    
    // 허용 IP → ALB로 정상 통과
    if (allowedIPs.includes(clientIP)) {
        return request;
    }
    
    // 비허용 IP → 서비스 종료 안내 페이지 HTML 직접 반환 (URL 변경 없음)
    return {
        status: '200',
        statusDescription: 'OK',
        headers: {
            'content-type': [{
                key: 'Content-Type',
                value: 'text/html; charset=utf-8'
            }],
            'cache-control': [{
                key: 'Cache-Control',
                value: 'no-cache, no-store, must-revalidate'
            }]
        },
        body: '<!DOCTYPE html><html lang="ko">...(서비스 종료 안내 HTML)...</html>'
    };
};
```

### 5-3. IAM 역할 설정

**신뢰 관계 정책**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "edgelambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**인라인 정책 (CloudWatch Logs)**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "logs:CreateLogGroup",
      "Resource": "arn:aws:logs:us-east-1:{account-id}:*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": [
        "arn:aws:logs:us-east-1:{account-id}:log-group:/aws/lambda/{function-name}:*"
      ]
    }
  ]
}
```

### 5-4. 리소스 기반 정책 (버전마다 등록 필요)

> Lambda@Edge 리소스 기반 정책은 **버전마다 별도로 등록** 필요.<br>
> 새 버전 게시 시마다 아래 두 정책 재등록
{: .prompt-danger }

**정책 1 - CloudFront 호출 권한**:

```
문 ID: AllowCFInvoke
Principal: edgelambda.amazonaws.com
Action: lambda:InvokeFunction
Source ARN: arn:aws:cloudfront::{account-id}:distribution/{cf-distribution-id}
```

**정책 2 - Lambda 엣지 복제 권한**:

```
문 ID: AllowReplicator
Principal: replicator.lambda.amazonaws.com
Action: lambda:GetFunction
Source ARN: (없음)
```

> Lambda@Edge 동작 방식<br>
> 1. `replicator.lambda.amazonaws.com`이 각 엣지 리전에 함수 복제<br>
> 2. `edgelambda.amazonaws.com`이 복제된 함수 실행<br>
> 두 권한 모두 없으면 503 에러 발생
{: .prompt-info }

### 5-5. CloudFront Behavior에 Lambda@Edge 연결

```
Behaviors → Default(*) 편집
함수 연결:
  뷰어 요청(Viewer Request): Lambda@Edge → {function-arn}:{version}
  뷰어 응답(Viewer Response): 연결 없음 ← 반드시 비워둘 것
```

> 뷰어 응답에 연결 시 `response.statusCode is missing` 503 에러 발생
{: .prompt-danger }

---

## 6. 서비스 종료 당일 작업 순서

```
1. CF Default Behavior 원본 → ALB에서 S3로 교체
2. CF Default Behavior에 Lambda@Edge 연결 (허용 IP 분기)
3. CF Invalidation 실행
4. 글로벌 DNS에 example.co.kr → CloudFront 도메인 등록 확인
5. Route53 비활성화
6. 서비스 종료 안내 페이지 정상 표시 확인 (비허용 IP)
7. 허용 IP에서 ALB 정상 접근 확인
```

---

## 7. Akamai 온보딩 완료 후 원복 순서

```
1. CF Default Behavior Lambda@Edge 연결 제거
2. CF Default Behavior 원본 → S3에서 ALB로 교체
3. CF Invalidation 실행
4. 글로벌 DNS 변경 요청 (CF → Akamai Edge IP/CNAME)
5. 정상 서비스 확인
```

---

## 8. 트러블슈팅 기록

### 8-1. S3 이미지 엑박 (앱 인앱 브라우저)

| 항목 | 내용 |
|---|---|
| 증상 | 모바일 앱 인앱 브라우저에서 이미지 엑박 |
| 원인 | S3 Website 엔드포인트는 HTTP only → 앱 HTTPS 웹뷰에서 Mixed Content 차단 |
| 해결 | CloudFront 앞에 두고 img src를 S3 절대 경로(`https://`)로 설정 |

### 8-2. S3 이미지 403

| 항목 | 내용 |
|---|---|
| 증상 | S3 직접 접근 시 이미지 403 |
| 원인 | 버킷 정책 미설정으로 퍼블릭 읽기 불가 |
| 해결 | 퍼블릭 액세스 차단 비활성화 + 버킷 정책 추가 |

### 8-3. CF Function에서 허용 IP 미인식

| 항목 | 내용 |
|---|---|
| 증상 | 허용 IP임에도 서비스 종료 안내 페이지로 redirect |
| 원인 | WAAP 방화벽이 CF 앞에 위치하여 `event.viewer.ip`가 방화벽 IP로 인식됨 |
| 해결 | `x-forwarded-for` 헤더의 첫 번째 IP 사용 |

```
실제 흐름: 사용자 → WAAP 방화벽 → CloudFront
CF Function에서 보이는 IP: 방화벽 IP (실제 사용자 IP 아님)
```

### 8-4. CF Function 503 (뷰어 응답 연결)

| 항목 | 내용 |
|---|---|
| 증상 | `response.statusCode is missing` 503 |
| 원인 | 뷰어 응답(Viewer Response)에도 함수 연결됨 |
| 해결 | 뷰어 요청(Viewer Request)에만 연결 |

### 8-5. Lambda@Edge 503 (권한 미설정)

| 항목 | 내용 |
|---|---|
| 증상 | `The Lambda function doesn't have the required permissions` |
| 원인 | 버전별 리소스 기반 정책 미설정 |
| 해결 | `AllowCFInvoke` + `AllowReplicator` 정책 해당 버전에 추가 |

### 8-6. Lambda@Edge 503 (ES Module 오류)

| 항목 | 내용 |
|---|---|
| 증상 | `exports is not defined in ES module scope` |
| 원인 | 파일명 `index.mjs`에 CommonJS 방식(`exports.handler`) 사용 |
| 해결 | ES Module 방식(`export const handler`)으로 변경 후 재배포 |

### 8-7. 새 버전 게시 시 권한 초기화

| 항목 | 내용 |
|---|---|
| 증상 | 코드 수정 후 새 버전 게시 시 다시 503 |
| 원인 | 리소스 기반 정책은 버전 단위로 관리됨. 새 버전에는 정책 없음 |
| 해결 | 새 버전 게시 후 반드시 해당 버전에 리소스 기반 정책 재등록 |

---

## 9. 핵심 교훈

> 1. **Lambda@Edge는 us-east-1에서만 생성 가능**<br>
> 2. **Lambda@Edge 리소스 기반 정책은 버전마다 별도 등록** (가장 자주 놓치는 부분)<br>
> 3. **WAAP 방화벽이 앞에 있으면 `viewer.ip`가 실제 IP가 아님** → `x-forwarded-for` 헤더 사용<br>
> 4. **S3 Website 엔드포인트는 HTTP only** → CloudFront 앞에 두어 HTTPS 처리<br>
> 5. **뷰어 응답에 Lambda@Edge 연결 금지** → `statusCode` 미포함으로 503 발생<br>
> 6. **파일 확장자 `.mjs` 사용 시 ES Module 방식으로 작성**<br>
> 7. **img src는 S3 절대 경로 사용** → 상대 경로 사용 시 CF 경유 시 이미지 로드 실패<br>
> 8. **CF Invalidation은 Lambda@Edge 권한 문제와 무관** (캐시 문제가 아님)
{: .prompt-tip }
