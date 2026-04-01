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

## 핵심 요약

- Notion Integration 생성 후 각 데이터베이스에 연결 권한 부여
- Database ID로 특정 데이터베이스를 타깃
- Create/Read/Update/Archive 4가지 주요 작업
- 프로퍼티 타입에 맞는 데이터 형식 사용 필수
- 이메일, Slack, Webhook 등 다양한 소스와 연결 가능

**다음 레슨**: 소셜 미디어 자동 포스팅 시스템을 구현합니다.
