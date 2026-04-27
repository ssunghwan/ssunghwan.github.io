---
title: "RDS Intermittent Failure During E-commerce Event - N+1 Query"
date: 2026-02-23 03:59:00 +0900
categories: [Infrastructure, Root Cause Analysis]
tags: [rds, database, n+1 query]
---

안녕하세요,

실제 운영 환경의 이커머스에서 이벤트 행사 중 발생한 RDS 관련 장애 조치에 대해서 포스팅 하려고 합니다.

저는 이 문제를 "N+1" 쿼리 문제로 보고 있는데, 이견이 있으시다면 언제든 댓글 환영합니다!

---

## 01. 개요 및 장애 증상

**📌 개요**

실제 운영 환경의 이커머스 사이트에서 선착순 이벤트 진행 중 관리자 페이지에서 재고 관리 카테고리에 접근 시 "Error: Database Query" 오류가 발생 하였으며, 실제 고객들의 사이트 접근에도 일정 시간동안 작동되지 않음.

추가적으로, 고객들의 몇 건의 결재 오류도 발생한 정황이 있습니다.

결론부터 말하자면, 인프라의 문제보다는 애플리케이션 코드의 "N+1 Query" 문제라고 판단이 돈다.

**📌 대략적 인프라 환경**

- RDS: db.r5.xlarge (RAM 32GB), MySQL 8.0, Multi-AZ
- EC2: 여러 대의 Instance로, 앞단에서 ALB로 트래픽이 분산되는 구조.

**📌 장애 증상**

- 모니터링 도구로 지표 확인 시
  - Database Connections가 100~200까지 급등, WriteThroughput 스파이크
  - RDS CPU 50% 미만, EC2 Instance 20% 미만.
- 로그 확인 시
  - RDS audit log : 다량의 특정 쿼리에 대한 무한 루프 현상 발견
  - RDS error log : 0 byte.
  - RDS slowquery log : 0 byte.

> **Database Connections 이란?**
>
> DB에 동시에 연결된 세션(connection) 수다. 고객이 페이지를 요청하면 웹 서버가 DB에 connection을 열고 쿼리를 실행한 뒤 닫는다. 쿼리가 빨리 끝나면 connection도 빨리 반환되지만, N+1처럼 쿼리가 폭발적으로 많아지면 connection이 쌓인다.
> RDS db.r5.xlarge의 max_connections는 약 2,594개이며, 이번 이벤트에서 최대 200개(약 12%)까지 올랐다.
> DB 자체가 다운될 수준은 아니었지만 단시간에 급등한 것이 비정상적인 쿼리 패턴을 나타낸다.

---

## 02. 원인 분석

**✅ RDS Audit Log 확인**

slowquery 및 error 로그는 0byte였으므로 audit log로 분석 하였으며, 이벤트 발생 시간대에 아래 패턴이 발생:

```sql
20260221 04:02:27, connection_id=3780476, query_id=107967516
select count(*) from rb_order_cart where mb_id = ''

20260221 04:02:27, connection_id=3780476, query_id=107967517
select * from rb_product_review_like where pr_idx = '6512'

20260221 04:02:27, connection_id=3780476, query_id=107967518
select count(*) from rb_order_cart where mb_id = ''

20260221 04:02:27, connection_id=3780476, query_id=107967519
select * from rb_product_review_like where pr_idx = '6498'

-- 이하 동일 패턴 반복...
```

동일 connection(3780476)에서 같은 초(04:02:27)에 쿼리 ID가 107967516 → 107967542까지 순식간에 증가했다. 상품 목록을 루프 돌면서 각 상품마다 개별 쿼리를 날리는 전형적인 N+1 패턴이라고 생각한다.

### N+1 쿼리란?

N+1 쿼리는 목록 조회 1번(N개의 결과) + 각 항목마다 추가 쿼리 N번 = **총 N+1번 실행**되는 비효율적인 패턴이다.

개발 초기엔 코딩이 편해서 자주 쓰이지만, 트래픽이 늘어날수록 DB에 기하급수적인 부하를 준다.

```python
products = get_product_list()     # 쿼리 1번 (목록 조회)

for product in products:
    reviews = get_reviews(product.id)  # 상품마다 쿼리 1번씩 추가 → N번
```

아래와 같이 효율적으로 관리가 된다면 목록 조회가 한번으로 끝나게 된다.

```sql
SELECT * FROM rb_product_review_like WHERE pr_idx IN (6512, 6498, 6477, ...)
```

하지만 현재와 같이 N+1 문제가 있는 방식으로 조회를 한다면:

```sql
SELECT * FROM rb_product_review_like WHERE pr_idx = '6512'
SELECT * FROM rb_product_review_like WHERE pr_idx = '6498'
SELECT * FROM rb_product_review_like WHERE pr_idx = '6477'
-- 상품 50개면 50번 반복...
```

각 쿼리 자체는 0.001초 안에 끝나는 빠른 쿼리이지만, 실제 문제는 실행 횟수다.

- 상품 50개 기준, 평소 동접 10명 → 쿼리 500번
- 행사 동접 1,000명 → 쿼리 **50,000번**

DB가 죽지 않더라도 **connection이 급격히 치솟아 응답 지연과 간헐적 오류를 유발**한다.

### 과다 및 이상 현상

**✅ 페이지 요청당 쿼리 수 과다**

고객 한 분이 메인페이지 접속 한 번에 32개의 쿼리가 실행된 것을 확인:

- rb_session_table → 세션 확인 (1회)
- rb_config → 설정값 조회 (1회)
- rb_member → 회원 정보 조회 (2회 — mb_idx=48386, mb_idx=1)
- rb_pet → 펫 정보 (2회)
- rb_cate → 카테고리 메뉴 (3회 — ca_step별로 각각)
- rb_banner → 배너 (3회 — bn_loc=301, 302, 303)
- rb_exhibition, rb_brand, rb_wiki, rb_coupon_record 등

동접 1,000명이면 페이지 로딩 쿼리만 **32,000개**다. 여기에 N+1 쿼리까지 더해지면 DB가 받는 부하는 기하급수적으로 늘어난다.

**📌 rb_cate를 매 요청마다 반복 조회**

카테고리 메뉴(rb_cate)는 거의 변하지 않는 정적 데이터임에도 불구하고 페이지 요청마다 DB에서 새로 가져오고 있다. 전체 로그에서 rb_cate 조회가 **348회**로 가장 많았다. **캐시(Redis 등)를 적용**하면 이 쿼리는 거의 0으로 줄일 수 있다.

**📌 rb_member 동일 요청에서 2회 조회**

한 번의 요청에서 mb_idx = 48386(실제 회원)과 mb_idx = 1(관리자 계정으로 추정)이 각각 조회됐다. 불필요한 중복 조회로 코드 확인이 필요하다.

**📌 DB 세션 관리 구조**

rb_session_table이라는 테이블에서 세션을 DB로 관리하고 있다. 트래픽이 적을 때는 문제없지만, 행사처럼 동접이 급증하면 세션 조회 자체가 병목이 된다. 세션 테이블 조회 실패 → mb_id를 읽지 못한 채로 쿼리 실행.

**✅ 고객 아이디 컬럼이 빈 값으로 출력되는 현상**

로그인 필수 서비스임에도 mb_id = ''(빈값) 상태로 쿼리가 실행되는 케이스가 확인되었으며, 정상 세션은 mb_id = 'ssunghwan'처럼 실제 값이 들어있다.

행사 트래픽 폭주로 세션이 깨지거나, mb_id 없을 때 예외처리 없이 쿼리를 그냥 날리는 코드 버그로 보인다. 이 경우 결과가 없으니 루프가 비정상적으로 반복된다.

---

## 03. 추가 원인 예상

> "이벤트 중 WMS 재고 동기화 작업을 진행했고, max_execution_time 파라미터가 DB 8.0 업그레이드 시 초기화되어 좀비 프로세스를 막지 못했다."

### max_execution_time 초기화가 원인인가?

`max_execution_time`은 MySQL에서 **단일 SELECT 쿼리 하나의 최대 실행 시간을 밀리초 단위로 제한하는 파라미터**다. 예를 들어, `max_execution_time = 30000`이면 30초가 넘는 쿼리는 자동으로 강제 종료(kill)된다.

실제로 현재 값이 0(무제한)으로 확인되어 DB 8.0 업그레이드 시 값이 초기화된 건 사실이나, 이번 문제는 이 파라미터와 무관하다고 생각한다. N+1 쿼리 각각은 0.001초 안에 끝나는 빠른 쿼리이기 때문에 30초 제한에 걸리지 않는다. 또한 좀비 프로세스가 실제 장애 원인이었다면 error 로그에 반드시 기록이 남았어야 하는데, **error 로그가 0 bytes인 게 이를 반증**한다.

### DB 클러스터의 읽기 인스턴스 업그레이드를 한다면?

Aurora는 읽기 전용 엔드포인트를 별도로 제공해 SELECT 쿼리를 여러 읽기 노드에 분산시킬 수 있다. 읽기 트래픽이 많아 성능이 저하되는 상황에서는 유효한 해결책이다.

그러나 이번 문제는 읽기 처리 성능이 부족한 게 아니라, **같은 쿼리가 중복으로 수만 번 날아온 것**이다. 수도관이 좁아서 문제가 아니라, 수도꼭지를 수만 번 껐다 켰다 하는 게 문제인 상황에서 수도관을 굵게 바꾼다고 해결되지 않는다.

즉, 읽기 인스턴스 업그레이드로는 근본적인 해결이 불가능하다.

**✅ 실제 원인 정리**

| 항목 | 내용 |
| --- | --- |
| 직접 원인 | rb_product_review_like, rb_order_cart 조회 시 N+1 쿼리 |
| 부가 원인 | mb_id 빈값 상태 예외처리 미흡 |
| 인프라 이상 | 없음 (CPU 정상, error 로그 0 bytes, connections 200/2594 = 12%) |

---

## 04. 향후 대책

**1) max_execution_time 파라미터 설정**

Dynamic 파라미터라 재시작 없이 적용 가능. 단, 이번 장애의 근본 원인을 해결하지는 않는다. 오래 걸리는 단일 쿼리(배치성 작업) 제어 용도로는 유효하다.

**2) N+1 쿼리 코드 수정**

상품 목록 조회 시 상품 ID를 한 번에 가져오도록 `WHERE IN`으로 일괄 처리:

```sql
-- 현재 (N번 실행)
SELECT * FROM rb_product_review_like WHERE pr_idx = '6512'
SELECT * FROM rb_product_review_like WHERE pr_idx = '6498'

-- 수정 후 (1번 실행)
SELECT * FROM rb_product_review_like WHERE pr_idx IN (6512, 6498, 6477, ...)
```

rb_order_cart도 동일하게 수정.

**3) mb_id 빈 값 예외처리**

세션이 없거나 mb_id가 빈값일 때 쿼리 자체를 실행하지 않도록 체크 로직 추가:

```php
// 수정 전
$result = query("SELECT ... WHERE mb_id = '$mb_id'");

// 수정 후
if (empty($mb_id)) return;
$result = query("SELECT ... WHERE mb_id = '$mb_id'");
```

**4) 정적 데이터 캐시 적용**

rb_cate(카테고리), rb_config(설정값), rb_banner(배너) 같은 거의 변하지 않는 데이터를 Redis나 Memcached에서 캐시하면 DB 부하를 대폭 줄일 수 있다. 로그에서 rb_cate가 1분에 348회 조회됐는데, 캐시 적용하면 거의 0이 될 수 있다.

---

추가적인 피드백은 언제든지 환영합니다. 감사합니다 :)
