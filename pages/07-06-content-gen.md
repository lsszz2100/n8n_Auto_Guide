# 레슨 7.6: AI 콘텐츠 생성 자동화

---

## 콘텐츠 생성 자동화의 가치

콘텐츠 팀이 주간 블로그 1편을 만드는 데 걸리는 시간: **8-16시간**

n8n + AI로 자동화하면: **아이디어 → 발행까지 30분**

---

## 완성 시스템 개요

```
[콘텐츠 아이디어 입력 (Notion/폼)]
    ↓
[AI: 키워드 리서치]
    ↓
[AI: 구조 설계 (개요)]
    ↓
[AI: 전체 초안 작성]
    ↓
[AI: SEO 최적화]
    ↓
[AI: 이미지 프롬프트 생성]
    ↓
[DALL-E: 이미지 생성]
    ↓
[Wordpress/Ghost: 자동 업로드]
    ↓
[소셜 미디어: 동시 발행]
```

---

## STEP 1: 키워드 리서치 자동화

```
[AI: 키워드 분석]
System: SEO 전문가. 한국 시장 중심으로 분석해.
User: |
  주제: {{ $json.topic }}
  다음을 JSON으로 제공해:
  {
    "mainKeyword": "주요 키워드",
    "relatedKeywords": ["관련 키워드 5개"],
    "searchIntent": "정보성|상업성|트랜잭션성",
    "difficulty": "쉬움|중간|어려움",
    "suggestedTitle": "클릭률 높은 제목 3개"
  }
```

---

## STEP 2: 긴 형식 블로그 포스트 생성

긴 콘텐츠는 여러 단계로 나누어 생성:

```javascript
// Step 1: 구조 설계
const outlinePrompt = `
주제: ${topic}
키워드: ${mainKeyword}

다음 구조의 블로그 포스트 개요를 만들어줘:
- 도입부 (독자의 문제 공감)
- 본론 3개 섹션 (각 소제목 + 핵심 포인트)
- 결론 (행동 촉구)

JSON 형식으로만 반환.
`;

// Step 2: 각 섹션 개별 생성 (더 상세하게)
for (const section of outline.sections) {
  const sectionContent = await generateSection(section);
}

// Step 3: 전체 조합
const fullPost = [intro, ...sectionContents, conclusion].join('\n\n');
```

---

## STEP 3: SEO 최적화

```
[AI: SEO 최적화]
System: SEO 전문가.
User: |
  다음 블로그 포스트를 SEO 최적화해줘:

  원문: {{ $json.draftContent }}
  주요 키워드: {{ $json.mainKeyword }}

  최적화 항목:
  1. 제목 태그 (H1) - 키워드 포함, 60자 이내
  2. 메타 설명 - 키워드 포함, 150자 이내
  3. H2/H3 소제목 최적화
  4. 키워드 밀도 확인 및 자연스러운 삽입
  5. 내부 링크 추천 위치

  JSON 형식으로 반환:
  {
    "title": "최적화된 제목",
    "metaDescription": "메타 설명",
    "optimizedContent": "최적화된 전체 내용",
    "internalLinkSuggestions": ["링크 위치 설명"]
  }
```

---

## STEP 4: AI 이미지 생성

```
[AI: 이미지 프롬프트 생성]
User: |
  블로그 포스트 주제: {{ $json.topic }}
  타깃: 한국 직장인
  스타일: 미니멀, 전문적, 한국 감성

  DALL-E 3에 최적화된 영어 이미지 프롬프트를 3개 만들어줘.
  각 프롬프트는 다른 시각적 접근으로.

[OpenAI: DALL-E 3 이미지 생성]
  Prompt: {{ $json.imagePrompts[0] }}
  Size: 1792x1024 (블로그 배너용)
  Quality: standard

[HTTP Request: 이미지 다운로드]
[Google Drive: 이미지 저장]
```

---

## STEP 5: 자동 발행

### WordPress 발행

```
[HTTP Request]
  Method: POST
  URL: https://yourblog.com/wp-json/wp/v2/posts
  Headers:
    Authorization: Basic {{ base64(username:app_password) }}
  Body:
    {
      "title": "{{ $json.title }}",
      "content": "{{ $json.optimizedContent }}",
      "status": "draft",  // 먼저 초안으로 저장
      "featured_media": {{ $json.mediaId }},
      "categories": [{{ $json.categoryId }}],
      "tags": {{ JSON.stringify($json.tags) }},
      "meta": {
        "_yoast_wpseo_metadesc": "{{ $json.metaDescription }}"
      }
    }
```

---

## 콘텐츠 캘린더 연동

```
[Notion: 콘텐츠 캘린더에서 이번 주 발행 예정 조회]
  Filter: 발행일 = 이번 주, 상태 = "초안 작성"
    ↓
[Loop: 각 콘텐츠]
    ↓
[AI 콘텐츠 생성 파이프라인]
    ↓
[Notion: 상태를 "AI 초안 완성"으로 업데이트]
    ↓
[Slack: 에디터에게 검토 요청]
```

---

## 품질 관리 체크리스트

```javascript
// AI 생성 콘텐츠 품질 자동 검사
const qualityChecks = {
  wordCount: content.split(' ').length >= 1000, // 최소 1000단어
  hasH2: content.includes('## '), // H2 소제목 있음
  hasKeyword: content.includes(mainKeyword), // 키워드 포함
  noRepetition: !hasRepetitiveContent(content), // 반복 없음
  readingLevel: calculateReadingLevel(content) <= 8 // 읽기 쉬움
};

const allPassed = Object.values(qualityChecks).every(v => v);
```

---

## 실전 예제 모음

---

### 예제 1: 유튜브 영상 제목/설명/태그 일괄 생성 (Google Sheets 입력 → 자동 생성)

Google Sheets에 영상 주제와 키워드를 입력하면 유튜브 알고리즘에 최적화된 제목, 설명, 태그를 자동으로 생성하여 시트에 다시 기록합니다.

워크플로우 구조:

```
[Schedule Trigger: 매일 오전 9시]
    ↓
[Google Sheets: "대기중" 상태인 행 조회]
  필터: 상태 열 === "대기중"
    ↓
[Loop: 각 행 처리]
    ↓
[OpenAI: 유튜브 메타데이터 일괄 생성]
    ↓
[Code: 생성 결과 파싱]
    ↓
[Google Sheets: 해당 행 결과 업데이트]
  - 제목 후보 3개 기록
  - 설명 기록
  - 태그 기록
  - 상태를 "완료"로 변경
    ↓
[Slack: 처리 완료 알림]
```

Google Sheets 입력 시트 구조:

```
열 구성:
A: 영상 주제 (예: "n8n으로 업무 자동화하는 방법")
B: 주요 키워드 (예: "n8n, 자동화, 노코드")
C: 영상 길이 (예: "10분")
D: 타깃 시청자 (예: "직장인, IT 비전공자")
E: 채널 카테고리 (예: "테크/생산성")
F: 상태 (대기중 / 처리중 / 완료)
G: 제목_1
H: 제목_2
I: 제목_3
J: 영상 설명
K: 태그
L: 생성일시
```

유튜브 메타데이터 생성 프롬프트:

```
[OpenAI: gpt-4o]
System: |
  유튜브 SEO 전문가야. 한국 유튜브 알고리즘에 최적화된 메타데이터를 생성해.
  반드시 JSON만 반환해.
  {
    "titles": ["제목1(35자 이내)", "제목2(35자 이내)", "제목3(35자 이내)"],
    "description": "영상 설명 (첫 2줄에 핵심 키워드 포함, 총 500자 내외)",
    "tags": ["태그1", "태그2", ...],  // 15~20개, 짧은 것부터 구체적인 것까지
    "thumbnailTextSuggestion": "썸네일에 들어갈 짧은 텍스트 (10자 이내)"
  }

  제목 작성 규칙:
  - 숫자 포함 시 클릭률 상승 (예: "3가지 방법")
  - 호기심/감정 자극 (예: "몰랐던", "놀라운", "이렇게 하면")
  - 키워드는 앞쪽에 배치
  - 낚시성 표현 금지

  설명 작성 규칙:
  - 첫 줄: 영상 핵심 내용 요약 (키워드 포함)
  - 2~4줄: 영상에서 배울 내용 목록
  - 마지막: 구독/좋아요 CTA 포함

User: |
  영상 주제: {{ $json.topic }}
  주요 키워드: {{ $json.keywords }}
  영상 길이: {{ $json.duration }}
  타깃 시청자: {{ $json.targetAudience }}
  채널 카테고리: {{ $json.channelCategory }}

Temperature: 0.7
Max Tokens: 800
```

Google Sheets 업데이트 설정:

```
[Google Sheets: Update Row]
  스프레드시트 ID: YOUR_SHEET_ID
  시트: 유튜브_메타데이터
  행 번호: {{ $json.rowNumber }}
  업데이트할 열:
    G (제목_1): {{ $json.titles[0] }}
    H (제목_2): {{ $json.titles[1] }}
    I (제목_3): {{ $json.titles[2] }}
    J (설명): {{ $json.description }}
    K (태그): {{ $json.tags.join(", ") }}
    F (상태): 완료
    L (생성일시): {{ $now.toISO() }}
```

---

### 예제 2: 블로그 초안 작성 자동화 (키워드 입력 → SEO 최적화 초안)

키워드만 입력하면 독자 검색 의도 분석부터 SEO 최적화 초안 작성까지 자동으로 처리합니다. 초안은 Notion에 저장되고 에디터에게 슬랙으로 알림이 갑니다.

워크플로우 구조:

```
[Webhook: 키워드 + 글 유형 수신]
  입력: { keyword: "재택근무 생산성", postType: "how-to", targetLength: 1500 }
    ↓
[OpenAI: 검색 의도 및 경쟁 분석]
  출력: searchIntent, relatedKeywords, recommendedStructure
    ↓
[OpenAI: 글 개요 생성]
  출력: title, sections(제목+핵심포인트 배열), intro, conclusion
    ↓
[Loop: 각 섹션 개별 작성]
  각 섹션을 독립적으로 생성 (더 높은 품질)
    ↓
[Code: 전체 초안 조합]
    ↓
[OpenAI: SEO 메타데이터 생성]
  출력: metaTitle, metaDescription, slug, internalLinkSuggestions
    ↓
[Notion: 새 페이지 생성]
    ↓
[Slack: 에디터에게 검토 요청]
```

검색 의도 분석 프롬프트:

```
[OpenAI: gpt-4o-mini]
System: |
  한국 SEO 전문가야. 검색 의도와 콘텐츠 전략을 분석해. JSON만 반환해.
  {
    "searchIntent": "informational" | "navigational" | "commercial" | "transactional",
    "targetReaderProfile": "독자 프로필 1문장",
    "relatedKeywords": ["관련 키워드 8개"],
    "lsiKeywords": ["LSI 키워드 5개"],
    "recommendedH2Titles": ["추천 소제목 4~5개"],
    "competitorAngle": "경쟁 글과 차별화할 접근 방식 1문장",
    "estimatedWordCount": 최적 글자 수(숫자)
  }

User: |
  주요 키워드: {{ $json.keyword }}
  글 유형: {{ $json.postType }}
  목표 길이: {{ $json.targetLength }}자

Temperature: 0.3
```

섹션별 작성 프롬프트 (루프 내부):

```
[OpenAI: gpt-4o]
System: |
  전문 콘텐츠 라이터야. 한국어로 읽기 쉽고 전문성 있는 글을 써.
  - 문장은 2~3줄로 짧게 끊어서 가독성 높임
  - 수치, 사례, 구체적 예시 포함
  - 과도한 미사여구 금지, 실용적 정보 중심
  - 소제목 바로 아래 핵심 내용 요약 1줄 포함

User: |
  블로그 주제: {{ $node["Webhook"].json.keyword }}
  전체 글 제목: {{ $node["개요생성"].json.title }}

  현재 작성할 섹션:
  소제목: {{ $json.sectionTitle }}
  핵심 포인트: {{ $json.keyPoints.join(", ") }}
  이 섹션 목표 길이: {{ Math.floor($node["Webhook"].json.targetLength / sectionsCount) }}자

  이 섹션 내용을 완전하게 작성해줘.
  다른 섹션 내용은 쓰지 말고 이 섹션만 작성해.

Temperature: 0.6
Max Tokens: 600
```

전체 초안 조합 코드:

```javascript
const sections = $input.all();
const meta = $node["개요생성"].json;

// 서론 + 각 섹션 + 결론 조합
const fullDraft = [
  `# ${meta.title}\n\n`,
  `${meta.intro}\n\n`,
  ...sections.map(s => `## ${s.json.sectionTitle}\n\n${s.json.content}\n\n`),
  `## 마무리\n\n${meta.conclusion}`
].join('');

const wordCount = fullDraft.replace(/\s+/g, '').length;

return [{
  json: {
    title: meta.title,
    fullDraft,
    wordCount,
    keyword: $node["Webhook"].json.keyword,
    generatedAt: new Date().toISOString()
  }
}];
```

Notion 페이지 생성 설정:

```
[HTTP Request: Notion API]
  Method: POST
  URL: https://api.notion.com/v1/pages
  Headers:
    Authorization: Bearer {{ NOTION_TOKEN }}
    Notion-Version: 2022-06-28
  Body:
    {
      "parent": { "database_id": "YOUR_BLOG_DB_ID" },
      "properties": {
        "제목": { "title": [{ "text": { "content": "{{ $json.title }}" } }] },
        "키워드": { "rich_text": [{ "text": { "content": "{{ $json.keyword }}" } }] },
        "상태": { "select": { "name": "AI 초안" } },
        "글자수": { "number": {{ $json.wordCount }} },
        "생성일": { "date": { "start": "{{ $json.generatedAt }}" } },
        "메타제목": { "rich_text": [{ "text": { "content": "{{ $json.metaTitle }}" } }] },
        "메타설명": { "rich_text": [{ "text": { "content": "{{ $json.metaDescription }}" } }] }
      },
      "children": [
        {
          "object": "block",
          "type": "paragraph",
          "paragraph": {
            "rich_text": [{ "type": "text", "text": { "content": "{{ $json.fullDraft.substring(0, 2000) }}" } }]
          }
        }
      ]
    }
```

---

### 예제 3: 카카오톡 채널 메시지 A/B 버전 동시 생성

마케팅 캠페인 내용을 입력하면 서로 다른 톤/접근법의 카카오톡 채널 메시지 A/B 버전을 동시에 생성하고, 결과를 Google Sheets에 저장하여 성과 비교를 준비합니다.

워크플로우 구조:

```
[Webhook: 캠페인 정보 수신]
  입력: { campaignName, product, offer, targetSegment, sendDate }
    ↓
[병렬 실행: A버전과 B버전 동시 생성]
  ├── [OpenAI: A버전 생성] (혜택/이성적 접근)
  └── [OpenAI: B버전 생성] (감성/스토리텔링 접근)
    ↓ (두 결과 병합)
[Code: A/B 버전 조합 + 캐릭터 수 검증]
    ↓
[Google Sheets: 캠페인 저장]
  - A/B 버전 메시지
  - 예상 발송 대상수
  - 성과 측정 준비 열 (오픈율, 클릭율 - 나중에 기록용)
    ↓
[Slack: 마케팅팀에 검토 요청]
  (A/B 버전 미리보기 포함)
```

A버전 생성 프롬프트 (이성적/혜택 중심):

```
[OpenAI: gpt-4o]
System: |
  카카오톡 채널 메시지 전문 카피라이터야.
  A버전은 혜택과 수치 중심의 이성적 접근으로 작성해.

  카카오톡 메시지 규칙:
  - 첫 줄: 강력한 훅 (이모지 1개 허용)
  - 총 길이: 90자 이내 (카카오 기본 메시지 기준)
  - 혜택을 명확하게 수치로 표현
  - CTA(행동 유도) 문구 포함
  - 과장 광고 금지

  JSON만 반환:
  {
    "version": "A",
    "approach": "benefit_rational",
    "message": "메시지 본문",
    "charCount": 글자수(숫자),
    "hook": "첫 문장",
    "cta": "CTA 문구",
    "copywritingNote": "이 버전의 전략 1문장"
  }

User: |
  캠페인명: {{ $json.campaignName }}
  상품/서비스: {{ $json.product }}
  핵심 혜택/오퍼: {{ $json.offer }}
  타깃 세그먼트: {{ $json.targetSegment }}
  발송 예정일: {{ $json.sendDate }}

Temperature: 0.7
Max Tokens: 300
```

B버전 생성 프롬프트 (감성/스토리텔링):

```
[OpenAI: gpt-4o]
System: |
  카카오톡 채널 메시지 전문 카피라이터야.
  B버전은 감성과 공감 중심의 스토리텔링 접근으로 작성해.

  카카오톡 메시지 규칙:
  - 첫 줄: 독자의 일상/감정에 공감하는 문장
  - 총 길이: 90자 이내
  - 독자의 감정에 먼저 연결한 후 혜택 제시
  - CTA는 부드럽고 자연스럽게
  - 이모지 2개까지 허용

  JSON만 반환:
  {
    "version": "B",
    "approach": "emotional_storytelling",
    "message": "메시지 본문",
    "charCount": 글자수(숫자),
    "hook": "첫 문장",
    "cta": "CTA 문구",
    "copywritingNote": "이 버전의 전략 1문장"
  }

User: |
  캠페인명: {{ $json.campaignName }}
  상품/서비스: {{ $json.product }}
  핵심 혜택/오퍼: {{ $json.offer }}
  타깃 세그먼트: {{ $json.targetSegment }}
  발송 예정일: {{ $json.sendDate }}

Temperature: 0.8
Max Tokens: 300
```

A/B 버전 조합 및 검증 코드:

```javascript
// A버전과 B버전 결과 병합
const versionA = JSON.parse($node["OpenAI A버전"].json.message.content);
const versionB = JSON.parse($node["OpenAI B버전"].json.message.content);

// 글자수 검증 (90자 초과 시 경고)
const aOverLimit = versionA.charCount > 90;
const bOverLimit = versionB.charCount > 90;

// 캠페인 고유 ID 생성
const campaignId = `ABT-${Date.now()}`;

return [{
  json: {
    campaignId,
    campaignName: $node["Webhook"].json.campaignName,
    product: $node["Webhook"].json.product,
    offer: $node["Webhook"].json.offer,
    targetSegment: $node["Webhook"].json.targetSegment,
    sendDate: $node["Webhook"].json.sendDate,
    versionA: {
      ...versionA,
      overCharLimit: aOverLimit
    },
    versionB: {
      ...versionB,
      overCharLimit: bOverLimit
    },
    generatedAt: new Date().toISOString(),
    status: "검토대기"
  }
}];
```

Google Sheets 저장 설정:

```
[Google Sheets: Append Row]
  시트: AB테스트_캠페인
  데이터:
    캠페인ID: {{ $json.campaignId }}
    캠페인명: {{ $json.campaignName }}
    상품: {{ $json.product }}
    타깃: {{ $json.targetSegment }}
    발송예정일: {{ $json.sendDate }}
    A버전메시지: {{ $json.versionA.message }}
    A버전전략: {{ $json.versionA.copywritingNote }}
    A글자수: {{ $json.versionA.charCount }}
    B버전메시지: {{ $json.versionB.message }}
    B버전전략: {{ $json.versionB.copywritingNote }}
    B글자수: {{ $json.versionB.charCount }}
    생성일시: {{ $json.generatedAt }}
    상태: {{ $json.status }}
    A오픈율: (성과 측정 후 기록)
    B오픈율: (성과 측정 후 기록)
    승리버전: (A/B 테스트 결과 후 기록)
```

Slack 검토 요청 알림 설정:

```
[Slack: 메시지 발송]
  채널: #marketing
  메시지:
    카카오톡 A/B 메시지가 생성되었습니다. 검토 후 승인해주세요.

    캠페인: {{ $json.campaignName }} ({{ $json.campaignId }})
    발송 예정: {{ $json.sendDate }} | 타깃: {{ $json.targetSegment }}

    ─── A버전 (혜택/이성 중심) ───
    {{ $json.versionA.message }}
    글자수: {{ $json.versionA.charCount }}자 {{ $json.versionA.overCharLimit ? "⚠️ 90자 초과" : "" }}
    전략: {{ $json.versionA.copywritingNote }}

    ─── B버전 (감성/스토리) ───
    {{ $json.versionB.message }}
    글자수: {{ $json.versionB.charCount }}자 {{ $json.versionB.overCharLimit ? "⚠️ 90자 초과" : "" }}
    전략: {{ $json.versionB.copywritingNote }}

    시트에서 확인: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID
```

---

## 핵심 요약

- 아이디어 → 발행까지 완전 자동화 파이프라인 구축
- SEO 키워드 분석, 구조 설계, 초안 작성을 AI가 단계별로 처리
- DALL-E로 맞춤형 이미지까지 자동 생성
- Notion 콘텐츠 캘린더로 발행 일정 자동 관리
- 품질 체크 로직으로 최소 기준 자동 확인

**다음 레슨**: 여러 AI 에이전트가 협력하는 멀티 에이전트 시스템을 설계합니다.
