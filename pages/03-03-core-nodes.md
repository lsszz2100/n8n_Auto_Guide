# 레슨 3.3: 핵심 노드 탐색

---

## 핵심 노드란?

핵심 노드(Core Nodes)는 특정 앱에 종속되지 않고 데이터 처리, 로직 제어, 흐름 관리에 사용되는 범용 노드들입니다.

이 노드들을 마스터하면 어떤 워크플로우든 만들 수 있게 됩니다.

---

## 1. Edit Fields (Set) — 데이터 생성 및 변환

용도: 데이터 필드 생성, 이름 변경, 값 변환

### 사용 예시

필드 생성:
```
입력: { "name": "홍길동", "email": "hong@test.com" }
설정: fullName = {{ $json.name + " 님" }}
출력: { "name": "홍길동", "email": "hong@test.com", "fullName": "홍길동 님" }
```

필드 이름 변경:
```
입력: { "cust_nm": "홍길동" }
설정: name = {{ $json.cust_nm }}
출력: { "name": "홍길동" }
```

데이터 정리 (필요한 필드만 남기기):
- Mode를 "Manual Mapping"으로 설정
- 필요한 필드만 추가
- Include in Output을 "No Other Fields"로 설정

### 고급 옵션
- Dot notation: 중첩 객체 생성 가능
  - `customer.name` → `{ "customer": { "name": "..." } }`

---

## 2. IF — 조건 분기

용도: 조건에 따라 다른 경로로 데이터 분기

### 구조

```
[IF 노드]
  ├── True 출력 → 조건이 참일 때
  └── False 출력 → 조건이 거짓일 때
```

### 조건 연산자

| 타입 | 연산자 |
|------|--------|
| 문자열 | equals, contains, starts with, ends with, regex |
| 숫자 | equals, greater than, less than, between |
| 불린 | is true, is false |
| 날짜 | before, after, between |
| 배열 | contains, length |

### 복합 조건
- AND: 모든 조건이 참이어야 함
- OR: 하나라도 참이면 됨

### 예시: 구매 금액 기준 분기

```
조건: {{ $json.amount }} Greater Than 50000

True → VIP 처리
False → 일반 처리
```

---

## 3. Switch — 다중 분기

용도: 하나의 값에 따라 여러 경로로 분기 (IF의 확장)

### IF vs Switch

| 상황 | 노드 |
|------|------|
| 2가지 경우 (참/거짓) | IF |
| 3가지 이상 경우 | Switch |

### 예시: 이메일 유형 분기

```
Switch 노드:
- 조건 1: subject contains "주문" → 주문 처리 경로
- 조건 2: subject contains "환불" → 환불 처리 경로
- 조건 3: subject contains "문의" → CS팀 경로
- Default → 기타 경로
```

---

## 4. Merge — 데이터 합치기

용도: 여러 출처의 데이터를 하나로 합치기

두 개의 입력 포트 (Input 1, Input 2)

### 합치기 모드

| 모드 | 설명 | 사용 예시 |
|------|------|-----------|
| Merge by Index | 순서대로 합치기 | 같은 순서의 데이터 병합 |
| Merge by Key | 공통 키로 매칭하여 합치기 | 이메일로 고객 정보 + 구매 이력 합치기 |
| Append | 단순히 이어 붙이기 | 두 리스트 합치기 |
| Keep Key Matches Only | 매칭된 것만 유지 | Inner Join |

### 예시: 고객 데이터 + 구매 이력 합치기

```
Input 1 (Webhook): { "email": "hong@test.com", "name": "홍길동" }
Input 2 (Google Sheets): { "email": "hong@test.com", "purchases": 5 }

Merge by Key: email

Output: { "email": "hong@test.com", "name": "홍길동", "purchases": 5 }
```

---

## 5. Split Out — 배열을 개별 아이템으로 분리

용도: 하나의 아이템 안에 있는 배열을 여러 개의 아이템으로 변환

### 예시

```
입력 (1개 아이템):
{ "users": ["홍길동", "김철수", "이영희"] }

Split Out: users 필드 기준

출력 (3개 아이템):
{ "users": "홍길동" }
{ "users": "김철수" }
{ "users": "이영희" }
```

각 아이템을 개별적으로 처리해야 할 때 사용합니다.

---

## 6. Aggregate — 여러 아이템을 하나로 묶기

용도: Split Out의 반대. 여러 아이템을 하나의 아이템으로 합치기

### 예시: 처리된 이메일들을 하나의 요약으로 만들기

```
입력 (3개 아이템):
{ "email": "hong@test.com", "status": "sent" }
{ "email": "kim@test.com", "status": "sent" }
{ "email": "lee@test.com", "status": "failed" }

Aggregate: email 필드를 배열로 수집

출력 (1개 아이템):
{ "email": ["hong@test.com", "kim@test.com", "lee@test.com"] }
```

---

## 7. Filter — 조건에 맞는 아이템만 통과

용도: 특정 조건을 만족하는 아이템만 다음으로 넘기기

### 예시: 활성 사용자만 필터링

```
입력:
{ "name": "홍길동", "active": true }
{ "name": "김철수", "active": false }
{ "name": "이영희", "active": true }

Filter: active equals true

출력:
{ "name": "홍길동", "active": true }
{ "name": "이영희", "active": true }
```

---

## 8. Sort — 데이터 정렬

용도: 특정 필드를 기준으로 아이템 정렬

```
예: 구매 금액 높은 순서로 정렬
    날짜 오래된 순서로 정렬
```

---

## 9. Limit — 처리 개수 제한

용도: 지정한 수만큼만 처리

```
예: 처음 10개 아이템만 처리
    무한 루프 방지
```

---

## 10. Loop Over Items — 반복 처리

용도: 대량의 아이템을 배치(batch) 단위로 처리

API 호출 횟수 제한이 있을 때 일정 단위로 나누어 처리하거나, 각 아이템에 대해 순차적으로 작업을 수행할 때 사용합니다.

---

## 11. Wait — 일시 정지

용도: 일정 시간 대기 후 다음 노드 실행

```
사용 예시:
- API 호출 사이에 1초 대기 (Rate Limit 방지)
- 이메일 발송 후 1시간 뒤 팔로업 체크
```

---

## 12. No Operation (No Op) — 아무것도 안 하기

용도: 플레이스홀더, 디버깅, 특정 분기의 종점

```
사용 예시: IF 노드에서 False 경로를 일단 No Op으로 연결하고 나중에 개발
```

---

## 핵심 노드 조합 패턴

### 패턴 1: 필터 → 처리
```
[Webhook] → [Filter: 활성 사용자만] → [Gmail: 이메일 발송]
```

### 패턴 2: 조건 분기 → 각각 처리
```
[Gmail Trigger] → [IF: VIP 여부] → True: [VIP 처리]
                                  → False: [일반 처리]
```

### 패턴 3: 다중 소스 합치기
```
[Webhook] ──────────►[Merge]──►[처리]
[Google Sheets] ────►
```

---

## 핵심 요약

| 노드 | 주요 용도 |
|------|-----------|
| Edit Fields | 데이터 생성/변환/이름 변경 |
| IF | 2분기 조건 처리 |
| Switch | 다중 분기 처리 |
| Merge | 여러 소스 데이터 합치기 |
| Split Out | 배열을 개별 아이템으로 |
| Aggregate | 아이템들을 하나로 묶기 |
| Filter | 조건 맞는 아이템만 통과 |
| Loop Over Items | 배치 단위 반복 처리 |
| Wait | 대기 시간 설정 |

**다음 레슨**: n8n에서 데이터가 어떤 형태로 흐르는지 JSON을 이해합니다.
