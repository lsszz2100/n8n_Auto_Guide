# 레슨 3.5: 표현식(Expressions) 마스터

---

## 표현식이란?

**표현식(Expression)**은 n8n에서 고정값 대신 **동적으로 값을 계산하거나 참조**하는 방법입니다.

```
고정값:  "홍길동"
표현식: {{ $json.name }}  ← 이전 노드에서 name 필드를 가져옴
```

표현식 없이는 워크플로우가 정적입니다. 표현식이 있으면 모든 것이 동적으로 변합니다.

---

## 표현식 기본 문법

표현식은 이중 중괄호 `{{ }}` 안에 작성합니다.

```
{{ 표현식 내용 }}
```

### 예시

```
Hello {{ $json.name }}님!
→ "Hello 홍길동님!" (name이 "홍길동"인 경우)

총 금액: {{ $json.price * $json.qty }}원
→ "총 금액: 150000원"
```

---

## $json — 현재 아이템 데이터

`$json`은 현재 노드에서 처리 중인 아이템의 데이터를 참조합니다.

```javascript
{{ $json.name }}         // 문자열 필드
{{ $json.amount }}       // 숫자 필드
{{ $json.is_active }}    // 불린 필드
{{ $json.tags[0] }}      // 배열의 첫 번째 항목
{{ $json.address.city }} // 중첩 객체 필드
```

### Bracket Notation (특수문자가 포함된 키)

키에 공백이나 특수문자가 있을 때:
```javascript
{{ $json['order-id'] }}       // 하이픈이 있는 키
{{ $json['고객 이름'] }}        // 한글, 공백이 있는 키
{{ $json['data.created_at'] }} // 점(.)이 포함된 키
```

---

## $node — 특정 노드의 데이터 참조

이전 노드가 아닌 **특정 노드의 데이터**를 참조합니다.

```javascript
{{ $node["Get User"].json.email }}
//         ↑ 노드 이름       ↑ 해당 노드의 email 필드
```

**사용 예시**: 여러 단계 전의 데이터를 다시 참조해야 할 때

```
[Webhook] → [Get User] → [Process] → [Send Email]
                ↑
        Send Email에서 Get User의 email을 참조하려면:
        {{ $node["Get User"].json.email }}
```

---

## $input — 직전 노드 데이터

`$input`은 바로 이전 노드의 데이터를 참조합니다.

```javascript
{{ $input.item.json.name }}      // 현재 아이템
{{ $input.first().json.name }}   // 첫 번째 아이템
{{ $input.last().json.name }}    // 마지막 아이템
{{ $input.all()[0].json.name }}  // 전체 아이템 배열의 첫 번째
```

---

## $now — 현재 시간

```javascript
{{ $now }}                    // ISO 형식: "2026-04-01T09:00:00.000Z"
{{ $now.toISO() }}            // 동일
{{ $now.toFormat('yyyy-MM-dd') }}    // "2026-04-01"
{{ $now.toFormat('HH:mm') }}         // "09:00"
{{ $now.toFormat('yyyy년 MM월 dd일') }} // "2026년 04월 01일"
```

n8n은 시간 처리에 **Luxon** 라이브러리를 사용합니다.

### 시간 계산

```javascript
{{ $now.plus({ days: 7 }) }}        // 7일 후
{{ $now.minus({ hours: 2 }) }}      // 2시간 전
{{ $now.startOf('month') }}         // 이번 달 1일 00:00
{{ $now.endOf('week') }}            // 이번 주 일요일 23:59
```

---

## 자주 사용하는 표현식 패턴

### 문자열 조합

```javascript
{{ $json.firstName + " " + $json.lastName }}
// → "홍 길동"

{{ `안녕하세요, ${$json.name}님! 주문번호는 ${$json.orderId}입니다.` }}
// → "안녕하세요, 홍길동님! 주문번호는 ORD-001입니다."
```

### 조건 표현 (삼항 연산자)

```javascript
{{ $json.amount > 50000 ? "VIP" : "일반" }}
// amount가 50000 초과면 "VIP", 아니면 "일반"

{{ $json.is_active ? "활성" : "비활성" }}
```

### 숫자 포맷

```javascript
{{ $json.price.toLocaleString('ko-KR') }}
// → "150,000" (천 단위 쉼표)

{{ Math.round($json.score * 100) / 100 }}
// → 소수점 2자리 반올림
```

### 문자열 처리

```javascript
{{ $json.email.toLowerCase() }}        // 소문자 변환
{{ $json.name.toUpperCase() }}         // 대문자 변환
{{ $json.phone.replace(/-/g, '') }}    // 하이픈 제거
{{ $json.title.slice(0, 50) }}         // 앞 50자만
{{ $json.text.trim() }}                // 앞뒤 공백 제거
```

### null/undefined 처리

```javascript
{{ $json.name ?? "이름 없음" }}
// name이 null이나 undefined면 "이름 없음"

{{ $json.tags?.length > 0 ? $json.tags[0] : "태그 없음" }}
// Optional chaining으로 안전하게 접근
```

---

## 날짜 처리 실전 예시

```javascript
// 오늘 날짜를 "YYYYMMDD" 형식으로 (파일명에 활용)
{{ $now.toFormat('yyyyMMdd') }}
// → "20260401"

// 30일 후 날짜
{{ $now.plus({ days: 30 }).toFormat('yyyy-MM-dd') }}
// → "2026-05-01"

// Unix 타임스탬프를 날짜로 변환
{{ DateTime.fromMillis($json.timestamp).toFormat('yyyy-MM-dd HH:mm') }}

// 날짜 문자열을 파싱
{{ DateTime.fromISO($json.created_at).toFormat('yyyy년 MM월 dd일') }}
```

---

## 배열 처리 표현식

```javascript
// 배열 길이
{{ $json.items.length }}

// 배열을 문자열로 합치기
{{ $json.tags.join(', ') }}
// → "긴급, VIP, 결제완료"

// 배열 포함 여부
{{ $json.roles.includes('admin') ? '관리자' : '일반 사용자' }}
```

---

## 표현식 작성 팁

### 1. 데이터 패널에서 복사하기
노드 실행 후 데이터 패널에서 필드명을 클릭하면 표현식이 자동 복사됩니다.

### 2. 표현식 편집기 사용
파라미터 필드에서 `fx` 아이콘 클릭 → 표현식 편집기가 열립니다.
- 자동완성 지원
- 실시간 미리보기
- 문법 오류 표시

### 3. 복잡한 로직은 Code 노드로
표현식이 너무 복잡해지면 Code 노드를 사용하는 것이 더 관리하기 쉽습니다.

---

## 자주 하는 실수

### 실수 1: 따옴표 중복
```javascript
// 잘못된 예
{{ "$json.name" }}  // 문자열 그대로 출력됨

// 올바른 예
{{ $json.name }}
```

### 실수 2: 존재하지 않는 필드 참조
```javascript
// 에러 발생 가능
{{ $json.user.email }}  // user가 null이면 에러

// 안전한 접근
{{ $json.user?.email ?? "이메일 없음" }}
```

### 실수 3: 타입 불일치
```javascript
// 문자열 + 숫자 (의도치 않은 결과)
{{ $json.price + $json.qty }}  // "150000" + 2 = "1500002"

// 올바른 숫자 계산
{{ Number($json.price) + Number($json.qty) }}
```

---

## 핵심 요약

- 표현식은 `{{ }}` 안에 작성하는 동적 데이터 참조 방법
- `$json`: 현재 아이템 데이터
- `$node["이름"].json`: 특정 노드 데이터
- `$now`: 현재 시간 (Luxon 라이브러리 사용)
- 삼항 연산자 `?:`로 조건 처리, `??`로 null 처리
- 복잡한 로직은 Code 노드로 분리

**다음 레슨**: JavaScript 코드를 직접 작성하는 Code(Function) 노드를 알아봅니다.
