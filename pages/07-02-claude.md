# 레슨 7.2: Anthropic Claude 연동하기

---

## Claude가 특별한 이유

Anthropic의 Claude는 GPT와 함께 2026년 가장 많이 사용되는 AI 모델입니다.

**Claude의 강점:**
- **긴 컨텍스트**: Claude 4 기준 200K 토큰 (약 책 150권 분량)
- **정교한 추론**: 복잡한 분석과 단계적 사고에 강함
- **안전성**: Anthropic의 Constitutional AI로 안전하고 신뢰할 수 있는 응답
- **코드 작성**: 코딩 작업에서 탁월한 성능
- **한국어 지원**: 한국어 이해 및 생성 품질 우수

---

## 2026년 Claude 모델 라인업

| 모델 | 컨텍스트 | 비용 (Input/Output) | 특징 |
|------|----------|---------------------|------|
| claude-opus-4 | 200K | $15/$75 per 1M | 최고 성능 |
| claude-sonnet-4 | 200K | $3/$15 per 1M | 균형 잡힌 성능/비용 |
| claude-haiku-4 | 200K | $0.8/$4 per 1M | 빠르고 저렴 |

---

## API 키 발급

1. `console.anthropic.com` 접속
2. API Keys 메뉴 → "Create Key"
3. 키 이름 설정 후 생성
4. 생성된 키 즉시 복사 (이후 재확인 불가!)

### n8n 크리덴셜 설정

Credentials → Anthropic → API Key 입력

---

## n8n에서 Claude 노드 사용

n8n 1.x 이상에서 Anthropic 노드가 네이티브 지원됩니다:

```
[트리거]
    ↓
[Anthropic: Message Claude]
설정:
  Model: claude-sonnet-4-5
  System: 너는 한국어 법률 문서 분석 전문가야. 명확하고 간결하게 답변해.
  Messages:
    Human: {{ $json.contractText }}를 분석하고 주요 리스크를 3가지로 요약해줘
  Max Tokens: 1000
  Temperature: 0.3
```

---

## HTTP Request로 Claude API 직접 호출

더 세밀한 제어가 필요할 때:

```
[HTTP Request]
설정:
  Method: POST
  URL: https://api.anthropic.com/v1/messages
  Headers:
    x-api-key: {{ $credentials.anthropicApiKey }}
    anthropic-version: 2023-06-01
    content-type: application/json
  Body (JSON):
    {
      "model": "claude-sonnet-4-5",
      "max_tokens": 1024,
      "system": "{{ $json.systemPrompt }}",
      "messages": [
        {
          "role": "user",
          "content": "{{ $json.userMessage }}"
        }
      ]
    }
```

---

## 멀티턴 대화 구성

Claude와 대화 형식으로 소통할 때:

```javascript
// Code 노드에서 대화 기록 구성
const conversationHistory = $json.history || [];

// 새 사용자 메시지 추가
conversationHistory.push({
  role: "user",
  content: $json.newMessage
});

return [{
  json: {
    messages: conversationHistory,
    systemPrompt: "너는 n8n 자동화 전문가야."
  }
}];

// HTTP Request로 Claude 호출 후
// 응답을 history에 추가하여 다음 대화에 사용
```

---

## Claude의 고급 기능

### Extended Thinking (확장 사고)

복잡한 문제에서 Claude가 단계적으로 생각하도록:

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [
    {
      "role": "user",
      "content": "이 비즈니스 전략의 장단점을 심층 분석해줘..."
    }
  ]
}
```

### 비전 (이미지 분석)

```json
{
  "model": "claude-sonnet-4-5",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "{{ $binary.data }}"
          }
        },
        {
          "type": "text",
          "text": "이 이미지의 텍스트를 모두 추출해줘"
        }
      ]
    }
  ]
}
```

---

## GPT vs Claude 선택 가이드

| 작업 유형 | 추천 모델 | 이유 |
|-----------|-----------|------|
| 긴 문서 분석 | Claude Sonnet | 200K 컨텍스트 |
| 법률/계약서 검토 | Claude Opus | 정교한 추론 |
| 빠른 분류/추출 | GPT-4o-mini / Claude Haiku | 저렴하고 빠름 |
| 코드 작성 | Claude Sonnet | 코딩 성능 우수 |
| 창의적 콘텐츠 | GPT-4o / Claude Sonnet | 비슷한 수준 |
| 비용 절감 우선 | Claude Haiku | 가장 저렴 |

---

## Claude Tool Use (도구 사용)

Claude가 외부 함수를 직접 호출할 수 있도록 도구를 정의하는 기능입니다. n8n에서는 HTTP Request로 직접 구현합니다.

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 4096,
  "tools": [
    {
      "name": "get_weather",
      "description": "특정 도시의 현재 날씨를 조회합니다",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "description": "날씨를 조회할 도시명 (예: 서울, 부산)"
          },
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "description": "온도 단위"
          }
        },
        "required": ["city"]
      }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "오늘 서울 날씨 어때?"
    }
  ]
}
```

Tool Use 응답 처리 패턴 (n8n Code 노드):

```javascript
const response = $input.item.json;

// Claude가 tool_use를 요청하는 경우
if (response.stop_reason === 'tool_use') {
  const toolUse = response.content.find(c => c.type === 'tool_use');
  const toolName = toolUse.name;
  const toolInput = toolUse.input;

  // 도구 실행 (예: 날씨 API 호출)
  // 이후 HTTP Request 노드에서 실제 API 호출

  return [{
    json: {
      toolName,
      toolInput,
      toolUseId: toolUse.id,
      previousMessages: response.content
    }
  }];
}

// 일반 텍스트 응답
const textContent = response.content.find(c => c.type === 'text');
return [{ json: { reply: textContent?.text ?? '' } }];
```

도구 실행 결과를 Claude에게 전달:

```json
{
  "model": "claude-sonnet-4-6",
  "messages": [
    { "role": "user", "content": "오늘 서울 날씨 어때?" },
    { "role": "assistant", "content": [{ "type": "tool_use", "id": "tool_123", "name": "get_weather", "input": {"city": "서울"} }] },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "tool_123",
          "content": "{\"temperature\": 18, \"description\": \"맑음\", \"humidity\": 45}"
        }
      ]
    }
  ]
}
```

---

## 비용 최적화

### Prompt Caching 활용

긴 시스템 프롬프트나 문서를 반복 사용할 때 캐싱으로 비용 절감:

```json
{
  "model": "claude-sonnet-4-6",
  "system": [
    {
      "type": "text",
      "text": "당신은 법률 문서 분석 전문가입니다...\n[5000자 분량의 법률 가이드라인]",
      "cache_control": { "type": "ephemeral" }
    }
  ],
  "messages": [
    { "role": "user", "content": "{{ $json.question }}" }
  ]
}
```

캐싱 비용 비교:
```
캐싱 없이 5000토큰 시스템 프롬프트 100회 호출:
  500,000 토큰 × $3/1M = $1.50

캐싱 사용 (첫 번째 호출은 캐시 저장 비용 1.25배, 이후는 0.1배):
  1회 캐시 저장: 5,000 × 1.25 = 6,250 토큰 비용
  99회 캐시 적중: 5,000 × 0.1 × 99 = 49,500 토큰 비용
  = 약 55,750 토큰 → $1.50 대비 90% 절감
```

### 모델 선택 전략

```javascript
// Code 노드: 작업 복잡도에 따라 모델 자동 선택
const taskType = $json.taskType;
const contentLength = ($json.content ?? '').length;

let model;

if (taskType === 'classification' || contentLength < 500) {
  model = 'claude-haiku-4-5-20251001';   // 빠르고 저렴
} else if (taskType === 'analysis' || contentLength < 10000) {
  model = 'claude-sonnet-4-6';           // 균형
} else {
  model = 'claude-opus-4-6';             // 최고 성능
}

return [{ json: { ...($json), model } }];
```

---

## 에러 처리

Claude API 응답 상태 코드별 대응:

```javascript
// Code 노드: HTTP Request 오류 처리
const statusCode = $input.item.json.statusCode;
const errorBody = $input.item.json.body;

const errorMap = {
  400: '요청 형식 오류 - 메시지 구조 확인 필요',
  401: 'API 키 인증 실패 - 크리덴셜 재확인',
  403: '사용 권한 없음 - 요금제 확인',
  429: 'API 호출 한도 초과 - 잠시 후 재시도',
  500: 'Anthropic 서버 오류 - 재시도 또는 대기',
  529: 'API 과부하 - 지수 백오프로 재시도'
};

if (statusCode && statusCode >= 400) {
  const reason = errorMap[statusCode] ?? '알 수 없는 오류';
  throw new Error(`Claude API 오류 (${statusCode}): ${reason}`);
}

return [$input.item];
```

429 재시도 패턴 (Rate Limit):

```
[HTTP Request: Claude API]
  (에러 발생 시)
    ↓
[IF: 상태코드 == 429?]
  True: → [Wait: 60초]
           ↓
        [HTTP Request: Claude API 재시도]
  False: → [Error 처리]
```

---

## 실전 예제 모음

---

### 예제 1: 긴 PDF 계약서 자동 분석 리포트 생성

Claude의 200K 컨텍스트를 활용하여 수십 페이지 분량의 계약서를 한 번에 분석합니다. 핵심 의무, 리스크, 협상 포인트를 추출하고 Notion에 구조화된 리포트를 자동 저장합니다.

워크플로우 구조:

```
[Webhook: POST /analyze-contract]
  Body: { fileUrl, contractType, partyName }
    ↓
[HTTP Request: PDF 다운로드]
  URL: {{ $json.fileUrl }}
  Response Format: File (Binary)
    ↓
[Code: PDF 바이너리 → Base64 텍스트 변환]
    ↓
[HTTP Request: Claude Opus API]
  긴 컨텍스트 활용 전체 계약서 분석
    ↓
[Code: JSON 분석 결과 파싱]
    ↓
[Notion: 분석 리포트 페이지 자동 생성]
    ↓
[Gmail: 검토 완료 알림 + Notion 링크 발송]
```

HTTP Request 노드 - Claude Opus API 호출:

```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 4096,
  "system": "당신은 기업 법무팀 소속 계약서 전문 변호사입니다. 의뢰 기업의 관점에서 계약서를 분석하고, 리스크와 협상 포인트를 명확히 지적합니다. 반드시 JSON 형식으로만 답변하세요.",
  "messages": [
    {
      "role": "user",
      "content": "계약 유형: {{ $json.contractType }}\n당사자: {{ $json.partyName }}\n\n아래 계약서를 분석하여 다음 JSON 형식으로 정확히 반환하세요:\n{\n  \"summary\": \"계약 핵심 내용 3문장\",\n  \"obligations\": [\"의무사항1\", \"의무사항2\", \"의무사항3\"],\n  \"risks\": [\n    {\"item\": \"리스크 내용\", \"severity\": \"high|medium|low\", \"reason\": \"이유\"}\n  ],\n  \"negotiation_points\": [\"협상포인트1\", \"협상포인트2\"],\n  \"recommendation\": \"최종 검토 의견\"\n}\n\n계약서 전문:\n{{ $json.contractText }}"
    }
  ]
}
```

Code 노드 - 분석 결과 파싱:

```javascript
const rawContent = $input.item.json.content[0].text;

let analysis;
try {
  // JSON 블록 추출 (```json ... ``` 형태도 처리)
  const jsonMatch = rawContent.match(/\{[\s\S]*\}/);
  analysis = JSON.parse(jsonMatch ? jsonMatch[0] : rawContent);
} catch {
  throw new Error('Claude 응답 파싱 실패: ' + rawContent.slice(0, 200));
}

// 리스크를 심각도 순으로 정렬
const severityOrder = { high: 0, medium: 1, low: 2 };
analysis.risks.sort((a, b) => severityOrder[a.severity] - severityOrder[b.severity]);

// Notion용 블록 생성
const riskBlocks = analysis.risks.map(r => ({
  type: 'bulleted_list_item',
  bulleted_list_item: {
    rich_text: [{
      type: 'text',
      text: { content: `[${r.severity.toUpperCase()}] ${r.item} - ${r.reason}` }
    }]
  }
}));

return [{
  json: {
    ...analysis,
    riskBlocks,
    analyzedAt: new Date().toISOString(),
    contractType: $('Webhook').item.json.contractType
  }
}];
```

Notion 페이지 생성 노드:

```
[Notion: Create Page]
  Database: {계약서 분석 DB}
  Properties:
    이름 (title): [{ "text": { "content": "{{ $json.contractType }} 분석 리포트 - {{ $json.analyzedAt.slice(0,10) }}" } }]
    상태 (select): { "name": "분석완료" }
    심각도 최고 리스크 (select): { "name": "{{ $json.risks[0]?.severity ?? 'low' }}" }
    분석일 (date): { "start": "{{ $json.analyzedAt.slice(0,10) }}" }
  Children:
    [heading_2: 계약 요약]
    [paragraph: {{ $json.summary }}]
    [heading_2: 핵심 의무사항]
    [bulleted_list: {{ $json.obligations }}]
    [heading_2: 리스크 분석]
    {{ $json.riskBlocks }}
    [heading_2: 협상 포인트]
    [bulleted_list: {{ $json.negotiation_points }}]
    [heading_2: 최종 검토 의견]
    [paragraph: {{ $json.recommendation }}]
```

---

### 예제 2: 고객 리뷰 대량 감정 분석 + 인사이트 도출

수백 개의 고객 리뷰를 Claude Haiku로 빠르고 저렴하게 감정 분류하고, 분류된 결과를 Claude Sonnet으로 종합 분석하여 제품 개선 인사이트를 도출합니다.

워크플로우 구조:

```
[Schedule: 매주 월요일 오전 8시]
    ↓
[Google Sheets: 지난 주 신규 리뷰 읽기]
  Filter: analyzed = FALSE
    ↓
[SplitInBatches: 10개씩 배치]
    ↓
[HTTP Request: Claude Haiku - 배치 감정 분류]
  10개 리뷰를 한 번에 분류 (비용 절감)
    ↓
[Code: 분류 결과 파싱 + 시트 업데이트]
    ↓
[Google Sheets: analyzed = TRUE로 표시]
    ↓
[배치 루프 종료 후]
    ↓
[Google Sheets: 전체 분류 결과 집계]
    ↓
[HTTP Request: Claude Sonnet - 종합 인사이트 분석]
    ↓
[Slack: #product 채널에 주간 리뷰 인사이트 발송]
    ↓
[Notion: 주간 리뷰 분석 리포트 페이지 생성]
```

HTTP Request 노드 - Claude Haiku 배치 분류:

```javascript
// Code 노드: 배치 요청 본문 구성
const reviews = $input.all().map((item, i) => `${i + 1}. ${item.json.reviewText}`).join('\n');

const requestBody = {
  model: 'claude-haiku-4-5-20251001',
  max_tokens: 1024,
  system: '당신은 고객 리뷰 감정 분석 전문가입니다. 주어진 리뷰 목록을 분석하여 JSON 배열로만 응답합니다.',
  messages: [{
    role: 'user',
    content: `아래 리뷰 ${$input.all().length}개를 분석하여 JSON 배열로 반환하세요.\n각 항목: { "index": 번호, "sentiment": "positive|negative|neutral", "score": 1-5, "keywords": ["키워드1", "키워드2"], "category": "품질|배송|가격|서비스|기타" }\n\n리뷰 목록:\n${reviews}`
  }]
};

return [{ json: requestBody }];
```

Code 노드 - 분류 결과 파싱:

```javascript
const rawText = $input.item.json.content[0].text;
const jsonMatch = rawText.match(/\[[\s\S]*\]/);

if (!jsonMatch) {
  throw new Error('Claude 배치 분류 파싱 실패');
}

const results = JSON.parse(jsonMatch[0]);
const originalItems = $('SplitInBatches').all();

// 각 결과를 원본 리뷰와 매핑
return results.map(result => {
  const original = originalItems[result.index - 1]?.json ?? {};
  return {
    json: {
      reviewId: original.reviewId,
      reviewText: original.reviewText,
      sentiment: result.sentiment,
      score: result.score,
      keywords: result.keywords.join(', '),
      category: result.category,
      analyzedAt: new Date().toISOString()
    }
  };
});
```

HTTP Request 노드 - Claude Sonnet 종합 인사이트:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 2048,
  "system": "당신은 제품 매니저입니다. 고객 리뷰 분석 데이터를 보고 실행 가능한 인사이트와 개선 방향을 제시합니다.",
  "messages": [{
    "role": "user",
    "content": "이번 주 고객 리뷰 {{ $json.totalCount }}개 분석 결과:\n긍정: {{ $json.positiveCount }}개 ({{ $json.positiveRate }}%)\n부정: {{ $json.negativeCount }}개\n중립: {{ $json.neutralCount }}개\n\n카테고리별 부정 리뷰 비율:\n{{ $json.categoryBreakdown }}\n\n자주 언급된 부정 키워드:\n{{ $json.negativeKeywords }}\n\n실행 가능한 개선 방향 3가지와 각 우선순위(높음/중간/낮음)를 제시하세요."
  }]
}
```

---

### 예제 3: 코드 리뷰 자동화 (GitHub PR → Claude 분석 → 리뷰 코멘트)

GitHub Pull Request가 열릴 때 변경된 코드를 Claude Sonnet으로 자동 리뷰합니다. 버그 가능성, 보안 취약점, 성능 문제, 코드 스타일을 분석하고 PR에 직접 코멘트를 달아줍니다.

워크플로우 구조:

```
[Webhook: GitHub PR opened/synchronize]
    ↓
[Code: 서명 검증 + PR 정보 파싱]
    ↓
[HTTP Request: GitHub API - diff 가져오기]
  GET /repos/{owner}/{repo}/pulls/{number}/files
    ↓
[Code: diff 파싱 + 리뷰할 파일 선별]
  (자동 생성 파일, lock 파일 등 제외)
    ↓
[HTTP Request: Claude Sonnet - 코드 리뷰]
    ↓
[Code: 리뷰 결과 파싱 + PR 코멘트 형식 변환]
    ↓
[HTTP Request: GitHub API - PR 코멘트 게시]
    ↓
[Respond to Webhook: 200 OK]
```

Code 노드 - 리뷰할 파일 선별:

```javascript
const files = $input.all();

// 리뷰 제외 파일 패턴
const skipPatterns = [
  /package-lock\.json$/,
  /yarn\.lock$/,
  /\.min\.(js|css)$/,
  /dist\//,
  /build\//,
  /node_modules\//,
  /\.generated\./
];

const reviewableFiles = files.filter(f => {
  const filename = f.json.filename;
  return !skipPatterns.some(pattern => pattern.test(filename));
}).map(f => ({
  filename: f.json.filename,
  status: f.json.status,        // added, modified, removed
  additions: f.json.additions,
  deletions: f.json.deletions,
  patch: f.json.patch ?? ''     // 실제 diff 내용
}));

// 변경량이 너무 큰 파일(500줄 이상)은 요약 경고만
const largeFiles = reviewableFiles.filter(f => f.additions + f.deletions > 500);
const normalFiles = reviewableFiles.filter(f => f.additions + f.deletions <= 500);

// Claude에 전달할 diff 텍스트 생성
const diffText = normalFiles.map(f =>
  `=== ${f.filename} (${f.status}, +${f.additions}/-${f.deletions}) ===\n${f.patch}`
).join('\n\n');

return [{
  json: {
    diffText: diffText.slice(0, 30000),  // 토큰 한도 고려
    largeFiles: largeFiles.map(f => f.filename),
    reviewFileCount: normalFiles.length
  }
}];
```

HTTP Request 노드 - Claude 코드 리뷰:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 3000,
  "system": "당신은 시니어 소프트웨어 엔지니어로서 코드 리뷰를 수행합니다. 건설적이고 구체적인 피드백을 제공하며, 심각도(critical/major/minor/suggestion)를 표시합니다.",
  "messages": [{
    "role": "user",
    "content": "아래 코드 변경사항을 리뷰해주세요.\n\n다음 JSON 형식으로만 응답하세요:\n{\n  \"overall_assessment\": \"LGTM|CHANGES_REQUESTED|COMMENT\",\n  \"summary\": \"전체 변경사항 요약 2-3문장\",\n  \"issues\": [\n    {\n      \"severity\": \"critical|major|minor|suggestion\",\n      \"file\": \"파일명\",\n      \"description\": \"문제 설명\",\n      \"suggestion\": \"개선 방법\"\n    }\n  ],\n  \"positives\": [\"잘 된 점1\", \"잘 된 점2\"]\n}\n\n변경 코드:\n{{ $json.diffText }}"
  }]
}
```

Code 노드 - GitHub 코멘트 형식 변환:

```javascript
const rawText = $input.item.json.content[0].text;
const prInfo = $('Code: 서명 검증').item.json;
const largeFiles = $('Code: 파일 선별').item.json.largeFiles;

let review;
try {
  const jsonMatch = rawText.match(/\{[\s\S]*\}/);
  review = JSON.parse(jsonMatch[0]);
} catch {
  review = { overall_assessment: 'COMMENT', summary: rawText, issues: [], positives: [] };
}

// GitHub PR 코멘트 마크다운 생성
const severityEmoji = { critical: '🚨', major: '⚠️', minor: '💡', suggestion: '✨' };

const issueLines = review.issues.map(issue =>
  `${severityEmoji[issue.severity] ?? '•'} **[${issue.severity.toUpperCase()}]** \`${issue.file}\`\n  ${issue.description}\n  > 개선: ${issue.suggestion}`
).join('\n\n');

const positivesLines = review.positives.map(p => `- ${p}`).join('\n');

const largeFileWarning = largeFiles.length > 0
  ? `\n\n> 아래 파일은 변경량이 많아 자동 리뷰에서 제외되었습니다: ${largeFiles.join(', ')}`
  : '';

const commentBody = [
  `## 🤖 Claude Code Review`,
  ``,
  `**전체 평가:** ${review.overall_assessment}`,
  ``,
  `### 요약`,
  review.summary,
  ``,
  review.issues.length > 0 ? `### 발견된 이슈 (${review.issues.length}건)\n\n${issueLines}` : '### 발견된 이슈\n이슈 없음 ✅',
  ``,
  review.positives.length > 0 ? `### 잘 된 점\n${positivesLines}` : '',
  largeFileWarning
].filter(Boolean).join('\n');

return [{
  json: {
    owner: prInfo.repoOwner,
    repo: prInfo.repoName,
    prNumber: prInfo.prNumber,
    commentBody,
    overallAssessment: review.overall_assessment
  }
}];
```

GitHub PR 코멘트 게시 노드:

```
[HTTP Request: GitHub PR 코멘트]
  Method: POST
  URL: https://api.github.com/repos/{{ $json.owner }}/{{ $json.repo }}/pulls/{{ $json.prNumber }}/reviews
  Headers:
    Authorization: Bearer {GITHUB_TOKEN}
    Accept: application/vnd.github+json
  Body (JSON):
    {
      "body": "{{ $json.commentBody }}",
      "event": "{{ $json.overallAssessment === 'CHANGES_REQUESTED' ? 'REQUEST_CHANGES' : 'COMMENT' }}"
    }
```

---

## 핵심 요약

- Claude는 GPT의 강력한 대안으로 특히 긴 문서, 추론, 코딩에 강점
- 2026년 기준: Opus 4.6(최고), Sonnet 4.6(균형), Haiku 4.5(저렴)
- Anthropic 노드 또는 HTTP Request로 직접 API 호출 가능
- Tool Use로 Claude가 외부 함수를 직접 호출하도록 설정 가능
- Prompt Caching으로 반복되는 긴 시스템 프롬프트 비용 90% 절감
- 작업 복잡도별 모델 선택 전략으로 비용 최적화
- Extended Thinking으로 복잡한 문제 단계적 해결

다음 레슨: AI 이메일 자동 분류 및 응답 시스템을 완성합니다.
