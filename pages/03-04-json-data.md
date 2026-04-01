# 레슨 3.4: JSON과 데이터 구조 이해

---

## JSON은 왜 알아야 하나요?

n8n에서 노드와 노드 사이를 흐르는 **모든 데이터는 JSON 형식**입니다.

JSON을 이해하지 못하면:
- 어떤 데이터가 흐르는지 모름
- 표현식을 올바르게 작성할 수 없음
- 에러가 나도 원인을 찾기 어려움

JSON을 이해하면:
- 데이터 패널을 완전히 읽을 수 있음
- 원하는 데이터를 자유롭게 꺼낼 수 있음
- 복잡한 워크플로우도 자신 있게 만들 수 있음

---

## JSON 기초

**JSON(JavaScript Object Notation)**은 데이터를 표현하는 표준 형식입니다.

### 기본 규칙

1. **중괄호 `{}`** 안에 데이터 작성
2. **키(Key)**: 큰따옴표로 감싼 문자열
3. **값(Value)**: 문자열, 숫자, 불린, 배열, 객체, null
4. **키와 값은 콜론 `:`으로 구분**
5. **여러 항목은 쉼표 `,`로 구분**

### 가장 간단한 JSON

```json
{
  "name": "홍길동"
}
```

### 실제 예시

```json
{
  "id": 42,
  "name": "홍길동",
  "email": "hong@example.com",
  "is_active": true,
  "age": 30,
  "score": 98.5
}
```

---

## JSON의 6가지 데이터 타입

| 타입 | 예시 | 설명 |
|------|------|------|
| **문자열** | `"홍길동"` | 큰따옴표로 감쌈 |
| **숫자** | `42`, `3.14` | 따옴표 없음 |
| **불린** | `true`, `false` | 참/거짓 |
| **null** | `null` | 빈 값 |
| **배열** | `["사과", "배"]` | 대괄호 |
| **객체** | `{"name": "홍길동"}` | 중괄호 |

---

## 중첩 객체 (Nested Objects)

JSON 안에 JSON을 담을 수 있습니다.

```json
{
  "id": 1,
  "name": "홍길동",
  "address": {
    "city": "서울",
    "district": "강남구",
    "zipcode": "06000"
  },
  "contacts": {
    "email": "hong@example.com",
    "phone": "010-1234-5678"
  }
}
```

`address`와 `contacts`가 중첩 객체입니다.

n8n에서 접근하는 방법:
- `$json.address.city` → `"서울"`
- `$json.contacts.email` → `"hong@example.com"`

---

## 배열 (Arrays)

여러 데이터를 순서대로 담는 구조입니다.

```json
{
  "users": [
    { "name": "홍길동", "role": "admin" },
    { "name": "김철수", "role": "user" },
    { "name": "이영희", "role": "user" }
  ],
  "tags": ["긴급", "VIP", "결제완료"]
}
```

n8n에서 배열 접근:
- `$json.tags[0]` → `"긴급"` (0번 인덱스)
- `$json.users[1].name` → `"김철수"`
- `$json.tags.length` → `3` (배열 길이)

---

## n8n에서 데이터가 흐르는 방식

n8n의 모든 노드는 **아이템 배열**로 데이터를 주고받습니다.

```json
[
  {
    "json": {
      "name": "홍길동",
      "email": "hong@test.com"
    }
  },
  {
    "json": {
      "name": "김철수",
      "email": "kim@test.com"
    }
  }
]
```

**구조 이해**:
- 최외부: 배열 `[]`
- 각 원소: 아이템 객체 `{}`
- 아이템 안의 `json`: 실제 데이터

> 표현식에서 `$json`을 사용하면 현재 아이템의 `json` 부분을 의미합니다.

---

## 데이터 패널 읽기

워크플로우 실행 후 노드를 클릭하면 데이터 패널이 나타납니다.

### JSON 보기 모드
```
{
  "name": "홍길동",
  "email": "hong@test.com",
  "amount": 75000
}
```

### 테이블 보기 모드

| name | email | amount |
|------|-------|--------|
| 홍길동 | hong@test.com | 75000 |

### 스키마 보기 모드
데이터 구조를 트리 형태로 확인:
```
▼ { }
  ├── name: "홍길동"
  ├── email: "hong@test.com"
  └── amount: 75000
```

---

## 실습: 데이터 구조 분석

다음 JSON을 보고 각 접근법을 작성해 보세요:

```json
{
  "order_id": "ORD-001",
  "customer": {
    "name": "홍길동",
    "email": "hong@test.com",
    "tier": "VIP"
  },
  "items": [
    { "product": "키보드", "price": 150000, "qty": 1 },
    { "product": "마우스", "price": 80000, "qty": 2 }
  ],
  "total": 310000,
  "paid": true
}
```

**문제**: 다음을 표현식으로 어떻게 접근하나요?
1. 주문 ID
2. 고객 이름
3. 고객 등급
4. 첫 번째 상품명
5. 두 번째 상품 수량
6. 총 결제 금액

**정답**:
1. `$json.order_id` → `"ORD-001"`
2. `$json.customer.name` → `"홍길동"`
3. `$json.customer.tier` → `"VIP"`
4. `$json.items[0].product` → `"키보드"`
5. `$json.items[1].qty` → `2`
6. `$json.total` → `310000`

---

## 자주 사용하는 JSON 조작 패턴

### 값이 있는지 확인
```javascript
$json.email !== null && $json.email !== undefined
```

### 빈 배열 확인
```javascript
$json.items.length > 0
```

### 배열에서 특정 항목 찾기 (Code 노드에서)
```javascript
const vipItem = $json.items.find(item => item.product === '키보드');
```

### 배열 값 합계 계산 (Code 노드에서)
```javascript
const total = $json.items.reduce((sum, item) => sum + item.price * item.qty, 0);
```

---

## 핵심 요약

- JSON은 n8n에서 데이터가 흐르는 표준 형식
- 기본 타입: 문자열, 숫자, 불린, null, 배열, 객체
- 중첩 객체: `.`으로 접근 (`$json.address.city`)
- 배열: `[인덱스]`로 접근 (`$json.items[0]`)
- n8n 데이터는 아이템 배열 형태이며, `$json`이 현재 아이템의 데이터
- 데이터 패널의 JSON/테이블/스키마 보기로 구조 확인 가능

**다음 레슨**: JSON 데이터를 동적으로 활용하는 표현식을 마스터합니다.
