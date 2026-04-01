# 레슨 2.4: Docker로 셀프 호스팅 설치

---

## Docker란?

Docker는 애플리케이션을 컨테이너라는 격리된 환경에서 실행하는 기술입니다.

n8n을 Docker로 설치하면:
- 환경 충돌 없이 깔끔하게 설치
- 한 줄 명령으로 시작/중지
- 다른 서버로 쉽게 이전
- 업데이트가 간편

---

## 사전 준비

### Docker 설치

Windows:
1. `docs.docker.com/desktop/install/windows-install/` 접속
2. Docker Desktop for Windows 다운로드 및 설치
3. WSL2 백엔드 활성화 (설치 과정에서 안내됨)

macOS:
1. `docs.docker.com/desktop/install/mac-install/` 접속
2. Apple Silicon(M1/M2/M3) 또는 Intel 버전 선택 후 설치

Ubuntu/Debian Linux:
```bash
# Docker 공식 설치 스크립트
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER
```

설치 확인:
```bash
docker --version
# Docker version 26.x.x 이상이 출력되면 성공
```

---

## 방법 1: 단순 Docker 실행 (로컬 테스트용)

가장 빠르게 n8n을 실행하는 방법입니다:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

명령어 설명:
- `-p 5678:5678`: 포트 5678 개방 (브라우저로 접근)
- `-v ~/.n8n:/home/node/.n8n`: 데이터를 로컬에 저장
- `--rm`: 컨테이너 종료 시 자동 삭제 (데이터는 볼륨에 보존)

실행 후 브라우저에서 `http://localhost:5678` 접속

---

## 방법 2: Docker Compose (운영 환경 권장)

Docker Compose를 사용하면 더 체계적으로 관리할 수 있습니다.

### docker-compose.yml 파일 생성

```yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=Asia/Seoul
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=your_password_here
    volumes:
      - n8n_data:/home/node/.n8n
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  n8n_data:
```

### 실행

```bash
# 백그라운드로 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f n8n

# 중지
docker-compose down

# 재시작
docker-compose restart n8n
```

---

## 방법 3: PostgreSQL과 함께 사용 (고급, 안정적인 운영)

기본 SQLite 대신 PostgreSQL을 사용하면 더 안정적이고 성능이 좋습니다.

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8n_password
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8n_password
      - N8N_HOST=your-domain.com
      - WEBHOOK_URL=https://your-domain.com/
      - GENERIC_TIMEZONE=Asia/Seoul
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

volumes:
  postgres_data:
  n8n_data:
```

---

## HTTPS 설정 (도메인이 있는 경우)

외부에서 접근하거나 Webhook을 사용하려면 HTTPS가 필요합니다.

### Nginx Proxy Manager 활용

```yaml
# nginx-proxy-manager 서비스를 추가
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

1. `http://서버IP:81` 접속
2. n8n 서비스에 대한 Proxy Host 추가
3. Let's Encrypt SSL 인증서 자동 발급

---

## n8n 업데이트

```bash
# 최신 이미지 가져오기
docker-compose pull n8n

# 재시작 (데이터 보존)
docker-compose up -d --force-recreate n8n
```

---

## 데이터 백업

```bash
# n8n 데이터 볼륨 백업
docker run --rm \
  -v n8n_data:/source \
  -v $(pwd):/backup \
  alpine tar czf /backup/n8n_backup_$(date +%Y%m%d).tar.gz -C /source .
```

---

## 문제 해결

### 포트 5678이 이미 사용 중일 때
```bash
# 사용 중인 프로세스 확인
lsof -i :5678

# 다른 포트로 변경
ports:
  - "5679:5678"
```

### 컨테이너가 계속 재시작될 때
```bash
# 로그 확인
docker logs n8n --tail 50
```

### 권한 오류가 날 때
```bash
# 볼륨 권한 수정
sudo chown -R 1000:1000 ~/.n8n
```

---

## 핵심 요약

- Docker는 n8n을 깔끔하게 설치하고 관리하는 가장 좋은 방법
- 로컬 테스트: `docker run` 한 줄로 시작
- 운영 환경: `docker-compose`로 체계적 관리
- 안정적인 운영을 위해 PostgreSQL 연동 권장
- 외부 접근 시 HTTPS 설정 필수

**다음 레슨**: npm으로 로컬에 설치하는 방법을 알아봅니다.
