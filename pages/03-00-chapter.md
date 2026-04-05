# 3장: n8n 핵심 개념과 노드

> 노드를 이해하면 워크플로우가 보입니다.

---

## 이 챕터에서 배울 내용

```
레슨 3.1: 노드(Node) 완전 이해
레슨 3.2: 트리거 노드 종류와 활용
레슨 3.3: 핵심 노드 탐색
레슨 3.4: JSON과 데이터 구조 이해
레슨 3.5: 표현식(Expressions) 마스터
레슨 3.6: Code 노드로 코드 활용
레슨 3.7: HTTP Request 노드로 API 호출
레슨 3.8: 서드파티 서비스 연동
```

---

## 왜 3장이 중요한가?

2장에서 n8n을 설치하고 첫 워크플로우를 만들었다면, 3장은 n8n을 **정말로 이해**하게 되는 챕터입니다.

노드, JSON, 표현식이라는 세 가지 핵심을 마스터하면:
- 어떤 워크플로우든 이해하고 만들 수 있습니다
- 다른 사람의 템플릿을 내 것으로 변형할 수 있습니다
- 에러가 나도 어디서 문제가 생겼는지 스스로 찾을 수 있습니다

가장 중요한 챕터이니 천천히, 꼭 직접 실습하면서 읽어 주세요.

---

## 3장의 전체 흐름

3장은 작은 것부터 큰 것 순서로 배웁니다:

```
노드 하나 이해
    ↓
노드들의 역할 분류 (트리거 / 처리 / 출력)
    ↓
노드 사이를 흐르는 데이터 (JSON) 이해
    ↓
데이터를 동적으로 참조하는 방법 (표현식)
    ↓
코드로 데이터를 직접 조작 (Code 노드)
    ↓
외부 서비스 연결 (HTTP Request, 통합)
```

---

## 3장을 읽기 전 알아야 할 것

### n8n에서 데이터가 흐르는 방식

모든 노드는 데이터를 받아서, 처리하고, 내보냅니다.

```
[노드 A] → 데이터 → [노드 B] → 데이터 → [노드 C]
```

이때 데이터는 항상 **JSON 배열** 형태입니다:

```json
[
  { "json": { "name": "홍길동", "email": "hong@example.com" } },
  { "json": { "name": "김철수", "email": "kim@example.com" } }
]
```

- 배열의 각 요소 = 하나의 아이템(Item)
- 각 아이템은 `json` 키 아래에 실제 데이터가 있음
- 노드는 이 배열을 받아서 처리하고, 새로운 배열을 내보냄

### 표현식이란?

표현식은 n8n에서 이전 노드의 데이터를 현재 노드에서 참조하는 방법입니다:

```
{{ $json.name }}  →  현재 아이템의 name 필드
```

이 개념만 미리 이해해도 3장 전체가 훨씬 쉬워집니다.

---

## 레슨별 핵심 미리보기

### 3.1 노드 완전 이해
노드는 레고 블록입니다. 블록의 역할을 이해하면 어떻게 연결할지 보입니다.

```
입력 핀(Input) → [노드] → 출력 핀(Output)
```

노드마다 설정할 수 있는 값이 있고, 표현식으로 이전 노드 데이터를 가져올 수 있습니다.

### 3.2 트리거 노드
워크플로우의 시작점. 언제 실행할지 정합니다.

```
Schedule Trigger  → 정해진 시간에 자동 실행
Webhook Trigger   → 외부에서 HTTP 요청이 올 때 실행
Gmail Trigger     → 새 이메일이 올 때 실행
Manual Trigger    → 버튼 클릭으로 수동 실행 (테스트용)
```

### 3.3 핵심 처리 노드
데이터를 변환하고, 필터링하고, 분기하는 노드들.

| 노드 | 하는 일 |
|------|---------|
| IF | 조건에 따라 두 갈래로 분기 |
| Switch | 여러 경우 중 하나로 분기 |
| Edit Fields | 필드 추가/수정/삭제 |
| Filter | 조건에 맞는 아이템만 통과 |
| Merge | 두 경로의 데이터를 합침 |
| Split Out | 배열을 개별 아이템으로 분리 |
| Code | JavaScript로 자유롭게 처리 |

### 3.4 JSON 구조 이해
n8n을 쓰면서 가장 많이 보게 되는 형태:

```json
{
  "id": 123,
  "user": {
    "name": "홍길동",
    "tags": ["VIP", "구독자"]
  },
  "amount": 59000
}
```

접근법:
```javascript
$json.id            // 123
$json.user.name     // "홍길동"
$json.user.tags[0]  // "VIP"
$json.amount        // 59000
```

### 3.5 표현식 마스터
노드 설정 필드에서 `{{ }}` 안에 JavaScript 표현식을 쓸 수 있습니다:

```javascript
{{ $json.name }}                         // 데이터 참조
{{ $json.price * 1.1 }}                  // 계산
{{ $json.name.toUpperCase() }}           // 문자열 변환
{{ $now.toFormat('yyyy-MM-dd') }}        // 날짜 포맷
{{ $json.score > 90 ? "우수" : "보통" }} // 조건 표현
```

### 3.6 Code 노드
표현식으로 처리하기 어려운 복잡한 로직은 Code 노드에서 JavaScript로 직접 작성합니다.

```javascript
// 입력 데이터 가공 예시
const items = $input.all();
const result = items.map(item => {
  return {
    json: {
      fullName: `${item.json.firstName} ${item.json.lastName}`,
      upperEmail: item.json.email.toUpperCase(),
      isVip: item.json.purchaseCount > 10
    }
  };
});
return result;
```

### 3.7 HTTP Request 노드
n8n에 통합이 없는 서비스도 HTTP Request 노드 하나로 연결할 수 있습니다.

```
GET  https://api.example.com/users        → 데이터 조회
POST https://api.example.com/users        → 데이터 생성
PUT  https://api.example.com/users/{{ $json.id }} → 데이터 수정
```

### 3.8 서드파티 서비스 연동
Google, Slack, Notion 같은 서비스들은 전용 노드가 있어 훨씬 편리하게 연결됩니다.
크리덴셜(Credentials)에 API 키나 OAuth를 한 번만 설정하면 재사용 가능합니다.

---

## 3장 시작 전 준비

- n8n 실행 중인 환경
- 크롬 개발자 도구 (JSON 확인용, 선택사항)
- 연습용 빈 워크플로우 하나 열어두기

가장 중요한 챕터입니다. 각 레슨을 읽으면서 반드시 직접 노드를 추가해보고 실행해보세요.
