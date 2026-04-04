# 레슨 4.2: 트리거 종류와 실전 활용

---

## 트리거 고급 활용

3장에서 트리거의 종류를 배웠다면, 이번 레슨에서는 실전에서 어떻게 활용하는지 상세히 다룹니다.

---

## Schedule Trigger 고급 활용

### Cron 표현식 완전 가이드

```
┌───────────── 분 (0-59)
│ ┌───────────── 시 (0-23)
│ │ ┌───────────── 일 (1-31)
│ │ │ ┌───────────── 월 (1-12)
│ │ │ │ ┌───────────── 요일 (0-6, 0=일요일)
│ │ │ │ │
* * * * *
```

#### 실전 Cron 표현식 모음

| 설명 | 표현식 |
|------|--------|
| 매 5분마다 | `*/5 * * * *` |
| 매시간 정각 | `0 * * * *` |
| 평일 오전 9시 | `0 9 * * 1-5` |
| 매일 자정 | `0 0 * * *` |
| 매주 월요일 오전 8시 | `0 8 * * 1` |
| 매월 1일 오전 9시 | `0 9 1 * *` |
| 매 30분마다 (업무시간) | `*/30 9-18 * * 1-5` |
| 분기 시작일 | `0 9 1 1,4,7,10 *` |

### 실전 예시: 일일 리포트 자동화

```
Schedule: 0 9 * * 1-5 (평일 오전 9시)
    ↓
[Google Sheets: 어제 데이터 조회]
    ↓
[Code: 통계 계산]
    ↓
[Slack: 채널에 리포트 발송]
```

---

## Webhook Trigger 심화

### URL 구조 이해

```
https://your-n8n.com/webhook/{path}
```

- path: 자유롭게 설정 가능 (영문, 숫자, 하이픈)
- 각 워크플로우마다 다른 path 사용 권장

### Webhook 보안

#### 방법 1: Header 인증
발신자가 헤더에 비밀 키를 포함하여 전송:
```
X-Secret-Key: my-secret-key-12345
```

n8n에서 확인:
```javascript
// Code 노드에서 검증
const secretKey = $request.headers['x-secret-key'];
if (secretKey !== 'my-secret-key-12345') {
  throw new Error('Unauthorized');
}
```

#### 방법 2: n8n Basic Auth
Webhook 노드 설정:
- Authentication: Basic Auth
- Username/Password 설정

#### 방법 3: JWT 검증
더 강력한 보안이 필요한 경우 JWT 토큰 검증 사용

### Webhook 응답 커스터마이즈

Webhook 노드 설정:
- Response Mode: Last Node (마지막 노드의 결과를 응답으로 반환)
- Response Code: HTTP 상태 코드
- Response Data: 반환할 데이터

```javascript
// Respond to Webhook 노드에서 커스텀 응답
{
  "success": true,
  "message": "주문이 접수되었습니다",
  "orderId": "{{ $json.orderId }}"
}
```

---

## 다중 트리거 패턴

하나의 워크플로우를 여러 방법으로 시작할 수 없지만, 같은 서브 워크플로우를 다양한 트리거에서 호출하는 패턴을 사용할 수 있습니다.

### 패턴: 공통 처리 로직 재사용

```
워크플로우 A (Webhook 트리거) ──────────►[공통 처리 Sub-Workflow]
워크플로우 B (Schedule 트리거) ─────────►
워크플로우 C (Gmail 트리거) ────────────►
```

---

## 트리거별 비용/효율 분석

### Polling 트리거의 실행 횟수 계산

Gmail Trigger (5분 폴링):
- 1시간: 12회
- 1일: 288회
- 1달: 8,640회

이 워크플로우가 Cloud에서 운영된다면 Starter 플랜(2,500회/월)을 1달도 안 돼서 소진합니다.

### Webhook으로 전환 시 절약

Polling → Webhook으로 바꾸면:
- 이메일이 실제로 왔을 때만 실행
- 하루 20통 이메일 기준: 월 600회 (vs 8,640회)
- 93% 절약

---

## 스마트 트리거 패턴

### 패턴 1: 중복 방지 트리거

같은 데이터가 여러 번 처리되는 것을 방지:

```javascript
// Code 노드에서 처리 여부 확인
const processedIds = await $getWorkflowStaticData('global');
const currentId = $json.orderId;

if (processedIds.includes(currentId)) {
  return []; // 이미 처리된 경우 스킵
}

processedIds.push(currentId);
await $setWorkflowStaticData('global', processedIds);

return { json: $json };
```

### 패턴 2: 조건부 트리거

Schedule로 트리거하되, 특정 조건에서만 실제 처리:

```
[Schedule: 5분마다] → [Redis/DB: 처리할 데이터 있는지 확인]
                         ↓ 있음
                      [실제 처리]
                         ↓ 없음
                      [종료]
```

---

## 실전 예제: 완전한 Webhook 주문 처리 시스템

**목표**: 외부 쇼핑몰에서 주문 발생 → n8n Webhook → 전체 처리 자동화

### 1단계: Webhook URL 발급 및 설정

```
n8n에서:
1. 새 워크플로우 생성
2. Webhook Trigger 노드 추가
3. HTTP Method: POST
4. Path: shop-order
5. Authentication: Header Auth
   - Header Name: X-Shop-Secret
   - Header Value: [임의의 강력한 비밀키]
6. 테스트 URL 복사:
   https://your-n8n.com/webhook-test/shop-order
```

### 2단계: 외부 쇼핑몰에서 Webhook 등록

```javascript
// 쇼핑몰 주문 완료 시 호출 예시 (Node.js)
const axios = require('axios');

async function notifyN8n(orderData) {
  await axios.post('https://your-n8n.com/webhook/shop-order', {
    orderId: orderData.id,
    customer: {
      name: orderData.customerName,
      email: orderData.customerEmail,
    },
    items: orderData.items,
    totalAmount: orderData.total,
    timestamp: new Date().toISOString(),
  }, {
    headers: { 'X-Shop-Secret': process.env.N8N_WEBHOOK_SECRET }
  });
}
```

### 3단계: 완성 워크플로우

```
[Webhook: 주문 접수]
    ↓
[Code: 데이터 검증 및 정리]
    ↓
[IF: 금액 10만원 이상?]
  ├── True  → [Slack: #vip-orders 알림]
  │              ↓
  │           [Gmail: VIP 특별 확인 이메일]
  └── False → [Gmail: 일반 주문 확인 이메일]
    ↓
[Google Sheets: 주문 내역 기록]
    ↓
[Respond to Webhook: {"success": true, "orderId": "..."}]
```

---

## Gmail Polling → Pub/Sub Webhook 전환 방법

n8n Cloud 사용 횟수를 93% 절약하는 방법입니다.

### 기존 방식 (Polling)

```
Gmail Trigger (5분마다 폴링) → 이메일 처리
문제: 이메일이 없어도 매 5분마다 실행됨
```

### 개선 방식 (Google Pub/Sub)

```
Gmail → Google Pub/Sub → n8n Webhook → 이메일 처리
효과: 이메일이 실제로 도착했을 때만 실행
```

### Pub/Sub 설정 단계

```
1. Google Cloud Console → Pub/Sub → 주제 생성
   주제 이름: gmail-notifications

2. Gmail API → Watch 설정 (Apps Script 또는 API 직접):
   POST https://gmail.googleapis.com/gmail/v1/users/me/watch
   Body: {
     "topicName": "projects/your-project/topics/gmail-notifications",
     "labelIds": ["INBOX"]
   }

3. n8n Webhook으로 Pub/Sub 메시지 수신 처리
```

---

## 핵심 요약

- Schedule: Cron 표현식으로 정밀한 실행 타이밍 제어
- Webhook: 보안 헤더나 Basic Auth로 인증 추가
- Polling → Webhook 전환으로 실행 횟수 대폭 절약
- 공통 로직은 Sub-Workflow로 분리하여 여러 트리거에서 재사용
- 중복 처리 방지 패턴으로 안정성 향상
- Webhook 응답을 커스터마이즈해 발신 서비스에 결과 전달 가능

**다음 레슨**: IF와 Switch로 복잡한 조건 처리를 마스터합니다.
