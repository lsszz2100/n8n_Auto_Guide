# 레슨 4.3: IF 조건과 Switch 분기 처리

---

## 조건 처리의 중요성

자동화에서 조건 처리는 핵심입니다. 모든 데이터를 똑같이 처리하는 것이 아니라, **상황에 따라 다르게 반응**할 수 있어야 합니다.

---

## IF 노드 심화

### 기본 구조

```
[이전 노드] → [IF: 조건] → True 출력 → [A 처리]
                         → False 출력 → [B 처리]
```

### 조건 타입별 활용

#### 문자열 조건

```javascript
// 이메일이 특정 도메인인지
email Contains "@company.com"

// 상태가 특정 값인지
status Equals "pending"

// 텍스트가 특정 단어로 시작하는지
subject Starts With "[긴급]"

// 정규식 매칭
phone Matches Regex "^01[0-9]-\d{4}-\d{4}$"
```

#### 숫자 조건

```javascript
// 금액이 특정 기준 이상
amount Greater Than 100000

// 나이 범위
age Between 20 And 30

// 수량이 0인지
quantity Equals 0
```

#### 날짜 조건

```javascript
// 오늘 이후인지
dueDate Is After Today

// 지난 7일 이내인지
createdAt Is After {{ $now.minus({days: 7}) }}
```

### 복합 조건 (AND / OR)

```javascript
// AND: 둘 다 만족해야 함
조건 1: amount Greater Than 50000
AND
조건 2: tier Equals "VIP"
→ VIP이면서 5만원 이상 구매

// OR: 하나만 만족해도 됨
조건 1: tier Equals "PLATINUM"
OR
조건 2: amount Greater Than 1000000
→ 플래티넘 등급이거나 100만원 이상 구매
```

---

## Switch 노드 심화

### 3가지 이상의 분기 처리

```
[Switch: 주문 상태]
  ├── "pending" → [결제 대기 처리]
  ├── "paid" → [배송 준비 처리]
  ├── "shipped" → [배송 알림 발송]
  ├── "delivered" → [완료 처리]
  └── Default → [알 수 없는 상태 알림]
```

### Switch 설정 방법

- **Mode**: Rules
- **Field**: `{{ $json.status }}`
- 각 Rule: 값 = "pending", "paid" 등

### 값 기반 vs 출력 인덱스 기반

**값 기반 (Rules mode)**:
- 각 조건별로 설정
- 가장 직관적

**출력 인덱스 기반**:
- `{{ $json.tierIndex }}` = 0, 1, 2, 3
- 숫자 인덱스로 경로 선택
- 더 동적인 처리 가능

---

## 실전 패턴

### 패턴 1: 리드 스코어링

```
[Webhook: 폼 제출]
    ↓
[Switch: 예산 규모]
  ├── 1억 이상 → [Slack: 영업팀장에게 즉시 알림]
  ├── 1천만~1억 → [CRM: 담당 영업사원 배정]
  ├── 백만~1천만 → [이메일: 자동 안내 발송]
  └── 백만 미만 → [이메일: 셀프서비스 안내]
```

### 패턴 2: 이메일 분류 자동화

```
[Gmail Trigger: 새 이메일]
    ↓
[Switch: 제목 키워드]
  ├── "환불" → [CS팀 Slack 알림 + 환불 처리 시작]
  ├── "주문" → [주문 처리 워크플로우 시작]
  ├── "채용" → [HR팀 Notion 등록]
  ├── "파트너" → [영업팀 CRM 등록]
  └── Default → [일반 수신함]
```

### 패턴 3: 다단계 IF

여러 IF를 연결하여 복잡한 조건 처리:

```
[데이터 수신]
    ↓
[IF: 이메일 유효?]
    │ False → [에러 처리]
    │ True ↓
[IF: 이미 등록됨?]
    │ True → [업데이트]
    │ False ↓
[IF: VIP 조건?]
    │ True → [VIP 온보딩]
    │ False → [일반 온보딩]
```

---

## 분기 후 합치기 패턴

분기한 경로를 나중에 하나로 합쳐야 할 때:

```
[IF: 고객 타입]
  ├── VIP → [VIP 메시지 생성]
  └── 일반 → [일반 메시지 생성]
         ↓ (두 경로 모두)
    [Merge: 아이템 병합]
         ↓
    [Gmail: 이메일 발송]
```

---

## 자주 하는 실수

### 실수 1: 모든 케이스를 커버하지 않음

```
[Switch]
  ├── "A" → ...
  └── "B" → ...
  (Default 없음!)
```

"C"나 다른 값이 들어오면 데이터가 사라집니다.
**항상 Default 경로를 만들어 두세요.**

### 실수 2: IF의 False 경로를 무시

IF의 False 출력을 연결하지 않으면 그 데이터는 조용히 사라집니다.
적어도 `No Operation` 노드나 에러 알림 노드에 연결하세요.

### 실수 3: 타입 불일치

```
// $json.amount가 문자열 "100000"인데
amount Equals 100000  // 숫자와 비교 → 매칭 안 됨

// 해결:
Number($json.amount) Equals 100000
// 또는
$json.amount.toString() Equals "100000"
```

---

## 핵심 요약

- IF: 참/거짓 2분기, AND/OR 복합 조건 지원
- Switch: 3개 이상 다중 분기, Default 경로 필수
- 분기 후 Merge로 재결합 가능
- 항상 False/Default 경로 처리하여 데이터 손실 방지
- 타입 일치 확인 필수

**다음 레슨**: Loop으로 반복 처리하는 방법을 배웁니다.
