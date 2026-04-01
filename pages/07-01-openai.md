# 레슨 7.1: OpenAI GPT 연동하기

---

## OpenAI API 개요

OpenAI는 GPT 시리즈를 API로 제공합니다. n8n은 OpenAI와 네이티브 통합을 지원합니다.

**2026년 기준 주요 모델:**

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

Credentials → OpenAI API → API Key 입력

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

## 텍스트 분류 (저렴한 gpt-4o-mini 활용)

```
[Gmail Trigger]
    ↓
[OpenAI: gpt-4o-mini]
설정:
  System: 이메일을 다음 중 하나로 분류해.
          반드시 JSON만 반환: {"category": "주문/환불/문의/스팸/기타"}
  User: 제목: {{ $json.subject }}, 내용: {{ $json.snippet }}
  Temperature: 0.0
  Response Format: JSON
    ↓
[Code: JSON 파싱]
  const result = JSON.parse($json.message.content);
  return [{ json: result }];
    ↓
[Switch: category 기반 분기]
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

---

## Function Calling (도구 호출)

AI가 특정 함수를 호출하도록 지시:

```javascript
// HTTP Request 노드
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": "서울의 현재 날씨를 알려줘"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "특정 도시의 날씨를 조회",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "도시 이름"
            }
          },
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

---

## 비용 최적화 전략

### 1. 적재적소 모델 선택

```
단순 분류/추출 → gpt-4o-mini (10배 저렴)
일반 텍스트 작업 → gpt-4o
복잡한 추론 → o4-mini 또는 o3
```

### 2. 프롬프트 최적화로 토큰 절약

```javascript
// 나쁜 예 (불필요하게 긴 프롬프트)
"당신은 매우 뛰어난 한국어 언어 전문가이며 이메일 분류 작업을 도와주시는 분입니다.
아래 이메일을 읽고 어떤 유형의 이메일인지 파악하여..."

// 좋은 예 (간결한 프롬프트)
"이메일 분류: 주문/환불/문의/스팸 중 하나를 JSON으로만 반환"
```

### 3. 캐싱

같은 입력에 대해 캐싱하여 중복 API 호출 방지:

```javascript
// Code 노드에서 캐시 키 생성
const cacheKey = require('crypto')
  .createHash('md5')
  .update($json.emailContent)
  .digest('hex');

// Redis에서 캐시 확인
const cached = await redis.get(cacheKey);
if (cached) return [{ json: JSON.parse(cached) }];
```

---

## 사용량 모니터링

```
[Schedule: 매일 오전 9시]
    ↓
[HTTP Request: OpenAI Usage API]
  GET https://api.openai.com/v1/usage?date={{ $now.toFormat('yyyy-MM-dd') }}
    ↓
[IF: 일일 비용 > $50]
  True → [Slack: 비용 경고 알림]
```

---

## 핵심 요약

- OpenAI API 키 발급 후 n8n 크리덴셜에 저장
- gpt-4o: 범용, gpt-4o-mini: 저렴한 단순 작업, o3: 고성능 추론
- Temperature 0: 일관성, Temperature 1: 창의성
- Function Calling으로 AI가 외부 도구 호출 가능
- 비용 절감: 적절한 모델 선택 + 간결한 프롬프트 + 캐싱

**다음 레슨**: Anthropic Claude를 n8n에 연동하는 방법을 배웁니다.
