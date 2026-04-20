---
title: "DMARC로 이메일 발송 변조를 예방하기"
date: 2026-03-07 03:49:00 +0900
categories: [Compliance & Vulnerability, Application & Network]
tags: [dmarc, spf, dkim, email security]
---

> DNS 기반 이메일 보안 메커니즘을 실무 관점에서 정리한 가이드입니다.<br>
> 대상 독자: 클라우드 인프라 엔지니어, DevOps, 도메인/메일 운영 담당자
{: .prompt-info }

---

## 01. 배경

SMTP(Simple Mail Transfer Protocol)는 1982년에 설계되었습니다. 당시에는 보안보다 연결성이 우선이었기 때문에, **발신자를 검증하는 메커니즘이 프로토콜에 내장되지 않았습니다.**

즉, 누구든지 기술적으로는 아래처럼 From: 헤더를 위조할 수 있습니다.

```
From: no-reply@trusted-bank.com
To: victim@example.com
Subject: 긴급 계정 확인 요청
```

이 구조적 취약점을 악용한 것이 **피싱(Phishing)** 과 **스푸핑(Spoofing)** 입니다.

| **위협 유형** | **설명** | **예시** |
| --- | --- | --- |
| 도메인 스푸핑 | 신뢰 도메인 위조 | @samsung.com 사칭 발신 |
| 피싱 | 위조 메일로 자격증명 탈취 | 가짜 로그인 링크 유도 |
| BEC (Business Email Compromise) | 임원 사칭 송금 지시 | CEO/CFO 사칭 |
| 이메일 중간자 변조 | 전송 중 내용 수정 | 계좌번호 교체 |

이 네 가지 위협을 각각 대응하기 위해 등장한 것이 **SPF, DKIM, DMARC** 입니다.

---

## 02. TXT 레코드

TXT 레코드는 DNS의 범용 텍스트 저장소입니다. 원래는 사람이 읽는 메모 용도로 설계되었지만, 현재는 다음 용도로 광범위하게 사용됩니다.

| **용도** | **예시 값** |
| --- | --- |
| SPF 발신 서버 선언 | v=spf1 include:... |
| DKIM 공개키 등록 | v=DKIM1; k=rsa; p=MIGf... |
| 도메인 소유권 인증 | google-site-verification=... |
| DMARC 정책 선언 | v=DMARC1; p=reject; rua=... |

> **중요한 특성**<br>
> - 하나의 도메인에 TXT 레코드를 여러 개 등록할 수 있습니다.<br>
> - 단, SPF는 도메인당 하나만 유효합니다 (여러 개 있으면 PermError 발생).<br>
> - TXT 레코드 자체는 단순한 문자열 컨테이너이며, 보안 기능은 그 값을 해석하는 수신 서버 측에서 처리합니다.
{: .prompt-warning }

---

### 2-1. SPF 란?

> "우리 도메인을 대신해서 메일을 보낼 수 있는 서버(IP)가 어디인지" DNS에 공개 선언하는 메커니즘<br>
> RFC 7208로 표준화되어 있으며, 수신 서버가 발신 IP를 DNS에서 조회해 허가 여부를 판단합니다.
{: .prompt-tip }

** 동작 원리**

```
발신 흐름:
  메일 서버 (IP: 203.0.x.x) → From: notice@example.com 로 메일 발송

수신 서버 검증 흐름:
  1. From 헤더의 도메인 추출: example.com
  2. DNS 조회: example.com TXT 레코드
  3. "v=spf1 include:spf.example.com ~all" 발견
  4. spf.example.com 내부에 203.0.x.x 포함 여부 확인
  5. 포함되면 → SPF PASS
     포함 안되면 → SPF SOFTFAIL (~all) 또는 HARDFAIL (-all)
```

** SPF 레코드 문법**

```
v=spf1 [메커니즘...] [all]
```

** 주요 메커니즘**

| 메커니즘 | 설명 | 예시 |
| --- | --- | --- |
| include: | 외부 SPF 레코드 참조 | include:spf.example.com |
| ip4: | 특정 IPv4 허가 | ip4:203.0.113.0/24 |
| ip6: | 특정 IPv6 허가 | ip6:2001:db8::/32 |
| a: | 해당 도메인 A 레코드 IP 허가 | a:mail.example.com |
| mx: | MX 레코드 IP 허가 | mx |

**Qualifier(판정)**

| **기호** | **의미** | **결과** |
| --- | --- | --- |
| + (기본값) | Pass | 정상 발신 |
| ~ | SoftFail | 의심, 스팸 마킹 |
| - | Fail | 명시적 거부 |
| ? | Neutral | 판정 보류 |

**실제 예시 (all 수식어 비교)**

```bash
# 느슨한 설정 (초기 단계 권장)
v=spf1 include:spf.example.com ~all

# 엄격한 설정 (정착 후 권장)
v=spf1 include:spf.example.com -all
```

> ** SPF의 한계**<br>
> SPF는 봉투 발신자(Envelope From, Return-Path)의 IP만 검증합니다.<br>
> 사용자가 메일 클라이언트에서 보는 From: 헤더(Header From)와는 별개입니다.<br>
> 이 간극을 이용한 공격(Display Name Spoofing)은 SPF만으로 막을 수 없으며, DMARC의 **Alignment 검사**가 이를 보완합니다.
{: .prompt-warning }

> ** DNS Lookup 제한**<br>
> SPF 평가 시 include:, a:, mx: 등의 DNS 조회는 **최대 10회**로 제한됩니다 (RFC 7208 §4.6.4).<br>
> 이를 초과하면 PermError 처리되어 SPF 검증이 실패합니다.
{: .prompt-danger }

```bash
# ❌ 위험 예시 - lookup이 10회를 초과할 수 있음
v=spf1 include:A include:B include:C include:D include:E \
       include:F include:G include:H include:I include:J ~all
```

---

### 2-2. DKIM 란?

> **"이 메일이 우리 도메인에서 서명되었고, 전송 중 내용이 변조되지 않았다"** 는 것을 공개키 암호화로 증명!<br>
> RFC 6376으로 표준화되어 있으며, 비대칭 암호화(공개키/개인키 쌍)를 사용합니다.
{: .prompt-tip }

** 동작 원리**

```
발신 측 (메일 서버):
  1. 메일 헤더 + 본문을 해시
  2. 보유한 개인키(private key)로 서명
  3. 서명값을 DKIM-Signature 헤더에 첨부해 발송

  DKIM-Signature: v=1; a=rsa-sha256; d=example.com;
                  s=selector; h=from:to:subject:date;
                  bh=base64(body hash); b=base64(signature)

수신 측 (Gmail 등):
  1. DKIM-Signature 헤더에서 d=도메인, s=셀렉터 추출
  2. DNS 조회: selector._domainkey.example.com
  3. 공개키(public key) 획득
  4. 공개키로 서명 검증
  5. 검증 성공 → DKIM PASS / 실패 → DKIM FAIL
```

** DKIM DNS 레코드 구조**

레코드 이름 형식: `{셀렉터}._domainkey.{도메인}`

```
; 레코드 이름
selector._domainkey.example.com

; 레코드 값
v=DKIM1; k=rsa; p=ABCDEFGHIJKLMNOP...
```

| **필드** | **의미** |
| --- | --- |
| v=DKIM1 | 버전 |
| k=rsa | 키 알고리즘 |
| p=... | Base64 인코딩된 공개키 |

**셀렉터(Selector)의 의미**

셀렉터는 하나의 도메인이 **여러 개의 DKIM 키를 동시에 운영**할 수 있게 해주는 식별자입니다.

```
wiseu._domainkey.example.com   → 이메일 발송 서비스 A용 키
ds._domainkey.example.com      → 이메일 발송 서비스 B용 키
google._domainkey.example.com  → Google Workspace용 키
```

이 구조 덕분에 서비스별로 DKIM 키를 독립적으로 관리하고 교체할 수 있습니다.

** SPF와 DKIM의 결정적 차이**

| **구분** | **SPF** | **DKIM** |
| --- | --- | --- |
| 무엇을 검증 | 발신 서버 IP | 메일 헤더 + 본문 무결성 |
| 변조 감지 | 불가능 | 가능 |
| 포워딩 시 | 중계 서버 IP 변경으로 FAIL | 서명은 유지되므로 PASS |
| 메일 본문 보호 | 없음 | 있음 |

---

### 2-3. DMARC 란?

> **"SPF와 DKIM 검증 결과를 종합해, 실패 시 수신 서버가 어떻게 처리할지 정책을 선언하고 리포트를 받는"** 메커니즘<br>
> RFC 7489로 표준화되어 있으며, SPF와 DKIM 위에 얹히는 **정책 레이어** 입니다.
{: .prompt-tip }

**DMARC가 추가하는 핵심 개념: Alignment**

SPF와 DKIM은 각각 독립적으로 검증됩니다. 그런데 이 둘이 PASS여도 Header From 도메인과 일치하지 않으면 스푸핑일 수 있습니다. DMARC의 **Alignment** 검사가 이를 잡아냅니다.

```
SPF Alignment:  Return-Path 도메인 == Header From 도메인?
DKIM Alignment: DKIM-Signature의 d= 도메인 == Header From 도메인?

둘 중 하나라도 Alignment 통과 + 해당 인증 PASS → DMARC PASS
둘 다 실패 → DMARC FAIL → p= 정책 적용
```

** DMARC 레코드 문법**

레코드 이름: `_dmarc.{도메인}` (항상 이 형식)

```
_dmarc.example.com  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

**주요 태그**

| **태그** | **의미** | **값 예시** |
| --- | --- | --- |
| v= | 버전 (필수) | DMARC1 |
| p= | 정책 (필수) | none, quarantine, reject |
| sp= | 서브도메인 정책 | none, quarantine, reject |
| rua= | 집계 리포트 수신 주소 | mailto:dmarc@example.com |
| ruf= | 포렌식 리포트 수신 주소 | mailto:forensic@example.com |
| pct= | 정책 적용 비율 (%) | 100 |
| adkim= | DKIM Alignment 강도 | r(relaxed), s(strict) |
| aspf= | SPF Alignment 강도 | r(relaxed), s(strict) |

**정책 단계별 의미**

```
p=none       → 아무것도 하지 않음. 리포트만 받음 (모니터링 단계)
p=quarantine → 실패 메일을 스팸함으로 격리
p=reject     → 실패 메일을 완전 거부 (가장 강력)
```

** DMARC 도입 권장 단계**

```
1단계: p=none; rua=mailto:dmarc@your-domain.com
       → 2~4주 리포트 수집, 정상 발신 경로 파악

2단계: p=quarantine; pct=10
       → 10%만 격리 적용, 점진적 강화

3단계: p=quarantine; pct=100
       → 전체 격리 적용

4단계: p=reject; pct=100
       → 완전 차단 (최종 목표)
```

---

## 03. 세 가지 레코드의 관계

- **SPF** : 발신 IP 확인 및 허가 여부 판정
- **DKIM** : 서명 검증 및 무결성 확인
- **DMARC** : SPF + DKIM 종합하여 Alignment 검사, 정책에 따라 집행

** DMARC 판정 매트릭스**

| SPF | DKIM | DMARC 결과 | 비고 |
| --- | --- | --- | --- |
| PASS (Aligned) | PASS (Aligned) | ✅ PASS | 이상적 상태 |
| PASS (Aligned) | FAIL | ✅ PASS | SPF 하나로 충분 |
| FAIL | PASS (Aligned) | ✅ PASS | DKIM 하나로 충분 |
| FAIL | FAIL | ❌ FAIL | p= 정책 실행 |
| PASS (Not Aligned) | FAIL | ❌ FAIL | Alignment 실패 |

**각각이 막는 위협**

| **위협** | **SPF** | **DKIM** | **DMARC** |
| --- | --- | --- | --- |
| 외부 IP 스푸핑 | ✅ 차단 | ❌ | ✅ (종합 판정) |
| 메일 내용 변조 | ❌ | ✅ 감지 | ❌ |
| Header From 위조 | ❌ | ❌ | ✅ Alignment |
| 포워딩 메일 오탐 | ❌ (오탐 발생) | ✅ 견고 | 부분적 |

---

## 04. 실무 설정 예시

** 다중 발송 서비스 SPF 통합**

여러 이메일 서비스를 쓸 때 SPF는 **반드시 하나의 레코드에 통합**해야 합니다.

```bash
# ❌ 잘못된 설정 - 두 개의 SPF가 있으면 PermError
example.com  TXT  "v=spf1 include:spf1.service-a.com ~all"
example.com  TXT  "v=spf1 include:spf1.service-b.com ~all"

# ✅ 올바른 설정 - 하나의 레코드에 모두 통합
example.com  TXT  "v=spf1 include:spf1.service-a.com include:spf1.service-b.com ~all"
```

** 서비스별 DKIM 레코드 구성**

```bash
# 이메일 서비스 A용 DKIM
wiseu._domainkey.example.com  TXT  "v=DKIM1; k=rsa; p=ABCDEFG..."

# 이메일 서비스 B용 DKIM
ds._domainkey.example.com     CNAME  dkim.service-b.co.kr

# Google Workspace용 DKIM
google._domainkey.example.com TXT  "v=DKIM1; k=rsa; p=GFEDCBA..."
```

** DMARC 설정**

```bash
# 초기 모니터링 단계
_dmarc.example.com  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"

# 강화 단계
_dmarc.example.com  TXT  "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc-reports@example.com; adkim=r; aspf=r"
```

---

## 05. 추가 권장 항목

SPF/DKIM/DMARC 외에 이메일 및 도메인 보안을 강화하는 추가 메커니즘들입니다.

### MTA-STS (Mail Transfer Agent Strict Transport Security)

> "우리 메일 서버로의 연결은 반드시 TLS로만 허용한다"<br>
> SMTP는 기본적으로 평문 통신이 가능합니다. MTA-STS는 발신 서버에게 TLS 강제를 선언하여 중간자 공격(MITM)을 방어합니다.
{: .prompt-info }

```bash
# MTA-STS 정책 위치 선언
_mta-sts.example.com  TXT  "v=STSv1; id=20240101"

# 실제 정책은 HTTPS로 제공
# https://mta-sts.example.com/.well-known/mta-sts.txt
```

```
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 86400
```

### TLS-RPT (TLS Reporting)

> "TLS 연결 실패를 리포트로 받는다"<br>
> MTA-STS와 함께 사용하며, TLS 협상 실패 시 알림을 받을 수 있습니다.
{: .prompt-info }

```bash
_smtp._tls.example.com  TXT  "v=TLSRPTv1; rua=mailto:tls-reports@example.com"
```

### BIMI (Brand Indicators for Message Identification)

> "DMARC p=reject 도달 후, 수신함에 브랜드 로고를 표시한다"<br>
> Gmail, Yahoo 등 주요 메일 클라이언트에서 발신자 아이콘 자리에 공식 로고를 표시합니다.<br>
> **DMARC p=reject 설정이 선행 조건입니다.**
{: .prompt-tip }

```bash
default._bimi.example.com  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

- `l=` : SVG 형식 로고 URL
- `a=` : VMC(Verified Mark Certificate) - CA로부터 발급받는 브랜드 인증서 (유료)

### CAA (Certification Authority Authorization)

> "우리 도메인의 SSL/TLS 인증서는 특정 CA만 발급할 수 있다"<br>
> 이메일과 직접 관련은 없지만, 도메인 보안의 일환으로 도메인 하이재킹 및 인증서 오발급을 방어합니다.
{: .prompt-info }

```bash
example.com  CAA  0 issue "amazon.com"
example.com  CAA  0 issue "letsencrypt.org"
example.com  CAA  0 issuewild ";"  # 와일드카드 발급 금지
```

### SPF Macro

> 일부 글로벌 표준에서 사용하는 동적 SPF 평가 방식입니다.<br>
> 발신 IP, 도메인 등을 변수로 활용해 더 정밀한 허가 목록을 구성할 수 있습니다.
{: .prompt-info }

```bash
# 매크로 방식 예시
v=SPF1 include:%{i}._ip.%{h}._ehlo.%{D}._spf.vali.email ~All

# %{i} = 발신 IP
# %{h} = EHLO 도메인
# %{D} = 헤더 From 도메인
```

---

긴 글 읽어주셔서 감사합니다.

> **참고 RFC**<br>
> RFC 7208 (SPF) / RFC 6376 (DKIM) / RFC 7489 (DMARC) / RFC 8461 (MTA-STS) / RFC 8460 (TLS-RPT)
{: .prompt-info }
