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

## 핵심 요약

- 아이디어 → 발행까지 완전 자동화 파이프라인 구축
- SEO 키워드 분석, 구조 설계, 초안 작성을 AI가 단계별로 처리
- DALL-E로 맞춤형 이미지까지 자동 생성
- Notion 콘텐츠 캘린더로 발행 일정 자동 관리
- 품질 체크 로직으로 최소 기준 자동 확인

**다음 레슨**: 여러 AI 에이전트가 협력하는 멀티 에이전트 시스템을 설계합니다.
