# 레슨 4.6: 에러 핸들링 완전 가이드

---

## 에러는 언제든 발생한다

아무리 잘 만든 워크플로우도 에러는 피할 수 없습니다:
- API 서버 일시 장애
- 예상치 못한 데이터 형식
- 네트워크 타임아웃
- 인증 만료

에러를 막는 것이 아니라 잘 처리하는 것이 목표입니다.

---

## 에러 처리 방법 3가지

### 방법 1: Stop and Error (기본값)

에러 발생 시 워크플로우 즉시 중단.

```
[API 호출] → ❌ 에러 → 워크플로우 중단
```

단순한 워크플로우에서 에러가 발생했을 때 이후 처리를 막아야 하는 경우 적합.

---

### 방법 2: Continue (에러 무시)

에러가 발생해도 다음 노드를 계속 실행.

노드 설정 → Settings → On Error: Continue

```
[API 호출] → ❌ 에러 → 다음 노드 계속 실행 (에러 데이터 전달)
```

주의: 에러 데이터를 받은 다음 노드에서 오류 여부를 확인해야 함.

---

### 방법 3: Continue (에러를 다른 출력으로)

에러를 별도 경로로 처리합니다. 가장 강력한 방법.

노드 설정 → Settings → On Error: Continue (using error output)

```
[API 호출] → 성공 출력 → [정상 처리]
           → 에러 출력 → [에러 처리]
```

---

## Error Trigger 워크플로우

워크플로우 전체에서 처리되지 않은 에러를 캐치하는 별도 워크플로우:

### 설정 방법

1. 새 워크플로우 생성 ("에러 알림 워크플로우")
2. Error Trigger 노드 추가
3. 원하는 알림 노드 연결 (Slack, 이메일 등)
4. 원래 워크플로우 설정 → Settings → Error Workflow → 생성한 워크플로우 선택

### 에러 알림 워크플로우 예시

```javascript
// Slack 메시지 내용
const workflow = $json.workflow.name;
const error = $json.execution.error.message;
const executionId = $json.execution.id;
const time = $now.toFormat('yyyy-MM-dd HH:mm:ss');

return [{
  json: {
    message: `🚨 워크플로우 오류 발생!\n\n워크플로우: ${workflow}\n실행 ID: ${executionId}\n시간: ${time}\n오류: ${error}`
  }
}];
```

---

## 노드별 에러 처리 설정

각 노드에서 개별적으로 에러 처리 방법 설정:

```
[HTTP Request: 외부 API 호출]
Settings:
  On Error: Continue (using error output)
  Retry on Fail: 3회
  Wait Between Retries: 5000ms (5초)
```

### 자동 재시도 설정

| 설정 | 값 | 설명 |
|------|-----|------|
| Retry on Fail | true | 실패 시 재시도 활성화 |
| Max Tries | 3 | 최대 재시도 횟수 |
| Wait Between Tries | 5000 | 재시도 간격 (밀리초) |

---

## 실전 에러 처리 패턴

### 패턴 1: Try-Catch 패턴

```
[API 호출]
  ├── 성공 → [정상 처리] → [성공 로그]
  └── 실패 → [에러 정보 추출] → [알림 발송] → [실패 로그]
```

### 패턴 2: Fallback 패턴

주 API가 실패하면 대체 방법 사용:

```
[주 API 호출]
  ├── 성공 → [처리]
  └── 실패 → [대체 API 호출]
                ├── 성공 → [처리]
                └── 실패 → [에러 알림]
```

### 패턴 3: Dead Letter Queue 패턴

처리 실패한 데이터를 별도로 보관하여 나중에 재처리:

```
[API 호출]
  ├── 성공 → [처리]
  └── 실패 → [Google Sheets: 실패 데이터 기록]
                       ↑
              (나중에 수동으로 재처리)
```

---

## 에러 데이터 구조

에러 발생 시 n8n이 제공하는 에러 정보:

```json
{
  "error": {
    "message": "Request failed with status code 404",
    "code": "ERR_HTTP_RESPONSE_TIMEOUT",
    "statusCode": 404,
    "description": "Not Found"
  }
}
```

에러 처리 노드에서 접근:
```javascript
const errorMessage = $json.error?.message || "알 수 없는 오류";
const statusCode = $json.error?.statusCode || 0;
```

---

## 알림 메시지 잘 작성하기

나쁜 에러 알림:
```
"워크플로우 실패"
```

좋은 에러 알림:
```
🚨 [주문 처리 워크플로우] 오류 발생

📋 상세 정보:
- 시간: 2026-04-01 15:30:00
- 실행 ID: exec_12345
- 실패 노드: "결제 API 호출"
- 오류 메시지: "Request timeout after 30s"
- 영향: 주문 5건 처리 대기 중

🔗 실행 기록: https://n8n.company.com/workflow/runs/exec_12345

⚡ 조치 필요: 결제 API 서버 상태 확인
```

좋은 알림은 즉시 상황을 파악하고 조치할 수 있게 해줍니다.

---

## 실전 예제: 외부 API 호출 완전 에러 처리

API 호출에서 발생하는 다양한 에러를 실전에서 처리하는 완전한 예제입니다.

### 완성 워크플로우

```
[HTTP Request: 결제 API]
    ├── 성공 → [결제 완료 처리] → [Gmail: 영수증 발송]
    └── 에러 → [Code: 에러 분류]
                  ├── 일시적 오류 (429, 503) → [Wait: 10초] → [HTTP Request: 재시도]
                  ├── 인증 오류 (401, 403)   → [Slack: 운영팀 즉시 알림]
                  └── 데이터 오류 (400, 422) → [Google Sheets: 실패 로그] → [Gmail: 처리 실패 안내]
```

### 에러 분류 코드

```javascript
// Code 노드: HTTP 상태 코드로 에러 유형 분류
const error = $json.error || {};
const statusCode = error.statusCode || error.code || 0;

let errorType, shouldRetry, notifyOps;

if ([429, 503, 504].includes(statusCode)) {
  errorType = 'temporary';   // 일시적 오류 - 재시도 가능
  shouldRetry = true;
  notifyOps = false;
} else if ([401, 403].includes(statusCode)) {
  errorType = 'auth';        // 인증 오류 - 즉시 운영팀 알림
  shouldRetry = false;
  notifyOps = true;
} else if ([400, 422].includes(statusCode)) {
  errorType = 'data';        // 데이터 오류 - 요청 데이터 확인 필요
  shouldRetry = false;
  notifyOps = false;
} else {
  errorType = 'unknown';
  shouldRetry = false;
  notifyOps = true;
}

return [{
  json: {
    ...$json,
    errorType,
    shouldRetry,
    notifyOps,
    statusCode,
    errorMessage: error.message || '알 수 없는 오류',
  }
}];
```

### Switch로 에러 유형별 분기

```
Switch 노드:
  Field: {{ $json.errorType }}
  Case 1: "temporary" → Wait + 재시도 경로
  Case 2: "auth"      → 운영팀 알림 경로
  Case 3: "data"      → 실패 로그 경로
  Default: 알 수 없는 오류 경로
```

---

## 실전 예제: Dead Letter Queue 구현

처리 실패한 데이터를 안전하게 보관하고 나중에 재처리하는 패턴입니다.

### Dead Letter Queue 워크플로우

```
[에러 출력]
    ↓
[Code: 실패 레코드 구성]
    ↓
[Google Sheets: "실패 큐" 시트에 기록]
    ↓
[Slack: 일일 실패 건수 알림]
```

### 실패 레코드 구성 코드

```javascript
// Code 노드: 재처리 가능한 형태로 실패 기록
const originalData = $json;

return [{
  json: {
    id: `fail_${Date.now()}`,
    timestamp: new Date().toISOString(),
    workflowName: $workflow.name,
    errorMessage: originalData.error?.message || '알 수 없는 오류',
    originalPayload: JSON.stringify(originalData),
    retryCount: 0,
    status: 'pending',        // pending / retrying / resolved / abandoned
    resolvedAt: null,
  }
}];
```

### 재처리 워크플로우 (수동 실행용)

```
[Manual Trigger]
    ↓
[Google Sheets: status="pending" 행 조회]
    ↓
[Loop Over Items: 각 실패 건 처리]
    ├── [Code: originalPayload 파싱]
    ├── [HTTP Request: 원래 API 재호출]
    ├── 성공 → [Google Sheets: status="resolved" 업데이트]
    └── 실패 → [Google Sheets: retryCount+1, status="abandoned" (3회 초과)]
```

---

## 자주 발생하는 에러 케이스

### Case 1: API Rate Limit (429)

```
증상: "Too Many Requests" 에러, 429 상태 코드
원인: 짧은 시간에 너무 많은 API 호출

해결:
1. HTTP Request 노드 → Retry on Fail: 활성화
2. Wait Between Tries: 60000 (1분)
3. 또는 Loop 앞에 Wait 노드 추가 (1~2초)
```

### Case 2: OAuth 토큰 만료

```
증상: "Unauthorized" 또는 "Token expired" 에러
원인: OAuth 액세스 토큰 만료 (보통 1시간)

해결:
1. n8n은 대부분의 OAuth를 자동 갱신함
2. 자동 갱신 실패 시: Credentials → 해당 서비스 → Reconnect
3. 장기 미사용 후 재시작 시 발생 빈번
```

### Case 3: null/undefined 데이터

```
증상: "Cannot read property of undefined" 에러
원인: 예상한 필드가 없는 데이터가 들어옴

해결 (방어적 코딩):
const value = $json?.nested?.field ?? '기본값';
// Optional chaining (?.) + Nullish coalescing (??) 사용
```

---

## 핵심 요약

- 에러 처리 방법 3가지: Stop, Continue, Continue with Error Output
- Error Trigger: 워크플로우 전체 에러를 캐치하는 별도 워크플로우
- 자동 재시도: Max Tries + Wait Between Tries 설정
- HTTP 상태 코드로 에러 유형 분류 후 유형별 대응
- 실패 데이터를 Dead Letter Queue에 보관하여 나중에 재처리
- 유용한 에러 알림에는 시간, 위치, 내용, 영향, 조치방법 포함

**다음 레슨**: 워크플로우의 문제를 찾고 해결하는 디버깅 기법을 배웁니다.
