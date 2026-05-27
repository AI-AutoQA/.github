**서비스 한 줄 소개**
URL 하나만 입력하면 웹 분석 → AI 시나리오 생성 → 브라우저 자동 실행 → 리포트까지 수행하는 AI 기반 웹 QA 자동화 SaaS

**서비스 설명/주요기능**
[ AutoQA — AI 기반 웹 QA 자동화 SaaS ]

1. Home URL 기반 자동 웹 탐색 QA
   - BFS 기반 사이트 전체 크롤링 (최대 200 페이지)
   - DOM 파싱으로 클릭 가능성이 있는 후보 요소(button, a, input 외 div·span·카드형·커스텀 드롭다운 포함)를 수집하고 정적 분석으로 클릭 우선순위 랭킹
   - 이중 인증 모드(익명/인증) 크롤로 로그인 이후 전용 영역까지 분석

2. 페이지별 기능 자동 분석 및 시나리오 생성
   - Qwen3-8B (LoRA fine-tuned) 모델로 분석 결과 기반 QA 시나리오 JSON 자동 생성
   - GPT-5.2 / GPT-5 mini를 활용한 라벨링·시나리오 우선순위 결정 보조
   - Smoke (핵심 기능 확인) / Custom (자연어 직접 입력) 테스트 유형 선택 지원
   - 시나리오 수동 편집 및 step 단위 재시도 지원

3. 실제 사용자 행동 기반 브라우저 자동 실행
   - Playwright 기반 실제 Chromium 브라우저에서 시나리오를 단계별로 실행
   - 각 step 스크린샷 캡처, self-healing locator로 요소 탐색 실패 시 대안 자동 선택
   - 데이터드리븐 실행(여러 입력 데이터 세트 반복) 및 flow 재사용 지원

4. UI 반응 + 요청/응답 통합 검증
   - DOM 변화뿐 아니라 네트워크 요청 발생 여부·응답 상태 코드·응답 시간까지 통합 검증
   - toHaveNoConsoleErrors / toHaveNoNetworkErrors assertion matcher로 JS 오류·네트워크 실패 감지

5. 서비스 건강도 리포트 생성
   - 통과율·flaky 비율·step별 소요 시간 등 품질 지표(buildMetrics) 리포트 자동 생성
   - 실패 시나리오를 Jira 이슈로 1-click 발급 (실패 step 스크린샷 자동 첨부)
   - QA 대시보드 HTML을 통한 시각화 및 S3 아티팩트 관리

6. 병렬 페이지 QA 및 작업 분산 처리
   - SQS 4개 큐로 분석·AI·실행·결과를 완전 분리, 각 단계 독립 스케일아웃
   - ECS Fargate 기반 Playwright Worker 0~4대 자동 확장

7. 실시간 QA 진행 상황 확인 및 결과 아티팩트 관리
   - WebSocket으로 [분석] → [AI] → [실행] → [완료] 전 단계를 실시간 로그 스트리밍
   - 크롤 중 페이지별 스크린샷 S3 즉시 업로드 + presigned URL 실시간 이벤트 발행
   - 로그인 정보는 Secrets Manager 임시 저장, 작업 완료 후 즉시 삭제

**프로젝트의 특장점 (기능 관점)**

1. BFS 기반 page graph 수집과 병렬 QA
   - Home URL에서 시작해 페이지를 graph 구조로 수집, 페이지별 분석·QA를 병렬 실행해 탐색형 QA의 속도와 확장성 확보

2. 비정형 UI 요소 탐색
   - button·a·input 외에도 DOM 파싱으로 클릭 가능성이 있는 div·span·카드형·커스텀 드롭다운 후보를 수집, 정적 분석으로 우선순위 랭킹

3. 이중 인증 크롤
   - 익명 크롤 → 로그인 자동화 → 인증 크롤로 로그인 후 전용 경로까지 빠짐없이 분석

4. UI + 네트워크 통합 검증
   - DOM 변화만 보는 것이 아니라 요청 발생 여부·응답 상태·응답 시간·UI/API 불일치까지 함께 분석

5. 실시간 가시성
   - WebSocket 기반 단계별 진행 상황 + 크롤 중 페이지별 스크린샷 presigned URL 즉시 제공

6. Jira 이슈 1-click 발급
   - 실패 시나리오를 Jira 이슈로 즉시 생성, 실패 step 스크린샷 자동 첨부로 버그 리포팅 자동화

7. 품질 지표 리포트
   - 통과율·flaky 비율·step별 소요 시간 buildMetrics 기반 QA 대시보드 자동 생성

8. self-healing locator
   - locator 실패 시 CSS selector → aria → text 순으로 대안 자동 탐색해 실행 안정성 확보

**프로젝트의 차별점/독창성 (기술 관점)**

1. 규칙 기반 + Qwen3 하이브리드 구조
   - 페이지 분류·기능 후보 추론·시나리오 우선순위·애매한 결과 해석에 AI 활용, 실제 브라우저 실행과 핵심 판정은 규칙 기반 유지 — 비용·신뢰성 동시 확보

2. SQS + Redis 이중 메시징 구조
   - 비동기 작업 분배는 SQS(분석·AI·실행·결과 큐), 실시간 이벤트 스트리밍은 Redis Pub/Sub → WebSocket으로 분리, 작업 처리와 이벤트 소비를 완전히 독립

3. AI 입력 최소화 전처리
   - 전체 HTML 대신 접근성 트리 요약 + functionalPaths + 스크린샷만 전달, 신뢰도 0~1 점수로 기준 미달 요소는 AI 입력에서 제외

4. QA 파이프라인 상태 머신
   - PAGE_ANALYZING → SCENARIO_GENERATING → READY_TO_EXECUTE → EXECUTING → COMPLETED/FAILED 단계별 상태 전이를 Backend 오케스트레이터가 관리, 각 전이는 Redis 기반 원자적 처리

5. 시나리오 자동 검증 및 fixer
   - AI가 생성한 시나리오를 Backend에서 nodeId 유효성·step 계약 위반·컨테이너 클릭 오류 등을 검증, GmsScenarioFixerService로 자동 보정 후 저장

6. Jira 연동 자동화
   - 실패 시나리오를 Jira 이슈로 자동 발급, 실패 step S3 스크린샷을 attachment로 업로드 (idempotent 처리, 사용자별 Jira 계정 분리 관리)

7. Zero-idle 인프라
   - Playwright Worker(ECS Fargate)·AI 서버(GPU EC2) 유휴 시 0대 유지, 요청 시에만 기동 — ECS pinned worker 예열로 콜드스타트 최소화

8. Cross-account IAM 설계
   - Jenkins EC2와 AWS 본 계정을 Cross-account AssumeRole로 분리, 서비스별 최소 권한 Role 운용

**AI를 사용한 팀들은 필수 작성**

1. Qwen3-8B (Stage C DPO LoRA fine-tuned) — QA 시나리오 생성 주 모델
   - SFT + DPO 2단계 파인튜닝 (r=32, alpha=64, 7개 projection target, Stage A SFT + Stage C DPO 단일 LoRA 흡수)
   - 추론 엔진: vLLM (NVIDIA g7e.2xlarge GPU, GPU Memory Util 0.85)
   - enable_thinking=False, <think> 블록 strip, ChatML 포맷 사용
   - guided_json (JSON Schema 강제 디코딩) 적용으로 출력 구조 보장
   - 입력: site-summary.json + 스크린샷 (최대 20,000자 입력, 최대 6,144 tokens 출력)

2. GPT-5.2 / GPT-5 mini — 라벨링·시나리오 우선순위 결정 보조
   - AI 데이터 전처리 단계에서 분류 라벨 생성
   - 시나리오 우선순위 결정 및 애매한 결과 해석 보조

3. AI 파이프라인 구성
   - ① 분석 데이터 수집: Playwright BFS 크롤, 어노테이션 스크린샷 생성, 이중 인증 모드(익명/인증)
   - ② 데이터 전처리: URL 정규화, 신뢰도 점수화(0~1), functionalPaths 구조화, site-summary.json 생성
   - ③ AI 추론: vLLM + Qwen3-8B, guided_json 출력, SQS 멱등성 체크(Redis key EXISTS)로 중복 방지
   - ④ 시나리오 생성: QaSuiteValidationService 검증 → GmsScenarioFixerService 자동 보정 → S3 저장

**역할별 담당자**
AI 서버 (Qwen3 파인튜닝·vLLM 추론): 김용준
Playwright (웹 분석·QA 실행) / Infra: 김승현
Frontend (React·WebSocket·Three.js 그래프): 전경원
Backend (API·파이프라인 오케스트레이터): 조민제, 조재훈, 한현진

**프론트/모바일 프레임워크**
React 18, TypeScript, Vite, TailwindCSS, Zustand, Three.js, WebSocket (native), Axios

**백엔드 프레임워크**
Spring Boot 3.4 (Java 21), Spring Security + JWT, JPA (Hibernate), AWS SDK v2, Redis (Spring Data Redis)
FastAPI (Python 3.12), vLLM, Qwen3-8B (LoRA), Hugging Face Transformers / PEFT
Node.js 20, Express, Playwright (Chromium)

**DB**
PostgreSQL 15 (AWS RDS), Redis (AWS ElastiCache)

**주요 기술 스택**
FE: React 18, TypeScript, Vite, TailwindCSS, Zustand, Three.js, WebSocket, Axios
BE: Java 21, Spring Boot 3.4, Spring Security, JWT, JPA, Node.js 20, Express, Playwright
AI Server: Python 3.12, FastAPI, Qwen3-8B (DPO LoRA), vLLM, GPT-5.2 / GPT-5 mini
DB: PostgreSQL 15, Redis
Infra: AWS ECS Fargate, EC2 (g7e.2xlarge GPU), RDS, S3, SQS, ECR, Secrets Manager, CloudWatch, VPC, Terraform, Jenkins, Docker, Nginx, Let's Encrypt

**요구사항 명세서**
[직접 입력]

**기획 (와이어프레임)**
[직접 입력]

**설계 (ERD)**
[직접 입력]

**설계 (API)**
[직접 입력]

**외부 API**
AWS S3, AWS SQS, AWS Secrets Manager, AWS ECR, AWS ECS Fargate, AWS CloudWatch, Jira REST API, OpenAI API (GPT-5.2 / GPT-5 mini)

**배포 자동화**
Jenkins, Docker, ECR, ECS Fargate
- GitLab develop → release-* 브랜치 생성 시 webhook 트리거
- Backend/Frontend: Docker 이미지 빌드 → EC2 docker compose pull & up → Nginx reload
- Playwright Worker: Docker 이미지 빌드 → ECR 푸시 → ECS Task Definition 업데이트 → ECS 서비스 재배포
- Playwright Worker (ECS Fargate): 유휴 시 0대, SQS 메시지 기반 최대 4대 자동 확장
- AI GPU Server (EC2 g7e.2xlarge): 유휴 시 0대, 요청 시 기동, prebaked AMI로 콜드스타트 최소화
- Terraform으로 모든 AWS 리소스 코드로 관리

**테스트 계획/결과**
k6 부하 테스트: Smoke (프로젝트 조회·잡 생성 API), 전체 파이프라인 동시 실행 흐름, WebSocket 다중 접속, N+1 성능 테스트
E2E 기능 검증: AutoQA 자체 실행 (dog-fooding)
결과: [직접 입력]

**서비스 URL**
https://www.autoqa.site
