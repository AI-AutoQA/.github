
# AutoQA

> AI 기반 웹 QA 자동화 플랫폼

AutoQA는 사용자가 테스트 대상 URL과 자연어 요구사항을 입력하면, 웹 페이지를 자동 분석하고 AI가 QA 시나리오를 생성한 뒤 Playwright 기반 E2E 테스트 실행 결과를 실시간으로 제공하는 서비스입니다. 분석, 시나리오 생성, 실행, 결과 리포트까지 이어지는 QA 파이프라인을 비동기 워커와 실시간 WebSocket 이벤트로 연결합니다.

---

## 프로젝트 개요

![AutoQA](././docs/image/AutoQA.png)

| 항목 | 내용 |
|---|---|
| 플랫폼 | 웹 기반 AI QA 자동화 SaaS |
| 개발 기간 | 2026.04.06 ~ 2026.06.02 |
| 개발 인원 | 6인 |

## Getting Started

### 사전준비

| 도구 | 권장 버전 | 용도 |
|---|---|---|
| Docker Desktop | 4.x 이상 | 로컬 인프라 및 전체 스택 실행 |
| Java | 17 | Backend 개별 실행 |
| Node.js | 20 LTS | Frontend, Playwright Worker 개별 실행 |
| Python | 3.12 | AI 서버 개별 실행 |
| AWS CLI | 2.x 이상 | 운영 AWS 리소스 접근 및 배포 |
| Terraform | 1.7 이상 | AWS 인프라 프로비저닝 |

자세한 로컬 실행 절차는 [docs/local-run.md](docs/local-run.md)를 참고하세요.

### 실행 방법

전체 스택을 Docker Compose로 실행합니다.

```bash
cd infra
cp .env.example .env
docker compose -f docker-compose.full.yml up -d --build
```

개발 중 애플리케이션을 개별로 실행하려면 인프라만 먼저 기동합니다.

```bash
cd infra
docker compose up -d postgres redis localstack minio
```

이후 필요한 서비스를 각각 실행합니다.

```bash
# Backend
cd backend
./gradlew bootRun

# Frontend
cd frontend
npm install
npm run dev

# Playwright Worker
cd playwright
cp .env.example .env
npm install
npx playwright install chromium
npm start

# AI Server
cd ai
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

### 접속 정보(로컬 기준)

| 구분 | 주소 | 비고 |
|---|---|---|
| Frontend | http://localhost:3000 | Docker Compose 기준 |
| Frontend Dev Server | http://localhost:5173 | `npm run dev` 기준 |
| Backend API | http://localhost:8080 | REST API |
| Swagger UI | http://localhost:8080/api/swagger-ui | 로컬 기본 활성화 |
| Backend Health | http://localhost:8080/actuator/health | Actuator health check |
| AI Server | http://localhost:8000 | FastAPI |
| LocalStack | http://localhost:4566 | 로컬 AWS 대체 |
| MinIO Console | http://localhost:9001 | `minioadmin` / `minioadmin1234` |
| PostgreSQL | localhost:5432 | DB `autoqa`, user `autoqa` |
| Redis | localhost:6379 | 기본 비밀번호 `redis1234` |

### 접속 정보(메인 EC2 기준)

| 구분 | 주소 | 비고 |
|---|---|---|
| 서비스 도메인 | https://www.autoqa.site | 메인 사용자 서비스 |
| SSAFY 도메인 | https://k14a101.p.ssafy.io | 동일 서비스 접근 도메인 |
| Backend API | https://www.autoqa.site/api | Nginx reverse proxy |
| WebSocket | wss://www.autoqa.site/ws | 실시간 QA 진행 이벤트 |
| Jenkins | https://k14a101.p.ssafy.io:8443 | 관리자 접근용 |
| Redis Commander | https://k14a101.p.ssafy.io:8081 | 관리자 접근용 |
| Container Logs | https://k14a101.p.ssafy.io:8082 | Dozzle 기반 로그 뷰어 |

운영 환경에서 Swagger UI는 보안 정책에 따라 비활성화될 수 있습니다. EC2 배포 절차는 [infra/SETUP-GUIDE.md](infra/SETUP-GUIDE.md)를 참고하세요.

## 환경 변수 및 인프라 가이드

### 환경 변수 관리

환경 변수는 예시 파일을 복사한 뒤 실제 값을 채워 사용합니다. 실제 `.env`, `terraform.tfvars`, AWS Access Key, DB 비밀번호, JWT Secret, 메일 앱 비밀번호, Redis 비밀번호는 저장소에 커밋하지 않습니다.

| 파일 | 용도 |
|---|---|
| `infra/.env.example` | 로컬 Docker Compose 공통 환경 변수 |
| `backend/.env.example` | Backend 운영 환경 변수 참고 템플릿 |
| `playwright/.env.example` | Playwright Worker 로컬/운영 환경 변수 |
| `ai/.env.example` | AI 서버 모델, Redis, SQS, S3 환경 변수 |
| `infra/ec2-setup/.env.example` | 메인 EC2 Docker Compose 운영 환경 변수 |
| `infra/terraform/main/terraform.tfvars.example` | Terraform 변수 템플릿 |

운영 배포 시에는 Terraform 출력값을 EC2 환경 변수로 반영합니다.

```bash
cd infra/terraform/main
terraform init
terraform plan
terraform apply
terraform output
```

주요 Terraform 출력값은 RDS endpoint, SQS queue URL, S3 artifacts bucket, ECR repository URL, ECS service name, CloudWatch dashboard URL, EC2/Jenkins용 IAM access key입니다.

### 인프라 구성 요약

- 메인 EC2는 Nginx, Frontend, Backend, Redis, Jenkins, Redis Commander, Dozzle을 Docker Compose로 실행합니다.
- Terraform은 VPC, public/private/db subnet, RDS PostgreSQL, SQS, S3, ECR, ECS Fargate Playwright Worker, ECS GPU AI Worker, IAM, CloudWatch, SNS를 관리합니다.
- Redis는 ElastiCache 대신 메인 EC2 Docker Redis를 사용합니다. Backend는 Compose network로 접근하고, ECS Worker는 EC2 공인 IP와 Redis 비밀번호를 통해 접근합니다.
- Playwright Worker와 AI Worker는 SQS 메시지 유무에 따라 0개에서 필요한 수만큼 확장되는 구조입니다.
- 테스트 산출물, 분석 결과, 시나리오 suite, 실행 결과는 S3 artifacts bucket에 저장합니다.

## Tech Stack

| 영역 | 기술 |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, Zustand, React Router, Axios, STOMP/SockJS, Three.js |
| Backend | Java 17, Spring Boot, Spring Security, JWT, Spring Data JPA, WebSocket, Redis, PostgreSQL, springdoc-openapi |
| Playwright Worker | Node.js, Express, Playwright, AWS SDK for JavaScript, ioredis |
| AI Server / Worker | Python 3.12, FastAPI, Uvicorn, Pydantic, vLLM, Transformers, LoRA 기반 LLM 추론 |
| Database / Cache | PostgreSQL, Redis |
| Storage / Queue | S3, SQS, SNS, MinIO, LocalStack |
| Infrastructure | Docker, Docker Compose, Nginx, Terraform, AWS VPC, RDS, ECR, ECS Fargate, ECS on EC2 GPU, IAM, CloudWatch |
| CI/CD / 운영 | Jenkins, CloudWatch Logs, CloudWatch Dashboard, Dozzle, Redis Commander |
| Load Test | k6 |

## 시스템 아키텍처

![시스템 아키텍처](./docs/image/architecture.svg)

## 기능구성

| 기능 | 설명 |
|---|---|
| 회원 인증 | 이메일 인증 기반 회원가입, 로그인, 로그아웃, JWT 재발급, 아이디/비밀번호 찾기를 제공합니다. |
| QA 파이프라인 시작 | 사용자가 대상 URL, 로그인 계정 정보, 자연어 요청을 입력하면 페이지 분석부터 시나리오 생성까지 자동 진행합니다. ![Main](./docs/image/Main.gif)|
| 웹 페이지 분석 | Playwright Worker가 대상 사이트를 탐색하며 페이지 구조, 주요 요소, 링크 관계, 스크린샷, DOM 정보를 수집합니다. ![Analyze](./docs/image/Analyze.gif)|
| 실시간 진행 표시 | Redis Pub/Sub과 Backend WebSocket을 통해 분석, AI 생성, 실행 단계의 상태와 로그를 Frontend에 실시간으로 전달합니다. ![Scenario](./docs/image/Scenario.png) |
| AI 시나리오 생성 | AI GPU Worker가 분석 산출물과 사용자 프롬프트를 기반으로 QA 시나리오와 step suite를 생성합니다.  ![AIScenario](./docs/image/AIScenario.gif)|
| 시나리오 검토/편집 | 생성된 시나리오를 실행 전 검토하고, 자연어로 시나리오를 추가하거나 기존 시나리오를 수정/삭제할 수 있습니다. ![ScenarioCheck](./docs/image/ScenarioCheck.png) |
| QA 자동 실행 | 검토가 끝난 시나리오 suite를 Playwright Worker가 브라우저에서 실행하고, step별 상태와 실행 로그를 기록합니다.  ![Execution](./docs/image/Execution.gif) |
| 결과 리포트 | 실행 결과, 성공/실패/경고 카운트, 소요 시간, 영상/스크린샷 등 산출물을 보고서 형태로 조회합니다. ![Report](./docs/image/Report.gif)|
| 히스토리 관리 | 사용자별 QA 리포트 목록과 상세 결과를 조회하고, 이전 분석 결과로 다시 진입할 수 있습니다. ![History](./docs/image/History.png)|
| 운영 모니터링 | CloudWatch Logs/Alarms, Jenkins, Redis Commander, Dozzle을 통해 배포와 운영 상태를 확인합니다. ![CloudWatchERD](./docs/image/CloudWatch.png)|

## 디렉토리 구조

### Backend

```text
backend/
├── src/
│   ├── main/
│   │   ├── java/com/autoqa/backend/
│   │   │   ├── BackendApplication.java
│   │   │   ├── common/          공통 응답, 설정, Swagger, 예외 처리
│   │   │   ├── domain/          인증, 사용자, 프로젝트, QA, 결과 도메인
│   │   │   └── infrastructure/  Redis, AWS, 외부 연동 인프라 코드
│   │   └── resources/           profile별 설정, 템플릿, 검증 리소스
│   └── test/                    테스트 코드
├── scripts/                     백엔드 보조 스크립트
├── gradle/                      Gradle Wrapper 리소스
├── build.gradle
├── settings.gradle
├── Dockerfile
├── Jenkinsfile
├── gradlew
└── gradlew.bat
```

- Spring Boot 기반 API 서버입니다.
- 인증/인가, 사용자 관리, QA 파이프라인 API, SQS 발행/결과 소비, Redis 상태 관리, WebSocket 이벤트 중계를 담당합니다.
- PostgreSQL에 사용자, 리포트, 페이지, 시나리오, 실행 결과를 저장합니다.

### Frontend

```text
frontend/
├── public/                      정적 리소스
├── src/
│   ├── api/                     REST API, WebSocket client
│   ├── components/              공통 UI 및 레이아웃 컴포넌트
│   ├── lib/                     공통 유틸리티
│   ├── pages/                   라우트 단위 화면
│   ├── store/                   Zustand 상태 관리
│   ├── styles/                  스타일 리소스
│   ├── types/                   공통 타입 정의
│   ├── App.tsx
│   ├── index.css
│   └── main.tsx
├── index.html
├── nginx.conf
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
├── Dockerfile
└── Jenkinsfile
```

- React, TypeScript, Vite 기반 SPA입니다.
- URL 입력, 분석 진행, 시나리오 검토, QA 실행 현황, 결과 리포트, 히스토리 화면을 제공합니다.
- WebSocket 이벤트를 구독해 QA 파이프라인 상태를 실시간으로 반영합니다.

### ai

```text
ai/
├── app/
│   ├── main.py                  FastAPI 엔트리포인트
│   ├── model_service.py         모델 로딩 및 추론 서비스
│   ├── schemas.py               요청/응답 스키마
│   ├── worker.py                SQS 기반 AI Worker
│   └── services/                AI 도메인 서비스
├── docs/                        AI 서버 문서
├── inference/                   별도 추론 서버 및 배포 보조 파일
│   ├── server.py
│   ├── worker.py
│   ├── proxy.py
│   └── startup-script.sh
├── models/                      로컬 모델/어댑터 저장 위치
├── scripts/                     실행 보조 스크립트
├── requirements.txt
└── Dockerfile
```

- FastAPI 기반 AI 서버와 SQS 기반 AI Worker를 포함합니다.
- 분석 산출물을 입력으로 받아 QA 시나리오를 생성하고, 결과를 S3와 SQS result queue로 전달합니다.
- 운영 환경에서는 ECS GPU Worker와 모델 캐시 경로를 사용합니다.

### playwright

```text
playwright/
├── config/                      Worker 설정
├── outputs/                     로컬 실행 산출물
├── public/                      정적 확인용 리소스
├── src/
│   ├── core/                    Playwright 분석/실행 핵심 로직
│   ├── http/                    HTTP 서버 및 health check
│   ├── infra/                   AWS, Redis 등 외부 인프라 연동
│   ├── scripts/                 정리/운영 보조 스크립트
│   ├── worker/                  SQS polling Worker
│   └── index.js                 Worker 엔트리포인트
├── package.json
├── Dockerfile
├── Jenkinsfile
├── web.md
└── WIKI.md
```

- Node.js와 Playwright 기반 워커입니다.
- SQS queue를 long polling하여 페이지 분석 작업과 QA 실행 작업을 처리합니다.
- 스크린샷, DOM JSON, 리포트, 영상 등 실행 산출물을 S3/MinIO에 저장하고 진행 이벤트를 Redis로 발행합니다.

### infra

```text
infra/
├── docker-compose.yml           로컬 인프라(PostgreSQL, Redis, LocalStack, MinIO)
├── docker-compose.full.yml      로컬 전체 스택 실행
├── ec2-setup/                   메인 EC2 운영 설정
│   ├── docker-compose.yml       운영 컨테이너 구성
│   ├── nginx.conf               HTTPS reverse proxy
│   ├── jenkins.Dockerfile       Jenkins 커스텀 이미지
│   ├── casc.yaml                Jenkins Configuration as Code
│   ├── setup.sh                 EC2 초기 설정
│   └── cloudwatch-agent.json    CloudWatch Agent 설정
├── localstack/                  로컬 AWS 리소스 초기화
├── scripts/                     인프라 운영 스크립트
├── terraform/
│   ├── README.md
│   └── main/
│       ├── vpc.tf
│       ├── security_groups.tf
│       ├── rds.tf
│       ├── sqs.tf
│       ├── s3.tf
│       ├── ecr.tf
│       ├── ecs.tf
│       ├── ai_ecs_gpu.tf
│       ├── iam.tf
│       ├── cloudwatch.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars.example
└── SETUP-GUIDE.md
```

- 로컬 개발용 Docker Compose와 운영 EC2 설정, Terraform 인프라 코드를 포함합니다.
- `infra/docker-compose.yml`은 PostgreSQL, Redis, LocalStack, MinIO만 실행합니다.
- `infra/docker-compose.full.yml`은 인프라와 애플리케이션 전체 스택을 함께 실행합니다.
- `infra/terraform/main`은 AWS 리소스를 생성하고 관리합니다.

### loadtest

```text
loadtest/
├── 01-smoke.js                  기본 API smoke test
├── 02-start-flow.js             QA 시작 플로우 부하 테스트
├── 03-scenarios-read.js         시나리오 조회 부하 테스트
├── 04-websocket.js              WebSocket 연결 테스트
├── 05-nplus1.js                 N+1 조회 성능 확인
├── lib/
│   └── auth.js                  k6 인증 헬퍼
├── results/                     테스트 결과 저장 위치
├── db-seed.sh                   테스트 데이터 시딩
├── db-seed-reportjson.sh        리포트 JSON 시딩
├── setup-test-data.sh           Redis/DB 테스트 데이터 준비
└── README.md
```

- k6 기반 부하 테스트 시나리오를 제공합니다.
- smoke test, QA 시작 플로우, 조회 부하, WebSocket, N+1 확인 시나리오를 포함합니다.
- 자세한 사용법은 [loadtest/README.md](loadtest/README.md)를 참고하세요.

## 프로젝트 산출물

| 산출물 | 위치 |
|---|---|
| ERD | ![ERD](./docs/image/ERD.png) |
| Swagger API Docs | 로컬 Swagger: http://localhost:8080/api/swagger-ui ![Swagger](./docs/image/Swagger.png) |
| 영상 포트폴리오 | [영상 포트폴리오](https://youtu.be/ZZ0NRTQ7WQA) |
| 기능명세서 | [기능명세서](https://www.notion.so/33a52d631e348106b4abd6b88058baba) |
| API 명세서 | [API명세서](https://www.notion.so/REST-API-33a52d631e3481c3aa77e0156bc76c72) |

## 참고 문서

| 문서 | 내용 |
|---|---|
| [docs/local-run.md](docs/local-run.md) | 로컬 개발 환경 구성 및 실행 방법 |
| [infra/SETUP-GUIDE.md](infra/SETUP-GUIDE.md) | 운영 인프라 구축 및 EC2 배포 가이드 |
| [docs/backend/api-overview.md](docs/backend/api-overview.md) | REST API 엔드포인트 개요 |
| [docs/frontend/realtime-events.md](docs/frontend/realtime-events.md) | Redis Pub/Sub 및 WebSocket 이벤트 명세 |
| [docs/backend/qa-pipeline-state-api.md](docs/backend/qa-pipeline-state-api.md) | QA 파이프라인 현재 상태 조회 API |
| [docs/playwright/qa-pipeline-sqs-contract.md](docs/playwright/qa-pipeline-sqs-contract.md) | SQS 기반 Playwright 분석/실행 계약 |
| [docs/playwright/qa-ai-scenario-generation-contract.md](docs/playwright/qa-ai-scenario-generation-contract.md) | AI 시나리오 생성 계약 |
| [docs/playwright/playwright-sqs-request-examples.md](docs/playwright/playwright-sqs-request-examples.md) | Playwright Worker SQS 요청/검증 예시 |
| [docs/infra/cloudwatch-server-debug-dashboard.md](docs/infra/cloudwatch-server-debug-dashboard.md) | CloudWatch 대시보드와 Logs Insights 쿼리 |
| [docs/playwright/qa-suite-report-schema.md](docs/playwright/qa-suite-report-schema.md) | QA Suite Report JSON 스키마 (대시보드 입력 포맷) |
| [docs/demo-scenario.md](docs/demo-scenario.md) | 데모 시연 시나리오 |
