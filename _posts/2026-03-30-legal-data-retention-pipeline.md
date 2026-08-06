---
title: "법이 요구하는 데이터를 남기는 법 - ISMSP 대응 주문,결제 보존 파이프라인"
date: 2026-03-30 00:00:00 +0900
categories: [Security & Compliance, Hardening]
tags: [aws, lambda, s3, glacier, eventbridge, sns, secrets-manager, rds, isms-p]
---

> **목적**: 전자상거래법 기반 주문/결제 데이터 5년 법적 보존
{: .prompt-info }

---

## 1. 개요

### 배경

전자상거래법에 따라 주문 및 결제 데이터는 **결제일 기준 5년 보관 후 파기**가 원칙이다. 기존 RDS 스냅샷(최대 35일) 및 AWS Backup(30일 보존, Weekly)으로는 법적 요건을 충족할 수 없어 별도 아카이빙 파이프라인을 구성하였다.

### 아키텍처 요약

```
EventBridge (매일 KST 00:00)
  → Lambda (VPC Private Subnet)
      ├─ Secrets Manager에서 DB credentials 획득
      ├─ RDS MySQL 접속
      ├─ 주문 테이블 전날 결제 완료 데이터 추출
      ├─ 개인정보(PII) 마스킹 처리
      ├─ CSV 변환
      ├─ S3 Compliance 버킷 업로드 (법적 보존)
      ├─ S3 DataLogs 버킷에 실행 로그 저장
      └─ SNS 이메일 알림 발송 (성공/실패/데이터없음)
```

### 백업 정책 전체 구조

| 레이어 | 수단 | 주기 | 보존 | 목적 |
|---|---|---|---|---|
| 운영 복구 | RDS 자동 스냅샷 | Daily | 7일 | 장애 복구 |
| 단기 아카이브 | AWS Backup (Warm) | Weekly | 30일 | 최근 복구 |
| 중기 아카이브 | AWS Backup (Cold) | Weekly | 1년 | 장기 복구 보험 |
| **법적 보존** | **S3 Glacier Deep Archive** | **Daily 추출** | **5년 (Object Lock)** | **법적 의무 이행** |

> AWS Backup Cold Storage: 생성 후 30일 → Glacier 자동 전환, 1년 후 자동 삭제
{: .prompt-info }

---

## 2. 데이터베이스 구조

### 대상 테이블: 주문 테이블

주문, 정기배송, 결제 데이터가 단일 테이블에 통합 저장된다.

| 컬럼 유형 | 설명 |
|---|---|
| PK | auto_increment |
| 주문번호 | 주문 식별자 |
| 회원 ID | 회원 식별자 |
| 주문상태 | tinyint |
| 결제방법 | varchar |
| 주문자명 | **(PII)** |
| 주문자전화번호 | **(PII)** |
| 주문자이메일 | **(PII)** |
| 받는분 | **(PII)** |
| 받는분전화번호 | **(PII)** |
| 받는분주소 | **(PII)** |
| 최종결제금액 | int |
| 주문일시 | datetime |
| **결제일시** | **datetime ← 5년 보관 기준 컬럼** |
| PG 거래번호 | varchar |
| 카드승인번호 | varchar |

> **추출 기준**: 결제일시 `IS NOT NULL` (실제 결제 완료된 건만 추출)
{: .prompt-warning }

### 회원 추적 구조

```
주문테이블.회원ID → 회원테이블.회원ID (JOIN)
→ 5년 이내 RDS 운영 중이면 실명 조회 가능
→ RDS 없어도 주문번호로 추적 가능
```

---

## 3. AWS 리소스 구성

### 3-1. S3 버킷

**컴플라이언스 버킷 (법적 보존용)**

- **버킷명**: `my-compliance-bucket`
- **Object Lock**: Compliance Mode, **1825일(5년)** - 루트 계정도 삭제 불가
- **Versioning**: Enabled (Object Lock 필수 요건)
- **Lifecycle**:
  - 생성 후 30일 → Glacier Deep Archive 자동 전환
  - 생성 후 1825일(5년) → 자동 삭제 (파기 의무 이행)

```bash
# 버킷 생성
aws s3api create-bucket \
  --bucket my-compliance-bucket \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2 \
  --object-lock-enabled-for-bucket

aws s3api put-bucket-versioning \
  --bucket my-compliance-bucket \
  --versioning-configuration Status=Enabled

aws s3api put-object-lock-configuration \
  --bucket my-compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 1825
      }
    }
  }'
```

**데이터 로그 버킷 (Lambda 실행 로그용)**

- **버킷명**: `my-datalogs-bucket`
- **Object Lock**: 없음
- **Lifecycle**:
  - 생성 후 30일 → Glacier 자동 전환
  - 생성 후 365일(1년) → 자동 삭제

### 3-2. S3 저장 구조

```
my-compliance-bucket/
  └─ orders/
      └─ 2026/
          └─ 03/
              └─ orders_20260329.csv   ← 결제 완료 데이터 (PII 마스킹)

my-datalogs-bucket/
  └─ lambda-logs/
      └─ 2026/
          └─ 03/
              └─ 30/
                  └─ result_20260330_000000.json   ← 실행 결과 로그
```

### 3-3. IAM Role / Policy

- **Role명**: `my-compliance-lambda-role`
- **Policy명**: `my-compliance-lambda-policy`
- **Trust Entity**: `lambda.amazonaws.com`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SecretsManagerRead",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-northeast-2:*:secret:*"
    },
    {
      "Sid": "S3ComplianceBucket",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-compliance-bucket/*"
    },
    {
      "Sid": "S3LogBucket",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-datalogs-bucket/*"
    },
    {
      "Sid": "VPCNetworkInterface",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:ap-northeast-2:*:*"
    },
    {
      "Sid": "SNSPublish",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:ap-northeast-2:<account-id>:my-compliance-alert-topic"
    }
  ]
}
```

### 3-4. SNS 이메일 알림

- **Topic명**: `my-compliance-alert-topic`
- **구독 프로토콜**: Email

**알림 케이스**

| 상태 | 제목 | 발송 조건 |
|---|---|---|
| ✅ 성공 | `[✅ 성공] 주문 데이터 아카이빙 완료 - YYYY-MM-DD` | CSV 정상 업로드 |
| ⚠️ 주의 | `[⚠️ 주의] 주문 데이터 없음 - YYYY-MM-DD` | 해당 날짜 결제 데이터 없음 |
| 🚨 실패 | `[🚨 실패] 주문 데이터 아카이빙 실패 - YYYY-MM-DD` | Lambda 오류 발생 |

**성공 이메일 포함 내용 예시**

```
상태          : 성공 ✅
실행 시간      : 2026-03-30 00:00:00 KST
추출 대상 날짜  : 2026-03-29
추출 건수      : 189건
저장 위치      : s3://my-compliance-bucket/orders/2026/03/orders_20260329.csv

버킷           : my-compliance-bucket
보존 기간       : 5년 (Object Lock Compliance Mode)
스토리지 전환   : 30일 후 → Glacier Deep Archive
파기 예정일     : 2026-03-29 기준 +5년
```

**SNS 구독 추가 방법**

```
SNS 콘솔 → Topics → my-compliance-alert-topic
→ Create subscription
  Protocol: Email
  Endpoint: 추가할 이메일 주소
→ 수신된 확인 메일에서 Confirm subscription 클릭
```

### 3-5. Lambda 함수

| 항목 | 값 |
|---|---|
| 함수명 | `my-compliance-lambda` |
| Runtime | Python 3.12 |
| Execution Role | `my-compliance-lambda-role` |
| Timeout | 60초 |
| Memory | 128MB |
| VPC | RDS와 동일한 VPC |
| Subnet | DB 서브넷 (Multi-AZ, 2개 권장) |
| Security Group | `my-lambda-sg` |

**Lambda 환경변수**

| 키 | 값 |
|---|---|
| `SECRET_NAME` | `<secrets-manager-secret-name>` |
| `S3_COMPLIANCE_BUCKET` | `my-compliance-bucket` |
| `S3_LOG_BUCKET` | `my-datalogs-bucket` |
| `DB_HOST` | `<rds-endpoint>` |
| `SNS_TOPIC_ARN` | `arn:aws:sns:ap-northeast-2:<account-id>:my-compliance-alert-topic` |

**Security Group 구성**

Lambda SG (`my-lambda-sg`)

| 방향 | 포트 | 대상 | 목적 |
|---|---|---|---|
| Outbound | 3306 | `my-rds-sg` | MySQL 접속 |
| Outbound | 443 | 0.0.0.0/0 | Secrets Manager, S3, SNS 통신 |

RDS SG (`my-rds-sg`)

| 방향 | 포트 | 대상 | 목적 |
|---|---|---|---|
| Inbound | 3306 | `my-lambda-sg` | Lambda에서 접속 허용 |

### 3-6. EventBridge 스케줄

| 항목 | 값 |
|---|---|
| Rule명 | `my-compliance-trigger` |
| Schedule | `cron(0 15 * * ? *)` |
| 실행 시간 | **매일 KST 00:00** (UTC 15:00) |
| Target | `my-compliance-lambda` |

---

## 4. Lambda 코드 전문

```python
import json
import boto3
import pymysql
import csv
import io
import os
import logging
from datetime import datetime, timedelta, timezone

logger = logging.getLogger()
logger.setLevel(logging.INFO)

SECRET_NAME          = os.environ['SECRET_NAME']
S3_COMPLIANCE_BUCKET = os.environ['S3_COMPLIANCE_BUCKET']
S3_LOG_BUCKET        = os.environ['S3_LOG_BUCKET']
DB_HOST              = os.environ['DB_HOST']
SNS_TOPIC_ARN        = os.environ['SNS_TOPIC_ARN']
AWS_REGION           = os.environ.get('AWS_REGION', 'ap-northeast-2')
DB_NAME              = '<database-name>'

KST = timezone(timedelta(hours=9))

# PII 컬럼 목록 (실제 컬럼명으로 교체)
PII_COLUMNS = [
    'col_name',       # 주문자명
    'col_tel',        # 주문자전화번호
    'col_email',      # 주문자이메일
    'col_recv_name',  # 받는분
    'col_recv_tel',   # 받는분전화번호
    'col_addr1',      # 받는분주소1
    'col_addr2',      # 받는분주소2
]

def get_db_credentials(secret_name, region):
    client = boto3.client('secretsmanager', region_name=region)
    secret = client.get_secret_value(SecretId=secret_name)
    creds  = json.loads(secret['SecretString'])
    return {
        'host':        DB_HOST,
        'port':        3306,
        'user':        creds['username'],
        'password':    creds['password'],
        'database':    DB_NAME,
        'charset':     'utf8mb4',
        'cursorclass': pymysql.cursors.DictCursor,
        'connect_timeout': 10,
    }

def mask_pii(row):
    masked = row.copy()
    for col in PII_COLUMNS:
        if col not in masked or masked[col] is None:
            continue
        value = str(masked[col])
        # 전화번호 마스킹
        if 'tel' in col:
            digits = value.replace('-', '').replace(' ', '')
            masked[col] = digits[:3] + '-****-' + digits[-4:] if len(digits) >= 8 else '****'
        # 이메일 마스킹
        elif 'email' in col:
            if '@' in value:
                local, domain = value.split('@', 1)
                masked[col] = local[0] + '***@' + domain
            else:
                masked[col] = '***@***.***'
        # 주소 마스킹
        elif 'addr' in col:
            parts = value.split()
            visible = ' '.join(parts[:2]) if len(parts) >= 2 else value[:6]
            masked[col] = visible + ' ***'
        # 이름 마스킹
        else:
            masked[col] = value[0] + '*' * (len(value) - 1) if len(value) > 1 else '*'
    return masked

def serialize_row(row):
    result = {}
    for k, v in row.items():
        if k == 'excluded_column':  # CSV 깨짐 방지를 위해 제외할 컬럼
            continue
        if v is None:
            result[k] = ''
        elif hasattr(v, 'isoformat'):
            result[k] = v.isoformat()
        elif isinstance(v, str):
            result[k] = v.replace('\r\n', ' ').replace('\n', ' ').replace('\r', ' ').replace('\t', ' ')
        else:
            result[k] = v
    return result

def rows_to_csv(rows):
    if not rows:
        return ''
    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=rows[0].keys(), lineterminator='\n', quoting=csv.QUOTE_ALL)
    writer.writeheader()
    writer.writerows(rows)
    return output.getvalue()

def upload_to_s3(s3_client, bucket, target_date, csv_content):
    key = f"orders/{target_date.year}/{target_date.month:02d}/orders_{target_date.strftime('%Y%m%d')}.csv"
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=('\ufeff' + csv_content).encode('utf-8'),
        ContentType='text/csv; charset=utf-8',
    )
    return key

def save_log(s3_client, now_kst, log_data):
    log_key = f"lambda-logs/{now_kst.strftime('%Y/%m/%d')}/result_{now_kst.strftime('%Y%m%d_%H%M%S')}.json"
    s3_client.put_object(
        Bucket=S3_LOG_BUCKET,
        Key=log_key,
        Body=json.dumps(log_data, ensure_ascii=False, indent=2).encode('utf-8'),
        ContentType='application/json',
    )
    logger.info(f"실행 로그 저장: s3://{S3_LOG_BUCKET}/{log_key}")

def send_sns(sns_client, status, log_data):
    target_date     = log_data['target_date']
    extracted_count = log_data['extracted_count']
    s3_key          = log_data['s3_key']
    execution_time  = log_data['execution_time']
    error           = log_data['error']

    if status == 'success':
        subject = f"[✅ 성공] 주문 데이터 아카이빙 완료 - {target_date}"
        message = f"""주문 데이터 아카이빙이 완료되었습니다.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  실행 결과 요약
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  상태          : 성공 ✅
  실행 시간      : {execution_time}
  추출 대상 날짜  : {target_date}
  추출 건수      : {extracted_count:,}건
  저장 위치      : s3://{S3_COMPLIANCE_BUCKET}/{s3_key}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  보존 정책
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  버킷           : {S3_COMPLIANCE_BUCKET}
  보존 기간       : 5년 (Object Lock Compliance Mode)
  스토리지 전환   : 30일 후 → Glacier Deep Archive
  파기 예정일     : {target_date} 기준 +5년
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

본 메일은 자동 발송됩니다.""".strip()

    elif status == 'empty':
        subject = f"[⚠️ 주의] 주문 데이터 없음 - {target_date}"
        message = f"""주문 데이터 아카이빙 실행 결과를 안내드립니다.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  실행 결과 요약
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  상태          : 데이터 없음 ⚠️
  실행 시간      : {execution_time}
  추출 대상 날짜  : {target_date}
  추출 건수      : 0건
  비고           : 해당 날짜 결제 완료 데이터가 없어 S3 업로드를 생략했습니다.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

비정상적으로 판단되는 경우 RDS 및 결제 시스템을 확인해 주세요.

본 메일은 자동 발송됩니다.""".strip()

    else:
        subject = f"[🚨 실패] 주문 데이터 아카이빙 실패 - {target_date}"
        message = f"""주문 데이터 아카이빙 중 오류가 발생했습니다.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  실행 결과 요약
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  상태          : 실패 🚨
  실행 시간      : {execution_time}
  추출 대상 날짜  : {target_date}
  오류 내용      : {error}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

즉시 확인이 필요합니다.

본 메일은 자동 발송됩니다.""".strip()

    sns_client.publish(TopicArn=SNS_TOPIC_ARN, Subject=subject, Message=message)
    logger.info(f"SNS 발송 완료: {subject}")

def lambda_handler(event, context):
    now_kst     = datetime.now(KST)
    target_date = (now_kst - timedelta(days=1)).date()

    s3  = boto3.client('s3',  region_name=AWS_REGION)
    sns = boto3.client('sns', region_name=AWS_REGION)
    logs = []

    def log(msg):
        timestamp = datetime.now(KST).strftime('%Y-%m-%d %H:%M:%S KST')
        logs.append(f"[{timestamp}] {msg}")
        logger.info(msg)

    log_data = {
        "execution_time":  now_kst.strftime('%Y-%m-%d %H:%M:%S KST'),
        "target_date":     target_date.isoformat(),
        "status":          None,
        "extracted_count": 0,
        "s3_key":          None,
        "error":           None,
        "logs":            logs
    }

    try:
        log("Secrets Manager에서 DB credentials 획득 시작")
        db_config  = get_db_credentials(SECRET_NAME, AWS_REGION)
        log("RDS 접속 시도")
        connection = pymysql.connect(**db_config)
        log("RDS 접속 성공")

        with connection.cursor() as cursor:
            log(f"주문 테이블 쿼리 실행 (결제일시 = {target_date})")
            cursor.execute("""
                SELECT * FROM <order_table>
                WHERE DATE(<paydate_column>) = %s
                  AND <paydate_column> IS NOT NULL
            """, (target_date.isoformat(),))
            rows = cursor.fetchall()

        connection.close()
        log("RDS 접속 종료")
        log(f"추출 완료: {len(rows)}건")

        if not rows:
            log_data['status'] = 'empty'
            log("추출 데이터 없음 - S3 업로드 생략")
            save_log(s3, now_kst, log_data)
            send_sns(sns, 'empty', log_data)
            return {'statusCode': 200, 'body': log_data}

        log("PII 마스킹 처리 시작")
        processed = [mask_pii(serialize_row(r)) for r in rows]
        log("PII 마스킹 완료")

        log("CSV 변환 시작")
        csv_content = rows_to_csv(processed)
        log("CSV 변환 완료")

        log(f"S3 업로드 시작: {S3_COMPLIANCE_BUCKET}")
        s3_key = upload_to_s3(s3, S3_COMPLIANCE_BUCKET, target_date, csv_content)
        log(f"S3 업로드 완료: {s3_key}")

        log_data['status']          = 'success'
        log_data['extracted_count'] = len(rows)
        log_data['s3_key']          = s3_key

    except Exception as e:
        log_data['status'] = 'error'
        log_data['error']  = str(e)
        log(f"오류 발생: {str(e)}")
        logger.error(f"Lambda 실패: {e}", exc_info=True)

    finally:
        save_log(s3, now_kst, log_data)
        send_sns(sns, log_data['status'], log_data)

    return {'statusCode': 200, 'body': log_data}
```

---

## 5. PII 마스킹 규칙

| 컬럼 유형 | 원본 예시 | 마스킹 결과 |
|---|---|---|
| 주문자명 | 홍길동 | 홍** |
| 전화번호 | 01012345678 | 010-****-5678 |
| 이메일 | user@example.com | u***@example.com |
| 받는분 | 김철수 | 김** |
| 주소 | 서울특별시 강남구 역삼동 123 | 서울특별시 강남구 *** |

> **회원 추적**: 회원ID + 주문번호로 RDS 또는 회원 테이블 JOIN 조회 가능
{: .prompt-info }

---

## 6. 실행 로그 형식

```json
{
  "execution_time": "2026-03-30 00:00:00 KST",
  "target_date": "2026-03-29",
  "status": "success",
  "extracted_count": 189,
  "s3_key": "orders/2026/03/orders_20260329.csv",
  "error": null,
  "logs": [
    "[2026-03-30 00:00:00 KST] Secrets Manager에서 DB credentials 획득 시작",
    "[2026-03-30 00:00:00 KST] RDS 접속 시도",
    "[2026-03-30 00:00:01 KST] RDS 접속 성공",
    "[2026-03-30 00:00:01 KST] 주문 테이블 쿼리 실행 (결제일시 = 2026-03-29)",
    "[2026-03-30 00:00:03 KST] 추출 완료: 189건",
    "[2026-03-30 00:00:03 KST] PII 마스킹 처리 시작",
    "[2026-03-30 00:00:03 KST] PII 마스킹 완료",
    "[2026-03-30 00:00:03 KST] CSV 변환 시작",
    "[2026-03-30 00:00:03 KST] CSV 변환 완료",
    "[2026-03-30 00:00:03 KST] S3 업로드 시작: my-compliance-bucket",
    "[2026-03-30 00:00:04 KST] S3 업로드 완료: orders/2026/03/orders_20260329.csv"
  ]
}
```

---

## 7. 패키징 및 재배포

```bash
# 1. EC2에서 코드 수정
vi /tmp/lambda-package/lambda_function.py

# 2. 재패키징
cd /tmp/lambda-package
zip -r /tmp/lambda-package.zip .

# 3. S3 업로드
aws s3 cp /tmp/lambda-package.zip \
  s3://my-datalogs-bucket/tmp/lambda-package.zip
```

**배포 순서**

```
1. EC2에서 코드 수정 후 zip 재패키징
2. S3 업로드
3. Lambda 콘솔 → Code → Upload from → Amazon S3 location → URL 입력 → Save
4. Test 실행으로 동작 확인
```

---

## 8. 구성 완료 체크리스트

> **S3 버킷**<br>
> - my-compliance-bucket (Object Lock Compliance 5년)<br>
> - my-datalogs-bucket (로그용, 1년 후 삭제)
>
> **IAM 구성**<br>
> - my-compliance-lambda-role 생성<br>
> - my-compliance-lambda-policy 연결 (SecretsManagerRead, S3ComplianceBucket, S3LogBucket, VPCNetworkInterface, CloudWatchLogs, SNSPublish)
>
> **SNS 알림**<br>
> - my-compliance-alert-topic 생성<br>
> - 담당자 이메일 구독 확인
>
> **Lambda 함수**<br>
> - my-compliance-lambda 생성<br>
> - pymysql 포함 zip 패키지 배포<br>
> - 환경변수 5개 설정 완료<br>
> - VPC 설정 (Private Subnet 2개)<br>
> - Security Group 설정 (my-lambda-sg, Outbound 3306/443)<br>
> - Timeout 60초 설정
>
> **EventBridge**<br>
> - my-compliance-trigger 생성<br>
> - cron(0 15 * * ? *) = KST 매일 00:00
>
> **테스트**<br>
> - Lambda 수동 테스트 성공<br>
> - compliance 버킷 CSV 파일 생성 확인<br>
> - datalogs 버킷 실행 로그 JSON 생성 확인<br>
> - SNS 이메일 수신 확인
{: .prompt-tip }

---

## 9. 운영 참고사항

### 데이터 조회 시나리오 (5년 이내)

```sql
-- 1. compliance 버킷에서 해당 날짜 CSV 다운로드
-- s3://my-compliance-bucket/orders/YYYY/MM/orders_YYYYMMDD.csv

-- 2. 주문번호 또는 회원ID로 검색

-- 3. 실명 확인이 필요한 경우
SELECT * FROM <member_table> WHERE <member_id_column> = 'target_id';
```

### Glacier Deep Archive 복원 (30일 이후 데이터)

```
S3 콘솔 → 해당 파일 선택 → Restore from Glacier
복원 소요시간: 12~48시간
복원 후 임시 접근 가능 기간: 권장 7일
```

### 이상 발생 시 확인 경로

> 1. 이메일 알림 확인 (SNS) - 성공/실패/데이터없음 자동 발송<br>
> 2. my-datalogs-bucket → lambda-logs/ → 해당 날짜 JSON 확인<br>
> 3. CloudWatch Logs → `/aws/lambda/my-compliance-lambda`<br>
> 4. Lambda 콘솔 → Monitor 탭 → 실행 지표 확인
{: .prompt-warning }
