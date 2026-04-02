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

### 1단계: Google Cloud Console 설정

**프로젝트 생성 및 API 활성화**

1. [Google Cloud Console](https://console.cloud.google.com) 접속
2. 상단 프로젝트 드롭다운 → "새 프로젝트" 클릭
3. 프로젝트 이름 입력 (예: `n8n-automation`) → 만들기
4. 왼쪽 메뉴 → "API 및 서비스" → "라이브러리"
5. 검색창에 `Google Sheets API` 입력 → 클릭 → "사용 설정"
6. 동일하게 `Google Drive API`도 사용 설정 (파일 접근에 필요)

**OAuth 2.0 클라이언트 생성**

1. "API 및 서비스" → "사용자 인증 정보" 클릭
2. "+ 사용자 인증 정보 만들기" → "OAuth 클라이언트 ID"
3. 애플리케이션 유형: "웹 애플리케이션" 선택
4. 이름: `n8n` 입력
5. "승인된 리디렉션 URI" → n8n 콜백 주소 추가:
   ```
   https://your-n8n-domain.com/rest/oauth2-credential/callback
   ```
   (로컬 환경: `http://localhost:5678/rest/oauth2-credential/callback`)
6. 만들기 → 클라이언트 ID와 보안 비밀 복사

### 2단계: n8n 크리덴셜 등록

1. n8n → "Credentials" → "+ Add Credential"
2. "Google Sheets OAuth2 API" 검색 후 선택
3. Client ID, Client Secret 붙여넣기
4. "Connect my account" 클릭 → Google 계정 로그인 및 권한 허용
5. 연결 성공 확인 후 저장

### Spreadsheet ID 확인 방법

Google Sheets URL에서 ID 추출:
```
https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit#gid=0
```

예시:
```
URL: https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit
ID:  1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms
```

---

## Google Sheets Trigger: 시트 변경 자동 감지

n8n의 Google Sheets Trigger 노드를 사용하면 시트 변경 시 자동으로 워크플로우를 실행할 수 있습니다.

```
[Google Sheets Trigger]
설정:
  Event: Row Added       ← 새 행 추가 시
       | Row Updated     ← 행 수정 시
       | Row Added or Updated
  Spreadsheet: {ID}
  Sheet: 주문관리
  Polling Interval: 1 minute  ← 변경 감지 주기
```

활용 예시:
```
[Google Sheets Trigger: 새 행 추가]
    ↓
[Switch: 주문유형 분기]
  case "온라인": → [Slack 알림]
  case "전화":   → [이메일 알림]
```

> Trigger 노드는 폴링 방식으로 작동합니다. 실시간이 아닌 설정한 주기(최소 1분)마다 변경을 확인합니다. 더 빠른 반응이 필요하면 Google Apps Script + Webhook 방식을 사용하세요 (아래 고급 기법 참고).

---

## 주요 작업 유형

### 1. 데이터 읽기 (Read Rows)

```
[Google Sheets: Read Rows]
설정:
  Operation: Read Rows
  Spreadsheet: {Spreadsheet ID}
  Sheet: Sheet1
  Options:
    Return All: true
```

필터 조건으로 특정 행만 읽기:
```
[Google Sheets: Read Rows]
설정:
  Operation: Read Rows
  Filters:
    - Column: status
      Condition: equals
      Value: pending
    - Column: priority
      Condition: equals
      Value: high
  Combine Filters: AND
```

범위 지정으로 읽기:
```
Options:
  Range: A2:E100      ← A2부터 E100까지만 읽음
  Key Row: 1          ← 1번 행을 헤더로 사용
```

### 2. 데이터 추가 (Append Row)

새 행을 추가합니다 (기존 데이터 유지):

```
[Google Sheets: Append Row]
설정:
  Operation: Append Row
  Spreadsheet: {ID}
  Sheet: Sheet1
  Columns:
    name:       {{ $json.name }}
    email:      {{ $json.email }}
    date:       {{ $now.toFormat('yyyy-MM-dd') }}
    status:     "신규"
    source:     {{ $json.source ?? "직접입력" }}
```

### 3. 데이터 업데이트 (Update Row)

특정 행의 값을 변경합니다:

```
[Google Sheets: Update Row]
설정:
  Operation: Update Row
  Lookup Column: email          ← 이 컬럼 값으로 행 찾기
  Lookup Value: {{ $json.email }}
  Columns to Update:
    status:     "처리완료"
    updated_at: {{ $now.toISO() }}
    memo:       {{ $json.memo }}
```

행 번호로 직접 업데이트:
```
[Google Sheets: Update Row]
설정:
  Operation: Update Row
  Row Number: {{ $json.rowNumber }}   ← Read Rows로 가져온 행 번호 사용
  Columns to Update:
    status: "완료"
```

### 4. 데이터 삭제 (Delete Row)

```
[Google Sheets: Delete Row]
설정:
  Operation: Delete Row
  Row Number: {{ $json.rowNumber }}
```

특정 조건의 행 삭제 (Read → Delete 조합):
```
[Google Sheets: Read Rows]
  Filter: status = "삭제예정"
    ↓
[Google Sheets: Delete Row]
  Row Number: {{ $json.row_number }}
```

### 5. 시트 지우기 (Clear)

```
[Google Sheets: Clear]
설정:
  Operation: Clear
  Spreadsheet: {ID}
  Sheet: 임시데이터
  Range: A2:Z1000   ← 헤더(1행)는 유지하고 데이터만 삭제
```

---

## 프로젝트 1: 폼 응답 → Sheets 자동 저장

Google Forms나 Typeform 제출 내용을 Sheets에 자동 기록:

```
[Webhook: POST /form-submit]
    ↓
[Code: 데이터 정제]
  const data = $input.first().json;
  return {
    제출일시: $now.toFormat('yyyy-MM-dd HH:mm'),
    이름: data.name?.trim(),
    이메일: data.email?.toLowerCase(),
    문의내용: data.inquiry,
    처리상태: "접수",
    담당자: "",
    완료일: ""
  };
    ↓
[Google Sheets: Append Row]
  Columns:
    제출일시:   {{ $json.제출일시 }}
    이름:       {{ $json.이름 }}
    이메일:     {{ $json.이메일 }}
    문의내용:   {{ $json.문의내용 }}
    처리상태:   {{ $json.처리상태 }}
    담당자:     {{ $json.담당자 }}
    완료일:     {{ $json.완료일 }}
    ↓
[Gmail: 접수 확인 메일]
  To: {{ $json.이메일 }}
  Subject: 문의 접수 완료
  Body: {{ $json.이름 }}님, 문의가 접수되었습니다.
```

---

## 프로젝트 2: Sheets 데이터 → 이메일 일괄 발송

Sheets에 있는 이메일 목록으로 개인화된 이메일 발송:

```
[Schedule: 매주 월요일 오전 9시]
  Cron: 0 9 * * 1
    ↓
[Google Sheets: Read Rows]
  Sheet: 구독자
  Filter:
    - Column: newsletter_subscriber
      Value: TRUE
    - Column: active
      Value: TRUE
    ↓
[SplitInBatches: 배치 크기 10]
  (Gmail API 속도 제한 방지)
    ↓
[Gmail: 개인화된 뉴스레터 발송]
  To:      {{ $json.email }}
  Subject: {{ $json.name }}님의 주간 업데이트
  Body:    안녕하세요 {{ $json.name }}님, 이번 주 업데이트입니다...
    ↓
[Google Sheets: Update Row]
  Lookup Column: email
  Lookup Value:  {{ $json.email }}
  Columns to Update:
    last_sent:    {{ $now.toISO() }}
    send_count:   {{ ($json.send_count ?? 0) + 1 }}
```

---

## 프로젝트 3: 자동 대시보드 업데이트

여러 소스의 데이터를 Sheets에 모아 대시보드 자동 업데이트:

```
[Schedule: 매일 오전 6시]
  Cron: 0 6 * * *
    ↓
병렬 실행:
  [HTTP Request: 어제 매출 조회]   ─────┐
  [HTTP Request: 신규 가입자 조회] ─────┤ [Merge: Merge by Position]
  [HTTP Request: 처리 대기 건수]   ─────┘
    ↓
[Code: 통계 계산]
  const items = $input.all();
  return {
    date:     $now.minus({days: 1}).toFormat('yyyy-MM-dd'),
    revenue:  items[0].json.total,
    newUsers: items[1].json.count,
    pending:  items[2].json.pending,
    rate:     (items[2].json.completed / items[2].json.total * 100).toFixed(1) + '%'
  };
    ↓
[Google Sheets: Append Row]
  Sheet: 일별통계
  Columns:
    날짜:       {{ $json.date }}
    매출:       {{ $json.revenue }}
    신규가입:   {{ $json.newUsers }}
    대기건수:   {{ $json.pending }}
    처리율:     {{ $json.rate }}
```

---

## 고급 기법: Apps Script로 실시간 변경 감지

n8n Trigger의 폴링 방식 대신, Google Apps Script를 사용하면 즉시 반응할 수 있습니다.

**Google Apps Script 설정 방법:**

1. Google Sheets → "확장 프로그램" → "Apps Script"
2. 아래 코드 붙여넣기
3. "트리거" 메뉴 → "+ 트리거 추가"
4. 함수: `onEdit`, 이벤트 유형: "스프레드시트에서" → "수정 시"

```javascript
// Google Apps Script
function onEdit(e) {
  const sheet = e.source.getActiveSheet();
  const range = e.range;

  // '주문' 시트의 5번 열(상태) 변경 시만 실행
  if (sheet.getName() === '주문' && range.getColumn() === 5) {
    const webhookUrl = 'https://your-n8n.com/webhook/sheets-update';

    UrlFetchApp.fetch(webhookUrl, {
      method: 'POST',
      contentType: 'application/json',
      payload: JSON.stringify({
        row:       range.getRow(),
        newValue:  e.value,
        oldValue:  e.oldValue,
        sheetName: sheet.getName(),
        timestamp: new Date().toISOString()
      })
    });
  }
}
```

n8n에서 수신:
```
[Webhook: POST /sheets-update]
    ↓
[Switch: newValue 분기]
  case "배송완료": → [카카오 알림톡 발송]
  case "환불요청": → [Slack #환불팀 알림]
  case "재고없음": → [구매팀 이메일 발송]
```

---

## 에러 처리

### 행을 찾지 못한 경우

Update Row 시 해당 이메일이 없으면 에러가 발생합니다. Read로 먼저 확인하세요:

```
[Google Sheets: Read Rows]
  Filter: email = {{ $json.email }}
    ↓
[IF: 결과 있는지 확인]
  Condition: {{ $input.all().length }} > 0
  True:  → [Google Sheets: Update Row]
  False: → [Google Sheets: Append Row]  ← 없으면 새로 추가
```

### API 한도 초과 대응

Google Sheets API는 분당 요청 수 제한이 있습니다:

```
[SplitInBatches]
  Batch Size: 50          ← 한 번에 50개씩 처리
    ↓
[Google Sheets: Append Row]
    ↓
[Wait: 1초 대기]          ← API 과부하 방지
```

### 크리덴셜 오류 처리

```
[Google Sheets: Read Rows]
    ↓ (에러 발생 시)
[Error Trigger]
    ↓
[Slack: 관리자 알림]
  text: "Google Sheets 연동 오류: {{ $json.error.message }}"
```

---

## Sheets 노드 활용 팁

### 헤더 행 처리

Sheets의 첫 행을 헤더로 사용하면 컬럼 이름으로 데이터 접근 가능:
```javascript
$json.name      // "name" 컬럼
$json["이메일"] // 한글 컬럼명은 대괄호 표기
$json.status    // "status" 컬럼
```

### 날짜 형식 통일

Sheets에 날짜를 저장할 때 일관된 형식 사용:
```javascript
// 날짜만
{{ $now.toFormat('yyyy-MM-dd') }}

// 날짜+시간
{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}

// ISO 형식 (정렬에 유리)
{{ $now.toISO() }}
```

### 대량 데이터 처리

1000행 이상을 읽을 때:
```
[Google Sheets: Read Rows]
Options:
  Return All: true        ← 모든 행 반환 (페이지네이션 자동 처리)
```

### 빈 셀 기본값 처리

```javascript
// 빈 셀이면 기본값 사용
{{ $json.memo ?? "메모 없음" }}
{{ $json.count ?? 0 }}
{{ $json.active ?? false }}
```

---

## 핵심 요약

- Google Sheets = 무료 클라우드 데이터베이스로 활용 가능
- Read / Append / Update / Delete / Clear 5가지 주요 작업
- Google Sheets Trigger: 시트 변경을 폴링 방식으로 감지
- Apps Script + Webhook: 실시간 변경 감지 (더 빠른 반응)
- SplitInBatches + Wait: API 한도 초과 방지
- 에러 처리: 행 존재 확인 후 Update vs Append 분기

다음 레슨: Notion을 연동하여 프로젝트 관리를 자동화합니다.
