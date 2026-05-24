---
title: "From containerizing a test Next.js app to MSA separation"
date: 2026-05-23 09:00:00 +0900
categories: [Kubernetes, Legacy PHP eCommerce - EKS Migration]
tags: [eks, docker, php-fpm, nginx, tomcat, nextjs, nodejs, karpenter, external-secrets, external-dns, acm, rds, route53, terraform, msa]
---

> EKS 마이그레이션 시리즈 다섯 번째 포스팅이다.<br>
> 앞선 포스팅에서 EKS 클러스터, Karpenter, ALBC, ESO, ArgoCD까지 구성을 완료했다. 이번에는 실제 레거시 PHP 이커머스 앱을 컨테이너화하고 EKS에 배포하는 전 과정을 기록한다.<br>
> PHP 앱 컨테이너화 → 수많은 500 에러 트러블슈팅 → 해결이 너무 복잡해서 Next.js로 핵심 페이지 재구현 → MSA 분리까지. 실제 현장에서 겪은 날것의 과정을 그대로 담았다.
{: .prompt-info }

---

## 1. 소스코드 수집

기존 운영서버에서 PHP 소스를 가져왔다. `/var/www/html` 전체를 S3를 경유해 VSCode Server로 옮겼다.

```bash
# 운영서버에서 S3로 업로드
aws s3 sync /var/www/html/ s3://<bucket>/purina-php-src/ \
  --exclude ".git/*" \
  --exclude "vendor/*" \
  --exclude "var/*" \
  --exclude "efs/*" \
  --exclude ".env" \
  --exclude ".env.prod"

# VSCode Server에서 S3에서 다운로드
aws s3 sync s3://<bucket>/purina-php-src/ ~/apps/php-app/ \
  --region ap-northeast-2
```

> `vendor/`는 약 625MB라 반드시 제외해야 한다. 포함하면 빌드 컨텍스트가 폭발해서 EC2 EBS를 금방 채워버린다. 빌드 시 `composer install --no-dev`로 재생성한다 (~175MB).
{: .prompt-warning }

Tomcat SSO는 소스코드가 없고 WAR만 있어서 운영서버에서 `Sso.war`만 복사해왔다.

---

## 2. Docker 이미지 설계

### Sidecar 패턴 vs Separate Pod

EKS에서 PHP-FPM + Nginx 배포 패턴은 두 가지다.

**Sidecar (같은 Pod)**

```
Pod
├── nginx container     (포트 8080 → ALB)
└── php-fpm container   (Unix socket 공유)
    └── /run/php/php7.2-fpm.sock
```

- Unix socket 통신으로 레이턴시 최소
- nginx + php-fpm 항상 같이 스케일
- 중소 트래픽에 적합

**Separate Pod**

```
Nginx Pod (N개) → PHP-FPM Pod (M개)  (TCP 통신)
```

- 독립 스케일 가능
- PHP가 진짜 병목일 때 유효

> 45K MAU 규모이므로 **Sidecar 패턴**으로 시작하고 나중에 필요시 분리하기로 결정했다.
{: .prompt-tip }

### EKS에서 소스 공유 방법 — initContainer 패턴

로컬 docker-compose의 bind mount는 EKS에서 쓸 수 없다. **initContainer + emptyDir** 패턴을 사용한다.

```
initContainer (copy-source)
  └── php 이미지에서 /var/www/html/* → /app/ 복사
         ↓
  emptyDir volume (app-source)
         ↓
  nginx + php-fpm 컨테이너가 /var/www/html 로 마운트해서 공유
```

---

## 3. Dockerfile 작성

### PHP-FPM Dockerfile

```dockerfile
FROM debian:bullseye-slim

RUN apt-get update && apt-get install -y \
    php7.4-fpm \
    php7.4-mysql \
    php7.4-mbstring \
    php7.4-zip \
    php7.4-gd \
    php7.4-intl \
    php7.4-xml \
    php7.4-bcmath \
    php7.4-opcache \
    composer \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html
COPY . .

RUN composer install --no-dev --optimize-autoloader --no-scripts

# php.ini 설정 (mod_php 방식의 .htaccess php_value 대신 여기서 처리)
RUN sed -i \
    -e 's/^upload_max_filesize = .*/upload_max_filesize = 32M/' \
    -e 's/^post_max_size = .*/post_max_size = 32M/' \
    -e 's/^;max_input_vars = .*/max_input_vars = 3000/' \
    -e 's/^short_open_tag = .*/short_open_tag = On/' \
    -e 's/^session.use_strict_mode = .*/session.use_strict_mode = On/' \
    /etc/php/7.4/fpm/php.ini

# OPcache system-level 설정 (PHP_INI_SYSTEM은 php.ini에서만 가능)
RUN echo "opcache.memory_consumption=128\nopcache.max_accelerated_files=4000" \
    > /etc/php/7.4/fpm/conf.d/99-opcache-override.ini

RUN mkdir -p /var/www/html/public/data/session \
    && mkdir -p /var/www/html/var/cache \
    && mkdir -p /var/www/html/var/log \
    && chown -R www-data:www-data /var/www/html/var /var/www/html/public/data

COPY docker/php-fpm/www.conf /etc/php/7.4/fpm/pool.d/www.conf
USER www-data
```

> **멀티스테이지 빌드를 쓰지 않은 이유**<br>
> PHP-FPM은 런타임에 소스코드 전체가 필요하다. Node.js처럼 빌드 아티팩트만 복사하는 패턴이 적용되지 않는다.<br>
> 대신 `--no-dev`로 개발 의존성을 제외하고 `.dockerignore`로 불필요한 파일을 제거해 이미지 크기를 줄였다.<br>
> `composer install` 전 `vendor/` 제외 덕분에 빌드 컨텍스트는 약 50MB, 최종 이미지는 약 400MB 수준이다.
{: .prompt-info }

### .dockerignore

```
vendor/
var/
efs/
.git/
.env
.env.local
.env.prod
*.log
```

### www.conf (PHP-FPM Pool 설정)

```ini
[www]
user = www-data
group = www-data

listen = /run/php/php7.2-fpm.sock
listen.mode = 0666
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 20
pm.start_servers = 5
pm.min_spare_servers = 3
pm.max_spare_servers = 10

; 환경변수를 PHP-FPM 프로세스로 전달 — 반드시 필요
clear_env = no

php_flag[display_errors] = off
php_flag[log_errors] = on

; opcache.memory_consumption은 PHP_INI_SYSTEM이라 php_value 사용 불가
; Dockerfile에서 ini 파일로 직접 주입
php_admin_value[opcache.enable] = 1
php_admin_value[opcache.revalidate_freq] = 60

pm.status_path = /fpm-status
ping.path = /fpm-ping

access.log = /proc/self/fd/2
```

> **`clear_env = no` 필수**: 없으면 Pod 환경변수가 PHP-FPM으로 전달 안 됨 → DB 연결 실패<br>
> **`listen.mode = 0666` 필수**: 기본값 0660이면 nginx 유저가 소켓 접근 불가 → 502<br>
> **`opcache.memory_consumption`**: PHP_INI_SYSTEM 설정이라 `php_value`로 설정 불가. Dockerfile에서 ini 파일로 직접 주입해야 한다.
{: .prompt-danger }

**`pm = dynamic`을 선택한 이유**

PHP-FPM의 프로세스 관리 방식은 세 가지다.

| 방식 | 동작 | 적합한 환경 |
|---|---|---|
| `static` | 항상 `pm.max_children`개 유지 | 트래픽이 일정하고 예측 가능한 경우 |
| `dynamic` | 최소/최대 사이에서 동적 조정 | 일반적인 웹 서비스 (권장) |
| `ondemand` | 요청 시에만 프로세스 생성 | 트래픽이 매우 적거나 간헐적인 경우 |

`dynamic`은 `pm.start_servers`로 초기 프로세스를 미리 띄워두고, 트래픽에 따라 `pm.min_spare_servers` ~ `pm.max_spare_servers` 사이에서 자동 조정한다. EKS에서는 HPA가 Pod 수를 조정하므로, 프로세스당 트래픽이 일정하게 유지되어 `dynamic`이 적합하다.

### nginx.conf

```nginx
worker_processes auto;
error_log /dev/stderr warn;
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    # Non-root 유저로 실행 시 /var/cache/nginx 쓰기 권한 없음 → /tmp로 이동
    client_body_temp_path /tmp/client_temp;
    proxy_temp_path       /tmp/proxy_temp;
    fastcgi_temp_path     /tmp/fastcgi_temp;
    uwsgi_temp_path       /tmp/uwsgi_temp;
    scgi_temp_path        /tmp/scgi_temp;

    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time';

    access_log /dev/stdout main;
    sendfile        on;
    keepalive_timeout 65;
    server_tokens off;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

> **Non-root 유저(`USER nginx`)로 실행하는 이유**<br>
> 컨테이너를 root로 실행하면 컨테이너 탈출 취약점 발생 시 호스트에 대한 권한을 가질 수 있다.<br>
> `USER nginx`로 non-root 유저를 지정하면 프로세스 권한을 최소화할 수 있다.<br>
> 단, nginx는 기본적으로 `/var/cache/nginx`, `/var/run/nginx.pid` 등에 쓰기 권한이 필요한데 non-root 유저는 이 경로에 접근할 수 없다.<br>
> 따라서 모든 임시 파일 경로를 `/tmp`로 변경했다.
{: .prompt-warning }

### default.conf

> **헬스체크 경로를 `/elb.php`로 분리한 이유**<br>
> ALB 헬스체크가 `/`를 타면 PHP 앱 전체 로직(DB 연결, 세션, include 체인)이 실행된다.<br>
> 이 과정에서 500이 발생하면 정상 파드도 Unhealthy로 처리되어 트래픽을 받지 못한다.<br>
> `/elb.php`는 아무 로직도 없는 빈 파일이라 즉시 200을 반환한다.<br>
> 헬스체크와 실제 서비스 로직을 분리하는 것은 이커머스처럼 복잡한 앱에서 필수적인 패턴이다.
{: .prompt-tip }

```nginx
server {
    listen 8080;
    server_name _;
    root /var/www/html/public;
    index index.php;

    # ALB 헬스체크 전용 (PHP 로직 없이 즉시 200)
    location = /elb.php {
        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # nginx 자체 헬스체크
    location = /nginx-health {
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # 세션 파일 직접 접근 차단
    location ^~ /data/ {
        deny all;
        return 403;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_X_FORWARDED_FOR $http_x_forwarded_for;
        fastcgi_read_timeout 60;
    }

    location ~* \.(sql|inc|log|java|pem|p12|bak)$ {
        deny all;
    }
}
```

### Nginx Dockerfile

```dockerfile
FROM nginx:1.24-alpine

COPY docker/nginx/nginx.conf /etc/nginx/nginx.conf
COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY docker/nginx/security.conf /etc/nginx/conf.d/security.conf

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/nginx-health || exit 1

EXPOSE 8080
USER nginx
```

### Tomcat Dockerfile

```dockerfile
FROM tomcat:9.0-jdk11-openjdk-slim

RUN rm -rf /usr/local/tomcat/webapps/*

COPY Sso.war /usr/local/tomcat/webapps/Sso.war
COPY server.xml /usr/local/tomcat/conf/server.xml
COPY context.xml /usr/local/tomcat/conf/context.xml

EXPOSE 8080
```

---

## 4. ECR 푸시

```bash
# ECR 로그인 (--username AWS 고정)
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com

# PHP-FPM
docker build -t <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-php:1.0.0 .
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-php:1.0.0

# Nginx
docker build -t <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nginx:1.0.0 .
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nginx:1.0.0

# Tomcat
docker build -t <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-tomcat:latest .
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-tomcat:latest
```

> ECR 로그인 시 `--username AWS`는 고정값이다. 본인 IAM 유저명을 쓰면 인증 실패한다.
{: .prompt-warning }

---

## 5. K8s 매니페스트 작성 (PHP 버전)

### initContainer + emptyDir 패턴을 선택한 이유

EKS에서 Sidecar 패턴으로 nginx + php-fpm을 같은 Pod에 올릴 때 핵심 문제가 있다. **nginx는 정적 파일을 직접 서빙하고 php-fpm은 PHP를 실행해야 하는데, 두 컨테이너가 동일한 소스코드에 접근해야 한다.**

로컬 docker-compose에서는 bind mount로 간단히 해결했지만 EKS에서는 불가능하다. 대안을 비교하면 다음과 같다.

| 방식 | 방법 | 단점 |
|---|---|---|
| 두 이미지에 소스 포함 | nginx 이미지, php 이미지 각각에 COPY | 소스 변경 시 두 이미지 모두 빌드 필요, 불일치 위험 |
| EFS 마운트 | PVC로 소스 공유 | EFS 의존성 추가, 레이턴시 발생 |
| **initContainer + emptyDir** | php 이미지에서 소스 복사 후 volume 공유 | **단일 이미지 빌드, 빠른 로컬 스토리지** ✅ |

`initContainer`는 메인 컨테이너 시작 전에 실행되고 종료되는 특수 컨테이너다. php 이미지에서 소스를 `emptyDir` volume으로 복사하면, nginx와 php-fpm이 동일한 volume을 마운트해 소스를 공유할 수 있다. 이미지는 php 하나만 빌드하면 되므로 소스 불일치 문제도 없다.

> **emptyDir volume을 두 개로 분리한 이유**<br>
> `php-socket`: nginx ↔ php-fpm 간 Unix socket 파일 공유 전용<br>
> `app-source`: 소스코드 공유 전용. initContainer가 복사한 전체 소스가 여기에 있다.<br>
> 두 volume을 분리해야 소켓 파일과 소스코드가 뒤섞이지 않고, 각 컨테이너가 필요한 경로에만 마운트할 수 있다.
{: .prompt-info }

> **`terminationGracePeriodSeconds: 60`을 설정한 이유**<br>
> Pod가 종료될 때 Kubernetes는 먼저 `preStop` 훅을 실행하고 `SIGTERM`을 보낸 뒤, 이 시간 안에 프로세스가 종료되지 않으면 `SIGKILL`로 강제 종료한다.<br>
> nginx는 `preStop`에서 `nginx -s quit`으로 현재 처리 중인 요청을 완료한 후 종료한다.<br>
> php-fpm도 처리 중인 요청이 있을 수 있으므로 충분한 유예 시간(60초)을 주어 응답 중단 없이 안전하게 종료되도록 했다.
{: .prompt-warning }

> **php-fpm Probe를 `exec`으로 구성한 이유**<br>
> nginx는 HTTP 엔드포인트(`/nginx-health`)가 있어 `httpGet`으로 Probe를 구성할 수 있다.<br>
> 반면 php-fpm은 FastCGI 프로토콜로 통신하므로 HTTP Probe를 직접 걸 수 없다.<br>
> 대신 Unix socket 파일(`/run/php/php7.2-fpm.sock`)의 존재 여부를 확인하는 방식을 사용했다.<br>
> socket 파일이 존재한다는 것은 php-fpm이 정상적으로 실행 중이며 요청을 받을 준비가 됐다는 의미다.
{: .prompt-info }

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <prefix>-web-apne2-deploy
  namespace: <namespace>
spec:
  replicas: 2
  selector:
    matchLabels:
      app: <prefix>-web
  template:
    metadata:
      labels:
        app: <prefix>-web
    spec:
      terminationGracePeriodSeconds: 60
      tolerations:
      - key: dedicated
        value: web
        effect: NoSchedule
      nodeSelector:
        role: web
      initContainers:
      - name: copy-source
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-php:1.0.0
        command: ['sh', '-c', 'cp -r /var/www/html/. /app/']
        volumeMounts:
        - name: app-source
          mountPath: /app
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
      containers:
      - name: nginx
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-nginx:1.0.0
        ports:
        - containerPort: 8080
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "nginx -s quit; sleep 5"]
        readinessProbe:
          httpGet:
            path: /nginx-health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /nginx-health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        volumeMounts:
        - name: php-socket
          mountPath: /run/php
        - name: app-source
          mountPath: /var/www/html
      - name: php-fpm
        image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/app-php:1.0.0
        envFrom:
        - secretRef:
            name: mysql-secret
        env:
        - name: APP_ENV
          value: "dev"
        readinessProbe:
          exec:
            command: ["sh", "-c", "test -S /run/php/php7.2-fpm.sock"]
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          exec:
            command: ["sh", "-c", "test -S /run/php/php7.2-fpm.sock && kill -0 1"]
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 3
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        volumeMounts:
        - name: php-socket
          mountPath: /run/php
        - name: app-source
          mountPath: /var/www/html
      volumes:
      - name: php-socket
        emptyDir: {}
      - name: app-source
        emptyDir: {}
```

---

## 6. External Secrets Operator 연동

Secrets Manager에 두 종류 시크릿이 있다.

- `<prefix>-mys-apne2-secret`: host, port, dbname (직접 생성)
- `rds!db-xxxxxxxx-...`: username, password (RDS 자동 생성, 7일마다 교체)

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mysql-secret
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: mysql-secret
    creationPolicy: Owner
  data:
  - secretKey: DB_HOST
    remoteRef:
      key: <prefix>-mys-apne2-secret
      property: host
  - secretKey: DB_PORT
    remoteRef:
      key: <prefix>-mys-apne2-secret
      property: port
  - secretKey: DB_NAME
    remoteRef:
      key: <prefix>-mys-apne2-secret
      property: dbname
  - secretKey: DB_USERNAME
    remoteRef:
      key: 'arn:aws:secretsmanager:ap-northeast-2:<account-id>:secret:rds!db-xxxxxxxx-...'
      property: username
  - secretKey: DB_PASSWORD
    remoteRef:
      key: 'arn:aws:secretsmanager:ap-northeast-2:<account-id>:secret:rds!db-xxxxxxxx-...'
      property: password
```

> ESO가 이를 읽어 K8s Secret `mysql-secret`을 생성하고, Deployment에서 `envFrom.secretRef`로 Pod 환경변수로 주입된다.<br>
> 결과적으로 `$_SERVER['DB_HOST']` 등으로 접근 가능하다.
{: .prompt-info }

---

## 7. external-dns + Route53 + ACM 연동

### Route53 호스팅 존 + ACM 인증서

```hcl
resource "aws_route53_zone" "main" {
  name = "<your-domain>"
}

resource "aws_acm_certificate" "main" {
  domain_name       = "*.<your-domain>"
  validation_method = "DNS"
  depends_on        = [aws_route53_zone.main]
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options :
    dvo.resource_record_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  zone_id = aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}
```

> `for_each` 키를 `resource_record_name`으로 해야 와일드카드/루트 도메인 검증 레코드 중복 충돌이 없다.
{: .prompt-tip }

apply 후 출력된 NS 레코드 4개를 도메인 등록기관 네임서버로 등록한다.

### external-dns EKS Addon

```hcl
resource "aws_eks_addon" "external_dns" {
  cluster_name                = aws_eks_cluster.this.name
  addon_name                  = "external-dns"
  resolve_conflicts_on_update = "OVERWRITE"
  configuration_values = jsonencode({
    policy = "sync"
    tolerations = [{ key = "dedicated", value = "system", effect = "NoSchedule", operator = "Equal" }]
    nodeSelector = { "role" = "system" }
  })
}

resource "kubernetes_annotations" "external_dns_sa" {
  api_version = "v1"
  kind        = "ServiceAccount"
  metadata {
    name      = "external-dns"
    namespace = "external-dns"
  }
  annotations = {
    "eks.amazonaws.com/role-arn" = module.external_dns.role_arn
  }
  depends_on = [aws_eks_addon.external_dns]
}
```

> external-dns addon의 schema에서 `serviceAccount` 필드를 직접 지원하지 않는다.<br>
> `kubernetes_annotations` 리소스로 따로 달아야 한다.
{: .prompt-warning }

---

## 8. RDS 생성 및 DB 마이그레이션

```hcl
resource "aws_db_instance" "this" {
  identifier        = "<prefix>-mys-apne2-rds"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  storage_type      = "gp3"
  storage_encrypted = true
  db_name           = "<dbname>"

  manage_master_user_password = true  # Secrets Manager 자동 관리, 7일마다 교체

  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  multi_az               = false
  publicly_accessible    = false
  skip_final_snapshot    = true
}
```

기존 RDS에서 덤프 후 새 RDS에 import:

```bash
# 기존 RDS 덤프
mysqldump -h <old-endpoint> -u admin -p \
  --databases <db1> <db2> \
  --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  > /tmp/dump.sql

# S3 경유해서 VSCode Server로 이동
aws s3 cp /tmp/dump.sql s3://<bucket>/backup/
aws s3 cp s3://<bucket>/backup/dump.sql /tmp/

# 새 RDS에 import
mysql -h <new-endpoint> -u admin -p < /tmp/dump.sql
```

---

## 9. 소스코드 수정 사항

레거시 PHP 앱을 EKS 환경에서 동작시키기 위한 소스 수정들이다.

### lib.awsSecretsHelper.php

기존 Redis 캐싱 + EC2 IMDS 방식에서, ESO가 환경변수로 주입하는 방식으로 단순화했다.

```php
<?php
class AwsSecretsHelper {
    public function getSecret($update = false) {
        $creds = [
            'host'     => $_SERVER['DB_HOST']     ?? $_ENV['DB_HOST']     ?? null,
            'port'     => $_SERVER['DB_PORT']     ?? $_ENV['DB_PORT']     ?? 3306,
            'username' => $_SERVER['DB_USERNAME'] ?? $_ENV['DB_USERNAME'] ?? null,
            'password' => $_SERVER['DB_PASSWORD'] ?? $_ENV['DB_PASSWORD'] ?? null,
        ];
        if (empty($creds['host']) || empty($creds['username'])) {
            throw new Exception("DB credentials not found in environment.");
        }
        return $creds;
    }
}
```

### config/bootstrap.php

Doctrine `EntityManager::create()` 코드 제거. 요청마다 DB 연결을 시도해서 타임아웃을 유발했다.

```php
// 제거
$em = EntityManager::create($conn, $config);
$helper = new AwsSecretsHelper();
$creds  = $helper->getSecret();
$conn   = array(...);
```

### public/index.php

```php
<?php
ob_start();  // 라이브러리 파일 ?> 뒤 개행이 HTTP 응답 바디로 나가는 것 방지

$_SERVER['APP_ENV'] = $_SERVER['APP_ENV'] ?? 'prod';  // 환경변수 우선, 없으면 prod 폴백
```

### _config.php — hostname 분기 제거

```php
// Before — EC2 hostname 하드코딩 (EKS에서 Pod마다 hostname이 달라짐)
$_cfg['is_test'] = (gethostname() === 'ip-10-xx-xx-xxx');

// After — 환경변수 기반
$_cfg['is_test'] = ($_SERVER['APP_ENV'] === 'dev');
```

### .htaccess 파일들 — php_value 제거

```apache
# 제거 (mod_php 전용, php-fpm에서 사용 시 500)
php_value post_max_size 32M
php_value upload_max_filesize 32M
php_value max_input_vars 3000
php_value session.cache_expire 120
php_value session.cookie_lifetime 7200
php_value session.gc_maxlifetime 7200
```

> `php_value`, `php_flag`는 Apache mod_php 전용 디렉티브다. PHP-FPM 환경에서는 nginx의 `fastcgi_param PHP_VALUE`로 처리하거나 `www.conf`에서 처리한다.
{: .prompt-warning }

---

## 10. PHP 앱 트러블슈팅

PHP 앱 배포 후 발생한 500 에러들의 원인과 해결 과정이다.

### Pod Pending — Karpenter taint 미설정

```
0/2 nodes are available: 2 node(s) had untolerated taint(s).
```

| 항목 | 내용 |
|---|---|
| 원인 | Karpenter nodepool에 `dedicated=web:NoSchedule` taint가 있는데 Deployment에 toleration 없음 |
| 해결 | `tolerations` + `nodeSelector` 추가 |

```yaml
tolerations:
- key: dedicated
  value: web
  effect: NoSchedule
nodeSelector:
  role: web
```

---

### nginx 404 — PHP-FPM 소켓 권한 문제

| 항목 | 내용 |
|---|---|
| 원인 | PHP-FPM 소켓이 `0660` 권한으로 생성되어 nginx 유저가 접근 불가 |
| 해결 | `www.conf`에서 소켓 권한 변경 |

```ini
listen.mode = 0666
listen.owner = www-data
listen.group = www-data
```

---

### ALB 헬스체크 Unhealthy

| 항목 | 내용 |
|---|---|
| 원인 | 헬스체크 경로 기본값 `/`가 PHP 앱 로직을 타서 500 반환 |
| 해결 | Ingress annotation으로 헬스체크 경로 변경 |

```yaml
alb.ingress.kubernetes.io/healthcheck-path: /elb.php
```

`/elb.php`는 빈 파일이라 200을 즉시 반환한다.

---

### PHP-FPM 환경변수 미전달 — clear_env

```
DB credentials not found in environment.
```

| 항목 | 내용 |
|---|---|
| 원인 | `www.conf`의 `clear_env = yes` (기본값)가 환경변수 초기화 |
| 해결 | `clear_env = no` 설정 |

---

### short_open_tag Off로 인한 include 체인 실패

```
Class 'Template' not found in /var/www/html/public/index.php:72
```

| 항목 | 내용 |
|---|---|
| 원인 | `inc/_common.php`가 `<?`(short tag)로 시작하는데 `short_open_tag = Off` → include 체인 전체가 끊김 |
| 해결 | 전체 PHP 파일 첫 줄 변환 또는 php.ini에서 활성화 |

```bash
# 전체 PHP 파일 첫 줄 <? → <?php 변환
find public/ -name "*.php" -exec sed -i '1s/^<?$/<?php/' {} \;
```

---

### PHP 파일 ?> 제거로 인한 Parse Error

```
PHP Parse error: syntax error, unexpected '<' in maintenance.php on line 74
```

| 항목 | 내용 |
|---|---|
| 원인 | PHP와 HTML이 혼재하는 파일에서 `?>` 제거 시 닫기 태그가 없어진 상태에서 HTML이 PHP 코드로 인식됨 |
| 해결 | PHP 블록이 끝나고 HTML이 시작되는 지점에 `?>` 복원 |

> PHP와 HTML이 섞인 파일에서는 `?>` 제거 금지. 순수 PHP 파일(클래스, 라이브러리)에서만 제거하는 게 안전하다.
{: .prompt-danger }

---

### APP_ENV 하드코딩

| 항목 | 내용 |
|---|---|
| 원인 | `index.php`에 `$_SERVER['APP_ENV'] = 'prod'`가 하드코딩되어 Pod 환경변수 `APP_ENV=dev`가 무시됨 |
| 해결 | 환경변수 우선, 없으면 폴백 |

```php
// Before
$_SERVER['APP_ENV'] = 'prod';

// After
$_SERVER['APP_ENV'] = $_SERVER['APP_ENV'] ?? 'prod';
```

---

### 출력 버퍼링 미적용으로 헤더 전송 문제

| 항목 | 내용 |
|---|---|
| 원인 | PHP 라이브러리 파일들의 `?>` 뒤 `\r\n` 개행이 HTTP 응답 바디로 출력 → 헤더가 이미 전송된 상태 → 세션 설정 실패 → include 체인 동작 불가 |
| 해결 | `index.php` 맨 위에 `ob_start()` 추가 |

```php
<?php
ob_start();
```

---

### Doctrine EntityManager — DB 타임아웃

```
"GET /index.php" 500 /var/www/html/public/index.php 48.726sec 2MB
```

| 항목 | 내용 |
|---|---|
| 원인 | `bootstrap.php`에 남아있던 `EntityManager::create()` 코드가 요청마다 실제 DB 연결 시도 → 타임아웃 |
| 해결 | `bootstrap.php`에서 DB 연결 코드 전체 제거. DB 연결은 `_db.php`에서만 |

---

### .env 파일의 구 DATABASE_URL

| 항목 | 내용 |
|---|---|
| 원인 | `.env`에 `DATABASE_URL`이 기존 RDS(다른 VPC)로 하드코딩되어 연결 불가 |
| 해결 | `.env`에서 `DATABASE_URL` 제거 후 재빌드 |

```bash
sed -i '/^DATABASE_URL=/d' .env
```

> 환경파일에 DB 자격증명이나 엔드포인트를 절대 커밋하지 않는다. Secrets Manager + ESO 패턴으로 런타임에 주입한다.
{: .prompt-danger }

---

## 11. Next.js 재구현 결정

### 결정 배경

PHP 앱의 500 에러들을 하나씩 해결해 나갔지만, 레거시 코드 특성상 문제가 끊이지 않았다. `DATABASE_URL`, `?>` 개행 문제, `short_open_tag`, `ob_start()` 등 레거시 특유의 문제들이 연쇄적으로 발생했다.

결국 **인프라 테스트가 주목적이었으므로**, 핵심 페이지만 Next.js로 재구현하는 것이 현실적이라고 판단했다.

```
infra-repo/
└── applications/
    ├── purina-nextjs/      # Next.js 14 App Router (BFF)
    └── purina-api/         # Node.js/Express 상품 카탈로그 API
```

---

## 12. Next.js 애플리케이션 구현

### 기술 스택

| 항목 | 선택 | 버전 |
|---|---|---|
| 프레임워크 | Next.js App Router | 14.2 |
| DB 클라이언트 | mysql2/promise | 3.x |
| 세션 | iron-session | 8.x |
| 비밀번호 | bcryptjs | 2.x |
| 스타일링 | Tailwind CSS | 3.x |
| 빌드 출력 | standalone | - |

### 디렉토리 구조

```
applications/purina-nextjs/
├── Dockerfile
├── next.config.mjs
├── package.json
└── src/
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx                # 메인 (인기상품, 카테고리)
    │   ├── login/page.tsx
    │   ├── register/page.tsx
    │   ├── products/
    │   │   ├── page.tsx            # 상품 목록 (카테고리 필터, 페이지네이션)
    │   │   └── [id]/page.tsx       # 상품 상세 + 장바구니 담기
    │   ├── cart/
    │   │   ├── page.tsx
    │   │   └── CartItems.tsx       # Client Component
    │   └── api/
    │       ├── health/route.ts     # K8s Probe 엔드포인트
    │       ├── auth/
    │       │   ├── login/route.ts
    │       │   ├── logout/route.ts
    │       │   └── register/route.ts
    │       └── cart/
    │           ├── route.ts
    │           └── [id]/route.ts
    ├── components/
    │   ├── Header.tsx
    │   └── ProductCard.tsx
    └── lib/
        ├── db.ts                   # mysql2 커넥션 풀
        ├── session.ts              # iron-session 설정
        └── apiClient.ts            # purina-api HTTP 클라이언트
```

### DB 커넥션 풀

```typescript
const pool = mysql.createPool({
  host:     process.env.DB_HOST,
  port:     parseInt(process.env.DB_PORT || "3306"),
  database: process.env.DB_NAME,
  user:     process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  waitForConnections: true,
  connectionLimit: 10,
  charset: "utf8mb4",
  timezone: "+09:00",
});
```

### 세션 관리 (iron-session v8)

AES 암호화 쿠키 기반 서버사이드 세션. `SESSION_SECRET`는 Secrets Manager → ESO → Pod 환경변수로 주입.

```typescript
export const sessionOptions: SessionOptions = {
  password: process.env.SESSION_SECRET || "fallback_32chars_min_required!!!",
  cookieName: "app_sess",
  cookieOptions: {
    secure: process.env.NODE_ENV === "production",
    httpOnly: true,
    maxAge: 60 * 60 * 24,  // 24시간
  },
};
```

### 로그인 API — 기존 PHP DB 비밀번호 포맷 지원

기존 PHP DB의 비밀번호 해시가 3종 혼재하므로 모두 지원했다.

```typescript
// bcrypt: PHP $2y$ → Node.js $2b$ 정규화
if (hash.startsWith("$2y$") || hash.startsWith("$2b$")) {
  const normalizedHash = hash.replace(/^\$2y\$/, "$2b$");
  valid = await bcrypt.compare(pass, normalizedHash);
}
// MD5 (레거시 회원)
else if (hash.length === 32) {
  valid = crypto.createHash("md5").update(pass).digest("hex") === hash;
}
// 평문 (관리자 테스트 계정)
else {
  valid = hash === pass;
}
```

### 헬스체크 API

K8s Readiness/Liveness Probe 겸용. DB SELECT 1로 연결 상태 포함.

```typescript
export async function GET() {
  try {
    await pool.query("SELECT 1");
    return NextResponse.json({ status: "ok", db: "connected" });
  } catch {
    return NextResponse.json({ status: "error", db: "disconnected" }, { status: 503 });
  }
}
```

### Dockerfile (멀티스테이지)

```dockerfile
FROM node:20-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs \
 && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:3000/api/health || exit 1
CMD ["node", "server.js"]
```

### next.config.mjs

```javascript
const nextConfig = {
  output: "standalone",
  experimental: {
    // mysql2는 native binding 사용 → 번들러에서 제외
    serverComponentsExternalPackages: ["mysql2", "bcryptjs"],
  },
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "**" },
      { protocol: "http", hostname: "**" },
    ],
  },
};
export default nextConfig;
```

---

## 13. MSA 분리 — purina-api 서비스

### 분리 배경

상품/카탈로그 도메인을 별도 서비스로 분리해 실질적인 서비스 간 통신을 검증하기로 했다.

| 도메인 | 담당 서비스 | 이유 |
|---|---|---|
| 상품 조회 / 카테고리 | purina-api | 읽기 전용, 인증 불필요, 독립 스케일 아웃 가능 |
| 회원 인증 / 세션 | purina-nextjs | 쿠키·세션과 직접 결합, BFF에서 처리가 자연스러움 |
| 장바구니 | purina-nextjs | 로그인 상태 확인 후 처리, 세션과 결합 |

### API 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/health` | DB 연결 포함 헬스체크 |
| GET | `/api/categories` | 전체 카테고리 목록 |
| GET | `/api/products` | 상품 목록 (category, page, limit, featured 쿼리) |
| GET | `/api/products/:id` | 상품 상세 |

### Dockerfile (단일스테이지)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["node", "src/app.js"]
```

### Next.js → purina-api 통신

Next.js Server Component에서 K8s 내부 DNS로 호출.

```typescript
// src/lib/apiClient.ts
const API_URL = process.env.PURINA_API_URL || "http://localhost:8080";

async function apiFetch<T>(path: string): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, { next: { revalidate: 0 } });
  if (!res.ok) throw new Error(`API error ${res.status}: ${path}`);
  return res.json();
}
```

`PURINA_API_URL=http://<api-service-name>:8080`으로 K8s 내부 DNS 서비스 디스커버리.

**변경된 데이터 흐름**

```
# Before (Next.js → DB 직접)
page.tsx → mysql2 pool → RDS MySQL

# After (Next.js → purina-api → DB)
page.tsx → apiClient.ts → HTTP → purina-api → mysql2 pool → RDS MySQL
```

---

## 14. K8s 매니페스트 최종

### Ingress (최종)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app>-ingress
  namespace: <namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/load-balancer-name: <alb-name>
    external-dns.alpha.kubernetes.io/hostname: <your-domain>
    alb.ingress.kubernetes.io/healthcheck-path: /api/health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
spec:
  rules:
  - host: <your-domain>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <nextjs-service-name>
            port:
              number: 80
```

---

## 15. 최종 아키텍처

```
인터넷
  │
  ▼ HTTPS
Route53 (<your-domain>)
  │ external-dns가 ALB DNS를 A 레코드로 자동 등록/삭제
  ▼
ALB
  ├── HTTP:80 → HTTPS:443 리다이렉트
  └── HTTPS:443 → 타겟그룹 (Pod IP:3000)
          ↓
  EKS Ingress (ALBC 관리)
          ↓
  purina-nextjs Service (ClusterIP :80 → Pod :3000)  [2 replicas, web nodepool]
    │
    ├── 상품/카탈로그 조회
    │     └─ HTTP → purina-api Service (ClusterIP :8080 → Pod :8080)  [2 replicas]
    │                   └─ mysql2 pool → RDS MySQL 8.0
    │
    ├── 회원 인증 / 세션 (iron-session AES 쿠키)
    │     └─ mysql2 pool → RDS MySQL 8.0
    │
    └── 장바구니 CRUD
          └─ mysql2 pool → RDS MySQL 8.0

AWS Secrets Manager
    └─ ESO (1시간 주기) → K8s Secret → Pod 환경변수
```

### Secrets Manager 구조

| 시크릿 | 관리 주체 | 키 | 용도 |
|---|---|---|---|
| `rds!db-xxxxxxxx-...` (ARN) | RDS 자동 | username, password | DB 인증 정보, 7일마다 교체 |
| `<prefix>-mys-apne2-secret` | Terraform | host, port, dbname | DB 연결 정보 |
| `<prefix>-nextjs-apne2-secret` | Terraform | session_secret | Next.js 세션 암호화 키 |

---

## 16. Terraform 변경사항

### Next.js Session Secret 리소스 추가

```hcl
# versions.tf에 random provider 추가
random = {
  source  = "hashicorp/random"
  version = "~> 3.0"
}

# main.tf — Next.js 시크릿
resource "random_password" "nextjs_session_secret" {
  length  = 64
  special = false
}

resource "aws_secretsmanager_secret" "nextjs" {
  name = "<prefix>-nextjs-apne2-secret"
}

resource "aws_secretsmanager_secret_version" "nextjs" {
  secret_id = aws_secretsmanager_secret.nextjs.id
  secret_string = jsonencode({
    session_secret = random_password.nextjs_session_secret.result
  })
}
```

> Terraform이 64자리 랜덤 비밀번호를 생성 → Secrets Manager 저장 → ESO가 읽어서 Pod에 `SESSION_SECRET` 환경변수로 주입한다.
{: .prompt-tip }

---

## 17. 배포 흐름

### Terraform

```bash
cd terraform/envs/dev
terraform init
terraform plan
terraform apply   # Secrets Manager에 nextjs 시크릿 생성
```

### ECR 이미지 빌드 & 푸시

```bash
# ECR 인증
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com

# Next.js
cd applications/purina-nextjs
docker build -t <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-nextjs:1.0.0 .
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-nextjs:1.0.0

# purina-api
cd applications/purina-api
docker build -t <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-api:1.0.0 .
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/purina-api:1.0.0
```

### K8s 매니페스트 적용

```bash
# ESO 시크릿 동기화
kubectl apply -f kubernetes/external-secrets/

# 앱 배포 (API 먼저 — Next.js가 의존)
kubectl apply -f kubernetes/apps/deployment-api.yaml
kubectl apply -f kubernetes/apps/deployment-nextjs.yaml

# Ingress
kubectl apply -f kubernetes/ingress/ingress.yaml

# 상태 확인
kubectl get pods -n <namespace>
kubectl get externalsecret -n <namespace>
kubectl get ingress -n <namespace>
```

### 접속 확인

| 경로 | 기능 | 데이터 소스 |
|---|---|---|
| `/` | 메인 (인기상품, 카테고리) | purina-api |
| `/products` | 상품 목록 (필터, 페이지네이션) | purina-api |
| `/products/[id]` | 상품 상세 + 장바구니 담기 | purina-api |
| `/register` | 회원가입 | purina-nextjs → RDS |
| `/login` | 로그인 (bcrypt/MD5/평문 지원) | purina-nextjs → RDS |
| `/cart` | 장바구니 (로그인 필요) | purina-nextjs → RDS |
| `/api/health` | 헬스체크 (DB 연결 상태 포함) | purina-nextjs |

---

## 18. 다음 단계

> 1. **ArgoCD Application 설정** — GitOps 배포 자동화<br>
> 2. **GitHub Actions CI 구성** — 이미지 빌드 자동화<br>
> 3. **HPA 설정** — CPU 기반 파드 오토스케일링<br>
> 4. **EFS CSI Driver** — 업로드 파일 영구 스토리지
{: .prompt-info }
