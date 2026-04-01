# 레슨 4.2: 트리거 종류와 실전 활용

---

## 트리거 고급 활용

3장에서 트리거의 종류를 배웠다면, 이번 레슨에서는 **실전에서 어떻게 활용하는지** 상세히 다룹니다.

---

## Schedule Trigger 고급 활용

### Cron 표현식 완전 가이드

```
┌───────────── 분 (0-59)
│ ┌───────────── 시 (0-23)
│ │ ┌───────────── 일 (1-31)
│ │ │ ┌───────────── 월 (1-12)
│ │ │ │ ┌───────────── 요일 (0-6, 0=일요일)
│ │ │ │ │
* * * * *
```

#### 실전 Cron 표현식 모음

| 설명 | 표현식 |
|------|--------|
| 매 5분마다 | `*/5 * * * *` |
| 매시간 정각 | `0 * * * *` |
| 평일 오전 9시 | `0 9 * * 1-5` |
| 매일 자정 | `0 0 * * *` |
| 매주 월요일 오전 8시 | `0 8 * * 1` |
| 매월 1일 오전 9시 | `0 9 1 * *` |
| 매 30분마다 (업무시간) | `*/30 9-18 * * 1-5` |
| 분기 시작일 | `0 9 1 1,4,7,10 *` |

### 실전 예시: 일일 리포트 자동화

```
Schedule: 0 9 * * 1-5 (평일 오전 9시)
    ↓
[Google Sheets: 어제 데이터 조회]
    ↓
[Code: 통계 계산]
    ↓
[Slack: 채널에 리포트 발송]
```

---

## Webhook Trigger 심화

### URL 구조 이해

```
https://your-n8n.com/webhook/{path}
```

- **path**: 자유롭게 설정 가능 (영문, 숫자, 하이픈)
- 각 워크플로우마다 다른 path 사용 권장

### Webhook 보안

#### 방법 1: Header 인증
발신자가 헤더에 비밀 키를 포함하여 전송:
```
X-Secret-Key: my-secret-key-12345
```

n8n에서 확인:
```javascript
// Code 노드에서 검증
const secretKey = $request.headers['x-secret-key'];
if (secretKey !== 'my-secret-key-12345') {
  throw new Error('Unauthorized');
}
```

#### 방법 2: n8n Basic Auth
Webhook 노드 설정:
- Authentication: Basic Auth
- Username/Password 설정

#### 방법 3: JWT 검증
더 강력한 보안이 필요한 경우 JWT 토큰 검증 사용

### Webhook 응답 커스터마이즈

Webhook 노드 설정:
- **Response Mode**: Last Node (마지막 노드의 결과를 응답으로 반환)
- **Response Code**: HTTP 상태 코드
- **Response Data**: 반환할 데이터

```javascript
// Respond to Webhook 노드에서 커스텀 응답
{
  "success": true,
  "message": "주문이 접수되었습니다",
  "orderId": "{{ $json.orderId }}"
}
```

---

## 다중 트리거 패턴

하나의 워크플로우를 여러 방법으로 시작할 수 없지만, 같은 서브 워크플로우를 다양한 트리거에서 호출하는 패턴을 사용할 수 있습니다.

### 패턴: 공통 처리 로직 재사용

```
워크플로우 A (Webhook 트리거) ──────────►[공통 처리 Sub-Workflow]
워크플로우 B (Schedule 트리거) ─────────►
워크플로우 C (Gmail 트리거) ────────────►
```

---

## 트리거별 비용/효율 분석

### Polling 트리거의 실행 횟수 계산

Gmail Trigger (5분 폴링):
- 1시간: 12회
- 1일: 288회
- 1달: 8,640회

이 워크플로우가 Cloud에서 운영된다면 Starter 플랜(2,500회/월)을 1달도 안 돼서 소진합니다.

### Webhook으로 전환 시 절약

Polling → Webhook으로 바꾸면:
- 이메일이 실제로 왔을 때만 실행
- 하루 20통 이메일 기준: 월 600회 (vs 8,640회)
- **93% 절약**

---

## 스마트 트리거 패턴

### 패턴 1: 중복 방지 트리거

같은 데이터가 여러 번 처리되는 것을 방지:

```javascript
// Code 노드에서 처리 여부 확인
const processedIds = await $getWorkflowStaticData('global');
const currentId = $json.orderId;

if (processedIds.includes(currentId)) {
  return []; // 이미 처리된 경우 스킵
}

processedIds.push(currentId);
await $setWorkflowStaticData('global', processedIds);

return { json: $json };
```

### 패턴 2: 조건부 트리거

Schedule로 트리거하되, 특정 조건에서만 실제 처리:

```
[Schedule: 5분마다] → [Redis/DB: 처리할 데이터 있는지 확인]
                         ↓ 있음
                      [실제 처리]
                         ↓ 없음
                      [종료]
```

---

## 핵심 요약

- Schedule: Cron 표현식으로 정밀한 실행 타이밍 제어
- Webhook: 보안 헤더나 Basic Auth로 인증 추가
- Polling → Webhook 전환으로 실행 횟수 대폭 절약
- 공통 로직은 Sub-Workflow로 분리하여 여러 트리거에서 재사용
- 중복 처리 방지 패턴으로 안정성 향상

**다음 레슨**: IF와 Switch로 복잡한 조건 처리를 마스터합니다.
