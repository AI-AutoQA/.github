# 로컬 실행 가이드

## 사전 요구사항

| 도구 | 권장 버전 |
|---|---|
| Docker Desktop | 4.x 이상 |
| Java (개별 백엔드 실행 시) | 17 |
| Node.js (개별 실행 시) | 20 LTS |
| Python (개별 실행 시) | 3.12 |

---

## 빠른 시작 (전체 docker compose)

```bash
cd infra
cp .env.example .env          # 환경변수 복사 (기본값으로 바로 사용 가능)
docker compose up --build
```

| 서비스 | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8080 |
| AI 서버 | http://localhost:8000 |
| MinIO Console | http://localhost:9001 |
| LocalStack | http://localhost:4566 |

---

## 서비스별 개별 실행

### 인프라만 먼저 기동

```bash
cd infra
docker compose up -d postgres redis localstack minio
```

### Backend

```bash
cd backend
./gradlew bootRun
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

### Playwright Worker

```bash
cd playwright
cp .env.example .env   # 없으면 infra/.env.example 참고
npm install
npx playwright install chromium
npm start
```

#### Playwright 디버깅 도구 활용

| 도구 | 우리 서비스 활용 | 주의점 |
|---|---|---|
| Playwright Inspector | QA 실행 실패 시 locator, actionability, timeout 원인 확인 | 운영 Worker에서는 사용하지 않고 로컬 재현에서만 사용 |
| Playwright Codegen | 대상 사이트의 안정적인 locator 후보(`getByRole`, `getByLabel`, `data-testid`)를 빠르게 확인 | 생성 코드를 그대로 커밋하지 말고 시나리오/selector 개선 참고 자료로 사용 |
| Chrome DevTools | 네트워크/렌더링/CSP/iframe/CAPTCHA 챌린지 원인 분석 | CAPTCHA나 봇 방지 우회 목적이 아니라 진단/수동 검증 판단에만 사용 |

예시:

```bash
cd playwright
npx playwright codegen https://www.google.com
```

분석 Worker의 관련 안전 옵션:

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `RESPECT_ROBOTS_TXT` | `true` | 크롤 시 `robots.txt` Disallow URL을 탐색하지 않음 |
| `CRAWL_REQUEST_DELAY_MS` | `750` | 같은 frontier 안에서 페이지 분석 시작 시점 분산 |
| `BLOCK_ANALYSIS_RESOURCES` | `false` | 비시각 대량 점검에서만 이미지/폰트/미디어 차단 |
| `STOP_ON_AUTOMATION_CHALLENGE` | `true` | CAPTCHA/Turnstile/브라우저 검증 감지 시 자동화를 중단하고 수동 확인 필요로 기록 |
| `QA_STOP_ON_AUTOMATION_CHALLENGE` | `true` | QA 시나리오 실행 중 챌린지 감지 시 해당 step을 실패 처리 |

작업 단위로는 `crawlOptions.respectRobotsTxt`, `crawlOptions.requestDelayMs`, `crawlOptions.blockResources`를 함께 내려 환경변수 기본값을 조정할 수 있습니다.

### AI 서버

```bash
cd ai
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

---

## 환경변수 설명

`infra/.env.example` 파일을 복사하여 `infra/.env`로 사용합니다.  
기본값은 로컬 docker compose 환경에 맞게 설정되어 있어 별도 수정 없이 실행 가능합니다.

| 변수 | 설명 | 기본값 |
|---|---|---|
| `POSTGRES_PASSWORD` | PostgreSQL 비밀번호 | `autoqa1234` |
| `REDIS_PASSWORD` | Redis 비밀번호 | `redis1234` |
| `AWS_REGION` | LocalStack 리전 | `ap-northeast-2` |
| `MINIO_ROOT_USER` | MinIO 루트 계정 | `minioadmin` |
| `MINIO_ROOT_PASSWORD` | MinIO 루트 비밀번호 | `minioadmin1234` |

---

## LocalStack SQS 확인

```bash
# 큐 목록 확인
aws --endpoint-url=http://localhost:4566 sqs list-queues --region ap-northeast-2

# 분석 요청 큐 테스트
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url http://localhost:4566/000000000000/qa-analysis-queue \
  --region ap-northeast-2

# 실행 요청 큐 테스트
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url http://localhost:4566/000000000000/qa-execute-queue \
  --region ap-northeast-2
```

---

## MinIO 버킷 확인

MinIO Console (http://localhost:9001) 에 접속하거나 mc CLI를 사용합니다.

```bash
mc alias set local http://localhost:9000 minioadmin minioadmin1234
mc ls local/autoqa-results
```

---

## 자주 겪는 문제

| 증상 | 해결 |
|---|---|
| LocalStack 헬스체크 실패 | `docker compose restart localstack` |
| PostgreSQL 연결 거부 | 포트 5432가 이미 사용 중인지 확인 |
| Playwright 브라우저 없음 | `cd playwright && npx playwright install chromium` |
| AI 서버 모듈 없음 | `pip install -r ai/requirements.txt` |
