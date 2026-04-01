# 레슨 6.7: 웹훅 기반 자동화

---

## 웹훅이 핵심인 이유

웹훅은 n8n의 심장입니다. 외부 세계와 n8n을 연결하는 가장 실시간적이고 효율적인 방법입니다.

**웹훅의 활용 범위:**
- 결제 완료 시 즉시 처리 (Stripe, 아임포트 등)
- GitHub 이벤트 발생 시 CI/CD 트리거
- 고객이 폼을 제출했을 때 즉시 처리
- IoT 디바이스에서 데이터 수신
- 내부 시스템 간 이벤트 통신

---

## 웹훅 URL 구조

n8n에서 Webhook 노드를 추가하면 두 가지 URL이 생성됩니다:

```
테스트 URL (개발용):
https://your-n8n.com/webhook-test/{path}
→ 워크플로우가 비활성화되어도 사용 가능
→ n8n에서 편집 중인 워크플로우로 데이터 전달

프로덕션 URL (운영용):
https://your-n8n.com/webhook/{path}
→ 워크플로우가 활성화되어 있어야 동작
→ 실제 서비스에 사용
```

---

## 프로젝트 1: 주문 처리 웹훅 시스템

쇼핑몰의 다양한 이벤트를 웹훅으로 처리:

### 엔드포인트 설계

| 이벤트 | 웹훅 경로 | 처리 내용 |
|--------|-----------|-----------|
| 새 주문 | `/webhook/order-new` | 재고 확인, 이메일, Slack |
| 결제 완료 | `/webhook/payment-complete` | 배송 시작, 영수증 발송 |
| 배송 시작 | `/webhook/shipping-start` | 고객 SMS/이메일 알림 |
| 배송 완료 | `/webhook/shipping-complete` | 리뷰 요청 이메일 |
| 환불 요청 | `/webhook/refund-request` | CS팀 알림, 처리 시작 |

### 새 주문 처리 워크플로우

```
[Webhook: /order-new]
  Method: POST
    ↓
[Code: 요청 검증 + 데이터 정제]
    ↓
[Respond to Webhook: 200 OK]
  (즉시 응답하여 타임아웃 방지)
    ↓
병렬 처리:
  ├── [재고 API: 재고 감소]
  ├── [Gmail: 주문 확인 이메일 발송]
  └── [Slack: 주문 알림]
    ↓
[Google Sheets: 주문 로그 기록]
```

---

## 웹훅 보안 강화

### 방법 1: Signature 검증 (Stripe 방식)

Stripe는 웹훅에 서명을 추가합니다. 이를 검증하여 위조 요청을 차단:

```javascript
// Code 노드에서 Stripe 서명 검증
const crypto = require('crypto');

const payload = $request.rawBody;
const signature = $request.headers['stripe-signature'];
const secret = 'whsec_your_webhook_secret';

// 타임스탬프 추출
const parts = signature.split(',');
const timestamp = parts.find(p => p.startsWith('t=')).split('=')[1];
const v1 = parts.find(p => p.startsWith('v1=')).split('=')[1];

// 서명 계산
const signedPayload = `${timestamp}.${payload}`;
const expectedSig = crypto
  .createHmac('sha256', secret)
  .update(signedPayload)
  .digest('hex');

if (expectedSig !== v1) {
  throw new Error('Invalid webhook signature');
}

return [{ json: JSON.parse(payload) }];
```

### 방법 2: API Key 헤더 검증

```javascript
// Code 노드
const apiKey = $request.headers['x-api-key'];
const validApiKey = 'your-secret-api-key';

if (apiKey !== validApiKey) {
  // Respond to Webhook으로 401 반환
  return [{
    json: {
      statusCode: 401,
      message: 'Unauthorized'
    }
  }];
}

return [{ json: $request.body }];
```

### 방법 3: IP 화이트리스트

```javascript
const allowedIPs = ['54.164.234.192', '52.87.15.116']; // Stripe IPs
const clientIP = $request.headers['x-forwarded-for']?.split(',')[0].trim();

if (!allowedIPs.includes(clientIP)) {
  throw new Error(`Unauthorized IP: ${clientIP}`);
}
```

---

## 웹훅 응답 처리

외부 서비스는 웹훅에 대한 응답을 기다립니다. 너무 오래 걸리면 타임아웃 에러가 발생합니다.

### 즉시 응답 후 비동기 처리 패턴

```
[Webhook]
    ↓
[Respond to Webhook: 200 OK]  ← 즉시 응답 (서비스 타임아웃 방지)
    ↓
[실제 처리 로직]  ← 비동기로 계속 처리
```

---

## 웹훅 재시도 처리

외부 서비스는 웹훅이 실패하면 재시도합니다. 중복 처리를 방지하세요:

```javascript
// Code 노드: 중복 처리 방지
const eventId = $json.id; // Stripe event ID 등
const processedKey = `processed_${eventId}`;

// Redis 또는 Google Sheets에서 처리 여부 확인
const isProcessed = await checkIfProcessed(processedKey);

if (isProcessed) {
  return [{ json: { status: 'already_processed', eventId } }];
}

// 처리 완료 기록
await markAsProcessed(processedKey);

return [{ json: { status: 'processing', eventId, ...$json } }];
```

---

## 웹훅 테스트 도구

### 로컬 개발 시 터널링

로컬 n8n에 외부에서 웹훅을 보내려면 터널링 도구 사용:

```bash
# ngrok 사용
ngrok http 5678
→ https://abc123.ngrok.io/webhook/test

# localtunnel 사용
lt --port 5678
→ https://random-name.loca.lt/webhook/test
```

### 웹훅 테스트 요청 보내기

```bash
# curl로 POST 요청
curl -X POST https://your-n8n.com/webhook-test/order-new \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-secret-key" \
  -d '{"orderId": "ORD-001", "customerName": "홍길동", "amount": 75000}'
```

---

## 핵심 요약

- 웹훅은 n8n과 외부 세계를 연결하는 가장 실시간적인 방법
- 테스트 URL vs 프로덕션 URL 구분하여 사용
- 서명 검증, API Key, IP 화이트리스트로 보안 강화
- 즉시 응답 후 비동기 처리로 타임아웃 방지
- 중복 처리 방지 로직으로 멱등성(Idempotency) 확보

**다음 레슨**: CRM 영업 자동화로 영업 생산성을 극대화합니다.
