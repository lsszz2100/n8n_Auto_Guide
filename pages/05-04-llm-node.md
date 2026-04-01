# 레슨 5.4: Basic LLM 노드와 Chain

---

## LLM Chain이란?

AI Agent가 자율적으로 판단하고 행동하는 "에이전트"라면, **LLM Chain**은 미리 정해진 순서로 AI를 활용하는 "파이프라인"입니다.

```
AI Agent:    목표를 주면 스스로 계획하고 실행
LLM Chain:   정해진 단계를 순서대로 실행 (예측 가능)

언제 Chain을 사용하나:
  - 항상 동일한 형태의 AI 처리가 필요할 때
  - 비용/속도를 예측 가능하게 관리하고 싶을 때
  - 단순한 텍스트 변환/생성 작업
```

---

## Basic LLM Chain 노드

### 개요

가장 기본적인 AI 노드입니다. 텍스트를 입력하면 AI가 응답을 생성합니다.

```
사용 사례:
  - 텍스트 번역
  - 내용 분류 (카테고리 지정)
  - 감정 분석
  - 형식 변환 (자유 텍스트 → JSON)
  - 간단한 콘텐츠 생성
```

### 설정 방법

```
1. 노드 추가: + → "Basic LLM Chain" 검색
2. Prompt 설정:
   - Prompt: AI에게 전달할 메시지
   - 표현식 사용: {{ $json.content }}
3. Chat Model 연결: 하단 "Add Chat Model" → 모델 선택
4. (선택) Output Parser: 출력 형식 지정
```

### 실전 예시: 이메일 감정 분석

**노드 설정:**
```
Prompt:
다음 이메일의 감정을 분석하세요.

이메일 내용:
{{ $json.email_body }}

다음 형식으로 JSON으로만 응답하세요:
{
  "sentiment": "positive/negative/neutral",
  "urgency": "high/medium/low",
  "summary": "한 문장 요약"
}
```

**입력:**
```
"배송이 3일이나 늦었는데 아무런 연락도 없었습니다.
정말 실망스럽습니다. 즉시 환불해 주세요."
```

**출력:**
```json
{
  "sentiment": "negative",
  "urgency": "high",
  "summary": "배송 지연으로 인한 즉각적인 환불 요청"
}
```

---

## Summarization Chain

### 개요

긴 텍스트를 요약하는 데 특화된 체인입니다. 특히 LLM의 컨텍스트 제한을 초과하는 긴 문서도 처리할 수 있습니다.

### 요약 전략 3가지

**1. Stuff (전체 처리)**
```
사용: 문서가 짧을 때 (4,000 토큰 이하)
방법: 전체 문서를 한 번에 AI에 전달
장점: 빠름, 문맥 유지
단점: 긴 문서 처리 불가
```

**2. Map-Reduce (분할 처리)**
```
사용: 문서가 길 때
방법:
  1. 문서를 청크로 분할
  2. 각 청크 개별 요약 (Map)
  3. 개별 요약을 다시 합쳐 최종 요약 (Reduce)
장점: 긴 문서 처리 가능
단점: 청크 경계에서 맥락 손실 가능
```

**3. Refine (점진적 개선)**
```
사용: 정확도가 중요할 때
방법:
  1. 첫 청크로 초기 요약 생성
  2. 다음 청크를 보며 요약을 점진적으로 개선
장점: 맥락 유지, 높은 정확도
단점: 가장 느리고 비쌈
```

### 설정 방법

```
1. 노드 추가: + → "Summarization Chain" 검색
2. 설정:
   - Type: Stuff / Map Reduce / Refine
   - Input: 요약할 텍스트 {{ $json.document }}
   - Chunk Size: 텍스트 분할 단위 (기본 3000 토큰)
3. Chat Model 연결
```

### 실전 예시: 긴 기사 요약

```
입력:
  긴 뉴스 기사 (3000단어)

Summarization Chain 설정:
  Type: Map Reduce
  Prompt Template:
    다음 내용을 핵심 포인트 3가지로 요약하세요.
    형식: 불릿 포인트

출력:
  • 삼성전자 4분기 매출 10% 감소, 반도체 부문 적자 전환
  • 2024년 AI 칩 사업 강화로 반등 기대
  • 주가는 실적 발표 후 3% 하락
```

---

## Q&A Chain

### 개요

문서나 데이터베이스를 기반으로 질의응답을 수행합니다. 특정 문서에 대한 질문에 그 문서의 내용만을 근거로 답변합니다.

```
일반 LLM:   학습된 전반적인 지식으로 답변 (환각 위험)
Q&A Chain:  제공된 문서만 참조해 답변 (근거 있는 답변)
```

### 사용 사례

```
- 회사 내부 문서에 대한 질문 응답
- 계약서/약관 내용 조회
- 제품 매뉴얼 기반 지원
- 법률 문서 분석
- 논문/보고서 내용 질의
```

### 설정 방법

**방법 1: 직접 문서 입력 (단순)**
```
1. Basic LLM Chain 노드 사용
2. Prompt:
   다음 문서를 참고해 질문에 답하세요.
   문서에 없는 내용은 "문서에서 찾을 수 없습니다"라고 답하세요.

   [문서]
   {{ $json.document }}

   [질문]
   {{ $json.question }}
```

**방법 2: 벡터 스토어 활용 (고급)**
```
사전 작업:
  문서들 → [임베딩 변환] → [벡터 DB 저장]

질의 시:
  질문 → [유사 청크 검색] → [관련 내용만 LLM에 전달] → 답변
```

### 실전 예시: FAQ 봇

```
문서 (FAQ DB):
  Q: 배송 기간은 얼마나 걸리나요?
  A: 일반 배송 3~5일, 빠른 배송 1~2일입니다.

  Q: 환불 정책은 어떻게 되나요?
  A: 수령 후 7일 이내 무조건 환불 가능합니다.

사용자 질문: "환불하려면 어떻게 해야 해요?"

Q&A Chain 응답:
  수령 후 7일 이내에 환불 신청이 가능합니다.
  환불 신청은 마이페이지 → 주문 내역 → 환불 신청에서 하실 수 있습니다.
  (마이페이지 방법은 제공된 FAQ에 없어 이 부분은 제외됨)
```

---

## 실전 예시: 이메일 요약 자동화

이메일을 받으면 자동으로 요약하고 슬랙으로 보내는 워크플로우입니다.

### 워크플로우 구조

```
[Gmail 트리거 - 새 이메일 수신]
           ↓
[Basic LLM Chain - 중요도 분류]
           ↓
     [Switch 노드]
    ↙           ↘
[중요] → [Summarization Chain - 요약]
                   ↓
          [Slack - 팀에 알림]
    ↘
[일반] → [자동 라벨 적용]
```

### Step 1: Gmail 트리거

```
트리거 설정:
  Label/Mailbox: INBOX
  Mark as Read: false (읽음 처리 X, 직접 확인 후 처리)
```

### Step 2: 중요도 분류 (Basic LLM Chain)

```
Prompt:
다음 이메일의 중요도를 분류하세요.

발신자: {{ $json.from }}
제목: {{ $json.subject }}
본문 첫 200자: {{ $json.snippet }}

"urgent", "important", "normal" 중 하나만 응답하세요.
```

### Step 3: 요약 생성 (Summarization Chain - 중요 이메일만)

```
Type: Stuff
Prompt Template:
  다음 이메일을 3줄로 요약하세요.
  - 요청 사항이 있다면 반드시 포함
  - 데드라인이 있다면 반드시 포함
  - 발신자 이름 포함

Input: {{ $json.body }}
```

### Step 4: Slack 알림

```
Channel: #urgent-emails
Message:
  *긴급 이메일 알림*

  *발신자:* {{ $('Gmail Trigger').item.json.from }}
  *제목:* {{ $('Gmail Trigger').item.json.subject }}

  *요약:*
  {{ $json.output }}

  <{{ $('Gmail Trigger').item.json.threadId }}|원문 보기>
```

---

## Output Parser로 구조화된 출력

Basic LLM Chain의 출력을 항상 JSON으로 받고 싶다면 Output Parser를 사용합니다.

```
노드 추가: + → "Structured Output Parser" 또는 "JSON Output Parser"

설정:
  JSON Schema:
  {
    "type": "object",
    "properties": {
      "category": {"type": "string"},
      "priority": {"type": "number", "minimum": 1, "maximum": 5},
      "action_required": {"type": "boolean"},
      "summary": {"type": "string"}
    }
  }
```

이렇게 하면 AI 응답이 항상 지정한 JSON 스키마로 반환됩니다. 다음 노드에서 `{{ $json.category }}` 처럼 바로 사용 가능합니다.

---

## 실전 예제 모음

---

### 예제 1: LLM Chain으로 상품 설명 자동 생성 (쇼핑몰 운영자용)

쇼핑몰에 신규 상품을 등록할 때, 상품명과 스펙만 입력하면 SEO에 최적화된 상품 설명을 자동으로 생성하는 워크플로우입니다.

워크플로우 구조:

```
[Google Sheets 트리거 - 새 상품 행 감지]
            ↓
[Basic LLM Chain - 상품 설명 생성]
            ↓
[Code 노드 - 출력 파싱]
            ↓
[Google Sheets - 설명 컬럼에 자동 기입]
```

Google Sheets 컬럼 구성:

```
A열: 상품명       예) 프리미엄 무선 이어폰 X100
B열: 핵심 스펙    예) 노이즈 캔슬링, 배터리 30시간, 블루투스 5.3
C열: 가격대       예) 89,000원
D열: 타겟 고객    예) 직장인, 운동 마니아
E열: 상품 설명    ← 자동 생성될 컬럼
```

Basic LLM Chain 프롬프트 설정:

```
당신은 대한민국 쇼핑몰의 전문 상품 기획자입니다.
아래 상품 정보를 바탕으로 고객이 구매하고 싶어지는 매력적인 상품 설명을 작성하세요.

상품명: {{ $json.product_name }}
핵심 스펙: {{ $json.specs }}
가격: {{ $json.price }}
타겟 고객: {{ $json.target }}

작성 규칙:
- 전체 300자 이내
- 첫 문장은 핵심 혜택을 강조하는 후킹 문장
- 스펙은 고객 혜택 중심으로 풀어서 설명 (예: "30시간 배터리" → "하루 종일 끊김 없이 사용")
- 마지막 문장은 구매를 유도하는 call-to-action
- 순수 텍스트로만 출력, 마크다운 기호 없이
```

Chat Model 설정:

```
Model: gpt-4o-mini
Temperature: 0.6
Max Tokens: 500
```

실제 출력 예시:

```
하루 종일 완벽한 음질을 경험하세요. 프리미엄 노이즈 캔슬링으로
지하철 소음도, 사무실 잡음도 완벽히 차단됩니다. 블루투스 5.3으로
끊김 없이 연결되고, 30시간 배터리로 출근부터 퇴근 후 운동까지
충전 걱정 없이 사용할 수 있습니다. 직장인과 운동 마니아 모두에게
최적화된 X100, 지금 바로 경험해보세요.
```

Code 노드에서 출력 정리:

```javascript
// LLM 출력에서 앞뒤 공백 제거 후 Sheets에 기입할 형태로 반환
const description = $json.output.trim();
return [{
  json: {
    row_number: $('Google Sheets Trigger').item.json.row,
    description: description
  }
}];
```

---

### 예제 2: 고객 후기 감정 분석 (긍정/부정/중립 + 점수)

쇼핑몰에 쌓이는 고객 후기를 자동으로 분석하여 감정 유형, 점수, 핵심 키워드를 추출합니다. 하루에 한 번 실행하여 Google Sheets에 분석 결과를 누적합니다.

워크플로우 구조:

```
[Schedule 트리거 - 매일 오전 8시]
            ↓
[HTTP Request - 후기 API 호출]
            ↓
[Split In Batches - 후기 10개씩 분리]
            ↓
[Basic LLM Chain - 감정 분석]
            ↓
[JSON Output Parser - 구조화]
            ↓
[Google Sheets - 분석 결과 저장]
```

Basic LLM Chain 프롬프트:

```
당신은 소비자 심리 전문 분석가입니다.
아래 고객 후기를 분석하여 정확히 지정된 JSON 형식으로만 응답하세요.

후기 내용:
{{ $json.review_text }}

별점: {{ $json.rating }}점 (5점 만점)

분석 기준:
- sentiment: 후기의 전체적인 감정 (positive / negative / neutral)
- score: 감정 강도를 -100(매우 부정)에서 100(매우 긍정) 사이의 숫자로 표현
- keywords: 후기에서 핵심 키워드 최대 3개 추출
- complaint: 불만 사항이 있다면 한 문장으로 요약, 없으면 null
- praise: 칭찬 내용이 있다면 한 문장으로 요약, 없으면 null
- action_needed: CS 팀의 즉각 대응이 필요한지 여부 (true/false)

응답 형식 (다른 텍스트 없이 JSON만):
{
  "sentiment": "positive",
  "score": 75,
  "keywords": ["배송", "포장", "품질"],
  "complaint": null,
  "praise": "빠른 배송과 꼼꼼한 포장이 인상적이었습니다",
  "action_needed": false
}
```

Structured Output Parser 스키마:

```json
{
  "type": "object",
  "properties": {
    "sentiment": { "type": "string", "enum": ["positive", "negative", "neutral"] },
    "score": { "type": "number", "minimum": -100, "maximum": 100 },
    "keywords": { "type": "array", "items": { "type": "string" } },
    "complaint": { "type": ["string", "null"] },
    "praise": { "type": ["string", "null"] },
    "action_needed": { "type": "boolean" }
  },
  "required": ["sentiment", "score", "keywords", "action_needed"]
}
```

Google Sheets 저장 설정:

```
시트명: 후기분석_{{ $now.format('YYYY-MM') }}

컬럼 매핑:
  A열: 후기 ID      → {{ $('HTTP Request').item.json.review_id }}
  B열: 작성일       → {{ $('HTTP Request').item.json.created_at }}
  C열: 감정         → {{ $json.sentiment }}
  D열: 점수         → {{ $json.score }}
  E열: 키워드       → {{ $json.keywords.join(', ') }}
  F열: 불만 요약    → {{ $json.complaint ?? '' }}
  G열: 칭찬 요약    → {{ $json.praise ?? '' }}
  H열: 즉시대응필요 → {{ $json.action_needed ? 'Y' : 'N' }}
```

action_needed가 true인 경우 Slack 알림 추가:

```
[IF 노드] action_needed === true
    ↓
[Slack]
  Channel: #cs-urgent
  Message: |
    CS 즉각 대응 필요 후기가 접수되었습니다.

    후기 ID: {{ $('HTTP Request').item.json.review_id }}
    감정 점수: {{ $json.score }}
    불만 내용: {{ $json.complaint }}

    Sheets에서 확인: [링크]
```

---

### 예제 3: 계약서 핵심 조항 자동 추출

PDF나 텍스트로 된 계약서를 입력하면, 검토가 필요한 핵심 조항을 자동으로 추출하고 리스크를 평가합니다. 법무팀의 1차 검토 시간을 단축하는 용도입니다.

워크플로우 구조:

```
[Webhook - 계약서 파일 수신]
            ↓
[HTTP Request - PDF 텍스트 추출 API]
            ↓
[Summarization Chain - 전체 요약 (Map-Reduce)]
            ↓
[Basic LLM Chain - 핵심 조항 추출]
            ↓
[Basic LLM Chain - 리스크 평가]
            ↓
[Gmail - 법무팀에 분석 결과 발송]
```

Summarization Chain 설정:

```
Type: Map Reduce
Chunk Size: 3000
Chunk Overlap: 200

Prompt Template:
  다음 계약서 내용을 요약하세요.
  계약의 핵심 의무사항, 금액, 기간, 조건을 중심으로 요약하세요.
  {{ text }}
```

핵심 조항 추출 - Basic LLM Chain 프롬프트:

```
당신은 10년 경력의 계약법 전문 변호사입니다.
아래 계약서 요약을 검토하고, 반드시 확인해야 할 핵심 조항을 추출하세요.

계약서 요약:
{{ $json.output }}

다음 카테고리별로 조항을 추출하세요. 해당 내용이 없으면 "명시 없음"으로 기재하세요:

1. 계약 기간 및 갱신 조건
2. 대금 지급 조건 및 지연 이자
3. 계약 해지 사유 및 절차
4. 손해배상 범위와 한도
5. 분쟁 해결 방법 (중재/소송, 관할 법원)
6. 지식재산권 귀속
7. 비밀유지 의무 범위와 기간
8. 불가항력 조항

응답 형식:
각 항목을 번호와 함께 나열하고, 해당 조항의 원문에서 핵심 문구를 인용하세요.
```

리스크 평가 - Basic LLM Chain 프롬프트:

```
앞서 추출한 계약서 핵심 조항을 바탕으로 리스크를 평가하세요.

추출된 조항:
{{ $json.output }}

평가 항목:
- 전체 리스크 등급: 상/중/하
- 즉시 재협상이 필요한 조항 목록
- 일반적인 계약 관행과 다른 이상한 조항
- 누락된 중요 조항 (있어야 하는데 없는 것)
- 법무팀에 전달할 주요 코멘트 (3줄 이내)

형식: 각 항목을 구분하여 명확하게 작성하세요.
```

Gmail 발송 설정:

```
To: legal@company.com
Subject: [계약서 1차 분석 완료] {{ $('Webhook').item.json.contract_name }}
Body:
계약서 자동 분석 결과를 전달드립니다.

계약서명: {{ $('Webhook').item.json.contract_name }}
분석 일시: {{ $now.format('YYYY-MM-DD HH:mm') }}
리스크 등급: {{ $json.risk_level }}

[핵심 조항 추출]
{{ $('핵심 조항 추출').item.json.output }}

[리스크 평가]
{{ $json.output }}

이 분석은 AI가 생성한 1차 검토 결과입니다.
최종 판단은 반드시 담당 법무사/변호사가 원문을 확인하여 결정하시기 바랍니다.
```

---

## 핵심 요약

- Basic LLM Chain: 단순한 AI 호출, 텍스트 변환/분류/생성
- Summarization Chain: 긴 문서 요약, Stuff/Map-Reduce/Refine 전략 선택
- Q&A Chain: 특정 문서 기반 질의응답, 환각 방지
- Output Parser로 AI 출력을 항상 일관된 구조로 받기
- LLM Chain은 예측 가능한 파이프라인, Agent는 자율적인 처리

다음 레슨: AI의 성능을 극대화하는 프롬프팅 전략을 배웁니다.
