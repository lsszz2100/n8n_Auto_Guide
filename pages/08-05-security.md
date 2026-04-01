# 레슨 8.5: 보안 강화

---

## n8n 보안의 중요성

n8n은 이메일, 데이터베이스, API, 파일 시스템 등 민감한 시스템에 접근합니다. 잘못된 보안 설정은 기업 데이터 유출, 무단 자동화 실행으로 이어질 수 있습니다.

**주요 보안 위협:**
- 인증되지 않은 웹훅 호출
- 크리덴셜 노출
- 과도한 접근 권한
- 주입 공격 (Injection)
- 데이터 평문 전송

---

## 1. 인증 및 접근 제어

### 기본 인증 설정

```env
# 필수: n8n 기본 인증 활성화
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=strong-password-here

# 또는 JWT 인증
N8N_JWT_AUTH_ACTIVE=true
N8N_JWT_AUTH_HEADER=Authorization
N8N_JWT_AUTH_HEADER_VALUE_PREFIX=Bearer
N8N_JWT_SECRET=your-jwt-secret
```

### 사용자 역할 관리 (n8n Enterprise)

```
역할 구조:
├── 오너 (Owner)
│   └── 모든 워크플로우, 크리덴셜, 사용자 관리
├── 관리자 (Admin)
│   └── 워크플로우, 크리덴셜 관리 가능
├── 편집자 (Editor)
│   └── 자신의 워크플로우만 편집
└── 뷰어 (Viewer)
    └── 읽기 전용
```

### IP 허용 목록 (방화벽)

```bash
# UFW로 n8n 포트 제한
ufw allow from 10.0.0.0/8 to any port 5678  # 사내 네트워크만
ufw allow from 203.0.113.0 to any port 5678  # 특정 IP만
ufw deny 5678  # 나머지 모두 차단
```

---

## 2. 웹훅 보안

외부에서 호출되는 웹훅은 특히 주의가 필요합니다.

### 시크릿 토큰 검증

```
[Webhook 트리거]
    ↓
[Code: 서명 검증]
```

```javascript
// Code 노드: HMAC 서명 검증
const crypto = require('crypto');

const receivedSignature = $json.headers['x-signature'];
const payload = JSON.stringify($json.body);
const secret = $env.WEBHOOK_SECRET;

const expectedSignature = 'sha256=' + crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

if (receivedSignature !== expectedSignature) {
  throw new Error('Invalid webhook signature');
}

return $input.all();
```

### API 키 헤더 검증

```
[Webhook]
    ↓
[IF: $json.headers['x-api-key'] === $env.WEBHOOK_API_KEY]
  True  → [처리 계속]
  False → [HTTP Response: 401 Unauthorized]
```

---

## 3. 크리덴셜 보안

### 안전한 크리덴셜 관리

```
✅ 올바른 방법:
- n8n Credentials에 저장 (암호화됨)
- n8n Variables에 환경변수로 저장
- 워크플로우 내에서 {{ $credentials.xxx }}로 참조

❌ 잘못된 방법:
- Code 노드에 API 키 하드코딩
- 워크플로우 JSON에 키 직접 입력
- 로그에 민감 정보 출력
```

### 환경 변수로 비밀 관리

```env
# .env 파일 (Git에 절대 커밋 금지!)
DB_PASSWORD=super-secret-db-password
API_KEY_OPENAI=sk-...
WEBHOOK_SECRET=random-32-char-string

# n8n에서 사용
{{ $env.DB_PASSWORD }}
```

```bash
# .gitignore 필수
echo ".env" >> .gitignore
echo "*.env" >> .gitignore
```

---

## 4. 데이터 보안

### 민감 데이터 마스킹

```javascript
// 로그 출력 시 민감 정보 제거
function sanitizeForLog(data) {
  const sensitiveFields = ['password', 'apiKey', 'token', 'ssn', 'creditCard'];
  const sanitized = { ...data };

  sensitiveFields.forEach(field => {
    if (sanitized[field]) {
      sanitized[field] = '***MASKED***';
    }
  });

  return sanitized;
}

console.log('처리 데이터:', JSON.stringify(sanitizeForLog($json)));
```

### 개인정보 처리 주의

```javascript
// GDPR/개인정보보호법 준수
// 불필요한 개인정보 즉시 제거
const processedData = {
  orderId: $json.orderId,
  amount: $json.amount,
  // email은 처리 후 즉시 제거
  // name, phone 등 불필요한 PII 제외
};
```

---

## 5. 네트워크 보안

### HTTPS 강제 적용

```env
# 외부 접근 URL
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
N8N_PORT=443

# SSL 인증서 (Let's Encrypt)
SSL_KEY=/path/to/privkey.pem
SSL_CERT=/path/to/fullchain.pem
```

### Nginx 리버스 프록시 설정

```nginx
# /etc/nginx/sites-available/n8n
server {
    listen 80;
    server_name n8n.yourcompany.com;
    return 301 https://$server_name$request_uri;  # HTTP → HTTPS 리다이렉트
}

server {
    listen 443 ssl http2;
    server_name n8n.yourcompany.com;

    ssl_certificate /etc/letsencrypt/live/n8n.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourcompany.com/privkey.pem;

    # 보안 헤더
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=n8n:10m rate=10r/s;
    limit_req zone=n8n burst=20 nodelay;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}
```

---

## 6. 코드 노드 보안

```javascript
// ❌ 위험: 사용자 입력을 eval로 실행
const result = eval($json.userInput);  // 코드 주입 위험!

// ✅ 안전: 입력값 검증 후 사용
const allowedValues = ['option1', 'option2', 'option3'];
if (!allowedValues.includes($json.userChoice)) {
  throw new Error('Invalid choice');
}

// ❌ 위험: 사용자 입력을 SQL에 직접 삽입
const query = `SELECT * FROM users WHERE name = '${$json.name}'`;

// ✅ 안전: 파라미터화된 쿼리 사용
const query = 'SELECT * FROM users WHERE name = $1';
const params = [$json.name];
```

---

## 7. 감사 로그 (Audit Log)

```
[모든 중요 작업 로깅]

워크플로우 실행 시:
- 누가 (사용자/API 키)
- 언제 (타임스탬프)
- 무엇을 (워크플로우 이름)
- 어디서 (IP 주소)
- 결과 (성공/실패)
```

```javascript
// 감사 로그 기록 코드
const auditLog = {
  timestamp: new Date().toISOString(),
  workflowName: $workflow.name,
  userId: $execution.resumeUrl ? 'webhook' : 'manual',
  action: $json.action,
  targetId: $json.targetId,
  status: 'started',
  ip: $json.headers?.['x-forwarded-for'] || 'unknown'
};

// DB에 감사 로그 저장
await $node["DB: 감사 로그 저장"].execute([{ json: auditLog }]);
```

---

## 보안 체크리스트

### 기본 보안
- [ ] 강력한 비밀번호 + MFA 설정
- [ ] HTTPS 적용
- [ ] 불필요한 포트 방화벽 차단
- [ ] 정기적 비밀번호/API 키 교체

### 크리덴셜 보안
- [ ] 모든 API 키는 n8n Credentials에만 저장
- [ ] 코드에 하드코딩된 비밀 없음
- [ ] .env 파일 .gitignore에 추가

### 웹훅 보안
- [ ] 모든 공개 웹훅에 서명 검증 또는 API 키 검증
- [ ] Rate limiting 적용
- [ ] 입력 데이터 검증

### 데이터 보안
- [ ] 개인정보 최소 수집
- [ ] 민감 데이터 마스킹
- [ ] 불필요한 실행 이력 정기 삭제

---

## 핵심 요약

- 인증 강화: 기본 인증 + IP 화이트리스트 + MFA
- 웹훅 보안: HMAC 서명 검증으로 위조 요청 차단
- 크리덴셜 보안: n8n Credentials에만 저장, 코드에 절대 하드코딩 금지
- HTTPS 필수 적용: Nginx 리버스 프록시로 SSL 처리
- 코드 주입 방지: 사용자 입력 검증, 파라미터화 쿼리 사용
- 감사 로그로 모든 중요 작업 기록

**다음 챕터**: n8n 운영 — 백업, 업그레이드, 팀 협업, 모니터링
