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

## 핵심 요약

- Error Trigger는 모든 n8n 워크플로우의 알림 중앙화 핵심
- 5분 헬스체크로 서버 다운 즉시 감지
- 성공률 모니터링으로 조용한 장애(오류 없이 잘못 작동) 탐지
- 일일 리포트로 팀 전체가 자동화 현황 파악
- 우선순위별 알림 채널로 적절한 대응 속도 확보

**다음 챕터**: 책을 마무리하며 n8n 학습 로드맵과 커뮤니티 자원을 안내합니다.
