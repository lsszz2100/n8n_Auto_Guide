# 레슨 2.5: npm으로 로컬 설치

---

## npm 설치 방식의 특징

npm(Node Package Manager)으로 n8n을 설치하는 방식은:
- 장점: 가장 간단한 로컬 설치, 빠른 시작
- 단점: Node.js 버전 관리가 필요, 운영 환경에는 부적합
- 추천 사용: 로컬 개발, 학습, 테스트

---

## 사전 요구사항

### Node.js 설치

n8n은 Node.js 18.x 이상이 필요합니다.

nvm(Node Version Manager) 사용 권장:

```bash
# macOS / Linux: nvm 설치
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# nvm 활성화 (터미널 재시작 후)
source ~/.bashrc  # 또는 ~/.zshrc

# Node.js 20 설치 (LTS)
nvm install 20
nvm use 20

# 버전 확인
node --version  # v20.x.x
npm --version   # 10.x.x
```

Windows:
1. `nodejs.org` 접속
2. LTS 버전 다운로드 및 설치
3. 명령 프롬프트(CMD)에서 `node --version` 확인

---

## n8n 설치

```bash
# 전역(global) 설치
npm install n8n -g

# 설치 확인
n8n --version
```

> Note: 설치에 2~5분 정도 소요됩니다.

---

## n8n 실행

```bash
# 기본 실행
n8n

# 또는 웹 인터페이스로 바로 시작
n8n start
```

실행 성공 시 터미널에 다음과 같은 메시지가 표시됩니다:

```
Editor is now accessible via:
http://localhost:5678/

Press "o" to open in Browser, "q" to quit
```

브라우저에서 `http://localhost:5678` 접속하면 n8n이 열립니다!

---

## 환경 변수 설정

n8n의 동작을 커스터마이즈하려면 환경 변수를 설정합니다.

### .env 파일 생성 (권장)

홈 디렉토리에 `.n8n` 폴더를 만들고 설정을 저장:

```bash
# macOS/Linux
mkdir -p ~/.n8n
cat > ~/.n8n/.env << EOF
N8N_PORT=5678
GENERIC_TIMEZONE=Asia/Seoul
N8N_EDITOR_BASE_URL=http://localhost:5678
N8N_PROTOCOL=http
EOF
```

### 주요 환경 변수

| 변수명 | 설명 | 예시 값 |
|--------|------|---------|
| `N8N_PORT` | 접속 포트 | `5678` |
| `GENERIC_TIMEZONE` | 기본 타임존 | `Asia/Seoul` |
| `N8N_BASIC_AUTH_ACTIVE` | 기본 인증 활성화 | `true` |
| `N8N_BASIC_AUTH_USER` | 로그인 사용자명 | `admin` |
| `N8N_BASIC_AUTH_PASSWORD` | 로그인 비밀번호 | `password` |
| `DB_TYPE` | 데이터베이스 타입 | `sqlite` |
| `N8N_LOG_LEVEL` | 로그 레벨 | `info` |

---

## 자동 시작 설정 (PM2 사용)

로컬에서 n8n을 항상 실행 상태로 유지하려면 PM2를 사용합니다:

```bash
# PM2 설치
npm install pm2 -g

# n8n을 PM2로 실행
pm2 start n8n --name "n8n"

# 시스템 재시작 시 자동 실행 설정
pm2 startup
pm2 save

# 상태 확인
pm2 status

# 로그 확인
pm2 logs n8n
```

---

## n8n 업데이트

```bash
# 현재 버전 확인
n8n --version

# 최신 버전으로 업데이트
npm update -g n8n

# 또는 특정 버전 설치
npm install -g n8n@1.x.x
```

> 주의: 업데이트 전에 워크플로우를 반드시 백업하세요!

---

## 데이터 저장 위치

npm으로 설치한 n8n의 데이터는 다음 위치에 저장됩니다:

- macOS/Linux: `~/.n8n/`
- Windows: `C:\Users\사용자명\.n8n\`

이 폴더에는:
- `database.sqlite`: 워크플로우, 크리덴셜, 실행 기록
- `config`: 설정 파일

---

## 데이터 백업

```bash
# macOS/Linux
cp -r ~/.n8n ~/.n8n_backup_$(date +%Y%m%d)

# Windows (PowerShell)
Copy-Item -Recurse "$env:USERPROFILE\.n8n" "$env:USERPROFILE\.n8n_backup"
```

---

## 문제 해결

### "command not found: n8n"
```bash
# npm 전역 바이너리 경로 확인
npm bin -g

# 경로를 PATH에 추가
export PATH="$(npm bin -g):$PATH"
```

### Node.js 버전 오류
```bash
# 현재 버전 확인
node --version

# nvm으로 버전 변경
nvm use 20
```

### 포트 충돌
```bash
# 다른 포트로 실행
N8N_PORT=5679 n8n start
```

---

## Docker vs npm: 무엇을 선택?

| 상황 | 추천 방법 |
|------|-----------|
| 운영 서버 (VPS) | Docker Compose |
| 로컬 개발/학습 | npm |
| 팀 환경 | Docker Compose |
| 단순 테스트 | npm |

---

## 핵심 요약

- npm 설치는 Node.js 18+ 필요, `npm install n8n -g`로 간단 설치
- 실행은 `n8n start`, 브라우저에서 `localhost:5678` 접속
- 운영 환경에서는 PM2로 자동 시작 설정
- 데이터는 `~/.n8n/` 폴더에 저장됨
- 장기 운영에는 Docker가 더 적합

**다음 레슨**: n8n 인터페이스의 모든 기능을 완전히 이해해 봅니다.
