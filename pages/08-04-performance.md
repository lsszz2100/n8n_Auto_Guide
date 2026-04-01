# 레슨 8.4: 성능 최적화

---

## n8n 성능 병목 유형

대규모 자동화를 운영하다 보면 다양한 성능 문제가 발생합니다.

| 병목 유형 | 증상 | 원인 |
|-----------|------|------|
| 메모리 부족 | 워크플로우 크래시 | 대용량 데이터 한번에 처리 |
| 실행 속도 저하 | 완료까지 수 시간 | 직렬 처리, API 호출 지연 |
| DB 연결 과부하 | 연결 오류 | 동시 실행 수 과다 |
| 타임아웃 | 실행 중단 | 긴 외부 API 응답 대기 |

---

## 핵심 최적화 1: 배치 처리

수천 건의 데이터를 한번에 처리하면 메모리가 부족합니다.

**나쁜 예: 1만 건을 한번에 처리**

```
[DB: 10,000건 조회] → [처리] → 메모리 폭발 💥
```

**좋은 예: 100건씩 배치 처리**

```
[DB: 10,000건 조회]
    ↓
[Split In Batches]
  Batch Size: 100
    ↓ (100건씩 반복)
[처리 로직]
    ↓
[다음 배치...]
```

```javascript
// Split In Batches 노드 설정
Batch Size: 100       // 한 번에 처리할 항목 수
Reset: false          // 전체 실행 시 초기화 여부
```

---

## 핵심 최적화 2: 병렬 처리

순차적 API 호출 대신 동시에 여러 요청:

**느린 방법: 순차 처리 (10개 → 10초)**
```
API 호출 1 (1초) → API 호출 2 (1초) → ... → API 호출 10 (1초)
총 10초
```

**빠른 방법: 병렬 처리 (10개 → 1초)**

```
[Split In Batches: 1개씩]
    ↓
[HTTP Request]
  ✅ "Wait for All Items" 비활성화
    ↓ (10개가 동시에 실행)
[Merge: 모든 결과 수집]
총 ~1초
```

### 병렬 워크플로우 실행

```javascript
// Code 노드로 여러 워크플로우 동시 실행
const workflowPromises = [
  $helpers.executeWorkflow('workflow-a', inputA),
  $helpers.executeWorkflow('workflow-b', inputB),
  $helpers.executeWorkflow('workflow-c', inputC),
];

const results = await Promise.all(workflowPromises);
```

---

## 핵심 최적화 3: 데이터 최소화

필요한 필드만 처리하여 메모리 절약:

```javascript
// 나쁜 예: 전체 고객 데이터 유지
// { id, name, email, address, phone, orders[], history[], ... }

// 좋은 예: 필요한 필드만 추출
const items = $input.all().map(item => ({
  json: {
    id: item.json.id,
    email: item.json.email
    // 필요한 것만!
  }
}));

return items;
```

---

## 핵심 최적화 4: 캐싱 전략

자주 사용하는 데이터는 캐싱:

```
[Schedule: 매 시간]
    ↓
[HTTP Request: 환율 API 조회]
    ↓
[Redis/DB: 환율 데이터 저장]
  key: "exchange_rate"
  ttl: 3600 (1시간)

─────────────────────────

[주문 처리 워크플로우]
    ↓
[Redis: 캐시에서 환율 조회]
  key: "exchange_rate"
    ↓
[IF: 캐시 존재?]
  True  → 캐시된 환율 사용
  False → [HTTP Request: 환율 API 직접 조회]
```

---

## 핵심 최적화 5: 데이터베이스 쿼리 최적화

```
[DB 조회 최적화]

나쁜 예:
  SELECT * FROM orders WHERE ...
  → 100개 컬럼 모두 가져옴

좋은 예:
  SELECT id, status, customer_id, amount FROM orders WHERE ...
  → 필요한 4개만 가져옴

나쁜 예:
  SELECT * FROM orders (100만 건)

좋은 예:
  SELECT * FROM orders WHERE created_at > '2026-03-01'
  LIMIT 1000
  → 조건과 제한 추가
```

---

## n8n 서버 설정 최적화

### 환경 변수 튜닝

```env
# 동시 실행 수 제한 (서버 부하 조절)
EXECUTIONS_CONCURRENCY_ACTIVE=25

# 실행 이력 보관 기간 (DB 절약)
EXECUTIONS_DATA_MAX_AGE=168  # 7일 (시간 단위)

# 큰 데이터 처리 시 타임아웃 설정
EXECUTIONS_TIMEOUT=3600      # 1시간 (초 단위)

# 메모리 설정 (Node.js)
NODE_OPTIONS=--max-old-space-size=4096  # 4GB
```

### 워커 프로세스 (대규모 처리)

```yaml
# docker-compose.yml - 워커 분리 설정
services:
  n8n-main:
    image: n8nio/n8n
    command: start
    environment:
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis

  n8n-worker:
    image: n8nio/n8n
    command: worker
    scale: 3  # 워커 3개 병렬 실행
    environment:
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis

  redis:
    image: redis:7-alpine
```

---

## 실행 시간 프로파일링

워크플로우 어느 노드가 느린지 파악:

```javascript
// Code 노드로 타이밍 측정
const startTime = Date.now();

// ... 처리 로직 ...

const elapsed = Date.now() - startTime;
console.log(`처리 시간: ${elapsed}ms`);

return [{
  json: {
    ...result,
    _timing: { elapsed, timestamp: new Date().toISOString() }
  }
}];
```

n8n UI에서 각 노드 실행 시간 확인:
- 워크플로우 실행 후 각 노드 클릭
- "Execution Time" 항목 확인

---

## 대용량 파일 처리

```javascript
// 큰 파일은 스트리밍으로 처리
// Code 노드에서 직접 처리하지 않고
// 파일을 청크로 나누어 처리

const CHUNK_SIZE = 10 * 1024 * 1024; // 10MB
const fileSize = $binary.data.fileSize;
const chunks = Math.ceil(fileSize / CHUNK_SIZE);

const chunkList = [];
for (let i = 0; i < chunks; i++) {
  chunkList.push({
    json: {
      chunkIndex: i,
      start: i * CHUNK_SIZE,
      end: Math.min((i + 1) * CHUNK_SIZE, fileSize)
    }
  });
}

return chunkList;
```

---

## 성능 모니터링 대시보드

```
[Schedule: 매 5분]
    ↓
[n8n API: 실행 중인 워크플로우 수 조회]
    ↓
[n8n API: 최근 실패 실행 조회]
    ↓
[시스템 메트릭 수집]
  - CPU 사용률
  - 메모리 사용량
  - 디스크 I/O
    ↓
[Grafana/DB: 메트릭 저장]
    ↓
[IF: 임계값 초과?]
  → [Slack: 경고 알림]
```

---

## 성능 체크리스트

- [ ] 대용량 데이터는 Split in Batches 사용
- [ ] 독립적인 API 호출은 병렬 처리
- [ ] 필요한 데이터 필드만 선택하여 전달
- [ ] 반복 조회 데이터는 캐싱 적용
- [ ] DB 쿼리에 인덱스, 조건, LIMIT 추가
- [ ] `EXECUTIONS_CONCURRENCY_ACTIVE` 적절히 설정
- [ ] 실행 이력 자동 삭제로 DB 관리
- [ ] 대규모 트래픽은 워커 분리 아키텍처 적용

---

## 핵심 요약

- 배치 처리로 메모리 부족 방지 (Split In Batches)
- 병렬 처리로 실행 속도 10배 향상
- 데이터 최소화로 메모리 절약
- 캐싱으로 반복 API 호출 제거
- 서버 환경변수 튜닝으로 동시 처리 능력 향상
- 워커 분리로 엔터프라이즈 규모 처리

**다음 레슨**: n8n 보안 강화로 안전한 자동화 환경을 구축합니다.
