# 레슨 6.4: Google Sheets 자동화

---

## Google Sheets가 강력한 이유

Google Sheets는 n8n과 함께 사용할 때 강력한 데이터 허브가 됩니다:
- 무료로 사용 가능한 클라우드 스프레드시트
- 팀이 함께 실시간으로 볼 수 있음
- API 연동이 쉬움
- 피벗 테이블, 차트 자동 업데이트 가능

---

## Google Sheets 연동 설정

### 크리덴셜 설정

1. Google Cloud Console → Google Sheets API 활성화
2. OAuth 2.0 클라이언트 생성
3. n8n → Credentials → Google Sheets OAuth2 API

### Spreadsheet ID 확인

URL에서 추출:
```
https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit
```

---

## 주요 작업 유형

### 1. 데이터 읽기 (Read)

```
[Google Sheets: Sheet Data]
설정:
  Operation: Read Rows
  Spreadsheet: {Spreadsheet ID}
  Sheet: Sheet1
  Filters:
    Column: status
    Value: pending
```

### 2. 데이터 추가 (Append)

새 행을 추가합니다 (기존 데이터 유지):

```
[Google Sheets: Append Row]
설정:
  Operation: Append Row
  Spreadsheet: {ID}
  Sheet: Sheet1
  Columns:
    name: {{ $json.name }}
    email: {{ $json.email }}
    date: {{ $now.toFormat('yyyy-MM-dd') }}
    status: "신규"
```

### 3. 데이터 업데이트 (Update)

특정 행의 값을 변경합니다:

```
[Google Sheets: Update Row]
설정:
  Operation: Update Row
  Lookup Column: email
  Lookup Value: {{ $json.email }}
  Columns to Update:
    status: "처리완료"
    updated_at: {{ $now.toISO() }}
```

### 4. 데이터 삭제 (Delete)

```
[Google Sheets: Delete Row]
설정:
  Operation: Delete Row
  Row Number: {{ $json.rowNumber }}
```

---

## 프로젝트 1: 폼 응답 → Sheets 자동 저장

Google Forms나 Typeform 제출 내용을 Sheets에 자동 기록:

```
[Webhook: 폼 제출 이벤트]
    ↓
[Code: 데이터 정제]
    ↓
[Google Sheets: Append Row]
  Columns:
    제출일시: {{ $now.toFormat('yyyy-MM-dd HH:mm') }}
    이름: {{ $json.name }}
    이메일: {{ $json.email }}
    문의내용: {{ $json.inquiry }}
    처리상태: "접수"
    담당자: ""
    완료일: ""
```

---

## 프로젝트 2: Sheets 데이터 → 이메일 일괄 발송

Sheets에 있는 이메일 목록으로 개인화된 이메일 발송:

```
[Schedule: 매주 월요일 오전 9시]
    ↓
[Google Sheets: Read Rows]
  Filter: newsletter_subscriber = TRUE
    ↓
[Loop: 각 구독자]
    ↓
[Gmail: 개인화된 뉴스레터 발송]
  To: {{ $json.email }}
  Subject: {{ $json.name }}님의 주간 업데이트
  Body: 안녕하세요 {{ $json.name }}님...
    ↓
[Google Sheets: Update Row]
  last_sent: {{ $now.toISO() }}
```

---

## 프로젝트 3: 자동 대시보드 업데이트

여러 소스의 데이터를 Sheets에 모아 대시보드 자동 업데이트:

```
[Schedule: 매일 오전 6시]
    ↓
병렬 실행:
  [API: 어제 매출 조회] ─────────►
  [API: 신규 가입자 조회] ─────────► [Merge]
  [API: 처리 대기 건수 조회] ──────►
    ↓
[Code: 통계 계산]
    ↓
[Google Sheets: 대시보드 시트 업데이트]
  A2: {{ $json.date }}
  B2: {{ $json.revenue }}
  C2: {{ $json.newUsers }}
  D2: {{ $json.pending }}
```

---

## 고급 기법: 스프레드시트 함수 트리거

n8n 없이 Sheets의 변경을 감지하는 방법:

1. Google Apps Script로 변경 감지
2. n8n Webhook 호출
3. n8n이 추가 처리

```javascript
// Google Apps Script
function onEdit(e) {
  const sheet = e.source.getActiveSheet();
  const range = e.range;

  // 특정 시트의 특정 열 변경 시
  if (sheet.getName() === '주문' && range.getColumn() === 5) {
    const webhookUrl = 'https://your-n8n.com/webhook/sheets-update';
    UrlFetchApp.fetch(webhookUrl, {
      method: 'POST',
      contentType: 'application/json',
      payload: JSON.stringify({
        row: range.getRow(),
        newValue: e.value,
        oldValue: e.oldValue,
        sheetName: sheet.getName()
      })
    });
  }
}
```

---

## Sheets 노드 활용 팁

### 헤더 행 처리

Sheets의 첫 행을 헤더로 사용하면 컬럼 이름으로 데이터 접근 가능:
```javascript
$json.name    // "이름" 컬럼
$json.email   // "이메일" 컬럼
$json.status  // "상태" 컬럼
```

### 날짜 형식 통일

Sheets에 날짜를 저장할 때 일관된 형식 사용:
```javascript
{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}
```

### 대량 데이터 처리

1000행 이상을 읽을 때는 페이지네이션 활용:
```
Options:
  Return All: true  // 모든 행 반환
```

---

## 핵심 요약

- Google Sheets = 무료 클라우드 데이터베이스로 활용 가능
- Read/Append/Update/Delete 4가지 주요 작업
- 폼 → Sheets: 자동 데이터 수집
- Sheets → 이메일: 개인화 이메일 일괄 발송
- Apps Script 트리거로 Sheets 변경을 n8n과 연결

**다음 레슨**: Notion을 연동하여 프로젝트 관리를 자동화합니다.
