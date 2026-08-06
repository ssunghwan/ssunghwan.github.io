---
title: "비밀번호를 사람이 바꾸지 마라 - Secrets Manager RDS 자동 로테이션 구축기"
date: 2026-04-23 00:00:00 +0900
categories: [Security & Compliance, Hardening]
tags: [aws, secrets-manager, rds, rotation, lambda, kms]
---

> **환경**: AWS ap-northeast-2
{: .prompt-info }

---

## 1. AWS Secrets Manager란?

AWS Secrets Manager는 데이터베이스 자격증명, API 키, OAuth 토큰 등 **민감한 정보(Secret)를 중앙에서 안전하게 저장·관리·교체**할 수 있는 AWS 관리형 서비스다.

### 1-1. 주요 기능

| 기능 | 설명 |
|---|---|
| 중앙 집중 관리 | 모든 자격증명을 단일 서비스에서 관리 |
| 자동 교체 (Rotation) | Lambda를 통해 주기적으로 패스워드 자동 변경 |
| 암호화 | KMS를 통한 저장 데이터 암호화 |
| 감사 | CloudTrail 연동으로 접근 이력 추적 |
| 버전 관리 | AWSCURRENT / AWSPENDING / AWSPREVIOUS 스테이지로 버전 관리 |

### 1-2. 활용 사례

- **RDS / Aurora 자격증명 관리**: 애플리케이션 코드에 하드코딩된 DB 패스워드 제거
- **API Key 관리**: 외부 서비스 연동 키를 코드베이스에서 분리
- **ISMS-P / 컴플라이언스 대응**: 정기적인 패스워드 교체 정책 자동화
- **멀티 환경 관리**: dev / staging / prod 환경별 Secret 분리 관리

### 1-3. 하드코딩 대비 장점

```python
# 기존 방식 (비권장)
DB_PASSWORD = "MyPassword123!"  # 코드에 직접 노출

# Secrets Manager 사용
import boto3, json
client = boto3.client('secretsmanager')
secret = client.get_secret_value(SecretId='prod/myapp/mysql')
credentials = json.loads(secret['SecretString'])
DB_PASSWORD = credentials['password']  # 런타임에 동적으로 로드
```

---

## 2. Rotation(자동 교체) 동작 구조

Secrets Manager Rotation은 **Lambda 함수를 4단계로 순차 호출**하여 패스워드를 안전하게 교체한다.

```
createSecret → setSecret → testSecret → finishSecret
  새 PW 생성     DB에 PW 반영   접속 검증     버전 전환 완료
```

| 단계 | 역할 |
|---|---|
| `createSecret` | 랜덤 패스워드 생성 후 `AWSPENDING` 버전으로 저장 |
| `setSecret` | `AWSCURRENT` or `AWSPREVIOUS` secret으로 DB 접속 후 패스워드 변경 |
| `testSecret` | `AWSPENDING` secret으로 DB 접속 검증 |
| `finishSecret` | `AWSPENDING` → `AWSCURRENT`로 스테이지 전환 |

### Secret 버전 스테이지

```
AWSPREVIOUS  ←  이전 버전 (롤백용)
AWSCURRENT   ←  현재 사용 중인 버전
AWSPENDING   ←  교체 진행 중인 버전 (rotation 완료 시 AWSCURRENT로 전환)
```

---

## 3. 구성 요소 및 네트워크 구조

Rotation이 정상 동작하려면 아래 구성 요소가 모두 올바르게 설정되어야 한다.

```
┌─────────────────────────────────────────────────────┐
│                        VPC                          │
│                                                     │
│  ┌──────────────┐         ┌──────────────────────┐  │
│  │    Lambda    │──3306──▶│        RDS           │  │
│  │  (Rotation)  │         │  (MySQL 8.0)         │  │
│  └──────┬───────┘         └──────────────────────┘  │
│         │                                           │
│         │ HTTPS (443)                               │
│         ▼                                           │
│  ┌──────────────────────┐                           │
│  │  Secrets Manager     │                           │
│  │  VPC Endpoint        │                           │
│  └──────────────────────┘                           │
└─────────────────────────────────────────────────────┘
```

**필수 조건**:
- Lambda가 VPC 내 DB 서브넷에 배치
- Lambda SG → RDS SG 3306 인바운드 허용
- Secrets Manager VPC Endpoint 설정 (Private 환경)

---

## 4. Rotation Lambda 구성

Secrets Manager에서 Rotation을 활성화하면 AWS가 자동으로 Lambda 함수를 생성한다. 단, 아래 항목들은 **수동으로 설정**해야 한다.

### 4-1. Lambda 기본 정보

| 항목 | 설명 |
|---|---|
| 런타임 | Python 3.12 |
| 메모리 | 128 MB (기본값) |
| 타임아웃 | 30초 이상 권장 (DB 접속 포함) |
| 함수명 규칙 | `SecretsManager<secret-name>-rotation-lambda` |

### 4-2. VPC 설정

Lambda가 Private 서브넷의 RDS에 접근하려면 반드시 **동일 VPC 내 DB 서브넷**에 배치해야 한다.

**콘솔**: Lambda → Configuration → VPC → 편집

| 항목 | 설정값 |
|---|---|
| VPC | RDS와 동일한 VPC |
| 서브넷 | RDS가 위치한 DB 서브넷 (Multi-AZ 권장) |
| 보안 그룹 | Rotation 전용 SG 생성 후 할당 |

### 4-3. 보안 그룹 설정

Rotation Lambda 전용 SG를 별도 생성하고 아래와 같이 구성한다.

**Lambda SG (아웃바운드)**:

| 유형 | 프로토콜 | 포트 | 대상 |
|---|---|---|---|
| Custom TCP | TCP | 3306 | `<rds-security-group-id>` |
| HTTPS | TCP | 443 | `0.0.0.0/0` (또는 Secrets Manager VPC Endpoint SG) |

**RDS SG (인바운드)**:

| 유형 | 프로토콜 | 포트 | 소스 |
|---|---|---|---|
| Custom TCP | TCP | 3306 | `<lambda-security-group-id>` |

> RDS SG 인바운드에 Lambda SG를 소스로 지정하지 않으면 `setSecret` 단계에서 DB 접속 실패가 발생한다.
{: .prompt-danger }

### 4-4. 환경 변수

| 키 | 값 | 설명 |
|---|---|---|
| `SECRETS_MANAGER_ENDPOINT` | `https://secretsmanager.<region>.amazonaws.com` | VPC Endpoint 사용 시 VPC Endpoint URL로 변경 |
| `EXCLUDE_CHARACTERS` | `/@"'\` | 생성 패스워드에서 제외할 특수문자 |
| `PASSWORD_LENGTH` | `32` | 생성 패스워드 길이 |

### 4-5. IAM 실행 역할 (Execution Role)

Lambda 실행 역할에 아래 권한이 필요하다.

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:DescribeSecret",
    "secretsmanager:GetSecretValue",
    "secretsmanager:PutSecretValue",
    "secretsmanager:UpdateSecretVersionStage"
  ],
  "Resource": "arn:aws:secretsmanager:<region>:<account-id>:secret:<secret-name>*"
}
```

추가로 KMS 커스텀 키 사용 시:

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:<region>:<account-id>:key/<kms-key-id>"
}
```

### 4-6. Secrets Manager 리소스 기반 정책

Secret이 Lambda 함수의 호출을 허용하도록 리소스 기반 정책도 필요하다.

**Secrets Manager → 보안 암호 → 리소스 기반 정책**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "secretsmanager:RotateSecret",
      "Resource": "*",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:lambda:<region>:<account-id>:function:<rotation-lambda-name>"
        }
      }
    }
  ]
}
```

### 4-7. Lambda 함수 코드 전체 (lambda_function.py)

AWS가 자동 생성하는 Rotation Lambda의 전체 코드다. 커스텀 수정이 필요한 경우 참고한다.

```python
import boto3
import json
import logging
import os
import pymysql
import re

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def lambda_handler(event, context):
    arn   = get_input_map_value(event, 'SecretId')
    token = get_input_map_value(event, 'ClientRequestToken')
    step  = get_input_map_value(event, 'Step')

    service_client = boto3.client('secretsmanager', endpoint_url=os.environ['SECRETS_MANAGER_ENDPOINT'])

    metadata = service_client.describe_secret(SecretId=arn)
    if "RotationEnabled" in metadata and not metadata['RotationEnabled']:
        raise ValueError("Secret %s is not enabled for rotation" % arn)

    versions = metadata['VersionIdsToStages']
    if token not in versions:
        raise ValueError("Secret version %s has no stage for rotation of secret %s." % (token, arn))
    if "AWSCURRENT" in versions[token]:
        logger.info("Secret version %s already set as AWSCURRENT for secret %s." % (token, arn))
        return
    elif "AWSPENDING" not in versions[token]:
        raise ValueError("Secret version %s not set as AWSPENDING for rotation of secret %s." % (token, arn))

    if step == "createSecret":
        create_secret(service_client, arn, token)
    elif step == "setSecret":
        set_secret(service_client, arn, token)
    elif step == "testSecret":
        test_secret(service_client, arn, token)
    elif step == "finishSecret":
        finish_secret(service_client, arn, token)
    else:
        raise ValueError("Invalid step parameter %s for secret %s" % (step, arn))


def create_secret(service_client, arn, token):
    current_dict = get_secret_dict(service_client, arn, "AWSCURRENT")
    try:
        get_secret_dict(service_client, arn, "AWSPENDING", token)
        logger.info("createSecret: Successfully retrieved secret for %s." % arn)
    except service_client.exceptions.ResourceNotFoundException:
        current_dict['password'] = get_random_password(service_client)
        service_client.put_secret_value(
            SecretId=arn,
            ClientRequestToken=token,
            SecretString=json.dumps(current_dict),
            VersionStages=['AWSPENDING']
        )
        logger.info("createSecret: Successfully put secret for ARN %s and version %s." % (arn, token))


def set_secret(service_client, arn, token):
    try:
        previous_dict = get_secret_dict(service_client, arn, "AWSPREVIOUS")
    except (service_client.exceptions.ResourceNotFoundException, KeyError):
        previous_dict = None

    current_dict = get_secret_dict(service_client, arn, "AWSCURRENT")
    pending_dict = get_secret_dict(service_client, arn, "AWSPENDING", token)

    conn = get_connection(pending_dict)
    if conn:
        conn.close()
        logger.info("setSecret: AWSPENDING secret is already set as password in MySQL DB for secret arn %s." % arn)
        return

    if current_dict['username'] != pending_dict['username']:
        raise ValueError("Attempting to modify user %s other than current user %s" % (pending_dict['username'], current_dict['username']))
    if current_dict['host'] != pending_dict['host']:
        raise ValueError("Attempting to modify user for host %s other than current host %s" % (pending_dict['host'], current_dict['host']))

    conn = get_connection(current_dict)

    if not conn and previous_dict:
        previous_dict.pop('ssl', None)
        if 'ssl' in current_dict:
            previous_dict['ssl'] = current_dict['ssl']
        conn = get_connection(previous_dict)
        if previous_dict['username'] != pending_dict['username']:
            raise ValueError("Attempting to modify user %s other than previous valid user %s" % (pending_dict['username'], previous_dict['username']))
        if previous_dict['host'] != pending_dict['host']:
            raise ValueError("Attempting to modify user for host %s other than previous host %s" % (pending_dict['host'], previous_dict['host']))

    if not conn:
        logger.error("setSecret: Unable to log into database with previous, current, or pending secret of secret arn %s" % arn)
        raise ValueError("Unable to log into database with previous, current, or pending secret of secret arn %s" % arn)

    try:
        with conn.cursor() as cur:
            cur.execute("SELECT VERSION()")
            ver = cur.fetchone()
            password_option = get_password_option(ver[0])
            cur.execute("SET PASSWORD = " + password_option, pending_dict['password'])
            conn.commit()
            logger.info("setSecret: Successfully set password for user %s in MySQL DB for secret arn %s." % (pending_dict['username'], arn))
    finally:
        conn.close()


def test_secret(service_client, arn, token):
    conn = get_connection(get_secret_dict(service_client, arn, "AWSPENDING", token))
    if conn:
        try:
            with conn.cursor() as cur:
                cur.execute("SELECT NOW()")
                conn.commit()
        finally:
            conn.close()
        logger.info("testSecret: Successfully signed into MySQL DB with AWSPENDING secret in %s." % arn)
    else:
        raise ValueError("Unable to log into database with pending secret of secret ARN %s" % arn)


def finish_secret(service_client, arn, token):
    metadata = service_client.describe_secret(SecretId=arn)
    current_version = None
    for version in metadata["VersionIdsToStages"]:
        if "AWSCURRENT" in metadata["VersionIdsToStages"][version]:
            if version == token:
                logger.info("finishSecret: Version %s already marked as AWSCURRENT for %s" % (version, arn))
                return
            current_version = version
            break

    service_client.update_secret_version_stage(
        SecretId=arn,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId=current_version
    )
    logger.info("finishSecret: Successfully set AWSCURRENT stage to version %s for secret %s." % (token, arn))


def get_connection(secret_dict):
    port   = int(secret_dict['port']) if 'port' in secret_dict else 3306
    # dbname 키가 없으면 None으로 처리 — 존재하지 않는 스키마 지정 시 접속 실패 원인이 됨
    dbname = secret_dict['dbname'] if 'dbname' in secret_dict else None

    use_ssl, fall_back = get_ssl_config(secret_dict)
    conn = connect_and_authenticate(secret_dict, port, dbname, use_ssl)
    if conn or not fall_back:
        return conn
    else:
        return connect_and_authenticate(secret_dict, port, dbname, False)


def get_ssl_config(secret_dict):
    if 'ssl' not in secret_dict:
        return True, True
    if isinstance(secret_dict['ssl'], bool):
        return secret_dict['ssl'], False
    if isinstance(secret_dict['ssl'], str):
        ssl = secret_dict['ssl'].lower()
        if ssl == "true":
            return True, False
        elif ssl == "false":
            return False, False
    return True, True


def connect_and_authenticate(secret_dict, port, dbname, use_ssl):
    ssl = {'ca': '/etc/pki/tls/cert.pem'} if use_ssl else None
    try:
        conn = pymysql.connect(
            host=secret_dict['host'],
            user=secret_dict['username'],
            password=secret_dict['password'],
            port=port,
            database=dbname,
            connect_timeout=5,
            ssl=ssl
        )
        logger.info("Successfully established %s connection as user '%s' with host: '%s'" % (
            "SSL/TLS" if use_ssl else "non SSL/TLS", secret_dict['username'], secret_dict['host']))
        return conn
    except pymysql.OperationalError as e:
        if 'certificate verify failed: IP address mismatch' in e.args[1]:
            logger.error("Hostname verification failed when establishing SSL/TLS Handshake with host: %s" % secret_dict['host'])
        return None


def get_secret_dict(service_client, arn, stage, token=None):
    required_fields = ['host', 'username', 'password']
    if token:
        secret = service_client.get_secret_value(SecretId=arn, VersionId=token, VersionStage=stage)
    else:
        secret = service_client.get_secret_value(SecretId=arn, VersionStage=stage)

    secret_dict = json.loads(secret['SecretString'])

    supported_engines = ["mysql", "aurora-mysql"]
    if 'engine' not in secret_dict or secret_dict['engine'] not in supported_engines:
        raise KeyError("Database engine must be set to 'mysql' in order to use this rotation lambda")
    for field in required_fields:
        if field not in secret_dict:
            raise KeyError("%s key is missing from secret JSON" % field)
    return secret_dict


def get_password_option(version):
    # MySQL 8.0+: SET PASSWORD = %s / 이하 버전: SET PASSWORD = PASSWORD(%s)
    if version.startswith("8"):
        return "%s"
    else:
        return "PASSWORD(%s)"


def get_random_password(service_client):
    passwd = service_client.get_random_password(
        ExcludeCharacters=os.environ.get('EXCLUDE_CHARACTERS', '/@"\'\\'),
        PasswordLength=int(os.environ.get('PASSWORD_LENGTH', 32)),
        ExcludeNumbers=get_environment_bool('EXCLUDE_NUMBERS', False),
        ExcludePunctuation=get_environment_bool('EXCLUDE_PUNCTUATION', False),
        ExcludeUppercase=get_environment_bool('EXCLUDE_UPPERCASE', False),
        ExcludeLowercase=get_environment_bool('EXCLUDE_LOWERCASE', False),
        RequireEachIncludedType=get_environment_bool('REQUIRE_EACH_INCLUDED_TYPE', True)
    )
    return passwd['RandomPassword']


def get_environment_bool(variable_name, default_value):
    variable = os.environ.get(variable_name, str(default_value))
    return variable.lower() in ['true', '1', 'y', 'yes']


def get_input_map_value(input_dict, field_name):
    if field_name in input_dict:
        raw_value = input_dict[field_name]
        if re.match(r'^[ -~]+$', raw_value) is not None:
            return raw_value
        else:
            raise ValueError("\"%s\" contains invalid characters." % field_name)
    raise ValueError("No value provided for \"%s\"." % field_name)
```

---

## 5. 트러블슈팅 (Rotation 실패)

### 5-1. 증상

Rotation Lambda 실행 시 `setSecret` 단계에서 반복 실패.

```
[ERROR] setSecret: Unable to log into database with previous, current,
or pending secret of secret arn arn:aws:secretsmanager:<region>:<account-id>:secret:<secret-name>

ValueError: Unable to log into database with previous, current, or pending secret
  File "/var/task/lambda_function.py", line 193, in set_secret
```

---

### 5-2. 원인 1 — AWSPENDING stuck 상태

이전 rotation 시도가 실패하면서 `AWSPENDING` 버전이 정리되지 않고 남아 있음. 이 상태에서 새 rotation을 트리거하면 아래 메시지가 발생하며 진행되지 않음.

```
A previous rotation isn't complete. That rotation will be reattempted.
```

**확인 방법**:

```bash
aws secretsmanager describe-secret \
  --secret-id <secret-arn> \
  --region ap-northeast-2 \
  | jq '.VersionIdsToStages'
```

```json
// stuck 상태 예시
{
  "버전-id (ex: abcd1234-45ab-89xz-4321dcba": ["AWSPENDING"],   // ← 이게 남아있으면 stuck
  "버전-id (ex: 4321dcba-ba54-zx98-abcd1234": ["AWSPREVIOUS"],
  "버전-id (ex: sunghwan-v0e1-r2s4-test0321": ["AWSCURRENT"]
}
```

**조치**:

```bash
aws secretsmanager update-secret-version-stage \
  --secret-id <secret-arn> \
  --version-stage AWSPENDING \
  --remove-from-version-id <PENDING-version-id> \
  --region ap-northeast-2
```

정상 상태 확인:

```json
// AWSPENDING이 사라진 것을 확인
{
  "버전-id (ex: 4321dcba-ba54-zx98-abcd1234": ["AWSPREVIOUS"],
  "버전-id (ex: sunghwan-v0e1-r2s4-test0321": ["AWSCURRENT"]
}
```

---

### 5-3. 원인 2 — Lambda SG → RDS SG 3306 인바운드 미설정

Lambda Rotation 함수가 VPC 내에서 RDS에 직접 TCP 3306으로 접속하는데, RDS Security Group 인바운드 규칙에 Lambda SG가 허용되지 않았음.

**확인 방법**: RDS SG 인바운드 규칙에서 Lambda SG ID 존재 여부 확인.

**조치**: RDS Security Group 인바운드에 아래 규칙 추가.

| 유형 | 프로토콜 | 포트 | 소스 |
|---|---|---|---|
| Custom TCP | TCP | 3306 | `<lambda-security-group-id>` |

---

### 5-4. 원인 3 — Secret의 dbname 필드에 존재하지 않는 스키마 지정 (근본 원인)

Secret JSON의 `dbname` 필드가 RDS 인스턴스에 실제로 존재하지 않는 스키마명으로 설정되어 있었음.

Lambda 코드는 `dbname` 키가 있으면 해당 값으로 `pymysql.connect(database=dbname)`을 호출한다.

```python
# lambda_function.py
dbname = secret_dict['dbname'] if 'dbname' in secret_dict else None
# ...
conn = pymysql.connect(
    host=secret_dict['host'],
    user=secret_dict['username'],
    password=secret_dict['password'],
    port=port,
    database=dbname,   # ← 여기서 존재하지 않는 스키마로 접속 시도
    connect_timeout=5,
    ssl=ssl
)
```

존재하지 않는 스키마로 접속을 시도하므로 `AWSPENDING`, `AWSCURRENT`, `AWSPREVIOUS` 세 버전 모두 접속 실패 → `ValueError` 발생.

> RDS 인스턴스 생성 시 설정하는 **Initial database name**은 실제 스키마 생성을 보장하지 않는 경우가 있음.<br>
> 반드시 `SHOW DATABASES`로 존재 여부를 확인해야 함.
{: .prompt-warning }

**실제 스키마 확인**:

```bash
mysql -h <rds-endpoint> -u <username> -p \
  -e "SHOW DATABASES;"
```

**조치**: Secret JSON에서 `dbname` 키 제거.

`dbname` 키가 없으면 Lambda는 `database=None`으로 접속하며, 이는 스키마 선택 없이 MySQL 서버 자체에 연결하는 것으로 rotation 동작에 문제없음.

```bash
aws secretsmanager update-secret \
  --secret-id <secret-arn> \
  --secret-string '{
    "username": "<db-username>",
    "password": "<current-password>",
    "engine": "mysql",
    "host": "<rds-endpoint>",
    "port": 3306,
    "dbInstanceIdentifier": "<rds-instance-id>",
    "dbname1": "<schema-name-1>",
    "dbname2": "<schema-name-2>"
  }' \
  --region ap-northeast-2
```

> `dbname1`, `dbname2`는 애플리케이션 참조용 커스텀 키로 Rotation Lambda 로직에 영향 없음.
{: .prompt-info }

---

## 6. Rotation 재트리거 및 결과 확인

**트리거**:

```bash
aws secretsmanager rotate-secret \
  --secret-id <secret-arn> \
  --region ap-northeast-2
```

**CloudWatch 정상 로그 확인**: `/aws/lambda/<rotation-lambda-name>`

```
[INFO] createSecret: Successfully put secret for ARN ... and version ...
[INFO] setSecret: Successfully set password for user <username> in MySQL DB ...
[INFO] testSecret: Successfully signed into MySQL DB with AWSPENDING secret ...
[INFO] finishSecret: Successfully set AWSCURRENT stage to version ...
```

---

## 7. Secret JSON 필드 정의

| 키 | 필수 여부 | 설명 | Rotation Lambda 사용 |
|---|---|---|---|
| `username` | ✅ 필수 | DB 계정명 | ✅ |
| `password` | ✅ 필수 | DB 패스워드 | ✅ |
| `engine` | ✅ 필수 | `mysql` 또는 `aurora-mysql` 고정 | ✅ |
| `host` | ✅ 필수 | RDS 엔드포인트 | ✅ |
| `port` | 선택 | 미설정 시 3306 기본값 | ✅ |
| `dbname` | 선택 | 접속 스키마명, 미설정 시 NULL | ✅ (있을 경우) |
| `dbInstanceIdentifier` | 선택 | RDS 인스턴스 ID (참조용) | ❌ |
| `dbname1`, `dbname2` | 선택 | 애플리케이션 참조용 커스텀 키 | ❌ |

---

## 8. 재발 방지 체크리스트

> - Secret `dbname` 필드: 실제 존재하는 스키마명 또는 키 자체 제거<br>
> - Lambda VPC 배치: RDS와 동일 VPC, DB 서브넷 확인<br>
> - Lambda SG → RDS SG: TCP 3306 인바운드 허용 확인<br>
> - Secrets Manager VPC Endpoint: Lambda 환경변수 `SECRETS_MANAGER_ENDPOINT` 설정 확인<br>
> - AWSPENDING 정리: Rotation 실패 후 재시도 전 PENDING 버전 제거 확인<br>
> - DB 유저 존재 여부: Secret의 `username`이 실제 DB에 존재하는지 확인
{: .prompt-tip }
