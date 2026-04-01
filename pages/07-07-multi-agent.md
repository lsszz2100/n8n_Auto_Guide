# 레슨 7.7: 멀티 에이전트 시스템 설계

---

## 멀티 에이전트란?

단일 AI 에이전트로는 복잡하고 대규모 작업을 처리하기 어렵습니다. **멀티 에이전트 시스템**은 여러 AI 에이전트가 각자의 역할을 맡아 협력하는 구조입니다.

**단일 에이전트 한계:**
- 하나의 에이전트가 모든 작업을 처리 → 맥락 한계 도달
- 전문성 부족: 모든 도메인에서 동등한 성능 불가
- 병렬 처리 불가: 순차적으로만 작업 수행

**멀티 에이전트 장점:**
- 역할 분담으로 각 에이전트가 전문 영역에 집중
- 병렬 처리로 작업 속도 향상
- 오류 검토: 다른 에이전트가 결과물 검증

---

## 멀티 에이전트 패턴

### 패턴 1: 오케스트레이터 - 워커

```
[오케스트레이터 에이전트]
  - 전체 작업 계획 수립
  - 워커 에이전트에게 작업 배분
  - 결과 수집 및 통합
        │
   ┌────┴────┐
   ▼         ▼
[워커 A]  [워커 B]
리서치     작성
        │
        ▼
   [워커 C]
   검토/편집
```

### 패턴 2: 파이프라인 체인

```
[에이전트 1: 데이터 수집]
    ↓ 결과 전달
[에이전트 2: 분석]
    ↓ 결과 전달
[에이전트 3: 보고서 작성]
    ↓ 결과 전달
[에이전트 4: 검토 및 승인]
```

### 패턴 3: 전문가 패널

```
주제: "신제품 출시 전략"
    ↓ (동시 분석)
[마케팅 전문가 에이전트] → 시장 분석 결과
[재무 전문가 에이전트]   → 수익성 분석 결과
[기술 전문가 에이전트]   → 기술 타당성 결과
    ↓ (결과 통합)
[최종 전략 수립 에이전트]
```

---

## n8n으로 구현하는 멀티 에이전트

### 예시: AI 리서치 팀 자동화

**목표**: 경쟁사 분석 보고서 자동 생성

```
[Webhook: 분석 요청 수신]
  { "company": "삼성전자", "focus": "AI 전략" }
    ↓
[오케스트레이터 에이전트]
  System: |
    너는 리서치 프로젝트 매니저야.
    다음 작업들을 계획하고 순서를 정해줘.
    각 작업의 담당 에이전트와 필요한 입력을 JSON으로 반환해.

  결과: {
    "tasks": [
      { "agent": "news_researcher", "input": "삼성전자 최근 AI 뉴스 수집" },
      { "agent": "financial_analyzer", "input": "삼성전자 AI 투자 현황 분석" },
      { "agent": "tech_analyzer", "input": "삼성전자 AI 특허 및 기술력 분석" }
    ]
  }
```

---

## STEP 1: 서브 워크플로우로 에이전트 분리

각 에이전트를 별도의 n8n 워크플로우로 구성:

```
[워크플로우: news-researcher]
    ↓
[HTTP Request: 뉴스 API 검색]
  URL: https://newsapi.org/v2/everything?q={{ $json.query }}
    ↓
[Claude: 뉴스 요약 및 분석]
  System: 뉴스 리서처. 핵심 인사이트를 추출해.
  User: 다음 뉴스들을 분석해서 회사의 AI 전략을 파악해줘:
        {{ $json.articles }}
    ↓
[결과 반환]
```

---

## STEP 2: 오케스트레이터 워크플로우

```javascript
// Code 노드: 병렬 에이전트 실행
const tasks = $json.orchestratorResult.tasks;

// 각 에이전트 서브 워크플로우 병렬 호출
const agentResults = await Promise.all(
  tasks.map(async (task) => {
    const response = await $workflow.executeByName(
      task.agent,
      { input: task.input }
    );
    return {
      agent: task.agent,
      result: response
    };
  })
);

return [{ json: { agentResults } }];
```

**n8n 구현 방식:**

```
[오케스트레이터 결과 파싱]
    ↓
[Split In Batches]
    ↓ (각 작업마다)
[Execute Workflow]
  Workflow: {{ $json.agentName }} 워크플로우
  Input: {{ $json.agentInput }}
    ↓
[Merge: 모든 에이전트 결과 수집]
```

---

## STEP 3: 결과 통합 에이전트

```
[Merge: 에이전트 결과 수집]
    ↓
[Code: 결과 구조화]

// 모든 에이전트 결과를 하나의 컨텍스트로 구성
const results = $input.all().map(item => item.json);

const synthesisContext = results.map(r =>
  `## ${r.agent} 분석 결과\n${r.result}`
).join('\n\n');
    ↓
[Claude Opus: 최종 보고서 생성]
  System: |
    너는 전략 컨설턴트야.
    여러 전문가의 분석을 종합해서
    경영진이 읽을 수 있는 전략 보고서를 작성해.

  User: |
    분석 대상: {{ $json.company }}

    각 전문가 분석 결과:
    {{ $json.synthesisContext }}

    다음 형식으로 최종 보고서를 작성해:
    1. 핵심 요약 (Executive Summary)
    2. 주요 발견 사항
    3. 전략적 시사점
    4. 권장 대응 방안
```

---

## 에이전트 간 통신 패턴

### 방법 1: 공유 데이터베이스

```
[에이전트 A] → [Supabase: 결과 저장] ← [에이전트 B 조회]
                      ↓
              [에이전트 C: 통합]
```

```javascript
// 에이전트 A: 결과 저장
await supabase.from('agent_results').insert({
  session_id: sessionId,
  agent_name: 'news_researcher',
  result: analysisResult,
  status: 'completed'
});

// 에이전트 B: 다른 에이전트 결과 참조
const { data } = await supabase
  .from('agent_results')
  .select('*')
  .eq('session_id', sessionId)
  .eq('agent_name', 'news_researcher');
```

### 방법 2: 웹훅 체인

```
에이전트 A 완료 → Webhook 호출 → 에이전트 B 트리거
에이전트 B 완료 → Webhook 호출 → 에이전트 C 트리거
```

---

## 실전 예시: AI 채용 프로세스 자동화

**시스템 구조:**

```
[지원서 수신 (이메일/폼)]
    ↓
[에이전트 1: 이력서 파서]
  - PDF에서 정보 추출
  - 구조화된 JSON 생성
  - 출력: { name, skills, experience, education }
    ↓
[에이전트 2: 적합도 평가]
  - 직무 요건과 이력서 비교
  - 점수 산정 (0-100)
  - 합격/불합격/보류 판단
    ↓
[IF: 점수 80 이상?]
  True → [에이전트 3: 면접 질문 생성]
          - 지원자 맞춤 질문 10개 생성
          - 합격 안내 이메일 초안 작성
  False → [에이전트 4: 불합격 안내 작성]
           - 정중하고 격려하는 내용
           - 피드백 포함 (선택적)
    ↓
[HR 담당자 검토 요청]
    ↓
[이메일 발송]
```

---

## 에이전트 품질 관리

### 검증 에이전트 패턴

```
[에이전트 A: 콘텐츠 생성]
    ↓
[에이전트 B: 검토 및 피드백]
  System: |
    너는 품질 관리 전문가야.
    다음 기준으로 콘텐츠를 평가해:
    1. 정확성 (사실 오류 있나?)
    2. 완성도 (요청 사항 모두 충족?)
    3. 품질 (가독성, 전문성)

    문제가 있으면 구체적인 수정 지시사항을 제공해.
    JSON: { "approved": true/false, "feedback": "..." }
    ↓
[IF: approved === false]
  → [에이전트 A: 수정 재시도] (최대 3회)
```

---

## 비용 최적화 전략

| 에이전트 역할 | 권장 모델 | 이유 |
|--------------|-----------|------|
| 오케스트레이터 | Claude Sonnet | 계획 수립, 추론 필요 |
| 데이터 수집/파싱 | GPT-4o-mini | 단순 추출, 저렴 |
| 전문 분석 | Claude Opus | 고품질 분석 필요 |
| 검토/검증 | Claude Sonnet | 균형 잡힌 성능 |
| 최종 보고서 | Claude Opus | 최고 품질 필요 |

**비용 계산 예시:**
```
경쟁사 분석 보고서 1건:
- 오케스트레이터 (Sonnet): $0.05
- 데이터 수집 x3 (mini): $0.03
- 전문 분석 x3 (Opus): $0.45
- 검토 (Sonnet): $0.05
- 최종 보고서 (Opus): $0.20
─────────────────
총 비용: 약 $0.78 (약 1,100원)
```

---

## 핵심 요약

- 멀티 에이전트 = 복잡한 작업을 전문화된 AI들이 분담 처리
- 오케스트레이터 패턴: 중앙 에이전트가 계획하고 워커에게 배분
- n8n에서는 서브 워크플로우 + Execute Workflow로 구현
- 공유 DB 또는 웹훅 체인으로 에이전트 간 통신
- 검증 에이전트로 품질 자동 관리
- 역할별 모델 선택으로 비용 최적화

**다음 챕터**: 고급 n8n 기능 — 서브 워크플로우, API, 커스텀 노드 개발
