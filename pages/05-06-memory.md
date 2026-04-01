# 레슨 5.6: 메모리와 컨텍스트 관리

---

## AI 에이전트에게 메모리가 필요한 이유

기본 LLM 노드는 **대화 기억이 없습니다.** 매번 새로운 대화처럼 처리합니다.

```
사용자: "내 이름은 홍길동이야"
AI: "안녕하세요, 홍길동님!"

사용자: "내 이름이 뭐야?"
AI: "죄송합니다, 이름을 알 수 없습니다." ← 기억 없음!
```

메모리를 추가하면:
```
사용자: "내 이름이 뭐야?"
AI: "홍길동님이시죠!" ← 기억 있음!
```

---

## n8n의 메모리 유형

### 1. Window Buffer Memory (세션 메모리)

최근 N개의 메시지를 메모리에 보관합니다.

**설정:**
- Memory Type: Window Buffer Memory
- Window Size: 최근 몇 개 메시지를 기억할지 (기본: 5)
- Session ID: 사용자별 고유 식별자 (예: `{{ $json.userId }}`)

**특징:**
- 설정이 간단
- 워크플로우 재시작 시 메모리 초기화
- 단기 대화에 적합

```
[AI Agent]
  Memory: Window Buffer Memory
  Window Size: 10
  Session ID: {{ $json.chatId }}
```

---

### 2. Postgres Chat Memory (영구 메모리)

PostgreSQL 데이터베이스에 대화 기록을 영구 저장합니다.

**설정:**
- Memory Type: Postgres Chat Memory
- Postgres Connection: DB 연결 정보
- Table Name: n8n_chat_histories
- Session ID: `{{ $json.userId }}`

**특징:**
- 워크플로우 재시작 후에도 기억 유지
- 사용자별 장기 기록 관리 가능
- 대용량 대화 기록 저장 가능

**사용 예시:**
```
사용자가 며칠 후 다시 접속:
"저번에 말했던 그 프로젝트 어떻게 됐나요?"
AI: "지난 화요일에 말씀하셨던 마케팅 캠페인 프로젝트 말씀이시죠?
     당시 예산은 500만원으로 논의하셨는데..."
```

---

### 3. Redis Chat Memory

Redis를 사용한 고성능 메모리.

- 빠른 읽기/쓰기
- 만료 시간 설정 가능 (TTL)
- 대규모 동시 사용자 환경에 적합

---

### 4. Vector Store Memory (의미 기반 검색)

단순 최근 메시지가 아닌, **의미적으로 관련있는** 메시지를 검색합니다.

```
사용자: "지난번에 얘기한 할인 정책이 뭐였지?"
AI: (수백 개의 메시지 중 '할인'과 관련된 대화를 검색)
    "3개월 전에 말씀하셨던 신규 회원 20% 할인 정책을 말씀하시는 건가요?"
```

**사용 예시:** 고객 지원 봇, 장기 프로젝트 관리 어시스턴트

---

## 실전 프로젝트: 대화형 챗봇 구현

### 목표
Telegram에서 동작하는 AI 챗봇으로, 이전 대화를 기억하는 개인 비서 구현

### 워크플로우 구조

```
[Telegram Trigger: 메시지 수신]
    ↓
[Set: sessionId = {{ $json.message.chat.id }}]
    ↓
[AI Agent]
  System: 너는 사용자의 개인 비서야.
          이전 대화 내용을 기억하고 연속성 있게 답변해.
  Memory: Postgres Chat Memory
  Session ID: {{ $json.sessionId }}
  Tools: [Calculator, HTTP Request]
    ↓
[Telegram: 답변 발송]
```

---

## 메모리 관리 고급 기법

### 세션 초기화 명령

사용자가 대화를 초기화하고 싶을 때:

```javascript
// Code 노드에서 세션 초기화 처리
const message = $json.message.text;

if (message === '/reset' || message === '대화 초기화') {
  // Postgres에서 해당 세션 삭제
  return [{
    json: {
      action: 'reset_session',
      sessionId: $json.sessionId,
      reply: '대화 기록이 초기화되었습니다. 새로 시작해요! 😊'
    }
  }];
}

return [{ json: { action: 'chat', ...($json) } }];
```

### 컨텍스트 길이 관리

메모리가 너무 커지면 API 비용이 증가하고 성능이 저하됩니다.

```javascript
// Code 노드에서 컨텍스트 요약
// 메시지가 50개를 초과하면 이전 내용 요약
const messageCount = await getMessageCount($json.sessionId);
if (messageCount > 50) {
  // 요약 API 호출 후 이전 기록 삭제
}
```

### 메모리 스키마 설계

```sql
-- Postgres 메모리 테이블 구조
CREATE TABLE n8n_chat_memories (
  id SERIAL PRIMARY KEY,
  session_id VARCHAR(255),
  role VARCHAR(10), -- 'human' or 'ai'
  content TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB
);

CREATE INDEX idx_session_id ON n8n_chat_memories(session_id);
```

---

## 메모리 유형 선택 가이드

```
단기 테스트/개발
    → Window Buffer Memory

장기 개인 기록 필요
    → Postgres Chat Memory

고성능/대규모 사용자
    → Redis Chat Memory

의미 기반 검색 필요
    → Vector Store Memory
```

---

## 핵심 요약

- AI 에이전트에 메모리가 없으면 매번 새 대화로 시작
- Window Buffer Memory: 최근 N개 메시지, 설정 간단, 임시 저장
- Postgres Chat Memory: 영구 저장, 사용자별 장기 기억
- Session ID로 사용자를 구분하여 개인화된 경험 제공
- 메모리 크기 관리로 API 비용 및 성능 최적화

**다음 레슨**: 5장 핵심 내용을 정리합니다.
