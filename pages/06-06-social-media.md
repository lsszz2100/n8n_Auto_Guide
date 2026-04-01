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

## 핵심 요약

- 하나의 원본 콘텐츠를 AI로 각 플랫폼에 최적화하여 발행
- RSS 피드 모니터링으로 블로그 글을 자동으로 소셜에 공유
- Google Sheets를 콘텐츠 캘린더로 활용한 예약 포스팅
- DALL-E API로 이미지까지 자동 생성
- 발행 후 성과 데이터 자동 수집으로 개선 루프 완성

**다음 레슨**: 웹훅 기반 자동화의 심화 활용을 배웁니다.
