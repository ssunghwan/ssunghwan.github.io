---
title: "EFS를 지웠더니 Disk가 죽었다."
date: 2026-01-26 01:16:00 +0900
categories: [Cloud & Infrastructure, Troubleshooting]
tags: [aws, efs, disk, stunnel]
---

안녕하세요? 해당 챕터에서는 저의 휴먼 에러로 발생한 EFS 관련 장애에 대해 다뤄볼 예정입니다.

---
**이번 글 요약**
> 사용 중이던 EFS 파일시스템 삭제 이후, 인스턴스의 stunnel 기반 EFS 마운트가 DNS 이름을 계속 조회하며 syslog에 초당 수백 줄의 에러 로그를 무한 기록.<br>
> `/var/log/syslog`가 수 십GB로 폭증 → 루트 디스크 100% → sudo, snapd, SSM Agent, vim, systemd-resolved까지 연쇄적으로 장애.<br>
> 디스크 공간 즉시 확보 → EFS 재시도 차단 → DNS 복구 → snapd/SSM 복구 순서로 해결.
{: .prompt-info }

---

## 01. 증상 & 현상

프라이빗한 서버에 접근하기 위해 기존에 Client VPN Endpoint를 사용하여 SSH로 접근을 하였지만, 보안성 강화를 위해 AWS에서 제공하는 System Session Manager를 사용하기로 하였고 기존에 있는 VPN은 폐기하기로 하였습니다.

다행히도 SSM을 도입하기 전에 이 이슈를 직면하게 되었고 VPN으로 접근하여 해결을 한 경험이 있습니다.

AWS Backup 서비스를 이용하고 있었으므로, 실제 서비스는 EBS를 활용하여 복구를 해놓은 상태에서 원인 파악을 위해 뜯어보았고 그 경험을 공유드리고자 합니다.

**📌 SSM으로 인스턴스에 접근이 불가능**

- AWS Console에서 인스턴스의 Session Manager로 접근이 불가능한 상태
- SSM에서 Fleet Manager로 에이전트 상태 확인 결과, 연결 끊김
- Run command로 에이전트 상태를 확인하려고 하였으나 명령 실패 → 에이전트 사망

**📌 VPN을 활용하여 내부 서버에 SSH로 접속하여 확인**

- SSM이 snap으로 설치되어 있었는데, snap 명령 입력 시 서버 먹통되는 현상
  - Agent뿐만 아니라 snap 자체가 안 되는 거면 서버에 문제가 있다고 판단
- sudo 명령 입력 시 : `unable to resolve host <hostname>: Resource temporarily unavailable`
- vi 편집기 확인 시 : `E297: Write error in swap file`
- 디스크 용량 확인 시 : 루트 디렉토리에서 Disk 100% 사용

뭔가 이상하다.. Disk 용량을 많이 잡아먹을 녀석이 없는데? 설마 해서 로그 확인.

```bash
sudo du -sh /var/* 2>/dev/null
# 50G    /var/log   ← 폭발

ls -lh /var/log/syslog
# -rw-r----- 1 syslog adm 50G ... syslog
```

**📌 반복 로그 패턴**

```
stunnel: LOG3: Error resolving "fs-1234abcdefg.efs.ap-northeast-2.amazonaws.com": Neither nodename nor servname known (EAI_NONAME)
stunnel: LOG3: No remote host resolved
```

**삭제된 EFS**로 계속 마운트 시도 → **DNS resolve 실패**를 무한 재시도 → syslog 폭증 → 디스크 100% → 시스템 마비

---

## 02. 원인 (Root Cause)

1. 사용 중이던 EFS를 삭제했으나 인스턴스에 남아있던 **stunnel 기반 EFS 마운트 구성**(service, scripts, fstab)이 삭제된 EFS의 DNS를 계속 조회.
2. 그 결과 stunnel이 초당 수십~수백 회 "DNS 이름 해결 실패" 로그를 **syslog**에 적재 → `/var/log/syslog` 단일 파일이 **50GB**로 팽창.
3. **root disk가 100%** 가 발생하자, 하기와 같은 현상 발생:
   - sudo가 실행 전 hostname lookup에서 실패 메시지 반복
   - vim swap 생성 실패
   - snapd가 state 파일을 쓸 수 없어 **activating/lock** 상태
   - systemd-resolved의 **stub resolv.conf**가 깨지면서 **DNS 자체 마비**
   - **SSM Agent**가 `RequestError: send request failed`로 백엔드와 통신 실패 → 콘솔에서 Offline

---

## 03. 복구 절차

### 3-1. Disk "쓰기 없이" 즉시 확보

disk가 0byte 상태에서는 쓰기 명령이 실패하므로 삭제로 빠르게 공간을 확보. sudo 명령을 쓰지 않기 위해 root 계정으로 진행.

```bash
# APT 캐시 제거 (몇백 MB~수 GB)
rm -rf /var/cache/apt/archives/*

# snap 캐시 제거
rm -rf /var/lib/snapd/cache/*

# journald 파일 삭제
rm -rf /var/log/journal/*

# 회전된 로그 일괄 삭제
rm -f /var/log/*.gz
rm -f /var/log/*.[0-9]
rm -f /var/log/*.[0-9].gz

# 공간 확보 확인
df -h
```

### 3-2. EFS로 마운트 재시도 차단

```bash
# stunnel4 중지/비활성화
sudo systemctl stop stunnel4
sudo systemctl disable stunnel4

# 잔여 런타임 설정 제거
sudo rm -rf /run/efs/* /var/run/efs/* 2>/dev/null || true

# fstab 확인 및 주석 상태 확인
cat /etc/fstab
# 예) #fs-1234...:/ /mnt/efs efs tls,_netdev 0 0
```

### 3-3. hostname 꼬임 증상 해결

```bash
cat /etc/hostname
# 예: ip-172-30-10-10

# 편집기 없이 재작성 (root로)
cat > /etc/hosts <<EOF
127.0.0.1   localhost
127.0.1.1   ip-172-30-10-10
EOF
```

더 이상 "unable to resolve host" 메시지가 없어야 정상.

### 3-4. DNS 복구

```bash
# systemd-resolved가 만든 stub 링크가 깨져있다면:
# /etc/resolv.conf → ../run/systemd/resolve/stub-resolv.conf (대상 없음)

# resolv.conf 재생성 (AWS 기본 DNS)
sudo rm -f /etc/resolv.conf
echo "nameserver 169.254.169.253" | sudo tee /etc/resolv.conf
echo "options timeout:1 attempts:1" | sudo tee -a /etc/resolv.conf

# resolved 재기동
sudo systemctl restart systemd-resolved

# 확인
nslookup amazon.com
nslookup ssm.ap-northeast-2.amazonaws.com
nslookup ssmmessages.ap-northeast-2.amazonaws.com
```

`/etc/resolv.conf` 링크가 파손되어, 고정으로 AWS default nameserver 입력. resolved 재기동 후 SSM 관련 endpoint들도 주소를 잘 불러오나 확인 필수.

### 3-5. Snapd & SSM Agent 복구

```bash
# snapd 상태 확인/재시작
snap version
sudo systemctl restart snapd snapd.socket

# 잠금/상태파일 꼬임이 의심되면(필요 시):
sudo pkill -9 snapd
sudo rm -f /var/lib/snapd/state.lock /var/lib/snapd/snaps.state /var/lib/snapd/seed/seed.yaml.lock
sudo systemctl restart snapd

# SSM runtimeconfig 손상 정리 후 재시작
sudo rm -rf /var/lib/amazon/ssm/* /etc/amazon/ssm/* 2>/dev/null || true
snap restart amazon-ssm-agent

# SSM 활성 확인
snap services amazon-ssm-agent
# Current: active

# 로그 확인 (연결 성공/에러 여부)
tail -n 100 /var/log/amazon/ssm/amazon-ssm-agent.log
```

정상화 후 더 이상 "RequestError: send request failed" 메시지가 발생하지 않아야 하며, SSM Fleet Manager에서 인스턴스 에이전트가 Online으로 전환되어야 한다.

---

## 04. 재발 방지

**1) EFS 삭제 전 체크리스트**

해당 EFS를 마운트한 모든 인스턴스의 아래 항목을 확실히 제거 또는 비활성화:
- `/etc/fstab`
- `/etc/init.d/stunnel4` (또는 관련 서비스 단위 파일)
- 사용자 정의 마운트 스크립트

**2) 로그 폭발 방지**

- `/etc/rsyslog.conf` 또는 journald 설정으로 **rate limit** 활성화
  - `/etc/systemd/journald.conf` → `RateLimitIntervalSec=` / `RateLimitBurst=` 조정
- logrotate 주기를 확인하고 `/var/log/syslog`의 최대 크기를 관리

**3) DNS 안정화**

- EC2에서는 `/etc/resolv.conf`에 **nameserver 169.254.169.253 직접 설정** (특히 Ubuntu 18.04)
- systemd-resolved 상태가 비정상일 경우 symlink 방식 대신 **고정 nameserver** 권장

**4) 모니터링**

- CloudWatch Agent 또는 node-exporter로 **디스크 사용률 / /var/log 사이즈** 알람 설정
- `/var/log/syslog` 파일 크기 임계치 알림
- SSM Managed Instances 상태(온라인/오프라인) 알람

**5) Snapd 안정화**

- snap이 꼬일 때는 **lock/state 삭제 → service restart** 패턴으로 복구
- 가능하면 시스템 핵심 에이전트(SSM)는 `.deb` 패키지로 고정 운용도 고려

---

EFS를 삭제하면서 `/etc/fstab`에서 해당 내용을 지우는 것을 바쁜 나머지 잊고.. 저의 휴먼 에러로 발생한 이슈였습니다..!
