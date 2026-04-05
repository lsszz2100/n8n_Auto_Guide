# 레슨 7.1: OpenAI GPT 연동하기

---

## OpenAI API 개요

OpenAI는 GPT 시리즈를 API로 제공합니다. n8n은 OpenAI와 네이티브 통합을 지원합니다.

2026년 기준 주요 모델:

| 모델 | 특징 | 비용 (Input/Output) | 추천 용도 |
|------|------|---------------------|-----------|
| gpt-4o | 빠름, 멀티모달 | $2.5/$10 per 1M tokens | 일반 텍스트 작업 |
| gpt-4o-mini | 매우 저렴 | $0.15/$0.6 per 1M tokens | 분류, 단순 추출 |
| o3 | 최고 추론 능력 | $10/$40 per 1M tokens | 복잡한 분석 |
| o4-mini | 추론 + 저렴 | $1.1/$4.4 per 1M tokens | 코드, 수학 |

---

## API 키 발급

1. `platform.openai.com` 접속
2. API Keys 메뉴 → "Create new secret key"
3. 키 이름 설정 후 생성
4. 생성된 키 즉시 복사 (이후 재확인 불가!)

### n8n 크리덴셜 설정

```
Settings → Credentials → Add Credential
→ OpenAI API 검색 → API Key 입력 → Save
```

한 번 저장하면 모든 워크플로우에서 재사용 가능합니다.

---

## Basic LLM Chain 노드 사용

가장 기본적인 OpenAI 호출:

```
[어떤 트리거]
    ↓
[OpenAI: Message a Model]
설정:
  Model: gpt-4o
  Messages:
    System: 너는 한국어 카피라이터야
    User: {{ $json.productDescription }}을 소셜 미디어 캡션으로 변환해줘
  Temperature: 0.7
  Max Tokens: 500
```

---

## 실전 예제 1: 이메일 자동 분류 시스템

수신 이메일을 AI로 분류하고 담당자에게 자동 배정하는 워크플로우입니다.

```
[Gmail Trigger: 새 이메일]
    ↓
[OpenAI: gpt-4o-mini]
  System: |
    이메일을 다음 카테고리 중 하나로 분류해.
    반드시 JSON만 반환: {"category": "주문|환불|문의|스팸|기타", "priority": "high|medium|low", "summary": "한 줄 요약"}
  User: 제목: {{ $json.subject }}, 내용: {{ $json.snippet }}
  Temperature: 0.0
  Response Format: JSON Object
    ↓
[Switch: category 기반 분기]
  "주문"    → [Slack: #order-team 채널에 알림]
  "환불"    → [Slack: #support-team 채널에 알림]
  "문의"    → [Gmail: 자동 답장 초안 발송]
  "스팸"    → [Gmail: 스팸 레이블 추가 및 삭제]
  "기타"    → [Gmail: "검토필요" 레이블 추가]
```

Slack 알림 메시지 예시:
```
새 {{ $json.category }} 이메일 (우선순위: {{ $json.priority }})
보낸 사람: {{ $('Gmail Trigger').item.json.from }}
요약: {{ $json.summary }}
```

---

## 실전 예제 2: AI 콘텐츠 생성 파이프라인

키워드를 입력하면 블로그 글을 자동 생성하고 Notion에 저장합니다.

```
[Manual Trigger]
    ↓
[Edit Fields]
  keyword: "n8n 자동화"
  tone: "친근한"
  length: "800자"
    ↓
[OpenAI: gpt-4o]
  System: 너는 SEO에 최적화된 한국어 블로그 작가야.
  User: |
    키워드: {{ $json.keyword }}
    톤: {{ $json.tone }}
    길이: {{ $json.length }}
    
    다음 형식으로 작성해:
    {"title": "제목", "content": "본문", "tags": ["태그1", "태그2"]}
  Response Format: JSON Object
    ↓
[Notion: Create Page]
  Database: 블로그 콘텐츠 DB
  Title: {{ $json.title }}
  Content: {{ $json.content }}
  Tags: {{ $json.tags }}
  Status: "초안"
```

---

## 실전 예제 3: 고객 리뷰 감성 분석 및 대응

고객 리뷰를 분석하고 부정 리뷰에 자동으로 대응 초안을 생성합니다.

```
[Webhook: 리뷰 수신]
    ↓
[OpenAI: gpt-4o-mini]
  System: |
    고객 리뷰를 분석해. JSON만 반환:
    {
      "sentiment": "positive|negative|neutral",
      "score": 1~10,
      "issues": ["이슈1", "이슈2"],
      "response_draft": "대응 초안 (negative일 때만)"
    }
  User: 리뷰: {{ $json.review }}
  Temperature: 0.0
    ↓
[IF: sentiment === "negative"]
  True:
    ↓
    [Google Sheets: 부정 리뷰 기록]
    [Slack: #cs-team 긴급 알림]
    [Gmail: 고객에게 response_draft로 초안 발송]
  False:
    ↓
    [Google Sheets: 긍정 리뷰 기록]
```

---

## 이미지 분석 (GPT-4o Vision)

이미지를 첨부하면 분석할 수 있습니다:

```javascript
// HTTP Request 노드로 직접 API 호출
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "이 이미지의 제품을 분석하고 설명해줘"
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "{{ $json.imageUrl }}"
          }
        }
      ]
    }
  ]
}
```

실전 활용: 고객이 보낸 불량품 사진 자동 분석, 인보이스 이미지에서 금액 추출

---

## Function Calling (도구 호출)

AI가 특정 함수를 호출하도록 지시합니다. n8n과 결합하면 AI가 직접 워크플로우 내 작업을 선택합니다:

```javascript
// HTTP Request 노드
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": "{{ $json.userRequest }}"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_order_status",
        "description": "주문 상태를 조회합니다",
        "parameters": {
          "type": "object",
          "properties": {
            "order_id": {
              "type": "string",
              "description": "주문 번호"
            }
          },
          "required": ["order_id"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "cancel_order",
        "description": "주문을 취소합니다",
        "parameters": {
          "type": "object",
          "properties": {
            "order_id": { "type": "string" },
            "reason": { "type": "string" }
          },
          "required": ["order_id", "reason"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

AI의 응답에서 `tool_calls`를 파싱한 후 해당 함수에 맞는 노드로 분기합니다.

---

## 비용 최적화 전략

### 1. 적재적소 모델 선택

```
단순 분류/추출 → gpt-4o-mini (10배 저렴)
일반 텍스트 작업 → gpt-4o
복잡한 추론/분석 → o4-mini 또는 o3
```

월 비용 예시:
```
하루 1,000건 이메일 분류 (gpt-4o-mini):
  1,000건 × 500 토큰 × $0.00015/1K = 하루 $0.075 = 월 $2.25

하루 1,000건 이메일 분류 (gpt-4o):
  1,000건 × 500 토큰 × $0.0025/1K = 하루 $1.25 = 월 $37.50
```

### 2. 프롬프트 최적화로 토큰 절약

```javascript
// 나쁜 예 (불필요하게 긴 프롬프트)
"당신은 매우 뛰어난 한국어 언어 전문가이며 이메일 분류 작업을 도와주시는 분입니다.
아래 이메일을 읽고 어떤 유형의 이메일인지 파악하여..."

// 좋은 예 (간결한 프롬프트)
"이메일 분류: 주문/환불/문의/스팸 중 하나를 JSON으로만 반환"
```

시스템 프롬프트 토큰은 모든 요청에 반복됩니다. 간결하게 유지하면 큰 비용 절감이 됩니다.

### 3. 캐싱

같은 입력에 대해 캐싱하여 중복 API 호출 방지:

```javascript
// Code 노드에서 캐시 키 생성
const crypto = require('crypto');
const cacheKey = crypto
  .createHash('md5')
  .update($json.emailContent)
  .digest('hex');

// Redis나 Google Sheets에서 캐시 확인 후 처리
return [{ json: { cacheKey, emailContent: $json.emailContent } }];
```

### 4. Max Tokens 설정

필요한 응답 길이를 제한합니다:

```
분류 작업:     Max Tokens 50~100
요약 작업:     Max Tokens 200~300
콘텐츠 생성:   Max Tokens 1,000~2,000
```

---

## 사용량 모니터링

일일 비용이 임계값을 초과하면 자동으로 알림을 받는 워크플로우:

```
[Schedule: 매일 오전 9시]
    ↓
[HTTP Request: OpenAI Usage API]
  GET https://api.openai.com/v1/usage
  Params: date={{ $now.toFormat('yyyy-MM-dd') }}
  Headers: Authorization: Bearer {{ $credentials.openai.apiKey }}
    ↓
[Code: 비용 계산]
  const usage = $json;
  const totalTokens = usage.data.reduce((sum, item) => sum + item.n_context_tokens_total, 0);
  const estimatedCost = (totalTokens / 1000000) * 2.5; // gpt-4o 기준
  return [{ json: { totalTokens, estimatedCost } }];
    ↓
[IF: estimatedCost > 50]
  True → [Slack: #billing 채널에 비용 경고 알림]
```

---

## Temperature 설정 가이드

| Temperature | 특성 | 추천 용도 |
|-------------|------|---------|
| 0.0 | 매우 일관적, 예측 가능 | 분류, 데이터 추출, JSON 반환 |
| 0.3~0.5 | 약간의 변형 허용 | 요약, 번역, 답변 생성 |
| 0.7~0.9 | 창의적 | 콘텐츠 생성, 카피라이팅 |
| 1.0+ | 매우 창의적, 예측 어려움 | 브레인스토밍, 아이디어 |

---

## 핵심 요약

- OpenAI API 키 발급 후 n8n 크리덴셜에 저장
- gpt-4o: 범용, gpt-4o-mini: 저렴한 단순 작업, o3: 고성능 추론
- Temperature 0: 일관성, Temperature 0.7+: 창의성
- Function Calling으로 AI가 외부 도구 호출 가능
- 비용 절감: 적절한 모델 선택 + 간결한 프롬프트 + 캐싱 + Max Tokens 제한
- 일일 사용량 모니터링으로 예상치 못한 비용 방지

다음 레슨: Anthropic Claude를 n8n에 연동하는 방법을 배웁니다.
