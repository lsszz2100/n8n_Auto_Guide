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

## 핵심 요약

- Gmail Trigger로 실시간 이메일 감지
- Code 노드에서 발신자/키워드 기반 자동 분류
- Switch로 VIP/일반/스팸 각각 다르게 처리
- 자동 답장으로 고객 응답 시간 즉시 단축
- Google Sheets 로그로 모든 문의 추적

**다음 레슨**: Slack 봇을 만들어 팀 업무를 자동화합니다.
