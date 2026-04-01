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

## 실전 예제 모음

---

### 예제 1: 쇼핑몰 결제 완료 Webhook → 재고 차감 + 배송 준비 + 영수증 발송

결제 완료 이벤트를 수신하는 즉시 세 가지 후속 작업(재고 차감, 배송 시스템 등록, 영수증 이메일)을 병렬로 처리합니다. 모든 작업이 완료되면 Slack에 주문 현황이 기록됩니다.

워크플로우 구조:

```
[Webhook: /payment-complete]
  Method: POST
  Path: payment-complete
    ↓
[Code: 서명 검증 + 데이터 파싱]
  아임포트 또는 Stripe Webhook Secret 검증
    ↓
[Respond to Webhook: 200 OK]
  즉시 응답 (결제 서버 타임아웃 방지, 5초 제한)
    ↓
[Code: 주문 데이터 정제 + 유효성 검사]
    ↓
병렬 실행:
  ├── 분기 A: [HTTP Request: 재고 관리 API - 재고 차감]
  │             ↓
  │           [IF: 재고 부족?]
  │             ├── True  → [Slack: #ops 재고 부족 긴급 알림]
  │             └── False → [계속]
  │
  ├── 분기 B: [HTTP Request: 배송 시스템 API - 배송 등록]
  │             ↓
  │           [Code: 송장 번호 추출]
  │
  └── 분기 C: [Gmail: 구매 영수증 이메일 발송]
    ↓
[Merge: 3개 분기 결과 합산]
    ↓
[Google Sheets: 주문 처리 로그 기록]
    ↓
[Slack: #orders 채널에 처리 완료 알림]
```

Code 노드 - 서명 검증 (아임포트 방식):

```javascript
// 아임포트(PortOne) Webhook 검증
const crypto = require('crypto');

const body = $request.rawBody;
const receivedHash = $request.headers['x-iamport-signature'];
const secretKey = 'your_iamport_webhook_secret';

const computedHash = crypto
  .createHmac('sha256', secretKey)
  .update(body)
  .digest('hex');

if (computedHash !== receivedHash) {
  throw new Error('Webhook 서명 불일치 - 위조 요청 차단');
}

const payload = JSON.parse(body);

return [{
  json: {
    orderId: payload.merchant_uid,
    paymentId: payload.imp_uid,
    status: payload.status,           // paid, cancelled, failed
    amount: payload.amount,
    customerName: payload.buyer_name,
    customerEmail: payload.buyer_email,
    customerPhone: payload.buyer_tel,
    productName: payload.name,
    items: payload.items ?? [],
    paidAt: new Date(payload.paid_at * 1000).toISOString()
  }
}];
```

재고 차감 API 호출 노드:

```
[HTTP Request: 재고 차감]
  Method: POST
  URL: https://your-shop-api.com/inventory/deduct
  Headers:
    Authorization: Bearer {SHOP_API_KEY}
    Content-Type: application/json
  Body:
    {
      "orderId": "{{ $json.orderId }}",
      "items": {{ $json.items }}
    }
```

배송 등록 API 호출 노드:

```
[HTTP Request: 배송 등록]
  Method: POST
  URL: https://your-delivery-api.com/shipments
  Headers:
    Authorization: Bearer {DELIVERY_API_KEY}
  Body:
    {
      "orderId": "{{ $json.orderId }}",
      "recipientName": "{{ $json.customerName }}",
      "recipientPhone": "{{ $json.customerPhone }}",
      "productName": "{{ $json.productName }}",
      "requestedAt": "{{ $json.paidAt }}"
    }
```

Gmail 영수증 발송 노드:

```
[Gmail: Send Email]
  To: {{ $json.customerEmail }}
  Subject: [주문완료] {{ $json.productName }} 결제가 완료되었습니다
  Body (HTML):
    <h2>구매해 주셔서 감사합니다, {{ $json.customerName }}님!</h2>
    <hr>
    <p>주문번호: {{ $json.orderId }}</p>
    <p>상품명: {{ $json.productName }}</p>
    <p>결제금액: {{ $json.amount.toLocaleString() }}원</p>
    <p>결제일시: {{ $json.paidAt }}</p>
    <hr>
    <p>배송 준비가 시작되었으며, 송장 번호는 발송 후 별도 안내드립니다.</p>
    <p>문의: support@your-shop.com</p>
```

Slack 처리 완료 알림 노드:

```javascript
// Code 노드: 처리 결과 요약
const order = $('Code: 서명 검증').item.json;
const deliveryResult = $('HTTP Request: 배송 등록').item.json;
const trackingNumber = deliveryResult.trackingNumber ?? '발급 중';

const message = [
  `주문 처리 완료`,
  `주문번호: ${order.orderId}`,
  `고객: ${order.customerName} (${order.customerEmail})`,
  `상품: ${order.productName}`,
  `금액: ${order.amount.toLocaleString()}원`,
  `송장번호: ${trackingNumber}`
].join('\n');

return [{ json: { message } }];
```

---

### 예제 2: GitHub Webhook → PR 생성 시 Slack 알림 + 리뷰어 자동 지정

Pull Request가 열릴 때 변경된 파일 영역에 따라 적합한 리뷰어를 자동으로 지정하고, Slack에 PR 요약을 발송합니다. 리뷰 요청 누락을 완전히 없앱니다.

워크플로우 구조:

```
[Webhook: /github/pr]
  Method: POST
  Path: github-pr
    ↓
[Code: GitHub 서명 검증 + PR 이벤트만 필터]
  action = opened 또는 ready_for_review 만 처리
    ↓
[HTTP Request: GitHub API - PR 변경 파일 목록 조회]
    ↓
[Code: 변경 파일 영역 분석 → 담당 리뷰어 매핑]
    ↓
[HTTP Request: GitHub API - 리뷰어 지정]
    ↓
[Slack: #dev-review 채널에 PR 알림]
    ↓
[Respond to Webhook: 200 OK]
```

Code 노드 - 서명 검증 및 이벤트 파싱:

```javascript
const crypto = require('crypto');

const secret = 'your_github_webhook_secret';
const signature = $request.headers['x-hub-signature-256'];
const body = $request.rawBody;

const expected = 'sha256=' + crypto
  .createHmac('sha256', secret)
  .update(body)
  .digest('hex');

if (signature !== expected) {
  throw new Error('GitHub 서명 검증 실패');
}

const payload = JSON.parse(body);
const action = payload.action;

// opened 또는 ready_for_review 이벤트만 처리
if (!['opened', 'ready_for_review'].includes(action)) {
  return [{ json: { skip: true } }];
}

const pr = payload.pull_request;

return [{
  json: {
    skip: false,
    prNumber: pr.number,
    prTitle: pr.title,
    prUrl: pr.html_url,
    prBody: pr.body ?? '',
    author: pr.user.login,
    baseBranch: pr.base.ref,
    headBranch: pr.head.ref,
    changedFiles: pr.changed_files,
    additions: pr.additions,
    deletions: pr.deletions,
    repoOwner: payload.repository.owner.login,
    repoName: payload.repository.name
  }
}];
```

HTTP Request 노드 - 변경 파일 목록 조회:

```
[HTTP Request: PR 변경 파일 조회]
  Method: GET
  URL: https://api.github.com/repos/{{ $json.repoOwner }}/{{ $json.repoName }}/pulls/{{ $json.prNumber }}/files
  Headers:
    Authorization: Bearer {GITHUB_TOKEN}
    Accept: application/vnd.github+json
```

Code 노드 - 리뷰어 자동 매핑:

```javascript
// 파일 경로 → 담당 리뷰어 매핑 규칙
const REVIEWER_RULES = [
  { pattern: /^frontend\/|\.tsx?$|\.css$/,  reviewer: 'frontend-lead' },
  { pattern: /^backend\/|\.py$|\.go$/,      reviewer: 'backend-lead' },
  { pattern: /^infra\/|\.ya?ml$|Dockerfile/, reviewer: 'devops-engineer' },
  { pattern: /^docs\/|\.md$/,               reviewer: 'tech-writer' },
  { pattern: /migration|schema/i,           reviewer: 'dba-engineer' }
];

const changedFiles = $input.all().map(f => f.json.filename);
const reviewersSet = new Set();

for (const filePath of changedFiles) {
  for (const rule of REVIEWER_RULES) {
    if (rule.pattern.test(filePath)) {
      reviewersSet.add(rule.reviewer);
    }
  }
}

// PR 작성자 본인은 리뷰어에서 제외
const author = $('Code: 서명 검증').item.json.author;
reviewersSet.delete(author);

// 리뷰어 없으면 기본 담당자 지정
if (reviewersSet.size === 0) {
  reviewersSet.add('default-reviewer');
}

const reviewers = Array.from(reviewersSet);

return [{
  json: {
    reviewers,
    changedFileSample: changedFiles.slice(0, 5),
    totalFiles: changedFiles.length,
    prInfo: $('Code: 서명 검증').item.json
  }
}];
```

GitHub 리뷰어 지정 노드:

```
[HTTP Request: 리뷰어 지정]
  Method: POST
  URL: https://api.github.com/repos/{{ $json.prInfo.repoOwner }}/{{ $json.prInfo.repoName }}/pulls/{{ $json.prInfo.prNumber }}/requested_reviewers
  Headers:
    Authorization: Bearer {GITHUB_TOKEN}
    Accept: application/vnd.github+json
  Body (JSON):
    {
      "reviewers": {{ $json.reviewers }}
    }
```

Slack 알림 노드:

```javascript
// Code 노드: Slack PR 알림 메시지 생성
const { reviewers, changedFileSample, totalFiles, prInfo } = $input.item.json;

const fileList = changedFileSample.join('\n  ');
const reviewerMentions = reviewers.map(r => `@${r}`).join(', ');

const message = [
  `새 PR이 열렸습니다: #${prInfo.prNumber} ${prInfo.prTitle}`,
  '',
  `작성자: @${prInfo.author}`,
  `브랜치: ${prInfo.headBranch} → ${prInfo.baseBranch}`,
  `변경: +${prInfo.additions} / -${prInfo.deletions} (총 ${totalFiles}개 파일)`,
  '',
  `주요 변경 파일:`,
  `  ${fileList}${totalFiles > 5 ? `\n  ... 외 ${totalFiles - 5}개` : ''}`,
  '',
  `지정된 리뷰어: ${reviewerMentions}`,
  `PR 링크: ${prInfo.prUrl}`
].join('\n');

return [{ json: { message } }];
```

---

### 예제 3: 폼 제출 → 데이터 검증 + DB 저장 + 확인 이메일

Typeform 또는 Google Form 제출 시 입력값을 검증하고, 유효한 경우 Notion DB에 저장 후 확인 이메일을 발송합니다. 유효하지 않은 경우 제출자에게 재입력 안내를 보냅니다.

워크플로우 구조:

```
[Webhook: /form/submit]
  Method: POST
  Path: form-submit
    ↓
[Respond to Webhook: 200 OK]
  즉시 응답 (Typeform 5초 타임아웃 방지)
    ↓
[Code: 폼 데이터 파싱 + 필드 매핑]
  Typeform / Google Form 구조 정규화
    ↓
[Code: 데이터 유효성 검사]
  이메일 형식, 필수 항목, 값 범위 등
    ↓
[IF: 유효성 검사 통과?]
  ├── True  → [Notion: Create Page in 응답 DB]
  │              ↓
  │           [Gmail: 접수 확인 이메일]
  │              ↓
  │           [Slack: #form-submissions 알림]
  │
  └── False → [Gmail: 오류 안내 이메일 (재입력 요청)]
```

Code 노드 - Typeform 데이터 파싱:

```javascript
// Typeform Webhook payload 파싱
const payload = $input.item.json;
const answers = payload.form_response?.answers ?? [];
const definition = payload.form_response?.definition?.fields ?? [];

// field ref → 한국어 필드명 매핑
const fieldMap = {
  'name_field': '이름',
  'email_field': '이메일',
  'company_field': '회사명',
  'phone_field': '전화번호',
  'budget_field': '예산',
  'message_field': '문의내용',
  'service_choice': '관심서비스'
};

const parsed = {};

for (const answer of answers) {
  const fieldRef = answer.field?.ref;
  const key = fieldMap[fieldRef] ?? fieldRef;

  switch (answer.type) {
    case 'text':
    case 'long_text':
      parsed[key] = answer.text;
      break;
    case 'email':
      parsed[key] = answer.email;
      break;
    case 'phone_number':
      parsed[key] = answer.phone_number;
      break;
    case 'number':
      parsed[key] = answer.number;
      break;
    case 'choice':
      parsed[key] = answer.choice?.label;
      break;
    case 'choices':
      parsed[key] = answer.choices?.labels?.join(', ');
      break;
  }
}

parsed['제출시각'] = new Date().toISOString();
parsed['소스'] = 'Typeform';

return [{ json: parsed }];
```

Code 노드 - 유효성 검사:

```javascript
const data = $input.item.json;
const errors = [];

// 필수 항목 검사
const required = ['이름', '이메일', '문의내용'];
for (const field of required) {
  if (!data[field] || String(data[field]).trim() === '') {
    errors.push(`${field} 항목이 비어 있습니다.`);
  }
}

// 이메일 형식 검사
if (data['이메일']) {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(data['이메일'])) {
    errors.push('이메일 형식이 올바르지 않습니다.');
  }
}

// 전화번호 형식 검사 (선택 항목이지만 입력 시 검증)
if (data['전화번호']) {
  const phoneRegex = /^0\d{1,2}-?\d{3,4}-?\d{4}$/;
  if (!phoneRegex.test(data['전화번호'].replace(/\s/g, ''))) {
    errors.push('전화번호 형식이 올바르지 않습니다. (예: 010-1234-5678)');
  }
}

// 문의내용 최소 길이 검사
if (data['문의내용'] && data['문의내용'].length < 10) {
  errors.push('문의내용을 10자 이상 입력해 주세요.');
}

const isValid = errors.length === 0;

return [{
  json: {
    ...data,
    isValid,
    validationErrors: errors
  }
}];
```

Notion 저장 노드 (유효한 경우):

```
[Notion: Create Page]
  Database: {문의 접수 DB ID}
  Properties:
    이름 (title):
      [{ "text": { "content": "{{ $json.이름 }}" } }]
    이메일 (email): "{{ $json.이메일 }}"
    회사명 (rich_text):
      [{ "text": { "content": "{{ $json.회사명 ?? '' }}" } }]
    문의내용 (rich_text):
      [{ "text": { "content": "{{ $json.문의내용 }}" } }]
    관심서비스 (select):
      { "name": "{{ $json.관심서비스 ?? '미선택' }}" }
    소스 (select):
      { "name": "{{ $json.소스 }}" }
    접수일 (date):
      { "start": "{{ $json.제출시각.slice(0, 10) }}" }
    처리상태 (select):
      { "name": "신규" }
```

Gmail 접수 확인 이메일 노드:

```
[Gmail: Send Email]
  To: {{ $json.이메일 }}
  Subject: 문의 접수 확인 - 영업일 24시간 내 답변드립니다
  Body (HTML):
    <h2>{{ $json.이름 }}님, 문의해 주셔서 감사합니다!</h2>
    <p>아래 내용으로 문의가 정상 접수되었습니다.</p>
    <table border="1" cellpadding="8" style="border-collapse:collapse">
      <tr><td>이름</td><td>{{ $json.이름 }}</td></tr>
      <tr><td>회사</td><td>{{ $json.회사명 ?? '-' }}</td></tr>
      <tr><td>문의내용</td><td>{{ $json.문의내용 }}</td></tr>
      <tr><td>접수 시각</td><td>{{ $json.제출시각 }}</td></tr>
    </table>
    <p>담당자가 영업일 기준 24시간 이내에 이메일로 답변 드리겠습니다.</p>
    <p>긴급 문의: 02-1234-5678</p>
```

Gmail 오류 안내 이메일 노드 (유효성 검사 실패 시):

```
[Gmail: Send Email]
  To: {{ $json.이메일 ?? 'unknown' }}
  Subject: 문의 내용 확인 요청
  Body:
    안녕하세요.

    제출하신 문의 내용에 아래와 같은 문제가 있어 정상 접수되지 않았습니다.

    오류 내용:
    {{ $json.validationErrors.join('\n') }}

    아래 링크에서 다시 제출해 주세요:
    https://your-form-link.com

    감사합니다.
```

---

## 핵심 요약

- 웹훅은 n8n과 외부 세계를 연결하는 가장 실시간적인 방법
- 테스트 URL vs 프로덕션 URL 구분하여 사용
- 서명 검증, API Key, IP 화이트리스트로 보안 강화
- 즉시 응답 후 비동기 처리로 타임아웃 방지
- 중복 처리 방지 로직으로 멱등성(Idempotency) 확보

**다음 레슨**: CRM 영업 자동화로 영업 생산성을 극대화합니다.
