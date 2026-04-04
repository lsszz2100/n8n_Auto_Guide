# 레슨 4.5: 워크플로우 실행과 모니터링

---

## 실행 모드의 2가지

### 1. 수동 실행 (Manual)
- "Execute Workflow" 버튼 클릭
- 단계별 데이터 확인 가능
- 개발 및 테스트에 사용
- 실행 기록에 "Manual" 표시

### 2. 자동 실행 (Production)
- 워크플로우 활성화 후 트리거에 의해 자동 실행
- 백그라운드에서 동작
- 실행 기록에 "Production" 표시

---

## 워크플로우 활성화/비활성화

### 활성화 방법
편집기 상단의 토글 스위치를 ON으로 전환

활성화 시 확인사항:
- 크리덴셜이 올바르게 설정되어 있는가?
- 트리거 설정이 올바른가?
- 마지막 수동 테스트가 성공했는가?

### 비활성화가 필요한 상황
- 워크플로우 수정 중
- API 서비스 점검 중
- 의도치 않은 실행을 방지할 때

---

## 실행 기록 활용

### Executions 페이지에서 확인 가능한 것

| 항목 | 설명 |
|------|------|
| 실행 ID | 각 실행의 고유 식별자 |
| 시작 시간 | 언제 실행이 시작됐는지 |
| 소요 시간 | 완료까지 걸린 시간 |
| 상태 | 성공/실패/대기 중 |
| 모드 | Manual/Production |
| 워크플로우 이름 | 어떤 워크플로우인지 |

### 특정 실행 상세 보기

실행 클릭 → 각 노드의 입력/출력 데이터 확인 가능

활용 예시:
- "왜 이 이메일이 이 고객에게 갔지?" → 실행 기록에서 해당 시점의 데이터 확인
- "API 응답 데이터가 어떻게 생겼지?" → 실행 기록에서 HTTP Request 노드의 Output 확인

---

## 실행 기록 필터링

대량의 실행 기록에서 필요한 것만 찾기:

| 필터 | 사용 예시 |
|------|-----------|
| 워크플로우 선택 | 특정 워크플로우만 보기 |
| 상태 필터 | 실패한 실행만 보기 |
| 날짜 범위 | 어제~오늘 실행만 보기 |
| 검색 | 특정 실행 ID로 검색 |

---

## 실행 상태 모니터링

### 성공률 추적

```
성공률 = 성공한 실행 수 / 전체 실행 수 × 100%

목표: 99% 이상
경고: 95% 미만
위험: 90% 미만
```

### 평균 실행 시간 추적

```
기준선 설정: 최초 1주일의 평균값
이상 감지: 평균의 3배 이상 소요 시 알림
```

---

## 모니터링 자동화

n8n으로 n8n 자체를 모니터링하는 패턴:

### 실패 알림 워크플로우

```
[Error Trigger] → [Slack: 실패 알림 발송]
설정:
  메시지: "⚠️ 워크플로우 실패!\n
           워크플로우: {{ $json.workflow.name }}\n
           시간: {{ $now.toFormat('yyyy-MM-dd HH:mm') }}\n
           에러: {{ $json.execution.error.message }}"
```

### Error Trigger 노드 설정

1. 새 워크플로우 생성
2. 노드 추가 → "Error Trigger" 검색
3. 알림 받을 워크플로우 선택 (또는 전체)
4. Slack/이메일 알림 연결

---

## 실행 기록 보관 정책

실행 기록은 저장공간을 차지합니다. 적절한 보관 기간을 설정하세요.

### 설정 방법

Settings → Execution Data:
- Executions to Keep: 최근 N개 보관
- Pruning Enabled: 오래된 기록 자동 삭제
- Pruning Time (hours): 보관 기간 (시간 단위)

### 권장 설정

| 환경 | 보관 기간 |
|------|-----------|
| 개발 | 7일 |
| 소규모 운영 | 30일 |
| 대규모 운영 | 7~14일 (용량 고려) |

---

## n8n API를 활용한 모니터링

n8n의 REST API로 외부 모니터링 시스템과 연동:

```
GET /api/v1/executions?status=error&workflowId=xxx
→ 특정 워크플로우의 실패 목록 조회

GET /api/v1/workflows
→ 모든 워크플로우 상태 조회
```

외부 대시보드(Grafana, DataDog 등)와 연동하면 실시간 모니터링 가능합니다.

---

## 실전 예제: 실패 실행 자동 재시도 시스템

**목표**: 워크플로우 실패 시 자동으로 재시도하고, 3회 초과하면 Slack 긴급 알림

```
[Error Trigger]
    ↓
[Code: 재시도 횟수 확인]
    ↓
[IF: 재시도 < 3회]
  ├── True  → [Wait: 5분] → [HTTP Request: 워크플로우 재실행]
  └── False → [Slack: 긴급 알림] → [Notion: 인시던트 기록]
```

### Error Trigger + 재시도 카운터

```javascript
// Code 노드: 재시도 횟수 추적
const workflowId = $json.workflow.id;
const executionId = $json.execution.id;
const errorMsg = $json.execution.error?.message || '알 수 없는 오류';

// Static Data를 활용한 재시도 카운터 (워크플로우 인스턴스별 유지)
const retryKey = `retry_${workflowId}`;
const retryCount = ($getWorkflowStaticData('global')[retryKey] || 0) + 1;
$getWorkflowStaticData('global')[retryKey] = retryCount;

return [{
  json: {
    workflowId,
    executionId,
    errorMsg,
    retryCount,
    shouldRetry: retryCount <= 3,
  }
}];
```

### Slack 긴급 알림 메시지

```
노드: Slack → Send Message
설정:
  Channel: #ops-alerts
  Message:
    🚨 *워크플로우 반복 실패 - 즉시 확인 필요*

    워크플로우 ID: {{ $json.workflowId }}
    에러: {{ $json.errorMsg }}
    재시도 횟수: {{ $json.retryCount }}회
    시간: {{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}

    n8n 대시보드에서 확인하세요.
```

---

## 실전 예제: 실행 성과 일일 보고서

**목표**: 매일 자정 전날 실행 통계를 집계해서 이메일로 발송

```
[Schedule Trigger: 매일 00:05]
    ↓
[HTTP Request: n8n API로 전날 실행 목록 조회]
    ↓
[Code: 통계 집계]
    ↓
[Gmail: 운영 팀에 보고서 발송]
```

### n8n API 호출 설정

```
노드: HTTP Request
설정:
  Method: GET
  URL: http://localhost:5678/api/v1/executions
  Query Parameters:
    status: all
    limit: 500
  Headers:
    X-N8N-API-KEY: {{ $env.N8N_API_KEY }}
```

### 통계 집계 코드

```javascript
// Code 노드: 일일 통계
const executions = $input.all().map(e => e.json);
const yesterday = new Date();
yesterday.setDate(yesterday.getDate() - 1);
yesterday.setHours(0, 0, 0, 0);

const todayExecs = executions.filter(e => {
  const startTime = new Date(e.startedAt);
  return startTime >= yesterday;
});

const stats = {
  total: todayExecs.length,
  success: todayExecs.filter(e => e.finished && !e.stoppedAt).length,
  failed: todayExecs.filter(e => e.status === 'error').length,
  avgDuration: 0,
};

stats.successRate = stats.total > 0
  ? ((stats.success / stats.total) * 100).toFixed(1)
  : '0';

return [{ json: stats }];
```

---

## 핵심 요약

- 수동 실행: 테스트용, 데이터 흐름 확인 가능
- 활성화 후 자동 실행: 트리거에 의해 백그라운드 동작
- Executions 페이지: 모든 실행 기록 확인 및 디버깅
- Error Trigger: 실패 시 자동 알림 워크플로우 구성
- 실행 기록 보관 정책 설정으로 용량 관리
- 실패 자동 재시도 + 임계치 초과 시 긴급 알림으로 안정성 확보
- n8n API로 일일 실행 통계 자동 집계 가능

**다음 레슨**: 에러 핸들링으로 견고한 워크플로우를 만드는 방법을 배웁니다.
