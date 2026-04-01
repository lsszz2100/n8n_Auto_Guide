# 레슨 5.5: AI 프롬프팅 전략

---

## 프롬프트가 결과를 결정한다

AI의 출력 품질은 **프롬프트의 품질에 직접 비례**합니다.

같은 AI 모델이라도:
- 나쁜 프롬프트 → 엉뚱하거나 일관성 없는 결과
- 좋은 프롬프트 → 정확하고 일관된 결과

프롬프트 작성은 AI와 대화하는 **기술**입니다.

---

## 좋은 프롬프트의 4대 요소

### 1. 명확성 (Clarity)
AI에게 정확히 무엇을 원하는지 알려주세요.

```
❌ 나쁜 예: "이메일 써줘"
✅ 좋은 예: "아래 주문 정보를 바탕으로 고객에게 보낼 주문 확인 이메일을 작성해줘.
            어조는 친근하지만 전문적이어야 하고, 200자 이내로 작성해."
```

### 2. 맥락 (Context)
AI가 상황을 이해할 수 있도록 배경 정보를 제공하세요.

```
"너는 한국의 전자상거래 회사 CS 팀의 담당자야.
고객은 방금 주문을 완료했고, 확인 이메일을 기다리고 있어."
```

### 3. 제약 (Constraints)
원하지 않는 결과를 방지하기 위한 제한 조건을 명시하세요.

```
"다음을 반드시 지켜줘:
- 200자 이내
- JSON 형식으로만 반환
- 영어 사용 금지
- 할인 정보 언급 금지"
```

### 4. 예시 (Examples)
원하는 출력 형식이나 스타일의 예를 보여주세요.

```
"다음 형식으로 출력해줘:
{
  'subject': '제목',
  'body': '본문',
  'tone': '친근함/공식적/중립'
}"
```

---

## n8n에서의 동적 프롬프트

n8n에서는 표현식을 사용하여 이전 노드의 데이터를 프롬프트에 삽입할 수 있습니다.

### 기본 동적 프롬프트

```
고객 이름: {{ $json.customerName }}
주문 번호: {{ $json.orderId }}
주문 상품: {{ $json.productName }}
금액: {{ $json.amount }}원

위 정보를 바탕으로 주문 확인 이메일을 작성해주세요.
어조는 친근하고 전문적으로, 한국어로 작성하세요.
```

### 조건부 프롬프트

```
고객 등급: {{ $json.tier }}

{% if $json.tier === 'VIP' %}
이 고객은 VIP 등급입니다. 특별히 감사 인사와 추가 혜택 안내를 포함하세요.
{% else %}
일반 고객용 표준 이메일을 작성하세요.
{% endif %}
```

---

## 4가지 핵심 프롬프팅 전략

### 전략 1: 작업 명세 (Task Specification)

AI에게 정확히 무엇을 해야 하는지 단계별로 지시합니다.

```
다음 단계로 이메일을 분석해줘:
1. 이메일의 주요 의도 파악 (문의/불만/칭찬/기타)
2. 긴급도 평가 (높음/중간/낮음)
3. 답변이 필요한 핵심 질문 추출
4. 담당 부서 추천 (CS/영업/기술지원)

출력은 반드시 다음 JSON 형식으로:
{
  "intent": "문의",
  "urgency": "높음",
  "questions": ["질문1", "질문2"],
  "department": "CS"
}
```

### 전략 2: 역할 부여 (Role Assignment)

AI에게 특정 역할을 맡겨 더 일관된 결과를 얻습니다.

```
너는 10년 경력의 한국어 카피라이터야.
브랜드 목소리: 친근하고 유머러스하지만 신뢰감 있는 전문가
타깃 독자: 30-40대 직장인

아래 제품 설명을 이 브랜드 목소리에 맞게 SNS 캡션으로 변환해줘.
```

### 전략 3: 출력 형식 제어 (Output Formatting)

AI 출력을 n8n에서 쉽게 처리할 수 있는 형식으로 받습니다.

```
분석 결과를 다음 JSON 형식으로만 반환해줘.
다른 설명이나 마크다운은 포함하지 마.

{
  "category": "string",
  "confidence": 0.0-1.0,
  "tags": ["tag1", "tag2"],
  "summary": "string (50자 이내)"
}
```

### 전략 4: Few-Shot 예시 제공

몇 가지 예시를 보여줘서 AI가 패턴을 학습하도록 합니다.

```
다음 예시를 참고해서 고객 이메일을 분류해줘:

예시 1:
이메일: "배송이 언제 오나요?"
분류: {"type": "배송문의", "urgency": "중간"}

예시 2:
이메일: "제품이 도착했는데 불량이에요! 당장 환불해주세요!"
분류: {"type": "불량/환불", "urgency": "높음"}

예시 3:
이메일: "상품 정말 마음에 들어요. 감사합니다!"
분류: {"type": "칭찬", "urgency": "낮음"}

이제 이 이메일을 분류해줘:
{{ $json.emailBody }}
```

---

## 시스템 프롬프트 vs 사용자 프롬프트

n8n의 AI 노드에서는 두 가지 프롬프트를 별도로 설정할 수 있습니다.

### 시스템 프롬프트 (System Prompt)
- AI의 역할과 행동 방식을 정의
- 모든 대화에 적용되는 기본 지시
- 일반적으로 고정된 내용

```
너는 '스마트쇼핑' 회사의 CS 봇이야.
항상 한국어로 답변하고, 친근하지만 전문적인 어조를 유지해.
개인정보는 절대 요청하지 마.
모르는 내용은 "담당자에게 연결해 드리겠습니다"라고 답해.
```

### 사용자 프롬프트 (User Prompt)
- 각 요청별 구체적인 지시
- 동적 데이터 포함
- 이전 노드의 데이터를 표현식으로 삽입

```
다음 고객 이메일을 처리해줘:
고객명: {{ $json.customerName }}
이메일 내용: {{ $json.emailBody }}
주문번호: {{ $json.orderId || '없음' }}
```

---

## 온도(Temperature) 설정 가이드

| 작업 유형 | 권장 온도 | 이유 |
|-----------|-----------|------|
| 데이터 추출/분류 | 0.0 ~ 0.3 | 일관성과 정확성 중요 |
| 요약/번역 | 0.3 ~ 0.5 | 정확하되 약간의 유연성 |
| 이메일 작성 | 0.5 ~ 0.7 | 자연스러움과 정확성 균형 |
| 창의적 콘텐츠 | 0.7 ~ 1.0 | 다양하고 창의적인 결과 |
| 코드 생성 | 0.0 ~ 0.2 | 정확성 최우선 |

---

## 프롬프트 테스트 방법

### n8n에서 프롬프트 A/B 테스트

```
[테스트 데이터 준비]
    ↓
[분기: 50% → 프롬프트 A]
     [50% → 프롬프트 B]
    ↓
[결과 비교 → Google Sheets에 기록]
```

### 프롬프트 버전 관리

중요한 프롬프트는 버전 관리를 권장합니다:

```javascript
// Code 노드에서 버전 관리
const promptVersions = {
  'v1': '기본 분류 프롬프트...',
  'v2': '개선된 분류 프롬프트 (2026-03-01 업데이트)...',
  'v3': '최신 Few-shot 추가 버전...'
};

const activeVersion = 'v3';
return { json: { prompt: promptVersions[activeVersion] } };
```

---

## 실전 예제 모음

---

### 예제 1: Few-shot 프롬프트로 일관된 이메일 답장 작성

고객 문의 이메일이 들어오면, 브랜드 톤앤매너에 맞는 답장을 자동으로 초안 작성합니다. Few-shot 예시를 프롬프트에 포함하여 AI가 항상 동일한 스타일로 작성하도록 강제합니다.

워크플로우 구조:

```
[Gmail 트리거 - 고객 문의 수신]
            ↓
[Code 노드 - 이메일 전처리]
            ↓
[Basic LLM Chain - Few-shot 답장 초안 생성]
            ↓
[Gmail - 담당자에게 초안 전달 (검토 후 발송)]
```

Code 노드 - 이메일 전처리:

```javascript
// 이메일 본문에서 불필요한 내용 제거
const rawBody = $json.body || '';
const subject = $json.subject || '';
const from = $json.from || '';

// 서명, 이전 대화 내용 제거 (일반적인 패턴)
const cleanBody = rawBody
  .split('--')[0]           // 서명 이전까지만
  .split('On ')[0]          // 인용된 이전 메일 제거
  .trim()
  .substring(0, 1000);      // 최대 1000자

return [{
  json: {
    from: from,
    subject: subject,
    body: cleanBody
  }
}];
```

Basic LLM Chain - Few-shot 프롬프트:

```
당신은 '클라우드핏' 패션 브랜드의 고객 서비스 담당자입니다.
브랜드 톤: 따뜻하고 친근하지만 프로페셔널, 과도한 존댓말 지양, 이모티콘 없음

아래 예시를 참고하여 고객 이메일에 대한 답장 초안을 작성하세요.

[예시 1]
고객 문의: "지난주에 주문한 코트가 아직 안 왔어요. 언제 도착하나요?"
답장 초안:
안녕하세요, 클라우드핏 고객서비스팀입니다.
불편을 드려 정말 죄송합니다. 주문 번호를 알려주시면 배송 현황을 바로 확인해 드리겠습니다.
일반적으로 주문 후 3~5 영업일 내 도착하며, 현재 물류 지연이 있는 경우 추가 안내를 드리고 있습니다.
빠르게 확인 후 연락드리겠습니다. 감사합니다.

[예시 2]
고객 문의: "사이즈가 맞지 않아서 교환하고 싶은데 어떻게 해야 하나요?"
답장 초안:
안녕하세요, 클라우드핏 고객서비스팀입니다.
교환은 수령일로부터 7일 이내에 신청 가능합니다.
마이페이지 > 주문내역 > 교환/반품 신청을 통해 진행하시면 되고, 왕복 배송비는 고객 부담입니다.
(불량/오배송의 경우 배송비 전액 당사 부담)
진행 중 궁금한 점이 있으시면 언제든지 문의해 주세요.

[예시 3]
고객 문의: "제품을 받았는데 실밥이 튀어나와 있어요. 불량 아닌가요?"
답장 초안:
안녕하세요, 클라우드핏 고객서비스팀입니다.
불량 제품을 받으셔서 정말 죄송합니다.
해당 부위를 촬영한 사진을 이 메일에 회신해 주시면, 확인 즉시 새 제품으로 교환해 드리겠습니다.
배송비는 당사에서 전액 부담합니다. 빠르게 처리해 드리겠습니다.

이제 아래 실제 고객 문의에 대한 답장 초안을 작성하세요.

고객 이메일:
발신자: {{ $json.from }}
제목: {{ $json.subject }}
내용: {{ $json.body }}

위 예시와 동일한 형식과 톤으로 답장 초안을 작성하세요.
초안만 출력하고 다른 설명은 하지 마세요.
```

Chat Model 설정:

```
Model: gpt-4o
Temperature: 0.4
(일관된 톤 유지를 위해 낮은 온도 사용)
```

Gmail 담당자 전달 설정:

```
To: cs-team@cloudfitt.com
Subject: [답장 초안] {{ $('Gmail Trigger').item.json.subject }}
Body:
다음 고객 이메일에 대한 AI 초안이 준비되었습니다.
검토 후 발송해 주세요.

--- 원본 고객 이메일 ---
발신: {{ $('Gmail Trigger').item.json.from }}
내용: {{ $('Code').item.json.body }}

--- AI 생성 답장 초안 ---
{{ $json.output }}

[주의] 발송 전 반드시 내용을 검토하고 필요시 수정하세요.
```

---

### 예제 2: Chain-of-Thought로 재고 부족 시 발주 여부 판단

재고 수량이 기준치 이하로 떨어졌을 때, 단순한 규칙이 아니라 판매 추세, 리드타임, 계절성 등 복합적인 요소를 고려하여 발주 여부와 수량을 판단합니다.

워크플로우 구조:

```
[Schedule 트리거 - 매일 오전 7시]
            ↓
[Google Sheets - 재고 현황 조회]
            ↓
[Code 노드 - 재고 부족 상품 필터링]
            ↓
[HTTP Request - 최근 30일 판매 데이터 조회]
            ↓
[Basic LLM Chain - CoT 발주 판단]
            ↓
[IF 노드 - 발주 필요 여부 분기]
    ↓             ↓
[발주 필요]   [발주 불필요]
    ↓
[Slack - 구매팀에 발주 요청]
```

Code 노드 - 재고 부족 필터링:

```javascript
// 재고가 안전 재고의 120% 이하인 상품만 추출
const items = $input.all();
const lowStockItems = items.filter(item => {
  const current = item.json.current_stock;
  const safetyStock = item.json.safety_stock;
  return current <= safetyStock * 1.2;
});

return lowStockItems.map(item => ({
  json: {
    product_id: item.json.product_id,
    product_name: item.json.product_name,
    current_stock: item.json.current_stock,
    safety_stock: item.json.safety_stock,
    lead_time_days: item.json.lead_time_days,  // 입고까지 걸리는 일수
    unit_cost: item.json.unit_cost
  }
}));
```

Basic LLM Chain - Chain-of-Thought 프롬프트:

```
당신은 재고 관리 전문가입니다.
아래 상품의 발주 여부를 단계적으로 분석하여 결론을 내려주세요.

[상품 정보]
상품명: {{ $json.product_name }}
현재 재고: {{ $json.current_stock }}개
안전 재고 기준: {{ $json.safety_stock }}개
공급사 리드타임: {{ $json.lead_time_days }}일
단가: {{ $json.unit_cost }}원

[최근 판매 데이터]
최근 7일 판매량: {{ $('HTTP Request').item.json.sales_7d }}개
최근 30일 판매량: {{ $('HTTP Request').item.json.sales_30d }}개
전월 동기간 판매량: {{ $('HTTP Request').item.json.sales_same_period_last_month }}개
현재 월: {{ $now.format('M') }}월

단계별로 분석하세요:

1단계 - 일평균 판매량 계산:
최근 7일 기준 일평균 = ?
최근 30일 기준 일평균 = ?
어느 수치가 더 신뢰할 만한지 판단

2단계 - 리드타임 동안 예상 판매량:
리드타임({{ $json.lead_time_days }}일) × 일평균 판매량 = ?
현재 재고로 리드타임 커버 가능한지 판단

3단계 - 계절성 분석:
전월 동기간 대비 판매량 증감률 = ?
현재 {{ $now.format('M') }}월의 계절적 특성 고려

4단계 - 발주 결론:
발주 여부: yes / no
권장 발주 수량: ?개 (30일치 수요 + 안전 재고 기준)
우선순위: 긴급 / 일반 / 여유

마지막에 반드시 아래 JSON 형식으로 최종 결론을 출력하세요:
DECISION:
{
  "order_needed": true,
  "quantity": 200,
  "priority": "긴급",
  "reason": "리드타임 5일 동안 예상 소진량이 현재 재고를 초과"
}
```

IF 노드 설정:

```
조건: {{ $json.output.includes('"order_needed": true') }}
```

Slack 발주 요청 메시지:

```
Channel: #구매팀
Message:
재고 발주 요청이 접수되었습니다.

상품명: {{ $json.product_name }}
현재 재고: {{ $json.current_stock }}개
권장 발주 수량: {{ $json.recommended_quantity }}개
우선순위: {{ $json.priority }}
판단 근거: {{ $json.reason }}

AI 분석 상세 내용 확인 후 발주서를 작성해 주세요.
```

---

### 예제 3: 역할 지정 프롬프트로 여러 언어 동시 번역

한국어 공지문 하나를 입력하면, 각 언어에 특화된 번역가 역할을 AI에게 부여하여 영어, 일본어, 중국어를 동시에 번역합니다. 단순 번역이 아니라 각 언어권 문화에 맞는 자연스러운 표현을 생성합니다.

워크플로우 구조:

```
[Webhook - 번역할 공지문 수신]
            ↓
[Basic LLM Chain - 영어 번역]    ← 동시 실행
[Basic LLM Chain - 일본어 번역]  ← 동시 실행
[Basic LLM Chain - 중국어 번역]  ← 동시 실행
            ↓
[Merge - 번역 결과 합치기]
            ↓
[HTTP Request - 다국어 CMS에 등록]
```

n8n에서 병렬 실행 구성:
Webhook 노드에서 분기하여 세 개의 LLM Chain 노드를 동시에 실행하고, Merge 노드에서 합칩니다.

영어 번역 - Basic LLM Chain 프롬프트:

```
You are a professional Korean-to-English translator specializing in business communications
for a Korean e-commerce company expanding to English-speaking markets (US, UK, Australia).

Your translation principles:
- Use natural, idiomatic English expressions (not literal translations)
- Adapt Korean honorifics and formality to appropriate English business tone
- Convert Korean-specific date formats, currency, and measurement units to English conventions
- If the Korean text uses formal/polite language, use professional business English
- Preserve the original meaning and intent completely

Korean text to translate:
{{ $json.korean_text }}

Output only the English translation. No explanations or original text.
```

일본어 번역 - Basic LLM Chain 프롬프트:

```
あなたは韓国語から日本語への専門翻訳家です。
韓国のECサービスで日本人顧客向けのコミュニケーションを担当しています。

翻訳の原則:
- 日本語として自然な表現を使用（直訳ではなく意訳）
- 韓国語の敬語・丁寧語を日本語の敬語体系に適切に変換
- 「〜です・〜ます」調の丁寧な文体を維持
- 日本の顧客が違和感を感じない表現を選択
- 原文の意味と意図を完全に保持

翻訳対象の韓国語テキスト:
{{ $json.korean_text }}

日本語訳のみを出力してください。説明や原文は含めないでください。
```

중국어 번역 - Basic LLM Chain 프롬프트:

```
您是一位专业的韩中翻译专家，专注于韩国电商企业面向中国大陆客户的商务沟通。

翻译原则：
- 使用符合中国大陆用语习惯的简体中文（非台湾繁体中文）
- 将韩语敬语体系转化为恰当的中文商务表达
- 使用中国消费者熟悉的表达方式和语气
- 避免直译，追求语义通顺自然
- 完整保留原文的含义和意图

待翻译的韩语原文：
{{ $json.korean_text }}

请仅输出中文译文，不要包含任何说明或原文。
```

Merge 노드 설정:

```
Mode: Combine
Combine By: Position

결합 후 데이터 구조:
{
  "original_korean": "...",
  "english": "...",
  "japanese": "...",
  "chinese": "..."
}
```

결합을 위한 Code 노드 (Merge 이후):

```javascript
// 세 번역 결과를 하나의 객체로 합치기
const results = $input.all();

return [{
  json: {
    original_korean: $('Webhook').first().json.korean_text,
    english: results[0].json.output,
    japanese: results[1].json.output,
    chinese: results[2].json.output,
    translated_at: new Date().toISOString()
  }
}];
```

CMS 등록 HTTP Request 설정:

```
Method: POST
URL: https://cms.company.com/api/notices/multilingual
Headers:
  Authorization: Bearer {{ $env.CMS_API_KEY }}
  Content-Type: application/json
Body:
{
  "notice_id": "{{ $('Webhook').item.json.notice_id }}",
  "translations": {
    "ko": "{{ $('Webhook').item.json.korean_text }}",
    "en": "{{ $json.english }}",
    "ja": "{{ $json.japanese }}",
    "zh": "{{ $json.chinese }}"
  },
  "translated_at": "{{ $json.translated_at }}"
}
```

---

## 핵심 요약

- 좋은 프롬프트의 4요소: 명확성, 맥락, 제약, 예시
- 동적 프롬프트: 표현식으로 실시간 데이터 삽입
- 역할 부여로 일관된 어조와 스타일 유지
- JSON 출력 형식 지정으로 n8n에서 쉽게 처리
- 온도 설정: 창의적 작업은 높게, 정확성이 필요하면 낮게

다음 레슨: AI의 메모리와 컨텍스트 관리로 대화형 봇을 만듭니다.
