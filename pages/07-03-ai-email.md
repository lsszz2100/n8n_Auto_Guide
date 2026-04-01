# 레슨 7.3: AI 이메일 자동 분류 및 응답

---

## 프로젝트 개요

**목표**: AI가 이메일을 읽고 분류, 우선순위 결정, 초안 작성까지 자동화

**비포/애프터:**
```
Before:
CS 팀원이 하루 200통 이메일을 수동으로 읽고 분류 → 하루 4시간 소요

After:
AI가 초당 수십 건 처리, 단순 문의는 자동 답변, 복잡한 건만 인간에게 → 10분으로 단축
```

---

## 완성 워크플로우

```
[Gmail Trigger: 새 이메일]
    ↓
[AI: 이메일 분석]
  - 카테고리 분류
  - 긴급도 평가
  - 감정 분석
  - 핵심 정보 추출
    ↓
[Switch: 카테고리]
  ├── 단순 FAQ → [AI: 자동 답변 생성] → [Gmail: 즉시 발송]
  ├── 주문/배송 → [주문 시스템 조회] → [AI: 맞춤 답변] → [Gmail 발송]
  ├── 환불 → [환불 정책 확인] → [CS팀 Slack 알림] → [AI: 접수 확인 이메일]
  └── 복잡한 문의 → [담당자 배정] → [Slack 알림] → [AI: 접수 확인 이메일]
```

---

## STEP 1: AI 이메일 분석

```
[OpenAI: gpt-4o-mini]
설정:
  System: |
    이메일 분석 전문가. 다음 JSON 형식으로만 응답해.
    절대 다른 텍스트 포함 금지.

  User: |
    이메일을 분석해줘:
    제목: {{ $json.subject }}
    발신자: {{ $json.from.value[0].address }}
    내용: {{ $json.text || $json.snippet }}

    분석 기준:
    - category: "faq" | "order" | "refund" | "complaint" | "partnership" | "other"
    - urgency: "high" | "medium" | "low"
    - sentiment: "positive" | "neutral" | "negative" | "angry"
    - canAutoReply: true | false (단순 FAQ만 true)
    - keyInfo: 이메일에서 추출한 핵심 정보 (주문번호, 날짜 등)
    - summary: 50자 이내 요약

  Temperature: 0.1
  Response Format: JSON Object
```

---

## STEP 2: FAQ 자동 답변 생성

```
[IF: canAutoReply === true AND category === "faq"]
    ↓
[OpenAI: gpt-4o-mini]
설정:
  System: |
    너는 '스마트쇼핑'의 CS 담당자야.
    항상 한국어로, 친근하고 전문적으로 답변해.
    고객 이름으로 시작하고, 감사 인사로 마무리해.

    FAQ 지식 베이스:
    - 배송 기간: 결제 후 2-3 영업일
    - 반품 기간: 수령 후 7일 이내
    - 교환 방법: 고객센터 1588-0000 또는 앱 내 교환 신청
    - 영업 시간: 평일 9시-18시

  User: |
    고객 정보:
    이름: {{ $json.senderName || "고객" }}
    원본 이메일: {{ $json.text || $json.snippet }}
    분석 결과: {{ $json.summary }}

    이 고객의 질문에 완전한 답변을 작성해줘.

  Temperature: 0.5
  Max Tokens: 500
```

---

## STEP 3: 주문/배송 문의 처리

```
[IF: category === "order"]
    ↓
[Code: 주문번호 추출]
  const orderNumberMatch = $json.text.match(/ORD-\d+|주문번호[:：]\s*(\d+)/);
  const orderNumber = orderNumberMatch ? orderNumberMatch[1] : null;
    ↓
[IF: 주문번호 있음?]
  ├── True → [HTTP Request: 주문 시스템에서 상태 조회]
  │              ↓
  │           [AI: 주문 상태 기반 맞춤 답변 생성]
  └── False → [AI: 주문번호 확인 요청 이메일 생성]
    ↓
[Gmail: 답변 발송]
```

---

## STEP 4: 컨텍스트 인식 답변 생성

AI가 고객 이력을 참고하여 더 개인화된 답변 생성:

```javascript
// System Prompt에 고객 정보 포함
const systemPrompt = `
너는 '스마트쇼핑' CS 담당자야.

고객 정보:
- 이름: ${$json.customerInfo.name}
- 회원 등급: ${$json.customerInfo.tier}
- 총 구매 횟수: ${$json.customerInfo.purchaseCount}회
- 최근 주문: ${$json.orderInfo?.orderId || '없음'}
- 최근 주문 상태: ${$json.orderInfo?.status || '없음'}

답변 시 고객 등급에 맞는 어조를 사용해:
- PLATINUM/GOLD: 특별히 감사한 마음을 표현
- SILVER: 친근하고 전문적으로
- BRONZE: 일반적인 CS 응대
`;
```

---

## STEP 5: 품질 관리 - 사람 검토 루프

중요한 답변은 발송 전 사람이 검토:

```
[AI: 답변 초안 생성]
    ↓
[IF: 환불 또는 불만 이메일?]
  True →
    [Gmail: CS팀장에게 초안 + 검토 요청 발송]
    [Wait: 최대 2시간]
    [IF: 승인 응답?]
      ├── 승인 → [Gmail: 고객에게 최종 답변 발송]
      └── 수정 요청 → [수정 후 재승인 루프]
  False →
    [Gmail: 즉시 고객에게 발송]
```

---

## 성과 측정

```
[Schedule: 매일 오후 11시]
    ↓
[Google Sheets: 오늘 처리된 이메일 통계 조회]
    ↓
[Code: 통계 계산]
  - 총 수신 이메일 수
  - AI 자동 처리율 (%)
  - 평균 응답 시간 (분)
  - 카테고리별 분포
  - 고객 만족도 (답장 받은 답장 분석)
    ↓
[Slack: 일일 CS 리포트 발송]
```

---

## 핵심 요약

- AI가 이메일 카테고리, 긴급도, 감정, 핵심 정보를 자동 추출
- 단순 FAQ는 즉시 자동 답변, 복잡한 건은 담당자 배정
- 고객 이력과 주문 정보를 컨텍스트로 활용한 개인화 답변
- 중요 이메일은 사람 검토 루프 포함하여 품질 보장
- 일일 통계로 성과 추적 및 지속 개선

**다음 레슨**: AI 고객 지원 봇을 구축합니다.
