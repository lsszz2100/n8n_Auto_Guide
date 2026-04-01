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

## 실전 예제 모음

### 예제 1: 매일 자정 전체 워크플로우 JSON 백업 + Google Drive 저장

n8n API를 사용해 모든 워크플로우의 전체 정의를 JSON으로 내보내고, Google Drive의 날짜별 폴더에 저장합니다. 저장 결과를 Slack으로 보고합니다.

```
워크플로우 이름: "일일 워크플로우 자동 백업"
트리거: Schedule (매일 자정 00:00)

[Schedule Trigger]
  Rule: 0 0 * * *
    ↓
[HTTP Request: 전체 워크플로우 목록 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/workflows
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    limit: 250
    ↓
[Code: 백업 파일 구성]
  const workflows = $json.data || [];
  const today = new Date().toISOString().split("T")[0]; // YYYY-MM-DD

  // 각 워크플로우의 상세 정보를 포함한 백업 구조 생성
  const backup = {
    backupDate: today,
    backupTimestamp: new Date().toISOString(),
    n8nVersion: $json.n8nVersion || "unknown",
    workflowCount: workflows.length,
    activeCount: workflows.filter(w => w.active).length,
    workflows: workflows.map(w => ({
      id: w.id,
      name: w.name,
      active: w.active,
      createdAt: w.createdAt,
      updatedAt: w.updatedAt,
      tags: w.tags?.map(t => t.name) || [],
      nodes: w.nodes,
      connections: w.connections,
      settings: w.settings
    }))
  };

  return [{
    json: {
      backupDate: today,
      fileName: `n8n-workflows-backup-${today}.json`,
      fileContent: JSON.stringify(backup, null, 2),
      workflowCount: workflows.length,
      activeCount: backup.activeCount
    }
  }];
    ↓
[Google Drive: 날짜별 폴더 생성 또는 조회]
  Operation: Create Folder
  Folder Name: {{ $now.format('YYYY-MM') }}
  Parent Folder: "n8n-backups"
  // 이미 존재하면 기존 폴더 ID 반환
    ↓
[Google Drive: JSON 백업 파일 업로드]
  Operation: Upload
  File Name: {{ $json.fileName }}
  File Content: {{ $json.fileContent }}
  Mime Type: application/json
  Parent Folder: {{ $json.folderId }}
    ↓
[Code: 이전 백업 정리 (30일 초과 파일 목록)]
  // 30일 이상 된 백업 파일 찾기
  const cutoff = new Date();
  cutoff.setDate(cutoff.getDate() - 30);

  // 이전 단계에서 폴더 내 파일 목록 조회 후 필터링
  const oldFiles = $input.all()
    .filter(item => new Date(item.json.createdTime) < cutoff)
    .map(item => item.json.id);

  return [{ json: { oldFileIds: oldFiles, count: oldFiles.length } }];
    ↓
[IF: 삭제할 오래된 파일 있음?]
  True →
    [Google Drive: 오래된 파일 삭제]
      (각 파일 ID에 대해 DELETE 처리)
  False → [계속]
    ↓
[Slack: 백업 완료 보고]
  Channel: #ops-backup
  Message: |
    n8n 워크플로우 백업 완료

    날짜: {{ $json.backupDate }}
    백업 워크플로우 수: {{ $json.workflowCount }}개
    활성 워크플로우: {{ $json.activeCount }}개
    저장 위치: Google Drive / n8n-backups / {{ $now.format('YYYY-MM') }}
    파일명: {{ $json.fileName }}
```

---

### 예제 2: 크리덴셜 목록 주기적 감사 + 미사용 크리덴셜 알림

n8n API로 전체 크리덴셜 목록과 워크플로우 목록을 조회하여, 어떤 워크플로우에서도 사용하지 않는 크리덴셜을 찾아 관리자에게 알립니다.

```
워크플로우 이름: "크리덴셜 감사 워크플로우"
트리거: Schedule (매주 일요일 오전 9시)

[Schedule Trigger]
  Rule: 0 9 * * 0
    ↓
[HTTP Request: 전체 크리덴셜 목록 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/credentials
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
    ↓
[Code: 크리덴셜 정보 저장]
  const credentials = $json.data || [];

  return [{
    json: {
      allCredentials: credentials.map(c => ({
        id: c.id,
        name: c.name,
        type: c.type,
        createdAt: c.createdAt,
        updatedAt: c.updatedAt
      })),
      credentialCount: credentials.length
    }
  }];
    ↓
[HTTP Request: 전체 워크플로우 목록 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/workflows
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    limit: 250
    ↓
[Code: 워크플로우에서 사용 중인 크리덴셜 ID 추출]
  const workflows = $json.data || [];
  const credentials = $('Code: 크리덴셜 정보 저장').item.json.allCredentials;

  // 모든 워크플로우의 모든 노드에서 크리덴셜 ID 수집
  const usedCredentialIds = new Set();
  for (const wf of workflows) {
    for (const node of (wf.nodes || [])) {
      if (node.credentials) {
        for (const credType of Object.values(node.credentials)) {
          if (credType.id) usedCredentialIds.add(credType.id);
        }
      }
    }
  }

  // 사용되지 않는 크리덴셜 찾기
  const unusedCredentials = credentials.filter(c => !usedCredentialIds.has(c.id));

  // 오래된 순으로 정렬
  unusedCredentials.sort((a, b) => new Date(a.updatedAt) - new Date(b.updatedAt));

  const now = Date.now();
  const unusedWithAge = unusedCredentials.map(c => ({
    ...c,
    daysSinceUpdate: Math.floor((now - new Date(c.updatedAt).getTime()) / 86400000)
  }));

  return [{
    json: {
      unusedCount: unusedCredentials.length,
      totalCount: credentials.length,
      usedCount: credentials.length - unusedCredentials.length,
      unusedCredentials: unusedWithAge,
      reportDate: new Date().toISOString().split("T")[0]
    }
  }];
    ↓
[IF: 미사용 크리덴셜 있음?]
  False →
    [Slack: 모든 크리덴셜 사용 중 (이상 없음) 보고]
  True →
    [Code: 알림 메시지 구성]
      const unused = $json.unusedCredentials;
      const lines = unused.map(c =>
        `- ${c.name} (${c.type}) | ${c.daysSinceUpdate}일 미사용 | 마지막 수정: ${c.updatedAt.split("T")[0]}`
      );
      return [{
        json: {
          message: lines.join("\n"),
          count: unused.length,
          reportDate: $json.reportDate
        }
      }];
      ↓
    [Slack: 미사용 크리덴셜 감사 알림]
      Channel: #security-audit
      Message: |
        크리덴셜 감사 보고 ({{ $json.reportDate }})

        전체 크리덴셜: {{ $('Code: 워크플로우에서 사용 중인 크리덴셜 ID 추출').item.json.totalCount }}개
        사용 중: {{ $('Code: 워크플로우에서 사용 중인 크리덴셜 ID 추출').item.json.usedCount }}개
        미사용 감지: {{ $json.count }}개

        미사용 크리덴셜 목록:
        {{ $json.message }}

        보안을 위해 불필요한 크리덴셜은 삭제를 검토해주세요.
        {{ $vars.N8N_BASE_URL }}/credentials
```

---

### 예제 3: 워크플로우 버전 관리 (변경 시 자동 GitHub 커밋)

워크플로우가 저장될 때마다 자동으로 GitHub 리포지토리에 커밋하여 버전 이력을 관리합니다. n8n의 Webhook 트리거와 GitHub API를 사용합니다.

```
워크플로우 이름: "워크플로우 변경 감지 및 GitHub 자동 커밋"
트리거: Schedule (매 30분 - 변경된 워크플로우 감지)

[Schedule Trigger]
  Rule: */30 * * * *
    ↓
[HTTP Request: 전체 워크플로우 목록 조회]
  Method: GET
  URL: {{ $vars.N8N_BASE_URL }}/api/v1/workflows
  Headers:
    X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
  Query Parameters:
    limit: 250
    ↓
[Code: 최근 30분 내 변경된 워크플로우 필터링]
  const workflows = $json.data || [];
  const THIRTY_MIN_AGO = new Date(Date.now() - 30 * 60 * 1000);

  const recentlyChanged = workflows.filter(wf => {
    return new Date(wf.updatedAt) >= THIRTY_MIN_AGO;
  });

  if (recentlyChanged.length === 0) {
    return [{ json: { noChanges: true } }];
  }

  return recentlyChanged.map(wf => ({
    json: {
      id: wf.id,
      name: wf.name,
      updatedAt: wf.updatedAt,
      active: wf.active,
      // 파일명으로 사용할 안전한 이름 (특수문자 제거)
      safeName: wf.name.replace(/[^가-힣a-zA-Z0-9-_]/g, "_").toLowerCase()
    }
  }));
    ↓
[IF: noChanges === true]
  True → [종료: 변경 없음]
  False →
    [Split In Batches: 1개씩]
      ↓
    [HTTP Request: 각 워크플로우 전체 정의 조회]
      Method: GET
      URL: {{ $vars.N8N_BASE_URL }}/api/v1/workflows/{{ $json.id }}
      Headers:
        X-N8N-API-KEY: {{ $vars.N8N_API_KEY }}
      ↓
    [Code: GitHub 커밋용 파일 구성]
      const wf = $json;
      const safeName = $('Code: 최근 30분 내 변경된 워크플로우 필터링').item.json.safeName;

      const content = JSON.stringify({
        id: wf.id,
        name: wf.name,
        active: wf.active,
        updatedAt: wf.updatedAt,
        nodes: wf.nodes,
        connections: wf.connections,
        settings: wf.settings
      }, null, 2);

      // GitHub API는 Base64 인코딩 필요
      const encoded = Buffer.from(content).toString("base64");
      const filePath = `workflows/${safeName}.json`;

      return [{
        json: {
          filePath,
          encodedContent: encoded,
          workflowName: wf.name,
          commitMessage: `자동 백업: ${wf.name} 업데이트 (${new Date(wf.updatedAt).toLocaleString("ko-KR")})`
        }
      }];
      ↓
    [HTTP Request: GitHub API - 현재 파일 SHA 조회]
      // 파일이 이미 있으면 SHA 필요 (업데이트 시)
      Method: GET
      URL: https://api.github.com/repos/{{ $vars.GITHUB_OWNER }}/{{ $vars.GITHUB_REPO }}/contents/{{ $json.filePath }}
      Headers:
        Authorization: Bearer {{ $vars.GITHUB_TOKEN }}
        Accept: application/vnd.github+json
      Options:
        Ignore Response Code: true
      ↓
    [Code: SHA 추출 (신규 파일이면 undefined)]
      const sha = $json.sha || undefined;
      return [{
        json: {
          ...$('Code: GitHub 커밋용 파일 구성').item.json,
          sha
        }
      }];
      ↓
    [HTTP Request: GitHub API - 파일 커밋]
      Method: PUT
      URL: https://api.github.com/repos/{{ $vars.GITHUB_OWNER }}/{{ $vars.GITHUB_REPO }}/contents/{{ $json.filePath }}
      Headers:
        Authorization: Bearer {{ $vars.GITHUB_TOKEN }}
        Accept: application/vnd.github+json
      Body:
        {
          "message": "{{ $json.commitMessage }}",
          "content": "{{ $json.encodedContent }}",
          "branch": "main",
          "sha": "{{ $json.sha }}"
        }
      ↓
    [다음 워크플로우 처리...]
    ↓
[Merge: 모든 커밋 완료 후]
    ↓
[Slack: 버전 관리 완료 보고]
  Channel: #ops-backup
  Message: |
    워크플로우 GitHub 자동 커밋 완료

    변경된 워크플로우: {{ $input.all().length }}개
    저장소: {{ $vars.GITHUB_OWNER }}/{{ $vars.GITHUB_REPO }}
    브랜치: main
    커밋 시각: {{ $now.format('YYYY-MM-DD HH:mm') }}
```

---

## 핵심 요약

- 백업 대상: 워크플로우 정의 + 크리덴셜 + 환경 설정
- SQLite는 파일 복사, PostgreSQL은 pg_dump
- n8n API로 워크플로우만 자동 백업 가능
- Git으로 워크플로우 버전 관리 (변경 이력 추적)
- 정기적 복구 테스트 필수 (백업만 하고 확인 안 하면 의미 없음)

**다음 레슨**: n8n 버전 업그레이드를 안전하게 진행하는 방법을 알아봅니다.
