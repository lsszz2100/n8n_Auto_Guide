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

## 실전 예제 모음

### 예제 1: 대용량 Google Sheets (10만 행) 배치 처리 최적화

Google Sheets에 저장된 10만 건의 고객 데이터를 읽어서 처리할 때, 한번에 로드하면 메모리가 부족해 워크플로우가 크래시됩니다. 페이지 단위로 나누어 처리하는 패턴입니다.

```
워크플로우 이름: "대용량 고객 데이터 일괄 처리"
트리거: Schedule (매일 새벽 2시)

[Schedule Trigger]
    ↓
[Code: 처리 상태 초기화]
  return [{
    json: {
      currentPage: 1,
      pageSize: 500,
      totalProcessed: 0,
      errors: 0,
      continueLoop: true
    }
  }];
    ↓
[루프 시작: Loop Over Items]
  ↓ (continueLoop가 true인 동안 반복)
[HTTP Request: Google Sheets API로 페이지 조회]
  Method: GET
  URL: https://sheets.googleapis.com/v4/spreadsheets/{{ $vars.SHEET_ID }}/values/Sheet1
  Query Parameters:
    majorDimension: ROWS
    valueRenderOption: UNFORMATTED_VALUE
  Headers:
    Authorization: Bearer {{ $credentials.googleSheetsOAuth2.accessToken }}

  // Google Sheets는 직접 페이징을 지원하지 않으므로
  // 전체 조회 후 슬라이싱 처리
    ↓
[Code: 페이지 슬라이싱 + 처리]
  const allRows = $json.values || [];
  const headers = allRows[0];
  const dataRows = allRows.slice(1); // 헤더 제외

  const page = $('Code: 처리 상태 초기화').item.json.currentPage;
  const pageSize = 500;
  const start = (page - 1) * pageSize;
  const end = start + pageSize;
  const chunk = dataRows.slice(start, end);

  if (chunk.length === 0) {
    // 처리 완료
    return [{ json: { continueLoop: false, done: true } }];
  }

  // 행을 객체로 변환
  const records = chunk.map(row => {
    const obj = {};
    headers.forEach((h, i) => { obj[h] = row[i] ?? ""; });
    return obj;
  });

  return records.map(r => ({ json: r }));
    ↓
[Split In Batches: 50개씩 처리]
  Batch Size: 50
    ↓
[Code: 각 레코드 처리 로직]
  // 예: 이메일 유효성 검사 + 상태 업데이트
  const items = $input.all();
  const results = items.map(item => {
    const { email, phone, status } = item.json;
    const isValidEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

    return {
      json: {
        ...item.json,
        processed: true,
        emailValid: isValidEmail,
        processedAt: new Date().toISOString()
      }
    };
  });
  return results;
    ↓
[Postgres: 처리 결과 DB 저장]
  Operation: Insert
  Table: customer_processed
  Columns: email, phone, status, processed, email_valid, processed_at
    ↓
[Code: 다음 페이지로 상태 업데이트]
  const prev = $('Code: 처리 상태 초기화').item.json;
  const processed = $input.all().length;

  return [{
    json: {
      currentPage: prev.currentPage + 1,
      pageSize: prev.pageSize,
      totalProcessed: prev.totalProcessed + processed,
      continueLoop: true
    }
  }];
    ↓
[루프로 돌아가기]
    ↓ (continueLoop === false 시 탈출)
[Slack: 처리 완료 보고]
  총 {{ $json.totalProcessed }}건 처리 완료
```

---

### 예제 2: API 속도 제한 우회 (Rate Limit 처리 패턴)

외부 API가 분당 60회 제한을 두고 있을 때, 자동으로 대기하고 재시도하는 패턴입니다. 429 응답 코드를 감지하여 Retry-After 헤더 값만큼 대기 후 재시도합니다.

```
워크플로우 이름: "Rate Limit 안전 API 호출 패턴"

[HTTP Request: 외부 API 호출]
  Method: GET
  URL: https://api.example.com/data/{{ $json.itemId }}
  Headers:
    Authorization: Bearer {{ $vars.API_TOKEN }}
  Options:
    Ignore Response Code: true  // 429도 에러 없이 수신
    ↓
[Code: Rate Limit 감지 및 처리]
  const statusCode = $json.$response?.statusCode || 200;
  const headers = $json.$response?.headers || {};

  if (statusCode === 429) {
    // Retry-After 헤더에서 대기 시간 추출
    const retryAfter = parseInt(headers["retry-after"] || "60", 10);
    const waitMs = retryAfter * 1000;

    return [{
      json: {
        rateLimited: true,
        waitSeconds: retryAfter,
        waitMs,
        originalItemId: $json.itemId,
        retryAt: new Date(Date.now() + waitMs).toISOString()
      }
    }];
  }

  if (statusCode >= 500) {
    return [{
      json: {
        serverError: true,
        statusCode,
        originalItemId: $json.itemId
      }
    }];
  }

  // 정상 응답
  return [{
    json: {
      rateLimited: false,
      serverError: false,
      data: $json,
      originalItemId: $json.itemId
    }
  }];
    ↓
[Switch: 응답 유형 분기]
  rateLimited === true →
    [Wait: Retry-After만큼 대기]
      Amount: {{ $json.waitSeconds }}
      Unit: seconds
      ↓
    [HTTP Request: 동일 API 재호출]
      (위의 API 호출 노드로 연결)

  serverError === true →
    [Code: 재시도 횟수 체크]
      const retries = $json.retries || 0;
      if (retries >= 3) {
        return [{ json: { failed: true, reason: "서버 오류 3회 초과" } }];
      }
      return [{ json: { ...item.json, retries: retries + 1 } }];
      ↓
    [Wait: 10초 대기 후 재시도]

  정상 →
    [다음 처리 단계...]
```

호출 속도를 처음부터 제한하는 사전 방어 패턴:

```javascript
// Code 노드: API 호출 전 속도 제한 준수
// 분당 60회 제한 = 요청 간격 최소 1초
const items = $input.all();
const DELAY_MS = 1100; // 1.1초 간격 (여유 포함)

// 각 아이템에 지연 시간 인덱스 부여
return items.map((item, index) => ({
  json: {
    ...item.json,
    delayMs: index * DELAY_MS,
    scheduledAt: new Date(Date.now() + index * DELAY_MS).toISOString()
  }
}));
// 이후 각 아이템의 delayMs를 Wait 노드에 활용
```

---

### 예제 3: 병렬 처리로 이미지 100장 동시 리사이즈

S3에 업로드된 이미지 100장을 순차 처리하면 100초가 걸리지만, 병렬로 처리하면 10~15초로 단축됩니다. n8n의 Execute Workflow 노드를 활용한 병렬화 패턴입니다.

서브 워크플로우: "이미지 리사이즈 단일 처리"

```
[Execute Workflow Trigger]
  수신 데이터:
    s3Key: 원본 이미지 S3 키
    targetSizes: [{ width: 800, suffix: "lg" }, { width: 400, suffix: "md" }, { width: 200, suffix: "sm" }]
    outputBucket: 결과 저장할 S3 버킷
    ↓
[HTTP Request: S3에서 이미지 다운로드]
  Method: GET
  URL: https://{{ $json.outputBucket }}.s3.amazonaws.com/{{ $json.s3Key }}
  Response Format: file
    ↓
[Split In Batches: targetSizes 기준으로 분리]
  // 각 크기별로 처리
    ↓
[Code: 리사이즈 설정 구성]
  const { width, suffix } = $json;
  const originalKey = $('Execute Workflow Trigger').item.json.s3Key;
  const outputKey = originalKey.replace(/(\.[^.]+)$/, `-${suffix}$1`);

  return [{
    json: {
      targetWidth: width,
      outputKey,
      quality: 85
    }
  }];
    ↓
[HTTP Request: 이미지 처리 외부 람다 호출]
  Method: POST
  URL: {{ $vars.IMAGE_RESIZE_LAMBDA_URL }}
  Body:
    {
      "imageData": "{{ $binary.data }}",
      "width": {{ $json.targetWidth }},
      "quality": {{ $json.quality }},
      "outputKey": "{{ $json.outputKey }}",
      "bucket": "{{ $vars.OUTPUT_BUCKET }}"
    }
    ↓
[Code: 처리 결과 반환]
  return [{
    json: {
      success: true,
      outputKey: $json.outputKey,
      url: $json.imageUrl
    }
  }];
```

메인 워크플로우: "100장 병렬 리사이즈 오케스트레이터"

```
[Schedule Trigger 또는 Webhook]
    ↓
[HTTP Request: S3 버킷에서 이미지 목록 조회]
  Method: GET
  URL: https://s3.amazonaws.com/{{ $vars.SOURCE_BUCKET }}?list-type=2&prefix=uploads/
    ↓
[Code: S3 XML 파싱 + 100개 추출]
  // S3 응답 XML에서 Key 목록 파싱
  const keys = $json.ListBucketResult?.Contents
    ?.filter(item => /\.(jpg|jpeg|png|webp)$/i.test(item.Key[0]))
    .map(item => item.Key[0])
    .slice(0, 100) || [];

  return keys.map(key => ({
    json: {
      s3Key: key,
      targetSizes: [
        { width: 1200, suffix: "xl" },
        { width: 800, suffix: "lg" },
        { width: 400, suffix: "md" },
        { width: 150, suffix: "thumb" }
      ],
      outputBucket: process.env.OUTPUT_BUCKET
    }
  }));
    ↓
[Split In Batches: 10개씩 병렬 그룹]
  Batch Size: 10
    ↓
[Execute Workflow: 이미지 리사이즈 단일 처리]
  Wait for Completion: false  // 비동기 병렬 실행 핵심
  Workflow: "이미지 리사이즈 단일 처리"
  Input Data: {{ $json }}
    ↓ (10개 동시 실행 → 완료 후 다음 배치)
[다음 배치 처리...]
    ↓
[Slack: 처리 완료 알림]
  100장 이미지 리사이즈 완료
  처리 시간: {{ $execution.duration }}초
```

---

## 핵심 요약

- 배치 처리로 메모리 부족 방지 (Split In Batches)
- 병렬 처리로 실행 속도 10배 향상
- 데이터 최소화로 메모리 절약
- 캐싱으로 반복 API 호출 제거
- 서버 환경변수 튜닝으로 동시 처리 능력 향상
- 워커 분리로 엔터프라이즈 규모 처리

**다음 레슨**: n8n 보안 강화로 안전한 자동화 환경을 구축합니다.
