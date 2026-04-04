# 레슨 6.1: Gmail 이메일 자동화

---

## 프로젝트 개요

**목표**: 고객 문의 이메일을 자동으로 분류하고 처리하는 시스템

**완성 워크플로우:**
```
[Gmail Trigger] → [분류] → [VIP] → [우선 처리 + Slack 알림]
                          → [일반] → [자동 답장 + CRM 기록]
                          → [스팸] → [라벨링 후 무시]
```

---

## 사전 준비

### Google OAuth 크리덴셜 설정

1. Google Cloud Console → 새 프로젝트 생성
2. Gmail API 활성화
3. OAuth 2.0 클라이언트 ID 생성
4. n8n → Credentials → Google OAuth2 API 추가
5. Client ID, Client Secret 입력 후 Google 로그인

---

## STEP 1: Gmail Trigger 설정

```
노드: Gmail Trigger
설정:
  Credentials: [Google 계정]
  Event: Message Received
  Filters:
    Label Names: INBOX
    Include Attachments: false
  Poll Times: Every Minute (테스트용)
```

**Production 팁**: Polling 대신 Google Pub/Sub Webhook으로 전환하면 실시간 + 실행 횟수 절약

---

## STEP 2: 이메일 데이터 추출

```javascript
// Code 노드: 이메일 핵심 정보 추출
const email = $input.item.json;

// 발신자 도메인 추출
const senderEmail = email.from?.value?.[0]?.address || '';
const senderDomain = senderEmail.split('@')[1] || '';
const subject = email.subject || '';
const snippet = email.snippet || '';
const threadId = email.threadId;
const messageId = email.id;

// VIP 도메인 목록 (팀에서 관리)
const vipDomains = ['bigclient.com', 'enterprise.co.kr', 'partner.com'];
const isVip = vipDomains.includes(senderDomain);

// 키워드 기반 분류
const urgentKeywords = ['긴급', '즉시', 'ASAP', '장애', '결제 오류'];
const spamKeywords = ['광고', '홍보', '이벤트 당첨', '무료 증정'];

const isUrgent = urgentKeywords.some(kw => subject.includes(kw) || snippet.includes(kw));
const isSpam = spamKeywords.some(kw => subject.includes(kw));

let category = 'general';
if (isSpam) category = 'spam';
else if (isVip || isUrgent) category = 'vip';

return [{
  json: {
    senderEmail, senderDomain, subject, snippet,
    threadId, messageId, isVip, isUrgent, isSpam, category
  }
}];
```

---

## STEP 3: Switch 분기 처리

```
Switch 노드:
  Field: {{ $json.category }}
  Case 1: "vip" → VIP 처리 경로
  Case 2: "spam" → 스팸 처리 경로
  Default: 일반 처리 경로
```

---

## STEP 4: VIP 경로 처리

```
[Gmail: 라벨 추가 "VIP-긴급"]
    ↓
[Slack: #sales-vip 채널 알림]
  메시지: "🚨 VIP 이메일 도착!
  발신: {{ $json.senderEmail }}
  제목: {{ $json.subject }}
  즉시 확인 필요!"
    ↓
[Google Sheets: VIP 문의 로그 기록]
```

---

## STEP 5: 일반 이메일 자동 답장

```
[Gmail: 답장 발송]
  To: {{ $json.senderEmail }}
  Subject: Re: {{ $json.subject }}
  Body:
    안녕하세요,

    문의 주셔서 감사합니다.
    담당 팀에서 영업일 기준 1-2일 내로 답변 드리겠습니다.

    문의 접수 번호: {{ $now.toFormat('yyyyMMdd') }}-{{ $json.messageId.slice(-6) }}

    감사합니다.
    고객지원팀
```

---

## STEP 6: 스팸 처리

```
[Gmail: 라벨 추가 "자동-스팸"]
    ↓
[Gmail: 읽음 처리]
    ↓
(종료 - 추가 처리 없음)
```

---

## 전체 워크플로우 완성 확인

테스트용 이메일 발송 후 확인:
- [ ] VIP 도메인 이메일 → Slack 알림 수신
- [ ] 일반 이메일 → 자동 답장 수신
- [ ] 스팸 키워드 이메일 → 라벨 적용 확인
- [ ] Google Sheets 로그 기록 확인

---

## 심화: 첨부파일 자동 처리

```
[Gmail Trigger]
    ↓
[IF: 첨부파일 있음?]
  ├── True → [Gmail: 첨부파일 다운로드]
  │              ↓
  │           [Google Drive: 폴더에 저장]
  │              ↓
  │           [Slack: "첨부파일 저장됨" 알림]
  └── False → 기존 처리 흐름
```

---

## 프로젝트 2: 주간 이메일 보고서 자동 생성

**목표**: 매주 월요일 오전, 지난 주 이메일 통계를 팀 Slack에 자동 발송

**완성 워크플로우:**
```
[Cron: 매주 월요일 09:00]
    ↓
[Gmail: 지난 7일 이메일 검색]
    ↓
[Code: 통계 집계]
    ↓
[Slack: 주간 보고서 발송]
```

### STEP 1: Cron 트리거 설정

```
노드: Schedule Trigger
설정:
  Mode: Custom (Cron)
  Expression: 0 9 * * 1   (매주 월요일 오전 9시)
```

### STEP 2: Gmail 검색 쿼리

```
노드: Gmail → Get Many Messages
설정:
  Filters:
    Query: after:{{ $now.minus({days: 7}).toFormat('yyyy/MM/dd') }}
    Max Results: 500
```

### STEP 3: 통계 집계 코드

```javascript
// Code 노드: 주간 통계 집계
const emails = $input.all();

const stats = {
  total: emails.length,
  byDay: {},
  topSenders: {},
  unread: 0,
};

emails.forEach(item => {
  const email = item.json;

  // 요일별 통계
  const date = new Date(parseInt(email.internalDate)).toLocaleDateString('ko-KR', { weekday: 'short' });
  stats.byDay[date] = (stats.byDay[date] || 0) + 1;

  // 발신자 통계
  const sender = email.from?.value?.[0]?.address || 'unknown';
  stats.topSenders[sender] = (stats.topSenders[sender] || 0) + 1;

  // 미읽음 카운트
  if (email.labelIds?.includes('UNREAD')) stats.unread++;
});

// 상위 5 발신자 정렬
const topSenders = Object.entries(stats.topSenders)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 5)
  .map(([email, count]) => `• ${email}: ${count}건`)
  .join('\n');

return [{
  json: {
    total: stats.total,
    unread: stats.unread,
    topSenders,
    byDay: JSON.stringify(stats.byDay),
  }
}];
```

### STEP 4: Slack 보고서 메시지

```
노드: Slack → Send Message
설정:
  Channel: #team-report
  Message:
    📊 *주간 이메일 보고서*
    기간: 지난 7일

    - 총 수신: {{ $json.total }}건
    - 미읽음: {{ $json.unread }}건

    *상위 발신자:*
    {{ $json.topSenders }}
```

---

## 프로젝트 3: 이메일 기반 업무 할당 시스템

**목표**: 특정 키워드가 포함된 이메일을 받으면 Notion 태스크 자동 생성 + 담당자 Slack DM 발송

**완성 워크플로우:**
```
[Gmail Trigger: 라벨 "업무요청"]
    ↓
[Code: 담당자 결정]
    ↓
[Notion: 태스크 생성]
    ↓
[Slack: 담당자 DM 발송]
    ↓
[Gmail: 접수 확인 답장]
```

### 담당자 라우팅 코드

```javascript
// Code 노드: 키워드로 담당자 배정
const subject = $json.subject || '';
const body = $json.snippet || '';
const text = subject + ' ' + body;

const routing = [
  { keywords: ['디자인', 'UI', '시안', '시각화'], assignee: 'design_team', slack: '@design-team' },
  { keywords: ['개발', '버그', 'API', '오류'], assignee: 'dev_team', slack: '@dev-team' },
  { keywords: ['결제', '환불', '청구'], assignee: 'finance_team', slack: '@finance' },
  { keywords: ['계약', '견적', '제안서'], assignee: 'sales_team', slack: '@sales' },
];

let assigned = { assignee: 'general', slack: '@general-support' };

for (const route of routing) {
  if (route.keywords.some(kw => text.includes(kw))) {
    assigned = { assignee: route.assignee, slack: route.slack };
    break;
  }
}

return [{
  json: {
    ...$json,
    assignee: assigned.assignee,
    slackMention: assigned.slack,
    taskTitle: `[이메일 업무] ${$json.subject}`,
    dueDate: new Date(Date.now() + 2 * 24 * 60 * 60 * 1000).toISOString().split('T')[0], // 2일 후
  }
}];
```

### Notion 태스크 생성 설정

```
노드: Notion → Create Page
설정:
  Database ID: [업무 관리 DB ID]
  Properties:
    Name: {{ $json.taskTitle }}
    담당팀: {{ $json.assignee }}
    마감일: {{ $json.dueDate }}
    상태: "진행중"
    출처: "이메일"
    발신자: {{ $json.senderEmail }}
```

---

## 트러블슈팅

### Gmail Trigger가 실행되지 않을 때

```
원인 1: OAuth 권한 만료
해결: Credentials 재인증 (n8n → Credentials → Gmail → Reconnect)

원인 2: Polling 간격 설정 오류
해결: Poll Times를 "Every Minute"으로 변경 후 테스트

원인 3: Gmail API 할당량 초과
해결: Google Cloud Console → API 할당량 확인
  일일 한도: 1,000,000 유닛
  사용자당: 250 유닛/초
```

### 자동 답장이 무한 루프를 일으킬 때

```
문제: 자동 답장 → 상대방 자동 답장 → 반복
해결: IF 노드 추가

조건: {{ $json.subject.startsWith('Re:') === false }}
      AND {{ $json.senderEmail !== 'noreply@' }}
      AND {{ $json.labelIds.includes('SENT') === false }}
```

### 이메일 파싱 오류

```javascript
// 안전한 파싱 패턴 (null 체크 필수)
const senderEmail = email.from?.value?.[0]?.address
  ?? email.from?.text?.match(/[\w.-]+@[\w.-]+/)?.[0]
  ?? 'unknown@unknown.com';

const subject = email.subject || '(제목 없음)';
const body = email.text || email.snippet || '';
```

---

## 핵심 요약

- Gmail Trigger로 실시간 이메일 감지
- Code 노드에서 발신자/키워드 기반 자동 분류
- Switch로 VIP/일반/스팸 각각 다르게 처리
- 자동 답장으로 고객 응답 시간 즉시 단축
- Google Sheets 로그로 모든 문의 추적
- 주간 통계 자동 집계로 팀 가시성 확보
- 키워드 기반 담당자 자동 배정으로 업무 분배 자동화

**다음 레슨**: Slack 봇을 만들어 팀 업무를 자동화합니다.
