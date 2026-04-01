# 레슨 6.2: Slack 봇 만들기

---

## 프로젝트 개요

**목표**: 팀 업무를 도와주는 Slack 명령어 봇 구현

**주요 기능:**
- `/report` → 오늘의 업무 현황 리포트
- `/status` → 시스템 상태 확인
- 새 이슈 발생 시 자동 채널 알림

---

## Slack 앱 설정

### 1. Slack App 생성

1. `api.slack.com/apps` 접속
2. "Create New App" → "From Scratch"
3. 앱 이름: "n8n 봇", 워크스페이스 선택

### 2. Bot Token Scopes 설정

OAuth & Permissions → Bot Token Scopes 추가:
```
chat:write        - 메시지 발송
channels:read     - 채널 목록 조회
users:read        - 사용자 정보 조회
commands          - 슬래시 명령어 처리
incoming-webhook  - Webhook 메시지 발송
```

### 3. 앱 설치 및 토큰 복사

"Install to Workspace" → Bot User OAuth Token 복사 (`xoxb-...`)

### 4. n8n 크리덴셜 설정

Credentials → Slack API → Access Token 입력

---

## 프로젝트 1: 슬래시 명령어 봇

### Slash Command 등록

Slack App 설정 → Slash Commands → Create New Command:
```
Command: /report
Request URL: https://your-n8n.com/webhook/slack-report
Short Description: 오늘의 업무 현황 리포트
```

### n8n 워크플로우

```
[Webhook Trigger]
  Path: slack-report
  Method: POST
    ↓
[Code: Slack 요청 검증]
    ↓
[Google Sheets: 오늘 데이터 조회]
    ↓
[Code: 리포트 생성]
    ↓
[Respond to Webhook: Slack Block Kit 형식으로 응답]
```

### Slack Block Kit 메시지 작성

Slack의 Block Kit은 리치 메시지를 만드는 형식입니다:

```javascript
// Code 노드에서 Block Kit 메시지 생성
const data = $input.item.json;

return [{
  json: {
    response_type: "in_channel",
    blocks: [
      {
        type: "header",
        text: {
          type: "plain_text",
          text: `📊 ${$now.toFormat('M월 d일')} 업무 현황`
        }
      },
      {
        type: "section",
        fields: [
          { type: "mrkdwn", text: `*총 주문*\n${data.totalOrders}건` },
          { type: "mrkdwn", text: `*매출*\n${data.revenue.toLocaleString()}원` },
          { type: "mrkdwn", text: `*신규 고객*\n${data.newCustomers}명` },
          { type: "mrkdwn", text: `*처리 대기*\n${data.pending}건` }
        ]
      },
      {
        type: "divider"
      },
      {
        type: "section",
        text: {
          type: "mrkdwn",
          text: data.pending > 10
            ? "⚠️ 처리 대기 건수가 많습니다. 즉시 확인이 필요합니다."
            : "✅ 모든 처리가 정상 범위입니다."
        }
      }
    ]
  }
}];
```

---

## 프로젝트 2: 자동 알림 봇

### 새 주문 발생 시 Slack 알림

```
[Webhook: 주문 시스템에서 호출]
    ↓
[Code: 주문 데이터 처리]
    ↓
[IF: 주문 금액 > 100만원?]
  ├── True → [Slack: #bigdeal 채널에 알림]
  └── False → [Slack: #orders 채널에 일반 알림]
```

```javascript
// Slack 노드 메시지 설정
{
  channel: "#orders",
  text: `새 주문 알림`,
  blocks: [
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: `🛒 *새 주문이 들어왔습니다!*\n\n고객: ${$json.customerName}\n상품: ${$json.productName}\n금액: ${$json.amount.toLocaleString()}원`
      }
    },
    {
      type: "actions",
      elements: [
        {
          type: "button",
          text: { type: "plain_text", text: "주문 확인" },
          url: `https://admin.yourstore.com/orders/${$json.orderId}`,
          style: "primary"
        }
      ]
    }
  ]
}
```

---

## 프로젝트 3: 데일리 스탠드업 봇

### 매일 오전 9시 팀 스탠드업 질문 발송

```
[Schedule: 0 9 * * 1-5] (평일 오전 9시)
    ↓
[Slack: #standup 채널에 스레드 생성]
  메시지:
  "🌅 *오늘의 스탠드업 시간입니다!*
  각자 다음 항목을 답변해 주세요:
  1. 어제 완료한 일
  2. 오늘 할 일
  3. 블로커 (있다면)"
    ↓
[Schedule: 30분 후]
    ↓
[Slack: 미응답자 DM 리마인더]
```

---

## Slack Webhook을 통한 단순 알림

Incoming Webhook은 가장 간단한 알림 방법입니다:

```
[어떤 트리거]
    ↓
[HTTP Request]
  Method: POST
  URL: https://hooks.slack.com/services/T.../B.../xxx
  Body:
    {
      "text": "알림 내용",
      "username": "n8n 봇",
      "icon_emoji": ":robot_face:"
    }
```

---

## 핵심 요약

- Slack 앱 생성 후 Bot Token으로 n8n 연동
- Slash Command: 사용자가 명령어 입력 → n8n Webhook 호출
- Block Kit: 버튼, 이미지, 테이블 등 리치 메시지 구성
- Incoming Webhook: 가장 빠르고 간단한 알림 방법
- 스케줄 + Slack = 자동 팀 커뮤니케이션 자동화

**다음 레슨**: Telegram 봇을 만들어 더 강력한 자동화를 구현합니다.
