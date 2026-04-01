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

## 실전 예제 모음

### 예제 1: 공통 에러 알림 서브워크플로우

여러 워크플로우가 동일한 에러 알림 로직을 재사용합니다. 에러가 발생한 워크플로우 이름, 오류 내용, 심각도를 받아서 Slack과 이메일로 알립니다.

워크플로우 이름: "공통 에러 알림 모듈"

```
[Execute Workflow Trigger]
  수신 데이터:
    workflowName: 오류가 발생한 워크플로우 이름
    errorMessage: 오류 메시지
    severity: "low" | "medium" | "high" | "critical"
    context: 추가 정보 (선택)
    ↓
[Code: 심각도별 알림 구성]
  const { workflowName, errorMessage, severity, context } = $json;

  const icons = {
    low: "ℹ️",
    medium: "⚠️",
    high: "🔴",
    critical: "🚨"
  };

  const slackChannel = severity === "critical"
    ? "#alerts-critical"
    : "#alerts-general";

  return [{
    json: {
      icon: icons[severity] || "⚠️",
      channel: slackChannel,
      workflowName,
      errorMessage,
      severity,
      context: context || "없음",
      timestamp: new Date().toLocaleString("ko-KR", { timeZone: "Asia/Seoul" }),
      shouldEmail: severity === "critical"
    }
  }];
    ↓
[Slack: 오류 알림 발송]
  Channel: {{ $json.channel }}
  Message: |
    {{ $json.icon }} 워크플로우 오류 발생

    워크플로우: {{ $json.workflowName }}
    심각도: {{ $json.severity }}
    오류 내용: {{ $json.errorMessage }}
    추가 정보: {{ $json.context }}
    발생 시각: {{ $json.timestamp }}
    ↓
[IF: shouldEmail === true?]
  True →
    [Gmail: 긴급 에러 이메일]
      To: devops@company.com
      Subject: [긴급] {{ $json.workflowName }} 오류 발생
      Body: 심각도 CRITICAL 오류가 발생했습니다.
            {{ $json.errorMessage }}
  False → [종료]
    ↓
[Code: 결과 반환]
  return [{ json: { success: true, notified: true } }];
```

호출하는 메인 워크플로우 예시:

```javascript
// 어떤 워크플로우에서든 오류 처리 구간에 아래 Execute Workflow 노드 추가
// Execute Workflow 노드 설정:
{
  "workflowName": "주문 처리 워크플로우",
  "errorMessage": "{{ $json.error.message }}",
  "severity": "high",
  "context": "주문 ID: {{ $json.orderId }}, 고객: {{ $json.customerEmail }}"
}
```

---

### 예제 2: 고객 데이터 정제 서브워크플로우

여러 워크플로우에서 수집된 고객 데이터의 전화번호와 이메일을 표준 형식으로 정규화합니다.

워크플로우 이름: "고객 데이터 정제 모듈"

```
[Execute Workflow Trigger]
  수신 데이터:
    phone: 정제할 전화번호 (다양한 형식 가능)
    email: 정제할 이메일 주소
    name: 고객 이름 (선택)
    ↓
[Code: 전화번호 정규화]
  const rawPhone = $json.phone || "";
  const rawEmail = $json.email || "";
  const rawName = $json.name || "";

  // 전화번호에서 숫자만 추출
  const digits = rawPhone.replace(/\D/g, "");

  // 한국 전화번호 형식으로 변환
  let normalizedPhone = "";
  if (digits.length === 11 && digits.startsWith("010")) {
    normalizedPhone = `${digits.slice(0,3)}-${digits.slice(3,7)}-${digits.slice(7)}`;
  } else if (digits.length === 10) {
    normalizedPhone = `${digits.slice(0,3)}-${digits.slice(3,6)}-${digits.slice(6)}`;
  } else {
    normalizedPhone = rawPhone; // 변환 불가 시 원본 유지
  }

  // 이메일 소문자 변환 + 공백 제거
  const normalizedEmail = rawEmail.trim().toLowerCase();

  // 이메일 유효성 검사
  const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  const isValidEmail = emailPattern.test(normalizedEmail);

  // 이름 앞뒤 공백 제거
  const normalizedName = rawName.trim();

  return [{
    json: {
      original: { phone: rawPhone, email: rawEmail, name: rawName },
      normalized: {
        phone: normalizedPhone,
        email: normalizedEmail,
        name: normalizedName
      },
      validation: {
        phoneValid: normalizedPhone.includes("-"),
        emailValid: isValidEmail
      }
    }
  }];
    ↓
[IF: 유효성 검사 실패?]
  True →
    [Code: 실패 결과 반환]
      return [{
        json: {
          success: false,
          errors: [
            !$json.validation.emailValid ? "이메일 형식 오류" : null,
            !$json.validation.phoneValid ? "전화번호 형식 오류" : null
          ].filter(Boolean)
        }
      }];
  False →
    [Code: 성공 결과 반환]
      return [{
        json: {
          success: true,
          phone: $json.normalized.phone,
          email: $json.normalized.email,
          name: $json.normalized.name
        }
      }];
```

호출 예시 (CRM 연동 워크플로우에서):

```javascript
// Execute Workflow 노드 Input Data:
{
  "phone": "010 1234 5678",   // → 010-1234-5678
  "email": "  User@COMPANY.COM  ", // → user@company.com
  "name": "  홍길동  "          // → 홍길동
}

// 반환 결과:
// { success: true, phone: "010-1234-5678", email: "user@company.com", name: "홍길동" }
```

---

### 예제 3: AI 요약 서브워크플로우

어떤 텍스트든 받아서 3줄 요약을 반환하는 범용 요약 모듈입니다. 이메일 분류, 고객 문의 처리, 뉴스 요약 등 다양한 워크플로우에서 재사용합니다.

워크플로우 이름: "AI 텍스트 요약 모듈"

```
[Execute Workflow Trigger]
  수신 데이터:
    text: 요약할 텍스트 (필수)
    language: 출력 언어, 기본값 "ko"
    maxLength: 요약 최대 글자 수, 기본값 200
    context: 텍스트 유형 힌트 (예: "고객문의", "이메일", "뉴스기사")
    ↓
[Code: 요약 요청 구성]
  const { text, language, maxLength, context } = $json;

  const lang = language || "ko";
  const max = maxLength || 200;
  const ctx = context || "일반 텍스트";

  // 텍스트가 너무 짧으면 요약 생략
  if (text.length < 100) {
    return [{
      json: {
        summary: text,
        skipped: true,
        reason: "텍스트가 100자 미만으로 요약 생략"
      }
    }];
  }

  return [{
    json: {
      prompt: `다음 ${ctx}를 ${lang === "ko" ? "한국어" : "영어"}로 3줄 이내로 요약해주세요.
핵심 내용만 포함하고 각 줄은 완결된 문장으로 작성하세요.
최대 ${max}자 이내로 작성하세요.

텍스트:
${text.slice(0, 8000)}`,  // 토큰 절약을 위해 8000자 제한
      textLength: text.length
    }
  }];
    ↓
[OpenAI: Chat Completion]
  Model: gpt-4o-mini
  System Message: 당신은 텍스트 요약 전문가입니다.
                  간결하고 정확한 요약을 제공합니다.
  User Message: {{ $json.prompt }}
  Max Tokens: 300
  Temperature: 0.3
    ↓
[Code: 결과 파싱 및 반환]
  const rawSummary = $json.message.content.trim();

  // 줄 단위로 분리
  const lines = rawSummary.split("\n")
    .map(l => l.replace(/^\d+\.\s*/, "").trim())
    .filter(l => l.length > 0);

  return [{
    json: {
      success: true,
      summary: rawSummary,
      summaryLines: lines,
      lineCount: lines.length,
      originalLength: $('Code: 요약 요청 구성').item.json.textLength
    }
  }];
```

호출 예시 (고객 문의 처리 워크플로우에서):

```javascript
// Execute Workflow 노드 Input Data:
{
  "text": "{{ $json.customerMessage }}",
  "language": "ko",
  "maxLength": 150,
  "context": "고객문의"
}

// 반환 결과:
// {
//   success: true,
//   summary: "고객이 배송 지연에 대해 문의함\n주문 번호는 #12345이며 3일 전 주문\n환불 또는 신속 배송 요청",
//   summaryLines: ["고객이 배송 지연에 대해 문의함", "..."],
//   lineCount: 3
// }
```

---

## 핵심 요약

- 서브 워크플로우 = 재사용 가능한 자동화 함수
- Execute Workflow 노드로 다른 워크플로우 호출
- 유틸리티 모듈(이메일, 슬랙 등)을 공통 서브 워크플로우로 관리
- 레이어 아키텍처로 트리거 → 비즈니스 로직 → 유틸리티 분리
- 오류 처리와 깊이 제한으로 안정성 확보

**다음 레슨**: n8n API로 외부 시스템에서 워크플로우를 제어합니다.
