# 레슨 5.3: AI Agent 노드 활용

---

## AI Agent 노드란?

AI Agent 노드는 n8n에서 가장 강력한 AI 노드입니다. 단순히 AI에 질문하고 답을 받는 것이 아니라, AI가 **여러 도구를 사용하며 자율적으로 작업을 수행**합니다.

```
일반 LLM 노드:   질문 → 답변 (1회성)
AI Agent 노드:   목표 → [계획] → [도구 실행] → [결과 확인] → [반복] → 최종 완료
```

---

## AI Agent 노드의 3대 구성 요소

### 1. Brain (브레인) - 판단의 핵심

에이전트의 "두뇌" 역할을 하는 LLM 모델입니다.

```
Brain 연결 방법:
  AI Agent 노드 → 하단 "Add Chat Model" →
  원하는 모델 선택 (OpenAI, Anthropic, Google AI 등)
```

**Brain 설정 옵션:**
- **Model**: 사용할 AI 모델 (gpt-4o, claude-3-5-sonnet 등)
- **Temperature**: 창의성 수준 (0.0 = 결정적, 1.0 = 창의적)
- **Max Tokens**: 응답 최대 길이

### 2. Tools (도구) - 행동의 수단

에이전트가 실제로 실행할 수 있는 기능들입니다.

```
도구 연결 방법:
  AI Agent 노드 → "Add Tool" → 원하는 도구 노드 추가
```

**기본 제공 도구:**

| 도구 | 기능 |
|-----|-----|
| Calculator | 수학 계산 수행 |
| Wikipedia | Wikipedia 검색 |
| SerpAPI | 구글 검색 결과 조회 |
| HTTP Request Tool | 임의 API 호출 |
| Code Tool | 코드 실행 |

**외부 서비스 도구:**
- Gmail Tool → 이메일 읽기/쓰기
- Google Sheets Tool → 스프레드시트 조작
- Slack Tool → 메시지 발송
- Notion Tool → 페이지 생성/조회
- PostgreSQL/MySQL Tool → 데이터베이스 쿼리

### 3. Memory (메모리) - 대화의 연속성

에이전트가 이전 대화를 기억하게 합니다.

```
메모리 연결 방법:
  AI Agent 노드 → "Add Memory" → 메모리 유형 선택
```

**메모리 유형:**
- **Window Buffer Memory**: 최근 N개 메시지 기억
- **Postgres Chat Memory**: DB에 영구 저장
- **Redis Chat Memory**: 빠른 캐시 기반 메모리
- **Vector Store Memory**: 의미 기반 장기 기억

---

## AI Agent 노드 설정

### 기본 설정

```
노드 추가:
  캔버스 → + 버튼 → "AI Agent" 검색 → 추가

주요 파라미터:
  Prompt (입력 텍스트): {{ $json.message }}
  System Message: 에이전트의 역할과 행동 지침
  Max Iterations: 최대 실행 반복 횟수 (기본: 10)
  Return Intermediate Steps: 중간 과정 포함 여부
```

### System Message (시스템 메시지) 작성

시스템 메시지는 에이전트의 정체성과 행동 규칙을 정의합니다.

**기본 구조:**
```
당신은 [역할]입니다.

## 임무
[에이전트가 해야 하는 일]

## 규칙
1. [규칙 1]
2. [규칙 2]

## 사용 가능한 도구
[도구 설명 - n8n이 자동 제공하지만 명시적 언급도 가능]

## 응답 형식
[원하는 출력 형식]
```

**실전 예시 - 고객 지원 에이전트:**
```
당신은 친절한 고객 지원 전문가입니다. 회사명: 테크샵

## 임무
고객의 문의를 접수받아 최적의 해결책을 제공합니다.

## 규칙
1. 항상 한국어로 응답합니다
2. 고객 이름을 알면 이름을 포함해 친근하게 대화합니다
3. 해결하지 못할 경우 "담당자 연결" 옵션을 제시합니다
4. 응답은 간결하고 명확하게 작성합니다

## 처리 순서
1. 문의 유형 파악 (배송/환불/기술지원/기타)
2. 필요한 경우 주문 조회 도구 사용
3. FAQ 데이터베이스 확인
4. 해결책 제시
```

---

## 도구(Tools) 상세 연결 방법

### HTTP Request Tool로 사용자 정의 API 연결

```
1. AI Agent 노드 → Add Tool → HTTP Request Tool
2. 설정:
   - Name: "주문 조회 API"
   - Description: "주문 번호로 주문 상태를 조회합니다"
   - Method: GET
   - URL: https://api.yourshop.com/orders/{{orderId}}
   - Authentication: Header Auth
```

AI는 이 설명을 보고 언제 이 도구를 사용해야 하는지 판단합니다.
**Description이 명확할수록 AI가 올바르게 도구를 선택합니다.**

### 데이터베이스 Tool 연결

```
1. Add Tool → Postgres Tool (또는 MySQL Tool)
2. 설정:
   - Connection: DB Credential 선택
   - Operation: Execute Query
   - Query: AI가 자동 생성 또는 템플릿 제공
```

### Google Sheets Tool 연결

```
1. Add Tool → Google Sheets Tool
2. 설정:
   - Credential: Google OAuth 선택
   - Spreadsheet: 대상 스프레드시트
   - Operation: Read/Append/Update
```

---

## 실전 예시: 고객 문의 처리 에이전트

### 워크플로우 구조

```
[Webhook 트리거]
      ↓
  (문의 수신)
      ↓
[AI Agent 노드]
  ├── Brain: GPT-4o
  ├── Tools:
  │   ├── 주문 조회 API (HTTP Request Tool)
  │   ├── FAQ 검색 (Postgres Tool)
  │   └── 이메일 발송 (Gmail Tool)
  └── Memory: Window Buffer Memory
      ↓
[응답 반환 또는 에이전트 직접 처리]
```

### Step 1: Webhook 트리거 설정

```json
수신 데이터 예시:
{
  "customer_name": "김철수",
  "customer_email": "kim@example.com",
  "message": "주문 12345 배송이 언제 오나요?",
  "session_id": "sess_abc123"
}
```

### Step 2: AI Agent 노드 설정

**System Message:**
```
당신은 테크샵의 고객 지원 에이전트입니다.

고객 정보:
- 이름: {{ $json.customer_name }}
- 이메일: {{ $json.customer_email }}

## 처리 규칙
1. 주문 관련 문의 → 주문 조회 API 도구 사용
2. 일반 질문 → FAQ 검색 도구 사용
3. 해결 후 고객 이메일로 요약 발송
4. 미해결 시 "support@techshop.com으로 에스컬레이션" 안내
```

**Prompt:**
```
{{ $json.message }}
```

### Step 3: 주문 조회 API Tool

```
Name: "주문 상태 조회"
Description: "주문 번호를 입력하면 현재 주문 상태, 배송 정보, 예상 도착일을 조회합니다"
URL: https://api.techshop.com/v1/orders/{{ orderId }}
Method: GET
Headers: Authorization: Bearer {{ $credentials.apiKey }}
```

### Step 4: FAQ 검색 Tool (Postgres)

```sql
-- AI가 자동으로 실행할 쿼리 예시
SELECT question, answer, category
FROM faq
WHERE to_tsvector('korean', question || ' ' || answer)
      @@ to_tsquery('korean', '{{ searchTerm }}')
LIMIT 3;
```

### 실행 결과 예시

```
고객: "주문 12345 배송이 언제 오나요?"

에이전트 내부 과정:
  Thought: 배송 관련 문의입니다. 주문 조회 API를 사용해야겠습니다.
  Action: 주문 조회 API(orderId=12345)
  Observation: {"status": "배송중", "carrier": "CJ대한통운", "tracking": "123456789", "eta": "2024-01-20"}
  Thought: 배송 중이고 내일 도착 예정입니다. 고객에게 답변을 드리겠습니다.

최종 응답:
  김철수님, 안녕하세요!
  주문 #12345는 현재 CJ대한통운을 통해 배송 중이며,
  내일(1월 20일) 도착 예정입니다.
  운송장 번호: 123456789 (CJ대한통운 사이트에서 실시간 확인 가능)
```

---

## 고급 설정

### Max Iterations 조정

```
기본값: 10 (최대 10번 도구 실행)
단순 작업: 3~5로 낮춰 비용 절약
복잡한 리서치: 15~20으로 높여 철저한 조사 허용
```

### Return Intermediate Steps

```
OFF: 최종 답변만 반환 (일반적인 사용)
ON: 각 도구 실행 과정도 반환 (디버깅/감사 목적)
```

---

## 핵심 요약

- AI Agent 노드 = Brain(LLM) + Tools(도구들) + Memory(기억)
- System Message로 에이전트의 역할과 규칙 정의
- 도구 Description이 명확할수록 AI가 올바르게 도구 선택
- HTTP Request Tool로 어떤 API든 도구로 연결 가능
- Max Iterations로 비용과 철저함의 균형 조절

**다음 레슨**: 단순하지만 강력한 Basic LLM Chain 노드 활용법을 배웁니다.
