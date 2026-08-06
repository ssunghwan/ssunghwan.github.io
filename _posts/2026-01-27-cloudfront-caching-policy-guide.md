---
title: "CloudFront, 잘못 쓰면 이렇게 터질 수 있다."
date: 2026-01-27 01:51:00 +0900
categories: [Cloud & Infrastructure, Troubleshooting]
tags: [aws, cloudfront, cdn, cache, cache-policy, origin-request-policy]
---

안녕하세요!

이커머스 웹사이트 운영 중 겪은 AWS CloudFront 캐싱 정책 변경 장애 사례와 해결 방법을 정리해봤습니다.

---

## 01. HTTP 통신과 헤더의 이해

### ✅ 기본 HTTP 통신 흐름

간단하게, 고객이 웹사이트에 방문하게 된다면 아래와 같은 흐름으로 Traffic이 흘러갑니다.

```
[사용자 브라우저] → [CloudFront] → [Origin Server] → [CloudFront] → [사용자 브라우저]
```

**첫번째 : 브라우저가 요청을 보냄**

```http
GET /index.php HTTP/1.1
Host: 이커머스 사이트 주소입니다.
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)
Accept: text/html
Accept-Language: ko-KR,ko;q=0.9
Cookie: PHPSESSID=abc123; user_id=456
Referer: https://google.com
```

- **Host** : 어느 사이트에 접속하는지
- **User-Agent** : 어떤 디바이스/브라우저인지 (모바일? 데스크톱?)
- **Accept** : 어떤 형식의 응답을 원하는지
- **Accept-Language** : 선호하는 언어
- **Cookie** : 로그인 세션, 장바구니 등 사용자 정보
- **Referer** : 어디서 왔는지 (사이트에 접근하기 이전 페이지)

**두번째 : CloudFront가 요청 받음**

CloudFront는 받은 요청을 보고 두 가지 결정을 한다.

아래와 같이 **캐시 키를 사용하여 캐시 확인**, **캐시가 있는지 없는지 요청을 받고 결정**합니다.

1. 캐시 확인 (Cache key 사용)

```
Cache Key 생성:
  URL: /index.php
  + User-Agent: iPhone (설정된 경우)
  + Cookie: PHPSESSID=abc123 (설정된 경우)

예시 Cache Key: "index.php|iPhone|abc123"
```

2-1. 캐시 Hit

> CloudFront: "아! 이거 캐시에 있네!" → 오리진 서버에 안 가고 → 바로 캐시된 응답 반환 → 빠름!

2-2. 캐시 Miss

> CloudFront: "캐시에 없네, 오리진 서버에 물어봐야겠다" → 오리진 서버로 요청 전달

**세번째 : CloudFront → 오리진 서버 (Cache Miss 시)**

Origin Request Policy에 따라 헤더 전달

```http
GET /index.php HTTP/1.1
Host: origin-server.internal
User-Agent: Mozilla/5.0 (iPhone...)
Cookie: PHPSESSID=abc123
Referer: https://google.com
```

이때 User-Agent, Cookie, Referer 헤더는 Origin Request Policy에 설정을 해야 전달되며, **설정을 안했을 시에는 헤더 전달이 안됩니다.** 즉, 서버는 전달받은 헤더만 알 수 있습니다.

**네번째 : 오리진 서버 응답**

```
HTTP/1.1 200 OK
Content-Type: text/html
Cache-Control: max-age=300
Set-Cookie: session_token=xyz789

<!DOCTYPE html>
<html>
...모바일 버전 HTML...
</html>
```

서버의 판단 과정 (PHP로 예시):

```php
<?php
// User-Agent 헤더가 전달되었는지 확인
if (isset($_SERVER['HTTP_USER_AGENT'])) {
    $userAgent = $_SERVER['HTTP_USER_AGENT'];

    // 모바일 감지
    if (strpos($userAgent, 'iPhone') !== false ||
        strpos($userAgent, 'Android') !== false) {
        // 모바일 버전 HTML 반환
        include 'mobile_template.php';
    } else {
        // 데스크톱 버전 HTML 반환
        include 'desktop_template.php';
    }
} else {
    // User-Agent가 없으면 기본값 (데스크톱)
    include 'desktop_template.php'; // ← 문제 발생!
}
?>
```

**다섯번째 : CloudFront가 응답 캐싱**

> 1. Origin 응답 받음
> 2. Cache Key로 캐시에 저장
>    - Key: "index.php|iPhone|abc123"
>    - Value: Origin 응답 전체
> 3. TTL 설정 (Cache-Control 헤더 기반)
> 4. 사용자에게 응답 전달

**여섯번째 : 다음 사용자 요청 시**

> 다음 모바일 사용자:
>   Cache Key: "index.php|iPhone|def456"
>   → 쿠키 다름 → Cache Miss → 오리진 요청
>
> 똑같은 모바일 사용자:
>   Cache Key: "index.php|iPhone|abc123"
>   → 완전히 동일 → Cache Hit! → 바로 응답

---

### 1-1. User-Agent 헤더의 중요성

**📌 User-Agent란?** 브라우저가 자신의 신원을 알려주는 헤더

**실제 User-Agent 예시**

```
iPhone:
Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)
AppleWebKit/605.1.15 (KHTML, like Gecko)
Version/16.0 Mobile/15E148 Safari/604.1

Android:
Mozilla/5.0 (Linux; Android 13)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/112.0.0.0 Mobile Safari/537.36

Desktop Chrome:
Mozilla/5.0 (Windows NT 10.0; Win64; x64)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/112.0.0.0 Safari/537.36
```

왜 User-Agent가 중요하냐면, **적응형 웹사이트에서는 같은 URL에서 디바이스별로 다른 HTML을 제공**합니다.

```
URL: https://example.com/

모바일 접속:
→ 모바일 최적화 레이아웃
→ 작은 이미지
→ 터치 친화적 버튼

데스크톱 접속:
→ 넓은 레이아웃
→ 큰 이미지
→ 마우스 호버 효과
```

만약, 서버가 User-Agent를 못 받는다면?

```php
<?php
// User-Agent 없음
if (!isset($_SERVER['HTTP_USER_AGENT'])) {
    // 기본값으로 데스크톱 버전 반환
    include 'desktop_template.php';
}
?>
```

**기본값으로 데스크톱 버전을 반환**하여 아래와 같은 결과를 초래한다.

- 모바일 사용자는 데스크톱 버전을 받는다
- 레이아웃 깨짐
- 이미지 사이즈 안 맞음
- 사용자 경험 최악 → 고객 재방문 X

---

### 1-2. Cookie 헤더의 역할

**📌 Cookie란?** 브라우저가 저장하는 작은 데이터 조각입니다.

**Cookie 종류**

1. 세션 쿠키 (ex: `PHPSESSID=abc123def456`)
   - 로그인 상태 유지
   - 로그아웃 시 세션 쿠키 삭제됨
2. 영구 쿠키 (ex: `user_pref=dark_mode`)
   - 사용자 설정 저장
   - 브라우저 닫아도 유지
3. 장바구니 쿠키 (ex: `cart_id=789xyz`)
   - 장바구니 내용 추적 가능

만약, 쿠키가 전달되지 않으면?

```php
<?php
session_start();

// 세션 쿠키가 없으면
if (!isset($_SESSION['user_id'])) {
    // 로그인 안 된 것으로 판단
    header('Location: /login.php');
    exit;
}

// 로그인된 사용자만 볼 수 있는 페이지
echo "환영합니다, " . $_SESSION['username'];
?>
```

- 로그인했는데 로그인 안 된 것으로 인식 (고객 입장에서는 최악)
- 관리자 역시 관리자 페이지에 접근 불가
- 장바구니 내용 사라짐 (추적 불가 → 고객 입장에서 쇼핑 불가)

---

## 02. CloudFront 동작 원리

**💁 CloudFront가 없는 경우**

```
서울 → 서울 서버 : 10ms
부산 → 서울 서버 : 30ms
LA   → 서울 서버 : 150ms
런던 → 서울 서버 : 300ms
```

**💁 CloudFront가 있는 경우**

```
[사용자] ---> [CloudFront 엣지] ---> [오리진 서버]

첫 요청:     121ms (약간 느림)
캐시 Hit 시:  21ms (5배 빠름!)
```

**📌 Cache Key 역할 :** 캐시를 구분하는 고유 식별자

- Cache Key 구성 요소: URL (필수), Query String (선택), Headers (선택), Cookies (선택)

예시 1) URL만 있을 경우 → 모바일/데스크톱 환경 구분 안 됨

```
Cache Key: "/index.php"
```

예시 2) URL + User-Agent → 디바이스별 다른 캐시, 환경 구분 가능

```
Cache Key: "/index.php|Mobile"
Cache Key: "/index.php|Desktop"
```

**📌 실제 시나리오**

사용자 A (Mobile)

```
요청: GET /index.php
Cache Key: "/index.php|Mobile"
→ Cache Miss → 오리진에서 모바일 HTML 받음 → 캐시 저장
```

사용자 B (Mobile)

```
요청: GET /index.php
Cache Key: "/index.php|Mobile" (동일!)
→ Cache Hit → 빠르게 응답 (Origin Server에 안 가고, CloudFront 엣지에서 해결)
```

사용자 C (Desktop)

```
요청: GET /index.php
Cache Key: "/index.php|Desktop"
→ Cache Miss → 오리진에서 데스크톱 HTML 받음 → 별도 캐시 저장
```

---

## 03. Cache Policy vs Origin Request Policy

**📌 Cache Policy (캐시 정책)**

목적 : 캐시를 어떻게 구분할까?

```
Cache Policy:
  Headers:
    - CloudFront-Is-Mobile-Viewer
  Cookies:
    - PHPSESSID
  Query Strings:
    - All
```

Cache Policy의 영향:
- 캐시 Hit/Miss 결정
- 캐시 효율성 결정
- Origin에 Header **전달 안 함**

**📌 Origin Request Policy (오리진 요청 정책)**

목적 : 오리진에 무엇을 전달할까?

```
Origin Request Policy:
  Headers:
    - User-Agent
    - Referer
  Cookies:
    - All
  Query Strings:
    - All
```

Origin Request Policy의 영향:
- 캐시 구분에 영향 없음
- 서버가 받는 정보 결정
- 서버 응답 내용을 결정

---

### 3-1. 왜 둘 다 필요할까?

**✅ 시나리오 1) Cache Policy만 설정하였을 때**

```
Cache Policy:
  Headers: CloudFront-Is-Mobile-Viewer

Origin Request Policy: (없음)
```

사용자 (Mobile) 요청 동작:

```
→ Cache Key: "/index.php|Mobile"
→ Cache Miss
→ 오리진 요청
→ User-Agent 헤더 없음
→ 서버가 데스크톱 버전 반환
→ 캐시 저장: "/index.php|Mobile" = 데스크톱 HTML
```

결과 : 모바일 캐시에 데스크톱 HTML이 저장되며, **모든 모바일 사용자가 데스크톱 버전을 받는다.**

**✅ 시나리오 2) Origin Request Policy만 설정하였을 때**

```
Cache Policy:
  URL: /index.php

Origin Request Policy:
  Headers: User-Agent
```

사용자별 요청 동작:

```
사용자 A (모바일):
→ Cache Key: "/index.php" (디바이스 구분 없음)
→ Cache Miss → 오리진 요청 (User-Agent 전달)
→ 서버가 모바일 HTML 반환
→ 캐시 저장: "/index.php" = 모바일 HTML

사용자 B (데스크톱):
→ Cache Key: "/index.php" (동일!)
→ Cache Hit!
→ 모바일 HTML 반환 ← 데스크톱인데 모바일 버전 받음
```

**✅ 시나리오 3) 둘 다 설정하였을 때 (올바른 방법)**

```
Cache Policy:
  Headers: CloudFront-Is-Mobile-Viewer

Origin Request Policy:
  Headers: User-Agent
```

사용자별 요청 동작:

```
사용자 A (모바일):
→ Cache Key: "/index.php|Mobile"
→ Cache Miss → 오리진 요청 (User-Agent 전달 ✓)
→ 서버가 모바일 HTML 반환
→ 캐시 저장: "/index.php|Mobile" = 모바일 HTML ✓

사용자 B (데스크톱):
→ Cache Key: "/index.php|Desktop" (다름!)
→ Cache Miss → 오리진 요청 (User-Agent 전달 ✓)
→ 서버가 데스크톱 HTML 반환
→ 캐시 저장: "/index.php|Desktop" = 데스크톱 HTML ✓

사용자 C (모바일):
→ Cache Key: "/index.php|Mobile"
→ Cache Hit! ✓ → 모바일 HTML 반환 ✓
```

---

## 04. 실제 장애 사례

**✅ 장애 발생 배경**

- 초기 상황: 일 방문자 약 5,000명 규모의 이커머스 운영 중, 모바일 트래픽 70%
- 발단:
  1. 업데이트된 파일을 운영계에 배포
  2. 하지만 웹사이트에 반영이 안 되는 현상
  3. Cache 때문이라고 판단됨
  4. 해당 파일의 경로에 대해 캐시 초기화(무효화) 진행
  5. 여전히 반영이 되지 않았음

**✅ 시도한 해결책**

**1차 시도 ❌ 실패**

CloudFront 이미지(jpg, jpeg, png ..) 경로의 캐시 정책 변경
- 기존 : 기본 TTL 1일, 최대 TTL 1년
- 변경 : 기본, 최대 TTL 10분
- 목적 : 빠르게 업데이트 반영

추가 조치:
- default 경로에 User-Agent 헤더 추가
- Cache Policy에 CloudFront-Is-Mobile-Viewer 포함

결과:
- 업데이트된 파일이 여전히 반영 안 됨
- 모바일 이미지 사이즈 미스매치 (레이아웃 깨짐)
- 관리자 페이지 로그인 불가
- 일부 카테고리 접근 불가

**2차 시도 ⚠️ 부분 성공**

CloudFront 동작 default 경로에 대한 쿠키 설정 변경
- Cookie를 "모두" 포함
- 목적 : 세션 인식 문제 해결

결과:
- 관리자 페이지 로그인 가능 (접근 가능)
- 모바일 이미지 여전히 문제
- 일부 카테고리 접근 여전히 불가

**3차 시도 : 원상 복구**

- 모든 설정 원상 복구 (TTL 10분 → TTL 1일, Cache Policy → Legacy cache settings)
- 이커머스 사이트이다 보니 고객 서비스에 문제가 직결되므로 빠른 복구
- 복구 이후 CloudFront Logs 및 Web Server Logs 파악

---

### 4-1. 로그 분석

**📌 CloudFront 로그 분석**

변경 전 (기존 TTL 1일, 최대 TTL 1년)

```
이미지:
Hit: 11,010건 / Miss: 1,481건 / RefreshHit: 1,588건
히트율: 89.48% ✓

동적 페이지:
Miss: 2,740건 (100%) / 히트율: 0%
→ php/html은 캐시 안 함 (정상)
```

변경 후 (기존, 최대 TTL 10분)

```
이미지:
Hit: 11,216건 / Miss: 1,216건 / RefreshHit: 330건 (-79%!)
히트율: 90.47% (여전히 양호)

동적 페이지:
Miss: 4,155건 (100%) / 히트율: 0%
→ 여전히 캐시 안 함
```

핵심 분석:
- RefreshHit 79% 급감
- 하지만 이미지 히트율은 양호
- 동적 페이지(php 등)는 원래 캐시 X
- **이미지 문제는 아닌 것으로 판단됨**

**📌 Apache Error Logs 분석**

```
[17:37:31] PHP Fatal Error: Cannot re-assign $this
[17:37:39] PHP Warning: Input variables exceeded 3000
[17:37:39] PHP Fatal Error: Cannot re-assign $this
[17:37:47] PHP Warning: Input variables exceeded 3000
[17:37:47] PHP Fatal Error: Cannot re-assign $this

→ 쿠키 처리 중 오류 (부차적 문제)
```

**📌 근본적인 원인 파악**

문제 핵심 : Legacy cache settings → Cache Policy 전환 시 누락된 것:

```
Legacy cache settings (기존):
✓ User-Agent를 자동으로 오리진에 전달
✓ 쿠키를 자동으로 전달
✓ 개발자가 신경 안 써도 됨

Cache Policy (변경 후):
✓ Cache Key에 CloudFront-Is-Mobile-Viewer 추가 (완료)
✓ Cache Key에 Cookie 추가 (완료)
❌ Origin Request Policy 설정 안 함! ← 원인
```

휴먼 에러로 인해 아래와 같은 결과를 초래:
- User-Agent가 서버에 전달 안 됨
- 서버가 모바일/데스크톱 구분 못 함
- 기본값(데스크톱) 응답
- 모바일 사용자가 데스크톱 버전 받음

---

## 05. 올바른 설정 방법

**1) 정적 이미지용 Cache Policy:**

```
TTL:
  Min: 3600
  Default: 14400
  Max: 86400

Cache Key Settings:
  Headers:
    - CloudFront-Is-Mobile-Viewer
    - Accept-Encoding
  Cookies: None
  Query Strings: All
```

**2) 동적 페이지용 Cache Policy:**

```
TTL:
  Min: 0
  Default: 300
  Max: 1800

Cache Key Settings:
  Headers:
    - CloudFront-Is-Mobile-Viewer
    - Accept-Encoding
  Cookies: All
  Query Strings: All
```

**3) Origin Request Policy:**

```
Headers:
  - User-Agent        # 필수!
  - CloudFront-Is-Mobile-Viewer
  - CloudFront-Is-Desktop-Viewer
  - CloudFront-Is-Tablet-Viewer
  - Referer
  - Accept-Language
Cookies: All
Query Strings: All
```

---

## 06. 검증 방법

**Test 1) User-Agent 전달 확인**

서버에 테스트 파일 생성 후 확인:

```php
<?php
// test_headers.php
header('Content-Type: text/plain');
echo "=== 받은 헤더 목록 ===\n\n";

echo "User-Agent: " . ($_SERVER['HTTP_USER_AGENT'] ?? '없음') . "\n";
echo "CloudFront-Is-Mobile-Viewer: " . ($_SERVER['HTTP_CLOUDFRONT_IS_MOBILE_VIEWER'] ?? '없음') . "\n";
echo "CloudFront-Is-Desktop-Viewer: " . ($_SERVER['HTTP_CLOUDFRONT_IS_DESKTOP_VIEWER'] ?? '없음') . "\n";
echo "Referer: " . ($_SERVER['HTTP_REFERER'] ?? '없음') . "\n";

echo "\n=== 쿠키 ===\n\n";
print_r($_COOKIE);
?>
```

테스트 진행:

```bash
# 모바일로 테스트
curl -H "User-Agent: iPhone" https://example.com/test_headers.php
# 기대 출력:
# User-Agent: iPhone
# CloudFront-Is-Mobile-Viewer: true
# CloudFront-Is-Desktop-Viewer: false

# 데스크톱으로 테스트
curl -H "User-Agent: Chrome" https://example.com/test_headers.php
# 기대 출력:
# User-Agent: Chrome
# CloudFront-Is-Mobile-Viewer: false
# CloudFront-Is-Desktop-Viewer: true
```

캐시 동작 확인:

```bash
# 첫 요청 (Cache Miss)
curl -I https://example.com/
# X-Cache: Miss from cloudfront

# 두 번째 요청 (Cache Hit)
curl -I https://example.com/
# X-Cache: Hit from cloudfront
```

모바일/데스크톱 구분 확인:

```
브라우저 개발자 도구 (F12) → Network 탭에서 확인

모바일 접속 → 모바일 레이아웃 ✓
데스크톱 접속 → 데스크톱 레이아웃 ✓
```

---

## 07. 배포 체크리스트

**📌 배포 전 체크리스트**

```
□ Cache Policy 2개 생성 완료
    □ 이미지용 (TTL 4시간)
    □ 동적 페이지용 (TTL 5분)

□ Origin Request Policy 1개 생성 완료
    □ User-Agent 포함 (필수!)
    □ CloudFront-Is-*-Viewer 포함
    □ 쿠키 All 설정

□ Behavior 설정 완료
    □ 이미지 경로: Cache Policy만
    □ Default 경로: Cache Policy + Origin Request Policy

□ 테스트 환경 검증
    □ 스테이징에서 먼저 테스트
    □ 모바일/데스크톱 둘 다 확인

□ 롤백 계획 수립
    □ 5~10분 내 원복 가능한 절차
```

**📌 배포 후 체크리스트**

```
□ User-Agent 전달 확인
    □ 테스트 파일로 확인
    □ 서버 로그 확인

□ 캐시 동작 확인
    □ X-Cache 헤더 확인
    □ Hit/Miss 정상 동작

□ 비즈니스 검증
    □ 모바일에서 접속
    □ 데스크톱에서 접속
    □ 관리자 로그인 확인
    □ 주요 기능 정상 동작

□ 30분간 모니터링
    □ 에러율 확인
    □ 캐시 히트율 확인
    □ 사용자 피드백 확인
```

---

추가적인 피드백은 언제든지 환영합니다. 감사합니다 :)
