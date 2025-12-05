# 레이트 리밋

## 토큰 버킷



토큰 수를 정해놓고 처리하는 방식으로 요청마다 토큰을 소비하여 속도를 제한 하는 방식이며
일정 속도로 지속적으로 capacity만큼 채움, 따라서 토큰이 충분히 쌓여있다면 한번에 처리가능하여 버스트 트래픽 허용해야 할때 사용함

- 장점: 버스트 트래픽 허용, 특정 속도 이상으로는 절대 못넘어가도록 제어 가능
- 단점: 토큰 리필 주기 관리 필요
- 사용 예시: 버스트 허용, 평균 요청 속도 유지하면 될때 
``` python
import time

class TokenBucket:
    def __init__(self, capacity=10, refill_rate=1.0):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = float(capacity)
        self.last_refill = time.time()

    def consume(self):
        now = time.time()
        # 리필 계산: 경과 시간 * refill_rate
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```
 

## fixed window

정해진 시간 단위별 카운트로 제한하는 방식
간단한 방식으로 트래픽이 많지 않은 백오피스, 운영 기능에 적용하기 좋음
경계에서 트래픽이 두배가 되거나 계속 거부하는 문제있을수 잇음

- 장점: 구현 쉬움, 메모리 효율적
- 단점: 윈도우 경계 문제
- 사용 예시: 트래픽 폭주가 거의 없는 서비스, 백오피스, 운영 사이트

## sliding window log

타임스탬프 기반으로 동작하므로 정확하게 rate 제한을 보장할 수 있음
타임스탬프를 관리하므로 메모리 부담이 큼

- 장점: 정확도
- 단점: 메모리 부담 큼
- 사용 예시: 정확한 rate 보장이 필요할때 

``` python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, window_size=60, max_requests=10):
        self.window_size = window_size
        self.max_requests = max_requests
        self.timestamps = deque()

    def is_allowed(self, now=None):
        if now is None:
            now = time.time()
        # 오래된 타임스탬프 제거 (O(1) amortized)
        while self.timestamps and now - self.timestamps[0] > self.window_size:
            self.timestamps.popleft()
        # 카운트 초과 시 차단
        if len(self.timestamps) >= self.max_requests:
            return False
        self.timestamps.append(now)
        return True
```

## sliding window counter

슬라이딩 윈도우 로그 방식보다는 정확도 떨어지나 효율이 좋음
카운트만 저장하면 되므로 메모리 효율성이 좋음

- 장점: 정확도, 성능 균형
- 단점: 로그 방식보다 정확도 떨어짐
- 사용 예시: 대규모 트래픽, 정확도, 메모리 효율 필요시

``` python
import time
from datetime import datetime
from math import floor

class SlidingWindowCounter:
    def __init__(self, window_size=10, max_requests=5, bucket_size=1):
        self.window_size = window_size
        self.max_requests = max_requests
        self.bucket_size = bucket_size
        self.buckets = {}  # {bucket_key: count}

    def _get_bucket_key(self, timestamp):
        return floor(timestamp / self.bucket_size)

    def is_allowed(self, now=None):
        if now is None:
            now = time.time()
        current_bucket = self._get_bucket_key(now)
        start_bucket = self._get_bucket_key(now - self.window_size)

        print(start_bucket, current_bucket)
        
        # Bucket 합산: 슬라이딩 윈도우 근사
        total = sum(self.buckets.get(bucket_key, 0) 
                    for bucket_key in range(start_bucket, current_bucket + 1))
        
        # 오래된 bucket 삭제 (메모리 최적화)
        for key in list(self.buckets.keys()):
            if key < start_bucket:
                del self.buckets[key]
        
        if total >= self.max_requests:
            return False

        self.buckets[current_bucket] = self.buckets.get(current_bucket, 0) + 1
        return True
```


| 알고리즘                | 정확도 | 버스트   | 메모리   | 구현 난이도 | 적합한 케이스             |
| ------------------- | --- | ----- | ----- | ------ | ------------------- |
| **Token Bucket**    | 중간  | 허용    | 낮음    | 쉬움     | UX 중시 / 버스트 허용 서비스  |
| **Fixed Window**    | 낮음  | 매우 허용 | 매우 낮음 | 매우 쉬움  | 단순 정책, 내부 API       |
| **Sliding Log**     | 최고  | 제한적   | 높음    | 중간     | 보안·정확도 중요 API       |
| **Sliding Counter** | 높음  | 낮음    | 낮음    | 중간     | 대규모 트래픽, Gateway 레벨 |
