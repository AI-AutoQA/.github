# QA 파이프라인 SQS 연동 규격

이 문서는 Backend 개발 팀에 전달할 **QA 파이프라인 큐 계약**입니다.  
현재 서비스는 URL 하나를 입력받으면 Playwright가 설정된 최대 탐색 페이지까지 직접 순회하고, 각 페이지의 요소 분석까지 수행합니다. 따라서 페이지별 `PAGE_ELEMENT_DISCOVER` 작업으로 쪼개지 않고, 아래 3단계만 표준 계약으로 둡니다.

```text
1. PAGE_ANALYSIS      Backend -> Playwright : URL 1개를 받아 사이트 탐색 + 요소 분석
2. SCENARIO_GENERATE  Backend -> AI         : 분석 결과를 받아 QA Suite 생성
3. QA_EXECUTE         Backend -> Playwright : QA Suite 전체 실행

완료/실패 이벤트는 모두 result queue로 회신
```

---

## 핵심 원칙

| 원칙 | 설명 |
|---|---|
| URL 하나가 분석 입력 | Backend는 분석 시작 시 `pageUrl` 하나와 `crawlOptions`만 전달합니다. |
| Playwright가 탐색 책임 | Playwright Worker가 `maxPages`, `maxDepth` 범위 안에서 페이지 탐색과 요소 분석을 함께 수행합니다. |
| AI는 시나리오만 생성 | AI Worker는 Playwright 분석 결과를 받아 QA Suite JSON만 생성합니다. |
| QA 실행은 suite 단위 | Backend는 기본적으로 시나리오를 SQS 단위로 쪼개지 않고 QA Suite 전체 실행을 요청합니다. |
| 완료/실패는 result queue | Redis Pub/Sub은 실시간 UX 용도이고, terminal event의 기준은 SQS result queue입니다. |
| 큰 산출물은 S3 | 분석 결과, QA Suite, 실행 리포트, 영상은 S3에 저장하고 SQS에는 URL만 담습니다. |

---

## 큐 구성

| 단계 | LocalStack 큐 | AWS 큐 | Producer | Consumer | 역할 |
|---|---|---|---|---|---|
| 사이트 분석 | `qa-analysis-queue` | `autoqa-analyze-queue` | Backend | Playwright Worker | `PAGE_ANALYSIS` 처리 |
| 시나리오 생성 | `qa-ai-queue` | `autoqa-ai-queue` | Backend | AI Worker | `SCENARIO_GENERATE` 처리 |
| QA 실행 | `qa-execute-queue` | `autoqa-execute-queue` | Backend | Playwright Worker | `QA_EXECUTE` 처리 |
| 결과 이벤트 | `qa-result-queue` | `autoqa-result-queue` | Playwright Worker, AI Worker | Backend EC2 | `*_DONE`, `*_FAILED` 수신 |
| 실패 보관 | `qa-dlq` 또는 `autoqa-dlq` | `autoqa-dlq` | SQS redrive | 운영자 | 재처리 실패 메시지 보관 |

---

## ID 규칙

| 필드 | 필수 | 의미 | 권장 사용처 |
|---|---|---|---|
| `jobId` | 예 | 사용자가 생성한 QA 리포트/파이프라인 전체 ID | Redis 채널, 화면 상태, 최종 리포트 묶음, SQS 작업 추적 |

본 계약에서는 Backend의 기존 리포트 식별자 값을 그대로 `jobId`로 사용합니다. 현재처럼 한 job이 `PAGE_ANALYSIS` 1개, `SCENARIO_GENERATE` 1개, `QA_EXECUTE` 1개를 순서대로 수행한다면 Backend는 `jobId + jobType` 조합으로 각 단계를 구분할 수 있습니다.

추가적인 하위 작업 ID가 필요해지는 경우에도 SQS payload 필드를 늘리기보다 `attempt`, `runIndex` 같은 보조 필드를 추가하는 것을 권장합니다.

| 상황 | 예시 |
|---|---|
| 같은 단계 재시도 이력을 분리해야 할 때 | `attempt: 2` |
| 같은 job에서 QA 실행을 여러 번 수행할 때 | `runIndex: 3` |
| CloudWatch/SQS 로그에서 메시지 단위 추적이 필요할 때 | `traceId: "job-20260428-001:QA_EXECUTE:run-3"` |

---

## 환경변수 계약

### Backend EC2

| 변수 | 설명 |
|---|---|
| `SQS_ANALYZE_QUEUE_URL` | `PAGE_ANALYSIS` 발행 대상 |
| `SQS_AI_QUEUE_URL` | `SCENARIO_GENERATE` 발행 대상 |
| `SQS_EXECUTE_QUEUE_URL` | `QA_EXECUTE` 발행 대상 |
| `SQS_RESULT_QUEUE_URL` | 완료/실패 이벤트 consume 대상 |
| `S3_BUCKET` | 분석/시나리오/실행 산출물 저장 버킷 |

### Playwright Worker

| 변수 | 설명 |
|---|---|
| `ANALYZE_QUEUE_URL` | `PAGE_ANALYSIS` consume |
| `EXECUTE_QUEUE_URL` | `QA_EXECUTE` consume |
| `RESULT_QUEUE_URL` | 완료/실패 이벤트 publish |
| `S3_BUCKET` | 산출물 저장 버킷 |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` | 실시간 진행 이벤트 publish 대상 |

### AI Worker

| 변수 | 설명 |
|---|---|
| `SQS_AI_URL` | `SCENARIO_GENERATE` consume |
| `SQS_RESULT_URL` | 완료/실패 이벤트 publish |
| `S3_BUCKET` 또는 이에 준하는 설정 | 입력/출력 산출물 저장소 |

---

## 공통 메시지 규칙

모든 SQS 메시지는 JSON object이며 필드명은 **camelCase**를 사용합니다.

### 작업 요청 공통 필드

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `jobType` | string | 예 | `PAGE_ANALYSIS`, `SCENARIO_GENERATE`, `QA_EXECUTE` 중 하나. Playwright Worker는 과거 명칭 `PAGE_DISCOVER`도 호환 처리합니다. |
| `jobId` | string | 예 | QA job/report/pipeline 식별자 |
| `requestedAt` | ISO 8601 string | 권장 | Backend 발행 시각 |
| `traceId` | string | 권장 | 로그 추적용 correlation id. 기본값은 `jobId` 권장 |

### 결과 이벤트 공통 필드

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `eventVersion` | string | 예 | 이벤트 스키마 버전. 현재 `1.0` |
| `producer` | string | 예 | `playwright-worker` 또는 `ai-worker` |
| `eventType` | string | 예 | `PAGE_ANALYSIS_DONE` 같은 라우팅 키 |
| `jobType` | string | 예 | 원 작업 종류 |
| `status` | string | 예 | `PASSED`, `FAILED`, `ERROR` |
| `jobId` | string | 예 | QA job/report/pipeline 식별자 |
| `durationMs` | number | 권장 | 처리 시간 |
| `completedAt` | ISO 8601 string | 권장 | 작업 완료 시각 |
| `emittedAt` | ISO 8601 string | 예 | 이벤트 발행 시각 |
| `errorMessage` | string | 실패 시 | 실패 사유 |

---

## 1. PAGE_ANALYSIS

URL 하나를 입력받아 사이트 탐색과 페이지별 요소 분석을 한 번에 수행합니다.

| 항목 | 값 |
|---|---|
| 발행 큐 | `SQS_ANALYZE_QUEUE_URL` |
| 소비자 | Playwright Worker |
| 성공 이벤트 | `PAGE_ANALYSIS_DONE` |
| 실패 이벤트 | `PAGE_ANALYSIS_FAILED` |

### 요청 payload

```json
{
  "jobType": "PAGE_ANALYSIS",
  "jobId": "job-20260428-001",
  "pageUrl": "https://www.example.com",
  "crawlOptions": {
    "maxPages": 20,
    "maxDepth": 3,
    "maxParallelPages": 3,
    "requestDelayMs": 750,
    "blockResources": false,
    "respectRobotsTxt": true
  },
  "requestedAt": "2026-04-28T05:00:00.000Z",
  "traceId": "job-20260428-001"
}
```

- `crawlOptions.respectRobotsTxt`는 기본값이 `true`입니다.
- `true`이면 같은 origin의 `robots.txt`를 조회하고 명시적인 `Disallow` 대상 URL은 분석 큐에 넣지 않습니다.
- 내부/소유 사이트처럼 정책상 허용된 대상에서만 `false`로 내려 기존 방식처럼 `robots.txt` 게이트 없이 탐색합니다.
- `crawlOptions.requestDelayMs`는 같은 frontier 안에서 페이지 분석 시작 시점을 분산하는 rate limit 옵션입니다. 기본값은 `750`이며, `0`이면 비활성화합니다.
- CAPTCHA, Turnstile, Cloudflare browser verification 등 자동화 챌린지가 감지되면 Playwright는 우회/자동 풀이를 시도하지 않고 해당 페이지를 `manual_verification_required`로 종료합니다.
- 비시각 대량 점검에서만 `crawlOptions.blockResources=true` 또는 Worker 환경변수 `BLOCK_ANALYSIS_RESOURCES=true`를 사용해 이미지/폰트/미디어 요청을 차단할 수 있습니다. 기본값은 `false`입니다.

### 성공 이벤트 payload

```json
{
  "eventVersion": "1.0",
  "producer": "playwright-worker",
  "eventType": "PAGE_ANALYSIS_DONE",
  "jobType": "PAGE_ANALYSIS",
  "status": "PASSED",
  "jobId": "job-20260428-001",
  "pageUrl": "https://www.example.com",
  "durationMs": 42000,
  "analysisS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/final-report.json",
  "reportUrl": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/final-report.json",
  "summary": {
    "totalPagesAnalyzed": 12,
    "totalPagesFailed": 0,
    "totalManualVerificationRequired": 0,
    "totalNodes": 1360,
    "totalInteractiveCandidates": 184
  },
  "pages": [
    {
      "pageIndex": 0,
      "pageUrl": "https://www.example.com",
      "artifactUrl": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/pages/0/final-report.json"
    }
  ],
  "completedAt": "2026-04-28T05:00:42.000Z",
  "emittedAt": "2026-04-28T05:00:42.100Z"
}
```

Backend 처리 기준:

- Redis pipeline 상태를 `SCENARIO_GENERATING` 전 단계로 전이합니다.
- `analysisS3Url` 또는 `reportUrl`, `briefS3Url`, `summaryS3Url`을 pipeline state에 저장합니다.
- 이후 `briefS3Url`, `summaryS3Url`을 담은 `SCENARIO_GENERATE` 작업을 AI queue로 발행합니다.

---

## 2. SCENARIO_GENERATE

Playwright 분석 결과를 기반으로 AI Worker가 QA Suite JSON을 생성합니다.

| 항목 | 값 |
|---|---|
| 발행 큐 | `SQS_AI_QUEUE_URL` |
| 소비자 | AI Worker |
| 성공 이벤트 | `SCENARIO_GENERATE_DONE` |
| 실패 이벤트 | `SCENARIO_GENERATE_FAILED` |

### 요청 payload

```json
{
  "jobType": "SCENARIO_GENERATE",
  "jobId": "job-20260428-001",
  "briefS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/analysis-summary.json",
  "summaryS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/site-summary.json",
  "requestedAt": "2026-04-28T05:01:00.000Z",
  "traceId": "job-20260428-001"
}
```

### 성공 이벤트 payload

```json
{
  "eventVersion": "1.0",
  "producer": "ai-worker",
  "eventType": "SCENARIO_GENERATE_DONE",
  "jobType": "SCENARIO_GENERATE",
  "status": "PASSED",
  "jobId": "job-20260428-001",
  "suiteS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/scenario/analysis-summary_qa_scenarios.json",
  "scenarioCount": 12,
  "completedAt": "2026-04-28T05:01:35.000Z",
  "emittedAt": "2026-04-28T05:01:35.100Z"
}
```

AI 생성 파일명은 S3 key에 고정하지 않아도 되지만, 현재 AI 샘플 UI는 입력 파일명에서 `.json`을 뺀 뒤 `_qa_scenarios.json`을 붙여 다운로드합니다. 예를 들어 `analysis-summary.json`을 입력하면 `analysis-summary_qa_scenarios.json`이 자연스러운 산출물 이름입니다. 확장자 없이 `analysis-summary_qa_scenarios`로 저장해도 S3 object body가 JSON이면 Playwright는 정상 파싱합니다. Backend와 Playwright는 파일명을 추론하지 말고 AI가 반환한 `suiteS3Url`을 그대로 저장하고 전달합니다.

Backend 처리 기준:

- `suiteS3Url`을 pipeline state에 저장합니다.
- report 상태를 `READY_TO_EXECUTE`로 전이합니다.
- 사용자가 실행을 요청하면 `QA_EXECUTE` 작업을 execute queue로 발행합니다.

---

## 3. QA_EXECUTE

AI가 생성한 QA Suite 전체를 Playwright Worker가 실행합니다.

| 항목 | 값 |
|---|---|
| 발행 큐 | `SQS_EXECUTE_QUEUE_URL` |
| 소비자 | Playwright Worker |
| 성공 이벤트 | `QA_EXECUTE_DONE` |
| 실패 이벤트 | `QA_EXECUTE_FAILED` |

### 요청 payload

```json
{
  "jobType": "QA_EXECUTE",
  "jobId": "job-20260428-001",
  "pageUrl": "https://www.example.com",
  "analysisS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/analysis/final-report.json",
  "suiteS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/scenario/analysis-summary_qa_scenarios.json",
  "testBrowser": "chromium",
  "executionOptions": {
    "headless": true,
    "maxParallelScenarios": 3
  },
  "requestedAt": "2026-04-28T05:02:00.000Z",
  "traceId": "job-20260428-001"
}
```

현재 Playwright 로컬 실행기는 `suite`와 `analysisReport` inline payload도 지원합니다. AWS 운영 계약은 SQS payload 크기 제한을 피하기 위해 `suiteS3Url`, `analysisS3Url`을 기본으로 둡니다.

### QA Suite JSON 확장 필드

SQS payload에는 큰 JSON을 넣지 않고 `suiteS3Url`만 전달하지만, 해당 S3 객체의 suite JSON은 아래 확장 필드를 포함할 수 있습니다.

| 필드 | 위치 | 설명 |
|---|---|---|
| `flows` | suite top-level | 반복 step 묶음. `flowRef`에서 참조 |
| `dataSets` | suite top-level | 데이터드리븐 row 묶음. `dataSetRef`에서 참조 |
| `flowRef`, `flowRefs` | scenario | scenario 시작부에 reusable flow를 삽입 |
| `dataSetRef` | scenario | dataSet row마다 scenario를 확장 |
| `dataRows` | scenario | 특정 scenario 전용 inline row |
| `type: "flow"` | step | 실행 전 `suite.flows`의 concrete step 배열로 확장 |

Playwright Worker는 `QA_EXECUTE` 처리 시 `flow` 확장 후 `dataSet` 확장을 수행하고, 확장된 suite를 대상으로 validation과 실행을 진행합니다. 따라서 SQS 계약은 그대로 유지되며, 변경점은 `suiteS3Url`이 가리키는 QA Suite JSON 계약에만 반영됩니다.

### 성공 이벤트 payload

```json
{
  "eventVersion": "1.0",
  "producer": "playwright-worker",
  "eventType": "QA_EXECUTE_DONE",
  "jobType": "QA_EXECUTE",
  "status": "PASSED",
  "jobId": "job-20260428-001",
  "durationMs": 175551,
  "resultS3Url": "s3://autoqa-artifacts-prod/reports/job-20260428-001/execute/suite-report.json",
  "reportUrl": "s3://autoqa-artifacts-prod/reports/job-20260428-001/execute/suite-report.json",
  "artifactPrefix": "s3://autoqa-artifacts-prod/reports/job-20260428-001/execute/",
  "summary": {
    "totalScenarios": 12,
    "passed": 6,
    "partial": 2,
    "failed": 4,
    "assertionPassedCount": 27,
    "assertionFailedCount": 4
  },
  "completedAt": "2026-04-28T05:04:55.551Z",
  "emittedAt": "2026-04-28T05:04:55.650Z"
}
```

Backend 처리 기준:

- `resultS3Url` 또는 `reportUrl`을 최종 실행 결과로 저장합니다.
- summary를 바탕으로 report 상태를 `COMPLETED`, `PARTIAL`, `FAILED` 중 하나로 확정합니다.
- Frontend에는 Redis/STOMP로 terminal event와 실행 요약을 전달합니다.

`QA_EXECUTE_DONE.summary`는 목록 화면용 compact summary입니다. 상세 분석은 `resultS3Url`의 `suite-report.json`을 기준으로 조회합니다.

| suite-report 필드 | 설명 |
|---|---|
| `executionPlan` | 전처리 여부, flow 확장 수, data-driven 확장 수, 실행 scenario 수 |
| `flakeAnalytics` | suite-level flaky 위험도, retry/locator/transient error 신호 집계 |
| `scenarioReports[].dataDriven` | data-driven row에서 생성된 scenario의 원본 scenarioId, dataSetRef, rowId |
| `scenarioReports[].summary.flakeAnalytics` | scenario-level flaky 위험도와 신호 |
| `scenarioReports[].stepResults[].resolutionResult` | adaptive locator 선택 결과, fallback 여부, confidence/reason 정보 |

---

## 실패 이벤트 공통 payload

실패 이벤트는 성공 이벤트와 같은 기본 필드를 사용하고 `eventType`, `status`, `errorMessage`를 포함합니다.

```json
{
  "eventVersion": "1.0",
  "producer": "playwright-worker",
  "eventType": "PAGE_ANALYSIS_FAILED",
  "jobType": "PAGE_ANALYSIS",
  "status": "ERROR",
  "jobId": "job-20260428-001",
  "pageUrl": "https://www.example.com",
  "durationMs": 5200,
  "errorMessage": "Navigation timeout exceeded",
  "completedAt": "2026-04-28T05:00:05.200Z",
  "emittedAt": "2026-04-28T05:00:05.250Z"
}
```

Backend 처리 기준:

- `*_FAILED`는 orchestration control event로 처리합니다.
- result queue 메시지는 Redis/STOMP 반영과 상태 저장이 끝난 뒤 delete 합니다.
- 재시도는 Backend가 새 작업 메시지를 다시 발행하는 방식으로 관리합니다.

---

## Redis 채널 매핑 권장안

| Result eventType | Redis 채널 |
|---|---|
| `PAGE_ANALYSIS_*` | `qa:report:{jobId}:page-discover` |
| `SCENARIO_GENERATE_*` | `qa:report:{jobId}:scenario-generate` |
| `QA_EXECUTE_*` | `qa:report:{jobId}:execute` |

진행률 이벤트는 Worker가 Redis에 직접 publish해도 됩니다. 다만 완료/실패 이벤트는 result queue를 통해 Backend가 받아 Redis로 중계하는 것을 기준으로 합니다.

---

## Backend 구현 체크리스트

- `PAGE_ANALYSIS`는 `SQS_ANALYZE_QUEUE_URL`로 발행합니다.
- `SCENARIO_GENERATE`는 `SQS_AI_QUEUE_URL`로 발행합니다.
- `QA_EXECUTE`는 `SQS_EXECUTE_QUEUE_URL`로 발행합니다.
- `SQS_RESULT_QUEUE_URL` long polling consumer를 추가합니다.
- result queue 메시지는 처리 성공 후에만 delete 합니다.
- `eventType` 기준으로 기존 Redis report 채널에 재발행합니다.
- terminal event는 Redis Pub/Sub만 믿지 않고 SQS result queue를 source of truth로 봅니다.
- SQS payload에는 큰 분석 결과나 QA Suite를 직접 넣지 않고 S3 URL만 넣습니다.
- 모든 S3 URL은 Backend가 읽을 수 있는 IAM 권한 범위 안에 있어야 합니다.

---

## 현재 적용 상태

| 영역 | 상태 |
|---|---|
| Terraform result queue | 추가됨 |
| Playwright `ANALYZE_QUEUE_URL`/`EXECUTE_QUEUE_URL` poll | 추가됨 |
| Playwright `RESULT_QUEUE_URL` publish | 추가됨 |
| AI EC2 `SQS_RESULT_URL` 전달 | 추가됨 |
| Backend result queue consumer | 구현됨 (page-analysis/scenario/execute consumer) |
| Backend `SCENARIO_GENERATE` -> AI queue 라우팅 | 구현됨 (`briefS3Url` + `summaryS3Url`) |
| Playwright S3 URL 기반 QA 실행 입력 로딩 | 구현됨 (`analysisS3Url` + `suiteS3Url`) |

---

## 확장 시나리오

아래 기능은 지금 표준 계약에 넣지 않습니다. 운영 중 필요가 생기면 별도 버전에서 추가합니다.

| 확장 항목 | 추가 시점 |
|---|---|
| `PAGE_ELEMENT_DISCOVER` 페이지별 분산 분석 | 페이지 수가 많아 분석 시간을 Fargate 태스크 여러 개로 분산해야 할 때 |
| 시나리오 단위 `QA_EXECUTE` | suite 전체 실행 시간이 길어 시나리오별 병렬 분산이 필요할 때 |
| page/scenario별 retry queue | 부분 재실행 정책이 Backend에 명확히 생겼을 때 |