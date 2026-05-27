# AutoQA 발표 자료

> AI 기반 웹 QA 자동화 SaaS 플랫폼

---

## 목차

1. [서비스 소개](#1-서비스-소개)
2. [해결하는 문제](#2-해결하는-문제)
3. [핵심 기능](#3-핵심-기능)
4. [시스템 아키텍처](#4-시스템-아키텍처)
5. [서비스 플로우](#5-서비스-플로우)
6. [기술 스택](#6-기술-스택)
7. [인프라 설계](#7-인프라-설계)
8. [보안 설계](#8-보안-설계)
9. [데모 시나리오](#9-데모-시나리오)
10. [성과 및 마무리](#10-성과-및-마무리)

---

## 1. 서비스 소개

### AutoQA란?

> **URL 하나만 입력하면, AI와 브라우저 자동화가 웹 분석 → QA 시나리오 생성 → 테스트 실행 → 리포트 생성까지 자동 수행하는 SaaS**

- **핵심 가치**: "수동 QA 작성 제로" — 사람은 URL만 던지고, 시스템이 나머지를 처리
- **프로젝트 기간**: 4주 스프린트

---

## 2. 해결하는 문제

### 기존 QA의 문제점

| 문제 | 설명 |
|---|---|
| 높은 진입 장벽 | Playwright, Selenium 등 테스트 코드를 직접 작성해야 함 |
| 반복 작업 | 서비스 변경 시마다 테스트 스크립트 수동 수정 필요 |
| 느린 피드백 | QA 엔지니어 없이는 자동화 테스트 불가 |
| 비용 문제 | 전담 QA 인력 채용·유지 비용 발생 |

### AutoQA의 해결책

```
개발자/기획자 → URL 입력 → AutoQA가 자동으로 → 테스트 리포트
```

개발 지식 없이도 웹 서비스 품질 검증이 가능합니다.

---

## 3. 핵심 기능

### 3-1. 웹사이트 자동 분석 (PAGE_ANALYSIS)

- 입력 URL에서 최대 N개 페이지를 크롤링
- 각 페이지의 DOM 요소, 인터랙션 가능 엘리먼트 자동 추출
- 스크린샷 + 분석 JSON 산출물 S3 저장

### 3-2. AI 시나리오 자동 생성 (SCENARIO_GENERATE)

- **Qwen3-8B (LoRA fine-tuned)** 모델 사용
- 분석 결과를 바탕으로 실행 가능한 QA 시나리오 JSON 자동 생성
- 테스트 유형 선택:
  - `Smoke` — 핵심 기능 동작 여부 확인
  - `Custom` — 자연어로 직접 시나리오 입력

### 3-3. 브라우저 자동 실행 (QA_EXECUTE)

- Playwright 기반 실제 브라우저에서 시나리오 실행
- 각 step별 결과 캡처 (스크린샷, 상태)
- 통과/실패 결과 리포트 생성

### 3-4. 실시간 진행 현황 (WebSocket)

- 테스트 시작부터 완료까지 실시간 로그 스트리밍
- `[START] → [AI] → [PLAYWRIGHT] → [DONE]` 단계별 진행 확인
- 별도 새로고침 없이 대시보드에서 즉시 확인

---

## 4. 시스템 아키텍처

### 4-tier 컴포넌트 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    외부 EC2 (xlarge 1대)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nginx (80/443)                                          │   │
│  │   ├─ /           → Frontend (React 정적 서빙)            │   │
│  │   ├─ /api, /ws   → Backend (Spring Boot :8080)           │   │
│  │   └─ /jenkins    → Jenkins (:8081)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│          Backend            Jenkins         Frontend (static)    │
└────────────────────────────┬────────────────────────────────────┘
                             │ AssumeRole (Cross-account)
                             ▼
┌─────────────── AWS 본 계정 (ap-northeast-2) ───────────────────┐
│  RDS PostgreSQL   Redis (ElastiCache)   SQS (4개 큐)           │
│  S3 (산출물)      Secrets Manager       ECR (이미지 저장소)     │
│  ECS Fargate (Playwright Worker 0~4)    CloudWatch              │
└────────────────────────────────────────────────────────────────┘
                             │
                             ▼ (AI 시나리오 생성)
┌──────────── GPU EC2 (g6e.xlarge, NVIDIA L4) ──────────────────┐
│  AI Server (FastAPI + Qwen3-8B, vLLM)                          │
│  모델: /mnt/nvme/models (NVMe SSD 로컬 캐시)                   │
└────────────────────────────────────────────────────────────────┘
```

### 컴포넌트별 역할

| 컴포넌트 | 스택 | 역할 |
|---|---|---|
| **Frontend** | React 18 + TypeScript + Vite + TailwindCSS + Zustand | 대시보드, 프로젝트/잡 관리, 실시간 로그 콘솔 |
| **Backend** | Spring Boot 3.4 (Java 17) + JPA | REST API, WebSocket 중계, SQS 오케스트레이션 |
| **Playwright Worker** | Node.js + Playwright | SQS long-polling, 웹 분석, 테스트 실행 |
| **AI Server** | FastAPI (Python 3.12) + Qwen3-8B + vLLM | QA 시나리오 JSON 자동 생성 |

---

## 5. 서비스 플로우

### 전체 파이프라인 흐름

```
[1] 사용자가 URL 입력
     └─→ Frontend → POST /api/v1/jobs → Backend

[2] Backend 처리
     ├─ PostgreSQL에 Job 레코드 생성
     ├─ (로그인 정보 있으면) Secrets Manager에 일시 저장
     └─ SQS qa-analysis-queue 발행

[3] 웹 분석 단계
     ├─ ECS Playwright Worker (0→1 스케일아웃)
     ├─ SQS 메시지 수신 → 대상 URL 크롤링
     ├─ DOM 요소 분석 → analysis-summary.json, final-report.json
     ├─ 스크린샷 캡처 → S3 업로드
     └─ Redis Pub/Sub으로 실시간 진행 이벤트 발행

[4] AI 시나리오 생성 단계
     ├─ SQS qa-ai-queue 발행
     ├─ AI Server (GPU EC2) 수신
     ├─ Qwen3-8B → QA 시나리오 JSON 생성
     ├─ Spring에서 시나리오 유효성 검증 (nodeId 등)
     └─ S3에 scenarios.json 업로드

[5] QA 실행 단계
     ├─ SQS qa-execute-queue 발행
     ├─ Playwright Worker 수신
     ├─ 실제 브라우저로 시나리오 step 순서대로 실행
     ├─ 각 step 스크린샷 + 결과 캡처
     └─ suite-report.json S3 업로드

[6] 결과 전달
     ├─ SQS qa-result-queue → Backend 수신
     ├─ PostgreSQL Job 상태 업데이트 (PASSED/FAILED)
     └─ WebSocket으로 Frontend에 최종 결과 전달
```

### 실시간 이벤트 전송 경로

```
Playwright Worker
  → Redis Pub/Sub (autoqa:progress:{jobId})
  → Backend WebSocket 중계
  → Frontend 실시간 로그 콘솔
```

### SQS 큐 구성

| 큐 | 역할 |
|---|---|
| `qa-analysis-queue` | URL 분석 작업 요청 |
| `qa-ai-queue` | AI 시나리오 생성 요청 |
| `qa-execute-queue` | QA 실행 요청 |
| `qa-result-queue` | 완료/실패 결과 수신 |
| `qa-dlq` | 실패 메시지 보관 (Dead Letter Queue) |

---

## 6. 기술 스택

### Frontend

| 기술 | 용도 |
|---|---|
| React 18 + TypeScript | UI 컴포넌트 |
| Vite | 빌드 도구 |
| TailwindCSS | 스타일링 |
| Zustand | 전역 상태 관리 |
| WebSocket | 실시간 로그 수신 |

### Backend

| 기술 | 용도 |
|---|---|
| Spring Boot 3.4 (Java 17) | REST API 서버 |
| JPA (Hibernate) | ORM |
| PostgreSQL | 메인 데이터베이스 |
| Redis | 실시간 이벤트 Pub/Sub, 캐시 |
| AWS SDK (SQS, S3, Secrets Manager) | 클라우드 서비스 연동 |

### Playwright Worker

| 기술 | 용도 |
|---|---|
| Node.js + Express | 워커 서버 |
| Playwright | 브라우저 자동화 |
| AWS SDK (SQS, S3) | 큐 소비 및 산출물 저장 |

### AI Server

| 기술 | 용도 |
|---|---|
| FastAPI (Python 3.12) | AI API 서버 |
| Qwen3-8B (LoRA fine-tuned) | QA 시나리오 생성 LLM |
| vLLM | GPU 추론 엔진 |
| NVIDIA L4 24GB | GPU 가속 |

### 인프라

| 기술 | 용도 |
|---|---|
| AWS ECS Fargate | Playwright Worker 컨테이너 실행 |
| AWS EC2 (g6e.xlarge) | AI GPU 서버 |
| AWS RDS PostgreSQL 15 | 영구 데이터 저장 |
| AWS S3 | 산출물 저장 (스크린샷, 리포트) |
| AWS SQS | 비동기 작업 큐 |
| AWS Secrets Manager | 시크릿 관리 |
| Terraform | 인프라 as Code |
| Jenkins | CI/CD 파이프라인 |
| Docker / Docker Compose | 컨테이너화 |

---

## 7. 인프라 설계

### 오토스케일링 전략

| 컴포넌트 | 평상시 | 최대 | 트리거 |
|---|---|---|---|
| 외부 EC2 (Frontend + Backend) | **1대** | 1대 | 상시 ON |
| Playwright Worker (ECS) | **0대** | 4대 | SQS 메시지 수 기반 |
| AI GPU Server (EC2) | **0대** | 1대 | 요청 발생 시 기동 |

> Playwright Worker와 AI 서버는 유휴 시 0대로 축소하여 비용 최소화

### 데이터 저장소 책임 분리

| 저장소 | 저장 대상 | 수명 |
|---|---|---|
| **PostgreSQL** | 프로젝트, 잡, 결과 메타데이터 | 영구 |
| **Redis** | 실시간 진행 상태, 단계 로그 | 휘발 (TTL 수 시간) |
| **S3** | 스크린샷, 분석 JSON, 리포트 | 30일 → IA → Glacier → 180일 삭제 |
| **SQS** | 작업 메시지 | 큐 처리 시점까지 |
| **Secrets Manager** | 시크릿, QA 로그인 정보 (일시) | 영구 / 작업 종료 시 삭제 |

### CI/CD 파이프라인 (Jenkins)

```
GitLab Push
  → Jenkins 파이프라인 트리거
  → Docker 이미지 빌드
  → ECR 푸시
  → EC2 배포 (docker compose pull & up)
  → Nginx reload
```

---

## 8. 보안 설계

### 민감 정보 보호 흐름

```
사용자 로그인 정보 입력
  → Backend 수신 즉시 Secrets Manager에 저장
  → SQS 메시지에는 secretRef(참조 ID)만 포함
  → Worker가 실행 직전 Secrets Manager에서 조회
  → 작업 완료 후 Secrets Manager에서 삭제
```

### 네트워크 보안

| 정책 | 내용 |
|---|---|
| 도메인 기반 접근만 허용 | Nginx `server_name` 미일치 시 444 응답 |
| HTTPS 강제 | Let's Encrypt, TLS 1.2+, HSTS, 80→443 리다이렉트 |
| IP 직접 접근 차단 | 공인 IP 직접 접근 불가 |
| Jenkins 보호 | IP 화이트리스트 + Basic Auth |
| IAM 최소 권한 | Cross-account AssumeRole, 용도별 Role 분리 |
| 전송/저장 암호화 | HTTPS, S3 SSE-S3, RDS/Redis at-rest 암호화 |

---

## 9. 데모 시나리오

### 시나리오 1 — Smoke 테스트 자동 실행

**"URL만 입력해도 테스트가 실행된다"**

1. 대시보드 → **+ 프로젝트 추가**
2. URL: `https://www.google.com` 입력 후 생성
3. **+ 테스트 실행** → 테스트 유형: `Smoke` 선택
4. 실시간 로그 콘솔에서 `[START] → [AI] → [PLAYWRIGHT] → [DONE]` 확인
5. 상태 **통과** 확인

### 시나리오 2 — Custom AI 시나리오

**"자연어로 원하는 테스트를 바로 생성한다"**

1. **+ 테스트 실행** → 테스트 유형: `Custom`
2. 시나리오 입력:
   ```
   메인 페이지에 접속하여 검색창이 존재하는지 확인하고 스크린샷을 찍는다
   ```
3. `[AI] Generating test script...` 메시지 확인
4. 결과 패널에서 통과/실패 수 및 소요 시간 확인

### 시나리오 3 — 실패 케이스 확인

**"실패 시 원인을 명확히 알려준다"**

1. URL: `http://localhost:9999` (존재하지 않는 주소)
2. Smoke 테스트 실행
3. `[ERROR]` 메시지 및 **오류** 상태 확인

---

## 10. 성과 및 마무리

### 구현 성과

| 항목 | 내용 |
|---|---|
| **완전한 자동화 파이프라인** | URL 입력 → 분석 → AI 생성 → 실행 → 리포트까지 End-to-End 완성 |
| **실시간 스트리밍** | WebSocket + Redis Pub/Sub 기반 실시간 진행 상황 전달 |
| **Fine-tuned AI 모델** | Qwen3-8B LoRA 학습으로 웹 QA 시나리오 특화 생성 |
| **운영형 QA 실행 엔진** | data-driven 실행, reusable flow, adaptive locator, flake analytics로 반복 운영성 강화 |
| **클라우드 네이티브** | ECS Fargate + Auto Scaling으로 0→N 탄력적 운영 |
| **보안 설계** | Secrets Manager 기반 민감 정보 비노출 파이프라인 |
| **MVP 검증 완료** | Naver 실서비스 대상 분석→실행 파이프라인 E2E 검증 |

### 핵심 강조 포인트

| 포인트 | 설명 |
|---|---|
| **코드 작성 불필요** | URL 입력만으로 Playwright 스크립트 자동 생성·실행 |
| **실시간 피드백** | 테스트 진행 상황을 즉시 대시보드에서 확인 |
| **AI 맞춤 시나리오** | 자연어 설명으로 원하는 테스트를 바로 생성 |
| **유지보수성** | 데이터만 바꿔 같은 시나리오를 반복 실행하고, 공통 flow와 locator/flake 분석으로 테스트 노후화를 줄임 |
| **비용 효율** | 유휴 시 Worker 0대 → 사용한 만큼만 과금 |
| **보안** | 사용자 로그인 정보를 코드·메시지에 노출하지 않음 |

---

## 발표 흐름 (추천)

```
[1분] 문제 정의
  → "QA 자동화, 지금은 이렇게 힘듭니다"

[2분] 솔루션 소개
  → "AutoQA: URL 하나로 해결합니다"
  → 핵심 기능 3가지 (분석 / AI 생성 / 실행)

[3분] 아키텍처 설명
  → 4-tier 구조 다이어그램
  → 전체 파이프라인 플로우

[3분] 라이브 데모
  → 시나리오 1: Smoke 테스트 (URL → 실시간 로그 → 결과)
  → (시간 여유 시) 시나리오 2: Custom 시나리오

[1분] 기술 스택 & 인프라 하이라이트
  → Qwen3-8B LoRA fine-tuning
  → ECS Auto Scaling 0→N
  → Secrets Manager 보안 설계

[1분] 마무리
  → MVP 검증 완료 (Naver 실서비스)
  → Q&A
```
