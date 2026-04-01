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

## 실전 예제 모음

---

### 예제 1: 고객 상담 챗봇 - 대화 이력 유지 (Redis 활용)

웹사이트 채팅 위젯을 통해 들어오는 고객 상담을 처리합니다. Redis Chat Memory를 활용하여 동일 세션 내 대화 이력을 유지하고, 2시간 후에는 자동으로 세션이 만료됩니다.

워크플로우 구조:

```
[Webhook - 채팅 메시지 수신]
            ↓
[Code 노드 - 세션 ID 및 메시지 전처리]
            ↓
[IF 노드 - 초기화 명령 여부 확인]
    ↓                   ↓
[초기화 요청]       [일반 대화]
    ↓                   ↓
[Redis - 세션 삭제] [AI Agent - Redis 메모리 포함]
    ↓                   ↓
[Webhook 응답]     [Webhook 응답]
```

Code 노드 - 세션 전처리:

```javascript
// 웹훅으로 들어온 채팅 데이터 처리
const body = $json.body || $json;
const userId = body.user_id || 'anonymous';
const sessionId = body.session_id || `${userId}_${Date.now()}`;
const message = body.message || '';
const isReset = message.trim() === '/reset' || message.trim() === '대화초기화';

return [{
  json: {
    user_id: userId,
    session_id: sessionId,
    message: message,
    is_reset: isReset,
    timestamp: new Date().toISOString()
  }
}];
```

IF 노드 설정:

```
조건: {{ $json.is_reset === true }}
True 경로: 세션 초기화 처리
False 경로: 일반 AI 상담 처리
```

세션 초기화 경로 - Redis 노드:

```
Operation: Delete
Key: chat_session_{{ $json.session_id }}
```

AI Agent 설정 (일반 대화 경로):

```
System Prompt:
  당신은 '테크플로우' 소프트웨어 회사의 고객 상담 AI입니다.
  이름은 '플로'이며, 친절하고 명확하게 답변합니다.

  담당 업무:
  - 제품 사용법 안내
  - 요금제 및 결제 문의
  - 기술 오류 1차 진단
  - 복잡한 문제는 담당자 연결 안내

  대화 규칙:
  - 항상 한국어로 답변
  - 이전 대화 내용을 기억하고 연속성 있게 응답
  - 모르는 내용은 솔직하게 "확인이 필요합니다"라고 답변
  - 담당자 연결 필요 시: "지금 바로 담당자에게 연결해 드리겠습니다"

User Prompt:
  {{ $json.message }}

Memory Type: Redis Chat Memory
Redis Connection: [Redis 자격증명]
Session Key: chat_session_{{ $json.session_id }}
Session TTL: 7200  (2시간 = 7200초)
Window Size: 20    (최근 20개 메시지 유지)
```

Webhook 응답 설정:

```javascript
// 응답 데이터 구성
const agentOutput = $('AI Agent').item.json.output;
const sessionId = $('Code').item.json.session_id;

return [{
  json: {
    session_id: sessionId,
    reply: agentOutput,
    timestamp: new Date().toISOString()
  }
}];
```

Redis 메모리의 실제 저장 구조:

```
Redis Key: chat_session_user123_1706771234567
TTL: 7200초 (2시간)

Value (JSON 배열):
[
  {"role": "human", "content": "요금제가 어떻게 되나요?"},
  {"role": "ai", "content": "테크플로우는 3가지 요금제를 제공합니다..."},
  {"role": "human", "content": "그럼 스타터 플랜으로 변경하고 싶어요"},
  {"role": "ai", "content": "스타터 플랜으로 변경하시려면..."}
]
```

---

### 예제 2: 주간 업무 누적 메모리 (매일 업데이트)

팀원들이 매일 Slack에 업무 완료 내용을 보고하면, AI가 이를 요약하여 주간 단위로 누적 저장합니다. 금요일에는 자동으로 주간 업무 보고서를 생성합니다.

워크플로우 구조:

```
[Slack 트리거 - #daily-report 채널 메시지 수신]
            ↓
[Code 노드 - 메시지 파싱 및 날짜/담당자 추출]
            ↓
[Postgres - 오늘치 업무 기록 추가]
            ↓
[Basic LLM Chain - 오늘 내용 요약]
            ↓
[Postgres - 주간 누적 요약 업데이트]
            ↓
[IF 노드 - 금요일 여부 확인]
    ↓
[금요일] → [Basic LLM Chain - 주간 보고서 생성]
               ↓
          [Slack - 주간 보고서 발송]
```

Slack 트리거 설정:

```
Event: Message Posted
Channel: daily-report
(봇 메시지는 제외하기 위해 Bot Messages 필터 해제)
```

Code 노드 - 메시지 파싱:

```javascript
const message = $json.text || '';
const userId = $json.user || '';
const ts = parseFloat($json.ts || '0');
const messageDate = new Date(ts * 1000);

// 요일 확인 (0=일, 5=금)
const dayOfWeek = messageDate.getDay();
const isFriday = dayOfWeek === 5;

// 주차 계산 (ISO 주차)
const startOfYear = new Date(messageDate.getFullYear(), 0, 1);
const weekNumber = Math.ceil(((messageDate - startOfYear) / 86400000 + startOfYear.getDay() + 1) / 7);
const weekKey = `${messageDate.getFullYear()}-W${String(weekNumber).padStart(2, '0')}`;

return [{
  json: {
    user_id: userId,
    raw_message: message,
    report_date: messageDate.toISOString().split('T')[0],
    week_key: weekKey,
    is_friday: isFriday
  }
}];
```

Postgres - 일일 업무 기록 저장:

```sql
INSERT INTO daily_work_logs (user_id, report_date, week_key, raw_content, created_at)
VALUES (
  '{{ $json.user_id }}',
  '{{ $json.report_date }}',
  '{{ $json.week_key }}',
  '{{ $json.raw_message }}',
  NOW()
);
```

Basic LLM Chain - 일일 요약 프롬프트:

```
다음 업무 보고 내용을 간결하게 요약하세요.

작성자: {{ $json.user_id }}
날짜: {{ $json.report_date }}
원문:
{{ $json.raw_message }}

요약 규칙:
- 완료한 작업 항목을 명사형으로 정리
- 중요한 성과나 완료 지표가 있다면 수치 포함
- 다음 날 예정 업무가 있다면 별도 표시
- 최대 3줄 이내

형식:
완료: [완료 항목들]
성과: [측정 가능한 성과, 없으면 생략]
예정: [내일 예정, 없으면 생략]
```

Postgres - 주간 누적 요약 업데이트:

```sql
INSERT INTO weekly_summaries (user_id, week_key, accumulated_summary, last_updated)
VALUES (
  '{{ $('Code').item.json.user_id }}',
  '{{ $('Code').item.json.week_key }}',
  '{{ $json.output }}',
  NOW()
)
ON CONFLICT (user_id, week_key)
DO UPDATE SET
  accumulated_summary = weekly_summaries.accumulated_summary || E'\n' || EXCLUDED.accumulated_summary,
  last_updated = NOW();
```

금요일 주간 보고서 생성 - Basic LLM Chain 프롬프트:

```
당신은 팀 업무를 정리하는 시니어 PM입니다.
아래 주간 누적 업무 기록을 바탕으로 팀 주간 보고서를 작성하세요.

주차: {{ $('Code').item.json.week_key }}
기간: {{ $json.week_start }} ~ {{ $json.week_end }}

팀원별 업무 기록:
{{ $json.all_summaries }}

보고서 구성:
1. 이번 주 팀 핵심 성과 (3줄 이내)
2. 팀원별 완료 업무 요약 (각 2줄 이내)
3. 다음 주 예정 업무 (언급된 내용 기반)
4. 특이사항 (지연, 이슈, 공유 필요 사항)

간결하고 명확하게 작성하세요.
```

---

### 예제 3: 사용자별 개인화 설정 기억

AI 어시스턴트가 사용자의 선호도, 자주 사용하는 형식, 특이사항을 학습하여 점점 더 개인화된 응답을 제공합니다. 사용자 프로필을 Postgres에 저장하고, 매 대화마다 참조합니다.

워크플로우 구조:

```
[Webhook - 사용자 메시지 수신]
            ↓
[Postgres - 사용자 프로필 조회]
            ↓
[Code 노드 - 프로필 기반 시스템 프롬프트 동적 생성]
            ↓
[AI Agent - 개인화된 응답 생성]
            ↓
[Basic LLM Chain - 대화에서 프로필 업데이트 사항 추출]
            ↓
[Postgres - 사용자 프로필 업데이트]
            ↓
[Webhook 응답]
```

사용자 프로필 테이블 구조:

```sql
CREATE TABLE user_profiles (
  user_id VARCHAR(100) PRIMARY KEY,
  display_name VARCHAR(200),
  preferred_language VARCHAR(10) DEFAULT 'ko',
  response_style VARCHAR(50) DEFAULT 'balanced',
  -- 선호 응답 스타일: brief(짧게), detailed(상세히), balanced(균형)
  expertise_areas JSONB DEFAULT '[]',
  -- 전문 분야: ["마케팅", "데이터 분석", "Python"]
  avoid_topics JSONB DEFAULT '[]',
  -- 다루지 말아야 할 주제
  custom_instructions TEXT,
  -- 사용자가 직접 입력한 맞춤 지시사항
  interaction_count INTEGER DEFAULT 0,
  last_active TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

Postgres - 사용자 프로필 조회:

```sql
SELECT *
FROM user_profiles
WHERE user_id = '{{ $json.user_id }}'
LIMIT 1;
```

Code 노드 - 개인화 시스템 프롬프트 생성:

```javascript
const profile = $('Postgres').item.json;
const message = $('Webhook').first().json.message;
const userId = $('Webhook').first().json.user_id;

// 프로필이 없으면 신규 사용자 기본값 사용
const isNewUser = !profile || !profile.user_id;

let systemPrompt = '당신은 개인 AI 어시스턴트입니다.\n';

if (!isNewUser) {
  // 이름 설정
  if (profile.display_name) {
    systemPrompt += `사용자 이름: ${profile.display_name}\n`;
    systemPrompt += `사용자를 항상 "${profile.display_name}님"이라고 부르세요.\n`;
  }

  // 응답 스타일 설정
  const styleGuide = {
    'brief': '답변은 핵심만 간결하게 3줄 이내로 작성하세요.',
    'detailed': '답변은 충분한 설명과 예시를 포함하여 상세하게 작성하세요.',
    'balanced': '답변은 핵심을 명확히 하되, 필요한 경우 추가 설명을 포함하세요.'
  };
  systemPrompt += styleGuide[profile.response_style] + '\n';

  // 전문 분야 반영
  if (profile.expertise_areas && profile.expertise_areas.length > 0) {
    systemPrompt += `사용자 전문 분야: ${profile.expertise_areas.join(', ')}\n`;
    systemPrompt += '해당 분야에서는 전문 용어를 사용하고 기초 설명은 생략해도 됩니다.\n';
  }

  // 회피 주제
  if (profile.avoid_topics && profile.avoid_topics.length > 0) {
    systemPrompt += `다루지 말아야 할 주제: ${profile.avoid_topics.join(', ')}\n`;
  }

  // 맞춤 지시사항
  if (profile.custom_instructions) {
    systemPrompt += `\n사용자 맞춤 지시사항:\n${profile.custom_instructions}\n`;
  }
} else {
  systemPrompt += '신규 사용자입니다. 친절하게 안내하며 선호도를 파악하세요.\n';
}

return [{
  json: {
    user_id: userId,
    message: message,
    system_prompt: systemPrompt,
    is_new_user: isNewUser,
    current_profile: profile || {}
  }
}];
```

AI Agent 설정:

```
System Prompt: {{ $json.system_prompt }}
User Prompt: {{ $json.message }}
Memory Type: Postgres Chat Memory
Session ID: {{ $json.user_id }}
Window Size: 15
```

프로필 업데이트 추출 - Basic LLM Chain 프롬프트:

```
아래 대화에서 사용자의 선호도나 특성에 대한 새로운 정보를 추출하세요.

사용자 메시지: {{ $('Code').item.json.message }}
AI 응답: {{ $('AI Agent').item.json.output }}

추출 기준:
- 사용자가 이름이나 호칭을 언급했는가
- 응답 스타일 선호도가 드러났는가 (짧게/상세히 요청 등)
- 전문 분야나 직업이 드러났는가
- 다루기 싫어하는 주제가 있는가
- 특별한 지시사항을 직접 말했는가

위 내용이 있으면 아래 JSON 형식으로 출력하세요.
없으면 정확히 {"no_update": true} 만 출력하세요.

{
  "no_update": false,
  "display_name": "홍길동",
  "response_style": "brief",
  "new_expertise": ["Python", "머신러닝"],
  "new_avoid_topics": [],
  "custom_instructions": "항상 코드 예시를 포함해줘"
}
```

Postgres - 프로필 업데이트:

```javascript
// Code 노드에서 프로필 업데이트 처리
const extracted = JSON.parse($('프로필 추출 LLM').item.json.output);

if (extracted.no_update) {
  // 업데이트 없음 - 상호작용 횟수만 증가
  return [{
    json: {
      sql: `UPDATE user_profiles
            SET interaction_count = interaction_count + 1,
                last_active = NOW()
            WHERE user_id = '${$('Code').item.json.user_id}'`
    }
  }];
}

// 변경된 필드만 업데이트하는 SET 절 동적 생성
const updates = ['interaction_count = interaction_count + 1', 'last_active = NOW()', 'updated_at = NOW()'];

if (extracted.display_name) {
  updates.push(`display_name = '${extracted.display_name}'`);
}
if (extracted.response_style) {
  updates.push(`response_style = '${extracted.response_style}'`);
}
if (extracted.new_expertise && extracted.new_expertise.length > 0) {
  const expertiseJson = JSON.stringify(extracted.new_expertise);
  updates.push(`expertise_areas = expertise_areas || '${expertiseJson}'::jsonb`);
}
if (extracted.custom_instructions) {
  updates.push(`custom_instructions = '${extracted.custom_instructions.replace(/'/g, "''")}'`);
}

return [{
  json: {
    sql: `INSERT INTO user_profiles (user_id, interaction_count)
          VALUES ('${$('Code').item.json.user_id}', 1)
          ON CONFLICT (user_id)
          DO UPDATE SET ${updates.join(', ')}`
  }
}];
```

개인화 동작 예시:

```
[1회차 대화]
사용자: "파이썬으로 리스트 정렬하는 법 알려줘"
AI: "sorted() 함수를 사용하세요: sorted([3,1,2]) → [1,2,3]
     역순은 sorted(lst, reverse=True)입니다."

[프로필 업데이트 추출]
→ expertise: ["Python"] 추가

[3회차 대화]
사용자: "딕셔너리 정렬은?"
AI: (프로필에 Python 전문가로 기록됨)
    "dict를 value 기준으로 정렬:
     sorted(d.items(), key=lambda x: x[1])
     또는 operator.itemgetter(1) 사용"
    (기초 설명 생략, 바로 실용 코드 제시)
```

---

## 핵심 요약

- AI 에이전트에 메모리가 없으면 매번 새 대화로 시작
- Window Buffer Memory: 최근 N개 메시지, 설정 간단, 임시 저장
- Postgres Chat Memory: 영구 저장, 사용자별 장기 기억
- Session ID로 사용자를 구분하여 개인화된 경험 제공
- 메모리 크기 관리로 API 비용 및 성능 최적화

다음 레슨: 5장 핵심 내용을 정리합니다.
