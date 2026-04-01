# 레슨 7.4: AI 고객 지원 봇 구축

---

## 프로젝트 개요

24시간 자동으로 고객 문의를 처리하는 AI 봇을 구축합니다.

**완성 시스템:**
- Telegram + 웹챗에서 동작하는 대화형 봇
- 제품 FAQ 자동 답변
- 주문 조회, 환불 처리 안내
- 해결 불가 시 인간 상담원으로 에스컬레이션

---

## 시스템 아키텍처

```
고객 채널:
  [Telegram] ──────────►
  [웹사이트 채팅] ───────► [n8n AI Agent] ──► [응답]
  [카카오톡 챗봇] ──────►         │
                              ┌───▼────┐
                              │도구(Tools)│
                              ├────────┤
                              │FAQ DB  │
                              │주문 API │
                              │CRM     │
                              └────────┘
```

---

## AI Agent 노드 설정

```
[Telegram Trigger / Webhook]
    ↓
[AI Agent 노드]
설정:
  Model: claude-sonnet-4-5 (또는 gpt-4o)

  System Prompt: |
    너는 '스마트쇼핑'의 AI 고객 지원 담당자 "스마티"야.

    규칙:
    1. 항상 한국어로 응답
    2. 친근하고 공감적인 어조 유지
    3. 모르는 내용은 솔직히 인정하고 인간 담당자 연결 제안
    4. 개인정보(주민번호, 비밀번호 등)는 절대 요청하지 않음
    5. 응답은 3-5문장으로 간결하게

    자주 묻는 질문:
    - 배송: 결제 후 2-3 영업일, 제주/도서산간 추가 2일
    - 반품: 수령 후 7일 이내, 미개봉 조건
    - 교환: 고객센터 1588-0000 또는 앱 내 신청
    - 환불: 반품 확인 후 3-5 영업일 내 처리

  Memory: Postgres Chat Memory
  Session ID: {{ $json.chatId }}

  Tools:
    - 주문 조회 (HTTP Request)
    - 상품 검색 (HTTP Request)
    - 인간 상담원 연결 (Webhook)
```

---

## 도구(Tools) 연결

### 도구 1: 주문 조회

```javascript
// Tool 정의 (AI Agent 노드의 Tools 설정)
이름: 주문조회
설명: 고객의 주문 번호로 주문 상태를 조회합니다
파라미터: orderId (주문번호)

// 실제 구현 (HTTP Request 노드)
GET https://api.yourshop.com/orders/{{ $json.orderId }}
Headers:
  Authorization: Bearer {API_KEY}
```

### 도구 2: 인간 상담원 연결

```javascript
이름: 상담원연결
설명: AI가 해결하기 어려운 문제일 때 인간 상담원에게 연결합니다
파라미터: reason (연결 이유), conversationSummary (대화 요약)

// 구현:
// 1. Slack의 #cs-queue 채널에 알림
// 2. 고객에게 "잠시만 기다려주세요" 메시지
// 3. 상담원이 직접 Telegram에서 응답
```

---

## 에스컬레이션 로직

AI가 스스로 판단하여 인간에게 넘기는 시점:

```javascript
// System Prompt에 포함
const escalationRules = `
다음 상황에서는 반드시 상담원 연결 도구를 사용해:
1. 환불 금액이 10만원 이상인 경우
2. 고객이 3회 이상 같은 문제를 반복 언급한 경우
3. 법적 분쟁이나 소비자원 언급이 있을 경우
4. 화가 많이 난 고객 (욕설, 강한 불만)
5. "사람이랑 얘기하고 싶다"는 직접 요청
`;
```

---

## 감정 인식 기반 응대

```javascript
// 메시지 수신 후 감정 분석 선행
const sentimentPrompt = `
이 메시지의 감정을 분석해. JSON만 반환.
메시지: "${$json.message}"

{"emotion": "happy|neutral|frustrated|angry", "urgency": "low|medium|high"}
`;

// 감정에 따른 응대 방식 조정
if (sentiment.emotion === 'angry') {
  // 공감 우선, 즉각 에스컬레이션 고려
} else if (sentiment.emotion === 'frustrated') {
  // 이해와 해결 의지 표현
}
```

---

## 프로액티브 메시지 발송

고객이 먼저 연락하기 전에 선제적으로 메시지 발송:

```
[배송 상태 변경 Webhook]
    ↓
[IF: 배송 완료]
    ↓
[Telegram: 프로액티브 메시지]
  "주문하신 상품이 배송 완료되었습니다! 📦
   혹시 상품에 문제가 있으시면 저에게 말씀해 주세요.
   리뷰 작성 시 500 포인트를 드립니다: [리뷰 쓰기]"
```

---

## 봇 성능 측정

```
측정 지표:
- 자동 해결률: AI가 인간 개입 없이 해결한 비율 (목표: 70%)
- 평균 응답 시간: 첫 응답까지 걸린 시간 (목표: 5초 이내)
- 고객 만족도: 대화 후 평가 (목표: 4.0/5.0 이상)
- 에스컬레이션율: 인간 상담원으로 넘어간 비율 (목표: 30% 이하)
```

---

## 실전 예제 모음

---

### 예제 1: FAQ 기반 자동 답변 (유사도 매칭)

미리 작성된 FAQ 목록에서 고객 질문과 가장 유사한 답변을 찾아 자동으로 응대합니다. 정확히 일치하지 않아도 의미적으로 유사한 질문에 답변할 수 있습니다.

워크플로우 구조:

```
[Telegram Trigger / Webhook]
    ↓
[OpenAI: 질문 임베딩 생성]
  모델: text-embedding-3-small
  입력: {{ $json.message.text }}
    ↓
[Supabase: 유사 FAQ 검색]
  SELECT * FROM faq_items
  ORDER BY embedding <=> '{{ $json.embedding }}'::vector
  LIMIT 3;
    ↓
[Code: 유사도 점수 기반 분기 판단]
    ↓
[Switch: 유사도 수준]
  ├── 0.85 이상 → [AI: FAQ 기반 맞춤 답변 생성]
  ├── 0.60~0.84 → [AI: 유사 질문 확인 후 답변]
  └── 0.60 미만 → [AI: "정확한 답변 어려움" + 상담사 연결 안내]
    ↓
[Telegram / Webhook: 답변 발송]
```

Supabase FAQ 테이블 구조:

```sql
create table faq_items (
  id bigserial primary key,
  category text,           -- 배송, 환불, 계정, 상품 등
  question text,           -- 대표 질문
  answer text,             -- 준비된 답변
  embedding vector(1536),  -- 질문의 임베딩 벡터
  hit_count int default 0, -- 조회 횟수 (인기 FAQ 파악용)
  created_at timestamptz default now()
);

-- 유사도 검색 함수
create function search_faq(
  query_embedding vector(1536),
  similarity_threshold float default 0.6,
  max_results int default 3
)
returns table (id bigint, category text, question text, answer text, similarity float)
language sql stable as $$
  select id, category, question, answer,
    1 - (embedding <=> query_embedding) as similarity
  from faq_items
  where 1 - (embedding <=> query_embedding) > similarity_threshold
  order by similarity desc
  limit max_results;
$$;
```

유사도 판단 코드 (Code 노드):

```javascript
const results = $input.all();

if (results.length === 0) {
  return [{ json: { route: "no_match", topSimilarity: 0, faqItems: [] } }];
}

const topResult = results[0].json;
const topSimilarity = topResult.similarity;

let route;
if (topSimilarity >= 0.85) {
  route = "high_confidence";
} else if (topSimilarity >= 0.60) {
  route = "medium_confidence";
} else {
  route = "no_match";
}

return [{
  json: {
    route,
    topSimilarity,
    faqItems: results.map(r => r.json),
    userQuestion: $node["Telegram Trigger"].json.message.text
  }
}];
```

높은 신뢰도 답변 프롬프트 (0.85 이상):

```
[OpenAI: gpt-4o-mini]
System: |
  '하모니쇼핑' 고객 지원 담당자야.
  제공된 FAQ 답변을 참고하여 고객 질문에 맞게 자연스럽게 응답해.
  FAQ 내용에서 벗어난 내용은 추가하지 마.
  2~4문장으로 간결하게 답변해.

User: |
  고객 질문: {{ $json.userQuestion }}

  가장 관련 높은 FAQ:
  Q: {{ $json.faqItems[0].question }}
  A: {{ $json.faqItems[0].answer }}

  이 FAQ를 바탕으로 고객 질문에 맞게 답변해줘.

Temperature: 0.3
Max Tokens: 300
```

---

### 예제 2: 감정이 격양된 고객 자동 감지 + 상담사 즉시 연결

고객의 메시지에서 분노나 극도의 불만을 감지하면 AI 봇 응대를 중단하고 사람 상담사에게 즉시 연결합니다.

워크플로우 구조:

```
[Telegram Trigger / Webhook]
    ↓
[OpenAI: 감정 분석]
  모델: gpt-4o-mini
  출력: emotion, intensity, triggerWords, requiresHuman
    ↓
[IF: requiresHuman === true]
  True →
    [Telegram: "잠시만요, 전문 상담사를 연결해드립니다" 즉시 발송]
    ↓
    [Supabase: 대화 이력 저장 + 에스컬레이션 표시]
    ↓
    [Slack: #cs-urgent 채널에 긴급 알림]
      - 고객 ID, 채팅 링크, 대화 요약 포함
    ↓
    [HTTP Request: 상담사 배정 API 호출]
    ↓
    [Telegram: 상담사 배정 완료 안내 + 예상 대기 시간]
  False →
    [AI Agent: 일반 응대 계속]
```

감정 분석 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  고객 서비스 감정 분석 전문가야.
  메시지에서 감정 상태를 분석하고 JSON만 반환해.
  {
    "emotion": "calm" | "frustrated" | "angry" | "furious",
    "intensity": 1~10 (1=매우 차분, 10=극도로 격양),
    "triggerWords": ["감지된 강한 감정 표현 단어들"],
    "requiresHuman": true | false,
    "summary": "감정 상태 한 줄 요약"
  }

  requiresHuman은 다음 중 하나라도 해당하면 true:
  - emotion이 angry 또는 furious
  - intensity가 7 이상
  - 법적 조치, 환불 거부, 사기 등의 표현 포함
  - "사람이랑 얘기하고 싶다" 직접 요청

User: |
  메시지: {{ $json.message.text }}

Temperature: 0.1
```

Slack 긴급 알림 설정:

```
[Slack: 메시지 발송]
  채널: #cs-urgent
  메시지:
    🚨 감정 격양 고객 감지 - 즉시 대응 필요

    고객 ID: {{ $json.chatId }}
    감정 상태: {{ $json.emotion }} (강도: {{ $json.intensity }}/10)
    주요 표현: {{ $json.triggerWords.join(", ") }}
    감정 요약: {{ $json.summary }}

    대화 이력: https://t.me/c/{{ $json.chatId }}

    담당자를 배정하고 직접 응대해 주세요.
```

상담사 배정 후 고객 안내 메시지 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  고객 서비스 담당자로서 공감 능력이 뛰어난 응답을 생성해.
  고객이 화가 나있어도 절대 방어적으로 답하지 말고,
  진심 어린 사과와 빠른 해결 의지를 표현해.

User: |
  고객 감정 상태: {{ $json.emotion }}, 강도 {{ $json.intensity }}/10
  고객이 언급한 내용: {{ $json.summary }}
  예상 상담사 연결 시간: 2분 이내

  상담사 연결 전 고객에게 보낼 안심 메시지를 2~3문장으로 작성해줘.
  성의없이 보이는 복붙 메시지 느낌 금지. 진심이 담긴 표현 사용.

Temperature: 0.5
Max Tokens: 200
```

---

### 예제 3: 지원 티켓 우선순위 자동 분류 (P1~P4)

접수된 지원 티켓을 AI가 분석하여 P1(긴급)부터 P4(낮음)까지 우선순위를 자동으로 지정하고, 우선순위에 따라 담당자 배정과 알림을 다르게 처리합니다.

워크플로우 구조:

```
[Webhook: 새 티켓 접수 (Zendesk / Freshdesk / 자체 시스템)]
    ↓
[OpenAI: 티켓 우선순위 분석]
  출력: priority(P1~P4), category, affectedUsers, businessImpact, suggestedTeam
    ↓
[Switch: 우선순위]
  ├── P1 (긴급) →
  │     [Slack: #incidents 채널 즉시 알림 (멘션 포함)]
  │     [PagerDuty / Webhook: 온콜 담당자 호출]
  │     [HTTP Request: 티켓 시스템에서 P1로 업데이트]
  │     [Slack: 30분마다 미해결 시 재알림 (Wait + Loop)]
  ├── P2 (높음) →
  │     [Slack: #cs-team 채널 알림]
  │     [HTTP Request: P2로 업데이트 + 시니어 담당자 배정]
  ├── P3 (보통) →
  │     [HTTP Request: P3로 업데이트 + 일반 큐 배정]
  │     [Email: 담당팀에 일일 배치 알림]
  └── P4 (낮음) →
        [HTTP Request: P4로 업데이트 + 자동 답변 발송]
        [Google Sheets: 주간 리뷰 목록에 추가]
```

티켓 우선순위 분석 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  IT 지원 티켓 우선순위 분류 전문가야.
  다음 기준으로 P1~P4를 판단하고 JSON만 반환해.

  P1 (긴급): 서비스 전체 중단, 데이터 손실 위험, 보안 사고, 전체 사용자 영향
  P2 (높음): 주요 기능 장애, 다수 사용자 영향, 매출에 직접 영향
  P3 (보통): 일부 기능 오류, 소수 사용자 영향, 임시 우회 가능
  P4 (낮음): 기능 개선 요청, 단순 사용법 문의, 미관 관련 이슈

  {
    "priority": "P1" | "P2" | "P3" | "P4",
    "category": "outage" | "bug" | "feature_request" | "inquiry" | "security",
    "affectedUsersEstimate": "all" | "many" | "few" | "single",
    "businessImpact": "critical" | "high" | "medium" | "low",
    "suggestedTeam": "infra" | "backend" | "frontend" | "cs" | "security",
    "priorityReason": "우선순위 판단 근거 1~2문장",
    "suggestedFirstResponse": "고객에게 즉시 보낼 접수 확인 메시지"
  }

User: |
  티켓 제목: {{ $json.title }}
  티켓 내용: {{ $json.description }}
  제출자: {{ $json.submitterEmail }}
  제출 시간: {{ $json.createdAt }}
  영향받는 환경: {{ $json.environment || "미입력" }}

Temperature: 0.1
```

P1 긴급 Slack 알림 설정:

```
[Slack: 메시지 발송]
  채널: #incidents
  메시지:
    🔴 P1 긴급 티켓 접수 — 즉각 대응 필요 @channel

    티켓 번호: #{{ $json.ticketId }}
    제목: {{ $json.title }}
    판단 근거: {{ $json.priorityReason }}
    예상 영향 범위: {{ $json.affectedUsersEstimate }}
    담당 팀: {{ $json.suggestedTeam }}

    티켓 바로가기: {{ $json.ticketUrl }}
    접수 시각: {{ $json.createdAt }}

    5분 내 첫 응답 필요. 미응답 시 자동 재알림됩니다.
```

30분 재알림 루프 (P1 전용):

```javascript
// Wait 노드 + Loop 구성
// Wait: 30분
// Loop 조건 체크 코드:
const ticketStatus = $node["Check Ticket Status"].json.status;

if (ticketStatus === "open" || ticketStatus === "pending") {
  // 아직 미해결 → Slack에 재알림
  return [{ json: { shouldReAlert: true, ...($input.item.json) } }];
} else {
  // 해결됨 → 루프 종료
  return [{ json: { shouldReAlert: false } }];
}
```

P4 자동 답변 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  IT 헬프데스크 담당자야. 친절하고 명확하게 답변해.
  낮은 우선순위 티켓이지만 성의있게 응대해.

User: |
  고객 문의: {{ $json.description }}
  문의 유형: {{ $json.category }}

  다음 사항을 포함한 답변을 작성해줘:
  - 티켓 접수 확인 (번호: #{{ $json.ticketId }})
  - 예상 처리 기간: 영업일 3~5일
  - 급한 경우 연락 방법: support@company.com 또는 1588-0000
  - 필요하다면 임시 해결 방법 안내

Temperature: 0.4
Max Tokens: 350
```

---

## 핵심 요약

- AI Agent + 도구(Tools) = 스스로 판단하고 행동하는 봇
- Postgres 메모리로 대화 연속성 유지
- 감정 분석으로 맥락에 맞는 응대
- 명확한 에스컬레이션 규칙으로 중요 케이스 인간 처리
- 자동 해결률, 응답 시간 등 KPI로 지속 개선

**다음 레슨**: RAG 시스템으로 내부 문서를 학습한 AI 봇을 만듭니다.
