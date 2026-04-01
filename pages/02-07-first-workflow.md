# 레슨 2.7: 첫 번째 워크플로우 만들기

---

## 목표: 5분 안에 작동하는 워크플로우 완성

이번 레슨에서는 실제로 작동하는 첫 번째 워크플로우를 만들어 봅니다.

**만들 것**: 버튼을 누르면 현재 시간과 환영 메시지가 담긴 데이터를 생성하는 워크플로우

(이 예제는 외부 서비스 연동 없이 n8n 자체 기능만 사용하므로 누구든 즉시 따라할 수 있습니다.)

---

## 프로젝트: "나만의 환영 메시지 생성기"

### 완성된 워크플로우 구조

```
[Manual Trigger] → [Set Fields] → [Code Node]
```

---

## 단계별 가이드

### STEP 1: 새 워크플로우 만들기

1. n8n 접속 후 **"+ New Workflow"** 클릭
2. 상단 이름 클릭 → **"나의 첫 번째 워크플로우"**로 변경
3. `Ctrl + S`로 저장

---

### STEP 2: Manual Trigger 추가

모든 워크플로우는 트리거로 시작합니다.

1. 캔버스 중앙에 **"Add first step"** 버튼 클릭
2. 검색창에 **"manual"** 입력
3. **"Manual Trigger"** 선택

**Manual Trigger**는 버튼을 클릭하면 워크플로우를 시작하는 트리거입니다. 테스트와 학습에 매우 유용합니다.

```
[⚡ Manual Trigger]
```

---

### STEP 3: Edit Fields(Set) 노드 추가

데이터를 만들고 변환하는 핵심 노드입니다.

1. Manual Trigger의 **"+"** 버튼 클릭
2. 검색창에 **"edit fields"** 입력
3. **"Edit Fields (Set)"** 선택

#### 노드 설정

**Mode**: Manual Mapping 선택

**필드 추가** ("+Add field" 클릭 3회):

| 필드명 | 타입 | 값 |
|--------|------|-----|
| `name` | String | `홍길동` |
| `greeting` | String | `안녕하세요!` |
| `timestamp` | Expression | `{{ $now }}` |

설정 완료 후 패널 닫기.

```
[⚡ Manual Trigger] → [✏️ Edit Fields]
```

---

### STEP 4: Code 노드 추가

JavaScript로 데이터를 가공합니다.

1. Edit Fields의 **"+"** 클릭
2. **"Code"** 검색 후 **"Code"** 노드 선택

#### 코드 입력

```javascript
// 이전 노드에서 받은 데이터 가져오기
const name = $input.item.json.name;
const greeting = $input.item.json.greeting;
const timestamp = $input.item.json.timestamp;

// 날짜 포맷 변환
const date = new Date(timestamp);
const formattedDate = `${date.getFullYear()}년 ${date.getMonth()+1}월 ${date.getDate()}일`;
const formattedTime = `${date.getHours()}시 ${date.getMinutes()}분`;

// 결과 반환
return [{
  json: {
    fullMessage: `${greeting} ${name}님, 오늘은 ${formattedDate} ${formattedTime}입니다.`,
    name: name,
    date: formattedDate,
    time: formattedTime
  }
}];
```

```
[⚡ Manual Trigger] → [✏️ Edit Fields] → [💛 Code]
```

---

### STEP 5: 워크플로우 실행하기

1. 상단의 **"Execute Workflow"** 버튼 클릭 (또는 `Ctrl + Enter`)
2. Manual Trigger 노드에 **"Execute step"** 버튼이 나타남
3. 클릭!

---

### STEP 6: 결과 확인

Code 노드를 클릭하면 Output 탭에서 결과를 볼 수 있습니다:

```json
{
  "fullMessage": "안녕하세요! 홍길동님, 오늘은 2026년 4월 1일 15시 30분입니다.",
  "name": "홍길동",
  "date": "2026년 4월 1일",
  "time": "15시 30분"
}
```

🎉 **축하합니다! 첫 번째 워크플로우가 완성되었습니다!**

---

## 심화: Webhook으로 업그레이드

이 워크플로우를 HTTP 요청으로 실행할 수 있도록 업그레이드해 봅시다.

### Manual Trigger를 Webhook으로 교체

1. Manual Trigger 우클릭 → Delete
2. 캔버스 빈 곳 더블클릭 → "Webhook" 검색
3. Webhook 노드 선택 및 추가

#### Webhook 노드 설정

- **HTTP Method**: GET
- **Path**: `greeting` (URL 경로)
- **Response Mode**: Immediately

4. Webhook 노드와 Edit Fields 연결
5. 워크플로우 저장 후 **활성화(토글 스위치 ON)**

#### 테스트 URL 복사

Webhook 노드를 클릭하면 테스트 URL이 표시됩니다:
```
http://localhost:5678/webhook-test/greeting
```

브라우저 주소창에 이 URL을 붙여넣으면 워크플로우가 실행되고 JSON 결과가 반환됩니다!

---

## 워크플로우 저장 및 공유

### 내보내기 (Export)
1. 편집기 우측 상단 **"..."** 메뉴
2. **"Download"** 클릭
3. JSON 파일로 저장됨

### 가져오기 (Import)
1. 대시보드에서 **"Import from file"**
2. 또는 편집기에서 **"..."** → **"Import"**

### 워크플로우 복제
1. 워크플로우 목록에서 우클릭
2. **"Duplicate"** 선택

---

## 다음 단계로

이제 여러분은 n8n의 기본 워크플로우를 만들 수 있게 되었습니다!

다음으로 배울 것들:
- 더 다양한 노드 사용법
- 실제 서비스(Gmail, Slack 등)와 연결하기
- 조건 분기와 루프 처리
- 에러 핸들링

---

## 핵심 요약

- 첫 워크플로우: Manual Trigger → Edit Fields → Code 노드 3개 연결
- 트리거: 워크플로우의 시작점
- Edit Fields: 데이터 생성 및 변환
- Code 노드: JavaScript로 자유로운 데이터 처리
- Webhook으로 교체하면 HTTP 요청으로 외부에서 실행 가능
- `Ctrl + Enter`로 빠른 실행, 결과는 노드 클릭으로 확인

**다음 레슨**: 2장 핵심 내용을 정리합니다.
