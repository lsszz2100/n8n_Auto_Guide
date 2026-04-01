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

## 실전 예제 모음

---

### 예제 1: 반품/환불 요청 이메일 자동 처리

고객이 보내는 반품/환불 이메일을 AI가 분류하고, 조건에 따라 자동으로 답장을 생성하거나 담당자에게 에스컬레이션합니다.

워크플로우 구조:

```
[Gmail Trigger: 새 이메일 수신]
    ↓
[OpenAI: 반품/환불 여부 분류]
    ↓
[IF: 카테고리 === "refund" OR "return"]
  True →
    [Code: 반품 조건 검증]
      - 구매일로부터 경과 일수 계산
      - 금액 기준 (5만원 초과 여부)
      - 사유 유형 분류 (단순 변심, 하자, 오배송)
    ↓
    [Switch: 반품 처리 경로]
      ├── 7일 이내 + 단순 변심 → [AI: 반품 안내 이메일 자동 발송]
      ├── 7일 초과 → [AI: 기간 초과 안내 + 예외 검토 안내]
      ├── 하자/오배송 → [Slack: CS팀 긴급 알림] → [AI: 접수 확인 이메일]
      └── 5만원 초과 → [CS팀장 Slack 알림 + 수동 검토 요청]
  False → [기존 분류 흐름으로]
```

AI 분류 프롬프트 설정:

```
[OpenAI: gpt-4o-mini]
System: |
  이메일 내용에서 반품/환불 요청을 분석해. 다음 JSON만 반환해.
  {
    "isRefundRequest": true | false,
    "refundType": "change_of_mind" | "defective" | "wrong_delivery" | "other",
    "purchaseDateMentioned": "날짜 문자열 또는 null",
    "orderIdMentioned": "주문번호 또는 null",
    "estimatedAmount": "금액(숫자) 또는 null",
    "urgency": "high" | "medium" | "low"
  }

User: |
  이메일 제목: {{ $json.subject }}
  이메일 본문: {{ $json.text }}

Temperature: 0.1
```

반품 조건 검증 코드 (Code 노드):

```javascript
const analysis = JSON.parse($node["OpenAI"].json.message.content);
const today = new Date();

// 구매일로부터 경과 일수 계산
let daysSincePurchase = null;
if (analysis.purchaseDateMentioned) {
  const purchaseDate = new Date(analysis.purchaseDateMentioned);
  daysSincePurchase = Math.floor((today - purchaseDate) / (1000 * 60 * 60 * 24));
}

const amount = analysis.estimatedAmount || 0;
const refundType = analysis.refundType;

let route = "standard";

if (refundType === "defective" || refundType === "wrong_delivery") {
  route = "priority";
} else if (amount > 50000) {
  route = "manager_review";
} else if (daysSincePurchase !== null && daysSincePurchase > 7) {
  route = "expired";
} else {
  route = "auto_approve";
}

return [{
  json: {
    ...analysis,
    daysSincePurchase,
    route,
    emailFrom: $node["Gmail Trigger"].json.from.value[0].address,
    emailSubject: $node["Gmail Trigger"].json.subject
  }
}];
```

자동 답장 프롬프트 (auto_approve 경로):

```
[OpenAI: gpt-4o-mini]
System: |
  너는 '하모니쇼핑' CS 담당자 "지수"야.
  정중하고 간결하게 한국어로 답변해.
  반품 접수 완료를 안내하고, 다음 절차를 명확하게 설명해.

User: |
  고객 이메일: {{ $json.emailFrom }}
  요청 유형: {{ $json.refundType }}
  구매 후 경과일: {{ $json.daysSincePurchase }}일

  반품 택배 발송 안내 이메일을 작성해줘.
  - 반품 주소: 서울시 강남구 역삼동 OO물류센터
  - 택배사: 편의점 택배 (CU/GS25) 또는 자택 방문 수거 (3일 내)
  - 환불 처리 기간: 반품 도착 후 영업일 3일

Temperature: 0.4
Max Tokens: 400
```

---

### 예제 2: 채용 지원 이메일 자동 1차 필터링 + 면접 일정 안내

채용 공고에 이메일로 들어오는 지원서를 AI가 1차 검토하여 적합/부적합을 판단하고, 적합 지원자에게 면접 일정을 자동으로 안내합니다.

워크플로우 구조:

```
[Gmail Trigger: 채용 지원 이메일 수신]
  (필터: 제목에 "[지원]" 포함)
    ↓
[OpenAI: 지원서 분석]
  - 직무 연관성 점수 (0~100)
  - 경력 연수 추출
  - 기술 스택 매칭
  - 포트폴리오 링크 존재 여부
    ↓
[Google Sheets: 지원자 목록에 결과 저장]
    ↓
[Switch: 점수 기반 분류]
  ├── 80점 이상 → [AI: 면접 일정 안내 이메일 발송] → [Google Calendar: 면접 일정 블록 생성]
  ├── 50~79점 → [AI: 서류 검토 중 안내 이메일] → [Slack: HR팀 검토 요청]
  └── 50점 미만 → [AI: 정중한 탈락 안내 이메일 발송]
```

지원서 분석 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  채용 담당자로서 지원 이메일을 분석해. JSON만 반환해.
  {
    "fitScore": 0~100 (직무 적합도 점수),
    "experienceYears": 숫자 (경력 연수, 신입이면 0),
    "detectedSkills": ["감지된 기술 스택 목록"],
    "hasPortfolio": true | false,
    "hasResume": true | false,
    "standoutPoints": "지원자의 강점 1~2문장",
    "concerns": "우려 사항 1문장 또는 null",
    "recommendedAction": "interview" | "review" | "reject"
  }

  채용 공고 요구 사항:
  - 직무: 백엔드 개발자
  - 필수: Node.js 또는 Python, 3년 이상 경력
  - 우대: Docker, AWS, PostgreSQL
  - 포트폴리오 또는 GitHub 필수

User: |
  지원 이메일 제목: {{ $json.subject }}
  지원 이메일 본문: {{ $json.text }}
  첨부 파일 존재 여부: {{ $json.attachments.length > 0 ? "있음" : "없음" }}

Temperature: 0.1
```

Google Sheets 저장 설정:

```
[Google Sheets: Append Row]
  스프레드시트 ID: YOUR_SHEET_ID
  시트 이름: 지원자_목록_2026
  데이터:
    접수일시: {{ $now.toISO() }}
    이름: {{ $json.from.value[0].name }}
    이메일: {{ $json.from.value[0].address }}
    적합도점수: {{ $json.fitScore }}
    경력연수: {{ $json.experienceYears }}
    감지기술: {{ $json.detectedSkills.join(", ") }}
    포트폴리오: {{ $json.hasPortfolio ? "있음" : "없음" }}
    추천결과: {{ $json.recommendedAction }}
    강점: {{ $json.standoutPoints }}
```

면접 일정 안내 이메일 프롬프트 (80점 이상):

```
[OpenAI: gpt-4o-mini]
System: |
  '테크스타트' 채용 담당자 "이민준"으로서 이메일을 작성해.
  전문적이고 따뜻한 톤으로, 지원자가 설레는 마음을 가지도록 작성해.

User: |
  지원자 이름: {{ $json.from.value[0].name }}
  직무 적합도: {{ $json.fitScore }}점
  지원자 강점: {{ $json.standoutPoints }}

  다음 내용이 포함된 면접 안내 이메일을 작성해줘:
  - 서류 합격 축하
  - 1차 면접 일정 안내: 다음 주 화요일~목요일, 오전 10시 / 오후 2시 / 오후 4시 중 선택
  - 면접 형식: 화상 면접 (Zoom), 약 45분
  - 준비 사항: 간단한 자기소개 및 최근 프로젝트 설명
  - 일정 확인 회신 요청

Temperature: 0.6
Max Tokens: 500
```

---

### 예제 3: 뉴스레터 오픈율 분석 후 재발송 대상 자동 선정

뉴스레터를 보낸 후 일정 시간이 지나면 오픈하지 않은 구독자를 자동으로 추출하고, 제목을 변경하여 재발송합니다.

워크플로우 구조:

```
[Schedule Trigger: 뉴스레터 발송 48시간 후 실행]
    ↓
[HTTP Request: Mailchimp API - 캠페인 통계 조회]
  GET https://us1.api.mailchimp.com/3.0/reports/{{ campaignId }}
    ↓
[Code: 오픈율 계산 및 재발송 판단]
    ↓
[IF: 오픈율 < 30%]
  True →
    [HTTP Request: 미오픈 수신자 목록 조회]
      GET https://us1.api.mailchimp.com/3.0/reports/{{ campaignId }}/unsubscribes
    ↓
    [OpenAI: 대안 제목 3개 생성]
    ↓
    [Code: 가장 클릭률 예상 높은 제목 선택]
    ↓
    [HTTP Request: 미오픈 세그먼트로 새 캠페인 생성 및 발송]
  False →
    [Slack: "오픈율 양호, 재발송 불필요" 알림]
```

통계 조회 및 판단 코드:

```javascript
const stats = $input.item.json;
const openRate = stats.opens.open_rate * 100;
const totalRecipients = stats.emails_sent;
const uniqueOpens = stats.opens.unique_opens;
const unopenedCount = totalRecipients - uniqueOpens;

const shouldResend = openRate < 30 && unopenedCount > 100;

return [{
  json: {
    campaignId: stats.id,
    campaignTitle: stats.campaign_title,
    sentAt: stats.send_time,
    totalRecipients,
    uniqueOpens,
    unopenedCount,
    openRatePercent: openRate.toFixed(1),
    shouldResend,
    originalSubject: stats.subject_line
  }
}];
```

대안 제목 생성 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  이메일 마케팅 전문가야. 오픈율을 높이는 제목을 작성해.
  - 숫자나 질문 형식 활용
  - 15~40자 이내
  - 원본 제목과 톤은 유지하되 다른 각도에서 접근
  - JSON 배열로만 반환: ["제목1", "제목2", "제목3"]

User: |
  원본 제목: {{ $json.originalSubject }}
  캠페인 내용 요약: {{ $json.campaignTitle }}
  현재 오픈율: {{ $json.openRatePercent }}%

  재발송용 대안 제목 3개를 생성해줘.

Temperature: 0.8
```

재발송 캠페인 생성 설정:

```
[HTTP Request: Mailchimp 재발송 캠페인 생성]
  Method: POST
  URL: https://us1.api.mailchimp.com/3.0/campaigns
  Headers:
    Authorization: Basic {{ base64("anystring:" + MAILCHIMP_API_KEY) }}
    Content-Type: application/json
  Body:
    {
      "type": "regular",
      "recipients": {
        "list_id": "YOUR_LIST_ID",
        "segment_opts": {
          "conditions": [{
            "field": "campaign",
            "op": "notopen",
            "value": "{{ $json.campaignId }}"
          }]
        }
      },
      "settings": {
        "subject_line": "{{ $json.alternativeSubjects[0] }}",
        "from_name": "하모니 뉴스레터",
        "reply_to": "newsletter@harmonyshop.co.kr"
      }
    }
```

---

## 핵심 요약

- AI가 이메일 카테고리, 긴급도, 감정, 핵심 정보를 자동 추출
- 단순 FAQ는 즉시 자동 답변, 복잡한 건은 담당자 배정
- 고객 이력과 주문 정보를 컨텍스트로 활용한 개인화 답변
- 중요 이메일은 사람 검토 루프 포함하여 품질 보장
- 일일 통계로 성과 추적 및 지속 개선

**다음 레슨**: AI 고객 지원 봇을 구축합니다.
