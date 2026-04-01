# 레슨 9.2: 버전 관리와 업그레이드

---

## n8n 버전 관리의 중요성

n8n은 활발히 개발되는 오픈소스 프로젝트로, 매주 새 버전이 릴리즈됩니다.

**업그레이드 이유:**
- 보안 취약점 패치
- 새 노드 및 기능 추가
- 버그 수정
- 성능 향상

**업그레이드 위험:**
- Breaking Changes로 기존 워크플로우 오동작
- 노드 API 변경
- DB 스키마 변경 (마이그레이션 필요)

---

## 현재 버전 확인

```bash
# Docker 환경
docker exec n8n n8n --version

# npm 환경
n8n --version

# UI에서 확인
Settings → About n8n
```

---

## 릴리즈 노트 확인

업그레이드 전 반드시 확인:

```
1. GitHub Releases 페이지 확인
   github.com/n8n-io/n8n/releases

2. 각 버전의 "Breaking Changes" 섹션 확인
   - 제거된 노드
   - API 변경사항
   - 필수 마이그레이션 단계

3. 현재 버전에서 목표 버전까지의 모든 릴리즈 확인
   (중간 버전 건너뛰기 주의)
```

---

## 업그레이드 전 준비

```bash
# 1단계: 현재 버전 기록
n8n --version > upgrade-notes.txt
echo "업그레이드 일시: $(date)" >> upgrade-notes.txt

# 2단계: 완전한 백업 생성
./backup-n8n.sh

# 3단계: 중요 워크플로우 목록 확인
# n8n UI에서 모든 활성 워크플로우 메모

# 4단계: 테스트 환경에서 먼저 업그레이드
# (가능하다면)
```

---

## Docker로 업그레이드

가장 안전하고 일반적인 방법:

```bash
# 1. 현재 컨테이너 중지
docker stop n8n

# 2. 새 이미지 다운로드
docker pull n8nio/n8n:latest
# 또는 특정 버전
docker pull n8nio/n8n:1.85.0

# 3. 기존 컨테이너 삭제 (데이터는 볼륨에 있어 안전)
docker rm n8n

# 4. 새 컨테이너 시작 (같은 볼륨 연결)
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_ENCRYPTION_KEY="your-key" \
  n8nio/n8n:latest

# 5. 로그 확인 (마이그레이션 진행 상황)
docker logs -f n8n
```

---

## Docker Compose로 업그레이드

```yaml
# docker-compose.yml에서 버전 명시 (권장)
services:
  n8n:
    image: n8nio/n8n:1.85.0  # latest 대신 명시적 버전
```

```bash
# 버전 변경 후 업그레이드
# 1. docker-compose.yml의 버전 번호 수정
# 2. 업데이트
docker-compose pull
docker-compose up -d

# 로그 확인
docker-compose logs -f n8n
```

---

## npm으로 업그레이드

```bash
# 전역 설치 업그레이드
npm update -g n8n

# 특정 버전 설치
npm install -g n8n@1.85.0

# 설치 후 확인
n8n --version
```

---

## 단계별 업그레이드 (대규모 버전 점프)

현재 버전 1.50 → 최신 1.85로 업그레이드 시:

```bash
# 한 번에 건너뛰지 않고 단계적으로
1.50 → 1.60 → 1.70 → 1.80 → 1.85

# 각 단계마다:
1. 업그레이드 실행
2. 워크플로우 동작 확인
3. 문제 없으면 다음 단계
```

---

## 업그레이드 후 검증

```bash
# 자동 검증 스크립트
#!/bin/bash

N8N_URL="http://localhost:5678"
API_KEY="your-api-key"

echo "=== n8n 업그레이드 후 검증 ==="

# 1. 서비스 접근 가능 확인
if curl -sf "$N8N_URL/healthz" > /dev/null; then
  echo "✅ 서비스 접근 가능"
else
  echo "❌ 서비스 접근 불가"
  exit 1
fi

# 2. API 동작 확인
WORKFLOW_COUNT=$(curl -sf \
  -H "X-N8N-API-KEY: $API_KEY" \
  "$N8N_URL/api/v1/workflows" | jq '.count')

echo "✅ 워크플로우 수: $WORKFLOW_COUNT"

# 3. 활성 워크플로우 확인
ACTIVE_COUNT=$(curl -sf \
  -H "X-N8N-API-KEY: $API_KEY" \
  "$N8N_URL/api/v1/workflows?active=true" | jq '.count')

echo "✅ 활성 워크플로우: $ACTIVE_COUNT"
echo "=== 검증 완료 ==="
```

---

## 롤백 방법

업그레이드 후 문제 발생 시:

```bash
# Docker 롤백 (이전 이미지 태그로)
docker stop n8n
docker rm n8n

# 이전 버전으로 재시작
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n:1.80.0  # 이전 버전

# DB 롤백이 필요한 경우 (드문 상황)
# 백업에서 DB 복구
pg_restore -d n8n n8n-backup-before-upgrade.dump
```

---

## 자동 업그레이드 설정 (n8n Cloud)

n8n Cloud 사용자는 자동 업그레이드 옵션 있음:

```
Settings → Updates
→ "Automatic Updates": Enabled
→ "Update Window": 새벽 2시-4시 (트래픽 적은 시간)
→ "Update Type": Minor versions only (안정성 우선)
```

---

## 버전 고정 전략

프로덕션 환경에서는 버전을 고정하고 계획적으로 업그레이드:

```
개발 환경: n8nio/n8n:latest (최신 버전 테스트)
스테이징:  n8nio/n8n:1.85.0 (검증 버전)
프로덕션:  n8nio/n8n:1.80.0 (안정 버전)

업그레이드 주기: 월 1회 (보안 패치는 즉시)
```

---

## 핵심 요약

- 업그레이드 전 반드시 릴리즈 노트 확인 (Breaking Changes!)
- 반드시 백업 후 업그레이드
- Docker 환경: 이미지 버전만 변경으로 간단 업그레이드
- 대규모 버전 점프는 단계적으로 진행
- 업그레이드 후 자동 검증 스크립트 실행
- 문제 시 이전 이미지 태그로 즉시 롤백

**다음 레슨**: 팀과 함께 n8n을 효율적으로 협업하는 방법을 알아봅니다.
