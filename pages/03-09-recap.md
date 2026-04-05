# 3장 핵심 정리

---

## 3장에서 배운 것들

3장은 n8n의 심장부를 이해하는 챕터였습니다. 여기서 배운 개념들이 앞으로 모든 워크플로우의 기반이 됩니다.

---

## 핵심 개념 요약

### 노드 유형

| 유형 | 역할 | 예시 |
|------|------|------|
| 트리거 노드 | 워크플로우 시작 | Schedule, Webhook, Gmail |
| 처리 노드 | 데이터 변환/분기/반복 | IF, Filter, Code, Edit Fields |
| 출력 노드 | 외부 서비스로 전송 | Gmail Send, Slack, Notion |

### 핵심 트리거

| 트리거 | 사용 시점 | 실행 방식 |
|--------|-----------|---------|
| Manual | 테스트, 수동 | 버튼 클릭 |
| Schedule | 정기 실행 | Cron 표현식 |
| Webhook | 외부 이벤트 수신 | HTTP POST/GET |
| 서비스 트리거 | 이메일/메시지 수신 | Polling |

### 핵심 처리 노드

| 노드 | 역할 | 사용 예시 |
|------|------|---------|
| Edit Fields | 데이터 생성/변환 | 필드 추가, 이름 변경 |
| IF / Switch | 조건 분기 | 스팸 분류, 상태별 처리 |
| Merge | 데이터 병합 | 두 경로 결과 합치기 |
| Split Out | 배열 → 개별 아이템 | 리스트 하나씩 처리 |
| Filter | 조건 필터링 | 특정 조건 아이템만 통과 |
| Code | JavaScript 코드 실행 | 복잡한 데이터 변환 |
| HTTP Request | 모든 API 호출 | 외부 서비스 연동 |

---

## 데이터 흐름 구조

n8n에서 모든 데이터는 이 형태로 흐릅니다:

```json
[
  {
    "json": {
      "field1": "값1",
      "field2": 123,
      "nested": { "key": "값" },
      "array": [1, 2, 3]
    }
  },
  {
    "json": {
      "field1": "값2"
    }
  }
]
```

중요한 점:
- 항상 배열(`[ ]`) 안에 아이템들이 있습니다
- 각 아이템은 `json` 키 아래에 실제 데이터가 있습니다
- 노드는 이 배열을 받아 처리하고, 새 배열을 내보냅니다

---

## JSON 접근 문법 요약

```javascript
$json.field              // 기본 필드 접근
$json.nested.field       // 중첩 객체
$json.array[0]           // 배열 첫 번째 요소
$json.array.length       // 배열 길이
$json.field ?? "기본값"  // null/undefined일 때 기본값
$json?.nested?.field     // 옵셔널 체이닝 (에러 방지)
```

---

## 표현식 핵심 패턴

```javascript
// 현재 아이템 데이터
{{ $json.name }}

// 특정 노드의 데이터
{{ $node["노드이름"].json.field }}

// 날짜/시간
{{ $now.toFormat('yyyy-MM-dd') }}
{{ $now.toISO() }}

// 조건 표현
{{ $json.amount > 0 ? "양수" : "음수" }}

// 문자열 조작
{{ $json.name.toUpperCase() }}
{{ $json.text.replace("old", "new") }}

// 여러 필드 조합
{{ $json.firstName + " " + $json.lastName }}
```

---

## Code 노드 기본 패턴

```javascript
// 모든 아이템 처리
const items = $input.all();

const result = items.map(item => {
  return {
    json: {
      // 새로운 데이터 구조
      fullName: `${item.json.firstName} ${item.json.lastName}`,
      processedAt: new Date().toISOString(),
      isValid: item.json.email.includes('@')
    }
  };
});

return result;
```

```javascript
// 단일 아이템 처리 (Single Item 모드)
const data = $json;

return {
  json: {
    summary: data.items.length + "개 처리됨",
    total: data.items.reduce((sum, i) => sum + i.price, 0)
  }
};
```

---

## HTTP Request 핵심 설정

```
Method:  GET / POST / PUT / DELETE
URL:     https://api.example.com/endpoint
Headers: Authorization: Bearer {{ $json.token }}
Body:    JSON 형식으로 데이터 전송
```

인증 방식:
```
API Key    → Header에 직접 추가
Bearer     → Authorization: Bearer {키}
Basic Auth → Credentials에 ID/PW 저장
OAuth 2.0  → n8n 크리덴셜에서 OAuth 설정
```

---

## 3장 실력 자가 점검

아래 질문에 모두 답할 수 있으면 3장 완료입니다:

- 트리거 노드와 일반 노드의 차이는?
- Webhook과 Schedule 트리거를 각각 언제 쓰나요?
- n8n에서 데이터가 흐르는 기본 형식(JSON 구조)은?
- `$json.user.profile.avatar`에서 각 `.`의 의미는?
- Code 노드의 출력 형식(`return` 값)은 어떻게 되나요?
- HTTP Request 노드로 POST 요청을 보낼 때 Body는 어디에 설정하나요?
- IF 노드와 Switch 노드의 차이점은?

---

## 3장 실력 확인 미니 실습

아래 워크플로우를 직접 만들어보세요 (15분 소요):

```
[Manual Trigger]
    ↓
[Edit Fields]
  name: "테스트 유저"
  score: 85
  email: "test@example.com"
    ↓
[IF]
  조건: score >= 80
  True  → [Edit Fields] grade: "우수"
  False → [Edit Fields] grade: "보통"
    ↓
[Code]
  return [{
    json: {
      message: `${$json.name}님은 ${$json.grade} 등급입니다.`,
      notifyEmail: $json.email
    }
  }];
```

실행 후 Output에서 `message` 필드를 확인하세요.

---

## 다음 챕터 미리보기

4장: 워크플로우 구축과 관리

- 효과적인 워크플로우 설계 원칙
- 트리거 고급 활용법 (Cron 표현식, Webhook 보안)
- IF/Switch로 복잡한 조건 처리
- Loop로 반복 작업 처리
- 에러 핸들링과 재시도 전략
- Execution 탭을 활용한 디버깅

4장에서는 실제로 사용 가능한 워크플로우를 설계하고 만드는 능력을 키웁니다.
3장에서 배운 노드들을 조합해서 안정적으로 동작하는 완성된 자동화를 만들게 됩니다.
