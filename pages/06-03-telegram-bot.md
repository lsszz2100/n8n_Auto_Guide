# 레슨 6.3: Telegram 봇 만들기

---

## Telegram 봇의 장점

Telegram 봇은 다음 이유로 매우 인기 있습니다:
- **무료**: API 사용료 없음
- **빠름**: 실시간 메시지 전달
- **강력함**: 버튼, 이미지, 파일, 인라인 키보드 지원
- **간편함**: BotFather로 1분 안에 봇 생성

---

## 봇 생성 (BotFather)

1. Telegram에서 `@BotFather` 검색 후 대화 시작
2. `/newbot` 명령 입력
3. 봇 이름 입력: `MyCompany 알림봇`
4. 봇 사용자명 입력: `mycompany_notify_bot` (bot으로 끝나야 함)
5. **Bot Token** 수령: `7123456789:AAF...`

### n8n 크리덴셜 설정

Credentials → Telegram API → Access Token 입력 (BotFather에서 받은 토큰)

---

## 봇 Chat ID 확인

메시지를 보낼 대상의 Chat ID가 필요합니다.

```
1. 봇에게 아무 메시지 전송
2. 브라우저에서 접속:
   https://api.telegram.org/bot{TOKEN}/getUpdates
3. 응답에서 "chat":{"id": 123456789} 확인
```

---

## 프로젝트 1: 명령어 기반 봇

### 워크플로우 구조

```
[Telegram Trigger: 메시지 수신]
    ↓
[Code: 명령어 파싱]
    ↓
[Switch: 명령어 종류]
  ├── /start → 환영 메시지
  ├── /help → 도움말
  ├── /status → 시스템 상태 조회
  ├── /report → 오늘 리포트
  └── Default → "모르는 명령어" 안내
```

### 명령어 파싱 코드

```javascript
// Code 노드
const message = $input.item.json.message;
const text = message?.text || '';
const chatId = message?.chat?.id;
const userId = message?.from?.id;
const username = message?.from?.username || '사용자';

// 명령어와 파라미터 분리
const parts = text.split(' ');
const command = parts[0].toLowerCase(); // /start, /help 등
const params = parts.slice(1).join(' '); // 명령어 뒤 텍스트

return [{
  json: { chatId, userId, username, command, params, fullText: text }
}];
```

### 각 명령어 응답 설정

**/start 환영 메시지:**
```javascript
// Telegram 노드 설정
{
  chatId: "{{ $json.chatId }}",
  text: `안녕하세요, ${$json.username}님! 👋\n\n저는 업무 알림 봇입니다.\n\n사용 가능한 명령어:\n/help - 도움말\n/status - 시스템 상태\n/report - 오늘 현황`
}
```

---

## 프로젝트 2: 인라인 키보드 봇

버튼을 클릭으로 메뉴를 탐색하는 인터랙티브 봇:

```javascript
// Telegram 노드 - 인라인 키보드 메시지
{
  chatId: "{{ $json.chatId }}",
  text: "무엇을 도와드릴까요?",
  reply_markup: {
    inline_keyboard: [
      [
        { text: "📊 오늘 현황", callback_data: "report_today" },
        { text: "📈 주간 리포트", callback_data: "report_week" }
      ],
      [
        { text: "🔔 알림 설정", callback_data: "notifications" },
        { text: "❓ 도움말", callback_data: "help" }
      ]
    ]
  }
}
```

### 버튼 클릭 처리

```
[Telegram Trigger: Callback Query]
    ↓
[Switch: callback_data 값]
  ├── "report_today" → [오늘 현황 조회 워크플로우]
  ├── "report_week" → [주간 리포트 워크플로우]
  └── "help" → [도움말 발송]
```

---

## 프로젝트 3: 주문 알림 봇

쇼핑몰 주문 발생 시 관리자에게 Telegram 알림:

```
[Webhook: 주문 시스템]
    ↓
[Code: 알림 메시지 구성]
    ↓
[Telegram: 관리자에게 발송]
```

```javascript
// 알림 메시지 구성
const order = $input.item.json;
const adminChatId = '123456789'; // 관리자 Chat ID

const message = `🛍️ *새 주문 알림*

📋 주문번호: \`${order.orderId}\`
👤 고객: ${order.customerName}
📦 상품: ${order.productName}
💰 금액: ${order.amount.toLocaleString()}원
🕐 시간: ${$now.toFormat('HH:mm:ss')}

[주문 관리 바로가기](https://admin.yourshop.com/orders/${order.orderId})`;

return [{
  json: {
    chatId: adminChatId,
    text: message,
    parseMode: 'Markdown'
  }
}];
```

---

## 프로젝트 4: 파일 전송 봇

자동 생성된 리포트 파일을 Telegram으로 전송:

```
[Schedule: 매일 오후 6시]
    ↓
[Google Sheets: 데이터 수집]
    ↓
[Code: CSV 생성]
    ↓
[Telegram: 파일 전송]
  Type: document
  File: {{ $binary.data }}
  Caption: "오늘의 일일 리포트입니다."
```

---

## 봇 보안 설정

허가된 사용자만 봇을 사용할 수 있도록:

```javascript
// Code 노드에서 사용자 검증
const allowedUsers = [123456789, 987654321]; // 허가된 User ID 목록
const userId = $json.userId;

if (!allowedUsers.includes(userId)) {
  return [{
    json: {
      action: 'deny',
      chatId: $json.chatId,
      message: '죄송합니다. 이 봇을 사용할 권한이 없습니다.'
    }
  }];
}

return [{ json: { action: 'allow', ...$json } }];
```

---

## 핵심 요약

- BotFather로 1분 안에 봇 생성, Bot Token으로 n8n 연동
- Telegram Trigger: 메시지, 버튼 클릭 등 모든 이벤트 감지
- 인라인 키보드로 클릭 기반 메뉴 구성
- Markdown 지원으로 볼드, 이탤릭, 링크 등 리치 메시지
- Chat ID 기반 사용자 권한 관리로 보안 강화

**다음 레슨**: Google Sheets를 활용한 데이터 자동화를 구현합니다.
