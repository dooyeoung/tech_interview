# 입찰 시스템 아키텍처 설계 (구매·판매 매칭 포함)


## **1) 전체 아키텍처 구성**

* 구매/판매 요청은 API 서버에서 검증 후 **Kafka로 이벤트 발행**
* Kafka는 `productId` 기준 파티셔닝
  → 동일 상품 이벤트가 **항상 동일 컨슈머**에 들어와 순서 보장
* 매칭 서버는 Redis 기반 실시간 매칭 수행
* 매칭 성공 시 `MATCHED` 이벤트 발행 → 결제/정산 서비스가 DB 트랜잭션 수행

---

# ✔ 2) Kafka 메시지 형태 (구매/판매 요청)

## **(1) 구매 요청 메시지 – BUY_REQUEST**

```json
{
  "eventType": "BUY_REQUEST",
  "orderId": "buy_12345",
  "productId": "p_001",
  "userId": "u_888",
  "price": 315000,
  "timestamp": 1733980000000
}
```

## **(2) 판매 요청 메시지 – SELL_REQUEST**

```json
{
  "eventType": "SELL_REQUEST",
  "orderId": "sell_99881",
  "productId": "p_001",
  "userId": "u_222",
  "price": 330000,
  "timestamp": 1733980010000
}
```

## **(3) 매칭 성공 메시지 – MATCHED**

```json
{
  "eventType": "MATCHED",
  "matchId": "m_39222",
  "buyOrderId": "buy_12345",
  "sellOrderId": "sell_99881",
  "productId": "p_001",
  "price": 330000,
  "matchedAt": 1733980020000
}
```

---

# ✔ 3) Redis 구조 (실시간 매칭 / 동시성 제어)

## **(1) 구매 ZSET (가격 오름차순)**

* key: `buy:{productId}`
* value: orderId / score: price

```
ZADD buy:p_001 315000 buy_12345
```

## **(2) 판매 ZSET (가격 오름차순)**

```
ZADD sell:p_001 330000 sell_99881
```

## **(3) 주문 상태 캐시 Hash**

```
HSET order:buy_12345 status "active"
HSET order:sell_99881 status "active"
```

---

# ✔ 4) 매칭 로직 (Redis 기반)

### **1) 구매 이벤트 처리**

* 최저 판매가 조회

  ```
  ZRANGE sell:{productId} 0 0 WITHSCORES
  ```
* 구매 가격 ≥ 최저 판매가 → 매칭 후보

### **2) 판매 이벤트 처리**

* 최고가 구매 조회

  ```
  ZREVRANGE buy:{productId} 0 0 WITHSCORES
  ```

---

## ✔ 5) Lua 스크립트로 원자적 매칭

1. 두 주문 상태(`active`)인지 체크
2. 두 주문 모두 `reserved`로 변경
3. ZSET에서 제거
4. 매칭 성공 데이터 반환

→ **동시성 완전 차단(중복 체결 없음)**

---

# ✔ 6) 매칭 성공 이후 (MATCHED 이벤트 후속 처리)

## **결제/정산 서비스(DB 담당)**

MATCHED 이벤트를 소비하여 DB 트랜잭션 처리:

* 구매/판매 주문 상태 → `completed`
* 주문 상태 변경 히스토리 insert
* 체결(match) 테이블 insert
* 정산 로직 처리

→ DB는 **영속성 확보·감사 목적** 중심
→ Redis는 **실시간 매칭 성능** 중심

---

# ✔ 7) DB 저장 시점 요약

| 데이터          | 시점                                    |
| ------------ | ------------------------------------- |
| 구매/판매 요청 원본  | **Kafka 이벤트 수신 직후 writer가 DB insert** |
| reserved 상태  | Redis에만 저장 (DB 저장 X)                  |
| completed 상태 | MATCHED 이벤트 처리 시 DB insert            |
| 체결 기록        | MATCHED 이벤트 처리 시 DB insert            |

---

# ✔ 최종 요약 (면접용)

* Kafka 파티션을 상품 기준으로 고정해 **직렬 처리·순서 보장**
* Redis ZSET/Bucket을 사용해 **즉시 최저·최고가 조회**
* Lua로 **원자적 매칭**, 중복 체결 절대 불가
* Redis는 실시간 매칭 캐시, DB는 영속 저장소
* 매칭 확정은 MATCHED 이벤트에서만 DB 트랜잭션으로 처리

---

필요하면 **그림(아키텍처 다이어그램)** 형태로도 재구성해줄게.
