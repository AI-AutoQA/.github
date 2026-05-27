# CloudWatch 서버 디버그 대시보드

운영 서버 환경에서 서비스별 상태와 로그를 한 화면에서 확인하기 위한 CloudWatch Dashboard입니다. Terraform 리소스 이름은 `aws_cloudwatch_dashboard.server_debug`이고, 대시보드 이름은 `autoqa-server-debug`입니다.

## 생성과 접속

```bash
cd /c/iforcu/S14P31A101/infra/terraform/main

terraform plan
terraform apply

terraform output -raw cloudwatch_dashboard_url
```

AWS Console에서 직접 열 때는 CloudWatch > Dashboards > `autoqa-server-debug`로 들어갑니다.

## 포함된 영역

| 영역 | 확인 목적 |
|---|---|
| Operational Alarms | SQS 적체, DLQ, ECS CPU, RDS 메모리, Worker scale in/out 상태 |
| SQS Backlog / Age | analyze, execute, ai, result, dlq 큐의 적체와 오래된 메시지 확인 |
| ECS Worker | Playwright Worker task 수, CPU, Memory 확인 |
| RDS PostgreSQL | DB CPU, 연결 수, 가용 메모리, 가용 스토리지 확인 |
| AI EC2 | AI 인스턴스 CPU, status check, 네트워크 사용량 확인 |
| AI Queue and Starter Lambda | AI queue 적체와 EC2 start Lambda 호출/오류 확인 |
| NAT Gateway | ECS Worker가 외부 URL/S3/ECR/SQS로 나갈 때 NAT 트래픽과 오류 확인 |
| External EC2 Host Metrics | Backend/Jenkins/Nginx가 있는 외부 EC2의 CPU, Memory, Disk 확인 |
| Logs Insights | Worker, Backend, Nginx, Lambda, system 로그의 오류와 job 이벤트 검색 |

Lambda 로그 그룹 `/aws/lambda/autoqa-ai-starter`는 대시보드에서 빈 로그 위젯 오류가 나지 않도록 Terraform에서 미리 생성합니다. 아직 AI queue가 트리거되지 않았다면 로그 스트림은 없을 수 있지만, 위젯 자체는 정상 표시됩니다.

## SQS 수동 검증 중 보는 순서

1. `SQS Backlog by Queue`에서 analyze/execute queue에 메시지가 들어왔는지 확인합니다.
2. `ECS Worker Task Count`에서 Worker가 0에서 1 이상으로 올라오는지 확인합니다.
3. `Playwright Worker Errors / Job Events`에서 `jobId` 또는 `traceId`가 찍히는지 확인합니다.
4. `Result Queue and DLQ`에서 result queue에 이벤트가 생기고 DLQ가 0인지 확인합니다.
5. result queue를 Backend가 먼저 consume했다면 S3 `reports/{jobId}/...` 산출물과 Backend 로그를 함께 확인합니다.

## 자주 쓰는 Logs Insights 쿼리

CloudWatch Console에서 Logs Insights를 직접 열 때는 제목 아래에 적힌 로그 그룹을 먼저 선택한 뒤 쿼리 본문만 실행합니다. 대시보드 위젯은 로그 그룹을 포함한 `SOURCE 'log-group' | ...` 형식으로 저장합니다.

### Playwright Worker에서 특정 job 추적

로그 그룹: `/autoqa/ecs/worker`

```sql
fields @timestamp, @message
| filter @message like /manual-execute-20260430/ or @message like /QA_EXECUTE/
| sort @timestamp desc
| limit 100
```

### Worker 오류만 보기

로그 그룹: `/autoqa/ecs/worker`

```sql
fields @timestamp, @message
| filter @message like /ERROR|Error|error|FAILED|failed|Exception/
| sort @timestamp desc
| limit 100
```

### Backend SQS consumer와 Redis 반영 확인

로그 그룹: `/autoqa/ec2/backend`

```sql
fields @timestamp, @message
| filter @message like /PAGE_ANALYSIS|SCENARIO_GENERATE|QA_EXECUTE|SQS|Redis|reportId|jobId/
| sort @timestamp desc
| limit 100
```

### Nginx 4xx/5xx 확인

로그 그룹: `/autoqa/ec2/nginx`

```sql
fields @timestamp, @message
| filter @message like / 4\d\d / or @message like / 5\d\d /
| sort @timestamp desc
| limit 100
```

### AI EC2 자동 시작 Lambda 확인

로그 그룹: `/aws/lambda/autoqa-ai-starter`

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 100
```

## 주의사항

- 외부 EC2 지표와 `/autoqa/ec2/*` 로그는 CloudWatch Agent가 실행 중이어야 보입니다.
- AI EC2가 stopped 상태면 EC2 CPU/네트워크 지표가 비어 보일 수 있습니다. AI queue에 메시지를 넣으면 starter Lambda가 인스턴스를 시작합니다.
- result queue는 Backend가 consume할 수 있으므로 수동 CLI에서 메시지가 보이지 않아도 실패로 단정하지 않습니다. S3 산출물과 Backend 로그를 함께 봅니다.
- 대시보드는 관측용입니다. 큐 메시지 삭제, DLQ redrive, ECS desired count 변경은 별도 AWS CLI나 Console에서 신중하게 수행합니다.