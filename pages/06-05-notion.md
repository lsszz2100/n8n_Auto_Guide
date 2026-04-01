# 레슨 6.5: Notion 연동 자동화

---

## Notion + n8n = 강력한 지식 관리 자동화

Notion은 단순한 메모 앱을 넘어 팀의 지식 베이스, 프로젝트 관리, CRM으로 활용됩니다.
n8n과 연동하면 수동으로 입력하던 모든 것을 자동화할 수 있습니다.

---

## Notion 연동 설정

### Integration 생성

1. `notion.so/my-integrations` 접속
2. "New integration" 클릭
3. 이름: "n8n 자동화", 워크스페이스 선택
4. **Internal Integration Token** 복사 (`secret_...`)

### 데이터베이스 연결 허용

n8n이 접근할 Notion 페이지/데이터베이스에서:
- 우측 상단 "..." → "Add connections"
- 생성한 Integration 선택

### n8n 크리덴셜 설정

Credentials → Notion API → API Key (token) 입력

---

## Database ID 확인

Notion 데이터베이스 URL에서 추출:
```
https://notion.so/workspace/{DATABASE_ID}?v=...
```

---

## 주요 작업

### 1. 페이지 생성 (Create Page)

```
[Notion: Create Page]
설정:
  Database: {Database ID}
  Properties:
    Name (title): {{ $json.name }}
    Status (select): "신규"
    Email (email): {{ $json.email }}
    Date (date): {{ $now.toISO() }}
    Tags (multi_select): ["자동화", "n8n"]
```

### 2. 데이터베이스 조회 (Get All)

```
[Notion: Get All]
설정:
  Resource: Database Page
  Database ID: {ID}
  Filters:
    Property: Status
    Equals: "진행중"
  Sort:
    Property: Created time
    Direction: Descending
  Page Size: 50
```

### 3. 페이지 업데이트 (Update)

```
[Notion: Update Page]
설정:
  Page ID: {{ $json.pageId }}
  Properties:
    Status: "완료"
    Completion Date: {{ $now.toISO() }}
```

### 4. 페이지 아카이브 (Delete)

```
[Notion: Archive Page]
설정:
  Page ID: {{ $json.pageId }}
```

---

## 프로젝트 1: 고객 문의 → Notion 자동 등록

웹사이트 문의 폼 제출 시 Notion CRM에 자동 등록:

```
[Webhook: 폼 제출]
    ↓
[Code: 데이터 정제 + 중복 확인]
    ↓
[Notion: Create Page in CRM Database]
  Name: {{ $json.name }}
  Email: {{ $json.email }}
  Company: {{ $json.company }}
  Inquiry: {{ $json.message }}
  Source: "웹사이트 폼"
  Status: "신규 문의"
  담당자: (빈값)
    ↓
[Slack: #crm 채널에 신규 리드 알림]
```

---

## 프로젝트 2: 할 일 자동 관리 시스템

이메일, Slack 메시지 등에서 할 일을 자동으로 Notion에 추가:

```
[Gmail Trigger: 라벨 "to-do" 이메일]
    ↓
[Notion: Create Page in Tasks Database]
  Task Name: {{ $json.subject }}
  Source: "이메일"
  From: {{ $json.from }}
  Due Date: {{ $now.plus({days: 3}).toISO() }}
  Priority: "중간"
  Link: [원본 이메일]
```

---

## 프로젝트 3: 주간 리포트 Notion 페이지 자동 생성

```
[Schedule: 매주 금요일 오후 5시]
    ↓
[여러 API: 주간 데이터 수집]
    ↓
[Code: 리포트 데이터 구성]
    ↓
[Notion: Create Page in Weekly Reports Database]
  Title: {{ $now.toFormat('yyyy년 W주차') }} 주간 리포트
  Content Block 1: 총 매출 {{ $json.revenue }}원
  Content Block 2: 신규 고객 {{ $json.newUsers }}명
  Content Block 3: 완료 프로젝트 {{ $json.completedProjects }}건
```

---

## Notion 블록 구조 이해

Notion 페이지는 블록(Block)으로 구성됩니다:

```json
{
  "type": "paragraph",
  "paragraph": {
    "rich_text": [
      {
        "type": "text",
        "text": { "content": "본문 내용" }
      }
    ]
  }
}
```

```json
{
  "type": "heading_2",
  "heading_2": {
    "rich_text": [
      {
        "type": "text",
        "text": { "content": "소제목" }
      }
    ]
  }
}
```

---

## 프로퍼티 타입별 입력 형식

| 타입 | 입력 형식 |
|------|-----------|
| title | `[{"text": {"content": "제목"}}]` |
| rich_text | `[{"text": {"content": "내용"}}]` |
| number | `42` |
| select | `{"name": "옵션값"}` |
| multi_select | `[{"name": "태그1"}, {"name": "태그2"}]` |
| date | `{"start": "2026-04-01"}` |
| checkbox | `true` 또는 `false` |
| url | `"https://..."` |
| email | `"user@example.com"` |

---

## 실전 예제 모음

---

### 예제 1: 회의록 자동화 (Google Meet 종료 → AI 요약 → Notion 페이지 자동 생성)

회의가 끝나면 녹취록을 AI로 요약하고 Notion에 구조화된 회의록 페이지를 자동으로 만드는 워크플로우입니다. 회의 후 수동으로 요약을 작성하는 반복 작업을 완전히 없앱니다.

워크플로우 구조:

```
[Webhook: /meeting-ended]
  Google Meet 또는 Zoom Webhook으로 회의 종료 이벤트 수신
  Body: { meetingId, title, duration, transcript, participants }
    ↓
[Code: 트랜스크립트 전처리]
  참가자 목록 정리, 발언자별 분류
    ↓
[OpenAI: 회의록 AI 요약]
  Model: gpt-4o
  프롬프트: 결정 사항, 액션 아이템, 다음 미팅 일정 추출
    ↓
[Code: Notion 블록 구조 생성]
  제목, 참가자, 요약, 액션 아이템을 블록 배열로 변환
    ↓
[Notion: Create Page]
  Database: 회의록 DB
  Properties + Children 블록 삽입
    ↓
[Slack: #meetings 채널에 회의록 링크 공유]
```

Code 노드 - 트랜스크립트 전처리:

```javascript
const transcript = $input.item.json.transcript;
const participants = $input.item.json.participants;
const title = $input.item.json.title;
const duration = $input.item.json.duration;

// 참가자 이름 목록 추출
const participantNames = participants.map(p => p.name).join(', ');

// 트랜스크립트가 너무 길면 앞 4000자만 사용 (토큰 절약)
const trimmedTranscript = transcript.length > 4000
  ? transcript.slice(0, 4000) + '\n...(이하 생략)'
  : transcript;

return [{
  json: {
    title,
    duration,
    participantNames,
    transcript: trimmedTranscript,
    meetingDate: new Date().toISOString().slice(0, 10)
  }
}];
```

OpenAI 노드 설정:

```
[OpenAI: Message a Model]
  Model: gpt-4o
  System: 당신은 회의록 작성 전문가입니다. 주어진 회의 트랜스크립트를 분석하여 구조화된 회의록을 JSON으로 작성합니다.
  User: |
    회의 제목: {{ $json.title }}
    참가자: {{ $json.participantNames }}
    트랜스크립트:
    {{ $json.transcript }}

    다음 JSON 형식으로만 답변하세요:
    {
      "summary": "3-5문장 핵심 요약",
      "decisions": ["결정 사항 1", "결정 사항 2"],
      "action_items": [
        { "task": "작업 내용", "owner": "담당자", "due": "기한" }
      ],
      "next_meeting": "다음 미팅 일정 (없으면 null)"
    }
```

Code 노드 - Notion 블록 구조 생성:

```javascript
const aiRaw = $input.item.json.message.content;
const ai = JSON.parse(aiRaw);
const meetingDate = $('Code: 트랜스크립트 전처리').item.json.meetingDate;
const title = $('Code: 트랜스크립트 전처리').item.json.title;

// 액션 아이템 텍스트 조립
const actionLines = ai.action_items.map(item =>
  `${item.task} (담당: ${item.owner} / 기한: ${item.due})`
);

// Notion children 블록 배열
const children = [
  {
    type: 'heading_2',
    heading_2: { rich_text: [{ type: 'text', text: { content: '회의 요약' } }] }
  },
  {
    type: 'paragraph',
    paragraph: { rich_text: [{ type: 'text', text: { content: ai.summary } }] }
  },
  {
    type: 'heading_2',
    heading_2: { rich_text: [{ type: 'text', text: { content: '결정 사항' } }] }
  },
  ...ai.decisions.map(d => ({
    type: 'bulleted_list_item',
    bulleted_list_item: { rich_text: [{ type: 'text', text: { content: d } }] }
  })),
  {
    type: 'heading_2',
    heading_2: { rich_text: [{ type: 'text', text: { content: '액션 아이템' } }] }
  },
  ...actionLines.map(line => ({
    type: 'to_do',
    to_do: {
      rich_text: [{ type: 'text', text: { content: line } }],
      checked: false
    }
  }))
];

return [{
  json: {
    pageTitle: `[회의록] ${title} - ${meetingDate}`,
    meetingDate,
    children,
    nextMeeting: ai.next_meeting
  }
}];
```

Notion Create Page 노드 설정:

```
[Notion: Create Page]
  Resource: Page
  Operation: Create
  Parent Type: Database ID
  Database: {회의록 데이터베이스 ID}
  Title: {{ $json.pageTitle }}
  Properties:
    날짜 (date): {{ $json.meetingDate }}
    다음 미팅 (rich_text): {{ $json.nextMeeting ?? '미정' }}
  Children: {{ $json.children }}
```

---

### 예제 2: 프로젝트 마감일 모니터링 (D-3 이하 항목 자동 알림)

Notion 프로젝트 DB에서 마감일이 3일 이내로 남은 항목을 매일 아침 자동으로 확인하고 담당자에게 슬랙 DM과 채널 메시지를 동시에 발송합니다.

워크플로우 구조:

```
[Schedule: 매일 오전 9시 (0 9 * * 1-5)]
    ↓
[Notion: Get All - 프로젝트 DB 조회]
  Filter: Status != "완료", Due Date 존재
    ↓
[Code: D-day 계산 + 위험 항목 필터링]
  D-3 이하이면서 완료되지 않은 항목만 추출
    ↓
[IF: 위험 항목 있음?]
  ├── True  →  [Split In Batches: 항목별 처리]
  │              ↓
  │           [Slack: 담당자 DM 발송]
  │              ↓
  │           [Slack: #project 채널에 종합 현황 발송]
  └── False  →  [종료]
```

Code 노드 - D-day 계산:

```javascript
const today = new Date();
today.setHours(0, 0, 0, 0);

const urgentItems = [];

for (const item of $input.all()) {
  const props = item.json.properties;

  // Notion date 프로퍼티 파싱
  const dueDateStr = props['마감일']?.date?.start;
  if (!dueDateStr) continue;

  const dueDate = new Date(dueDateStr);
  dueDate.setHours(0, 0, 0, 0);

  const diffMs = dueDate - today;
  const diffDays = Math.ceil(diffMs / (1000 * 60 * 60 * 24));

  // D-3 이하이고 완료 안 된 항목
  if (diffDays <= 3) {
    const status = props['상태']?.select?.name ?? '미지정';
    const assignee = props['담당자']?.people?.[0]?.name ?? '미지정';
    const assigneeSlackId = props['SlackID']?.rich_text?.[0]?.text?.content ?? null;
    const projectName = props['이름']?.title?.[0]?.text?.content ?? '(제목 없음)';
    const notionUrl = item.json.url;

    let urgencyLabel;
    if (diffDays < 0) urgencyLabel = `D+${Math.abs(diffDays)} (기한 초과!)`;
    else if (diffDays === 0) urgencyLabel = 'D-Day';
    else urgencyLabel = `D-${diffDays}`;

    urgentItems.push({
      projectName,
      dueDate: dueDateStr,
      diffDays,
      urgencyLabel,
      status,
      assignee,
      assigneeSlackId,
      notionUrl
    });
  }
}

// 마감 급한 순 정렬
urgentItems.sort((a, b) => a.diffDays - b.diffDays);

return urgentItems.map(item => ({ json: item }));
```

Slack 담당자 DM 노드:

```
[Slack: Send Message]
  Channel: {{ $json.assigneeSlackId ? '@' + $json.assigneeSlackId : '#project-alert' }}
  Message: |
    마감 임박 알림이 도착했습니다.

    프로젝트: {{ $json.projectName }}
    마감일: {{ $json.dueDate }} ({{ $json.urgencyLabel }})
    현재 상태: {{ $json.status }}
    Notion 링크: {{ $json.notionUrl }}

    오늘 안에 진행 상황을 업데이트해 주세요.
```

Slack 채널 종합 현황 노드 (루프 밖에서 한 번만 실행):

```javascript
// Code 노드: 슬랙 채널용 종합 메시지 생성
const items = $input.all();

const lines = items.map(item => {
  const { projectName, urgencyLabel, assignee, status } = item.json;
  return `• ${urgencyLabel}  ${projectName}  (담당: ${assignee} / 상태: ${status})`;
});

const message = [
  `오늘 날짜 기준 마감 임박 프로젝트 현황 (${items.length}건)`,
  '',
  ...lines,
  '',
  '각 담당자는 오늘 중으로 Notion 상태를 업데이트해 주세요.'
].join('\n');

return [{ json: { message } }];
```

---

### 예제 3: GitHub 이슈 → Notion 작업 DB 자동 동기화

GitHub에서 이슈가 생성되거나 상태가 바뀌면 Notion 작업 데이터베이스에 자동으로 반영됩니다. 개발팀은 GitHub, 기획팀은 Notion에서 같은 데이터를 보게 됩니다.

워크플로우 구조:

```
[Webhook: GitHub Webhook]
  Secret: {GITHUB_WEBHOOK_SECRET}
  Events: issues (opened, closed, labeled, assigned)
    ↓
[Code: 서명 검증 + 이벤트 분류]
    ↓
[Switch: action 분기]
  ├── opened   → [Notion: Create Page]
  ├── closed   → [Notion: Update Page - 상태를 완료로]
  ├── labeled  → [Notion: Update Page - 태그 추가]
  └── assigned → [Notion: Update Page - 담당자 업데이트]
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
  throw new Error('GitHub Webhook 서명 검증 실패');
}

const payload = JSON.parse(body);
const action = payload.action;
const issue = payload.issue;

return [{
  json: {
    action,
    issueNumber: issue.number,
    issueTitle: issue.title,
    issueBody: issue.body ?? '',
    issueUrl: issue.html_url,
    issueState: issue.state,
    labels: issue.labels.map(l => l.name),
    assignees: issue.assignees.map(a => a.login),
    createdAt: issue.created_at,
    repoName: payload.repository.full_name
  }
}];
```

Notion Create Page 노드 (opened 분기):

```
[Notion: Create Page]
  Database: {작업 DB ID}
  Properties:
    이름 (title):
      [{ "text": { "content": "[#{{ $json.issueNumber }}] {{ $json.issueTitle }}" } }]
    상태 (select):
      { "name": "진행중" }
    레포지토리 (select):
      { "name": "{{ $json.repoName }}" }
    GitHub URL (url):
      "{{ $json.issueUrl }}"
    라벨 (multi_select):
      {{ $json.labels.map(l => ({ name: l })) }}
    생성일 (date):
      { "start": "{{ $json.createdAt.slice(0, 10) }}" }
    GitHub 이슈 번호 (number):
      {{ $json.issueNumber }}
```

Notion Update Page 노드 (closed 분기):

```javascript
// Code 노드: 이슈 번호로 Notion 페이지 ID 조회 후 업데이트
// 먼저 [Notion: Get All] 노드로 "GitHub 이슈 번호" = issueNumber 인 페이지 검색

const pages = $input.all();
if (pages.length === 0) {
  return [{ json: { skip: true } }];
}

const pageId = pages[0].json.id;

return [{
  json: {
    pageId,
    issueNumber: $('Code: 서명 검증').item.json.issueNumber
  }
}];
```

```
[Notion: Update Page]
  Page ID: {{ $json.pageId }}
  Properties:
    상태 (select): { "name": "완료" }
    완료일 (date): { "start": "{{ $now.toFormat('yyyy-MM-dd') }}" }
```

---

## 핵심 요약

- Notion Integration 생성 후 각 데이터베이스에 연결 권한 부여
- Database ID로 특정 데이터베이스를 타깃
- Create/Read/Update/Archive 4가지 주요 작업
- 프로퍼티 타입에 맞는 데이터 형식 사용 필수
- 이메일, Slack, Webhook 등 다양한 소스와 연결 가능

**다음 레슨**: 소셜 미디어 자동 포스팅 시스템을 구현합니다.
