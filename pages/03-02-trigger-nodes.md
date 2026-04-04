# 레슨 3.2: 트리거 노드 종류와 활용

---

## 트리거가 중요한 이유

워크플로우는 트리거 없이 시작할 수 없습니다. 어떤 트리거를 사용하느냐가 워크플로우의 용도와 효율성을 결정합니다.

잘못된 트리거 선택의 예:
- Polling 방식 사용 → 불필요한 실행 횟수 소비
- 너무 자주 실행되는 스케줄 → 서버 과부하
- Webhook 대신 Polling → 지연 시간 발생

---

## 트리거 노드 총정리

### 1. Manual Trigger
언제 사용: 테스트, 수동 실행이 필요한 경우

```
사용 예시: 월말 보고서를 수동으로 생성할 때
```

- 설정: 없음 (클릭 한 번으로 실행)
- 장점: 가장 단순, 테스트에 최적
- 단점: 자동 실행 불가

---

### 2. Schedule Trigger (Cron)
언제 사용: 정기적으로 자동 실행할 때

```
사용 예시:
- 매일 오전 9시 일일 보고서 발송
- 매주 월요일 주간 요약 생성
- 매시간 데이터 동기화
```

#### 설정 방법

간단 모드: 분/시간/일/주/월 단위로 선택

| 설정 | 실행 빈도 |
|------|-----------|
| Every Minute | 매 1분마다 |
| Every Hour | 매 시간 정각 |
| Every Day | 매일 지정 시간 |
| Every Week | 매주 지정 요일 |
| Every Month | 매월 지정 날짜 |

Cron 표현식 모드: 더 세밀한 제어
```
┌──── 분 (0-59)
│ ┌──── 시 (0-23)
│ │ ┌──── 일 (1-31)
│ │ │ ┌──── 월 (1-12)
│ │ │ │ ┌──── 요일 (0-6, 0=일요일)
│ │ │ │ │
* * * * *

자주 쓰는 Cron 표현식:
0 9 * * 1-5     → 평일 오전 9시
0 */2 * * *     → 2시간마다
0 9,18 * * *    → 매일 9시, 18시
30 8 1 * *      → 매월 1일 오전 8시 30분
```

---

### 3. Webhook Trigger
언제 사용: 외부 서비스에서 n8n을 직접 호출할 때

```
사용 예시:
- Stripe 결제 완료 시 처리
- GitHub PR 생성 시 알림
- 외부 폼 제출 시 처리
```

#### 설정 항목

| 항목 | 설명 |
|------|------|
| HTTP Method | GET/POST/PUT/DELETE 중 선택 |
| Path | URL 경로 (예: `/order-complete`) |
| Authentication | 보안을 위한 인증 방식 |
| Response Mode | 즉시 응답 vs 워크플로우 완료 후 응답 |

#### 생성되는 URL

```
테스트 URL:   https://your-n8n.com/webhook-test/order-complete
프로덕션 URL: https://your-n8n.com/webhook/order-complete
```

> 중요: 테스트 URL은 워크플로우가 활성화되지 않아도 사용 가능합니다. 프로덕션 URL은 워크플로우를 활성화해야 동작합니다.

---

### 4. Email Trigger (IMAP)
언제 사용: 특정 이메일 수신 시 자동화

```
사용 예시:
- 고객 문의 이메일 수신 시 CRM 등록
- 구직 신청서 이메일 수신 시 처리
- 인보이스 이메일 수신 시 회계 처리
```

#### 설정 항목
- IMAP 서버 주소 및 포트
- 이메일 계정 정보
- 모니터링할 폴더 (INBOX, SENT 등)
- 읽음 처리 여부

> Gmail 사용자: Gmail Trigger 노드를 별도로 사용하는 것이 더 편리합니다.

---

### 5. 앱별 트리거 노드

주요 앱들은 자체 트리거 노드를 제공합니다:

| 노드 | 트리거 이벤트 |
|------|-------------|
| Gmail Trigger | 새 이메일 수신, 라벨 변경 |
| Slack Trigger | 새 메시지, 멘션, 채널 이벤트 |
| Google Sheets Trigger | 스프레드시트 변경 |
| Typeform Trigger | 폼 제출 |
| GitHub Trigger | PR, 이슈, 커밋 이벤트 |
| Stripe Trigger | 결제, 환불, 구독 이벤트 |
| Telegram Trigger | 새 메시지 수신 |
| Twitter/X Trigger | 멘션, 트윗 이벤트 |
| Notion Trigger | 데이터베이스 변경 |
| Airtable Trigger | 레코드 생성/수정 |

---

## Polling vs Webhook: 언제 무엇을 써야 하나?

트리거 방식에는 두 가지 패턴이 있습니다:

### Polling (폴링)
n8n이 주기적으로 서비스를 확인하는 방식

```
n8n → (5분마다) "새 이메일 있어?" → Gmail
Gmail → "없음" (대부분)
Gmail → "있음! 여기 있어" (가끔)
```

- 장점: 설정이 간단
- 단점: 실시간이 아님, 실행 횟수 소비, 서버 부하

사용 예: Gmail Trigger, Google Sheets Trigger

### Webhook (웹훅)
서비스가 이벤트 발생 시 n8n에 직접 알려주는 방식

```
Gmail → (새 이메일 도착 즉시) → n8n에 POST 요청 전송
```

- 장점: 실시간, 효율적, 실행 횟수 절약
- 단점: 서비스가 Webhook을 지원해야 함

사용 예: Stripe Webhook, GitHub Webhook, 직접 만든 폼

---

## 트리거 선택 가이드

```
실시간 응답 필요?
    → 예: Webhook 또는 앱 트리거 (Webhook 방식)
    → 아니오: Schedule Trigger

정기 작업?
    → Schedule Trigger

외부 서비스에서 이벤트 수신?
    → Webhook Trigger 또는 앱 전용 트리거

수동으로 가끔 실행?
    → Manual Trigger
```

---

## 실전 예제 1: Webhook으로 외부 폼 연동

**목표**: 구글 폼, Typeform 등 외부 폼 제출 시 n8n으로 데이터 전송

### Webhook Trigger 설정

```
1. n8n에서 Webhook Trigger 노드 추가
2. HTTP Method: POST
3. Path: form-submit
4. Response Mode: Immediately

생성된 URL 예시:
https://your-n8n.com/webhook/form-submit
```

### 외부 서비스에서 Webhook 등록

```
Google Form (Apps Script로 연결):
function onFormSubmit(e) {
  const payload = {
    name: e.values[1],
    email: e.values[2],
    message: e.values[3],
    timestamp: new Date().toISOString()
  };
  UrlFetchApp.fetch('https://your-n8n.com/webhook/form-submit', {
    method: 'POST',
    contentType: 'application/json',
    payload: JSON.stringify(payload)
  });
}
```

### 받은 데이터 처리

```
[Webhook Trigger]
    ↓ (수신: { name, email, message, timestamp })
[Gmail: 접수 확인 이메일 발송]
    ↓
[Google Sheets: 신청자 목록에 행 추가]
    ↓
[Slack: #leads 채널에 알림]
```

---

## 실전 예제 2: Schedule Trigger 실무 패턴

### 패턴 1: 매일 오전 업무 브리핑

```
Cron: 0 8 * * 1-5   (평일 오전 8시)

[Schedule Trigger]
    ↓
[Google Calendar: 오늘 일정 조회]
    ↓
[Gmail: 미답장 이메일 조회]
    ↓
[Code: 브리핑 메시지 조합]
    ↓
[Slack DM: 본인에게 발송]
```

### 패턴 2: 매시간 데이터 동기화

```
Cron: 0 * * * *   (매시간 정각)

[Schedule Trigger]
    ↓
[HTTP Request: 외부 API에서 최신 데이터 수신]
    ↓
[IF: 변경사항 있음?]
  ├── True  → [Airtable: 레코드 업데이트]
  └── False → (종료)
```

### 패턴 3: 월말 정산 자동화

```
Cron: 0 9 28-31 * *   (매월 28~31일 오전 9시)

[Schedule Trigger]
    ↓
[IF: 오늘이 해당 월 마지막 날?]
    ↓
[Google Sheets: 월간 데이터 집계]
    ↓
[Gmail: 정산 보고서 발송]
```

---

## Webhook 보안 설정

외부에서 호출 가능한 Webhook은 보안이 중요합니다.

### 방법 1: Header Auth

```
Webhook Trigger 설정:
  Authentication: Header Auth
  Header Name: X-Webhook-Secret
  Header Value: my-secret-key-1234

외부 서비스 호출 시:
  Headers: { "X-Webhook-Secret": "my-secret-key-1234" }
```

### 방법 2: Basic Auth

```
Webhook Trigger 설정:
  Authentication: Basic Auth
  Username: n8n
  Password: [강력한 비밀번호]

URL 형식: https://user:pass@your-n8n.com/webhook/path
```

### 방법 3: 서명 검증 (HMAC)

```javascript
// Code 노드: 서명 유효성 검증
const crypto = require('crypto');
const secret = 'your-webhook-secret';
const signature = $json.headers['x-signature'];
const body = JSON.stringify($json.body);

const expectedSig = crypto
  .createHmac('sha256', secret)
  .update(body)
  .digest('hex');

if (signature !== `sha256=${expectedSig}`) {
  throw new Error('Invalid webhook signature');
}
```

---

## 트리거 디버깅

### Webhook이 실행되지 않을 때

```
체크 1: 워크플로우가 활성화되어 있는가?
  → 프로덕션 URL은 활성화 필수
  → 테스트 URL은 비활성화 상태에서도 동작

체크 2: URL이 올바른가?
  → 테스트 URL vs 프로덕션 URL 혼동 여부 확인

체크 3: Content-Type 헤더 확인
  → POST 요청 시 Content-Type: application/json 필요

체크 4: n8n 서버 방화벽/포트 개방 여부
  → 외부에서 접근 가능한지 확인
```

### Schedule Trigger가 실행되지 않을 때

```
체크 1: 워크플로우 활성화 여부
체크 2: Timezone 설정 확인
  → n8n Settings → Timezone이 올바른지
  → 한국 시간: Asia/Seoul
체크 3: 서버 시간 동기화 여부
  → 서버 NTP 동기화 상태 확인
```

---

## 핵심 요약

- Manual: 테스트용, 수동 실행
- Schedule: 정기 자동 실행, Cron 표현식 지원
- Webhook: 외부 서비스 이벤트 수신, 실시간
- 앱별 트리거: Gmail, Slack 등 특정 서비스 전용
- Webhook이 Polling보다 효율적이지만 서비스 지원 필요
- Webhook 보안은 Header Auth 또는 HMAC 서명으로 강화
- 프로덕션 Webhook URL은 반드시 워크플로우 활성화 필요

**다음 레슨**: Set, Code, HTTP Request 등 핵심 노드들을 상세히 알아봅니다.
