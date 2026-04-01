# 레슨 1.4: API란 무엇인가?

---

## API를 이해하면 자동화가 보인다

자동화를 배우다 보면 반드시 만나는 단어가 있습니다. 바로 API입니다.

API를 이해하지 못하면 n8n의 많은 기능이 블랙박스처럼 느껴집니다. 하지만 알고 나면 "아, 이게 이런 거였구나!" 하고 모든 게 명쾌해집니다.

---

## API = 앱들 사이의 중개인

API(Application Programming Interface)는 두 개의 소프트웨어가 서로 대화할 수 있게 해주는 규칙과 도구의 집합입니다.

### 레스토랑 비유

레스토랑을 생각해 보세요:
- 손님 = 우리의 앱 (n8n)
- 주방 = 다른 서비스 (Gmail, Slack, Google Sheets)
- 웨이터 = API

```
손님(n8n) → "시저 샐러드 주세요" → 웨이터(API) → 주방(Gmail)
주방(Gmail) → 요리 완성 → 웨이터(API) → 손님(n8n)에게 전달
```

손님은 주방이 어떻게 요리하는지 알 필요가 없습니다. 웨이터에게 무엇을 원하는지 말하기만 하면 됩니다. 마찬가지로, n8n은 Gmail의 내부 구조를 알 필요 없이 API를 통해 이메일을 보낼 수 있습니다.

---

## API의 실제 작동 방식

API는 요청(Request)과 응답(Response)으로 작동합니다.

```
n8n → [요청] "홍길동의 이메일 주소를 알려줘"
         ↓
    Google Contacts API
         ↓
n8n ← [응답] { "email": "hong@example.com" }
```

### 요청의 4가지 유형 (HTTP Methods)

| 메서드 | 의미 | 예시 |
|--------|------|------|
| GET | 데이터 가져오기 | 이메일 목록 조회 |
| POST | 새 데이터 생성 | 새 이메일 발송 |
| PUT/PATCH | 기존 데이터 수정 | 연락처 정보 업데이트 |
| DELETE | 데이터 삭제 | 이메일 삭제 |

---

## API Key: 출입증

API를 사용하려면 대부분 API 키(API Key)가 필요합니다.

API 키는 마치 건물의 출입 카드와 같습니다:
- 누가 접근하는지 식별
- 어떤 권한을 가졌는지 확인
- 사용량 추적 및 제한

### API 키를 발급받는 법

대부분의 서비스는 개발자 설정 페이지에서 API 키를 발급해 줍니다:
- Google: console.cloud.google.com
- Slack: api.slack.com
- OpenAI: platform.openai.com
- Anthropic: console.anthropic.com

> 중요: API 키는 절대 공개하면 안 됩니다. GitHub에 올리거나 다른 사람에게 공유하지 마세요!

---

## Webhook: 반대 방향의 API

일반 API는 우리가 먼저 요청을 보내는 방식이지만, Webhook은 다릅니다.

Webhook은 외부 서비스가 무언가 발생했을 때 우리에게 먼저 알려주는 방식입니다.

```
일반 API (Polling 방식):
n8n → 매 5분마다 "새 이메일 있어?" → Gmail
      (대부분 "없음"이라는 불필요한 응답)

Webhook (Push 방식):
Gmail → 새 이메일 도착 시 즉시 → n8n에 알림
        (필요할 때만 신호 전달)
```

Webhook은 더 효율적이고 즉각적입니다. n8n은 Webhook 트리거를 기본으로 제공합니다.

---

## REST API vs GraphQL

n8n을 사용하다 보면 이 두 가지 방식을 접하게 됩니다.

### REST API
- 가장 일반적인 방식
- URL로 자원을 지정, HTTP 메서드로 행동을 지정
- 예: `GET /users/123` → ID가 123인 사용자 정보 조회

### GraphQL
- Facebook이 만든 더 유연한 방식
- 필요한 데이터만 정확히 요청 가능
- 주로 복잡한 데이터를 다루는 서비스에서 사용

n8n의 대부분의 내장 노드는 REST API를 사용하며, HTTP Request 노드로 직접 호출도 가능합니다.

---

## n8n에서 API를 사용하는 3가지 방법

### 방법 1: 내장 노드 사용 (가장 쉬움)
Gmail, Slack, Google Sheets 등은 n8n에 이미 내장되어 있어 설정만 하면 됩니다.

```
[Gmail 트리거 노드] → 새 이메일 감지 자동화
```

### 방법 2: HTTP Request 노드 (중간 난이도)
n8n에 전용 노드가 없는 서비스라면 HTTP Request 노드로 직접 API를 호출할 수 있습니다.

```
[HTTP Request 노드]
URL: https://api.example.com/data
Method: GET
Headers: Authorization: Bearer {API_KEY}
```

### 방법 3: Webhook 수신 (트리거로 사용)
외부 서비스에서 데이터를 받아 워크플로우를 시작할 때 사용합니다.

---

## API 응답 데이터 이해하기

API는 대부분 JSON 형식으로 데이터를 반환합니다.

```json
{
  "id": "email_123",
  "from": "customer@example.com",
  "subject": "주문 문의드립니다",
  "body": "안녕하세요. 주문한 상품이...",
  "received_at": "2026-04-01T09:00:00Z",
  "is_read": false
}
```

이 데이터에서 필요한 부분만 추출해서 다음 노드로 전달하는 것이 n8n의 핵심입니다. JSON 데이터 다루기는 3장에서 자세히 배웁니다.

---

## 핵심 요약

- API는 앱들 사이의 중개인으로, 서로 대화할 수 있게 해주는 규칙
- 요청(Request)와 응답(Response)으로 작동하며, GET/POST/PUT/DELETE 4가지 주요 메서드가 있음
- API 키는 서비스 접근을 위한 출입 카드 — 절대 공개 금지
- Webhook은 외부 서비스가 먼저 우리에게 알려주는 역방향 방식
- n8n은 내장 노드, HTTP Request, Webhook 등 3가지 방법으로 API를 활용

**다음 레슨**: 내 업무에서 자동화할 프로세스를 찾는 실전 방법을 배웁니다.
