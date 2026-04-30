---
title: "AWS CloudTrail Based ISMS-P Real-Time Security Monitoring Setup Guide"
date: 2026-04-30 00:00:00 +0900
categories: [Compliance & Vulnerability, Application & Network]
tags: [aws, cloudtrail, cloudwatch, lambda, glue, athena, isms-p, kms, sns, eventbridge]
---

> **환경**: AWS ap-northeast-2 / Production<br>
> **목적**: ISMS-P 인증 대응을 위한 CloudTrail 기반 실시간 보안 모니터링 및 이상 이벤트 알람 체계 구축
{: .prompt-info }

---

## 1. 개요

### 배경

이커머스 플랫폼 운영 환경에서 ISMS-P 인증 대응을 위해 AWS CloudTrail 로그를 실시간으로 모니터링하고, 이상 이벤트 발생 시 즉시 알람을 수신할 수 있는 체계가 필요했다.

기존에는 CloudTrail 로그가 S3에만 저장되어 있어 실시간 탐지가 불가능했으며, 사후 분석 시에도 로그를 수동으로 확인해야 하는 불편함이 있었다.

### 요구사항

- **실시간 알람**: 보안 이벤트 발생 시 즉시 이메일 알람 수신
- **전 리전 커버리지**: IAM(us-east-1), 콘솔 로그인(랜덤 리전) 등 모든 리전 이벤트 탐지
- **비허가 IP 탐지**: 허가된 사내 IP 외 접속 시 즉시 알람
- **사후 분석**: ISMS-P 심사 대비 SQL 기반 감사 로그 조회
- **ISMS-P 대응**: 관련 통제 항목 자동화

### ISMS-P 대응 항목

| ISMS-P 항목 | 구현 내용 |
|---|---|
| 2.9.4 로그 및 접속기록 관리 | CloudWatch Logs 연동 + 365일 보존 + KMS 암호화 |
| 2.9.1 변경관리 | IAM/SG 변경 Lambda 탐지 + SNS 알람 |
| 2.11.1 사고 예방 및 대응 | Lambda + SNS 실시간 알람 |
| 2.11.3 이상행위 분석 | Athena 쿼리 기반 주기적 분석 |
| 2.5.1 사용자 계정 관리 | Root/MFA 미사용/비허가 IP 탐지 |

---

## 2. 사용 서비스 및 역할

| 서비스 | 역할 |
|---|---|
| **AWS CloudTrail** | AWS 계정 내 모든 API 호출 이력을 기록하는 감사 로그 서비스 |
| **Amazon CloudWatch Logs** | 로그를 실시간으로 수집·저장하는 모니터링 서비스 |
| **CloudWatch Logs Subscription Filter** | 로그 그룹의 로그를 Lambda로 실시간 스트리밍하는 필터 |
| **CloudWatch Metric Filter** | 로그에서 패턴을 탐지하여 수치(지표)로 변환하는 규칙 |
| **CloudWatch Alarm** | 지표가 임계값 초과 시 경보를 발송하는 서비스 (대시보드용) |
| **Amazon SNS** | 이메일·SMS 등 다양한 채널로 알림을 전달하는 서비스 |
| **AWS Lambda** | 이벤트 분석 및 비허가 IP 검증 후 SNS 알람을 발송하는 함수 |
| **AWS KMS** | 데이터 암호화 키를 중앙에서 안전하게 관리하는 서비스 |
| **AWS Glue** | S3 데이터의 스키마를 자동 감지하고 카탈로그화하는 ETL 서비스 |
| **Amazon Athena** | S3에 저장된 로그를 SQL로 분석하는 서버리스 쿼리 서비스 |

---

## 3. 최종 아키텍처

```
CloudTrail (전 리전 API 호출 기록)
    │
    ├── S3 (로그 저장 + KMS 암호화)
    │       │
    │       └── Glue Crawler (매일 01:00) ──→ Glue Data Catalog
    │                                               │
    │                                          Athena SQL 분석
    │                                       (ISMS-P 감사 증적)
    │
    └── CloudWatch Logs (실시간 스트리밍 + KMS 암호화)
            │
            ├── Metric Filter 7개 ──→ CloudWatch Alarm ──→ 대시보드
            │
            └── Subscription Filter
                        │
                        ▼
               Lambda (이벤트 분석)
                        │
                        ├── Root 계정 사용 탐지
                        ├── IAM Policy 변경 탐지
                        ├── 콘솔 로그인 실패 탐지
                        ├── MFA 미사용 로그인 탐지
                        ├── Security Group 변경 탐지
                        ├── CloudTrail 변경 탐지
                        ├── RDS 인스턴스 변경 탐지
                        └── 비허가 IP 로그인 탐지
                                │
                                ▼
                           SNS Topic
                                │
                           이메일 알람
```

---

## 4. 구성 작업

### 4-1. CloudWatch Log Group 생성

```bash
# Log Group 생성
aws logs create-log-group \
  --log-group-name /aws/cloudtrail/<your-trail-name> \
  --region ap-northeast-2

# 보존 기간 365일 설정 (ISMS-P 2.9.4 요건)
aws logs put-retention-policy \
  --log-group-name /aws/cloudtrail/<your-trail-name> \
  --retention-in-days 365 \
  --region ap-northeast-2
```

### 4-2. CloudTrail → CloudWatch Logs 전송용 IAM Role 생성

CloudTrail이 CloudWatch Logs로 로그를 전송하려면 전용 IAM Role이 필요하다.

> **인라인 정책을 사용하는 이유**<br>
> AWS 관리형 정책 중 이 용도에 맞는 정책이 존재하지 않는다.<br>
> `CloudWatchLogsFullAccess` 같은 광범위한 정책 사용은 ISMS-P 최소 권한 원칙 위반으로 심사 시 지적받을 수 있다.
{: .prompt-warning }

**Trust Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudtrail.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**인라인 Permission Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
    "Resource": "arn:aws:logs:ap-northeast-2:<account-id>:log-group:/aws/cloudtrail/<your-trail-name>:*"
  }]
}
```

### 4-3. KMS 키 정책 업데이트

CloudWatch Logs와 Glue Crawler가 KMS 암호화된 S3 버킷에 접근할 수 있도록 키 정책에 Statement를 추가한다.

**CloudWatch Logs 권한**:

```json
{
  "Sid": "Allow CloudWatch Logs All Regions",
  "Effect": "Allow",
  "Principal": { "Service": "logs.amazonaws.com" },
  "Action": ["kms:Encrypt*", "kms:Decrypt*", "kms:ReEncrypt*", "kms:GenerateDataKey*", "kms:Describe*"],
  "Resource": "*",
  "Condition": {
    "ArnLike": {
      "kms:EncryptionContext:aws:logs:arn": "arn:aws:logs:*:<account-id>:log-group:/aws/cloudtrail/<your-trail-name>"
    }
  }
}
```

**Glue Crawler 권한**:

```json
{
  "Sid": "Allow Glue Crawler",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::<account-id>:role/<glue-crawler-role>" },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey*", "kms:DescribeKey"],
  "Resource": "*"
}
```

### 4-4. CloudWatch Log Group KMS 연결

콘솔에서 직접 설정이 불가능하므로 CLI를 사용한다.

```bash
aws logs associate-kms-key \
  --log-group-name /aws/cloudtrail/<your-trail-name> \
  --kms-key-id arn:aws:kms:ap-northeast-2:<account-id>:key/<kms-key-id> \
  --region ap-northeast-2
```

### 4-5. Metric Filter 7개 생성 (대시보드용)

> **CloudWatch Metric Filter란?**<br>
> CloudWatch Logs로 수집되는 로그에서 특정 패턴을 실시간으로 감지하여 수치(지표)로 변환하는 규칙이다.<br>
> 실제 알람 발송은 Lambda + SNS가 담당하며, Metric Filter는 CloudWatch 대시보드 시각화에 활용한다.
{: .prompt-info }

| 필터명 | 패턴 | 지표명 |
|---|---|---|
| `RootAccountUsage` | `{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }` | `RootAccountUsage` |
| `IAMPolicyChanges` | `{ ($.eventName = DeleteGroupPolicy) \|\| ($.eventName = PutGroupPolicy) \|\| ... }` | `IAMPolicyChanges` |
| `ConsoleLoginFailed` | `{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }` | `ConsoleLoginFailed` |
| `MFANotUsed` | `{ ($.eventName = ConsoleLogin) && ($.additionalEventData.MFAUsed != "Yes") && ($.userIdentity.type = "IAMUser") }` | `MFANotUsed` |
| `SecurityGroupChanges` | `{ ($.eventName = AuthorizeSecurityGroupIngress) \|\| ... }` | `SecurityGroupChanges` |
| `CloudTrailChanges` | `{ ($.eventName = CreateTrail) \|\| ($.eventName = UpdateTrail) \|\| ... }` | `CloudTrailChanges` |
| `RDSInstanceChanges` | `{ ($.eventSource = "rds.amazonaws.com") && (($.eventName = CreateDBInstance) \|\| ...) }` | `RDSInstanceChanges` |

**공통 설정**: 네임스페이스 `ISMS-CloudTrail` / 지표 값 `1` / 기본값 `0`

### 4-6. SNS Topic 생성

```
SNS → 주제 → 주제 생성
  유형: 표준
  이름: ISMS-CloudTrail-Alert
```

> 구독 생성 후 수신 이메일 주소로 발송된 **Confirm subscription** 링크를 반드시 클릭해야 한다.
{: .prompt-warning }

### 4-7. Lambda 함수 생성

> **CloudWatch Logs Subscription Filter + Lambda 방식을 선택한 이유**<br>
> EventBridge를 통한 직접 탐지를 먼저 시도했으나, IAM 이벤트는 `us-east-1`, 콘솔 로그인은 로그인 시 선택한 리전에 기록되는 등 리전 분산 문제로 단일 리전 EventBridge Rule만으로는 전 리전 이벤트를 커버할 수 없었다.<br>
> CloudTrail → CloudWatch Logs는 **전 리전 이벤트를 ap-northeast-2 단일 Log Group으로 수집**하므로, Subscription Filter + Lambda 방식이 유일하게 전 리전을 커버할 수 있는 구조다.
{: .prompt-tip }

**함수명**: `ISMSP-UnauthorizedIPDetector` / **런타임**: Python 3.12

```python
import json
import boto3
import base64
import gzip
import os

sns = boto3.client('sns', region_name='ap-northeast-2')

ALLOWED_IPS   = [ip.strip() for ip in os.environ.get('ALLOWED_IPS', '').split(',')]
SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN', '')

SECURITY_EVENTS = [
    # IAM
    'DeleteGroupPolicy', 'PutGroupPolicy', 'PutRolePolicy', 'PutUserPolicy',
    'CreatePolicy', 'DeletePolicy', 'AttachRolePolicy', 'DetachRolePolicy',
    # Security Group
    'AuthorizeSecurityGroupIngress', 'RevokeSecurityGroupIngress',
    'CreateSecurityGroup', 'DeleteSecurityGroup',
    # CloudTrail
    'CreateTrail', 'UpdateTrail', 'DeleteTrail', 'StopLogging',
    # RDS
    'CreateDBInstance', 'DeleteDBInstance', 'ModifyDBInstance',
]

def send_alert(subject, message):
    sns.publish(TopicArn=SNS_TOPIC_ARN, Subject=subject, Message=message)

def format_message(title, event_name, event_source, event_time, aws_region, username, user_type, source_ip, extra=''):
    return f"""
⚠️ [ISMS-P 보안 알람] {title}

━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 이벤트 정보
━━━━━━━━━━━━━━━━━━━━━━━━━━
- 이벤트명   : {event_name}
- 이벤트 소스: {event_source}
- 발생 시간  : {event_time}
- 발생 리전  : {aws_region}

━━━━━━━━━━━━━━━━━━━━━━━━━━
👤 사용자 정보
━━━━━━━━━━━━━━━━━━━━━━━━━━
- 사용자명   : {username}
- 사용자 유형: {user_type}
- 접속 IP   : {source_ip}
{extra}
━━━━━━━━━━━━━━━━━━━━━━━━━━
즉시 확인이 필요합니다.
AWS 콘솔 → CloudTrail → 이벤트 기록에서 상세 내용을 확인하세요.
"""

def lambda_handler(event, context):
    log_data     = event.get('awslogs', {}).get('data', '')
    compressed   = base64.b64decode(log_data)
    decompressed = gzip.decompress(compressed)
    log_events   = json.loads(decompressed)

    for log_event in log_events.get('logEvents', []):
        try:
            ct_event = json.loads(log_event['message'])
        except Exception:
            continue

        event_name    = ct_event.get('eventName', '')
        event_source  = ct_event.get('eventSource', '')
        event_time    = ct_event.get('eventTime', '')
        aws_region    = ct_event.get('awsRegion', '')
        source_ip     = ct_event.get('sourceIPAddress', '')
        error_message = ct_event.get('errorMessage', '')

        user_identity = ct_event.get('userIdentity', {})
        username  = user_identity.get('userName', user_identity.get('type', 'Unknown'))
        user_type = user_identity.get('type', 'Unknown')
        mfa_used  = ct_event.get('additionalEventData', {}).get('MFAUsed', '')

        # ① Root 계정 사용 탐지
        if user_type == 'Root' and ct_event.get('eventType') != 'AwsServiceEvent':
            send_alert('[ISMS-P] Root 계정 사용 탐지',
                format_message('Root 계정 사용 탐지', event_name, event_source,
                    event_time, aws_region, username, user_type, source_ip))

        # ② 보안 이벤트 탐지 (IAM/SG/CloudTrail/RDS)
        if event_name in SECURITY_EVENTS:
            send_alert(f'[ISMS-P] {event_name} 탐지',
                format_message(f'{event_name} 탐지', event_name, event_source,
                    event_time, aws_region, username, user_type, source_ip))

        # ③ 콘솔 로그인 실패
        if event_name == 'ConsoleLogin' and error_message == 'Failed authentication':
            send_alert('[ISMS-P] 콘솔 로그인 실패 탐지',
                format_message('콘솔 로그인 실패 탐지', event_name, event_source,
                    event_time, aws_region, username, user_type, source_ip))

        # ④ MFA 미사용 로그인 (로그인 성공한 경우만)
        if event_name == 'ConsoleLogin' and mfa_used == 'No' and user_type == 'IAMUser' and not error_message:
            send_alert('[ISMS-P] MFA 미사용 로그인 탐지',
                format_message('MFA 미사용 로그인 탐지', event_name, event_source,
                    event_time, aws_region, username, user_type, source_ip))

        # ⑤ 비허가 IP 로그인 성공
        if event_name == 'ConsoleLogin' and not error_message:
            if source_ip not in ALLOWED_IPS:
                extra = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━
🚨 허가된 IP 목록
━━━━━━━━━━━━━━━━━━━━━━━━━━
{chr(10).join(f'• {ip}' for ip in ALLOWED_IPS)}
"""
                send_alert('[ISMS-P] 비허가 IP 콘솔 로그인 탐지',
                    format_message('비허가 IP 콘솔 로그인 탐지', event_name, event_source,
                        event_time, aws_region, username, user_type, source_ip, extra=extra))

    return {'statusCode': 200}
```

**환경변수**:

| 키 | 값 |
|---|---|
| `ALLOWED_IPS` | `허가된IP1,허가된IP2,...` |
| `SNS_TOPIC_ARN` | `arn:aws:sns:ap-northeast-2:<account-id>:ISMS-CloudTrail-Alert` |

**Lambda IAM 인라인 정책**:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:ap-northeast-2:<account-id>:ISMS-CloudTrail-Alert"
  }]
}
```

### 4-8. CloudWatch Logs Subscription Filter 생성

```
CloudWatch → 로그 그룹 → /aws/cloudtrail/<your-trail-name>
→ 구독 필터 탭 → Lambda 구독 필터 생성
```

| 항목 | 값 |
|---|---|
| Lambda 함수 | `ISMSP-UnauthorizedIPDetector` |
| 필터 이름 | `ISMSP-SecurityFilter` |
| 필터 패턴 | 비워두기 (전체 로그 전달) |
| 로그 형식 | 기타 |

### 4-9. Glue + Athena 분석 환경 구성

> **Athena는 실시간 도구가 아니다**<br>
> Athena는 S3에 저장된 로그를 SQL로 분석하는 도구로, 실시간 알람과는 역할이 다르다.<br>
> ISMS-P 심사 시 "지난 달 특정 IP 접근 이력", "IAM 권한 변경 전체 이력" 등을 SQL로 즉시 추출하는 감사 증적 용도로 활용한다.
{: .prompt-info }

**Glue 크롤러 설정**:

| 항목 | 값 |
|---|---|
| 크롤러 이름 | `<prefix>-monitoring-crawler` |
| S3 경로 | `s3://<cloudtrail-bucket>/AWSLogs/<account-id>/CloudTrail/ap-northeast-2/` |
| Recrawl behavior | 최초: `Crawl all sub-folders` → 이후: `Crawl new sub-folders only` |
| 스케줄 | 매일 01:00 AM |
| 출력 데이터베이스 | `cloudtrail_analysis` |

**Athena 테이블 생성**:

Glue 크롤러가 자동 생성한 테이블은 `requestparameters` 등의 컬럼을 복잡한 STRUCT로 감지하여 `HIVE_INVALID_METADATA` 오류가 발생할 수 있다. 모든 컬럼을 STRING으로 지정하여 수동 생성한다.

```sql
CREATE EXTERNAL TABLE cloudtrail_analysis.cloudtrail_logs (
    eventversion STRING, useridentity STRING, eventtime STRING,
    eventsource STRING, eventname STRING, awsregion STRING,
    sourceipaddress STRING, useragent STRING, errorcode STRING,
    errormessage STRING, requestparameters STRING, responseelements STRING,
    additionaleventdata STRING, requestid STRING, eventid STRING,
    readonly STRING, resources STRING, eventtype STRING, apiversion STRING,
    recipientaccountid STRING, serviceeventdetails STRING,
    sharedeventid STRING, vpcendpointid STRING
)
PARTITIONED BY (partition_0 STRING, partition_1 STRING, partition_2 STRING)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://<cloudtrail-bucket>/AWSLogs/<account-id>/CloudTrail/ap-northeast-2/';
```

**ISMS-P 감사 분석 쿼리**:

```sql
-- 콘솔 로그인 이력 조회
SELECT eventtime, sourceipaddress, errorcode, errormessage
FROM cloudtrail_analysis.cloudtrail_logs
WHERE partition_0='2026' AND partition_1='04' AND partition_2='30'
  AND eventname = 'ConsoleLogin'
ORDER BY eventtime DESC;

-- IAM 권한 변경 이력 조회
SELECT eventtime, eventname, eventsource, sourceipaddress
FROM cloudtrail_analysis.cloudtrail_logs
WHERE partition_0='2026' AND partition_1='04'
  AND eventname IN ('CreatePolicy','DeletePolicy','AttachRolePolicy','DetachRolePolicy')
ORDER BY eventtime DESC;

-- 특정 IP 행위 추적
SELECT eventtime, eventname, eventsource
FROM cloudtrail_analysis.cloudtrail_logs
WHERE partition_0='2026' AND partition_1='04'
  AND sourceipaddress = '대상IP'
ORDER BY eventtime DESC;
```

### 4-10. CloudWatch 대시보드 구성

| 위젯 | 유형 | 내용 |
|---|---|---|
| ISMS-P 보안 경보 현황 | 경보 상태 | 7개 Alarm 전체 상태 |
| 보안 이벤트 발생 추이 | 행 차트 | `ISMS-CloudTrail` 네임스페이스 7개 지표 |
| 이벤트 유형별 발생 건수 | 막대 차트 | Logs Insights 쿼리 |
| 실시간 CloudTrail 이벤트 로그 | 테이블 | 최근 이벤트 스트림 |

---

## 5. 트러블슈팅

### 5-1. EventBridge로는 전 리전 CloudTrail 이벤트를 커버할 수 없다

**문제 상황**

처음에는 EventBridge Rule → SNS 직접 연결 방식을 시도했다. `ap-northeast-2`에 Rule을 생성하고 Security Group 변경 이벤트를 테스트했을 때는 정상 동작했다. 그러나 IAM 정책 생성(`CreatePolicy`) 이벤트는 알람이 오지 않았고, 콘솔 로그인 실패도 탐지되지 않았다.

**원인 분석**

CloudTrail 이벤트가 EventBridge로 전달되는 리전은 **이벤트가 실제로 발생한 리전**이다.

```
IAM 이벤트       → us-east-1 EventBridge로 전달 (글로벌 서비스)
ConsoleLogin    → 로그인 시 선택한 리전 EventBridge로 전달 (랜덤)
EC2/RDS/SG      → 리소스가 존재하는 리전 EventBridge로 전달
```

`ap-northeast-2`에만 Rule을 생성했으므로 IAM 이벤트(`us-east-1`)와 콘솔 로그인(랜덤 리전)은 탐지가 불가능한 구조였다.

**해결**

CloudTrail → CloudWatch Logs 연동이 `ap-northeast-2` Log Group 하나로 **전 리전 이벤트를 중앙 집중**하고 있다는 점을 활용했다. Subscription Filter로 Lambda를 트리거하면 리전에 관계없이 모든 이벤트를 처리할 수 있다.

```
[기존 시도 — 실패]
CloudTrail (전 리전) → EventBridge (ap-northeast-2 only) → SNS
                       ↑ IAM(us-east-1), ConsoleLogin(랜덤) 누락

[최종 구성 — 성공]
CloudTrail (전 리전) → CloudWatch Logs (ap-northeast-2, 전 리전 수집)
                               ↓ Subscription Filter
                            Lambda → SNS
```

---

### 5-2. Glue 크롤러가 테이블을 생성하지 못하는 두 가지 함정

**문제 1 — KMS 복호화 권한 부재**

CloudTrail S3 버킷이 KMS로 암호화되어 있어서 크롤러가 파일을 읽지 못하고 `Table changes: -`로 완료되었다.

```
ERROR: glue.amazonaws.com is not authorized to perform: kms:Decrypt on resource
```

KMS 키 정책에 Glue Crawler IAM Role을 Principal로 추가하지 않으면 파일 내용을 읽을 수 없어 스키마 감지가 불가능하다.

**조치**: KMS 키 정책에 `Allow Glue Crawler` Statement 추가

```json
{
  "Sid": "Allow Glue Crawler",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<account-id>:role/<glue-crawler-role>"
  },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey*", "kms:DescribeKey"],
  "Resource": "*"
}
```

**문제 2 — Recrawl behavior 설정**

KMS 문제를 해결했음에도 크롤러 실행 후 `Table changes: -`가 반복되었다. 크롤러를 여러 번 실행했기 때문에 `Crawl new sub-folders only`가 적용된 상태에서 이미 스캔한 경로를 스킵하고 있었던 것이다.

**조치**: 최초 테이블 생성 시에는 반드시 `Crawl all sub-folders`로 설정 후 실행. 테이블 생성 확인 후 `Crawl new sub-folders only`로 변경.

| Recrawl 설정 | 동작 | 사용 시점 |
|---|---|---|
| `Crawl all sub-folders` | 전체 경로 스캔 | 최초 테이블 생성 시 |
| `Crawl new sub-folders only` | 새 폴더만 스캔 | 이후 운영 (비용 절감) |

---

### 5-3. Athena 테이블 스키마 문제 — STRUCT vs STRING

**문제 상황**

Glue 크롤러가 테이블을 자동 생성했지만, Athena에서 쿼리 시 아래 오류가 발생했다.

```
HIVE_INVALID_METADATA: Glue table column 'requestparameters' has invalid data type:
struct<keyId:string,limit:string,encryptionAlgorithm:string,...>
```

`requestparameters`, `responseelements` 컬럼이 수천 개의 중첩 필드를 가진 복잡한 STRUCT로 자동 감지되어 Athena 엔진이 파싱하지 못하는 상황이었다.

**원인**

CloudTrail 로그는 이벤트 종류마다 `requestparameters` 구조가 완전히 다르다. Glue 크롤러가 이를 통합된 단일 STRUCT로 표현하려다 보니 수백 개의 필드가 생기고 Athena가 이를 처리하지 못했다.

**조치**

모든 문제 컬럼을 STRING으로 지정하여 테이블을 수동 생성한다. CloudTrail Serde(`JsonSerDe`)가 JSON 파싱을 처리하므로 STRING으로 저장해도 `json_extract`로 필요한 값을 추출할 수 있다.

```sql
-- 문제가 되는 컬럼들을 모두 STRING으로 지정
CREATE EXTERNAL TABLE cloudtrail_analysis.cloudtrail_logs (
    ...
    requestparameters STRING,  -- STRUCT 대신 STRING
    responseelements  STRING,  -- STRUCT 대신 STRING
    ...
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
...
```

---

## 6. 역할 구분 정리

| 구성 | 역할 | 특징 |
|---|---|---|
| **CloudWatch Logs + Lambda + SNS** | 실시간 보안 알람 | 이벤트 발생 후 1~2분 내 이메일 알람 |
| **CloudWatch Metric Filter + Alarm** | 대시보드 시각화 | 보안 이벤트 추이 모니터링 |
| **Glue + Athena** | 사후 분석 / 감사 증적 | 일별 배치, SQL로 상세 분석 |

---

## 7. 최종 구성 체크리스트

> **CloudTrail**<br>
> - S3 저장 + KMS 암호화 ✅<br>
> - CloudWatch Logs 연동 + KMS 암호화 ✅<br>
> - Log Group 보존 기간 365일 ✅
>
> **CloudWatch**<br>
> - Metric Filter 7개 (대시보드용) ✅<br>
> - CloudWatch Alarm 7개 (대시보드용) ✅<br>
> - 대시보드 ✅
>
> **알람 체계**<br>
> - SNS Topic + 이메일 구독 확인 ✅<br>
> - Lambda 함수 (이벤트 분석 + IP 검증) ✅<br>
> - Subscription Filter → Lambda 연결 ✅
>
> **분석 환경**<br>
> - Glue Crawler 일일 스케줄 (매일 01:00) ✅<br>
> - Athena 테이블 + 파티션 등록 ✅
{: .prompt-tip }

---

## 참고

- [AWS CloudTrail Security Best Practices](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html)
- [CloudWatch Logs Subscription Filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html)
- [Amazon Athena CloudTrail Logs](https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html)
- [AWS Glue Crawler Best Practices](https://docs.aws.amazon.com/glue/latest/dg/crawler-configuration.html)
