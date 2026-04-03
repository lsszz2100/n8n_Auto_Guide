# 레슨 4.4: Loop와 반복 처리

---

## n8n에서의 반복 처리

n8n에서 반복 처리는 두 가지 방식으로 동작합니다:

1. 자동 반복: 아이템 배열의 각 항목에 대해 노드가 자동 실행
2. 명시적 루프: Loop Over Items 노드로 직접 반복 제어

---

## 자동 반복: 아이템별 처리

대부분의 노드는 들어오는 아이템을 자동으로 하나씩 처리합니다.

```
[Google Sheets: 100명의 사용자 목록 조회]
    ↓ (100개 아이템)
[Gmail: 이메일 발송]
    → 자동으로 100번 실행됨!
```

별도의 루프 설정 없이 각 사용자에게 이메일이 자동으로 발송됩니다.

---

## Split In Batches (Loop Over Items)

대량의 아이템을 한꺼번에 처리하면 API 제한에 걸리거나 서버에 과부하가 생길 수 있습니다.
Split In Batches 노드로 배치 단위로 처리하세요.

### 구조

```
[1000개 아이템]
    ↓
[Split In Batches: 10개씩]
    ↓
[HTTP Request: API 호출]
    ↓
[Wait: 1초]
    ↓ (배치 완료 후 다시 위로)
[Split In Batches]
```

### 설정 방법

| 설정 | 값 | 설명 |
|------|-----|------|
| Batch Size | 10 | 한 번에 처리할 아이템 수 |
| Options > Reset | false | 전체 처리 완료 전까지 반복 |

### 실전 예시: 대량 이메일 발송

```javascript
// Split In Batches: 50명씩 처리
// Wait: 1초 (Rate Limit 방지)
// Gmail: 이메일 발송
// 총 500명: 10배치 × 50명 × 1초 간격 = 약 10초 완료
```

---

## 실전 루프 패턴

### 패턴 1: 페이지네이션 API 전체 수집

API가 한 번에 100개만 반환할 때 전체 데이터를 수집:

```
[Set: page = 1]
    ↓
[HTTP Request: /api/data?page={{ $json.page }}&limit=100]
    ↓
[IF: 결과가 100개? (더 있음)]
  ├── True ─→ [Set: page = page + 1]
  │              ↓ (루프백)
  │           [HTTP Request]
  └── False → [Aggregate: 전체 수집]
                  ↓
              [다음 처리]
```

### 패턴 2: 재시도 로직

API 호출 실패 시 최대 3번까지 재시도:

```
[Set: retryCount = 0]
    ↓
[HTTP Request: API 호출]
    ↓
[IF: 성공?]
  ├── True → [다음 처리]
  └── False → [IF: retryCount < 3]
                ├── True → [Set: retryCount += 1]
                │             ↓
                │          [Wait: 5초]
                │             ↓ (재시도)
                │          [HTTP Request]
                └── False → [에러 알림]
```

### 패턴 3: 주기적인 상태 폴링

작업 완료까지 주기적으로 상태 확인:

```
[작업 시작 요청]
    ↓
[Set: jobId = 결과 jobId]
    ↓
[Wait: 5초]
    ↓
[HTTP Request: 상태 확인 /jobs/{{ $json.jobId }}]
    ↓
[IF: status === "completed"?]
  ├── True → [결과 처리]
  └── False → [IF: status === "failed"?]
                ├── True → [에러 처리]
                └── False → 다시 Wait으로 (루프)
```

---

## Code 노드에서의 반복 처리

복잡한 반복 로직은 Code 노드의 JavaScript로 처리:

```javascript
// Run Once for All Items 모드
const items = $input.all();
const results = [];

for (const item of items) {
  const data = item.json;

  // 각 아이템 처리
  const processed = {
    id: data.id,
    name: data.name.toUpperCase(),
    score: data.scores.reduce((a, b) => a + b, 0) / data.scores.length
  };

  results.push({ json: processed });
}

return results;
```

---

## 무한 루프 방지

루프를 만들 때는 반드시 종료 조건을 명확히 해야 합니다.

### 체크리스트

- [ ] 언제 루프가 끝나는지 명확한가?
- [ ] 최대 반복 횟수를 설정했는가?
- [ ] 타임아웃이 있는가?

### 안전 카운터 패턴

```javascript
// Code 노드에서 안전 카운터
const maxRetries = 10;
let retryCount = $json.retryCount || 0;

if (retryCount >= maxRetries) {
  throw new Error(`최대 재시도 횟수(${maxRetries})를 초과했습니다.`);
}

return {
  json: {
    ...$json,
    retryCount: retryCount + 1
  }
};
```

---

## 지수 백오프(Exponential Backoff) 재시도

API 실패 시 무작정 즉시 재시도하면 서버에 부담을 줄 수 있습니다. 재시도 간격을 점점 늘리는 지수 백오프가 표준 방식입니다.

```javascript
// Code 노드: 지수 백오프 대기 시간 계산
const retryCount = $json.retryCount || 0;
const baseDelay = 1;      // 기본 1초
const maxDelay = 60;      // 최대 60초
const jitter = Math.random() * 0.3;  // 0~0.3초 랜덤 추가 (동시 재시도 방지)

const delay = Math.min(
  baseDelay * Math.pow(2, retryCount) + jitter,
  maxDelay
);

return [{
  json: {
    ...$json,
    retryDelay: delay,
    retryCount: retryCount + 1
  }
}];
```

워크플로우 구성:
```
[HTTP Request: 실패]
    ↓
[Code: 지수 백오프 계산]
  retryCount=0 → 1초 대기
  retryCount=1 → 2초 대기
  retryCount=2 → 4초 대기
  retryCount=3 → 8초 대기
    ↓
[Wait: {{ $json.retryDelay }}초]
    ↓
[IF: retryCount < 5]
  True → 재시도
  False → Dead Letter Queue
```

---

## $workflow.setStaticData() — 실행 간 상태 유지

워크플로우 실행이 끝나도 데이터를 보존하고 싶을 때 사용합니다.

```javascript
// Code 노드 (Run Once for All Items)

// 마지막으로 처리한 ID 저장 (증분 동기화)
const staticData = $getWorkflowStaticData('global');
const lastProcessedId = staticData.lastId || 0;

// 새로운 데이터만 필터링
const newItems = $input.all()
  .filter(item => item.json.id > lastProcessedId);

// 처리 후 마지막 ID 업데이트
if (newItems.length > 0) {
  const maxId = Math.max(...newItems.map(i => i.json.id));
  staticData.lastId = maxId;
}

return newItems;
```

활용 사례:
- 증분 동기화: 마지막으로 처리한 레코드 ID 기억
- 실행 횟수 추적: 주간 누적 처리 건수
- 임시 캐시: 자주 조회하는 설정값 저장

---

## 병렬 처리 vs 순차 처리

### 순차 처리 (기본)

아이템을 하나씩 순서대로 처리합니다.
- 안전하지만 느림
- Rate Limit 있는 API에 적합

### 병렬 처리

여러 아이템을 동시에 처리합니다.
- 빠르지만 주의 필요
- n8n에서는 Execute Workflow 노드의 "Execute in parallel" 옵션 사용

---

## 핵심 요약

- n8n은 기본적으로 아이템을 자동으로 하나씩 처리
- Split In Batches로 대량 데이터를 배치 단위로 처리
- Wait 노드와 함께 사용하여 Rate Limit 방지
- 페이지네이션 API, 재시도, 상태 폴링에 루프 패턴 활용
- 지수 백오프: 재시도 간격을 1→2→4→8초로 늘려 서버 보호
- $getWorkflowStaticData()로 실행 간 상태(마지막 처리 ID 등) 유지
- 무한 루프를 방지하기 위한 종료 조건과 최대 횟수 설정 필수

**다음 레슨**: 워크플로우 실행을 모니터링하고 관리하는 방법을 배웁니다.
