# 레슨 10.2: 커뮤니티와 학습 리소스

---

## 공식 리소스

### n8n 공식 문서

| 리소스 | 내용 |
|--------|------|
| docs.n8n.io | 공식 문서 (노드별 상세 설명) |
| n8n.io/blog | 공식 블로그 (사용 사례, 튜토리얼) |
| github.com/n8n-io/n8n | 소스코드, 이슈 트래커 |
| community.n8n.io | 공식 커뮤니티 포럼 |

### 공식 문서 활용 팁

```
특정 노드 사용법 모를 때:
docs.n8n.io/integrations/builtin/[노드이름]/

예시:
docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/
docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.openai/
```

---

## 커뮤니티

### n8n 공식 커뮤니티 포럼

**community.n8n.io**에서:
- 막히는 문제 질문하기
- 다른 사람들의 워크플로우 구경하기
- 버그 리포트 및 기능 요청
- 정기적으로 "워크플로우 공유" 쓰레드 있음

**좋은 질문하는 법:**
```
1. 문제 상황 명확히 설명
2. 시도한 방법 언급
3. 오류 메시지 전문 포함
4. n8n 버전 명시
5. 관련 워크플로우 JSON 첨부 (민감정보 제거 후)
```

### Reddit

- r/n8n - n8n 전용 서브레딧
- r/automation - 자동화 일반
- r/nocode - 노코드 도구 전반

### Discord

- n8n 공식 Discord 서버 (공식 사이트에서 링크 확인)
- 실시간 질문 및 답변
- 채널별로 주제 구분 (general, help, showcase)

---

## YouTube 학습 채널

**영어:**
- **n8n 공식 채널**: 공식 튜토리얼 및 새 기능 소개
- **Liam Ottley**: AI 자동화 중심 n8n 튜토리얼
- **Cole Medin**: 고급 n8n + AI 에이전트

**한국어:**
- n8n 한국어 자료는 아직 많지 않음
- 영어 자료 + 이 책으로 시작하고
- 한국어 콘텐츠 직접 만들어서 기여하기!

---

## 워크플로우 템플릿

### n8n 공식 템플릿

**n8n.io/workflows** 에서:
- 900개 이상의 무료 워크플로우 템플릿
- 카테고리별 탐색
- 클릭 한 번으로 n8n에 가져오기

**유용한 카테고리:**
```
AI → AI 에이전트, RAG 시스템, 콘텐츠 생성
Marketing → 소셜 미디어, 이메일 마케팅
Sales → CRM, 리드 관리
DevOps → CI/CD, 모니터링
Productivity → 업무 자동화
```

---

## 도움이 되는 도구들

### API 테스트

```
Postman (postman.com)
  - API 호출 테스트
  - 컬렉션으로 API 관리
  - 자동화 테스트

Insomnia (insomnia.rest)
  - Postman 대안, 오픈소스
```

### JSON 처리

```
jq (명령줄 JSON 처리)
  echo '{"name":"n8n"}' | jq '.name'

JSON Formatter (jsonformatter.org)
  - 온라인 JSON 포맷팅/검증

JSONPath Online Evaluator
  - JSONPath 표현식 테스트
```

### 정규표현식

```
regex101.com
  - 정규표현식 테스트 및 설명
  - n8n Code 노드에서 정규식 사용 전 검증

regexr.com
  - 시각적인 정규식 학습
```

### 크론 표현식

```
crontab.guru
  - 크론 표현식 생성 및 설명
  - "매주 월요일 오전 9시" → 0 9 * * 1
```

---

## 학습 경로별 추천 자료

### 자동화 전략

```
책:
- "The 4-Hour Work Week" - Timothy Ferriss
  자동화 마인드셋 형성

- "Atomic Habits" - James Clear
  작은 자동화를 습관으로

영상:
- YouTube: "Automating your entire life with n8n"
```

### 프롬프트 엔지니어링

```
공식 가이드:
- Anthropic Claude 가이드 (docs.anthropic.com/prompting)
- OpenAI 프롬프트 엔지니어링 가이드

실습:
- promptingguide.ai (한국어 포함)
```

### AI/ML 기초

```
무료 강좌:
- fast.ai - 실용적인 딥러닝
- DeepLearning.AI - Andrew Ng 강좌

책:
- "Hands-On Machine Learning" - Aurélien Géron
```

---

## n8n 관련 뉴스레터

자동화 트렌드를 놓치지 않으려면:

```
Lenny's Newsletter - 제품 및 성장 자동화 사례
The Batch (DeepLearning.AI) - AI 주간 업데이트
n8n Blog RSS - n8n 공식 블로그
```

---

## 한국 자동화 커뮤니티

```
현재 활성화된 한국어 n8n 커뮤니티는 성장 중입니다.

오픈 채팅방, 카카오 오픈톡 등에서
"n8n 한국" "자동화" 키워드로 검색해보세요.

또는 직접 커뮤니티를 만들어서 이 책을 시작점으로
한국 n8n 생태계를 함께 키워가는 것도 좋습니다!
```

---

## 이 책의 업데이트

```
이 책은 wikidocs.net에서 계속 업데이트됩니다.

n8n은 빠르게 발전하는 도구입니다.
새 버전이 나오면 내용을 최신 상태로 유지할 예정입니다.

제안이나 오류 발견 시:
- 댓글로 알려주세요
- GitHub Issues로 제보해주세요
```

---

## 핵심 요약

- 공식 문서 (docs.n8n.io)와 커뮤니티 포럼은 최고의 무료 리소스
- n8n.io/workflows에서 900개 이상의 템플릿 활용
- regex101, crontab.guru 등 보조 도구 활용
- YouTube 영어 채널로 고급 기술 습득
- 한국어 n8n 커뮤니티를 함께 만들어갈 동료 찾기

**다음 레슨**: n8n 핵심 용어 사전으로 책을 마무리합니다.
