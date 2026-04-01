# 레슨 3.6: Code 노드로 코드 활용

---

## Code 노드란?

Code 노드(구 Function 노드)는 n8n에서 JavaScript 코드를 직접 작성하여 데이터를 처리할 수 있는 노드입니다.

표현식으로 해결하기 어려운 복잡한 로직을 처리할 때 사용합니다.

> 꼭 코딩을 잘 알아야 하나요?
> 아닙니다. 기본 JavaScript를 모르더라도 AI(ChatGPT, Claude)에게 코드를 작성해 달라고 하면 됩니다. 이 레슨에서는 자주 쓰는 패턴을 중심으로 배웁니다.

---

## Code 노드의 2가지 모드

### 모드 1: Run Once for All Items (전체 아이템 한 번에 처리)

들어오는 모든 아이템을 한꺼번에 처리합니다.

```javascript
// 모든 아이템을 배열로 받음
const items = $input.all();

// 처리 후 결과 반환
return items.map(item => ({
  json: {
    ...item.json,
    processed: true
  }
}));
```

사용 예: 여러 아이템을 비교/정렬하거나 통계를 내야 할 때

### 모드 2: Run Once for Each Item (아이템 하나씩 처리)

아이템마다 별도로 실행됩니다. (기본 모드)

```javascript
// 현재 아이템 데이터 접근
const data = $input.item.json;

// 처리
const result = {
  ...data,
  fullName: data.firstName + ' ' + data.lastName
};

// 결과 반환
return { json: result };
```

---

## 핵심 규칙

### 출력 형식
Code 노드의 출력은 반드시 다음 형식이어야 합니다:

단일 아이템 반환:
```javascript
return { json: { key: "value" } };
```

여러 아이템 반환:
```javascript
return [
  { json: { id: 1, name: "홍길동" } },
  { json: { id: 2, name: "김철수" } }
];
```

빈 결과 (다음 노드에 아무것도 전달 안 함):
```javascript
return [];
```

---

## 자주 쓰는 코드 패턴

### 패턴 1: 날짜 포맷 변환

```javascript
const data = $input.item.json;
const date = new Date(data.timestamp);

return {
  json: {
    ...data,
    formattedDate: `${date.getFullYear()}-${String(date.getMonth()+1).padStart(2,'0')}-${String(date.getDate()).padStart(2,'0')}`,
    formattedTime: `${String(date.getHours()).padStart(2,'0')}:${String(date.getMinutes()).padStart(2,'0')}`
  }
};
```

### 패턴 2: 배열 필터링 및 변환

```javascript
const items = $input.all();

// 활성 사용자만 필터링 후 필요한 필드만 추출
const activeUsers = items
  .filter(item => item.json.is_active === true)
  .map(item => ({
    json: {
      id: item.json.id,
      name: item.json.name,
      email: item.json.email
    }
  }));

return activeUsers;
```

### 패턴 3: 집계 및 통계

```javascript
const items = $input.all();

const total = items.reduce((sum, item) => sum + item.json.amount, 0);
const count = items.length;
const average = count > 0 ? total / count : 0;
const max = Math.max(...items.map(item => item.json.amount));

return [{
  json: {
    total,
    count,
    average: Math.round(average * 100) / 100,
    max,
    summary: `${count}건, 총 ${total.toLocaleString()}원`
  }
}];
```

### 패턴 4: 데이터 그룹화

```javascript
const items = $input.all();

// 카테고리별 그룹화
const grouped = {};
items.forEach(item => {
  const category = item.json.category;
  if (!grouped[category]) {
    grouped[category] = [];
  }
  grouped[category].push(item.json);
});

// 각 카테고리를 하나의 아이템으로 반환
return Object.keys(grouped).map(category => ({
  json: {
    category,
    items: grouped[category],
    count: grouped[category].length
  }
}));
```

### 패턴 5: 문자열 처리

```javascript
const data = $input.item.json;

// 이메일 도메인 추출
const emailDomain = data.email.split('@')[1];

// 전화번호 정규화 (하이픈 추가)
const rawPhone = data.phone.replace(/[^0-9]/g, '');
let formattedPhone = rawPhone;
if (rawPhone.length === 11) {
  formattedPhone = `${rawPhone.slice(0,3)}-${rawPhone.slice(3,7)}-${rawPhone.slice(7)}`;
}

return {
  json: {
    ...data,
    emailDomain,
    formattedPhone
  }
};
```

### 패턴 6: 조건부 데이터 구성

```javascript
const data = $input.item.json;

// 구매 금액에 따른 등급 분류
let tier;
if (data.totalPurchase >= 1000000) {
  tier = 'PLATINUM';
} else if (data.totalPurchase >= 500000) {
  tier = 'GOLD';
} else if (data.totalPurchase >= 100000) {
  tier = 'SILVER';
} else {
  tier = 'BRONZE';
}

// 다음 등급까지 필요한 금액
const nextTierThresholds = {
  'BRONZE': 100000,
  'SILVER': 500000,
  'GOLD': 1000000,
  'PLATINUM': Infinity
};

const nextThreshold = nextTierThresholds[tier];
const amountNeeded = nextThreshold === Infinity ? 0 : nextThreshold - data.totalPurchase;

return {
  json: {
    ...data,
    tier,
    amountNeededForNextTier: amountNeeded
  }
};
```

---

## 외부 변수 접근

### 이전 노드 데이터 참조

```javascript
// 특정 노드의 데이터 가져오기
const prevData = $node["Get Customer"].json;

// 현재 아이템과 결합
const current = $input.item.json;
return {
  json: {
    ...current,
    customerName: prevData.name
  }
};
```

### 환경 변수 접근

```javascript
// n8n 환경 변수 접근
const apiKey = $env.MY_API_KEY;
```

---

## 에러 처리

```javascript
try {
  const data = $input.item.json;

  // JSON 파싱 (문자열로 온 경우)
  const parsedData = JSON.parse(data.rawJson);

  return {
    json: {
      ...data,
      parsed: parsedData,
      success: true
    }
  };
} catch (error) {
  return {
    json: {
      success: false,
      error: error.message,
      originalData: $input.item.json
    }
  };
}
```

---

## AI에게 Code 노드 작성 요청하기

코드를 모르더라도 AI를 활용하면 됩니다.

좋은 프롬프트 예시:

```
n8n의 Code 노드에서 사용할 JavaScript 코드를 작성해 주세요.

입력 데이터:
{
  "orders": [
    {"product": "키보드", "price": 150000, "qty": 2},
    {"product": "마우스", "price": 80000, "qty": 1}
  ],
  "customer": "홍길동"
}

목표: 각 주문의 소계(price × qty)를 계산하고, 전체 합계를 추가한 데이터를 반환해 주세요.
출력은 n8n Code 노드의 형식(return [{json: {...}}])이어야 합니다.
```

---

## 핵심 요약

- Code 노드로 표현식으로 불가능한 복잡한 로직 처리 가능
- 출력은 반드시 `{ json: {...} }` 형식으로 반환
- "Run Once for All Items"로 여러 아이템을 한번에 처리
- 날짜 변환, 배열 처리, 통계, 그룹화 등 다양한 패턴 활용
- 코드를 모른다면 AI에게 작성 요청 가능

**다음 레슨**: HTTP Request 노드로 어떤 API든 호출하는 방법을 배웁니다.
