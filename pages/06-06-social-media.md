# 레슨 6.6: 소셜 미디어 자동 포스팅

---

## 소셜 미디어 자동화의 가치

마케터가 매주 각 플랫폼에 맞게 콘텐츠를 수동으로 올리는 데 걸리는 시간: **평균 8시간**

n8n으로 자동화하면: **30분으로 단축** (아이디어 입력 시간만 필요)

---

## 지원 플랫폼

n8n에서 지원하는 소셜 미디어:
- **Twitter/X**: 트윗 발행, 리플라이, 좋아요
- **LinkedIn**: 개인/기업 페이지 포스팅
- **Instagram**: 이미지/영상 포스팅 (Graph API)
- **Facebook**: 페이지 포스팅
- **YouTube**: 비디오 업로드, 댓글 관리
- **Pinterest**: 핀 생성

---

## 크리덴셜 설정

### Twitter/X API

1. `developer.twitter.com` 접속, 앱 생성
2. API Key, API Secret, Access Token, Access Secret 발급
3. n8n → Credentials → Twitter OAuth1 API

### LinkedIn API

1. `linkedin.com/developers` 앱 생성
2. "Share on LinkedIn" 권한 추가
3. OAuth 2.0 인증
4. n8n → Credentials → LinkedIn OAuth2 API

---

## 프로젝트 1: 멀티 채널 자동 발행 시스템

하나의 콘텐츠 아이디어를 입력하면 각 플랫폼에 맞게 자동 변환하여 발행:

### 워크플로우 구조

```
[Notion: 발행 예정 콘텐츠 DB 조회]
  Filter: Status = "발행 대기"
    ↓
[AI: 각 플랫폼용 텍스트 생성]
    ↓
병렬 발행:
  ├── [Twitter: 트윗 (280자)]
  ├── [LinkedIn: 포스팅 (긴 형식)]
  └── [Instagram: 캡션 생성]
    ↓
[Notion: Status를 "발행 완료"로 업데이트]
    ↓
[Slack: 발행 완료 알림]
```

### AI 기반 플랫폼별 텍스트 변환

```javascript
// Code 노드: 원본 콘텐츠를 각 플랫폼 형식으로 변환
const originalContent = $input.item.json.content;
const topic = $input.item.json.topic;

return [{
  json: {
    original: originalContent,
    topic: topic,
    twitterPrompt: `다음 내용을 Twitter용으로 280자 이내로 변환. 해시태그 2-3개 포함: "${originalContent}"`,
    linkedinPrompt: `다음 내용을 LinkedIn 전문가 포스팅 형식으로 변환. 통찰력 있고 전문적인 어조, 500-1000자: "${originalContent}"`,
    instagramPrompt: `다음 내용을 Instagram 캡션으로 변환. 친근하고 시각적 묘사 포함, 150자 + 해시태그 5개: "${originalContent}"`
  }
}];
```

---

## 프로젝트 2: RSS → 소셜 미디어 자동화

블로그 새 글 발행 시 자동으로 소셜 미디어에 공유:

```
[Schedule: 매시간]
    ↓
[HTTP Request: RSS 피드 조회]
  URL: https://yourblog.com/feed.xml
    ↓
[Code: 새 글 확인 (마지막 체크 이후)]
    ↓
[AI: 각 플랫폼 최적화 텍스트 생성]
    ↓
[Twitter: 새 글 공유]
[LinkedIn: 새 글 공유]
    ↓
[Redis/Sheets: 마지막 체크 시간 업데이트]
```

---

## 프로젝트 3: 예약 포스팅 시스템

Google Sheets를 편집기로 활용하는 예약 포스팅:

```
콘텐츠 캘린더 (Google Sheets):
| 날짜 | 플랫폼 | 내용 | 이미지 | 상태 |
|------|--------|------|--------|------|
| 2026-04-02 10:00 | Twitter | 신제품 출시! | img1.jpg | 대기 |
| 2026-04-02 11:00 | LinkedIn | 상세 소개... | img2.jpg | 대기 |
```

```
[Schedule: 매 5분]
    ↓
[Google Sheets: 발행 시간 도래한 콘텐츠 조회]
  Filter: 발행시간 <= 지금, 상태 = "대기"
    ↓
[Switch: 플랫폼]
  ├── Twitter → [Twitter 노드]
  ├── LinkedIn → [LinkedIn 노드]
  └── Instagram → [Instagram 노드]
    ↓
[Google Sheets: 상태를 "완료"로 업데이트]
```

---

## 해시태그 자동 생성

```javascript
// Code 노드에서 AI 해시태그 생성 후 처리
const aiResponse = $input.item.json.aiOutput;

// AI가 생성한 해시태그 추출
const hashtagRegex = /#[가-힣a-zA-Z0-9]+/g;
const hashtags = aiResponse.match(hashtagRegex) || [];

// 인기 고정 해시태그 추가
const fixedHashtags = ['#n8n', '#자동화', '#AI'];
const allHashtags = [...new Set([...hashtags, ...fixedHashtags])];

return [{
  json: {
    content: aiResponse,
    hashtags: allHashtags,
    hashtagString: allHashtags.join(' ')
  }
}];
```

---

## 이미지 자동화

이미지 생성 AI(DALL-E, Midjourney API)와 연동:

```
[콘텐츠 아이디어 입력]
    ↓
[OpenAI DALL-E: 이미지 생성]
  Prompt: "{{ $json.topic }}에 관한 미니멀한 인포그래픽, 흰 배경"
    ↓
[HTTP Request: 이미지 다운로드]
    ↓
[Google Drive: 이미지 저장]
    ↓
[소셜 미디어: 이미지 + 캡션 포스팅]
```

---

## 성과 추적 자동화

발행 후 성과 데이터를 자동 수집:

```
[Schedule: 매일 오후 11시]
    ↓
[Twitter API: 오늘 트윗 인게이지먼트 조회]
    ↓
[Google Sheets: 성과 데이터 기록]
  날짜, 플랫폼, 노출수, 좋아요, 리트윗, 클릭수
```

---

## 실전 예제 모음

---

### 예제 1: 인스타그램 게시물 반응 모니터링 → 인기 게시물 분석 리포트

매일 자정에 인스타그램 비즈니스 계정의 최근 30일 게시물 성과를 수집하고, 상위 5개 인기 게시물을 분석하여 어떤 콘텐츠가 잘 되는지 자동으로 파악합니다.

워크플로우 구조:

```
[Schedule: 매일 오전 6시 (0 6 * * *)]
    ↓
[HTTP Request: Instagram Graph API - 게시물 목록 조회]
  GET /me/media?fields=id,caption,media_type,timestamp,like_count,comments_count
    ↓
[Code: 최근 30일 게시물 필터링 + 인게이지먼트 점수 계산]
    ↓
[Code: 상위 5개 추출 + 리포트 텍스트 생성]
    ↓
[OpenAI: 인기 게시물 패턴 분석]
    ↓
[Slack: #marketing 채널에 리포트 발송]
    ↓
[Google Sheets: 성과 데이터 누적 기록]
```

HTTP Request 노드 - Instagram API 호출:

```
[HTTP Request]
  Method: GET
  URL: https://graph.instagram.com/me/media
  Query Parameters:
    fields: id,caption,media_type,timestamp,like_count,comments_count,permalink
    access_token: {INSTAGRAM_ACCESS_TOKEN}
    limit: 100
```

Code 노드 - 인게이지먼트 점수 계산:

```javascript
const today = new Date();
const thirtyDaysAgo = new Date(today - 30 * 24 * 60 * 60 * 1000);

const posts = $input.item.json.data;

const scoredPosts = posts
  .filter(post => {
    const postDate = new Date(post.timestamp);
    return postDate >= thirtyDaysAgo;
  })
  .map(post => {
    const likes = post.like_count ?? 0;
    const comments = post.comments_count ?? 0;

    // 댓글은 좋아요보다 가중치 3배 (더 적극적 반응)
    const engagementScore = likes + comments * 3;

    // 캡션에서 첫 줄(제목)만 추출
    const captionFirstLine = (post.caption ?? '(캡션 없음)').split('\n')[0].slice(0, 60);

    return {
      id: post.id,
      captionPreview: captionFirstLine,
      mediaType: post.media_type,
      timestamp: post.timestamp,
      likes,
      comments,
      engagementScore,
      permalink: post.permalink
    };
  })
  .sort((a, b) => b.engagementScore - a.engagementScore);

// 전체 평균 인게이지먼트
const totalPosts = scoredPosts.length;
const avgScore = totalPosts > 0
  ? Math.round(scoredPosts.reduce((sum, p) => sum + p.engagementScore, 0) / totalPosts)
  : 0;

const top5 = scoredPosts.slice(0, 5);

return [{
  json: {
    top5,
    totalPosts,
    avgEngagementScore: avgScore,
    reportDate: today.toISOString().slice(0, 10)
  }
}];
```

OpenAI 노드 - 패턴 분석:

```
[OpenAI: Message a Model]
  Model: gpt-4o-mini
  System: 당신은 소셜 미디어 마케팅 전문가입니다. 인스타그램 게시물 데이터를 보고 어떤 패턴이 인기를 끌었는지 간결하게 분석합니다.
  User: |
    최근 30일 인스타그램 상위 5개 게시물입니다.
    {{ $json.top5.map((p, i) => `${i+1}위: "${p.captionPreview}" / 좋아요 ${p.likes} / 댓글 ${p.comments} / 점수 ${p.engagementScore}`).join('\n') }}

    위 데이터를 바탕으로 다음을 분석해 주세요:
    1. 공통적으로 잘 된 콘텐츠 유형
    2. 다음 주에 시도하면 좋을 콘텐츠 방향 2가지
    분석 결과는 200자 이내로 작성하세요.
```

Slack 발송 노드:

```javascript
// Code 노드: Slack 메시지 조립
const { top5, totalPosts, avgEngagementScore, reportDate } = $('Code: 인게이지먼트 점수').item.json;
const analysis = $('OpenAI: 패턴 분석').item.json.message.content;

const rankLines = top5.map((p, i) =>
  `${i + 1}위  좋아요 ${p.likes} / 댓글 ${p.comments}  "${p.captionPreview}"\n   ${p.permalink}`
).join('\n\n');

const message = [
  `${reportDate} 인스타그램 인기 게시물 리포트`,
  `분석 대상: ${totalPosts}개 게시물 / 평균 인게이지먼트 점수: ${avgEngagementScore}`,
  '',
  '상위 5개 게시물:',
  rankLines,
  '',
  'AI 패턴 분석:',
  analysis
].join('\n');

return [{ json: { message } }];
```

---

### 예제 2: 여러 SNS 동시 예약 발행 (하나의 콘텐츠 → 각 플랫폼 최적화)

Notion 콘텐츠 캘린더에 원본 글을 등록하면, 예약 시간에 맞춰 플랫폼별로 자동 최적화하여 Twitter, LinkedIn, Instagram에 동시 발행합니다.

워크플로우 구조:

```
[Schedule: 매 10분 (*/10 * * * *)]
    ↓
[Notion: Get All - 콘텐츠 캘린더 DB]
  Filter: 발행 예정 시간 <= 지금 AND 상태 = "예약됨"
    ↓
[IF: 발행할 콘텐츠 없음?]
  └── True → 종료
    ↓
[Split In Batches: 콘텐츠 1개씩 처리]
    ↓
[OpenAI: 플랫폼별 텍스트 3종 동시 생성]
    ↓
[Code: AI 응답 파싱 + 플랫폼별 데이터 분리]
    ↓
병렬 실행:
  ├── [HTTP Request: Twitter API v2 - 트윗 발행]
  ├── [LinkedIn: Create Post]
  └── [HTTP Request: Instagram Graph API - 미디어 게시]
    ↓
[Notion: Update Page - 상태를 "발행완료"로 변경]
    ↓
[Slack: 발행 완료 요약 알림]
```

OpenAI 노드 - 플랫폼별 텍스트 3종 생성:

```
[OpenAI: Message a Model]
  Model: gpt-4o
  System: 당신은 소셜 미디어 카피라이터입니다. 하나의 원본 콘텐츠를 각 플랫폼 특성에 맞게 변환합니다.
  User: |
    원본 콘텐츠: {{ $json.원본내용 }}
    업종/주제: {{ $json.주제 }}

    아래 JSON 형식으로만 응답하세요:
    {
      "twitter": "280자 이내. 핵심 한 줄 + 해시태그 2개. 구어체로 작성.",
      "linkedin": "전문적이고 통찰력 있는 어조. 첫 문장은 독자의 관심을 끄는 훅. 3-5개 단락. 마지막에 질문으로 마무리. 해시태그 3개.",
      "instagram": "감성적이고 시각적인 표현. 이모티콘 2-3개 포함. 150자 본문 + 해시태그 7개."
    }
```

Code 노드 - AI 응답 파싱:

```javascript
const raw = $input.item.json.message.content;
let parsed;

try {
  parsed = JSON.parse(raw);
} catch {
  // JSON 파싱 실패 시 빈값으로 폴백
  parsed = { twitter: '', linkedin: '', instagram: '' };
}

const notionPageId = $('Notion: Get All').item.json.id;
const mediaUrl = $('Notion: Get All').item.json.properties['이미지URL']?.url ?? null;

return [{
  json: {
    notionPageId,
    twitter: parsed.twitter,
    linkedin: parsed.linkedin,
    instagram: parsed.instagram,
    mediaUrl
  }
}];
```

Twitter API v2 발행 노드:

```
[HTTP Request: Twitter 트윗 발행]
  Method: POST
  URL: https://api.twitter.com/2/tweets
  Authentication: OAuth1.0
    Consumer Key: {TWITTER_API_KEY}
    Consumer Secret: {TWITTER_API_SECRET}
    Access Token: {TWITTER_ACCESS_TOKEN}
    Token Secret: {TWITTER_TOKEN_SECRET}
  Body (JSON):
    {
      "text": "{{ $json.twitter }}"
    }
```

LinkedIn 발행 노드:

```
[LinkedIn: Create Post]
  Person URN: urn:li:person:{YOUR_PERSON_ID}
  Text: {{ $json.linkedin }}
  Visibility: PUBLIC
```

Instagram 게시 노드 (2단계: 미디어 생성 → 게시):

```
1단계: 미디어 컨테이너 생성
[HTTP Request]
  Method: POST
  URL: https://graph.instagram.com/{INSTAGRAM_USER_ID}/media
  Query Params:
    image_url: {{ $json.mediaUrl }}
    caption: {{ $json.instagram }}
    access_token: {INSTAGRAM_ACCESS_TOKEN}

2단계: 게시 확정
[HTTP Request]
  Method: POST
  URL: https://graph.instagram.com/{INSTAGRAM_USER_ID}/media_publish
  Query Params:
    creation_id: {{ $json.id }}
    access_token: {INSTAGRAM_ACCESS_TOKEN}
```

---

### 예제 3: 경쟁사 소셜 미디어 키워드 모니터링 + 알림

경쟁사 키워드나 업계 트렌드 키워드가 Twitter에서 언급될 때 실시간으로 감지하고, 중요도를 AI로 판단하여 마케팅팀에 알립니다.

워크플로우 구조:

```
[Schedule: 매 30분 (*/30 * * * *)]
    ↓
[HTTP Request: Twitter Search API v2]
  키워드 쿼리로 최신 트윗 검색
    ↓
[Code: 이미 처리한 트윗 중복 제거]
    ↓
[IF: 새 트윗 없음?]
  └── True → 종료
    ↓
[Split In Batches]
    ↓
[OpenAI: 트윗 중요도 판단]
  경쟁사 동향인지, 위기 신호인지, 기회인지 분류
    ↓
[IF: 중요도 높음 (High / Critical)?]
  ├── True → [Slack: 즉시 알림 #marketing-alert]
  └── False → [Google Sheets: 일반 로그 누적]
    ↓
[Google Sheets: 마지막 처리 트윗 ID 저장]
```

HTTP Request 노드 - Twitter 검색:

```
[HTTP Request]
  Method: GET
  URL: https://api.twitter.com/2/tweets/search/recent
  Authentication: Bearer Token
    Token: {TWITTER_BEARER_TOKEN}
  Query Parameters:
    query: (경쟁사A OR 경쟁사B OR "업계키워드") -is:retweet lang:ko
    max_results: 50
    tweet.fields: id,text,created_at,public_metrics,author_id
    since_id: {{ $vars.lastTweetId ?? '' }}
```

Code 노드 - 중복 제거 및 데이터 정리:

```javascript
const tweets = $input.item.json.data ?? [];

if (tweets.length === 0) {
  return [{ json: { isEmpty: true } }];
}

// 가장 최근 트윗 ID를 저장해 다음 실행 시 중복 방지
const latestId = tweets[0].id;

const cleaned = tweets.map(tweet => ({
  tweetId: tweet.id,
  text: tweet.text,
  createdAt: tweet.created_at,
  likes: tweet.public_metrics?.like_count ?? 0,
  retweets: tweet.public_metrics?.retweet_count ?? 0,
  replies: tweet.public_metrics?.reply_count ?? 0,
  tweetUrl: `https://twitter.com/i/web/status/${tweet.id}`
}));

return [{ json: { tweets: cleaned, latestId, isEmpty: false } }];
```

OpenAI 노드 - 중요도 판단:

```
[OpenAI: Message a Model]
  Model: gpt-4o-mini
  System: 당신은 마케팅 인텔리전스 분석가입니다. 소셜 미디어 트윗을 보고 기업에게 중요한 신호인지 판단합니다.
  User: |
    트윗 내용: "{{ $json.text }}"
    좋아요: {{ $json.likes }}, 리트윗: {{ $json.retweets }}

    다음 JSON으로만 응답하세요:
    {
      "importance": "Low | Medium | High | Critical",
      "category": "경쟁사동향 | 위기신호 | 트렌드기회 | 고객불만 | 기타",
      "reason": "중요도 판단 이유 (30자 이내)",
      "action": "마케팅팀 권장 액션 (30자 이내)"
    }
```

Slack 알림 노드 (중요도 High/Critical일 때만):

```javascript
// Code 노드: Slack 알림 메시지 조립
const tweet = $('Split In Batches').item.json;
const ai = JSON.parse($('OpenAI: 중요도 판단').item.json.message.content);

const emoji = ai.importance === 'Critical' ? '🚨' : '⚠️';

const message = [
  `${emoji} 경쟁사/키워드 모니터링 알림 (${ai.importance})`,
  '',
  `분류: ${ai.category}`,
  `판단 이유: ${ai.reason}`,
  `권장 액션: ${ai.action}`,
  '',
  `트윗 내용:`,
  `"${tweet.text}"`,
  '',
  `반응: 좋아요 ${tweet.likes} / 리트윗 ${tweet.retweets}`,
  `링크: ${tweet.tweetUrl}`
].join('\n');

return [{ json: { message, channel: '#marketing-alert' } }];
```

---

## 핵심 요약

- 하나의 원본 콘텐츠를 AI로 각 플랫폼에 최적화하여 발행
- RSS 피드 모니터링으로 블로그 글을 자동으로 소셜에 공유
- Google Sheets를 콘텐츠 캘린더로 활용한 예약 포스팅
- DALL-E API로 이미지까지 자동 생성
- 발행 후 성과 데이터 자동 수집으로 개선 루프 완성

**다음 레슨**: 웹훅 기반 자동화의 심화 활용을 배웁니다.
