# 레슨 8.1: 서브 워크플로우 패턴

---

## 서브 워크플로우란?

**서브 워크플로우**는 다른 워크플로우에서 호출할 수 있는 독립적인 워크플로우입니다. 마치 프로그래밍의 함수(function)처럼 재사용 가능한 자동화 모듈을 만드는 개념입니다.

**언제 사용하나요?**

- 동일한 로직이 여러 워크플로우에서 반복될 때
- 워크플로우가 너무 커지고 복잡해질 때
- 팀원들과 공통 기능을 공유하고 싶을 때
- 특정 기능을 독립적으로 테스트하고 싶을 때

---

## Execute Workflow 노드

n8n에서 서브 워크플로우는 **Execute Workflow** 노드로 호출합니다.

```
[메인 워크플로우]
    ↓
[Execute Workflow]
  설정:
    Source: Database (워크플로우 ID로 선택)
    Workflow: "이메일 발송 모듈"
    Wait for Completion: true
    Input Data: {{ $json }}
    ↓
[서브 워크플로우 결과 받기]
```

---

## 서브 워크플로우 만들기

### Step 1: 트리거 설정

서브 워크플로우는 **Execute Workflow Trigger** 노드로 시작합니다:

```
[Execute Workflow Trigger]
  설명: 다른 워크플로우에서 호출될 때 시작됨
  Input Data: 호출자가 전달한 데이터 수신
```

### Step 2: 처리 로직 구현

```
[Execute Workflow Trigger]
    ↓
[처리 로직 노드들...]
    ↓
[마지막 노드: 결과 반환]
```

### Step 3: 결과 반환

서브 워크플로우의 마지막 노드 출력이 자동으로 호출자에게 반환됩니다.

---

## 실전 예시: 이메일 발송 모듈

여러 워크플로우에서 사용하는 공통 이메일 발송 서브 워크플로우:

```
워크플로우 이름: "📧 이메일 발송 모듈"

[Execute Workflow Trigger]
  수신 데이터:
    to: 수신자 이메일
    subject: 제목
    template: "order_confirm" | "welcome" | "notification"
    data: 템플릿 변수들
    ↓
[Switch: template]
  "order_confirm" → [Code: 주문 확인 HTML 생성]
  "welcome"       → [Code: 환영 이메일 HTML 생성]
  "notification"  → [Code: 알림 이메일 HTML 생성]
    ↓
[Gmail: 이메일 발송]
  To: {{ $json.to }}
  Subject: {{ $json.subject }}
  HTML: {{ $json.emailHtml }}
    ↓
[결과 반환]
  { success: true, messageId: "..." }
```

**호출하는 메인 워크플로우:**

```
[주문 완료 이벤트]
    ↓
[Execute Workflow]
  워크플로우: "📧 이메일 발송 모듈"
  Input: {
    "to": "{{ $json.customerEmail }}",
    "subject": "주문이 완료되었습니다!",
    "template": "order_confirm",
    "data": {
      "orderNumber": "{{ $json.orderId }}",
      "items": {{ $json.items }},
      "total": "{{ $json.totalAmount }}"
    }
  }
```

---

## 모듈화 아키텍처 설계

### 레이어 구조

```
[레이어 1: 트리거 워크플로우]
  - Gmail 트리거
  - Webhook 트리거
  - Schedule 트리거
        ↓ 호출
[레이어 2: 비즈니스 로직 워크플로우]
  - 주문 처리 워크플로우
  - 고객 관리 워크플로우
        ↓ 호출
[레이어 3: 유틸리티 모듈]
  - 이메일 발송 모듈
  - 슬랙 알림 모듈
  - 데이터 검증 모듈
  - 로깅 모듈
```

### 공통 유틸리티 모듈 예시

```
📁 워크플로우 모듈 라이브러리
│
├── 📧 이메일 발송 모듈
├── 💬 Slack 알림 모듈
├── 📊 Google Sheets 저장 모듈
├── 🔔 긴급 알림 모듈 (전화/SMS)
├── 📝 로그 기록 모듈
└── ✅ 데이터 검증 모듈
```

---

## 데이터 전달 방법

### 방법 1: Input Data (권장)

```javascript
// 메인 워크플로우에서
{
  "Input Data": {
    "userId": "{{ $json.id }}",
    "action": "process_order",
    "payload": {{ $json.orderData }}
  }
}

// 서브 워크플로우에서
const userId = $json.userId;
const action = $json.action;
```

### 방법 2: 전역 변수 (n8n Variables)

n8n 설정에서 공통 변수 정의:
```
Settings → Variables:
  COMPANY_NAME = "스마트쇼핑"
  SUPPORT_EMAIL = "support@shop.com"
  API_BASE_URL = "https://api.shop.com"

// 워크플로우에서 사용
{{ $vars.COMPANY_NAME }}
```

---

## 오류 처리와 서브 워크플로우

```
[Execute Workflow]
  Continue on Fail: true  ← 서브 워크플로우 실패 시 계속 진행
    ↓
[IF: $json.error exists?]
  True → [오류 처리 로직]
  False → [정상 흐름]
```

서브 워크플로우 내부에서도 오류 반환:

```javascript
// 서브 워크플로우의 마지막 Code 노드
try {
  // 처리 로직
  return [{ json: { success: true, data: result } }];
} catch (error) {
  return [{ json: { success: false, error: error.message } }];
}
```

---

## 재귀 워크플로우 (주의)

서브 워크플로우가 자기 자신을 호출하는 재귀 패턴은 가능하지만 **무한 루프 주의**:

```javascript
// 안전한 재귀: 깊이 제한 필수
const depth = $json.depth || 0;
if (depth >= 5) {
  return [{ json: { result: "최대 깊이 도달", items: $json.items } }];
}

// 재귀 호출
// Execute Workflow 노드에서 depth: depth + 1 전달
```

---

## 성능 고려사항

| 상황 | 권장 방법 |
|------|-----------|
| 순차 처리 필요 | Wait for Completion: true |
| 빠른 병렬 처리 | Wait for Completion: false |
| 대량 데이터 | 배치로 나누어 서브 워크플로우 호출 |
| 단순 재사용 | 서브 워크플로우 대신 Code 노드 함수 사용 |

---

## 핵심 요약

- 서브 워크플로우 = 재사용 가능한 자동화 함수
- Execute Workflow 노드로 다른 워크플로우 호출
- 유틸리티 모듈(이메일, 슬랙 등)을 공통 서브 워크플로우로 관리
- 레이어 아키텍처로 트리거 → 비즈니스 로직 → 유틸리티 분리
- 오류 처리와 깊이 제한으로 안정성 확보

**다음 레슨**: n8n API로 외부 시스템에서 워크플로우를 제어합니다.
