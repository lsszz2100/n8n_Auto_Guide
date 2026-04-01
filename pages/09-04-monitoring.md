# 레슨 9.4: 모니터링과 알림

---

## 모니터링이 필요한 이유

자동화는 24시간 혼자 돌아갑니다. 아무도 지켜보지 않으면 문제가 몇 시간씩 방치될 수 있습니다.

**모니터링 없이 발생하는 일:**
```
새벽 2시: 외부 API 인증 만료
새벽 2시~오전 9시: 7시간 동안 워크플로우 실패 반복
오전 9시: 직원 출근 후 발견
→ 7시간치 이메일 미처리, 주문 미처리
```

**모니터링 있으면:**
```
새벽 2시: API 실패 감지
새벽 2시 01분: Slack/문자로 즉시 알림
새벽 2시 10분: 담당자 확인 후 API 키 갱신
새벽 2시 15분: 자동화 정상 복구
```

---

## 1. n8n 내장 오류 알림

### 워크플로우 오류 발생 시 Slack 알림

```
[Error Trigger] ← 모든 워크플로우 오류 캐치
    ↓
[Slack: 오류 알림]
  Channel: #n8n-alerts
  Message: |
    🚨 *워크플로우 오류 발생*

    *워크플로우*: {{ $json.workflow.name }}
    *오류 노드*: {{ $json.execution.error.node.name }}
    *오류 메시지*: {{ $json.execution.error.message }}
    *실행 ID*: {{ $json.execution.id }}
    *발생 시각*: {{ $now.format('YYYY-MM-DD HH:mm:ss') }}

    🔗 실행 확인: {{ $vars.N8N_URL }}/executions/{{ $json.execution.id }}
```

**Error Trigger 설정:**
```
워크플로우 설정 → Error Workflow → 알림 워크플로우 선택
```

---

## 2. 헬스체크 모니터링

### 정기 헬스체크 워크플로우

```
[Schedule: 매 5분]
    ↓
[HTTP Request: n8n 상태 확인]
  URL: {{ $vars.N8N_URL }}/healthz
  Timeout: 10초
    ↓
[IF: 응답 코드 200?]
  False → [Slack: ❌ n8n 서버 응답 없음!]
           [SMS: 긴급 담당자 문자 발송]
```

### 주요 워크플로우 활성화 확인

```
[Schedule: 매 시간]
    ↓
[n8n API: 활성 워크플로우 조회]
    ↓
[Code: 필수 워크플로우 확인]

const criticalWorkflows = [
  '주문처리-자동화',
  '이메일-분류',
  '재고-모니터링'
];

const activeNames = $input.all()
  .map(item => item.json.name);

const missingWorkflows = criticalWorkflows.filter(
  name => !activeNames.includes(name)
);

if (missingWorkflows.length > 0) {
  // 알림 트리거
  return [{ json: { alert: true, missing: missingWorkflows } }];
}
    ↓
[IF: 누락된 필수 워크플로우?]
  → [Slack: ⚠️ 필수 워크플로우 비활성화 감지]
```

---

## 3. 실행 성공률 모니터링

```
[Schedule: 매 시간]
    ↓
[n8n API: 최근 1시간 실행 조회]
  GET /executions?limit=500&startedAfter={{ $now.minus(1, 'hour').toISO() }}
    ↓
[Code: 성공률 계산]

const executions = $input.all();
const total = executions.length;
const errors = executions.filter(e => e.json.status === 'error').length;
const successRate = total > 0
  ? ((total - errors) / total * 100).toFixed(1)
  : 100;

return [{
  json: {
    total,
    errors,
    successRate: parseFloat(successRate),
    period: '1시간'
  }
}];
    ↓
[IF: successRate < 90%]
  → [Slack: ⚠️ 워크플로우 성공률 저하: {{ $json.successRate }}%]
```

---

## 4. 외부 서비스 모니터링

의존하는 외부 API들의 상태도 감시:

```
[Schedule: 매 15분]
    ↓
[HTTP Request: OpenAI API 상태]
  URL: https://status.openai.com/api/v2/status.json
    ↓
[HTTP Request: Slack API 상태]
  URL: https://status.slack.com/api/v2.0.0/current
    ↓
[Code: 이상 감지]

const services = $input.all();
const incidents = services.filter(s =>
  s.json.status?.indicator !== 'none' &&
  s.json.status?.description !== 'All Systems Operational'
);

if (incidents.length > 0) {
  return [{ json: { alert: true, incidents } }];
}
    ↓
[IF: 이상 감지?]
  → [Slack: 🔴 외부 서비스 장애 감지]
```

---

## 5. 일일 운영 리포트

```
[Schedule: 매일 오전 9시]
    ↓
[n8n API: 어제 실행 통계 조회]
    ↓
[Code: 통계 계산 및 리포트 구성]

const yesterday = $now.minus(1, 'day').startOf('day');
const executions = /* API 조회 결과 */;

const stats = {
  total: executions.length,
  success: executions.filter(e => e.status === 'success').length,
  error: executions.filter(e => e.status === 'error').length,
  avgDuration: executions.reduce((sum, e) => sum + e.duration, 0) / executions.length,
};

// 워크플로우별 실행 횟수 Top 5
const byWorkflow = {};
executions.forEach(e => {
  byWorkflow[e.workflowName] = (byWorkflow[e.workflowName] || 0) + 1;
});

const top5 = Object.entries(byWorkflow)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 5);
    ↓
[Slack: 일일 운영 리포트]
  Channel: #ops-daily
  Blocks:
    📊 n8n 일일 운영 리포트 ({{ yesterday.format('YYYY-MM-DD') }})

    총 실행: {{ $json.stats.total }}건
    성공: {{ $json.stats.success }}건 ✅
    실패: {{ $json.stats.error }}건 ❌
    성공률: {{ ($json.stats.success / $json.stats.total * 100).toFixed(1) }}%
    평균 실행 시간: {{ ($json.stats.avgDuration / 1000).toFixed(1) }}초

    🏆 가장 많이 실행된 워크플로우:
    {{ $json.top5 }}
```

---

## 6. Grafana + Prometheus 모니터링

대규모 운영을 위한 메트릭 수집:

```yaml
# docker-compose.yml에 추가
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'n8n'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['n8n:5678']
```

```env
# n8n에서 메트릭 활성화
N8N_METRICS=true
N8N_METRICS_PREFIX=n8n_
```

**Grafana 대시보드 지표:**
- `n8n_workflow_executions_total` - 총 실행 수
- `n8n_workflow_execution_duration_seconds` - 실행 시간
- `n8n_active_workflows` - 활성 워크플로우 수

---

## 7. 알림 채널 설정

| 우선순위 | 알림 채널 | 응답 기대 시간 |
|----------|-----------|----------------|
| 긴급 (P1) | 전화/SMS | 5분 이내 |
| 높음 (P2) | Slack DM | 30분 이내 |
| 보통 (P3) | Slack 채널 | 4시간 이내 |
| 낮음 (P4) | 이메일 | 24시간 이내 |

```
오류 분류:
P1: 모든 워크플로우 중단, 서버 다운
P2: 중요 워크플로우(주문, 결제) 오류
P3: 보조 워크플로우 오류, 성공률 90% 미만
P4: 단일 실행 실패, 외부 서비스 일시 장애
```

---

## 모니터링 체크리스트

- [ ] Error Trigger로 모든 오류 알림 설정
- [ ] 5분마다 헬스체크 실행
- [ ] 필수 워크플로우 활성화 상태 정기 확인
- [ ] 일일 운영 리포트 자동 발송
- [ ] 외부 API 의존성 모니터링
- [ ] 알림 에스컬레이션 규칙 설정

---

## 실전 예제 모음

### 예제 1: 실행 실패 감지 + 자동 재시도 + Slack 에스컬레이션

워크플로우 실패를 감지하면 자동으로 재실행을 시도하고, 재시도도 실패하면 담당자에게 에스컬레이션하는 패턴입니다.

```
워크플로우 이름: "장애 감지 자동 재시도 에스컬레이션"
트리거: Schedule (매 5분)

[Schedule Trigger]
  Rule: */5 * * * *
    ↓
[HTTP Request: 최근 5분 실패 실행 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/executions
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    status: error
    limit: 50
    startedAfter: {{ $now.minus(5, 'minutes').toISO() }}
    ↓
[Code: 재시도 대상 필터링]
  const executions = $json.data || [];

  if (executions.length === 0) {
    return [{ json: { noFailures: true } }];
  }

  // 워크플로우별로 최신 실패 1건씩 선택
  const latest = {};
  for (const exec of executions) {
    const wfId = exec.workflowId;
    if (!latest[wfId] || new Date(exec.startedAt) > new Date(latest[wfId].startedAt)) {
      latest[wfId] = exec;
    }
  }

  return Object.values(latest).map(exec => ({
    json: {
      executionId: exec.id,
      workflowId: exec.workflowId,
      workflowName: exec.workflowData?.name || "알 수 없음",
      errorMessage: exec.data?.resultData?.error?.message || "오류 내용 없음",
      failedAt: exec.startedAt,
      retryKey: `retry-${exec.workflowId}`,
      alreadyRetried: false
    }
  }));
    ↓
[IF: noFailures === true]
  True → [종료: 이상 없음]
  False →
    [Split In Batches: 1개씩]
      ↓
    [Redis: 재시도 이력 조회]
      Operation: Get
      Key: {{ $json.retryKey }}
      // 이전에 재시도한 적 있는지 확인
      ↓
    [Code: 재시도 여부 판단]
      const retryCount = parseInt($json.value || "0", 10);
      const MAX_RETRY = 2;

      return [{
        json: {
          ...$('Split In Batches').item.json,
          retryCount,
          shouldRetry: retryCount < MAX_RETRY,
          shouldEscalate: retryCount >= MAX_RETRY
        }
      }];
      ↓
    [Switch: 처리 방식 분기]
      shouldRetry === true →
        [Redis: 재시도 횟수 증가]
          Operation: Set
          Key: {{ $json.retryKey }}
          Value: {{ $json.retryCount + 1 }}
          TTL: 3600  // 1시간 후 만료
          ↓
        [HTTP Request: 워크플로우 재실행]
          Method: POST
          URL: {{ $vars.N8N_BASE_URL }}/api/v1/workflows/{{ $json.workflowId }}/run
          Headers:
            X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
          ↓
        [Slack: 재시도 알림]
          Channel: #n8n-alerts
          Message: |
            워크플로우 자동 재시도 중 ({{ $json.retryCount + 1 }}/2회)

            워크플로우: {{ $json.workflowName }}
            오류: {{ $json.errorMessage }}
            재시도 시각: {{ $now.format('HH:mm:ss') }}

      shouldEscalate === true →
        [Slack: 에스컬레이션 알림 (담당자 멘션)]
          Channel: #alerts-critical
          Message: |
            자동화 장애 에스컬레이션 @channel

            워크플로우: {{ $json.workflowName }}
            재시도 횟수: {{ $json.retryCount }}회 모두 실패
            마지막 오류: {{ $json.errorMessage }}
            최초 실패: {{ $json.failedAt }}

            즉시 확인이 필요합니다.
            {{ $vars.N8N_BASE_URL }}/workflows/{{ $json.workflowId }}
        ↓
        [Redis: 에스컬레이션 상태 기록]
          Key: {{ $json.retryKey }}
          Value: escalated
          TTL: 86400  // 24시간
```

---

### 예제 2: 워크플로우 실행 시간 이상 감지 (평균 대비 2배 초과 시 알림)

각 워크플로우의 평균 실행 시간을 기준으로, 최근 실행이 평균의 2배를 초과하면 성능 저하 알림을 보냅니다.

```
워크플로우 이름: "실행 시간 이상 감지"
트리거: Schedule (매 시간)

[Schedule Trigger]
  Rule: 0 * * * *
    ↓
[HTTP Request: 최근 24시간 완료된 실행 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/executions
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    status: success
    limit: 500
    startedAfter: {{ $now.minus(24, 'hours').toISO() }}
    ↓
[Code: 워크플로우별 실행 시간 통계 계산]
  const executions = $json.data || [];

  // 워크플로우별로 그룹화
  const groups = {};
  for (const exec of executions) {
    const wfId = exec.workflowId;
    const wfName = exec.workflowData?.name || "알 수 없음";
    const duration = exec.stoppedAt && exec.startedAt
      ? new Date(exec.stoppedAt) - new Date(exec.startedAt)
      : null;

    if (duration === null || duration <= 0) continue;

    if (!groups[wfId]) {
      groups[wfId] = { wfId, wfName, durations: [] };
    }
    groups[wfId].durations.push(duration);
  }

  // 평균, 표준편차 계산
  const stats = Object.values(groups)
    .filter(g => g.durations.length >= 5) // 최소 5건 이상인 것만 분석
    .map(g => {
      const avg = g.durations.reduce((s, d) => s + d, 0) / g.durations.length;
      const latest = g.durations[g.durations.length - 1];
      const ratio = latest / avg;

      return {
        wfId: g.wfId,
        wfName: g.wfName,
        avgMs: Math.round(avg),
        avgSec: (avg / 1000).toFixed(1),
        latestMs: latest,
        latestSec: (latest / 1000).toFixed(1),
        ratio: ratio.toFixed(2),
        isAnomaly: ratio > 2.0,
        sampleCount: g.durations.length
      };
    });

  const anomalies = stats.filter(s => s.isAnomaly);

  if (anomalies.length === 0) {
    return [{ json: { noAnomalies: true, analyzedCount: stats.length } }];
  }

  return anomalies.map(a => ({ json: a }));
    ↓
[IF: noAnomalies === true]
  True →
    [Code: 정상 로그만 남기기]
      console.log(`이상 없음. ${$json.analyzedCount}개 워크플로우 분석 완료.`);
      return [{ json: { status: "normal" } }];
  False →
    [Split In Batches: 1개씩]
      ↓
    [Slack: 실행 시간 이상 알림]
      Channel: #n8n-performance
      Message: |
        워크플로우 실행 시간 이상 감지

        워크플로우: {{ $json.wfName }}
        평균 실행 시간: {{ $json.avgSec }}초
        최근 실행 시간: {{ $json.latestSec }}초
        비율: 평균의 {{ $json.ratio }}배
        분석 샘플 수: {{ $json.sampleCount }}건

        성능 저하 원인을 확인해보세요.
        {{ $vars.N8N_BASE_URL }}/workflows/{{ $json.wfId }}
      ↓
    [Postgres: 이상 이력 기록]
      Operation: Insert
      Table: performance_anomalies
      Data:
        workflow_id: {{ $json.wfId }}
        workflow_name: {{ $json.wfName }}
        avg_ms: {{ $json.avgMs }}
        latest_ms: {{ $json.latestMs }}
        ratio: {{ $json.ratio }}
        detected_at: {{ $now.toISO() }}
```

---

### 예제 3: 일일 자동화 성과 대시보드 자동 생성

매일 오전 9시에 전날의 자동화 실적을 집계하여 Google Slides 대시보드를 자동 생성하고, Slack과 이메일로 공유합니다.

```
워크플로우 이름: "일일 자동화 성과 대시보드"
트리거: Schedule (매일 오전 9시)

[Schedule Trigger]
  Rule: 0 9 * * *
    ↓
[Code: 어제 날짜 범위 계산]
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  const dateStr = yesterday.toISOString().split("T")[0];

  return [{
    json: {
      reportDate: dateStr,
      startTime: `${dateStr}T00:00:00.000Z`,
      endTime: `${dateStr}T23:59:59.999Z`
    }
  }];
    ↓
[HTTP Request: 어제 전체 실행 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/executions
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    limit: 500
    startedAfter: {{ $json.startTime }}
    startedBefore: {{ $json.endTime }}
    ↓
[Code: 성과 지표 종합 계산]
  const executions = $json.data || [];
  const reportDate = $('Code: 어제 날짜 범위 계산').item.json.reportDate;

  const total = executions.length;
  const success = executions.filter(e => e.status === "success").length;
  const errors = executions.filter(e => e.status === "error").length;
  const successRate = total > 0 ? ((success / total) * 100).toFixed(1) : "0.0";

  // 총 실행 시간 합산 (절약된 시간 추정: 자동화가 없었다면 수동으로 걸릴 시간)
  const totalDurationMs = executions.reduce((sum, e) => {
    if (e.startedAt && e.stoppedAt) {
      return sum + (new Date(e.stoppedAt) - new Date(e.startedAt));
    }
    return sum;
  }, 0);
  const totalDurationMin = (totalDurationMs / 60000).toFixed(0);

  // 자동화 절약 시간: 평균 수동 처리 대비 10배 절약 추정
  const estimatedSavedHours = ((totalDurationMs * 10) / 3600000).toFixed(1);

  // 워크플로우별 실행 횟수 Top 5
  const wfCounts = {};
  for (const exec of executions) {
    const name = exec.workflowData?.name || "알 수 없음";
    wfCounts[name] = (wfCounts[name] || 0) + 1;
  }
  const top5 = Object.entries(wfCounts)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5)
    .map(([name, count], i) => ({
      rank: i + 1,
      name,
      count
    }));

  // 시간대별 실행 분포 (0-23시)
  const hourly = Array(24).fill(0);
  for (const exec of executions) {
    const hour = new Date(exec.startedAt).getUTCHours() + 9; // KST 변환
    hourly[hour % 24]++;
  }
  const peakHour = hourly.indexOf(Math.max(...hourly));

  return [{
    json: {
      reportDate,
      metrics: {
        total,
        success,
        errors,
        successRate,
        totalDurationMin,
        estimatedSavedHours
      },
      top5Workflows: top5,
      peakHour: `${peakHour}시`,
      peakCount: hourly[peakHour]
    }
  }];
    ↓
[Code: Slack 메시지 블록 구성]
  const { reportDate, metrics, top5Workflows, peakHour, peakCount } = $json;

  const top5Text = top5Workflows
    .map(w => `${w.rank}위. ${w.name}: ${w.count}회`)
    .join("\n");

  return [{
    json: {
      reportDate,
      slackMessage: `일일 자동화 성과 리포트 (${reportDate})

총 실행: ${metrics.total}건
성공: ${metrics.success}건 (${metrics.successRate}%)
실패: ${metrics.errors}건
추정 절약 시간: ${metrics.estimatedSavedHours}시간

가장 많이 실행된 워크플로우 TOP 5:
${top5Text}

피크 시간대: ${peakHour} (${peakCount}건)`,
      csvData: [
        "항목,값",
        `총 실행,${metrics.total}`,
        `성공률,${metrics.successRate}%`,
        `실패 건수,${metrics.errors}`,
        `절약 추정 시간,${metrics.estimatedSavedHours}시간`
      ].join("\n")
    }
  }];
    ↓
[Slack: 일일 성과 리포트 발송]
  Channel: #daily-automation-report
  Message: {{ $json.slackMessage }}
    ↓
[Google Drive: CSV 리포트 저장]
  File Name: automation-report-{{ $json.reportDate }}.csv
  Content: {{ $json.csvData }}
  Folder: "일일 리포트 / {{ $now.format('YYYY-MM') }}"
    ↓
[Gmail: 경영진 이메일 발송]
  To: management@company.com
  Subject: [n8n] 일일 자동화 성과 리포트 - {{ $json.reportDate }}
  Body: |
    안녕하세요.

    {{ $json.reportDate }} 자동화 성과 리포트입니다.

    {{ $json.slackMessage }}

    상세 데이터는 Google Drive에서 확인하실 수 있습니다.
    {{ $json.driveFileUrl }}
```

---

## 핵심 요약

- Error Trigger는 모든 n8n 워크플로우의 알림 중앙화 핵심
- 5분 헬스체크로 서버 다운 즉시 감지
- 성공률 모니터링으로 조용한 장애(오류 없이 잘못 작동) 탐지
- 일일 리포트로 팀 전체가 자동화 현황 파악
- 우선순위별 알림 채널로 적절한 대응 속도 확보

**다음 챕터**: 책을 마무리하며 n8n 학습 로드맵과 커뮤니티 자원을 안내합니다.
