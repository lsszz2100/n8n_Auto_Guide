# 레슨 3.7: HTTP Request 노드로 API 호출

---

## HTTP Request 노드란?

HTTP Request 노드는 n8n에서 가장 강력하고 다재다능한 노드 중 하나입니다.

전용 노드가 없는 모든 API를 이 노드로 호출할 수 있습니다.
즉, 세상의 모든 API를 n8n에서 사용 가능하게 만드는 만능 도구입니다.

---

## 기본 설정

### Method (메서드)
| 메서드 | 사용 목적 |
|--------|-----------|
| GET | 데이터 조회 |
| POST | 데이터 생성/전송 |
| PUT | 전체 수정 |
| PATCH | 부분 수정 |
| DELETE | 삭제 |

### URL
요청을 보낼 엔드포인트 주소입니다.

```
https://api.example.com/v1/users
https://api.openweathermap.org/data/2.5/weather
```

표현식 사용 가능:
```
https://api.example.com/users/{{ $json.userId }}
```

---

## 인증 방법

### API Key 방식

Header에 추가:
```
Authorization: Bearer {API_KEY}
X-API-Key: {API_KEY}
```

노드 설정:
- Authentication: None
- Headers: `Authorization` = `Bearer {{ $credentials.apiKey }}`

쿼리 파라미터에 추가:
```
https://api.example.com/data?api_key=YOUR_KEY
```

### n8n Credentials 사용 (권장)
1. 별도 크리덴셜 노드에 API 키 저장
2. HTTP Request 노드에서 해당 크리덴셜 선택
3. API 키가 소스코드에 노출되지 않아 안전

---

## 요청 본문 (Body)

POST/PUT/PATCH 요청 시 전송할 데이터를 설정합니다.

### JSON Body

```json
{
  "name": "{{ $json.name }}",
  "email": "{{ $json.email }}",
  "message": "{{ $json.message }}"
}
```

### Form Data

키-값 쌍으로 데이터를 전송합니다.

### Raw Body

직접 텍스트나 XML을 전송할 때 사용합니다.

---

## 헤더 설정

```
Content-Type: application/json
Accept: application/json
User-Agent: n8n/1.0
Authorization: Bearer {{ $json.token }}
```

---

## 쿼리 파라미터

URL에 붙는 `?key=value` 형태의 파라미터입니다.

```
URL: https://api.weather.com/v1/current
파라미터:
  city = {{ $json.city }}
  units = metric
  lang = ko

→ https://api.weather.com/v1/current?city=Seoul&units=metric&lang=ko
```

---

## 실전 예시 1: 날씨 API 호출

OpenWeatherMap API로 현재 날씨를 가져오는 예시:

```
Method: GET
URL: https://api.openweathermap.org/data/2.5/weather
Query Parameters:
  q: Seoul,KR
  appid: {API_KEY}
  units: metric
  lang: kr
```

응답 데이터:
```json
{
  "main": {
    "temp": 15.2,
    "humidity": 65
  },
  "weather": [
    { "description": "맑음" }
  ],
  "wind": {
    "speed": 3.1
  }
}
```

---

## 실전 예시 2: Slack 메시지 발송 (API 직접 호출)

```
Method: POST
URL: https://slack.com/api/chat.postMessage
Headers:
  Authorization: Bearer {{ $credentials.slackToken }}
  Content-Type: application/json
Body (JSON):
{
  "channel": "C1234567890",
  "text": "안녕하세요! {{ $json.name }}님의 주문이 완료되었습니다."
}
```

---

## 실전 예시 3: 카카오 메시지 API 호출

```
Method: POST
URL: https://kapi.kakao.com/v2/api/talk/memo/default/send
Headers:
  Authorization: Bearer {{ $json.accessToken }}
  Content-Type: application/x-www-form-urlencoded
Body (Form Data):
  template_object: {"object_type": "text", "text": "{{ $json.message }}", "link": {}}
```

---

## 응답 처리

### 응답 형식 설정

HTTP Request 노드의 응답은 자동으로 JSON 파싱됩니다.

응답 코드 확인:
- Options → Include Response Headers and Status

응답 구조:
```json
{
  "statusCode": 200,
  "headers": { ... },
  "body": { ... }  // 실제 데이터
}
```

### 에러 응답 처리

4xx, 5xx 상태 코드 처리:
- Settings → "Continue On Fail" 활성화
- 에러 응답도 데이터로 받아 처리 가능

```javascript
// Code 노드에서 상태 코드 확인
if ($json.statusCode === 200) {
  // 성공 처리
} else if ($json.statusCode === 404) {
  // 없는 리소스
} else {
  // 기타 에러
}
```

---

## Pagination (페이지네이션)

대량 데이터를 여러 페이지로 나누어 제공하는 API 처리:

### 오프셋 기반

```
URL: https://api.example.com/items?limit=100&offset={{ $json.offset }}
```

### 커서 기반

```
URL: https://api.example.com/items?cursor={{ $json.nextCursor }}
```

n8n의 HTTP Request 노드에는 자동 Pagination 기능이 있습니다:
- Options → Pagination

---

## Rate Limiting 처리

API 호출 횟수 제한이 있을 때:

```
[HTTP Request] → [Wait: 1초] → [HTTP Request] → ...
```

또는 Loop Over Items로 배치 단위 처리:
```
[Split In Batches: 10개씩] → [HTTP Request] → [Wait: 1초]
```

---

## 자주 쓰는 무료 API 목록

| API | 설명 | URL |
|-----|------|-----|
| OpenWeatherMap | 날씨 정보 | openweathermap.org/api |
| Rest Countries | 국가 정보 | restcountries.com |
| JSONPlaceholder | 테스트용 더미 API | jsonplaceholder.typicode.com |
| ExchangeRate-API | 환율 정보 | exchangerate-api.com |
| NewsAPI | 뉴스 기사 | newsapi.org |
| PokeAPI | 포켓몬 데이터 | pokeapi.co |

---

## 디버깅 팁

### 테스트 방법
1. n8n 워크플로우에서 해당 노드만 실행
2. Postman이나 curl로 먼저 테스트
3. 응답 데이터를 확인하고 표현식 작성

### 자주 발생하는 에러

| 에러 | 원인 | 해결책 |
|------|------|--------|
| 401 Unauthorized | 인증 실패 | API 키 확인 |
| 403 Forbidden | 권한 없음 | 스코프/권한 확인 |
| 404 Not Found | URL 잘못됨 | 엔드포인트 확인 |
| 429 Too Many Requests | Rate Limit | Wait 노드 추가 |
| 500 Internal Server Error | 서버 오류 | 요청 형식 확인 |

---

## 핵심 요약

- HTTP Request 노드는 모든 API를 호출할 수 있는 만능 노드
- GET/POST/PUT/DELETE 4가지 메서드 지원
- 인증은 크리덴셜에 저장하여 재사용 (보안상 권장)
- 자동 Pagination 기능으로 대량 데이터 처리 가능
- Rate Limit이 있으면 Wait 노드로 요청 간격 조절

**다음 레슨**: 다양한 앱들과 실제로 연동하는 방법을 배웁니다.
