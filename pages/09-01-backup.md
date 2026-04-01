# 레슨 9.1: 백업과 복구

---

## 백업해야 할 것들

n8n에서 잃어버리면 안 되는 데이터:

| 데이터 | 위치 | 중요도 |
|--------|------|--------|
| 워크플로우 정의 | DB (sqlite/postgres) | ⭐⭐⭐ 매우 높음 |
| 크리덴셜 | DB (암호화) | ⭐⭐⭐ 매우 높음 |
| 실행 이력 | DB | ⭐⭐ 중간 |
| 환경 설정 (.env) | 파일 시스템 | ⭐⭐⭐ 매우 높음 |
| 커스텀 노드 | 파일 시스템 | ⭐⭐ 중간 |

---

## SQLite 백업 (기본 설정)

n8n 기본 설정은 SQLite를 사용합니다.

### 수동 백업

```bash
# n8n 데이터 디렉토리 위치
ls ~/.n8n/

# SQLite DB 파일 백업
cp ~/.n8n/database.sqlite ~/backups/n8n-$(date +%Y%m%d).sqlite

# 전체 .n8n 디렉토리 백업
tar -czf ~/backups/n8n-full-$(date +%Y%m%d).tar.gz ~/.n8n/
```

### 자동 백업 스크립트

```bash
#!/bin/bash
# /usr/local/bin/backup-n8n.sh

BACKUP_DIR="/mnt/backup/n8n"
DATE=$(date +%Y%m%d_%H%M%S)
N8N_DIR="$HOME/.n8n"

# 백업 디렉토리 생성
mkdir -p "$BACKUP_DIR"

# n8n 데이터 압축 백업
tar -czf "$BACKUP_DIR/n8n-$DATE.tar.gz" "$N8N_DIR"

# 30일 이상 된 백업 삭제
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete

echo "백업 완료: n8n-$DATE.tar.gz"
```

```bash
# crontab으로 매일 새벽 3시 자동 실행
crontab -e

# 추가할 내용:
0 3 * * * /usr/local/bin/backup-n8n.sh >> /var/log/n8n-backup.log 2>&1
```

---

## PostgreSQL 백업 (권장)

프로덕션 환경에서는 PostgreSQL 사용을 권장합니다.

### pg_dump로 백업

```bash
#!/bin/bash
# PostgreSQL 백업 스크립트

DB_HOST="localhost"
DB_PORT="5432"
DB_NAME="n8n"
DB_USER="n8n"
BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

# 백업 실행
PGPASSWORD="$DB_PASSWORD" pg_dump \
  -h "$DB_HOST" \
  -p "$DB_PORT" \
  -U "$DB_USER" \
  -d "$DB_NAME" \
  --format=custom \
  --compress=9 \
  -f "$BACKUP_DIR/n8n-$DATE.dump"

# S3에 업로드 (AWS CLI 설치 필요)
aws s3 cp "$BACKUP_DIR/n8n-$DATE.dump" \
  "s3://your-backup-bucket/n8n/$DATE/"

echo "PostgreSQL 백업 완료"
```

---

## n8n 내부에서 자동 백업 워크플로우

n8n으로 n8n 자신을 백업하는 워크플로우:

```
[Schedule: 매일 새벽 3시]
    ↓
[n8n API: 모든 워크플로우 JSON 조회]
  GET /api/v1/workflows
    ↓
[Code: 백업 데이터 구성]
  const backup = {
    timestamp: new Date().toISOString(),
    workflows: $json.data,
    count: $json.data.length
  };
    ↓
[Google Drive: 백업 파일 저장]
  File Name: n8n-backup-{{ $now.format('YYYY-MM-DD') }}.json
  Content: {{ JSON.stringify($json.backup) }}
    ↓
[IF: 저장 성공?]
  True  → [Slack: ✅ 백업 완료 알림]
  False → [Slack: ❌ 백업 실패 알림]
```

---

## 워크플로우 내보내기/가져오기

### 개별 워크플로우 내보내기

```
n8n UI → 워크플로우 선택
→ 오른쪽 상단 메뉴 (...) → Download
→ workflow.json 파일 저장
```

### 전체 워크플로우 내보내기

```bash
# n8n CLI 사용
n8n export:workflow --all --output=./workflows-backup/

# 특정 워크플로우만
n8n export:workflow --id=abc123 --output=./workflow.json
```

### 워크플로우 가져오기

```bash
# 파일에서 가져오기
n8n import:workflow --input=./workflows-backup/

# n8n UI에서 가져오기
워크플로우 목록 → 우상단 "Import" 버튼
→ JSON 파일 선택 또는 JSON 직접 붙여넣기
```

---

## Git으로 워크플로우 버전 관리

```bash
# 워크플로우를 Git으로 관리하는 스크립트
#!/bin/bash

WORKFLOWS_DIR="./n8n-workflows"
mkdir -p "$WORKFLOWS_DIR"

# 모든 워크플로우 내보내기
n8n export:workflow --all --output="$WORKFLOWS_DIR"

# Git 커밋
cd "$WORKFLOWS_DIR"
git add .
git commit -m "워크플로우 자동 백업: $(date '+%Y-%m-%d %H:%M')"
git push origin main

echo "Git 백업 완료"
```

---

## 복구 절차

### 시나리오: 서버가 완전히 망가진 경우

```bash
# 1. 새 서버에 n8n 설치
docker pull n8nio/n8n

# 2. 백업 파일 복원
tar -xzf n8n-backup-20260401.tar.gz -C /

# 3. n8n 시작
docker run -d \
  --name n8n \
  -v ~/.n8n:/home/node/.n8n \
  -p 5678:5678 \
  n8nio/n8n

# 4. 워크플로우 및 크리덴셜 확인
# 브라우저에서 접속 후 확인
```

### 시나리오: 특정 워크플로우만 복구

```bash
# 백업에서 특정 워크플로우 JSON 추출 후 가져오기
n8n import:workflow --input=./backup/workflow-email-automation.json
```

### PostgreSQL 복구

```bash
# DB 복구
PGPASSWORD="$DB_PASSWORD" pg_restore \
  -h localhost \
  -U n8n \
  -d n8n \
  --clean \
  n8n-20260401.dump
```

---

## 백업 전략 요약

| 빈도 | 방법 | 보관 기간 |
|------|------|-----------|
| 매일 | 자동 DB 스냅샷 | 30일 |
| 매주 | 전체 시스템 백업 | 3개월 |
| 변경 시 | 워크플로우 Git 커밋 | 영구 |
| 분기별 | 외부 스토리지 아카이브 | 1년 |

---

## 핵심 요약

- 백업 대상: 워크플로우 정의 + 크리덴셜 + 환경 설정
- SQLite는 파일 복사, PostgreSQL은 pg_dump
- n8n API로 워크플로우만 자동 백업 가능
- Git으로 워크플로우 버전 관리 (변경 이력 추적)
- 정기적 복구 테스트 필수 (백업만 하고 확인 안 하면 의미 없음)

**다음 레슨**: n8n 버전 업그레이드를 안전하게 진행하는 방법을 알아봅니다.
