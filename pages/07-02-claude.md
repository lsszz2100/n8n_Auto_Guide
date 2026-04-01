# 레슨 7.2: Anthropic Claude 연동하기

---

## Claude가 특별한 이유

Anthropic의 Claude는 GPT와 함께 2026년 가장 많이 사용되는 AI 모델입니다.

**Claude의 강점:**
- **긴 컨텍스트**: Claude 4 기준 200K 토큰 (약 책 150권 분량)
- **정교한 추론**: 복잡한 분석과 단계적 사고에 강함
- **안전성**: Anthropic의 Constitutional AI로 안전하고 신뢰할 수 있는 응답
- **코드 작성**: 코딩 작업에서 탁월한 성능
- **한국어 지원**: 한국어 이해 및 생성 품질 우수

---

## 2026년 Claude 모델 라인업

| 모델 | 컨텍스트 | 비용 (Input/Output) | 특징 |
|------|----------|---------------------|------|
| claude-opus-4 | 200K | $15/$75 per 1M | 최고 성능 |
| claude-sonnet-4 | 200K | $3/$15 per 1M | 균형 잡힌 성능/비용 |
| claude-haiku-4 | 200K | $0.8/$4 per 1M | 빠르고 저렴 |

---

## API 키 발급

1. `console.anthropic.com` 접속
2. API Keys 메뉴 → "Create Key"
3. 키 이름 설정 후 생성
4. 생성된 키 즉시 복사 (이후 재확인 불가!)

### n8n 크리덴셜 설정

Credentials → Anthropic → API Key 입력

---

## n8n에서 Claude 노드 사용

n8n 1.x 이상에서 Anthropic 노드가 네이티브 지원됩니다:

```
[트리거]
    ↓
[Anthropic: Message Claude]
설정:
  Model: claude-sonnet-4-5
  System: 너는 한국어 법률 문서 분석 전문가야. 명확하고 간결하게 답변해.
  Messages:
    Human: {{ $json.contractText }}를 분석하고 주요 리스크를 3가지로 요약해줘
  Max Tokens: 1000
  Temperature: 0.3
```

---

## HTTP Request로 Claude API 직접 호출

더 세밀한 제어가 필요할 때:

```
[HTTP Request]
설정:
  Method: POST
  URL: https://api.anthropic.com/v1/messages
  Headers:
    x-api-key: {{ $credentials.anthropicApiKey }}
    anthropic-version: 2023-06-01
    content-type: application/json
  Body (JSON):
    {
      "model": "claude-sonnet-4-5",
      "max_tokens": 1024,
      "system": "{{ $json.systemPrompt }}",
      "messages": [
        {
          "role": "user",
          "content": "{{ $json.userMessage }}"
        }
      ]
    }
```

---

## 멀티턴 대화 구성

Claude와 대화 형식으로 소통할 때:

```javascript
// Code 노드에서 대화 기록 구성
const conversationHistory = $json.history || [];

// 새 사용자 메시지 추가
conversationHistory.push({
  role: "user",
  content: $json.newMessage
});

return [{
  json: {
    messages: conversationHistory,
    systemPrompt: "너는 n8n 자동화 전문가야."
  }
}];

// HTTP Request로 Claude 호출 후
// 응답을 history에 추가하여 다음 대화에 사용
```

---

## Claude의 고급 기능

### Extended Thinking (확장 사고)

복잡한 문제에서 Claude가 단계적으로 생각하도록:

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [
    {
      "role": "user",
      "content": "이 비즈니스 전략의 장단점을 심층 분석해줘..."
    }
  ]
}
```

### 비전 (이미지 분석)

```json
{
  "model": "claude-sonnet-4-5",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "{{ $binary.data }}"
          }
        },
        {
          "type": "text",
          "text": "이 이미지의 텍스트를 모두 추출해줘"
        }
      ]
    }
  ]
}
```

---

## GPT vs Claude 선택 가이드

| 작업 유형 | 추천 모델 | 이유 |
|-----------|-----------|------|
| 긴 문서 분석 | Claude Sonnet | 200K 컨텍스트 |
| 법률/계약서 검토 | Claude Opus | 정교한 추론 |
| 빠른 분류/추출 | GPT-4o-mini / Claude Haiku | 저렴하고 빠름 |
| 코드 작성 | Claude Sonnet | 코딩 성능 우수 |
| 창의적 콘텐츠 | GPT-4o / Claude Sonnet | 비슷한 수준 |
| 비용 절감 우선 | Claude Haiku | 가장 저렴 |

---

## 실전 예시: 긴 PDF 계약서 분석

Claude의 긴 컨텍스트를 활용한 계약서 자동 분석:

```
[Google Drive: 계약서 PDF 다운로드]
    ↓
[Code: PDF 텍스트 추출]
    ↓
[Claude Opus: 계약서 분석]
  System: 너는 계약서 전문 변호사야. 의뢰인 관점에서 분석해.
  User: 다음 계약서에서:
        1. 핵심 의무사항 3가지
        2. 리스크 요소 3가지
        3. 협상 포인트 2가지
        를 한국어로 요약해줘.

        계약서 전문:
        {{ $json.contractText }}
    ↓
[Notion: 분석 결과 저장]
[Slack: 검토 요청 알림]
```

---

## 핵심 요약

- Claude는 GPT의 강력한 대안으로 특히 긴 문서, 추론, 코딩에 강점
- 2026년 기준: Opus(최고), Sonnet(균형), Haiku(저렴)
- Anthropic 노드 또는 HTTP Request로 직접 API 호출 가능
- Extended Thinking으로 복잡한 문제 단계적 해결
- 긴 컨텍스트(200K)로 방대한 문서도 한 번에 처리

**다음 레슨**: AI 이메일 자동 분류 및 응답 시스템을 완성합니다.
