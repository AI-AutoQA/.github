# Playwright SQS 요청 검증 예시

이 문서는 배포된 Playwright Worker가 SQS 요청을 정상 처리하는지 확인하기 위한 실전 예시입니다. 현재 운영 흐름에서 Playwright가 직접 처리하는 SQS 요청은 두 가지입니다.

1. `PAGE_ANALYSIS`: URL을 받아 사이트 탐색과 요소 분석을 수행하고 분석 산출물을 S3에 업로드
2. `QA_EXECUTE`: S3에 저장된 `final-report.json`과 QA suite JSON을 받아 웹 QA 실행 수행

`SCENARIO_GENERATE`는 Playwright가 아니라 AI Worker가 처리합니다.

---

## 사전 조건

Playwright Worker 컨테이너/EC2에는 아래 환경변수가 있어야 합니다.

| 변수 | 설명 |
|---|---|
| `ANALYZE_QUEUE_URL` 또는 `SQS_ANALYZE_QUEUE_URL` | `PAGE_ANALYSIS` 수신 큐 |
| `EXECUTE_QUEUE_URL` 또는 `SQS_EXECUTE_QUEUE_URL` | `QA_EXECUTE` 수신 큐 |
| `RESULT_QUEUE_URL` 또는 `SQS_RESULT_QUEUE_URL` | 처리 결과 발행 큐 |
| `S3_BUCKET` | 분석/실행 산출물 버킷 |
| `AWS_REGION` | 예: `ap-northeast-2` |

Worker IAM 권한에는 최소한 아래 동작이 필요합니다.

- analyze/execute queue: `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`
- result queue: `sqs:SendMessage`
- artifact bucket: `s3:GetObject`, `s3:PutObject`

---

## 로컬 수동 검증 빠른 시작

Backend를 거치지 않고 로컬 터미널에서 직접 SQS 메시지를 넣어 검증할 때는 아래 순서대로 실행합니다. 예시는 Git Bash 기준입니다.

서버 환경 상태와 로그는 [CloudWatch 서버 디버그 대시보드](cloudwatch-server-debug-dashboard.md)를 함께 열어두고 확인합니다. 대시보드 이름은 `autoqa-server-debug`입니다.

### 0. AWS 로그인과 배포 값 확인

```bash
export AWS_PROFILE=autoqa-main
export AWS_REGION=ap-northeast-2

aws sts get-caller-identity \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION"

cd /c/iforcu/S14P31A101/infra/terraform/main

export SQS_ANALYZE_QUEUE_URL="$(terraform output -raw analyze_queue_url)"
export SQS_EXECUTE_QUEUE_URL="$(terraform output -raw execute_queue_url)"
export SQS_RESULT_QUEUE_URL="$(terraform output -raw result_queue_url)"
export S3_BUCKET="$(terraform output -raw artifacts_bucket)"
export ECS_CLUSTER="$(terraform output -raw ecs_cluster_name)"
export ECS_SERVICE="$(terraform output -raw ecs_worker_service_name)"

printf 'analyze=%s\nexecute=%s\nresult=%s\nbucket=%s\n' \
  "$SQS_ANALYZE_QUEUE_URL" \
  "$SQS_EXECUTE_QUEUE_URL" \
  "$SQS_RESULT_QUEUE_URL" \
  "$S3_BUCKET"
```

CI/CD가 ECR에 새 이미지를 올린 직후라면 최신 이미지와 ECS 서비스 상태만 수동으로 확인합니다.

```bash
aws ecr describe-images \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --repository-name autoqa/playwright \
  --query 'sort_by(imageDetails,&imagePushedAt)[-1].{tags:imageTags,digest:imageDigest,pushedAt:imagePushedAt}' \
  --output table

aws ecs describe-services \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --query 'services[0].{desired:desiredCount,running:runningCount,taskDefinition:taskDefinition}' \
  --output table
```

이 Terraform 구성의 Worker 서비스는 기본 `desired_count=0`이고 SQS 메시지가 들어오면 자동 스케일됩니다. 수동 검증 중 바로 워커를 띄워두고 싶으면 아래 명령으로 1개만 임시 기동합니다.

```bash
aws ecs update-service \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --desired-count 1

aws logs tail /autoqa/ecs/worker \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --since 10m \
  --follow
```

CloudWatch Console에서는 Dashboard > `autoqa-server-debug`에서 SQS 적체, ECS Worker 상태, Backend/Worker 로그를 같이 볼 수 있습니다.

검증이 끝나면 비용 방지를 위해 다시 0으로 내려도 됩니다.

```bash
aws ecs update-service \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --desired-count 0
```

### 1. 분석 요청만 먼저 보내기

```bash
export JOB_ID="manual-analysis-$(date +%Y%m%d-%H%M%S)"

cat > /tmp/page-analysis-message.json <<JSON
{
  "jobType": "PAGE_ANALYSIS",
  "jobId": "$JOB_ID",
  "pageUrl": "https://www.naver.com/",
  "crawlOptions": {
    "maxPages": 3,
    "maxDepth": 2,
    "maxParallelPages": 2,
    "requestDelayMs": 750,
    "blockResources": false,
    "respectRobotsTxt": true
  },
  "requestedAt": "$(date -u +%Y-%m-%dT%H:%M:%S.000Z)",
  "traceId": "$JOB_ID"
}
JSON

aws sqs send-message \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --queue-url "$SQS_ANALYZE_QUEUE_URL" \
  --message-body file:///tmp/page-analysis-message.json
```

분석 결과는 아래 S3 prefix에 생기는지 확인합니다.

```bash
aws s3 ls "s3://$S3_BUCKET/reports/$JOB_ID/analysis/" \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --recursive
```

### 2. 첨부 JSON으로 QA 실행 요청 보내기

첨부로 받은 NAVER `final-report.json`, `output2.json`을 S3에 올린 뒤 `QA_EXECUTE`를 직접 보냅니다.

```bash
export JOB_ID="manual-execute-$(date +%Y%m%d-%H%M%S)"
export ANALYSIS_KEY="reports/$JOB_ID/analysis/final-report.json"
export SUITE_KEY="reports/$JOB_ID/scenario/analysis-summary_qa_scenarios.json"

aws s3 cp "c:/Users/SSAFY/Downloads/인풋/final-report.json" \
  "s3://$S3_BUCKET/$ANALYSIS_KEY" \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION"

aws s3 cp "c:/Users/SSAFY/Downloads/output2.json" \
  "s3://$S3_BUCKET/$SUITE_KEY" \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION"

cat > /tmp/qa-execute-message.json <<JSON
{
  "jobType": "QA_EXECUTE",
  "jobId": "$JOB_ID",
  "pageUrl": "https://www.naver.com/",
  "analysisS3Url": "s3://$S3_BUCKET/$ANALYSIS_KEY",
  "suiteS3Url": "s3://$S3_BUCKET/$SUITE_KEY",
  "testBrowser": "chromium",
  "executionOptions": {
    "headless": true,
    "maxParallelScenarios": 1
  },
  "requestedAt": "$(date -u +%Y-%m-%dT%H:%M:%S.000Z)",
  "traceId": "$JOB_ID:QA_EXECUTE"
}
JSON

aws sqs send-message \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --queue-url "$SQS_EXECUTE_QUEUE_URL" \
  --message-body file:///tmp/qa-execute-message.json
```

실행 결과는 아래 S3 prefix와 result queue에서 확인합니다.

```bash
aws s3 ls "s3://$S3_BUCKET/reports/$JOB_ID/execute/" \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --recursive

aws sqs receive-message \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --queue-url "$SQS_RESULT_QUEUE_URL" \
  --wait-time-seconds 20 \
  --max-number-of-messages 10 \
  --visibility-timeout 30 \
  --query 'Messages[].Body' \
  --output text
```

Backend가 동시에 result queue를 consume 중이면 CLI에서 결과 메시지를 못 볼 수 있습니다. 그 경우 S3 `reports/$JOB_ID/execute/suite-report.json`와 CloudWatch `/autoqa/ecs/worker` 로그를 기준으로 확인합니다.

QA 실행 증적은 `execute/` 아래에 함께 업로드되어야 합니다.

```text
reports/{jobId}/execute/suite-report.json
reports/{jobId}/execute/scenarios/{scenarioId}/result.json
reports/{jobId}/execute/scenarios/{scenarioId}/screenshots/fail-*.png
reports/{jobId}/execute/scenarios/{scenarioId}/videos/*.webm
```

`suite-report.json`에는 각 시나리오의 `videoS3Url`, `videoArtifact`, `executionArtifacts`, `failureArtifacts`와 실패 step의 `artifacts[].s3Url`이 포함됩니다. 영상 파일이 생성되지 않고 `ffmpeg` 관련 오류가 보이면 Playwright Worker 이미지에 ffmpeg가 포함되어 있는지 먼저 확인합니다.

### 3. 큐에 남은 메시지와 DLQ 확인

```bash
aws sqs get-queue-attributes \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --queue-url "$SQS_EXECUTE_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible

aws sqs get-queue-attributes \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --queue-url "$SQS_ANALYZE_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

---

## 1. 분석 요청 보내기

Backend의 현재 DTO는 `jobType=PAGE_ANALYSIS`를 발행합니다. Playwright Worker는 기존 문서의 `PAGE_DISCOVER`도 호환하지만, 신규 요청은 `PAGE_ANALYSIS`를 기준으로 맞춥니다.

### 요청 JSON

```json
{
  "jobType": "PAGE_ANALYSIS",
  "jobId": "naver-20260430-001",
  "pageUrl": "https://www.naver.com/",
  "crawlOptions": {
    "maxPages": 3,
    "maxDepth": 2,
    "maxParallelPages": 2,
    "maxParallelPreflightChecks": 8,
    "requestDelayMs": 750,
    "blockResources": false,
    "respectRobotsTxt": true
  },
  "requestedAt": "2026-04-30T00:00:00.000Z",
  "traceId": "naver-20260430-001"
}
```

### AWS CLI 전송

```bash
export AWS_REGION=ap-northeast-2
export SQS_ANALYZE_QUEUE_URL="https://sqs.ap-northeast-2.amazonaws.com/<account-id>/autoqa-analyze-queue"

cat > /tmp/page-analysis-message.json <<'JSON'
{
  "jobType": "PAGE_ANALYSIS",
  "jobId": "naver-20260430-001",
  "pageUrl": "https://www.naver.com/",
  "crawlOptions": {
    "maxPages": 3,
    "maxDepth": 2,
    "maxParallelPages": 2,
    "maxParallelPreflightChecks": 8,
    "requestDelayMs": 750,
    "blockResources": false,
    "respectRobotsTxt": true
  },
  "requestedAt": "2026-04-30T00:00:00.000Z",
  "traceId": "naver-20260430-001"
}
JSON

aws sqs send-message \
  --region "$AWS_REGION" \
  --queue-url "$SQS_ANALYZE_QUEUE_URL" \
  --message-body file:///tmp/page-analysis-message.json
```

### 기대 결과 이벤트

성공 시 Worker는 result queue에 아래 형태의 이벤트를 보냅니다.

```json
{
  "eventVersion": "1.0",
  "producer": "playwright-worker",
  "eventType": "PAGE_ANALYSIS_DONE",
  "jobType": "PAGE_ANALYSIS",
  "status": "PASSED",
  "jobId": "naver-20260430-001",
  "analysisS3Url": "s3://<bucket>/reports/naver-20260430-001/analysis/final-report.json",
  "briefS3Url": "s3://<bucket>/reports/naver-20260430-001/analysis/analysis-summary.json",
  "summaryS3Url": "s3://<bucket>/reports/naver-20260430-001/analysis/site-summary.json",
  "reportUrl": "s3://<bucket>/reports/naver-20260430-001/analysis/final-report.json"
}
```

Backend는 `analysisS3Url`, `briefS3Url`, `summaryS3Url`을 pipeline state에 저장한 뒤 AI 시나리오 생성 단계로 넘깁니다.

---

## 2. QA 실행 요청 보내기

QA 실행은 SQS 메시지에 큰 JSON 본문을 직접 넣지 않고, S3 주소 두 개만 넣습니다.

| 필드 | 의미 |
|---|---|
| `analysisS3Url` | Playwright 분석 결과 `final-report.json` S3 주소 |
| `suiteS3Url` | AI가 생성한 QA scenario suite JSON S3 주소 |

Playwright Worker는 호환을 위해 `analysisReportS3Url`, `finalReportS3Url`, `qaSuiteS3Url`, `scenarioSuiteS3Url`도 읽을 수 있지만, 표준 필드는 `analysisS3Url`, `suiteS3Url`입니다.

현재 AI 샘플 UI는 입력 파일명에서 `.json`을 제거하고 `_qa_scenarios.json`을 붙여 다운로드합니다. `analysis-summary.json`을 입력했다면 `analysis-summary_qa_scenarios.json`이 자연스러운 S3 객체명입니다. 확장자 없이 `analysis-summary_qa_scenarios`로 저장해도 S3 object body가 JSON이면 됩니다. Playwright는 파일명을 고정하지 않고 `suiteS3Url`이 가리키는 JSON을 그대로 읽습니다.

### 데이터드리븐 + 재사용 flow suite 예시

SQS 메시지는 `suiteS3Url`만 전달합니다. 아래처럼 S3에 올리는 suite JSON 내부에 `flows`와 `dataSets`를 넣으면 Worker가 실행 전에 실제 scenario/step으로 확장합니다.

```json
{
  "suiteId": "naver-search-data-driven",
  "title": "NAVER 검색어별 결과 확인",
  "environment": {
    "baseURL": "https://www.naver.com"
  },
  "flows": {
    "search.basic": [
      {
        "stepId": "fill",
        "type": "fill",
        "targetRef": { "nodeId": "p001_www_naver_com_:node-search" },
        "input": { "valueTemplate": "${flow.keyword}" }
      },
      { "stepId": "submit", "type": "press", "key": "Enter" }
    ]
  },
  "dataSets": {
    "keywords": [
      { "id": "autoqa", "label": "AutoQA", "data": { "keyword": "AutoQA" } },
      { "id": "playwright", "label": "Playwright", "data": { "keyword": "Playwright" } }
    ]
  },
  "scenarios": [
    {
      "scenarioId": "NAVER-SEARCH",
      "title": "검색어별 검색 결과 페이지 진입",
      "dataSetRef": "keywords",
      "preconditions": [
        { "stepId": "open", "type": "goto", "url": "/" }
      ],
      "steps": [
        { "stepId": "search", "type": "flow", "flowRef": "search.basic", "data": { "keyword": "${data.keyword}" } },
        { "stepId": "assert-url", "type": "expect", "assertion": { "matcher": "toContainURL", "value": "search" } }
      ]
    }
  ]
}
```

위 suite는 실행 시 `NAVER-SEARCH__autoqa`, `NAVER-SEARCH__playwright` 두 시나리오로 확장됩니다. 전체 확장 수와 row 메타데이터는 실행 후 S3의 `suite-report.json`에서 `executionPlan`, `scenarioReports[].dataDriven`으로 확인합니다.

### 샘플 JSON을 S3에 업로드

첨부로 받은 NAVER 샘플을 직접 검증하려면 로컬 파일을 먼저 S3에 올립니다.

```bash
export AWS_REGION=ap-northeast-2
export S3_BUCKET="<artifact-bucket>"
export JOB_ID="naver-20260430-001"

aws s3 cp "c:/Users/SSAFY/Downloads/인풋/final-report.json" \
  "s3://$S3_BUCKET/reports/$JOB_ID/analysis/final-report.json" \
  --region "$AWS_REGION"

aws s3 cp "c:/Users/SSAFY/Downloads/output2.json" \
  "s3://$S3_BUCKET/reports/$JOB_ID/scenario/analysis-summary_qa_scenarios.json" \
  --region "$AWS_REGION"
```

Git Bash에서 Windows 경로가 잘 안 먹으면 `/c/Users/SSAFY/Downloads/...` 형태로 바꿔서 실행합니다.

### 요청 JSON

```json
{
  "jobType": "QA_EXECUTE",
  "jobId": "naver-20260430-001",
  "pageUrl": "https://www.naver.com/",
  "analysisS3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/analysis/final-report.json",
  "suiteS3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/scenario/analysis-summary_qa_scenarios.json",
  "testBrowser": "chromium",
  "executionOptions": {
    "headless": true,
    "maxParallelScenarios": 1
  },
  "requestedAt": "2026-04-30T00:05:00.000Z",
  "traceId": "naver-20260430-001:QA_EXECUTE"
}
```

### AWS CLI 전송

```bash
export SQS_EXECUTE_QUEUE_URL="https://sqs.ap-northeast-2.amazonaws.com/<account-id>/autoqa-execute-queue"

cat > /tmp/qa-execute-message.json <<JSON
{
  "jobType": "QA_EXECUTE",
  "jobId": "$JOB_ID",
  "pageUrl": "https://www.naver.com/",
  "analysisS3Url": "s3://$S3_BUCKET/reports/$JOB_ID/analysis/final-report.json",
  "suiteS3Url": "s3://$S3_BUCKET/reports/$JOB_ID/scenario/analysis-summary_qa_scenarios.json",
  "testBrowser": "chromium",
  "executionOptions": {
    "headless": true,
    "maxParallelScenarios": 1
  },
  "requestedAt": "2026-04-30T00:05:00.000Z",
  "traceId": "$JOB_ID:QA_EXECUTE"
}
JSON

aws sqs send-message \
  --region "$AWS_REGION" \
  --queue-url "$SQS_EXECUTE_QUEUE_URL" \
  --message-body file:///tmp/qa-execute-message.json
```

### 기대 결과 이벤트

```json
{
  "eventVersion": "1.0",
  "producer": "playwright-worker",
  "eventType": "QA_EXECUTE_DONE",
  "jobType": "QA_EXECUTE",
  "status": "PASSED",
  "jobId": "naver-20260430-001",
  "analysisS3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/analysis/final-report.json",
  "suiteS3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/scenario/analysis-summary_qa_scenarios.json",
  "resultS3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/execute/suite-report.json",
  "reportUrl": "s3://<artifact-bucket>/reports/naver-20260430-001/execute/suite-report.json",
  "artifactPrefix": "s3://<artifact-bucket>/reports/naver-20260430-001/execute/",
  "failureArtifacts": [
    {
      "type": "screenshot",
      "scenarioId": "NAVER-SEARCH-002",
      "s3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/execute/scenarios/NAVER-SEARCH-002/screenshots/fail-01-s1.png"
    },
    {
      "type": "video",
      "scenarioId": "NAVER-SEARCH-002",
      "s3Url": "s3://<artifact-bucket>/reports/naver-20260430-001/execute/scenarios/NAVER-SEARCH-002/videos/<video>.webm"
    }
  ],
  "summary": {
    "totalScenarios": 4,
    "passed": 0,
    "partial": 0,
    "failed": 4,
    "assertionPassedCount": 0,
    "assertionFailedCount": 0
  }
}
```

`status`는 `PASSED`, `PARTIAL`, `FAILED`, `ERROR` 중 하나입니다. `QA_EXECUTE`에서 `PARTIAL`도 terminal done 이벤트로 보고 `QA_EXECUTE_DONE`으로 발행합니다.

---

## 3. Result Queue 확인

```bash
export SQS_RESULT_QUEUE_URL="https://sqs.ap-northeast-2.amazonaws.com/<account-id>/autoqa-result-queue"

aws sqs receive-message \
  --region "$AWS_REGION" \
  --queue-url "$SQS_RESULT_QUEUE_URL" \
  --wait-time-seconds 20 \
  --max-number-of-messages 1
```

운영 Backend consumer는 result message 처리와 Redis/STOMP 반영이 끝난 뒤에만 delete 해야 합니다. CLI로 수동 확인할 때는 메시지를 지우지 않는 편이 안전합니다.

---

## 4. Backend API와 SQS 매핑

현재 Backend API 기준 흐름은 아래와 같습니다.

| Backend API | 내부 동작 | 발행 SQS |
|---|---|---|
| `POST /api/qa/start` | `PageAnalysisMessage` 생성 | `PAGE_ANALYSIS` -> analyze queue |
| `POST /api/qa/analyze` | 기존 분석 S3 URL로 AI 시나리오 생성 수동 트리거 | `SCENARIO_GENERATE` -> AI queue |
| `POST /api/qa/execute` | Redis pipeline state의 `analysisS3Url`, `suiteS3Url`을 읽어 실행 | `QA_EXECUTE` -> execute queue |

`POST /api/qa/execute`는 요청 본문으로 S3 URL을 직접 받지 않습니다. Backend가 `PAGE_ANALYSIS_DONE`, `SCENARIO_GENERATE_DONE` result event를 처리하면서 Redis pipeline state에 저장한 값을 사용합니다.

```json
{
  "reportId": "naver-20260430-001",
  "testBrowser": "chromium"
}
```

수동으로 배포 Worker만 검증할 때는 Backend API를 거치지 않고 위의 AWS CLI 방식으로 SQS에 직접 넣으면 됩니다.

---

## 5. 빠른 판정 체크리스트

- Worker 로그에 `SQS:analyze 폴링 시작`, `SQS:execute 폴링 시작`이 모두 보이는가?
- `PAGE_ANALYSIS` 전송 후 result queue에 `PAGE_ANALYSIS_DONE` 또는 `PAGE_ANALYSIS_FAILED`가 들어오는가?
- 성공 이벤트에 `analysisS3Url`, `briefS3Url`, `summaryS3Url`이 있는가?
- `QA_EXECUTE` 전송 전 `analysisS3Url`, `suiteS3Url` 두 객체가 실제 S3에서 읽히는가?
- `QA_EXECUTE` 성공/부분성공/실패 이벤트에 `resultS3Url`이 있는가?
- Backend Redis pipeline state에 `analysisS3Url`, `suiteS3Url`, `resultS3Url`이 단계별로 반영되는가?
