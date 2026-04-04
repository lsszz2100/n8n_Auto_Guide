# 레슨 3.8: 서드파티 서비스 연동

---

## 연동의 핵심: 크리덴셜(Credentials)

서드파티 서비스와 연동하려면 인증 정보(Credentials)가 필요합니다.

n8n은 크리덴셜을 암호화하여 안전하게 보관합니다. 한 번 설정하면 여러 워크플로우에서 재사용할 수 있습니다.

---

## 크리덴셜 설정 방법

### OAuth2 방식 (Google, Slack, GitHub 등)

1. n8n 좌측 메뉴 → Credentials → + Add Credential
2. 서비스 선택 (예: Google)
3. "Sign in with Google" 클릭
4. 구글 계정으로 로그인 및 권한 허용
5. 완료!

### API Key 방식 (OpenAI, Airtable 등)

1. 서비스의 개발자 콘솔에서 API 키 발급
2. n8n → Credentials → + Add Credential
3. 서비스 유형 선택
4. API 키 입력 및 저장

---

## 주요 서비스 연동 가이드

### Google 서비스 연동

지원 노드: Gmail, Google Sheets, Google Drive, Google Calendar, Google Docs 등

연동 방법 (OAuth2):
1. Google Cloud Console에서 프로젝트 생성
2. API 활성화 (Gmail API, Sheets API 등)
3. OAuth 2.0 클라이언트 ID 생성
4. n8n에서 Google 크리덴셜 추가

주의사항:
- 각 Google API는 별도로 활성화해야 함
- 개인 Gmail은 무료, G Suite는 도메인 관리자 승인 필요

---

### Slack 연동

지원 노드: Slack (메시지 발송, 채널 관리, 사용자 조회 등)

연동 방법:
1. api.slack.com → Your Apps → Create New App
2. "From scratch" 선택
3. 앱 이름 입력, 워크스페이스 선택
4. Bot Token Scopes 추가:
   - `chat:write` (메시지 발송)
   - `channels:read` (채널 목록)
   - `users:read` (사용자 정보)
5. 앱을 워크스페이스에 설치
6. Bot User OAuth Token 복사
7. n8n 크리덴셜에 토큰 입력

사용 예시:
```
[Webhook] → [Slack: 메시지 발송]
설정:
  채널: #general
  메시지: 새 주문이 들어왔습니다! 주문번호: {{ $json.orderId }}
```

---

### Notion 연동

지원 노드: Notion (페이지 생성, 데이터베이스 조회/추가/수정)

연동 방법:
1. notion.so/my-integrations 접속
2. "New Integration" 클릭
3. 이름 입력, 워크스페이스 선택
4. Internal Integration Token 복사
5. 연동할 Notion 페이지에서 "Connect to" → 생성한 인테그레이션 추가
6. n8n 크리덴셜에 토큰 입력

사용 예시:
```
[Webhook] → [Notion: Create Database Item]
설정:
  Database ID: {데이터베이스 ID}
  Properties:
    Name: {{ $json.name }}
    Email: {{ $json.email }}
    Status: 신규
```

---

### Telegram 연동

지원 노드: Telegram (메시지 수신/발송, 봇 관리)

연동 방법:
1. Telegram에서 @BotFather에게 메시지
2. `/newbot` 명령
3. 봇 이름과 사용자명 설정
4. Bot Token 복사
5. n8n 크리덴셜에 토큰 입력

사용 예시:
```
[Telegram Trigger] → [IF: 명령어 확인] → [Telegram: 메시지 발송]
```

---

### Airtable 연동

지원 노드: Airtable (레코드 조회/생성/수정/삭제)

연동 방법:
1. airtable.com → Account → Developer Hub
2. Create new token
3. 필요한 스코프 선택 (data.records:read, data.records:write 등)
4. n8n 크리덴셜에 토큰 입력

---

### GitHub 연동

지원 노드: GitHub (이슈, PR, 파일, 커밋 등)

연동 방법:
1. GitHub Settings → Developer settings → Personal access tokens
2. "Generate new token (classic)"
3. 필요한 권한 체크 (repo, issues 등)
4. 토큰 복사 → n8n 크리덴셜에 입력

---

## Credential 보안 모범 사례

### 1. 최소 권한 원칙
필요한 권한만 부여하세요.
- 읽기만 필요하다면 읽기 권한만
- 특정 리소스만 필요하다면 해당 리소스만

### 2. 정기적인 토큰 갱신
API 키와 토큰은 정기적으로(분기별) 재발급받고 갱신하세요.

### 3. 환경별 분리
개발/운영 환경의 크리덴셜을 별도로 관리하세요.

### 4. 크리덴셜 공유 시 주의
팀원과 공유 시 실제 값이 아닌 n8n 내의 크리덴셜 참조로 공유하세요.

---

## OAuth vs API Key: 어떤 것을 써야 하나?

| 항목 | OAuth2 | API Key |
|------|--------|---------|
| 보안 | 높음 | 중간 |
| 설정 복잡도 | 높음 | 낮음 |
| 만료 | 자동 갱신 | 수동 갱신 |
| 권한 범위 | 세밀하게 제어 | 고정 |
| 주요 서비스 | Google, Slack | OpenAI, Airtable |

---

## 실전 예제 1: Google Sheets + Gmail 연동

**목표**: 구글 시트에 새 행이 추가되면 해당 이메일로 자동 안내 메일 발송

```
[Google Sheets Trigger: 새 행 감지]
    ↓
[Code: 이메일 본문 조합]
    ↓
[Gmail: 이메일 발송]
    ↓
[Google Sheets: 발송 상태 업데이트]
```

### 워크플로우 설정

```
1. Google Sheets Trigger
   - Spreadsheet ID: [스프레드시트 ID]
   - Sheet Name: "신청자목록"
   - Event: Row Added

2. Code 노드: 본문 생성
```
```javascript
const row = $json;
return [{
  json: {
    to: row['이메일'],
    subject: `[확인] ${row['이름']}님의 신청이 접수되었습니다`,
    body: `안녕하세요, ${row['이름']}님!\n\n신청일: ${row['신청일자']}\n내용: ${row['신청내용']}\n\n담당자가 확인 후 연락드리겠습니다.`
  }
}];
```

```
3. Gmail: Send Email
   - To: {{ $json.to }}
   - Subject: {{ $json.subject }}
   - Message: {{ $json.body }}

4. Google Sheets: Update Row
   - 발송상태 컬럼: "발송완료"
   - 발송시각: {{ $now }}
```

---

## 실전 예제 2: GitHub Issues + Slack 알림

**목표**: 새 GitHub 이슈가 등록되면 Slack 개발팀 채널에 즉시 알림

```
[GitHub Trigger: Issue Opened]
    ↓
[IF: 라벨 "bug" 포함?]
  ├── True  → [Slack: #bug-alerts 채널 긴급 알림]
  └── False → [Slack: #dev-general 채널 일반 알림]
```

### Slack 메시지 구성

```
버그 긴급 알림 메시지:
🚨 *새 버그 이슈 등록됨!*

제목: {{ $json.issue.title }}
등록자: {{ $json.issue.user.login }}
라벨: {{ $json.issue.labels.map(l => l.name).join(', ') }}
링크: {{ $json.issue.html_url }}

즉시 확인 부탁드립니다.
```

---

## 실전 예제 3: OpenAI + Notion 연동

**목표**: Notion 데이터베이스에 새 항목이 추가되면 OpenAI로 요약 생성 후 자동 저장

```
[Notion Trigger: New Database Item]
    ↓
[OpenAI: 텍스트 요약 요청]
    ↓
[Notion: Update Page - 요약 필드에 저장]
```

### OpenAI 노드 설정

```
노드: OpenAI → Message a Model
설정:
  Model: gpt-4o-mini
  Messages:
    System: "다음 내용을 3줄 이내로 핵심만 요약해 주세요."
    User: {{ $json.properties.Content.rich_text[0].plain_text }}
```

### Notion 업데이트 설정

```
노드: Notion → Update Page
설정:
  Page ID: {{ $('Notion Trigger').item.json.id }}
  Properties:
    Summary: {{ $json.choices[0].message.content }}
    요약생성일: {{ $now.toISO() }}
```

---

## Rate Limit 처리

연동 서비스마다 API 호출 횟수 제한이 있습니다. 대량 처리 시 반드시 고려해야 합니다.

### 주요 서비스 Rate Limit

| 서비스 | 제한 | 대응 방법 |
|--------|------|-----------|
| Gmail | 250 quota/초 | Wait 노드로 간격 조절 |
| Slack | 1 msg/초 (채널당) | 배치 처리 |
| Notion | 3 req/초 | Wait 노드 추가 |
| GitHub | 5000 req/시간 | 캐싱 활용 |
| OpenAI | 모델별 상이 | 에러 재시도 로직 |

### Wait 노드로 Rate Limit 회피

```
[Loop Over Items]
    ↓
[API 호출 노드]
    ↓
[Wait: 1초]   ← Rate Limit 방지
    ↓
[다음 반복]
```

### 429 에러 자동 재시도

```javascript
// Code 노드: 재시도 로직
const maxRetries = 3;
let attempt = 0;

while (attempt < maxRetries) {
  try {
    // API 호출 로직
    break;
  } catch (error) {
    if (error.message.includes('429')) {
      attempt++;
      await new Promise(r => setTimeout(r, 2000 * attempt)); // 점진적 대기
    } else {
      throw error;
    }
  }
}
```

---

## 연동이 안 될 때 체크리스트

1. 크리덴셜이 올바른지 확인
   - API 키를 다시 복사하여 저장
   - OAuth는 재인증 시도

2. 권한(Scope)이 충분한지 확인
   - 필요한 API 권한이 모두 활성화되었는지
   - Google API는 Cloud Console에서 해당 API 활성화 여부 확인

3. API 제한 확인
   - 무료 플랜의 경우 API 제한이 있을 수 있음
   - Rate Limit 초과 여부 확인 (HTTP 429 응답)

4. 서비스 상태 확인
   - 해당 서비스의 상태 페이지 확인 (status.slack.com 등)

5. n8n 버전 호환성 확인
   - 노드 버전이 서비스 API 버전과 맞는지 확인
   - n8n 업데이트 후 크리덴셜 재설정 필요한 경우 있음

---

## 핵심 요약

- 크리덴셜은 한 번 설정하면 여러 워크플로우에서 재사용
- OAuth2: 보안이 높고 자동 갱신 (Google, Slack 등)
- API Key: 설정이 간단 (OpenAI, Airtable 등)
- 최소 권한 원칙, 정기 갱신으로 보안 유지
- Rate Limit은 Wait 노드와 재시도 로직으로 대응
- 연동 문제 발생 시 크리덴셜 → 권한 → Rate Limit 순으로 확인

**다음 레슨**: 3장 핵심 내용을 정리합니다.
