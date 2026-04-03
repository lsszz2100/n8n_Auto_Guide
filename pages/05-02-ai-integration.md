# 레슨 5.2: n8n과 AI 연동하기

---

## 지원하는 AI 제공업체

n8n은 주요 AI 제공업체를 모두 지원합니다.

| 제공업체 | 주요 모델 | 특징 |
|---------|---------|------|
| OpenAI | GPT-4o, GPT-4o mini, o1 | 가장 광범위하게 사용, 풍부한 생태계 |
| Anthropic | Claude Sonnet 4, Claude Haiku 4 | 긴 컨텍스트, 정교한 추론 |
| Google AI | Gemini 1.5 Pro, Gemini Flash | 멀티모달, 구글 생태계 연동 |
| Mistral | Mistral Large, Mixtral | 오픈소스 기반, 비용 효율 |
| Ollama | Llama 3, Mistral 등 | 완전 로컬 실행, 무료 |
| Azure OpenAI | GPT-4o (Azure 호스팅) | 기업 보안, 규정 준수 |

---

## OpenAI 연동

### Step 1: API 키 발급

1. [platform.openai.com](https://platform.openai.com) 접속
2. 우측 상단 계정 아이콘 → **API Keys** 클릭
3. **Create new secret key** 클릭
4. 이름 입력 후 생성 → 키를 반드시 복사 (다시 볼 수 없음)
5. **결제 수단 등록**: Settings → Billing → Add payment method

### Step 2: n8n에 Credential 추가

```
1. n8n 좌측 메뉴 → Credentials → Add Credential
2. "OpenAI" 검색 → 선택
3. API Key 붙여넣기
4. Save 클릭
5. 초록색 체크 표시 확인
```

### Step 3: 워크플로우에서 사용

```
AI Agent 노드 추가 → Chat Model 선택 →
"OpenAI Chat Model" 추가 → Credential 선택 →
Model: gpt-4o 선택
```

### OpenAI 모델 선택 가이드

```
GPT-4o           → 가장 강력, 멀티모달(이미지), 비용 중간
GPT-4o mini      → 빠르고 저렴, 일반 작업에 충분
o1-preview       → 복잡한 추론, 수학/코딩, 느리고 비쌈
o1-mini          → 추론 특화, o1보다 빠르고 저렴
GPT-3.5 Turbo    → 가장 저렴, 단순 작업
```

---

## Anthropic 연동

### Step 1: API 키 발급

1. [console.anthropic.com](https://console.anthropic.com) 접속
2. 회원가입/로그인
3. **Get API Keys** → **Create Key**
4. 키 복사 (sk-ant-로 시작)
5. 결제 수단 등록 필수

### Step 2: n8n에 Credential 추가

```
1. Credentials → Add Credential
2. "Anthropic" 검색 → 선택
3. API Key 입력
4. Save
```

### Anthropic 모델 선택 가이드

```
Claude Opus 4       → 최고 성능, 복잡한 추론
Claude Sonnet 4     → 균형 잡힌 성능/비용, 일반 작업 권장
Claude Haiku 4      → 가장 빠르고 저렴, 단순 작업
```

**Claude의 강점:**
- 200K 토큰 컨텍스트 윈도우 (약 150,000단어)
- 긴 문서 요약, 대규모 코드베이스 분석에 탁월
- 안전하고 도움이 되는 답변

---

## Google AI 연동

### Step 1: API 키 발급

1. [aistudio.google.com](https://aistudio.google.com) 접속
2. **Get API key** → **Create API key**
3. Google Cloud 프로젝트 선택 또는 새로 생성
4. 키 복사 (AIza로 시작)

> **참고**: Google AI Studio의 Gemini API는 일정 사용량까지 무료입니다.

### Step 2: n8n에 Credential 추가

```
1. Credentials → Add Credential
2. "Google AI" 또는 "Gemini" 검색 → 선택
3. API Key 입력
4. Save
```

### Google AI 모델 선택 가이드

```
Gemini 1.5 Pro     → 가장 강력, 1M 토큰 컨텍스트, 멀티모달
Gemini 1.5 Flash   → 빠르고 저렴, Gemini Pro의 경량 버전
Gemini 1.0 Pro     → 이전 버전, 안정적
```

---

## Ollama (로컬 LLM) 연동

비용 없이 로컬에서 AI를 실행하고 싶다면 Ollama를 사용하세요.

### Step 1: Ollama 설치 및 모델 다운로드

```bash
# Ollama 설치 (macOS/Linux)
curl -fsSL https://ollama.ai/install.sh | sh

# 모델 다운로드 (예: Llama 3)
ollama pull llama3

# 실행 확인
ollama list
```

### Step 2: n8n에서 연동

```
1. Credentials → Add Credential → "Ollama" 선택
2. Base URL: http://localhost:11434 (기본값)
3. Save
```

> **주의**: n8n이 Docker에서 실행 중이라면 `http://host.docker.internal:11434` 사용

### 추천 로컬 모델

```
llama3:8b      → 일반 대화, 8GB RAM 필요
llama3:70b     → 고성능, 40GB RAM 필요
mistral:7b     → 빠른 응답, 8GB RAM 필요
codellama      → 코드 특화
phi3:mini      → 초경량, 4GB RAM으로 가능
```

---

## 모델 선택 가이드

### 작업별 추천 모델

| 작업 유형 | 추천 모델 | 이유 |
|---------|---------|------|
| 일반 텍스트 처리 | GPT-4o mini | 속도/비용/성능 균형 |
| 긴 문서 요약 | Claude Sonnet 4 | 200K 컨텍스트 |
| 이미지 분석 | GPT-4o 또는 Gemini 1.5 Pro | 멀티모달 |
| 코드 생성/분석 | Claude Sonnet 4 | 코딩 성능 우수 |
| 대량 처리 (비용 중시) | GPT-4o mini 또는 Gemini Flash | 가장 저렴 |
| 보안 민감 데이터 | Ollama (로컬) | 외부 전송 없음 |
| 복잡한 추론 | o1-mini | 추론 특화 |

### 성능/비용 매트릭스

```
       낮은 비용 ←─────────────────→ 높은 비용
           │                              │
높은 성능  │  Claude Sonnet 4      GPT-4o │
           │  Gemini 1.5 Pro              │
           │                              │
낮은 성능  │  GPT-4o mini         o1      │
           │  Gemini Flash        GPT-4   │
           │  Ollama(로컬)                │
```

---

## 비용 관리 팁

### 1. 토큰(Token) 이해하기

```
1 토큰 ≈ 영어 4글자 ≈ 한국어 1~2글자

"안녕하세요" = 약 4 토큰
100 단어 영어 ≈ 75 토큰

비용 = (입력 토큰 + 출력 토큰) × 단가
```

### 2. 실제 비용 예시 (참고용, 변동 가능)

```
GPT-4o:
  입력: $2.50 / 1M 토큰
  출력: $10.00 / 1M 토큰

GPT-4o mini:
  입력: $0.15 / 1M 토큰
  출력: $0.60 / 1M 토큰

Claude Sonnet 4:
  입력: $3.00 / 1M 토큰
  출력: $15.00 / 1M 토큰
```

### 3. 비용 절약 전략

**전략 1: 모델 티어링**
```
단순 분류/추출 → GPT-4o mini (저렴)
복잡한 분석   → GPT-4o (중간)
창의적 작업   → Claude Sonnet 4 (필요할 때만)
```

**전략 2: 프롬프트 최적화**
```
나쁜 예: 엄청나게 긴 시스템 프롬프트 (매 요청마다 비용 발생)
좋은 예: 핵심만 담은 간결한 프롬프트
```

**전략 3: 캐싱 활용**
```
동일한 요청이 반복되는 경우:
[AI 요청] → [결과를 Redis/DB에 저장]
다음 번에: [캐시 확인] → 있으면 캐시 반환, 없으면 AI 호출
```

**전략 4: 사용량 모니터링**
```
OpenAI: platform.openai.com → Usage
Anthropic: console.anthropic.com → Usage
n8n: 실행 로그에서 토큰 수 확인 가능
```

**전략 5: 테스트는 저렴한 모델로**
```
개발/테스트: GPT-4o mini 또는 Ollama 사용
프로덕션: 필요한 경우에만 고성능 모델로 전환
```

---

## 프롬프트 엔지니어링 기초

좋은 프롬프트가 좋은 결과를 만듭니다. 모델을 바꾸기 전에 프롬프트를 먼저 개선하세요.

### 효과적인 프롬프트의 4가지 원칙

**1. 역할(Role) 부여**
```
나쁜 예: "이메일을 분류해줘"
좋은 예: "당신은 B2B SaaS 고객 지원 전문가입니다. 고객 이메일의 유형을 분류해주세요."
```

**2. 구체적인 출력 형식 지정**
```
나쁜 예: "결과를 정리해줘"
좋은 예: |
  다음 JSON 형식으로만 응답해:
  {
    "category": "billing|technical|general",
    "priority": "high|medium|low",
    "summary": "한 줄 요약"
  }
```

**3. 예시(Few-shot) 제공**
```
입력 예시: "결제가 두 번 됐어요"
출력 예시: {"category": "billing", "priority": "high", "summary": "중복 결제 문제"}

입력 예시: "비밀번호를 잊었어요"
출력 예시: {"category": "technical", "priority": "low", "summary": "비밀번호 분실"}

이제 다음을 분류해: {{ $json.message }}
```

**4. 제약 조건 명시**
```
규칙:
- 반드시 한국어로 응답
- 응답은 3문장 이내
- 확실하지 않은 내용은 추측하지 말고 "확인 필요"라고 표시
```

---

## 프롬프트 캐싱 (비용 절약)

반복되는 긴 시스템 프롬프트는 캐싱으로 비용을 크게 줄일 수 있습니다.

### Claude의 캐시 프리픽스 기능

Anthropic Claude는 반복 사용되는 프롬프트 앞부분을 캐시에 저장합니다.

```javascript
// HTTP Request 노드로 직접 API 호출 시 캐싱 활성화
{
  "model": "claude-sonnet-4-5",
  "system": [
    {
      "type": "text",
      "text": "당신은 전문 고객 지원 에이전트입니다...(긴 시스템 프롬프트)",
      "cache_control": { "type": "ephemeral" }  // 캐시 활성화
    }
  ],
  "messages": [{ "role": "user", "content": "{{ $json.message }}" }]
}
```

캐시된 토큰은 일반 입력 토큰 대비 약 90% 저렴합니다.

### n8n 캐싱 패턴 (모든 모델 적용 가능)

```
[AI 요청 준비]
    ↓
[Code: 캐시 키 생성]
  cacheKey = hash(systemPrompt + userInput)
    ↓
[Redis: 캐시 조회]
    ↓
[IF: 캐시 있음?]
  True → [캐시된 결과 반환]
  False → [AI 호출] → [Redis: 결과 저장 (TTL: 1시간)]
```

---

## 여러 AI를 동시에 활용하는 패턴

n8n에서는 여러 AI를 함께 사용할 수 있습니다.

```
[입력 텍스트]
     ↓
[GPT-4o mini로 1차 분류]  (저렴한 비용)
     ↓
[일반 문의] → [Rule-based 처리]
[복잡한 문의] → [Claude Sonnet 4로 정교한 답변]
[이미지 포함] → [GPT-4o로 이미지+텍스트 처리]
```

---

## 핵심 요약

- OpenAI, Anthropic, Google AI 모두 n8n Credentials에서 간단히 연동
- 일반 작업: GPT-4o mini 또는 Gemini Flash로 비용 절약
- 긴 문서: Claude Sonnet 4의 200K 컨텍스트 활용
- 보안 데이터: Ollama로 로컬 실행
- 토큰 = 비용이므로 프롬프트를 간결하게, 작업에 맞는 모델 선택
- 좋은 프롬프트: 역할 부여 + 출력 형식 지정 + Few-shot 예시 + 제약 조건
- Claude 캐시 프리픽스로 반복 시스템 프롬프트 비용 최대 90% 절약

**다음 레슨**: AI Agent 노드를 활용해 실제 에이전트를 만드는 방법을 배웁니다.
